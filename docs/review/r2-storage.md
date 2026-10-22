# R2 存储层对抗式审查报告 — 检查点（Checkpoint）存储

- **仓库**：`/root/workspace/AyuGramDesktop`（分支 `dev`，工作区未提交改动）
- **审查对象**：检查点存储层已落地代码 + 其 UI 契约调用点
- **审查方式**：**只读**。仅执行 `git --no-pager diff/status/rev-parse`、`git submodule status`（只读子命令）、`read`、`grep`、`awk`、`sed -n`、`wc`、`find`、`ls`。**未修改任何源文件，未执行任何 git 写操作，未构建。**
- **产出**：仅本文件 `docs/review/r2-storage.md`（任务允许的唯一写入）
- **日期基准**：工作区状态 = `git --no-pager status --porcelain` 输出（见 §7 命令清单）

---

## 0. 结论速览（Verdict）

| 严重度 | 数量 | 编号 |
|---|---|---|
| 【确定 BUG】高 | 2 | BUG-1、BUG-2 |
| 【确定 BUG】中 | 2 | BUG-3、BUG-4 |
| 【风险】 | 6 | R-1 … R-6 |
| 【无法确定】 | 3 | U-1 … U-3 |

**Verdict：存储层本身的 sqlite_orm 用法与事务语义成立（无数据丢失），但 API 契约存在确定性缺陷；跨层看该特性在论坛（topic）会话中完全失效，且消息 id 存在真实窄化截断。**

---

## 1. 【确定 BUG】

### BUG-1（高）`MsgId::bare` 是 `int64` 而非 `int32`，`messageId` 存在真实窄化截断；且双击手势路径缺少 `isLocal()` 守卫

**任务前提「`MsgId::bare` 语义是 int32」不成立。** 证据：

`Telegram/SourceFiles/data/data_msg_id.h:48`
```cpp
	int64 bare = 0;
```

`Telegram/SourceFiles/data/data_msg_id.h:21`
```cpp
	constexpr MsgId(int64 value) noexcept : bare(value) {
```

**本地/待发消息 id 是巨大负数**（窄化必然发生）：

`Telegram/SourceFiles/data/data_msg_id.h:78`
```cpp
constexpr auto StartClientMsgId = MsgId(0x01 - (1LL << 58));
```
`data_msg_id.h:101-103`
```cpp
[[nodiscard]] constexpr inline bool IsClientMsgId(MsgId id) noexcept {
	return (id >= StartClientMsgId && id < EndClientMsgId);
}
```

**服务端 id 上界为 2^56**（超出 int32）：

`Telegram/SourceFiles/data/data_msg_id.h:85`
```cpp
constexpr auto ServerMaxMsgId = MsgId(1LL << 56);
```
`data_msg_id.h:129-131`
```cpp
[[nodiscard]] constexpr inline bool IsServerMsgId(MsgId id) noexcept {
	return (id > 0 && id < ServerMaxMsgId);
}
```

`Telegram/SourceFiles/history/history_item.h:90`
```cpp
	MsgId id = 0;
```

**写入侧的窄化点**：

`Telegram/SourceFiles/ayu/data/messages_storage.cpp:149-158`
```cpp
void setCheckpoint(not_null<HistoryItem*> item) {
	const auto peer = item->history()->peer;
	const ID userId = peer->session().userId().bare & PeerId::kChatTypeMask;
	const ID topicId = item->topic() ? item->topicRootId().bare : 0;
	AyuDatabase::setCheckpoint(
		userId,
		getDialogIdFromPeer(peer),
		topicId,
		item->id.bare);
}
```

`Telegram/SourceFiles/ayu/data/ayu_database.h:34`
```cpp
void setCheckpoint(ID userId, ID dialogId, ID topicId, int messageId);
```

`item->id.bare`（`int64`）→ 形参 `int messageId`：**隐式窄化转换**。

**守卫不对称（确定 BUG 的核心）**：

| 入口 | 是否过滤 `isLocal()` | 证据 |
|---|---|---|
| 右键菜单 `AyuUi::AddCheckpointAction` | ✅ 有 | `Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:1031-1034` — `void AddCheckpointAction(...) { if (item->isLocal()) { return; }` |
| 双击手势 `HistoryInner::checkpointItemAt` | ❌ **无** | `Telegram/SourceFiles/history/history_inner_widget.cpp:2886-2899` — 回调体只有几何比较 + `result = view->data();`，无任何 `isLocal()` 过滤 |

`Telegram/SourceFiles/history/history_inner_widget.cpp:2886`
```cpp
HistoryItem *HistoryInner::checkpointItemAt(int y) {
	auto result = static_cast<HistoryItem*>(nullptr);
	enumerateItems<EnumItemsDirection::TopToBottom>(
		[&](not_null<Element*> view, int top, int bottom) {
```

