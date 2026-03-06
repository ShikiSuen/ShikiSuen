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

## 5. IMKInputController 不要持有任何物件

這一點非常有必要。這裡先給出一個（筆者此前在其他場合提到過的）應用場景：

> macOS 10.12 的這個 CpLk 切換功能的本質不是中英文打字模式切換，而是輸入法切換。macOS 哪怕英文打字也是由一個專門的輸入法負責的。大部分英語鍵盤的電腦上，這個輸入法叫 Apple ABC，對應美規鍵盤。每個輸入法在剛被切換出來時，會觸發這個輸入法自身的 IMKInputController Instance 的創建以及其 activateServer 操作（以及可能有的一系列追加操作）。然後才是這個 Client 之前對接的輸入法的 IMKInputController 副本的 Deactivation。
>
> 很多中英文混合打字的用戶經常會在 ABC 與中文輸入法之間來回切換。由於這種情況下兩者所服務的 IMKTextInput Client 是相同的，所以就出現了 MainActor 塞車。而且，過於高頻的來回切換，會給 IMKInputController 所用的 Objective-C ARC 帶來壓力。ARC 廢件釋放與物件交互都發生在 MainActor 上，必然會發生塞車。
>
> 「在同一個 client 切換輸入法」的過程會牽涉到前後兩個 IMKInputController 副本各自的對 client() 的操作。輸入法開發者現在最佳的範式就是讓 deactivateServer 在 MainActor 上 Async 脫手操作、且不在 deactivateServer 階段做 client() API 的文字寫入/內容顯示交互，因為這種擦除操作會由系統代勞。但是，這個由系統代勞的擦除操作也是發生在 MainActor 上的。這就出現了 MainActor 的任務的時序衝突。InputMethodKit 內部應該是有自己的方式處理這個衝突，然而代價就是阻塞開銷。

這就導致那些經常用 CpLk 超高頻中英切換打字的使用者們必然會罵娘。但他們不知道問題爛在系統層面，於是就只能罵輸入法。或罵系統內建注音爛，或罵自己在用的副廠輸入法不修故障。

雖然目前的自力救濟方法就是「輸入法用戶關掉 macOS 內建的 CpLk 中英文輸入法切換」且「輸入法開發者給自己的輸入法實作原生的 CpLk 英文模式」。但 Apple 的市場策略似乎趨向於「不鼓勵使用者這麼做」。Apple Silicon 筆電剛剛問世時的筆電鍵盤左下角的地球鍵被當作輸入法輪流按鍵，就是這個理想想法的進一步延伸。

於是乎，擺在開發者面前要做的事情還有兩個：其中一個是剛才講過的「該脫手的任務一定要脫手」；而另一個則是： **IMKInputController 不要持有任何物件**。

剛才提到的「在單個 client 接收文字輸入時，用 CpLk 在中英輸入法之間經常切換」的情況當中，為什麼說 client 是相同的呢？因為這個 client 是 IMK 統一派發的 NSConnection Distributed Object，具有記憶體位址一致性。

於是，這裡有個簡單的解法，就是用 NSMapTable 或者任何類似的「弱 Key 雜湊表」：Key 本身是對物件的「弱持有」的；在 Key Object 被析構之後，該 Table 下次自身被存取時，會連同這 Key 帶 Value 一同移除。

這就好辦了：**IMKInputController 不要持有任何物件**。具體的作法是把所有實際的業務邏輯放在一個額外的 Swift 型別（例如本範例裡的 `InputSession`）中，並且只透過弱引用或 Lambda Expression 存取它。控制器自身只負責轉發事件並建立/查詢該業務物件的快取，而絕不直接強持有；這樣每次切換輸入法時，ARC 不會被迫釋放或重建大量物件，且同一個 client 只會對應到一個 Session 物件。下面的範例示範了這種策略——使用 `NSMapTable` 以 client 的分布式物件為鍵，實作一個弱鍵強值的快取，並在 controller 初期化時查詢或建立對應的 `InputSession`。

