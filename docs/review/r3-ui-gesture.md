# R3 对抗式审查：AyuGram 检查点 UI 与手势代码

- **仓库**：`/root/workspace/AyuGramDesktop`（tdesktop v7.0.9 fork）
- **基线**：`v7.0.9`，审查对象为**未提交工作区改动**（`git --no-pager diff`）
- **模式**：只读审查。未修改任何源码，未执行 git 写操作，未构建。仅新建本报告。
- **结论强度**：【确定 BUG】= 代码内已逐行验证；【风险】= 有代码证据但依赖不可见依赖或运行期行为；【无法确定】= 证据不足，明确列出缺口。

---

## 0. 关键环境限制（影响多条结论的可验证性）

**所有 git 子模块目录为空**，包括 `Telegram/lib_base`、`Telegram/lib_ui`、`Telegram/codegen`。

```
$ ls Telegram/lib_base/ | wc -l   → 0
$ git submodule status | head -3
-87f9d768866548b5b86e72be66c60c5abd4d9b37 Telegram/ThirdParty/GSL
-7059649a44c24b640c831adef1e5f5b86e77299f Telegram/ThirdParty/MicroTeX
-82d182a275e197fd717fecc86193d9d91f4fc5b5 Telegram/lib_base     ← lib_base 未 checkout
```

因此 `base::Timer` / `not_null` / `enumerateItems` 的**头文件实现不可读**。相关结论一律降级为【无法确定】或【风险】，并改用仓内既有用法作为间接证据。**本审查未编译**，因此所有"能否编译通过"的判断均为【无法确定】。

---

## 1. 问题清单（按严重程度）

### B1 —— 【确定 BUG / Critical】新增的子菜单早退是死代码，"清除检查点"入口在默认配置下根本不存在

`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:262-273`

```cpp
262	const auto saveDeletedMessages = settings.saveDeletedMessages();
263	if (!showFilters && !saveDeletedMessages) {
264		return;
265	}
266
267	const auto topic = peerData->isForum() && thread ? thread->asTopic() : nullptr;
268	const auto topicId = topic ? topic->rootId().bare : 0;
269	const auto checkpoint = AyuMessages::getCheckpoint(peerData, topicId);
270	const auto hasCheckpoint = checkpoint.messageId > 0;
271	if (!showFilters && !saveDeletedMessages && !hasCheckpoint) {
272		return;
273	}
```

**原因**：既有的早退在 **263-265 行**，新增的守卫在 **271-273 行**，位于其**之后**。能执行到 271 行意味着 `showFilters || saveDeletedMessages` 必为真，此时 271 的条件恒为假 —— 新守卫**逻辑上不可达**。

**后果**：
- 当 `saveDeletedMessages == false` 且过滤器未启用时，函数在 264 行就 return，**"Jump to AG Checkpoint" / "Clear AG Checkpoint"（328-347 行）永远不会出现在 AyuGram 子菜单里**。新增代码想实现的效果（"仅有检查点时也显示 AyuGram 菜单"）**完全未生效**。
- `context_menu.cpp:269` 的同步 SQLite 查询因此变成**每次打开任意 peer 右键菜单都执行、且结果永远不被使用**的纯浪费。
- **与 B3 组合构成死锁式陷阱**：见下。

---

### B2 —— 【确定 BUG / Critical】搜索结果被检查点覆盖，普通账号 100% 触发

`Telegram/SourceFiles/dialogs/dialogs_widget.cpp:1191-1202`

```cpp
1191		const auto inSearchResults = controller()->uniqueChatsInSearchResults(
1192			_searchState
1193		);
1194		auto showAtMsgId = inSearchResults
1195			? ShowAtUnreadMsgId
1196			: row.message.fullId.msg;
1197		if (!inSearchResults) {
1198			const auto checkpoint = AyuMessages::getCheckpoint(peer, 0);
1199			if (checkpoint.messageId > 0) {
1200				showAtMsgId = MsgId(checkpoint.messageId);
1201			}
1202		}
```

`Telegram/SourceFiles/window/window_session_controller.cpp:2046-2055`

```cpp
2046	bool SessionController::uniqueChatsInSearchResults(
2047		const Dialogs::SearchState &state) const {
2048		const auto global = (state.tab == Dialogs::ChatSearchTab::MyMessages)
...
2052		return session().supportMode()
2053			&& !session().settings().supportAllSearchResults()
2054			&& (global || !state.inChat);
2055	}
```

**原因**：该谓词**首先要求 `session().supportMode()` 为真**。对绝大多数普通账号，`supportMode()` 恒为 false → `uniqueChatsInSearchResults()` **恒返回 false**，无论当前是否处于搜索结果列表。

**后果**：普通账号只要该会话存在检查点，**在搜索结果里点中一条消息，会被强制跳到检查点消息**，搜索命中被静默丢弃。任务要求的"搜索结果必须不受影响"**未满足**。

**正确守卫**应是 `_searchState.query.isEmpty()` —— 同一个函数 `dialogs_widget.cpp:1042` 已经在用这个判据：

```cpp
1042	} else if (!_searchState.query.isEmpty()) {
```

---

### B3 —— 【确定 BUG / Critical】检查点永不清除，会话列表被永久钉在旧消息

`Telegram/SourceFiles/dialogs/dialogs_widget.cpp:1197-1212`

