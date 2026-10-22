# AG 检查点 —— 跳转 + 进群默认定位 实现方案设计

> 只做设计，未修改任何 `Telegram/` 下文件；未执行 git / 构建。
> 仓库：`/root/workspace/AyuGramDesktop`（AyuGramDesktop，tdesktop v7.0.9 fork，dev 分支）。
> 约定：所有结论给出 `相对路径:行号` + 摘录；【确定】= 代码直接可读；【无法确定】= 需运行期验证或不在本仓范围。

---

## 0. 结论速览（TL;DR）

| 项 | 结论 | 置信度 |
|---|---|---|
| 推荐拦截点 | **`Telegram/SourceFiles/dialogs/dialogs_widget.cpp:1200` / `:1182-1184` / `:1177-1180` / `:1161-1164` / `:1197`**（会话列表 `chosenRow` 出口），**不在** `MainWidget::showHistory` | 【确定】 |
| 绝不能拦的入口 | `window_session_controller.cpp:848/850/853`（t.me permalink）、`:757`（forum permalink 兜底）、`:783` | 【确定】 |
| 为什么不能在 `MainWidget::showHistory` 拦 | permalink 与会话列表**共用**该函数，入口处 `params.way` 不是可区分判据（两派都有 Forward 和 ClearStack） | 【确定】 |
| 判据 | `showAtMsgId == ShowAtUnreadMsgId`（`MsgId(0)`）**且**来自 `dialogs_widget.cpp` 的 `chosenRow` | 【确定】 |
| 存储读取 | 方案 A（启动预载内存 map）为主推，与 `FiltersCacheController` 既有模式一致 | 【确定】（模式存在）／【无法确定】（性能数据） |
| 失败降级 | `countInitialScrollTop()` `:8224-8227` 已有 `message_not_found` toast + 回落到 unread，无需自研 | 【确定】 |
| 论坛话题定位 | 传 `rootId` 走 `showTopic` → `showRepliesForMessage`，不是 `showPeerHistory` | 【确定】 |

---

## 1. 跳转链核实（逐跳）

### 1.1 `SessionController::showPeerHistory(PeerId, params, msgId)`

`Telegram/SourceFiles/window/window_session_controller.cpp:3136-3149`

```cpp
void SessionController::showPeerHistory(
		PeerId peerId,
		const SectionShow &params,
		MsgId msgId) {
	if (const auto peer = session().data().peerLoaded(peerId)) {
		if (const auto channel = peer->asChannel()) {
			if (channel->isCommunity()) {
				showPeerInfo(channel, SectionShow());
				return;
			}
		}
	}
	content()->showHistory(peerId, params, msgId);
}
```

【确定】`content()` 返回 `not_null<MainWidget*>`（`Telegram/SourceFiles/window/window_session_controller.h:423`），故第 3148 行即进入 `MainWidget::showHistory`。
【确定】社区频道在此被改道为 `showPeerInfo`（`:3143`），不进入历史视图 —— 检查点对该 peer 无效（见 §6.4）。

### 1.2 `MainWidget::showHistory`

`Telegram/SourceFiles/mainwidget.cpp:1386`（声明 `Telegram/SourceFiles/mainwidget.h:204-207`）。关键行：

| 行号 | 摘录 | 说明 |
|---|---|---|
| `mainwidget.cpp:1386-1389` | `void MainWidget::showHistory(PeerId peerId, const SectionShow &params, MsgId showAtMsgId) {` | 入口 |
| `:1390-1394` | `if (peerId && _controller->window().locked()) { ... return; }` | 窗口锁定早退 |
| `:1396-1402` | `if (peer->migrateTo()) { peer = peer->migrateTo(); peerId = peer->id; if (showAtMsgId > 0) { showAtMsgId = -showAtMsgId; } }` | **migrateTo 负 id 反转** |
| `:1403-1412` | `computeUnavailableReason()` → InformBox | 不可用早退 |
| `:1414-1421` | `if ((IsServerMsgId(showAtMsgId) \|\| Data::IsScheduledMsgId(showAtMsgId)) && _mainSection && _mainSection->showMessage(peerId, params, showAtMsgId)) { ... return; }` | section 短路 |
| `:1422-1423` | `} else if (showHistoryInDifferentWindow(peerId, params, showAtMsgId)) { return; }` | 跨窗口改道 |
| `:1431-1438` | `preventsCloseSection([=] { showHistory(peerId, params, showAtMsgId); }, params)` | 未保存确认 |
| `:1440-1447` | `using OriginMessage = SectionShow::OriginMessage; if (const auto origin = std::get_if<OriginMessage>(&params.origin)) { ... _history->pushReplyReturn(returnTo); }` | **OriginMessage 压栈** |
| `:1451-1452` | `using Way = SectionShow::Way; auto way = params.way;` | 栈策略 |
| `:1535` | `_history->showHistory(peerId, showAtMsgId, params);` | 交给 `HistoryWidget` |

### 1.3 `HistoryWidget::showHistory`

`Telegram/SourceFiles/history/history_widget.cpp:2999-3002`

```cpp
void HistoryWidget::showHistory(
		PeerId peerId,
		MsgId showAtMsgId,
		const Window::SectionShow &params) {
```

关键分支：

| 行号 | 摘录 |
|---|---|
| `:3008-3012` | `const auto startBot = (showAtMsgId == ShowAndStartBotMsgId); ... if (startBot \|\| _showAndMaybeSendStart) { showAtMsgId = ShowAtTheEndMsgId; }` |
| `:3018-3019` | `if (_history) { if (_peer->id == peerId) {` —— **同 peer 复用分支** |
| `:3024-3031` | `showAtMsgId == ShowAtUnreadMsgId && insideJumpToEndInsteadOfToUnread()` → 改写为 `ShowAtTheEndMsgId` |
| `:3050-3064` | `const auto canShowNow = _history->isReadyFor(showAtMsgId); if (!canShowNow) { ... delayedShowAt(showAtMsgId, params); }` |
| `:3065-3098` | 已就绪：`setMsgId(showAtMsgId, params)` + `countInitialScrollTop()` + `animatedScrollToY(...)`（`:3090-3095`） |
| `:3195-3197` | `_showAtMsgId = showAtMsgId; _showAtMsgParams = params; _historyInited = false;` |
| `:3273-3277` | `if (_showAtMsgId == ShowAtUnreadMsgId && !_history->trackUnreadMessages() && !hasSavedScroll()) { _showAtMsgId = ShowAtTheEndMsgId; }` |
| `:3293-3302` | `if (_showAtMsgId == ShowAtUnreadMsgId) { if (_history->scrollTopItem) { _showAtMsgId = _history->showAtMsgId; } } else { _history->forgetScrollState(); ... }` |

【确定】**`:3293-3302` 是"检查点默认定位"必须绕开的语义**：`ShowAtUnreadMsgId` 分支保留 `scrollTopItem` 记忆位置，而任何**显式 msgId** 都会走 `else` 分支执行 `_history->forgetScrollState()`（`:3298`）。若把检查点 msgId 填进去，会**破坏"记住上次阅读位置"这一既有行为**——这是设计取舍点（见 §5.1）。

### 1.4 `History::isReadyFor` / `getReadyFor`

`Telegram/SourceFiles/history/history.cpp:2987-3015`（声明 `Telegram/SourceFiles/history/history.h:255-256`）

```cpp
bool History::isReadyFor(MsgId msgId) {
	if (msgId < 0 && -msgId < ServerMaxMsgId && peer->migrateFrom()) {
		// Old group history.
		return owner().history(peer->migrateFrom()->id)->isReadyFor(-msgId);
	}
	if (msgId == ShowAtTheEndMsgId) { return loadedAtBottom(); }
	if (msgId == ShowAtUnreadMsgId) { ... }
	const auto item = owner().message(peer, msgId);
	return item && (item->history() == this) && item->mainView();
}
```

`Telegram/SourceFiles/history/history.cpp:3017-3049`：

```cpp
void History::getReadyFor(MsgId msgId) {
	if (msgId < 0 && -msgId < ServerMaxMsgId && peer->migrateFrom()) { ... return; }
	...
	if (!isReadyFor(msgId)) {
		clear(ClearType::Unload);
		...
		if ((msgId == ShowAtTheEndMsgId)
			|| (msgId == ShowAtUnreadMsgId && !unreadCount())) {
			_loadedAtBottom = true;
		}
	}
}
```

【确定】`isReadyFor` 对显式正 msgId 的语义是**"消息已在本地内存且属于本 history 且有 mainView"**（`:3013-3014`）。检查点 msgId 若不在缓存 → `isReadyFor` 返回 false → 走 `delayedShowAt`（`history_widget.cpp:3058`）→ `messagesReceived` 的 `_delayedShowAtRequest` 分支（`:4722-4757`）→ `getReadyFor` → 服务端补拉 → `setMsgId` → `historyLoaded()`。**这条降级路径与 permalink 跳转完全共用，无需新增网络逻辑。**

### 1.5 `firstLoadMessages` 以 `offsetId=msgId` 拉取

