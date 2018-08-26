# 理解 ARC

## 簡介

ARC \(Automatic Reference Counting\) 是一種編譯器的特性, 從 iOS 5.0 後引進作為 Apple 管理記憶體的機制，為 Objective-C 物件在編譯時自動加入對應 retain / release / autorelease，使得開發者（大部分時）毋須再去費神管理記憶體持有與釋放。不過，雖然 Apple 好心地替開發者們 handle 了程式碼的持有與釋放，但了解它「究竟做了什麼」，也能幫助開發的理解（盲目地依賴是危險的）。

這篇便是針對個人對 ARC 的淺見做基礎整理，幾個方向如下：

1. ARC 和過去  MRC 使用的差異
2. \_\_strong / \_\_weak / \_\_unsafe\_unretained / \_\_autoreleasing 標示符
3. 需要手動管理記憶體的情境

本書中所有範例程式碼會以 Objective-C 撰寫，若有任何錯誤煩請不吝指正，感謝。

