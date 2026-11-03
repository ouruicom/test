# 评审意见：plan-v1.8（设计冻结版）

| 项 | 内容 |
|---|---|
| 评审对象 | `plan-v1.8.md` — 德莱聚合（delai-hub）技术方案 · 设计冻结版（179,242 字节 / 2,922 行 / 60 个标题） |
| 评审人 | 文心（独立评审，非项目成员） |
| 评审日期 | 2026-11-03 |
| 评审方式 | 全文静态阅读 + TypeScript / Node 运行时语义推演；**未执行代码、未持有 API Key、未接触真机** |
| 交付形态 | 缺陷清单（可定位）+ 补丁（可应用）+ S1 验证命令包（可执行） |

---

## 0. 结论（先看这三段）

**第一，§13 的转向是对的，但你只冻结了"设计"，没冻结"代码"。**
文档自称"修掉 7 处不依赖实测的代码逻辑缺陷"，我逐条核过：其中 4 处引入了新错误，2 处只修了一半，1 处修在注释里没修在代码里。也就是说，"冻结版"里躺着的 TypeScript，**大概率一次 `tsc --noEmit` 都过不了**（见 P1-4：`selectRoute()` 读取了 `TranslateOpts` 上不存在的两个字段）。

**第二，本轮最该做的不是第 8 次读文档，而是花半天把代码抽出来编译一次。**
把 §4.2 / §4.4 / §12.5 三段合计约 300 行的 TypeScript 抽进 `packages/core`，配 `tsc --noEmit` 和 3 个单测，就能一次性消掉本文 P0-1、P0-3、P0-4、P1-1、P1-2、P1-4、P1-5 共 7 条缺陷。**这件事不需要 API Key、不需要真机、不产生费用**，却比再写 2000 行设计更能收敛风险——因为"读文档挑错"和"编译挑错"的漏检率不是一个量级。

**第三，关于 §0 那七项外部事实，我不给结论。**
没有 Key 就没有实测，任何"该接口应该支持"的判断都是编造。我能给的是**可执行的验证命令**（§5），把 §13.2 的七行表变成七条可以直接粘贴的 `curl`，跑完把结果填回去即可。

---

## 1. 评审口径与自限

为免"第 8 轮评审"重演互相指认，先把边界写死：

| 我做了什么 | 我**没有**做什么 |
|---|---|
| 通读 2,922 行，逐段核对注释与代码是否一致 | 没有发起任何 TokenHub / 混元真实请求（无 Key） |
| 对 4.2 / 4.2.1 / 4.4 / 12.5 的 TypeScript 做类型与并发语义推演 | 没有编译、没有跑单测（结论可按文中方法复核） |
| 核对 §5 与 §13 的排期口径是否自洽 | 没有评价 §14.1 的合规可行性（属业务/法务判断，非技术评审） |
| 核对仓库现状（`plan-v1.8.md` sha `bd08fb5…`、`review-wenxin-v1.md` 不存在） | 没有改动 `plan-v1.8.md` 一个字节 |

**每一条缺陷都给了三样东西：位置、为什么错、怎么验。** 没有这三样的意见，请直接退回，不必进 §13.4 缺陷池。

---

## 2. 缺陷清单（15 条，按修复优先级）

