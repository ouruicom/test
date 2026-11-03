# 德莱聚合（delai-hub）技术方案 v1.8 · 设计冻结版

> 聚合聊天 + 实时翻译桌面客户端 · 完整开发方案
> 版本：v1.8 · 2026-11-03（基于 v1.0 方案稿修订）
> **状态：设计冻结。** v1.7.6 之后不再迭代文档内部细节；引擎与术语相关章节在 §13 的 spike 结果回填前保持冻结。

---

## 冻结声明（v1.8，请先读这一段）

外部评审的结论成立：**连续 7 轮都是"读文档挑错"，没有一行代码、没有一次真实请求；再出一版细节修订不会产生新信息。** 本版因此不是 v1.7.7，而是一次状态切换：

1. **不再迭代设计细节。** 下面 §13 之前的正文是**冻结快照**，供 spike 期间照着写代码用；发现缺陷记入 §13.4「缺陷池」，不单独发新版。
2. **§0 的七项外部接口事实是唯一阻塞项。** 引擎路由（`selectRoute`）、术语库建模（§4.7.1）、R19 自适应并发都建在这些假设上。S1 完成前，任何"要不要改引擎设计"的讨论都不成立。
3. **本版只做两类不依赖实测的事**：① 修掉 7 处纯代码逻辑缺陷（§4.2 / §4.2.1 / §4.4，见下）；② 把评审指出的产品级缺口写成**待回填占位**，不编造数字。
4. **不编数据。** CPU 占用、耗电、首字 P95、队列等待 P95、真实并发阈值——这些必须来自真机实测，文档里一律留空标注「待 S2/S3 回填」。
5. **真实 API 请求需要 Key，只能由项目侧执行。** 联网可核的是"文档层事实"（接口是否存在、参数表写了什么），核不了"带 Key 的真实返回"。分工见 §13.1。

### 本版修订（v1.7.6 → v1.8）

| # | 来源 | 处置 |
|---|---|---|
| 1 | 评审 A / 核心主张 | **停止迭代**，新增 §13 spike 三连计划；§0 升级为阻塞项 |
| 2 | 评审 C-1 | `https.request` 的 `timeout` 注释纠正为 **socket 空闲超时**，补**绝对 deadline**（`Date.now()` 重算剩余时间），防心跳让流永不结束、占死队列槽位 |
| 3 | 评审 C-2 | 定义 `HttpError{status,retryable,body}`；429 退避 / 401-403 熔断 / R19 自适应改为读结构化字段，**不再正则解析 message** |
| 4 | 评审 C-3 | `httpJson` / `httpStream` **内部强制** `Content-Type: application/json`，不靠调用方传 |
| 5 | 评审 C-4 | `httpJson` 补 `AbortSignal`，A1「取消翻译」能中断在途请求 |
| 6 | 评审 C-5 | `abort` 监听在结算时 `removeEventListener`，避免长跑进程累积监听器 |
| 7 | 评审 C-6 | 定义**流式不自动重试** + 幂等边界：重试前清空半截译文、同一 `requestId` 不重复计费 |
| 8 | 评审 C-7 | 队列补**优先级（当前会话 > 后台账号）与背压**（超限丢弃低优先级、合并同文本） |
| 9 | 评审 B-2 | §3.3.4 补 CPU / 耗电账目**占位**（数值待 S3 真机回填） |
| 10 | 评审 B-5 | 修 Messenger 口径矛盾：类型保留但标注「预留、本期不实现」，删掉 §5.4 里会给排期错觉的表述 |
| 11 | 评审 B-3/B-4、D | 新增 §14「产品级风险与验收基线」占位章节（封号与 ToS、性能 SLA、本地数据安全、发布运维、人天估算），**只列待答问题，不写结论** |
| — | 评审 C-8 | ❌ **驳回**：选择器外置 JSON + 远端下发 + 灰度 + 回滚**已在 §4.6.7（R30）完整实现**，含校验和、`minAppVersion`、保留上一版回滚、10% 灰度。评审自述"只读到 §4.3"，此条系阅读范围所致 |

> 关于 B-3「封号风险全篇零字」：**纠正定位**。文档已有三处零散提及（防关联封号、多账号同 IP + WebRTC、新 Session 风控引导），但**确实没有独立章节**，也没有 UA 一致化、操作频率上限、封号后数据迁移路径。按"缺独立章节"处理，不按"零字"处理。

---
## §0 API 事实上线前实测清单 — **唯一阻塞项**（v1.8 升级）

> **v1.8 状态**：这是整个方案里**唯一还在阻塞设计的东西**。引擎路由（`selectRoute`）、术语库建模（§4.7.1）、R19 自适应并发、`ApiCaps` 全部建在下面这些假设上。**在 S1（§13.2）完成并把结果回填之前，引擎与术语相关章节冻结，不做进一步设计讨论。**
>
> 需要带 API Key 的真实请求，只能由项目侧执行；可复制的判定表见 §13.2 S1。预计半天可完成。

连续几轮评审的核心分歧都是"某个接口到底存不存在、参数叫什么"。**文档里的每个外部接口断言都必须可验证**，以下七项各发一次真实请求确认：

| # | 待验项 | 怎么验 | 不验的后果 |
|---|---|---|---|
| 1 | TokenHub 术语库管理接口可用（`CreateGlossary` 等，`tokenhub.tencentcloudapi.com`，Version `2026-03-22`） | 控制台创建一个测试术语库，取回 `GlossaryId` | A3 路径② 建在假设上 |
| 2 | **独立翻译接口的确切路径与参数**：`POST https://tokenhub.tencentmaas.com/v1/api/translations`，含 `context` / `glossary_ids` / `source` / `target` / `stream` | 带 `glossary_ids` 发一次请求，看返回；**确认 `context` 字段名与类型（string 而非数组？）** | A1 上下文翻译按错误的字段名编码，静默失效 |
| 3 | 该独立接口是否支持 `hy-mt2-lite` | 用 lite 档发一次 | 若仅 plus/pro 支持术语库，lite 档默认路径要改 |
| 4 | **TokenHub 实际并发上限** | 逐步加压至出现 429，记录阈值 | 自适应并发的上/下限无依据 |
| 5 | **独立翻译接口的鉴权方式与 API Key 格式** | 用同一 API Key 分别试 `Authorization: Bearer <key>` 与 TC3-HMAC-SHA256 签名，看哪个返回 200 | 引擎鉴权写错，写代码第一天就卡住 |
| 6 | **【v1.7.4 补】`glossary_ids` 与 `references` 的"最多 10 个"上限** | 创建 11 个术语库并一次传入 11 个 ID，看是报错还是静默截断 | §4.7 的术语库建模直接依赖这个数字 |
| 7 | **【v1.7.5 补】`context` 的实际拼接格式** | 分别传纯文本上下文、`user:/assistant:` 角色前缀、JSON 串三种，对比末句译文的承接质量 | §12.5 写死的角色前缀格式可能模型不识别，等于没传上下文 |
| 附 | **`references` 是否真的接受"术语"对象**（参数表写"参考示例（句子或术语）"，但类型未细化） | 传一条 `{type:'term', ...}` 看是否被采纳 | 决定客户级术语能否走路径②（§4.7.1 混合方案） |

> **第 2 项尤其重要**：`context` 字段在官方《混元调用指南》的独立翻译接口参数表中**已明确列出**（详见 §12.1 问题 2），但**端点路径、鉴权方式与字段精确类型（文档标注为 string，**需实测验证**）仍需一次真实请求最终确认**。确认之前，A1 按 §12.5 的 `contextText` 设计编码，并保留回退：若实测字段名或类型不符，A1 回退 prompt 内嵌（路径①），功能不变、仅实现差异。

> **第 3 / 6 / 7 项共同决定引擎的路由回退条件**，见 §12.5 `selectRoute()`：任一能力不满足，自动回退 `/v1/chat/completions`。

---

## 版本演进（历史修订已压缩，当前结论为准）

> 本方案经多轮评审迭代。**顶层结论以正文为准**，下表只用于回答"某个结论是哪个版本改的"。
> 已被后续版本推翻的旧结论不再保留（例：v1.7 曾写"并发上限改为 5"，v1.7.1 已推翻，见 §12.1 补充更正）。

| 版本 | 主题 | 一句话结论 |
|---|---|---|
| v1.1 | 阻塞级纠错 | 修掉 6 个"不改则核心功能不可用"的缺陷（关证书校验、React 受控写入、虚拟列表丢译文、缓存 key、Key 熔断、native rebuild） |
| v1.2 | 写入与选择器加固 | 出站写入改**三层降级链**；选择器改**四档降级链**；缓存加预热实现"秒回" |
| v1.3 | 稳定性加固 | 保活三件套、SQLite WAL 并发、IPC 安全加固、多显示器 DPI 适配 |
| **v1.4** | **方向性纠错** | **推翻自动休眠**：聊天账号默认永不销毁，实时性优先于内存（§3.3） |
| v1.5 | 吸收外部评审 21 条 | 采纳 20、驳回 1（共享渲染进程池不可行）；补启动懒加载、崩溃恢复、Key 加密、迁移框架等 |
| v1.5.1 / v1.5.2 | 代码级纠错 | 修 `resolve` 签名、术语表版本号永不递增、KeyVault 明文口径矛盾等实现级缺陷 |
| **v1.7** | 功能取舍 | 处置 21 项功能建议：直接采纳 10、改设计后采纳 5、驳回/降级 6（§12） |
| v1.7.1 | 事实复议 | **推翻 v1.7 两条错误的事实订正**：TokenHub 术语库未下线（A3 路径② 恢复）；"并发 5 路"属另一产品，改回以实测为准 |
| **v1.7.2** | 残留清理 | 清理并发口径残留；新增 §0「API 上线前实测清单」 |
| **v1.7.3** | 正文纠错 | 12 处硬问题：`user_corrected` 缺列、`last_active_at` TEXT→INTEGER、迁移 v5 补两表、术语/`context` 口径统一、拆 `contextText/contextTurns`、修 `LruSet` 伪 LRU 等 |
| **v1.7.4** | 路径② 缺口 | 清理版本口径残留；补 `glossary_ids` 上限待实测；修路径② 的三条设计缺口（术语库个数上限、双请求体结构、复用 http-client 与限流） |
| **v1.7.5** | 阻断级修复 | 修断裂的 `CREATE TABLE`（改迁移 v6）；补 `httpStream()`；术语库建模补完为"公共库 + 客户级回退"；TranslateQueue 改自适应；补索引 |
| **v1.7.6** | 照抄必返工项 | 更正评审评级为 7/1/1；`httpStream` 改显式 `StringDecoder` + SSE 空行分帧 + 多行 data 合并；迁移改为幂等 `addColumnIfMissing()`，补 ADD COLUMN 硬限制与启动顺序 |
| **v1.8** | **设计冻结** | **停止迭代**；修 7 处不依赖实测的代码缺陷（deadline / HttpError / Content-Type / AbortSignal / 监听器解绑 / 流式幂等 / 队列优先级与背压）；新增 §13 spike 三连、§14 产品级风险占位；补 CPU/耗电与性能 SLA 待回填项 |

**当前版本：v1.8（设计冻结）。** 正文是供 spike 期间照着写代码的冻结快照，不再迭代；R1–R45 编号可直接按章节索引查阅。

---
## 一、项目概述

### 1.1 产品定位
面向外贸从业者的**多社交软件聚合聊天 + 自动翻译**桌面客户端（对标"译想"）。用户在一个窗口内管理 WhatsApp / Telegram / Messenger 等账号，收到外语消息自动翻译成中文显示在原文下方，中文回复经预览确认后翻译成客户语言发出。

### 1.2 核心价值
- **边聊边译**：不复制、不切应用，聊天窗口内直接完成翻译
- **多账号聚合**：一个界面管理多个账号，独立 Cookie / 存储 / 代理，防关联封号
- **多翻译引擎**：腾讯混元（境内默认）/ DeepL / Google（境外，需告知）可插拔切换
- **外贸定制**：产品术语表保证前后文译名一致（v1.1 新增）

### 1.3 目标用户
外贸从业者（乔永亮及团队）、跨境电商客服、海外社群运营。

### 1.4 非目标（v1.1 明确边界）
- 不做聊天记录的云端同步与备份（本地数据，用户自负保管责任）
- 不做全自动无确认发送（出站翻译一律走预览确认，R2 相关）
- **本期不实现 Messenger**（反自动化最强、选择器最脆）。**v1.8 口径统一**：类型定义、schema 枚举、zod 枚举里保留 `'messenger'` 是** schema 预留**（避免以后加平台要做数据迁移），不代表本期范围；§5.4 里"排最后"的措辞已改为"无排期"。见 §14.5「非目标清单」。

---

## 二、技术选型（v1.1 修订）

| 层 | 选型 | 版本建议 | 理由 | 修订 |
|---|---|---|---|---|
| 桌面框架 | Electron | 最新稳定版 | 可深度控制内嵌页面、绕过 CSP、注入脚本，Tauri 无法满足 | — |
| 内嵌视图 | **WebContentsView** | Electron 30+ | 官方已不推荐 `<webview>`（基于 Chromium webview，正经历架构变更、不保证未来可用） | **R7 变更** |
| 工程化 | electron-vite | 最新 | 主进程/preload/渲染三端一体化构建，HMR | — |
| 语言 | TypeScript | 5.x | 全链路类型安全，IPC 接口可生成类型 | — |
| 前端框架 | Vue 3 | 3.5+ | 模板语法接近 HTML，维护门槛低 | — |
| 状态管理 | Pinia | 2.x | Vue 官方，轻量 | — |
| UI 样式 | TailwindCSS | 3.x | 快速构建现代界面 | — |
| 数据持久化 | better-sqlite3（默认） / **node:sqlite（零编译备选）** | 11.x | 同步 API 简单；后者为 Node 内置、无需 rebuild | **R6 备选** |
| 原生模块构建 | electron-rebuild / @electron/rebuild | 最新 | 对齐 Electron 的 Node ABI，**Day 1 必须打通验证** | **R6 新增** |
| 打包 | electron-builder | 最新 | 出 Windows 安装包 + NSIS | — |
| 代码签名 | sectigo / 其它 OV 代码签名证书 | — | 未签名 exe 会被 SmartScreen 拦截、杀软报毒，用户装不上 | **R13 新增** |
| 调试 | electron-devtools-installer | 最新 | Vue DevTools | — |

**选型说明**：
- 数据库若目标 Electron 内置 Node ≥ 22.5，优先用 `node:sqlite`，彻底省去 native 编译与 Windows build toolchain 依赖（注意该模块仍在演进，需锁定 Electron 版本验证 API）。
- UI 框架也可选 React 18 + Zustand，但 Vue 3 对非专业开发者更友好。

---

## 三、系统架构（v1.1 重绘）

```
┌──────────────────────────────────────────────────────────────┐
│                    主进程 Main Process                        │
│                                                              │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────────┐  │
│  │ SessionMgr   │  │ TranslateSvc  │  │ Database         │  │
│  │ 多账号隔离    │  │ 队列/限流/缓存 │  │ SQLite           │  │
│  │ + 代理       │  │ + 引擎抽象     │  │                  │  │
│  │ + WebRTC防护 │  │ + Key熔断     │  │                  │  │
│  └──────┬───────┘  └───────┬───────┘  └────────┬─────────┘  │
│         │                  │                    │            │
│  ┌──────┴──────────────────┴────────────────────┴─────────┐  │
│  │              IPC 总线（类型安全 + 入参 schema 校验）      │  │
│  └──────┬────────────────────────────────┬───────────────┘  │
│         │                                │                  │
│  ┌──────┴───────────────┐      ┌─────────┴──────────────┐   │
│  │ ViewManager          │      │  Renderer UI (Vue 3)   │   │
│  │ (WebContentsView)    │      │  - 侧边栏账号          │   │
│  │                      │      │  - 设置面板            │   │
│  │ ┌──────────────────┐ │      │  - 快捷回复            │   │
│  │ │ acct_wa_1 (chat) │ │      │  - 客户档案抽屉        │   │
│  │ │ acct_wa_2 (idle) │ │◄────►│  - 未读提醒            │   │
│  │ │ acct_tg_1 (chat) │ │bounds│  - 选择器诊断面板 ★新增│   │
│  │ └──────────────────┘ │      │  - 翻译失败重试        │   │
│  │  preload + inject.js │      └────────────────────────┘   │
│  │  (host 白名单校验) ★ │                                   │
│  └──────────────────────┘                                   │
└──────────────────────────────────────────────────────────────┘
                    ▲ 层叠顺序：UI 视图最后 addChildView（置顶）
```

### 3.1 分层职责

| 层 | 职责 | 禁止做的事 |
|---|---|---|
| 主进程 | 网络请求、Session 管理、翻译引擎与调度队列、SQLite、视图管理 | 不碰 DOM |
| Preload | 暴露白名单 API（翻译、通知）、注入抓取脚本 | 不做业务逻辑 |
| 渲染进程 | 侧边栏/设置/快捷回复/客户档案/诊断 UI | 不直接发网络请求 |
| 页面注入 | DOM 监听、消息提取、译文渲染、输入框写入 | 不跨域请求、不在非白名单 host 上运行 |

### 3.2 视图层：WebContentsView 要点（R7）

```typescript
// 主进程 - ViewManager
const view = new WebContentsView({
  webPreferences: {
    session: session.fromPartition(account.partition),   // 每账号独立
    preload: path.join(__dirname, 'preload.js'),          // 必须绝对路径
    contextIsolation: true,
    nodeIntegration: false,
    sandbox: true,
  },
});
mainWindow.contentView.addChildView(view);   // 后添加者在上层
```

必须自己处理的三件事：
1. **resize**：`setAutoResize` 已移除，监听 `mainWindow.on('resize')` 手动 `setBounds`。详见 §3.4（含多显示器 DPI）。
2. **层叠顺序**：`addChildView` 顺序即 z 序；浮层（抽屉、Toast）最后添加。重排需 `removeChildView` + 重新 `addChildView`。
3. **生命周期**：**默认全部保活**，后台账号只 `setVisible(false)`，不销毁（详见 §3.3，v1.4 已推翻自动休眠策略）；严禁 `setBackgroundThrottling(true)`；监听 `render-process-gone` 自动重建。

---

### 3.3 多账号保活与内存管理（R20 → R24 重写）

> **本节在 v1.4 做了方向性纠错。** v1.3 的"三级休眠、后台账号销毁"是错的：聊天工具的核心价值就是"所有号都在线、都能实时收"，用内存换掉实时性等于把产品做残。Ferdium / Franz 这类多账号聚合器能同时挂十几个服务并实时收消息，走的就是**全部保活**路线。

#### 3.3.1 实时性的真实约束（只有一条）

**渲染进程活着 → 就能实时收消息；渲染进程被销毁 → 一定收不到。** 就这么简单。

关键在于：Chromium 默认会对"不可见 / 被遮挡 / 失去焦点"的渲染进程做后台化与定时器节流，长时间无交互后甚至挂起渲染进程，**导致 WebSocket 断连、通知中断**。这正是同类客户端（如 Discord 第三方客户端）出现"后台挂 15–20 分钟后收不到消息、切回来重新加载"的原因。

所以保活要做两件事：**让渲染进程保持存活**，且**不被 Chromium 节流/挂起**。

#### 3.3.2 保活三件套（必做）

```typescript
// main/index.ts - app ready 之前
app.commandLine.appendSwitch('disable-renderer-backgrounding');          // 禁止后台化渲染进程
app.commandLine.appendSwitch('disable-background-timer-throttling');     // 禁止后台定时器节流
app.commandLine.appendSwitch('disable-backgrounding-occluded-windows');  // 被遮挡窗口同样不后台化
```

三个都要加，**只加一个会漏掉部分场景**（前台切换、最小化、被其它窗口完全遮挡分别是不同路径）。

```typescript
// 每个账号视图创建时
const view = new WebContentsView({
  webPreferences: {
    backgroundThrottling: false,     // 窗口级禁用后台节流
    // ... 其余同 §3.2
  },
});
view.webContents.setBackgroundThrottling(false);   // 双保险，运行时再设一次
```

补充：`setVisible(false)` **不会**停掉 JS，只是不参与合成，省的是 GPU/合成开销，消息照收——这是隐藏后台账号的正确方式。

#### 3.3.3 v1.4 内存模型（保活优先）

| 状态 | 渲染进程 | 内存 | 实时收消息 | 说明 |
|---|---|---|---|---|
| **active**（当前可见） | 存活 | 100% | ✅ 实时 | 1 个 |
| **hidden**（后台保活） | 存活，仅 `setVisible(false)` | ~85–95% | ✅ 实时 | **默认状态，其余全部账号** |
| **sleep**（手动省电） | 已关闭 | ~0 | ❌ 不收 | **仅用户手动开启，默认关闭** |

**取消了 v1.3 的 frozen 级**：真正的"冻结"会暂停 JS 执行，与实时收消息直接冲突，没有折中空间，删掉以免误导。

**默认策略**：所有账号永久保活，不做任何自动销毁。仅 1 个 visible，其余 `setVisible(false)`。

#### 3.3.4 内存账目（要如实告诉用户）

保活的代价就是内存，这个账必须提前算给用户看，而不是等他挂 6 个号发现卡了：

| 账号数 | 估算常驻内存 | 说明 |
|---|---|---|
| 1 | ~300 MB | 基线（WhatsApp Web 本体 + Electron 开销） |
| 4 | ~1.0–1.4 GB | 可接受 |
| 8 | ~2.0–3.0 GB | 需 16GB 内存机器 |
| 12+ | 4 GB+ | 不推荐，需拆分为多实例或引导用户取舍 |

> 以上为量级估算，实际取决于 WhatsApp Web 自身内存占用与会话数量，**必须在真机实测后回填**。同时要在设置页显示"当前各账号实时内存占用"，让用户自己判断。

**⚠️ v1.8 B-2：这张表只算了内存，没算 CPU 和耗电——这是保活方案的真实缺口。**

`disable-renderer-backgrounding` + `backgroundThrottling:false` 的含义是：**8 个后台账号的 JS 全部全速运行**（WhatsApp Web 自身有轮询、心跳、会话列表重算、DOM 更新）。内存是静态占用，CPU 是持续开销，后者直接决定风扇、耗电和笔记本续航。产品说明里只报内存是不完整的。

| 指标 | 目标口径 | 数值 |
|---|---|---|
| 8 账号闲置 30 分钟的平均 CPU% | 真机实测，取 30 分钟均值与峰值 | **待 S3 回填** |
| 8 账号闲置 30 分钟的耗电 / 电池下降 | 笔记本真机、断开电源 | **待 S3 回填** |
| 1 / 4 / 8 账号的 RSS 常驻内存 | 真机实测，替换上表估算 | **待 S3 回填** |
| 后台账号 JS 定时器频率 | 用 CDP 采样 WhatsApp Web 的 timer / rAF | **待 S3 回填** |

**缓解方向（不破坏实时性，待 S3 数据验证后再决定是否实施）**：
1. 后台账号的译文渲染降频（已有，见下）
2. 注入层自身的 `setInterval` 在后台账号降到最低频（连接状态轮询 30s→60s，MutationObserver 保留）
3. 只给"当前可见 + 最近活跃 2 个"保留完整 DOM 观察，其余账号仅监听 `page-title-updated` 拿未读数（**注意：这会降低这些账号译文的即时性，须在设置里明示**）

> **此处不填数字。** 8 账号闲置 CPU 是多少、是否可接受，必须真机测出来才知道；凭感觉写"约 15%"会误导采购与产品决策。

**在不破坏实时性的前提下能省的内存**（有限，但值得做）：

