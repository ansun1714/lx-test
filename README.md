# LX Music 界面复现版（Android + HarmonyOS）

按洛雪音乐（LX Music）移动端 v1.9.2 的界面与交互复刻的客户端：**WebView / ArkWeb 渲染界面 + 原生壳**，音源由用户自行导入的自定义 JS 音源提供，安装包内不内置任何音源与曲库。

- `app/` 是一份共用界面，Android 打成 APK，HarmonyOS 打成 HAP（ArkWeb 加载同一份 `standalone.html`）
- 源码提交后由 GitHub Actions 自动构建两个版本并发布到 Releases

> 仅个人学习与界面研究用；不随包分发音频内容，音源脚本由使用者自行准备，请遵守各音乐平台的授权条款。

## 当前状态

- Android：版本 **3.13.7**（versionCode 24），产物 `LXMusic-v3.13.7.apk`（160,284 B），包名 `com.lxmusic.replica`，minSdk 23 / targetSdk 34
- HarmonyOS：同一版本号，HAP 由 OpenHarmony 5.0.3-Release SDK（API 15）构建，**未签名**
- 模拟器回归全绿：逻辑冒烟 463/463、设备验收 85/85、平台 35/35、播放器 41/41、队列 22/22、点行播放 25/25 + 重启 4/4


## 目录结构

```
app/                 界面与逻辑（纯前端，可直接在浏览器打开 standalone.html）
  index.html         页面骨架
  app.js             视图 / 事件 / 播放调度
  engine.js          音频引擎、Web Audio 接管（音效）
  platform.js        五个平台（酷我/酷狗/咪咕/网易/QQ）的搜索、榜单、歌单适配
  source-api.js      自定义 JS 音源（user-api 2.0.0 兼容层）
  sources.js         音源导入与校验
  download.js        下载队列、落盘、本地播放
  sound-effects.js   9 个预设 + 10 段 EQ + 混响/环绕
  quality.js         音质档位与降级链
  styles.css         对齐原版的样式（含 16 套主题变量）
android/             Java 壳 + 原生桥（EchoBridge）
  src/com/lxmusic/replica/MainActivity.java
  build.py          aapt2/d8/apksigner 一把构建，产出根目录 APK
harmony/             HarmonyOS 壳（ArkTS + ArkWeb，hvigor 工程）
  AppScope/app.json5            包名 / 版本号
  entry/src/main/ets/pages/Index.ets   Web 容器，加载 rawfile/index.html
  entry/src/main/module.json5     权限（INTERNET / GET_NETWORK_INFO）
docs/               构建说明、界面还原规格
evidence/           验收脚本（smoke / device-check / device-platform / device-player /
                    device-queue / device-dlplay / device-settings）与报告 JSON
                    fixtures 内是自造测试音源（返回本地 WAV，不联网）
```

## 构建

```powershell
node evidence/build-standalone.js          # app/*.js → app/standalone.html（供 WebView 加载）
node evidence/smoke-test.js                # 纯 Node 逻辑回归，无需设备
python android/build.py                    # 需要 JDK 21 在 PATH 内
```

APK 输出到仓库根目录，版本号在 `android/build.py` 的 `VERSION_NAME / VERSION_CODE`
与 `android/AndroidManifest.xml` 两处同步（CI 会校验一致性）。

构建脚本不依赖 Gradle，只用 SDK 里的 `aapt2 / javac / d8 / zipalign / apksigner`，
以下环境变量可覆盖默认行为：

| 变量 | 默认 | 用途 |
|---|---|---|
| `ANDROID_SDK_ROOT` / `ANDROID_HOME` | `C:\Users\maoxu\tools\android-sdk` | Android SDK 根目录 |
| `ANDROID_BUILD_TOOLS_VERSION` | `35.0.0` | build-tools 版本（34.0.0 的 R8 在 JDK 21 下会 NPE） |
| `LX_BUILD_WORK` | `%TEMP%\lx-build` | 中间产物目录（必须是纯 ASCII 路径） |
| `LX_KEYSTORE_DIR` | `%TEMP%\lx-replica-keystore` | 签名密钥目录 |
| `LX_KS_PASS` / `LX_KS_ALIAS` | `lxmusic-replica` / `lxreplica` | 密钥口令与别名 |
| `LX_VERSION_NAME` / `LX_VERSION_CODE` | 脚本内常量 | 临时覆盖版本号 |
| `LX_FINAL_APK` | 仓库根 `LXMusic-v<版本>.apk` | 产物输出路径 |

