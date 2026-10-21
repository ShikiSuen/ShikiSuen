# macOS Input Method Development Guidelines for 2026

The Input Method Kit (IMK) framework emerged during the macOS 10.5 Leopard era—predating Objective-C ARC, XPC communication, and the Sandbox (all introduced in macOS 10.7). Naturally, it also came before Swift 5 and SwiftUI gained popularity. In other words, InputMethodKit is a legacy framework that has spanned two generations of technology paradigm shifts. Apple's original reference manual for macOS 10.5 Leopard, the *[Input Method Kit Framework Reference](https://leopard-adc.pepas.com/documentation/Cocoa/Reference/InputMethodKitFrameworkRef/InputMethodKitFrameworkRef.pdf)* (henceforth "IMKFR"), no longer meets the requirements imposed by these shifts—particularly those of Swift 6 Concurrency. Based on my experience developing the [vChewing input method (targeting macOS 10.09 Mavericks to the latest macOS 27)](https://vchewing.github.io/), I've compiled these guidelines for other engineers seeking to develop input methods for macOS.

> I also developed the [IMKSwift](https://github.com/vChewing/IMKSwift) package to make it easier for Swift developers to write IMK input methods. IMKSwift provides the IMKInputSessionController base type, which replaces the ObjC header definitions of IMKInputController with expressions more friendly to Modern Swift Concurrency. The package also ships the MRC private-surface layer described in §5, so that section can be read as a reference implementation rather than a to-do list.

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

## 5. IMKInputController Must Be MRC-Managed, and the Client Binding Must Be Re-Established per Event

### 5.1 The IMKServer Controller-Leak Flaw (Reverse‑Engineered)

Through extensive reverse engineering of InputMethodKit across macOS 10.9 Mavericks, 10.15 Catalina, and macOS 27 GoldenGate (arm64e), the vChewing project uncovered a fundamental defect in IMKServer's internal controller lifecycle management:

1. IMKServer internally maintains `_private._controllers` — an `NSMutableDictionary` keyed by `[NSNumber numberWithUnsignedLong:clientProxyAddr]` — to track all `IMKInputController` instances. The key is the memory address of the client proxy object passed to each controller at `init`.

2. **Every CapsLock toggle creates a brand-new DO/XPC proxy object** with a different memory address. The old controller's dictionary key can never be hit by a lookup again, because IMKServer's built-in controller reuse logic is keyed on the proxy address that has now changed. The old controller becomes permanently orphaned.

3. IMKServer provides `sessionFinished:` to clean up accumulated controllers for a given app, but `sessionFinished:` is dispatched per-client-proxy. After a CapsLock toggle, the old proxy has been replaced; IMKServer no longer sends any messages to it, so `sessionFinished:` is never triggered for the orphaned controller. **Each CapsLock toggle produces a permanently leaked controller.**

4. The orphaned controller holds a client wrapper (`IPMDServerClientWrapper` on macOS ≤ 10.15; split into `_IPMDServerClientWrapperModern` and `_IPMDServerClientWrapperLegacy` on macOS 15 Sequoia and later) which in turn holds `_xpcConnection` (NSXPCConnection) or `_clientDOProxy` — these connection resources are also permanently leaked. IMK manages these wrappers via a global cache, and they can only be removed by calling private class methods (`+terminateForClientXPCConn:`, `+terminateForClientDOProxy:`).

5. From macOS 15 Sequoia onward, `+terminateForClient:` (the unsuffixed version) has been removed entirely; only the XPC and DO variants remain.

6. Under ARC, the retain/release churn from frequent CapsLock switching can cause perceptible stutter. Disabling ARC (`-fno-objc-arc`) eliminates the stutter completely, but then you must handle orphaned controller cleanup and residual XPC connection teardown manually.

7. **The orphan is not merely leaked — it keeps the client binding it had when it was deactivated.** Nothing in the framework will ever update it again, because nothing in the framework still believes the controller is in service. Any keystroke the system routes to such a controller is then answered with a retired client's identity.

### 5.2 The Fix: MRC + KVC Prune + Address-Keyed Sessions + Per-Event Adoption

**a) Full MRC for the IMK interaction layer.** The `IMKInputSessionController` target is compiled with `-fno-objc-arc`. All controller alloc/dealloc/retain/release happens in the ObjC MRC layer. The Swift side interacts with controllers only through raw memory addresses (`uintptr_t`), never performing Swift ARC operations on controller instances.

**b) KVC-based orphaned controller pruning.** A monotonically increasing generation number is assigned to each controller at init. On every controller creation — after the new controller has registered itself — the oldest inactive controller (lowest generation, excluding the current controller and the new one) is removed via KVC: `[server valueForKeyPath:@"_private._controllers"] removeObjectForKey:oldKey]`. This prunes orphaned controllers in MRC so they are properly deallocated.

**c) A small session pool keyed by controller address.** vChewing first routed sessions by generation parity (even → session A, odd → session B — the "double buffer" described in earlier revisions of this document). Generations are monotonic, so parity looks stable, and that was the trap: two controllers whose generations share a parity share one session. When the first retires, the session's client binding is cleared; the second keeps dispatching through a session that no longer knows its client, and every key is dropped before it reaches the IME. The current design keys sessions by the controller's own address in a pool of at most four, reaps a slot only when the lifetime tracker says the controller is gone, and never evicts by recency.

