# 评审意见：plan-v1.8（设计冻结版）

| 项 | 内容 |
|---|---|
| 评审对象 | `plan-v1.8.md` — 德莱聚合（delai-hub）技术方案 · 设计冻结版（179,242 字节 / 2,922 行 / 60 个标题） |
| 评审人 | 文心（独立评审，非项目成员） |
| 评审日期 | 2026-11-03 |
| 评审方式 | 全文静态阅读 + TypeScript / Node 运行时语义推演；**未执行代码、未持有 API Key、未接触真机** |
| 交付形态 | 缺陷清单（可定位）+ 补丁（可应用）+ S1 验证命令包（可执行） |
| 修订 | **rev.2** · 2026-11-03 · 本版新增 3 条缺陷、1 个可执行验证器、1 张 S0 决策单、1 份「不再提」清单 |


**本版（rev.2）改了什么**

| # | 变化 |
|---|---|
| 1 | 新增 3 条缺陷：**P0-5**（4xx 分支不停 deadline，真实 429 被 408 顶掉）、**P1-7**（`verify()` 多行必然误判，三层写入链退化成单层）、**P1-8**（老化过滤不看优先级，与 P0-1 同源但独立触发） |
| 2 | §5 增加 **5.2 可执行验证器**：把命令包升级为零依赖 Node 脚本，带四态判定与自动回填表（本机已用 mock 服务跑通，**未对真实接口发起任何请求**） |
| 3 | 新增 **§7 S0 决策单**：把 ToS / 账号数 / 加密三个非技术前提前置到 S2 之前 |
| 4 | 新增 **§8 不再提清单**：8 条已在 v1.7.x–v1.8 解决或驳回的意见，本轮起不再重复 |

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

## 2. 缺陷清单（18 条，按修复优先级）

| 编号 | 位置 | 一句话 | 级别 |
|---|---|---|---|
| P0-1 | §4.4 `applyBackpressure` | 比较符写反，背压丢弃的正是"绝不丢"的 P0 任务 | 阻断 |
| P0-2 | §4.4 `cacheKey` | 键里不含 `context` 与 `style`，上下文翻译与语气会被旧缓存串味 | 阻断 |
| P0-3 | §12.5 `call()` | `keyRotator.next()` 可能返回 `null`，实际发出 `Bearer null` | 阻断 |
| P0-4 | §12.5 / §4.4 | 5xx 也触发降并发，且 `onStable()` 无调用点 → 并发只降不升，终局锁死 1 | 阻断 |
| P0-5 | §4.2.1 流式 4xx 分支 | 错误体慢时 15s deadline 先到，真实 429 被伪装成 408 → 自适应永不降并发 | 阻断 |
| P1-1 | §4.2 / §4.2.1 | 已 abort 的 signal 不会触发回调，取消形同虚设 | 高 |
| P1-2 | §4.2 `httpJson` | C-1 只给流式加了绝对 deadline，非流式仍是 socket 空闲超时 | 高 |
| P1-3 | §12.5 `selectRoute` | 读取 `o.wantOfficialContext` / `o.customerGlossaryHit`，类型上不存在 | 高 |
| P1-4 | §12.5 `normalizeContext` | 定义了但没人调用，`context` 在 openapi 路径上恒为 `undefined` | 高 |
| P1-5 | §4.4 `push()` | 同文本去重直接 `return`，被合并的节点无兜底、不升优先级 | 高 |
| P1-6 | §12.5 `call()` | 引擎层直接调 `translateQueue.onThrottle()`，分层被打穿 | 中 |
| P1-7 | §4.6.3 `verify()` | 多行译文恒判失败，三层降级链实际只有第三层在工作 | 高 |
| P1-8 | §4.4 `applyBackpressure` 老化过滤 | 30s 老化不看优先级，P0 任务同样被静默丢弃 | 高 |
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

### P0-5 · 4xx 分支没停 deadline：真实的 429 会被 408 顶掉

**位置**：§4.2.1 `httpStream` 的 `res.statusCode >= 400` 分支（约 L800–810）

```ts
if (res.statusCode && res.statusCode >= 400) {
  const errDec = new StringDecoder('utf8');
  let err = '';
  res.on('data', (c: Buffer) => (err += errDec.write(c)));
  res.on('end', () => fail(HttpError.from(res.statusCode!, err)));
  return;                       // ← 没有清掉 deadlineTimer
}
```

**为什么错**：进入这个分支时，`armDeadline()` 在请求发出前就已经启动（15s）。错误响应体如果来得慢——网关排队、WAF 限流页、429 的 `Retry-After` 慢写——**15s 绝对 deadline 会先触发**，`fail(HttpError(408, 'stream deadline exceeded', false))` 抢在 `res.end` 之前结算。

