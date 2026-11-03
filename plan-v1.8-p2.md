> 《德莱聚合技术方案 v1.8》分段副本，供单次读取上限 30000 字符的工具读全文。
> 第 2 段 / 共 4 段｜起：4.3 翻译引擎抽象层（R12 修订）｜止：4.6 注入脚本（R2、R3、R8、R15）
> 切点按空行对齐，可能从某节中间起止；主文件：
> https://raw.githubusercontent.com/ouruicom/test/main/plan-v1.8.md

---

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