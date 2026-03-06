# macOS Input Method Development Guidelines for 2026

The Input Method Kit (IMK) framework emerged during the macOS 10.5 Leopard era—predating Objective-C ARC, XPC communication, and the Sandbox (all introduced in macOS 10.7). Naturally, it also came before Swift 5 and SwiftUI gained popularity. In other words, InputMethodKit is a legacy framework that has spanned two generations of technology paradigm shifts. Apple's original reference manual for macOS 10.5 Leopard, the *[Input Method Kit Framework Reference](https://leopard-adc.pepas.com/documentation/Cocoa/Reference/InputMethodKitFrameworkRef/InputMethodKitFrameworkRef.pdf)* (henceforth "IMKFR"), no longer meets the requirements imposed by these shifts—particularly those of Swift 6 Concurrency. Based on my experience developing the [vChewing input method (targeting macOS 10.09 Mavericks to the latest macOS 26)](https://vchewing.github.io/), I've compiled these guidelines for other engineers seeking to develop input methods for macOS.

> I also developed the [IMKSwift](https://github.com/vChewing/IMKSwift) package to make it easier for Swift developers to write IMK input methods. IMKSwift provides the IMKInputSessionController base type, which replaces the ObjC header definitions of IMKInputController with expressions more friendly to Modern Swift Concurrency. Using this package benefits the devs that some of the verbose details in the following sections may not need to be strictly adhered to.

## 1. NSConnection Naming

The IMKFR doesn't cover this, but there's only one correct answer: the `InputMethodConnectionName` key in the input method's `Info.plist` can only be set to `$(PRODUCT_BUNDLE_IDENTIFIER)_Connection`.

> ⚠️ **This is the NSConnection naming convention that macOS 10.7 Lion introduced.**
>
> If you don't follow this convention, your input method may fail to load properly after enabling the Sandbox—when users try to switch to your input method, they'll encounter NSConnection-related failure which will have messages in `Console.app`.

Apple's own "NumberInput" reference project provided a *[poor example](https://github.com/pkamb/NumberInput_IMKit_Sample/blob/6c37ea05d85d0b7b5af9378a0ce88e191ca07241/NumberInput/main.m#L53-L55)*, misleading macOS input method developers worldwide. When the authorities err, the misdirection is most damaging.

![image](./macOS_Input_Method_Development_Guidelines_2026-illust1.png)

Apple even granted input methods that hadn't enabled Sandbox special treatment, allowing them to continue functioning normally despite using non-standard NSConnection names. Some input method developers unfortunately misinterpreted this as "the Sandbox actually causes more problems," which is fundamentally incorrect reasoning.

## 2. Sandbox Entitlements

Always enable the Sandbox. The instant an input method enables Sandbox on macOS, it becomes *theoretically* impossible to gain system-wide keyboard access. Because your input method is forced by the system framework to rely on NSConnection —- a fragile mechanism—failing to enable the Sandbox is turning the input method into the town bicycle; everyone’s had a ride.

Sandbox support represents an input method's best security credential to users.

> Consequently, most remaining input methods lack Sandbox support: either they face technical challenges or they offer vague excuses.

Define the Sandbox entitlements file as follows:

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

Notice that this whitelist includes the input method's own `UserDefaults`. This is essential because once an input method enables Sandbox, macOS deprives it of direct `UserDefaults` access.

## 3. MainActor Constraints and Swift 6.2+

All IMKInputController API interactions run on the MainActor. However, InputMethodKit's exposed headers are incompatible with Swift Concurrency, making it impossible to properly confine IMKInputController to the MainActor.

The best approach to ensure InputMethodKit compatibility with Swift 6 Concurrency is to set the entire target's default isolation to `MainActor`. While this occasionally requires awkward hacks for IMKInputController API calls, it represents the minimal-effort solution.

First, introduce these two extension APIs:

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

Then use this `mainSync` API (designed to prevent Russian doll deadlocks):

```swift
@discardableResult
public func mainSync<T>(execute work: @MainActor () throws -> T) rethrows -> T {
  if Thread.isMainThread {
    return try work()
  }
  return try DispatchQueue.main.sync(execute: work)
}
```

Here's a template demonstrating how to translate API parameters to the MainActor:

```swift
/// nonisolated is required by the IMKStateSetting & IMKMouseHandling protocols.
/// Technically, Apple doesn't require it, but the header's Swift compatibility was not well-designed.
@objc(MyIMKInputController) // @objc is mandatory because IMK is written in Objective-C.
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
      // Business logic resides here.
    }
  }
}
```

Note that I've declared `MyIMKInputController` as `Sendable`. Otherwise, `mainSync` becomes ineffective.

## 4. IMKInputController Must Async-Run Tasks Appropriately

Some input methods inevitably interact with `client()` during the `activateServer` phase, but this overhead may be unavoidable—you might need to apply a specific Ukelele layout using `client.overrideKeyboard()`. Since the IMKTextInput Client lacks true Async APIs, input method developers must assume all such Client operations are MainActor-blocking operations and accept the limitation.

Therefore, all Client methods *except* `client()?.setMarkedText` & `client()?.insertText` can be async-run on the MainActor. By strictly confining all IMKInputController API interactions to the MainActor (as described above), you needn't worry about ordering issues from async-running.

> Note: `client()?` is a MainActor-confined object. You can async-run, but the lambda expression calling Client methods must run on the MainActor.

## 5. IMKInputController Must Not Hold Anything

This is critically important. Here's a scenario (I've mentioned this previously):

