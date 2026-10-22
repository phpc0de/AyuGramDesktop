# T2：Checkpoint 孤儿行清理钩子 —— 调查 + 落地

- 仓库：`/root/workspace/AyuGramDesktop`（AyuGramDesktop = tdesktop v7.0.9 fork，分支 `dev`，HEAD `db3b9891cb`）
- 任务性质：**静态核验**（无 cmake/ninja/clang/docker、子模块未初始化、无 `out/`）→ 未构建、未编译、未 commit
- 落地改动：仅 `Telegram/SourceFiles/ayu/data/ayu_database.{h,cpp}`（新增 2 个清理 API）
- 未落地（需改非 `ayu/` 核心文件，按硬性约束只输出精确补丁）：2 个钩子调用点

---

## 0. 结论先行

**推荐方案：「两者」——主 peer/历史级 + 辅 账号级。**

理由（每条都有下节证据）：

1. 账号级单独不够：用户最常触发的孤儿来源是「删除/清空单个聊天」，此时 session 不结束，账号级钩子永不触发。`History::clear()` 已经是这条路径的唯一汇聚点，且**已有现成 rpl 通知** `Session::historyCleared()`（`data/data_session.cpp:2379`），挂载成本最低。
2. peer 级单独不够：`Main::Session::finishLogout()`（`main/main_session.cpp:348`）走的是 `Data::Session::clear()` → `Histories::clearAll()`（`data/data_histories.cpp:178`）的 `_map.clear()`，**不触发任何 history 级通知**；注销后整账号行全留。而 `finishLogout` 只在 `DestroyReason::LoggedOut` 分支被调用（`main/main_account.cpp:213-215`），退出程序不走这条，天然安全。
3. 两处钩子都很便宜：DB 层只要 2 个 `ID` 级函数（已落地），调用点各 1 行。

**Cursor（Cursor 级/会话级）不可行**：`ayu/` 目录内**不存在**任何 per-session、per-history 的初始化或订阅点（证据见 §3.3），故 2 个钩子都必须挂在核心文件里 → 本次只出补丁文本，未改核心。

---

## 1. 取证 A：`DeletedMessage` 表当前在什么时机被清理

| 动作 | 实现位置 | 触发者（file:line） |
|---|---|---|
| 写入 | `ayu/data/ayu_database.cpp:303` `addDeletedMessage` ← `ayu/data/messages_storage.cpp:114` `AyuMessages::addDeletedMessage` | `ayu/utils/telegram_helpers.cpp:735`（在 `processMessageDelete`，`telegram_helpers.cpp:727`）→ 上游 `item->setDeleted()` 路径 |
| 逐行删除 | `ayu/data/ayu_database.cpp:372` `removeDeletedMessage` ← `ayu/data/messages_storage.cpp:137` | **`data/data_histories.cpp:998`**（`Histories::deleteMessages` 内：`if (item->isDeleted()) AyuMessages::removeDeletedMessage(item);`） |
| 按会话批量删除 | `ayu/data/ayu_database.cpp:386` `clearDeletedMessages`（`topicId==0` 即整个 dialog） ← `ayu/data/messages_storage.cpp:143` | **仅 1 处且是手工人肉菜单**：`ayu/ui/context_menu/context_menu.cpp:68`（`ClearDeletedMessagesHandler`，`context_menu.cpp:54`，确认框后执行） |
| 清空整表 | **不存在**（`grep -rn "remove_all<DeletedMessage>" Telegram/SourceFiles/` 只命中 `ayu_database.cpp:374` 与 `:388` 两处，均为带 `where` 的单行/单会话删除） | — |

结论：

- `DeletedMessage` **没有自动清理先例**。`remove_all<DeletedMessage>` 的两处调用都被 `where` 收窄到 `(userId, dialogId, messageId)` 或 `(userId, dialogId, topicId)`；账号注销、删除聊天历史时**不会**清空该表。
- 唯一"自动"的一半是 `data_histories.cpp:998`：只在用户**显式删除一批消息**、且该 item 处于 `isDeleted()` 状态时，删掉对应那一行 —— 粒度是"消息行"，不是"会话/账号"。
- 所以本任务的 Checkpoint 清理**会引入一条新行为**（`DeletedMessage` 没有对等物）。若主线程后续想要对齐，钩子位置与本报告 §6 完全相同（把函数名换成 `clearDeletedMessages`），但那会**删掉用户刻意保留的已删消息历史**，属产品决策，本次不做。

