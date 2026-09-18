# 开发实施计划

本轮只形成文档；以下任务须在进入开发阶段后执行。预估按 1 名熟悉 Rust/React 的开发者、可使用双平台真机计算，不含签名证书办理时间，也不是交付承诺。

## 1. 分期与发布门槛

| 阶段 | 目标 | 预估有效工作日 | 输出 | 通过条件 |
| --- | --- | --- | --- | --- |
| G0 兼容探索 | 原生菜单与路由闭环可行 | 3–5 | 两平台实验记录、schema fixture、架构 ADR 更新 | 两供应商菜单可选、路由正确、可恢复 |
| G1 基础工程 | 最小可信核心和设计骨架 | 3–4 | Tauri 工程、类型生成、CI、凭据与 DB | 两平台安装运行，秘密不落普通存储 |
| G2 配置核心 | 供应商/Key/模型及保真配置 | 5–7 | CRUD、能力编辑、计划/提交/恢复 | 冲突、崩溃恢复测试通过 |
| G3 网关闭环 | Responses + 受约束 Chat | 6–9 | SSE、工具续接、取消、能力执行 | 协议 conformance 门禁通过 |
| G4 产品体验 | 全页面、诊断、托盘、状态 | 4–6 | 核心用户流程、视觉验收 | 非技术用户可完成首次接入/恢复 |
| G5 双平台稳定 | 真机、签名、更新、卸载 | 5–7 | 发布候选包、测试报告、恢复说明 | 零阻断缺陷，版本兼容矩阵完整 |

合计约 26–38 个有效工作日；Bridge 若被 G0 判为必要，另预留约 5–10 天，复杂度视目标 Desktop 的真实行为调整。界面可以与核心逐步对接，但不应在 G0 未证明核心路径前投入完整视觉实现。

## 2. G0：最高优先级探索

### G0-01 环境与版本

在 macOS 和 Windows 记录 Desktop 版本、内置 CLI、启动路径、安装渠道、schema hash、测试 `CODEX_HOME`。采用隔离用户/测试配置根，不能覆盖当前工作配置。

### G0-02 官方目录路径

用无秘密 mock Responses 服务提供两个不同供应商的同名模型；编译合法目录，写测试 provider。通过实际 Desktop 验证菜单、默认选择、新任务/旧任务行为、模型 ID、provider、输入模态和 reasoning 选项。不能只看 `model/list` 结果。

### G0-03 热变更与重载

分别改变 Key 引用、输出上限、上下文、菜单名称、新增模型，记录需要重新发布、重新开任务、重启 app-server 或重启 Desktop 的差异。测清正在运行的任务是否保留旧版本。

### G0-04 恢复

停止网关、损坏目录、占用端口、外部改配置，验证能保留原配置、登录和历史；若宿主不可观察，证明 UI 会停在“加载未确认”。

### G0-05 路线判定

- 两平台基线通过：采用官方配置 + 本地网关。
- 一个平台失败：定位为目录、认证、启动还是任务 provider 固定；记录原始错误，再评估该平台 Bridge。
- Bridge 成本/稳定性不能满足目标：明确版本支持限制，回到产品决策；不得把“右下角显示”从验收里删掉。
- 官方订阅同菜单共存不强塞进基线；独立实验官方请求绝不误入第三方。

### G0-06 桌面壳与界面边界

按当前“配置工具、网站外开”范围保留 Tauri；若明确需要高频内嵌控制台、多标签/隔离登录，则在 G1 前优先改 Electron + Rust core-host，并更新架构、工期和资源预算。DSH 的本地 Web 界面承载不等于本项目必须增加浏览器，具体见 [DSH 调研](../research/04-dsh-reference.md) 和 [壳选型复审](../research/03-desktop-shell-decision.md)。没有这一需求时，不为比较框架额外开发两套 PoC。

G1 实施时核心独立为 crate；界面通过 DesktopClient 接口访问壳适配。参考 DSH 的资源所有权与平台策略，窗口/监听器统一释放，UI 重建不触发模型重新应用。原生菜单链路失败仍在 Codex 接入层排查，不能把换壳当默认修复。

## 3. G1/G2 任务拆解