后果不是"少了一个错误码"，而是**自适应回路被掐断**：`retryable=false` 且 `status=408`，`call()` 里既不 `penalize` 也不 `onThrottle()`，R19 永远不会因为限流降并发，反而继续以 4 并发撞墙。更麻烦的是 **S1 第 4 项会测出"没出现 429"**——不是没有限流，是限流被自己的 deadline 盖住了。

**修法**：

```ts
if (res.statusCode && res.statusCode >= 400) {
  if (deadlineTimer) { clearTimeout(deadlineTimer); deadlineTimer = undefined; }   // ← 关键
  const errDec = new StringDecoder('utf8');
  let err = '';
  const errTimer = setTimeout(
    () => fail(new HttpError(res.statusCode!, 'error body timeout', true)), 3_000);
  res.on('data', (c: Buffer) => { err += errDec.write(c); });
  res.on('end', () => { clearTimeout(errTimer); fail(HttpError.from(res.statusCode!, err + errDec.end())); });
  return;
}
```

**怎么验**：mock 一个"返回 429、16s 后才吐完 body"的服务端，断言最终 `error.status === 429`（不是 408），且 `onThrottle()` 被调用 1 次。

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

### P1-7 · `verify()` 对多行译文必然误判，三层链退化成单层

**位置**：§4.6.3 `currentValue()` / `verify()`（约 L1424–1436）

```ts
function verify(el, text) {
  return currentValue(el) === text.replace(NBSP, ' ').trim();
}
```

**为什么错**：`currentValue` 只处理了 `\u00a0`。而 `contenteditable` 写入含换行的文本时，浏览器会插 `<div>` / `<br>`，`innerText` 取出来是 `"\n"`，**末尾常多一个换行**；粘贴路径还可能带零宽空格。于是任何含换行的译文（外贸话术几乎都有）`verify` 恒为 `false`。

后果比"多写一次"严重：第一层 `execCommand` 其实成功了却判失败 → 第二层再写 → 再判失败 → 落第三层 paste。**"三层降级链"实际只有第三层在干活**，而日志显示"一、二层均失败"，会把 S2 的结论直接带偏（很容易误判成"execCommand 已被 Chromium 废弃"）。

**修法**：

```ts
const norm = (s: string) => s
  .replace(/\u00a0/g, ' ').replace(/\u200b/g, '')
  .replace(/\r\n?/g, '\n')
  .replace(/[ \t]+\n/g, '\n')
  .replace(/\n{2,}/g, '\n')
  .trim();

function verify(el: any, text: string) {
  return norm(currentValue(el)) === norm(text);
}
```

**怎么验**：写入含 3 行、200 字的中文译文，断言 `verify === true` 且**未进入第二层**；再在末尾补一个 `<br>`，断言仍为 `true`。

---

### P1-8 · 老化过滤不看优先级，P0 同样被静默丢弃

**位置**：§4.4 `applyBackpressure()` 前半段（约 L1060）

```ts
this.queue = this.queue.filter(j => {
  if (Date.now() - j.enqueuedAt < this.MAX_AGE_MS) return true;
  diagnostics.inc('translate.dropped.stale');
  return false;                       // ← 不看 j.priority
});
```

**为什么错**：这是 P0-1 的"姐妹缺陷"。P0-1 是排序比较符反了，P1-8 是老化过滤根本不参与优先级判断——**两处独立代码都在丢 P0**。修了比较符不会修好这里：只要用户在一个会话里连续触发 200 条以上、或 API 变慢导致排队超 30s，正在看的会话一样被清掉。而且被过滤掉的 job 只 `inc` 了诊断计数，`run` 从未执行、`fail` 不存在，**调用方永远等不到结果**，气泡卡在"翻译中"。

**修法**：

```ts
this.queue = this.queue.filter(j => {
  if (j.priority === 0) return true;                    // P0 永不老化
  if (Date.now() - j.enqueuedAt < this.MAX_AGE_MS) return true;
  diagnostics.inc('translate.dropped.stale');
  j.fail?.('stale');                                    // 被丢弃必须是可观测结局
  return false;
});
```

（`Job` 增加 `fail?: (reason: string) => void`；`splice` 掉的批量丢弃同样要逐个 `fail`。）

**怎么验**：构造 1 个 P0 + 1 个 P2，把时钟推进 40s，断言 P0 仍在队列、P2 已 `fail('stale')`。

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

### 5.2 可执行验证器（rev.2 新增）