`Telegram/SourceFiles/history/history_inner_widget.cpp:2845-2852`（双击手势写入点）
```cpp
	if (AyuSettings::getInstance().saveCheckpointOnDoubleRightClick()
		&& e->reason() == QContextMenuEvent::Mouse) {
		if (_checkpointGestureTimer.isActive()) {
			_checkpointGestureTimer.cancel();
			if (_checkpointGestureItem) {
				AyuMessages::setCheckpoint(_checkpointGestureItem);
			}
```

**后果链**：
1. 对本地（待发送）消息双击右键 → `item->id.bare` 是 `≈ -2^58` 的 `int64`。
2. 窄化为 `int` 后得到任意垃圾值（可能是任意正数）。
3. 读回时 `checkpoint.messageId > 0` 判据成立 → 菜单出现「跳转到检查点」→ `MsgId(checkpoint.messageId)` 跳向不存在/错误的消息。

**旁证（该代码库自身把 `id <= 0` 当无效）**：
`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:815`
```cpp
	if (!item || !item->isHistoryEntry() || item->isService() || item->isLocal() || !item->allowsForward() || !item->id <= 0) {
```

**注**：既有的 `AyuMessageBase::messageId` 也是 `int`（`Telegram/SourceFiles/ayu/data/entities.h:23`），且 `messages_storage.cpp:45` `message.messageId = item->id.bare;` 同样窄化 —— 属**既有技术债**，但检查点新代码复刻了该窄化且**新增了一条无守卫的可达路径**。

---

### BUG-2（高）`dialogs_widget.cpp` 硬编码 `topicId = 0`，论坛会话的检查点永远读不到

写入侧 key 里的 `topicId` 来自消息：

`Telegram/SourceFiles/ayu/data/messages_storage.cpp:152`
```cpp
	const ID topicId = item->topic() ? item->topicRootId().bare : 0;
```

读取侧**固定传 0**：

`Telegram/SourceFiles/dialogs/dialogs_widget.cpp:1197-1201`
```cpp
		if (!inSearchResults) {
			const auto checkpoint = AyuMessages::getCheckpoint(peer, 0);
			if (checkpoint.messageId > 0) {
				showAtMsgId = MsgId(checkpoint.messageId);
			}
		}
```

存储层的 where 条件是三元组（`ayu_database.cpp:628-632`）：
```cpp
		const auto result = storage.get_all<Checkpoint>(
			where(
				column<Checkpoint>(&Checkpoint::userId) == userId and
				column<Checkpoint>(&Checkpoint::dialogId) == dialogId and
				column<Checkpoint>(&Checkpoint::topicId) == topicId
			),
```

→ **论坛（forum）会话中，保存时 `topicId != 0`、读取时 `topicId == 0`，永远不匹配。** 该特性只在非 topic 会话生效。

**同类第二处**：`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:267-269`
```cpp
	const auto topic = peerData->isForum() && thread ? thread->asTopic() : nullptr;
	const auto topicId = topic ? topic->rootId().bare : 0;
	const auto checkpoint = AyuMessages::getCheckpoint(peerData, topicId);
```
`topicId` 仅在调用方传入 `thread` 时非 0。从会话列表直接右键（无 thread）→ `topicId = 0` → 同样读不到 forum topic 的检查点。

---

### BUG-3（中）`context_menu.cpp` 新增的早退是死代码，检查点菜单项仍被旧设置门控

`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:260-273`
```cpp
	const auto showFilters = settings.filtersEnabled()
		&& (!user || user->isBot());
	const auto saveDeletedMessages = settings.saveDeletedMessages();
	if (!showFilters && !saveDeletedMessages) {
		return;
	}

	const auto topic = peerData->isForum() && thread ? thread->asTopic() : nullptr;
	const auto topicId = topic ? topic->rootId().bare : 0;
	const auto checkpoint = AyuMessages::getCheckpoint(peerData, topicId);
	const auto hasCheckpoint = checkpoint.messageId > 0;
	if (!showFilters && !saveDeletedMessages && !hasCheckpoint) {
		return;
	}
```

- 第 263-265 行是**既有**早退：`!showFilters && !saveDeletedMessages` 时已 `return`。
- 第 271-273 行是**新增**早退，其条件是第 263 行条件的**严格超集** → **永远不可达**。

**后果**：当 `filtersEnabled` 与 `saveDeletedMessages` 都关闭时，即使用户已保存检查点，AyuGram 子菜单也不出现（「跳转到检查点」/「清除检查点」两项被吞掉）。`hasCheckpoint` 分支（`context_menu.cpp:328-347`）是纯死代码。

---

### BUG-4（中）`getCheckpoint` 的「无记录」与「读取失败」返回同一个值，调用方无法区分

`Telegram/SourceFiles/ayu/data/ayu_database.cpp:625-641`
```cpp
Checkpoint getCheckpoint(ID userId, ID dialogId, ID topicId) {
	try {
		const auto result = storage.get_all<Checkpoint>(
			...
		);
		return result.empty() ? Checkpoint() : result.front();
	} catch (std::exception &ex) {
		LOG(("Failed to read checkpoint for some reason: %1").arg(ex.what()));
		return Checkpoint();
	}
}
```