---

## 2. 取证 B：History 销毁 / 历史删除路径

### 2.1 唯一汇聚点：`History::clear()`

`Telegram/SourceFiles/history/history.cpp:4371` `void History::clear(ClearType type, bool markEmpty)`：

| 行 | 内容 | 语义 |
|---|---|---|
| `history.cpp:4381` | `if (type == ClearType::Unload)` | 仅 lazy 卸载分支 |
| `history.cpp:4379` | `owner().notifyHistoryUnloaded(this)` | **两条分支都会fire**（在 if 之前） |
| `history.cpp:4399` | `owner().notifyHistoryCleared(this)` | **只在 `else` 分支 fire** = `DeleteChat` / `ClearHistory`，`Unload` 不 fire |
| `history.cpp:4403` | `if (type == ClearType::DeleteChat)` | 只影响 lastServerMessage 处理 |

`Telegram/SourceFiles/data/data_session.cpp:2375-2381`：

```cpp
void Session::notifyHistoryCleared(not_null<const History*> history) {
	_historyCleared.fire_copy(history);
}

rpl::producer<not_null<const History*>> Session::historyCleared() const {
	return _historyCleared.events();
}
```

`notifyHistoryCleared` 全仓库**只有 1 个调用点**：`history/history.cpp:4399`。

### 2.2 `History::clear()` 的全部调用点（`grep -rn "clear(ClearType"` / `"History::ClearType"`）

| file:line | 类型 | 场景 |
|---|---|---|
| `history/history.cpp:2872` | `Unload` | 滚动/缓存回收 |
| `history/history.cpp:3022` | `Unload` | 同上 |
| `history/history.cpp:3030` | `Unload` | 同上 |
| `history/history.cpp:3038` | `Unload` | 同上 |
| `history/history.cpp:3041` | `Unload`（migrated） | 同上 |
| `history/history.cpp:3509` | `Unload, true` | 同上 |
| `data/data_session.cpp:1644-1646` | 频道 → `Unload`，非频道 → `DeleteChat` | `Session::deleteConversationLocally`（定义 `data_session.cpp:1623`） |
| `apiwrap.cpp:2293` | `ClearHistory` | `ApiWrap::deleteHistory(justClear=true)` |

`Session::deleteConversationLocally` 的唯一调用者：`apiwrap.cpp:2291`（`ApiWrap::deleteHistory`，定义 `apiwrap.cpp:2237`；入口 `apiwrap.cpp:2217` = 清空历史，`apiwrap.cpp:2228/2230/2233` = 删除聊天）。

**由此得出 peer/历史级覆盖矩阵：**

| 用户操作 | 走到哪 | `historyCleared()` 是否 fire |
|---|---|---|
| 清空聊天历史（非频道） | `apiwrap.cpp:2217` → `2293` `ClearHistory` | ✅ |
| 清空聊天历史（频道/群） | `apiwrap.cpp:2217` → `2293` `ClearHistory` | ✅ |
| 删除聊天（用户/普通群/secret） | `apiwrap.cpp:2228` → `deleteConversationLocally` → `1646` `DeleteChat` | ✅ |
| 删除聊天（频道/超级群） | `apiwrap.cpp:2228` → `1645` `Unload` + `2276/2284` `deleteAllMessages` | ❌ **缺口** |
| lazy 卸载（滚走/内存回收） | `history.cpp:2872/3022/3030/3038/3509` | ❌（正确，不该清） |

⚠️ 关键负向结论：**绝不能订阅 `historyUnloaded()`**（`data_session.cpp:2367`）。它在 `history.cpp:4379` 无条件 fire，lazy 卸载也会触发，会把用户检查点在滚动时随机清空。已有订阅者正是 `historyCleared()` + `historyUnloaded()` 的组合用法，可对照 `history/history_inner_widget.cpp:487-493` 与 `data/data_search_controller.cpp:464-470`（后者按 `history->peer->id == peerId` 过滤，是本仓库既有范式）。

### 2.3 `Histories::deleteAllMessages`