* `MyIMKInputController.core` 為 `weak`，可在會話結束時自動斷開。  
* `getClientProvider()` 產生一個安全的 Lambda Expression 供 `InputSession` 呼叫 client()，避免 controller 強持有 client。  
* `callCoreAtLeastOnce(client:)` 在 MainActor 內運行，先於快取中尋找既有的 `InputSession`；如命中便重新綁定控制器，否則建立新的會話。
* **會話建構子直接使用傳入的 `inputClient` 參數。** 在 macOS 10.9 ~ 10.12 上，`super.init(server:delegate:client:)` 返回後 `self.client()` 仍回傳 `nil`—這是 Distributed Object 的特性所致。IMK 使用 NSConnection 跨進程通訊，`client()` 返回的是 Distributed Object 代理（macOS 10.9 上的 `NSDistantObject` Mach port proxy）。代理物件的初期化並非同步完成：IMK 在建構子同步執行期間尚未完成與遠端 client 物件的代理協商和建立。然而，建構子的 `inputClient` 參數本身就是這個 client 物件——可利用 `wrap`/`unwrap` 把它安全地傳入 `mainSync` lambda expression，從而以 `mainSync` 同步完成 `callCoreAtLeastOnce(client:)`，使 `core` 在建構子返回時即保證為非 nil。當快取未命中而需新建 `InputSession` 時，先以傳入的 client 物件建構靜態 closure 作為臨時的 `theClient`，再立即排入 `DispatchQueue.main.async` 脫手操作將其替換為正常的動態 `getClientProvider()`，確保 `NSMapTable` 弱鍵機制不因短暫的強持有而失效。

這僅是一個簡化的樣板，實際專案裡你可以把這些概念封裝成你自己的工廠/管理器。核心觀念是讓 `IMKInputController` 本身保持「乾淨」——沒有長期住著的強參照，所有狀態都擺在可以全局共用、以 client 為鍵的 session 物件裡。

筆者這裡舉個例子：輸入法業務模組是一個純 Swift 的 Class `InputSession` 會話模組。當作 IMKInputController 的 Delegate Class，但 IMKInputController 不持有它。見下文：