> macOS 10.12's CapsLock toggle isn't fundamentally a Chinese/English input mode switch—it's an input method switch. On macOS, even English input is handled by a dedicated input method. On most English keyboards, this is called Apple ABC (corresponding to the US layout). Each time an input method is switched to, it triggers the creation of a new IMKInputController instance for that method and its `activateServer` operation (plus potentially other follow-up operations). Only afterward does the previous input method's IMKInputController instance undergo deactivation.
>
> Users frequently mixing Chinese and English often switch back-and-forth between ABC and a Chinese input method. Because both serve the same IMKTextInput Client, MainActor congestion occurs. Moreover, high-frequency switching stresses the ARC reference counting used by IMKInputController. Both ARC cleanup and object interaction occur on the MainActor, inevitably causing contention.
>
> The process of "switching input methods on the same client" involves both IMKInputController instances operating on the same `client()`. The optimal pattern is to have `deactivateServer` async-run on the MainActor and avoid text writing/display interactions with `client()` during deactivation, since the system handles cleanup automatically. However, this system-managed cleanup also occurs on the MainActor, causing task sequence conflicts. InputMethodKit likely has internal mechanisms to handle this, but the cost is blocking overhead.

This results in frustrated users who frequently CapsLock-switch between input methods experiencing lag. But they don't realize the problem lies at the system level—they blame the input method. Some complain that macOS's built-in Zhuyin is broken; others fault the third-party input method they're using.

Though the current workaround is "disable macOS's built-in CapsLock input method switching" and "implement native CapsLock English mode in your input method," Apple's market strategy seems to discourage this. The Globe key on the first-generation Apple Silicon MacBook keyboards being repurposed as an input method rotation key reflects Apple's ongoing push toward its idealized vision.

This leaves developers with two tasks: one we covered earlier ("async-run appropriate tasks"), and another: **IMKInputController must not hold any objects**.

Why is the client identical across different input method switches? Because this client is an NSConnection Distributed Object uniformly dispatched by IMK, possessing memory address consistency. This is the crucial detail that makes the weak-key solution viable.

The solution is simple: use an `NSMapTable` (or any equivalent weak‑key hash) where the *key* is a weak reference to the client object. When the client deallocates, the table automatically ejects the entry on next access, so you never end up holding stale strong pointers.

The practical upshot is: **IMKInputController holds no objects itself**. Instead you maintain a separate business‑logic class and access it via weak references or provider lambda expressions. In the sample below `InputSession` represents such a session object. The controller merely looks up (or creates) the session in a global cache keyed by the distributed client, then forwards events. This keeps the controller “clean”: it never retains the session, nor the client, avoiding ARC churn when input methods are switched rapidly.

Key points of the example:

- `MyIMKInputController.core` is a `weak` property; the session can outlive the controller and the reference will nil‑out automatically.
- `getClientProvider()` returns a lambda expression that safely invokes `client()` without trapping a strong reference to the controller.
- `callCoreAtLeastOnce(client:)` is executed on the MainActor; it first tries to reuse an existing `InputSession` from the cache (mitigating ARC pressure during CapsLock spam), reassigning it to the new controller if found. Otherwise it constructs a fresh session.
- **The constructor can use the provided `inputClient` parameter directly.** On macOS 10.9 ~ 10.12, `client()` still returns `nil` even after `super.init(server:delegate:client:)` returns—this is due to the nature of Distributed Objects. IMK uses NSConnection for cross-process communication, and `client()` returns a Distributed Object proxy (the `NSDistantObject` Mach port proxy on macOS 10.9). Proxy object initialization is not synchronous: IMK has not yet completed the proxy negotiation and establishment with the remote client object during synchronous constructor execution. However, the constructor’s `inputClient` parameter *is* the client object—using `wrap`/`unwrap` to safely pass it into the `mainSync` lambda expression, `callCoreAtLeastOnce(client:)` can run synchronously, ensuring `core` is guaranteed non-nil when the constructor returns. When a cache miss requires a new `InputSession`, the session is first created with a static lambda expression capturing the passed-in client, then a `DispatchQueue.main.async` block immediately replaces it with the proper dynamic `getClientProvider()`, so the `NSMapTable` weak-key mechanism is not undermined by the brief strong capture.

