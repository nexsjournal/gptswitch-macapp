# DSH Desktop 补充调研

日期：2026-09-18 · 范围：桌面壳、网页承载、平台适配与生命周期；只读源码研究，未安装依赖、运行应用或测试。

关联：[桌面壳选型复审](03-desktop-shell-decision.md) · [总体架构](../architecture/01-system-architecture.md)

## 1. 直接回答

**DSH Desktop 确实使用 Electron。它内嵌浏览器引擎来显示自己的 Web 界面；在本次检查的桌面核心路径中，没有看到供用户浏览任意网站的多标签浏览器。** 两句话可以同时成立。

对用户的区别是：

- “在应用窗口里填写供应商、模型、Key”属于本工具的本地界面，Tauri 与 Electron 都能实现。
- “在应用窗口里直接打开供应商网站、登录后台、切换多个网站标签”属于额外的浏览器产品能力。
- “点击供应商官网后跳到 Safari / Edge / Chrome”只是外部链接，不需要在应用里建设浏览器。

DSH 的主路径属于第一种：本地 Host 提供 Web UI，Electron 显示它。其主窗口拦截跨源导航；新开链接经允许的协议交给系统外部处理，拒绝在应用内自动创建新网页窗口。[窗口实现][dsh-shell]

因此，这个参考补充了 Electron 桌面工程的做法，但没有为 GPTSwitch 新增“必须内嵌网站”的需求。**按当前配置工具范围，仍保留 Tauri + Rust；参考 DSH 的窗口生命周期、平台适配与恢复设计。** Electron 是可选的桌面壳，并不要求产品先拥有浏览器功能；保留 Tauri 是当前范围下的取舍，不是认定 Electron 不适合配置工具。

## 2. 快照与读取范围