- 第 636 行（无行）与第 639 行（**任何** `std::exception`：SQLITE_BUSY / SQLITE_CORRUPT / 磁盘错误）返回完全相同的值。
- 调用方判据只有 `messageId > 0`（`context_menu.cpp:270`、`dialogs_widget.cpp:1199`）。
- **后果**：数据库异常被静默降级为「用户没有检查点」。用户既看不到错误、也无法区分「误删了」和「DB 坏了」，检查点凭空消失且无提示。

**返回值语义不明确的第二个面**：`messageId == 0` 被复用为「空」标记。该值作为存储 key 的一部分可读回，但由于判据只看 `> 0`，任何 `messageId <= 0` 的行（见 BUG-1 造成的截断垃圾值）都会被当作「无检查点」而永不显示、永不清理 —— 形成不可见的僵尸行。

---

## 2. 【风险】（非确定 BUG，但应记录）

### R-1 `setCheckpoint` 在 UI 线程做同步事务提交（含 fsync）

`Telegram/SourceFiles/ayu/data/ayu_database.cpp:597-623` — 完整事务：`begin_transaction()` → `remove_all` → `insert` → `commit()`。
`sqlite_orm.h:18028-18030`
```cpp
            void begin_transaction() {
                this->begin_transaction_internal("BEGIN TRANSACTION");
            }
```
`sqlite_orm.h:18044-18051` — `COMMIT` 同步下发，触发 WAL fsync。

调用点均在 UI 线程且属**手势路径**（双击右键，`history_inner_widget.cpp:2845-2860`）。既有 `addEditedMessage`（`ayu_database.cpp:258-270`）同款，但那是低频的用户主动操作。缓解因素：单行表 + 三列复合索引，事务极短。仍属可感知卡顿风险。

### R-2 COMMIT 失败时 sqlite_orm 连接引用计数泄漏

`Telegram/SourceFiles/ayu/libs/sqlite/sqlite_orm.h:18044-18060`
```cpp
            void commit() {
                sqlite3* db = this->connection->get();
                perform_void_exec(db, "COMMIT");
                this->connection->release();
                if (this->connection->retain_count() < 0) {
                    throw std::system_error{orm_error_code::no_active_transaction};
                }
            }

            void rollback() {
                sqlite3* db = this->connection->get();
                perform_void_exec(db, "ROLLBACK");
                this->connection->release();
```

`sqlite_orm.h:13745-13776` — `retain_count()` 归零时才 `sqlite3_close`。
`sqlite_orm.h:18169-18176` — `begin_transaction_internal` 先 `retain()`。

`ayu_database.cpp:615-620` 的 catch 会再调 `storage.rollback()`；若 COMMIT 已因 SQLITE_BUSY/SQLITE_FULL 失败，ROLLBACK 同样失败 → 两次 `release()` 都被跳过 → 引用计数单调偏高 → `sqlite3_close` 永不执行 → Windows 上 `moveCurrentDatabase()`（`ayu_database.cpp:224-238`，用 `QFile::rename`）会因文件句柄占用而失败。属窄路径风险。

### R-3 全局 `storage` 无锁，`AyuDatabase` 是进程级共享对象

`Telegram/SourceFiles/ayu/data/ayu_database.cpp:14`
```cpp
auto storage = make_storage(
	"./tdata/ayudata.db",
```
本次新增的三个检查点函数**全部**只在 UI 线程被调用，暂不构成并发。但 `AyuDatabase::` 命名空间还有 filters / deleted-messages 等 API 被 `Telegram/SourceFiles/ayu/features/filters/*`、`Telegram/SourceFiles/ayu/ui/settings/filters/*` 使用；若其中存在非 UI 线程调用并与检查点事务交错，`retain_count` 与 ROLLBACK 语义会错乱。【无法确定】——未逐一枚举其它 `AyuDatabase::` 调用点。

### R-4 `checkpoint.fakeId = 0;` 是误导性死代码（但**无害**——任务点 1 的疑虑不成立）

`Telegram/SourceFiles/ayu/data/ayu_database.cpp:607-608`
```cpp
		Checkpoint checkpoint;
		checkpoint.fakeId = 0;
```

**sqlite_orm 对非 WITHOUT ROWID 表的 `insert` 会排除主键列** —— 两处证据：

序列化端 `Telegram/SourceFiles/ayu/libs/sqlite/sqlite_orm.h:20811-20832`
```cpp
        template<class T>
        struct statement_serializer<insert_t<T>, void> {
            ...
                table.template for_each_column_excluding<
                    mpl::conjunction<mpl::not_<mpl::always<is_without_rowid>>,
                                     mpl::disjunction_fn<is_primary_key, is_generated_always>>>(
                    [&table, &columnNames](auto& column) {
                        if (exists_in_composite_primary_key(table, column)) {
                            return;
                        }
                        columnNames.push_back(cref(column.name));
                    });
                ...
                std::stringstream ss;
                ss << "INSERT INTO " << streaming_identifier(table.name) << " ";
```

