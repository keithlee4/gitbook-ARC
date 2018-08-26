---
description: 野指針 Bug - 除錯裡的地獄級關卡
---

# 野指針 - Wild Pointer

## 野指針是什麼？

野指針 \(Wild Pointer\)，指的是不指向任何合法物件的指針。定義請參照 [wiki](https://zh.wikipedia.org/wiki/%E8%BF%B7%E9%80%94%E6%8C%87%E9%92%88)。

## 產生野指針的可能情境

### 物件釋放後指針沒有置 nil

#### 狀況 1： \_\_unsafe\_unretained（現在應該看不太到）

```objectivec
@property (nonatomic, unsafe_unretained) id obj;
```

使用修飾符 `__unsafe_unretained` 的 `obj`，在前面有提及，並不會在物件釋放後將指針設置為 nil，因此若物件釋放後，持續使用 obj，便可能出現野指針問題。

解決方法很單純：**不要用**。

...

...

我是說改用 `__weak` 來取代這個修飾符。

#### 狀況 2: objc\_setAssociatedObject 中誤用 OBJC\_ASSOCIATION\_ASSIGN 

原本動態擴增的關聯物件，要使用 `OBJC_ASSOCIATION_RETAIN_NONATOMIC` 修飾，卻誤用成 `OBJC_ASSOCIATION_ASSIGN`，造成修飾符宣告成 `__unsafe_unretained` 的型態，和情境 1 狀況類似。

#### 狀況 3: KVO 忘記 removeObserver

在 Notification / KVO 裡 addObserver 要和 removeObserver 成對。原因是如果 observer 是個 `__unsafe_unretained` 指針（通常就是 `self`），引用觀察物件時，若是只有 addObserver 沒有 removeObserver，在物件釋放後，如果 keyPath 發生變化，仍會朝已釋放的觀察者發送通知，就可能會產生野指針。

所以在 addObserver 時也一定要確保在物件釋放前 removeObserver。

### 物件提前被釋放

#### 狀況 1: 非同步的 block callback 裡，沒有使用 strongSelf

這個不要跟之前提到的 block 裡使用 strong 會造成 retain cycle 的情境混淆哦！

先來看這個情境的代碼

```objectivec
__weak typeof(self) weakSelf = self;
[obj method: ^(id result) {
    [weakSelf doSomething];
}];

- (void)doSomething {
    self.someVar = ......;
    ...
}
```

首先先要釐清一個概念，這個 block 如果不是 property 或 ivar，不需使用 weak self。因為 self 並沒有強引用這個 block，使用了 weak 反而會造成潛在的風險。

如上例，method 裡的 block 並沒有被 `self` 持有，但卻在外頭使用了一個弱引用，因此，在**非同步**呼叫 `doSomething()` 這個 function 時，有可能 `self`  已經被釋放，那就造成野指針問題。所以這個狀況應該改寫成：（請記得也不能只寫 `self`，`self` 是 `__unsafe_unretained` 的\)

```objectivec
__weak typeof(self) weakSelf = self;
[obj method: ^(id result) {
    //使用 __strong 修飾符持有 self，確保 self 在這個作用域里一定存在。
    __strong typeof(self) strongSelf = weakSelf
    [strongSelf doSomething];
}];
```

#### 狀況 2: 在 controller dismiss / pop 後持續呼叫 self.method\(\)

```objectivec
@interface ViewController: UIViewController
@end

@implementation ViewController
- (void)dismiss {
    [self.navigationController popViewControllerAnimated: YES];
    //畫面已經 pop 掉，self 可能已被釋放。
    [self doSomething];
}
@end
```

如上例，這個解法其實比較少見（個人經驗），但記得 pop \|\| dismiss 後避免再使用 `self` 即可，這其實也比較合理，畫面都移除了，不應該還有執行該畫面定義的行為。如果一定要這樣使用，就一樣要寫個 strong 來強引用 `self` 了。

### 重複釋放物件

#### 狀況 1: 在多線程中重複 release 相同的物件

情境可能像是

```objectivec
@interface MyClass: NSObject
@property (nonatomic, strong) NSArray *array;
@end

@implementation MyClass
- (void)doSomething {
    dispatch_async(dispatch_get_global_queue((DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        self.array = @[@"wild pointer"];
    });
    
    self.array = @[@"wild pointer"];
}
@end
```

若兩個線程同時對舊有值 release （藉由 `setArray:`），便會釋放兩次。解決方法很單純，在賦值前加上線程鎖即可。簡單說可以換成 `atomic` 屬性，保持 array 的原子性。

#### 狀況 2: 在 Core Foundation 中做了重複釋放

在[這個章節](../arc-core-foundation.md)有提到如果對 Core Foundation 對象使用 `__bridge_transfer` 轉移至 ARC 做內存管理，就不需再額外做 `CFRelease()`。此時要是不慎使用了 `CFRelease()`，便可能導致重複釋放。舉例來說：

```objectivec
CFUUIDRef uuid = CFUUIDCreate(NULL);
CFStringRef cfStr = CFUUIDCreateString(NULL, uuid);
NSString *ocStr = (__bridge_transfer NSString *)cfString;
//或是 NSString *ocStr = (NSString *)CFStringRelease(cfStr);
...
...
CFRelease(cfStr); //重複釋放，可能導致野指針。
```

解決方式，要不就是只使用 `__bridge`，不轉移內存管理的方式，要不就拿掉 `CFRelease(cfStr)`。