`Telegram/SourceFiles/history/history_widget.cpp:4813-4882`，关键段：

```cpp
4840	} else if (_showAtMsgId > 0) {
4841		_history->getReadyFor(_showAtMsgId);
4842		offset = -loadCount / 2;
4843		offsetId = _showAtMsgId;
4844	} else if (_showAtMsgId < 0 && _history->peer->isChannel()) {
```

```cpp
4863	_firstLoadRequest = histories.sendRequest(history, type, [=](
4864			Fn<void()> finish) {
4865		return history->session().api().request(MTPmessages_GetHistory(
4866			history->peer->input(),
4867			MTP_int(offsetId),
...
4874		)).done([=](const MTPmessages_GetHistory &result) {   // 实为 MTPmessages_Messages
```

【确定】传正 msgId 即以 `offsetId=msgId, offset=-loadCount/2` 居中拉取一页。`delayedShowAt` 走同一逻辑（`:5081-5083`）。**检查点定位复用现成能力。**

### 1.6 `countInitialScrollTop` 显式 msgId 自动高亮 + 滚动

`Telegram/SourceFiles/history/history_widget.cpp:8215-8254`

```cpp
8215	int HistoryWidget::countInitialScrollTop() {
8216		if (hasSavedScroll()) {
8217			return _list->historyScrollTop();
8218		} else if (_showAtMsgId
8219			&& (IsServerMsgId(_showAtMsgId)
8220				|| IsClientMsgId(_showAtMsgId)
8221				|| IsServerMsgId(-_showAtMsgId))) {
8222			const auto item = getItemFromHistoryOrMigrated(_showAtMsgId);
8223			const auto itemTop = _list->itemTop(item);
8224			if (itemTop < 0) {
8225				setMsgId(ShowAtUnreadMsgId);
8226				controller()->showToast(tr::lng_message_not_found(tr::now));
8227				return countInitialScrollTop();
8228			} else {
8229				const auto view = item->mainView();
8230				Assert(view != nullptr);
8231				enqueueMessageHighlight({
8232					item,
8233					base::take(_showAtMsgParams.highlight),
8234				});
8236				const auto result = itemTopForHighlight(view);
8237				createUnreadBarIfBelowVisibleArea(result);
8238				return result;
8239			}
8240		} else if (_showAtMsgId == ShowAtTheEndMsgId) { ... }
```

【确定】这就是"显式 msgId → 高亮 + 居中滚动"的全部实现。**检查点默认定位完全不需要新代码，只需要把正确的 msgId 送到 `showAtMsgId`。**

---

## 2. `showPeerHistory` 全部调用点枚举与语义分类

### 2.1 定义与转发（不产生"进入"语义）

| 位置 | 摘录 | 说明 |
|---|---|---|
| `window_session_controller.h:274-277` | `virtual void showPeerHistory(PeerId peerId, const SectionShow &params = SectionShow::Way::ClearStack, MsgId msgId = ShowAtUnreadMsgId) = 0;` | 纯虚，默认 `Way::ClearStack` |
| `window_session_controller.h:278-285` | `void showPeerHistory(not_null<PeerData*>, ...); void showPeerHistory(not_null<History*>, ...);` | 非虚重载 |
| `window_session_controller.cpp:1406-1411` | `SessionNavigation::showPeerHistory(not_null<PeerData*> peer, ...) { showPeerHistory(peer->id, params, msgId); }` | 转发 |
| `window_session_controller.cpp:1413-1418` | `SessionNavigation::showPeerHistory(not_null<History*> history, ...) { showPeerHistory(history->peer->id, params, msgId); }` | 转发 |
| `window_session_controller.h:542-546` | `using SessionNavigation::showPeerHistory; void showPeerHistory(PeerId, ...) override;` | `SessionController` 覆盖 |
| `info_controller.cpp:315-319` | `void AbstractController::showPeerHistory(...) { return parentController()->showPeerHistory(peerId, params, msgId); }` | 纯转发（`info_controller.h:296`） |
| `window_session_controller.h:287-292` | `clearSectionStack` → `showPeerHistory(PeerId(0), params, ShowAtUnreadMsgId)` | 清栈，peerId=0 |

### 2.2 A 类：**从会话列表（dialogs list）进入** —— 这是"进群默认定位"唯一的合法拦截面

| 位置 | 摘录 | `params.way` | `msgId` |
|---|---|---|---|
| `dialogs_widget.cpp:1182-1184` | `controller()->showPeerHistory(history, Window::SectionShow::Way::ClearStack);` | `ClearStack` | **默认 `ShowAtUnreadMsgId`**（monoforum 无消息行） |
| `dialogs_widget.cpp:1177-1180` | `controller()->showThread(active, ShowAtUnreadMsgId, Window::SectionShow::Way::ClearStack);` | `ClearStack` | `ShowAtUnreadMsgId` |
| `dialogs_widget.cpp:1161-1164` | `controller()->showThread(history, ShowAtUnreadMsgId, Window::SectionShow::Way::ClearStack);` | `ClearStack` | `ShowAtUnreadMsgId` |
| `dialogs_widget.cpp:1083-1086` | `controller()->showThread(topicJump, ShowAtUnreadMsgId, Window::SectionShow::Way::ClearStack);` | `ClearStack` | `ShowAtUnreadMsgId`（topicJump） |
| `dialogs_widget.cpp:1093-1096` | `controller()->showThread(sublistJump, ShowAtUnreadMsgId, Window::SectionShow::Way::ClearStack);` | `ClearStack` | `ShowAtUnreadMsgId`（sublistJump） |
| `dialogs_widget.cpp:1190-1192` | `const auto showAtMsgId = controller()->uniqueChatsInSearchResults(_searchState) ? ShowAtUnreadMsgId : row.message.fullId.msg;` | — | **条件** |
| `dialogs_widget.cpp:1193-1195` | `auto params = Window::SectionShow(Window::SectionShow::Way::ClearStack); params.highlight = Window::SearchHighlightId(_searchState.query);` | `ClearStack` | — |
| `dialogs_widget.cpp:1200` | `controller()->showThread(history, showAtMsgId, params);` | `ClearStack` | **主入口** |
| `dialogs_widget.cpp:1197` | `controller()->showInNewWindow(peer, showAtMsgId);` | — | 新窗口分支 |
| `dialogs_widget.cpp:1104-1106` | `controller()->showInNewWindow(Window::SeparateId(topic), row.message.fullId.msg);` | — | 新窗口 + topic |
| `dialogs_widget.cpp:1109` | `controller()->showThread(topic, row.message.fullId.msg, params);` | `ClearStack` | 搜索命中 topic |
| `dialogs_widget.cpp:5042` | `controller()->showThread(thread);` | **默认 `Way::Forward`** | 默认 `MsgId()` = 0 | 取消聊天内搜索回列表 |
| `dialogs_widget.cpp:1113` | `(row.message.fullId.msg == ShowAtUnreadMsgId)` 判定 avatar 点击 | — | — |

【确定】`chosenRow` 是 `DialogsWidget` 的**唯一**行点击出口：
`Telegram/SourceFiles/dialogs/dialogs_widget.cpp:647-650`
```cpp
_inner->chosenRow(
) | rpl::on_next([=](const ChosenRow &row) {
	chosenRow(row);
}, lifetime());
```
`chosenRow` 定义在 `dialogs_widget.cpp:1035`。另有两处 `chosenRow({...})` 直接构造（`:2226`、`:2248`）。

### 2.3 B 类：**t.me permalink / 消息链接跳转** —— 必须完全绕过

| 位置 | 摘录 | 触发源 |
|---|---|---|
| `core/local_url_handlers.cpp:651-660` | `auto post = ShowAtUnreadMsgId; ... const auto postParam = params.value(u"post"_q); if (const auto postId = postParam.toInt()) { post = postId; }` | URL 解析 |
| `core/local_url_handlers.cpp:690-694` | `controller->window().activate(); controller->showPeerByLink(Window::PeerByLinkInfo{ .usernameOrId = domain, ... .messageId = post, ... });` | 入口 |
| `window_session_controller.cpp:611-620` | `void SessionNavigation::showPeerByLinkResolved(...) { auto params = SectionShow{ SectionShow::Way::Forward }; params.origin = SectionShow::OriginMessage{ info.clickFromMessageId }; params.highlight.pollOption = info.pollOption;` | **所有 permalink 固定 `Way::Forward` + 设置 `origin`** |
| `window_session_controller.cpp:651-659` | `const auto useRequestedMessageId = peer->isChannel(); const auto msgId = useRequestedMessageId ? info.messageId : ...` | msgId 决议 |
| `window_session_controller.cpp:833-855` | `crl::on_main(this, [=] { ... showPeerHistory(peer, params, msgId); });`（`:853`）／新窗口分支 `:848`、`:850` | **普通 permalink 主落点** |
| `window_session_controller.cpp:746-761` | `} else if (peer->isForum()) { ... } else { const auto callback = crl::guard(this, [=] { ... showPeerHistory(peer, params, msgId); }); ...`（`:757`） | **论坛 permalink 兜底** |
| `window_session_controller.cpp:780-791` | `if (!attachBotUsername.isEmpty()) { crl::on_main(this, [=] { ... showPeerHistory(history, params, msgId); ... }); }`（`:783`） | attach bot 链接 |
| `window_session_controller.cpp:621-629` | `if (info.voicechatHash && peer->isChannel()) { crl::on_main(this, [=] { showPeerHistory(peer, params, ShowAtUnreadMsgId); }); ... }`（`:624`） | 语音聊天链接 |
| `window_session_controller.cpp:457-465` | `void SessionNavigation::fullInfoLoadedHook(...) { ... showPeerHistory(open, SectionShow::Way::Forward, ShowAtUnreadMsgId); }`（`:464`） | `?direct` 链接的延迟补全 |
| `window_session_controller.cpp:774-776` | `} else if (const auto monoforum = peer->broadcastMonoforum() ... ) { showPeerHistory(monoforum, params, ShowAtUnreadMsgId); }` | 广播 monoforum 链接 |
| `window_session_controller.cpp:762-769` | `} else if (resolveType == ResolveType::Mention) { if (bot \|\| peer->isChannel()) { crl::on_main(this, [=] { showPeerHistory(peer, params); }); } ... }`（`:765`） | Mention 解析 |