`data/data_histories.cpp:835` —— 只批量销毁 item，**不 fire** `historyCleared`（其内部走 `data_histories.cpp:998` 的 `removeDeletedMessage`）。所以频道"删除聊天"这条缺口既不会清 DeletedMessage 也不会清 Checkpoint。

---

## 3. 取证 C：账号会话结束 / 注销路径

### 3.1 调用链（全部取到行）

| file:line | 内容 |
|---|---|
| `core/application.cpp:954` | `Application::logout(Main::Account*)` → `account->logOut()` |
| `core/application.cpp:962` | `Application::logoutWithChecks`（上传/下载/导出守卫后仍进 `logout`） |
| `main/main_account.cpp:520` | `Account::logOut()`（`_loggingOut` 去重） → `_mtp->logout([=]{ loggedOut(); })` |
| `main/main_account.cpp:544` | `Account::loggedOut()` → `destroySession(DestroyReason::LoggedOut)`（`:547`） |
| `main/main_account.cpp:203` | `Account::destroySession(reason)`；`:213-215` **只在 `LoggedOut` 分支**调 `_session->finishLogout()`；`:216` `_session = nullptr` |
| `main/main_session.cpp:348` | `Session::finishLogout()`：`:350` `data().clear()`、`:351` `data().clearLocalStorage()` |
| `main/main_session.cpp:354` | `Session::~Session()`：`:356` `data().clear()` |
| `data/data_session.cpp:422` | `Data::Session::clear()`：`:484` `_histories->clearAll()` |
| `data/data_histories.cpp:178` | `Histories::clearAll()` = `_map.clear()`（**无通知、无 per-history 回调**） |
| `data/data_session.cpp:5887` | `Session::clearLocalStorage()`（只关/清 cache，不碰 ayudata.db） |

另有 `main/main_domain.cpp:123` 批量 `account->logOut()`，仍收敛到上面的 `finishLogout` 链。

### 3.2 判定

- 账号级清理点的**唯一正确位置**是 `main/main_session.cpp:348-352`（`finishLogout`）：此处 `Main::Session` 仍存活，`userId()` 可用（`main/main_session.h:136`），且该函数**仅**在 `DestroyReason::LoggedOut`（`main_account.cpp:213`）被调用 —— 退出程序（`main_account.cpp:59` `DestroyReason::Quitting`）不会触发，不会误删。
- `Session::~Session()`（`main_session.cpp:354`）不合适：退出程序也会走，且 userId 处于"即将失效"语义。
- `Data::Session::clear()`（`data_session.cpp:422`）不合适：同样被 Quitting 复用，且没有 `userId` 上下文以外的判据。

### 3.3 `ayu/` 侧有没有可安全挂载点？——**没有**

已查范围与命令：

```
grep -rln "Main::Session" Telegram/SourceFiles/ayu/            # 命中 20 文件，全部按需传入，无 session 级生命周期所有者
grep -n "rpl::" Telegram/SourceFiles/ayu/**          # 全部是局部 lifetime（message_shot / ayu_sync / telegram_helpers）
grep -rn "ayu/" Telegram/SourceFiles/data/data_session.cpp  # :95-98 已有 4 个 ayu 头，但只有函数级调用，无订阅
grep -n "init" Telegram/SourceFiles/ayu/ayu_infra.cpp        # AyuInfra::init() 只做语言/DB/UI 设置/worker/RC/翻译，无 session 参数
```

- `AyuInfra::init()`（`ayu/ayu_infra.cpp:70`，由 `core/application.cpp:306` 调用）是**应用级**、发生在任何账号 session 之前 → 拿不到 session，无法订阅 `historyCleared()`。
- `AyuState`（`ayu/ayu_state.h:16-25`）只有 `hide/isHidden/disableGhostModeOnStoryClose`，是纯静态函数，**无 rpl lifetime 宿主**、无 session 订阅先例 → 不是可挂载点。
- `AyuWorker`（`ayu/ayu_worker.h:11-14`）虽是应用级常驻，但它是 3 秒轮询（`ayu_worker.cpp:36` `callEach(3000)`），且其 `.cpp/.h` 同样不在本次允许改动清单内；用轮询做 GC 属不确定行为，不推荐。
- 已有 ayu 侧"被核心调用"的先例都是**函数级埋点**（如 `ayu/utils/telegram_helpers.cpp:727` `processMessageDelete` 被 `history/history_item.cpp` 调），即：ayu 想参与清理，**必然要在核心文件里加 1 行**，这正是本次只出补丁的原因。

