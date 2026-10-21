# 写在2026年的macOS输入法开发规范

InputMethodKit 自 macOS 10.5 Leopard 时代问世，早于 Objective-C ARC 技术、XPC 通讯技术、Sandbox 技术问世（均为 macOS 10.7）之前。自然，这也是早于 Swift 5 与 SwiftUI 流行之前。也就是说，InputMethodKit 是横跨了两代技术大变革的祖产级 OS Framework。当年 Apple 写给 macOS 10.5 Leopard 的 IMK 参考手册《[Input Method Kit Framework Reference](https://leopard-adc.pepas.com/documentation/Cocoa/Reference/InputMethodKitFrameworkRef/InputMethodKitFrameworkRef.pdf)》（下文简称《IMKFR》）早已不符合这些变革所带来的新要求（特别是 Swift 6 Concurrency）。笔者根据自己开发[《唯音输入法》(for macOS 10.09 Mavericks ~ macOS 27)](https://vchewing.github.io/)的经验，将一些注意事项整理在此，留给其他想给 macOS 开发输入法的工程师们参考。

> 笔者另外制作了 [IMKSwift](https://github.com/vChewing/IMKSwift) 套件，允许 Swift 工程师们在写 IMK 输入法时更顺利：IMKSwift 提供了 IMKInputSessionController 基础型别、是在 IMKInputController 的基础上整体换用了对 Modern Swift Concurrency 更友好的 ObjC Header 表达。该套件另已内置 §5 所述的 MRC 私有面层；下文 §5 的实现细节可以当作参考实现，而非待办清单。

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

## 5. IMKInputController 必须是 MRC 管理，且客体绑定必须逐事件重建

### 5.1 IMKServer 控制器泄漏缺陷（逆向工程结论）

笔者针对 InputMethodKit 在 macOS 10.9 Mavericks、10.15 Catalina 与 macOS 27 GoldenGate（arm64e）上做了大量逆向工程，发现 IMKServer 在控制器生命周期管理上有一个根本缺陷：

1. IMKServer 内部以 `_private._controllers` 这个 `NSMutableDictionary` 追踪所有 `IMKInputController` 实例，键是 `[NSNumber numberWithUnsignedLong:clientProxyAddr]`——也就是每个控制器在 `init` 时所收到的客体代理对象的地址。

2. **每一次 CapsLock 切换都会产生一个全新的 DO/XPC 代理对象**，地址不同。旧控制器在字典里的键从此再也查不到，因为 IMKServer 内建的控制器重用逻辑是以业已改变的代理地址为键。旧控制器就此成为永久的孤儿。

3. IMKServer 有 `sessionFinished:` 可以清理某个 App 累积下来的控制器，但 `sessionFinished:` 是按客体代理逐一派发的。CapsLock 一换，旧代理已被替换，IMKServer 不会再对它发任何消息，于是孤儿控制器永远收不到 `sessionFinished:`。**每切一次 CapsLock，就漏掉一个控制器。**

4. 孤儿控制器持有客体连接包装对象（macOS ≤ 10.15 为 `IPMDServerClientWrapper`；macOS 15 Sequoia 起拆为 `_IPMDServerClientWrapperModern` 与 `_IPMDServerClientWrapperLegacy`），其内又持有 `_xpcConnection`（NSXPCConnection）或 `_clientDOProxy`——这些连接资源同样永久泄漏。IMK 以全局缓存管理这些包装对象，只能通过私有的类方法（`+terminateForClientXPCConn:`、`+terminateForClientDOProxy:`）移除。

5. macOS 15 Sequoia 起，无后缀的 `+terminateForClient:` 已被完全移除，只剩 XPC 与 DO 两个变体。

6. 在 ARC 之下，频繁 CapsLock 切换造成的 retain/release churn 会导致可感知的卡顿。关闭 ARC（`-fno-objc-arc`）可以完全消除卡顿，但从那以后孤儿控制器的清理与残余 XPC 连接的拆除，都得自己来。

7. **孤儿不只是被漏掉——它还保留着退役当时的客体绑定。** 框架里没有任何东西会再更新那个绑定，因为框架里没有任何东西还认为这个控制器在服役。系统此后投给这个控制器的按键，都会以一个已退役客体的身份被回应。

### 5.2 修法：MRC ＋ KVC 修剪 ＋ 按地址索引的会话池 ＋ 逐事件认领

**a) IMK 交互层全面改用 MRC。** `IMKInputSessionController` 的 target 以 `-fno-objc-arc` 编译，控制器的 alloc/dealloc/retain/release 全在 ObjC MRC 层完成。Swift 端只通过原始运存地址（`uintptr_t`）与控制器互动，绝不对控制器实例做 Swift ARC 操作。

**b) 以 KVC 修剪孤儿控制器。** 每个控制器在 init 时取得一个单调递增的世代号。每当有一个控制器被创建、并完成自身登记之后，就以 KVC 移除最旧的闲置控制器（世代号最小者，排除当前控制器与新建的这一个）：`[server valueForKeyPath:@"_private._controllers"] removeObjectForKey:oldKey]`。这让孤儿控制器在 MRC 下被正常析构。