### 2.4 C 类：机器人 / 邀请 / 转发 / 主题 / 日历 / 键盘回复

| 位置 | 摘录 | 语义 |
|---|---|---|
| `mainwidget.cpp:788-791` | `_controller->showPeerHistory(request.peer, SectionShow::Way::Forward, ShowAtTheEndMsgId);` | bot 命令 fallback |
| `mainwidget.cpp:1322-1325` | `_controller->showPeerHistory(peer, SectionShow::Way::Forward, ShowForChooseMessagesMsgId);` | 举报选消息 |
| `mainwidget.cpp:1359-1362` / `:1377-1380` | `window->sessionController()->showPeerHistory(peerId, params, showAtMsgId);` / `primary->sessionController()->showPeerHistory(...)` | **跨窗口改道**（`showHistoryInDifferentWindow`，`mainwidget.cpp:1339-1384`） |
| `mainwidget.cpp:1651-1654` | `_controller->showPeerHistory(item->history(), params, item->id);` | `MainWidget::showMessage`，**带显式 msgId**（通知/搜索点击） |
| `mainwidget.cpp:2138-2141` | `_controller->showPeerHistory(historyItem->peer()->id, params.withWay(SectionShow::Way::Backward), ShowAtUnreadMsgId);` | 后退回栈 |
| `window_session_controller.cpp:2201-2211` | `} else if (!forced && forum->peer()->useSubsectionTabs()) { ... showPeerHistory(forum->peer(), params, showAtMsgId); }`（`:2208`） | **论坛 tab → 历史**（`showForum` 内部） |
| `window_session_controller.cpp:2226-2239` | `showPeerHistory(history, { SectionShow::Way::Backward, anim::type::normal, anim::activation::background });`（`:2233-2237`） | 关闭论坛时回退 |
| `window_session_controller.cpp:2465-2479` | `} else { showPeerHistory(history->peer, params); }`（`:2478`） | `switchInlineQuery` |
| `window_session_controller.cpp:2815-2831` | `showPeerHistory(peer->id, SectionShow(), msgId);`（`:2827`） | `showInNewWindow` 的 `Window::SectionShow()` **默认 way = Forward**（`window_session_controller.h:208`） |
| `window_session_controller.cpp:3024-3045` | `strong->showPeerHistory(peer, SectionShow::Way::Forward, id);`（`:3030-3033`）／`strong->showThread(strongThread, id, SectionShow::Way::Forward);`（`:3035-3038`） | 日历跳转到日期 |
| `window_session_controller.cpp:3125` | `showPeerHistory(peer);`（`finishChatThemeEdit` 内） | 主题编辑后 |
| `history_widget.cpp:2754-2758` | `} else if (const auto monoforum = channel->monoforumLink()) { controller()->showPeerHistory(monoforum, Window::SectionShow::Way::Forward); }` | 邀请预览点击 |
| `history_widget.cpp:6022` | `controller()->showPeerHistory(chat, Window::SectionShow::Way::Forward);` | 跳到讨论群 |
| `history_widget.cpp:6892` | `controller()->showPeerHistory(peer);` | **默认 `SectionShow::Way::ClearStack`**（头文件默认），在 `switchToPeer` 内 |
| `history_widget.cpp:6935-6941` | `auto params = ::Window::SectionShow(::Window::SectionShow::Way::ClearStack); params.highlight = ...; controller()->showPeerHistory(item->history()->peer->id, params, item->fullId().msg);` | **聊天内搜索命中跳转** |
| `history_widget.cpp:8951-8954` | `controller()->showPeerHistory(_peer, Window::SectionShow::Way::Forward, _editMsgId);` | 编辑态退出定位 |
| `history_widget.cpp:8966-8969` | `controller()->showPeerHistory(_kbReplyTo->history()->peer->id, Window::SectionShow::Way::Forward, _kbReplyTo->id);` | 键盘回复定位 |
| `history_widget.cpp:9509-9512` | `controller()->showPeerHistory(item->history()->peer, Window::SectionShow::Way::Forward, item->id);` | **置顶栏点击** |
| `history_widget.cpp:9558-9561` | `controller()->showPeerHistory(session().data().peer(targetId.peer), Window::SectionShow::Way::Forward, targetId.msg);` | 置顶栏下一个 |
| `history_item_helpers.cpp:839` | `controller->showPeerHistory(peer, params, msgId);` | 消息点击 |
| `history_item_components.cpp:139` | `controller->showPeerHistory(...)` | — |
| `history_view_pull_to_next_channel.cpp:1087` | `_controller->showPeerHistory(history, params);` | pull-to-next |
| `history/view/history_view_message.cpp:6081` / `:6109` | `controller->showPeerHistory(...)` | — |
| `history/view/history_view_contact.cpp` | 1 处 | 联系人卡跳转 |
| `history/view/history_view_sticker_toast.cpp` | 1 处 | 贴纸 toast |
| `history/view/history_view_self_forwards_tagger.cpp` | 1 处 | — |
| `history/admin_log/history_admin_log_item.cpp` | 1 处 | 管理日志 |
| `info/profile/info_profile_top_bar.cpp:371`/`:374`/`:944`/`:970`/`:1076` | `controller->showPeerHistory(...)` | 资料页 |
| `info/profile/info_profile_actions.cpp:1518`/`:2274`/`:2645`/`:3239` | `showPeerHistory(...)` | 资料页动作 |
| `info/common_groups/info_common_groups_inner_widget.cpp:184` | `controller->showPeerHistory(...)` | 共同群组 |
| `info/community_requests/info_community_requests_widget.cpp:289` | `navigation->showPeerHistory(...)` | 社区申请 |
| `info/similar_peers/info_similar_peers_widget.cpp:262` | `_controller->parentController()->showPeerHistory(...)` | 相似 peer |
| `info/statistics/info_statistics_widget.cpp:70` | `controller->showPeerHistory(...)` | 统计 |
| `info/bot/starref/info_bot_starref_join_widget.cpp:634` | `_controller->showPeerHistory(bot);` | 星星加入 |
| `inline_bots/bot_attach_web_view.cpp`（3 处） | `showPeerHistory(...)` | 附加 bot webview |
| `inline_bots/inline_results_inner.cpp`（1 处） | — | inline 结果 |
| `boxes/transfer_gift_box.cpp:649`/`:750`/`:893`、`boxes/star_gift_box.cpp`（7 处）、`star_gift_cover_box.cpp`、`star_gift_auction_box.cpp`、`gift_premium_box.cpp:1944` | `showPeerHistory(peer, {}, msgId)` 等 | 礼物/Stars 流程 |
| `boxes/add_contact_box.cpp:112`/`:485`、`boxes/peers/*`（community_box / add_participants_box / edit_contact_box / add_bot_to_chat_box / edit_peer_info_box / edit_discussion_link_box / prepare_short_info_box） | `showPeerHistory(...)` | 设置类流程 |
| `boxes/peer_list_controllers.cpp:795` | `window->showPeerHistory(peer);` | peer 选择器 |
| `api/api_chat_invite.cpp`（2 处）、`api/api_bot.cpp`（2 处） | `showPeerHistory(...)` | API 层跳转 |
| `calls/calls_box_controller.cpp`（2 处）、`calls/group/calls_group_members.cpp` | — | 通话 |
| `media/stories/media_stories_controller.cpp`（2 处） | — | 故事 |
| `core/deep_links/deep_links_settings.cpp` / `deep_links_new.cpp` | — | 应用内深链（**非** t.me permalink） |
| `window/window_main_menu.cpp` / `window_main_menu_helpers.cpp` / `window_peer_menu.cpp`（5 处） | — | 菜单 |
| `window/notifications_manager.cpp` | — | 通知点击 |
| `dialogs/ui/dialogs_suggestions.cpp` / `dialogs_stories_content.cpp` | — | 建议列表 |
| `settings/settings_main.cpp:1231` / `settings_privacy_controllers.cpp:386` | — | 设置 |
| `chat_helpers/share_message_phrase_factory.cpp` | — | 分享 |
| `iv/iv_instance.cpp`（2 处） | — | 官方版本残留 |
| `ayu/ayu_url_handlers.cpp:82`/`:92`、`ayu/ui/context_menu/context_menu.cpp:365-371`/`:427` | AyuGram 自有跳转 | 见 §7 |