绑定端 `sqlite_orm.h:23590-23605`（`prepared_statement_t` 的 insert 执行器）
```cpp
                    using is_without_rowid = typename std::remove_reference_t<decltype(table)>::is_without_rowid;
                    table.template for_each_column_excluding<
                        mpl::conjunction<mpl::not_<mpl::always<is_without_rowid>>,
                                         mpl::disjunction_fn<is_primary_key, is_generated_always>>>(
```

→ 实际 SQL 是 `INSERT INTO "Checkpoint" ("userId","dialogId","topicId","messageId","date") VALUES (...)`，`fakeId` 由 AUTOINCREMENT 赋值。**与既有 `EditedMessage`（`ayu_database.cpp:66-101`，`fakeId` 从不赋值却被 `addEditedMessage` insert，`ayu_database.cpp:258-270`）行为完全一致。**

编译期静态断言也满足：`sqlite_orm.h:22345-22358` 要求 PK 列数 ≤ 1 且全部为「标准 PK」——`Checkpoint` 只有 1 个 PK 列，符合。

结论：不会因 `fakeId = 0` 触发 UNIQUE 冲突。但保留这行会让人误以为 PK 参与写入。

### R-5 `date` 字段窄化 + 无任何消费者

`Telegram/SourceFiles/ayu/data/ayu_database.cpp:613`
```cpp
		checkpoint.date = base::unixtime::now();
```
`base::unixtime::now()` 返回 `TimeId`（int64）→ 存入 `int date`（`entities.h:147`）。2038 问题实际不触发，因为 `getCheckpoint` 只用 `messageId` / `fakeId`，`date` 列**无读取方**——是纯死存储（每行多占 4 字节 + 一次无用的写入）。

### R-6 `entities.h` 非自包含（既有债，新代码未加剧）

`Telegram/SourceFiles/ayu/data/entities.h:9`
```cpp
#include <string>
```
但同文件 `:68` 用 `std::unique_ptr`、`:78/:79` 用 `std::vector`、`:83` 用 `std::optional`、`:93-104` 用 `QJsonObject` / `QString` / `QStringLiteral`。

`Telegram/SourceFiles/ayu/data/ayu_database.h:9,35`
```cpp
#include "ayu/data/entities.h"
...
Checkpoint getCheckpoint(ID userId, ID dialogId, ID topicId);
```

`Telegram/SourceFiles/ayu/data/messages_storage.h:9,13-25`
```cpp
#include "ayu/data/entities.h"
...
void addEditedMessage(not_null<HistoryItem *> item);
std::vector<AyuMessageBase> getEditedMessages(..., const QString &searchQuery = QString());
...
void setCheckpoint(not_null<HistoryItem*> item);
```

`messages_storage.h` 只包含 `entities.h`，却使用 `not_null<>` / `QString` / `QStringLiteral`。依赖调用方的 include 顺序兜底。新增的 `Checkpoint`（`entities.h:139-148`）是纯 POD，未引入新依赖，故不构成新问题——但值得记录。

---

## 3. 【无法确定】

### U-1 `not_null<HistoryItem*>` → `HistoryItem*` 的隐式转换是否成立

`Telegram/SourceFiles/history/history_inner_widget.cpp:2892`（新代码）
```cpp
			result = view->data();
```
`Telegram/SourceFiles/history/view/history_view_element.h:460`
```cpp
	[[nodiscard]] not_null<HistoryItem*> data() const;
```

`result` 声明为 `HistoryItem *`（`history_inner_widget.cpp:2887`）。若 `base::not_null` 提供非 explicit 的 `operator T()` 则编译通过，否则为编译错误。

**未找到**。执行过的确切命令与范围：

| 命令 | 范围/结果 |
|---|---|
| `find Telegram -name "not_null*"` | 整个 `Telegram/`，0 命中 |
| `find . -maxdepth 4 -type d -name base` | 仓库根起 4 层，0 命中 |
| `ls Telegram/SourceFiles/ \| head -40` | 59 个条目，**无 `base/` 目录** |
| `ls -la Telegram/lib_base/` | 目录存在但**完全为空**（仅 `.` 与 `..`） |
| `git submodule status \| head -20` | 全部以 `-` 前缀 = **子模块未初始化** |
| `grep -rn "operator T()" --include=*.h Telegram/` | 0 命中 |

原因：`Telegram/lib_base` 是 git submodule（`.gitmodules` 中 `path = Telegram/lib_base`），本 checkout 未 `init`/`update`，`base/not_null.h` 根本不在磁盘上。同理 `base/unixtime.h`（`ayu_database.cpp:11` 包含）也不在。

### U-2 项目是否把窄化转换视为编译错误

`Telegram/SourceFiles/ayu/data/messages_storage.cpp:157` 的 `int64 → int` 隐式窄化是否触发 `-Werror` / clang-tidy `bugprone-narrowing-conversions`：

| 命令 | 结果 |
|---|---|
| `ls -la .clang-tidy` | 不存在 |
| `grep -n "narrowing" .clang-tidy` | 无此文件 |
| `grep -rn "WX\|Werror\|WARNINGS_AS_ERRORS" cmake/*.cmake CMakeLists.txt` | 0 命中 |

