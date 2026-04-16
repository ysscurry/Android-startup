# `init.cpp` 文件解读与前后调用关系说明

## 1. 文档目的

本文用于解读 `\system\core\init\init.cpp` 的职责定位、核心流程以及前后调用关系，便于快速理解 Android `init` 第二阶段（second stage）的启动与调度逻辑。

---

## 2. 文件定位

`init.cpp` 不是 Android 启动最早的入口，而是 **`init` 进程完成 first stage 和 SELinux setup 之后进入的 second stage 主控文件**。

它的核心职责包括：

1. 初始化属性系统
2. 导入 kernel cmdline / device tree 启动参数
3. 完成 second stage 的 SELinux 运行环境准备
4. 启动 property service、signal 处理、epoll 主循环
5. 解析 `init.rc` 及各分区的 `*.rc`
6. 将 `on xxx` / `service xxx` 转换为内存中的 `Action` / `Service`
7. 驱动系统启动触发器（`early-init`、`init`、`late-init`、property triggers）
8. 在运行期管理服务启停、子进程回收、重启和关机

可以把它理解为：

> `init.cpp` 是 Android `init` second stage 的“总调度器”。

---

## 3. 前置调用关系：它是怎么被调用到的

### 3.1 入口在 `main.cpp`

`Y:\UIS8581awes\system\core\init\main.cpp` 中的 `main()` 会根据参数进入不同分支：

- 无参数：进入 `FirstStageMain()`
- 参数为 `selinux_setup`：进入 `SetupSelinux()`
- 参数为 `second_stage`：进入 `SecondStageMain()`

调用关系如下：

```text
main()
 ├─ FirstStageMain()
 ├─ SetupSelinux()
 └─ SecondStageMain()
```

### 3.2 更完整的启动链

结合 `first_stage_init.cpp` 与 `selinux.cpp`，完整前向链路如下：

```text
kernel
  → /init
    → main()
      → FirstStageMain()
        → execv("/system/bin/init", "selinux_setup")
          → SetupSelinux()
            → execv("/system/bin/init", "second_stage")
              → SecondStageMain()
```

### 3.3 含义

因此，`init.cpp` 中的 `SecondStageMain()` 是：

- 已完成基础挂载后的第二阶段入口
- 已加载 SELinux policy 并切换到正确上下文后的主循环入口
- Android 系统正式执行 `rc` 启动逻辑的起点

---

## 4. 文件整体结构概览

从源码职责看，`init.cpp` 可以拆成以下几部分：

### 4.1 全局状态管理

文件开头维护了一批 second stage 运行状态：

- `property_triggers_enabled`：属性触发器是否启用
- `default_console`：默认控制台设备
- `signal_fd`：用于接收 `SIGCHLD` / `SIGTERM`
- `property_fd`：与 property service 通信的 fd
- `waiting_for_prop`：是否处于 `wait_for_prop` 状态
- `shutdown_command` / `do_shutdown` / `shutting_down`：关机/重启控制状态
- `late_import_paths`：延迟导入的 rc 路径
- `subcontexts`：SELinux 子上下文支持

这些状态说明该文件本身就是 second stage 的主状态中心。

### 4.2 rc 解析入口

关键函数：

- `CreateParser()`
- `CreateServiceOnlyParser()`
- `LoadBootScripts()`

作用是把 `.rc` 文件里的：

- `on xxx`
- `service xxx`
- `import xxx`

转换为内存中的：

- `Action`
- `Service`
- 递归导入的 parser 行为

### 4.3 事件处理入口

关键函数：

- `property_changed()`
- `HandleControlMessage()`
- `HandleSignalFd()`
- `HandlePropertyFd()`
- `HandleProcessActions()`

这些函数共同把属性变化、控制消息、信号事件、服务超时和重启逻辑接到主循环里。

### 4.4 second stage 主函数

核心入口：

- `SecondStageMain(int argc, char** argv)`

这是整个文件的执行主线。

---

## 5. `SecondStageMain()` 主流程解读

下面按代码执行顺序解读 second stage。

### 5.1 very early 初始化

进入 `SecondStageMain()` 后，首先执行：

- panic reboot handler 安装
- `SetStdioToDevNull(argv)`
- `InitKernelLogging(argv)`
- 写 `/proc/1/oom_score_adj = -1000`
- `GlobalSeccomp()`
- 初始化 session keyring
- 创建 `/dev/.booting`

