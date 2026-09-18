# 参考项目与本地应用调研

日期：2026-09-17。源码定位与固定提交见 [证据索引](../appendix/01-source-index.md)。本报告是围绕需求的文档、调用链、实现及测试定向阅读，不声称逐行审计全部仓库。

## 1. 调研方法与证据等级

- **A 官方契约**：官方文档、配置 schema、公开协议定义；仍需匹配实际版本。
- **B 源码事实**：固定 SHA 中可定位的实现与测试。测试文件存在不代表本次已经运行。
- **C 本地观察**：安装包结构、版本、只读 UI 与本机导出的 schema。
- **D 待验证判断**：架构推断、性能目标、跨平台可行性；必须通过实验再升级为结论。

对每个项目检查：定位与范围、配置写入、模型发现、能力映射、菜单接入、凭据存储、运行生命周期、失败处理、UI 和可借鉴程度。

## 2. 对比摘要

| 参考 | 本次快照 | 核心路径 | 适合借鉴 | 不宜直接沿用 |
| --- | --- | --- | --- | --- |
| CodexSplit | 源码 2.0.4；本机 2.0.1 | macOS SwiftUI/AppKit + WKWebView；Node 网关 + 自定义目录 + 进程级 Bridge | 供应商分组、模型命名空间、原生菜单接入、黑灰 UI | macOS 壳无法直接覆盖 Windows；缓存同步与进程校准的复杂性 |
| Prodex | 0.429.4 | Rust CLI 包装与运行时代理 | 纯协议转换、能力契约、请求提交边界、续接绑定 | 企业治理、多账户、多运行时规模 |
| Codex Switcher | 0.2.18 | Tauri/React + Rust 认证文件切换 | 跨平台桌面壳、托盘、串行化切换、退出检测 | 将账号轮换等同于供应商/模型管理 |
| 星算助手 | 本机 1.6.1 | Electron → IPC → Rust core-host | 模型中心与工具配置分离、扫描检测、应用动作 | 综合工作台的功能密度与云端业务 |

## 3. CodexSplit

### 3.1 产品与实现

项目明确为 OpenCodex 的更名延续。源码分为 `src_v2` 核心、Node 控制服务、Dashboard、macOS 原生壳和额外伴随功能。macOS 壳管理随包 Node 与网关进程。当前 README 的版本描述落后于 `package.json`：不能用网页首屏的 2.0.1 替代本次源码 2.0.4。

2026-09-18 追加核对：远端 HEAD 仍为已固定的 `6cab7ed6fe1d2ad910aa7fef3128e67669221604`。**公开 macOS 主应用没有使用 Electron 或 Tauri，而是自行实现原生壳 + 系统 WebView + Node sidecar。** 分层如下：