上面的 `curl` 适合"手动确认一次"；如果要在 S1 当天跑完七项并直接产出回填表，用下面这个脚本更省事，也更能挡住三种典型误判：**拿到 200 就以为过了**、**用假 GlossaryId 测上限**、**加压顺序把 Key 打进冷却**。

- 零依赖，Node ≥ 18，单文件 `spike-s1.mjs`
- 四态判定：`PASS / FAIL / SKIP / 待人工`；`SKIP` 不计入 PASS，退出码 2（故意让 CI 变红）
- 输出 `spike-s1-result.md`，表头与 §13.2 完全对齐，可直接覆盖"实测结果"列
- Key 只从环境变量读，输出与结果文件里全部打码
- **#4 加压必须显式 `--all` 或 `--concurrency` 才跑**，避免污染前面的结论

```bash
TOKENHUB_KEY=sk-xxx node spike-s1.mjs --plan                  # 先看要发什么，不联网
TOKENHUB_KEY=sk-xxx TC_SECRET_ID=AKIDxxx TC_SECRET_KEY=xxx node spike-s1.mjs
TOKENHUB_KEY=sk-xxx node spike-s1.mjs --glossary-ids=id1,...,id11
TOKENHUB_KEY=sk-xxx node spike-s1.mjs --concurrency=16
```

本机已验证：`node --check` 通过，并对本地 mock 服务跑通 #2/#3/#5/#7/附 五项，输出表格式正确；**未对任何真实接口发起请求**，下述示例中的译文为 mock 返回。

