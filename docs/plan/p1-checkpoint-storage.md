# P1 · AG 检查点 —— **存储层**实现方案设计

> 范围：仅存储层（schema / 迁移 / CRUD API / 缓存 / 并发与崩溃安全）。
> 不含 UI 交互实现（跳转链与拦截点见姊妹文档 `docs/plan/p1-checkpoint-navigation.md`）。
> 本文件**只做设计**，未落任何应用代码。全部结论标注【确定】（代码已验证，附 `路径:行号` + 摘录）或【无法确定（需外部构建机核对）】。
> 依据决策：`CONTEXT.md` §2/§4/§9、`docs/adr/0001-ag-checkpoint.md`（§Decision 2：选定 `tdata/ayudata.db` 新增 `Checkpoint` 表，模式照搬 `DeletedMessage`）。

---

## 0. 结论速览（TL;DR）

| # | 结论 | 性质 |
|---|---|---|
| 1 | `AyuDatabase::storage` 是**文件级全局单例**（`ayu_database.cpp:14`），**不是每 Session 一个 DB**；多账号靠 `userId` 列隔离，与 `DeletedMessage` 完全同构 | 【确定】 |
| 2 | DB 在 `core/application.cpp:306`（`Application::run()` 内）就完成 `sync_schema` + 迁移，**早于任何账号/窗口创建**（`:349`）→ 打开 peer 前 DB 必然已就绪 | 【确定】 |
| 3 | 但 `AyuInfra::init()` 时**一个 account 都不存在**，userId 未知 → 预载**不能**在 `initDatabase()` 里按账号做；须延后到 per-session 就绪点，或做 per-account 惰性加载 | 【确定】 |
| 4 | 现有所有 `AyuDatabase`/`AyuMessages` 调用点**全部解析到主线程**（菜单构建、`Api::Updates` 经 `InvokeQueued` 落到主线程）→ **UI 线程可直接同步调用，无需 proxy 到数据库线程**（因为根本不存在 DB 线程） | 【确定】（含一处依赖未落盘 helper 的推断，见 §5.3） |
| 5 | `open_forever()` 未调用 → **每条语句各自 open/close 一次 sqlite 连接**；无 `busy_timeout`/`journal_mode` 设置 | 【确定】 |
| 6 | 主键用**复合主键 `(userId, dialogId, topicId)`**，`topicId` 沿用 `0 = 主线` 约定（`messages_storage.cpp:40-44`）；PK 本身即索引，无需额外 `make_index` | 【确定】（设计决策，理由见 §3.3） |
| 7 | 迁移 V2 **必须是 no-op**：表由 `sync_schema(true)` 自动创建（`ayu_database.cpp:229` 先于 `:231` 的 `runMigrations`）；而 `runMigrations` 失败会 `moveCurrentDatabase()` **轮转掉整个 Ayu 库** | 【确定】 |
| 8 | 最大风险：`storage` 单例**无任何互斥量**；线程安全依赖一条未落盘 helper 的推断链 | 【无法确定（需外部构建机核对）】 |

---

## 1. sqlite_orm schema 约定（`ayu/data/ayu_database.cpp`）

### 1.1 storage 是全局单例 + 相对路径

`Telegram/SourceFiles/ayu/data/ayu_database.cpp:13-15`
```cpp
using namespace sqlite_orm;
auto storage = make_storage(
	"./tdata/ayudata.db",
```

【确定】**文件作用域的非局部 `auto storage`**，进程内唯一实例。**不是每个 Session 一个 DB，也不是每账号一个文件。**

多账号隔离完全依赖表里的 `userId` 列 —— 与 `DeletedMessage`（`ayu_database.cpp:33` `make_column("userId", &DeletedMessage::userId)`）、`EditedMessage`（`:69`）、`SpyMessageRead`（`:132`）同构。

**相对路径 `./tdata/ayudata.db` 的解析基准**：

- `Telegram/SourceFiles/core/launcher.cpp:379-389` `Launcher::exec()`：`init()` → `Logs::start()`（`:387`）→ …… → `executeApplication()`（`:631`）才建 `Application`。
- `Telegram/SourceFiles/logs.cpp:382` `QDir::setCurrent(cWorkingDir());`，`:385` `QDir().mkpath(cWorkingDir() + u"tdata"_q);`（均在 `Logs::start()` 内，`#ifndef Q_OS_WINRT`）。

【确定】`AyuInfra::init()` 运行时 CWD 已被设为 `cWorkingDir()`，且 `tdata/` 已被创建 → 相对路径安全。
（对照：`Telegram/SourceFiles/storage/localstorage.cpp:363` 用绝对式 `cWorkingDir() + u"tdata/"_q`；Ayu 库是 Ayu 自己的写法，**不建议在本次改动中顺手改**，保持现状。）

### 1.2 表定义约定

`ayu_database.cpp:102-113`（`DeletedDialog`，最接近本需求的"小表"）
```cpp
	make_table<DeletedDialog>(
		"DeletedDialog",
		make_column("fakeId", &DeletedDialog::fakeId, primary_key().autoincrement()),
		make_column("userId", &DeletedDialog::userId),
		make_column("dialogId", &DeletedDialog::dialogId),
		make_column("peerId", &DeletedDialog::peerId),
		...
	),
```

约定提炼：
| 约定 | 证据 |
|---|---|
| `make_table<Struct>("PascalCaseName", make_column("camelCaseName", &Struct::field, constraints...))` | `ayu_database.cpp:30-65, 66-101, 102-113, 114-122, 123-128, 129-136, 137-144` |
| 追加型表用 `fakeId` + `primary_key().autoincrement()` | `:32, :68, :104, :125, :131, :139` |
| 自然键表用裸 `primary_key()` | `:18`（`SchemaVersion::id`）、`:116`（`RegexFilter::id`） |
| 表名与 struct 名同名 | 全部 9 张表 |

### 1.3 索引定义约定

`ayu_database.cpp:21-29`
```cpp
	make_index("idx_deleted_message_userId_dialogId_topicId_messageId",
			   column<DeletedMessage>(&DeletedMessage::userId),
			   column<DeletedMessage>(&DeletedMessage::dialogId),
			   column<DeletedMessage>(&DeletedMessage::topicId),
			   column<DeletedMessage>(&DeletedMessage::messageId)),
	make_index("idx_edited_message_userId_dialogId_messageId",
			   column<EditedMessage>(&EditedMessage::userId),
			   column<EditedMessage>(&EditedMessage::dialogId),
			   column<EditedMessage>(&EditedMessage::messageId)),
```

【确定】约定：
- 命名 `idx_<snake_case表名>_<snake_case列名串联>`；
- `make_index(...)` 一律排在**所有 `make_table` 之前**（`:21-29` 是索引块，`:30-144` 是表块）；
- 索引列顺序与查询的 `where(...)` 顺序一致（对照 `getDeletedMessages` 的 `where`，`ayu_database.cpp:308-313`：`userId` → `dialogId` → `topicId` → `messageId`）。

**索引不会自动清理**：`Telegram/SourceFiles/ayu/libs/sqlite/sqlite_orm.h:23249-23256`
```cpp
            sync_schema_result sync_table(const index_t<Cols...>& index, sqlite3* db, bool) {
                auto res = sync_schema_result::already_in_sync;
                ...
                perform_void_exec(db, query);
                return res;
            }
```
【确定】`sync_schema` 对索引只做 `CREATE INDEX IF NOT EXISTS`（序列化串执行），**从不 DROP**。⇒ 索引改名会留下孤儿索引，写入放大但不会出错。

### 1.4 版本 / 迁移处理

`ayu_database.cpp:161-207`
```cpp
void runMigrations(decltype(storage) &storage) {
	constexpr int kLatestVersion = 1;

	const std::map<int, Fn<void(decltype(storage) &)>> migrations = {
		{1, AyuMigrations::migrateToV1},
	};
	...
	if (currentVersion >= kLatestVersion) {
		LOG(("Database is ok"));
		return;
	}
	...
	for (int v = currentVersion + 1; v <= kLatestVersion; ++v) {
		if (migrations.contains(v)) {
			try {
				storage.begin_transaction();
				migrations.at(v)(storage);
				storage.update_all(set(c(&SchemaVersion::version) = v), where(c(&SchemaVersion::id) == 1));
				storage.commit();
			} catch (...) {
				storage.rollback();
				LOG(...);
				AyuDatabase::moveCurrentDatabase();
				return;
			}
		}
	}
}
```