目的：

- 准备 second stage 基础运行环境
- 确保 `init` 本身不会轻易被 OOM 杀掉
- 根据启动参数决定是否全局启用 seccomp

---

### 5.2 初始化属性系统

调用：

- `property_init()`

其内部在 `property_service.cpp` 中完成：

- 初始化 `/dev/__properties__`
- 创建 serialized property info
- 初始化系统 property area
- 加载 property contexts

调用链：

```text
SecondStageMain()
  → property_init()
    → CreateSerializedPropertyInfo()
    → __system_property_area_init()
    → property_info_area.LoadDefaultPath()
```

这一步完成后，系统属性机制才真正准备就绪。

---

### 5.3 导入 kernel / DT 启动参数

依次调用：

- `process_kernel_dt()`
- `process_kernel_cmdline()`
- `export_kernel_boot_props()`

#### `process_kernel_dt()`

从 Android device tree 目录读取信息，写入 `ro.boot.xxx`。

#### `process_kernel_cmdline()`

通过 `import_kernel_cmdline(false, import_kernel_nv)` 导入 kernel cmdline 中的参数。

其中 `import_kernel_nv()` 负责：

- `androidboot.xxx` → `ro.boot.xxx`
- `qemu` → 标记 emulator
- `wifionly` → `ro.radio.noril`

#### `export_kernel_boot_props()`

把一部分 `ro.boot.xxx` 重新映射成通用属性，例如：

- `ro.boot.mode` → `ro.bootmode`
- `ro.boot.baseband` → `ro.baseband`
- `ro.boot.hardware` → `ro.hardware`
- `ro.boot.ddrsize` → `ro.ramsize`

说明：

这一组逻辑是 second stage 获取板级启动信息的关键入口。

---

### 5.4 设置 boot timing / AVB / debug 状态

这里会把前一阶段通过环境变量传递下来的信息写成属性：

- `ro.boottime.init`
- `ro.boottime.init.selinux`
- `ro.boot.avb_version`

同时判断：

- 如果设备解锁且 `INIT_FORCE_DEBUGGABLE=true`
- 则允许在后续加载 debug props

然后清理这些环境变量：

- `INIT_STARTED_AT`
- `INIT_SELINUX_TOOK`
- `INIT_AVB_VERSION`
- `INIT_FORCE_DEBUGGABLE`

说明前后调用关系为：

```text
FirstStageMain()/SetupSelinux()
  → setenv(...)
    → SecondStageMain()
      → property_set("ro.boottime.*", ...)
```

---

### 5.5 second stage SELinux 收尾

调用：

- `SelinuxSetupKernelLogging()`
- `SelabelInitialize()`
- `SelinuxRestoreContext()`

注意：

这里不是再次加载 SELinux policy，而是：

- 建立 second stage 下的 SELinux logging
- 初始化 file context lookup handle
- restorecon 一些 second stage 运行必须的路径

---

### 5.6 建立 epoll 与 signal 处理框架

先创建：

- `Epoll epoll`
- `epoll.Open()`

再调用：

- `InstallSignalFdHandler(&epoll)`

其内部做了这些事：

1. 屏蔽 `SIGCHLD`
2. 容器环境下额外屏蔽 `SIGTERM`
3. 使用 `signalfd()` 创建 `signal_fd`
4. 将 `signal_fd` 注册进 epoll
5. 用 `pthread_atfork()` 保证子进程解除 signal block

形成链路：

```text
SecondStageMain()
  → InstallSignalFdHandler(&epoll)
    → signalfd(...)
    → epoll.RegisterHandler(signal_fd, HandleSignalFd)
```

意义：

`init` 不依赖传统异步 signal handler，而是把 signal 统一纳入 epoll 事件循环。

---

### 5.7 加载默认属性并启动 property service

依次调用：

- `property_load_boot_defaults(load_debug_prop)`
- `UmountDebugRamdisk()`
- `fs_mgr_vendor_overlay_mount_all()`
- `export_oem_lock_status()`

其中 `property_load_boot_defaults()` 在 `property_service.cpp` 中负责加载：

- `prop.default`
- `build.prop`
- vendor / odm / product 等分区属性
- 派生 `ro.product.*`
- 派生 `ro.build.fingerprint`

你的这套代码里还扩展了设备级属性初始化：