## 自动构建（GitHub Actions）

两个工作流并行跑，产物各自发布：

| 工作流 | 产物 | Release |
|---|---|---|
| `build-apk.yml` | `LXMusic-v<版本>.apk`（已签名） | `v<版本>`（覆盖同名资产，标记 latest） |
| `build-hap.yml` | `entry-default-unsigned.hap`（未签名） | `hap-latest`（固定地址，覆盖同名资产） |

两者都在 `ubuntu-latest` 上跑，`main` push / PR / 手动 dispatch 触发；PR 只校验不发布。
失败时会把日志尾部提交回当前分支的 `ci-logs/last-failure.log`（私有仓库的 Actions 日志外部读不到）。

### APK 流水线

```
setup JDK21 + 自装 cmdline-tools（platform 34 / build-tools 35.0.0）
→ build-standalone.js → smoke-test.js（463 项，不过直接失败）
→ python3 android/build.py（签名 + apksigner verify + aapt badging 校验）
→ 上传 artifact「LXMusic-apk」→ 非 PR 事件时发布到 Release
```

产物信息（文件名、字节数、SHA-256）写在每次运行的 Step Summary 里。artifact 保留 14 天。

**Release 发布规则**：`main` 上每次构建成功都会把 APK 发到 Release `v<版本>`
（版本取自 `LXMusic-v3.13.7.apk` 的名字，即 `v3.13.7`），同一个版本重复构建是
覆盖资产（`gh release upload --clobber`）并标记为 latest，不会堆出多个 Release。
发新版时先把 `android/build.py` 和 `AndroidManifest.xml` 的版本号改掉，再 push（或打 tag）。

### HAP 流水线

```
下载 OpenHarmony 5.0.3-Release 公共 SDK（repo.huaweicloud.com，约 4.3 GB，走 cache）
→ 从 v2raym-hos 的 Release 拉 hvigor 5.19.8 两个自包含 tarball（华为源在境外不稳）
→ build-standalone.js → 拷 app/standalone.html 到 harmony/entry/src/main/resources/rawfile/index.html
→ hvigor assembleHap（module=entry@default, product=default）
→ 上传 artifact「LXMusic-hap-api15」→ 非 PR 事件时发布到 Release `hap-latest`
```

本地复现 HAP 构建（需自行准备 OpenHarmony SDK 与 hvigor）：

```powershell
node evidence/build-standalone.js
Copy-Item app\standalone.html harmony\entry\src\main\resources\rawfile\index.html -Force
$env:OHOS_SDK_HOME = "D:\ohos-sdk"          # 内含 15\{ets,js,native,previewer,toolchains}
Push-Location harmony
node node_modules\@ohos\hvigor\bin\hvigor.js --mode module `
  -p module=entry@default -p product=default assembleHap --no-daemon
Pop-Location
# 产物：harmony\entry\build\default\outputs\default\entry-default-unsigned.hap
```

**HAP 现状与限制**：

- CI 只出**未签名** HAP，装机前要自己签（和 v2raym-hos 一样，签名证书涉及华为账号，不进 CI）。
- `EchoBridge` 已在 ArkTS 侧落地（`harmony/entry/src/main/ets/core/EchoBridge.ets`），
  与 Android 的 `MainActivity` 同名同签名，页面侧 `app/` 的代码一行都不改：
  `httpRequest / cancelRequest`（分块 Base64 回传、24MB/8MB 上限）、
  `downloadStart / downloadCancel / downloadClear / downloadPartialSize`（流式落盘 + Range 续传）、
  `saveDownloadStart|Chunk|Finish / saveText`、`pickScript / pickScriptPull / pickScriptAck`、
  `localMediaUrl`、`toast / exitApp`；回调仍是 `LXHttp` / `LXDownloadBridge` / `LXFilePick` 三个对象。
- 与 APK 的两处语义差异（鸿蒙没有等价物）：
  1. 下载落盘到应用沙箱 `filesDir/downloads`，不是公共 `Download/LX Music`；导出到文件管理器待补。
  2. `mediaProxyBase()` 返回空串（不启用应用内 HTTPS 媒体代理），音效改走 `httpRequest` 取字节 → Blob → Web Audio。
- 待真机验证：ArkWeb 对 `file://` 音频的加载与 Range 支持、Web Audio 音效链路、
  `runJavaScript` 注入 180KB 分块的频率开销（不行就换 MessagePort）。
