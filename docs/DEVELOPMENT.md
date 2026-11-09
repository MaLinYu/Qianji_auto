# 本分支开发说明

本轮开发先恢复“支付宝/微信等账单来源 → 自动记账插件 → 钱迹”的链路。
钱迹是写入目标，支付宝、微信是交易信息来源，两者的适配分别维护。
现有一木记账适配暂时保留，尚未验证当前版本支付宝、微信、钱迹的兼容性。

## 手机连接与日志

电脑安装 Android Studio 和 Android SDK Platform-Tools，将 `platform-tools` 加入 PATH。
手机开启开发者选项和 USB 调试，连接 USB 后，在手机上允许这台电脑调试。
以下示例在 Windows PowerShell 中执行；连接多台设备时给 adb 增加 `-s 设备序列号`。

```powershell
adb devices -l
```

设备应显示为 `device`。`unauthorized` 表示需要在手机上确认授权。

### 查看完整链路

先打开自动记账 App，然后执行下面的命令，再在手机上打开支付宝支付页或账单详情复现。

```powershell
adb logcat -v threadtime | Select-String -Pattern '自动记账|记账流程|Ankio-|Xposed|AndroidRuntime'
```

这些日志横跨自动记账和被 Hook 的支付宝、钱迹等进程。
排查整个流程时不要只按自动记账 App 的 PID 过滤，否则会漏掉 Hook 端日志。
需要保留未过滤的现场时，在单独的 PowerShell 窗口运行，复现后按 Ctrl+C：

```powershell
adb logcat -v threadtime | Tee-Object -FilePath .\phone-logcat.txt
```

单独查看崩溃缓冲区：

```powershell
adb logcat -b crash -d -v threadtime
```

只调试插件自身界面或规则解析时，可以使用进程过滤：

```powershell
$appPid = (adb shell pidof -s cn.dreamn.qianji_auto).Trim()
adb logcat --pid=$appPid -v threadtime
```

执行前需启动插件并确认 `$appPid` 非空。进程重启后 PID 会改变，需要重新获取。
日志可能包含真实金额、商户和账号信息，分享前先脱敏。

### 项目内置日志与调试开关

- 插件的数据页面提供“日志”入口，可切换日志模式、刷新、导出完整日志。
- 当前的隐藏调试开关位于“关于”页面：在顶部图片上 1 秒内连续点击 5 次。
- 开启后首页显示调试标记，点击该标记可关闭。
- APK 的 `debug` 构建类型与这个运行时开关是两件事；安装 Debug APK 不会自动打开此开关。
- Xposed 模式还应查看所用框架管理器的模块日志。`Utils.log()` 会写入 `XposedBridge.log()`，并广播回插件的日志列表。
- 修改 Hook 代码后，更新插件，并重新启动被 Hook 的目标 App；具体模块加载方式由所用框架决定。

### 断点调试

用 Android Studio 打开本仓库，选择 `app` 的 `debug` variant，将 APK 安装到手机后使用 Debug，
或使用“Attach debugger to Android process”附加到 `cn.dreamn.qianji_auto`。
优先在 `XposedBroadcast.onReceive()`、`RegularCenter.run()`、`SendDataToApp.call()`、
`QianJi.sendToApp()` 设置断点，依次检查数据接收、字段解析和钱迹调用。

支付宝等正式版 App 通常不允许普通方式直接附加调试器；Hook 端先通过日志定位。
安装旧版或其他渠道 APK 时如果提示 `INSTALL_FAILED_UPDATE_INCOMPATIBLE`，表示签名不同；
先备份规则与账单，再处理旧安装。临时 CI Debug 签名也可能随构建环境变化，不用于正式发布。

## 构建与 GitHub Actions

当前仓库使用 Gradle 7.2、Android Gradle Plugin 4.1.3 和 Kotlin 1.4.31。
CI 为旧构建链使用 JDK 8；Android SDK 命令行工具另用 JDK 17。
本地 Android Studio 的 Gradle JDK 也应选择 JDK 8，不直接使用新版 IDE 自带的 JDK 21/25。
Android SDK 需要 Platform 30、31 和 Build-Tools 30.0.3。

在 SDK Manager 安装这些组件，并通过 IDE 或被 gitignore 排除的 `local.properties` 配置 SDK 路径。
Windows 路径示例：

```properties
sdk.dir=C:/Users/你的用户名/AppData/Local/Android/Sdk
```

```powershell
.\gradlew.bat :app:assembleDebug --stacktrace
Get-ChildItem .\app\build\outputs\apk\debug\*.apk
```

此项目的旧 AGP 在 Windows 上可能拒绝含中文的工程目录。若出现对应报错，
优先使用不含中文的工作目录；也可在本机用户级 Gradle 配置中设置 `android.overridePathCheck=true`，
该选项仅跳过路径检查，并不保证所有旧工具都支持中文路径。

`.github/workflows/android-ci.yml` 会在代码 **push 到 GitHub**、PR 更新时自动编译，
也支持在 Actions 页面手动运行。本地 `git commit` 不联网，不会触发 GitHub Actions。
同一个分支出现更新的提交时，取消较旧的正在执行的任务。

编译成功后，在该次运行的 Artifacts 中下载 `qianji-auto-debug-提交SHA`，解压得到 APK；
构建日志保存为 `build-log-提交SHA`。产物保留 14 天。
Debug APK 使用开发签名，不需要配置正式签名密钥；不会自动发布 Release 或上传酷安。
fork 仓库首次运行可能需要仓库所有者先在 Actions 页面启用工作流。

## 现有适配范围

| 层次 | 软件/入口 | 代码现状 |
| --- | --- | --- |
| 记账目标 | 钱迹 | `app/QianJi.java`，通过自动记账 URL Scheme 写入；另有钱迹进程 Hook 提供同步、报销等扩展 |
| 记账目标 | 一木记账 | `app/Wangc.java` 及对应 Hook |
| Xposed 采集 | 支付宝、微信 | 有采集代码和内置账单规则，是优先恢复的来源 |
| Xposed 采集 | QQ、山大 v 卡通、短信、系统通知 | 有注册入口；不能仅凭入口代码判断当前版本已可用 |
| 无障碍页面识别 | 支付宝、微信、云闪付、美团/美团外卖、京东/京东金融、拼多多 | 有页面判断；内置 `helper_regular.json` 当前仅包含支付宝和微信规则 |

监控包名、页面识别、规则匹配是三个条件。某个应用出现在监控配置中，
不代表已经具备完整、可用的账单适配。短信和通知也需要相应的规则。

钱迹链路的主要代码位置：

1. `core/hook/hooks/alipay/` 或 `core/helper/inner/`：采集交易信息。
2. `core/broadcast/XposedBroadcast.java`：接收跨进程数据。
3. `data/data/RegularCenter.java`、`assets/regulars/`：解析与分类。
4. `bills/SendDataToApp.java`：补充信息、确认和提交。
5. `app/QianJi.java`：拼接 `qianji://publicapi/addbill` 并调用钱迹。

官方参考：[Logcat](https://developer.android.com/tools/logcat)、
[Android 断点调试](https://developer.android.com/studio/debug)、
[钱迹自动化接口](https://docs.qianjiapp.com/plugin/auto_tasker.html)。