**因此：「`ayu/` 目录内、且明确安全的自带通知钩子」不存在 → 按硬性约束 5，不改核心，只给补丁文本。**

---

## 4. 已落地 diff（`ayu/data/ayu_database.{h,cpp}`）

新增 2 个纯 `ID` 级 API，零调用点、零行为变化（不接线就不执行），签名与既有 `clearCheckpoint`/`clearDeletedMessages` 完全同风格。

`Telegram/SourceFiles/ayu/data/ayu_database.h`（在 `:38` 之后插入，现为 `:39-40`）：

```cpp
void clearCheckpoint(ID userId, ID dialogId, ID topicId);
void clearCheckpointsForDialog(ID userId, ID dialogId);
void clearAllCheckpoints(ID userId);
```

`Telegram/SourceFiles/ayu/data/ayu_database.cpp`（新增于 `clearCheckpoint` 之后，现为 `:673-697`）：

```cpp
void clearCheckpointsForDialog(ID userId, ID dialogId) {
	try {
		storage.remove_all<Checkpoint>(
			where(
				column<Checkpoint>(&Checkpoint::userId) == userId and
				column<Checkpoint>(&Checkpoint::dialogId) == dialogId
			)
		);
	} catch (std::exception &ex) {
		LOG(("Failed to remove dialog checkpoints for some reason: %1").arg(ex.what()));
	}
}

void clearAllCheckpoints(ID userId) {
	try {
		storage.remove_all<Checkpoint>(
			where(
				column<Checkpoint>(&Checkpoint::userId) == userId
			)
		);
	} catch (std::exception &ex) {
		LOG(("Failed to remove all checkpoints for some reason: %1").arg(ex.what()));
	}
}
```

静态核验要点：

- `storage.remove_all<T>(where(...))` 形态与 `ayu_database.cpp:388`（`clearDeletedMessages`）、`:600`（`setCheckpoint`）逐字同构 → sqlite_orm 用法已被本文件既有代码证明可用。
- `dialogId` 覆盖全部 topic（不传 `topicId`），语义为"该会话所有话题的检查点"，与 §2 的"peer 级"对齐。
- 风格：无注释、无 `Q_OS_LINUX`、未引入 `QString`（`_q` 规则不涉及）。
- 未新增 include、未改 `make_storage`/schema → **不影响 `ayudata.db` 结构，不触发迁移**。

---

## 5. 候选钩子清单（每个的 file:line + 落地状态）

| # | 候选点 | file:line | 覆盖面 | 是否已落地代码 |
|---|---|---|---|---|
| 1 | `Session::notifyHistoryCleared` 内联清理（peer/历史级，**推荐主钩子**） | `data/data_session.cpp:2375`（函数体 `:2376`） | 清空历史（全部 peer）+ 删除聊天（非频道） | ❌ **仅补丁文本**（§6.1） |
| 2 | `Session::finishLogout` 内清理（账号级，**推荐辅钩子**） | `main/main_session.cpp:348`（函数体 `:349-352`） | 账号注销/被踢/忘记密码 | ❌ **仅补丁文本**（§6.2） |
| 3 | `History::clear()` 内 `history.cpp:4399` 旁直接清理 | `history/history.cpp:4399` | 同 #1，但会**漏掉**任何未来不发通知的 `ClearType`；且历史层无 `userId` 手边可得 | ❌ 不推荐 |
| 4 | 订阅 `historyUnloaded()` | `data/data_session.cpp:2367` | 会误伤 lazy 卸载 | ❌ **禁止**（§2.2 负向结论） |
| 5 | `Session::~Session()` 内清理 | `main/main_session.cpp:354` | 含退出程序 | ❌ 不推荐（退出也触发） |
| 6 | `Data::Session::clear()` 内清理 | `data/data_session.cpp:422` | 含退出程序 | ❌ 不推荐（同上） |
| 7 | `ayu/ayu_state.*` / `ayu/ayu_infra.*` / `ayu/ayu_worker.*` 挂载 | `ayu/ayu_state.h:16`、`ayu/ayu_infra.cpp:70`、`ayu/ayu_worker.cpp:36` | 无 session 级通知宿主 / 轮询不可靠 | ❌ **不可行**（§3.3） |
| 8 | DB 层 2 个 API | `ayu/data/ayu_database.h:39-40`、`ayu/data/ayu_database.cpp:673-697` | 供 #1/#2 调用 | ✅ **已落地** |

