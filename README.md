# 在 Windows 上用 xtool 开发 iOS App（WSL2 + 免费 Apple ID）

> 没有 Mac、不想交 99 美元开发者年费，也能把自己的 App 装到 iPhone 上。
> 这份文档记录了我从零跑通的完整流程和踩过的坑，重点是：**把一个 PWA 包成原生壳，用上 PWA 用不了的 iOS 功能。**

我是在 TikTok 的评论区第一次听说 xtool 的。之前没有参考过别人的项目，下面的流程和坑都是自己一步步试出来的。我原本有一个自己做的 PWA，想用上闹钟、灵动岛这些 PWA 做不到的功能，于是把它包成了原生壳。

> 本文所有构建、签名、安装都基于 **[xtool](https://github.com/xtool-org/xtool)**（MIT 协议），一个跨平台的 Xcode 替代品，可以在 Linux / Windows / macOS 上用 SwiftPM 构建并部署 iOS App。感谢 xtool 的作者和贡献者。本文只是使用经验，不是官方文档，遇到问题请以 xtool 仓库为准。

## 适合谁

- 用 Windows，没有 Mac
- 已经有一个网页 / PWA，想让它在 iPhone 上更像一个真 App
- 只给自己或少数人用，不上架 App Store

不适合：要上架、要给很多人分发、要用远程推送（APNs）的项目。

## 我的环境

| 项目 | 版本 |
|---|---|
| 系统 | Windows + WSL2（Ubuntu 24.04，NAT 网络） |
| Swift | 6.4.2 |
| xtool | 1.24.0 |
| iOS SDK | 从 Xcode 27.1 提取 |
| 手机 | iOS 27.2 |
| 账号 | 免费 Apple ID |

---

## 1. 先看：免费账号能做什么，不能做什么

| | PWA（添加到主屏幕） | 原生壳（免费账号签名） |
|---|---|---|
| 安装 | Safari 一键添加 | 需要电脑 + 数据线 |
| 有效期 | 永久 | **7 天**，到期要用电脑续签 |
| 页面更新 | 刷新即生效 | 壳加载线上网页，前端改了照样即时生效；只有改原生代码才要重新签 |
| 系统闹钟（AlarmKit） | ❌ 只能靠 Web Push，可能晚到、可能被系统吞 | ✅ 系统闹钟界面，准点、持续响 |
| 灵动岛 / 实时活动 | ❌ | ✅ 本地更新即可，不需要推送 |
| 桌面小组件 | ❌ | ✅ |
| 通话音频（扬声器、后台） | 受限 | ✅ 可以自己控制音频会话 |
| 远程推送 | ✅ Web Push（iOS 16.4+） | ❌ 免费账号没有 APNs |


免费账号的其他限制：同一时间最多装 3 个自签 App；每 7 天能注册的 App ID 数量有限（加小组件会多占一个）。

### 没有 APNs 怎么推送？

免费账号拿不到 APNs，但可以借用别人已经上架的推送 App，你的服务器调它们的接口，由它们发通知到手机：

| 工具 | 怎么用 | 特点 |
|---|---|---|
| [Bark](https://github.com/Finb/Bark) | App Store 下载，拿到一个设备 key，服务器发一个 HTTP 请求就推送 | 免费、开源，服务端可以自己部署 |
| [Pushcut](https://www.pushcut.io/) | 通过 webhook 推送，还能触发快捷指令 | 免费档功能有限 |
| [ntfy](https://ntfy.sh/) | 订阅一个 topic，往 topic 发消息就推送 | 开源，可自建服务器 |

配合自定义 URL Scheme（在 `Info.plist` 里注册，免费账号可用），点通知可以直接跳回你的 App 的某个页面。

注意：推送内容会经过第三方的服务器。**敏感内容不要放在推送正文里**，推送只写“有新消息”，点开再进 App 看。

我的使用体验：

- **Bark**：最简单，能直接推送具体内容。缺点是通知上显示的是 Bark 的图标，不是你自己 App 的，看着有点别扭。
- **Pushcut**：通过触发快捷指令发通知，看起来更像原生通知。缺点是只能发固定内容，比如“XX 给你发了一条消息”，看不到具体消息是什么，得点进去看。
- **ntfy**：我没用过。

---

## 2. 安装

### 2.1 Swift 工具链

官方推荐 swiftly，但我遇到 swiftly 1.2.0 装不上（`gpg --import` 退出码 2，原因是密钥文件被 gzip 压缩、swiftly 没解压）。**直接下载 tarball 更稳**，还能断点续传：

```bash
curl -fsSL --compressed https://www.swift.org/keys/all-keys.asc | gpg --import
U=https://download.swift.org/swift-6.4.2-release/ubuntu2404/swift-6.4.2-RELEASE/swift-6.4.2-RELEASE-ubuntu24.04.tar.gz
curl -fL -C - -o swift.tar.gz "$U"     # 断了就重跑，-C - 会续传
curl -fL -o swift.tar.gz.sig "$U.sig"
gpg --verify swift.tar.gz.sig swift.tar.gz   # 看到 Good signature 即可
mkdir -p ~/.local/share/swift/swift-6.4.2-RELEASE
tar xzf swift.tar.gz -C ~/.local/share/swift/swift-6.4.2-RELEASE --strip-components=1
ln -sfn swift-6.4.2-RELEASE ~/.local/share/swift/current
echo 'export PATH="$HOME/.local/share/swift/current/usr/bin:$PATH"' >> ~/.bashrc
```

### 2.2 xtool

从 [xtool 的 GitHub Releases](https://github.com/xtool-org/xtool/releases) 下载 AppImage，放进 PATH（`~/.local/bin` 就行，不需要 root，也不需要 libfuse2）。

### 2.3 iOS SDK

iOS SDK 只能从 Xcode 里提取：

1. 用任意 Apple ID 登录 [Apple Developer 下载页](https://developer.apple.com/download/all/)，下载 `Xcode_xx.x.xip`（约 2 GB）。
2. 运行 `xtool setup`，或 `xtool sdk install <xip 路径>`。不用手动解压。
3. 装完 xip 可以删，以后构建和续签都不需要它。

注意：
- Xcode 版本不用和手机系统完全一致（我用 27.1 的 SDK 给 27.2 的手机装没问题）。
- `xtool auth` 登录的 Apple ID **不能开启“高级数据保护”**。
- 磁盘：工具链约 3.5 GB，SDK 约 7 GB。

---

## 3. 把手机连到 WSL（最容易卡住的一步）

### ❌ 不要用 usbipd

网上很多教程用 usbipd 把手机 attach 进 WSL。我试过：签名、打包都通过，最后卡在 `Connecting`，日志里是 `message was too large (65536 bytes, max = 65535)`。这是 usbipd-win 的已知 bug（usbipd-win#959、xtool#19），重试没用。

### ✅ 让 Windows 管手机，WSL 通过网络连过去

手机插在 **Windows** 上，由 Apple Mobile Device Service 管理（装 iTunes 或 Apple Devices 会自带）。WSL 通过 TCP 27015 端口访问它。

**管理员 PowerShell（只需做一次）：**

```powershell
# 防火墙放行 27015
New-NetFirewallRule -DisplayName "WSL usbmuxd" -Direction Inbound -InterfaceAlias "vEthernet (WSL (Hyper-V firewall))" -Protocol TCP -LocalPort 27015 -Action Allow

# 端口转发。<WSL网关IP> 在 WSL 里用 ip route list default 查
netsh interface portproxy set v4tov4 listenport=27015 listenaddress=<WSL网关IP> connectaddress=127.0.0.1 connectport=27015
```

**WSL（每个新终端都要设）：**

```bash
export USBMUXD_SOCKET_ADDRESS="$(ip route list default | awk '{print $3}'):27015"
xtool devices   # 能列出手机就通了
```

### 手机端

- 解锁手机，停在主屏幕，弹出“信任此电脑”时点信任。
- 打开开发者模式：设置 → 隐私与安全性 → 开发者模式。找不到的话，先跑一次 `xtool dev`，装过开发版 App 后才会出现。
- 第一次打开 App 提示不受信任：设置 → 通用 → VPN 与设备管理，信任你的 Apple ID。

---

## 4. 把 PWA 包成原生壳

### 4.1 新建项目

```bash
xtool new MyApp
cd MyApp
```

项目要放在 **WSL 原生路径**（如 `~/MyApp`），不要放 `/mnt/c` 或 `/mnt/d`。

### 4.2 最小的 WebView 壳

```swift
import SwiftUI
import WebKit

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            WebView(url: URL(string: "https://your-pwa.example.com")!)
                .ignoresSafeArea()
        }
    }
}

struct WebView: UIViewRepresentable {
    let url: URL

    func makeUIView(context: Context) -> WKWebView {
        let config = WKWebViewConfiguration()
        config.allowsInlineMediaPlayback = true
        let webView = WKWebView(frame: .zero, configuration: config)
        webView.load(URLRequest(url: url))
        return webView
    }

    func updateUIView(_ uiView: WKWebView, context: Context) {}
}
```

壳只负责加载你的线上网页，所以**前端改了不用重新签名**，用户重开 App 就是新版本。

### 4.3 构建和安装

```bash
xtool dev          # 构建 + 签名 + 安装到手机 + 启动
xtool dev build    # 只构建，产物在 ./xtool/MyApp.app
```

**只能用 xtool 构建**。直接 `swift build --swift-sdk arm64-apple-ios` 会报 `unable to find platform for 'iphoneos'`。

免费账号签名时，xtool 会给 bundle ID 自动加一个 `XTL-xxxx.` 前缀，这是正常的。

### 4.4 网页和原生怎么通信

网页想用原生功能（比如排一个系统闹钟），需要一座“桥”。思路是双向发消息：

**网页 → 原生**：原生注册一个消息通道，网页往里发 JSON。

```swift
// 原生：注册名为 "bridge" 的通道
final class Bridge: NSObject, WKScriptMessageHandler {
    func userContentController(_ controller: WKUserContentController, didReceive message: WKScriptMessage) {
        // 只接受自己域名的消息，防止别的网页调用原生功能
        guard message.frameInfo.securityOrigin.host == "your-pwa.example.com",
              let body = message.body as? [String: Any],
              let type = body["type"] as? String else { return }
        switch type {
        case "alarm.set": break   // 调 AlarmKit
        default: break
        }
    }
}
// 创建 WebView 时：config.userContentController.add(Bridge(), name: "bridge")
```

```js
// 网页：判断是否在原生壳里，在的话就发消息
const native = window.webkit?.messageHandlers?.bridge;
if (native) native.postMessage({ type: "alarm.set", time: "07:30" });
```

**原生 → 网页**：原生执行一段 JS，派发一个自定义事件，网页监听它。

```swift
let json = #"{"type":"alarm.result","ok":true}"#
webView.evaluateJavaScript("window.dispatchEvent(new CustomEvent('native', { detail: \(json) }))")
```

```js
window.addEventListener("native", (e) => console.log(e.detail));
```

这样同一套网页代码，在 Safari 里照常运行，在原生壳里就能多出原生功能。

---

## 5. 每 7 天续签

免费账号签的 App，7 天后打不开。续签就是重新装一遍：

1. 手机插 Windows，解锁。
2. WSL 里：

```bash
export USBMUXD_SOCKET_ADDRESS="$(ip route list default | awk '{print $3}'):27015"
cd ~/MyApp && xtool dev
```

- **始终用同一个 Apple ID**。换账号会导致 bundle ID 变化，App 里的本地数据不会带过去。
- 覆盖安装会保留 App 里的登录状态，不用重新登录。
- 建议给自己设个每周的提醒。也可以在 App 里读 `embedded.mobileprovision` 的过期时间，到期前弹本地通知提醒自己。

### 能不能自动续签？

可以写成一键脚本，但**完全无人值守不太现实**，原因是：

- 手机得连着 Windows（数据线，或在 Apple Devices 里开启 Wi-Fi 同步）。
- 手机必须是解锁状态，锁屏时装不上。
- Apple ID 的登录会过期，过期后 `xtool auth` 需要你输入密码和双重认证验证码，脚本没法替你点。
- WSL 得开着，网关 IP 变了还要重跑 `netsh`。

所以推荐**半自动**：定时提醒 + 一键脚本。

`~/renew.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail
export USBMUXD_SOCKET_ADDRESS="$(ip route list default | awk '{print $3}'):27015"
echo "手机地址：$USBMUXD_SOCKET_ADDRESS"
xtool devices
cd ~/MyApp && xtool dev
```

```bash
chmod +x ~/renew.sh
```

到时间插上手机、解锁，在 WSL 里跑 `~/renew.sh` 就行。

提醒可以用 Windows 任务计划程序、手机日历，或者在 App 里读 `embedded.mobileprovision` 的过期时间，到期前一天弹本地通知。

用 AI agent 定时跑也行，好处是出错时它能帮你读报错、判断是哪一步出了问题；但它同样过不了“手机要连着、要解锁、要输验证码”这几关，所以本质上还是半自动。

---

## 6. 踩坑速查

| 现象 | 原因和解决 |
|---|---|
| `xtool dev` 卡在 `Connecting` | 用了 usbipd。改用第 3 节的 TCP 方式 |
| `waiting for device to be connected` | 当前终端没设 `USBMUXD_SOCKET_ADDRESS`，或者手机没解锁 |
| `socket_connect: getaddrinfo ... Name or service not known` | 命令是从微信 / 网页复制的，引号变成了中文弯引号。手动重打，或直接写 IP |
| `Invalid HostID (-21)` | Windows 上的配对记录过期。在 WSL 设好地址后跑 `idevicepair pair`；如果新记录卡在 `C:\ProgramData\Apple\Lockdown\<UDID>.plist.tmp`，停掉 Apple 服务后用它替换旧的 `.plist`（先备份）。这些文件含私钥，别发给别人 |
| `Password protected (-17)` | 手机锁屏或没点信任 |
| `usbipd attach` 报 `Device busy` | Apple 服务或爱思助手占着手机。改用 TCP 方式就不会遇到 |
| 某天突然连不上 | WSL 的网关 IP 变了，重新查 `ip route list default`，重跑 `netsh` 那行 |
| 加了小组件后签名报 `Can't find free space of LoadCommands for CodeSignature` | 链接器把小组件代码全裁掉了（xtool#379）。在小组件 target 加 `linkerSettings: [.unsafeFlags(["-Xlinker", "-u", "-Xlinker", "_main"])]` |
| 编译通过，但有 `nearly matches optional requirement` 警告 | 可选协议方法名写错了，回调永远不会被调用。**警告也要看**；看警告前先 `rm -rf .build xtool` 再构建 |
| 用 AlarmKit 的部分 API 报版本错误 | 在 `Package.swift` 里把最低版本写成 `.iOS("26.1")` |

---

## 7. 遇到其他问题

这份文档只覆盖我走过的路。遇到这里没写的问题，去 [xtool 仓库](https://github.com/xtool-org/xtool) 的 README 和 issues 里找，那里的信息最全、最新。

## 参考

- [xtool](https://github.com/xtool-org/xtool)
- [xtool#19](https://github.com/xtool-org/xtool/issues/19)、usbipd-win#959：usbipd 连接问题
- [xtool#379](https://github.com/xtool-org/xtool/issues/379)：小组件被链接器裁掉
