# 寫在2026年的macOS輸入法開發規範

InputMethodKit 自 macOS 10.5 Leopard 時代問世，早於 Objective-C ARC 技術、XPC 通訊技術、Sandbox 技術問世（均為 macOS 10.7）之前。自然，這也是早於 Swift 5 與 SwiftUI 流行之前。也就是說，InputMethodKit 是橫跨了兩代技術大變革的祖產級 OS Framework。當年 Apple 寫給 macOS 10.5 Leopard 的 IMK 參考手冊《[Input Method Kit Framework Reference](https://leopard-adc.pepas.com/documentation/Cocoa/Reference/InputMethodKitFrameworkRef/InputMethodKitFrameworkRef.pdf)》（下文簡稱《IMKFR》）早已不符合這些變革所帶來的新要求（特別是 Swift 6 Concurrency）。筆者根據自己開發[《唯音輸入法》(for macOS 10.09 Mavericks ~ macOS 26)](https://vchewing.github.io/)的經驗，將一些注意事項整理在此，留給其他想給 macOS 開發輸入法的工程師們參考。

> 筆者另外製作了 [IMKSwift](https://github.com/vChewing/IMKSwift) 套件，允許 Swift 工程師們在寫 IMK 輸入法時更順利：IMKSwift 提供了 IMKInputSessionController 基礎型別、是在 IMKInputController 的基礎上整體換用了對 Modern Swift Concurrency 更友好的 ObjC Header 表達。使用這個套件的話，下文某些繁文縟節或可不必嚴苛遵守。

## 1. NSConnection 名稱

《IMKFR》沒提及，但正確答案只有一個：輸入法的 `Info.plist` 的 `InputMethodConnectionName` 欄位只能填寫 `$(PRODUCT_BUNDLE_IDENTIFIER)_Connection`。

> ⚠️ **這是 macOS 10.7 Lion 開始對 NSConnection 的命名規範**。
>
> 不按照這個規範命名的話，你的輸入法在開啟 Sandbox 之後，可能就會在使用者嘗試切換到該輸入法的時候無法正常載入。此時可以在 `Console.app` 內觀測到與 NSConnection 有關的失敗訊息。

當年由 Apple 同步提供的「NumberInput」這個範例專案就給了[錯誤示範](https://github.com/pkamb/NumberInput_IMKit_Sample/blob/6c37ea05d85d0b7b5af9378a0ce88e191ca07241/NumberInput/main.m#L53-L55)，誤導了全球的 macOS 輸入法開發者們。官方誤導，最為致命。

![image](./macOS_Input_Method_Development_Guidelines_2026-illust1.png)

Apple 甚至都不得不給那些沒開 Sandbox 的輸入法們開小灶、允許它們在使用非正規命名的 NSConnection 名稱的前提下繼續正常工作。但這被某些輸入法開發者們錯誤地視為「Sandbox 開了反而會壞事」。

## 2. Sandbox Entitlements

一定要開 Sandbox。macOS 輸入法只要開了 Sandbox，就在**原理上**絕對無法拿到系統全局鍵盤權限了。**你的輸入法因為系統框架限制的原因，不得不用 NSConnection 這麼脆弱的東西，再不開 Sandbox 的話，就等於北港香爐人人插**。

「Sandbox 支援」對一款 macOS 輸入法而言，堪稱對使用者的最佳的資安投名狀。

> 於是剩下的幾乎都是不敢開 Sandbox 的輸入法了：或有技術難題，或支支吾吾。

Sandbox 權能檔案的定義如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.apple.security.app-sandbox</key>
  <true/>
  <key>com.apple.security.files.bookmarks.app-scope</key>
  <true/>
  <key>com.apple.security.files.user-selected.read-write</key>
  <true/>
  <key>com.apple.security.network.client</key>
  <true/>
  <key>com.apple.security.temporary-exception.files.home-relative-path.read-write</key>
  <array>
    <string>/Library/Preferences/$(PRODUCT_BUNDLE_IDENTIFIER).plist</string>
  </array>
  <key>com.apple.security.temporary-exception.mach-register.global-name</key>
  <string>$(PRODUCT_BUNDLE_IDENTIFIER)_Connection</string>
  <key>com.apple.security.temporary-exception.shared-preference.read-only</key>
  <string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
</dict>
</plist>
```

可以看到這裡將輸入法自身的 UserDefaults 拉入白名單了。這是必需的，因為 macOS 的輸入法做了 Sandbox 處理之後確實會喪失對自身 UserDefaults 的存取能力。

## 3. MainActor 約束與 Swift 6.2+

整個 IMKInputController 所有 API 交互都是走 MainActor 的。但是，InputMethodKit 曝露出來的 Header 與 Swift Concurrency 不相容，導致你在使用時反而無法將 IMKInputController 釘死在 MainActor 上。

讓 InputMethodKit 與 Swift 6 Concurrency 相容性最佳的處理方法就是將整個 target 的 default isolation 設為 MainActor。這樣雖然也難免需要對 IMKInputController 的 API 呼叫處理過程實施一些硬 Hack，但這算是相對而言工作量最小的。

你先引入這兩個 extension API：

```swift
extension IMKInputController {
  nonisolated fileprivate func wrap(_ object: Any?) -> UInt? {
    guard let object = object as? AnyObject else { return nil }
    return UInt(bitPattern: Unmanaged.passUnretained(object).toOpaque())
  }

  nonisolated fileprivate func unwrap(_ addr: UInt?) -> Any? {
    guard let addr = addr, let ptr = UnsafeMutableRawPointer(bitPattern: addr) else { return nil }
    return Unmanaged<AnyObject>.fromOpaque(ptr).takeUnretainedValue()
  }
}
```

再使用這個 MainSync API（有經過處理，防止俄羅斯套娃 DeadLock）：
```swift
@discardableResult
public func mainSync<T>(execute work: @MainActor () throws -> T) rethrows -> T {
  if Thread.isMainThread {
    return try work()
  }
  return try DispatchQueue.main.sync(execute: work)
}
```

然後，這是範本，專門示範怎樣將 API 的參數翻譯到 MainActor 上：
```swift
/// nonisolated 是 IMKStateSetting & IMKMouseHandling 協定要求的。
/// 或者說，官方沒要求，但是是 Swift 相容性沒做好導致的現狀。
@objc(MyIMKInputController) // 必須加上 ObjC，因為 IMK 是用 ObjC 寫的。
nonisolated public final class MyIMKInputController: IMKInputController, @unchecked Sendable {

  @objc(handleEvent:client:)
  nonisolated override public func handle(
    _ event: NSEvent?,
    client sender: Any?
  )
    -> Bool {
    let eventRef = wrap(event)
    let senderRef = wrap(sender)
    return mainSync {
      let clientOnMain = unwrap(senderRef)
      let eventOnMain = unwrap(eventRef)
      // 此處存放業務邏輯。
    }
  }
}
```

可能有人注意到筆者將 `MyIMKInputController` 定義為 Sendable 了。不然 mainSync 無效。

## 4. IMKInputController 該脫手的任務一定要脫手

有些輸入法難免會在 activateServer 階段引入與 client() 有關的交互，但這個開銷可能在所難免，因為你可能必須得對 client 使用 `client.overrideKeyboard()` 套用指定的 Ukelele 佈局。再加上 client() 身為 IMKTextInput Client 沒有真正意義上的 Async API，輸入法開發者只能假設所有這類 Client 的這些操作都是 MainActor 阻塞操作，然後乾瞪眼。

於是乎，除了 `client()?.setMarkedText` 與 `client()?.insertText` 以外，其餘的 client methods 應該是都可以在 MainActor 上面 Async 脫手操作的。只要你嚴格按照前文所述將 IMKInputController 所有 API 交互都釘死在 MainActor 上，你就不用擔心脫手操作所帶來的亂序的問題。

> 注意：`client()?` 是 MainActor 限定物件。你脫手可以，但脫手操作的 Lambda Expression 在呼叫 client() 方法時必須在 MainActor 上。

## 5. IMKInputController 必須是 MRC 管理且以極性路由

### 5.1 IMKServer 的 Controller 洩漏缺陷（逆向工程發現）

藉由對 InputMethodKit 在 macOS 10.9 Mavericks、10.15 Catalina 及 macOS 27 GoldenGate (arm64e) 的逆向工程調查，唯音輸入法專案發現了 IMKServer 內部 controller 生命週期管理的根本缺陷：

1. IMKServer 內部以 `_private._controllers`（`NSMutableDictionary`）管理所有 `IMKInputController` 實例，key 為 `[NSNumber numberWithUnsignedLong:clientProxyAddr]`——即每次 init 時傳入的 client proxy 物件的記憶體位址。

2. **每次 CapsLock 切換都會建立全新的 DO/XPC proxy 物件**，其記憶體位址與舊 proxy 不同。舊 controller 的 dictionary key 永遠無法再被 lookup 命中——IMKServer 原廠的 controller 複用邏輯以 proxy 位址為鍵，CpLk 切換後該鍵已失效。舊 controller 成為**永不被清理的孤棄 controller**。

3. IMKServer 雖提供 `sessionFinished:` 清理機制，但該方法是 per-client-proxy 的通知：CpLk 切換後舊 proxy 已被取代，IMKServer 不再為舊 proxy 派送任何訊息，`sessionFinished:` 永遠不會對孤棄 controller 觸發。**每次 CpLk 切換都產生一個永久洩漏的 controller。**

4. 孤棄 controller 持有的 client wrapper（macOS ≤10.15 為 `IPMDServerClientWrapper`；macOS 15 Sequoia 起拆分為 `_IPMDServerClientWrapperModern` 與 `_IPMDServerClientWrapperLegacy`）包含 `_xpcConnection`（NSXPCConnection）或 `_clientDOProxy`——這些連線資源同樣永久洩漏。IMK 以全域快取管理這些 wrapper，必須透過 private class method（`+terminateForClientXPCConn:` / `+terminateForClientDOProxy:`）才能將其移除。

5. macOS 15 Sequoia 起已移除 `+terminateForClient:`（無後綴版本），僅保留 XPC 與 DO 兩變體。

6. ARC 模式下，retain/release 的高頻開銷在 CapsLock 快速切換時導致可感知的卡頓。停用 ARC（`-fno-objc-arc`）可徹底消除卡頓，但必須自行處理孤棄 controller 與殘留 XPC 連線的清理。

### 5.2 修復方案：MRC + KVC Prune + 極性雙緩衝 Session 池

基於上述發現，唯音輸入法專案採用了以下架構（代號「Phase 115」）：

**a) IMK 交互層全面採用 MRC。** `IMKInputSessionController` target 以 `-fno-objc-arc` 編譯。所有 controller 的 alloc/dealloc/retain/release 均在 ObjC MRC 層完成。Swift 端僅透過 raw 記憶體位址（`uintptr_t`）與 controller 互動，不執行任何 Swift ARC 操作。

**b) 以 KVC 清理孤棄 controller。** 每個 controller 在 init 時被分配一個單調遞增的 generation number。當 `_controllers` dictionary 超過 2 個條目時，自動找出 generation 最舊的非活躍 controller，透過 KVC（`[server valueForKeyPath:@"_private._controllers"] removeObjectForKey:oldKey]`）將其移除，使其在 MRC 下正常釋放。

**c) 極性雙緩衝 Session 池。** 以 controller generation number 的奇偶性（`generation % 2`）將所有 controller 對映至兩枚 static `InputSession` singleton：偶數→session A，奇數→session B。每次 CpLk toggle 帶來的新 controller 自動對接至**另一個** session；上一個 session 保留不釋放，等待下一個同極性 controller 接手。此舉完全消除了 InputSession 的動態分配、LRU 淘汰邏輯，以及快取過期的風險。

**d) 延遲釋放與 XPC 清理。** controller 在 `deactivateServer:` 後不會立即釋放，而是排程 3 秒後執行清理：釋放所有 blocks、呼叫 client wrapper 的 private `+terminateForClient*` class method（依序嘗試 `_IPMDServerClientWrapperModern` → `_IPMDServerClientWrapperLegacy` → `IPMDServerClientWrapper` 以向前相容 macOS 27）、清除 controller↔session 對照表。若 3 秒內 controller 被重新啟用（`activateServer:`），則取消排程、恢復正常服務。這確保了 CpLk 快速切換時不會出現「舊 controller 已死、新 controller 尚未就緒」的真空期。

**e) Dangling pointer 防護。** `IMKControllerLifetimeTracker` singleton 在每次以 `takeUnretainedValue()` 解讀 raw 位址前複查 controller 是否仍存活。`unregisterSessionAddr` 僅在 session 仍歸屬於該 controller 時才清空 `inputControllerAssignedAddr`（防止已 reassign 給新 controller 的 session 被舊 controller 的延遲 dealloc 誤清）。`reassign` 同步清除舊 controller 在 `sessionAddrByControllerAddr` 中的殘留 mapping。

**f) IME 選單注入。** `IMEMenuSputnik` 利用 ObjC Runtime 的 `class_addMethod` + `imp_implementationWithBlock` 將 Swift closure 直接註冊為 `IMKInputSessionController` 的 method IMP，無需 `@objc` Selector 暴露即可動態構建輸入法選單。選單注入兩項即時監測資料：(1) `IMKControllerLifetimeTracker` 所追蹤的 controller 存活數量；(2) 以 `task_vm_info.internal` 取得的 anonymous private memory 用量。選單開啟前會先呼叫 `purgeMallocZones()` 強制回收 allocator cache，以確保記憶體讀數真實。

### 5.3 架構總結

ARC 從 IMK dispatch 路徑中完全移除，CapsLock 卡頓徹底消除。兩枚 singleton session 取代 LRU 快取，極性路由消除了快取複雜度。KVC prune 路徑補償了 IMKServer 無法自行清理孤棄 controller 的缺陷。延遲釋放確保 XPC 連線被正確終止而非洩漏。

以下為 controller 側架構的最小化示意：

```swift
// IMKInputSessionController（MRC 編譯的 .m 檔案）：
// - Class-level static blocks 透過 raw address 向 Swift 分派。
// - +IMKSwift_configureWithActivatingServer: 等在啟動時一次性設定。

// Swift 端——極性路由：
public final class InputSession {
  /// 兩枚預先配置的 singleton session。
  fileprivate static let sessionEven = InputSession(preallocated: ())
  fileprivate static let sessionOdd  = InputSession(preallocated: ())

  /// 依 controller 的 generation parity 解析對應 session。
  static func session(for ctlAddr: UInt) -> InputSession? {
    let parity = IMKControllerLifetimeTracker.shared().generationForAddress( ctlAddr) % 2
    return parity == 0 ? sessionEven : sessionOdd
  }
}

// SessionControllerSputnik.callCoreAtLeastOnce:
// 1. 由 controller generation 決定 parity。
// 2. 路由至 sessionEven 或 sessionOdd。
// 3. 重新指派 client provider（無快取查詢、無淘汰邏輯）。
```

> **注意：** 極性池刻意不以 client 位址為鍵。它接受 CapsLock 切換會產生不同位址的新 proxy 這一現實，改以 controller generation parity——一個穩定、單調遞增的計數器——作為路由鍵。兩個 session 的上限是充分的，因為同時最多只有兩個 controller 可能活躍（當前的一個與正在被 deactivate 的一個）。

## 6. 將輸入法所有程式內容寫成 Swift Package Library

macOS 的輸入法無法用 breakpoint 等方式偵錯，因為會無限凍結任何沾過你的輸入法的 clients，進而凍結你的整個桌面，最終得依賴外部 SSH 連到你的電腦上殺掉輸入法執行緒才行。你需要自己寫單元測試搭配自己寫的 mockup client 來測試。這樣的話，將輸入法的所有業務內容寫成 Library 會更便於這種偵錯，還能允許開發者靈活地指定專用的 UserDefaults 容器來實現封閉測試。更甚者，你還可以寫個標準的 AppKit App 模擬這個單元測試打字過程，然後用 Instruments 監測是否有記憶體洩漏。這遠比僅保留一個輸入法本體 Xcode Target 要靈活得多。

## 7. 記憶體佔用量自查自糾，必要時自盡以釋放記憶體

使用者電腦的記憶體空間寸土寸金。雖然 macOS 26 的 AppKit 糟糕的 NSWindow 繪製效率導致一款輸入法平均佔用的運存可能從 80MB 暴漲到 200MB 左右。但筆者在這裡介紹的一個設計應該不壞：讓輸入法每次 activateServer 切換到新的打字會話的時候，檢查輸入法自身佔用的記憶體。如果發現佔用的記憶體的量超過 1024MB 的話，就讓輸入法拋出 NSNotification 使用者通知之後自盡。這個 NSNotification 使用者通知的內容就是告知這個情況，免得使用者以為輸入法崩潰掉。

當然，這個技巧只是兜底策略、防止在使用者的電腦上發生像是「記憶體用盡」這樣的災難性的後果。但開發者仍有義務主動檢查自己寫的東西是否有記憶體洩漏的危險。

> ⚠️ 如果你的輸入法有在用 SQLite 的話，需要額外注意一個冷門常識：用 SQLite 跑完每一筆查詢之後一定要用 `sqlite3_finalize(StatementPointer)` 釋放記憶體，不然會產生**連 Xcode Instruments 都抓不到**的記憶體洩漏。

## 8. 讓輸入法用到的 NSWindow 數量盡可能地少

這一條是針對 macOS 26 開始的現狀而不得已的規範，因為：自 macOS 26 開始，只要是 NSWindow 用過的記憶體空間，就都不會被系統刻意回收掉 NSWindow 每個副本的基礎開銷、且這個基礎開銷因為 LiquidGlass 的原因而非常高昂。哪怕你確實沒啟用 LiquidGlass 效果，也沒差。在 Info.plist 當中啟用 `UIDesignRequiresCompatibility` 雖然可以讓記憶體佔用量下降到 macOS 15 的水準，但這只是緩兵之計、且 Apple 隨時都會廢掉 `UIDesignRequiresCompatibility` 這個 InfoPlist 屬性。

> 筆者推測：macOS 26 佔用硬碟空間這麼大，很可能是系統卷宗裡面包了一個 macOS 15 AppKit 環境、專門用來對這個 InfoPlist 屬性提供 backward compatibility。

現在 SwiftUI 這麼強了，開發者完全可以考慮將「工具提示 Panel」與「自己搓的選字窗」整合到同一個 NSPanel 裡面，這樣就少了一份 NSWindow 基礎開銷。輸入法的「關於」視窗也可以整入輸入法自身的「偏好設定」裡面。

> NSPanel 是 NSWindow 的變種。

## 9. IMKCandidates 不要用就對了

前文提到的那個 NumberInput 範例都不敢用 IMKCandidates 選字窗，因為 IMKCandidates 就是一包陳年糞便、臭到現在。你看 macOS 26 系統內建的日語輸入法就是 IMKCandidates 的受害者，連文字都看不清：

![image](./macOS_Input_Method_Development_Guidelines_2026-illust2.png)

玻璃背景居然全透明了、把白色整個透上來。偏偏選字窗的文字也是白色的。這種問題一眼看出來就是缺乏單元測試惹的禍，因為這很明顯就是 Liquid Glass API 沒正確使用所導致的。

現在 AI 技術這麼發達，你用 AI 幫你寫一個類似 IMKCandidates 那種佈局的輸入法選字窗面板應該也不難。當然，如果你用強行曝露 IMKCandidates 內部 API 的方式來使用的話，有些 API 從 macOS 10.14 Mojave 開始是固定可用的，但將來就不好說了。

> 筆者給自己開發的唯音輸入法就使用了自己搓的田所選字窗，與 IMK 多行選字窗相比也算提供了比較迫真的體驗。

![image](./macOS_Input_Method_Development_Guidelines_2026-illust3.png)

## 結尾

InputMethodKit 是歷史產物，但它至今仍是 macOS 輸入法唯一的官方入口。既然如此，開發者就必須接受這套框架的歷史包袱，並在其結構性缺陷之上建立自己的工程紀律。

本文所列規範，本質上並非「技巧」，而是一套風險控制模型：將 IMKInputController 變為純轉接層、將業務邏輯完全模組化、將 MainActor 當作不可違抗的事實、將記憶體壓力視為設計輸入條件、將 Sandbox 視為最低限度的道德底線。

若有一天 Apple 徹底重寫 InputMethodKit，這些規範或許會過時；但在那之前，macOS 輸入法若想在 2026 年仍保持工程品質與資安可信度，就必須把「自我約束」寫進架構，而不是寫在 README 裡。

$ EOF.