---

## 6. 主线程下一步要动的精确位置（补丁文本）

### 6.1 peer/历史级（必做）

**文件**：`Telegram/SourceFiles/data/data_session.cpp`

(a) 第 `96` 行后（现有 `#include "ayu/data/messages_storage.h"` 之后）加一行：

```cpp
#include "ayu/data/ayu_database.h"
```

(b) 把 `:2375-2377` 改成：

```cpp
void Session::notifyHistoryCleared(not_null<const History*> history) {
	_historyCleared.fire_copy(history);
	AyuDatabase::clearCheckpointsForDialog(
		session().userId().bare & PeerId::kChatTypeMask,
		getDialogIdFromPeer(history->peer));
}
```

依据与前提（全部已核验）：

- `session().userId()`：`data/data_session.h:179` 提供 `Main::Session &session() const`，`Main::Session::userId()` 见 `main/main_session.h:136`。
- `PeerId::kChatTypeMask`：`data/data_peer_id.h:110`；掩码写法与写入侧 `ayu/data/messages_storage.cpp:33`（`item->history()->owner().session().userId().bare & PeerId::kChatTypeMask`）逐字一致 —— **必须**保持同样的掩码，否则与 `Checkpoint::userId` 存的值对不上，删不掉。
- `getDialogIdFromPeer`：声明 `ayu/utils/telegram_helpers.h:59`，而 `data_session.cpp:98` 已 `#include "ayu/utils/telegram_helpers.h"` → 无需新增 include。
- `History::peer` 是 `const not_null<PeerData*>`（`history/history.h:483`），函数参数是 `not_null<const History*>` → `history->peer` 可直接传 `getDialogIdFromPeer(not_null<PeerData*>)`。

**风险**：

1. `data_session.cpp:2375` 是热路径（每次 `History::clear` 都会走）。只在内联执行一条 SQL `DELETE ... WHERE userId=? AND dialogId=?`，命中 `idx_checkpoint_userId_dialogId_topicId` 前缀 → 可接受；但**若主线程想更保守**，可改为 `crl::on_main` 延后一拍（`Data::Session` 是 `rpl::lifetime` 宿主，注意不要在 session 销毁后调用）。
2. 若 `PeerId` 掩码常量在 `data_session.cpp` 未直接可见（该文件只显式包含了 `data/data_peer_values.h:83`，`kChatTypeMask` 走 `history.h → data_peer.h` 间接链），编译期报"未声明"时补 `#include "data/data_peer_id.h"`。**本次无法编译验证**。
3. 覆盖缺口保留：频道/超级群"删除聊天"走 `ClearType::Unload`（`data_session.cpp:1645`）不发通知 → 该场景仍留孤儿（见 §7）。

### 6.2 账号级（建议同批做）

**文件**：`Telegram/SourceFiles/main/main_session.cpp`

(a) 第 `69` 行后（现有 `#include "ayu/ayu_settings.h"` 之后）加一行：

```cpp
#include "ayu/data/ayu_database.h"
```

(b) 把 `:348-352` 改成：

```cpp
void Session::finishLogout() {
	unlockTerms();
	AyuDatabase::clearAllCheckpoints(userId().bare & PeerId::kChatTypeMask);
	data().clear();
	data().clearLocalStorage();
}
```

**依据与前提**：

- `finishLogout` 仅在 `DestroyReason::LoggedOut` 触发（`main/main_account.cpp:213-215`），`Quitting`（`main_account.cpp:59`）不触发 → 退出程序不误删。
- 调用点必须在 `data().clear()` **之前或之后都安全**（`Checkpoint` 只依赖 `userId`，不依赖 `Data::Session` 存活状态）；放在 `data().clear()` 之前可避免与 `clearLocalStorage()`（`data_session.cpp:5887`，只清 cache）语义混在一起。
- 与 6.1 不重叠：注销时 `Histories::clearAll()`（`data_histories.cpp:178`）不发通知，所以 6.1 抓不到；两者互补。
- `PeerId::kChatTypeMask` 若不可见，补 `#include "data/data_peer_id.h"`。**本次无法编译验证**。