- `setScreenRotation()`
- `setStorageSizeProp()`
- `setStorageSerial()`
- `updataWallclock()`
- `setBoardId()`

随后启动 property service：

- `StartPropertyService(&property_fd)`
- `epoll.RegisterHandler(property_fd, HandlePropertyFd)`

这一步建立了 `init` 与 property service 的消息通道。

---

### 5.8 初始化其它 second stage 基础设施

继续执行：

- `MountHandler mount_handler(&epoll)`
- `set_usb_controller()`
- `SetupMountNamespaces()`
- `subcontexts = InitializeSubcontexts()`

说明这个文件还负责：

- mount 相关事件接入
- USB gadget controller 自动选择
- mount namespace 初始化
- subcontext 初始化

---

### 5.9 建立 builtins 映射并解析启动脚本

调用：

- `const BuiltinFunctionMap function_map;`
- `Action::set_function_map(&function_map);`
- `LoadBootScripts(am, sm);`

#### `Action::set_function_map(&function_map)`

作用：

将 rc 里的 builtin 命令映射到 `builtins.cpp` 中的实现，例如：

- `start`
- `stop`
- `setprop`
- `mount_all`
- `mkdir`
- `wait_for_prop`

#### `LoadBootScripts(am, sm)`

这是 second stage 中 `.rc` 脚本加载入口。

默认逻辑：

- 若 `ro.boot.init_rc` 未指定
  - 若 `ro.bootmode == charger`，解析 `/vendor/etc/init/charge.rc`
  - 否则解析：
    - `/init.rc`
    - `/system/etc/init`
    - `/product/etc/init`
    - `/product_services/etc/init`
    - `/odm/etc/init`
    - `/vendor/etc/init`
- 若目录当下解析失败，会放到 `late_import_paths`

你这套代码里的启动模式还额外区分：

- `charger`
- `cali`
- `factorytest`

---

### 5.10 设置初始 trigger 并启动主循环前的动作队列

脚本解析完成后，会依次入队一批 action：

- `SetupCgroupsAction`
- `early-init`
- `wait_for_coldboot_done_action`
- `MixHwrngIntoLinuxRngAction`
- `SetMmapRndBitsAction`
- `SetKptrRestrictAction`
- `KeychordInit`
- `console_init_action`
- `init`
- `StartBoringSslSelfTest`
- 再次 `MixHwrngIntoLinuxRngAction`
- `InitBinder`
- 根据 bootmode 触发：
  - `charger`
  - `cali`
  - `factorytest`
  - 或 `late-init`
- `queue_property_triggers_action`

这意味着从这里开始：

- `on early-init`
- `on init`
- `on late-init`
- `on property:xxx=yyy`

等 action 都开始有机会被执行。

---

## 6. rc 解析链路

这部分是理解 `init.cpp` 的关键之一。

### 6.1 `CreateParser()`

注册三类 section parser：

- `service` → `ServiceParser`
- `on` → `ActionParser`
- `import` → `ImportParser`

### 6.2 `LoadBootScripts()`

调用：

- `parser.ParseConfig(...)`

### 6.3 `Parser::ParseConfig()`

在 `parser.cpp` 中，会判断传入路径是文件还是目录：

- 文件 → `ParseConfigFile()`
- 目录 → `ParseConfigDir()`

最终都走到 `ParseData()`。

### 6.4 `ParseData()` 如何分发

`ParseData()` 会逐行 token 化并分发：

- 遇到 `on` → 交给 `ActionParser`
- 遇到 `service` → 交给 `ServiceParser`
- 遇到 `import` → 交给 `ImportParser`

### 6.5 各 parser 的落点

#### `ActionParser`

- 解析 trigger
- 构造 `Action`
- 通过 `ActionManager::AddAction()` 放入动作管理器

#### `ServiceParser`

- 解析 service 定义与 option
- 构造 `Service`
- 通过 `ServiceList::AddService()` 放入服务列表

#### `ImportParser`

- 收集 import 路径
- 在 `EndFile()` 里递归调用 `parser_->ParseConfig(import)`

### 6.6 rc 解析总链路

```text
SecondStageMain()
  → LoadBootScripts()
    → CreateParser()
      → parser.ParseConfig(...)
        → Parser::ParseData()
          ├─ ActionParser::ParseSection()
          │    → ActionManager::AddAction()
          ├─ ServiceParser::ParseSection()
          │    → ServiceList::AddService()
          └─ ImportParser::ParseSection()
               → ImportParser::EndFile()
                 → parser.ParseConfig(被 import 的 rc)
```

