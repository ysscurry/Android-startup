# `main.cpp` 文件解读与前后调用关系说明

## 1. 文档目的

本文用于解读 `Y:\UIS8581awes\system\core\init\main.cpp` 的职责定位、代码分支逻辑以及前后调用关系，帮助开发同事快速理解 Android `init` 的统一入口机制。

---

## 2. 文件定位

`main.cpp` 的代码很短，但它在 Android 启动链中的位置非常关键。

它本身并不负责复杂业务逻辑，而是负责：

1. 作为 `/init` 可执行程序的统一入口
2. 根据程序名 `argv[0]` 和启动参数 `argv[1]` 选择运行模式
3. 将流程分发到不同主函数：
   - `ueventd_main()`
   - `SubcontextMain()`
   - `SetupSelinux()`
   - `SecondStageMain()`
   - `FirstStageMain()`

可以把它理解为：

> `main.cpp` 是 Android `init` 的统一入口分发器。

---

## 3. 文件整体职责

从设计角度看，`main.cpp` 解决的是一个核心问题：

> 同一个二进制程序 `/init`，在不同场景下要扮演不同角色。

它支持的角色包括：

- `init` first stage
- `init` second stage
- `SELinux setup` 阶段
- `subcontext` 子进程
- `ueventd`

因此这个文件的主要任务不是“执行启动细节”，而是“判断当前应该进入哪个入口函数”。

---

## 4. 代码结构解读

### 4.1 头文件

文件引入了多个入口对应的头文件：

- `first_stage_init.h`
- `init.h`
- `selinux.h`
- `subcontext.h`
- `ueventd.h`

这说明 `main.cpp` 的职责就是在这些模块之间做分发。

---

### 4.2 ASAN 相关辅助逻辑

当编译启用了 AddressSanitizer 时，文件中定义了：

- `__asan_default_options()`
- `__sanitizer_report_error_summary()`
- `AsanReportCallback()`

这些逻辑用于：

- 指定 ASAN 默认参数
- 将 ASAN 报错通过 Android 日志系统打印出来

在 `main()` 入口最开始会调用：

```cpp
__asan_set_error_report_callback(AsanReportCallback);
```

这部分属于调试增强逻辑，不影响主分发流程。

---

## 5. `main()` 主流程逐段解读

`main()` 的判断顺序非常清晰，本质上是一个多分支调度器。

### 5.1 ASAN 初始化

在开启 ASAN 的构建下，首先安装错误回调：

```cpp
#if __has_feature(address_sanitizer)
    __asan_set_error_report_callback(AsanReportCallback);
#endif
```

目的：

- 在真正进入 init/ueventd/subcontext 等逻辑前，先把 sanitizer 报错接入日志

---

### 5.2 程序名判断：是否作为 `ueventd` 运行

```cpp
if (!strcmp(basename(argv[0]), "ueventd")) {
    return ueventd_main(argc, argv);
}
```

这里不是看参数，而是看程序名 `argv[0]`。

#### 含义

如果当前二进制是以 `ueventd` 名字运行，则直接进入：

- `ueventd_main(argc, argv)`

#### 说明

这是一种“同一二进制多角色复用”的典型设计：

- 当它叫 `init` 时，走 init 启动流程
- 当它叫 `ueventd` 时，走 ueventd 逻辑

调用关系：

```text
main()
  → if basename(argv[0]) == "ueventd"
    → ueventd_main(argc, argv)
```

---

### 5.3 参数判断：是否进入 `subcontext`

```cpp
if (argc > 1) {
    if (!strcmp(argv[1], "subcontext")) {
        android::base::InitLogging(argv, &android::base::KernelLogger);
        const BuiltinFunctionMap function_map;

        return SubcontextMain(argc, argv, &function_map);
    }
```

#### 含义

如果命令行是：

```text
/init subcontext ...
```

则该进程作为 `subcontext` 子进程运行。

#### 逻辑步骤

1. 初始化 logging
2. 构造 `BuiltinFunctionMap`
3. 进入 `SubcontextMain(argc, argv, &function_map)`

#### 作用

`subcontext` 机制用于：

- 在特定 SELinux 上下文下处理某些 init 命令
- 避免所有 rc 行为都直接在主 init 进程里执行