| 编号 | 位置 | 一句话 | 级别 |
|---|---|---|---|
| P0-1 | §4.4 `applyBackpressure` | 比较符写反，背压丢弃的正是"绝不丢"的 P0 任务 | 阻断 |
| P0-2 | §4.4 `cacheKey` | 键里不含 `context` 与 `style`，上下文翻译与语气会被旧缓存串味 | 阻断 |
| P0-3 | §12.5 `call()` | `keyRotator.next()` 可能返回 `null`，实际发出 `Bearer null` | 阻断 |
| P0-4 | §12.5 / §4.4 | 5xx 也触发降并发，且 `onStable()` 无调用点 → 并发只降不升，终局锁死 1 | 阻断 |
| P1-1 | §4.2 / §4.2.1 | 已 abort 的 signal 不会触发回调，取消形同虚设 | 高 |
| P1-2 | §4.2 `httpJson` | C-1 只给流式加了绝对 deadline，非流式仍是 socket 空闲超时 | 高 |
| P1-3 | §12.5 `selectRoute` | 读取 `o.wantOfficialContext` / `o.customerGlossaryHit`，类型上不存在 | 高 |
| P1-4 | §12.5 `normalizeContext` | 定义了但没人调用，`context` 在 openapi 路径上恒为 `undefined` | 高 |
| P1-5 | §4.4 `push()` | 同文本去重直接 `return`，被合并的节点无兜底、不升优先级 | 高 |
| P1-6 | §12.5 `call()` | 引擎层直接调 `translateQueue.onThrottle()`，分层被打穿 | 中 |
| P2-1 | §4.2.1 B-1 | 中文场景 `\b` 词边界失效，术语命中率接近 0 | 中 |
| P2-2 | §12.5 `deadlineMs` | 15s 固定上限与"仅长句启用流式"自相矛盾 | 中 |
| P2-3 | §12.5 `idleTimeoutMs` | 5s 空闲超时可能砍掉正常的慢首字 | 中 |
| P2-4 | §4.2 错误表 | 408 同时被标 `retryable=true` 和 `false`，消费方读哪个未定义 | 中 |
| P2-5 | §5 vs §13 | 12–15 人天 与 5.5–7.5 天两套排期并行维护 | 低 |

---

### P0-1 · 背压比较符写反：丢掉的正是"绝不丢"的任务

**位置**：§4.4 `applyBackpressure()`（约 L1060–1071）

**原文**：

```ts
// 超限：优先丢【后台账号】的旧任务，当前可见会话的任务绝不丢
this.queue.sort((a, b) => b.priority - a.priority || b.enqueuedAt - a.enqueuedAt);
const dropped = this.queue.splice(this.MAX_QUEUE);
```

**为什么错**：`Priority` 的定义是 `0=当前可见会话 / 1=同账号其它会话 / 2=后台账号`。`b.priority - a.priority` 是**降序**，排完序数组头是 P2、尾是 P0；`splice(200)` 从尾部切掉，**被切掉的恰好是 P0**。注释写"绝不丢"，代码实现的是"优先丢"。而且次级比较符 `a.enqueuedAt - b.enqueuedAt` 也是反的：它把最老的排在头部保留、把最新的切掉，与"丢旧任务"同样相反。

这不是边界情况：8 账号刷屏时队列到 200 是常态，**用户正在看的那个会话最先瞎**。

**修法**：

```ts
// 保留序 = 优先级升序（P0 在头）+ 同级新任务在头（老任务先丢）
this.queue.sort((a, b) => a.priority - b.priority || b.enqueuedAt - a.enqueuedAt);
const dropped = this.queue.splice(this.MAX_QUEUE);
// 断言：dropped 中不允许出现 priority === 0
if (dropped.some(j => j.priority === 0)) {
  diagnostics.inc('translate.backpressure.violation', dropped.length);
}
```

**怎么验**：单测直接构造 240 个 P0 + 20 个 P2，调 `applyBackpressure()`，断言 `queue` 中 P0 数量为 240、`dropped` 中 P0 数量为 0。当前代码这条断言必挂。

---

### P0-2 · 缓存键漏掉 `context` 与 `style`

**位置**：§4.4 `cacheKey()`（约 L1146）

```ts
function cacheKey(p: { engine; model; from; to; text; scope }): string {
  return md5(`${p.engine}|${p.model}|${p.from}|${p.to}|${glossaryScopeKey(p.scope)}|${p.text}`);
}
```

**为什么错**：A1 的核心卖点是"带上下文翻译"（`contextText` 3 轮 / 2000 字符），`style` 是"商务 / 口语"。这两个字段**都会改变译文**，却都不在键里。后果很具体：

- 同一句 "please confirm" 在"催发货"上下文里译成"请确认发货时间"，在"报价"上下文里应译成"请确认价格"。第一次的结果被缓存，**第二次直接命中错误译文，且 90 天内不会自愈**。
- `style=business` 与 `style=casual` 互相污染。

这与文档自己修 R29（把术语版本混进 key）是**同一类问题的同一个解法**——只修了术语那一半。

**修法**：