| ID | 工作包 | 依赖 | 验收输出 |
| --- | --- | --- | --- |
| ENG-01 | Tauri 壳、独立 switch-core、DesktopSession、脚本与 CI | G0 架构确认 | 两平台构建产物；核心不依赖 Tauri；平台窗口差异集中 |
| ENG-02 | 领域类型、schema、IPC 错误生成、DesktopClient transport | ENG-01 | TS/Rust 契约一致；业务组件不直接调用 Tauri API |
| ENG-03 | OS 凭据适配及锁定错误 | ENG-02 | 新增/替换/删除、安全失败测试 |
| ENG-04 | SQLite migration、Revision 和 journal | ENG-02 | 中断恢复 fixture |
| CFG-01 | 实例检测与配置层检查 | G0 | 正确路径与有效值说明 |
| CFG-02 | TOML 保真读写、所有权 | ENG-04 | 注释/未知字段 golden tests |
| CFG-03 | ApplyPlan、CAS、原子替换 | CFG-02 | 外部冲突测试 |
| CFG-04 | rollback、恢复原生、幂等 | CFG-03 | 崩溃阶段全覆盖 |
| MOD-01 | 供应商/Key/模型 CRUD | ENG-03/04 | 页面与后端验证一致 |
| MOD-02 | 发现结果与用户覆盖分层 | MOD-01 | 刷新不覆盖手工值 |
| MOD-03 | 能力约束与 CatalogCompiler | G0 schema + MOD-02 | 合法目录和否定案例 |

## 4. G3/G4 任务拆解

| ID | 工作包 | 依赖 | 验收输出 |
| --- | --- | --- | --- |
| GW-01 | 受认证 loopback、实例端口 | ENG-03 | 非法请求被拒绝 |
| GW-02 | 不可变路由快照与请求绑定 | ENG-04 | 热更新不改变在途请求 |
| GW-03 | Responses SSE 透传/取消 | GW-01/02 | 断流、取消、背压测试 |
| GW-04 | Chat 文本与 function tools 适配 | GW-03 | 多轮调用、参数碎片 fixture |
| GW-05 | 输出/推理/模态执行 | MOD-03 + GW-03 | 请求参数捕获和上游真实测试 |
| GW-06 | continuation 绑定与错误分类 | GW-02/04 | 不能跨 Key/路由误续接 |
| UX-01 | token 与基础组件 | ENG-01 | 深色组件状态预览；两套主题与强调色已落地，切换见 [视觉基础](../design/01-foundations.md) |
| UX-02 | 供应商、模型与首次接入 | MOD-01/03 + UX-01 | 键盘完整流程 |
| UX-03 | 配置差异、应用/恢复结果 | CFG-04 | 成功/等待/冲突/恢复页面 |
| UX-04 | 诊断、日志与托盘 | GW-03/05 | 可定位失败且不泄露秘密 |
| UX-05 | 双平台布局与无障碍 | UX-02/03/04 | 屏幕尺寸与读屏记录 |

请求网关不应等待所有页面完成才测试；UI 使用同一 DTO 的合成 fixture，最终换接 IPC，不能另造前端业务真相。

## 5. 建议工程目录

以下仅为未来目录规划，本轮未创建这些源代码目录。

```text
src/
  app/                 路由、窗口、全局错误
  features/            providers、models、codex、diagnostics、settings
  components/          primitives、business、templates
  styles/              tokens、global、platform
  contracts/           生成的 IPC types
  desktop/             DesktopClient 接口与唯一 Tauri transport
  locales/             zh-CN、en
src-tauri/
  src/                 装配、command handlers、DesktopSession、窗口/托盘平台适配
  capabilities/        桌面 IPC 权限
crates/switch-core/
  src/domain/          实体、约束、能力状态
  src/application/     用例、apply coordinator
  src/storage/         SQLite、migration、journal
  src/credentials/     凭据 ports 和 OS adapters
  src/codex/           检测、schema、catalog、config、compatibility
  src/gateway/         HTTP、路由、SSE、取消
  src/protocols/       Responses、Chat、future adapters
  src/platform/        OS 路径、文件、进程；不依赖窗口框架
  src/diagnostics/     probe、错误、脱敏
tests/
  fixtures/            已脱敏协议与配置样本
  integration/         mock gateway、配置故障注入
  desktop/             双平台手工/自动验收说明
docs/                  本套设计与后续实验记录
```

## 6. 编码规则与完成定义

配置和网关核心不依赖 UI；所有网络转换必须返回显式能力/损失状态。Rust 不将任意内部错误直接送给前端；日志字段使用 allowlist。时间和 ID 生成可注入，便于确定性测试。

`switch-core` 不引用 Tauri 的 AppHandle、State 或事件 API；壳将核心事件映射到 DTO。窗口重建后读取既有 operation snapshot，不能新建相同配置事务。当前仅实现 Tauri，Electron 备选结构不进入首发代码。

每个任务完成需：相关测试通过、错误/取消路径可用、无秘密进入日志、文档契约更新、两平台相关路径至少编译通过。涉及 Desktop 行为的任务还必须有实际 UI 和请求记录。反复运行相同 happy-path 不替代新的失败场景覆盖。

## 7. 版本计划

0.1 alpha：完成 G0–G3 的最小闭环，内部验证；0.2 beta：双平台产品流程和诊断；1.0：达到正式发布门禁。阶段编号不代表确定发布日期。P1 账号相关功能不因参考项目存在而自动进入 roadmap。

第一版的取舍原则是宁可协议支持清楚而有限，也不提供“添加任意模型后所有 Codex 工具都能用”的承诺。