```cpp
1197		if (!inSearchResults) {
1198			const auto checkpoint = AyuMessages::getCheckpoint(peer, 0);
1199			if (checkpoint.messageId > 0) {
1200				showAtMsgId = MsgId(checkpoint.messageId);
1201			}
1202		}
...
1206		if (row.newWindow) {
1207			controller()->showInNewWindow(peer, showAtMsgId);
1208			closeSuggestions();
1209		} else {
1210			controller()->showThread(history, showAtMsgId, params);
1211			hideChildList();
1212		}
```

**原因**：覆盖是**无条件**的，既不看用户意图，也不消费检查点。

**后果**：
- 一旦某会话存在检查点，**此后每次从会话列表点入都被拉回旧消息**。用户即使手动滚到最底部读完新消息，下次点列表又被拽回。
- `row.newWindow`（1206-1208）分支**同样受影响** —— 新窗口打开也落在旧检查点。任务点名的 `row.newWindow` 确认**被破坏**。
- 文件夹/归档里的会话行同样走 `row.key.history()` → 该分支（`dialogs_widget.cpp:1213` 的 `row.key.folder()` 只处理文件夹行本身），因此**文件夹内点会话也被覆盖**。
- **逃生口问题（与 B1 组合成陷阱）**：唯一能清除检查点的 UI 是 `context_menu.cpp:339-346` 的 "Clear AG Checkpoint"，而 B1 证明该入口在 `saveDeletedMessages=false` 时不存在。用户可能被永久困在"点会话就被拉回旧消息"的状态，只能手改 `ayugram.db`。

**未受影响的路径（已确认安全）**：
- 话题行 `row.key.topic()`（1100-1111）在 1189 之前 return ✅
- 故事点击（1112-1119）、社区（1120-1122）、社区/论坛/monoforum 特殊分支（1123-1188）均提前 return ✅
- permalink / story 跳转不走 `Widget::chosenRow`，不经过此逻辑 ✅

---

### B4 —— 【确定 BUG / High】500ms 全局右键延迟；菜单在物理右键仍按下时弹出

`Telegram/SourceFiles/history/history_inner_widget.cpp:2844-2884`

```cpp
2844	void HistoryInner::contextMenuEvent(QContextMenuEvent *e) {
2845		if (AyuSettings::getInstance().saveCheckpointOnDoubleRightClick()
2846			&& e->reason() == QContextMenuEvent::Mouse) {
2847			if (_checkpointGestureTimer.isActive()) {
...
2857			const auto item = checkpointItemAt(mapFromGlobal(e->globalPos()).y());
2858			if (item) {
2859				_checkpointGestureItem = item;
2860				_checkpointGesturePos = e->globalPos();
2861				_checkpointGestureTimer.callOnce(kCheckpointGestureInterval);
2862				e->accept();
2863				return;
2864			}
...
2869		showContextMenu(e);
2870	}
2871
2872	void HistoryInner::showDeferredContextMenu() {
...
2879		auto contextMenu = QContextMenuEvent(
2880			QContextMenuEvent::Mouse,
2881			mapFromGlobal(pos),
2882			pos);
2883		showContextMenu(&contextMenu);
2884	}
```

**原因与后果**：
1. **默认开启**（`ayu_settings.h:632` `rpl::variable<bool> _saveCheckpointOnDoubleRightClick = true;`），因此**所有单次右键都被延迟 500ms**，即使用户从未打算用双击功能。这是无条件的全局交互回归。
2. `showDeferredContextMenu` 由**定时器回调**触发，此时用户的物理右键**仍处于按下状态**。`_menu->popup(desiredPosition)`（`history_inner_widget.cpp:4027`）会在按键未释放时弹出菜单。
3. 位置取自 500ms **之前**记录的 `_checkpointGesturePos`（2876），用户在 500ms 内移动鼠标或滚动列表，菜单仍在旧位置弹出。
4. `kCheckpointGestureInterval` 硬编码为 `500`（`history_inner_widget.cpp:150`），未使用 `QApplication::doubleClickInterval()`。仓内同型双击计时器先例是使用系统间隔的：

`Telegram/SourceFiles/history/admin_log/history_admin_log_inner.cpp:2628, 2642`
```cpp
2628		if (_trippleClickTimer.isActive() && (screenPos - _trippleClickPoint).manhattanLength() < QApplication::startDragDistance()) {
2642					_trippleClickTimer.callOnce(QApplication::doubleClickInterval());
```

5. 第二次右键命中时**只写检查点、不弹菜单**（2850-2855）。一个想"右键看菜单"的用户如果连点两下，第二次右键被吞掉，什么也没发生。
6. 第二次右键用的是**第一次**的 item（`_checkpointGestureItem`），第二次的实际命中项被忽略；`_checkpointGesturePos`（2853）被清空，第二次的位置信息完全丢弃。

---

### B5 —— 【确定 BUG / High】`_checkpointGestureItem` 清理位于 `itemRemoved` 的历史过滤早退之后 → 悬垂指针

`Telegram/SourceFiles/history/history_inner_widget.cpp:2531-2583`

```cpp
2531	void HistoryInner::itemRemoved(not_null<const HistoryItem*> item) {
2532		if (_pinnedItem == item) {
2533			_pinnedItem = nullptr;
2534		}
2535		if (_history != item->history() && _migrated != item->history()) {
2536			return;                      // ← 早退
2537		}
...
2580		if (_checkpointGestureItem == item) {      // ← 新增清理，在早退之后
2581			_checkpointGestureTimer.cancel();
2582			_checkpointGestureItem = nullptr;
2583		}
2584		mouseActionUpdate();
2585	}
```

**原因**：
- `itemRemoved` 的数据源是**会话级**事件流，不是本 history 私有：

