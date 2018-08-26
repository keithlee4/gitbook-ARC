---
description: ARC is just for NSObject.
---

# ARC 與 Core Foundation

過去在 MRC 時，Objective-C 和 Core Foundation 類型的物件都是使用相同的 retain / release 規則。但是在進入 ARC 時代後，Objective-C 物件改由 ARC 管理內存，**但 Core Foundation 依然需要手動管理，Core Foundation 不支持 ARC**。

所以使用 Core Foundation 類型時，當要與 Objective-C 類型轉換，就必須考慮用哪一種規則來管理內存。所以針對這個問題，Apple 也提供了在執行類型間的 Toll-Free Bridging 操作時，對應的方法與修飾符，以指明使用哪種規則 \(ARC \|\| MRC\) 管理內存。

這些方法和修飾符包括

* `__bridge`
* `__bridge_retained` or `CFBridgingRetain`
* `__bridge_transfer` or `CFBridingRelease`

### \_\_bridge

只聲明轉換類型，不改變內存管理規則。例如:

```objectivec
CFStringRef str = (__bridge CFStringRef) [[NSString alloc] initWithFormat: @"call me young bridge"];
```

所以 `CFStringRef` 仍然會用 `NSString` 的 ARC 去做管理，無法使用 `CFRelease()` 釋放。

### \_\_bridge\_retained or CFBridgingRetain

類型轉換時，將內存管理從 Objective-C 交給 Core Foundation。**也就是 ARC -&gt; MRC**。

用同個例子來看

```objectivec
NSString *ocStr = [[NSString alloc] initWithFormat: @"call me young bridge"];
CFStringRef cfStr = (__bridge_retained CFStringRef) ocStr;
//也可以使用 CFStringRef cfStr = (CFStringRef)CFBridgingRetain(ocStr);
// ...
// ...

CFRelease(cfStr); //使用完要手動釋放
```

### \_\_bridge\_transfer or CFBridgingRelease

和上述相反，內存管理規則由 Core Foundation 交給 Objective-C，**也就是 MRC -&gt; ARC**。

```objectivec
CFStringRef result = CFURLCreateStringByAddingPercentEscapes(. . .);
NSString *s = (__bridge_transfer NSString *)result;
//也可以寫成 NSString *s = (NSString *)CFBridgingRelease(result);
return s;
```

`result` 的管理責任交給了ARC，不需要再寫 `CFRelease()` 了。