**未找到**相关配置，无法确定 MSVC `/W4` C4244 是否被提升为错误。既有 `messages_storage.cpp:45` 有同样的窄化且项目在发布，故大概率只是警告。

### U-3 编译/运行期行为

**未执行构建**（任务禁止；且 `git submodule status` 显示所有子模块未初始化，本 checkout 客观上无法构建）。所有结论均来自静态阅读 + sqlite_orm 头文件源码取证。

---

## 4. 任务 6 项必查问题的逐条回答

### Q1 — sqlite_orm 用法是否成立？✅ **全部成立**

| 用法 | 结论 | 证据 |
|---|---|---|
| `storage.remove_all<Checkpoint>(where(...))` | ✅ 成立 | 既有同款：`ayu_database.cpp:374-380`、`:388-394`、`:521-523`、`:541-545` |
| `storage.get_all<T>(where, order_by, limit)` | ✅ 成立 | 既有同款：`ayu_database.cpp:317-329`、`:341-352`、`:272-283` |
| `begin_transaction` / `commit` / `rollback` | ✅ 成立 | `sqlite_orm.h:18028-18030` / `18044-18051` / `18053-18060`；既有同款：`ayu_database.cpp:260-267`、`:305-312`、`:476-483` |
| 自增主键在 `insert` 时被自动排除 | ✅ **成立（任务疑虑不成立）** | `sqlite_orm.h:20811-20832`（序列化）+ `sqlite_orm.h:23590-23605`（绑定），均带 `for_each_column_excluding<... disjunction_fn<is_primary_key, is_generated_always>>`；与 `EditedMessage`（`ayu_database.cpp:66-101` + `:258-270`）行为一致 |

补充：`Checkpoint` 表定义（`ayu_database.cpp:145-153`）与 `DeletedMessage`（`:30-65`）形状一致（1 个 `primary_key().autoincrement()` + N 个普通列），满足 `assert_insertable_type` 的静态断言（`sqlite_orm.h:22345-22358`）。

### Q2 — 事务语义：会不会留下「已删旧行、未插新行」的空检查点？✅ **不会**

`Telegram/SourceFiles/ayu/data/ayu_database.cpp:598-615`
```cpp
	try {
		storage.begin_transaction();
		storage.remove_all<Checkpoint>(
			where(...)
		);
		Checkpoint checkpoint;
		...
		storage.insert(checkpoint);
		storage.commit();
	} catch (std::exception &ex) {
		try {
			storage.rollback();
		} catch (...) {
		}
```

- `remove_all` 与 `insert` **在同一个 `BEGIN`…`COMMIT` 内**，异常路径第 618 行 `storage.rollback()` 会原子地恢复旧行。
- 逐异常点核对：
  - `begin_transaction()` 抛 → 无任何写入，catch 中的 `rollback()` 会因「no transaction is active」抛异常，被内层 `catch (...)` 吞掉（`:619-620`）。**无数据影响。**
  - `remove_all` 抛 → ROLLBACK 生效，旧行保留。**无数据丢失。**
  - `insert` 抛 → ROLLBACK 生效，旧行恢复。**无数据丢失。**
  - `commit()` 抛 → 见 R-2（引用计数泄漏），**但不是「旧行被删」场景**。
- **唯一缺口**（R-2）：COMMIT 阶段失败时 rollback 也失败，此时 SQLite 自身已自动回滚该事务，数据仍完整；泄漏的是 sqlite_orm 的 `connection_holder::_retain_count`。

### Q3 — `Checkpoint()` 默认构造是否真能给出 `messageId == 0`？✅ **能（确定）**

`Telegram/SourceFiles/ayu/data/entities.h:139-148`
```cpp
class Checkpoint
{
public:
	ID fakeId;
	ID userId;
	ID dialogId;
	ID topicId;
	int messageId;
	int date;
};
```

无用户提供的构造函数、无基类、无虚函数、全部数据成员 public 且非 static → 满足聚合体（C++17 起 aggregate 定义）。`Checkpoint()`（带空括号的函数式转换）执行**值初始化**：对聚合体先零初始化再默认初始化 → 全部成员为 0。**`Checkpoint().messageId == 0` 成立。**

（`RegexFilter` 同款写法 `ayu_database.cpp:448 return {};` 已在生产使用，佐证该模式在本项目可行。）

**返回值语义：不明确** → 见 BUG-4。

### Q4 — 类型与截断

| 子问题 | 结论 |
|---|---|
| `messageId` 是 `int`，`MsgId::bare` 语义？ | **任务前提错误**：`bare` 是 `int64`（`data_msg_id.h:48`），且可为负（`data_msg_id.h:78` `StartClientMsgId = MsgId(0x01 - (1LL << 58))`）→ **存在窄化风险，且已实际发生**，见 BUG-1 |
| `userId` 用 `peer->session().userId().bare & PeerId::kChatTypeMask` 是否与既有 `map()` 完全一致？ | ✅ **完全一致** |
| `topicId` 语义 `item->topic() ? topicRootId().bare : 0`？ | ✅ **完全一致** |