`Telegram/SourceFiles/data/data_session.h:432`
```cpp
432	[[nodiscard]] rpl::producer<not_null<const HistoryItem*>> itemRemoved() const;
```
`Telegram/SourceFiles/data/data_session.cpp:3233-3236`
```cpp
3233	void Session::unregisterMessage(not_null<HistoryItem*> item) {
3234		const auto peerId = item->history()->peer->id;
3235		const auto itemId = item->id;
3236		_itemRemoved.fire_copy(item);
```

- 因此**全会话任意消息**被销毁都会调用本函数，而 2535-2537 的早退会**跳过**所有按 item 判等的清理。
- 新增的 `_checkpointGestureItem` 清理被放在早退**之后**，而同类成员 `_pinnedItem` 被放在早退**之前**（2532-2534）—— 这是明显的遗漏位置。

**可复现路径**：`_checkpointGestureItem` 来自 `checkpointItemAt` → `enumerateItems`，只会命中 `_history` / `_migrated`（`history_inner_widget.h:379-388`）。但 500ms 窗口内若发生 history 迁移/拆分（`HistoryInner` 会 split history，切换 peer 时 `_history` 被换掉），该 item 就落入"既非 `_history` 也非 `_migrated`"的分支 → 早退 → 指针不置空 → 500ms 后 `history_inner_widget.cpp:2850` 的 `AyuMessages::setCheckpoint(_checkpointGestureItem)` 解引用已释放对象。

**类型比较本身正确**（`HistoryItem*` vs `not_null<const HistoryItem*>`，const 限定转换 + 复合指针类型，合法），与既有 `_scrollDateLastItem->data() == item`（2577）同构 —— **问题纯粹是放置位置**。

---

### B6 —— 【确定 BUG / Medium】论坛话题检查点的"跳转"丢弃 topic

`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:267-268, 328-338`

```cpp
267	const auto topic = peerData->isForum() && thread ? thread->asTopic() : nullptr;
268	const auto topicId = topic ? topic->rootId().bare : 0;
...
333						sessionController->showPeerHistory(
334							peerData,
335							Window::SectionShow::Way::ClearStack,
336							MsgId(checkpoint.messageId));
337					},
```

**原因**：`showPeerHistory(not_null<PeerData*>, ...)`（`Telegram/SourceFiles/window/window_session_controller.h:278-281`）**不接受 topic**。`topicId` 只用于查库，跳转时完全没带上。

**后果**：论坛里针对某个话题设置的检查点，"Jump to AG Checkpoint" 会打开**普通话题**，而不是检查点所在的话题。

**对比**：同函数里"查看已删除消息"分支**正确传了** topic：
```cpp
313						window->showSection(std::make_shared<MessageHistory::SectionMemento>(
314							peerData,
315							nullptr,
316							topicId));
```

---

### B7 —— 【确定 BUG / Medium】`AddCheckpointAction` 捕获裸 `HistoryItem*`，偏离既有安全写法

`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:1031-1044`

```cpp
1031	void AddCheckpointAction(not_null<Ui::PopupMenu*> menu, HistoryItem *item) {
1032		if (item->isLocal()) {
1033			return;
1034		}
1035
1036		menu->addAction(
1037			tr::ayu_SaveCheckpointContextMenuText(tr::now),
1038			[=]
1039			{
1040				AyuMessages::setCheckpoint(item);
1041			},
1042			&st::menuIconInfo);
1043	}
```

**原因**：handler 捕获裸 `item` 并在**点击时**解引用。`AyuMessages::setCheckpoint` 会访问三个成员（`Telegram/SourceFiles/ayu/data/messages_storage.cpp:149-158`）：

```cpp
149	void setCheckpoint(not_null<HistoryItem*> item) {
150		const auto peer = item->history()->peer;
151		const ID userId = peer->session().userId().bare & PeerId::kChatTypeMask;
152		const auto topicId = item->topic() ? item->topicRootId().bare : 0;
```

弹窗打开期间消息可能被删除 / 被 history 裁剪（`unregisterMessage` → 对象释放）→ **UAF**。

**既有同族写法是安全的**（`context_menu.cpp:825, 844-847`）：
```cpp
825		const auto itemId = item->fullId();
...
844				const auto currentItem = history->owner().message(itemId);
845				if (!currentItem) {
846					return;
847				}
```
即**按值捕获 `FullMsgId` 并在使用时重新解析 + 判空**。`AddCheckpointAction` 偏离了这一模式。

（`AddMessageDetailsAction` 同样只捕获按值拷贝的 `messageId` 等标量，见 `context_menu.cpp:608-611`。）

---

### B8 —— 【确定 BUG / Medium】设置开关无任何 UI 入口，500ms 延迟无法关闭

`Telegram/SourceFiles/ayu/ayu_settings.h:359` 声明、`ayu_settings.cpp:541-545` 定义 `setSaveCheckpointOnDoubleRightClick`。

```
$ grep -rn "setSaveCheckpointOnDoubleRightClick\|saveCheckpointOnDoubleRightClick" \
      Telegram/SourceFiles/ | grep -v "ayu_settings.h:\|ayu_settings.cpp:"
Telegram/SourceFiles/history/history_inner_widget.cpp:2845:	if (AyuSettings::getInstance().saveCheckpointOnDoubleRightClick()
```

**唯一命中是 getter 的使用点，setter 零调用者。**

**对比**：`setSaveDeletedMessages` 有 UI 绑定（`Telegram/SourceFiles/ayu/ui/settings/settings_ayu.cpp:645`）：
```cpp
645		.setter = &AyuSettings::setSaveDeletedMessages,
```

