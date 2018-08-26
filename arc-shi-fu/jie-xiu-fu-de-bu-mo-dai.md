# 簡介修飾符的內部模擬代碼

此頁的內容主要參考至[這篇](https://juejin.im/post/5a29fbf15188257dd2398df9) ，希望我有理解正確並且給予夠清楚的資訊。

## \_\_strong

宣告時

```objectivec
id obj = [[NSObject alloc] init];
```

編譯時的模擬代碼

```objectivec
id obj = objc_msgSend(NSObject, @selector(alloc));
objc_msgSend(obj, @selector(init));
objc_release(obj);
```

編譯時會拆成 2 次的 objc\_msgSend \( alloc / init \)，結束時使用 objc\_release 釋放。

如果使用的是 alloc / new / copy / mutableCopy 以外的方法，像是

```objectivec
id __strong obj = [NSMutableArray array];
```

編譯器的模擬代碼會是

```objectivec
id obj = objc_msgSend(NSMutableArray, @selector(array));
objc_retainAutoreleasedReturnValue(obj); //主要差異點
objc_release(obj);
```

所以說這個  `objc_retainAutoreleasedReturnValue()` 的作用是什麼呢？

這個得和 `objc_autoreleaseReturnValue()` 一併了解。

### objc\_retainAutoreleasedReturnValue\(\) 與  objc\_autoreleaseReturnValue\(\)

在[修飾符的章節](./#fang-fa-fan-hui-de-zhi)中提到的 函數返回值會預設給予 \_\_autoreleasing 的修飾符，所以 `[NSMutableArray array]` 編譯時會像是這樣子的：

```objectivec
+ (id) array {
    id obj = obj_msgSend(NSArray, @selector(alloc));
    objc_msgSend(obj, @selector(init));
    //在回傳時 “若有必要”，會使用 autorelease 延長生命週期
    return objc_autoreleaseReturnValue(obj); 
}
```

所以在外部調用 `array` 方法時，就會使用 `objc_retainAutoreleasedReturnValue(obj)`，持有這個 autorelease 物件。

但  `objc_autoreleaseReturnValue()` 其實有更近一步的優化，它會判斷是否在調用後緊接著調用了 `objc_retainAutoreleasedReturnValue()`。如果是，便不會把物件存入 autorelease pool 中，而會直接傳遞到函數的調用方。通過這兩者的協作，將函數調用最佳化。

## \_\_weak

{% hint style="info" %}
在這裡只整理粗淺的實現邏輯。實際對於 \_\_weak 如何實現的分析，可以看看[這篇](https://juejin.im/post/58ffe5fb5c497d0058158fee)。（~~到 TaggedPoint 我的小白腦就超頻了~~）
{% endhint %}

先複習兩個 \_\_weak 修飾符的特性：

*  \_\_weak 修飾的變量，在引用物件釋放後，會被置為 nil.
* 使用 \_\_weak 修飾的變量，是使用註冊到 autorelease pool 中的物件。

宣告時

```objectivec
id __weak obj1 = obj;
```

編譯時會像是

```objectivec
id obj1;
objc_initWeak(&obj1, obj); //初始化 __weak 修飾符變量
objc_detroyWeak(&obj1); //作用域結束，摧毀 obj1 的指針變量
```

好的一行一行來分解

首先，`objc_initWeak(&obj1, obj)`，等同於

```objectivec
obj1 = 0;
objc_storeWeak(&obj1, obj); 
```

再來，`objc_destroyWeak(&obj)`，等同於

```objectivec
objc_storeWeak(&obj, 0)
```

統整一下，所以編譯時代碼我們可以這樣看

```objectivec
id obj1;
obj1 = 0;
objc_storeWeak(&obj1,obj);
objc_storeWeak(&obj1,0);
```

所以我們只要搞懂 `objc_storeWeak` 的用途是什麼就行。

### objc\_storeWeak

weak 相關的引用儲存在一個全局的 hash table 裡。以物件地址作為 key，並且對應參照的弱引用變量地址們。

objc\_storeWeak 將輸入的第二參數的物件地址作為 key，將第一個帶有 `__weak` 修飾符的變量註冊到表中。如果第二參數為 0，則變量地址會從 weak 表刪除。

由這個邏輯不難想像，大量使用 `__weak` 修飾符，會在需要銷毀物件時進行 hash table 的檢查，以刪除對應的地址。這會消耗相對應的 CPU 資源做 hash table 裡的 deallocate，因此我們盡量只在避免 retain cycle 時使用 `__weak`。

這裡特別要拿兩個易混淆的 case 來做比較

```objectivec
id __weak obj = [[NSObject alloc] init];
NSLog(@"%@", obj); // (null)
```

```objectivec
id __weak obj1 = obj;
NSLog(@"%@", obj1); //有值
```

第一個 case 編譯器會警告你「Assigning retained object to weak variable; object will be released after assignment」，因為藉由 alloc + init，是將自己生成並持有的物件賦值給 `__weak` 修飾符的變量中，但該變量不能持有這個物件，所以會直接被釋放。

第二個 case 是使用 `__weak` 修飾符的變量，就是使用 autorelease pool 中的物件（[這裡有提到](./#weak-xiu-de-liang)）。編譯器的模擬代碼如下:

```objectivec
id obj1;
objc_initWeak(&obj, obj);
//取出有 __weak 修飾的變量所引用的物件並且 retain
id tmp = objc_loadWeakRetained(&obj); 
//將物件註冊到 autoreleasepool
objc_autorelease(tmp);
NSLog(@"%@", tmp);
objc_destroyWeak(&obj1);
```

所以這個 case 能夠成功傳值，是因為編譯器將物件註冊進 autorelease pool 中的緣故。

## \_\_autoreleasing

帶有 \_autoreleasing 修飾符的變量等同於調用物件的 autorelease 方法

```objectivec
@autoreleasepool {
    id __autoreleasing obj = [[NSObject alloc] init];
}
```

編譯器模擬代碼為

```objectivec
id pool = objc_autoreleasePoolPush();
id obj = objc_msgSend(NSObject,@selector(alloc));
objc_msgSend(obj,@selector(init));
objc_autoreleas(obj);
objc_autoreleasePoolPop(pool);
```

如果是非 alloc / new / copy / mutableCopy 的方法

```objectivec
@autoreleasepool{
	id __autoreleasing obj = [NSMutableArray array];
}
```

編譯器模擬代碼為

```objectivec
id pool = objc_autoreleasePoolPush();
id obj = objc_msgSend(NSMutableArray,@selector(array));
objc_retainAutorelesedReturnedValue(obj);
objc_autorelease(obj);
objc_autoreleasePoolPop(pool);
```

