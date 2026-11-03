> 《德莱聚合技术方案 v1.8》分段副本，供单次读取上限 30000 字符的工具读全文。
> 第 3 段 / 共 4 段｜起：4.7 数据持久化（R10、R4 修订）｜止：9.2 操作审计日志（R42，新增）
> 切点按空行对齐，可能从某节中间起止；主文件：
> https://raw.githubusercontent.com/ouruicom/test/main/plan-v1.8.md

---



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