This is only a minimal template—you can wrap the caching logic inside your own factory or manager in a real project. The essential concept is that all state lives in sharable session objects keyed by client; the IMKInputController itself is just a thin, non‑holding facade.

Here's a practical example: **IMKInputController doesn't hold any objects** but can establish weak-reference relationships with business module objects. For instance, an input method's business logic is a pure Swift `InputSession` object. Make it the delegate, but don't let IMKInputController hold it:

```swift
/// nonisolated is required by the IMKStateSetting & IMKMouseHandling protocols.
/// Technically, Apple doesn't require it, but the header's Swift compatibility was not well-designed.
@objc(MyIMKInputController) // @objc is mandatory because IMK is written in Objective-C.
nonisolated public final class MyIMKInputController: IMKInputController, @unchecked Sendable {
  // MARK: Lifecycle

  /// Initialize the controller for setting the delegate object.
  nonisolated override public init() {
    super.init()
  }

  /// Initialize the controller for setting the delegate object.
  ///
  /// The inputClient parameter is the object on the client application side that communicates with the input method via IMKServer. This object always conforms to the IMKTextInput protocol.
  /// - Remark: All methods required by the delegate protocol implementations will have a parameter accepting the client object. Within IMKInputController, you don't need this parameter because the `client()` accessor already exists.
  /// - Parameters:
  ///   - server: IMKServer
  ///   - delegate: The client object
  ///   - inputClient: The client application object receiving input
  nonisolated override public init!(server: IMKServer!, delegate: Any!, client inputClient: Any!) {
    // Note: this constructor gets called every time this IME gets switched to.
    // This happens even if the client() is the same IMKTextInput instance.
    super.init(server: server, delegate: delegate, client: inputClient)
    // Compatibility with macOS 10.9 ~ 10.12: use the provided client parameter because
    // `client()` is not ready yet—it is nil.
    // On these older systems, IMK has not finished binding the client object by the time
    // super.init returns, so `client()` always returns nil during synchronous constructor
    // execution. This means that the session cannot be registered in the cache.
    // The reliable approach is to use the constructor’s inputClient parameter directly,
    // ensuring the client object is available regardless of IMK’s binding timing.
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
      let newValue = callCoreAtLeastOnce(client: nil) // <- Use `client()`.
      self.core = newValue
      return newValue
    }
    set {
      _core = newValue
    }
  }

  // MARK: Private

  @MainActor
  private weak var _core: InputSession? // <- Use `weak`, don't HOLD.

  nonisolated private func getClientProvider() -> (() -> InputSession.ClientObj?) {
    { [weak self] in
      self?.client() as? InputSession.ClientObj
    }
  }

  nonisolated private func callCoreAtLeastOnce(client maybeClient: Any!) -> InputSession {
    let senderRef = wrap(maybeClient)
    return mainSync {
      // Attempt to reuse an existing InputSession from cache to mitigate ARC
      // pressure during high-frequency CapsLock switching scenarios.
      let maybeClientOnMain = unwrap(senderRef) as? NSObject
      let clientObj = maybeClientOnMain ?? (self.client() as? NSObject)
      if let clientObj, let cached = InputSession.cachedSession(for: clientObj) {
        cached.reassign(to: self, clientProvider: getClientProvider())
        vCLog("InputSession reused. ID: \(cached.id.uuidString)")
        return cached
      }
      // First create the InputSession with the passed-in client parameter,
      // which also triggers cache registration.
      let newSession = InputSession(controller: self) {
        clientObj as? InputSession.ClientObj
      }
      // Then async-reassign with the proper dynamic clientProvider.
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
    construct(client: theClient()) // <- This is a separate specialized constructor.
    registerInCache()
    print("InputSession constructed. ID: \(id.uuidString)")
  }

  nonisolated deinit {
    print("InputSession deconstructing. ID: \(id.uuidString)")
  }

  // MARK: Public

  public typealias ClientObj = IMKTextInput & NSObject

  public var theClient: () -> ClientObj?

  /// The assigned IMKInputController instance.
  public weak var inputControllerAssigned: MyIMKInputController?

  // MARK: Internal

  /// Retrieve an existing InputSession from cache (keyed by client NSObject's memory address).
  static func cachedSession(for clientObj: NSObject) -> InputSession? {
    sessionsByClient.object(forKey: clientObj)
  }

  /// Register this instance in the cache. Call when first constructing InputSession.
  func registerInCache() {
    guard let clientObj = theClient() else { return }
    Self.sessionsByClient.setObject(self, forKey: clientObj)
  }

  /// Reassign to a new MyIMKInputController (used on cache hits).
  /// Only updates controller and client references; doesn't reconstruct the typing module.
  func reassign(to controller: MyIMKInputController, clientProvider: @escaping () -> ClientObj?) {
    inputControllerAssigned = controller
    theClient = clientProvider
  }

  // MARK: Private

  private static var _current: InputSession?

  // MARK: - Session Cache (Mitigates ARC pressure during high-frequency CapsLock switching)

  /// Weak-key cache: maps client NSObject (weak reference) to InputSession (strong reference).
  /// When the client is deallocated by ARC, the corresponding entry is automatically
  /// removed on the next cache access.
  private static var sessionsByClient = NSMapTable<NSObject, InputSession>.weakToStrongObjects()
}
```

