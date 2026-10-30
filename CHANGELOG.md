# Changelog

## 1.2.9 — 2026-10-31

桌面 GUI 可用性修复 + 项目自查修正（含打包器加固）。

### 桌面 GUI（6080 noVNC）修复

以下每一项都对应一个在**无显卡 NAS**上实测到的具体故障：

- **无法输入中文（已修）**：Electron 进程缺少 fcitx 所需的
  `GTK_IM_MODULE` / `QT_IM_MODULE` / `XMODIFIERS=@im=fcitx` 与 `zh_CN.UTF-8`
  语言环境。Electron **只在启动时**连接 XIM/DBus，进程里没有这些变量，之后
  无论如何都切不出中文。已随包生成 `start_desktop.sh` 注入上述变量，
  并加入 fcitx 进程保活（挂掉自动拉起）。
- **英文全大写、无法切换（已修）**：VNC 客户端会把本机的 CapsLock 状态
  同步到虚拟屏，并在 X server 上形成 latch（实测 `XKB indicators: Caps Lock: on`）。
  已用 `setxkbmap -option caps:none` 彻底禁用该键，解除锁定且不会被再次同步。
- **Connect 后黑屏（已修）**：无 GPU 环境下 Electron 窗口能映射但不绘制。
  启动参数强制 SwiftShader 软渲染
  （`--enable-unsafe-swiftshader --use-gl=angle --use-angle=swiftshader`）。
- **点登录调不起浏览器（已修）**：裸 Xvfb 缺 `xdg-open`，
  Electron 的 `shell.openExternal()` 静默失败。已安装 `xdg-utils`
  并注册 http/https 默认处理器，同时注入 `BROWSER`。
- **点 X 关闭后只剩黑屏（已修）**：新增 `while true` 自愈守护，
  Electron 退出后 3 秒自动重启，虚拟屏上始终有内容。

> 依赖顺序固定为 **Xvfb → fcitx → Electron**：Electron 启动时才连输入法，
> 若 fcitx 尚未就绪，之后再启动 fcitx 也不会重连。

### 项目自查修正

- **桌面入口写入时机错误（严重）**：`_write_desktop_entry` 原先在服务启动
  **之前**执行，但网关密码是 `start.sh` 首次运行时才写入 `ACCESS_PASS`，
  因此读到的永远是空密码 → iframe 请求不带凭证 → **打开即黑屏**；
  而「卸载重装 / 升级一次」后又正常，正因为第二次运行时密码文件已存在。
  现将写入移到服务启动并确认监听之后。
- **`.gateway_pass` 幽灵引用**：收尾输出读的是这个从不存在的旧版文件名，
  导致永远打印「见 .gateway_pass」，用户会去找一个不存在的文件。
  改为读真实的 `ACCESS_PASS`，并在未启用密码认证时明确说明。
- **桌面入口 JSON 未转义密码**：用户自定义密码若含 `&` `#` 空格等字符会
  破坏 URL 语义，表现为「密码明明对了却打不开」。新增 URL 转义。
- **`cmd/main` 的 `stop_process` 缺兜底**：只调 `ss`，与 `port_listening()`
  的三重兜底不一致；在 ss 返回空的机器上停不掉进程，残留进程持续占用
  8090，下次 start 必然冲突。改用 `port_owner()` 并补 `lsof` 兜底。
- **`cmd/main` 启动失败仍 `exit 0`**：fnOS 无法感知失败，界面一直转圈。
  现正确回传非零退出码。
- **`common.sh` 的 `start_service` 缺 `setsid`**：与已修复的 SIGHUP 问题同源。
- **`install_init` 重复查询更新接口**：正常路径上多一次网络往返，且两次
  查询之间官方若发版，会导致记录版本与实际下载的包不一致、升级判断错乱。
- **`build.sh` 加固**：打包前统一清理 `cmd/` 下脚本的 CRLF（换机打包易踩）；
  新增 `APP_VERSION` / `manifest version` / `install_init INSTALLER_VERSION`
  三处版本号一致性硬校验，不一致直接中止而非仅告警。

## 1.2.8 — 2026-10-31

文档更正（无功能 / 打包变更）：

- **纠正 README 关于 8090 与远程桌面的错误说明（此前 1.2.7 写反了）**：实际是「fnOS 应用中心『打开』按钮当前指向 8090 网关占位页（无登录 / 聊天入口）」；而**带登录 / 扫码聊天的桌面 GUI 是另一套**——Electron 版 WorkBuddy 跑在 NAS 虚拟显示（Xvfb）上，经 VNC 提供，访问方式为浏览器开 `http://<NAS>:6080`（noVNC，点 Connect）或 fnOS 应用中心「远程桌面 / 虚拟机」应用（trim.vm）。8090 网关仅暴露 `/api/v1/*` 与 `/api/docs`，根路径占位页属预期行为。
- 同步 README / README_EN 版本号引用（1.2.7 / 1.2.4 → 1.2.8）。