```ts
function cacheKey(p: {
  engine: string; model: string; from: string; to: string;
  text: string; scope: string;
  contextText?: string; style?: string;
}): string {
  const ctx = p.contextText ? md5(p.contextText) : '-';   // 上下文参与键，但只存摘要
  const st  = p.style ?? 'default';
  return md5(`${p.engine}|${p.model}|${p.from}|${p.to}|${glossaryScopeKey(p.scope)}|${st}|${ctx}|${p.text}`);
}
```

**注意**：`contextText` 每次滑动都会变（最近 3 轮），命中率会下降。所以建议**只在 `contextText` 为空时读缓存**，带上下文时只写不读，或者干脆不缓存——并在 §4.4 里明确写出这个取舍，别让它成为实现者的自由裁量。

---

### P0-3 · `keyRotator.next()` 返回 `null` 时，请求照发

**位置**：§12.5 `call()`（约 L2757–2775）

```ts
headers: { Authorization: `Bearer ${keyRotator.next()}` },
...
if (e.status === 401 || e.status === 403) keyRotator.penalize(/* 当前 key */);
```

**为什么错**：`next()` 在全部 Key 处于冷却时 `return null`（§4.4 明确写了"全部熔断"分支）。模板字符串把 `null` 拼成字符串，**实际发出 `Authorization: Bearer null`**：

1. 服务端返回 401；
2. 401 触发 `penalize(undefined)`——把 `undefined` 写进 cooldown Map；
3. 用户看到的是"翻译失败"，而不是文档 §4.4 承诺的 `translate:quotaExhausted` 明确提示。

另外 `penalize(/* 当前 key */)` 是注释占位，**当前用的是哪把 Key 在 catch 里根本拿不到**，这一行现在无法实现。

**修法**：

```ts
const key = keyRotator.next();
if (!key) throw new NoAvailableKeyError();          // 专用错误，UI 直接映射 quotaExhausted
const body = JSON.stringify(this.buildBody(o, text));
...
  if (e instanceof HttpError && (e.status === 401 || e.status === 403)) keyRotator.penalize(key);
```

`NoAvailableKeyError` 不继承 `HttpError`，避免被 `retryable` 逻辑吞掉后反复重试一把已经烧掉的 Key。

---

### P0-4 · 并发只降不升，最终锁死在 1

**位置**：§4.4 `onThrottle()` / `onStable()`（L1038–1050）+ §12.5 `call()`（L2775）

两个独立缺陷叠加成同一个事故：

**4a — 5xx 被当成限流。** `call()` 里 `else if (e.retryable) translateQueue.onThrottle();`，而 `HttpError.from()` 把 `429 || 408 || >=500` 全标成 `retryable`。于是一次服务端 500 抖动（与 QPS 无关）就会把并发腰斩。3 次抖动后 `maxConcurrent` 到 `min=1`，**全应用退化成串行翻译**。

**4b — `onStable()` 没有任何调用点。** 全文检索 `onStable` 只出现在定义处（L1045），`run()` 成功后没有任何代码触发它。也就是说 R19 自适应在实现上是**单向阀**：只降不升，"稳定 60s 回升 +1"永远不执行。

**修法**：

```ts
// 4a：只有限流才降并发，5xx / 408 只重试不动并发
if (e instanceof HttpError) {
  if (e.status === 401 || e.status === 403) keyRotator.penalize(key);
  else if (e.status === 429) translateQueue.onThrottle();   // 唯一降并发的信号
}

// 4b：成功路径必须有人喂
private okStreak = 0;
onSuccess() {
  if (++this.okStreak < 20) return;      // 连续 20 次成功才算"稳定"
  this.okStreak = 0;
  this.onStable();
}
// pump 内：job.run().then(() => this.onSuccess()).finally(...)
```

**怎么验**：注入 3 次 500，断言 `maxConcurrent` 仍为 4；再连续 20 次成功、推进 60s 假时钟，断言 `maxConcurrent` 回到 5。

---

### P1-1 · 已 abort 的 signal 不会触发回调

**位置**：§4.2 `httpJson`（L687–690）、§4.2.1 `httpStream`（L740–742）

```ts
const onAbort = () => req.destroy(new Error('已取消'));
options.signal?.addEventListener('abort', onAbort, { once: true });
```

**为什么错**：`addEventListener` 只对**之后**发生的 abort 生效。若用户点"取消翻译"时请求还在队列里排队、尚未走到这一行（排队 200 条时这是常态），signal 已经 `aborted=true`，监听器注册上去**永远不会被调用**，请求照发、照计费，UI 显示已取消。