1. `setVisible(false)` 省合成与 GPU 开销（不影响 JS）
2. `session.setCacheSize()` / `--disk-cache-size` 限制磁盘缓存
3. 隐藏视图的译文渲染降频（后台账号只做未读计数，不逐条渲染译文，切前台时再补渲染）
4. 限制同时预加载的账号数（启动时不一次性 load 全部，按需首次加载后即保活）

**做不到的事（说清楚）**：只要 WhatsApp Web 页面活着，它的 JS 堆就在。不销毁进程就省不掉这部分。这是物理下限。

#### 3.3.5 省电模式（可选，手动）

若用户机器内存紧张，可提供**手动开关**，而不是自动策略：

```typescript
// 用户在账号上右键 → "休眠此账号（将不再实时收消息）"
private async toSleep(id: string) {
  await this.persistSnapshot(id);               // 未读数、当前会话、草稿
  const v = this.get(id);
  mainWindow.contentView.removeChildView(v);     // 只摘视图树，不释放内存
  v.webContents.close();                         // ← 必须调，否则进程还在
  this.markSleeping(id);
}

async restore(id: string) {
  const view = new WebContentsView({ /* 同 §3.2 */ });
  mainWindow.contentView.addChildView(view);
  this.reorderOverlays();                        // 新视图在最上层，浮层需重排
  await view.webContents.loadURL(this.accountUrl(id));
  this.applyBounds(view);
  this.restoreSnapshot(id);
}
```

要点：
- **必须用户显式确认**，且开启时明确提示"该账号休眠期间不会实时收到消息"
- `removeChildView` ≠ 释放内存，必须配 `webContents.close()`
- 恢复后 z 序会乱，需重新把 UI 浮层 `addChildView` 置顶
- 休眠账号在侧边栏显示月亮图标，点击进入即唤醒（数秒）

#### 3.3.6 未读提醒在保活模型下怎么实现（R25）

页面活着就简单了，两条路：

1. **监听 `page-title-updated`**（最省）：WhatsApp Web 会把未读数写进 `document.title`（如 `(3) WhatsApp`），主进程直接监听该事件解析数字，零 DOM 操作、后台账号也准。
2. **注入层轻量上报**：注入脚本读取页面自身的未读徽标元素，变化时才通过 IPC 上报主进程（不要轮询）。

跨账号未读聚合 = 各账号未读数求和 + 角标。因为全部保活，**这两个数都是实时的**，不再需要 v1.3 里"重建后补拉"的兜底。

---

### 3.4 窗口缩放与多显示器 DPI 适配（R23，新增）

**问题**：WebContentsView 不在 DOM 里，无法靠 CSS 自适应，所有 bounds 都由主进程手动算。多显示器混合缩放（如笔记本 100% + 外接 4K 屏 200%）时，坐标换算与取整误差会导致黑边、错位、拖动时闪烁。

```typescript
class LayoutManager {
  applyBounds() {
    const [w, h] = mainWindow.getContentSize();     // 内容区尺寸，非窗口尺寸
    // v1.7.1：此处不读 getZoomFactor()——bounds 用 DIP，乘 scale 反而会错位

    // 关键：bounds 用 DIP（设备无关像素），不要乘 scale
    // 取整规则统一：起点向下取整，尺寸向上取整，避免累积缝隙
    const sidebar = Math.floor(SIDEBAR_W);
    const drawer  = this.drawerOpen ? Math.floor(DRAWER_W) : 0;

    const x = Math.floor(sidebar);
    const y = 0;
    const width  = Math.ceil(w - sidebar - drawer);
    const height = Math.ceil(h);

    this.chatView.setBounds({ x, y, width, height });
    this.overlayView.setBounds({ x: Math.ceil(w - drawer), y: 0, width: drawer, height });
  }
}
```

**必须覆盖的全部事件**（漏一个就会出现"某种操作后布局错乱"）：

```typescript
mainWindow.on('resize', () => layout.applyBounds());
mainWindow.on('maximize', () => layout.applyBounds());
mainWindow.on('unmaximize', () => layout.applyBounds());
mainWindow.on('restore', () => layout.applyBounds());
mainWindow.on('enter-full-screen', () => layout.applyBounds());
mainWindow.on('leave-full-screen', () => layout.applyBounds());
// 拖到另一块不同 DPI 的显示器时
mainWindow.on('moved', () => layout.applyBounds());
// 系统缩放比例变更（用户在设置里改了显示缩放）
screen.on('display-metrics-changed', () => layout.applyBounds());
screen.on('display-added', () => layout.applyBounds());
screen.on('display-removed', () => layout.applyBounds());
```

**实践要点**：
- **`resize` 里别做重活**：只更新 bounds，不做 DOM 扫描或数据库写入；必要时 rAF/微任务合并，避免拖动窗口时每帧抖动。
- **统一取整方向**：起点 `Math.floor`、尺寸 `Math.ceil`，避免相邻视图之间出现 1px 黑边。不要混用 `Math.round`。
- **全屏/最大化后要延迟一帧**：`maximize` 事件触发时窗口尺寸可能尚未最终确定，建议 `setTimeout(..., 0)` 或等下一次 `resize` 再应用。
- **DPI 变化后重新加载注入脚本的样式**：注入的译文气泡若用了固定 px，在 200% 缩放下会偏小；建议注入样式用相对单位或跟随页面根字号。
- 这块建议做成**可视化调试开关**：在诊断面板显示当前 `scaleFactor`、`getContentSize()`、各视图 bounds，实测对齐情况——多显示器问题几乎不可能靠"看代码"发现。

---

### 3.5 启动加载策略（R26，新增）

**问题**：§3.3 确立了"全部保活"，但没说**启动时怎么加载**。若配置 8 个账号，冷启动时同时 `loadURL` 8 个 WhatsApp Web，会出现：CPU 瞬间打满（8 个渲染进程同时解析 JS + 建 WebSocket）、内存峰值远超稳态（初始化阶段可达空闲态 2–3 倍）、带宽争抢拉取离线消息与媒体。用户感知就是"启动后卡死半分钟"，严重时直接 OOM。

Ferdium 的做法也是**按上次使用顺序依次加载**，并提供 "Delay loading of services" 开关——不是同时加载全部。

**分级懒加载**：

```typescript
interface StartupPolicy {
  immediate: string[];   // 第一梯队：上次 active 账号，立即加载（1 个）
  priority: string[];    // 第二梯队：最近活跃的 2–3 个，延迟 3s 起，间隔 2s
  deferred: string[];    // 第三梯队：其余，延迟 10s 起，间隔 3s
}

function buildStartupPolicy(accounts: Account[]): StartupPolicy {
  const sorted = [...accounts].sort((a, b) =>
    (b.lastActiveAt ?? 0) - (a.lastActiveAt ?? 0));       // 按最近活跃排序
  return {
    immediate: sorted.slice(0, 1).map(a => a.id),
    priority:  sorted.slice(1, 4).map(a => a.id),
    deferred:  sorted.slice(4).map(a => a.id),
  };
}

async function bootAccounts(accounts: Account[]) {
  const policy = buildStartupPolicy(accounts);
  const loaded = new Set<string>();

  const load = async (id: string) => {
    if (loaded.has(id)) return;
    loaded.add(id);
    try {
      await viewManager.create(id);          // 侧边栏该账号显示 loading
      sidebar.markLoaded(id);
    } catch (err) {
      // v1.5.1：加载失败不能静默吞掉，否则侧边栏永远显示"加载中"
      sidebar.markFailed(id, err);           // 侧边栏显示"加载失败，点击重试"
      loaded.delete(id);                     // 允许重试
    }
  };

  // v1.5.2：空账号列表守卫（immediate[0] 否则为 undefined）
  if (!accounts.length) return;
  await load(policy.immediate[0]);           // 首屏尽快可用
  setTimeout(() => policy.priority.forEach((id, i) =>
    setTimeout(() => load(id), i * 2000)), 3000);
  setTimeout(() => policy.deferred.forEach((id, i) =>
    setTimeout(() => load(id), i * 3000)), 10000);
}

// 用户点击未加载账号 → 插队立即加载
// v1.5.2：原写 loadNow（未定义），应为 load
sidebar.onAccountClick = (id) => load(id);
```

**必须接受的冲突（写进产品说明）**：第三梯队账号在启动后约 10–25 秒内尚未加载完成，**这段窗口内该账号收不到消息**。这是启动期的一次性窗口，加载完成后即进入全保活实时状态。侧边栏用转圈 + 灰显明确标识"加载中"，不要让用户误以为在线。

加载顺序依据 `accounts.last_active_at`（见 R33 迁移 v3 新增该字段；**INTEGER、Unix 毫秒**——v1.7.3 由原 `TEXT` 更正，否则本节的数字相减排序会得 `NaN`）。

---

### 3.6 渲染进程崩溃恢复（R27，新增）

**问题**：v1.4 只有一句"监听 `render-process-gone` 自动重建"。WhatsApp Web 渲染进程崩溃（GPU 异常、内存不足被 OS kill、Electron 自身 bug）在长时间挂机中是**必然事件**，不是边缘情况。没有完整设计就意味着崩溃后用户丢失会话、草稿，且可能陷入崩溃→重建→崩溃的死循环。

```typescript
interface CrashRecoveryState {
  accountId: string;
  crashCount: number;            // 连续崩溃次数
  lastCrashAt: number;
  reason?: string;               // 'crashed' | 'killed' | 'oom' | 'abnormal-exit'
  conversationId?: string;       // 崩溃前查看的会话
  scrollTop?: number;
  draftText?: string;
}
```

**① 周期快照**（不要更频繁，30s 是平衡点）：

```typescript
class CrashRecoveryManager {
  // v1.7.1 补充：稳定期定时器表（此前代码引用了 this.stabilityTimers 却未声明）
  private stabilityTimers = new Map<string, NodeJS.Timeout>();

  startPeriodicSnapshot() {
    setInterval(() => this.snapshotAll(), 30_000);
  }
  private async snapshotAll() {
    for (const [id, view] of this.viewManager.getAllViews()) {
      // v1.5.2：显式排除 sleep 账号（用户手动休眠，不该被快照"唤醒"或误判）
      if (this.viewManager.isSleeping(id)) continue;
      if (view.webContents.isDestroyed()) continue;
      const json = await view.webContents.executeJavaScript(
        `(function(){try{return JSON.stringify({
            convId: document.querySelector('[data-testid="conversation"]')?.dataset?.id,
            scroll: document.querySelector('[data-testid="conversation-panel-messages"]')?.scrollTop,
            draft: document.querySelector('div[contenteditable="true"]')?.textContent,
         })}catch(e){return null}})()`
      ).catch(() => null);
      if (json) this.merge(id, JSON.parse(json));
    }
  }
```

**② 崩溃处理（指数退避 + 熔断）**：

```typescript
  async handleCrash(accountId: string, reason: string) {
    const st = this.getState(accountId);
    st.crashCount++;
    st.reason = reason;                      // killed = OS OOM，需提示用户内存不足

    // ⚠️ v1.7.1 修正：先取消正在进行的稳定期计时
    // 否则崩溃重建后 5 分钟定时器仍会触发、把 crashCount 清零，削弱"连续 3 次熔断"
    const timer = this.stabilityTimers.get(accountId);
    if (timer !== undefined) {
      clearTimeout(timer);
      this.stabilityTimers.delete(accountId);
    }

    if (st.crashCount >= 3) {                // 熔断：否则死循环吃满 CPU
      this.notifyUI('account:crashed', {
        accountId, reason, canRetry: true,
        message: '该账号反复崩溃，已停止自动恢复，请手动重试或重启应用',
      });
      return;
    }
    // v1.7.1：crashCount >= 3 已在上面熔断返回，最大只会到 2 → delay 仅 1s / 4s
    const delay = st.crashCount * st.crashCount * 1000;   // 1s / 4s
    setTimeout(() => this.rebuild(accountId), delay);
  }
```

**③ 重建与状态恢复（v1.5 关键修正）**：

```typescript
  private async rebuild(accountId: string) {
    const st = this.getState(accountId);
    const view = this.viewManager.create(accountId);
    await view.webContents.loadURL(this.accountUrl(accountId));
    view.webContents.once('did-finish-load', async () => {
      await this.waitForLoginReady(view);                 // 等 WhatsApp 自身初始化
      if (st.conversationId) await this.navigateTo(view, st.conversationId);
      if (st.scrollTop) await this.scrollTo(view, st.scrollTop);

      // ⚠️ 修正：草稿不自动回填输入框，改为在预览条显示"上次的草稿：…（点击恢复）"
      // 原因：若崩溃发生在"已点发送但消息未确认"的时刻，自动回填 + 用户再次点发送 = 重复发送
      if (st.draftText) this.offerDraftRestore(accountId, st.draftText);

      // ⚠️ v1.5.1 修正：不在此处清零 crashCount，改为启动 5 分钟稳定期定时器
      // 原因：若页面恢复后立即再次崩溃，清零会导致熔断计数器失效，无法触发"连续 3 次熔断"
      this.startStabilityTimer(accountId);
    });
  }

  // 稳定运行 5 分钟后才清零崩溃计数
  private startStabilityTimer(accountId: string) {
    const timer = setTimeout(() => {
      this.getState(accountId).crashCount = 0;
      this.stabilityTimers.delete(accountId);
    }, 5 * 60 * 1000);
    // 若期间再次崩溃，handleCrash 会清除这个定时器
    this.stabilityTimers.set(accountId, timer);
  }
```

**其它要点**：
- `render-process-gone` 的 `reason` 必须记录：`killed`（OS 杀进程，通常内存不足）与 `crashed`（Chromium 内部异常）提示文案不同，前者要引导用户减少账号数。
- 崩溃计数在**稳定运行 5 分钟后**清零（由 `startStabilityTimer` 控制），避免历史崩溃累积导致误熔断。`handleCrash` 中须先 `clearTimeout(stabilityTimers.get(accountId))` 取消正在进行的稳定期计时（**v1.7.1 已落地**）。
- 恢复期间侧边栏该账号显示"恢复中"，不显示"在线"。

---

## 四、核心模块详细设计

### 4.1 多账号 Session 隔离与防关联（R9 补充）

**原理**：`session.fromPartition('persist:xxx')` 为每个账号分配独立 Cookie / LocalStorage / IndexedDB / 缓存；`ses.setProxy()` 为每个账号单独指定代理。

**partition 命名规范**：
- 必须以 `persist:` 开头，否则 session 是内存态，重启后全部掉线。
- 隔离键用**本地生成的稳定账号 ID**（`crypto.randomUUID()`），不要用用户名——用户名可能改、含特殊字符、跨平台重复。
- 建议格式：`persist:acct_${accountId}`，业务上再加一层授权范围前缀以防未来多租户复用 Cookie。

```typescript
interface Account {
  id: string;              // 本地生成的稳定 ID（uuid），非用户名
  name: string;            // 显示名，如"小号-亮"（v1.7.3 订正：原注释"WS亮"含义不明）
  platform: 'whatsapp' | 'telegram' | 'messenger';
  url: string;
  proxy?: string;          // socks5://127.0.0.1:7890
  partition: string;       // persist:acct_${id}
  userAgent?: string;
  enabled: boolean;
  lastActiveAt?: number;   // v1.7.3 补：上次活跃时间（Unix ms），对应 accounts.last_active_at，供 §3.5 分级懒加载排序
  sortOrder?: number;      // v1.7.3 补：侧边栏拖拽排序，对应 accounts.sort_order（§4.8.4）
}

class SessionManager {
  private sessions = new Map<string, Electron.Session>();

  getSession(account: Account): Electron.Session {
    let ses = this.sessions.get(account.id);
    if (!ses) {
      ses = session.fromPartition(account.partition);
      if (account.proxy) {
        ses.setProxy({ proxyRules: account.proxy });
      }
      this.disableWebRtcLeak(ses);
      this.sessions.set(account.id, ses);
    }
    return ses;
  }
}
```

**WebRTC 本地 IP 泄漏处理（R9，v1.1 新增）**：
只配 HTTP 层代理不够，WebRTC 的 ICE 候选会暴露真实本地 IP，代理等于白配。两层处理：

```typescript
// 1) 应用启动参数（app.commandLine.appendSwitch）
//    仅暴露公网接口 IP（即代理出口），隐藏内网 IP
app.commandLine.appendSwitch('force-webrtc-ip-handling-policy', 'default_public_interface_only');

// 2) 注入层兜底：在白名单 host 的页面里覆写 RTCPeerConnection，
//    阻断本地候选上报（需在页面脚本执行前注入）
```

> 注意：以上两项都需**实测验证**（用浏览器 WebRTC 泄漏检测页对照），不同 Electron/Chromium 版本行为可能变化。

**其他要点**：
- 每个账号可设独立 UA 与 `--lang`，配合代理做环境级区分
- 只给登录视图设 partition 不够：所有请求入口（注入脚本的 XHR、图片加载、上传）都必须走同一 session，否则出现"页面显示已登录、接口却未登录"
- 封装单一 `getSession(account)`，禁止业务代码自行拼 partition

### 4.2 主进程网络层（R1 修正）

**原理**：所有翻译请求不在页面上下文 fetch（会被 CSP 拦截），而是 `ipcRenderer.invoke` 发到主进程，由主进程用 Node 发出。

**R1 关键修正**：**删除 `rejectUnauthorized: false`**。聊天原文是商业内容，关闭证书校验等于对链路上任何人明文开放。腾讯云域名证书链正常，出现 `UNABLE_TO_VERIFY_LEAF_SIGNATURE` 通常是缺中间 CA 或走了公司代理，正解是补 CA：

```typescript
// app 启动早期（main/index.ts 顶部）
// 方式 A：指定企业/中间 CA 证书包
process.env.NODE_EXTRA_CA_CERTS = '/path/to/corp-ca-bundle.pem';

// 方式 B：显式传入（更精确，推荐）
import fs from 'node:fs';
const ca = [fs.readFileSync('/path/to/intermediate.pem')];
```

```typescript
// 主进程 - http-client.ts（v1.8：与 httpStream 共用错误类型与请求头）
import https from 'node:https';

// ⚠️ v1.8 C-2：结构化错误。429 退避、401/403 熔断、R19 自适应全部读 status，
// 禁止再用正则去解析 new Error("HTTP 429: ...") 的字符串。
class HttpError extends Error {
  constructor(
    public readonly status: number,
    public readonly body: string,
    public readonly retryable: boolean,
  ) { super(`HTTP ${status}: ${body.slice(0, 200)}`); }
  static from(status: number, body: string) {
    return new HttpError(status, body,
      status === 429 || status === 408 || status >= 500);   // 401/403 不重试，直接熔断
  }
}

function commonHeaders(h?: Record<string, string>): Record<string, string> {
  // ⚠️ v1.8 C-3：内部强制，不靠调用方传；漏传即 400 的坑在这里堵死
  return { 'Content-Type': 'application/json', ...h };
}

async function httpJson(url: string, options: {
  method?: string;
  headers?: Record<string, string>;
  body?: string;
  timeoutMs?: number;
  ca?: Buffer[];
  signal?: AbortSignal;            // ⚠️ v1.8 C-4：A1「取消翻译」要能中断在途请求
}): Promise<{ status: number; json: any }> {
  return new Promise((resolve, reject) => {
    const req = https.request(url, {
      method: options.method ?? 'GET',
      headers: commonHeaders(options.headers),
      ca: options.ca,               // 不再关闭校验；缺省走系统根证书
      timeout: options.timeoutMs ?? 8000,   // 注意：socket 空闲超时，非总时长
    }, (res) => {
      let data = '';
      res.setEncoding('utf8');
      res.on('data', (c) => (data += c));
      res.on('end', () => {
        if (res.statusCode && res.statusCode >= 400) {
          return reject(HttpError.from(res.statusCode, data));
        }
        try {
          resolve({ status: res.statusCode ?? 0, json: JSON.parse(data) });
        } catch {
          reject(new Error('响应非JSON: ' + data.slice(0, 200)));
        }
      });
    });
    req.on('timeout', () => req.destroy(new HttpError(408, 'timeout', true)));
    req.on('error', reject);

    // ⚠️ v1.8 C-5：结算时必须解绑，否则长跑进程每次请求都累积一个监听器
    const onAbort = () => req.destroy(new Error('已取消'));
    const cleanup = () => options.signal?.removeEventListener('abort', onAbort);
    options.signal?.addEventListener('abort', onAbort, { once: true });
    req.on('close', cleanup);

    if (options.body) req.write(options.body);
    req.end();
  });
}
```

**错误处置表（v1.8 C-2，供 §4.4 直接消费）**

| status | retryable | 处置 |
|---|---|---|
| 429 / 408 / 5xx | `true` | 退避重试 + `TranslateQueue.onThrottle()` 下调并发 |
| 401 / 403 | `false` | `KeyRotator.penalize()` 熔断换 Key，**不重试** |
| 400 / 422 | `false` | 参数问题，写诊断日志，提示用户（重试无意义） |

> `HttpError` 由 `httpJson` 与 `httpStream` 共用；§12.5 `call()` 捕获后统一交给调度层，引擎不自行判断重试。

#### 4.2.1 流式请求封装（v1.7.5 新增，A2 与路径② 共用的前置依赖）

**问题**：§12.3 A2 要求流式 SSE，且明令"禁止 `fetch`（undici 不接受 `ca`，会绕过 R1 证书修复）"；但此前文档只有一次性收齐的 `httpJson`，没有基于 `https.request` 的流式实现——**A2 与路径② 流式都落不了地**。这里补上唯一的一条出站管道。

