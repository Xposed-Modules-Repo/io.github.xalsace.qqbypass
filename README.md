# QQ Enhanced Bypass

一个 QQ 环境检测绕过模块，适配 QQ 9.3.50 (com.tencent.mobileqq)。

本项目经过多轮测试，已没有太大问题，如过账号还是频繁掉线，可能是账号风控，请尝试QQ会员解决（

## 适配版本

- **目标版本：QQ 9.3.50 (versionCode 15730)**
- 其他版本可能无法正常工作**其他版本请使用dexkit分支**

## 项目特点

### 1. 多层 Hook 架构
通过多层 Hook 机制覆盖 QQ 的各类检测点，包括网络上报、Root 检测、Xposed 检测、设备信息验证等。

### 2. 框架级通用 Hook
Root/命令探测走 Android 框架的固定接口：
- `File.exists()` → 对 su 路径等返回 false
- `ProcessBuilder.start()` → 把 `type su` 等 root 探测重写为无害的 `exit 1`
- `Debug.isDebuggerConnected()` → 返回 false

### 3. Native 层支持
通过 [ByteHook](https://github.com/bytedance/bytehook) PLT hook 处理 native 检测（libc 的 `fopen`/`system` 等）。

### 4. 配置系统
通过 `HookConfig` 类集中管理功能开关、设备信息伪造配置、日志开关。

## 架构与 Hook 层

模块入口 `XposedEntry` 只对目标包 `com.tencent.mobileqq` 生效，并按进程门控。
各层由 `UnifiedHookCoordinator` 协调，Native 层在后台异步安装。

### Layer 1 — 网络上报拦截 (NetworkReportHook)
在 report 入口按命令字符串过滤：
- `MsfCore.sendSsoMsg`、`ChannelProxyExt.sendMessage/sendMessageInner`
- 拦截命令：`trpc.o3.report`、`trpc.o3.mobile_security`、`trpc.ilive_cdn.report`、`OidbSvc.0xd79`
- TuringFD `/proc` 进程读取器拦截
- 返回类型安全：只在 `void`/`int`/`long` 等安全类型上改写返回值，避免 `ClassCastException`

### Layer 2 — Root 检测 (DynamicRootDetectionHook)
- 通用 `File.exists()` hook：对 su 路径 / magisk / lsposed 等返回 false
- 定位引用了 su 路径且调用 `File.exists` 的检测方法

### Layer 3 — Xposed/Hook 框架检测 (DynamicXposedDetectionHook)
定位 ArtMethod 检测、`/proc/self/maps` 读取器、QSec hook 检测，
中和其布尔/整型结果，并过滤 maps 输出中的 xposed/lsposed/magisk/自身库特征。

### Layer 4 — 设备信息一致性 (DynamicDeviceInfoHook)
定位 IMEI/AndroidID/Serial 读取器，监控交叉验证（默认不伪造，`FAKE_*` 为 null）。

### Layer 5 — 调试/模拟器检测 (DynamicDebugDetectionHook)
- 通用 `Debug.isDebuggerConnected()` → false
- 定位 TracerPid 读取器、模拟器特征检测

### Layer 6 — Runtime 监控 (DynamicRuntimeMonitorHook)
- 通用 `ProcessBuilder.start()` hook：root 探测（`type su` 等）重写为 `exit 1`

### Layer 7 — QQ 9.3.50 检测点补全 (QQ950PatchHook)
针对 QQ 9.3.50 版本的特定检测点：
- 踢下线处理器 `NTKickProcessor`
- Turing 缓存写入 `TuringWrapper`
- `TuringRiskService.reqRiskDetectV2`、`TuringIDService.getTuringDID*`、MSF 遥测上报

### Native 层 (NativeBypass / UnifiedHookCoordinator)
通过 ByteHook 安装 libc PLT hook（`fopen`/`system` 等），处理 native 侧的
`/proc` 扫描与命令执行探测。

## 安装与使用

### 前置条件
- Root 设备（Magisk / KernelSU）
- LSPosed 或 EdXposed 框架
- Android 8.0+ (API 26+)
- 目标：QQ 9.3.50 (com.tencent.mobileqq)

### 编译
用 Android Studio 打开项目直接构建，或用本机 Gradle（AGP 8.0.2，需 JDK 17）：
```bash
gradle assembleDebug     # 或 assembleRelease
```
需配置 `local.properties` 指向 Android SDK / NDK。生成的 APK 在
`app/build/outputs/apk/debug/app-debug.apk`。

### 安装
1. 安装生成的 APK
2. 在 LSPosed 管理器中启用模块
3. 勾选作用域：`com.tencent.mobileqq`
4. 重启 QQ

### 配置

编辑 `HookConfig.java` 自定义行为：

```java
// 功能开关
public static boolean ENABLE_ROOT_BYPASS = true;
public static boolean ENABLE_XPOSED_BYPASS = true;
public static boolean ENABLE_DEBUG_BYPASS = true;
public static boolean ENABLE_DEVICE_SPOOF = true;
public static boolean ENABLE_NETWORK_INTERCEPT = true;
public static boolean ENABLE_RUNTIME_BYPASS = true;

// 详细日志
public static boolean VERBOSE_LOGGING = true;

// 设备信息伪造（null = 使用真实值）
public static String FAKE_IMEI = null;
public static String FAKE_ANDROID_ID = null;
public static String FAKE_SERIAL = null;
```

## 局限性

### 1. 版本兼容性
**本模块仅适配 QQ 9.3.50 (versionCode 15730)**，其他版本由于类名、方法名、调用关系不同，无法保证正常工作。QQ 更新后需要重新适配。

### 2. Native 层检测限制
本模块处理了部分 Java 层检测。Native 库（libfekit.so, libturingxq.so）的深度检测可能需要：
- Frida/Dobby 等 native hook 框架
- 内存补丁
- 二进制修改

### 3. 服务端风控决策
客户端绕过只是第一步，腾讯服务端会综合判断：
- 历史行为模式
- 设备指纹变化
- 多维度风险评分
- IP/网络环境

### 4. LSPosed 自身检测
本模块试图隐藏 LSPosed，但 native 层检测（maps 扫描、ArtMethod 校验）仍然有效。最佳实践：
- 使用 Zygisk 模式（相对隐蔽）
- 配合 Shamiko 等隐藏模块
- 考虑使用专门的 native hook 方案

## 技术细节

### Hook 策略
- **框架级接口**（`File.exists`/`ProcessBuilder.start`/`Debug.isDebuggerConnected`）——
  Android 固定 API，版本无关，作为兜底。
- **特征定位**——针对混淆名，按字符串常量/调用/参数类型命中。
- **反射按签名**——用返回类型 + 参数类型在类内定位，避免写死方法名。

### 返回类型安全
改写被 hook 方法的返回值时，只对 `void`/`int`/`long`/`Object` 等类型安全的情况操作。
对返回基本类型的方法误用 `setResult(null)` 会触发 `ClassCastException`，导致整进程崩溃。

## 进阶方案

要实现更完整的绕过，需要组合使用：

### 1. Native Hook 层
使用 Frida/Dobby hook native 函数：
- `fopen/fgets` - 过滤 maps/status 读取
- `dlopen/dlsym` - 隐藏注入库
- libfekit 的检测函数 - 直接返回安全值
- libturingxq 的风控函数 - 阻止上报

### 2. 内核层隐藏
- Magisk Zygisk：进程隔离
- KernelSU：内核级权限管理
- Zygisk-Next: 还原挂载，匿名内存

### 3. 虚拟化方案
- VirtualXposed（已过时）
- 太极/无极（兼容性问题）
- Patch 版 QQ（风险高）

## 免责声明

本项目仅供安全研究和学习使用。使用本模块可能：
- 违反 QQ 服务条款
- 导致账号被封禁
- 触发额外的安全审查

**请在测试环境中使用，风险自负。**

## 致谢与参考

- [1.txt](https://github.com/jhl337/QQNTHookBypass/issues/5) - 项目分析及灵感，后续测试也与YIDYIF一同完成
- [DexKit](https://github.com/LuckyPray/DexKit) - 高性能 dex 反混淆/特征定位库
- [ByteHook](https://github.com/bytedance/bytehook) - Android PLT hook 框架
- [LSPosed](https://github.com/LSPosed/LSPosed) - Xposed 框架

## 许可证

本项目采用 GNU General Public License v3.0 (GPL-3.0) 授权，详见 [LICENSE](LICENSE)。

## 贡献

欢迎提交 Issue 和 Pull Request。

特别需要：
- 新版本 QQ 的类名/方法名更新
- Native hook 实现方案
- 效果测试反馈