`initialize()` 的执行顺序 —— **这是本次设计最关键的一段**：`ayu_database.cpp:227-243`
```cpp
void initialize() {
	try {
		storage.sync_schema(true);   // :229  ← 先建表/加列
		runMigrations(storage);      // :231  ← 后跑版本迁移
		storage.sync_schema(true);   // :233
	} catch (const std::exception &ex) {
		LOG(...);
		moveCurrentDatabase();
		storage.sync_schema(true);
		if (!storage.get_pointer<SchemaVersion>(1)) storage.insert(SchemaVersion{1, 0});
	}
}
```

【确定】**只要把 `make_table<Checkpoint>` 加进 `make_storage(...)`，`sync_schema(true)`（`:229`）就会在旧库文件上自动创建该表，迁移函数本身可以做任何事。**

`sync_schema(bool preserve)` 语义：`sqlite_orm.h:23352` `std::map<std::string, sync_schema_result> sync_schema(bool preserve = false) {`；文档注释 `sqlite_orm.h:23343-23345`：
> `@param preserve affects function's behaviour in case it is needed to remove a column. If it is `false` ... if `true` - table is being copied into another table, dropped and copied table is renamed with source table name.`

【确定】现有代码传 `true` ⇒ **删列时走"建新表→拷数据→drop→改名"的保数据路径**。新增表不触发该路径。

### 1.5 打开位置与连接生命周期

`sqlite_orm.h:17756-17771`
```cpp
            /**
             *  Call it once during storage lifetime to make it keeping its connection opened till dtor call.
             *  By default if storage is not in-memory it calls `sqlite3_open` only when the connection is really
             *  needed and closes when it is not needed. ...
             */
            void open_forever() {
                this->isOpenedForever = true;
                this->connection->retain();
```

【确定】`open_forever()` **在 Ayu 代码中从未被调用**（grep 见 §7）。`storage_base` 构造函数 `sqlite_orm.h:18137-18147` 只对 in-memory 存储 retain；析构 `:18160-18167` 只在 `isOpenedForever` / `inMemory` 时 release。
`begin_transaction_internal` `:18169-18176` 与 `get_connection` `:18178-18184` 都是"取用即 retain / 归还即 release"。

⇒ **每条 SQL 语句各自 open/close 一次 `sqlite3` 连接**（`sqlite3_open`+`sqlite3_close` 是真实开销）。
⇒ 由此得到本设计的一条硬性推论：**打开 peer 的一次性路径上不要做同步 DB 查询**（见 §4.3）。

**无任何 pragma 设置**（见 §7 命令 18）：`storage` 未调用 `busy_timeout()` / `journal_mode()` / `foreign_keys()`。
`SQLITE_DEFAULT_JOURNAL_MODE` 即 rollback journal；`synchronous` 默认 `FULL`。

**连接不含互斥量**：`connection_holder` 的 retain/release 是普通整型加减，无锁。见 §5.3 风险。

### 1.6 数据失败时的降级（重要）

`ayu_database.cpp:211-225`
```cpp
void moveCurrentDatabase() {
	const auto time = base::unixtime::now();
	if (QFile::exists("./tdata/ayudata.db")) {
		QFile::rename("./tdata/ayudata.db", QString("./tdata/ayudata_%1.db").arg(time));
	}
	if (QFile::exists("./tdata/ayudata.db-shm")) { ... }
	if (QFile::exists("./tdata/ayudata.db-wal")) { ... }
}
```

【确定】迁移失败（`:198-204`）或 `initialize()` 异常（`:234-242`）会**把整个 Ayu 库重命名轮转掉**（删除消息历史、正则过滤器、检查点全部丢失）。
【确定】`-shm`/`-wal` 的处理是死代码 —— 从未启用 WAL（§7 命令 18 无命中）。这说明作者曾打算开 WAL。**不要在本任务中顺手启用 WAL**：一旦启用，`sync_schema`/`CREATE TABLE` 的写锁会让并发 open 更易撞 `SQLITE_BUSY`（无 busy_timeout），反而放大风险。若要启用，须同批加 `storage.busy_timeout(5000)`。

---

## 2. `messages_storage` CRUD API 形状（Checkpoint 的模板）

### 2.1 两层结构

| 层 | 文件 | 入参形态 | 证据 |
|---|---|---|---|
| `AyuMessages`（类型化门面） | `ayu/data/messages_storage.{h,cpp}` | `not_null<HistoryItem*>` / `not_null<PeerData*>` + `ID topicId` | `messages_storage.h:11-23` |
| `AyuDatabase`（裸 `ID` + sqlite_orm） | `ayu/data/ayu_database.{h,cpp}` | `ID userId, ID dialogId, ID topicId, ID messageId` | `ayu_database.h:24-32` |

**Checkpoint 沿用同两层**（`AyuCheckpoints` + `AyuDatabase` 扩展），不引入第三种风格。

### 2.2 签名与返回风格（逐条）

`ayu/data/messages_storage.h:13-21`
```cpp
void addEditedMessage(not_null<HistoryItem *> item);
std::vector<AyuMessageBase> getEditedMessages(not_null<HistoryItem*> item, ID minId, ID maxId, int totalLimit);
bool hasRevisions(not_null<HistoryItem*> item);

void addDeletedMessage(not_null<HistoryItem*> item);
std::vector<AyuMessageBase> getDeletedMessages(not_null<PeerData*> peer, ID topicId, ID minId, ID maxId, int totalLimit, const QString &searchQuery = QString());
bool hasDeletedMessages(not_null<PeerData*> peer, ID topicId);
void removeDeletedMessage(not_null<HistoryItem*> item);
void clearDeletedMessages(not_null<PeerData*> peer, ID topicId);
```

`ayu/data/ayu_database.h:28-32`
```cpp
void addDeletedMessage(const DeletedMessage &message);
std::vector<DeletedMessage> getDeletedMessages(ID userId, ID dialogId, ID topicId, ID minId, ID maxId, int totalLimit, const std::string &searchQuery = "");
bool hasDeletedMessages(ID userId, ID dialogId, ID topicId);
void removeDeletedMessage(ID userId, ID dialogId, ID messageId);
void clearDeletedMessages(ID userId, ID dialogId, ID topicId);
```

### 2.3 错误处理形状：**完全同步、绝不抛出、静默降级**

写：`ayu_database.cpp:290-302`
```cpp
void addDeletedMessage(const DeletedMessage &message) {
	try {
		storage.begin_transaction();
		storage.insert(message);
		storage.commit();
	} catch (std::exception &ex) {
		try { storage.rollback(); } catch (...) {}
		LOG(("Failed to save edited message for some reason: %1").arg(ex.what()));
	}
}
```

查：`ayu_database.cpp:342-357`
```cpp
bool hasDeletedMessages(ID userId, ID dialogId, ID topicId) {
	try {
		return !storage.select(
			columns(column<DeletedMessage>(&DeletedMessage::dialogId)),
			where(...), limit(1)
		).empty();
	} catch (std::exception &ex) {
		LOG(("Failed to check if dialog has deleted message: %1").arg(ex.what()));
		return false;
	}
}
```

【确定】**同步**（无 `crl::`/`Ui::PostponeCall`/future/promise，无异步返回）；**返回 `void` / `bool` / `std::vector`**，无 `std::optional`、无 `expected`、无错误码。

【确定】**已存在的不一致（照搬时不要继承）**：
- `getEditedMessages`（`ayu_database.cpp:259-271`）与 `getDeletedMessages`（`:304-340`）**没有 try/catch** —— sqlite 异常会穿透到调用方（UI 菜单构建、`Api::Updates`）。**Checkpoint 的所有新函数必须一律 try/catch**。
- `getAllT<T>()`（`:386-394`）、`getById`（`:428-437`）用返回 `{}` / `0` 兜底。
- 提前 return 守卫：`messages_storage.cpp:91-93`（text 空则不写）、`:118-120`（同上）—— **Checkpoint 无此类守卫**。