### 2.5 关键判据分析

【确定】**`params.way` 不能作为判据**。反例：

- A 类里 `dialogs_widget.cpp:5042` 用默认 `SectionShow()` → `way = Way::Forward`（`window_session_controller.h:208`），**同一入口的两种 way 都有**。
- B 类里 `window_session_controller.cpp:2827`（`showInNewWindow`）也用 `SectionShow()` → `Forward`。
- 反过来，A 类主入口 `dialogs_widget.cpp:1200` 用 `ClearStack`，而 C 类的 `history_widget.cpp:6935-6941`（聊天内搜索）**也**用 `ClearStack`。
- C 类 `history_widget.cpp:6892`（`switchToPeer`）用的是**头文件默认** `SectionShow::Way::ClearStack`。

【确定】`params.origin` 也不能单独作判据：B 类 `showPeerByLinkResolved` 在 `window_session_controller.cpp:617-619` **总是**设置 `OriginMessage{info.clickFromMessageId}`（可能是空 `FullMsgId`），而 A 类从不设置。但 B 类的 `:624`（voicechat）、`:765`（Mention）等分支同样经过 `:614-620`，所以 **"设置了 origin" 确实等价于 "来自链接解析"**——这是一个**可用**的补充判据，不是唯一判据。

【确定】**唯一可靠判据是"调用点"本身**。因此拦截必须放在 A 类调用点或其专属包装里。

---

## 3. 拦截点方案

### 3.1 方案 R（推荐）：在 `DialogsWidget::chosenRow` 出口替换 `showAtMsgId`

**位置**：`Telegram/SourceFiles/dialogs/dialogs_widget.cpp:1190-1202`

```cpp
1188	} else if (history) {
1189		const auto peer = history->peer;
1190		const auto showAtMsgId = controller()->uniqueChatsInSearchResults(
1191			_searchState
1192		) ? ShowAtUnreadMsgId : row.message.fullId.msg;
1193		auto params = Window::SectionShow(
1194			Window::SectionShow::Way::ClearStack);
1195		params.highlight = Window::SearchHighlightId(_searchState.query);
1196		if (row.newWindow) {
1197			controller()->showInNewWindow(peer, showAtMsgId);
1198			closeSuggestions();
1199		} else {
1200			controller()->showThread(history, showAtMsgId, params);
1201			hideChildList();
1202		}
```

**注入逻辑**（伪码，仅设计）：

```cpp
// 在 1189 与 1190 之间
auto showAtMsgId = /* 现有表达式 */;
if (showAtMsgId == ShowAtUnreadMsgId) {                 // 无显式目标 → 允许检查点接管
    if (const auto cp = AyuCheckpoint::lookup(peer->id)) {
        showAtMsgId = cp.msgId;                          // 已按 §3.4 做话题/migrate 前置校验
    }
}
```

**判定条件**（三合一，缺一不可）：

1. 调用点限定在 `dialogs_widget.cpp` 的 `chosenRow` 分支 —— B 类 permalink 物理上到不了这里；
2. `showAtMsgId == ShowAtUnreadMsgId`（`MsgId(0)`，定义于 `data/data_msg_id.h:88`）—— 排除搜索命中跳转（`row.message.fullId.msg` 为真实 id）与 `uniqueChatsInSearchResults` 之外的消息命中；
3. 可选第 3 重：`params.origin` 未被设置（`std::holds_alternative<SectionShow::OriginMessage>(params.origin) == false`），防御性冗余。

**同源需一并处理的兄弟出口**（否则会出现"点普通群能定位、点论坛/monoforum 不能定位"的不一致）：

| 位置 | 处理 |
|---|---|
| `dialogs_widget.cpp:1182-1184` | `showPeerHistory(history, Way::ClearStack)`（默认 msgId = `ShowAtUnreadMsgId`）→ 同样注入 |
| `dialogs_widget.cpp:1177-1180` | `showThread(active, ShowAtUnreadMsgId, Way::ClearStack)` → 若 `active` 是 topic，须走 §7 的 `showTopic` 语义，**不要**用 peer 级 msgId |
| `dialogs_widget.cpp:1161-1164` | 同上（forum `viewForumAsMessages`） |
| `dialogs_widget.cpp:1083-1086` | `showThread(topicJump, ShowAtUnreadMsgId, ...)` → topic 级，§7 |
| `dialogs_widget.cpp:1093-1096` | `showThread(sublistJump, ShowAtUnreadMsgId, ...)` → Saved Sublist，**建议不接管**（检查点模型未覆盖） |
| `dialogs_widget.cpp:1197` | `showInNewWindow(peer, showAtMsgId)` → 新窗口分支也要注入 |
| `dialogs_widget.cpp:5042` | `showThread(thread)` —— 语义是"取消聊天内搜索回到列表"，**不建议接管**（用户意图是回退，不是进入） |

### 3.2 方案 M（不推荐）：在 `MainWidget::showHistory` 入口替换

【确定】**不可行 / 危险**，理由：

- **B 类与 A 类共用此函数**：`window_session_controller.cpp:3148` `content()->showHistory(peerId, params, msgId);` 是**所有** `showPeerHistory` 调用的唯一收敛点，包括 permalink 的 `:853`。
- 若在入口无条件替换，**点 t.me 链接会被劫持到检查点** —— 直接违反硬性要求。
- 若想用参数区分，§2.5 已证明 `params.way` 不可靠。剩下两个候选判据都不可行：
  - `params.origin`：A 类从不为 `OriginMessage`（B 类 `:617-619` 恒设），**看似**可用；但 `MainWidget::showMessage`（`mainwidget.cpp:1651-1654`）转发 notification 点击时也带 `params`（由 `window_session_controller.cpp:597-599` 构造，带 origin），而 B 类的 `:624`（voicechat）、`:765`（Mention）分支**也**走同一 origin 构造——所以 "有 origin ⇒ 来自链接" 成立，但它拦不住 C 类里大量 `Way::Forward` + 显式 msgId 的正常跳转（`history_widget.cpp:9509` 置顶栏、`:8966` 键盘回复等），需要再叠一层 `msgId > 0` 判据，条件组合膨胀且脆弱。
  - `msgId == ShowAtUnreadMsgId`：A 类主出口满足，但 C 类大量**非列表**入口也满足（`history_widget.cpp:6892` `switchToPeer`、`window_session_controller.cpp:3125` 主题编辑、`window_session_controller.cpp:2478` `switchInlineQuery`、`mainwidget.cpp:2138` 后退回栈），会把这些也劫持。

**结论**：`MainWidget::showHistory` 只能作为"方案 R 的兜底断言点"（加 `Expects`/日志），不能作为拦截点。

### 3.3 方案 S（可选加固）：`SessionNavigation` 新增语义化重载

【确定】`SessionController::showPeerHistory` 是 `virtual`（`window_session_controller.h:274-277`），`SessionNavigation` 也有非虚重载（`:278-285`）。可以新增一个语义化入口，例如 `showPeerHistoryAtCheckpoint(peer)`，内部查表后转调既有重载。

- 优点：调用点语义显式，grep 可追踪；不污染通用路径。
- 缺点：需改 `Telegram/` 下多个文件（`window_session_controller.h/.cpp` + `dialogs_widget.cpp`），改动面比方案 R 大。
- 【无法确定】`SessionController::showPeerHistory` 的 `virtual` 有无跨模块 override（`.cpp` 中未见其它 override 实现，`.h` 中仅 `:543` 一处），未做全仓 `-Woverloaded-virtual` 级别验证。

### 3.4 顺序约束（`MainWidget::showHistory` 内的三个关键点）

即便最终不在 `MainWidget::showHistory` 注入，若将来要在 `mainwidget.cpp` 追加兜底，以下顺序约束是【确定】的：