**userId 一致性逐字对照**：

既有 `map()` — `Telegram/SourceFiles/ayu/data/messages_storage.cpp:32-33`
```cpp
void map(not_null<HistoryItem*> item, AyuMessageBase &message) {
	const ID userId = item->history()->owner().session().userId().bare & PeerId::kChatTypeMask;
```

既有 peer 版（3 处）— `messages_storage.cpp:127`、`:133`、`:139`、`:144`
```cpp
	const ID userId = peer->session().userId().bare & PeerId::kChatTypeMask;
```

新代码 — `messages_storage.cpp:151`、`:161`、`:169`
```cpp
	const ID userId = peer->session().userId().bare & PeerId::kChatTypeMask;
```

新代码与既有 peer 版**逐字相同**。唯一差异是 `map()` 用 `item->history()->owner().session()`、新代码用 `item->history()->peer` 再 `peer->session()` —— 但 `PeerData` 归属唯一 session，且本文件既有的 `removeDeletedMessage`（`:138-139`）/ `clearDeletedMessages`（`:143-144`）已是「先取 peer 再 peer->session()」写法。

`PeerId::kChatTypeMask` 定义 — `Telegram/SourceFiles/data/data_peer_id.h:110`
```cpp
	static constexpr BareId kChatTypeMask = BareId(0xFFFFFFFFFFFFULL);
```

**topicId 一致性逐字对照**：

既有 `map()` — `messages_storage.cpp:40-44`
```cpp
	if (item->topic()) {
		message.topicId = item->topicRootId().bare;
	} else {
		message.topicId = 0;
	}
```

新代码 — `messages_storage.cpp:152`
```cpp
	const ID topicId = item->topic() ? item->topicRootId().bare : 0;
```

逻辑等价。补充事实（不构成新问题）：`HistoryItem::topicRootId()` 在非 topic 时返回的是 `Data::ForumTopic::kGeneralId` 而非 0 —— `Telegram/SourceFiles/history/history_item.cpp:4367-4380`
```cpp
MsgId HistoryItem::topicRootId() const {
	...
	return Data::ForumTopic::kGeneralId;
}
```
`Telegram/SourceFiles/data/data_forum_topic.h:80` — `static constexpr auto kGeneralId = 1;`
由于两侧都被 `item->topic()` 三元短路守卫，行为完全相同。

### Q5 — 多账号隔离 / topicId 语义 / 崩溃安全 / UI 线程

| 子问题 | 结论 |
|---|---|
| **多账号隔离** | ✅ **成立**。`userId` 出现在全部三个函数的 where 三元组中：`ayu_database.cpp:602-604`（写/删）、`:629-631`（读）、`:647-649`（删）。读写用同一 key 空间，跨账号不串。 |
| **残留清理** | ⚠️ 未发现登出 / 删号 / 清空聊天历史时清理检查点的路径。`clearCheckpoint`（`ayu_database.cpp:643-655`）只能按精确三元组删 → 账号删除后会留下孤儿行。属轻微泄漏。 |
| **topicId 语义** | ⚠️ 存储层语义本身自洽，但**读取侧两处硬编码/条件化**（BUG-2）使 forum 场景失效。 |
| **DB 位置 / 崩溃安全** | `ayu_database.cpp:15` — `"./tdata/ayudata.db"`，与既有表同一文件、同一个 `storage` 实例 → 崩溃安全等级与既有数据相同。`remove_all`+`insert` 处于单事务，进程被杀时 WAL 保证原子性（要么旧行、要么新行，不会出现「无行」）。 |
| **UI 线程同步调用是否卡界面** | ⚠️ **是风险**（R-1）。`setCheckpoint` 每次调用一次完整的 `BEGIN; DELETE; INSERT; COMMIT`（含 fsync），位于 UI 线程的手势路径上。 |

### Q6 — 是否需要改 `runMigrations` 版本号？✅ **不需要**

`Telegram/SourceFiles/ayu/data/ayu_database.cpp:240-256`
```cpp
void initialize() {
	try {
		storage.sync_schema(true);

		runMigrations(storage);

		storage.sync_schema(true);
	} catch (const std::exception &ex) {
		LOG(("Database initialization failed: %1").arg(ex.what()));
		moveCurrentDatabase();

		storage.sync_schema(true);
		if (!storage.get_pointer<SchemaVersion>(1)) {
			storage.insert(SchemaVersion{1, 0});
		}
	}
}
```

`runMigrations` — `ayu_database.cpp:174-179`
```cpp
void runMigrations(decltype(storage) &storage) {
	constexpr int kLatestVersion = 1;
```

**顺序关键**：`sync_schema(true)` 在 `runMigrations` **之前**执行（`:242` vs `:244`）。