### 2.4 映射样板

`ayu/data/messages_storage.cpp:32-45`（`map()` 的账号/键四行）
```cpp
void map(not_null<HistoryItem*> item, AyuMessageBase &message) {
	const ID userId = item->history()->owner().session().userId().bare & PeerId::kChatTypeMask;

	message.userId = userId;
	message.dialogId = getDialogIdFromPeer(item->history()->peer);
	message.groupedId = item->groupId().raw();
	message.peerId = item->history()->peer->id.value & PeerId::kChatTypeMask;
	message.fromId = item->from()->id.value & PeerId::kChatTypeMask;
	if (item->topic()) {
		message.topicId = item->topicRootId().bare;
	} else {
		message.topicId = 0;
	}
	message.messageId = item->id.bare;
```
时间戳：`:71` `message.entityCreateDate = base::unixtime::now();`

【确定】**Checkpoint 的键（userId, dialogId, topicId, messageId）逐字复用上面这四行 + `msgId`**，`date` 复用 `base::unixtime::now()`。

---

## 3. 账号维度与类型来源

### 3.1 `userId` 的来源与多账号隔离

- 来源：`messages_storage.cpp:33` `item->history()->owner().session().userId().bare & PeerId::kChatTypeMask`（另见 `:99, :107, :127, :133, :139, :144`）。
- `Main::Session::userId()` 返回的 `_userId` 在**构造函数**即确定：`Telegram/SourceFiles/main/main_session.cpp:144` `, _userId(user.c_user().vid())`。
- 【确定】掩码 `PeerId::kChatTypeMask` = `BareId(0xFFFFFFFFFFFFULL)`，`Telegram/SourceFiles/data/data_peer_id.h:110`
  ```cpp
  static constexpr BareId kChatTypeMask = BareId(0xFFFFFFFFFFFFULL);
  ```
  ⇒ 掩掉 `:48` 起的 type shift 位（`data_peer_id.h:115-118` `value(id.bare | (BareId(Shift) << 48))`）。**作用：防止"用户 123"与"频道 123"在裸 ID 上撞车。** Checkpoint 必须同样加掩码。

- 【确定】**隔离粒度 = 一个文件 + 一个 `userId` 列**，与 `DeletedMessage`/`EditedMessage`/`SpyMessageRead` 一致。**不引入 `account` 列。**
  > 姊妹文档 `p1-checkpoint-navigation.md:459` 建议"显式加 account 列"。**本设计否决该建议**：`account` 与 `userId` 语义重复，且 `SpyMessageRead` 已有 `userId` 列先例（`ayu_database.cpp:132`）；偏离会破坏"模式照搬 DeletedMessage"的定稿决策（ADR-0001 §Decision 2）。

### 3.2 `DialogId` —— 本 fork **不存在**该类型

命令见 §7（命令 16）。范围：`Telegram/SourceFiles/data/*.h`，模式 `using DialogId|class DialogId|struct DialogId|using PeerDialogId|DialogId =` → **0 命中**。

【确定】v7.0.9 用 `PeerId` 统一了 user/chat/channel，没有 `DialogId`。
Ayu 代码里的 `dialogId` 是**fork 私有的 signed `ID` 约定**：

`ayu/utils/telegram_helpers.h:59` `ID getDialogIdFromPeer(not_null<PeerData*> peer);`
`ayu/utils/telegram_helpers.cpp:164-171`
```cpp
ID getDialogIdFromPeer(not_null<PeerData*> peer) {
	ID peerId = peer->id.value & PeerId::kChatTypeMask;
	if (peer->isChannel() || peer->isChat()) {
		peerId = -peerId;
	}
	return peerId;
}
```
【确定】**正数 = 用户，负数 = 群/频道/超级群**。Checkpoint 的 `dialogId` 列沿用此约定（**不新增 `peerId` 列**：`DeletedMessage` 虽有 `peerId` 列（`:36`），但所有查询（`:308-313`、`:347-350`、`:363-366`、`:377-380`）只用 `dialogId` + `userId`；Checkpoint 照此精简）。

### 3.3 `MsgId` / `PeerId` 类型

- `Telegram/SourceFiles/data/data_msg_id.h:21` `constexpr MsgId(int64 value) noexcept : bare(value) {`；`:48` `int64 bare = 0;`
- `Telegram/SourceFiles/data/data_msg_id.h:88` `constexpr auto ShowAtUnreadMsgId = MsgId(0);`
  【确定】服务端 `msgId` 恒 > 0 ⇒ **`0` 是安全的"未设置"哨兵**，Checkpoint 的"无检查点"用 `std::nullopt` 表达，落库时 `topicId` 用 `0` 表达"主线"（与 `messages_storage.cpp:40-44` 一致）。
- `Telegram/SourceFiles/data/data_peer_id.h:108-143` `struct PeerId { BareId value = 0; ... }`；`std::hash<PeerId>` 特化 `data_peer_id.h:248-253`
  ```cpp
  template <>
  struct hash<PeerId> : private hash<BareId> {
  	size_t operator()(PeerId value) const noexcept {
  		return hash<BareId>::operator()(value.value);
  	}
  };
  ```
  【确定】`PeerId` 可直接做 `std::unordered_map` 键（既有先例 `ayu/ayu_state.cpp:13` `std::unordered_map<PeerId, std::unordered_set<MsgId>> hiddenMessages;`）⇒ 内存缓存无需自定义 hash。

### 3.4 sqlite_orm 的类型映射

- `ID = long long`（`ayu/data/entities.h:11` `using ID = long long;`）→ `sqlite_orm.h:8366` `Default nullability status for all types is NOT NULL.` ⇒ 映射为 `INTEGER NOT NULL`。
- 可空类型特化 `sqlite_orm.h:8377-8389`：`std::optional` / `std::unique_ptr` / `std::shared_ptr` → `NULL`。既有用法：`entities.h:83` `std::optional<ID> dialogId;`（查询 `ayu_database.cpp:442` `is_null(...)`）、`entities.h:68` `std::unique_ptr<int> folderId;`。
  ⇒ **Checkpoint 的 `topicId` 不用 `std::optional<ID>`，用 `ID` + 0**（与 `AyuMessageBase::topicId`，`entities.h:22` `ID topicId;` 一致），保持 PK 简单。

### 3.5 ⚠️ 整数宽度陷阱（不要照抄兄弟表）

`ayu/data/entities.h:23` `int messageId;` ← 赋值自 `messages_storage.cpp:45` `message.messageId = item->id.bare;`（`int64` → `int` 窄化）。
【确定】这是**既有缺陷**（本 checkout 的 `MsgId` 理论上限是 `ScheduledMaxMsgId`/`ShortcutMaxMsgId`，`data_msg_id.h:86-87`，远超 `int`）。
**Checkpoint 必须用 `ID messageId`（int64）**，不复制该缺陷。SQLite 侧两者都是 `INTEGER`，无损。

---

## 4. 方案本体

### 4.1 DDL

#### 4.1.1 实体（`ayu/data/entities.h` 追加）

```cpp
class Checkpoint {
public:
	ID userId;
	ID dialogId;
	ID topicId;
	ID messageId;
	int date;
};
```
（【确定】`date` 用 `int` 对齐 `entities.h:41` `int entityCreateDate;` + `base::unixtime::now()`，见 `messages_storage.cpp:71`。）

#### 4.1.2 SQL（sqlite_orm 生成的实际 DDL）

```sql
CREATE TABLE IF NOT EXISTS "Checkpoint" (
  "userId"   INTEGER NOT NULL,
  "dialogId" INTEGER NOT NULL,
  "topicId"  INTEGER NOT NULL,
  "messageId" INTEGER NOT NULL,
  "date"     INTEGER NOT NULL,
  PRIMARY KEY ("userId", "dialogId", "topicId")
);
```

