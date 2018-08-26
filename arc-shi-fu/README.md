# ARC 與 修飾符

在 ARC 中，對於變量的修飾符包含四種：

1. `__strong`
2. `__weak`
3. `__unsafe_unretained`
4. `__autoreleasing`

以下個別做解釋

## \_\_strong

`__strong` ，大家都很熟悉的最基本的 strong，是 ARC 下對於 「**id**」 和「**物件類型**」默認的修飾符，表示強引用，對應 property 裡的 `strong` 。只要有一個 `__strong` 指針指向該物件，物件就會保持存活。**如果需要釋放一個強引用持有的物件，需要將所有強引用指針設為 nil**。這個就不多做著墨。

## \_\_weak

`__weak`，弱引用，對應 property 裡的 `weak`。弱引用不持有物件，當物件被釋放時，指針會被置為 nil，防止野指針（[什麼是野指針？](wild-pointer-ye-zhi.md)）發生。最常用來避免 retain cycle，幾個 `__weak` 常見的使用情境包含：

### Delegate

在 Delegate 中為了避免產生 retain cycle，所以通常對於 delegate 的 property 宣告會像是：

```objectivec
@property (nonatomic, weak) id<Delegate> delegate;
```

weak 修飾符的使用，便是為了確保在對 delegate 賦值時，不會 retain 原本的物件。

### Block

block 會持有使用到的變量，因此可能造成引用循環，常見例子如：

```objectivec
@interface BlockContainer: NSObject
@property (copy) void (^block)(void);
@end

@implementation BlockContainer
- (void)configBlock {
    self.block = {
        //此處 block 會對 self 產生強引用，而 self 又強引用 block，造成 retain cycle。
        [self someAction];
    }
}
@end
```

如果使用 `__weak` 來弱引用 `self` 便能解開這個 retain cycle

```objectivec
@interface BlockContainer: NSObject
@property (copy) void (^block)(void);
@end

@implementation BlockContainer
- (void)configBlock {
    //使用 __weak 修飾符
    __weak typeof(self) weakSelf = self
    self.block = {
        //若 self 在執行時已被釋放，weakSelf 會被置為 nil。
        [weakSelf someAction];
    }
}
@end
```

值得一提的是，其實~~不少人~~（我）在一開始發現 block 和 retain cycle 的關聯後，會一股腦地把所有 block 內部持有的變量都先 weak 一輪，這也不是正確的作法。只要沒有去 retain block，不使用 `weakSelf` 也不會造成 retain cycle。當然，是否使用 weak 還有很多種情境，日後會整理一篇關於 block 的介紹（~~因為還沒準備好~~），`__strong`, `__weak`, `__block` 之於 block 的關係會在那做進一步的說明。

### IBOutlet - Interface Builder 創建的控制元件

第三個情境大家都熟悉 

```objectivec
@property (weak, nonatomic) IBOutlet UIButton *button;
```