---

## 7. Action / Service 调度链路

### 7.1 `ActionManager` 的角色

`ActionManager` 负责：

- 保存所有 `Action`
- 保存事件队列 `event_queue_`
- 匹配 trigger
- 一次执行一个命令

### 7.2 事件入队方式

`init.cpp` 中的几种典型入队方式：

- `QueueBuiltinAction(...)`
- `QueueEventTrigger("early-init")`
- `QueueEventTrigger("init")`
- `QueueEventTrigger("late-init")`
- `QueueAllPropertyActions()`
- `QueuePropertyChange(name, value)`

### 7.3 执行逻辑

主循环中调用：

- `am.ExecuteOneCommand()`

其内部流程：

1. 从 `event_queue_` 取一个事件
2. 遍历所有 `Action`
3. 调 `Action::CheckEvent(...)` 判断是否命中
4. 命中的 action 加入当前执行队列
5. 执行一条命令 `Action::ExecuteOneCommand()`
6. 命令实际通过 `Command::InvokeFunc()` 调到 builtin 或 subcontext

总链路：

```text
SecondStageMain() 主循环
  → ActionManager::ExecuteOneCommand()
    → Action::CheckEvent()
    → Action::ExecuteOneCommand()
      → Command::InvokeFunc()
        → builtins.cpp 中对应函数
```

### 7.4 builtins 最终如何影响服务

例如：

- `start xxx` → `do_start()` → `Service::Start()`
- `stop xxx` → `do_stop()` → `Service::Stop()`
- `restart xxx` → `do_restart()` → `Service::Restart()`
- `class_start core` → `do_class_start()` → 遍历该 class 下所有 service 执行 `StartIfNotDisabled()`

这就是 rc 命令最终落到 service 生命周期控制的过程。

---

## 8. property / control 主链路

这条链是 second stage 的另一条核心主线。

### 8.1 外部进程设置属性

当外部进程调用 setprop 时，property service 在 `property_service.cpp` 中处理：

- `handle_property_set_fd()`
- `HandlePropertySet()`

### 8.2 两种主要分支

#### 普通属性

- `PropertySet(name, value)`
- `SendPropertyChanged(name, value)`
- 通过 socket 发给 init

#### 控制属性 `ctl.*`

- `SendControlMessage(msg, name, pid, ...)`
- 通过 socket 发给 init

### 8.3 `init.cpp` 里接收 property service 消息

`StartPropertyService(&property_fd)` 后，`init.cpp` 把 `property_fd` 注册给 epoll：

- `epoll.RegisterHandler(property_fd, HandlePropertyFd)`

`HandlePropertyFd()` 读取 `PropertyMessage`，然后分两类：

#### `kChangedMessage`

调用：

- `property_changed(name, value)`

职责：

- 特判 `sys.powerctl`
- 若已启用 property triggers，则 `QueuePropertyChange(name, value)`
- 若当前正在 `wait_for_prop`，则判断是否满足等待条件

#### `kControlMessage`

调用：

- `HandleControlMessage(msg, name, pid)`

职责：

- 根据 `msg` 找到控制函数
- 根据 target 查找 `Service` 或 `Interface`
- 最终调用：
  - `Service::Start()`
  - `Service::Stop()`
  - `Service::Restart()`

### 8.4 property / control 链路总图

```text
外部进程 setprop
  → property_service.cpp::handle_property_set_fd()
    → HandlePropertySet()
      ├─ 普通属性 → PropertySet() → SendPropertyChanged()
      └─ ctl.*   → SendControlMessage()
            ↓
      通过 init_socket 发给 init
            ↓
init.cpp::HandlePropertyFd()
  ├─ kChangedMessage
  │   → property_changed()
  │      ├─ sys.powerctl → do_shutdown = true
  │      ├─ QueuePropertyChange(name, value)
  │      └─ 检查 wait_for_prop
  └─ kControlMessage
      → HandleControlMessage()
         → Service::Start()/Stop()/Restart()
```

---

## 9. signal / service 生命周期链路

### 9.1 启动 service

service 可以通过以下入口启动：

- rc 中的 `start xxx`
- rc 中的 `class_start xxx`
- `ctl.start=xxx`
- keychord 触发
- 自动 restart