调用关系：

```text
main()
  → if argv[1] == "subcontext"
    → InitLogging(...)
    → SubcontextMain(argc, argv, &function_map)
```

---

### 5.4 参数判断：是否进入 `selinux_setup`

```cpp
if (!strcmp(argv[1], "selinux_setup")) {
    android::mboot::mdb("SELinux Setup ...");
    return SetupSelinux(argv);
}
```

#### 含义

如果命令行是：

```text
/init selinux_setup
```

则进入 SELinux 初始化阶段。

#### `SetupSelinux()` 的职责

这个函数定义在 `selinux.cpp` 中，主要负责：

- 初始化 SELinux 日志
- 加载 SELinux policy
- restorecon `/system/bin/init`
- 最终 `execv("/system/bin/init", {"init", "second_stage"})`

调用关系：

```text
main()
  → if argv[1] == "selinux_setup"
    → SetupSelinux(argv)
```

---

### 5.5 参数判断：是否进入 `second_stage`

```cpp
if (!strcmp(argv[1], "second_stage")) {
    return SecondStageMain(argc, argv);
}
```

#### 含义

如果命令行是：

```text
/init second_stage
```

则进入 init 的 second stage。

#### `SecondStageMain()` 的职责

`SecondStageMain()` 定义在 `init.cpp` 中，主要负责：

- 初始化属性系统
- 解析 rc 脚本
- 建立 `ActionManager` / `ServiceList`
- 启动 property service
- 安装 signal fd handler
- 启动 epoll 主循环
- 管理服务生命周期、属性触发器和关机重启流程

调用关系：

```text
main()
  → if argv[1] == "second_stage"
    → SecondStageMain(argc, argv)
```

---

### 5.6 默认分支：进入 first stage

```cpp
return FirstStageMain(argc, argv);
```

如果：

- 程序名不是 `ueventd`
- 参数不是 `subcontext`
- 参数不是 `selinux_setup`
- 参数不是 `second_stage`

那么默认进入：

- `FirstStageMain(argc, argv)`

#### `FirstStageMain()` 的职责

定义在 `first_stage_init.cpp` 中，主要做：

- 挂载 `/dev` `/proc` `/sys`
- 创建设备节点
- 搭建最小运行环境
- 执行 first-stage mount
- 最终 `execv("/system/bin/init", {"init", "selinux_setup"})`

调用关系：

```text
main()
  → default
    → FirstStageMain(argc, argv)
```

---

## 6. 前向调用关系：谁会进入 `main.cpp::main()`

`main.cpp::main()` 会在多种场景下被进入。

### 6.1 内核直接启动 `/init`

最标准场景：

```text
kernel
  → /init
    → main()
      → FirstStageMain()
```

这是 Android 用户空间启动链的起点。

---

### 6.2 `FirstStageMain()` 通过 `execv()` 重新进入 `main()`

在 `first_stage_init.cpp` 中，first stage 完成后会执行：

```text
FirstStageMain()
  → execv("/system/bin/init", "selinux_setup")
    → main()
      → SetupSelinux()
```

说明：

同一个可执行文件会以新的参数重新进入 `main()`，从而进入下一阶段。

---

### 6.3 `SetupSelinux()` 通过 `execv()` 再次进入 `main()`

在 `selinux.cpp` 中：

```text
SetupSelinux()
  → execv("/system/bin/init", "second_stage")
    → main()
      → SecondStageMain()
```

说明：

`main()` 在 Android 启动过程中并不只执行一次，而是随着阶段切换被多次重新进入。

---

### 6.4 主 init 拉起 `subcontext`

当主 init 需要在子上下文中执行某些逻辑时，会拉起：

```text
/init subcontext ...
```

于是流程为：

```text
主 init
  → 启动 /init subcontext
    → main()
      → SubcontextMain()
```

---

### 6.5 以 `ueventd` 身份运行

如果该二进制以 `ueventd` 名字运行，则：

```text
ueventd
  → main()
    → ueventd_main()
```

---

## 7. 后向调用关系：`main.cpp` 会把流程交给谁

从 `main()` 往后，可能进入以下几个主要分支：