生成机制证据：
- `sqlite_orm.h:3788-3790` `inline internal::primary_key_t<> primary_key() { return {{}}; }` —— 无参 `primary_key()` 是**列约束**，序列化为内联 `PRIMARY KEY`（`sqlite_orm.h:20411-20418` `ss << "PRIMARY KEY";`）。
- 多列各自 `PRIMARY KEY` 列约束 ⇒ SQLite 合成为复合主键；`sqlite_orm` 有对应支持：`sqlite_orm.h:12406` `bool exists_in_composite_primary_key(const table_t<O, WithoutRowId, Cs...>& table, ...)`。
- `CREATE TABLE` 串接：`sqlite_orm.h:19542-19543` `ss << "CREATE TABLE " << streaming_identifier(tableName) << " (" << streaming_expressions_tuple(...) << ")";`

#### 4.1.3 索引决策

**主键 `(userId, dialogId, topicId)` 本身就是索引**（SQLite 自动建 `sqlite_autoindex_Checkpoint_1`），它同时满足本功能的三类查询：

| 查询 | 走 PK 的方式 |
|---|---|
| 点查某会话检查点 | 三列全等 → 最左前缀，O(log n) |
| 预载某账号全部检查点 | `userId` 限定 → 最左前缀，范围扫描 |
| 跨账号列举（设置页"检查点列表"，若将来要） | **不覆盖** |

【确定】**因此不额外 `make_index`**，理由：任何 `(userId, dialogId, ...)` 开头的索引都是复合主键的最左前缀，SQLite 查询规划器会直接用 PK，额外索引**只增加每次写入的维护成本、零查询收益**。

若评审要求与 `DeletedMessage`（`ayu_database.cpp:21-25`）**形式完全对齐**（4 列显式索引），追加这一行即可（代价：纯写放大，**不推荐**）：
```cpp
	make_index("idx_checkpoint_userId_dialogId_topicId_messageId",
			   column<Checkpoint>(&Checkpoint::userId),
			   column<Checkpoint>(&Checkpoint::dialogId),
			   column<Checkpoint>(&Checkpoint::topicId),
			   column<Checkpoint>(&Checkpoint::messageId)),
```
⚠️ 未来若新增"跨账号列举"需求，正确的索引是 `make_index("idx_checkpoint_dialogId", column<Checkpoint>(&Checkpoint::dialogId))`，**不是**上面这条。

#### 4.1.4 sqlite_orm 侧声明（`ayu_database.cpp` 精确改动点）

在索引块 `ayu_database.cpp:21-29` 之后、表块末尾 `:137-144`（`SpyMessageContentsRead` 表）之后追加：

```cpp
	make_table<Checkpoint>(
		"Checkpoint",
		make_column("userId", &Checkpoint::userId, primary_key()),
		make_column("dialogId", &Checkpoint::dialogId, primary_key()),
		make_column("topicId", &Checkpoint::topicId, primary_key()),
		make_column("messageId", &Checkpoint::messageId),
		make_column("date", &Checkpoint::date)
	)
```

【确定】`make_storage(...)` 的参数顺序不影响 `sync_schema`（`sqlite_orm.h:23352-23360` `iterate_tuple<true>(this->db_objects, ...)` 遍历全部对象），但保持"索引在前、表在后"的文件视觉约定。

### 4.2 迁移

`ayu_database.cpp:147-159`（现有形态）
```cpp
namespace AyuMigrations {

void migrateToV1(decltype(storage) &storage) {
	// drop RegexFilter table as we've added primary_key()
	try {
		storage.drop_table_if_exists("RegexFilter");
		LOG(("Migration to V1 successful."));
	} catch (const std::exception &ex) {
		LOG(("Migration to V1 failed: %1").arg(ex.what()));
	}
}
```

**推荐改动（最小、零风险）**：
```cpp
void migrateToV2(decltype(storage) &storage) {
	LOG(("Migration to V2 successful."));
}
```
+ `ayu_database.cpp:162` `constexpr int kLatestVersion = 1;` → `2`
+ `ayu_database.cpp:164-166` 的 map 增加 `{2, AyuMigrations::migrateToV2},`

【确定】**V2 必须是 no-op**，因为：
1. 表由 `ayu_database.cpp:229` 的 `sync_schema(true)` 在迁移**之前**就建好了；
2. 任何会抛异常的迁移体都会命中 `ayu_database.cpp:198-204` → `moveCurrentDatabase()` → **整个 Ayu 库被轮转**（§1.6）。

【确定】**严格来说不 bump 版本也能工作**（`runMigrations` 在 `currentVersion >= kLatestVersion` 时于 `:180-183` 提前返回，新表仍由 sync_schema 建出）。bump 的价值是让 `SchemaVersion` 表如实记录 schema 代次、给未来真迁移留锚点。两种都正确，**推荐 bump + no-op**（自解释、零运行时代价）。

### 4.3 CRUD API 签名清单

#### 层 1：`AyuDatabase`（`ayu/data/ayu_database.h` 追加于 `:32` 之后，沿用 `:24-32` 区块）

```cpp
void setCheckpoint(ID userId, ID dialogId, ID topicId, ID messageId);
std::optional<Checkpoint> getCheckpoint(ID userId, ID dialogId, ID topicId);
std::vector<Checkpoint> getCheckpoints(ID userId);
void clearCheckpoint(ID userId, ID dialogId, ID topicId);
void clearCheckpointsForAccount(ID userId);
```

| 函数 | 返回 | 语义 | 模板来源 |
|---|---|---|---|
| `setCheckpoint` | `void` | **upsert**（新覆盖旧，ADR-0001 §Decision 1） | `ayu_database.cpp:290-302` 写 + `:461-473` 的 `replace` 意图 |
| `getCheckpoint` | `std::optional<Checkpoint>` | 单点查；无行 → `nullopt`；异常 → `nullopt` | `:342-357` `hasRevisions` 的 try/catch 形态 |
| `getCheckpoints` | `std::vector<Checkpoint>` | **预载用**：按 `userId` 过滤全量 | `:304-340` |
| `clearCheckpoint` | `void` | 删单条 | `:373-384` `clearDeletedMessages` |
| `clearCheckpointsForAccount` | `void` | 登出/切账号清理 | 同上 |

【确定】**`std::optional` 是本文件的首例**（现有全部 `std::vector` / `bool` / `void`）。可选替代以严格贴合既有风格：
```cpp
// 备选（无 optional）
int getCheckpointMessageId(ID userId, ID dialogId, ID topicId, int &date);  // 返回 0 = 无
```
但 `0` 与"主线的 topicId"、`ShowAtUnreadMsgId` 撞语义，**推荐 `std::optional`**（C++17，仓库已用 `std::optional`：`ayu_settings.h:518`、`entities.h:83`）。

#### 层 2：`AyuCheckpoints`（新建 `ayu/data/checkpoints_storage.{h,cpp}`，严格照 `messages_storage.h:11-23` 形状）

```cpp
namespace AyuCheckpoints {

void set(not_null<HistoryItem*> item);
void set(not_null<PeerData*> peer, ID topicId, MsgId msgId);
std::optional<MsgId> get(not_null<PeerData*> peer, ID topicId);
void clear(not_null<PeerData*> peer, ID topicId);
void clearForAccount(not_null<Main::Session*> session);
void preload(not_null<Main::Session*> session);
}
```

【确定】`not_null<HistoryItem*>` / `not_null<PeerData*>` 形参 + `namespace` 单字母风格（`messages_storage.h:11` `namespace AyuMessages {`）均有先例。

#### 层 1 实现要点（骨架，非最终代码）

```cpp
void setCheckpoint(ID userId, ID dialogId, ID topicId, ID messageId) {
	try {
		storage.begin_transaction();
		storage.remove_all<Checkpoint>(
			where(column<Checkpoint>(&Checkpoint::userId) == userId and
				  column<Checkpoint>(&Checkpoint::dialogId) == dialogId and
				  column<Checkpoint>(&Checkpoint::topicId) == topicId)
		);
		storage.insert(Checkpoint{userId, dialogId, topicId, messageId, base::unixtime::now()});
		storage.commit();
	} catch (std::exception &ex) {
		try { storage.rollback(); } catch (...) {}
		LOG(("Failed to save checkpoint: %1").arg(ex.what()));
	}
}
```