最终都会落到：

- `Service::Start()`

`Service::Start()` 内部完成：

- 检查 service 状态
- 检查 console / 可执行文件 / seclabel
- `fork()` 或 `clone()`
- 子进程设置 namespace、环境变量、descriptor、uid/gid/caps
- 最终 `execv()` 启动真实服务进程

### 9.2 子进程退出后的处理

子进程退出后，内核会发 `SIGCHLD`。

由于 `init.cpp` 已经通过 `signalfd` 统一接管信号，因此流程如下：

```text
child exit
  → signal_fd 可读
    → HandleSignalFd()
      → ReapAnyOutstandingChildren()
```

### 9.3 `ReapAnyOutstandingChildren()`

在 `sigchld_handler.cpp` 中：

- `waitid()` 找到僵尸子进程
- 定位它对应的 `Service`
- 记录退出日志
- 调 `Service::Reap(siginfo)`

### 9.4 `Service::Reap()` 做什么

主要处理：

- 清理 process group
- 清理 descriptor 资源
- 临时服务移除
- 对 oneshot / disabled / reset / restart 状态做转换
- 非 disabled 服务进入 `SVC_RESTARTING`
- 执行 `onrestart` 命令
- `NotifyStateChange("restarting")`

### 9.5 `HandleProcessActions()` 如何续上 restart 链

在 `init.cpp` 主循环里：

- `HandleProcessActions()` 遍历所有 service
- 若运行中且超时 → `s->Timeout()`
- 若处于 `SVC_RESTARTING` 且 restart 时间已到 → `s->Start()`

因此完整 service 生命周期链路是：

```text
Service::Start()
  → fork/clone + exec

子进程退出
  → SIGCHLD
    → HandleSignalFd()
      → ReapAnyOutstandingChildren()
        → Service::Reap()

主循环下一轮
  → HandleProcessActions()
    → 若满足重启条件 → Service::Start()
```

---

## 10. 关机/重启链路

### 10.1 `sys.powerctl` 的特殊处理

`property_changed()` 中特判：

- 如果属性名为 `sys.powerctl`
- 不直接调用 `HandlePowerctlMessage()`
- 而是设置：
  - `shutdown_command = value`
  - `do_shutdown = true`

原因是：

- 直接在属性回调里修改 action queue 可能破坏调度状态
- 所以延迟到主循环统一处理

### 10.2 主循环中的关机处理

在 `while (true)` 开头：

```text
if (do_shutdown && !shutting_down)
  → HandlePowerctlMessage(shutdown_command)
  → shutting_down = true
```

### 10.3 `HandlePowerctlMessage()` 在 `reboot.cpp` 中的动作

它会：

1. 解析命令：`shutdown,...` 或 `reboot,...`
2. `ActionManager::ClearQueue()`
3. `QueueEventTrigger("shutdown")`
4. `QueueBuiltinAction(shutdown_done)`
5. `ResetWaitForProp()`
6. 清理 exec service 状态
7. 通知 property service 停止向 init 发送 property changed 消息
8. 最终走到 `DoReboot(...)`

总链路：

```text
某进程设置 sys.powerctl=...
  → property_service
    → init.cpp::property_changed("sys.powerctl", value)
      → do_shutdown = true

主循环下一轮
  → HandlePowerctlMessage(value)
    → reboot.cpp
      → ClearQueue()
      → QueueEventTrigger("shutdown")
      → QueueBuiltinAction(shutdown_done)
      → DoReboot(...)
```

---

## 11. 主循环：`init.cpp` 的“心脏”

`while (true)` 这一段可以理解为 second stage 的主调度器。

逻辑顺序如下：

1. 默认 `epoll_timeout = nullopt`
2. 若存在待处理的关机命令，则优先触发 shutdown 流程
3. 若当前没有 `wait_for_prop` 且没有 exec service 正在运行，则执行一条 action 命令
4. 若系统未处于关机中，则调用 `HandleProcessActions()` 处理超时与重启
5. 若 action 队列还有待执行命令，则 `epoll_timeout = 0ms`
6. 调 `epoll.Wait(epoll_timeout)` 等待 fd 事件

它等到的事件主要有：

- `signal_fd` → `HandleSignalFd()`
- `property_fd` → `HandlePropertyFd()`
- mount handler 相关 fd
- keychord 相关 fd

这说明 second stage 本质上是一个：