**c) 以控制器地址为键的小型会话池。** 唯音最初以世代奇偶路由会话（偶→会话 A、奇→会话 B，即本文旧版所述的「双缓冲」）。世代单调递增，所以奇偶看起来很稳定——而那正是陷阱：两个世代同奇偶的控制器会共用同一份会话。第一个退役时，该会话的客体绑定被清空；第二个仍继续通过一份不再认得自己客体的会话派发，于是每一个按键都在送达输入法之前就被丢掉。现行设计改以控制器自身的地址为键，会话池最多四份，只有在生命周期追踪器判定控制器已死时才回收该槽，且绝不做「最近最少使用」式的淘汰。

**d) 延迟拆除，并清理 XPC。** `deactivateServer:` 之后不立即析构控制器，而是启动一个三秒计时器。计时器触发时，控制器会：释放所有 block、对适当的客体包装类别调用私有的 `+terminateForClient*` 类方法（依 `_IPMDServerClientWrapperModern` → `_IPMDServerClientWrapperLegacy` → `IPMDServerClientWrapper` 的顺序回退，以兼容 macOS 27），并清除控制器↔会话的对应。若控制器在三秒内被重新启用（`activateServer:`），计时器即取消、正常服务。这避免了快速 CapsLock 切换时「旧控制器已死、新控制器还没准备好」的空窗。

> 延迟拆除买到的是「重用窗口」；它**不**代表控制器已经死了。控制器外壳可能还留在 `_controllers` 里，直到下一次修剪为止——所以千万不要把「计时器已触发」当成存活判据。

**e) 逐事件认领客体。** 每一个送达的事件，在派发之前，会话先把自身重新绑定到「送来这个事件之控制器」的地址。若值未变则跳过写入，因此成本是每按一键一次比较。就是这一行让整个设计能自愈：无论系统把事件路由给哪个副本，会话的客体永远是当下正在送事件的那一位。`IMKControllerLifetimeTracker` 仍然守着每一次对原始地址的 `takeUnretainedValue()`。

**f) 以 ObjC runtime 注入输入法菜单。** 输入法菜单由 `IMEMenuSputnik` 动态组建：以 `class_addMethod` ＋ `imp_implementationWithBlock` 把 Swift 闭包直接注册为 `IMKInputSessionController` 上的方法 IMP，无需暴露 `@objc` selector。菜单会注入两项即时指标：(1) `IMKControllerLifetimeTracker` 追踪的存活控制器数；(2) 由 `task_vm_info.internal` 取得的匿名私有运存用量。每次菜单打开前会调用 `purgeMallocZones()` 清空分配器缓存，使运存读数反映实际用量。

### 5.3 架构摘要

整体效果是：ARC 彻底退出 IMK 派发路径，CapsLock 卡顿消失；以地址为键的小型会话池取代了极性配对，也就消除了别名共享（aliasing）的隐患；KVC 修剪补上了 IMKServer 无力自行清理孤儿的缺口；逐事件认领让「过期的客体绑定」不可能存在；延迟拆除则确保 XPC 连接被正常终止，而不是泄漏。

