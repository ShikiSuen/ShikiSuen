# 写在2026年的macOS输入法开发规范

InputMethodKit 自 macOS 10.5 Leopard 时代问世，早于 Objective-C ARC 技术、XPC 通讯技术、Sandbox 技术问世（均为 macOS 10.7）之前。自然，这也是早于 Swift 5 与 SwiftUI 流行之前。也就是说，InputMethodKit 是横跨了两代技术大变革的祖产级 OS Framework。当年 Apple 写给 macOS 10.5 Leopard 的 IMK 参考手册《[Input Method Kit Framework Reference](https://leopard-adc.pepas.com/documentation/Cocoa/Reference/InputMethodKitFrameworkRef/InputMethodKitFrameworkRef.pdf)》（下文简称《IMKFR》）早已不符合这些变革所带来的新要求（特别是 Swift 6 Concurrency）。笔者根据自己开发[《唯音输入法》(for macOS 10.09 Mavericks ~ macOS 26)](https://vchewing.github.io/)的经验，将一些注意事项整理在此，留给其他想给 macOS 开发输入法的工程师们参考。

> 笔者另外制作了 [IMKSwift](https://github.com/vChewing/IMKSwift) 套件，允许 Swift 工程师们在写 IMK 输入法时更顺利：IMKSwift 提供了 IMKInputSessionController 基础型别、是在 IMKInputController 的基础上整体换用了对 Modern Swift Concurrency 更友好的 ObjC Header 表达。使用这个套件的话，下文某些繁文缛节或可不必严苛遵守。

## 1. NSConnection 名称

《IMKFR》没提及，但正确答案只有一个：输入法的 `Info.plist` 的 `InputMethodConnectionName` 栏位只能填写 `$(PRODUCT_BUNDLE_IDENTIFIER)_Connection`。

> ⚠️ **这是 macOS 10.7 Lion 开始对 NSConnection 的命名规范**。
>
> 不按照这个规范命名的话，你的输入法在开启 Sandbox 之后，可能就会在用户尝试切换到该输入法的时候无法正常载入。此时可以在 `Console.app` 内观测到与 NSConnection 有关的失败讯息。

当年由 Apple 同步提供的「NumberInput」这个范例专案就给了[错误示范](https://github.com/pkamb/NumberInput_IMKit_Sample/blob/6c37ea05d85d0b7b5af9378a0ce88e191ca07241/NumberInput/main.m#L53-L55)，误导了全球的 macOS 输入法开发者们。官方误导，最为致命。

![image](./macOS_Input_Method_Development_Guidelines_2026-illust1.png)

Apple 甚至都不得不给那些没开 Sandbox 的输入法们开小灶、允许它们在使用非正规命名的 NSConnection 名称的前提下继续正常工作。但这被某些输入法开发者们错误地视为「Sandbox 开了反而会坏事」。

## 2. Sandbox Entitlements

一定要开 Sandbox。macOS 输入法只要开了 Sandbox，就在**原理上**绝对无法拿到系统全局键盘权限了。**你的输入法因为系统框架限制的原因，不得不用 NSConnection 这么脆弱的东西，再不开 Sandbox 的话，就等于北港香炉人人插**。

「Sandbox 支援」对一款 macOS 输入法而言，堪称对用户的最佳的资安投名状。

> 于是剩下的几乎都是不敢开 Sandbox 的输入法了：或有技术难题，或支支吾吾。

Sandbox 权能档案的定义如下：

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

可以看到这里将输入法自身的 UserDefaults 拉入白名单了。这是必需的，因为 macOS 的输入法做了 Sandbox 处理之后确实会丧失对自身 UserDefaults 的存取能力。

## 3. MainActor 约束与 Swift 6.2+

整个 IMKInputController 所有 API 交互都是走 MainActor 的。但是，InputMethodKit 曝露出来的 Header 与 Swift Concurrency 不相容，导致你在使用时反而无法将 IMKInputController 钉死在 MainActor 上。

让 InputMethodKit 与 Swift 6 Concurrency 相容性最佳的处理方法就是将整个 target 的 default isolation 设为 MainActor。这样虽然也难免需要对 IMKInputController 的 API 呼叫处理过程实施一些硬 Hack，但这算是相对而言工作量最小的。

你先引入这两个 extension API：

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

再使用这个 MainSync API（有经过处理，防止俄罗斯套娃 DeadLock）：
```swift
@discardableResult
public func mainSync<T>(execute work: @MainActor () throws -> T) rethrows -> T {
  if Thread.isMainThread {
    return try work()
  }
  return try DispatchQueue.main.sync(execute: work)
}
```

然后，这是范本，专门示范怎样将 API 的参数翻译到 MainActor 上：
```swift
/// nonisolated 是 IMKStateSetting & IMKMouseHandling 协定要求的。
/// 或者说，官方没要求，但是是 Swift 相容性没做好导致的现状。
@objc(MyIMKInputController) // 必须加上 ObjC，因为 IMK 是用 ObjC 写的。
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
      // 此处存放业务逻辑。
    }
  }
}
```

可能有人注意到笔者将 `MyIMKInputController` 定义为 Sendable 了。不然 mainSync 无效。

## 4. IMKInputController 该脱手的任务一定要脱手

有些输入法难免会在 activateServer 阶段引入与 client() 有关的交互，但这个开销可能在所难免，因为你可能必须得对 client 使用 `client.overrideKeyboard()` 套用指定的 Ukelele 布局。再加上 client() 身为 IMKTextInput Client 没有真正意义上的 Async API，输入法开发者只能假设所有这类 Client 的这些操作都是 MainActor 阻塞操作，然后干瞪眼。

于是乎，除了 `client()?.setMarkedText` 与 `client()?.insertText` 以外，其余的 client methods 应该是都可以在 MainActor 上面 Async 脱手操作的。只要你严格按照前文所述将 IMKInputController 所有 API 交互都钉死在 MainActor 上，你就不用担心脱手操作所带来的乱序的问题。

> 注意：`client()?` 是 MainActor 限定物件。你脱手可以，但脱手操作的 Lambda Expression 在呼叫 client() 方法时必须在 MainActor 上。

## 5. IMKInputController 必须是 MRC 管理且以极性路由

### 5.1 IMKServer 的 Controller 泄漏缺陷（逆向工程发现）

藉由对 InputMethodKit 在 macOS 10.9 Mavericks、10.15 Catalina 及 macOS 27 GoldenGate (arm64e) 的逆向工程调查，唯音输入法项目发现了 IMKServer 内部 controller 生命周期管理的根本缺陷：

1. IMKServer 内部以 `_private._controllers`（`NSMutableDictionary`）管理所有 `IMKInputController` 实例，key 为 `[NSNumber numberWithUnsignedLong:clientProxyAddr]`——即每次 init 时传入的 client proxy 对象的内存位址。

2. **每次 CapsLock 切换都会建立全新的 DO/XPC proxy 对象**，其内存位址与旧 proxy 不同。旧 controller 的 dictionary key 永远无法再被 lookup 命中——IMKServer 原厂的 controller 复用逻辑以 proxy 位址为键，CpLk 切换后该键已失效。旧 controller 成为**永不被清理的孤弃 controller**。

3. IMKServer 虽提供 `sessionFinished:` 清理机制，但该方法是 per-client-proxy 的通知：CpLk 切换后旧 proxy 已被取代，IMKServer 不再为旧 proxy 派送任何讯息，`sessionFinished:` 永远不会对孤弃 controller 触发。**每次 CpLk 切换都产生一个永久泄漏的 controller。**

4. 孤弃 controller 持有的 client wrapper（macOS ≤10.15 为 `IPMDServerClientWrapper`；macOS 15 Sequoia 起拆分为 `_IPMDServerClientWrapperModern` 与 `_IPMDServerClientWrapperLegacy`）包含 `_xpcConnection`（NSXPCConnection）或 `_clientDOProxy`——这些连线资源同样永久泄漏。IMK 以全域缓存管理这些 wrapper，必须透过 private class method（`+terminateForClientXPCConn:` / `+terminateForClientDOProxy:`）才能将其移除。

5. macOS 15 Sequoia 起已移除 `+terminateForClient:`（无后缀版本），仅保留 XPC 与 DO 两变体。

6. ARC 模式下，retain/release 的高频开销在 CapsLock 快速切换时导致可感知的卡顿。停用 ARC（`-fno-objc-arc`）可彻底消除卡顿，但必须自行处理孤弃 controller 与残留 XPC 连线的清理。

### 5.2 修复方案：MRC + KVC Prune + 极性双缓冲 Session 池

基于上述发现，唯音输入法项目采用了以下架构（代号「Phase 115」）：

**a) IMK 交互层全面采用 MRC。** `IMKInputSessionController` target 以 `-fno-objc-arc` 编译。所有 controller 的 alloc/dealloc/retain/release 均在 ObjC MRC 层完成。Swift 端仅透过 raw 内存位址（`uintptr_t`）与 controller 交互，不执行任何 Swift ARC 操作。

**b) 以 KVC 清理孤弃 controller。** 每个 controller 在 init 时被分配一个单调递增的 generation number。当 `_controllers` dictionary 超过 2 个条目时，自动找出 generation 最旧的非活跃 controller，透过 KVC（`[server valueForKeyPath:@"_private._controllers"] removeObjectForKey:oldKey]`）将其移除，使其在 MRC 下正常释放。

**c) 极性双缓冲 Session 池。** 以 controller generation number 的奇偶性（`generation % 2`）将所有 controller 映射至两枚 static `InputSession` singleton：偶数→session A，奇数→session B。每次 CpLk toggle 带来的新 controller 自动对接至**另一个** session；上一个 session 保留不释放，等待下一个同极性 controller 接手。此举完全消除了 InputSession 的动态分配、LRU 淘汰逻辑，以及缓存过期的风险。