**d) Delayed dealloc with XPC cleanup.** After `deactivateServer:`, the controller is not immediately deallocated. Instead, a 3-second timer is started. When it fires, the controller: releases all blocks, invokes the private `+terminateForClient*` class methods on the appropriate client wrapper class (falling back through `_IPMDServerClientWrapperModern` → `_IPMDServerClientWrapperLegacy` → `IPMDServerClientWrapper` for forward compatibility with macOS 27), and clears the controller↔session mapping. If the controller is reactivated within 3 seconds (`activateServer:`), the timer is cancelled and normal service resumes. This prevents a "dead controller, not-yet-ready new controller" gap during rapid CapsLock toggling.

> Delayed dealloc buys the reuse window; it does **not** mean the controller has died. The shell may remain in `_controllers` until the next prune, so never use "the timer fired" as a liveness test.

**e) Per-event client adoption.** On every delivered event, before dispatch, the session rebinds itself to the sending controller's address. The write is skipped when the value is unchanged, so it costs one comparison per keystroke. This single line is what makes the design self-healing: whichever copy the system routes to, the session's client is whoever is sending *now*. `IMKControllerLifetimeTracker` still guards every `takeUnretainedValue()` on a raw address.

**f) IME menu injection via ObjC runtime.** The IME menu is dynamically built by `IMEMenuSputnik` which uses `class_addMethod` + `imp_implementationWithBlock` to register Swift closures directly as method IMPs on `IMKInputSessionController`. No `@objc` selector exposure is needed. The menu injects two live metrics: (1) the count of alive controller instances tracked by `IMKControllerLifetimeTracker`, and (2) anonymous private memory usage obtained from `task_vm_info.internal`. Before each menu open, `purgeMallocZones()` is called to flush allocator caches so the memory reading reflects actual usage.

### 5.3 Architectural Summary

The net effect is that ARC is fully removed from the IMK dispatch path, eliminating CapsLock stutter. A small address-keyed session pool replaces the parity pair and eliminates the aliasing hazard. The KVC prune path compensates for IMKServer's inability to clean up orphaned controllers on its own. Per-event adoption makes a stale binding impossible. The delayed dealloc ensures XPC connections are properly terminated rather than leaked.

Below is a minimal sketch of the controller-side architecture:

```swift
// IMKInputSessionController (MRC-compiled .m file):
// - Class-level static blocks dispatch to Swift via raw addresses.
// - +IMKSwift_configureWithActivatingServer: etc. set once at startup.

// Swift side — address-keyed session pool:
@MainActor
final class SessionPool {
  static let shared = SessionPool(capacity: 4)
  private var slots: [UInt: InputSession] = [:]

  func session(for ctlAddr: UInt) -> InputSession { /* create or reuse this address's slot */ }
  func reap(using tracker: IMKControllerLifetimeTracker) { /* drop slots whose controller is gone */ }
}

// SessionControllerSputnik event block:
// 1. Resolve (or create) the session for the sending controller's address.
// 2. adoptClientAddr(ctlAddr) — the sender is the authority.
// 3. Dispatch into the session's core.
```

> **Note:** the pool is keyed by *controller* address, not client address. The controller is the object whose lifetime the framework actually manages; the client is a proxy that can be replaced beneath it at any time. Four slots suffice because the system keeps at most one active controller plus a short tail of retiring ones.

### 5.4 Observability and Kill Switches

You cannot breakpoint any of this. What you can do is instrument it, and make every intervention individually switchable.

**Log vocabulary.** The ObjC layer emits a fixed set of tokens: `ControllerInit:`, `Prune:` (`controllers=`, `current=`, `self=`, `evicted … remaining=`), `ActivateServer:` (with `clientTerminatedBefore=`), `DeactivateServer:`, `DelayedDealloc:` (scheduled / skipped / fired), `EventEntry:` (`addr`, `alive`, `gen`, `session=`, plus the cached client name), and `guard fail:` (`hasHandler`, `hasClient`, `proxyNil`). On macOS 26 and later they go through `Logger(subsystem: "vChewing", category: "Log")`; earlier systems fall back to `NSLog`. A debug-mode flag gates them.

**Four diagnostics-only switches**, exposed in the DevZone pane and as raw `UserDefaults` keys:

| Switch | Disables |
|---|---|
| `configureServerControllerPruningDisabled(_:)` | the KVC prune |
| `configureEvictedClientInvalidationDisabled(_:)` | invalidating the evicted controller's client entry (only the aggressive half of the eviction) |
| `configureClientWrapperTerminationDisabled(_:)` | `+terminateForClient*` on the wrapper, keeping the dealloc bookkeeping |
| `configureControllerDelayedDeallocationDisabled(_:)` | the whole deferred teardown |

Each block is evaluated at the decision point and never cached; vChewing feeds them from four `_IMKSwift_disable…` preferences, so the switch states of a running process can be changed without a rebuild.

Sixteen raw combinations, nine distinguishable behaviours: the delayed-dealloc switch already makes the wrapper-termination and eviction switches unreachable in the same sequence, and the prune switch makes the eviction switch moot. They exist so that a report from the field can be turned into a bisection instead of a guess.

**Three failure signatures** the tokens let you tell apart:

1. no `EventEntry:` for the dead window ⇒ the event was never delivered;
2. `EventEntry:` with `alive=false`, or `UnresolvableController:` ⇒ the address was lost;
3. `EventEntry:` with `alive=true` but `session=` naming another client, or `assigned=nil` ⇒ the event went to the wrong client.

**Installers.** Capture the running PIDs *after* the payload swap, escalate `SIGTERM` → `SIGKILL`, poll for vacancy, and verify. Never `killall` and hope; never kill before the swap (IMK relaunches the IME from the registered bundle, so a pre-swap kill only restarts the old code under a new PID). When running as root, scope the kill to the console user via `/usr/bin/stat -f%Su /dev/console`.

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