**后果**：6 个触点（getter/producers/成员/setter/序列化/反序列化）确实齐全，但**用户没有任何途径关闭这个默认开启的 500ms 延迟**，只能手改 `ayugram.db` 之外的 settings JSON。任务点 5 的"触点齐全"成立，但"可用性闭环"不成立。

---

### B9 —— 【确定 BUG / Low-Medium】手势可对 context menu 永远不会提供的对象设置检查点

`Telegram/SourceFiles/history/history_inner_widget.cpp:2886-2899`

```cpp
2886	HistoryItem *HistoryInner::checkpointItemAt(int y) {
2887		auto result = static_cast<HistoryItem*>(nullptr);
2888		enumerateItems<EnumItemsDirection::TopToBottom>(
2889			[&](not_null<Element*> view, int top, int bottom) {
...
2895				result = view->data();
```

**原因**：`checkpointItemAt` 是**纯几何命中**，不做任何类型过滤。而菜单路径有过滤：
- `addItemActions` 在 `history_inner_widget.cpp:3097-3101` 有 `|| !item->isRegular()` 早退；
- `AddCheckpointAction` 在 `context_menu.cpp:1032-1034` 有 `if (item->isLocal()) return;`。

**后果**：服务消息、相册分片、**本地消息（local）** 都能通过双击右键被写入检查点，但菜单里永远不会为它们提供该选项。两条路径行为不一致，且会写入菜单侧无法查看/清除的脏数据（因为菜单侧不显示）。

**正面结论（已验证正确）**：
- `enumerateItems` 的提前返回语义与既有用法**一致**。`history_inner_widget.h:373-374` 明确规定：
  ```
  // Method has "bool (*Method)(not_null<Element*> view, int itemtop, int itembottom)" signature
  // if it returns false the enumeration stops immediately.
  ```
  新代码 `bottom <= y → return true; top > y → return false;` 与既有用法 `history_inner_widget.cpp:3052-3064` **逐行同构**（同一 `mapFromGlobal(...).y()` 坐标空间、同一对边界判断）。**坐标空间正确，无 off-by-one。**
- `contextMenuEvent` 的守卫含 `e->reason() == QContextMenuEvent::Mouse`（2846），因此**键盘上下文菜单（Keyboard reason）不被延迟** ✅；`showContextMenu` 对 `Keyboard` 的分支（2904-2907）走原路径 ✅。

---

### B10 —— 【确定 BUG / Low】栈上合成 `QContextMenuEvent` + fallthrough 清理（任务点 2 的正面结论与一处状态机不完整）

**栈上事件是安全的**（有先例）：`showDeferredContextMenu`（2879-2882）在栈上构造事件并传指针，与触摸长按路径**完全同构**：

`Telegram/SourceFiles/history/history_inner_widget.cpp:2172-2177`
```cpp
2172					mouseActionFinish(_touchPos, Qt::RightButton);
2173					auto contextMenu = QContextMenuEvent(
2174						QContextMenuEvent::Mouse,
2175						mapFromGlobal(_touchPos),
2176						_touchPos);
2177					showContextMenu(&contextMenu, true);
```
且 `showContextMenu` 只同步消费 `e->globalPos()`（`2903`、`3072`、`4004`）与 `e->reason()`（`2902`、`2904`），不保存指针，无悬垂。

**fallthrough 清理完整**：当 `checkpointItemAt` 返回空时，2865-2867 显式重置了三项状态再 `showContextMenu(e)`，无残留。

**触摸路径未受影响（确认）**：触摸长按走 `QEvent::TouchEnd` → **直接调用 `showContextMenu(&contextMenu, true)`（2177）**，**绕过 `contextMenuEvent`**，因此新增手势逻辑对触摸/长按**完全无影响** ✅。

**状态机不完整（Low）**：
```cpp
2872	void HistoryInner::showDeferredContextMenu() {
2873		if (!_checkpointGestureItem) {
2874			return;                      // ← 早退未清 _checkpointGesturePos
2875		}
```
早退时 `_checkpointGesturePos` 残留旧坐标。实践上无害（下次首击必然覆盖），但属于不完整状态机。

---

### B11 —— 【确定 BUG / Low】每次会话点击 / 每次 peer 菜单打开都做同步 SQLite 访问

`Telegram/SourceFiles/dialogs/dialogs_widget.cpp:1198`（每次点会话）
`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:269`（每次打开 peer 右键菜单）
`Telegram/SourceFiles/history/history_inner_widget.cpp:2850`（每次双击右键，**含写事务**）

`Telegram/SourceFiles/ayu/data/ayu_database.cpp:597-620` 的 `setCheckpoint` 执行 `begin_transaction` + `remove_all` + `insert` + `commit`，全部在 UI 线程同步完成。

**缓解事实**：仓内有先例（`AyuMessages::hasDeletedMessages` 同样在 UI 线程查库），故非全新模式。但 `context_menu.cpp:269` 因为 B1 属于**恒定无效开销**。

**正面结论**：`Checkpoint` 表通过 `make_storage` 的 `make_table`/`make_index` 注册（`ayu_database.cpp:144-156`），且 `storage.sync_schema(true)`（`ayu_database.cpp:242/246/251`）会自动建表，**无需手写迁移** ✅。

---

## 2. 任务七个必查点的逐条回答