| 约束 | 位置 | 含义 |
|---|---|---|
| **C1：必须在 `migrateTo` 负 id 反转之前注入** | `mainwidget.cpp:1396-1402` | `if (peer->migrateTo()) { ... if (showAtMsgId > 0) { showAtMsgId = -showAtMsgId; } }`。若在 `:1402` 之后注入正 msgId，**不会**再被翻负，语义就错了——检查点记录的是**目标 peer**（迁移后的 supergroup）的 msgId，而历史视图可能挂 `_migrated`（`history_widget.cpp:3072-3076` 期望负 id）。**注入点必须早于或与之对齐**。 |
| **C2：OriginMessage 压栈与 msgId 无关** | `mainwidget.cpp:1440-1447` | `_history->pushReplyReturn(returnTo)` 只依赖 `params.origin->id` 与 `peerId`，不看 `showAtMsgId`。**注入 msgId 不影响压栈**。反过来说，拦截器**不能**用"有没有 origin"去改 `showAtMsgId`（那是 §2.5 的脆弱判据）。 |
| **C3：`_mainSection->showMessage` 短路会吞掉显式 msgId** | `mainwidget.cpp:1414-1421` | 当一个 section（设置/资料/日历…）正开在主栏且 `IsServerMsgId(showAtMsgId)` 时，`showHistory` 会被 `_mainSection->showMessage` 截胡并 `return`，**根本不会**走到 `preventsCloseSection`（`:1433`）和 `_history->showHistory`（`:1535`）。这意味着：**注入正 msgId 会改变"section 是否截胡"的判定**（`IsServerMsgId(0)` 为 false vs `IsServerMsgId(cpId)` 为 true），可能让原本只弹出未保存确认的流程变成直接切 section。这是**在 `MainWidget::showHistory` 注入的额外副作用**，也是方案 R（在上游注入）同样会遇到的副作用 —— 见 §5.2 的缓解。 |
| **C4：`preventsCloseSection` 回调捕获** | `mainwidget.cpp:1433-1436` | `[=] { showHistory(peerId, params, showAtMsgId); }` —— 回调捕获的是**已改写**的 `showAtMsgId`。在 `mainwidget.cpp` 内部注入时，若注入点晚于 `:1433`，用户拒绝"放弃草稿"后重试会**丢失**检查点定位。这是"注入必须尽量靠前"的又一论据。 |

【确定】C1/C4 共同推出：**若必须在 `MainWidget::showHistory` 注入，位置应在 `:1386` 之后、`:1390` 的 `window().locked()` 早退之后、`:1396` 的 `migrateTo` 块之前**（即函数第 5 行附近），并且要在 `alreadyThatPeer`/同 peer 语义上额外判断（见 §5.1）。方案 R 从根本上绕开了 C1–C4。

---

## 4. 数据库异步读取：方案 A vs 方案 B

### 4.1 现状

- AyuGram 已有 SQLite 栈：`Telegram/SourceFiles/ayu/libs/sqlite/sqlite3.c|sqlite3.h|sqlite_orm.h`
- ORM storage 定义：`Telegram/SourceFiles/ayu/data/ayu_database.cpp:14-15`
  ```cpp
  auto storage = make_storage(
      "./tdata/ayudata.db",
  ```
- 初始化：`Telegram/SourceFiles/ayu/data/ayu_database.cpp:227-243`
  ```cpp
  void initialize() {
  	try {
  		storage.sync_schema(true);
  		runMigrations(storage);
  		storage.sync_schema(true);
  	} catch (const std::exception &ex) { ... }
  }
  ```
- 调用时机：`Telegram/SourceFiles/ayu/ayu_infra.cpp:47-49` `void initDatabase() { AyuDatabase::initialize(); }`，由 `Telegram/SourceFiles/ayu/ayu_infra.cpp:70-78` 的 `AyuInfra::init()` 调用，再由 `Telegram/SourceFiles/core/application.cpp:306` `AyuInfra::init();` 触发。

【确定】`AyuInfra::init()` 位于 `Application::start()` 内（`core/application.cpp:306`），**早于**任何窗口/会话创建 → **远早于**任何 `showPeerHistory`。因此"预载"在时间上可行。

### 4.2 既有"预载进内存 map"先例（强烈支持方案 A）

`Telegram/SourceFiles/ayu/features/filters/filters_cache_controller.cpp`：

- `:24-26` `std::mutex rebuildMutex; std::mutex cacheMutex; std::mutex filteredMessagesMutex;`
- `:38` `std::shared_ptr<const Cache> cache;`
- `:43-45` `std::shared_ptr<const Cache> buildCache() { const auto filters = AyuDatabase::getAllRegexFilters(); ... }`
- `:90-100` `void rebuildCache() { std::lock_guard rebuildLock(rebuildMutex); auto next = buildCache(); { ... cache = std::move(next); ... } }`
- `:120-142` `std::shared_ptr<const Cache> snapshot()` —— **双检锁 + 惰性构建**，`snapshot()` 本身是同步阻塞的。

【确定】这就是 AyuGram 现成的"SQLite → 内存快照"模式。检查点缓存应照抄这一形状（`std::shared_ptr<const Cache>` 原子替换 + `std::mutex`）。

### 4.3 方案 A（推荐）：会话启动时预载进内存 map，打开时同步查

**形态**

1. `AyuInfra::initDatabase()`（`ayu_infra.cpp:47-49`）中，`AyuDatabase::initialize()` 之后追加 `AyuCheckpointCache::initialize()`，把 `Checkpoint{peerId, msgId, topicRootId, monoforumPeerId, time}` 全表读入 `std::shared_ptr<const std::unordered_map<PeerId, Checkpoint>>`。
2. 打开 peer 时（§3.1 注入点）调 `AyuCheckpointCache::snapshot()->lookup(peerId)` —— **同步、无 IO、无延迟**。
3. 写入检查点时（另任务）`rebuildCache()` 或定点更新，沿用 `filters_cache_controller.cpp:90-100` 的写法。

**用户可见代价**

| 维度 | 代价 |
|---|---|
| 冷启动 | 多一次 `SELECT`（表规模 = 有检查点的会话数，通常几十~几百行，**<1ms**，**【无法确定】**需实测） |
| 打开会话 | **零**额外延迟（内存 hash 查找） |
| 多账号 | `AyuDatabase` 是**全局单例**（`ayu_database.cpp:14` 文件级 `auto storage`），预载需按 `Session*`/`account` 分桶，**这是方案 A 的主要工程成本** |
| 复杂度 | 需要缓存失效策略（会话被删/登出） |
| 内存 | 可忽略 |

**A 的多账号问题**【确定】：`AyuDatabase` 的所有 API 签名都**不带** session/account 参数（如 `ayu_database.h:26` `getEditedMessages(ID userId, ID dialogId, ...)`），靠 `userId`/`dialogId` 隐式区分。检查点表若沿用此风格，预载时**无法**天然按账号切分，只能全量载入后在内存里按 `account` 过滤，或给新表显式加 `account` 列。**建议在 schema 里显式加 account 列**，而不是靠 dialogId 反查。

### 4.4 方案 B：打开后异步定位

**形态**：先按原样打开（`showAtMsgId` 保持 `ShowAtUnreadMsgId`），在 `MainWidget::showHistory` 成功返回后（或 `HistoryWidget::historyLoaded()` 回调）再查库，命中则 `controller()->showPeerHistory(peer, Way::Forward, cpId)` 或 `historyWidget->jumpTo(...)` 二次跳转。

**用户可见代价**

| 维度 | 代价 |
|---|---|
| 打开会话 | **可见的二次跳动**——先按 unread/上次位置渲染，再跳到检查点。至少一次额外 `MTPmessages_GetHistory`（`history_widget.cpp:4865`），可能 200~800ms（**【无法确定】**需实测） |
| 视觉 | 滚动条"先停后跳"，高亮闪两次 |
| 状态污染 | 二次 `showPeerHistory` 走 `Way::Forward`，会**压栈**（`mainwidget.cpp:1515-1518` `saveSectionInStack(params)`），后退键多一次；且 `history_widget.cpp:3293-3302` 的 `scrollTopItem` 记忆语义会被二次跳转覆盖 |
| 正确性 | 若用户在跳动完成前再次点击，有竞态 |
| 实现 | 需在 `HistoryWidget` 找一个安全的"加载完成"回调点，并加 `crl::guard` 防 UAF |

**结论**：方案 B 的**用户可见代价显著且难消除**，仅在"检查点表极大 / 预载会拖慢冷启动"这一未验证前提下才考虑。**推荐 A。**

### 4.5 混合兜底（若采纳 A 但担心预载失败）

`AyuInfra::init()` 的 `initDatabase()` 已有 `try/catch` + 失败降级先例（`ayu_database.cpp:234-242` 会 `moveCurrentDatabase()` 并重建）。检查点预载应**同样吞异常**：读不到就退化成"无检查点"（等同功能关闭），绝不阻塞启动。

---

## 5. 失败降级

### 5.1 检查点 msgId 被删除 / 不在拉取窗口

【确定】**已有完整降级，无需自研**。链路：

1. `History::isReadyFor`（`history.cpp:3013-3014`）返回 false（消息不在内存）
2. → `history_widget.cpp:3051-3058` `delayedShowAt(showAtMsgId, params)`
3. → `delayedShowAt`（`history_widget.cpp:5042-5100`）以 `offsetId = _delayedShowAtMsgId`（`:5081-5083`）向服务端拉窗口
4. → 拉回后 `messagesReceived` 的 `_delayedShowAtRequest` 分支（`:4722-4757`）：`getReadyFor` → `setMsgId` → `historyLoaded()`
5. → `countInitialScrollTop()`（`:8215`）：`getItemFromHistoryOrMigrated(_showAtMsgId)`（`:8222`，定义在 `:10822-10829`）返回 `nullptr` → `_list->itemTop(nullptr)` 返回 -1
6. → **`:8224-8227`**：
   ```cpp
   if (itemTop < 0) {
   	setMsgId(ShowAtUnreadMsgId);
   	controller()->showToast(tr::lng_message_not_found(tr::now));
   	return countInitialScrollTop();
   }
   ```
   即：**回落 `ShowAtUnreadMsgId`**（重走 `:8242-8253` 的 unread/首条未读分支）+ **弹出 "消息未找到" toast**。