**为什么用 delete+insert 而不是 `storage.replace(...)`**：
- `replace` 会序列化成 `REPLACE INTO`（`sqlite_orm.h:20652-20659`），SQL 层面带齐全部非生成列，复合主键**应该**可用；
- 但**本仓库 9 张表的 `primary_key()` 出现次数恰为 9**（`grep -c "primary_key()" ayu_database.cpp` = 9，§7 命令 39）⇒ **无任何复合主键先例**，该路径从未被本 checkout 编译过。
- delete+insert 在一个事务内**行为完全确定**，且与 `ayu_database.cpp:290-302` 的既有写形态一致。
- 【无法确定（需外部构建机核对）】若外部构建机确认 `replace` 在复合主键下可用，可简化为单语句 `storage.replace(Checkpoint{...})`（少一次 DELETE，天然原子）。

【确定】**禁止嵌套事务**：`setCheckpoint` 内部 `begin_transaction()`。若未来从某个已在 `AyuDatabase` 事务中的调用点进入，`begin_transaction` 会失败。现有 `addDeletedMessage`/`addEditedMessage`/`addRegexFilter` 同样如此（`:247, :292, :463`）——保持一致，但新代码不得新增嵌套调用点。

### 4.4 读取时机与内存缓存方案

#### 4.4.1 DB 何时就绪（决定"能否同步读"）

`Telegram/SourceFiles/core/application.cpp`
```
:277   startLocalStorage();
...
:306   AyuInfra::init();
...
:349   _windows.emplace(nullptr, std::make_unique<Window::Controller>());
```
`Telegram/SourceFiles/ayu/ayu_infra.cpp:70-78`
```cpp
void init() {
	initLang();
	initDatabase();     // :72  → AyuDatabase::initialize()
	initUiSettings();
	...
}
```
`Telegram/SourceFiles/ayu/ayu_infra.cpp:47-49`
```cpp
void initDatabase() {
	AyuDatabase::initialize();
}
```

【确定】**结论 1**：`AyuDatabase::initialize()`（含 `sync_schema` + 迁移）在 `Application::run()` 内、**早于**任何 `Window::Controller` / `Account` / `Session` 创建（`:306` < `:349`）。
⇒ **打开 peer 时 DB 一定已 ready，同步读技术上可行。**

【确定】**结论 2（对姊妹文档 §4.3 的修正）**：`AyuInfra::init()` 执行时**一个 account 都不存在** ⇒ 不知道任何 `userId`。
姊妹文档 `p1-checkpoint-navigation.md:445` 建议"在 `AyuInfra::initDatabase()` 中全表读入并按 `Session*` 分桶"——**在那一刻无法分桶**（分桶所需的 `userId` 要到 `main_account.cpp:194` `_session = std::make_unique<Session>(this, user, std::move(settings));` 才确定，其 userId 来自 `main_session.cpp:144` `, _userId(user.c_user().vid())`）。
⇒ **预载必须延后，或做成 per-account 惰性加载。**

#### 4.4.2 per-session 就绪点

`Telegram/SourceFiles/main/main_session.cpp:235-293`
```cpp
	crl::on_main_queue(this, { [=] {
		...
	}, [=] {
		// Storage::Account uses Main::Account::session() in those methods.
		// So they can't be called during Main::Session construction.
		//
		// They are deferred via crl::on_main which fires after the
		// constructor returns and _session is set.
		...
		local().readInstalledStickers();
	}, [=] { local().readInstalledMasks(); },
	  ...
	  [=] {
		data().stickers().notifySavedGifsUpdated();
		DEBUG_LOG(("Init: Account stored data load finished."));
	} }).dispatch();
```

【确定】这是"该账号本地数据加载完毕"的既有钩子，主线程、ctor 返回后触发、`userId` 可用。
【确定】但它是**主仓库代码**（`Telegram/SourceFiles/main/main_session.cpp`）——**本次铁律是不得修改 `Telegram/` 下任何文件**。因此它只是**设计锚点**，实现期是否接线、接在哪，属于另一份变更单的范围。

#### 4.4.3 缓存形态（照抄 `FiltersCacheController`）

先例：`Telegram/SourceFiles/ayu/features/filters/filters_cache_controller.cpp`
```cpp
:24-26   std::mutex rebuildMutex;
         std::mutex cacheMutex;
         std::mutex filteredMessagesMutex;
:28      rpl::event_stream<> filtersUpdateStream;
:38      std::shared_ptr<const Cache> cache;
:90-100  void rebuildCache() { std::lock_guard rebuildLock(rebuildMutex); auto next = buildCache(); { ... cache = std::move(next); ... } }
:120-142 std::shared_ptr<const Cache> snapshot()   // 双检锁 + 惰性构建
```

【确定】这就是 Ayu 现成的"SQLite → 内存快照"模式。Checkpoint 缓存照此形状，但**键结构不同**（见下）。

#### 4.4.4 推荐形态：per-account 惰性加载 + 三层嵌套 map

```cpp
// ayu/data/checkpoints_cache.h 内部
namespace AyuCheckpointsCache {

// userId -> dialogId -> topicId -> {msgId, date}
std::unordered_map<ID, std::unordered_map<ID, std::unordered_map<ID, Checkpoint>>> cache;
std::mutex mutex;
std::unordered_set<ID> loaded;   // 已从 DB 预载过的 userId

void ensureLoaded(ID userId);              // 首次访问该账号时同步全量载入
std::optional<MsgId> get(ID userId, ID dialogId, ID topicId);
void put(ID userId, ID dialogId, ID topicId, MsgId msgId);
void remove(ID userId, ID dialogId, ID topicId);
void removeAccount(ID userId);
}
```

**为何三层 `ID` 嵌套而不是 `unordered_map<PeerId, ...>`**：
- 【确定】`hash<PeerId>` 可用（`data_peer_id.h:248-253`），但 `PeerId` **不含 userId**，用它做键会丢掉账号维度 → 与 §3.1 的隔离约定冲突。
- `unordered_map<ID, ...>` 的三层结构天然支持 `removeAccount(userId) = cache.erase(userId)`。
- 表规模极小（"有检查点的会话数"，通常几十~几百行），三层 `unordered_map` 的开销可忽略。

**加载策略（关键决策）**：
| 方案 | 描述 | 评价 |
|---|---|---|
| **A（推荐）** `ensureLoaded(userId)` **惰性** + 在 4.4.2 的 per-session 钩子主动调一次 `preload(session)` | 首次访问某账号时 `std::lock_guard` + 一次 `getCheckpoints(userId)`；钩子调用只是把这次开销挪离"用户点开会话"那一刻 | **正确性兜底**（任何调用路径都不会漏载）+ 打开 peer 零 IO。**采用** |
| B 纯钩子预载，缓存未命中直接返 `nullopt` | 钩子万一没触发（多账号后加入、旧账号恢复）就**静默丢检查点** | ❌ 不可靠 |
| C 每次 `get` 直查 DB | `MainWidget::showHistory` 在打开 peer 的热路径上，每次多一次 `sqlite3_open`+`SELECT`+`sqlite3_close`（§1.5） | ❌ 把 IO 放进热路径 |

【确定】**A 是唯一同时满足"打开 peer 零延迟"和"不漏载"的组合。**

**与姊妹文档的一致性**：`p1-checkpoint-navigation.md:441-461` 推荐的方案 A（预载进内存、打开时同步查）方向一致；本设计补上它缺的两点：① per-account 分桶的实际可行点（4.4.2/4.4.4）；② 缓存必须带 `userId` 维度（否决 `:459` 的 `account` 列建议，理由见 §3.1）。

### 4.5 写入时机（保存动作回调）

【确定】落点文件与调用点：