| # | 问题 | 结论 | 证据 |
|---|---|---|---|
| 1a | 默认开启后所有单次右键延迟 500ms | **确认回归** | `history_inner_widget.cpp:2844-2869` + `ayu_settings.h:632` |
| 1b | Keyboard reason 是否被误延迟 | **否，未受影响** | `history_inner_widget.cpp:2846` 守卫 + `2904-2907` |
| 1c | 触摸/长按 `showFromTouch` 是否受影响 | **否，完全绕过** | `history_inner_widget.cpp:2177` 直接调 `showContextMenu`，不经 `contextMenuEvent` |
| 1d | `_checkpointGestureItem` 悬垂是否被 `itemRemoved` 覆盖 | **部分覆盖，有洞** | 清理在 `itemRemoved:2580`，但早退在 `2535-2537` → B5 |
| 1e | `enumerateItems` 提前返回语义一致性 | **一致，无问题** | `history_inner_widget.h:373-374` 契约 + 与 `3052-3064` 逐行同构 |
| 2 | 事件 accept 与 fallthrough 清理 | **正确** | 两次 `e->accept()`（2854/2862）+ 完整重置（2865-2867） |
| 3 | `base::Timer` 的 `callOnce`/`cancel`/析构 | **见 R1/R2** | 头文件不可读，用仓内 250+ 处用法佐证 |
| 4 | 会话列表覆盖是否破坏 search/置顶/归档/文件夹/story/permalink/`newWindow` | **search 破坏（B2）、newWindow 破坏（B3）、文件夹内会话破坏（B3）；story/permalink/话题行安全** | `dialogs_widget.cpp:1191-1212` + `2046-2055` |
| 5 | 设置 6 触点 + 老用户 JSON 兼容 | **触点齐全，兼容正确，但无 UI（B8）** | `ayu_settings.h:271,359,450-451,632`；`ayu_settings.cpp:541-545,1088,1193` |
| 6 | lang 键 codegen + 样式常量存在性 | **全部通过** | 见下 |
| 7 | 菜单项捕获裸 `HistoryItem*` 是否安全 | **不安全，偏离既有写法（B7）** | `context_menu.cpp:1038-1041` vs `825/844-847` |

### 任务点 5 补充：6 触点清单与 JSON 兼容性

| 触点 | 位置 | 状态 |
|---|---|---|
| getter | `ayu_settings.h:271` | ✅ |
| producers | `ayu_settings.h:450-451`（Value + Changes） | ✅ 与 `saveDeletedMessagesValue/Changes`（448-449）写法一致 |
| 成员 | `ayu_settings.h:632` `= true` | ✅ 位置紧邻 `_saveDeletedMessages`（631） |
| setter | `ayu_settings.h:359` / `ayu_settings.cpp:541-545` | ✅ 含 `if (current() == val) return;` 早退，与 `setSaveDeletedMessages`（535-539）一致 |
| 序列化 | `ayu_settings.cpp:1088` | ✅ 位置紧邻 `saveDeletedMessages`（1087） |
| 反序列化 | `ayu_settings.cpp:1193` | ✅ `j.value("saveCheckpointOnDoubleRightClick", defaults->...)` |

**老用户兼容**：`j.value(key, defaults->...)` 模式 —— 老 settings JSON 缺该键时回落到 `defaults._saveCheckpointOnDoubleRightClick.current()`。**但是**：`defaults` 从哪来需要确认它不是"同一份已加载的、同样缺键的 JSON"，否则 `defaults` 会是默认构造值。这条是【无法确定】（需读 `from_json` 上文确认 `defaults` 来源），但写法与既有 414 个 `ayu_` 字段完全一致，风险低。

**注意**：反序列化是 `j.value(..., defaults->...)`，而 `defaults` 若是**默认构造的 `AyuSettings`**，则缺键 → `true`，即**老用户升级后自动获得 500ms 延迟且无 UI 可关**（与 B8 叠加）。

### 任务点 6 补充：lang 键与样式常量

**lang 键 —— 全部通过**：
- `Telegram/Resources/langs/lang.strings:8913-8915`
  ```
  8913	"ayu_SaveCheckpointContextMenuText" = "Save AG Checkpoint";
  8914	"ayu_JumpToCheckpointMenuText" = "Jump to AG Checkpoint";
  8915	"ayu_ClearCheckpointMenuText" = "Clear AG Checkpoint";
  ```
- 命名与同块既有关键（`ayu_EditsHistoryMenuText` 等）**完全一致**，`tr::ayu_XXX` 转换规则成立。
- 位置在 `ayu_` 块内（该块共 414 个 `ayu_` 键），插入在 `ayu_ClearDeletedMenuText` 与 `ayu_ViewDeletedMenuText` 之间，**不破坏**该块（该块本身非字母序）。
- **小瑕疵**：AyuGram 无独立语言包（`ls Telegram/Resources/langs/` 只有 `lang.strings` + `*.lproj`），故这 3 个键对所有非英文用户显示英文。属既有机制限制，非本次引入。

**样式常量 —— 三个全部存在**：
| 常量 | 定义位置 | 摘录 |
|---|---|---|
| `st::menuIconShowInChat` | `Telegram/SourceFiles/ui/menu_icons.style:26` | `menuIconShowInChat: icon {{ "menu/show_in_chat", menuIconColor }};` |
| `st::menuIconInfo` | `Telegram/SourceFiles/ui/menu_icons.style:40` | `menuIconInfo: icon {{ "menu/info", menuIconColor }};` |
| `st::menuIconClearAttention` | `Telegram/SourceFiles/ayu/ui/ayu_icons.style:82` | `menuIconClearAttention: icon {{ "menu/clear", menuIconAttentionColor }};` |