【确定】这与 permalink 跳转到已删消息的行为**完全一致**。

**代价与建议**：

- 用户会看到 "Message not found" toast，即使他只是正常点开聊天。**建议：检查点定位时不传 `highlight`**（§7 已如此设计），并在拦截点自行判断——但由于判定发生在 5 秒后的异步回调里，拦截点**无法**同步感知。两条出路：
  - **(a) 接受**：toast 是诚实反馈，改动最小。
  - **(b) 静默降级**：在 `countInitialScrollTop` 附近加一个 AyuGram 标志（如 `_ayuCheckpointJump`），命中 `:8224` 时跳过 toast、直接回落。代价是**侵入 `history_widget.cpp`**，与"只做设计"无关，但**实现时**需权衡。
- 另一种前置校验思路【确定可行】：在 `HistoryWidget::showHistory` 的 `isReadyFor` 失败分支（`:3051`）不发请求、改回落 —— **不可行**，因为 `isReadyFor` 失败恰恰意味着"要去服务端拉"，正是正常路径。

### 5.2 话题（forum topic）被删除

【确定】检查点若记录了 `topicRootId`，在拦截点应做校验：

- `DialogsWidget::chosenRow` 已有同类先例：`dialogs_widget.cpp:334-344`（`AddJumpToBeginningAction`）检查 `topic->creating()`，以及 `dialogs_widget.cpp:1161-1164` 对 `viewForumAsMessages()` 的处理。
- `SessionNavigation::showRepliesForMessage`（`window_session_controller.cpp:1225-1245`）对未知 rootId 会走 `_showingRepliesRequestId` 的异步补全（`:1246-1250` 起）——【无法确定】该异步路径的最终降级行为（需读 `:1250-1338` 完整实现 + 运行验证），本设计**不依赖**它。

**设计决策**：在拦截点用 `peer->forumTopicFor(rootId)` 校验；话题不存在 → **不注入**，按原样打开（用户无感，功能静默跳过）。校验在打开前完成，无需异步。

### 5.3 peer 被 `migrateTo`（群升级为 supergroup）

【确定】`mainwidget.cpp:1396-1402` 会把 `peerId` 改写为 `peer->migrateTo()->id` 并把正 msgId 翻负。若检查点按**旧 chat peerId** 存储，打开新 supergroup 时查不到 → 静默不注入。

**建议**：schema 同时存 `peerId` 与 `migrateToPeerId`（或在建检查点时归一化到 `migrateTo` 目标），拦截时以 `peer->migrateTo() ? peer->migrateTo()->id : peer->id` 为主键查。

### 5.4 何时清除失效检查点

【确定】可用信号：

| 信号 | 位置 | 建议动作 |
|---|---|---|
| `:8224` `itemTop < 0` 走 not-found | `history_widget.cpp:8224-8227` | 需要侵入才能捕获；**建议**：清理放到**写入侧** + 定期巡检，不依赖此回调 |
| 定期巡检 | — | 用 `Api::requestMessageData`（`window_session_controller.cpp:760` 有先例）验证 msgId 是否存在；**【无法确定】**其对已删消息的返回 |
| 用户手动 | — | 菜单里提供"清除本会话检查点"（§7） |

**推荐**：**不在 `history_widget.cpp` 加清理钩子**（侵入核心路径，收益低）。改为：(1) 用户可见 toast 提示"消息已删除"；(2) 菜单提供手动清除；(3) 写入侧在新消息时更新检查点，自然淘汰。

### 5.5 社区频道

【确定】`window_session_controller.cpp:3140-3147`：社区频道的 `showPeerHistory` 被改道到 `showPeerInfo`，**根本不进历史视图**。拦截点若在 `dialogs_widget.cpp`，会先注入 msgId 再被此处丢弃 —— 行为无害但应显式短路：`peer->asChannel() && peer->asChannel()->isCommunity()` → 不注入（省一次查表）。

---

## 6. 与既有 AyuGram 跳转设施的关系

【确定】仓库中**已存在**一个几乎同形的 AyuGram 跳转实现——"跳到开头"（`ayu_JumpToBeginning`），它就是"检查点跳转菜单项"的模板：

`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:347-408`
```cpp
347	const auto jumpToDate = [=](auto history, auto callback)
348	{
349		const auto weak = base::make_weak(controller);
350		controller->session().api().resolveJumpToDate(
351			history, QDate(2013, 8, 1),
352			[=](not_null<PeerData*> peer, MsgId id)
353			{
354				if (weak.get()) {
356					// API returns 0 if message "Channel created" (ID: 1) was deleted, which scrolls to the bottom
357					if (id.bare == 0) { id = MsgId(2); }
360					callback(peer, id);
361				}
362			});
363	};
365	const auto showPeerHistory = [=](auto peer, MsgId id)
366	{
367		controller->showPeerHistory(
368			peer,
369			Window::SectionShow::Way::Forward,
370			id);
371	};
373	const auto showTopic = [=](auto topic, MsgId id)
374	{
375		controller->showTopic(
376			topic, id, Window::SectionShow::Way::Forward);
379	};
...
391					showPeerHistory(chat, 1);
...
403								showTopic(topic, id);
```

**这份代码同时回答了 §7 的两个问题**：非话题 → `showPeerHistory(peer, Way::Forward, id)`；话题 → `showTopic(topic, id, Way::Forward)`。

挂载点：`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:327-330`（声明 `ayu/ui/context_menu/context_menu.h:29-32`），被 `Telegram/SourceFiles/window/window_peer_menu.cpp:1881`（`Filler::fillHistoryActions`）与 `:1937`（`Filler::fillRepliesActions`）调用。

---

## 7. 『跳到检查点』菜单项实现

### 7.1 参数语义确认（问题 5 的直接答案）

**非论坛**：

```cpp
controller->showPeerHistory(peer, Window::SectionShow::Way::Forward, checkpointMsgId);
```

【确定】`showPeerHistory` 的三个重载签名（`window_session_controller.h:274-285`）：

```cpp
virtual void showPeerHistory(
		PeerId peerId,
		const SectionShow &params = SectionShow::Way::ClearStack,
		MsgId msgId = ShowAtUnreadMsgId) = 0;
void showPeerHistory(
		not_null<PeerData*> peer,
		const SectionShow &params = SectionShow::Way::ClearStack,
		MsgId msgId = ShowAtUnreadMsgId);
```

【确定】"空 SharedElement / params"的正确写法就是**显式传 `Window::SectionShow::Way::Forward`**——因为 `SectionShow::Way` 是 `enum class`（`window_session_controller.h:164-168`），**没有** `SectionShow{}` → way 的隐式转换路径可言（`SectionShow` 的构造函数 `window_session_controller.h:175-188` 接受 `Way`、`anim::type` 等；**没有** `nullptr` / 空 SharedElement 形态）。AyuGram 现有代码（`context_menu.cpp:369`）也是这么写的。

【确定】若想连 `PeerId` 形式都用，直接 `showPeerHistory(peer->id, SectionShow::Way::Forward, msgId)`（`context_menu.cpp:391` 用的就是 `PeerData*` 重载）。

**论坛话题**：

```cpp
controller->showTopic(topic, checkpointMsgId, Window::SectionShow::Way::Forward);
```

【确定】签名 `window_session_controller.h:251-254`：
```cpp
void showTopic(
		not_null<Data::ForumTopic*> topic,
		MsgId itemId = 0,
		const SectionShow &params = SectionShow());
```
【确定】实现 `window_session_controller.cpp:1340-1349` 直接转 `showRepliesForMessage(topic->history(), topic->rootId(), itemId, params)`。
【确定】**`showPeerHistory(peer, ...)` 对论坛消息是错的**：`showPeerByLinkResolved` 在 `window_session_controller.cpp:746-761` 明确区分 `peer->isForum()` 分支（走 `showForum` 或 `showMessageByLinkResolved`），只有非 forum 才在 `:853` 走 `showPeerHistory(peer, params, msgId)`。

**Saved Sublist**：【确定】走 `showSublist(sublist, itemId, params)`（`window_session_controller.h:255-258` / `window_session_controller.cpp:1351-1364`）。**建议检查点模型不覆盖 Saved Sublist**（`dialogs_widget.cpp:1093-1096` 的 sublistJump 出口不做注入）。

### 7.2 菜单项落点

【确定】三处已有 AyuGram 挂载点，全部在 `Telegram/SourceFiles/window/window_peer_menu.cpp`：