```swift
/// nonisolated 是 IMKStateSetting & IMKMouseHandling 協定要求的。
/// 或者說，官方沒要求，但是是 Swift 相容性沒做好導致的現狀。
@objc(MyIMKInputController) // 必須加上 ObjC，因為 IMK 是用 ObjC 寫的。
nonisolated public final class MyIMKInputController: IMKInputController, @unchecked Sendable {
  // MARK: Lifecycle

  /// 對用以設定委任物件的控制器型別進行初期化處理。
  nonisolated override public init() {
    super.init()
  }

  /// 對用以設定委任物件的控制器型別進行初期化處理。
  ///
  /// inputClient 參數是客體應用側存在的用以藉由 IMKServer 伺服器向輸入法傳訊的物件。該物件始終遵守 IMKTextInput 協定。
  /// - Remark: 所有由委任物件實裝的「被協定要求實裝的方法」都會有一個用來接受客體物件的參數。在 IMKInputController 內部的型別不需要接受這個參數，因為已經有「client()」這個參數存在了。
  /// - Parameters:
  ///   - server: IMKServer
  ///   - delegate: 客體物件
  ///   - inputClient: 用以接受輸入的客體應用物件
  nonisolated override public init!(server: IMKServer!, delegate: Any!, client inputClient: Any!) {
    // Note: this constuctor gets called everytime this IME gets switched to.
    // This happens even if the client() is the same IMKTextInput instance.
    super.init(server: server, delegate: delegate, client: inputClient)
    // macOS 10.9 ~ 10.12 的相容性處理：此處得使用傳入的 client 參數，因為 `client()` 沒有就緒、是 nil。
    // 在這些舊版系統上，IMK 尚未在 super.init 返回時就完成 client 物件的綁定，
    // 因此 `client()` 在建構子同步執行期間始終回傳 nil，導致 Session 無法登記至快取。
    // 穩妥的做法是使用當前建構子內傳入的 client 參數，可確保 IMK 已完成 client 綁定。
    let senderRef = wrap(inputClient)
    mainSync {
      // Force initialization.
      self.core = callCoreAtLeastOnce(client: unwrap(senderRef))
    }
  }

  // MARK: Public

  @MainActor
  public var core: InputSession? {
    get {
      if let workingValue = _core { return workingValue }
      let newValue = callCoreAtLeastOnce(client: nil) // <- 使用 `client()`。
      self.core = newValue
      return newValue
    }
    set {
      _core = newValue
    }
  }

  // MARK: Private

  @MainActor
  private weak var _core: InputSession? // <- 必須 `weak`，不然就是「持有」了。

  nonisolated private func getClientProvider() -> (() -> InputSession.ClientObj?) {
    { [weak self] in
      self?.client() as? InputSession.ClientObj
    }
  }

  nonisolated private func callCoreAtLeastOnce(client maybeClient: Any!) -> InputSession {
    let senderRef = wrap(maybeClient)
    return mainSync {
      // 嘗試從快取中複用既有的 InputSession，以緩解 CapsLock 頻繁切換場景下的 ARC 壓力。
      let maybeClientOnMain = unwrap(senderRef) as? NSObject
      let clientObj = maybeClientOnMain ?? (self.client() as? NSObject)
      if let clientObj, let cached = InputSession.cachedSession(for: clientObj) {
        cached.reassign(to: self, clientProvider: getClientProvider())
        vCLog("InputSession reused. ID: \(cached.id.uuidString)")
        return cached
      }
      // 先用傳入的參數完成 InputSession 的初期化，其中包括了對這個 Session 的登記過程。
      let newSession = InputSession(controller: self) {
        clientObj as? InputSession.ClientObj
      }
      // 然後再用脫手操作給這個 Session 重新指派 clientProvider。
      DispatchQueue.main.async { [weak self] in
        guard let this = self else { return }
        newSession.reassign(to: this, clientProvider: this.getClientProvider())
      }
      return newSession
    }
  }
}

@MainActor
public final class InputSession: Sendable {
  // MARK: Lifecycle

  public init(
    controller inputController: MyIMKInputController?,
    client inputClient: @escaping (() -> ClientObj?)
  ) {
    self.theClient = inputClient
    self.inputControllerAssigned = inputController
    construct(client: theClient()) // <- 這是單獨的專項建構子。
    registerInCache()
    print("InputSession constructed. ID: \(id.uuidString)")
  }

  nonisolated deinit {
    print("InputSession deconstructing. ID: \(id.uuidString)")
  }

  // MARK: Public

  public typealias ClientObj = IMKTextInput & NSObject

  public var theClient: () -> ClientObj?

  /// IMKInputController 副本。
  public weak var inputControllerAssigned: MyIMKInputController?

  // MARK: Internal

  /// 從快取中查詢既有的 InputSession（以 client NSObject 的記憶體位址為鍵）。
  static func cachedSession(for clientObj: NSObject) -> InputSession? {
    sessionsByClient.object(forKey: clientObj)
  }

  /// 將自身登記至快取。首次建構 InputSession 時呼叫。
  func registerInCache() {
    guard let clientObj = theClient() else { return }
    Self.sessionsByClient.setObject(self, forKey: clientObj)
  }

  /// 重新綁定至新的 MyIMKInputController（快取命中時使用）。
  /// 僅更新控制器參照與 clientProvider ，不重新建構打字模組。
  func reassign(to controller: MyIMKInputController, clientProvider: @escaping () -> ClientObj?) {
    inputControllerAssigned = controller
    theClient = clientProvider
  }

  // MARK: Private

  private static var _current: InputSession?

  // MARK: - Session 快取 (緩解 CapsLock 高頻切換場景下的 ARC 壓力)

  /// 弱鍵快取：將 client NSObject（弱引用）映射至 InputSession（強引用）。
  /// 當 client 被 ARC 回收後，對應條目會在下次存取時自動清除。
  private static var sessionsByClient = NSMapTable<NSObject, InputSession>.weakToStrongObjects()
}
```

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