| # | 触发 | 新增/改动位置 | 回调内容 |
|---|---|---|---|
| W1 | 右键菜单项「保存为 AG 检查点」 | 声明 → `ayu/ui/context_menu/context_menu.h:44-54` 区块内追加<br>实现 → `ayu/ui/context_menu/context_menu.cpp`（紧邻 `AddHistoryAction` `:489-514`）<br>调用 → `Telegram/SourceFiles/history/history_inner_widget.cpp:3128-3132` 的 Ayu 钩子块 | `AyuCheckpoints::set(item)` |
| W2 | 消息上快速右键两次（双击右键） | `HistoryInnerWidget::contextMenuEvent` 同路 | 同一个 `AyuCheckpoints::set(item)` |
| W3 | 「跳到检查点」菜单项（peer 菜单） | 声明 → `context_menu.h:29-32` 邻域<br>实现 → 紧邻 `AddJumpToBeginningAction` `context_menu.cpp:327-371` | `controller->showPeerHistory(peer, Window::SectionShow::Way::Forward, checkpointMsgId)` |
| W4 | 降级清理（`lng_message_not_found`） | 见姊妹文档 §5.4 | `AyuCheckpoints::clear(peer, topicId)` |
| W5 | 登出清理（可选） | `AyuCheckpoints::clearForAccount(session)`，挂在 `Telegram/SourceFiles/main/main_account.cpp:203-217` `Account::destroySession` | 仅清内存 + DB |

**W1 的实现模板**（`context_menu.cpp:489-514` `AddHistoryAction` 的形状）
```cpp
menu->addAction(
	tr::ayu_SaveCheckpointMenuText(tr::now),
	[=] { AyuCheckpoints::set(item); },
	&st::ayuSomeIcon);
```
【确定】`menu->addAction(text, callback, icon)` 形状见 `context_menu.cpp:504-513`、`:528-541`、`:555-569`。
【确定】可见性门控样板：`context_menu.h:22` `bool needToShowItem(ContextMenuVisibility state);` + `context_menu.cpp:518` `needToShowItem(settings.showHideMessageInContextMenu())`，设置项声明 `ayu/ayu_settings.h:306`。

**W3 的实现模板**（`context_menu.cpp:365-371`，既有"跳到某消息"唯一先例）
```cpp
	const auto showPeerHistory = [=](auto peer, MsgId id)
	{
		controller->showPeerHistory(
			peer,
			Window::SectionShow::Way::Forward,
			id);
	};
```
【确定】重载签名 `Telegram/SourceFiles/window/window_session_controller.h:278-281`
```cpp
	void showPeerHistory(
		not_null<PeerData*> peer,
		const SectionShow &params = SectionShow::Way::ClearStack,
		MsgId msgId = ShowAtUnreadMsgId);
```

**W1 需要的 `topicId` 推导**：逐字复用 `messages_storage.cpp:40-44`（有 topic 用 `topicRootId().bare`，否则 `0`）。

#### 4.5.1 读取时机（供存储层契约对齐；拦截点细节见姊妹文档）

| 读点 | 调用 | 线程 | 成本 |
|---|---|---|---|
| peer 打开默认定位 | `AyuCheckpoints::get(peer, topicId)` | 主线程 | 内存查找，~0 |
| 「跳到检查点」菜单项显隐 | 同上 | 主线程 | 内存查找 |
| 设置页「检查点列表」（若做） | `AyuCheckpoints::getCheckpoints(session)` | 主线程 | 内存遍历 |

【确定】**peer 打开的最终收敛点在** `Telegram/SourceFiles/mainwidget.cpp:1386-1389`
```cpp
void MainWidget::showHistory(
		PeerId peerId,
		const SectionShow &params,
		MsgId showAtMsgId) {
```
及其 migrateTo 取反分支 `:1399-1401`
```cpp
		if (peer->migrateTo()) {
			peer = peer->migrateTo();
			peerId = peer->id;
			if (showAtMsgId > 0) {
				showAtMsgId = -showAtMsgId;
			}
		}
```
⚠️ **存储层给应用层的两条硬契约**：
1. 替换 `showAtMsgId` **必须**在 `:1399-1401` **之前**完成，让既有的 migrateTo 取反逻辑继续生效（ADR-0001 §Consequences 已记此点）。
2. 替换条件**必须是 `showAtMsgId == ShowAtUnreadMsgId`**（即 `MsgId(0)`，`data_msg_id.h:88`），否则会覆盖 t.me 链接跳转、消息回复跳转等显式定位（姊妹文档 §2.3/§2.4 的 B/C 类调用点）。

【确定】`dialogs_widget.cpp:1190-1199` 是对话列表侧的具体出口：
```cpp
		const auto showAtMsgId = controller()->uniqueChatsInSearchResults(
			_searchState
		) ? ShowAtUnreadMsgId : row.message.fullId.msg;
		...
			controller()->showInNewWindow(peer, showAtMsgId);
		} else {
			controller()->showThread(history, showAtMsgId, params);
```

### 4.6 落点文件清单（汇总）

| 文件 | 动作 | 说明 |
|---|---|---|
| `Telegram/SourceFiles/ayu/data/entities.h` | 追加 `class Checkpoint` | 追加在 `SpyMessageContentsRead`（`:119-137`）之后 |
| `Telegram/SourceFiles/ayu/data/ayu_database.cpp` | `make_table<Checkpoint>`；`kLatestVersion` 1→2；`migrateToV2`；migrations map | §4.1.4 / §4.2 |
| `Telegram/SourceFiles/ayu/data/ayu_database.h` | 5 个声明（§4.3 层 1） | 追加于 `:32` 之后 |
| `Telegram/SourceFiles/ayu/data/checkpoints_storage.h/.cpp` | **新建** | `namespace AyuCheckpoints`（§4.3 层 2）+ 缓存（§4.4.4） |
| `Telegram/SourceFiles/ayu/ui/context_menu/context_menu.h/.cpp` | 2 个新 action | W1/W3 |
| `Telegram/Resources/langs/lang.strings` | 2 个 `ayu_*` key | 既有 ayu key 在主 lang.strings，例 `:9010` `"ayu_ContextHideMessage" = "Hide";` |
| `Telegram/CMakeLists.txt` | **登记 2 个新源文件** | 【确定】ayu 源是**显式列举**，非 glob：`Telegram/CMakeLists.txt:104` `set(ayugram_files` … `:262` `${ayugram_files}`，ayu/data 段在 `:244-248`。**漏登记 = 链接期 undefined symbol** |
| `Telegram/SourceFiles/ayu/ayu_settings.h/.cpp` | （可选）菜单显隐开关 | 样板 `:306` |

---

## 5. 崩溃安全与并发

### 5.1 崩溃安全 —— 【确定】单行提交是安全的，但有两个真实风险

| 维度 | 事实 | 证据 |
|---|---|---|
| 事务 | 每个写函数 `begin_transaction` / `commit` / `rollback` | `ayu_database.cpp:290-302, 245-257, 461-473` |
| journal 模式 | **未设置** → SQLite 默认 `delete`（rollback journal） | §7 命令 18 全树无 `journal_mode` 命中 |
| synchronous | **未设置** → 默认 `FULL` | 同上 |
| 崩溃后果 | 事务中途崩溃 → 残留 `-journal`，下次打开自动回滚；**不会损坏文件**，最多丢最后一次未提交的写 | SQLite 标准行为 |

**风险 R1：整库轮转（高影响）**。`runMigrations` 失败（`ayu_database.cpp:198-204`）或 `initialize()` 异常（`:234-242`）→ `moveCurrentDatabase()`（`:211-225`）把 `ayudata.db` 改名。⇒ **V2 必须 no-op**（§4.2）；未来任何真迁移都必须包 try/catch 且**不得让失败逃出**（V1 的形态是对的：`:154-156` catch 后只 LOG）。

**风险 R2：死 WAL 代码暗示作者的 WAL 意图**（`:218-224` 处理 `-shm`/`-wal`）。**本次不要动**；真要启用 WAL，必须同批加 `storage.busy_timeout(...)`，否则并发 open 撞 `SQLITE_BUSY` 且**无自动重试**（sqlite 默认 busy_timeout=0）。

### 5.2 并发 —— Checkpoint 自身不引入新线程

【确定】所有 `AyuCheckpoints` 调用点都在**主线程**：
- W1/W2/W3：Qt 菜单/事件回调（`context_menu.cpp` 全部在 UI 线程）。
- peer 打开：`MainWidget::showHistory`（`mainwidget.cpp:1386`）在主线程。