| 菜单 | 函数 | 现有 AyuGram 钩子行 | 建议 |
|---|---|---|---|
| 会话历史菜单 | `Filler::fillHistoryActions()` `:1876-1901` | `:1878` `AddAyuGramActions`、`:1881` `AddJumpToBeginningAction` | **紧跟 `:1881`** 加 `AddJumpToCheckpointAction` |
| 话题回复菜单 | `Filler::fillRepliesActions()` `:1933-1945` | `:1934`、`:1937`（仅 `if (_topic)` 时） | **紧跟 `:1937`** |
| 会话列表右键菜单 | `Filler::fillDefaultActions()`（`:1870` 附近） | `:1870` `AddDeleteOwnMessagesAction` | 可选 |

新增函数签名应与 `AddJumpToBeginningAction`（`context_menu.h:29-32`）一致：
```cpp
void AddJumpToCheckpointAction(
		PeerData *peerData,
		Data::Thread *thread,
		not_null<Window::SessionController*> sessionController,
		const Window::PeerMenuCallback &addCallback);
```

【确定】`_peer` / `_thread` / `_controller` / `_addAction` 四个成员在 `Filler` 中均可用（由 `:1870-1937` 的既有用法证明）。

### 7.3 与 `Way` 的关系

【确定】菜单项应传 `Way::Forward`（与 `context_menu.cpp:369`/`:377` 一致），因为这是"在已打开的 chat 里跳一下"，不应清栈。`Way::Forward` 在 `mainwidget.cpp:1525` 触发 `saveSectionInStack(params)`（`:1515-1518` 区块），因此**后退键会回到跳转前的位置** —— 这是期望行为。

### 7.4 高亮

【确定】`params.highlight` 在 `history_widget.cpp:8232-8235` 被 `base::take(_showAtMsgParams.highlight)` 消费，用于 `enqueueMessageHighlight`。**检查点定位不应设置 `highlight`**（避免每次点开聊天都闪一个搜索高亮）。

---

## 8. 未找到项（含命令与范围）

| 项 | 状态 | 命令 / 范围 |
|---|---|---|
| 仓库中是否已存在 "AG 检查点" 功能的代码 | **未找到** | `grep -rni "checkpoint" Telegram/SourceFiles --include=*.cpp --include=*.h -l` → 仅命中 `Telegram/SourceFiles/ayu/libs/sqlite/sqlite3.h`（SQLite 自身的 CHECKPOINT 关键字，与本功能无关） |
| 是否存在检查点专用的存储 / schema / 迁移 | **未找到** | `ls Telegram/SourceFiles/ayu/data/` → 仅 `ayu_database.{h,cpp}`、`entities.h`、`messages_storage.{h,cpp}`；`ayu_database.h:20-60` 的 API 清单中无检查点相关函数 |
| 是否存在检查点专用菜单文案 | **未找到** | `grep -rn "AddJumpToCheckpoint\|ayu_JumpToCheckpoint" Telegram/SourceFiles` 无结果；`context_menu.h:29-32` 仅有 `AddJumpToBeginningAction` |
| codegraph 的 callers/callees 索引 | **本会话不可用** | 已改用 `grep -rn` + 逐点 `sed`/`awk` 上下文展开做等价的全量枚举（`grep -rn "showPeerHistory" Telegram/SourceFiles --include=*.cpp --include=*.h` → 147 处，逐文件按 `-c` 统计后逐个读取上下文） |
| `showPeerByLink` 的完整实现体 | **未完整读取** | 已读 `core/local_url_handlers.cpp:630-700` 与 `window_session_controller.cpp:611-858`；`showPeerByLink` 本身在 `window_session_controller.cpp` 内（`:611` 为 `showPeerByLinkResolved`）。**未**追到 `showPeerByLink` 定义体的完整重试/防抖逻辑 |
| `SectionWidget::showMessage` 的内部行为 | **未读取** | 位置 `mainwidget.cpp:1416` 调用；未展开其实现（只确认了它的存在与短路语义） |
| `History::scrollTopItem` 记忆逻辑与 `_showAtMsgId` 的完整交互 | **部分未读取** | 已读 `history_widget.cpp:3293-3302`；未展开 `Data::Histories` 的 `scrollTopItem` 持久化 |
| `delayedShowAt` 请求返回后的**完整**错误分支 | **部分未读取** | 已读 `history_widget.cpp:5042-5100` 与 `:4722-4757`；未读 `messagesFailed` 内部 |
| 多账号下 `AyuDatabase` 的隔离机制 | **无法确定** | `ayu_database.cpp:14` 是文件级单例 `auto storage`；`ayu_database.h:24-60` 全部 API 无 account 参数。**未见**任何 account 隔离代码（`grep -rn "account" Telegram/SourceFiles/ayu/data/` 无有效结果） |
| 性能数据（预载耗时、跳转延迟） | **无法确定** | 需实测/benchmark；本设计未做（按硬性要求禁构建） |
| `SessionController::showPeerHistory` 是否有跨模块 override | **无法确定** | `grep -n "showPeerHistory" window_session_controller.h` 显示仅 `:543` 一处 `override`；未做全仓继承关系遍历 |
| 历史滚动动画冲突 | **无法确定** | `mainwidget.cpp:1489-1493` `animatedShow()` 的返回依赖 `_history->isHidden()` / `way`；**未**逐一推演"检查点注入后动画时长"的影响 |

---

## 9. 推荐实现路线（总结）

1. **存储**（另任务）：`AyuDatabase` 加 `Checkpoint` 表，**显式带 account 列**；`AyuInfra::initDatabase()`（`ayu_infra.cpp:47-49`）里预载进 `std::shared_ptr<const Cache>`，照抄 `filters_cache_controller.cpp:90-142` 的双检锁形状。
2. **进群默认定位**：只在 `dialogs_widget.cpp:1190`（+ `:1182`、`:1177`、`:1161`、`:1197`）注入，条件为 `showAtMsgId == ShowAtUnreadMsgId` + topic/migrate 前置校验通过 + 非社区频道。
3. **跳转菜单项**：新写 `AyuUi::AddJumpToCheckpointAction`，挂 `window_peer_menu.cpp:1881` 与 `:1937` 之后；非话题传 `showPeerHistory(peer, Way::Forward, msgId)`，话题传 `showTopic(topic, msgId, Way::Forward)`。
4. **不做的**：不动 `MainWidget::showHistory`、不动 `HistoryWidget::showHistory`、不动 `countInitialScrollTop`、不在 `Telegram/` 引入任何新依赖现有 checkpoint 的逻辑分支。
5. **permalink 零影响论证**（见下）。

### 9.1 为什么本方案 100% 不影响 permalink 跳转

三层论证，每层都可静态验证：

1. **物理隔离**：permalink 的收敛路径是 `core/local_url_handlers.cpp:691` `controller->showPeerByLink(...)` → `window_session_controller.cpp:611` `showPeerByLinkResolved(...)` → 最终在 `:853` / `:757` / `:783` / `:848` / `:850` 调 `showPeerHistory(peer, params, msgId)`，其中 `params` 由 `:614-616` 构造为 `SectionShow{ SectionShow::Way::Forward }`。**这条路径从不经过 `dialogs_widget.cpp` 的 `chosenRow`**（`dialogs_widget.cpp:647-650` 是 `DialogsWidget` 的私有行点击流）。
2. **数据隔离**：拦截器只改写 `showAtMsgId`，permalink 的 `msgId` 由 `:651-659` 从 URL 的 `?post=` 解析而来，**在拦截点之外产生**，不会被 `showAtMsgId == ShowAtUnreadMsgId` 判据命中。
3. **判据隔离**：即使将来有人在 `MainWidget::showHistory`（`:1386`）加兜底，也**不应该**加 —— §3.2/§3.4 已论证该位置无法区分 A/B 两类调用（C1 migrateTo、C3 section 短路、C4 回调捕获三重副作用）。本设计明确**不在**该位置注入。

**反例检验**：如果错误地把拦截放在 `MainWidget::showHistory:1386`，用户点 `https://t.me/username/123` → `:853` `showPeerHistory(peer, params{Forward, origin={clickFromMessageId}}, 123)` → 进入 `MainWidget::showHistory(peerId, params, 123)` → 若无条件替换成检查点 msgId，**点链接会跳到检查点而非第 123 条**，且 `:1414` 的 `_mainSection->showMessage` 短路判定会从"不命中"变成"命中"（`IsServerMsgId(0)==false` → `IsServerMsgId(cpId)==true`），行为进一步恶化。**这正是本方案要避免的。**

---

## 10. 执行过的命令清单