| 层 | 实际实现 | 证据 |
| --- | --- | --- |
| 窗口与应用生命周期 | Swift、SwiftUI、AppKit；原生启动/失败页，关闭窗口后保持后台 | [OpenCodexApp.swift](https://github.com/AITabby/codexsplit/blob/6cab7ed6fe1d2ad910aa7fef3128e67669221604/macos-app/Sources/OpenCodex/OpenCodexApp.swift#L27) |
| 主界面承载 | WebKit 的 WKWebView，通过 NSViewRepresentable 嵌入 SwiftUI | [DashboardView.swift](https://github.com/AITabby/codexsplit/blob/6cab7ed6fe1d2ad910aa7fef3128e67669221604/macos-app/Sources/OpenCodex/DashboardView.swift#L5) |
| Dashboard 界面 | 原生 HTML/CSS/JavaScript；getDashboardHtml 返回页面字符串，fetch 调用本地接口；这条路径未用 React/Vue | [dashboard.ts](https://github.com/AITabby/codexsplit/blob/6cab7ed6fe1d2ad910aa7fef3128e67669221604/src_v2/services/dashboard.ts#L1) |
| 本地后台 | TypeScript 由 tsc 编译为 JavaScript，在 Node.js 中运行；HTTP 入口使用 node:http 的 createServer | [package.json](https://github.com/AITabby/codexsplit/blob/6cab7ed6fe1d2ad910aa7fef3128e67669221604/package.json#L6)、[gateway.ts](https://github.com/AITabby/codexsplit/blob/6cab7ed6fe1d2ad910aa7fef3128e67669221604/src_v2/server/gateway.ts#L4407) |
| 壳与后台连接 | Swift Process 启动随包 Node / dist/server.js，传入端口与父进程信息；WKWebView 加载本机 /dashboard | [GatewayProcess.swift](https://github.com/AITabby/codexsplit/blob/6cab7ed6fe1d2ad910aa7fef3128e67669221604/macos-app/Sources/OpenCodex/GatewayProcess.swift#L55) |
| 模型接入 | 自定义模型目录、Codex 配置、TypeScript Bridge、协议适配和请求路由 | 见下方核心调用链及证据索引 |

这里使用 Node.js 不代表使用 Electron；Node 负责后台，页面由系统 WebKit 渲染。你喜欢的主控制台风格来自网页的 CSS 与交互组织，不依赖 SwiftUI 或 Electron 才能复现。

**Windows 范围限制**：README 明确提供 v1.2.0 EXE，未公开 Windows 源码；本次没有分析该 EXE。不能由 macOS 源码推断 Windows 也用 Swift、Electron、Tauri 或相同 Node 网关，也不能把两个版本当作功能完全一致。[README 平台说明](https://github.com/AITabby/codexsplit/blob/6cab7ed6fe1d2ad910aa7fef3128e67669221604/README.md#L63)

对本项目而言，“桌面壳显示本地界面，独立后台负责模型与路由”值得借鉴。若直接采用其 macOS 原生壳，Windows 仍需另一套壳及平台适配；因此它没有提供一套可直接套用到两平台的公开桌面工程。当前 Tauri 方案也使用系统 WebView，但本项目选择 Rust 核心和桌面 IPC，与 CodexSplit 的 Node sidecar / HTTP Dashboard 是不同实现，不能混为同一技术栈。

核心调用链（B）：

```text
Dashboard 录入供应商 / Key / 模型
  → credential_store：元数据、凭据引用、Key 策略
  → catalog_sync：模型能力与自定义目录
  → gateway：写受管理配置、管理 Desktop 启动
  → CODEX_CLI_PATH 指向 Bridge
  → Bridge 启动原生 app-server、控制请求出口
  → 第三方网关 / 原生官方请求路径
```

关键阅读结果：

- `buildManagedCodexConfig` 写 `model_catalog_json` 和自定义 provider，并保留原生 OpenAI 默认身份；实际第三方路由由 Bridge 协调。
- `buildDesktopLaunchEnvironment` 将 `CODEX_CLI_PATH` 限定在被启动的 Desktop 进程环境，不用全局环境变量接管所有实例。
- `CatalogSyncService` 除写自有目录，还会合并 Codex 的 `models_cache.json`；这引入“业务模型、目录文件、宿主缓存、运行进程”多个状态面。
- `codex-provider-bridge.ts` 涉及线程/回合路由、原生出口、运行时恢复和显示字段修饰，远超一个简单配置编辑器。
- `credential_store.ts` 有 macOS Keychain 引用和多凭据选择逻辑；不能由此推断 Windows 同样实现。
- `model_catalog_identity.test.mjs` 覆盖精确模型 ID、目录能力、原生与第三方身份隔离等回归点，本轮未执行测试。

### 3.2 与用户体验问题的关系

源码中确有 `reconcilePreferredBridgeAfterGatewayReady`：在满足偏好和运行条件时重新附着 Bridge；目录又存在宿主缓存投影。它们为“配置看似成功、实际进程仍未加载”提供了可能机制。但本次没有复现用户失败现场，**不能断言某个函数就是当前故障根因**。

用户截图中的“模型能力目录”侧重 Agent Profile 的擅长领域与推理强度。2.0.4 的内部类型和目录逻辑已处理上下文、视觉和推理元数据，不能说它完全没有这些能力；本次检查到的供应商表单没有覆盖用户要求的完整手工编辑字段。

值得特别区分的实现（B）：

- 目录构建会启用视觉桥，因此模型本身不支持图片时，也可能向 Desktop 声明可接收图片，再交由其他视觉路径转换。
- 推理档位逻辑默认补入 low / medium / high，除非明确声明非推理模型。
- 这些是项目的兼容策略，不是供应商模型的能力证据。本项目默认不照搬，避免能力虚报和隐式跨模型发送。

### 3.3 本项目取舍

保留黑灰层级、供应商列表、Key 管理、模型命名空间和明确“应用”入口。将能力编辑从 Agent 路由中独立出来；将状态同步收敛到一个事务控制器；默认不修改 Codex 自有模型缓存、不自动重启修复、不附加视觉模型调用。

仓库本次跟踪文件中未发现项目级 LICENSE；第三方声明不能代替项目授权。采取独立实现，仅参考行为和架构；直接复用源码前另行确认授权。

## 4. Prodex

### 4.1 定位

这是多供应商、多账户的 Codex / Claude 运行包装器，而非用户期望的简洁桌面控制台。本次重点阅读架构、状态模型、供应商契约、模型目录生成及其测试数据，不以其全部企业功能作为产品范围。

### 4.2 有价值的代码边界

`prodex-provider-core` 管纯能力判断与请求/响应转换；`prodex-app` 管网络、凭据、选择与重试。`runtime_external_provider_config` 负责运行目录与配置覆盖；`catalog_model.rs` 生成 Codex 模型目录对象。公开代码中的字段是重要参考，但某些目录默认值仍是项目策略，不等于上游支持。

三个可直接转为本项目设计原则的实现思路：

1. **转换有结果等级**：lossless、degraded、rejected、unsupported；不能默默丢字段后宣称兼容。
2. **续接绑定优先于调度**：`previous_response_id`、回合状态和会话标识绑定原 profile，不能为换 Key 就将增量上下文发给另一家。
3. **输出开始后不自动换路**：区分请求被接受、流式已提交及未提交失败，防止重试造成重复内容与工具执行。

本项目会采用这些边界，保留轻量本地实现，不引入其 PostgreSQL、Redis、控制平面、多租户及完整 profile 体系。仓库 LICENSE 为 Apache-2.0，若后续实际复用仍需记录来源、修改与随包声明。

## 5. Lampese / codex-switcher

### 5.1 定位与技术

Tauri 2、React、TypeScript、Rust；有 macOS / Windows 配置、托盘、更新与进程检测。主要解决账号切换、用量查询、认证导入和自动预热。

### 5.2 实现阅读

`commands/account.rs::switch_account_by_id` 获取认证操作锁，检查 Codex 是否运行，将当前账号最新 token 同步回存储后，再写目标账号。`auth/switcher.rs` 尊重 `CODEX_HOME`，将认证写到 `auth.json`，Unix 设置权限。

这说明“切换前同步状态、并发串行化”值得学习；同时也说明它不是供应商模型目录解决方案。API Key 写入 `auth.json` 本身不解决自定义 URL、模型能力或 Desktop 菜单。

本项目借鉴 Tauri 托盘、平台入口和清晰切换反馈；不复制账号存储和 OAuth 逻辑。该快照未发现项目级 LICENSE，源码复用与分发权限列为待确认。

## 6. 星算助手：本地安装包与界面

### 6.1 观察范围

只读检查 `/Applications/星算助手.app`：版本 1.6.1，Bundle ID `xyz.xsai5.desktop`。实际打开首页、模型中心、第三方服务商标签、添加模型表单、应用管理及 Codex 详情。未登录、未提交表单、未点应用模型、恢复订阅或注入功能。

UI 观察（C）：

- 模型中心区分自带 Key 的服务商模型与平台目录，支持表格/卡片和手动刷新延迟。
- 添加模型表单包含名称、OpenAI 地址、Anthropic 地址、模型 ID、模型用途和 API 密钥。
- 工具配置页可扫描安装位置、配置文件和版本；当前工具与待应用模型分开选择。
- Codex 页面有修改模型配置、恢复订阅登录、启动应用等入口，提示代理使用时保持助手运行。
- 当前观察到的手工模型表单未提供完整的上下文、输出和模态矩阵编辑。

### 6.2 安装包技术证据

`app.asar/package.json` 指向 `dist-electron/main.cjs`；preload 通过 `contextBridge` 暴露 invoke、shell 与事件。main 启动 `core-host/xingsuan-core-host`，以 stdin/stdout 消息传输，带协议版本握手；窗口启用 contextIsolation / sandbox，关闭 nodeIntegration。界面依赖可见 React 生态、Zustand、Radix 等。

前端能定位 `apply_model_to_tool`、`get_tool_model_info` 等命令；核心可执行文件可检索到配置与模型目录字段字符串。这只能证明实现线索，**不能证明具体写入事务、密钥算法、路由转换正确性或所有能力已接通**。没有公开 Rust 源码，也没有反汇编或绕过保护。

官网 FAQ 仍出现 Tauri 描述，与本机 Electron 安装包不完全一致。2026-09-18 文档补充：用户转述开发者表示此前使用 Tauri，因内嵌浏览器成本改成 Electron；与本机观察吻合。迁移历史及原因按用户转述记录，具体日期、成本和性能未独立验证。官网的数据本地性、价格和系统要求属于厂商声明，本次不做独立背书。

### 6.3 对本项目的启发

采用“模型一次定义、应用到指定工具实例”的组织方式；保留安装检测和统一错误入口。本项目只有 Codex 一个目标，不引入综合工作台、平台目录、登录/余额和推荐内容。

该迁移反馈适用于桌面壳复审，不足以证明本项目也必须内置浏览器；现阶段保留 Tauri，抽离壳无关核心，确定浏览器范围后可在创建工程前改 Electron。详见 [桌面壳选型复审](03-desktop-shell-decision.md)。

## 7. 九张截图的设计阅读

| 截图 | 可观察结构 | 本项目处理 |
| --- | --- | --- |
| split-01 | 固定侧栏、网关双列卡片、待应用模型 | 保留层级；将首页压缩成连接状态、供应商与待应用清单 |
| split-02 | 长表单、Key 池、URL、模型发现、手动模型 | 拆供应商详情与模型能力编辑；固定操作栏 |
| split-03 | 分配规则与模型目录 | 不做 Agent 分配，模型目录进入独立导航 |
| split-04 | 模型 Profile、工作说明、推理选择 | 仅借鉴分组；增加真实参数映射与能力证据 |
| split-05 | 账号调度与添加账号 | 不纳入首发 |
| split-06 | 语音多卡片设置 | 不纳入首发，借鉴设置分组 |
| split-07 | 左列表右详情 | 借鉴到供应商详情与诊断，不读取会话内容 |
| split-08 | 日志列表与刷新 | 增加结构化过滤、阶段和恢复入口 |
| split-09 | 恢复原生与应用操作 | 保留，恢复说明改为准确的字段级撤销 |

共同特征：接近黑色底、深灰卡片、细灰边框、白色主按钮、灰阶导航选中、圆角、少量状态色。截图中重复语言按钮、较长说明、重叠配置入口不是需要继承的设计目标。色值和尺寸的项目规范是后续主动制定，不声称从压缩截图精确还原。

## 8. 调研结论

真正难点在“目录、路由、运行进程和配置的一致性”，不在 CRUD。采用自有单一配置源、不可变运行版本、真实能力元数据、字段级配置事务和明确兼容矩阵，才能比参考产品更简单。具体实现与验证方案见 [可行性分析](02-codex-feasibility.md)。