**修法**（两个函数都要加，且在**建连之前**）：

```ts
if (options.signal?.aborted) {
  return reject(new HttpError(499, 'aborted before dispatch', false));   // httpJson
}
```

`httpStream` 侧同理，在建 `req` 之前 `h.onError(...)` 后直接 return。499 单独定义，语义是"客户端主动取消"，不重试、不计入熔断。

---

### P1-2 · C-1 只修了流式，非流式仍会被心跳拖死

**位置**：§4.2 `httpJson`（L667）

```ts
timeout: options.timeoutMs ?? 8000,   // 注意：socket 空闲超时，非总时长
```

注释自己承认了，但**没有补 deadline**。C-1 的整段论证（"心跳不断重置计时器 → 槽位被永久占用 → 队列假死"）对 `httpJson` **同样成立**：只要服务端以小于 8s 的间隔持续输出字节（keep-alive 空格、分块 JSON），`timeout` 永不触发，而 `TranslateQueue.active` 会一直被占。§4.4 说的"单条超时 8s"在实现上不成立。

**修法**：抽一个 `armDeadline()` 给两个函数共用，非流式也按 `deadlineMs`（默认 15s）绝对计时，与 `timeout`（空闲）并存。

**顺带**：`httpJson` 没有响应体大小上限，一个 10MB 的错误页会被整个拼进 `data` 字符串再 `slice(0,200)` 报错。加 `MAX_BODY = 1MB`，超限即 `req.destroy()`。

---

### P1-3 · `selectRoute()` 读取的类型上不存在的字段

**位置**：§12.5（L2709–2717）对照 `TranslateOpts`（L2672–2689）

```ts
if (!caps.referencesAcceptTerms && o.customerGlossaryHit) return 'chat';   // ← 字段不存在
if (o.wantOfficialContext && !caps.contextFieldConfirmed) return 'chat';    // ← 字段不存在
```

`TranslateOpts` 只有 `from / to / model / glossary / glossaryIds / contextText / contextTurns / references / style / stream / signal`。这两行在 TS 下是 `TS2339: Property does not exist`，**当前代码无法编译**。

**修法**：要么把两个字段补进 `TranslateOpts`（并规定由谁填），要么把 `selectRoute` 的入参显式写成：

```ts
function selectRoute(
  o: TranslateOpts,
  flags: { customerGlossaryHit: boolean; wantOfficialContext: boolean },
  caps: ApiCaps,
): 'openapi' | 'chat'
```

推荐后者：`selectRoute` 是个纯函数，把隐式依赖显式化，单测才好写。

**这一条是"7 处修复从未编译过"的直接证据**——补丁是照着文档写的，不是照着编译器写的。

---

### P1-4 · `normalizeContext()` 定义了却没人调用，上下文静默丢失

**位置**：§12.5（L2696 定义，L2742 `buildBody` 使用）

注释说"调用方只管往上塞 turns"，但 `buildBody` 里 openapi 分支写的是 `context: o.contextText`，而 `contextText` 由调用方填、`contextTurns` 才是"只管往上塞"的那个。两者之间**没有任何代码桥接**，`normalizeContext` 全文零调用。

实际后果：A1 的上下文翻译在 §0 第 7 项实测通过后也**不会生效**——`context` 恒为 `undefined`，服务端按无上下文处理，译文质量与不传完全一样，而且没有任何报错。这是"静默失效"，比报错难查一个量级。

**修法**：

```ts
buildBody(o: TranslateOpts, text: string): unknown {
  const route = this.route(o);
  const ctx = normalizeContext(o, route);          // ← 唯一入口
  return route === 'openapi'
    ? { model: o.model, text, target: o.to, source: o.from,
        context: ctx.text, glossary_ids: o.glossaryIds,
        references: o.references, stream: o.stream ?? false }
    : { model: o.model, stream: o.stream ?? false,
        messages: this.buildMessages(o, text, ctx.turns) };
}
```

**怎么验**：用例直接断言——`contextTurns` 非空时，openapi 分支的 `body.context` 必须等于三行 `role: text` 拼接串。当前代码这条断言拿到的是 `undefined`。

---