```typescript
// 主进程 - http-client.ts
import { StringDecoder } from 'node:string_decoder';
import type http from 'node:http';

type StreamHandlers = {
  onDelta: (text: string) => void;
  onDone: (full: string) => void;
  onError: (e: Error) => void;
};

function httpStream(url: string, options: {
  headers?: Record<string, string>;
  body?: string;
  idleTimeoutMs?: number;     // v1.8：socket 空闲超时（心跳会不断重置它）
  deadlineMs?: number;        // v1.8：绝对总时长上限，不受心跳影响
  ca?: Buffer[];              // ← 与 httpJson 同一份 CA 配置，R1 不失效
  signal?: AbortSignal;
  requestId: string;          // v1.8：幂等键，用于计费去重与译文回滚
}, h: StreamHandlers): void {
  let req!: http.ClientRequest;

  // ⚠️ v1.8 C-1：Node 的 https.request({timeout}) 是【socket 空闲超时】，不是总时长。
  // 服务端只要持续发 `: ping` 心跳，这个计时器就被反复重置，流永不结束 →
  // 并发槽位被永久占用 → 整个 TranslateQueue 假死。必须另设绝对 deadline。
  const deadlineAt = Date.now() + (options.deadlineMs ?? 15_000);
  let deadlineTimer: NodeJS.Timeout | undefined;

  const onAbort = () => fail(new Error('已取消'));
  const cleanupAbort = () => options.signal?.removeEventListener('abort', onAbort);
  options.signal?.addEventListener('abort', onAbort, { once: true });

  // 单次结算守卫：idle timeout / deadline / abort / res error / req error / [DONE]
  // 可能同时触发，否则 onDone 与 onError 双调用，注入层出现"既成功又失败"的竞态
  let settled = false;
  const finish = (fn: () => void) => {
    if (settled) return;
    settled = true;
    if (deadlineTimer) clearTimeout(deadlineTimer);
    cleanupAbort();                    // ⚠️ v1.8 C-5：结算即解绑，避免长跑累积监听器
    fn();
  };
  const fail = (e: Error) => finish(() => { req?.destroy(); h.onError(e); });

  // 每次收到数据后按【绝对截止时间】重算剩余时间，心跳无法把它推后
  const armDeadline = () => {
    const remaining = Math.max(1, deadlineAt - Date.now());
    if (deadlineTimer) clearTimeout(deadlineTimer);
    deadlineTimer = setTimeout(
      () => fail(new HttpError(408, 'stream deadline exceeded', false)), remaining);
  };

  // ⚠️ v1.7.6：显式 StringDecoder 处理跨 chunk 的多字节 UTF-8 序列。
  // 禁止改成 Buffer 拼接或 chunk.toString('utf8') 逐块转换——一个汉字的 3 字节
  // 若被 TCP 分在两包里，逐块解码会得到两个 U+FFFD，译文直接乱码。
  const decoder = new StringDecoder('utf8');
  let carry = '';                 // 未成行的文本残留
  let eventLines: string[] = [];  // 当前 SSE 事件累积的行
  let full = '';

  const dispatchEvent = () => {
    if (!eventLines.length) return;
    // 标准 SSE：以【空行】为事件边界；同一事件内多个 data: 行按 \n 连接后再解析
    const dataParts = eventLines
      .filter(l => l.startsWith('data:'))
      .map(l => l.slice(5).replace(/^ /, ''));   // 去掉规范中可选的一个前导空格
    eventLines = [];
    if (!dataParts.length) return;
    const payload = dataParts.join('\n').trim();
    if (payload === '[DONE]') {
      finish(() => { req.destroy(); h.onDone(full); });   // 主动结束，不依赖连接关闭
      return;
    }
    try {
      const j = JSON.parse(payload);
      if (j?.error) { fail(new Error(`流式返回错误: ${JSON.stringify(j.error)}`)); return; }
      // 两种协议都兼容：chat 端点取 choices[0].delta.content；
      // 独立翻译接口若返回结构不同，在此处加一个归一化分支即可（§0 第 2 项）
      const d = j.choices?.[0]?.delta?.content ?? j.delta ?? j.text ?? '';
      if (d) { full += d; h.onDelta(d); }
    } catch { /* 注释行 / keep-alive / 非 JSON 心跳，安全跳过 */ }
  };

  const feed = (s: string) => {
    carry += s;
    const lines = carry.split('\n');
    carry = lines.pop() ?? '';                    // 保留不完整的尾行
    for (const raw of lines) {
      const line = raw.replace(/\r$/, '');        // 兼容 \r\n
      if (line === '') { dispatchEvent(); continue; }   // ← 空行分帧，不是按单行解析
      if (line.startsWith(':')) continue;               // 注释 / keep-alive
      eventLines.push(line);                            // event:/id:/retry:/data: 都收下
    }
  };

  armDeadline();                         // 请求发出前就启动绝对计时

  req = https.request(url, {
    method: 'POST',
    // ⚠️ v1.8 C-3：Content-Type 由 commonHeaders 强制，Accept 由流式覆盖
    headers: { ...commonHeaders(options.headers), Accept: 'text/event-stream' },
    ca: options.ca,
    timeout: options.idleTimeoutMs ?? 5_000,   // 仅 socket 空闲超时；总时长看 armDeadline
  }, (res) => {
    if (res.statusCode && res.statusCode >= 400) {
      const errDec = new StringDecoder('utf8');
      let err = '';
      res.on('data', (c: Buffer) => (err += errDec.write(c)));
      // ⚠️ v1.8 C-2：抛结构化 HttpError，调度层直接读 status 决定退避/熔断
      res.on('end', () => fail(HttpError.from(res.statusCode!, err)));
      return;
    }
    // 注意：这里【不要】再调用 res.setEncoding('utf8')。
    // setEncoding 与 StringDecoder 二选一，同时用会把 string 喂给 decoder.write 而报错。
    res.on('data', (chunk: Buffer) => { armDeadline(); feed(decoder.write(chunk)); });
    res.on('end', () => {
      const tail = decoder.end();          // 冲出末端不完整字节（若有）
      if (tail) feed(tail);
      if (carry.trim()) { eventLines.push(carry); carry = ''; }
      dispatchEvent();                     // 处理最后一个无空行结尾的事件
      finish(() => h.onDone(full));        // [DONE] 已结算则不再触发
    });
    res.on('error', fail);
  });
  req.on('timeout', () => fail(new HttpError(408, 'idle timeout', true)));
  req.on('error', fail);
  // 注意：abort 监听已在上面注册，由 finish() 统一解绑，此处不再重复 addEventListener
  if (options.body) req.write(options.body);
  req.end();
}
```

**要点（v1.7.6 修订）**：
- **UTF-8 边界**：`StringDecoder` 显式缓冲跨 chunk 的多字节序列。旧版靠 `res.setEncoding('utf8')` 隐式兜住，但一旦有人改成 Buffer 拼接 / undici `ReadableStream<Uint8Array>` / 逐块 `toString('utf8')`，汉字立刻变 U+FFFD。**三者只能选一个，且必须写进 code review 检查项**（用例 #63）。
- **SSE 按空行分帧，不是按单行 `\n`**：旧版要点写"按 `\n\n` 分块"、代码却逐行 `JSON.parse`，同一事件的多行 `data:` 会各自解析失败而**静默丢译文**。现按空行 dispatch、多行 data 用 `\n` 合并（用例 #64）。
- `[DONE]` 主动结算并 `req.destroy()`，不依赖服务端关闭连接；`:` 注释行、`\r\n`、非 JSON 心跳安全跳过。
- **单次结算守卫**：timeout / abort / res error / req error / `[DONE]` 可能并发触发，无 `settled` 会出现 onDone + onError 双调用。
- **两种协议共用一个解析器**，delta 取值路径不同（`delta.content` vs 独立接口字段，后者待 §0 第 2 项实测后补归一化分支）。
- 与 `httpJson` 共用 `ca`、`KeyRotator`、`TranslateQueue`——**不准为流式另起一套 HTTP 调用**（§12.5 缺口 3）。
- 流结束后才写缓存（A2 原结论，保留）；中途失败时注入层须清掉半截灰字（用例 #46）。
- **⚠️ v1.8 C-1：`timeout` 是 socket 空闲超时，不是总时长。** 服务端持续发 `: ping` 会不断重置它，流永不结束、并发槽位被永久占用、队列假死。绝对 deadline 独立计时、按 `deadlineAt - Date.now()` 重算剩余，心跳推不动（用例 #67）。

**流式重试与幂等边界（v1.8 C-6，新增）**

§4.4 的"失败按 1s/2s/4s 退避，最多 2 次重试"**只能用于非流式**。流式若照搬，会出现两处事故：已吐出半截译文后又追加一份（气泡里出现重复文本），以及同一段原文被计费两次。规则：

| 规则 | 定义 |
|---|---|
| **默认不自动重试** | 流式中途失败 → 清掉半截灰字 → 显示"翻译失败 · 点击重试"，由用户决定是否重来（与 §4.4「失败可见」一致） |
| **用户主动重试** | 复用同一 `requestId`，先 `clearPartial(requestId)` 清空该节点已渲染的增量，再发起；绝不追加 |
| **计费去重** | 主进程按 `requestId` 记录本次是否已向引擎发出请求；同一 id 的手动重试走新请求但**诊断日志标注为重试**，用量看板（E1）不去重显示但给出提示 |
| **超时判定** | 触发 deadline 的请求按 `HttpError(408, retryable=false)` 结算，不进自动退避队列 |
| **上层联动** | 429（可重试）由 `HttpError.retryable` 交给调度层；401/403 直接熔断换 Key，均不涉及译文回滚 |

> 简言之：**非流式 = 系统自动重试；流式 = 用户点一下重来，且先把上一段擦干净。**

**⚠️ v1.8 B-1：术语命中筛选规则（原方案只写"子串匹配 + ≤20 条"，规则不足）**

| 项 | 规则 |
|---|---|
| 匹配方式 | 先做**词边界匹配**（`\b` / Unicode 分词），子串匹配仅作为型号类术语（`X-200`、`MOQ-500`）的补充；避免 "LED" 命中 "LEDer" 这类误伤 |
| 大小写 | 默认忽略大小写；术语可标 `case_sensitive`（型号/缩写建议开启） |
| 单次注入上限 | **20 条**，按术语优先级（客户级 > 全局）排序后截取 |
| **超限丢弃策略** | 超过 20 条时：① 保留前 20；② 丢弃的条数写诊断日志；③ 气泡不提示用户（不打扰），但设置页"术语表"显示"本次有 N 条未注入"；④ 用户可把被丢的条目标为 `pinned` 强制进入前 20 |
| 与上下文预算的关系 | 20 条术语 + 3 轮上下文 + 原文，合计控制在 **2k 字符内**（模型输入 4k 上限），不会顶爆 8k；**上百条术语全量注入是禁止的** |

### 4.3 翻译引擎抽象层（R12 修订）

**接口**：

```typescript
interface TranslatorProvider {
  readonly id: string;                    // 'hunyuan' | 'deepl' | 'google'
  readonly name: string;
  readonly models: string[];              // 可选档位
  translate(text: string, opts: {
    from?: string;      // 'auto'
    to: string;         // 'zh-CN'
    model?: string;     // 'hy-mt2-lite' | 'hy-mt2-pro'
    glossary?: GlossaryEntry[];  // 术语表
    signal?: AbortSignal;
  }): Promise<string | null>;
}

interface GlossaryEntry { src: string; dst: string; note?: string; }
```

**模型选档（R12）**：TokenHub 上翻译模型实际有三档，上下文 8k / 最大输出 4k：

| model 参数 | 定位 | 本产品用法 |
|---|---|---|
| `hy-mt2-lite` | 对耗时有高要求的场景 | **聊天实时翻译默认档** |
| `hy-mt2-plus` | 有优秀的指令遵循能力 | 备选默认 |
| `hy-mt2-pro` | 旗舰，专业领域高质量 | **"这段话翻得不准，点此精翻"手动档** |

**术语表（R12，外贸核心差异点）**：Hy-MT2 支持术语表翻译与上下文翻译。把产品名、型号、规格、公司名做成术语表投喂，前后文译名一致性会明显提升——这比"多引擎切换"更能拉开与竞品的差距。

```json
{
  "model": "hy-mt2-lite",
  "messages": [
    { "role": "system", "content": "你是一个专业的外贸翻译助手。把用户发来的消息翻译成中文简体，只输出翻译结果，不要任何解释。若原文已是中文则直接返回原文。术语表：<glossary>内术语须按给定译法，不得另译。" },
    { "role": "user", "content": "原文文本" }
  ],
  "temperature": 0.3,
  "max_tokens": 1000
}
```

> **术语传参两条路（v1.7.3 更正：原文"当前以 prompt 注入实现"与 §12.1 结论冲突）**：① **默认**走 `/v1/chat/completions`，把术语拼进 prompt（三档模型通用）；② **需工程化术语管理时，切换到 TokenHub 独立翻译接口**（`POST /v1/api/translations`）+ `glossary_ids` 引用持久化术语库（**仅 OpenAPI 协议支持**，见 §12.1 问题 1）。结构化术语字段**现已提供，不再是"后续能力"**；两条路的实际效果都需在真实话术上回归验证。

**合规提示（R14）**：DeepL / Google 属境外服务，把客户聊天内容送出去构成商业信息出境。默认引擎设为混元（境内），界面在用户首次启用境外引擎时弹出明确告知并需确认。

#### 4.3.1 语种检测与跳过规则（R36，新增）

`from: 'auto'` 不能只靠引擎自动判断。外贸场景里短句（"ok"、"thanks"、"👍"）、中英混排（"Hello，请问这个 price 是多少"）、罗马化阿拉伯语/印地语都很常见，语种误判会直接导致译文质量崩掉（例如把葡萄牙语当西班牙语翻）。

```typescript
function shouldTranslate(text: string): { skip: boolean; reason?: string } {
  const t = text.trim();
  if (!t) return { skip: true, reason: 'empty' };
  if (t.length < 2) return { skip: true, reason: 'too_short' };       // 单字符/单 emoji
  if (/^[\p{Extended_Pictographic}\s\u200d\ufe0f]+$/u.test(t))
    return { skip: true, reason: 'emoji_only' };                       // 👍👍👍
  if (/^[\d\s.,%$€£¥+-]+$/.test(t))
    return { skip: true, reason: 'number_only' };                      // 报价数字
  if (/^https?:\/\/\S+$/.test(t)) return { skip: true, reason: 'url_only' };
  return { skip: false };
}
```

- **短句（< 10 字符）**：仍可翻译，但不缓存判定、且结果标记为"低置信"，UI 上不显示 🤖 前缀以免误导。
- **已含中文**：若原文中文占比 > 60%，跳过翻译直接显示。
- **检测优先级**：混元返回的 `detected_language` 字段（若有） > 本地轻量检测（`tinyld` / `franc`，< 100KB，纯本地零延迟） > `auto`。
- 跳过规则同时是**省钱机制**：emoji 与纯数字在外贸聊天里占比不低，全部送去翻译纯属浪费额度。

#### 4.3.2 翻译质量后处理（R35，采纳 + 收敛）

引擎返回什么就信什么是有风险的：可能返回空串、原文、语种误判、术语未按表翻译。

```typescript
class TranslationPostProcessor {
  process(original: string, translated: string, o: {
    from: string; to: string; glossary: GlossaryEntry[];
  }): { text: string; warnings: string[] } {
    const w: string[] = [];
    if (!translated?.trim()) { w.push('empty'); return { text: original, warnings: w }; }

    if (translated.trim() === original.trim() && !['zh-CN','zh-TW','ja','ko'].includes(o.from))
      w.push('identical');                             // 可能没翻

    for (const g of o.glossary) {
      if (original.toLowerCase().includes(g.src.toLowerCase()) &&
          !translated.includes(g.dst)) w.push(`glossary_miss:${g.src}`);
    }
    const ratio = translated.length / original.length;
    if (ratio < 0.1 || ratio > 10) w.push(`length_ratio:${ratio.toFixed(2)}`);

    return { text: translated, warnings: w };
  }
}
```

**收敛说明（我对评审方案的调整）**：这些检测**误报率不低**——原文与译文相同在"客户用中文问"时完全正常，长度比异常在"中文→英文"时本就常见。因此：

- 警告**只写诊断日志**，绝不打断用户、不弹窗、不阻止发送
- 唯一自动降级的是**空译文**（回退显示原文，避免气泡空白）
- 聚合统计（"最近 100 次翻译中 12 次术语未命中"）显示在设置面板，作为用户调整术语表/换引擎的参考

### 4.4 翻译调度：队列 / 限流 / 熔断 / 缓存（R4、R5，v1.1 新增章节）

这是 v1.0 完全缺失、但决定"能不能扛住真实聊天"的一层。

```typescript
// 1) Key 轮换 + 熔断
class KeyRotator {
  private idx = 0;
  private cooldown = new Map<string, number>();

  constructor(private keys: string[]) {}

  next(): string | null {
    const now = Date.now();
    const avail = this.keys.filter(k => (this.cooldown.get(k) ?? 0) <= now);
    if (!avail.length) return null;                 // 全部熔断
    const k = avail[this.idx % avail.length];
    this.idx++;
    return k;
  }

  penalize(key: string, ms = 60_000) {              // 401/403/连续失败
    this.cooldown.set(key, Date.now() + ms);
  }
}

// 2) 并发队列：MutationObserver 在刷屏时会瞬间打出几十个请求
// v1.7.5：并发值改为可动态调整，与 R19 自适应一致（原先 readonly 4 是写死的）
// ⚠️ v1.8 C-7：补【优先级】与【背压】。原先是纯 FIFO + 并发数——
// 当前正在聊的会话与后台账号同时来消息时谁先翻全靠运气，恰是体验最差的地方。
type Priority = 0 | 1 | 2;   // 0=当前可见会话 1=同账号其它会话 2=后台账号
interface Job {
  key: string;                // 文本去重键（同文本合并，不重复请求）
  priority: Priority;
  accountId: string;
  run: () => Promise<void>;
  enqueuedAt: number;
}

class TranslateQueue {
  private queue: Job[] = [];
  private active = 0;
  private maxConcurrent: number;
  private readonly min: number;
  private readonly ceiling: number;
  private stableSince = Date.now();

  private readonly MAX_QUEUE = 200;       // 背压上限
  private readonly MAX_AGE_MS = 30_000;   // 排队超 30s 的旧任务直接作废

  constructor(initial = 4, min = 1, ceiling = 8) {
    this.maxConcurrent = initial;
    this.min = min;
    this.ceiling = ceiling;   // 上限以 §0 第 4 项实测为准，不写死 5
  }

  // 收到 429 / 连续超时 → 立即砍半，并记录不稳定
  onThrottle() {
    this.maxConcurrent = Math.max(this.min, Math.floor(this.maxConcurrent / 2));
    this.stableSince = Date.now();
    diagnostics.set('translate.concurrency', this.maxConcurrent);
  }

  // 稳定成功一段时间（建议 60s）→ 缓慢回升 +1，避免刚恢复又撞限流
  onStable() {
    if (Date.now() - this.stableSince < 60_000) return;
    this.maxConcurrent = Math.min(this.ceiling, this.maxConcurrent + 1);
    this.stableSince = Date.now();
    diagnostics.set('translate.concurrency', this.maxConcurrent);
  }

  push(job: Job) {
    // 同文本合并：已在队列里就直接丢弃（结果会由先到的那个任务写入共享缓存）
    if (this.queue.some(j => j.key === job.key)) return;
    this.queue.push(job);
    this.applyBackpressure();
    this.pump();
  }

  private applyBackpressure() {
    this.queue = this.queue.filter(j => {
      if (Date.now() - j.enqueuedAt < this.MAX_AGE_MS) return true;
      diagnostics.inc('translate.dropped.stale');
      return false;
    });
    if (this.queue.length <= this.MAX_QUEUE) return;
    // 超限：优先丢【后台账号】的旧任务，当前可见会话的任务绝不丢
    this.queue.sort((a, b) => b.priority - a.priority || b.enqueuedAt - a.enqueuedAt);
    const dropped = this.queue.splice(this.MAX_QUEUE);
    diagnostics.inc('translate.dropped.backpressure', dropped.length);
  }

  private pump() {
    // 优先级升序 → 同优先级按入队时间（FIFO，不饿死）
    this.queue.sort((a, b) => a.priority - b.priority || a.enqueuedAt - b.enqueuedAt);
    while (this.active < this.maxConcurrent && this.queue.length) {
      const job = this.queue.shift()!;
      this.active++;
      job.run().finally(() => { this.active--; this.pump(); });
    }
  }
}

// 优先级判定：主进程知道哪个 accountId 当前 visible
function priorityOf(accountId: string, isCurrentConversation: boolean): Priority {
  if (isCurrentConversation) return 0;
  return viewManager.isActive(accountId) ? 1 : 2;
}
```

**调度规则**：
- **合并**：MutationObserver 回调先做 rAF 节流，把同一批新增节点合并成一次扫描；相同文本天然命中缓存。
- **优先级（v1.8 C-7）**：当前可见会话 `P0` > 同账号其它会话 `P1` > 后台账号 `P2`；同级按入队时间 FIFO，避免低优先级饿死。用户切账号时，在途任务不改优先级，新任务按新状态入队。
- **背压（v1.8 C-7）**：队列上限 200、任务存活 30s；超限时优先丢后台账号的旧任务，**当前可见会话的任务绝不丢**；丢弃数进诊断面板。没有这条，8 账号刷屏会把队列堆到几千条、内存与 API 额度一起爆。
- **限流**：并发走 R19 自适应（初始 4、上限 8，**以实测为准**，见 §12.1 补充更正）；单条超时 8s；**仅非流式**按 1s / 2s / 4s 指数退避，最多 2 次重试（流式的重试规则见 §4.2.1，不自动重试）。
- **熔断**：`HttpError.status` 为 401/403 立即 `penalize` 并换下一个 Key；全部熔断时向 UI 推送 `translate:quotaExhausted`。
- **失败可见**：译文位置渲染"翻译失败 · 点击重试"，而不是什么都不显示——否则用户只会反馈"不翻译了"，无法自助排查。
- **队列等待 P95**：待 S2/S3 真机实测后回填（§14.2）。没有这个数，"实时翻译"无法验收。

**去重与缓存（R4）**：
```typescript
// 内存去重改 LRU，防止无限增长
class LruSet {
  constructor(private max = 3000) {}
  private map = new Map<string, true>();
  // v1.7.3：命中也要提升新鲜度。原实现只有 add() 才把 key 移到末尾，
  // 读命中（最常见路径）不更新访问序 → 热 key 仍会被冷 add 挤掉，实为"插入序 + 写时升级"的伪 LRU。
  has(k: string) {
    if (!this.map.has(k)) return false;
    this.map.delete(k); this.map.set(k, true);   // 移到末尾
    return true;
  }
  add(k: string) {
    this.map.delete(k); this.map.set(k, true);
    if (this.map.size > this.max) {
      this.map.delete(this.map.keys().next().value!);
    }
  }
}
```

**去重与缓存 · 高频话术"秒回"（R18，新增）**

聊天翻译里真正影响体感的是"发出去那一刻的等待"。把缓存从"省钱的副产品"升级为"时延优化的主路径"：

1. **命中计数**：缓存表加 `hit_count`，每次命中自增，作为热度排序依据。
2. **启动预热**：应用启动时把 `hit_count` Top N（建议 2000）载入内存 Map，首屏高频话术**零网络往返**，直接渲染。
3. **话术库预翻译**：`quick_replies` 存的是中文话术，可在保存时（或空闲时段）预先翻译成常用目标语言写入缓存。用户点话术按钮 → 译文**立即可用**，不等 API。
4. **入站高频兜底**：外贸常见问句（"how much"、"MOQ"、"shipping cost"）重复率极高，缓存命中率会显著高于一般场景，预热收益明显。

**缓存 key 与术语表一致性（R29 + R40，修正）**

v1.4 的 key 是 `md5(engine|model|from|to|text)`，**不含术语表版本**——用户把 "LED" 的译法从"发光二极管"改成"LED灯"后，已缓存的译文仍是旧的，且要等 90 天才过期。

修正方案：**key 混入术语表版本号**，且**按 scope 拆分版本号**（这是我对评审方案的修正——单值全局版本号会让任何一次客户级术语表改动导致全部缓存失效）：

```typescript
// 版本号分两级：全局 + 客户级
// settings: glossary_version_global = '1'；glossary 表每行带 updated_at
function glossaryScopeKey(scope: string): string {
  if (scope === 'global') return `g${globalGlossaryVersion()}`;   // v1.7.1：改用带兜底的取值函数
  // 客户级：用该客户术语表最近更新时间做版本，避免影响全局缓存
  return `c${scope}:${glossaryLastUpdated(scope)}`;
}

function cacheKey(p: {
  engine: string; model: string; from: string; to: string;
  text: string; scope: string;
}): string {
  return md5(`${p.engine}|${p.model}|${p.from}|${p.to}|${glossaryScopeKey(p.scope)}|${p.text}`);
}
```

```sql
-- v1.5.2 修正：原写法在该 key 不存在时影响 0 行，版本号永远递增不了，
-- 导致 glossaryScopeKey('global') 恒为 "gundefined"，R29 完全失效。
-- ① 初始化 seed（放进迁移 v2，保证一定存在）
INSERT OR IGNORE INTO settings (key, value) VALUES ('glossary_version_global', '1');

-- ② 递增必须用 UPSERT，不能只 UPDATE
INSERT INTO settings (key, value)
VALUES ('glossary_version_global',
        CAST(COALESCE((SELECT CAST(value AS INTEGER) FROM settings
                       WHERE key = 'glossary_version_global'), 0) + 1 AS TEXT))
ON CONFLICT(key) DO UPDATE
SET value = CAST(CAST(settings.value AS INTEGER) + 1 AS TEXT);
```