`sqlite_orm.h:23325-23343` 文档：
> * every table from storage is compared with it's db analog and
> * if table doesn't exist it is being created
> * if table exists its colums are being compared with table_info from db and
> * if there are columns in storage that do not exist in db they will be added using `ALTER TABLE ... ADD COLUMN ...' command

`preserve=true` 的作用范围（`sqlite_orm.h:23341-23343`）：
> * `@param preserve` affects function's behaviour **in case it is needed to remove a column**. ... if `true` - table is being copied into another table, dropped and copied table is renamed with source table name.

→ `preserve=true` **只影响删列路径**，不影响建表。

**结论：`Checkpoint` 表与 `idx_checkpoint_userId_dialogId_topicId` 索引会在启动时被 `sync_schema` 自动创建，已有用户数据库无需手动迁移，`kLatestVersion = 1` 无需变更。**

**附带既有风险**：`initialize()` 的 catch 分支（`:247-255`）会调 `moveCurrentDatabase()`（`:224-238`）把整个 DB 改名腾空。新增表增加了 `sync_schema` 的失败面（多一次 `CREATE TABLE` + `CREATE INDEX`），但该行为是既有的，且 `moveCurrentDatabase` 只改名不删除，用户数据文件仍在。

---

## 5. 与既有代码的一致性总评

| 维度 | 一致性 |
|---|---|
| `make_table` 形状 | ✅ 与 `DeletedMessage`/`EditedMessage` 完全同构 |
| 索引命名 | ✅ 遵循既有 `idx_<table>_<cols>` 约定（`ayu_database.cpp:21-29` vs `:154-157`） |
| 异常处理形状 | ✅ `try { begin; ...; commit; } catch { try{rollback;}catch(...){} LOG(...); }` 与 `addEditedMessage`（`:258-270`）逐字同构 |
| `userId` 派生 | ✅ 与 `map()` / 既有 peer 版逐字一致 |
| `topicId` 派生 | ✅ 与 `map()` 逻辑等价 |
| 事务粒度 | ✅ 单事务，无裸 remove-then-insert 竞态 |
| PK 写入 | ⚠️ `fakeId = 0` 多余（R-4） |
| 消息 id 类型 | ❌ 复刻了既有的 `int` 窄化债，且新增无守卫路径（BUG-1） |
| 空值语义 | ❌ 「无记录」与「异常」不可区分（BUG-4） |
| 读取侧 key 一致性 | ❌ 两处读取硬编码/条件化 `topicId`（BUG-2） |

---

## 6. 建议修复方向（仅建议，本次不改任何代码）

1. **BUG-1**：`Checkpoint::messageId` 改 `int64`（或 `ID`），`setCheckpoint` 形参同步改 `int64`；并让 `checkpointItemAt` 复用与 `AddCheckpointAction` 相同的 `isLocal()` / `id > 0` 守卫（`context_menu.cpp:1032`）。
2. **BUG-2**：读取侧改为传入真实 topicId。`dialogs_widget.cpp:1198` 至少应在 forum 会话下取该 dialog 的某个规范 topic；更好的做法是让存储层提供「该 dialog 的检查点（忽略 topic）」查询或让调用方传 `std::optional<ID>`。
3. **BUG-3**：删除 `context_menu.cpp:271-273` 的死早退，并把 263-265 的条件并入（即 `( !showFilters && !saveDeletedMessages && !hasCheckpoint )` 提前到读 DB 之前，避免无谓查询）。
4. **BUG-4**：`getCheckpoint` 改用 `std::optional<Checkpoint>`（区分「无行」与「异常」），或新增 `hasCheckpoint` out-param。
5. **R-1**：`setCheckpoint` 改为单条 `INSERT ... ON CONFLICT`（或 `replace`），减少一次 DELETE + 缩短事务；或加节流。
6. **R-4**：删除 `ayu_database.cpp:608` 的 `checkpoint.fakeId = 0;`。

---

## 7. 执行过的命令清单（全部只读）

```
git --no-pager status --porcelain | head -50
git --no-pager rev-parse --abbrev-ref HEAD
git --no-pager diff -- Telegram/SourceFiles/ayu/data/entities.h
git --no-pager diff -- Telegram/SourceFiles/ayu/data/ayu_database.h Telegram/SourceFiles/ayu/data/ayu_database.cpp
git --no-pager diff -- Telegram/SourceFiles/ayu/data/messages_storage.h Telegram/SourceFiles/ayu/data/messages_storage.cpp
git --no-pager diff -- Telegram/SourceFiles/history/history_inner_widget.cpp Telegram/SourceFiles/history/view/history_view_context_menu.cpp Telegram/SourceFiles/dialogs/dialogs_widget.cpp Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp Telegram/SourceFiles/ayu/ui/context_menu/context_menu.h
git --no-pager diff -- Telegram/SourceFiles/history/history_inner_widget.h Telegram/SourceFiles/ayu/ayu_settings.h Telegram/SourceFiles/ayu/ayu_settings.cpp
git submodule status | head -20