- 已构建成功样本：Release `hap-latest`，`entry-default-unsigned.hap` 545,631 B。

**HAP 本地签名**（装真机前必须做）：用 DevEco Studio 打开 `harmony/`，在
Project Structure → Signing Configs 勾选 Automatic（登录华为账号）生成调试证书；
或手工在 `harmony/build-profile.json5` 的 `signingConfigs` 里填 `.p12 / .cer / .p7b` 三件套，
再跑一次 `assembleHap` 产出已签名包。


**签名说明**：CI 每次跑都是全新容器，默认会在临时目录新生成一个 keystore，
因此 CI 出的包签名与本地 `%TEMP%\lx-replica-keystore` 的包不同，直接覆盖安装会失败
（需先卸载旧包）。要固定签名，把本地 keystore 转 base64 存成仓库 Secret `LX_KEYSTORE_B64`：

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("$env:TEMP\lx-replica-keystore\lx-replica.jks")) |
  Set-Clipboard
```

然后到仓库 Settings → Secrets and variables → Actions 新建 `LX_KEYSTORE_B64` 粘贴即可。

## 设备验收

```powershell
adb -s <serial> install -r LXMusic-v3.13.7.apk
pwsh C:\temp\run-one313.ps1 device-check.js   # 需先把 WebView devtools 转发到 127.0.0.1:9222
```

脚本通过 CDP 在真实 WebView 里模拟点击并断言 DOM 与状态，报告写到 `evidence/*.json`。

## 关键技术点

- **WebView + 原生桥**：`EchoBridge` 提供 HTTP（分块流式 + 24MB 二进制上限）、文件落盘、文件选择、通知栏等能力，避免 `fetch` 直连的跨域与证书问题。
- **音效接管**：原生播放时音效不介入；一旦开启音效或 EQ 偏离平直，才把音频取成本地 Blob 换 `<audio>` 元素接进 Web Audio（同源 + Blob 规避 CORS 污染）。
- **大文件保护**：Java 侧边读边 Base64 分块回传（块长取 3 的倍数），JS 侧按 512KB 分片 `atob` 组 Blob，全程不出现 `byte[] + 大字符串` 的 OOM（此前真机闪退即此因）。
- **目录页**：歌单与榜单都是浮层目录，浮层底边上移播放条高度，滚动列表时底部播放条常驻。
- **菜单定位**：`menu()` 先 `visibility:hidden` 参与布局量真实尺寸，再按视口三边夹住，贴底自动向上翻。

## 版本记录（节选）

| 版本 | 内容 |
|---|---|
| 3.13.2 | 原生桥取字节 + Blob 接管音频图，解决搜索曲目开音效无声/报错 |
| 3.13.3 | 修真机「点歌闪退」：OOM（整份 Base64）→ 分块流式 + 24MB 上限 |
| 3.13.4 | 榜单卡片进入目录页；歌单/榜单目录播放条常驻；修「加载中点播放全部无响应」 |
| 3.13.5 | 删除搜索页右上角「歌曲/歌单」死标签（只切高亮，不影响搜索） |
| 3.13.6 | 播放详情页「收藏」→ 选列表加入，重复不添加 |
| 3.13.7 | 修目录最后一行「⋮」菜单被屏幕底边裁掉 |

## 已知限制

- 音源需自行导入，APK 不内置、仓库也不附带第三方音源脚本。
- 二进制响应上限 24MB，超出直接提示「文件过大，已跳过音效」（无损整曲常见 30–50MB）。
- 真机（vivo V2011A）闪退修复目前只有模拟器证据与 30MB 大文件回归，仍待真机复测。
- 时间轴歌词依赖音源实现 `lyric` 动作，未实现时显示「暂无歌词」。