| 项目 | 本次记录 |
| --- | --- |
| 仓库 | [anywhere-labs/dsh-desktop](https://github.com/anywhere-labs/dsh-desktop) |
| 固定 commit | `cb736b6edfdd1e0584dea2252c23c7b4bf11420c` |
| commit 时间 | 2026-09-17 17:54:05 +08:00 |
| 稳定桌面包版本 | `dsh-plugin-desktop` 2.0.11 |
| 壳及打包依赖 | Electron 43.3.0；electron-builder 26.15.7，为该快照版本，不是本项目锁定版本 |
| 授权文件 | 仓库 LICENSE 为 MIT；依赖及上游组件仍需分别检查 |
| 规模 | 1,703 个 tracked entries；深度固定的稀疏检出 |
| 已阅读重点 | 架构文档、package.json、窗口创建、WebContentsView 组合、导航/外链、preload、动作派发、平台策略、恢复状态机、代表性窗口测试 |
| 未覆盖 | 所有第三方插件、未拉取的上游子模块实现、完整安全审计、运行性能和真实双平台交互 |

版本与依赖见 [package.json][dsh-package]；授权见 [LICENSE][dsh-license]。本次没有复制其实现代码到应用工程。

## 3. 源码到底怎样使用浏览器能力

### 本地应用承载

架构文档将其描述为 Electron 宿主与 DSH Host/Web carrier 的组合；业务页面与 Host 通过 HTTP/WebSocket 交互。这是复用已有 Web 应用的路线。[架构说明][dsh-architecture]

`ElectronShellGeneration` 创建窗口，在 renderer 会话完成本地认证后加载 `spec.url`，并统一管理托盘、窗口事件和释放。它不是一个直接替用户配置 Codex 的代理工具。[窗口实现][dsh-shell]

### WebContentsView 不等于多标签浏览器

`CompatibilityShell` 创建两个 `WebContentsView`：一个显示本地窗口栏，另一个显示 DSH 内容。这里的 `chromeView` 中 chrome 指应用窗口栏，不是在运行独立 Google Chrome。其职责包括两块区域的边界、缩放/恢复后的重绘和销毁。[视图组合][dsh-views]

主窗口的 `will-frame-navigate` / `will-redirect` 限制主 frame 的 origin，`setWindowOpenHandler` 将允许的新开链接交给 `shell.openExternal` 并返回 deny。这些具体行为支持“本地应用承载”的判断，不能因为源码中出现 WebContentsView 就推导出“完整网站浏览器”。[导航与外链][dsh-shell]

`createDesktopLocalWindow` 对本地辅助窗口显式关闭 `webviewTag`，拒绝附加 webview 和新窗口；相应测试检查这一策略。这个限制的范围是辅助窗口，不能扩大成对整个插件生态的断言。[辅助窗口策略][dsh-local]、[相关测试][dsh-local-test]

### 文档与当前代码有差异

架构文档仍有“不使用 renderer IPC 插件系统、不向页面暴露 Electron API”的描述。固定快照的 `preload.ts` 已有窄接口：解析用户选择的 File 路径、发出有限的 Desktop 动作；main 验证 sender、主 frame 与来源再派发动作。准确结论是“没有向页面开放任意 Electron API”，不能继续概括为“完全没有 preload/IPC”。[preload][dsh-preload]、[动作派发][dsh-dispatch]、[main 校验][dsh-shell]

## 4. 本项目具体借鉴什么

| DSH 做法 | GPTSwitch 的采用方式 | 边界 |
| --- | --- | --- |
| 一代 shell 统一持有窗口、托盘、监听器 | `DesktopSession` 统一装配与幂等释放；核心服务独立于窗口显示 | shell 生命周期不等于配置 Revision，重开窗口不能重新应用配置 |
| 平台策略集中处理差异 | macOS / Windows 的窗口、菜单、托盘与文件定位集中在壳适配 | 不把平台判断散到 React 页面 |
| 有限动作接口与来源校验 | `DesktopClient` + DTO；只开放具体业务用例 | 不开放任意命令/文件接口 |
| renderer 恢复有次数和健康条件 | 窗口故障恢复与网关/配置恢复分开，重开后同步 operation snapshot | 不无限重启 Codex，不靠重新写配置修 UI |
| 最小化、恢复、窗口状态有专门处理 | 两平台检查隐藏到托盘、唤回、缩放、窗口位置、多显示器 | 不默认禁用后台节流来掩盖重绘问题 |
| 打包与原生依赖有验证脚本 | 使用实际安装包做路径、签名、升级与恢复验收 | 不能把开发环境启动成功当打包通过 |

平台结构见 [electron-platform.ts][dsh-platform]；恢复逻辑见 [renderer-recovery.ts][dsh-recovery]。这些是设计借鉴，不声称 DSH 的 Electron API 可以直接搬进 Tauri。

## 5. 不照搬的部分

DSH 已经有 Host、插件与 Web 客户端，使用本地 HTTP/WebSocket carrier 有其上下文。GPTSwitch 的界面是新建的配置管理界面，没有浏览器远程访问要求，继续使用受限桌面 IPC，不另开 Web 管理端口。

DSH 的 profile 插件装配、终端、局域网入口和多种桌面呈现模式不进入本项目 P0；Rust 核心与模型网关也不改写成 DSH 插件。视觉仍沿用 CodexSplit 黑灰风格，不引入另一套玻璃材质和多套窗口栏。

未来若改 Electron，可以直接借鉴其平台策略与资源所有权模式；但针对本工具，首发一个本地 BrowserWindow 已够用，无需仅为了相似而复制两个 WebContentsView。确有独立网页内容时再增加隔离视图。

## 6. 对选型的影响

这次复审把“浏览器”拆成可理解的产品行为，不要求用户先判断技术名词：**首版完成供应商/Key/模型配置，网站外开；保留 Tauri，核心独立。** DSH 与星算助手共同支持“桌面壳和核心职责分离”的方向。

若以后确认要把供应商网站长期放在工具内使用，再按 [壳选型门槛](03-desktop-shell-decision.md) 优先切 Electron。不要一边保留配置工具的范围和工期，一边悄悄加入多标签浏览器工作量。

[dsh-package]: https://github.com/anywhere-labs/dsh-desktop/blob/cb736b6edfdd1e0584dea2252c23c7b4bf11420c/dsh-plugin-desktop/package.json
[dsh-license]: https://github.com/anywhere-labs/dsh-desktop/blob/cb736b6edfdd1e0584dea2252c23c7b4bf11420c/LICENSE
[dsh-architecture]: https://github.com/anywhere-labs/dsh-desktop/blob/cb736b6edfdd1e0584dea2252c23c7b4bf11420c/docs/architecture.md
[dsh-shell]: https://github.com/anywhere-labs/dsh-desktop/blob/cb736b6edfdd1e0584dea2252c23c7b4bf11420c/dsh-plugin-desktop/src/electron-shell-generation.ts
[dsh-views]: https://github.com/anywhere-labs/dsh-desktop/blob/cb736b6edfdd1e0584dea2252c23c7b4bf11420c/dsh-plugin-desktop/src/compatibility-shell.ts
[dsh-local]: https://github.com/anywhere-labs/dsh-desktop/blob/cb736b6edfdd1e0584dea2252c23c7b4bf11420c/dsh-plugin-desktop/src/local-window-policy.ts
[dsh-local-test]: https://github.com/anywhere-labs/dsh-desktop/blob/cb736b6edfdd1e0584dea2252c23c7b4bf11420c/dsh-plugin-desktop/tests/local-window-policy.spec.ts
[dsh-preload]: https://github.com/anywhere-labs/dsh-desktop/blob/cb736b6edfdd1e0584dea2252c23c7b4bf11420c/dsh-plugin-desktop/src/preload.ts
[dsh-dispatch]: https://github.com/anywhere-labs/dsh-desktop/blob/cb736b6edfdd1e0584dea2252c23c7b4bf11420c/dsh-plugin-desktop/src/renderer-actions-dispatch.ts
[dsh-platform]: https://github.com/anywhere-labs/dsh-desktop/blob/cb736b6edfdd1e0584dea2252c23c7b4bf11420c/dsh-plugin-desktop/src/electron-platform.ts
[dsh-recovery]: https://github.com/anywhere-labs/dsh-desktop/blob/cb736b6edfdd1e0584dea2252c23c7b4bf11420c/dsh-plugin-desktop/src/renderer-recovery.ts