### 6.3 可选的更"分层"写法（若主线程愿意多改 1 个文件）

把 `& PeerId::kChatTypeMask` 这类细节从核心文件里赶走，在 `ayu/data/messages_storage.{h,cpp}` 加两个 peer 级包装（与 `messages_storage.cpp:143/175` 同构）：

```cpp
void clearCheckpointsForDialog(not_null<PeerData*> peer);
void clearAllCheckpoints(Main::Session *session);
```

然后 6.1 / 6.2 各自只剩一行调用。代价：动到 `ayu/data/messages_storage.*`（超出本次允许改动的文件范围，故未落地）。

---

## 7. 剩余风险 / 未修缺口（明确列出）

| # | 风险 / 缺口 | 证据 / 影响 |
|---|---|---|
| R1 | 频道/超级群"删除聊天"不发 `historyCleared` | `data/data_session.cpp:1645` 用 `ClearType::Unload`；缺口仍产生孤儿行。可选补丁：在 `apiwrap.cpp:2291` 之前（`ApiWrap::deleteHistory` 的 `if (!justClear)` 分支）直接调 `clearCheckpointsForDialog` |
| R2 | 论坛单个话题被删除时无钩子 | 已查 `data/data_forum_topic.cpp:258`（仅析构）/ `data/data_session.h` 无 topic-deleted 通知 → 话题级孤儿行只能等 peer 级或账号级兜底 |
| R3 | 撤销/普通"逐条删除消息"不清理 | 检查点指向的 `messageId` 被单条删除后仍留行，`getLatestCheckpoint` 返回值会让"进群默认定位"跳到不存在的消息。现状已有手工人肉逃生口（`ayu/ui/context_menu/context_menu.cpp:346` "Clear AG Checkpoint"）→ 与 `DeletedMessage` 现状一致，本次不扩 |
| R4 | `DeletedMessage` 同样有孤儿 | §1 全表证据：无账号级/会话级自动清理；若主线程要对齐，钩子位置与 §6 相同（换函数名），但会删掉用户刻意保留的已删消息 → 产品决策 |
| R5 | 无编译/无运行验证 | 环境零构建能力（子模块 37/37 未初始化）。`remove_all<T>(where(...))`、`session().userId()`、`PeerId::kChatTypeMask` 可见性均为静态核验；首个编译错误块需回填 |
| R6 | 多账号共用单文件库 | `storage = make_storage("./tdata/ayudata.db")`（`ayu/data/ayu_database.cpp:14-15`）是**全局单库**，所有行靠 `userId` 区分 → 两处钩子都必须带 userId 过滤，绝不可 `remove_all<Checkpoint>()` 全清（会清掉其他账号） |
| R7 | 【无法确定】`historyCleared` 是否覆盖"服务端把我从群里踢出/对方删除会话" | 已查 `grep -rn "notifyHistoryCleared" Telegram/SourceFiles/` 只 1 处（`history.cpp:4399`）；"被踢"路径未追到 `History::clear(DeleteChat)`，需补查 `applyUpdates` 里 `messageActionChatDeleteUser`/`ChannelData::applyEdit` 分支 |

---

## 8. 复现命令（本次全部取证均可重跑）

```bash
cd /root/workspace/AyuGramDesktop
grep -rn "remove_all<DeletedMessage>\|remove_all<Checkpoint>" Telegram/SourceFiles/
grep -rn "AyuMessages::clearDeletedMessages\|AyuMessages::removeDeletedMessage\|AyuMessages::clearCheckpoint" Telegram/SourceFiles/ --include=*.cpp
grep -rn "notifyHistoryCleared\|historyCleared()" Telegram/SourceFiles/ --include=*.cpp --include=*.h
grep -rn "ClearType::DeleteChat\|clear(ClearType" Telegram/SourceFiles/ --include=*.cpp --include=*.h
grep -rn "deleteConversationLocally\|finishLogout\|destroySession" Telegram/SourceFiles/ --include=*.cpp --include=*.h
git diff Telegram/SourceFiles/ayu/data/ayu_database.h Telegram/SourceFiles/ayu/data/ayu_database.cpp
```