（`menuIconClearAttention` 在改动前已被 `context_menu.cpp:324` 使用，生成链已存在。）

---

## 3. 风险（【风险】）

**R1 —— `base::Timer` 语义无法直接验证**
- `base::Timer::callOnce` / `cancel` / `isActive` **存在性：高置信**。仓内 250+ 处 `.callOnce(`，以及 `mtproto/session_private.cpp:1007` `_retryTimer.cancel();`、`:1126` `if (_retryTimer.isActive())` 等大量 `.cancel()`/`.isActive()` 用法。
- **同型先例存在**：`admin_log` 的 `_trippleClickTimer`（`history_admin_log_inner.h:457`）是完全相同形态的"双击窗口计时器"，`isActive()` + `callOnce()` + 几何阈值判断（`2628`）。
- **「已激活时再次 `callOnce`」在本改动中不会发生**：`contextMenuEvent` 先 `isActive()` 判定并 `cancel()`（2847-2848）才走写入分支；定时器触发后 Qt 单次定时器不再 active。**故该语义对本改动不敏感。**
- **无法确定**：定时器回调触发时 `isActive()` 的确切值、析构时的自动注销机制。**未找到** `base/timer.h`（`Telegram/lib_base` 为空目录）。

**R2 —— 析构安全与既有代码同级，不新增风险**
`HistoryInner::~HistoryInner`（`history_inner_widget.cpp:4993-5011`）**不取消任何定时器**（连既有的 `_scrollDateHideTimer` 也不取消）。新增的 `_checkpointGestureTimer([this] { showDeferredContextMenu(); })`（`history_inner_widget.cpp:412`）与既有 `_scrollDateHideTimer([this] { scrollDateHideByTimer(); })`（`411`）**构造形态完全同构**，因此依赖同一套 `base::Timer` 析构保障。**风险等级与既有代码持平**。

**R3 —— `not_null` 隐式转换未编译验证**
`AyuMessages::getCheckpoint(peerData, topicId)`（`context_menu.cpp:269`）传裸 `PeerData*` 给形参 `not_null<PeerData*>`（`messages_storage.h:23`）。既有同型调用 `AyuMessages::clearDeletedMessages(peer, topicId)`（`context_menu.cpp:68`）证明该模式在仓内已存在。**【无法确定】**：未构建，无法确认编译通过。

**R4 —— `Checkpoint` 实体零初始化正确**
`Telegram/SourceFiles/ayu/data/entities.h:139-148` 无默认成员初值，与同文件 `SpyMessageRead`（119-127）、`SpyMessageContentsRead`（129-137）**风格一致** ✅。`ayu_database.cpp:637` `return result.empty() ? Checkpoint() : result.front();` 中 `Checkpoint()` 为值初始化 → 成员归零 → `messageId > 0` 判空安全 ✅。

**R5 —— `MsgId(checkpoint.messageId)` 构造安全**
被 `checkpoint.messageId > 0` 守卫（`context_menu.cpp:270`、`dialogs_widget.cpp:1199`），不会构造非法 `MsgId` ✅。`showPeerHistory(not_null<PeerData*>, const SectionShow&, MsgId)` 签名匹配（`window_session_controller.h:278-281`）✅。

---

## 4. 无法确定（明确列出缺口）

| 项 | 原因 | 执行过的命令 / 范围 |
|---|---|---|
| `base::Timer::callOnce` 在已激活时的行为、`cancel()` 声明、析构语义 | `Telegram/lib_base` 子模块为空 | `ls Telegram/lib_base/base/`；`find / -name timer.h -path "*lib_base*"`；`find . -name "timer.h" -path "*base*"` —— 全部无结果 |
| `not_null<T>` 隐式转换能否编译 | 未构建（禁止） | — |
| `tr::ayu_*` codegen 规则 | `Telegram/codegen` 子模块为空 | `git submodule status` 显示 `-8845d9d45ac754400e19cc2c706bba5a16ef0d5 Telegram/codegen` |
| 整体能否编译通过 | 禁止构建 | — |
| `from_json` 中 `defaults` 的来源（决定老用户缺键时是否真为 `true`） | 需读 `ayu_settings.cpp` 更上文，本次未展开 | 已确认 `ayu_settings.cpp:1193` 使用 `defaults->` 模式 |
| 触摸长按是否会被 Qt 派发为 `reason()==Mouse` 的 `contextMenuEvent`（进而绕过 2177 的直接调用） | 需运行期验证 | 代码层面 tdesktop 已用 `_touchSelectTimer` + `QEvent::TouchEnd` 显式接管，故判定为不受影响；此为代码推断，非运行验证 |

---

## 5. 修复建议

### P0 — 必修（会导致用户功能不可用或数据被静默丢弃）

**P0-1：把新守卫移到旧守卫之前，或合并两者**（B1）

```cpp
// context_menu.cpp:262-273
const auto saveDeletedMessages = settings.saveDeletedMessages();
const auto topic = peerData->isForum() && thread ? thread->asTopic() : nullptr;
const auto topicId = topic ? topic->rootId().bare : 0;
const auto checkpoint = AyuMessages::getCheckpoint(peerData, topicId);
const auto hasCheckpoint = checkpoint.messageId > 0;
if (!showFilters && !saveDeletedMessages && !hasCheckpoint) {
    return;
}
```
即**删除 263-265 的旧早退**，只保留合并后的单一守卫。顺带消除 269 行恒定无效的 DB 查询（可把 `getCheckpoint` 延后到 `!showFilters && !saveDeletedMessages` 时才调用）。