### P1-5 · 同文本去重：被合并的节点既没兜底，也不会升优先级

**位置**：§4.4 `push()`（L1052–1058）

```ts
if (this.queue.some(j => j.key === job.key)) return;   // 直接丢弃
```

三个后果：

1. **无兜底**：注释说"结果会由先到的那个任务写入共享缓存"，但先到的任务可能被 30s 过期清理掉（`applyBackpressure`），也可能失败。一旦如此，**被合并的那个节点永远不会被翻译，也没有重试入口**。
2. **不升优先级**：先到的是 P2（后台账号），用户随后切到该会话，同名任务因去重被丢弃，P0 的渲染仍在等 P2 的请求——体验上就是"切过去还是不翻"。
3. **无回调**：调用方拿到的是 `void`，无法知道自己的任务被吞了。

**修法**：去重不是"丢弃"，是"合并 + 登记"。

```ts
private byKey = new Map<string, Job>();

push(job: Job) {
  const exist = this.byKey.get(job.key);
  if (exist) {
    exist.priority = Math.min(exist.priority, job.priority) as Priority;  // 只升不降
    exist.waiters.push(job.run);                                          // 结果共享
    return;
  }
  this.byKey.set(job.key, job);
  this.queue.push(job);
  this.applyBackpressure();
  this.pump();
}
```

任务结算时 `byKey.delete(key)`，并统一 resolve 所有 `waiters`；失败时 clear 掉半截译文再交给 UI 的"点击重试"。

---

### P1-6 · 引擎层直接调队列的降并发

**位置**：§12.5 `call()`（L2775）

`call()` 是"一次翻译调用"，`onThrottle()` 是"集群级的并发决策"。放在一起有两个问题：换引擎（DeepL / Google）时这段逻辑要复制一遍，且各引擎对 429 的解读不同；更麻烦的是**单测无法隔离**——测 `call()` 就必然改全局并发状态。

**修法**：`call()` 只 `throw`，由 `TranslateQueue` 的 `run()` 包装层统一 catch 并决策。这也和 §4.2 那句"引擎不自行判断重试"一致——**现在这句话在代码里没有兑现**。

---

### P2-1 · 中文原文下 `\b` 词边界基本失效

**位置**：§4.2.1 B-1 术语命中规则（L872）

> 先做**词边界匹配**（`\b` / Unicode 分词）

`\b` 的语义是"`\w` 与非 `\w` 的交界"，而汉字在 JS 正则里属非 `\w`。所以对中文术语（"起订量"、"交期"），`\b起订量\b` 在 "这个起订量是多少" 里**匹配不上**——前后都是汉字，没有边界。

出站方向（中→外）恰好是术语最需要命中的方向（客户级术语多为中文）。建议：

- 源语言为 CJK 时改用**最长匹配 + 前后字符排除**（前后一个字符不是同语种汉字即可），或直接上 Aho-Corasick；
- `\b` 只用于拉丁字母术语（"LED"、"MOQ"），且必须配 `case_sensitive` 开关；
- 把"中文术语命中率"写进 §14.2 的验收指标（用 20 条真实术语跑一遍，命中率应 ≥ 95%）。

---

### P2-2 · 15s 固定 deadline 与"仅长句启用流式"互相打架

**位置**：§12.5 `call()`（L2769）`deadlineMs: 15_000`；§12.5 缺口 3（L2789）"仅长句启用、总时长 15s 封顶"。

逻辑闭环是断的：**长句才走流式，而长句恰恰最可能超过 15s**。plus / pro 档译 300 字，首字 2s + 逐字 20 字/秒，就要 17s。于是"最需要流式的场景"稳定触发 `deadline exceeded`，且按 §4.2.1 规则 `retryable=false`，用户只能手动点重试，**重试还是超时**——功能等于不可用。

**修法**：deadline 随输入长度伸缩，并设上限：

```ts
const deadlineMs = Math.min(60_000, 8_000 + text.length * 15);   // 300 字 ≈ 12.5s，1000 字 ≈ 23s
```

同时把"超时"从"不可重试"改为"可重试一次"：deadline 命中往往只是这台机器/这条链路慢，一次重试的成本远低于让用户放弃。

---

### P2-3 · 5s 空闲超时可能砍掉正常的慢首字