```bash
# 目录与文件面
ls /root/workspace/AyuGramDesktop
ls /root/workspace/AyuGramDesktop/docs ; ls /root/workspace/AyuGramDesktop/docs/plan
find Telegram/SourceFiles/ayu -maxdepth 2 -type d | sort
ls -la Telegram/SourceFiles/ayu/libs/sqlite/ Telegram/SourceFiles/ayu/libs/ Telegram/SourceFiles/ayu/data/ Telegram/SourceFiles/ayu/
find Telegram/SourceFiles/ayu/ui -maxdepth 2 -type d   # （并入上面的 find）

# showPeerHistory 全量枚举（等价 codegraph callers）
grep -n "showPeerHistory" -r Telegram/SourceFiles --include=*.cpp --include=*.h | head -80
grep -rn "showPeerHistory" Telegram/SourceFiles --include=*.cpp --include=*.h | wc -l                       # 147
grep -rc "showPeerHistory" Telegram/SourceFiles --include=*.cpp --include=*.h | grep -v ":0" | sort -t: -k2 -rn
grep -n "showPeerHistory\|void SessionController::showThread\|void SessionController::showPeerInfo" Telegram/SourceFiles/window/window_session_controller.cpp
grep -n "showPeerHistory\|void MainWidget::showHistory\|ShowAtUnreadMsgId" Telegram/SourceFiles/window/window_session_controller.h Telegram/SourceFiles/data/data_msg_id.h
grep -n "AddJumpToBeginningAction" Telegram/SourceFiles --include=*.cpp
grep -n "AyuUi::" Telegram/SourceFiles/window/window_peer_menu.cpp
grep -rn "ayu/ui/context_menu/context_menu.h" Telegram/SourceFiles --include=*.cpp
grep -rn "AyuDatabase\|AyuDatabase::" Telegram/SourceFiles --include=*.cpp --include=*.h | grep -v "ayu/data/ayu_database"
grep -rn "FiltersCacheController::init\|AyuInfra::init\b\|AyuInfra::init(" Telegram/SourceFiles --include=*.cpp
grep -rn "uniqueChatsInSearchResults" Telegram/SourceFiles/window/window_session_controller.cpp Telegram/SourceFiles/window/window_session_controller.h
grep -n "content()" Telegram/SourceFiles/window/window_session_controller.h
grep -n "showHistory" Telegram/SourceFiles/mainwidget.h
grep -n -i "ayu" Telegram/SourceFiles/mainwidget.cpp
grep -n "showTopic\|showRepliesForMessage" Telegram/SourceFiles/window/window_session_controller.h
grep -n "void SessionNavigation::showTopic\|void SessionNavigation::showRepliesForMessage" Telegram/SourceFiles/window/window_session_controller.cpp
grep -n "isReadyFor\|getReadyFor\|firstLoadMessages\|countInitialScrollTop" Telegram/SourceFiles/history/history.{cpp,h} Telegram/SourceFiles/history/history_widget.{cpp,h}
grep -n "void HistoryWidget::setMsgId\|void HistoryWidget::delayedShowAt\|void HistoryWidget::clearDelayedShowAt\|_delayedShowAtMsgId" Telegram/SourceFiles/history/history_widget.cpp
grep -n "getItemFromHistoryOrMigrated" Telegram/SourceFiles/history/history_widget.h Telegram/SourceFiles/history/history_widget.cpp
grep -n "IsServerMsgId" Telegram/SourceFiles/data/data_msg_id.h
grep -n "showAtMsgId\|showThread\|message.fullId.msg" Telegram/SourceFiles/dialogs/dialogs_widget.cpp
grep -n "chosenRow\|::activate(\|currentIndex\|enterEvent\|keyEnterPress" Telegram/SourceFiles/dialogs/dialogs_widget.cpp
grep -n "showContextMenu\|ContextMenu" Telegram/SourceFiles/dialogs/dialogs_widget.cpp
grep -rn "checkpoint" Telegram/SourceFiles --include=*.cpp --include=*.h -l
grep -n "AddAyuGramActions\|AddJumpToBeginningAction" Telegram/SourceFiles/ayu/ui/context_menu/context_menu.h
grep -n "void initialize" -A 30 Telegram/SourceFiles/ayu/data/ayu_database.cpp
mkdir -p docs/plan ; ls -ld docs/plan

# 上下文逐点展开（sed -n / awk NR>=..&&NR<=..）
sed -n '3120,3160p'  Telegram/SourceFiles/window/window_session_controller.cpp
sed -n '75,100p'    Telegram/SourceFiles/data/data_msg_id.h
sed -n '1355,1425p' Telegram/SourceFiles/window/window_session_controller.cpp
sed -n '595,640p'   Telegram/SourceFiles/window/window_session_controller.cpp
sed -n '640,700p'   Telegram/SourceFiles/window/window_session_controller.cpp
sed -n '700,870p'   Telegram/SourceFiles/window/window_session_controller.cpp
sed -n '1300,1385p' Telegram/SourceFiles/mainwidget.cpp
sed -n '1380,1545p' Telegram/SourceFiles/mainwidget.cpp
sed -n '1430,1452p' Telegram/SourceFiles/mainwidget.cpp
awk 'NR>=1386 && NR<=1560 {...}' Telegram/SourceFiles/mainwidget.cpp   # 精确定位关键行号
awk 'NR>=3124 && NR<=3150 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/window/window_session_controller.cpp
awk 'NR>=3136 && NR<=3150 {printf "%d\t%s\n", NR, $0}' ...
awk 'NR>=1300 && NR<=1385 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/mainwidget.cpp
awk 'NR>=180,280   {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/dialogs/dialogs_widget.cpp   # chosenRow
awk 'NR>=1030 && NR<=1215 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/dialogs/dialogs_widget.cpp
awk 'NR>=2990 && NR<=3060 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/history/history_widget.cpp
awk 'NR>=3060 && NR<=3200 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/history/history_widget.cpp
awk 'NR>=3200 && NR<=3320 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/history/history_widget.cpp
awk 'NR>=2980 && NR<=3055 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/history/history.cpp
awk 'NR>=4813 && NR<=4900 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/history/history_widget.cpp
awk 'NR>=4700 && NR<=4760 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/history/history_widget.cpp
awk 'NR>=5042 && NR<=5100 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/history/history_widget.cpp
awk 'NR>=6061 && NR<=6100 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/history/history_widget.cpp
awk 'NR>=8200 && NR<=8320 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/history/history_widget.cpp
awk 'NR>=340  && NR<=440  {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp
awk 'NR>=320  && NR<=345  {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp
awk 'NR>=20   && NR<=45   {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/ayu/ui/context_menu/context_menu.h
awk 'NR>=1855 && NR<=1950 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/window/window_peer_menu.cpp
awk 'NR>=1   && NR<=80   {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/ayu/ayu_infra.cpp
awk 'NR>=1   && NR<=90   {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/ayu/data/ayu_database.cpp
awk 'NR>=80  && NR<=175  {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/ayu/features/filters/filters_cache_controller.cpp
awk 'NR>=290 && NR<=320  {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/core/application.cpp
awk 'NR>=630 && NR<=700  {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/core/local_url_handlers.cpp
awk 'NR>=5020&& NR<=5055 {printf "%d\t%s\n", NR, $0}' Telegram/SourceFiles/dialogs/dialogs_widget.cpp
awk 'NR>=2748&& NR<=2770 ...' / awk 'NR>=6015&& NR<=6025 ...' / awk 'NR>=6885&& NR<=6945 ...' / awk 'NR>=8940&& NR<=8975 ...' / awk 'NR>=9500&& NR<=9570 ...'  Telegram/SourceFiles/history/history_widget.cpp
awk 'NR>=780 && NR<=800  ...' / awk 'NR>=1640&& NR<=1660 ...' / awk 'NR>=2125&& NR<=2150 ...' / awk 'NR>=1386&& NR<=1405 ...'  Telegram/SourceFiles/mainwidget.cpp
awk 'NR>=455 ...' / awk 'NR>=2185 ...' / awk 'NR>=2225 ...' / awk 'NR>=2470 ...' / awk 'NR>=2820 ...' / awk 'NR>=3020 ...'  Telegram/SourceFiles/window/window_session_controller.cpp
awk 'NR>=240 && NR<=265 ...' / awk 'NR>=455 && NR<=470 ...' / awk 'NR>=530 && NR<=560 ...' / awk 'NR>=265 && NR<=300 ...' / awk 'NR>=155 && NR<=240 ...'  Telegram/SourceFiles/window/window_session_controller.h
head -60 Telegram/SourceFiles/ayu/ayu_worker.h
```

> 注：`codegraph` 的 `callers`/`callees` 接口在本会话不可用，已用 `grep -rn` 全量枚举 + `awk` 逐点上下文展开替代，覆盖等价范围。
> 全程未执行 `git`、未执行任何构建/测试/docker 命令、未修改 `Telegram/` 下任何文件。

## 主线程抽查（2025-09-27）
- ✅ 链接隔离关键论据实测：`window_session_controller.cpp` 中 `dialogs_widget|DialogsWidget` 引用数 = **0**；permalink 走 `local_url_handlers.cpp:691 → showPeerByLink → showPeerByLinkResolved`（`window_session_controller.cpp:428`），不经过会话列表 → 在列表出口拦截不会劫持链接跳转。
- ⚠️ 勘误：报告称「列表点击用 `Way::Forward`、聊天内搜索用 `ClearStack`」说反了——实测 `dialogs_widget.cpp:1194` 列表点击为 `Way::ClearStack`；不影响其拦截点结论（不使用 way 作判据）。