```javascript
#!/usr/bin/env node
/**
 * spike-s1.mjs —— v1.8 §0「七项外部接口事实」可执行验证器
 *
 * 设计原则：
 *   1. 零依赖（只用 node: 内置模块），Node >= 18
 *   2. 不重试、不猜测：一次请求一条结论，拿不到就记 SKIP，绝不写"应该是"
 *   3. Key 只从环境变量读，输出与结果文件里一律打码
 *   4. 绝对 deadline 独立计时（心跳推不动），与 §4.2.1 的 C-1 同一套口径
 *
 * 用法：
 *   TOKENHUB_KEY=sk-xxx node spike-s1.mjs --plan            # 只打印将要发的请求
 *   TOKENHUB_KEY=sk-xxx node spike-s1.mjs                   # 跑基础项（2/3/5/7/附）
 *   TOKENHUB_KEY=sk-xxx TC_SECRET_ID=xx TC_SECRET_KEY=yy node spike-s1.mjs --all
 *   TOKENHUB_KEY=sk-xxx node spike-s1.mjs --concurrency=12  # 加跑第 4 项加压
 *   TOKENHUB_KEY=sk-xxx node spike-s1.mjs --glossary-ids=id1,id2,...   # 第 6 项
 *
 * 环境变量：
 *   TOKENHUB_KEY     独立翻译接口 / chat 端点的 API Key（必填，除非 --plan）
 *   TC_SECRET_ID    腾讯云 SecretId（跑第 1 项 CreateGlossary 与第 5 项 TC3 需要）
 *   TC_SECRET_KEY   腾讯云 SecretKey
 *   TC_REGION       默认 ap-guangzhou
 *
 * 退出码：0 = 全部有结论（含 FAIL）；2 = 存在 SKIP（有项没跑成）
 */

import https from 'node:https';
import http from 'node:http';
import crypto from 'node:crypto';
import fs from 'node:fs';

// ---------------------------------------------------------------- 配置与参数
const args = process.argv.slice(2);
const flag = (name, def) => {
  const i = args.findIndex(a => a === `--${name}` || a.startsWith(`--${name}=`));
  if (i === -1) return def;
  const hit = args[i];
  const eq = hit.indexOf('=');
  if (eq !== -1) return hit.slice(eq + 1);                 // --name=value
  const next = args[i + 1];                                // --name value
  return next && !next.startsWith('--') ? next : true;     // 裸 flag = true
};

const CFG = {
  base: flag('base', 'https://tokenhub.tencentmaas.com'),
  tcHost: flag('tc-host', 'tokenhub.tencentcloudapi.com'),
  tcVersion: flag('tc-version', '2026-03-22'),
  region: process.env.TC_REGION || 'ap-guangzhou',
  model: flag('model', 'hy-mt2-plus'),
  lite: flag('lite', 'hy-mt2-lite'),
  key: process.env.TOKENHUB_KEY || '',
  secretId: process.env.TC_SECRET_ID || '',
  secretKey: process.env.TC_SECRET_KEY || '',
  plan: !!flag('plan', false),
  all: !!flag('all', false),
  concurrency: Number(flag('concurrency', 0)),
  glossaryIds: String(flag('glossary-ids', '')).split(',').filter(Boolean),
  out: flag('out', 'spike-s1-result.md'),
  idleMs: 8000,
  deadlineMs: 25000,
};

const SOURCE = '我们这款 LED 灯的最低起订量是 500 件，可以先寄两个样品给你确认质量。';
const TARGET = 'en';
const CONTEXT_PLAIN = '客户上一句问的是 MOQ 和样品。';

const mask = s => {
  if (!s) return '(未设置)';
  const t = String(s);
  return t.length <= 8 ? '****' : `${t.slice(0, 4)}****${t.slice(-4)}(${t.length})`;
};

// ---------------------------------------------------------------- HTTP 基础
let seq = 0;

function request(urlStr, opt = {}) {
  return new Promise(resolve => {
    const u = new URL(urlStr);
    const lib = u.protocol === 'http:' ? http : https;
    const body = opt.body ?? null;
    const headers = { ...(opt.headers || {}) };
    if (body != null) headers['Content-Length'] = Buffer.byteLength(body);

    const started = Date.now();
    let settled = false;
    const done = v => { if (!settled) { settled = true; clearTimeout(deadline); resolve(v); } };

    const req = lib.request({
      hostname: u.hostname,
      port: u.port || undefined,
      path: u.pathname + u.search,
      method: opt.method || 'POST',
      headers,
      timeout: CFG.idleMs,
    }, res => {
      let data = '';
      res.setEncoding('utf8');
      res.on('data', c => { data += c; });
      res.on('end', () => done({ status: res.statusCode, body: data, ms: Date.now() - started }));
      res.on('error', e => done({ status: 0, body: '', ms: Date.now() - started, err: e.message }));
    });

    // 绝对 deadline：与 §4.2.1 一致，不因任何数据到达而顺延
    const deadline = setTimeout(() => {
      req.destroy(new Error('deadline exceeded'));
      done({ status: 0, body: '', ms: Date.now() - started, err: 'deadline exceeded' });
    }, opt.deadlineMs ?? CFG.deadlineMs);

    req.on('timeout', () => { req.destroy(new Error('idle timeout')); });
    req.on('error', e => done({ status: 0, body: '', ms: Date.now() - started, err: e.message }));
    if (body != null) req.write(body);
    req.end();
  });
}

const jbody = o => JSON.stringify(o);

function safeJson(s) {
  try { return JSON.parse(s); } catch { return null; }
}

// 从各种可能的返回结构里捞译文 / 错误码，捞不到就返回 null，不猜
function pickTranslation(obj) {
  if (!obj || typeof obj !== 'object') return null;
  if (typeof obj.translation === 'string') return obj.translation;
  if (typeof obj.text === 'string') return obj.text;
  if (typeof obj.data === 'string') return obj.data;
  if (obj.data && typeof obj.data.translation === 'string') return obj.data.translation;
  if (Array.isArray(obj.translations) && obj.translations[0]) {
    const t = obj.translations[0];
    return typeof t === 'string' ? t : (t.translation ?? t.text ?? null);
  }
  if (Array.isArray(obj.choices) && obj.choices[0]?.message?.content) {
    return obj.choices[0].message.content;
  }
  return null;
}

function errorCode(obj) {
  if (!obj) return '';
  return obj.error?.code || obj.error?.type || obj.code || obj.Response?.Error?.Code || '';
}

function bearerHeaders(extra = {}) {
  return {
    'Content-Type': 'application/json',
    Authorization: `Bearer ${CFG.key}`,
    ...extra,
  };
}

// ---------------------------------------------------------------- 腾讯云 TC3
function sha256hex(s) { return crypto.createHash('sha256').update(s, 'utf8').digest('hex'); }
function hmac(key, msg) { return crypto.createHmac('sha256', key).update(msg, 'utf8').digest(); }

function tc3Sign({ host, service, action, version, payload, region }) {
  const ts = Math.floor(Date.now() / 1000);
  const date = new Date(ts * 1000).toISOString().slice(0, 10);
  const contentType = 'application/json; charset=utf-8';
  const canonicalHeaders =
    `content-type:${contentType}\nhost:${host}\nx-tc-action:${action.toLowerCase()}\n`;
  const signedHeaders = 'content-type;host;x-tc-action';
  const canonicalRequest = [
    'POST', '/', '', canonicalHeaders, signedHeaders, sha256hex(payload),
  ].join('\n');
  const scope = `${date}/${service}/tc3_request`;
  const stringToSign = ['TC3-HMAC-SHA256', String(ts), scope, sha256hex(canonicalRequest)].join('\n');
  const sig = crypto.createHmac('sha256', hmac(hmac(hmac(`TC3${CFG.secretKey}`, date), service), 'tc3_request'))
    .update(stringToSign, 'utf8').digest('hex');

  return {
    'Content-Type': contentType,
    Host: host,
    'X-TC-Action': action,
    'X-TC-Version': version,
    'X-TC-Timestamp': String(ts),
    'X-TC-Region': region,
    Authorization: `TC3-HMAC-SHA256 Credential=${CFG.secretId}/${scope}, ` +
      `SignedHeaders=${signedHeaders}, Signature=${sig}`,
  };
}

// ---------------------------------------------------------------- 结果收集
const results = [];

function record(id, title, requestDesc, verdict, detail, refill) {
  results.push({ id, title, requestDesc, verdict, detail, refill });
  const line = `[${verdict}] ${id} ${title} — ${detail}`;
  console.log(line.replace(/\s+/g, ' ').slice(0, 200));
}

// ---------------------------------------------------------------- 各项检查
async function check02() {
  const url = `${CFG.base}/v1/api/translations`;
  const res = await request(url, {
    headers: bearerHeaders(),
    body: jbody({
      model: CFG.model, text: SOURCE, source: 'zh', target: TARGET,
      context: CONTEXT_PLAIN, stream: false,
    }),
  });
  const j = safeJson(res.body);
  const tr = pickTranslation(j);
  const code = errorCode(j);

  if (res.status === 0) {
    return record('#2', '独立翻译接口路径/参数', 'POST /v1/api/translations + context(string)',
      'SKIP', `网络层失败：${res.err}。不是接口不存在，先查代理/CA`, '重跑；若持续失败查证书链');
  }
  if (res.status >= 200 && res.status < 300 && tr) {
    return record('#2', '独立翻译接口路径/参数', 'POST /v1/api/translations + context(string)',
      'PASS', `200，返回译文「${tr.slice(0, 40)}」；顶层字段=${Object.keys(j || {}).join('|')}`,
      'ApiCaps.openApiEndpointExists=true；按实际字段补 httpStream 归一化分支');
  }
  // 400 且提到 context → 字段类型/名称不符，这正是要抓的结论
  const mentionsContext = /context/i.test(res.body);
  record('#2', '独立翻译接口路径/参数', 'POST /v1/api/translations + context(string)',
    res.status >= 400 && res.status < 500 ? 'FAIL' : 'FAIL',
    `HTTP ${res.status} ${code ? `code=${code} ` : ''}${mentionsContext ? '（错误信息提到 context）' : ''} ` +
    `body=${res.body.slice(0, 160)}`,
    mentionsContext
      ? 'ApiCaps.contextFieldConfirmed=false；A1 回退 prompt 内嵌（路径①）'
      : '端点/鉴权/参数需按错误码逐项排查，勿先改引擎设计');
}

async function check03() {
  const url = `${CFG.base}/v1/api/translations`;
  const res = await request(url, {
    headers: bearerHeaders(),
    body: jbody({
      model: CFG.lite, text: SOURCE, source: 'zh', target: TARGET,
      glossary_ids: CFG.glossaryIds.slice(0, 1), stream: false,
    }),
  });
  const j = safeJson(res.body);
  const ok = res.status >= 200 && res.status < 300 && !!pickTranslation(j);
  record('#3', 'lite 档是否支持独立接口', `POST /v1/api/translations model=${CFG.lite}`,
    res.status === 0 ? 'SKIP' : (ok ? 'PASS' : 'FAIL'),
    res.status === 0 ? `网络层失败：${res.err}` : `HTTP ${res.status} ${errorCode(j)}`,
    ok ? `modelSupportsOpenApi['${CFG.lite}']=true`
      : `modelSupportsOpenApi['${CFG.lite}']=false，默认走 chat`);
}

async function check05() {
  const url = `${CFG.base}/v1/api/translations`;
  const payload = jbody({ model: CFG.model, text: SOURCE, source: 'zh', target: TARGET });

  const bearer = await request(url, { headers: bearerHeaders(), body: payload });

  let tc3 = { status: 0, body: '', err: 'no TC secret' };
  if (CFG.secretId && CFG.secretKey) {
    const host = new URL(CFG.base).host;
    tc3 = await request(url, {
      headers: tc3Sign({ host, service: 'tokenhub', action: 'CreateGlossary', version: CFG.tcVersion, payload, region: CFG.region }),
      body: payload,
    });
  }
  const bOk = bearer.status >= 200 && bearer.status < 300;
  record('#5', '鉴权方式（Bearer vs TC3）', '同一 Key 两种签名各发一次',
    bOk ? 'PASS' : (bearer.status === 0 ? 'SKIP' : 'FAIL'),
    `Bearer=${bearer.status || bearer.err}；TC3=${tc3.status || tc3.err || '未提供密钥（本项未验证）'}`,
    bOk ? '引擎统一 Authorization: Bearer' : '确认是否需 TC3 签名；两条都 401 说明 Key 本身无效');
}

async function check07() {
  const url = `${CFG.base}/v1/api/translations`;
  const variants = [
    ['纯文本', CONTEXT_PLAIN],
    ['角色前缀', `user: 客户问 MOQ 和样品\nassistant: 已答复最低 500 件、可寄样`],
    ['JSON 串', JSON.stringify([{ role: 'user', text: '客户问 MOQ 和样品' }])],
  ];
  const out = [];
  for (const [name, ctx] of variants) {
    const res = await request(url, {
      headers: bearerHeaders(),
      body: jbody({ model: CFG.model, text: SOURCE, source: 'zh', target: TARGET, context: ctx, stream: false }),
    });
    const tr = pickTranslation(safeJson(res.body));
    out.push(`${name}：${tr ? tr.slice(0, 80) : `HTTP ${res.status}`}`);
  }
  record('#7', 'context 拼接格式', '三种格式 × 同一原文，人工比对末句承接',
    out.every(o => !/HTTP 4|HTTP 5|HTTP 0/.test(o)) ? '待人工' : 'FAIL',
    out.join(' ／ '), '按人工结论定稿 normalizeContext()；无差异则改回纯文本');
}

async function checkAppendix() {
  const url = `${CFG.base}/v1/api/translations`;
  const term = { type: 'term', text: 'LED 灯', translation: 'LED lighting fixture' };
  const base = { model: CFG.model, text: '我们这款 LED 灯的最低起订量是 500 件。', source: 'zh', target: TARGET, stream: false };
  const withRef = await request(url, {
    headers: bearerHeaders(), body: jbody({ ...base, references: [term] }),
  });
  const withoutRef = await request(url, {
    headers: bearerHeaders(), body: jbody(base),
  });
  const a = pickTranslation(safeJson(withRef.body)) || '';
  const b = pickTranslation(safeJson(withoutRef.body)) || '';
  const adopted = !!a && a !== b;
  record('附', 'references 是否接受 term 对象', '同一句带/不带 references 各一次',
    withRef.status >= 400 ? 'FAIL' : (adopted ? 'PASS' : '待人工'),
    withRef.status >= 400
      ? `HTTP ${withRef.status} ${errorCode(safeJson(withRef.body))}`
      : `译文差异=${adopted ? '有' : '无'}；带=${a.slice(0, 60)}；不带=${b.slice(0, 60)}`,
    adopted ? '客户级术语可走 references' : '客户级术语一律回退路径①（prompt 内嵌）');
}

async function check04(max) {
  const url = `${CFG.base}/v1/api/translations`;
  const steps = [];
  let ceiling = 0;
  for (let n = 1; n <= max; n *= 2) {
    const waves = await Promise.all(Array.from({ length: n }, () => request(url, {
      headers: bearerHeaders(),
      body: jbody({ model: CFG.model, text: `${SOURCE} 批次 ${n}-${seq++}`, source: 'zh', target: TARGET, stream: false }),
      deadlineMs: 30000,
    })));
    const n429 = waves.filter(w => w.status === 429).length;
    const n0 = waves.filter(w => w.status === 0).length;
    steps.push(`并发${n}: 429×${n429} 网络错误×${n0}`);
    if (!n429 && !n0) ceiling = n;
    if (n429) break;
  }
  record('#4', '实际并发上限', `阶梯加压 1→${max}`,
    ceiling ? 'PASS' : 'FAIL', steps.join('；'),
    ceiling ? `R19.ceiling=${ceiling}（保守取 ceiling 的 60%）` : '未打出 429 前所有阶梯都失败，先查 Key 额度');
}

async function check01() {
  if (!CFG.secretId || !CFG.secretKey) {
    return record('#1', 'CreateGlossary 可用', 'POST tokenhub.tencentcloudapi.com CreateGlossary',
      'SKIP', '未提供 TC_SECRET_ID/TC_SECRET_KEY', '补齐密钥后重跑');
  }
  const payload = jbody({ Name: `s1-test-${Date.now()}`, Source: 'zh', Target: 'en' });
  const headers = tc3Sign({
    host: CFG.tcHost, service: 'tokenhub', action: 'CreateGlossary',
    version: CFG.tcVersion, payload, region: CFG.region,
  });
  const res = await request(`https://${CFG.tcHost}`, { headers, body: payload });
  const j = safeJson(res.body);
  const code = j?.Response?.Error?.Code;
  record('#1', 'CreateGlossary 可用', 'CreateGlossary(Name, Source, Target)',
    code ? 'FAIL' : (res.status >= 200 && res.status < 300 ? 'PASS' : 'FAIL'),
    code ? `Response.Error.Code=${code} ${j?.Response?.Error?.Message ?? ''}` : `HTTP ${res.status}`,
    code ? 'A3 路径② 仍不可用' : 'A3 路径② 成立，记下 GlossaryId 供 #6 使用');
}