## 1.2.7 — 2026-10-31

文档更新（无功能 / 打包变更）：

- README 补充「8090 为 headless API 网关」说明：根路径显示「网页界面还没建成」占位页属预期，用于 API / 自动化接入；交互式 WorkBuddy 界面为桌面版（fnOS 远程桌面 / VNC）。
- 同步 README 中过时的版本号引用（1.2.4 → 1.2.7）。

## 1.2.6 — 2026-10-31

修复网关启动崩溃导致应用中心一直转圈（与 1.2.5 的 SIGHUP 修复是**不同根因**）。

### 根因
`start.sh` 用 `codebuddy --serve --host 0.0.0.0 --port 8090` 拉起网关，但未设置
`CODEBUDDY_FORCE_HEADLESS_BUNDLE` / `CODEBUDDY_FORCE_LITE_WB_BUNDLE`。codebuddy 启动器
对 `--serve` 不会自动判定为 headless，默认走 `require('../dist/codebuddy')`；而 WorkBuddy
5.6.x 包只发 `codebuddy-headless.js` / `codebuddy-lite-wb.mjs`、缺完整包，于是网关启动即
退出、8090 永不监听、应用中心持续转圈（run.log 报 `Cannot find module '../dist/codebuddy'`）。

### 1.2.6 改动
- `install_init` 生成的 `start.sh` 在 `exec codebuddy` 前注入 `export CODEBUDDY_FORCE_HEADLESS_BUNDLE=1`，
  强制路由到已存在的 headless bundle，网关正常监听 8090。
- 已在 NAS 实测：加该变量后 `codebuddy --serve` 成功绑定 8090 并输出 HTTP Server 端点，`curl` 返回 200。

> 升级即生效：安装器记录自身版本，升级时若安装器版本变化会自动刷新 `start.sh`，
> 即使 WorkBuddy 本体版本不变。直接升级即可，无需卸载。

## 1.2.4 — 2026-10-10

移除 1.2.3 误注入的 `CODEBUDDY_BASE_URL` 逻辑，纠正对登录机制的误解，并澄清「跨窗口历史丢失」的真实根因与解法。

### 关键澄清（1.2.3 的改法为什么不对）
- WorkBuddy 网关的网页登录用的是**腾讯 SSO 扫码**：`POST /api/v1/auth/account/login` 返回 `authUrl = https://copilot.tencent.com/login?platform=workbuddy&state=...&version=...`。
- 登录完成依赖**网关主动向外轮询腾讯 SSO**（`tencent.sso.copilot.tencent.com`）来确认该 `state` 已授权。**整个代码里不存在 `/api/v1/auth/callback` 这类需要公网回跳的入口**，因此：
  - 登录**不需要公网 IP / 隧道 / 反向代理**；
  - 登录**不依赖 `localUrl`**（即 `getLocalNetworkIP()` 选到 `172.17.0.1` / `10.222.222.1` 不影响登录）；
  - 登录**不依赖 `CODEBUDDY_BASE_URL`**。
- `CODEBUDDY_BASE_URL` 实际是**模型与 Remote Control 的 API 地址**。若把它设成局域网地址（如 `http://192.168.x.x:8090`），模型 / API 调用会打到网关自身而失效。1.2.3 的注入基于「它是对外回跳地址」的错误假设，已在 1.2.4 移除。

### 跨窗口历史丢失的真正根因
- 网关**未绑定 CodeBuddy 账号**时，每个窗口 / 会话都是隔离的匿名会话，对话历史无法跨窗口保留。
- 只要成功完成一次扫码登录（绑定账号），历史即随账号持久化。因此「历史丢失」与「登录转圈」是同一件事的两面：**让扫码登录成功即可**。

### 1.2.4 实际改动
- 删除 `install_init` 里错误的 `CODEBUDDY_BASE_URL` 注入块，`start.sh` 恢复为干净、默认的配置。
- 增加「安装器版本感知」的 `start.sh` 刷新机制：安装器把自身版本写入 `$APP_DATA/.installer_version`；
  当**仅升级了安装器、WorkBuddy 本体版本未变**（此时 fnOS 会跳过下载与解包）时，
  仍会按当前安装器模板重新生成 `start.sh`，确保登录 / 认证修复能落到已安装的 NAS 上。
  此前该场景下 `start.sh` 不会被刷新，是修复无法生效的主要盲区。