當使用 IBOutlet 連結 Interface Builder 生成的元件時，**除了 top level object 以外 \(之後再解釋為什麼，可以先參考** [**Apple 的文檔**](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/LoadingResources/CocoaNibs/CocoaNibs.html#//apple_ref/doc/uid/10000051i-CH4)**）**，都應使用 weak。邏輯上這些子元件都應讓 IB 去持有，在物件裡額外的強引用可能造成元件已經從畫面移除但卻未被釋放的風險。

## \_\_unsafe\_unretained

`__unsafe_unretained` 修飾符不會持有物件，同時，若該物件已被釋放，指針也不會設為 nil。因此如果該位址內容被覆寫，再次訪問該指針時，程序則會崩潰，此便為**野指針**問題。那為什麼不用 `__weak`  就好？其實沒錯，現在不管任何情境中，都會建議使用 `__weak` 而非 `__unsafe_unretained`，會出現這個修飾符，多半為支援 iOS 4.0 （沒錯是 **4.0**）的緣故，因為 `__weak` 在 iOS 5.0 後才引入。（所以現在幾乎不會用到這個修飾符了）

## \_\_autoreleasing

`__autoreleasing` 修飾的物件會註冊到 Autorelease Pool 中，並在 Pool 銷毀時釋放。Property 不能使用該修飾符，因為任何一個 property 都不應該是 autorelease。

ARC 啟用後，幾乎不會需要額外使用 `__autoreleasing`，但這個修飾符也確實在許多地方作用，場景包含：

### 方法返回的值

在方法中返回的值會需要使用 autorelease 機制，讓返回的物件能超出原本的作用域，被呼叫方法的對象接收。舉例來說:

```objectivec
- (NSObject *)object {
    //默認修飾符 __strong
    NSObject *obj = [[NSObject alloc] init];
    //作為返回值時，編譯器會將其加上 __autoreleasing, 註冊到 Autorelease Pool 中
    return obj;
}
```

為什麼需要這個機制？因為如果沒有延長方法內宣告物件的生命週期，返回的值就會是 nil，這沒有意義。所以當物件被當成參數返回，會註冊至 Autorelease Pool 中，Pool 會強引用這個物件，延長其生命週期。那這就會產生另一個思考：要是 Pool 銷毀或釋放了物件，接收方還沒接收到呢？

這個問題牽涉 Autorelease Pool 的運行方式，這會再額外整理。簡短來說，Pool 會在恰當的時間點上執行銷毀，像是：

1. Runloop 每次迭代的時候 \(Runloop 相關概念可以參考[先前的這篇](https://keithlee4.gitbook.io/runloop/)\)
2. GCD 的 block 執行結束時

### 訪問  \_\_weak 修飾的變量

訪問帶有 `__weak` 修飾符的變量時，實際上也會訪問到註冊到 Autorelease Pool  的物件。例如:

```objectivec
id __weak obj1 = obj0;
NSLog(@"%@", [obj1 class]);

id __weak obj1 = obj0;
id __autoreleasing tmp = obj1;
NSLog(@"%@", [tmp class]);
```

兩段代碼效果一樣，因為 `__weak` 是弱引用，在訪問過程物件可能被釋放，因此編譯時編譯器會把該物件註冊到 Autorelease Pool 中，確保在 Pool 銷毀前，該對象存在。

### id 指針

最常見的情境就是使用 error type 的指針作為方法的傳入參數，例如：

```objectivec
NSError *error; 
￼if (![data writeToFile:filename options:NSDataWritingAtomic error:&error]) { 
　　NSLog(@"Error: %@", error); 
}

// 即使上面沒有使用 __autoreleasing 來修飾 error，編譯器也會轉換成如下的邏輯：
NSError *error; 
NSError *__autoreleasing tempError = error; // 額外添加
if (![data writeToFile:filename options:NSDataWritingAtomic error:&tempError]) 
￼{ 
　　error = tempError; // 编译器添加 
　　NSLog(@"Error: %@", error); 
}
```

對於指針類型的參數 \(`id *`\)，ARC 會自動認定為 `__autoreleasing`。所以有些特別的情況需要注意，比如某些 class 的方法裡會**偷偷使用自定義的 Autorelease Pool**，這時便要慎用 `__autoreleasing`。比如 NSDictionary 的 `enumerateKeysAndObjectsUsingBlock`，其實調用時會：

```objectivec
- (void)enumerateDictionary:(NSDictionary *)dict error:(NSError **)error {
    [dict enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
          //這裡其實會創建一個 @autoreleasepool，參照下方代碼
          //如果沒有在參數型別裡聲明 __autorealsing，
          //編譯器也會跳出對應的 warning 告知你該指針會被釋放
          if ( && error != nil) {
                *error = [NSError errorWithDomain:@"MyError" ￼code:1 userInfo:nil];
          }
    }];
￼}

- (void)enumerateDictionary:(NSDictionary *)dict error:(NSError **)error {
    [dict enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
          @autoreleasepool { //隱藏的 Pool
                if ( && error != nil) {
                      *error = [NSError errorWithDomain:@"MyError" ￼code:1 userInfo:nil];
                }
          }
    }];
    
    //因此在這裡時，*error 已經被釋放。
￼}
```

所以如果想要避免指針被釋放，必須要加入一個 strong 的臨時引用。如下：

```objectivec
- (void)enumerateDictionary:(NSDictionary *)dict error:(NSError **)error {
    NSError * __block tmp; //__block 是為了可以在 Block 內被修改
    [dict enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
          if () {
                tmp = [NSError errorWithDomain:@"MyError" ￼code:1 userInfo:nil];
          }
    }];
    
    if (error != nil) {
          *error = tmp;
    }
￼}
```

以上是四個修飾符的粗淺整理，如果想進一步了解各個修飾符在編譯器裡方法調用的內容是什麼？內部實現都做了哪些優化？可以前往[這裏](jie-xiu-fu-de-bu-mo-dai.md)。