【确定】`Main::Session` **不是线程**：`Telegram/SourceFiles/main/main_session.h:107` `class Session final : public base::has_weak_ptr {`；`Telegram/SourceFiles/data/data_session.h:167` `class Session final {`（无 `: public Thread`）。`grep -rn "ThreadPool" Telegram/SourceFiles/main/ Telegram/SourceFiles/data/data_session.h` → **0 命中**（§7 命令 30）。

【确定】MTP 侧的网络 I/O 确实有 QThread：`mtp_instance.cpp:226-228`
```cpp
    std::unique_ptr<QThread> _mainSessionThread;
    std::unique_ptr<QThread> _otherSessionsThread;
    std::vector<std::unique_ptr<QThread>> _fileSessionThreads;
```
但响应处理被排回主线程：`Telegram/SourceFiles/mtproto/session.h:133` `class Session final : public QObject {`；`Telegram/SourceFiles/mtproto/session.cpp:542-559` `void Session::tryToReceive() { ... auto messages = base::take(_data->haveReceivedMessages()); ... }`（`:553`），由 `session_private.cpp:1426-1429` 的 `queueTryToReceive()` 触发（DEBUG 文本字面写着 `need to parse in another thread`）；`mtp_instance.cpp:1204-1208`
```cpp
void Instance::Private::processUpdate(const Response &message) {
	if (_updatesHandler) {
		_updatesHandler(message);
	}
}
```

【确定】现有全部 `AyuDatabase` 调用点的线程归属（`grep -rn "AyuMessages::"` + codegraph callers）：

| 调用点 | 触发源 | 线程 |
|---|---|---|
| `AyuMessages::hasRevisions` ← `ayu/ui/context_menu/context_menu.cpp:499` | 右键菜单构建 | 主线程【确定】 |
| `AyuMessages::getDeletedMessages/getEditedMessages` ← `ayu/ui/message_history/history_inner.cpp:758, 760` | 历史面板翻页 | 主线程【确定】 |
| `AyuMessages::clearDeletedMessages` ← `ayu/ui/context_menu/context_menu.cpp:68` | 菜单 | 主线程【确定】 |
| `AyuMessages::addEditedMessage` ← `Telegram/SourceFiles/data/data_session.cpp:2937` | 编辑消息落库路径 | 主线程【确定】（经 `Api::Updates`） |
| `AyuMessages::addDeletedMessage` ← `ayu/utils/telegram_helpers.cpp:735`（`processMessageDelete`）← `data_session.cpp:3150`（`processMessagesDeleted`）← **codegraph** `api/api_updates.cpp:1314` `applyUpdateNoPtsCheck` | 更新流 | 主线程【确定】 |
| `AyuMessages::removeDeletedMessage` ← `Telegram/SourceFiles/data/data_histories.cpp:998`；`ayu/ui/context_menu/context_menu.cpp:81` | 删除 / 菜单 | 主线程【确定】 |
| `FiltersCacheController::rebuildCache` ← `ayu/ayu_settings.cpp:468, 476`；`ayu/ui/settings/settings_filters.cpp:64, 82, 99, 244`；`.../settings_filters_list.cpp:121, 134, 166, 186`；`ayu/features/filters/filters_utils.cpp:939` | 设置/UI | 主线程【确定】 |

**⇒ 直接回答问题 1 的最后一问：UI 线程可以直接同步调用 `AyuDatabase`，不需要 proxy 到数据库线程——因为 AyuGram 根本没有数据库线程。**（这与 tdesktop 上层的 `Storage::Facade` / `Storage::Worker` 线程池是两套东西：Ayu 的 sqlite_orm 栈**完全绕开**了 `Storage::Facade`，见 `main_session.cpp:153` `_storage(std::make_unique<Storage::Facade>())` 与 Ayu 代码零交叉。）

### 5.3 残余不确定性（**最高风险项**）

【无法确定（需外部构建机核对）】`storage` 单例**无互斥量**，其线程安全依赖 §5.2 那条"MTP 响应经 `InvokeQueued` 排回主线程"的链条，而**该链条的最后一环 `InvokeQueued` 的定义不在本 checkout 上**（见 §8 第 1 行）。

- 【确定】`InvokeQueued` 的**调用点**在盘上：`mtproto/session.cpp:43, 187, 201, 211`；`mtproto/details/mtproto_tcp_socket.cpp:31` `InvokeQueued(this, [=] { handler(args...); });`；`mtproto/details/mtproto_tls_socket.cpp:621, 789, 905`；`mtproto/details/mtproto_domain_resolver.cpp:251`；`mtproto/connection_resolving.cpp:215`。
- 【确定】`sqlite_orm` 的 `connection_holder` retain/release 是**无锁整型**（`sqlite_orm.h:17765-17771`、`:18169-18176`、`:18178-18184`）。若任一调用点不在主线程 → retain 计数竞争 → 连接提前关闭/重复打开。
- 【无法确定】`InvokeQueued` 的确切实现（是否 `QMetaObject::invokeMethod(..., Qt::QueuedConnection, ...)`）——**未找到**，因为 `Telegram/lib_base/` 子模块未初始化（`Telegram/lib_base/base/peer_id.h` 不存在，§7 命令 17）。因此该推断链缺一环。

**对 Checkpoint 的处置（无论上述能否确认，都应做）**：
1. 所有 `AyuCheckpoints` API 内部 `std::lock_guard<std::mutex>` 保护缓存 —— 成本为零（QPS 极低），换来对未来重新引入 session 线程的免疫。
2. `AyuDatabase::setCheckpoint` / `clearCheckpoint` / `getCheckpoints` 内部各加一把**独立的** `std::mutex`（同 §5.3-1），**不要**复用缓存锁（避免"持锁期间做 DB IO"）。
3. **不要**为了规避这个风险去改 `Main::Session` 的线程模型——那是重写 tdesktop 核心。

### 5.4 其他已识别风险

| # | 风险 | 等级 | 缓解 |
|---|---|---|---|
| K1 | 漏登记 `Telegram/CMakeLists.txt:104-262` → 链接期 undefined symbol | 中（必然踩） | §4.6 已列出；与 `messages_storage`（`:244-248`）并列登记 |
| K2 | `msgId` 用 `int` 会窄化（`entities.h:23` 既有缺陷） | 低（本需求必然命中正常范围） | Checkpoint 用 `ID`（§3.5） |
| K3 | 索引改名留下孤儿索引（`sqlite_orm.h:23249-23256` 只建不删） | 低 | 本设计不新增索引（§4.1.3） |
| K4 | `DialogId` 负号约定被后来者误当 bug "修掉" | 中（隐性） | 在 `entities.h` 的 `Checkpoint` 上方写注释说明（唯一允许的注释类型：解释约定而非复述代码） |
| K5 | 登出不清理 → 换号重登后残留旧检查点 | 低 | W5；与 `DeletedMessage` 的既有行为一致（同样不清理），非本任务引入 |
| K6 | 论坛 topic 检查点随 topic 删除而失效 | 低 | 降级清理 W4（姊妹文档 §5.2） |
| K7 | 迁移 V2 非 no-op | **高**（会整库轮转） | §4.2 明令；code review 卡点 |
| K8 | 每语句 open/close 连接在保存检查点时产生可感知卡顿 | 低 | 单行事务 ≈ 一次 open/close，亚毫秒级【无法确定，需实测】 |

---

## 6. 外部构建机验证清单（承接 ADR-0004）

| # | 验证项 | 期望 |
|---|---|---|
| V1 | `make_table<Checkpoint>` + 复合主键能通过 sqlite_orm 编译 | 通过 |
| V2 | 旧 `ayudata.db`（无 Checkpoint 表）上启动 → 自动建表 | 通过；`SELECT name FROM sqlite_master WHERE name='Checkpoint'` 有行 |
| V3 | `SchemaVersion.version` 变为 2 | 通过 |
| V4 | `storage.replace(...)` 在复合主键下是否可用 | 【无法确定】——若可用可简化 `setCheckpoint`；不可用则用既有 delete+insert（默认走这条） |
| V5 | `sync_schema(true)` 在**新表已存在**时是否幂等（第二次启动不报错） | 通过（`initialize()` 会连调两次，`:229` 与 `:233`） |
| V6 | 迁移失败路径不会被触发 | 通过（V2 no-op） |
| V7 | 多账号：A/B 两账号各存一个检查点，`tdata/ayudata.db` 内应有 2 行，`userId` 不同 | 通过 |
| V8 | 反复保存同一 peer 100 次 → 表内仍只有 1 行 | 通过（upsert 语义） |