**位置**：§12.5 `call()`（L2769）`idleTimeoutMs: 5_000`

流式的空闲超时同时覆盖了**首字等待期**。plus / pro 在长上下文 + 术语表下首字 6–8s 是正常量级，5s 会把它判成"idle timeout"→ 重试 → 再超时。文档 §14.2 把"首字延迟 P95"列为待实测项，**在实测出来之前，5s 这个数没有依据**。

**修法**：首字到达前用 `firstByteTimeoutMs`（建议先取 12s，实测后收紧），首字到达后切到 `idleTimeoutMs=5s`。两个计时器状态机切换，只多 10 行代码。

---

### P2-4 · 408 同时是"可重试"和"不可重试"

**位置**：§4.2 错误表（L700–704）vs §4.2.1 流式重试边界（L863）

- 表里：`429 / 408 / 5xx → retryable=true`；
- 流式：`HttpError(408, 'stream deadline exceeded', false)`，并写明"按 retryable=false 结算"。

同一个 status 两种语义，全靠调用方**读字段而不是读 status**才不出错。但 §4.4 的熔断描述又是"`HttpError.status` 为 401/403 立即 penalize"——文档自己就在混用两种判据。

**修法**：给 `HttpError` 加 `kind: 'http' | 'idle' | 'deadline' | 'abort' | 'network'`，消费方一律读 `kind` + `retryable`，禁止读 `status` 做分支（401/403 用 `kind==='http' && status` 判断）。诊断日志按 `kind` 聚合，能直接算出"超时里有多少是 deadline、多少是 idle"——这正是 S2 要回填的数据。

---

### P2-5 · 两套排期并行

§5.1–5.3 给出 4 + 5~6 + 3~5 = **12–15 人天**的实现排期；§13.2 给出 S1 0.5 + S2 3–5 + S3 2 = **5.5–7.5 天**的验证排期，并在 §12.6 声明"排期以 §13.2 为准"。但 §13.2 只覆盖**验证**，不含验证通过后的**开发**。结果是：老板问"多久能交"，文档里有两个互不引用的答案。

**修法**：§5 的工期明确标注为"spike 通过后的实现估算（不含 S1–S3）"，或直接合并成一条时间轴：S1(0.5d) → S2(3–5d) → S3(2d) → 实现(12–15d)。**一条轴，两个刻度，不要两份表。**

---

## 3. 三个方法论问题（比单条 bug 更贵）

### 3.1 "已修复"是文档状态，不是代码状态

v1.8 的 7 处修复里有 4 处引入了新错误（P0-1 比较符、P0-3 null 拼接、P1-3 字段不存在、P1-4 函数不接线）。共同特征：**改动都是在文本里完成的，从未经过编译器**。

这不是能力问题，是流程问题。**任何写进"冻结版"的 TypeScript，都必须能通过 `tsc --noEmit`**，否则它只是伪代码，不该承担"设计已冻结"的信用。建议加一条硬规则：`§4.2 / §4.4 / §12.5 的代码块必须与 `packages/core` 下的真实文件逐字一致`，文档引用文件而不是内嵌副本——**内嵌副本必然腐化**，这是第 8 轮也一样。

### 3.2 修复是"点状"的，不是"类"的

| 修复项 | 修了什么 | 漏了什么 |
|---|---|---|
| C-1 绝对 deadline | 只修流式 | 非流式同样会被心跳拖死（P1-2） |
| C-2 结构化错误 | 定义了 `retryable` | 408 两种语义、消费方仍在读 status（P2-4） |
| C-4 AbortSignal | 只注册监听 | 未检查 `aborted` 前置状态（P1-1） |
| C-7 优先级与背压 | 加了字段和注释 | 比较符反了，功能与注释相反（P0-1） |

**判断标准应该从"这个 bug 修了吗"换成"这类 bug 还有几处"**。建议 v2.0 的修订表加一列"同类点排查结果"，逼着每次修复都横向扫一遍。

### 3.3 文档里约五分之一是"评审往返记录"

2,922 行里，"评审 C-8 驳回""评审自述只读到 §4.3""按评审意见纠正定位"这类元讨论占了相当篇幅。对**新读者**（比如两周后接手的人）来说，这些内容是纯噪音：他要的是"现在是什么"，不是"第几轮谁说了什么"。