以下是控制器侧架构的最小示意：

```swift
// IMKInputSessionController（以 MRC 编译的 .m 文件）：
// - 类别层级的 static block 以原始地址把调用派发给 Swift。
// - +IMKSwift_configureWithActivatingServer: 等在启动时设置一次。

// Swift 端：以地址为键的会话池
@MainActor
final class SessionPool {
  static let shared = SessionPool(capacity: 4)
  private var slots: [UInt: InputSession] = [:]

  func session(for ctlAddr: UInt) -> InputSession { /* 创建或重用此地址的槽位 */ }
  func reap(using tracker: IMKControllerLifetimeTracker) { /* 收掉控制器已死的槽位 */ }
}

// SessionControllerSputnik 的事件区块：
// 1. 为送来事件的控制器地址取得（或创建）会话。
// 2. adoptClientAddr(ctlAddr)——以送事件的人为准。
// 3. 派发进该会话的核心。
```

> **注：** 会话池以**控制器**地址为键，不是客体地址。控制器才是框架真正管理生命周期的对象；客体只是一个随时可能在其脚下被替换的代理。四个槽位足够，因为系统同时最多只会有一个现役控制器，外加一小截正在退役的尾巴。

### 5.4 可观测性与 kill switch

这些东西没法用断点调试。你能做的是给它装上观测点，并且让你每一项干涉都能被单独关掉。

**日志词汇。** ObjC 层只吐一组固定的词汇：`ControllerInit:`、`Prune:`（`controllers=`、`current=`、`self=`、`evicted … remaining=`）、`ActivateServer:`（含 `clientTerminatedBefore=`）、`DeactivateServer:`、`DelayedDealloc:`（排定／跳过／执行）、`EventEntry:`（`addr`、`alive`、`gen`、`session=`，以及缓存下来的客体名称）、`guard fail:`（`hasHandler`、`hasClient`、`proxyNil`）。macOS 26 起走 `Logger(subsystem: "vChewing", category: "Log")`，更早的系统回退到 `NSLog`；另有调试模式旗标可整体关闭。

**四个仅供诊断的开关**，见于「开发者道场」面板，亦可用原始 `UserDefaults` 键设置：

| 开关 | 停用什么 |
|---|---|
| `configureServerControllerPruningDisabled(_:)` | KVC 修剪 |
| `configureEvictedClientInvalidationDisabled(_:)` | 对被修剪控制器的客体条目作失效处理（只关掉淘汰行为中较激进的那一半） |
| `configureClientWrapperTerminationDisabled(_:)` | 对包装对象调用 `+terminateForClient*`，但保留 dealloc 簿记 |
| `configureControllerDelayedDeallocationDisabled(_:)` | 整个延迟拆除 |

每一个 block 都是在「决策当下」求值、且从不缓存；唯音以四个 `_IMKSwift_disable…` 偏好设置喂给它们，所以运行中进程的开关状态可以即时改变，无需重编。

十六种原始组合，九种可分辨的行为：延迟拆除一关，客体终止与客体失效这两个开关在同一序列里就不可达；而修剪一关，客体失效的开关也就无意义了。加这些开关的理由很单纯：让现场回报可以被二分，而不是只能猜。

**三种故障签名**，这些词汇能帮你分辨：

1. 失灵时段完全没有 `EventEntry:` ⇒ 事件从未送达；
2. 有 `EventEntry:` 但 `alive=false`，或出现 `UnresolvableController:` ⇒ 地址弄丢了；
3. 有 `EventEntry:` 且 `alive=true`，但 `session=` 指向另一个客体，或 `assigned=nil` ⇒ 事件送给了错的客体。

**安装器。** 请在套件本体替换**之后**再抓取进程 PID，依序 `SIGTERM` → `SIGKILL`、轮询确认已空、再验证。不要 `killall` 了事，也不要在替换之前就杀（IMK 会从已注册的套件按需重启输入法，替换前杀只是让旧代码以新 PID 回来）。以 root 身份执行时，请以 `/usr/bin/stat -f%Su /dev/console` 把终止范围限制在当前控制台用户。

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