# 只读文件读取
read  Telegram/SourceFiles/ayu/data/entities.h
read  Telegram/SourceFiles/ayu/data/ayu_database.cpp
read  Telegram/SourceFiles/ayu/data/ayu_database.h
read  Telegram/SourceFiles/ayu/data/messages_storage.cpp
read  Telegram/SourceFiles/ayu/data/messages_storage.h
read  Telegram/SourceFiles/data/data_msg_id.h
read  Telegram/SourceFiles/history/history_inner_widget.h        (offset 370 / 400)
read  Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp (offset 240, limit 120)

# sqlite_orm 取证
wc -l Telegram/SourceFiles/ayu/libs/sqlite/sqlite_orm.h                                   -> 24841
grep -n "SQLITE_ORM_VERSION\|#define SQLITE_ORM" .../sqlite_orm.h | head -20
grep -n "insert_t" .../sqlite_orm.h | head -40
grep -n "insert_t<O> insert(" -A 45 .../sqlite_orm.h
grep -n "statement_serializer<insert_t<T>, void>" -A 40 .../sqlite_orm.h        (20811-20860)
grep -n "insert_t<T> insert(" -B3 -A6 .../sqlite_orm.h                            (14710-14745)
awk 'NR>=22214 && NR<=23829' .../sqlite_orm.h | grep -n "insert" | head -30
sed -n '22980,23000p' .../sqlite_orm.h                                                (storage_t::insert 成员)
sed -n '22330,22370p' .../sqlite_orm.h                                                (assert_insertable_type)
sed -n '23520,23660p' .../sqlite_orm.h                                                (prepared insert 执行器)
sed -n '20811,20860p' .../sqlite_orm.h
grep -n "void begin_transaction()\|void commit()\|void rollback()" -A 12 .../sqlite_orm.h
grep -n "begin_transaction_internal" -A 20 .../sqlite_orm.h
grep -n "class connection_holder" -A 60 .../sqlite_orm.h
sed -n '13745,13820p' .../sqlite_orm.h
sed -n '23315,23340p' .../sqlite_orm.h                                                (sync_schema 文档)
grep -n "sync_schema" .../sqlite_orm.h
grep -n "last_insert_rowid\|inserted_id" .../sqlite_orm.h | head -40
grep -n "remove_all\|get_all" .../sqlite_orm.h

# 调用点 / 类型取证
grep -n "AyuDatabase::initialize()" -r --include=*.cpp --include=*.h Telegram/SourceFiles/     -> ayu_infra.cpp:48
grep -rn "AyuInfra::init()" --include=*.cpp Telegram/SourceFiles/                            -> core/application.cpp:306
grep -rn "\bCheckpoint\b" --include=*.h --include=*.cpp Telegram/SourceFiles/                 -> 无同名类型冲突
grep -rln "Checkpoint" Telegram/SourceFiles/
grep -n "AyuMessages::getCheckpoint\|checkpoint" Telegram/SourceFiles/dialogs/dialogs_widget.cpp
grep -n "isLocal\|AddCheckpointAction\|setCheckpoint" Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp
grep -n "checkpointItemAt\|setCheckpoint\|saveCheckpointOnDoubleRightClick\|isLocal" Telegram/SourceFiles/history/history_inner_widget.cpp
grep -n "topicRootId\|HistoryMessageTopic \*topic()" Telegram/SourceFiles/history/history_item.h
grep -n "HistoryItem::topicRootId" -A 20 Telegram/SourceFiles/history/history_item.cpp
grep -n "kChatTypeMask" Telegram/SourceFiles/data/data_peer_id.h
grep -n "kGeneralId" Telegram/SourceFiles/data/data_forum_topic.h
grep -n "getDialogIdFromPeer" -A 12 Telegram/SourceFiles/ayu/utils/telegram_helpers.cpp    (164-171)
grep -n "MsgId id = 0\|MsgId id;" Telegram/SourceFiles/history/history_item.h               (:90, :687)
grep -n "isLocal() const" -B2 -A2 Telegram/SourceFiles/history/history_item.h               (:347)
grep -n "data() const" -B4 -A4 Telegram/SourceFiles/history/view/history_view_element.h    (:460)
grep -n "enumerateItems" Telegram/SourceFiles/history/history_inner_widget.h                 (:376-388 签名)
sed -n '770,830p;970,1080p' Telegram/SourceFiles/history/history_inner_widget.cpp
sed -n '1,70p' Telegram/SourceFiles/ayu/ayu_infra.cpp
sed -n '280,320p' Telegram/SourceFiles/core/application.cpp
ls Telegram/SourceFiles/ | head -40 ; ls -a ; ls lib/
find . -maxdepth 4 -type d -name base ; find . -name "not_null*" -not -path "./.git/*"
grep -rn "operator T()" --include=*.h Telegram/
ls -la .clang-tidy ; grep -n "narrowing" .clang-tidy
grep -rn "WX\|Werror\|WARNINGS_AS_ERRORS" cmake/*.cmake CMakeLists.txt
cat .gitmodules
```

**未执行**：任何 `git add/commit/checkout/stash/reset`、任何 `cmake`/`ninja`/编译、任何 `docker`、任何对 `.cpp/.h` 的写入。

**唯一写入**：`docs/review/r2-storage.md`（本文件，任务明确许可）。