### 7.1 `ueventd_main(argc, argv)`

用途：

- 处理内核 uevent
- 管理设备节点相关逻辑

### 7.2 `SubcontextMain(argc, argv, &function_map)`

用途：

- 在特定 SELinux 子上下文中执行命令
- 协助主 init 完成一部分受限操作

### 7.3 `SetupSelinux(argv)`

用途：

- 初始化 SELinux 环境
- 为进入 second stage 做准备
- 最终切换到 `SecondStageMain()`

### 7.4 `SecondStageMain(argc, argv)`

用途：

- 承担 init second stage 主逻辑
- 进入 rc/action/service/property 的核心调度循环

### 7.5 `FirstStageMain(argc, argv)`

用途：

- 完成最小文件系统与 early mount 初始化
- 启动后续 `selinux_setup` 阶段

---

## 8. 整个启动链中的位置

如果只看 Android init 的主路径，`main.cpp` 所处位置如下：

```text
kernel
  ↓
/init
  ↓
main.cpp::main()
  ↓
FirstStageMain()
  ↓ exec("/system/bin/init", "selinux_setup")
main.cpp::main()
  ↓
SetupSelinux()
  ↓ exec("/system/bin/init", "second_stage")
main.cpp::main()
  ↓
SecondStageMain()
```

所以 `main.cpp` 本身并不负责实际的 stage 业务，而是负责把 `/init` 切换到正确阶段。

---

## 9. 与 `init.cpp` 的关系

`main.cpp` 和 `init.cpp` 的关系可以概括为：

- `main.cpp` 负责判断“当前该去哪条路径”
- `init.cpp` 负责执行 second stage 的具体逻辑

也就是说：

```text
main.cpp::main()
  → SecondStageMain()
      （SecondStageMain 定义在 init.cpp）
```

可以简单理解为：

> `main.cpp` 决定“去哪”，`init.cpp` 决定“到那以后怎么运行”。

---

## 10. 设计意义

### 10.1 职责分离

这种设计把各阶段拆开：

- `main.cpp`：统一入口分发
- `first_stage_init.cpp`：first stage
- `selinux.cpp`：SELinux setup
- `init.cpp`：second stage
- `ueventd.cpp`：ueventd 主逻辑

这样职责非常清晰。

### 10.2 复用同一二进制

通过：

- 程序名 `argv[0]`
- 启动参数 `argv[1]`

同一个二进制可以承担多个角色，而不必维护多个独立可执行文件。

### 10.3 便于多阶段 `execv()` 切换

Android init 的阶段切换不是简单函数调用，而是 `execv()` 重新执行自身。

这样做的好处：

- 状态更加干净
- 阶段边界更清晰
- 更适合 SELinux policy 加载后的上下文切换

---

## 11. 总结

一句话总结：

> `main.cpp` 是 Android `init` 的统一入口分发器，它根据程序名和参数，把同一个 `/init` 二进制分发到 `ueventd`、`subcontext`、`first stage`、`SELinux setup` 和 `second stage` 等不同执行路径。

如果按前后关系来理解：

- **前面**：它承接 kernel 启动 `/init`，以及不同阶段通过 `execv()` 对自身的重新进入
- **后面**：它把流程交给真正的执行主体，如 `FirstStageMain()`、`SetupSelinux()`、`SecondStageMain()` 等

---

## 12. 总调用关系图

### 12.1 前向来源

```text
kernel
  → /init
    → main()
      → FirstStageMain()

FirstStageMain()
  → execv("/system/bin/init", "selinux_setup")
    → main()
      → SetupSelinux()

SetupSelinux()
  → execv("/system/bin/init", "second_stage")
    → main()
      → SecondStageMain()

主 init
  → 启动 /init subcontext ...
    → main()
      → SubcontextMain()

系统以 ueventd 名字运行该二进制
  → ueventd
    → main()
      → ueventd_main()
```

### 12.2 后向分发

```text
main()
 ├─ ueventd_main(argc, argv)
 ├─ SubcontextMain(argc, argv, &function_map)
 ├─ SetupSelinux(argv)
 ├─ SecondStageMain(argc, argv)
 └─ FirstStageMain(argc, argv)
```