**取值侧也要兜底**（避免任何情况下返回 undefined 导致 key 恒定）：

```typescript
function globalGlossaryVersion(): string {
  const v = settings.get('glossary_version_global');
  return /^\d+$/.test(v ?? '') ? v! : '1';    // 缺失或非法 → 回落 '1'
}
```

- 旧缓存不做批量删除（浪费），版本号变了自然不再命中，靠 90 天策略自然过期。
- **跨账号共享**：`scope = global` 的缓存天然跨账号共享（key 不含 account_id），符合预期；`scope = customer:<peer_id>` 的缓存带 scope 前缀，不冲突。

```sql
-- 缓存 key 必须带 engine，否则切换引擎后拿到旧译文
-- cache_key = md5(engine | model | from | to | <glossaryScope> | text)
ALTER TABLE translation_cache ADD COLUMN hit_count INTEGER DEFAULT 0;
CREATE INDEX idx_cache_hot ON translation_cache(hit_count DESC);

-- 启动预热：取最热的一批进内存
SELECT cache_key, result FROM translation_cache
ORDER BY hit_count DESC LIMIT 2000;

-- 清理策略：保留最近 90 天 / 上限 50000 条，启动时与每日首次执行
-- 注意：清理时排除高热度条目，避免热词被时间策略误删
DELETE FROM translation_cache
WHERE hit_count < 10
  AND (created_at < datetime('now', '-90 days')
       OR cache_key NOT IN (SELECT cache_key FROM translation_cache
                            ORDER BY created_at DESC LIMIT 50000));
```

**自适应并发（R19，新增）**：混元的 QPS 上限目前未知，固定并发为 4 可能触发限流、也可能过于保守。改为动态调节：初始值 4；收到 429 或连续超时则减半（下限 1）；稳定成功一段时间后缓慢回升（上限 8）。当前值显示在诊断面板，便于据此反推可用 QPS。

### 4.5 IPC 类型安全落地（R11）

v1.0 只列了通道清单，没有落地机制。三处约束形成闭环：

```typescript
// shared/types.ts - 唯一真源
export interface DelaiAPI {
  translate(text: string, to?: string, model?: string): Promise<string | null>;
  getAccounts(): Promise<Account[]>;
  setEngine(engineId: string): Promise<void>;
  quickReply: {
    list(): Promise<QuickReply[]>;
    send(text: string): Promise<void>;
  };
  customer: {
    get(key: CustomerKey): Promise<Customer | null>;
    save(data: Customer): Promise<void>;
  };
  onTranslateResult(cb: (r: TranslateResult) => void): () => void;
}
```

```typescript
// preload/index.ts - 编译期约束实现完整性
const api = { /* ... */ } satisfies DelaiAPI;
contextBridge.exposeInMainWorld('delai', api);

// renderer/src/env.d.ts - 渲染进程拿到类型
declare global {
  interface Window { delai: DelaiAPI }
}
```

**主进程侧必须做入参校验**：渲染进程不可信（用户可能装了第三方插件、或未来开放了外部调用），每个 handler 入参用 zod schema 校验后再用，避免脏数据写坏 SQLite。

#### 4.5.1 IPC 安全加固（R22，新增）

`contextIsolation: true` 只解决了"页面拿不到 Node"，**不代表渲染进程可信**。三条防线：

**① 最小权限暴露**

preload 只暴露业务所需的方法，**绝不暴露**：通用 `ipcRenderer.invoke(channel, ...)`（等于开放任意通道调用）、`ipcRenderer.on` 原始监听、任何能传 channel 名的函数、fs/path/child_process 相关能力。

```typescript
// ❌ 禁止：等于把整个 IPC 面暴露给页面
contextBridge.exposeInMainWorld('delai', {
  invoke: (ch: string, ...args: any[]) => ipcRenderer.invoke(ch, ...args),
});

// ✅ 正确：逐方法白名单，通道名硬编码在 preload 内
const api = {
  translate: (text: string, to?: string, model?: string) =>
    ipcRenderer.invoke('translate:text', { text, to, model }),
  // ...
} satisfies DelaiAPI;
```

**② 每个 handler 逐通道 Zod 校验**

```typescript
import { z } from 'zod';

const TranslateReq = z.object({
  text: z.string().min(1).max(4000),          // 上限防止大文本打爆 API
  to: z.string().regex(/^[a-z]{2}(-[A-Z]{2})?$/).default('zh-CN'),
  model: z.enum(['hy-mt2-lite', 'hy-mt2-plus', 'hy-mt2-pro']).optional(),
});

ipcMain.handle('translate:text', async (_e, raw) => {
  const req = TranslateReq.parse(raw);         // 失败直接抛，不落库、不发请求
  return translateSvc.translate(req);
});

const CustomerSave = z.object({
  platform: z.enum(['whatsapp', 'telegram', 'messenger']),
  account_id: z.string().uuid(),
  peer_id: z.string().min(1).max(128),
  display_name: z.string().max(200).optional(),
  notes: z.string().max(5000).optional(),
  tags: z.array(z.string().max(50)).max(50).optional(),
});
```

要点：字符串一律设**长度上限**（防止超长文本写爆 SQLite / 打爆 API）；枚举用 `z.enum` 收窄到白名单；`uuid()` 约束账号 ID；不信任任何来自渲染进程的 `id`/路径/文件路径参数。

**③ 渲染进程侧 XSS 防护**

翻译结果、客户备注、话术内容最终都要插进 DOM（含注入到页面里的译文气泡）。规则：

- 注入层渲染译文一律用 **`textContent`，禁用 `innerHTML`**（v1.0 示例中的 `el.innerHTML` 必须改掉）
- 渲染进程 UI 中若确需富文本，先做白名单消毒（DOMPurify）
- 客户名、话术标题等用户输入在列表中展示时走 Vue 的 `{{ }}` 插值（自动转义），不用 `v-html`

```javascript
// 注入层：必须用 textContent
el.textContent = `🤖 ${result}`;
```

**补充**：`webFrame.executeJavaScript` 注入的脚本运行在页面上下文，页面自身若存在 XSS 也可能反向读取注入脚本的变量。注入层的临时变量应放在闭包内，不挂在 `window` 上；`window.delai` 仅在白名单 host 内挂载（见 §4.6.1）。

#### 4.5.2 IPC 速率限制（R31，新增）

Zod 只校验了**格式**，没限制**频率**。异常或页面被 XSS 后可在一秒内发出数千次 `translate:text`，打爆队列并耗尽 API 额度——这是"能花钱的漏洞"，必须堵。

```typescript
class IpcRateLimiter {
  private buckets = new Map<number, { count: number; resetAt: number }>();

  check(senderId: number, limit = 20, windowMs = 1000): boolean {
    const now = Date.now();
    let b = this.buckets.get(senderId);
    if (!b || now > b.resetAt) {
      b = { count: 0, resetAt: now + windowMs };
      this.buckets.set(senderId, b);
    }
    b.count++;
    return b.count <= limit;
  }
}

ipcMain.handle('translate:text', async (event, raw) => {
  if (!rateLimiter.check(event.sender.id)) {
    log.warn('IPC 速率超限', { sender: event.sender.id });
    throw new Error('翻译请求过于频繁，请稍后再试');
  }
  const req = TranslateReq.parse(raw);
  return translateSvc.translate(req);
});
```

要点：
- 按 `event.sender.id`（渲染进程）分桶，而非按账号——一个页面异常不该影响其它账号。
- 限流**在 Zod 校验之前**执行，避免为超限请求付出解析成本。
- 限流阈值写入诊断面板；正常聊天的翻译频率远低于 20 次/秒，误伤概率极低。
- `settings` 写入类通道单独设更严格的阈值（如 5 次/秒）。

### 4.6 注入脚本（R2、R3、R8、R15）

#### 4.6.1 host 白名单与导航拦截（R8，新增）

v1.0 每次加载完成都注入，用户点聊天里的链接后脚本会跑进任意第三方站点，且 `window.delai` 一起暴露。

```typescript
// 主进程 - 视图层
const ALLOWED_HOSTS = new Set(['web.whatsapp.com', 'web.telegram.org']);

view.webContents.on('did-finish-load', () => {
  const host = new URL(view.webContents.getURL()).host;
  if (!ALLOWED_HOSTS.has(host)) return;          // 不注入
  view.webContents.executeJavaScript(injectScript);
});

// 外链走系统浏览器，不在应用内打开
view.webContents.setWindowOpenHandler(({ url }) => {
  if (!ALLOWED_HOSTS.has(new URL(url).host)) {
    shell.openExternal(url);
    return { action: 'deny' };
  }
  return { action: 'allow' };
});
view.webContents.on('will-navigate', (e, url) => {
  if (!ALLOWED_HOSTS.has(new URL(url).host)) { e.preventDefault(); shell.openExternal(url); }
});
```

#### 4.6.2 译文渲染适配虚拟列表（R3，修正）

WhatsApp Web 是虚拟滚动：切会话/滚动时消息节点销毁重建。v1.0 的"同文本只翻一次 + 译文挂 DOM"组合会导致——翻过一次后不再翻，而译文随节点销毁一起丢失。

```javascript
// 1) 节流 + 全量重扫（不做"只处理新增"，避免虚拟列表复用节点漏判）
let scheduled = false;
const observer = new MutationObserver(() => {
  if (scheduled) return;
  scheduled = true;
  requestAnimationFrame(() => { scheduled = false; scanAll(); });
});
observer.observe(document.body, { childList: true, subtree: true });

// 2) 节点级标记：用 WeakMap 记录已处理节点 + data 属性做二次兜底
const processed = new WeakMap();
function scanAll() {
  document.querySelectorAll(SEL.msgContainer).forEach(node => {
    if (processed.get(node) === node.dataset.delaiSig) return;   // 已处理且签名未变
    handle(node);
  });
}

// 3) 文本签名（内容 + 方向），DOM 里留 data-delai-sig 便于自检面板排查
function sig(node) {
  return hash(directionOf(node) + '|' + extractText(node));
}

// 4) 渲染：原文下方绿色气泡，已存在则更新而非重复插入
function renderTranslation(container, text, state /* ok|loading|error */) {
  let el = container.querySelector(':scope > .delai-translation');
  if (!el) {
    el = document.createElement('div');
    el.className = 'delai-translation';
    container.appendChild(el);
  }
  el.dataset.state = state;
  el.textContent = state === 'ok' ? `🤖 ${text}` : (state === 'error' ? '翻译失败 · 点击重试' : '翻译中…');
  if (state === 'error') el.onclick = () => retry(container);
}
```

**去重策略（R4）**：`translatedKeys` 用 LRU（上限 3000），SQLite 缓存做跨会话兜底；节点销毁后重扫时命中缓存直接渲染，不重复消耗额度。

#### 4.6.3 出站翻译：三层降级写入（R2 + R16）

WhatsApp Web 输入框是 React 受控的 `contenteditable`。**直接 `inputEl.innerText = text` 不会更新编辑器内部 state，点发送时发出去的仍是旧内容（或空）**——这是第一阶段验收标准里最可能翻车的一条。

**核心原则（决定成败）**：编辑器（Draft.js / Lexical / Slate）的 'input' 处理器是**读 DOM 实际内容再与内部 state 做 diff** 的。因此——

> **只派发 `input` 事件而 DOM 内容未变，等于空转**：处理器读到的内容与 state 一致，判定无变化，state 不更新。
> 必须**先真实修改 DOM，再派发事件**。顺序不可颠倒。

```javascript
// preload/inject/input-writer.ts

const NBSP = /\u00a0/g;

function currentValue(el) {
  return ((el.isContentEditable ? el.innerText : el.value) || '')
    .replace(NBSP, ' ').trim();
}

function verify(el, text) {
  return currentValue(el) === text.replace(NBSP, ' ').trim();
}

// caret 必须落在元素内，否则 execCommand 静默返回 false
function placeCaret(el, atEnd = true) {
  const range = document.createRange();
  range.selectNodeContents(el);
  range.collapse(!atEnd);              // true = 折叠到末尾
  const sel = window.getSelection();
  sel.removeAllRanges();
  sel.addRange(range);
}

// 派发事件：必须带 inputType/data，且 bubbles
// （React 17+ 把监听器挂在 root container 上，不冒泡的事件收不到）
function dispatchInput(el, data, inputType = 'insertText') {
  el.dispatchEvent(new InputEvent('input', {
    bubbles: true, cancelable: true, composed: true,
    inputType, data,
  }));
}
```

**第一层：`execCommand('insertText')`**（默认路径）

```javascript
function writeViaExecCommand(el, text) {
  el.focus();
  placeCaret(el, false);                            // ⚠️ v1.7.1 订正：collapse 后只剩光标（落开头），并非"全选"
  document.execCommand('delete');                   // 清空同样要走 execCommand（能否真清空须真机实测）
  const ok = document.execCommand('insertText', false, text);
  if (!ok) return false;                            // ← 必须检查返回值
  dispatchInput(el, text);                          // 双保险：已触发也无害
  return verify(el, text);
}
```

**第二层：真实写 DOM + 派发**（execCommand 失败 / 被弃用时）

`contenteditable` 用 Range 插入文本节点；`input` / `textarea` 必须绕过 React 的 `_valueTracker`——React 在受控值未变时会拦截 onChange，直接赋 `el.value` 不生效，需走原型上的原生 setter：

```javascript
function writeViaDom(el, text) {
  el.focus();

  if (el.isContentEditable) {
    const range = document.createRange();
    range.selectNodeContents(el);
    range.deleteContents();
    const node = document.createTextNode(text);
    range.insertNode(node);
    range.setStartAfter(node);
    range.collapse(true);
    const sel = window.getSelection();
    sel.removeAllRanges();
    sel.addRange(range);
  } else {
    // 绕过 React _valueTracker：用原型上的原生 setter 赋值
    const proto = el instanceof HTMLTextAreaElement
      ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
    Object.getOwnPropertyDescriptor(proto, 'value').set.call(el, text);
  }

  dispatchInput(el, text);
  return verify(el, text);
}
```

**第三层：paste 事件兜底**（前两层都失败时）

```javascript
function writeViaPaste(el, text) {
  el.focus();
  placeCaret(el, false);
  document.execCommand('delete');
  const dt = new DataTransfer();
  dt.setData('text/plain', text);
  el.dispatchEvent(new ClipboardEvent('paste', {
    bubbles: true, cancelable: true, clipboardData: dt,
  }));
  return verify(el, text);
}
```

**编排与发送时序**：

```javascript
async function fillInput(el, text) {
  const ok = writeViaExecCommand(el, text)
          || writeViaDom(el, text)
          || writeViaPaste(el, text);
  if (!ok) throw new Error('出站写入失败：三层方案均未生效');

  // 编辑器的 state 更新与 React re-render 未必同步完成，
  // 校验通过后至少等两帧再触发发送，否则可能发出去空内容
  await new Promise(r => requestAnimationFrame(() => requestAnimationFrame(r)));
  if (!verify(el, text)) throw new Error('写入后校验失效（可能被编辑器回滚）');

  // v1.5.2：发送按钮必须走四档降级链，不能硬编码单个 data-testid
  // 否则前端一改版就退化到模拟 Enter（更脆），等于降级链在最关键一步失效
  const sendBtn = resolve(PLATFORM, 'sendButton');   // v1.7.1：resolve 现为 (platform, key)
  if (sendBtn?.el) {
    diag.record('sendButton', sendBtn.level);     // 诊断面板显示实际命中档位
    sendBtn.el.click();
    return;
  }
  log.warn('发送按钮选择器全部失效，回退模拟 Enter');
  el.dispatchEvent(new KeyboardEvent('keydown', {
    key: 'Enter', code: 'Enter', keyCode: 13, which: 13,
    bubbles: true, cancelable: true,
  }));
}
```

**交互流程保持"预览确认"**：
1. 用户输入中文 → debounce 800ms → 输入框下方出现翻译预览条
2. 点"发送译文" → `fillInput` → verify → 点击发送
3. 不自动拦截发送；任何一层失败都明确提示"自动填入失败，已复制到剪贴板，请手动粘贴"

```javascript
// 仅当含中文才触发（v1.7.1：变量名由未定义的 cn 订正为待发送文本 text）
if (!/[\u4e00-\u9fa5]/.test(text)) return;
```

> **必须实测**：以上三层均基于浏览器行为与编辑器（Draft.js / Lexical）的事件处理原理推导，不同平台与 Electron/Chromium 版本行为可能不同。Day 2 需在真实 WhatsApp Web 上逐层验证，记录哪一层实际生效，并把结论写回本节。

#### 4.6.4 选择器四档降级链（R17，重构）

**问题**：`._ak1r`（WhatsApp）、`.bq4j9vrk`（Messenger）这类编译期生成的混淆类名，每次前端发版就可能整体重排，是这类工具维护成本的主要来源。

**对策**：不再依赖单一选择器，而是为每类目标定义**按稳定性排序的降级链**，运行时逐档尝试，命中即停并记录档位。

| 档位 | 手段 | 稳定性 | 说明 |
|---|---|---|---|
| L1 | `data-testid` / `data-*` 语义属性 | 高 | 官方测试钩子，产品自身 QA 依赖，改动成本高于样式类名 |
| L2 | **ARIA 语义属性**：`div[role="row"]`、`[role="textbox"]`、`[aria-label]` | 高 | 无障碍属性受规范约束，且改动会导致可访问性回归，比类名更抗改版 |
| L3 | **相对树遍历**：从稳定锚点出发的 parent/child/sibling 相对路径 | 中 | 不依赖任何类名，靠结构定位；结构变更时会失效，但通常晚于类名 |
| L4 | 混淆类名（`._ak1r` 等） | 低 | **最后手段**，命中即标记为高风险并在诊断面板告警 |

```typescript
// shared/selectors.ts - 每个目标是一组候选，而非一个字符串
export const SELECTORS = {
  whatsapp: {
    msgContainer: [
      { level: 'L1', sel: 'div[data-testid="conversation-panel"]' },
      { level: 'L2', sel: 'div[role="row"]' },
      { level: 'L3', sel: 'ANCESTOR_OF(#main, 2) > div[role="list"] > div' },
      { level: 'L4', sel: '._ak1r' },
    ],
    msgText: [
      { level: 'L1', sel: 'span.selectable-text' },
      { level: 'L2', sel: 'div[role="row"] span[dir]' },
      { level: 'L4', sel: '._ao3e span' },
    ],
    // v1.5.2 新增：发送按钮（§4.6.3 出站流程依赖，原先硬编码在代码里）
    sendButton: [
      { level: 'L1', sel: 'button[data-testid="compose-btn-send"]' },
      { level: 'L2', sel: 'button[aria-label="Send"]' },
      { level: 'L2', sel: 'span[data-icon="send"]' },
      { level: 'L3', sel: 'FOOTER_SIBLING_OF(input)' },
      { level: 'L4', sel: '._ak1q' },
    ],
    input: [
      { level: 'L1', sel: 'div[contenteditable="true"]' },
      { level: 'L2', sel: 'div[role="textbox"][aria-label]' },
      { level: 'L4', sel: '._ak1l' },
    ],
    // v1.5.1 新增：连接状态指示（供 §4.6.6 连接健康监控使用）
    connStatus: [
      { level: 'L1', sel: '[data-testid="alert-banner"]' },
      { level: 'L2', sel: 'div[role="alert"]' },
      { level: 'L3', sel: '#main ~ header' },
      { level: 'L4', sel: '._2ts2m' },                   // 需实测确认
    ],
  },
  telegram: {
    msgContainer: [
      { level: 'L1', sel: '.MessageList' },
      { level: 'L2', sel: 'div[role="list"]' },
    ],
    msgText: [
      { level: 'L1', sel: '.text-content' },
      { level: 'L2', sel: '.message span[dir="auto"]' },
    ],
    input: [
      { level: 'L1', sel: 'div[contenteditable="true"].input-message-field' },
      { level: 'L2', sel: 'div[role="textbox"]' },
    ],
    // Telegram 的连接状态指示与 WhatsApp 完全不同（顶部细长条）
    connStatus: [
      { level: 'L1', sel: '.connecting' },
      { level: 'L2', sel: '#header-content .status' },
      { level: 'L4', sel: '.transition' },               // 需实测确认
    ],
  },
  // messenger 同构（延后实现）
} as const;

// 运行时解析：返回命中的档位，供诊断面板展示
// v1.7.1 修正：原签名只有 key，函数体却引用未定义的 platform（运行必抛 ReferenceError）
export function resolve(platform: keyof typeof SELECTORS, key: string): { el: Element | null; level: string | null } {
  for (const { level, sel } of SELECTORS[platform][key]) {
    const el = document.querySelector(sel);
    if (el) return { el, level };
  }
  return { el: null, level: null };   // 全部失效 → 诊断面板报"选择器全部失效"
}
```

**平台优先级**：WhatsApp（P0）> Telegram（P1）> Messenger（P2，混淆类名最脆、反自动化最强，建议延后或不做）。

选择器配置文件支持**远端热更新**：不发版即可修正失效档位，是这类工具能否长期存活的关键能力。

> **L3 相对树遍历**需要为每类目标写一个小的定位函数（如"从 `#main` 上溯两级再取其 role=list 子节点"），比纯 CSS 选择器代码量稍大，但完全不依赖类名，是 L1/L2 同时失效时的真正保命层。

#### 4.6.5 选择器诊断面板（R15 + R17，新增）

平台一改版，用户反馈只能是"不能翻译了"，排查全靠猜。加一个诊断页，**第一行就回答"当前靠哪一档选择器在撑着"**：

| 显示项 | 作用 |
|---|---|
| 每类目标的**实际命中档位**（L1/L2/L3/L4） | 掉到 L4 即黄色告警：说明 L1–L3 已全部失效，随时可能彻底挂掉 |
| 各类目标命中数量 | 容器 / 文本 / 输入框分别命中几个节点，为 0 的直接标红 |
| 最近一次提取的文本样本 | 判断是"没找到"还是"找到了但内容错" |
| 出站写入**实际生效的层级** | execCommand / DOM / paste，失败则显示三层全挂 |
| 当前 host 是否命中白名单 | 排查"为什么没注入" |
| 注入脚本版本 + 选择器配置版本 | 确认是否加载了最新配置 |

支持一键复制全部诊断信息（用户直接发给你，无需反复追问）。这是这类工具后期维护成本的大头，第一天就要留。

**自动化预警**：启动时若 `resolve()` 落到 L4 或返回 null，主动弹一次提示并写入日志——不要等用户来报"不能翻译了"才发现。

#### 4.6.6 连接健康监控（R32，采纳 + 技术修正）

**问题**：WebSocket 长时间运行后可能**静默断连**（网络抖动、代理超时、平台侧断开），页面不一定报错——表现为"账号看着在线，实际收不到消息"。这是保活模型下最危险的失效模式：用户以为在收，其实没在收。

**⚠️ 对评审方案的技术修正（v1.5.2 恢复双理由）**：原建议用 `window.WebSocket = class extends origWS {...}` 做 Hook。这里存在**两个彼此独立**的问题，缺一不可，不是二选一：

| # | 问题 | 说明 |
|---|---|---|
| **A** | **世界隔离**（若 Hook 写在 preload） | `contextIsolation: true` 下 preload 运行在 isolated world，它的 `window` 与页面主世界**不是同一个全局对象**，改 isolated world 的 `WebSocket` 对页面创建的连接毫无影响 |
| **B** | **注入时序**（若 Hook 写在注入脚本） | 注入脚本用 `executeJavaScript` 在主世界执行（能避开 A），但它在 `did-finish-load` 才执行，此时页面的 WS **早已建立**；Hook 只能拦截之后新建的连接，覆盖不到已有的那条 |