**建议**：正文只留当前结论与代码；所有"某轮评审说了什么 / 我为什么驳回"移进 `CHANGELOG.md` 或 `docs/reviews/`。§13.4 缺陷池保留在正文，但只保留**未关闭项**，关闭的直接删。

---

## 4. 修改版建议：v2.0 该做的 6 个结构性决定

不重写文档。v2.0 只需要在以下 6 处做**一次性决定**，其余章节照搬：

| # | 决定 | 影响范围 |
|---|---|---|
| D1 | **超时四层模型**：`connect(3s) / firstByte(12s) / idle(5s) / deadline(按长度 8s+15ms·n，上限 60s)`，流式与非流式共用 | §4.2、§4.2.1、§4.4、§12.5 |
| D2 | **错误模型**：`HttpError{status, kind, retryable, retryAfterMs, body}`，消费方只读 `kind`/`retryable`；新增 `NoAvailableKeyError` | §4.2、§4.4、§12.5 |
| D3 | **缓存键**：`engine\|model\|from\|to\|glossaryScope\|style\|ctxHash\|text`，且规定"带上下文只写不读" | §4.4 |
| D4 | **队列四件事**：比较符修正、去重登记 waiter、优先级只升不降、成功计数驱动回升 | §4.4 |
| D5 | **类型收敛**：`TranslateOpts` 与 `selectRoute` 签名对齐，`normalizeContext` 成为 `buildBody` 唯一入口 | §12.5 |
| D6 | **排期单轴**：S1→S2→S3→实现，一条时间轴 | §5、§13 |

这 6 项落地后，本文 15 条缺陷关闭 13 条；剩余 P2-1（CJK 分词）与 P2-3（首字超时）依赖实测，按 §13.3 回填。

---

## 5. S1 可执行验证包：把七行表变成七条命令

> **说明**：以下命令是**骨架**，参数名以控制台/官方文档为准——**参数名本身正是 §0 要确认的东西**。命令会向你的账号发起真实（可能计费）请求，请在测试 Key 下执行。所有结果直接填进 §13.2 的表。

```bash
# 0. 环境（Key 只放环境变量，不要写进 .sh、不要进 shell history）
export TH_KEY='<你的 TokenHub 测试 Key>'
BASE_MAAS='https://tokenhub.tencentmaas.com'
BASE_API='https://tokenhub.tencentcloudapi.com'
```

**S1-2 / S1-3 / S1-5：端点存在性 + 鉴权方式 + lite 支持（一条命令三件事）**

```bash
probe() {   # $1=鉴权头  $2=model
  curl -sS -o /tmp/r.json -w "HTTP %{http_code}  %{time_total}s\n" \
    -X POST "$BASE_MAAS/v1/api/translations" \
    -H "$1" -H 'Content-Type: application/json' \
    -d "{\"model\":\"$2\",\"text\":\"Please confirm the MOQ.\",\"source\":\"en\",\"target\":\"zh\"}"
  head -c 400 /tmp/r.json; echo; echo '---'
}

probe "Authorization: Bearer $TH_KEY" hy-mt2-lite    # 判定：200 → 端点存在 + Bearer 可用 + lite 可用
probe "Authorization: Bearer $TH_KEY" hy-mt2-plus    # 对照组：若 plus 通而 lite 4xx → modelSupportsOpenApi['lite']=false
```

TC3 签名**不要手写**（签名串错一个字节就 401，会误判成"鉴权失败"）；用 `tccli` 或 `tencentcloud-sdk-nodejs` 的 `commonClient` 发同一请求，对比状态码：

```bash
tccli tokenhub CreateGlossary --cli-unfold-argument --Name probe-$(date +%s) --Source zh --Target en
```

**S1-6：`glossary_ids` 的"最多 10 个"到底是报错还是静默截断**

```bash
# 先用 CLI 建 11 个术语库，取回 11 个 GlossaryId 填进数组
IDS=$(python3 -c "print(','.join(['\"gls-REPLACE-%02d\"'%i for i in range(1,12)]))")
curl -sS -o /tmp/r6.json -w "HTTP %{http_code}\n" \
  -X POST "$BASE_MAAS/v1/api/translations" \
  -H "Authorization: Bearer $TH_KEY" -H 'Content-Type: application/json' \
  -d "{\"model\":\"hy-mt2-plus\",\"text\":\"MOQ 500 pcs\",\"source\":\"en\",\"target\":\"zh\",\"glossary_ids\":[$IDS]}"
# 判定：400/422 → 硬上限；200 且译文用了第 11 个术语的译法 → 超出被采纳（上限是软建议）
```