- `_write_fallback_scripts` 改为「仅在 `AUTH_MODE` 未设置时才写入 `none`」，
  避免重复刷新时覆盖用户已启用的密码模式。
- 保留 `AUTH_MODE`（默认 `none`；写 `$BASE/AUTH_MODE=password` 可启用密码）等既有逻辑。

### 若重装后仍「扫码转圈」——极可能是 NAS 出网可达性问题
登录能否完成，取决于 **NAS 上网关进程能否访问外网** `copilot.tencent.com` 与 `tencent.sso.copilot.tencent.com`。可在 NAS 终端执行自测：
```
curl -sS -m 8 -o /dev/null -w 'copilot:%{http_code}\n' https://copilot.tencent.com/login
curl -sS -m 8 -o /dev/null -w 'sso:%{http_code}\n' https://tencent.sso.copilot.tencent.com
```
返回非 2xx/3xx 即说明 NAS 出网被限制（DNS / 代理 / 防火墙），需放行上述域名后再试。OEC 实测可正常绑定（`/api/v1/auth/account/status` 返回 `authenticated:true, userName:"Ming"`），证明该机制在能出网的 NAS 上可用。

> 部署说明：本版修复**随 fpk 重新安装 / 升级即可生效**。安装器会记录自身版本，升级时若发现安装器版本变化，
> 即使 WorkBuddy 本体版本未变也会自动刷新 `start.sh`；因此**不必卸载旧版，直接升级即可**。
> （若升级后仍有异常，可卸载后重装作为兜底。）此前的在线补丁（直接改 NAS 上的 `start.sh`）会在重装 / 重启后被覆盖，
> 且曾把 OEC 的 `start.sh` 改坏，请勿继续依赖。

## 1.2.3 — 2026-10-10

修复 start.sh 内存检测在打包期被错误展开的 bug。

### 修复
- **根因**：`install_init` 里生成 `start.sh` 的 heredoc 未转义 `$`，打包时 `$_t`/`$_total`/`$2` 等被提前展开成空，
  生成的脚本里总内存为空，启动报 `[: : integer expression expected`，且 V8 堆上限恒为 1536MB。
  这意味着 1.2.2 宣传的"OEC 2GB 机型按 768MB 限制堆"从未真正生效——所有机器都跑 1536MB。
- **方案**：内存函数改为多来源探测（`/proc/meminfo` → `free -m` → `vmstat -s`），并用 `case` 强制正整数兜底（非数字一律回退 1536）。
- **效果**：OEC 2GB 机型实测 `机器内存 1953MB，V8 堆上限 768MB`；大内存机型按 1024/1536 自适应，无 integer 报错。
- deb 落存储池而非 `/tmp`；空间检查跟随实际安装卷。

## 1.2.2 — 2026-10-08

### 新增
- start.sh 按机器实际内存自适应 V8 堆上限（<3GB→768MB，<12GB→1024MB，否则 1536MB），缓解小内存机型 OOM。
- 安装日志输出内存与堆上限信息，便于排查。

> 注意：1.2.2 的上述自适应逻辑存在打包期变量展开 bug，实际未生效，已在 1.2.3 修复。

## 1.0.0 — 2026-10-06

首个版本。

### 新增
- 纯安装器 fpk，包体约 2MB，不含 WorkBuddy 任何二进制
- `uname -m` 架构自适应，支持 x86_64 与 aarch64
- 双安装源：默认走国内官方镜像，失败自动回退到 cdn；
  亦可 `--url` 完全自定义
- 本地 deb 安装（`--deb`），0 流量，适合大带宽敏感用户
- `dpkg -x` 解包至独立目录，不注册进 dpkg 数据库，
  避免与官方 deb 的安装/升级/卸载互相破坏
- 幂等：重复执行不报错，已是目标版本自动跳过（`--force` 可强制重装）
- 升级场景自动停服务，安装后重新拉起
- 安装完默认删除 deb（`--keep-deb` 可保留），省367MB
- 访问密码由用户在 WorkBuddy 首次打开时自行设置，
  安装器不预置、不硬编码任何密码
- `status` 子命令：查看安装版本、占用空间、端口状态、HTTP 探测
- `uninstall`：默认保留数据；`--purge` 才删除，且需交互确认
- 自绘中性图标（64/256 两档），不使用官方商标图形
- 安装前磁盘空间预检（需约 2GB）

### 安全
- 不接管官方程序：解包目录独立，不写入 dpkg 数据库
- `uninstall --purge` 为不可逆操作，强制二次确认
- 安装器不写死任何访问密码，避免"所有人共用一个默认密码"

### 已知限制
- WorkBuddy 内部密码存储机制尚未实测验证，装机测试时确认
- arm64 架构的 deb 未实测体积