**d) 延迟释放与 XPC 清理。** controller 在 `deactivateServer:` 后不会立即释放，而是排程 3 秒后执行清理：释放所有 blocks、调用 client wrapper 的 private `+terminateForClient*` class method（依序尝试 `_IPMDServerClientWrapperModern` → `_IPMDServerClientWrapperLegacy` → `IPMDServerClientWrapper` 以向前兼容 macOS 27）、清除 controller↔session 对照表。若 3 秒内 controller 被重新启用（`activateServer:`），则取消排程、恢复正常服务。这确保了 CpLk 快速切换时不会出现「旧 controller 已死、新 controller 尚未就绪」的真空期。

**e) Dangling pointer 防护。** `IMKControllerLifetimeTracker` singleton 在每次以 `takeUnretainedValue()` 解读 raw 位址前复查 controller 是否仍存活。`unregisterSessionAddr` 仅在 session 仍归属于该 controller 时才清空 `inputControllerAssignedAddr`（防止已 reassign 给新 controller 的 session 被旧 controller 的延迟 dealloc 误清）。`reassign` 同步清除旧 controller 在 `sessionAddrByControllerAddr` 中的残留 mapping。

**f) IME 菜单注入。** `IMEMenuSputnik` 利用 ObjC Runtime 的 `class_addMethod` + `imp_implementationWithBlock` 将 Swift closure 直接注册为 `IMKInputSessionController` 的 method IMP，无需 `@objc` Selector 暴露即可动态构建输入法菜单。菜单注入两项即时监测数据：(1) `IMKControllerLifetimeTracker` 所追踪的 controller 存活数量；(2) 以 `task_vm_info.internal` 取得的 anonymous private memory 用量。菜单开启前会先调用 `purgeMallocZones()` 强制回收 allocator cache，以确保内存读数真实。