**S1-7：`context` 三种格式，谁的承接质量最好**

```bash
ctx_test() {  # $1=context 内容
  jq -n --arg c "$1" '{model:"hy-mt2-plus",text:"ok, ship it",source:"en",target:"zh",context:$c}' > /tmp/b.json
  curl -sS -X POST "$BASE_MAAS/v1/api/translations" \
    -H "Authorization: Bearer $TH_KEY" -H 'Content-Type: application/json' \
    --data-binary @/tmp/b.json | jq -r '.translation // .text // .'
}
ctx_test '客户问交期，业务答下周发'                       # A 纯文本
ctx_test 'user: 什么时候能发货？
assistant: 下周三'                                        # B 角色前缀
ctx_test '{"role":"user","text":"什么时候能发货？"}'      # C JSON 串
# 判定：谁的"ship it"译出了"发货"语义（而不是"装船/运输"），就走哪种
```

**S1-4：并发上限（阶梯加压，记录首个 429）**

```bash
for P in 2 4 8 16 32; do
  echo "== -P $P =="
  seq 1 $P | xargs -P $P -I{} curl -sS -o /dev/null -w '%{http_code}\n' \
    -X POST "$BASE_MAAS/v1/api/translations" \
    -H "Authorization: Bearer $TH_KEY" -H 'Content-Type: application/json' \
    -d '{"model":"hy-mt2-lite","text":"hi","source":"en","target":"zh"}' | sort | uniq -c
done
# 判定：首个出现 429 的档位 - 1 就是 R19 的 ceiling（不是 8，8 是猜的）
```

**回填**：把每条的输出贴进 §13.2 的表；按 §13.3 规则 2，**只改结论不改版本号**。

---

## 6. 附：截图那段回复的三处更正

你转来的那段回复里有三句话需要更正，否则会误导后续决策：

**① "文心未必发得出这种请求（它是 APP 里的助手，不一定有任意 HTTP 调用能力）"。**
能。本次评审过程中我已实际调用 GitHub API：`GET /repos/ouruicom/test/contents/` 返回 200，取到 `plan-v1.8.md`（179,242 字节，sha `bd08fb5718…`）。判断可以基于事实，不必基于推测。

**② "这串只动得了这一个空仓库"。**
两点不准确：一是这个 token 是 `github_pat_` 开头的细粒度令牌，**权限范围由创建时勾选的 repository 和 permissions 决定**，从字符串前缀推不出来（本串能否写其它仓库，只有 GitHub 的 token 设置页能回答）；二是**仓库不是空的**，里面已有 `plan-v1.8.md` 与 `clash` 两个文件。

**③ 安全建议不完整。**
"别贴到群或公开页面"只说对了一半——**贴进任何第三方 AI 对话框，等同于贴进第三方日志**。建议现在就做三件事：去 Settings → Developer settings 把该 token **revoke**；换一把 fine-grained、只勾 `Contents: Read and write`、只授本仓库、**过期时间设 7 天**的新 token；以后传文件由本机 `curl` 直接发，不经任何对话框。

---

## 7. 本次交付说明

| 项 | 状态 |
|---|---|
| 上传路径 | `review-wenxin-v1.md`（按你给的 URL） |
| 是否覆盖 `plan-v1.8.md` | **否**，该文件未做任何写操作，sha 仍为 `bd08fb5718…` |
| 该文件此前是否存在 | **不存在**（`GET .../review-wenxin-v1.md` 返回 404），本次为新建，无覆盖风险 |
| 关于"下一版叫 v2" | 若你要保留本文作为 v1 存档、下一轮另发 v2，告诉我，我按 `review-wenxin-v2.md` 再出一份；本次按你指定的 v1 路径交付 |

---

*本文只做两件事：把"已修复"里没修好的地方指出来，把"待实测"变成可以今天就跑的命令。第 8 轮不该是又一份 3000 行文档，而该是一次 `tsc --noEmit` 和七条 `curl`。*