> v1.5.1 把理由"改为时序问题"，等于删掉了 A。但 A 是真实存在的——评审原文建议的就是在 preload 里 Hook，那条路径必然失效。保留两条才能解释为什么方案 B 必须**同时**满足两个条件：注入到主世界（解决 A）+ 在文档创建时注入（解决 B）。

结论不变：改用 DOM 状态轮询（方案 A）最稳。

可行路径按优先级：

| 方案 | 做法 | 评价 |
|---|---|---|
| **A（推荐）** | 轮询页面自身的连接状态指示元素（WhatsApp 顶部的"正在连接 / 手机已断开"横幅），命中即上报 | 无需 Hook，选择器走 §4.6.4 四档降级链即可，最省最稳 |
| **B** | CDP `Page.addScriptToEvaluateOnNewDocument`（`webContents.debugger.attach()`）在页面脚本前注入到主世界，Hook WS | 真正能 Hook 到，但需常驻 debugger，有性能与稳定性代价 |
| **C** | 心跳探测：注入层记录最后一次收到新消息节点的时间，超阈值上报 `stale` | 只能发现"长时间没消息"，无法区分"断连"和"客户没说话"，**仅作辅助** |

```javascript
// 方案 A：注入层轮询状态指示（30s 一次）
setInterval(() => {
  const banner = resolve(PLATFORM, 'connStatus');  // v1.7.1：四档降级链，resolve(platform, key)
  const text = banner?.el?.textContent?.trim();
  if (text && /connecting|disconnected|连接中|已断开/i.test(text)) {
    window.delai?.reportConn({ state: 'down', text });
  } else {
    window.delai?.reportConn({ state: 'up' });
  }
}, 30_000);
```

主进程处置分级：
1. `down` 持续 < 30s → 侧边栏图标转黄，不打扰用户（多数是抖动，会自愈）
2. `down` 持续 > 5 分钟 → 自动 `webContents.reload()`
3. 重载后 2 次仍 down → 红色告警 + 系统通知"账号 X 连接异常，请检查网络/代理"

**不要一发现 down 就立即 reload**——网络抖动时频繁重载会把账号刷下线，比不处理更糟。

#### 4.6.7 选择器配置的版本控制与回滚（R30，新增）

**问题**：v1.4 说选择器支持远端热更新，但没有版本控制与回滚——推一个有 bug 的配置，所有用户同时挂掉且无法快速回退。更要命的是：选择器配置是**注入到页面里执行的**，被篡改等同于 RCE。

```typescript
interface SelectorConfig {
  version: string;        // semver
  minAppVersion: string;  // 最低兼容应用版本
  publishedAt: string;
  checksum: string;       // sha256(selectors)，防篡改
  selectors: typeof SELECTORS;
}

class SelectorUpdater {
  private current: SelectorConfig;
  private previous: SelectorConfig | null = null;   // 保留上一版用于回滚

  async checkUpdate(): Promise<SelectorConfig | null> {
    const latest = await this.fetchLatest();
    if (!latest) return null;
    if (sha256(JSON.stringify(latest.selectors)) !== latest.checksum) {
      log.warn('选择器配置校验和不符，拒绝应用');
      return null;
    }
    if (semver.lt(APP_VERSION, latest.minAppVersion)) {
      log.warn(`选择器 ${latest.version} 要求应用 ≥ ${latest.minAppVersion}`);
      return null;
    }
    return latest;
  }

  async apply(cfg: SelectorConfig) {
    this.previous = this.current;
    this.current = cfg;
    await this.persist(cfg);                       // 本地持久化，下次启动直接用
    this.viewManager.broadcastSelectorUpdate(cfg.version);
  }

  rollback(): boolean {
    if (!this.previous) return false;
    [this.current, this.previous] = [this.previous, this.current];
    this.viewManager.broadcastSelectorUpdate(this.current.version);
    return true;
  }
}
```

要点：
- **校验和必做**，且仅接受 HTTPS + 固定域名下发；发布端私钥签名更好（校验和只能防传输篡改，签名能防源被控）。
- 保留 `previous`，诊断面板提供"回滚到上一版"按钮。
- `minAppVersion` 防止旧版应用加载不兼容配置。
- 灰度：先推 10% 客户端，观察诊断上报的命中档位无异常后再全量（见 §11 待确认 12）。

### 4.7 数据持久化（R10、R4 修订）

#### 4.7.1 并发写入加固（R21，新增）

**问题**：本产品的写入源是并发的——翻译缓存写入（刷屏时高频）、客户档案保存、快捷回复编辑、设置变更、诊断日志，全部打向同一个 SQLite 文件。默认配置下并发写会抛 `SQLITE_BUSY`，且默认 `journal_mode=DELETE` 时读写互相阻塞，表现为"聊着聊着某个操作报错"甚至数据库损坏。

**必开的 PRAGMA（better-sqlite3）**：

```typescript
import Database from 'better-sqlite3';

const db = new Database(path.join(app.getPath('userData'), 'delai.db'));

db.pragma('journal_mode = WAL');        // 读写不互斥，并发能力质变
db.pragma('synchronous = NORMAL');      // WAL 下安全性与性能的最佳平衡
db.pragma('busy_timeout = 5000');       // 遇锁等待 5s 而非立即抛 SQLITE_BUSY
db.pragma('foreign_keys = ON');         // 外键约束（默认关闭）
db.pragma('wal_autocheckpoint = 1000'); // 控制 WAL 文件大小，避免无限增长
```

**各参数取舍说明**：

| PRAGMA | 作用 | 为什么这样设 |
|---|---|---|
| `journal_mode = WAL` | 读写并发 | 核心项。rollback journal 模式下写会阻塞所有读，聊天场景不可接受 |
| `synchronous = NORMAL` | 落盘策略 | WAL 模式下 `NORMAL` 在断电/崩溃时不会损坏数据库（仅可能丢失最后几笔事务）；`FULL` 每次写都 fsync，高频缓存写入时性能损失明显 |
| `busy_timeout = 5000` | 锁等待 | 不设则并发瞬间抛错；设过长会卡住主进程。5s 是经验值 |
| `wal_autocheckpoint` | WAL 回收 | 不设则 `-wal` 文件可能涨到很大；定期 checkpoint 回收 |

**写操作串行化**：`busy_timeout` 只能缓解，不能根治。更好做法是**所有写操作走单一写队列**，天然串行：

```typescript
class DbWriter {
  private chain: Promise<unknown> = Promise.resolve();

  // 所有写操作排队执行，彻底消除并发写冲突
  run<T>(fn: () => T): Promise<T> {
    const next = this.chain.then(fn, fn);   // 前序失败不阻断后续
    this.chain = next.catch(() => {});
    return next;
  }
}
// 读操作保持同步直连（better-sqlite3 的读是同步的，WAL 下不阻塞写）
```

**SQLITE_BUSY 兜底重试**：即使有队列，仍需处理（例如用户用其它工具打开了 db 文件）：

```typescript
function withBusyRetry<T>(fn: () => T, max = 3): T {
  let lastErr: any;
  for (let i = 0; i < max; i++) {
    try { return fn(); }
    catch (e: any) {
      if (!/SQLITE_BUSY|SQLITE_LOCKED/.test(e?.code ?? '')) throw e;
      lastErr = e;
      sleepSync(50 * (i + 1));            // 50 / 100 / 150ms
    }
  }
  throw lastErr;
}
```

**其它要点**：
- **只开一个连接**：不要每个模块各自 `new Database()`，全应用单例，避免连接间锁竞争。
- **缓存写入合并**：翻译结果不要每来一条就立即写库，攒批（如 100 条或 5 秒）一次性 `INSERT OR REPLACE`，显著降低写压力。
- **关闭时 checkpoint**：`app.on('before-quit')` 执行 `db.pragma('wal_checkpoint(TRUNCATE)')`，保证干净退出。
- **备份**：SQLite 文件在 `%APPDATA%/delai-hub/` 下，提供"导出数据"按钮（复制 db 文件即可，WAL 模式下需先 checkpoint）。
- 若采用 `node:sqlite` 零编译方案，同样需要执行上述 PRAGMA，API 形态不同但参数一致。

#### 4.7.2 API Key 加密存储（R28，新增）

v1.4 只写了"设置表 `hunyuanKeys(加密JSON)`"和"第三阶段用 safeStorage"，没有实现细节。API Key 是付费资源，明文存 SQLite 等于裸奔——任何能读 `%APPDATA%` 的程序都能拿走。

```typescript
class KeyVault {
  // v1.5.2：safeStorage 在 app ready 前调用会抛错（迁移/初始化阶段容易踩到）
  private async ensureReady(): Promise<void> {
    if (!app.isReady()) await app.whenReady();
  }

  async saveKeys(engineId: string, keys: string[]) {
    await this.ensureReady();
    if (!safeStorage.isEncryptionAvailable()) {
      // v1.5.2 统一口径：默认拒绝明文写入，不允许"静默降级"
      if (!settings.get('allow_plaintext_keys')) {
        notifyUI('keys:noEncryption', { engineId });
        throw new Error('当前系统不支持加密存储，Key 未保存');
      }
      log.warn('已启用明文 Key 存储（用户显式授权）');
      db.prepare('INSERT OR REPLACE INTO settings (key, value) VALUES (?, ?)')
        .run(`keys_${engineId}`, JSON.stringify(keys));
      return;
    }
    const enc = safeStorage.encryptString(JSON.stringify(keys));
    db.prepare('INSERT OR REPLACE INTO settings (key, value) VALUES (?, ?)')
      .run(`keys_${engineId}`, enc.toString('base64'));
  }

  async loadKeys(engineId: string): Promise<string[]> {
    await this.ensureReady();
    const row = db.prepare('SELECT value FROM settings WHERE key = ?')
      .get(`keys_${engineId}`);
    if (!row) return [];
    if (!safeStorage.isEncryptionAvailable()) {
      try { return JSON.parse(row.value); }          // 明文分支
      catch { return []; }
    }
    try {
      return JSON.parse(
        safeStorage.decryptString(Buffer.from(row.value, 'base64')));
    } catch {
      notifyUI('keys:decryptFailed', { engineId });   // 换了机器/重装系统
      return [];
    }
  }
}
```

**必须提前告知用户的三个限制**：
1. `safeStorage` 基于 OS 凭据存储（Windows 凭据管理器 / macOS Keychain / Linux libsecret），**密文绑定当前用户 + 当前机器**。换机器、重装系统、换用户后无法解密，只能重新录入 Key。
2. **导出数据不包含 Key**，"备份/迁移"功能导出的 db 里 Key 字段要么剔除要么无法解密。
3. Linux 依赖 `libsecret`/`kwallet`，缺失时 `isEncryptionAvailable()` 返回 false。

> **v1.5.2 统一口径（原文档此处自相矛盾）**：代码写的是 `throw` 拒绝保存，文字却说"会降级为明文"。二者必须一致，本版取**更安全的一种**：默认拒绝明文写入，仅在用户在设置里显式勾选"允许明文存储 API Key（不安全）"后才写入，并在启动时持续提示。理由是——用户往往不知道"降级明文"意味着什么，静默降级等于把付费 Key 暴露给任何能读 `%APPDATA%` 的程序。

补充：**添加 Key 时立即做一次有效性探测**（发一个最短的翻译请求），不要等到实际用的时候才发现 Key 是错的。

#### 4.7.3 数据库迁移框架（R33，新增）

v1.4 只给了建表 SQL，没有迁移机制。上线后必然要改表结构，裸写 `ALTER TABLE` 没有版本追踪，用户升级后容易出现"表结构不对"。用极简迁移框架：`schema_migrations` 表 + 有序 migration 数组。

```typescript
interface Migration { version: number; name: string; up: (db: Database) => void }

const migrations: Migration[] = [
  { version: 1, name: 'initial_schema', up: (db) => db.exec(/* 全部建表 SQL */) },
  { version: 2, name: 'cache_key_add_glossary_scope', up: (db) => {
      db.exec('DELETE FROM translation_cache');        // key 算法变更，旧缓存作废
      // v1.5.2：必须 seed 版本号，否则 §4.4 的递增语句永不生效（R29 形同虚设）
      db.exec(`INSERT OR IGNORE INTO settings (key, value)
               VALUES ('glossary_version_global', '1')`);
    } },
  // v1.7.3：last_active_at 存 Unix 毫秒时间戳（INTEGER）。原为 TEXT（ISO 串），
  // 而 §3.5 启动排序按 `(b.lastActiveAt ?? 0) - (a.lastActiveAt ?? 0)` 做数字相减，
  // 字符串相减得 NaN，排序失效、分级懒加载顺序错乱。
  { version: 3, name: 'add_account_last_active', up: (db) =>
      addColumnIfMissing(db, 'accounts', 'last_active_at', 'INTEGER') },
  { version: 4, name: 'add_audit_log', up: (db) => db.exec(/* audit_log 建表 */) },
  // v1.7.3 新增 v5：v1.6 功能优化项落地所需的新列与新表。
  // v1.5.2 的 migrations 已占用 v1–v4；本批新表为 v5（v1_7_feature_tables），
  // v1.7.5 的术语库映射列/索引为 v6（glossary_library_mapping）。
  { version: 5, name: 'v1_7_feature_tables', up: (db) => {
      // A4 纠错闭环：标记用户纠正过的缓存条目，清理策略不得删除（见 §12.2）
      addColumnIfMissing(db, 'translation_cache', 'user_corrected', 'INTEGER DEFAULT 0');
      // E2 翻译历史：src_text 可空且默认不写，遵守 §八"不留存聊天记录"
      db.exec(`CREATE TABLE IF NOT EXISTS translation_history (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        dst_text TEXT NOT NULL,          -- 只存译文
        src_text TEXT,                   -- 客户原文；可空，默认 NULL（除非用户显式开启"保存原文"）
        from_lang TEXT, to_lang TEXT,
        engine TEXT, model TEXT,
        created_at TEXT DEFAULT (datetime('now'))
      )`);
      db.exec('CREATE INDEX IF NOT EXISTS idx_th_created ON translation_history(created_at)');
      // B1 统一收件箱：只聚合"元数据"，正文摘要默认不写（见 §12.3 B1 第 3 点）
      db.exec(`CREATE TABLE IF NOT EXISTS conversations (
        platform TEXT NOT NULL, account_id TEXT NOT NULL, peer_id TEXT NOT NULL,
        display_name TEXT,
        unread_count INTEGER DEFAULT 0,
        last_msg_preview TEXT,           -- 正文摘要；默认 NULL（设置项关闭时不写）
        last_active_at INTEGER,          -- Unix ms
        PRIMARY KEY (platform, account_id, peer_id)
      )`);
      // v1.7.5：统一收件箱按 last_active_at 排序、按 (platform, account_id) 查某账号会话
      db.exec('CREATE INDEX IF NOT EXISTS idx_conv_active ON conversations(last_active_at DESC)');
      db.exec('CREATE INDEX IF NOT EXISTS idx_conv_account ON conversations(platform, account_id)');
    } },
  // v1.7.5 新增 v6：术语库建模补完所需的列与索引。
  // 必须放 v6 而非改 v1：老用户的 v1 早已执行过，改 v1 会导致升级后缺列。
  { version: 6, name: 'glossary_library_mapping', up: (db) => {
      addColumnIfMissing(db, 'glossary', 'glossary_id', 'TEXT');
      db.exec('CREATE INDEX IF NOT EXISTS idx_glossary_gid ON glossary(glossary_id)');
    } },
];

// ⚠️ v1.7.6：SQLite 的 ALTER TABLE ADD COLUMN 【没有 IF NOT EXISTS】。
// 同一个 up() 里混用幂等的 CREATE ... IF NOT EXISTS 与非幂等的 ALTER，
// 一旦重跑（手动重跑 / schema_migrations 记录丢失 / 曾在事务外执行过），
// 就会报 `duplicate column name: xxx` → 迁移失败 → 应用起不来。
function addColumnIfMissing(db: Database, table: string, column: string, decl: string) {
  // PRAGMA 不支持参数绑定表名；table/column 均来自上面的代码常量，非用户输入
  const cols = db.prepare(`PRAGMA table_info(${table})`).all() as Array<{ name: string }>;
  if (cols.some(c => c.name === column)) {
    log.warn(`列已存在，跳过: ${table}.${column}`);
    return;
  }
  db.exec(`ALTER TABLE ${table} ADD COLUMN ${column} ${decl}`);
}

function runMigrations(db: Database) {
  backupDbFile();                 // v1.7.6：迁移前自动备份 .bak（救命措施，显式调用）
  db.exec(`CREATE TABLE IF NOT EXISTS schema_migrations (
    version INTEGER PRIMARY KEY, name TEXT NOT NULL,
    applied_at TEXT DEFAULT (datetime('now')))`);
  const applied = new Set(db.prepare('SELECT version FROM schema_migrations')
    .all().map((r: any) => r.version));
  for (const m of migrations.filter(m => !applied.has(m.version))
                            .sort((a, b) => a.version - b.version)) {
    db.transaction(() => {
      m.up(db);
      db.prepare('INSERT INTO schema_migrations (version,name) VALUES (?,?)')
        .run(m.version, m.name);
    })();
    log.info(`Migration ${m.version} (${m.name}) applied`);
  }
  // schema 变更会让已 prepare 的语句失效（SQLITE_SCHEMA）
  db.pragma('optimize');
  invalidateStatementCache();     // 重新 prepare 常驻语句，或统一改为按需 prepare
}
```

**要点（v1.7.6 补：SQLite ALTER 的硬限制与启动顺序）**

1. **幂等**：`ALTER TABLE ADD COLUMN` 没有 `IF NOT EXISTS`，所有加列一律走 `addColumnIfMissing()`（`PRAGMA table_info` 探测）。`CREATE TABLE/INDEX IF NOT EXISTS` 可保留。这样任何迁移重跑都安全（用例 #65）。
2. **ADD COLUMN 硬限制（未来迁移必读）**：
   - 不能加 `PRIMARY KEY` / `UNIQUE` 列
   - 不能加 `NOT NULL` 且无**恒定**默认值的列
   - 默认值不能是 `CURRENT_TIME` / `CURRENT_DATE` / `CURRENT_TIMESTAMP`，不能是表达式或子查询
   - 不能加带 `REFERENCES` 的列（`foreign_keys=ON` 且默认值非 NULL 时）
   - 一条语句只能加一列，不支持 `ADD COLUMN a, ADD COLUMN b`
   - 本方案三处加列都是「可空 / 常量默认值」，**合规**
   - 若将来确需 UNIQUE 或 NOT NULL 新列，必须走「建新表 → 拷数据 → 删旧表 → 改名」的 12 步流程，且整体包在事务里
3. **启动顺序硬约束**：`runMigrations()` 必须在**任何业务查询之前**完成。尤其 §4.4 的缓存清理 SQL `WHERE user_corrected = 0 ...` 若在 v5 执行前跑，会直接 `no such column: user_corrected`。启动序列固定为：打开 db → PRAGMA（WAL 等）→ `runMigrations()` → 缓存预热 → 启动清理定时器 → 创建视图。
4. **WAL 下 ALTER 需要 schema 写锁**：若「导出/备份」功能开了第二个连接读同一 db 文件，迁移期间可能 `SQLITE_LOCKED`。迁移阶段只允许单连接，备份用**文件复制**而非第二个连接（与 §4.7.1「只开一个连接」一致）。
5. **prepared statement 失效**：schema 变更后模块级缓存的 `db.prepare()` 结果会报 `SQLITE_SCHEMA`。迁移后必须 `invalidateStatementCache()`，或干脆不做模块级 statement 缓存。
6. 每个迁移跑在**事务**里（失败整体回滚）；`schema_migrations` 可追溯；启动时自动执行，无需用户操作。

---

```sql
-- 账号表
CREATE TABLE accounts (
  id TEXT PRIMARY KEY,            -- uuid，非用户名
  name TEXT NOT NULL,
  platform TEXT NOT NULL,         -- whatsapp | telegram | messenger
  url TEXT NOT NULL,
  proxy TEXT,
  partition TEXT UNIQUE NOT NULL, -- persist:acct_${id}
  user_agent TEXT,
  enabled INTEGER DEFAULT 1,
  sort_order INTEGER DEFAULT 0
);

-- 客户档案表（R10：主键改为复合键）
-- 原因：WhatsApp 有群组、Telegram 是 username、Messenger 是 PSID，
--      用 phone 单键会导致接 Telegram 时数据迁移
CREATE TABLE customers (
  platform TEXT NOT NULL,         -- whatsapp | telegram | messenger
  account_id TEXT NOT NULL,       -- 我方账号（同一个客户可能由不同账号接待）
  peer_id TEXT NOT NULL,          -- 对方号码 / username / PSID / 群 ID
  display_name TEXT,
  phone TEXT,                     -- 冗余，便于检索
  country TEXT,
  city TEXT,
  language TEXT,
  tags TEXT,                      -- 需求标签（JSON 数组）
  notes TEXT,
  last_progress TEXT,
  last_touch_at TEXT,
  created_at TEXT DEFAULT (datetime('now')),
  PRIMARY KEY (platform, account_id, peer_id)
);
CREATE INDEX idx_customers_phone ON customers(phone);

-- 快捷回复表
CREATE TABLE quick_replies (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  lang TEXT DEFAULT 'en',
  sort_order INTEGER DEFAULT 0
);

-- 翻译缓存表（R4：key 带 engine + model；R18：加 hit_count 支持预热）
CREATE TABLE translation_cache (
  cache_key TEXT PRIMARY KEY,     -- md5(engine|model|from|to|glossaryScope|text)
  result TEXT NOT NULL,
  hit_count INTEGER DEFAULT 0,    -- 命中次数，用于热度排序与启动预热
  created_at TEXT DEFAULT (datetime('now'))
);
CREATE INDEX idx_cache_created ON translation_cache(created_at);
CREATE INDEX idx_cache_hot ON translation_cache(hit_count DESC);

-- 术语表（R12 新增）
-- ⚠️ v1.7.5：此处只放【建表语句】。v1.7.4 曾把注释、ALTER TABLE、CREATE INDEX
-- 插进括号内，导致 SQL 语法错误、迁移 v1 直接失败（阻断级）。
-- 新增列与索引统一由【迁移 v6】追加（见 §4.7.3）：
-- 新库跑 v1→v6、老库只跑 v6，两者 schema 一致；只改 v1 会让老用户升级后缺列。
CREATE TABLE glossary (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  src TEXT NOT NULL,
  dst TEXT NOT NULL,
  note TEXT,
  scope TEXT DEFAULT 'global'     -- global | customer:<peer_id>
);
CREATE UNIQUE INDEX idx_glossary_src ON glossary(scope, src);

#### 4.7.1 术语库建模：公共库 + 客户级回退（v1.7.5 补完）

**v1.7.4 的漏洞**：说"术语库按语种对建、客户级差异用术语条目表达"，但没给机制。若把客户级条目也塞进同一个公共术语库，**两个客户对同一 `src` 要求不同 `dst` 时直接冲突**——`UNIQUE(scope, src)` 会拒绝后者，或后者覆盖前者，某客户的定制译法静默丢失。用例 #56（50 个客户各自术语）因此跑不通。

**核心约束**：路径② 的请求体是 `text` + `glossary_ids`，**没有 prompt 字段**。所以"客户级术语用 prompt 命中注入"这句话若坚持走路径② 是不成立的。能走的路只有两条：

| 层 | 内容 | 走哪条路 | 理由 |
|---|---|---|---|
| **公共层** | 产品名 / 型号 / 公司名等全局通用术语 | **路径② `glossary_ids`**，按**语种对**建库（zh↔en、zh↔pt 各一个），总数恒定为个位数 | 工程化管理、不随客户数增长、不撞 `glossary_ids` 上限 |
| **客户层** | 某客户专属译法、昵称、内部简称 | **默认自动回退路径①**（chat 端点 prompt 内嵌，仅注入当前消息命中的术语，≤20 条） | 路径② 无 prompt；且客户级条目本就少，内嵌成本可控、无译名冲突 |

备选（**仅在 §0 第 7 项实测确认 `references` 接受术语后启用**）：客户层走 `references` 数组（上限待实测，同样只注入命中项）。若实测不支持，维持回退路径①——**不额外增加机制**。

**为什么这样反而更符合 v1.7.1 的原始定位**：v1.7.1 写的是"默认走路径①，术语量大 / 需工程化管理时切路径②"。本方案里"量大"的是公共库，客户级差异天然是少量命中项，各走各的，没有推翻任何结论。

**schema 落地**（`scope` 语义不变，`glossary_id` 记录公共库映射）：

```sql
-- 迁移 v6 追加（新库 v1→v6、老库仅 v6，最终一致）
ALTER TABLE glossary ADD COLUMN glossary_id TEXT;   -- 属于哪个语种对公共库；NULL = 客户级，不入库
CREATE INDEX IF NOT EXISTS idx_glossary_gid ON glossary(glossary_id);
-- scope=global 的条目同步到公共库并回填 glossary_id；
-- scope=customer:<peer_id> 的条目 glossary_id 恒为 NULL，只用于本地命中过滤 + prompt/references 注入
```

**缓存口径同步**：`glossaryScopeKey(scope)` 按 scope 拆版本号的既有设计（R29）不变；客户级术语改动只影响该客户的缓存，不会因公共库更新而全量失效。

-- 设置表
CREATE TABLE settings (
  key TEXT PRIMARY KEY,
  value TEXT
);
-- 设置项：engine、model、hunyuanKeys(加密JSON)、deeplKey、outbound_enabled、ui_theme、glossary_enabled
```