> **基于 epoll 的事件驱动调度循环**。

---

## 12. 关键函数速查

### `LoadBootScripts()`

- second stage 的 rc 装载入口
- 根据 bootmode 选择要解析的 rc
- 解析失败的目录暂存到 `late_import_paths`

### `property_changed()`

- 属性变化进入 init 的统一入口
- 将 property change 转换为 action trigger
- 特判 `sys.powerctl`
- 处理 `wait_for_prop`

### `HandleControlMessage()`

- 处理 `ctl.start` / `ctl.stop` / `ctl.restart`
- 根据 service 名或 interface 名定位目标
- 最终调用 `Service::Start/Stop/Restart`

### `HandleSignalFd()`

- 处理 `SIGCHLD`
- 处理容器场景中的 `SIGTERM`
- 把 signal 纳入 epoll 驱动

### `HandlePropertyFd()`

- 从 property service 接收 protobuf 消息
- 分发到：
  - `property_changed()`
  - `HandleControlMessage()`

### `HandleProcessActions()`

- 检查运行中服务是否超时
- 检查 `SVC_RESTARTING` 服务是否达到重启时间

### `SecondStageMain()`

- 整个 second stage 的总入口与主循环

---

## 13. 这个项目中的定制点

在这份代码中，除了 AOSP 标准逻辑，还能看到一些厂商定制：

### 13.1 启动模式扩展

除标准 `charger` 外，还支持：

- `cali`
- `factorytest`

并分别触发对应 trigger：

- `am.QueueEventTrigger("cali")`
- `am.QueueEventTrigger("factorytest")`

### 13.2 `ro.tiny` 扩大冷启动等待时间

`wait_for_coldboot_done_action()` 中：

- 默认等待 `COLDBOOT_DONE` 60 秒
- 若 `ro.tiny` 非空，改为 600 秒

### 13.3 `wifionly` cmdline 属性映射

在 `import_kernel_nv()` 中：

- `wifionly` → `ro.radio.noril`

这是设备裁剪逻辑。

### 13.4 USB controller 自动探测

`set_usb_controller()` 从 `/sys/class/udc` 读取第一个可用 UDC，写入：

- `sys.usb.controller`

---

## 14. 总结

一句话总结：

> `init.cpp` 是 Android `init` second stage 的主调度器，前面承接 first stage 与 SELinux setup，后面通过 epoll 主循环驱动 rc 脚本、属性变化、服务生命周期、信号回收以及关机重启流程。

如果按职责来理解，它主要连接了以下几个方向：

- **前向承接**：`FirstStageMain()` / `SetupSelinux()`
- **向下驱动**：`ActionManager` / `ServiceList`
- **横向连接**：`property_service` / `sigchld_handler` / `reboot`
- **运行时维持**：通过 `while(true) + epoll.Wait()` 保持系统调度

---

## 15. 一张总调用关系图

```text
kernel
  ↓
/init
  ↓
main.cpp::main()
  ↓
first_stage_init.cpp::FirstStageMain()
  ↓ exec("/system/bin/init", "selinux_setup")
selinux.cpp::SetupSelinux()
  ↓ exec("/system/bin/init", "second_stage")
init.cpp::SecondStageMain()
  ├─ property_init()
  ├─ process_kernel_dt()
  ├─ process_kernel_cmdline()
  ├─ export_kernel_boot_props()
  ├─ SelabelInitialize()
  ├─ SelinuxRestoreContext()
  ├─ InstallSignalFdHandler()
  ├─ property_load_boot_defaults()
  ├─ StartPropertyService()
  ├─ LoadBootScripts()
  │    ├─ ActionParser → ActionManager
  │    ├─ ServiceParser → ServiceList
  │    └─ ImportParser → 继续解析 rc
  ├─ QueueBuiltinAction(...)
  ├─ QueueEventTrigger("early-init"/"init"/"late-init"...)
  └─ while(true)
       ├─ HandlePowerctlMessage()
       ├─ ActionManager::ExecuteOneCommand()
       ├─ HandleProcessActions()
       └─ epoll.Wait()
            ├─ HandleSignalFd()
            │    └─ ReapAnyOutstandingChildren()
            │         └─ Service::Reap()
            └─ HandlePropertyFd()
                 ├─ property_changed()
                 └─ HandleControlMessage()
                      └─ Service::Start/Stop/Restart()
```