## 6. Write All Input Method Code as Swift Package Libraries

macOS input methods cannot be debugged using breakpoints—they will indefinitely freeze any client application that has touched your input method, and consequently freeze your entire desktop. You're left with no choice but to SSH into your machine externally and force-kill the input method process. The only viable approach is to write your own unit tests with mocked clients. Structuring all business logic as libraries greatly facilitates this debugging approach and allows developers to specify dedicated `UserDefaults` containers for heumetic tests. Better still, you can write a standard AppKit app simulating this unit test typing process and use Instruments to detect memory leaks. This is far more flexible than maintaining only an input method Xcode target.

## 7. Self-Monitor Memory Usage; Force Termination When Necessary

User computer memory is precious. While macOS 26's AppKit NSWindow rendering inefficiency means an input method might consume 80–200 MB of RAM, consider this design pattern: have your input method check its own memory footprint each time `activateServer` switches to a new typing session. If it exceeds 1024 MB, emit an NSNotification to inform the user, then terminate gracefully. This notification warns users and prevents them from assuming the input method has crashed.

Of course, this is a fallback safeguard preventing catastrophic outcomes like memory exhaustion on user systems. Developers remain obligated to actively check their code for memory leaks.

> ⚠️ If your input method uses SQLite, you need to pay extra attention to a little-known fact: after running each query with SQLite, you must use `sqlite3_finalize(StatementPointer)` to release the memory, otherwise a memory leak will occur that **cannot even be detected by Xcode Instruments**.

## 8. Minimize NSWindow Count in Input Methods

This guideline responds to macOS 26's dire reality: beginning with macOS 26, memory consumed by NSWindow instances is never reclaimed by the system. The baseline overhead per NSWindow is substantial, and that overhead is amplified by the LiquidGlass rendering framework. Even if you don't explicitly enable LiquidGlass effects, it still applies. Setting `UIDesignRequiresCompatibility` in `Info.plist` can reduce memory footprint to macOS 15 levels, but this is a temporary measure—Apple can revoke `UIDesignRequiresCompatibility` support at any moment.

> I speculate that macOS 26's massive disk footprint includes a macOS 15 AppKit environment specifically providing backward compatibility for this `Info.plist` property.

With Swift UI now mature, consider consolidating "tooltip panels" and "custom candidate windows" into a single NSPanel, reducing NSWindow baseline overhead. The input method's "About" window can be folded into the Settings UI of the input method.

> NSPanel is a variant of NSWindow.

## 9. Avoid Using IMKCandidates

Even the NumberInput sample didn't dare use IMKCandidates—because IMKCandidates is ancient rubbish, still stinking to this day. Look at macOS 26's built-in Japanese input method: it's a victim of IMKCandidates, with text barely legible:

![image](./macOS_Input_Method_Development_Guidelines_2026-illust2.png)

The glass background is completely transparent, exposing white behind it. The candidate window text is also white. This is clearly a result of inadequate unit testing—misuse of the LiquidGlass API.

With modern AI capabilities, it's straightforward to ask an AI to write a custom candidate panel resembling IMKCandidates's layout. Alternatively, you can force-expose IMKCandidates's internal APIs (some have been stable since macOS 10.14 Mojave), though future availability is uncertain.

> In my vChewing Input Method I use my self-crafted Tadokoro Candidate Window to provide an experience hakushin-enough to IMKCandidates (dynamic grid layout).

![image](./macOS_Input_Method_Development_Guidelines_2026-illust3.png)

## Conclusion

InputMethodKit is a historical artifact, yet it remains the only official entry point for macOS input methods. Developers must therefore accept this framework's baggage and establish engineering discipline atop its structural deficiencies.

The guidelines presented here aren't mere "tricks"—they constitute a risk-containment model: transforming IMKInputController into a pure pass-through layer, modularizing all business logic completely, treating MainActor as an immutable fact, regarding memory pressure as a design constraint, and viewing the Sandbox as a non-negotiable ethical baseline.

Should Apple someday completely rewrite InputMethodKit, these guidelines may become obsolete. But until then, macOS input methods that wish to maintain engineering quality and security trustworthiness in 2026 must encode "self-discipline" into their architecture, not just into their README files.

$ EOF.