**P0-2：改用查询串作为搜索守卫**（B2）

```cpp
// dialogs_widget.cpp:1191-1202
const auto inSearchResults = !_searchState.query.isEmpty();
auto showAtMsgId = inSearchResults
    ? ShowAtUnreadMsgId
    : row.message.fullId.msg;
if (!inSearchResults) { /* checkpoint 覆盖 */ }
```
与同函数 `dialogs_widget.cpp:1042` 保持一致。`uniqueChatsInSearchResults` 是 support-mode 专用谓词，语义完全不同，不可复用。

**P0-3：给检查点加"消费即失效"或显式豁免**（B3）

最小改动方案（二选一）：
- **方案 a（推荐）**：跳转后清除检查点。`dialogs_widget.cpp:1200` 之后调 `AyuMessages::clearCheckpoint(peer, 0)`，语义变为"检查点是待读提醒"；
- **方案 b**：仅当 `row.message.fullId.msg != ShowAtUnreadMsgId` 且无未读时才覆盖 —— 仍不解决"永久钉住"。

**方案 a 才能真正解除 B1+B3 组合出的死局**，且最符合"用户想看新消息却被拉到旧检查点"这一可用性诉求。

**P0-4：`_checkpointGestureItem` 清理前移到历史过滤早退之前**（B5）

```cpp
// history_inner_widget.cpp:2531
void HistoryInner::itemRemoved(not_null<const HistoryItem*> item) {
    if (_checkpointGestureItem == item) {
        _checkpointGestureTimer.cancel();
        _checkpointGestureItem = nullptr;
        _checkpointGesturePos = QPoint();
    }
    if (_pinnedItem == item) {
        _pinnedItem = nullptr;
    }
    if (_history != item->history() && _migrated != item->history()) {
        return;
    }
    ...
```
与 `_pinnedItem`（2532-2534）保持同构。

**P0-5：`AddCheckpointAction` 改为按值捕获 `FullMsgId` 并在使用时解析**（B7）

```cpp
// context_menu.cpp:1031
void AddCheckpointAction(not_null<Ui::PopupMenu*> menu, HistoryItem *item) {
    if (item->isLocal()) {
        return;
    }
    const auto itemId = item->fullId();
    const auto history = item->history();
    menu->addAction(
        tr::ayu_SaveCheckpointContextMenuText(tr::now),
        [=] {
            if (const auto currentItem = history->owner().message(itemId)) {
                AyuMessages::setCheckpoint(currentItem);
            }
        },
        &st::menuIconInfo);
}
```
与 `AddRepeatMessageAction`（`context_menu.cpp:825, 844-847`）完全一致。

### P1 —— 应修

**P1-1：接入设置 UI**（B8）。在 `Telegram/SourceFiles/ayu/ui/settings/settings_ayu.cpp` 中参照 `settings_ayu.cpp:645` 的 `.setter = &AyuSettings::setSaveDeletedMessages` 增加一个 toggle，否则 500ms 延迟不可关闭。

**P1-2：间隔改用系统双击间隔，并加几何阈值**（B4）

```cpp
// history_inner_widget.cpp:150
constexpr auto kCheckpointGestureInterval = 500;
```
建议改为 `QApplication::doubleClickInterval()`，并像 `admin_log:2628` 那样加 `manhattanLength() < QApplication::startDragDistance()` 阈值——否则在两处不同消息上快速右键会误触发检查点写入。

**P1-3：话题检查点跳转带上 topic**（B6）。改用 `sessionController->showPeerHistory(peerData, ...)` 的同时，对论坛场景改调 `showThread(topic, MsgId(...), params)`，或在 `AyuMessages::setCheckpoint` 时对 forum 一律归一化到 `topicId = 0`。

**P1-4：给 `checkpointItemAt` 加类型过滤，与菜单路径对齐**（B9）

```cpp
// history_inner_widget.cpp:2895
if (auto data = view->data(); data && data->isRegular() && !data->isLocal()) {
    result = data;
}
return false;
```

**P1-5：第二次右键改为"仍然弹菜单"**。当前第二次右键在 2850-2855 写完检查点就 `return`，不弹菜单。建议写检查点后**继续** `showContextMenu(e)`（用第二次的 `e`），这样双击既存检查点又有菜单，不破坏基本预期。

### P2 —— 可选

**P2-1**：`showDeferredContextMenu`（2873-2875）早退时补 `_checkpointGesturePos = QPoint();`。
**P2-2**：把 `AyuMessages::getCheckpoint/setCheckpoint` 的 DB 操作移出 UI 线程（参考 `api_read_metrics.cpp` 的 `crl::async` 模式）。
**P2-3**：`showDeferredContextMenu` 在弹出菜单前检测右键是否仍按下，若已释放则考虑改为在 `mouseReleaseEvent` 中触发。

---

## 6. 跑过的命令清单