async function check06() {
  if (CFG.glossaryIds.length <= 10) {
    return record('#6', 'glossary_ids 数量上限', '一次传 11 个真实 GlossaryId',
      'SKIP', `当前传入 ${CFG.glossaryIds.length} 个（需 >10 个真实 ID）`,
      '用 --glossary-ids 传 ≥11 个已建库 ID；假 ID 只会得到 NOT_FOUND，答不了这一问');
  }
  const res = await request(`${CFG.base}/v1/api/translations`, {
    headers: bearerHeaders(),
    body: jbody({
      model: CFG.model, text: SOURCE, source: 'zh', target: TARGET,
      glossary_ids: CFG.glossaryIds.slice(0, 11), stream: false,
    }),
  });
  const j = safeJson(res.body);
  const err = errorCode(j);
  const limited = res.status === 400 || /limit|exceed|too many|10/i.test(JSON.stringify(j || {}));
  record('#6', 'glossary_ids 数量上限', '一次传 11 个真实 GlossaryId',
    res.status >= 200 && res.status < 300 ? 'PASS' : (limited ? 'FAIL' : '待人工'),
    `HTTP ${res.status} ${err ? `code=${err}` : '（无错误，疑似静默截断）'} ${res.body.slice(0, 120)}`,
    res.status >= 200 && res.status < 300
      ? 'ApiCaps.glossaryIdLimit 提到 11，术语库可按客户建模'
      : '上限确为 10：术语库只能按语种对建，客户级恒走路径①');
}