### 5.3 架构总结

ARC 从 IMK dispatch 路径中完全移除，CapsLock 卡顿彻底消除。两枚 singleton session 取代 LRU 缓存，极性路由消除了缓存复杂度。KVC prune 路径补偿了 IMKServer 无法自行清理孤弃 controller 的缺陷。延迟释放确保 XPC 连线被正确终止而非泄漏。

以下为 controller 侧架构的最小化示意：

```swift
// IMKInputSessionController（MRC 编译的 .m 文件）：
// - Class-level static blocks 透过 raw address 向 Swift 分派。
// - +IMKSwift_configureWithActivatingServer: 等在启动时一次性设定。

// Swift 端——极性路由：
public final class InputSession {
  /// 两枚预先配置的 singleton session。
  fileprivate static let sessionEven = InputSession(preallocated: ())
  fileprivate static let sessionOdd  = InputSession(preallocated: ())

  /// 依 controller 的 generation parity 解析对应 session。
  static func session(for ctlAddr: UInt) -> InputSession? {
    let parity = IMKControllerLifetimeTracker.shared().generationForAddress(ctlAddr) % 2
    return parity == 0 ? sessionEven : sessionOdd
  }
}

// SessionControllerSputnik.callCoreAtLeastOnce:
// 1. 由 controller generation 决定 parity。
// 2. 路由至 sessionEven 或 sessionOdd。
// 3. 重新指派 client provider（无缓存查询、无淘汰逻辑）。
```

> **注意：** 极性池刻意不以 client 位址为键。它接受 CapsLock 切换会产生不同位址的新 proxy 这一现实，改以 controller generation parity——一个稳定、单调递增的计数器——作为路由键。两个 session 的上限是充分的，因为同时最多只有两个 controller 可能活跃（当前的一个与正在被 deactivate 的一个）。

## 6. 将输入法所有程式内容写成 Swift Package Library

macOS 的输入法无法用 breakpoint 等方式侦错，因为会无限冻结任何沾过你的输入法的 clients，进而冻结你的整个桌面，最终得依赖外部 SSH 连到你的电脑上杀掉输入法执行绪才行。你需要自己写单元测试搭配自己写的 mockup client 来测试。这样的话，将输入法的所有业务内容写成 Library 会更便于这种侦错，还能允许开发者灵活地指定专用的 UserDefaults 容器来实现封闭测试。更甚者，你还可以写个标准的 AppKit App 模拟这个单元测试打字过程，然后用 Instruments 监测是否有运存泄漏。这远比仅保留一个输入法本体 Xcode Target 要灵活得多。