---

## 7. 实际执行的命令（完整清单）

> 全部为只读命令。**未执行**任何 `git` / 构建 / docker / install / 文件写入（除本报告）。

**codegraph**
```bash
codegraph status                                    # 2,908 files / 98,735 nodes / 334,971 edges
codegraph callers processMessageDelete
codegraph callers addDeletedMessage                 # → No callers found（走 messages_storage 的门面，非直连）
codegraph callers processMessagesDeleted
codegraph callers "Session::handleMessages"
codegraph callers "Session::start"                  # → 命中 mtproto/session.cpp（非 Main::Session）
codegraph callers startMtp
codegraph callers rebuildCache                      # → No callers found（命名空间限定导致）
codegraph callers processUpdate
codegraph callers tryToReceive
codegraph query "Session::start"
codegraph query "handleMessages"
```

**grep / glob / find（节选，均带路径范围）**
```bash
grep -rn "AyuInfra::init\("                     Telegram/SourceFiles            # → application.cpp:306
grep -rn "AyuMessages::"                        Telegram/SourceFiles            # → 7 个调用点
grep -rn "AyuDatabase::initialize\|FiltersCacheController::" Telegram/SourceFiles
grep -rn "getDialogIdFromPeer"                  Telegram/SourceFiles            # → .h:59 + 8 个调用点
grep -rn "using DialogId\|class DialogId\|struct DialogId\|using PeerDialogId\|DialogId =" Telegram/SourceFiles/data/*.h   # → 0 命中
grep -rn "kChatTypeMask"                        Telegram/SourceFiles
grep -rn "PeerId\b" / "hash<PeerId>"            Telegram/SourceFiles/data/data_peer_id.h
grep -rn "busy_timeout\|journal_mode\|storage.pragma\|open_forever\|foreign_keys" Telegram/SourceFiles/ayu --include=*.cpp --include=*.h | grep -v libs/sqlite/sqlite_orm.h
                                                  # → 仅 sqlite3.h 注释命中，ayu 代码 0 命中
grep -n "sync_schema" / "retain_connection\|on_open\|on_close" / "CREATE TABLE" / "PRIMARY KEY" / "REPLACE INTO" / "sqlite_type<long long>" / "NOT NULL" \
                                                    Telegram/SourceFiles/ayu/libs/sqlite/sqlite_orm.h
grep -rn "SpyMessageRead"                       Telegram/SourceFiles            # → 仅 schema，0 处使用
grep -rn "startMtp\|Session::Session"           Telegram/SourceFiles/main/
grep -rn "Local::start" / "void Local::start"   Telegram/SourceFiles
grep -rn "QDir::setCurrent"                     Telegram/SourceFiles            # → logs.cpp:382
grep -rn "ThreadPool"                           Telegram/SourceFiles/main/ Telegram/SourceFiles/data/data_session.h   # → 0 命中
grep -rn "crl::on_main"                         Telegram/SourceFiles/data/data_session.cpp Telegram/SourceFiles/mtproto/*.cpp
grep -rn "InvokeQueued"                         Telegram/SourceFiles/mtproto/   # → 仅调用点，无定义
grep -rn "processUpdate" / "tryToReceive\|readStreamFromSocket" Telegram/SourceFiles/mtproto/mtp_instance.cpp
grep -rn "ayu/data/"                            Telegram/CMakeLists.txt          # → :244-248
grep -n "ayu_ContextHideMessage"                Telegram/Resources/langs/lang.strings  # → :9010
grep -rn "session thread\|Session thread"       Telegram/SourceFiles/main/*.cpp Telegram/SourceFiles/data/data_session.cpp   # → 0 命中
grep -c "primary_key()"                         Telegram/SourceFiles/ayu/data/ayu_database.cpp  # → 9（9 张表各 1 个，无复合主键先例）
ls -la Telegram/lib_base/base/peer_id.h         # → No such file（子模块未初始化）
grep -rn "InvokeQueued"                         Telegram/SourceFiles/base/*.h   # → No such file or directory
find Telegram/SourceFiles -name "history_view_context_menu*" -o -name "mainwidget*" -o -name "main_local*" -o -name "main_widget.h" -o -name "main_local*"
glob  Telegram/SourceFiles/ayu/**/*.{h,cpp}     # glob  CONTEXT.md   glob  docs/adr/*.md
```

---

## 8. 未找到项（逐条，含命令与搜索范围）

| 项 | 状态 | 搜索范围与命令 |
|---|---|---|
| `InvokeQueued` 定义 | **未找到** | `grep -rn "InvokeQueued" Telegram/SourceFiles/mtproto/`（仅调用点）；`grep -rn "void InvokeQueued\|auto InvokeQueued\|template.*InvokeQueued" Telegram/SourceFiles/base/*.h` → `No such file or directory`（`Telegram/SourceFiles/base/` 目录不存在；`Telegram/lib_base/` 为未初始化子模块，`Telegram/lib_base/base/peer_id.h` 不存在） |
| tdesktop `DialogId` 类型 | **不存在于本 fork** | `grep -rn "using DialogId\|class DialogId\|struct DialogId\|using PeerDialogId\|DialogId =" Telegram/SourceFiles/data/*.h` → 0 命中 |
| Ayu 代码里的 `busy_timeout` / `journal_mode` / `open_forever` | **未设置** | `grep -rn "busy_timeout\|journal_mode\|storage.pragma\|open_forever\|foreign_keys" Telegram/SourceFiles/ayu --include=*.cpp --include=*.h \| grep -v libs/sqlite/sqlite_orm.h` → 仅 `libs/sqlite/sqlite3.h` 的注释命中 |
| 复合主键 + `storage.replace` 的本仓库先例 | **不存在** | `grep -c "primary_key()" Telegram/SourceFiles/ayu/data/ayu_database.cpp` = 9，对应 9 张表各 1 个 → 无一表使用复合主键 |
| `AyuDatabase` 的 mutex / 线程约束注释 | **未找到** | `grep -rn "mutex\|QMutex\|thread" Telegram/SourceFiles/ayu/data/*.cpp Telegram/SourceFiles/ayu/data/*.h` → 无命中（`messages_storage.cpp`/`ayu_database.cpp` 全文无同步原语） |
| 数据库查询耗时基线（预载 <1ms？） | **无法确定，需实测** | 无 profiling 设施在盘上；ADR-0004 已定"构建与实测均在外部构建机" |

---

## 9. 一句话总结

在 `ayu_database.cpp:14` 的全局 `storage` 上按 `DeletedMessage` 模式加一张 **5 列、复合主键 `(userId, dialogId, topicId)`、零额外索引**的 `Checkpoint` 表，用一个 **同步、try/catch 包裹、绝不抛出** 的 5 函数 `AyuDatabase` 层 + **per-account 惰性加载的三层内存缓存**（照抄 `filters_cache_controller.cpp:90-142` 形状，但键带 `userId` 维度）；**打开 peer 前 DB 必然已就绪**（`application.cpp:306 < :349`），但**账号维度要延后到 per-session 或惰性加载**；Checkpoint 不引入新线程（全部主线程），唯一无法自证的风险是 `storage` 单例无锁而其线程安全依赖一条因 `lib_base` 未初始化而**缺一环**的推断。

## 主线程抽查（2025-09-27）
- ✅ 复核 `ayu_database.cpp:14` `auto storage = make_storage("./tdata/ayudata.db", ...)`：文件级单例断言成立。
- ⚠️ `DialogId` 类型是否存在：主线程 grep 未定位到 `data_dialog_id.h`，该点维持【无法确定】，需外部机核对。