### 4.8 UI 设计（渲染进程）

**布局**：侧边栏（账号列表 + 快捷回复 + 设置）｜ 主区（WebContentsView）｜ 右侧客户档案抽屉。WebContentsView 浮于底层，UI 层最后 `addChildView` 置顶。

**组件清单**：
- `Sidebar.vue`：账号列表（图标 + 名称 + 在线状态）、添加入口
- `ChatView.vue`：视图容器占位（bounds 由主进程 LayoutManager 控制）
- `SettingsPanel.vue`：引擎与模型档位选择、多 Key 管理（带健康状态）、代理配置、出站翻译开关、术语表入口
- `QuickReplyPanel.vue`：话术按钮网格，点击填入输入框
- `CustomerDrawer.vue`：当前聊天对象档案卡
- `ToastList.vue`：跨账号未读集中通知
- **`DiagnosticsPanel.vue`（新增）**：选择器命中诊断 + 一键复制
- **`TranslationBubble`（注入层）**：含 loading / ok / error 三态，error 可点击重试

#### 4.8.1 系统托盘与窗口关闭行为（R34，新增）

**聊天工具点 X 直接退出 = 断线 = 丢消息**，这不是用户期望的行为。

```typescript
mainWindow.on('close', (e) => {
  if (!app.isQuitting) {
    e.preventDefault();
    mainWindow.hide();                       // 关窗 = 隐藏到托盘，账号全部保持在线
    if (process.platform === 'darwin') app.dock?.hide();
    tray.setToolTip('德莱聚合 - 后台运行中');
  }
});

app.on('before-quit', () => {
  app.isQuitting = true;
  db.pragma('wal_checkpoint(TRUNCATE)');     // 干净退出
});
```

托盘右键菜单：打开主窗口 / 未读总数 / 各账号在线状态 / 退出（真正退出）。首次隐藏到托盘时用气泡提示一次"应用已最小化到托盘，账号保持在线"，否则用户会以为退掉了、反复双击图标启动多个实例（**单例锁必须做**）。

#### 4.8.2 快捷键体系（R37，新增）

| 快捷键 | 功能 |
|---|---|
| `Ctrl+1~9` | 切换到第 N 个账号 |
| `Ctrl+Tab` / `Ctrl+Shift+Tab` | 上/下一个账号 |
| `Ctrl+Enter` | 发送译文（预览条可见时） |
| `Ctrl+Shift+T` | 重新翻译当前选中消息 |
| `Ctrl+K` | 快速搜索会话 |
| `Ctrl+/` | 打开/关闭诊断面板 |
| `Escape` | 关闭抽屉/面板 |
| `Ctrl+Q` | 真正退出应用 |

全局（应用不在前台时）：`Ctrl+Shift+D` 唤出主窗口；托盘双击同效。

> `Ctrl+Tab` 在部分系统/输入法下可能被占用，需在设置里允许自定义；`Ctrl+1~9` 与 WhatsApp 内部快捷键可能冲突，注册时应测试并支持禁用。

#### 4.8.3 通知系统（R39，新增）

- **系统通知**（OS 通知中心）：应用最小化后必须能看到新消息；点击通知 → 唤起窗口 + 切到对应账号
- **通知声音**：可开关，支持不同账号设不同提示音
- **通知分级**：单聊 > 群聊 > 系统消息；群聊默认不弹（外贸群里刷屏很凶）
- **勿扰模式**：按时间段关闭通知，但**未读角标照常累加**

```typescript
function notifyNewMessage(account: Account, m: {from:string;text:string;isGroup:boolean}) {
  if (settings.get('dnd_enabled')) return;
  if (m.isGroup && !settings.get('notify_group')) return;
  const n = new Notification({
    title: `${account.name} - ${m.from}`,
    body: m.text.slice(0, 100),
    silent: !settings.get('notification_sound'),
  });
  n.on('click', () => { mainWindow.show(); viewManager.switchTo(account.id); });
  n.show();
}
```

#### 4.8.4 账号拖拽排序（R38，新增）

`accounts.sort_order` 字段已有，补 UI：侧边栏支持拖拽，`dragend` 时批量更新 `sort_order` 并持久化。外贸用户习惯按客户重要性或活跃度排列。

---

## 五、分阶段开发计划（R15 重排）

v1.0 第一阶段塞了 7 项（含打包）称 1–2 天，不现实。重排如下：

### 5.1 第一阶段：最小可信闭环（3 天 → **4 天**）

v1.5 起并入 4 项 P0/P1：启动加载策略、崩溃恢复基础版、Key 加密、系统托盘。这四项不做，用户挂 4+ 账号第一天就会遇到问题，不能拖到第二阶段。

| 天 | 任务 | 产出 |
|---|---|---|
| **Day 1** | 项目骨架（TS + Vue3 + Tailwind）；**native 模块 rebuild 链路打通验证（或确认 node:sqlite 可用）**；SessionManager；混元 lite 引擎；IPC 总线 + 类型约束 + Zod 校验；单账号视图加载；入站翻译渲染 | 单账号能翻 |
| **Day 2** | **出站三层写入逐层实测（最高风险，必须当天确认哪层生效）**；host 白名单 + 导航拦截；翻译队列/限流/自适应并发/Key 熔断；缓存 + LRU 去重 + 预热；选择器四档降级链；2 账号 partition 隔离验证 | 双向可用、2 账号不串号 |
| **Day 3** | 打包链路（含 rebuild）+ **代码签名验证**；选择器诊断面板 v1；安装到干净机器实测 | 可交付 exe |
| **Day 4（新增）** | **启动分级懒加载（R26）**；崩溃恢复基础版（R27：快照 + 熔断，草稿恢复可延后）；**KeyVault 加密（R28）**；**系统托盘 + 关闭行为（R34）**；单例锁；IPC 速率限制（R31） | 4 账号可用、挂得住 |

> **诚实的风险提示**：4 天仍是紧的，且**最大的不确定性不在这些新增项，而在 Day 2 的三层写入**。若三层全挂，出站翻译要退化为"翻译后复制到剪贴板手动粘贴"，需要额外时间设计替代交互。排期应视为"已知工作的排期"，不含未知故障的返工。

**验收标准（修订）**：
- 2 个 WhatsApp 账号同时登录互不串号，重启后登录态保持（`persist:` 生效）
- 外语消息自动显示中文译文，无时间戳 / 类名污染；滚动与切会话后译文仍在
- 中文输入 → 预览 → 发送，对方收到的是目标语言（**实测确认，不是"看起来填进去了"**）
- 连续刷屏 20 条消息不雪崩、不丢译
- 未签名/已签名安装包分别在干净 Windows 上的表现有记录

> 将 v1.0 的"4 账号同时登录"降级为"2 账号验证隔离正确"，4 账号属容量问题而非正确性问题，放到第二阶段。

### 5.2 第二阶段：提效（3–4 天 → **5–6 天**）

v1.5 追加 9 项（原清单保留，以下为新增）：
- [ ] **SQLite 迁移框架（R33）** + 迁移前自动备份
- [ ] **IPC 速率限制（R31）**（若 Day 4 未完成）
- [ ] **连接健康监控（R32）**：状态指示轮询 + 分级处置
- [ ] **翻译质量后处理（R35）** + 诊断聚合统计
- [ ] **语种检测与跳过规则（R36）**
- [ ] **快捷键体系（R37）** + **账号拖拽排序（R38）**
- [ ] **选择器热更新版本控制与回滚（R30）** + 校验和
- [ ] **缓存术语表版本号（R29）** + scope 拆分
- [ ] **通知系统完善（R39）**：系统通知 / 声音 / 分级 / 勿扰
- [ ] 崩溃恢复的草稿恢复交互（R27 完整版）
- [ ] 操作审计日志（R42）

### 5.2.1 原第二阶段清单（保留）
- [ ] 设置面板（引擎 / 模型档位 / 多 Key 健康管理 / 代理 / 出站开关）
- [ ] 术语表 CRUD + 全局与按客户两级作用域
- [ ] 快捷回复库（增删改 + 一键发送）
- [ ] 4+ 账号容量验证（懒加载 / 后台节流 / 崩溃重建）
- [ ] Telegram 注入适配 + host 白名单扩展
- [ ] 翻译缓存持久化与清理策略
- [ ] 诊断面板完善（含选择器热更新入口）

### 5.3 第三阶段：CRM 化（3–5 天）
- [ ] 客户档案抽屉（复合键绑定，备注 / 标签 / 进展）
- [ ] 未读提醒（跨账号集中通知 + 角标）
- [ ] AI 智能回复建议
- [ ] 自动更新（electron-updater）
- [ ] ~~API Key 加密存储~~ → **已提前到 Day 4（R28）**
- [ ] 语音消息转文字（本地 whisper 或 API）后走翻译管线

### 5.4 v2 方向（本期不做）
- **图片消息 OCR + 翻译（R44）**：外贸场景产品图/包装图/证书照片很多，图上文字目前无法翻译。点击图片 → 大图预览 + OCR（Tesseract.js 本地或腾讯云 OCR）+ 翻译。
- **离线队列持久化（R41）**：断网时待翻译请求写入 `pending_translations` 表，恢复后批量翻译。对展会现场这类弱网环境有价值。
- **UI i18n 预留（R45）**：`vue-i18n` 接口预留，中英双语。优先级最低——目标用户是中文外贸从业者。
- **Messenger**：反自动化最强，`.bq4j9vrk` 这类混淆类名最脆，投入产出比最低。**本期无排期**（v1.8 口径统一，不再写"排最后"以免被读成待办项）；仅在 §14.5 非目标清单中登记，schema 保留枚举值以便未来扩展。
- 群组消息的翻译策略（群内多语言混杂、发言人识别）未在本版设计，需单独讨论。

### 5.5 开发效率项（Day 1 就配）
- **注入脚本热重载（R43）**：开发模式下用 `webContents.debugger` + CDP 实现 `Ctrl+Shift+R` 重新注入，避免每次改注入脚本都重启应用。这件事在调选择器时会节省大量时间，**第一天就该配上**。

---

## 六、项目目录结构（v1.1）

```
delai-hub/
├── electron.vite.config.ts
├── package.json
├── src/
│   ├── main/
│   │   ├── index.ts                 # 入口：CA 证书、命令行开关、窗口、生命周期
│   │   ├── ipc/                     # IPC 处理器（每处含 zod 入参校验）
│   │   │   ├── translate.ts
│   │   │   ├── accounts.ts
│   │   │   ├── settings.ts
│   │   │   ├── quickreply.ts
│   │   │   ├── customer.ts
│   │   │   └── glossary.ts          # 【新增】术语表
│   │   ├── services/
│   │   │   ├── view-manager.ts      # 【改】WebContentsView 管理 + LayoutManager
│   │   │   ├── session-manager.ts   # + WebRTC 泄漏防护
│   │   │   ├── startup-policy.ts    # 【v1.5】分级懒加载
│   │   │   ├── crash-recovery.ts    # 【v1.5】快照 + 退避重建 + 熔断
│   │   │   ├── key-vault.ts         # 【v1.5】safeStorage 加密
│   │   │   ├── migrations.ts        # 【v1.5】schema_migrations 迁移框架
│   │   │   ├── selector-updater.ts  # 【v1.5】版本/校验和/回滚
│   │   │   ├── conn-health.ts       # 【v1.5】连接状态监控与分级处置
│   │   │   ├── rate-limiter.ts      # 【v1.5】IPC 速率限制
│   │   │   ├── tray.ts              # 【v1.5】系统托盘 + 关闭行为
│   │   │   ├── notifications.ts     # 【v1.5】系统通知 / 分级 / 勿扰
│   │   │   ├── translate-scheduler.ts # 【新增】队列/限流/重试/熔断
│   │   │   ├── post-processor.ts    # 【v1.5】译文质量检测
│   │   │   ├── lang-detect.ts       # 【v1.5】语种检测与跳过规则
│   │   │   ├── http-client.ts       # 【改】证书校验 + 超时 + 重试
│   │   │   ├── database.ts          # SQLite + 缓存清理 + 迁移
│   │   │   └── injection.ts         # 【新增】host 白名单 + 注入时机
│   │   └── engines/
│   │       ├── types.ts             # TranslatorProvider（含 model/glossary）
│   │       ├── hunyuan.ts           # lite / plus / pro 三档
│   │       ├── deepl.ts             # 境外，需出境告知
│   │       ├── google.ts            # 境外，需出境告知
│   │       └── key-rotator.ts       # 【改】+ 熔断冷却
│   ├── preload/
│   │   ├── index.ts                 # satisfies DelaiAPI
│   │   └── inject/
│   │       ├── translator.ts        # 通用注入（重扫 + 节点级标记 + 三态渲染）
│   │       └── input-writer.ts      # 【新增】execCommand 写入 + paste 兜底
│   ├── renderer/
│   │   ├── index.html
│   │   └── src/
│   │       ├── main.ts
│   │       ├── App.vue
│   │       ├── env.d.ts             # 【新增】declare global Window.delai
│   │       ├── components/
│   │       │   ├── Sidebar.vue
│   │       │   ├── ChatView.vue
│   │       │   ├── SettingsPanel.vue
│   │       │   ├── QuickReplyPanel.vue
│   │       │   ├── CustomerDrawer.vue
│   │       │   ├── GlossaryPanel.vue      # 【新增】
│   │       │   └── DiagnosticsPanel.vue   # 【新增】选择器诊断
│   │       ├── stores/              # Pinia
│   │       └── assets/
│   └── shared/
│       ├── types.ts                 # DelaiAPI / Account / Customer（唯一真源）
│       └── selectors.ts             # 【新增】平台选择器配置化
├── resources/                       # 图标、CA 证书包
├── build/                           # 打包配置 + 签名配置
└── scripts/
    └── rebuild.mjs                  # 【新增】native 模块 rebuild 校验
```

---

## 七、关键技术难点与对策（v1.1 更新）

| 难点 | 对策 | 状态 |
|---|---|---|
| 平台 CSP 拦截跨域请求 | 全部翻译走主进程，页面内零跨域请求 | v1.0 已有 |
| 多账号 Cookie 串号 | `persist:` 分区 + 稳定 uuid 隔离键 + 单一 getSession 入口 | 补强 |
| 多账号同 IP 封号 | 每账号独立 socks5 + **WebRTC 泄漏防护** | **新增** |
| **React 受控输入框写入无效** | 三层降级写入（execCommand → 原生 setter/Range → paste）+ caret 定位 + verify + 等帧再发送；只派发事件不改 DOM 无效 | **最高风险，需实测** |
| **虚拟列表导致译文丢失** | 节点级标记 + rAF 节流全量重扫 + 缓存兜底 | **新增** |
| 译文污染（时间戳 / 类名） | 只取文本节点 + 正则清洗 | v1.0 已有 |
| 重复翻译 / 缓存无限增长 | LRU（3000）+ 缓存带 engine/model + 90 天清理 | **修正** |
| 刷屏时请求雪崩 | 队列并发 4 + 超时 8s + 指数退避 + Key 熔断 | **新增** |
| **注入脚本逃逸到第三方站点** | host 白名单 + `will-navigate` / `setWindowOpenHandler` 拦截 | **新增** |
| 腾讯云证书报错 | `NODE_EXTRA_CA_CERTS` / 显式 ca，**不关校验** | **修正** |
| **exe 被 SmartScreen / 杀软拦截** | 代码签名 + 干净机器实测 | **新增** |
| native 模块 ABI 不匹配 | electron-rebuild（Day 1 验证）或 node:sqlite 零编译方案 | **新增** |
| 平台 DOM 改版（混淆类名重排） | 四档降级链（data-testid → ARIA → 相对树 → 类名）+ 命中档位告警 + 远端热更新 | **补强** |
| 出站写入时编辑器 state 未同步 | 写入后 verify + 等两帧 + 优先点真实发送按钮 | **新增** |
| 混元 QPS 未知导致限流 | 并发自适应（429/超时减半，稳定后回升） | **新增** |
| 发送语言判断错误 | 出站预览确认，不自动发 | v1.0 已有 |
| 客户档案跨平台主键冲突 | 复合主键 (platform, account_id, peer_id) | **修正** |
| **多账号常驻导致内存失控 / OOM** | **保活优先**（v1.4 纠错）：三组防后台化开关 + `backgroundThrottling:false` + 后台仅 `setVisible(false)`；销毁降级为手动"省电模式"；提供内存账目与实时占用显示 | **v1.4 方向性纠错** |
| **后台账号被 Chromium 挂起导致断连** | `disable-renderer-backgrounding` / `disable-background-timer-throttling` / `disable-backgrounding-occluded-windows` 三件套缺一不可 | **新增** |
| **多账号同时加载打爆启动** | 分级懒加载（immediate/priority/deferred），按 `last_active_at` 排序；点击可插队 | **新增** |
| **渲染进程崩溃丢失会话与草稿** | 30s 周期快照 + 指数退避重建 + 连续 3 次熔断 + 草稿不自动回填（防重复发送） | **新增** |
| **WebSocket 静默断连但页面不报错** | 状态指示轮询（主）+ CDP 主世界 Hook（备选）+ 分级处置（不立即 reload） | **新增，含对评审方案的修正** |
| **API Key 明文存储被窃取** | safeStorage 加密入库；明确告知换机不可解密、导出不含 Key | **新增** |
| **术语表更新后缓存仍返回旧译文** | 缓存 key 混入术语表版本号，且按 global/customer scope 拆分 | **新增** |
| **热更新推送坏配置导致全量挂掉** | 版本 + 校验和 + minAppVersion + 保留上一版可回滚 + 灰度 | **新增** |
| **数据库升级后表结构不一致** | schema_migrations 迁移框架 + 事务 + 迁移前自动备份 | **新增** |
| **SQLite 并发写 SQLITE_BUSY / 损坏** | WAL + `synchronous=NORMAL` + `busy_timeout` + 写队列串行化 + BUSY 重试 + 单连接 | **新增** |
| **渲染进程传入脏数据 / 越权调用** | 最小权限暴露（禁通用 invoke）+ 逐通道 Zod 校验 + 长度上限 + 注入层禁用 innerHTML | **新增** |
| **多显示器 DPI 下布局错位 / 黑边** | 全事件覆盖（含 moved/display-metrics-changed）+ 统一取整方向 + 延迟一帧应用 | **新增** |

---

## 八、安全与隐私（v1.1）

- `contextIsolation: true`、`nodeIntegration: false`、`sandbox: true`
- **保持 TLS 证书校验开启**，证书问题用 CA 配置解决（R1）
- API Key 存入 SQLite 时用 `safeStorage` 加密
- **注入脚本仅在 host 白名单内生效**，第三方站点不注入、不暴露 `window.delai`（R8）
- 翻译请求仅发原文文本，不发送账号 Cookie / 聊天记录
- 不存储聊天记录（翻译完成后释放），客户档案由用户手动触发保存
- **数据出境告知**：启用 DeepL / Google 前明确告知聊天内容将发往境外服务，需用户确认，默认引擎为境内混元（R14）

---

## 九、风险与备选（v1.1）

### 9.1 明确驳回的建议：共享渲染进程池（原 P2-1） ❌

评审建议"同一平台的多个账号共享一个渲染进程（iframe 隔离），session 仍独立"以降低内存。**这条不可行，予以驳回**，理由：

1. **Session 是 WebContents 级别的属性**，不是 frame 级别。一个渲染进程内无法同时承载两个 `persist:` 分区——分区决定了 Cookie/LocalStorage/IndexedDB 的存储后端，iframe 只能隔离 Browsing Context，**不能隔离 Session**。
2. 即便强行同进程，也会破坏 §4.1 的防关联前提：共享进程意味着共享部分进程级状态（内存、可能的指纹面），多账号防关联是这个产品的核心价值之一，不能为此冒险。
3. 退一步说，WhatsApp Web 是重 SPA，单实例内存主要来自页面自身的 JS 堆与 DOM，**这部分无论如何共享不掉**。省下的只有 Chromium 运行时开销（约几十 MB/实例），收益有限、风险很高。

真正能省内存的路子只有 §3.3.4 列的那几条（后台不合成、限缓存、后台账号译文降频、按需首次加载），以及限制账号数上限。**不要在这条路上投入时间。**

### 9.2 操作审计日志（R42，新增）

Key 增删、引擎切换、账号启停、省电模式开关等操作应有审计日志，用于回答"为什么某个时间点翻译质量变了/费用突然涨了"。

```sql
CREATE TABLE audit_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  action TEXT NOT NULL,      -- 'key:add' | 'engine:switch' | 'account:sleep' | ...
  target TEXT,               -- engineId / accountId
  detail TEXT,               -- JSON
  created_at TEXT DEFAULT (datetime('now'))
);
CREATE INDEX idx_audit_time ON audit_log(created_at);
```

成本极低（一次 insert），排查价值很高。

| 风险 | 等级 | 应对 |
|---|---|---|
| 平台 DOM 改版导致注入失效 | **高** | 选择器配置化 + 诊断面板 + 支持热更新；Messenger 优先级最低 |
| **出站写入方案在目标平台失效** | **高** | `execCommand` 主路径 + paste 事件兜底 + 写入后校验；Day 2 必须实测 |
| exe 未签名导致用户装不上 | **高** | 采购 OV 代码签名证书，Day 3 干净机器实测 |
| native 模块编译/ABI 问题 | 中 | Day 1 打通 rebuild；备选 `node:sqlite` 零编译 |
| 账号风控（新 Session 首次登录） | 中 | 引导用户先手动扫码登录一次；配合独立代理与 UA |
| WebRTC 泄漏导致代理失效 | 中 | 启动参数 + 注入层覆写，需实测验证 |
| 混元额度用尽 | 低 | 多引擎可切换；lite 档 + 缓存降低消耗 |
| 打包体积大 | 低 | asar 压缩 |
| 群组多语言混杂 | 未评估 | 本版不处理，需单独设计发言人识别 |