```bash
# 变更面
git --no-pager status --short
git --no-pager diff --stat
git --no-pager diff -- <5 个目标文件>
git --no-pager diff -- Telegram/SourceFiles/ayu/data/ Telegram/SourceFiles/history/view/history_view_context_menu.cpp \
                      Telegram/SourceFiles/history/history_item.cpp Telegram/SourceFiles/data/

# 子模块可用性（决定 base::Timer / not_null / codegen 能否验证）
ls -la Telegram/lib_base/
ls Telegram/lib_base/ | wc -l
find . -name "timer.h" -path "*base*"
find / -name "timer.h" -path "*lib_base*"
find . -type d -name "not_null"
ls Telegram/lib_{ui,rpl,storage}/ | wc -l
git submodule status
ls -la .codegraph/

# 符号定位
grep -rn "enumerateItems<EnumItemsDirection::TopToBottom>"      Telegram/SourceFiles
grep -rn "\.callOnce("                                          Telegram/SourceFiles
grep -rn "Timer\.cancel()\|Timer\.isActive()\|timer\.isActive()" Telegram/SourceFiles
grep -rn "setSaveCheckpointOnDoubleRightClick\|saveCheckpointOnDoubleRightClick" Telegram/SourceFiles
grep -rn "setSaveDeletedMessages"                               Telegram/SourceFiles
grep -rn "uniqueChatsInSearchResults"                           Telegram/SourceFiles
grep -rn "AddCheckpointAction"                                  Telegram/SourceFiles
grep -rn "AyuMessages::getCheckpoint\|AyuMessages::setCheckpoint\|AyuMessages::clearCheckpoint" Telegram/SourceFiles
grep -rn "itemRemoved"                                          Telegram/SourceFiles/data/data_session.{h,cpp}
grep -rn "menuIconShowInChat\|menuIconInfo\|menuIconClearAttention" Telegram/Resources Telegram/SourceFiles --include=*.style
grep -n "showPeerHistory"                                       Telegram/SourceFiles/window/window_session_controller.h
grep -n "showContextMenu("                                      Telegram/SourceFiles/history/history_inner_widget.{h,cpp}
grep -n "_menu->popup\|_menu->empty()"                           Telegram/SourceFiles/history/history_inner_widget.cpp
grep -c "^\"ayu_"                                               Telegram/Resources/langs/lang.strings

# 精读（read 工具）
history_inner_widget.cpp   : 460-489, 978-1037, 1165-1185, 2125-2189, 2531-2585, 2842-2911,
                             2911-3020, 3025-3099, 3099-3213, 3390-3450, 3560-3720, 3720-3900,
                             3995-4039, 4993-5020
history_admin_log_inner.cpp: 405-444（经 sed）, 2075-2100, 2630-2660
history_inner_widget.h     : 370-392, 440, 455, 710-716
context_menu.cpp           : 55-80, 245-359, 598-700, 820-870, 1028-1045
dialogs_widget.cpp         : 1036-1230
ayu_settings.h / .cpp      : diff 全文 + 触点行
data_session.cpp           : 3220-3245
entities.h                 : 100-149
window_session_controller.{h,cpp}: 270-300, 2046-2060
history_item.h             : 313, 347, 582
history_view_context_menu.cpp : 1110-1145
menu_icons.style / ayu_icons.style
lang.strings               : 8895-8925
```

**未执行**（遵守只读约束）：任何 `git add/commit/checkout/stash`、任何 cmake/构建/测试/代码生成、任何对源码的写操作。唯一写入为本报告 `docs/review/r3-ui-gesture.md`。

---

## 7. 总体评价

这批改动在**局部工程质量上是合格的**——`checkpointItemAt` 的 `enumerateItems` 用法与坐标空间经逐行比对与既有 `history_inner_widget.cpp:3052-3064` 完全一致，是本审查中唯一"零缺陷"的部分；`MsgId` 构造有 `> 0` 守卫；`Checkpoint` 实体风格与同文件既有实体一致；`Checkpoint` 表靠 `sync_schema(true)` 自动建表，无遗漏迁移；栈上合成 `QContextMenuEvent` 有 `2172-2177` 的同构先例；`cancel()`/`isActive()` 的用法有 `admin_log` 双击计时器这一近乎同型的仓内先例支撑。lang 键命名与三个样式常量全部验证存在。

但**在系统层面是不合格的**，问题集中在三处：

1. **改动只加了"写"，没加"读的对称性"**。检查点有写入口（手势 + 菜单）、有读入口（会话列表覆盖），却没有"消费即失效"，也没有在任何受影响的分支上提供可达的清除入口——而那个清除入口因为 `context_menu.cpp:263-265` 的旧早退被证明**逻辑不可达**。这直接构成一个用户无解的状态。

2. **守卫写在了错误的位置**。`context_menu.cpp:271` 的新早退在 `263` 之后因而成为死代码；`itemRemoved` 的新清理在 `2535` 的早退之后因而留下悬垂洞。两处都是同一个模式：**没有先读懂既有守卫的控制流就插入新逻辑**。B2 同理——`uniqueChatsInSearchResults` 是一个 support-mode 专用谓词，被误用来表达"是否处于搜索态"。

3. **默认开启 + 无 UI 关闭 + 全局副作用**。`ayu_settings.h:632` 默认 `true`，`setSaveCheckpointOnDoubleRightClick` 零调用者，而它影响的是**全应用每一个右键菜单的响应时机**。这违反了"影响面广的默认行为变更必须可撤销"这一基本约束。

**Verdict：Request Changes。** B1/B2/B3/B5 应在合入前修复（其中 B1+B3 组合是功能性死局，B2 是静默数据丢弃，B5 是内存安全）。B6/B7/B8/B9 建议同批修复。B4 的 500ms 延迟本身是产品决策，但需要 P1-1 提供的关闭开关与 P1-2/P1-5 的误触防护才算可接受。

在 `base::Timer` / `not_null` / codegen 三处不可验证的背景下，**本报告不对"能否编译通过"作任何断言**；R1/R2/R3 及第 4 节所列各项请在 submodule 完整的环境下复核。