## 7. 运存占用量自查自纠，必要时自尽以释放运存

用户电脑的运存空间寸土寸金。虽然 macOS 26 的 AppKit 糟糕的 NSWindow 绘制效率导致一款输入法平均占用的运存可能从 80MB 暴涨到 200MB 左右。但笔者在这里介绍的一个设计应该不坏：让输入法每次 activateServer 切换到新的打字会话的时候，检查输入法自身占用的运存。如果发现占用的运存的量超过 1024MB 的话，就让输入法抛出 NSNotification 用户通知之后自尽。这个 NSNotification 用户通知的内容就是告知这个情况，免得用户以为输入法崩溃掉。

当然，这个技巧只是兜底策略、防止在用户的电脑上发生像是「运存用尽」这样的灾难性的后果。但开发者仍有义务主动检查自己写的东西是否有运存泄漏的危险。

> ⚠️ 如果你的输入法有在用 SQLite 的话，需要额外注意一个冷门常识：用 SQLite 跑完每一笔查询之后一定要用 `sqlite3_finalize(StatementPointer)` 释放运存，不然会产生**连 Xcode Instruments 都抓不到**的运存泄漏。

## 8. 让输入法用到的 NSWindow 数量尽可能地少

这一条是针对 macOS 26 开始的现状而不得已的规范，因为：自 macOS 26 开始，只要是 NSWindow 用过的运存空间，就都不会被系统刻意回收掉 NSWindow 每个副本的基础开销、且这个基础开销因为 LiquidGlass 的原因而非常高昂。哪怕你确实没启用 LiquidGlass 效果，也没差。在 Info.plist 当中启用 `UIDesignRequiresCompatibility` 虽然可以让运存占用量下降到 macOS 15 的水准，但这只是缓兵之计、且 Apple 随时都会废掉 `UIDesignRequiresCompatibility` 这个 InfoPlist 属性。

> 笔者推测：macOS 26 占用硬碟空间这么大，很可能是系统卷宗里面包了一个 macOS 15 AppKit 环境、专门用来对这个 InfoPlist 属性提供 backward compatibility。

现在 SwiftUI 这么强了，开发者完全可以考虑将「工具提示 Panel」与「自己搓的选字窗」整合到同一个 NSPanel 里面，这样就少了一份 NSWindow 基础开销。输入法的「关于」视窗也可以整入输入法自身的「偏好设定」里面。

> NSPanel 是 NSWindow 的变种。

## 9. IMKCandidates 不要用就对了

前文提到的那个 NumberInput 范例都不敢用 IMKCandidates 选字窗，因为 IMKCandidates 就是一包陈年粪便、臭到现在。你看 macOS 26 系统内建的日语输入法就是 IMKCandidates 的受害者，连文字都看不清：

![image](./macOS_Input_Method_Development_Guidelines_2026-illust2.png)

玻璃背景居然全透明了、把白色整个透上来。偏偏选字窗的文字也是白色的。这种问题一眼看出来就是缺乏单元测试惹的祸，因为这很明显就是 Liquid Glass API 没正确使用所导致的。

现在 AI 技术这么发达，你用 AI 帮你写一个类似 IMKCandidates 那种布局的输入法选字窗面板应该也不难。当然，如果你用强行曝露 IMKCandidates 内部 API 的方式来使用的话，有些 API 从 macOS 10.14 Mojave 开始是固定可用的，但将来就不好说了。

> 笔者给自己开发的唯音输入法就使用了自己搓的田所选字窗，与 IMK 多行选字窗相比也算提供了比较迫真的体验。

![image](./macOS_Input_Method_Development_Guidelines_2026-illust3.png)

## 结尾

InputMethodKit 是历史产物，但它至今仍是 macOS 输入法唯一的官方入口。既然如此，开发者就必须接受这套框架的历史包袱，并在其结构性缺陷之上建立自己的工程纪律。

本文所列规范，本质上并非「技巧」，而是一套风险控制模型：将 IMKInputController 变为纯转接层、将业务逻辑完全模组化、将 MainActor 当作不可违抗的事实、将运存压力视为设计输入条件、将 Sandbox 视为最低限度的道德底线。

若有一天 Apple 彻底重写 InputMethodKit，这些规范或许会过时；但在那之前，macOS 输入法若想在 2026 年仍保持工程品质与资安可信度，就必须把「自我约束」写进架构，而不是写在 README 里。

$ EOF.