---

## 十、验收测试用例（v1.1 新增）

| # | 用例 | 期望 |
|---|---|---|
| 1 | 两个账号分别登录不同 WhatsApp 号，互发消息 | 消息出现在各自会话，无串号 |
| 2 | 重启应用 | 两账号登录态保持 |
| 3 | 对方连发 20 条外语消息 | 全部译出，无漏译、无卡死，并发受控 |
| 4 | 滚动到历史消息再滚回 | 译文仍在，未重复消耗额度 |
| 5 | 切换会话再切回 | 译文正确重建 |
| 6 | 输入中文，点发送译文 | **对方实际收到的是目标语言**（输入框显示正确不算通过，必须看接收端） |
| 6b | 禁用 execCommand（测试时 monkey-patch 使其返回 false） | 自动降级到 DOM 写入层，仍能正确发出 |
| 6c | 三层全部禁用 | 明确提示失败并把译文复制到剪贴板，不发空消息 |
| 7 | 输入英文（不含中文） | 不触发翻译预览 |
| 8 | 点击聊天中的外部链接 | 系统浏览器打开，应用内不加载、不注入 |
| 9 | 填入无效 API Key | 熔断生效，UI 提示，切换下一个 Key |
| 10 | 断网 | 显示"翻译失败 · 点击重试"，恢复后点击可成功 |
| 11 | 从混元切到 DeepL 后重翻同一句 | 拿到新引擎译文（非缓存旧值） |
| 12 | 安装到干净 Windows | 无 SmartScreen 拦截或仅有可确认的提示 |
| 13 | 平台选择器失效时打开诊断面板 | 明确显示哪个选择器命中为 0 |
| 14 | 人为使 L1/L2 选择器失效 | 自动降级到 L3/L4，诊断面板显示实际命中档位并告警 |
| 15 | 重启应用后发送高频话术 | 命中预热缓存，零网络延迟（诊断面板显示 cache hit） |
| 16 | 短时发送 50 条触发限流 | 并发自适应下调，不雪崩，恢复后自动回升 |
| 17 | 开 6 个账号挂机 8 小时（覆盖最小化 / 遮挡 / 锁屏） | **每个账号都能实时收到消息**；内存无持续增长；无 OOM |
| 18 | 后台账号（不可见 30 分钟以上）收到新消息 | 未读角标实时更新，切过去即可见消息，**无需重新加载页面** |
| 19 | 手动开启某账号"省电模式" | 提示"将不再实时收消息"需确认；开启后侧边栏显示休眠图标；点击唤醒后消息补到 |
| 19b | 从省电模式唤醒 | 登录态保持（未掉线）、会话与草稿恢复、UI 浮层未被新视图遮挡 |
| 20 | 刷屏同时编辑客户档案、改设置 | 无 SQLITE_BUSY 报错，写入均生效 |
| 21 | 强制 kill 进程后重启 | 数据库未损坏，数据完整（WAL 恢复） |
| 22 | 通过 IPC 传入超长文本 / 非法枚举值 | 被 Zod 拦截并提示，不落库、不发请求 |
| 23 | 拖动窗口到不同 DPI 的外接显示器 | 视图无黑边、无错位、无闪烁；布局自动重算 |
| 24 | 最大化 / 还原 / 全屏切换各 5 次 | 每次布局均正确 |
| 25 | 客户备注填写 `<script>alert(1)</script>` | 界面与译文气泡均原样显示文本，不执行 |
| 26 | 配置 8 个账号后冷启动 | 首屏（active 账号）3s 内可用；其余 30s 内陆续上线；无 OOM、无卡死 |
| 27 | 强制 kill 某账号渲染进程 | 自动重建并恢复到原会话；连续 3 次后停止自动重建并通知用户 |
| 28 | 修改术语表后翻译同一句话 | 新译文使用更新后术语；**全局术语改动不误伤客户级缓存** |
| 29 | 推送新选择器配置（含一个故意写错的版本） | 校验失败则拒绝应用；已应用的坏配置可一键回滚 |
| 30 | 1 秒内发出 100 次翻译请求 | 被速率限制拦截，不崩溃、不耗尽额度 |
| 31 | 拔网线 30 秒后插回 | 连接状态先转黄后恢复，积压消息正常到达，无需手动刷新 |
| 32 | 点击窗口 X | 最小化到托盘，所有账号保持在线；双击图标不启动第二个实例 |
| 33 | `Ctrl+3` | 切换到第 3 个账号 |
| 34 | 收到 "👍👍👍" | 不触发翻译（也不消耗额度） |
| 35 | 升级应用版本后启动 | 自动执行迁移，数据完整；迁移前存在 .bak 备份 |
| 36 | 换一台机器导入数据库备份 | Key 解密失败并明确提示需重新录入（不静默失败） |
| 37 | **全新安装后修改全局术语表，再翻译同一句话** | 拿到新译文（验证 `glossary_version_global` 真的递增了，不是恒为 undefined） |
| 38 | 使发送按钮的 L1 选择器失效 | 自动降级到 L2/L3 仍能发送；全部失效时才回退模拟 Enter，且诊断面板可见 |
| 39 | Linux 无 libsecret 环境下添加 API Key | 明确拒绝保存并提示，**不静默写明文**（除非用户显式授权） |
| 40 | 崩溃恢复后 1 分钟内再次崩溃 3 次 | 触发熔断停止自动重建（验证 crashCount 未被提前清零） |
| 41 | 手动休眠某账号后等待一个快照周期 | 不对该账号执行快照，也不将其唤醒 |
| 42 | 客户连问 `Do you have the blue one?` → `Yes` → `Great, send me the price` | 末句译文体现"蓝色那款"的承接（对照无上下文基线） |
| 43 | 同一句 `ok` 在 A/B 两个会话分别翻译 | 两条缓存 key 不同，译文不串 |
| 44 | **修正**：统计开启上下文前后**普通消息（非短句）**的缓存命中率 | 命中率下降幅度被量化并记录（原稿只测短句，测不到真实损失） |
| 45 | 长句（> 200 字符）翻译 | 首字 < 1.5s；短句**不**走流式 |
| 46 | 流式中途断网 | 显示"翻译失败 · 点击重试"，不残留半截灰字 |
| 47 | **并发压测**：同时发起 12 条流式请求，逐步加压至出现 429 | 记录**实测**到的限流阈值（**不预设 5**——该数字属另一产品）；并发按 R19 自适应下调，不崩溃 |
| 48 | **修正**：企业代理/中间 CA 环境下发一次流式请求 | 不出现 TLS 错误（验证 fetch→https.request 的修正生效） |
| 49 | 修改某条译文并确认 | 再翻同一句直接命中修正版；**90 天后仍未被清理策略删除** |
| 50 | 客户首句用英语（实际母语葡语） | 语言**不**被立即锁死；手工设置优先于检测值 |
| 51 | 点击统一收件箱中某条会话 | 能跳转则跳转，不能跳转则**明确提示**"该会话当前不可定位"（不静默无反应） |
| 52 | 默认配置下检查 db | 迁移 v5 建表后：`translation_history` 有行但 `src_text` 全为 NULL、`conversations.last_msg_preview` 全为 NULL（两表由 v5 创建，见 §4.7.3） |
| 53 | 点发送后模拟发送失败 | 提示"可能未发送"并提供重试；**不自动重发** |
| 54 | 账号被登出（出现二维码） | 通知"账号 X 需重新登录" |
| 55 | 话术 `Dear {客户名}, price is {报价}` | 占位符先替换再翻译，未被翻坏 |
| 56 | **配置 50 个客户各自的术语**（含两个客户对同一 `src` 指定不同 `dst`） | 公共库个数不随客户数增长；两个客户的定制译法**各自生效、互不覆盖**；客户级条目走回退路径①（或实测支持后走 `references`），不塞进公共库 |
| 57 | **一次传入 11 个 glossary_ids** | 记录实际行为（报错 / 截断），据此回填 §0 第 6 项与建模方案 |
| 58 | 路径② 下故意用无效 Key 发起 20 次请求 | 走同一 KeyRotator 与队列，触发熔断；**未绕过限流** |
| 59 | **全新安装 + 从 v1.5.2 升级两种路径各跑一遍迁移** | `glossary` 表都含 `glossary_id` 列与 `idx_glossary_gid` 索引（验证 v6 而非改 v1 的正确性） |
| 60 | 分别用纯文本 / `user:/assistant:` 前缀 / JSON 串传 `context` | 记录三者译文承接质量，据此定稿 `normalizeContext`（§0 第 7 项） |
| 61 | 把 §0 实测结果设为"lite 不支持独立接口"后，用 lite 档发起术语翻译 | `selectRoute()` 自动回退 chat，译文仍正确，诊断面板记录回退原因 |
| 62 | 流式响应中一个 SSE chunk 被截断成两半到达 | 译文不丢字（验证 `httpStream` 的尾行保留逻辑） |
| 63 | **一个汉字的 3 字节被 TCP 分在两个 chunk 里到达** | 译文不出现 U+FFFD（验证 `StringDecoder`；若改 Buffer 拼接必挂） |
| 64 | SSE 单个事件里返回**两行 `data:`**（合起来才是完整 JSON） | 译文完整，不静默丢字（验证空行分帧 + 多行 data 合并） |
| 65 | 手动重跑迁移 v5 / v6（模拟 `schema_migrations` 记录丢失） | 不报 `duplicate column name`，应用正常启动，schema 一致（验证 `addColumnIfMissing`） |
| 66 | 从 v1.5.2 老库升级，且启动时缓存清理任务抢跑 | 不报 `no such column: user_corrected`（验证迁移先于一切业务查询的启动顺序） |
| 67 | **流式期间服务端每 2s 发一次 `: ping` 心跳、永不发 `[DONE]`** | 15s 绝对 deadline 到点结束并释放队列槽位；**队列不假死**（`idleTimeout` 被心跳反复重置也不影响） |
| 68 | 模拟 429 / 401 / 400 / 5xx 四种响应 | 429/5xx 退避 + 并发下调；401 熔断换 Key 且不重试；400 只记日志（验证 `HttpError` 结构化判定，非字符串解析） |
| 69 | **流式中途断开后用户点"重试"** | 气泡先清空半截译文再重新渲染，**不出现重复文本**；用量看板标注为重试 |
| 70 | **当前会话 + 7 个后台账号同时刷屏**，并制造 300 条排队 | P0 任务优先完成且一条不丢；丢弃的全是 P2 旧任务；诊断面板显示丢弃数，队列不超 200 |

---

## 十一、待确认事项

1. **代码签名证书**：是否有现成 OV 证书？采购周期会影响 Day 3 交付。
2. **代理来源**：socks5 代理是自建 / 第三方服务？是否需要应用内做代理健康检查？
3. **混元额度**：当前 Key 的 QPS 与月度额度上限是多少？决定并发上限与缓存策略的松紧。
4. **Messenger 是否本期必做**：建议延后，需业务确认。
5. **群组消息**：是否本期支持？如需支持，发言人识别需单独设计。
6. ~~休眠期间不收消息是否可接受~~ → **已解决（v1.4）**：确认必须所有账号实时收消息，故改为全部保活，不再自动休眠。代价是内存，见 §3.3.4 账目。
7. **目标机器配置与账号数上限**：8GB 内存机器建议上限 4 个号，16GB 建议 8 个。需确定最低支持配置，并在设置页显示实时内存占用供用户判断。
8. **是否需要"省电模式"**：若目标用户机器配置普遍偏低（8GB 及以下），手动休眠仍有价值；否则可以不做，减少复杂度。
9. **选择器配置的分发方式**：自建 CDN / GitHub Release / 应用内更新服务？需确定（涉及 R30 的校验与灰度能力）。
10. **崩溃与诊断日志是否上报服务端**：Sentry / 自建 / 完全本地？若上报，需告知用户并取得同意（聊天工具涉及商业信息，日志脱敏是硬要求）。
11. **目标 Electron 版本锁定**：`WebContentsView` API 在不同版本有差异（如 `setAutoResize` 移除时机、`session.registerPreloadScript` 可用性），需先锁版本再写代码。
12. **选择器热更新是否灰度**：先 10% → 观察诊断上报 → 全量。若用户量小可不做，但"保留上一版可回滚"无论如何都要有。
13. **【v1.7.1】并发上限确认**：官方"单账号并发 5 路"出自**混元大模型 API 的翻译接口 `ChatTranslations`**（非 TokenHub）。TokenHub 侧未查到公开并发数字。**维持 R19 自适应并发（初始 4、上限 8），以实测为准**；实测确认限流后再据 429 反馈下调。
14. **【v1.7】统一收件箱的准确度可接受吗**：受会话列表虚拟化限制，非可视会话无法定位到具体会话，只能给"该账号有 N 条未读"。需业务确认这种"部分准确"是否可接受，否则 B1 应降级为"账号级未读列表"。
15. **【v1.7】语音音频可获取性**：需先做技术预研，确认能否从页面稳定取出可解码音频流，再决定 D1 是否排期。
16. **【v1.7】目标客户是否常用英语沟通**：决定 A5 自动语言回填是"帮忙"还是"添乱"。若多数客户本就用英语，自动回填应默认关闭。

---

## 十二、v1.6 功能优化项处置（v1.7 新增）

### 12.1 事实订正的三处问题（先于功能项）

> **v1.7.1 复议提示**：本节为 v1.7 原文。经 v1.7.1 重新核对腾讯云官方文档，**其中"问题 1""问题 2"两条判定有误，已就地更正**（以 ✅/❌ 标注）；"问题 3"维持。

#### ✅ 问题 1（复议：原判有误）：术语库接口**未预下线**，A3 路径② 恢复有效

v1.7 原判"术语库接口预下线（2026-12-21），故驳回 A3 路径②"。经核对官方文档，**该判定混淆了两个产品**：

- 处于预下线状态的，是**混元大模型 API**（`hunyuan.tencentcloudapi.com`，Version `2023-09-01`）下的术语库接口——`CreateGlossary` / `ListGlossary` / `DeleteGlossary` / `ListGlossaryEntry` 等，页面标注"此接口处于预下线状态，预计下线时间：2026-12-21 16:33:34"。
- 而 **TokenHub 有自己独立的一套术语库接口**：`CreateGlossary` / `DescribeGlossaries` / `DeleteGlossary` / `CreateGlossaryEntries` / `DescribeGlossaryEntries` / `ModifyGlossaryEntries` / `DeleteGlossaryEntries`，请求域名 **`tokenhub.tencentcloudapi.com`，Version `2026-03-22`**（见《大模型服务平台 TokenHub API 概览》"术语库管理相关接口"一节），**无预下线标注**。

TokenHub《混元调用指南》"术语库"一节亦明确："与术语表翻译（prompt 内嵌术语）不同，本节使用独立的翻译接口与持久化术语库……目前仅 OpenAPI 协议支持使用混元翻译术语库。"

> **处置（v1.7.1 更正）**：**A3 路径② 恢复有效，从"驳回"改回"可选路线"。** 落地方式：引擎在 OpenAI 兼容 `/v1/chat/completions` 之外，**扩展一条独立翻译接口**——`POST https://tokenhub.tencentmaas.com/v1/api/translations`（Bearer 鉴权），术语库管理走 `tokenhub.tencentcloudapi.com`（Version `2026-03-22`）。建议：默认仍走路径①，术语量大/需工程化管理的场景切路径②。

#### ✅ 问题 2（复议：原判有误）：`context` **是** TokenHub 独立翻译接口的原生参数

v1.7 原判"`context` 不在 TokenHub 协议上"。该判定**只在"坚持只用 `/v1/chat/completions`"时部分成立**，作为对 TokenHub 整体能力的描述并不准确：

《混元调用指南》"术语库"一节给出的**独立翻译接口**参数表（`POST /v1/api/translations`）明确列出：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `model` | string | 是 | 如 `hy-mt2-plus` |
| `text` | string | 是 | 待翻译的文本 |
| `target` | string | 是 | 目标语言代码 |
| `source` | string | 否 | 源语言代码，不传则自动识别 |
| `stream` | bool | 否 | 是否流式返回，默认 false |
| **`context`** | string | 否 | **上下文信息，用于让译文更连贯** |
| `references` | list of object | 否 | 参考示例（句子或术语），**最多 10 个**（待实测，§0 第 6 项） |
| `glossary_ids` | list of string | 否 | 术语库 ID 列表，**最多 10 个**（待实测，§0 第 6 项）⚠️ 该上限决定术语库建模方式，见 §4.7 缺口 1 |

> **处置（v1.7.1 更正）**：**A1 可"优先用官方字段"**。走独立翻译接口时直接传 `context`；若为复用现有 OpenAI SDK 与流式封装而走 `/v1/chat/completions`，则按官方"上下文翻译"示例做 prompt 内嵌【背景信息】。两条路线官方都支持。

#### ✅ 问题 3：Hy-SimulMT 不适用于本场景（维持）

Hy-SimulMT 于 2026-10-26 开源，中英语音到语音实时翻译、时延约 3 秒。但它是**中英双向、语音→语音、开源权重需自部署**的模型。我们的场景是"客户发来葡萄牙语/阿拉伯语语音 → 转成中文**文字**"，语种和模态都不匹配；且自部署同传模型对桌面客户端不现实。

> **处置**：**D1 降级为远期**，维持。语音转写若做，路线是 ASR（腾讯云 ASR / 本地 whisper）→ 文本进翻译管线。

#### ⚠️ 补充更正："并发 5 路"来自另一产品，不可直接沿用

- 流式：TokenHub 的 `/v1/chat/completions` 与独立翻译接口均支持 `stream`。**A2 技术上可行（维持）。**
- **"单账号并发 5 路"出自另一个产品**：该数字来自**混元大模型 API 的翻译接口 `ChatTranslations`**（`hunyuan.ai.intl.tencentcloudapi.com`，Version `2023-09-01`），文档原文"默认该接口下单账号限制并发数为 5 路"。**TokenHub 侧未查到公开并发数字。**
- **处置（v1.7.1 更正）**：**不把上限写死为 5**，维持 §4.4 的 R19 自适应并发（初始 4、上限 8），**以实测为准**；实测若确认存在限流，再据 429 反馈下调。

---

### 12.2 直接采纳的功能项（10 项）

| 项 | 内容 | 落地位置 |
|---|---|---|
| **A4** | 译文纠错闭环：气泡加"编辑 / 加入术语表"，编辑后覆盖缓存并标 `user_corrected`，诊断面板加"术语未命中 Top N" | §4.3.2 + §4.6.2 |
| **A6** | 双语对照三态（仅译文 / 对照 / 仅原文）+ 字号可调；气泡样式改用 `em`/`rem` | §4.6.2 + §3.4 |
| **B2** | 关键词提醒（price / MOQ / 交期…），命中高亮 + 通知提级 | §4.8.3 |
| **B4** | **出站"点了发送但没发出去"的检测**（原方案只测写入失败）——补"N 秒未出现新消息节点 → 提示可能未发送 + 重试/复制剪贴板"，**不自动重发** | §4.6.3 |
| **C1** | 客户跟进提醒（`last_touch_at` 超 N 天进待跟进列表） | §4.8 |
| **C3** | 客户语言/国家回填（与 A5 同一实现，但见 12.3 的修正） | §4.1 |
| **C5** | 话术模板变量 `{客户名}` 等，**先替换再翻译** | §4.7 |
| **E1** | 用量/费用看板（字符数、调用量、估算费用、Top 消耗会话） | §4.8 |
| **E3** | 代理健康检查与自动切换：定期探测各账号出口 IP、异常告警、支持备用代理（**v1.7.1 补入，原稿漏处置**） | §4.1 |
| **E4** | 登录态失效提醒：检测二维码/登录页出现 → 通知"需重新登录" | §4.6.6（合并进 connStatus 监控） |

**A4 的一个必须修正**：用户纠正后的缓存条目**不能被清理策略删掉**。§4.4 的清理是"hit_count < 10 且 90 天/超 5 万条"，用户纠正的条目往往命中次数低，会被误删。修正：

```sql
-- 用户纠正过的条目永不按时间清理
DELETE FROM translation_cache
WHERE user_corrected = 0 AND hit_count < 10 AND (...);
```

> **依赖列（v1.7.3 补）**：`user_corrected` 由**迁移 v5** 追加（`ALTER TABLE translation_cache ADD COLUMN user_corrected INTEGER DEFAULT 0`，见 §4.7.3）；用户点"编辑修正"时置 1。**缺这一列，上面这条 SQL 会直接报 `no such column: user_corrected`。**

---

### 12.3 采纳但必须改设计的功能项（5 项）

#### A1 上下文翻译 —— 改实现 + 补两个约束

采纳上下文翻译，但原设计有三处问题：

1. **可优先用官方 `context` 字段**（TokenHub 独立翻译接口有原生 `context`，见 12.1 问题 2）→ 仅当为复用现有 SDK / 流式封装而走 OpenAI 兼容 chat 端点时，才降级为 prompt 内嵌。
2. **与"不存储聊天记录"口径冲突**：上下文需要在内存里按会话累积最近 3–5 轮原文。这是**留存聊天内容**，哪怕只是内存态。必须在隐私说明里写清楚，并加约束：**仅内存、不落库、应用退出即弃、每个会话上限 5 轮 / 2000 字符**。
3. **缓存命中率会下降，原验收用例 #44 测不出来**：混入上下文指纹后，普通句子（非短句）的缓存命中率必然低于基线，而 #44 只测了短句——短句本来就被强制 `ctx=[]`，本来就不会掉。**这个用例等于没测到真正的损失。** 修正为：统计"开启上下文前后，普通消息（非短句）的缓存命中率"，接受下降，但需量化并写进设置页说明（"开启上下文翻译会提高准确度、降低缓存命中率、增加用量"）。

```typescript
// 修正后：上下文只取最近 3 轮，且总长封顶
function buildContext(ctx?: ContextTurn[]): ContextTurn[] {
  if (!ctx?.length) return [];
  const picked: ContextTurn[] = [];
  let len = 0;
  for (const t of ctx.slice(-3).reverse()) {       // 最多 3 轮
    if (len + t.text.length > 2000) break;
    picked.unshift(t); len += t.text.length;
  }
  return picked;
}
```

#### A2 流式翻译 —— 两个技术坑

1. **⚠️ 用 `fetch` 会破坏 R1 的证书修复。** 原稿的 `httpStream` 写"fetch + ReadableStream"。但 §4.2（R1）的证书方案是给 `https.request` 传 `ca` / `NODE_EXTRA_CA_CERTS`。Node 的 `fetch`（undici）**不接受 `ca` 选项**，企业代理/中间 CA 环境下会直接 TLS 失败——等于把 v1.1 修掉的坑重新挖开。
   → 修正：流式也走 `https.request`（手动解析 SSE 分块），或使用 undici 时显式传入带 `ca` 的 `Agent`/`dispatcher`。**Day 1 就要在目标网络环境验证一次。**

2. **⚠️ 流式会放大并发压力。** 刷屏 20 条 = 20 条长连接 SSE。注意"并发 5 路"是**混元大模型 API** 的数字，**不能直接套到 TokenHub**（见 §12.1 补充更正）。
   → 修正：流式**只对"长句"启用**（如 > 200 字符），短句仍走非流式；并发上限**以实测为准**，维持 R19 自适应（初始 4、上限 8）；总时长阈值从 30s 收到 **15s**（聊天气泡等 30 秒没有意义）。