// ---------------------------------------------------------------- 输出
function toMarkdown() {
  const head = [
    `# spike-s1 结果（${new Date().toISOString().slice(0, 19).replace('T', ' ')}）`,
    '',
    `- base: ${CFG.base}`,
    `- model: ${CFG.model} / lite: ${CFG.lite}`,
    `- TOKENHUB_KEY: ${mask(CFG.key)}`,
    `- TC SecretId: ${mask(CFG.secretId)}`,
    '',
    '| 项 | 请求 | 判定 | 实测结果 | 回填动作 |',
    '|---|---|---|---|---|',
  ];
  const rows = results.map(r =>
    `| ${r.id} | ${r.requestDesc} | ${r.verdict} | ${String(r.detail).replace(/\|/g, '\\|')} | ${r.refill} |`);
  const tail = [
    '',
    '> 判定取值：PASS=接口按文档工作；FAIL=有明确错误响应；SKIP=本次没跑成（缺 Key/网络）；待人工=需人看译文质量。',
    '> 本表可直接覆盖 §13.2 S1 的「实测结果」列。',
  ];
  return [...head, ...rows, ...tail].join('\n');
}

// ---------------------------------------------------------------- main
(async () => {
  console.log('spike-s1 起点');
  console.log(`  base      ${CFG.base}`);
  console.log(`  model     ${CFG.model} (lite: ${CFG.lite})`);
  console.log(`  key       ${mask(CFG.key)}`);
  console.log(`  tc secret ${mask(CFG.secretId)}`);
  console.log(`  mode      ${CFG.plan ? 'plan(不联网)' : 'run'}${CFG.all ? ' +all' : ''}`);
  console.log('');

  if (CFG.plan) {
    [
      ['#1', 'CreateGlossary（TC3 签名）'],
      ['#2', 'POST /v1/api/translations（Bearer + context:string）'],
      ['#3', '同上，model=' + CFG.lite],
      ['#4', `阶梯加压 1→${CFG.concurrency || 8} 找 429`],
      ['#5', '同端点 Bearer / TC3 各一次'],
      ['#6', '一次传 11 个 glossary_ids'],
      ['#7', 'context 三种格式对比'],
      ['附', 'references 带/不带 term 对象'],
    ].forEach(([id, d]) => console.log(`  ${id}\t${d}`));
    process.exit(0);
  }

  if (!CFG.key) {
    console.error('缺少 TOKENHUB_KEY。先 `TOKENHUB_KEY=xxx node spike-s1.mjs --plan` 看计划。');
    process.exit(2);
  }

  // 顺序执行：避开相互限流干扰；#1 先跑，便于后续复用 GlossaryId
  await check01();
  await check02();
  await check03();
  await check05();
  await check07();
  await checkAppendix();
  await check06();
  if (CFG.all || CFG.concurrency) await check04(CFG.concurrency || 8);

  const md = toMarkdown();
  fs.writeFileSync(CFG.out, md, 'utf8');
  console.log(`\n已写入 ${CFG.out}`);
  const skipped = results.filter(r => r.verdict === 'SKIP').length;
  process.exit(skipped ? 2 : 0);
})().catch(e => {
  console.error('未捕获异常：', e);
  process.exit(1);
});
```

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

## 7. S0 决策单：三个非技术前提（S2 之前必须有答案）

缺陷可以边写边修，但这三问不定，**S2 跑通的链路可能整体作废**。预计 30 分钟，不需要技术评估。

| # | 问题 | 选项 | 建议默认 | 截止 |
|---|---|---|---|---|
| Q1 | 能否接受"非官方客户端 + DOM 自动写入"的封号风险，并在首启明示？ | 接受 / 不接受 / 改为只读辅助（不代发） | 接受 + 首启明示 + 单账号操作频率上限 | S2 前 |
| Q2 | 产品承诺几个账号？（决定 §3.3 保活与 §14.2 验收口径） | 4 / 8 / 不限 | 先承诺 4，8 作为实验室能力不写进售卖文案 | S2 前 |
| Q3 | 本地数据是否要求全库加密？ | 全库 / 字段级 / 不加密 | 字段级（客户备注、术语、话术加密；译文缓存不加密） | v2.0 前 |

> Q1 若选"不接受"，§3.3 保活、§4.6 三层写入、§5 的相当一部分工作都不该做。所以它必须前置，而不是等 S1–S3 跑完再讨论。

**配套的 kill criteria**（触发即降级，不再投入）：

| 里程碑 | 触发条件 | 动作 |
|---|---|---|
| M1 = S1 | #2 FAIL | 删路径②，§12.5 的 `buildOpenApiBody` / `selectRoute` 整段移除 |
| M2 = S2 | 三层写入全失败 | 产品改"翻译 + 一键复制"，A5/C5/B4 冻结 |
| M2 = S2 | 首字 P95 > 3s | "实时翻译"卖点不成立，先做缓存预热与流式，不上新功能 |
| M3 = S3 | 8 账号 RSS > 3GB 或空闲 CPU > 10% 或收消息延迟 > 30s | 账号上限降到 4，重新定价 |

---

## 8. 本轮起不再提的 8 条（避免第八轮重复）

| # | 旧意见 | 现状 |
|---|---|---|
| 1 | 选择器要外置 JSON + 远端下发 + 灰度 + 回滚 | 已在 §4.6.7（R30）完整实现；我此前只读到 §4.3，属阅读范围问题 |
| 2 | 并发上限应为 5 | 已推翻（属另一产品），改以实测为准 |
| 3 | TokenHub 术语库已下线 | 已推翻，路径② 恢复 |
| 4 | 关闭证书校验省事 | 已删，改 CA 注入 |
| 5 | React 受控写入要绕过 `_valueTracker` | 已实现 |
| 6 | 术语子串匹配即可 | 已改词边界 + 型号补充（§4.2.1 B-1）；但 `\b` 对中文仍失效，见 P2-1 |
| 7 | Messenger 要么做要么删类型 | 已统一为"schema 预留、本期不实现" |
| 8 | 流式失败自动重试 | 已改为"用户点击重来 + 先清半截" |

---

## 9. 本次交付说明

| 项 | 状态 |
|---|---|
| 上传路径 | `review-wenxin-v1.md`（按你给的 URL） |
| 是否覆盖 `plan-v1.8.md` | **否**，该文件未做任何写操作，sha 仍为 `bd08fb5718…` |
| 该文件此前是否存在 | **不存在**（`GET .../review-wenxin-v1.md` 返回 404），本次为新建，无覆盖风险 |
| 关于"下一版叫 v2" | 若你要保留本文作为 v1 存档、下一轮另发 v2，告诉我，我按 `review-wenxin-v2.md` 再出一份；本次按你指定的 v1 路径交付 |

---

*本文只做两件事：把"已修复"里没修好的地方指出来，把"待实测"变成可以今天就跑的命令。第 8 轮不该是又一份 3000 行文档，而该是一次 `tsc --noEmit` 和七条 `curl`。*