3. 缓存只在流结束后落库——原稿这条是对的，保留。另需注意：**流式中途若触发虚拟列表重扫**，增量灰字会被 `renderTranslation` 覆盖，流式状态需按节点 id 维持（与 §4.6.2 的节点级标记合并）。

#### A5 出站语言自动判定 —— 自动回填会锁死错误语种

"入站检测到语言即回填 `customers.language`，出站 `to` 默认取该客户语言"——**这条在真实外贸场景会出问题**：大量客户第一句用英语（哪怕母语是葡语/西语），一旦回填成 `en`，后续永远翻成英语，而客户切回母语时我们仍在发英语。且 B2B 场景中"客户用英语、我们回英语"本来就是常态，自动切换语种反而添乱。

→ 修正：
- 语言**只在"高置信 + 连续 2 次一致"**时才回填，且**永不覆盖用户手工设置的值**（手工值优先级最高）
- 出站默认语言以**用户手工设置 > 上次对该客户使用的语言 > 检测值**的顺序取值
- 提供"该客户语言"下拉，可随时改
- 风格（商务正式/日常亲切）采纳，映射到 system prompt 指令

#### B1 统一收件箱 —— 最大体验缺口，但有两个硬约束

这是 21 项里价值最高的，但原设计低估了难度：

1. **⚠️ 会话列表同样被虚拟化，取不到全量未读数。** WhatsApp Web 的会话列表是虚拟滚动的，**只有可视区域内的会话节点存在于 DOM**。原稿说"注入层按会话级上报元数据、变化才上报（不轮询）"——问题是未渲染的会话根本没有节点，无从"变化上报"。
   → 修正：可获得的只有两类数据——① `document.title` 里的**总未读数**（§3.3.6 已在用）；② **当前可视区域内**的会话。因此统一收件箱**只能做到"部分准确"**：可视会话准确，非可视会话只能显示"该账号有 N 条未读"而不能定位到具体会话。**这一点必须如实告知用户，不要做成看起来能定位但实际漏报。**
2. **⚠️ 跳转到指定会话没有可靠深链。** 原稿只写"注入层导航到该 peer"。实际可选：`web.whatsapp.com/send?phone=xxx`（会触发整页重载，慢且丢失当前草稿）vs **在会话列表里找到该行并点击**（列表虚拟化意味着目标行可能不在 DOM 里，需先滚动查找）。
   → 修正：优先"滚动查找 + 点击行"，失败再回退到 `send?phone=` 深链，并明确提示会重载。**这个功能必须做降级提示，不能点了没反应。**
3. 合规口径（只聚合元数据、正文摘要默认关）原稿是对的，**保留并写进设置项**。

#### E2 翻译历史 —— schema 与隐私口径自相矛盾

原稿 §八 的口径是"默认只存译文"，但迁移草案里的 `translation_history` 表**有 `src_text` 字段**。`src_text` 就是客户的原始消息，存了它等于存聊天记录，直接违反 §八 的承诺。
→ 修正：`src_text` 必须为**可空且默认不写**；若用户开启"保存原文"，需二次确认并单独说明。默认行为：只存 `dst_text` + 语种 + 时间 + 引擎，且**整体可关闭**。
→ **落表（v1.7.3 补）**：`translation_history` 由**迁移 v5** 创建（见 §4.7.3），`src_text` 可空、默认 NULL——原文只提"迁移草案"却从未真正建表，验收 #52 此前无法执行。

---

### 12.4 驳回 / 降级项（6 项）

| 项 | 内容 | 理由 |
|---|---|---|
| **A3 路径②** | OpenAPI 持久化术语库 | ✅ **本项已复议撤销**：术语库接口**未预下线**（处于预下线的是混元大模型 API，TokenHub 另有一套独立术语库接口）。详见 §12.1 问题 1——**A3 路径② 恢复为可选路线** |
| **B3** | 会话级静音与置顶 | 依赖 B1 的会话元数据，而 B1 受虚拟化限制只能部分准确；等 B1 落地后再说 |
| **B5** | 全局搜索（跨账号/客户档案） | 同上，依赖会话数据完整性；且本地搜索需要索引聊天内容，与"不留存"冲突 |
| **C2** | 产品/报价库 + 一键发送 | 是 ERP 功能不是翻译工具功能，范围蔓延；术语表由产品库自动生成这条**可拆出来先做**（纯数据导入） |
| **C4** | AI 回复建议 | 依赖上下文（A1）且成本不可控，排在核心闭环与 A1 之后 |
| **D1** | 语音转写 → 实时同传 | 卡点不在 ASR，而在 WhatsApp 语音是**加密媒体**，注入层能否稳定取出可解码音频未经验证；Hy-SimulMT 为中英语音→语音、需自部署，不适用（见 §12.1 问题 3）。降级为远期，先做技术预研 |

---

### 12.5 采纳项的接口与数据结构（修正版）

引擎接口（v1.7.1 更正：`context` 与 `glossaryIds` 在独立翻译接口下**均为可用官方字段**，仅走 chat 端点时才降级为 prompt 内嵌）：

```typescript
export interface TranslateOpts {
  from?: string;
  to: string;
  model?: 'hy-mt2-lite' | 'hy-mt2-plus' | 'hy-mt2-pro';
  // 术语：两条路二选一
  glossary?: GlossaryEntry[];      // 路径①：prompt 内嵌（走 /v1/chat/completions）
  glossaryIds?: string[];          // 路径②：引用持久化术语库（走 /v1/api/translations）
                                   // ⚠️ v1.7.4：官方记为"最多 10 个"，该上限【待实测确认】（§0 第 6 项）。
                                   // 受此上限约束，术语库须按【语种对】而非【客户】建模，见 §4.7 缺口 1。
  // 上下文（v1.7.3 订正）：原 `context?: string | ContextTurn[]` 未区分两条路线的格式，语义不明。
  // 分派规则——走独立翻译接口（路径②）取 contextText；走 chat 端点（路径①）取 contextTurns。
  contextText?: string;            // 已拼接的上下文全文，直接传给官方 context 字段（官方为 string）
  contextTurns?: ContextTurn[];    // 结构化轮次，供 chat 端点 prompt 内嵌【背景信息】
  references?: { type: 'sentence' | 'term'; text: string; translation: string }[];
  style?: 'business' | 'casual';
  stream?: boolean;
  signal?: AbortSignal;
}

// 引擎层归一化：两条路线各取所需，调用方只管往上塞 turns
// ⚠️ v1.7.5：官方只说明 context 是 string「用于让译文更连贯」，并未规定
// "user:/assistant:" 角色前缀格式。下面这个拼法是【暂定实现】，须经 §0 第 7 项
// 实测确认（对比纯文本 / 角色前缀 / JSON 串三种的承接质量）后再定稿。
// 实测不支持时改为纯文本拼接，或直接回退路径① prompt 内嵌。
function normalizeContext(o: TranslateOpts, route: 'openapi' | 'chat') {
  const turns = buildContext(o.contextTurns);        // 3 轮 / 2000 字符封顶，见 §12.3 A1
  return route === 'openapi'
    ? { text: turns.map(t => `${t.role}: ${t.text}`).join('\n') }   // 待实测：角色前缀格式
    : { turns };                                                      // chat 端点：拼进 messages
}
```

**⚠️ v1.7.5：路由选择必须带能力回退，不能只判模型档位。**

§0 第 3 项验的是"lite 能否走独立翻译接口"，但触发回退的条件不止这一条。统一收敛到 `selectRoute()`：

```typescript
function selectRoute(o: TranslateOpts, caps: ApiCaps): 'openapi' | 'chat' {
  // 任一能力不满足 → 回退 chat（功能不丢，仅走 prompt 内嵌）
  if (!caps.openApiEndpointExists) return 'chat';        // §0 第 2 项：端点不存在
  if (!caps.modelSupportsOpenApi[o.model ?? 'hy-mt2-lite']) return 'chat';  // §0 第 3 项：lite 可能不支持
  if (!caps.referencesAcceptTerms && o.customerGlossaryHit) return 'chat';   // 客户级术语，见 §4.7.1
  if ((o.glossaryIds?.length ?? 0) > (caps.glossaryIdLimit ?? 10)) return 'chat';  // §0 第 6 项
  if (o.wantOfficialContext && !caps.contextFieldConfirmed) return 'chat';   // §0 第 7 项
  return 'openapi';
}
```

> **回退不是降级失败**：路径① 的 prompt 内嵌能覆盖术语、上下文、风格全部能力，只是少了工程化术语库管理。UI 无需报错，诊断面板记录"本次走 chat（原因：xxx）"即可。

**⚠️ v1.7.4 缺口 2：两条路径的【请求体】完全不同，不只是"换个 URL + 多几个字段"。**

| | 路径① `/v1/chat/completions` | 路径② `/v1/api/translations` |
|---|---|---|
| 承载文本 | `messages: [{role, content}]` | `text: string` |
| 语种 | 写在 prompt 里（"translate into French"） | `source` / `target` 独立字段 |
| 术语 | prompt 内嵌 | `glossary_ids` |
| 上下文 | prompt 内嵌【背景信息】 | `context: string` |

`TranslatorProvider` 必须实现**两套 body 构建**（`buildChatBody` / `buildOpenApiBody`），由 `route` 分派；不要写成一个函数里加 if——两条路之后还会各自演进，混在一起必然腐化。

```typescript
class HunyuanEngine implements TranslatorProvider {
  private caps: ApiCaps;   // 由 §0 七项实测结果在启动时注入，未实测前全部按"不支持"保守取值

  // v1.7.5：每次请求动态选路，而非固定字段
  route(o: TranslateOpts): 'openapi' | 'chat' {
    return selectRoute(o, this.caps);
  }

  buildBody(o: TranslateOpts, text: string): unknown {
    return this.route(o) === 'openapi'
      ? { model: o.model, text, target: o.to, source: o.from,
          context: o.contextText, glossary_ids: o.glossaryIds,
          references: o.references, stream: o.stream ?? false }
      : { model: o.model, stream: o.stream ?? false,
          messages: this.buildMessages(o, text) };   // system=术语+风格+背景, user=原文
  }

  // 流式与非流式共用同一条出站管道（§4.2.1 / §4.4 队列 / KeyRotator）
  // v1.8：与 httpStream 新签名对齐（requestId / signal / 绝对 deadline / 结构化错误）
  async call(o: TranslateOpts, text: string, ctx: {
    requestId: string;                 // 幂等键，见 §4.2.1 流式重试边界
    onDelta?: (s: string) => void;
  }) {
    const body = JSON.stringify(this.buildBody(o, text));
    const common = {
      headers: { Authorization: `Bearer ${keyRotator.next()}` },   // Content-Type 由 http-client 强制
      body, ca: CA_POOL,
      signal: o.signal,
    };
    const url = this.route(o) === 'openapi'
      ? `${BASE}/v1/api/translations` : `${BASE}/v1/chat/completions`;
    try {
      return ctx.onDelta
        ? await new Promise<string>((res, rej) => httpStream(url, {
            ...common, requestId: ctx.requestId,
            idleTimeoutMs: 5_000, deadlineMs: 15_000,  // 绝对 deadline，心跳推不动
          }, { onDelta: ctx.onDelta!, onDone: res, onError: rej }))
        : (await httpJson(url, { ...common, timeoutMs: 8_000 })).json;
    } catch (e) {
      if (e instanceof HttpError) {
        if (e.status === 401 || e.status === 403) keyRotator.penalize(/* 当前 key */);
        else if (e.retryable) translateQueue.onThrottle();
      }
      throw e;
    }
  }
}
```

**⚠️ v1.7.4 缺口 3：路径② 必须复用同一条出站管道。**

`/v1/api/translations` 是新端点，但**不能另起一套 HTTP 调用**——否则会绕过 §4.2 的 R1 证书修复（`https.request` + `ca`）与 §4.4 的队列/限流/Key 熔断。约束：

- 一律走 `http-client.ts`（`https.request` + 显式 `ca`），**禁止 `fetch`**（undici 不接受 `ca`，企业代理环境直接 TLS 失败）
- 请求统一进 `TranslateQueue`（并发按 R19 自适应）与 `KeyRotator`（401/403 触发熔断）
- 路径② 的 `stream` 同样按 §12.3 A2 收敛：仅长句启用、总时长 15s 封顶

**流式 IPC 不能用"返回带回调的对象"**。原稿 `translateStream()` 返回 `{ onDelta, done }`——函数作为返回值穿过 `contextBridge` 是脆弱的（contextBridge 只可靠代理**暴露对象上的方法**和**作为入参传入的回调**）。沿用本项目已有的 `onTranslateResult(cb)` 模式：

```typescript
// ✅ 正确：回调作为入参，与现有 onTranslateResult 一致
export interface DelaiAPI {
  translateStream(text: string, to: string, model: string,
                  onDelta: (s: string) => void): Promise<string | null>;
  // 或：onTranslateDelta(cb) + translate() 返回 Promise
}
```

**迁移版本号修正**：原稿标题写"迁移 v4"、代码写 `version: 5`、结尾写"上线 v4 后"——三处不一致。v1.5.2 的 migrations 已占用 v1–v4（v4 = `add_audit_log`），因此本批新表统一为 **v5**，命名为 `v1_7_feature_tables`。

**术语 prompt 的成本约束（原稿遗漏）**：prompt 内嵌术语表会**每次调用都消耗输入 token**，且模型最大输入仅 4k。200 条术语 × 每条消息 = 成本与延迟双杀。
→ 修正：**入站（外语→中文）与出站（中文→外语）双向都走"命中过滤"**——先在本地对当前消息做一次子串匹配，只把命中的术语注入 prompt，并对单次注入条数设上限（如 20 条）。两条链路都带术语表 prompt、token 成本相同，不能只顾出站（v1.7.3 补充）。另：若走独立翻译接口，可直接用 `glossary_ids` 引用持久化术语库（最多 10 个），无需把术语拼进 prompt——这条路已确认可用（见 §12.1 问题 1）。

---

### 12.6 排期硬提醒（写在最后，但优先级最高）

v1.5.2 的 Day 2「出站三层写入」**至今零实测**——三层全部是基于浏览器行为推导的。在这 21 项功能上投入任何工期之前，必须先确认核心链路成立：

- 若三层写入全部失效，出站翻译退化为"复制到剪贴板手动粘贴"，届时 A5（语气/语种）、C5（模板变量）、B4（发送结果校验）的价值都要重新评估
- B1 统一收件箱依赖注入层稳定读取会话列表，若 §4.6.4 的选择器降级链在会话列表上就命中不了 L4，B1 无从谈起
- **建议顺序**：核心闭环实测通过 → A4/A6/B4/E4（低成本高收益）→ A1/A2 → B1 预研（先验证能否取到未读数与跳转）→ 其余

> **v1.8：本节已被 §13 的 spike 三连接管。** 这里的"核心闭环实测"即 S2，"4/8 账号容量验证"并入 S3；排期以 §13.2 为准，不再单独维护两份顺序。

---

## 十三、收敛执行计划：spike 三连（v1.8 新增，替代"继续出修订版"）

外部评审的核心主张成立并采纳：**七轮下来没有一行代码、没有一次真实请求，再出 v1.7.7 不会产生新信息。** 因此把"下一步"从"改文档"切换为"跑三段实测，用结果回填 v2.0 冻结版"。

### 13.1 分工边界（先说清楚谁能做什么）

| 事项 | 谁做 | 说明 |
|---|---|---|
| 文档层事实核查（接口是否存在、参数表、下线公告、公开并发数字） | 我（联网检索） | 前几轮已核过：TokenHub 术语库接口存在且未下线；"并发 5 路"属另一产品。**这部分结论已稳定，不必重复核** |
| **带 API Key 的真实请求**（§0 第 2/3/4/6/7 项） | **项目侧执行** | 我没有你的 Key，也无法代表你发付费请求。能做的最多是给出可复制的 curl 脚本 |
| 真机保活 / CPU / 内存 / 延迟数据（S3） | 项目侧执行 | 需要目标机器与真实 WhatsApp 账号 |

> 换句话说：**§0 剩下的阻塞不是"没人查文档"，是"没人发过一次真请求"。** 这件事只能你来做，我负责把要发什么、怎么判、结果填哪儿写成清单（§13.2 S1）。

### 13.2 三段 spike

| 段 | 工期 | 唯一目标 | 交付物（回填到文档哪里） |
|---|---|---|---|
| **S1 · 外部事实钉死** | 半天 | 把 §0 七项 + 附带项从"待实测"变成"确定" | 一张实测结论表 → 回填 §0、§12.1 参数表、`ApiCaps` 初始值、**R19 并发上下限**、§4.7.1 术语库建模、`selectRoute()` 的回退条件 |
| **S2 · 单账号最小闭环** | 3–5 天 | 验证最脆的一环：**注入抓消息 → 主进程翻译 → 译文渲染 → 写回发送** | 三层写入哪一档生效（§4.6.3）、选择器实际命中档位（§4.6.4）、**首字/整句延迟 P95**（§14.2）、出站发送实测结果 |
| **S3 · 4/8 账号真机保活** | 2 天 | 验证保活模型的真实代价 | 内存 / **CPU%** / 耗电（§3.3.4）、后台收消息延迟、8 账号并发时**队列等待 P95**（§14.2）、是否需要降级策略 |

**S1 建议产物形态**（每项一行，直接贴回文档即可）：

| 项 | 请求 | 判定 | 实测结果 | 回填动作 |
|---|---|---|---|---|
| 1 | `CreateGlossary` | 返回 `GlossaryId` | 待填 | 确认 A3 路径② |
| 2 | `POST /v1/api/translations` | 200 且返回译文；确认 `context` 字段类型 | 待填 | 定 `ApiCaps.openApiEndpointExists` / `contextFieldConfirmed` |
| 3 | 同上用 `hy-mt2-lite` | 200 还是 4xx | 待填 | 定 `modelSupportsOpenApi` |
| 4 | 逐步加压 | 出现 429 时的并发数 | 待填 | **定 R19 的 ceiling**（现在 8 是猜的） |
| 5 | Bearer vs TC3 签名 | 哪个返回 200 | 待填 | 定引擎鉴权 |
| 6 | 一次传 11 个 `glossary_ids` | 报错 / 静默截断 | 待填 | 术语库按语种对建模是否成立 |
| 7 | `context` 三种格式对比 | 末句承接质量 | 待填 | 定稿 `normalizeContext` |
| 附 | `references` 传 term 对象 | 是否采纳 | 待填 | 客户级术语走 references 还是回退路径① |

### 13.3 回填规则（防止又变成"改文档"）

1. **三段跑完之前，不再新增设计章节、不新增 R 编号、不发 v1.8.x。**
2. 实测结果与正文冲突时，**只改结论、不改版本号**，在 §0 结论表里注明日期与请求片段。
3. 三段全部完成后，一次性出 **v2.0 冻结版**：把"待实测/待确认/估算"全部替换为实数，删掉已失效的备选路径，§14 的占位补成正式章节。
4. spike 期间发现的缺陷记入 §13.4 缺陷池，**按批次修**，不逐条发版。

### 13.4 缺陷池（spike 期间只记录，不单独发版）

| # | 来源 | 状态 |
|---|---|---|
| P-1 | 文心反馈 6 条中未展开的 4 条 | **待补入**（分享页只给了结构与前 2 条） |
| P-2 | §0 七项实测引出的引擎调整 | 待 S1 |
| P-3 | 三层写入实测引出的替代交互 | 待 S2 |
| P-4 | CPU/耗电不达标引出的降级策略 | 待 S3 |

---

## 十四、产品级风险与验收基线（v1.8 占位，**待回填，不编结论**）

外部评审指出四类"产品能不能卖"的缺口，前面十三章基本没覆盖。这里**只列待答问题**——数字和结论必须来自 S2/S3 或业务决策，现在写就是编。

### 14.1 账号封禁与平台 ToS（B-3）

文档此前只有三处零散提及（防关联封号、多账号同 IP + WebRTC、新 Session 风控引导），**没有独立风险章节**。待答：

- WhatsApp / Meta 的 ToS 对**非官方客户端**的立场，以及本产品（第三方客户端 + DOM 自动写入）的合规边界 → **需要业务/法务判断，不是技术问题**
- 可被识别的特征清单：UA 伪造、WebRTC 覆写、CDP 调试端口、`contenteditable` 写入行为、操作频率
- 缓解项是否采纳：UA 与真实 Chrome 一致化、发送前人工确认（**已有**）、写入走真实输入事件而非直接赋值（**§4.6.3 三层已按此设计**）、**单账号操作频率上限**（待定数值）、**封号后数据迁移路径**（待设计）
- 是否需要在首次启动时向用户明示风险并取得确认

> **这一项可能直接影响产品是否成立。** 建议在 S1 之前就由业务侧拍板，不要等技术方案冻结后才发现不能做。

### 14.2 性能指标与验收基线（B-4）

"实时翻译"目前没有定义，`maxConcurrent` / 缓存 / lite-plus 选档全都无法验收。待 S2/S3 回填：

| 指标 | 口径 | 数值 |
|---|---|---|
| 首字延迟 P95 | 从消息节点出现到译文首字渲染（含缓存命中与未命中分开统计） | **待 S2** |
| 整句完成 P95 | 到整句译文稳定显示 | **待 S2** |
| 队列等待 P95 | 8 账号并发时，P0/P1/P2 各档从入队到开始请求 | **待 S3** |
| 缓存命中率 | 开启/关闭上下文两种口径（用例 #44） | **待 S2** |
| 8 账号闲置 CPU% / 耗电 | 见 §3.3.4 | **待 S3** |
| 后台收消息延迟 | 不可见 30 分钟后新消息到达角标的耗时 | **待 S3** |

### 14.3 本地数据安全（D）

§八已有"不留存聊天记录"口径，但 **SQLite 里缓存译文、术语表、话术库、客户备注都是明文**（`safeStorage` 目前只保护 API Key）。待答：

- 是否需要 SQLCipher / 全库加密，还是按字段级加密（客户备注、译文缓存）
- 磁盘上的 `.bak` 迁移备份（§4.7.3）同样是明文，是否纳入清理策略
- 诊断日志"一键复制"会带出原文样本，**是否脱敏**、是否需要用户二次确认
- 崩溃上报若开启（§11 待确认 10），脱敏规则是什么

### 14.4 发布运维（D）

已有但**分散**，建议 v2.0 收拢成独立章节：代码签名（§5.1 Day 3）、自动更新（§5.3）、崩溃上报（§11-10）、选择器灰度与回滚（§4.6.7）、安装包在干净机器的表现（用例 #12）。待答：分发渠道、更新服务器、是否需要崩溃日志上报（涉及商业信息，需用户同意）。

### 14.5 非目标清单与排期口径（B-5 / D）

- **本期非目标**：Messenger（schema 仅预留枚举，无排期）、群组消息翻译、语音转写（D1 远期）、UI i18n
- **45 个 R 编号目前没有人天估算**。§5.1 的 4 天是"模块清单"不是"工作量估算"；v2.0 需按模块给出人天与依赖顺序，并明确"先做哪个模块"——**建议顺序已由 §13.2 的 S1→S2→S3 给出**，不再另写一份排期

---

*v1.8 设计冻结版。本版停止迭代文档内部细节，只做两件事：修掉 7 处不依赖实测的代码逻辑缺陷（绝对 deadline、HttpError、强制 Content-Type、AbortSignal、监听器解绑、流式幂等边界、队列优先级与背压），以及把产品级缺口写成待回填占位。**下一步不是出 v1.8.1，而是执行 §13.2 的 S1（半天）→ S2（3–5 天）→ S3（2 天），三段跑完后一次性回填 v2.0 冻结版。***
