# AG 检查点 —— 保存入口（右键菜单项 + 双击右键）实现方案设计

> 只做设计，未修改任何 `Telegram/` 下文件；未执行 git / 构建 / docker。
> 仓库：`/root/workspace/AyuGramDesktop`（AyuGramDesktop，tdesktop v7.0.9 fork，dev 分支）。
> 姊妹文档：`docs/plan/p1-checkpoint-navigation.md`（跳转 + 进群默认定位），本篇负责**写入侧入口**，术语与之一致（`AyuCheckpoint` / `AddJumpToCheckpointAction` / schema 字段 `peerId, msgId, topicRootId, monoforumPeerId, time`）。
> 约定：每条结论给 `相对路径:行号` + 摘录；【确定】= 代码直接可读；【无法确定】= 需运行期验证或不在本 checkout 范围。

---

## 0. 结论速览（TL;DR）

| 项 | 结论 | 置信度 |
|---|---|---|
| 插入点 A（主聊天历史菜单） | `Telegram/SourceFiles/history/history_inner_widget.cpp:3132` 之后（`AyuUi::AddMessageDetailsAction(_menu, item);` 下一行），即 `addItemActions` lambda 尾部 | 【确定】 |
| 插入点 B（其它消息列表菜单） | `Telegram/SourceFiles/history/view/history_view_context_menu.cpp:1134` 之后（`AyuUi::AddMessageDetailsAction(menu, request.item);` 下一行），即 `AddMessageActions` 的 `if (request.item)` 块尾部 | 【确定】 |
| 新动作完整链路 | `context_menu.h:48` 声明 → `context_menu.cpp:784-902` 实现 → `settings_chats.cpp:347` 模式挂 UI → `ayu_settings.{h,cpp}` 10 处开关 → `lang.strings:9013` 文案 → `ayu_icons.style:22` 图标 | 【确定】 |
| 回调取 id | `item->fullId()` 得 `FullMsgId{PeerId peer; MsgId msg;}`（`data_msg_id.h:146-164`）；`item->topicRootId()` 得 `MsgId` topic；`item->history()->peer` 得 `not_null<PeerData*>`（`history.h:483`） | 【确定】 |
| 开关模式 | **nlohmann::json 独立文件 `tdata/ayu_settings.json`**（`ayu_settings.cpp:31-33`），三态枚举 `ContextMenuVisibility`（`ayu_settings.h:35-39`），**不是** `QDataStream`，无追加顺序陷阱 | 【确定】 |
| 双击右键（弹出菜单后再识别第二次） | **不可行/不可靠**：菜单在第一次右键的事件处理内同步 `popup()`（`history_inner_widget.cpp:3965`），且 `mousePressEvent:2269-2272` 在 `_menu` 存在时主动忽略按下；第二次右键能否到达 `HistoryInner` 取决于 Qt 平台派发与 PopupMenu 的鼠标抓取，而 `popup_menu.{h,cpp}` / `layer_widget.h` 所在的 **`Telegram/lib_ui` 子模块在本 checkout 为空目录** | 【无法确定】→ 结论按不可靠处理 |
| 双击右键（推荐路径） | **延迟弹出（deferred）**：第一次右键只记录 + 起 `base::Timer`（窗口取 `QApplication::doubleClickInterval()` 或 `crl::time(500)`）**不弹菜单**；窗口内的第二次右键取消定时器并执行保存；定时器到期才用**合成 `QContextMenuEvent`**（先例 `:2169-2173`）重放原 `showContextMenu` | 【确定】可行，见 §4 |
| 代价 | 开启后所有右键菜单有固定延迟（建议 ~500ms；用独立开关控制，建议默认关） | 【确定】 |
| 退化方案 | 仅菜单项（零风险，推荐默认）／菜单项 + 键盘快捷键（`HistoryInner::keyPressEvent:4214-4433`）／app 级事件过滤器（先例 `core/application.cpp:383`+`:783-787`、`mainwidget.cpp:403`+`:2852-2863`） | 【确定】 |

---

## 1. 两处菜单构建器的插入锚点

### 1.1 插入点 A —— `HistoryInner::addItemActions`（仅主聊天历史）

**位置**：`Telegram/SourceFiles/history/history_inner_widget.cpp:3033`（lambda 定义）→ 尾部 `:3126-3133`：

```cpp
3126		}
3127
3128		AyuUi::AddHistoryAction(_menu, item);
3129		AyuUi::AddHideMessageAction(_menu, item);
3130		AyuUi::AddUserMessagesAction(_menu, item);
3131		AyuUi::AddRepeatMessageAction(_menu, item, HistoryView::Context::History);
3132		AyuUi::AddMessageDetailsAction(_menu, item);
3133	};
```

**唯一可识别的相邻代码行**：`Telegram/SourceFiles/history/history_inner_widget.cpp:3132`
`		AyuUi::AddMessageDetailsAction(_menu, item);`
→ **新动作插在 `:3132` 之后、`:3133` 的 `};` 之前**。

**函数边界与早退条件**（影响动作可用性判定）：

| 行号 | 摘录 | 说明 |
|---|---|---|
| `:3033-3035` | `const auto addItemActions = [&](HistoryItem *item, HistoryItem *albumPartItem) {` | 入参：`item`（= `_dragStateItem`）、`albumPartItem` |
| `:3036-3041` | `if (!item \|\| !item->isRegular() \|\| isUponSelected == 2 \|\| isUponSelected == -2) { return; }` | **只对 regular 消息生效**；完全选中（±2）时整块不构建 |
| `:3116-3125` | `if (canViewMessageStats(item)) { ... &st::menuIconStats` | 统计项，位于 Ayu 钩子之前 |
| `:3128-3132` | 五个 `AyuUi::Add*Action` | Ayu 钩子簇 |

**调用点**（决定该菜单在哪些场景出现）：

| 行号 | 摘录 | 场景 |
|---|---|---|
| `:3434` | `const auto item = _dragStateItem;` | 菜单目标 item 就是 `_dragStateItem`（`history_inner_widget.h:655 HistoryItem *_dragStateItem = nullptr;`） |
| `:3461` | `addItemActions(item, item);` | 普通消息 |
| `:3632` | `addItemActions(item, editItem);` | 可编辑分支 |
| `:3636` | `addItemActions(item, albumPartItem);` | 相册（album） |

【确定】**A 点只覆盖主聊天历史**（`HistoryInner`），回复视图 / 置顶列表 / Ayu 消息历史视图走的是 B 点。

### 1.2 插入点 B —— `HistoryView::AddMessageActions`（所有 `ListWidget` 派生消息列表）

**位置**：`Telegram/SourceFiles/history/view/history_view_context_menu.cpp:1124`（函数定义）→ Ayu 钩子簇 `:1128-1135`：

```cpp
1124	void AddMessageActions(
1125			not_null<Ui::PopupMenu*> menu,
1126			const ContextMenuRequest &request,
1127			not_null<ListWidget*> list) {
1128		if (request.item) {
1129			const auto context = request.view ? request.view->context() : Context::History;
1130			AyuUi::AddHistoryAction(menu, request.item);
1131			AyuUi::AddHideMessageAction(menu, request.item);
1132			AyuUi::AddUserMessagesAction(menu, request.item);
1133			AyuUi::AddRepeatMessageAction(menu, request.item, context);
1134			AyuUi::AddMessageDetailsAction(menu, request.item);
1135		}
```

**唯一可识别的相邻代码行**：`Telegram/SourceFiles/history/view/history_view_context_menu.cpp:1134`
`		AyuUi::AddMessageDetailsAction(menu, request.item);`
→ **新动作插在 `:1134` 之后、`:1135` 的 `}` 之前**。

**上溯链**（谁调 B）：

| 行号 | 摘录 | 说明 |
|---|---|---|
| `history_view_context_menu.cpp:1636` | `AddMessageActions(result, request, list);` | `FillContextMenuItems` 内唯一调用点 |
| `history_view_context_menu.cpp:1488-1492` | `void FillContextMenuItems(not_null<Ui::PopupMenu*> result, not_null<ListWidget*> list, const ContextMenuRequest &request, bool skipWhoReacted = false) {` | 题述的第二个构建器 |
| `history_view_context_menu.cpp:1678` | `FillContextMenuItems(result, list, request, hasPollOption);` | `FillContextMenu` 唯一调用点 |
| `history_view_context_menu.cpp:1662-1664` | `base::unique_qptr<Ui::PopupMenu> FillContextMenu(not_null<ListWidget*> list, const ContextMenuRequest &request) {` | 工厂 |
| `history/view/history_view_list_widget.cpp:3556` | `_menu = FillContextMenu(this, request);` | `ListWidget::showContextMenu` 内唯一调用点 |
| `history/view/history_view_list_widget.cpp:3430-3432` | `void ListWidget::contextMenuEvent(QContextMenuEvent *e) { showContextMenu(e); }` | 事件入口 |

【确定】`FillContextMenuItems` 全文**只有一处** `AddMessageActions` 调用（`:1636`），`FillContextMenu` 全文**只有一处** `FillContextMenuItems` 调用（`:1678`）—— B 点是单点插入，无遗漏。

【确定】`ContextMenuRequest::item` 的类型是 `HistoryItem *`（`history_view_context_menu.h:50 HistoryItem *item = nullptr;`），与 A 点的 `HistoryItem *` 一致，**同一个 `Add*Action` 签名可同时服务两点**（这正是 `AddRepeatMessageAction` 的现状）。

### 1.3 第三个（可选、非题述）构建器

`Telegram/SourceFiles/ayu/ui/message_history/history_inner.cpp:1122-1124`
```cpp
1122	void InnerWidget::contextMenuEvent(QContextMenuEvent *e) {
1123		showContextMenu(e);
1124	}
```
这是 AyuGram 自有的「消息历史」列表（编辑历史视图），有独立的 `showContextMenu`（`:1126`）。**本次不接**（题述只要求两处），但若检查点要覆盖「编辑历史」列表，需另加一行；记录在此以免遗漏。

### 1.4 以 `AddRepeatMessageAction` 为模板的完整调用链样例

以「新增 `AyuUi::AddSaveCheckpointAction`」为例，**8 步**（全部是既有模式的复制）：

| 步 | 文件:行（锚点） | 现状摘录 | 新增内容 |
|---|---|---|---|
| ① 头文件声明 | `ayu/ui/context_menu/context_menu.h:48` | `void AddRepeatMessageAction(not_null<Ui::PopupMenu*> menu, HistoryItem *item, HistoryView::Context context);` | `void AddSaveCheckpointAction(not_null<Ui::PopupMenu*> menu, HistoryItem *item);`（若不区分视图则**不需要** `HistoryView::Context` 形参） |
| ② 实现（守卫 + 条件 + 动作） | `ayu/ui/context_menu/context_menu.cpp:784-902` | `785-788` 开关守卫 → `790-798` item/peer 早退 → `800-801` 取 `itemId`/`session` → `803-901` `menu->addAction(text, lambda, icon)` | 同形复制 |
| ③ 设置项（`AyuSettings` 成员） | `ayu/ayu_settings.h:666` | `rpl::variable<ContextMenuVisibility> _showRepeatMessageInContextMenu = ContextMenuVisibility::Hidden;` | 新增一行 |
| ④ getter / setter 声明 | `ayu/ayu_settings.h:309` / `:396` | 见 §3.2 | 各加一行 |
| ⑤ 校验 | `ayu/ayu_settings.cpp:508` | `validateEnum(_showRepeatMessageInContextMenu, defaults._showRepeatMessageInContextMenu);` | 加一行（置于 `:509` 之后） |
| ⑥ setter 实现 | `ayu/ayu_settings.cpp:774-778` | `if (_show...current() == val) return; _show... = val; save();` | 复制 |
| ⑦ JSON 写 / 读 | `ayu/ayu_settings.cpp:1121` / `:1225` | 见 §3.3 | 各加一行 |
| ⑧ 消费 + UI + 文案 + 图标 | `context_menu.cpp:786` / `settings_chats.cpp:347-355` / `lang.strings:9013` / `ayu_icons.style:22` | 见 §3.5–§3.7 | 见 §5.2 |

**无需改动任何 CMake**【确定】：`ayu/ui/ayu_icons.style` 已在 `Telegram/cmake/td_ui.cmake:14-17` 注册：

```
14: set(ayugram_style_files
15:         ayu/ui/ayu_icons.style
16:         ayu/ui/ayu_styles.style
17:         ayu/ui/settings/ayu_settings.style
```

**运行时消费者恰好 2 个**【确定】（`codegraph callers "AyuSettings::showRepeatMessageInContextMenu"` → 2 hits）：
`context_menu.cpp:786`（菜单守卫）与 `settings_chats.cpp:351`（设置行初值）。`.Value()/.Changes()` producer（`ayu_settings.h:524-525`）在全仓 `.cpp` 中零使用，是预留样板。

---

## 2. 回调里如何拿 `FullMsgId` / `MsgId` / `PeerId` / topicId

### 2.1 类型定义

`Telegram/SourceFiles/data/data_msg_id.h:19-49`
```cpp
19: struct MsgId {
...
48: 	int64 bare = 0;
```

`Telegram/SourceFiles/data/data_msg_id.h:146-164`
```cpp
146: struct FullMsgId {
147: 	constexpr FullMsgId() noexcept = default;
148: 	constexpr FullMsgId(PeerId peer, MsgId msg) noexcept
149: 	: peer(peer), msg(msg) {
...
162: 	PeerId peer = 0;
163: 	MsgId msg = 0;
164: };
```

### 2.2 取值表（A 点与 B 点通用，因为形参都是 `HistoryItem *item`）

| 需要的量 | 写法 | 依据 |
|---|---|---|
| `FullMsgId` | `item->fullId()` | `history/history_item.cpp:3693-3695` `return FullMsgId(_history->peer->id, id);` |
| `MsgId`（消息 id） | `item->id`（`MsgId`）或 `item->fullId().msg` | `history_item.h` 的 `id` 成员；`data_msg_id.h:163` |
| `PeerId` | `item->fullId().peer` **或** `item->history()->peer->id` | `history/history.h:483 const not_null<PeerData*> peer;` |
| `PeerData*` | `item->history()->peer` | 同上 |
| `Session&` | `&item->history()->session()` 或 `item->history()->session().tryResolveWindow()` | 用例见 `context_menu.cpp:508`、`:559`、`:801` |
| **topicId（forum）** | `item->topicRootId()` → `MsgId`，取 `.bare` | `history/history_item.cpp:4355-4367`；`history_item.h:582` |
| 话题对象 | `item->topic()` → `Data::ForumTopic*` | `history/history_item.cpp:2212-2220`；`history_item.h:240` |
| 是否 forum | `item->history()->isForum()` | `history/history.h:86 [[nodiscard]] bool isForum() const;` |
| 话题常量 | `Data::ForumTopic::kGeneralId = 1` | `data/data_forum_topic.h:80` |
| monoforum 附带 id | `currentItem->sublistPeerId()` | 用例 `context_menu.cpp:828` |
| 相册组 | `item->history()->owner().itemOrItsGroup(item)` / `groups().find(item)` | 用例 `context_menu.cpp:532`、`:905` |

### 2.3 既有取法逐条摘录

`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:794-801`（`AddRepeatMessageAction`）
```cpp
794: 	const auto history = item->history();
795: 	const auto peer = history->peer;
796: 	if (!peer->isUser() && !peer->isChat() && !peer->isMegagroup() && !peer->isGigagroup()) {
797: 		return;
798: 	}
799:
800: 	const auto itemId = item->fullId();
801: 	const auto session = &history->session();
```

`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:834-836`（**topicId 标准取法**）
```cpp
834: 			if (currentItem->topic()) {
835: 				action.replyTo.topicRootId = currentItem->topicRootId();
836: 			}
```

`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:63`（另一种 topic 比较写法）
```cpp
63: 						&& (!topicId || (item->topicRootId().bare == topicId))) {
```

`Telegram/SourceFiles/history/history_item.cpp:4355-4367`
```cpp
4355: MsgId HistoryItem::topicRootId() const {
4356: 	if (const auto reply = Get<HistoryMessageReply>()
4357: 		; reply && reply->topicPost()) {
4358: 		return reply->topMessageId();
4359: 	} else if (const auto data = GetServiceDependentData()
4360: 		; data && data->topicPost && data->topId) {
4361: 		return data->topId;
4362: 	} else if (const auto info = Get<HistoryServiceTopicInfo>()) {
4363: 		if (info->created()) {
4364: 			return id;
4365: 		}
4366: 	}
4367: 	return Data::ForumTopic::kGeneralId;
```

【确定】**非 forum 时 `topicRootId()` 返回 `kGeneralId`(=1)，不是 0** —— 落库时必须显式判 forum，否则会把 1 当成真实 topicId 存进去。
【确定】A 点已用同一判定式：`history_inner_widget.cpp:3045-3047`
```cpp
3045		const auto topicRootId = item->history()->isForum()
3046			? item->topicRootId()
3047			: 0;
```

### 2.4 回调里解析「当前仍在的消息」

【确定】既有动作在 lambda 内**重新查一次** `history->owner().message(itemId)`，避免 `HistoryItem*` 悬空（`context_menu.cpp:819-822`）：
```cpp
819: 			const auto currentItem = history->owner().message(itemId);
820: 			if (!currentItem) {
821: 				return;
822: 			}
```
检查点动作应照抄这一形状。

---

## 3. Ayu 开关模式（新增一个可见性开关）

### 3.1 存储与序列化形态

【确定】**不走 `QDataStream`、不走 `Core::Settings::serialize()`、不走 `writePref/readPref`**；走 **nlohmann::json 独立文件**。

`Telegram/SourceFiles/ayu/ayu_settings.cpp:31-33`
```cpp
31: std::string getSettingsPath() {
32: 	return (cWorkingDir() + u"tdata/ayu_settings.json"_q).toStdString();
33: }
```

`Telegram/SourceFiles/ayu/ayu_settings.cpp:367` / `:419` / `:429`：`AyuSettings::load()` / `save()` / `reset()`；单例入口 `ayu_settings.cpp:362-365`（Meyers）。
【确定】`AyuSettings::load()` 的唯一调用点：`Telegram/SourceFiles/core/core_settings.cpp:541 AyuSettings::load();`（早于 `QDataStream stream(serialized);`）。

【确定】因此**不存在** `AGENTS.md` 里警告的「`QDataStream` 字段必须追加到流尾」陷阱：`to_json`（`ayu_settings.cpp:1072 void to_json(nlohmann::json &j, const AyuSettings &s) {`）与 `from_json`（`:1172 void from_json(const nlohmann::json &j, AyuSettings &s) {`，`:1173 AyuSettings defaults;`）都是整表重建，新增字段加一行即可，旧 JSON 缺 key 时自动回落到成员声明处的默认值。

### 3.2 `AyuSettings` 的 4 个必改点（结构定义）

| # | 文件:行 | 现状摘录 |
|---|---|---|
| 1 | `ayu/ayu_settings.h:666` | `rpl::variable<ContextMenuVisibility> _showRepeatMessageInContextMenu = ContextMenuVisibility::Hidden;` |
| 2 | `ayu/ayu_settings.h:309` | `[[nodiscard]] ContextMenuVisibility showRepeatMessageInContextMenu() const { return _showRepeatMessageInContextMenu.current(); }` |
| 3 | `ayu/ayu_settings.h:396` | `void setShowRepeatMessageInContextMenu(ContextMenuVisibility val);` |
| 4 | `ayu/ayu_settings.h:524-525` | `...Value() const { return _x.value(); }` / `...Changes() const { return _x.changes(); }`（样板，**全仓无 `.cpp` 消费**） |

同族区段：成员 `:661-667`、getter `:304-310`、setter 声明 `:391-397`、producer `:514-527`。

### 3.3 序列化 3 处（`.cpp`）

| # | 文件:行 | 摘录 |
|---|---|---|
| 5 | `ayu/ayu_settings.cpp:508` | `validateEnum(_showRepeatMessageInContextMenu, defaults._showRepeatMessageInContextMenu);`（`validateEnum` 定义 `:493-499`，`max=2`；`:530-532` 有 `modified` 时 `save()`） |
| 6 | `ayu/ayu_settings.cpp:1121` | `{"showRepeatMessageInContextMenu", s._showRepeatMessageInContextMenu.current()},` |
| 7 | `ayu/ayu_settings.cpp:1225` | `s._showRepeatMessageInContextMenu = j.value("showRepeatMessageInContextMenu", defaults._showRepeatMessageInContextMenu.current());` |
| 8 | `ayu/ayu_settings.cpp:774-778` | setter 实现：`<br>if (curr == val) return; _x = val; save();` |

### 3.4 三态枚举与 `needToShowItem`

`Telegram/SourceFiles/ayu/ayu_settings.h:35-39`
```cpp
35: enum class ContextMenuVisibility {
36: 	Hidden = 0,
37: 	Visible = 1,
38: 	VisibleWithModifier = 2,
39: };
```

`Telegram/SourceFiles/ayu/ayu_settings.h:66-70`
```cpp
66: NLOHMANN_JSON_SERIALIZE_ENUM(ContextMenuVisibility, {
67: 	{ContextMenuVisibility::Hidden, 0},
68: 	{ContextMenuVisibility::Visible, 1},
69: 	{ContextMenuVisibility::VisibleWithModifier, 2},
70: })
```

`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:245-248`
```cpp
245: bool needToShowItem(ContextMenuVisibility state) {
246: 	return state == ContextMenuVisibility::Visible
247: 		|| (state == ContextMenuVisibility::VisibleWithModifier && base::IsExtendedContextMenuModifierPressed());
248: }
```

`Telegram/SourceFiles/ayu/utils/qt_key_modifiers_extended.h:13-15`
```cpp
13: [[nodiscard]] inline bool IsExtendedContextMenuModifierPressed() {
14: 	return IsShiftPressed() || IsCtrlPressed();
15: }
```

【确定】语义：`Hidden` 永不显示；`Visible` 总是显示；`VisibleWithModifier` 按住 **Ctrl 或 Shift** 右键才显示。
【确定】**不存在 per-chat / per-peer 例外机制**（`ignorePeerMenuItems` 一类全仓零命中）。例外只来自三层叠加：3 态枚举 × 修饰键判定 × 每个 `Add*Action` 内部的硬编码早退（`item->isLocal()` / `peer->isSelf()` / peer 类型白名单，如 `context_menu.cpp:790-798`）。

### 3.5 设置 UI 挂载点（唯一）

`Telegram/SourceFiles/ayu/ui/settings/settings_chats.cpp:291` `void BuildContextMenuElements(SectionBuilder &builder, AyuSectionBuilder &ayu)`

共享三态文案 `settings_chats.cpp:296-300`：
```cpp
296: 	const auto options = std::vector{
297: 		tr::ayu_SettingsContextMenuItemHidden(tr::now),
298: 		tr::ayu_SettingsContextMenuItemShown(tr::now),
299: 		tr::ayu_SettingsContextMenuItemExtended(tr::now),
300: 	};
```

模板行 `settings_chats.cpp:347-355`：
```cpp
347: 	ayu.addChooseButton({
348: 		.id = u"ayu/showRepeatMessageInContextMenu"_q,
349: 		.title = tr::ayu_RepeatMessage(),
350: 		.boxTitle = tr::ayu_SettingsContextMenuTitle(),
351: 		.initialSelection = static_cast<int>(settings->showRepeatMessageInContextMenu()),
352: 		.options = options,
353: 		.setter = [](int i) { AyuSettings::getInstance().setShowRepeatMessageInContextMenu(static_cast<ContextMenuVisibility>(i)); },
354: 		.icon = { &st::ayuRepeatMenuIcon },
355: 	});
```

**插入锚点**：`settings_chats.cpp:355` 的 `});` 之后、`:356` 的 `if (settings->filtersEnabled()) {` 之前。
结构体定义：`ayu/ui/settings/ayu_builder.h:56-67`（`ChooseButtonArgs` 七字段 + `void addChooseButton(ChooseButtonArgs &&args);`）。
挂载链【确定】：`settings_chats.cpp:462 BuildContextMenuElements(builder, ayu);` ← `kMeta`（`:448-466`）← `AyuChats::setupContent()`（`:481-485`）← 侧边栏 `settings_main.cpp` 的 `AyuChats::Id()`。

### 3.6 文案 key

`Telegram/Resources/langs/lang.strings:9013`
```
"ayu_RepeatMessage" = "Repeat Message";
```
区块上下文 `:9010-9016`；文件共 **9036 行**，`ayu_*` 键集中在尾段。**无 `custom_*.lang`**（`find . -name "*.lang"` 零命中）。

### 3.7 图标

`Telegram/SourceFiles/ayu/ui/ayu_icons.style:15-24`
```
15: ayuGhostIcon: icon {{ "ayu/ghost", menuIconColor }};
...
22: ayuRepeatMenuIcon: icon {{ "ayu/repeat", menuIconColor }};
23: ayuContactsMutualIcon: icon {{ "ayu/contacts_mutual", contactsStatusFg }};
```
资源实体为 PNG，位于 `Telegram/Resources/icons/ayu/`（实测存在 `repeat.png`、`edits_history.png`）。引用处：`context_menu.cpp:901 &st::ayuRepeatMenuIcon`、`settings_chats.cpp:354 .icon = { &st::ayuRepeatMenuIcon }`、`settings_chats.cpp:18 #include "styles/style_ayu_icons.h"`。

【无法确定】新图标 `ayu/checkpoint` 的资源落盘规则（PNG/SVG、是否需打包脚本登记）——本轮未追 `Telegram/cmake/td_ui.cmake` 之外的资源打包逻辑。

---

## 4. 双击右键可行性

### 4.1 右键事件流（【确定】，逐跳可读）

```
QContextMenuEvent
  └─ HistoryInner::contextMenuEvent           history_inner_widget.cpp:2836-2838
       └─ HistoryInner::showContextMenu       history_inner_widget.cpp:2840
            ├─ :2841-2842  reason()==Mouse → mouseActionUpdate(e->globalPos())  // 填充 _dragStateItem
            ├─ :2849      ClickHandler::getActive()
            ├─ :2852-2884 link 相关早退（reaction / fast share / phone）
            ├─ :2885-2916 选区状态 isUponSelected
            ├─ :2981      _menu = base::make_unique_q<Ui::PopupMenu>(this, st::popupMenuWithIcons);
            ├─ :3434      const auto item = _dragStateItem;
            ├─ :3461      addItemActions(item, item);        // ← 插入点 A
            ├─ :3937-3940 if (_menu->empty()) { _menu = nullptr; return; }
            ├─ :3965      _menu->popup(desiredPosition);      // ★ 菜单在此同步弹出
            └─ :3967      e->accept();
```

`ListWidget` 侧（其余列表）：
```
  └─ ListWidget::contextMenuEvent             history_view_list_widget.cpp:3430-3432
       └─ ListWidget::showContextMenu         history_view_list_widget.cpp:3434
            └─ :3556 _menu = FillContextMenu(this, request);
                 └─ history_view_context_menu.cpp:1662 FillContextMenu
                      └─ :1678 FillContextMenuItems(...)
                           └─ :1636 AddMessageActions(...)  // ← 插入点 B（:1134 之后）
```

【确定】`HistoryInner::showContextMenu` 的调用者**只有 2 个**（`codegraph callers "HistoryInner::showContextMenu"` = 2，与 `grep -n "showContextMenu("` 一致）：
- `contextMenuEvent`（`:2836`）
- `touchEvent`（`:2085` 起的长按路径，`:2169-2173` 合成事件）

触摸长按合成事件的先例（**延迟重放方案的关键**）：
```cpp
2169		const auto contextMenu = QContextMenuEvent(
2170			QContextMenuEvent::Mouse,
2171			mapFromGlobal(_touchPos),
2172			_touchPos);
2173			showContextMenu(&contextMenu, true);
```

### 4.2 「弹出菜单后再识别第二次右键」为什么不可靠

| 编号 | 事实 | 置信度 |
|---|---|---|
| F1 | 第一次右键在**同一次事件处理内**同步调用 `_menu->popup()`（`:3965`），事件随后 `accept()`（`:3967`） | 【确定】 |
| F2 | `HistoryInner::mousePressEvent` 开头有 `if (_menu) { e->accept(); return; }`（`:2269-2272`，注释 `// ignore mouse press, that was hiding context menu`）——widget 在菜单存在时会**主动忽略**按下 | 【确定】 |
| F3 | 菜单弹出后是否会形成 Qt 鼠标抓取（mouse grab）使第二次右键**不再**投递到 `HistoryInner`，取决于 `Ui::PopupMenu` / `Ui::Layer` 实现 | 【无法确定】——`Telegram/SourceFiles/ui/widgets/popup_menu.{h,cpp}`、`Telegram/SourceFiles/ui/layers/layer_widget.h`、`Telegram/SourceFiles/base/` **在本 checkout 中不存在**（实测 `ls`/`find` 零命中）。**原因已定位**：它们位于未 checkout 的空子模块 —— `Telegram/lib_ui`（`ls -la` → 空目录，`.gitmodules` 声明 `path = Telegram/lib_ui`）、`lib_base` / `lib_crl` / `lib_rpl` / `lib_storage` 同样为空；`Telegram/CMakeLists.txt:13 add_subdirectory(lib_ui)`。`grep -rn "ShowPopupMenu" Telegram/` → 0 命中 |
| F4 | 右键按下时 Qt 派发 `QContextMenuEvent`（X11 通常 press，Win32 通常 release/WM_CONTEXTMENU），仓库内无法证明任一平台的具体时序 | 【无法确定】——Qt 为外部依赖，仓内无相关断言 |
| F5 | 即使第二次右键能到达，`contextMenuEvent` 里还要重新走 `_menu` 的整套构建（`:2981` 起），与已弹出的菜单实例互相干扰（`:3937` `_menu = nullptr` 会把旧菜单所有权清空） | 【确定】（代码结构层面） |
| **F6** | **应用级事件过滤器已经存在，并且已经在按「右键上下文菜单事件」和「每一次原始鼠标按下」做状态跟踪** —— `Telegram/SourceFiles/core/application.cpp:383` `QCoreApplication::instance()->installEventFilter(this);`；`:766-781` `case QEvent::MouseButtonPress / MouseButtonRelease / MouseButtonDblClick / MouseMove:` 维护 `_lastMouseIgnored`（仅当事件是 `Qt::MouseEventSynthesizedBySystem` 且被 touch 处理过才置 true，`:776`）；`:783-787` `case QEvent::ContextMenu: { const auto ev = static_cast<QContextMenuEvent*>(e); return (ev->reason() == QContextMenuEvent::Mouse) && _lastMouseIgnored; }` —— **即「在 app 级按 reason() 分流 + 条件吞掉右键上下文菜单」已有成熟先例**（当前只在触摸合成场景生效，不是通用右键开关） | 【确定】 |
| **F7** | 右键双击目前**没有任何业务逻辑**：`HistoryInner::mouseDoubleClickEvent` 的快捷动作被 `&& e->button() == Qt::LeftButton`（`history_inner_widget.cpp:2791`）显式限定在左键 | 【确定】 |

**结论**：在「第一次右键立即弹菜单」的前提下做双击右键，**判据不可靠**——它依赖 F3/F4 两个本仓无法证明的前提。**不推荐**。

### 4.3 推荐的 deferred（延迟弹出）方案

**核心洞察**：只要第一次右键**不弹菜单**，就不会有菜单抓取，第二次右键就一定还会产生一个 `QContextMenuEvent`；定时器到期后再用**合成事件**重放原路径（`:2169-2173` 已有先例）。整个方案不依赖 F3/F4。

**落点**：`HistoryInner::contextMenuEvent`（`history_inner_widget.cpp:2836-2838`）——这是最窄的入口，且在 `showContextMenu` 之前。

```cpp
// 现状：history_inner_widget.cpp:2836-2838
void HistoryInner::contextMenuEvent(QContextMenuEvent *e) {
	showContextMenu(e);
}
```

**建议伪码**（设计，不落码）：

```cpp
// 头文件：history_inner_widget.h（成员锚点 :655 _dragStateItem、:668 _trippleClickTimer、:705 _menu）
	base::Timer _ayuCheckpointGestureTimer;
	FullMsgId _ayuGestureItem;
	QPoint _ayuGesturePos;
	crl::time _ayuGestureAt = 0;

	// 新增私有方法（锚点 :440 showContextMenu 声明之后）
	void handleAyuContextMenuGesture(QContextMenuEvent *e, Fn<void()> showMenu);
	void showDeferredContextMenu(const QPoint &globalPos);
```

```cpp
// cpp：history_inner_widget.cpp:2836 附近
void HistoryInner::contextMenuEvent(QContextMenuEvent *e) {
	if (e->reason() == QContextMenuEvent::Mouse) {
		mouseActionUpdate(e->globalPos());          // 与 :2842 同源，幂等
		if (handleAyuContextMenuGesture(e, [=] { showContextMenu(e); })) {
			e->accept();
			return;
		}
	}
	showContextMenu(e);
}
```

```cpp
// 伪码：返回 true 表示"本次已消费，不要弹菜单"
bool HistoryInner::handleAyuContextMenuGesture(
		QContextMenuEvent *e,
		Fn<void()> showMenu) {
	// 0) 总开关（见 §5.3）
	if (!AyuSettings::getInstance().showDoubleRightClickCheckpoint()) {
		return false;
	}
	// 1) 目标消息：与菜单完全一致的 _dragStateItem
	const auto item = _dragStateItem;
	if (!item || !item->isHistoryEntry() || item->isService() || item->isLocal() || item->id <= 0) {
		return false;                                 // 不适用 → 立刻走原菜单，零延迟
	}
	const auto id = item->fullId();
	const auto pos = e->globalPos();
	// 2) 第二次？同一条消息 + 距离阈值 + 时间窗
	if (_ayuCheckpointGestureTimer.isActive()
		&& _ayuGestureItem == id
		&& (pos - _ayuGesturePos).manhattanLength() < kAyuGestureRadiusPx
		&& (crl::now() - _ayuGestureAt) < kAyuGestureWindow) {
		_ayuCheckpointGestureTimer.stop();
		_ayuGestureItem = FullMsgId();
		crl::async(this, [=] {                 // 落库 + toast，见 §5.4
			AyuUi::SaveCheckpoint(id, item->history()->isForum() ? item->topicRootId() : MsgId());
		});
		return true;
	}
	// 3) 第一次：记录 + 起定时器，本次不弹菜单
	_ayuGestureItem = id;
	_ayuGesturePos = pos;
	_ayuGestureAt = crl::now();
	// 【无法确定】base::Timer 的「注册回调」API 名 —— base/timer.h 不在本 checkout。
	//  两种可行写法（均有仓内先例，实现时择一）：
	//   (a) base::fn_delayed(kAyuGestureWindow, this, [=] { showDeferredContextMenu(pos); });
	//       先例：include "base/call_delayed.h"（context_menu.cpp:25）
	//             base::fn_delayed(st::defaultDropdownMenu.menu.ripple.hideDuration, this, [=]{...})
	//             （history_inner_widget.cpp:3138-3140）
	//   (b) 仿 _trippleClickTimer 成员定时器（history_inner_widget.h:668 + :2782 callOnce），
	//       回调注册方式须实现时核对 base/timer.h 的真实签名
	_ayuCheckpointGestureTimer.callOnce(kAyuGestureWindow);
	return true;
}

// 伪码：定时器到期 → 用合成事件重放原菜单路径（先例 :2169-2173）
void HistoryInner::showDeferredContextMenu(const QPoint &globalPos) {
	_ayuGestureItem = FullMsgId();
	auto e = QContextMenuEvent(QContextMenuEvent::Mouse, mapFromGlobal(globalPos), globalPos);
	showContextMenu(&e);
}
```

**可用的既有 API 与先例**：

| 需求 | 既有物 | 位置 |
|---|---|---|
| 一次性定时器 | `base::Timer::callOnce(crl::time(200));` | `data/components/promo_suggestions.cpp:286` |
| HistoryInner 内的定时器成员 | `base::Timer _trippleClickTimer;` | `history_inner_widget.h:668`；用法 `history_inner_widget.cpp:2782 _trippleClickTimer.callOnce(QApplication::doubleClickInterval());` |
| 手势窗口常量 | `QApplication::doubleClickInterval()` | 既有用法 `history_inner_widget.cpp:2782`、`info/media/info_media_list_widget.cpp:1861` |
| 距离阈值 | `QApplication::startDragDistance()` | 既有用法 `history_inner_widget.cpp:2164`、`:2332` |
| 合成 `QContextMenuEvent` | `QContextMenuEvent(QContextMenuEvent::Mouse, mapFromGlobal(_touchPos), _touchPos)` | `history_inner_widget.cpp:2169-2173` |
| 延迟执行 | `base::fn_delayed(...)`（`context_menu.cpp:25` 已 include `"base/call_delayed.h"`；用法 `history_inner_widget.cpp:3138`） | 可替换定时器回调 |
| 异步回到主循环 | `crl::async([=] { ... });` | `context_menu.cpp:845`、`:887` |
| **「按下时刻 + 超时判定」手势的完整模板** | `Ui::MiddleClickAutoscroll`：`crl::time _middlePressedAt = 0;`（`ui/widgets/middle_click_autoscroll.h:51`）、`base::Timer _timer;`（`:53`）、`constexpr auto kHoldToToggleThreshold = crl::time(220);`（`.cpp:18`）、`_middlePressedAt = crl::now();`（`.cpp:108`，在 `toggleOrBeginHold` `:102-110` 内）、`if ((crl::now() - _middlePressedAt) >= kHoldToToggleThreshold) {`（`.cpp:117`）；在 `HistoryInner` 的持有点 `history_inner_widget.h:698`、消费点 `history_inner_widget.cpp:2276-2287` | **本方案的成员与判定式应照抄它** |
| 自定义 500ms 时长常量的既有风格 | `constexpr auto kEphemeralHintHoverDelay = crl::time(500);`（`chat_helpers/field_autocomplete.cpp:69`）、`constexpr auto kQueryDelay = crl::time(500);`（`dialogs/dialogs_search_posts.cpp:20`）；同文件风格 `history_inner_widget.cpp:148-149` `constexpr auto kScrollDateHideTimeout = 800;` | 想要固定 500ms 就照此写 `constexpr auto kAyuGestureWindow = crl::time(500);` |
| 应用级「吞右键上下文菜单」先例 | `core/application.cpp:383` 安装；`:783-787` `case QEvent::ContextMenu:` → `return (ev->reason() == QContextMenuEvent::Mouse) && _lastMouseIgnored;` | 备选方案 F 的落点 |
| 其它 `qApp` 级过滤器先例 | `calls/group/calls_group_messages_ui.cpp:192-201`、`media/view/media_view_overlay_widget.cpp:8460-8463`、`iv/markdown/iv_markdown_embed_overlay.cpp:438`、`calls/calls_panel.cpp:704`、`ui/controls/popup_selector.cpp:53,62` | 同上 |

【确定】**没有** `kDoubleClickIntervalMs` / `kDoubleClick` 之类常量：`grep -rn "doubleClickInterval\|kDoubleClick\|DoubleClickInterval" Telegram/SourceFiles/` 的 12 处命中全部是 `QApplication::doubleClickInterval()` 直接调用。
【确定】`QTimer::singleShot` 在整个 `SourceFiles` 仅 2 处（`export_api_wrap.cpp:3437` 已注释、`test/test_runner.cpp:280`）→ 正确惯用法是 `base::Timer::callOnce`。

【无法确定】`base::Timer` 的完整 API（`callOnce` / `stop` / `isActive` / 回调注册方式）——`Telegram/SourceFiles/base/timer.h` **不在本 checkout**（`ls Telegram/SourceFiles/base/` → `No such file or directory`；根因是 `lib_base` 子模块为空），仅能从 `history_inner_widget.h:668`、`:2782`、`promo_suggestions.cpp:286` 推断存在 `isActive()`（`:2336` 用了 `_trippleClickTimer.isActive()`）与 `callOnce(crl::time)`。**实现时须核对 `base/timer.h` 真实签名。**

### 4.4 备选路径与风险

| 方案 | 做法 | 先例 | 风险 |
|---|---|---|---|
| **D（推荐）** | 第一次右键延迟 `kAyuGestureWindow` 再弹菜单 | 合成事件 `:2169-2173`；手势成员照抄 `MiddleClickAutoscroll`（`ui/widgets/middle_click_autoscroll.{h,cpp}`） | ① 所有右键菜单有固定延迟（可关）；② 若 `HistoryInner` 析构时定时器未停需在析构 `stop()`（`_menu` 成员析构在 `:705`，同理）；③ 触摸长按路径（`:2173` `showContextMenu(&contextMenu, true)`）**不应**进入手势逻辑，需用 `showFromTouch` 或 `reason()` 区分 |
| **E** | 菜单项 + 键盘快捷键 | `HistoryInner::keyPressEvent`（`:4214-4433`），落点 `:4393` 起的 if 链；目标取 `Element::Hovered()`（`:3943-3947` 同款） | 需用户先悬停/选中消息；无 `QShortcut` 注册（`grep -rn "QShortcut" Telegram/SourceFiles/ayu/` 零命中），依赖 HistoryInner 焦点 |
| **F** | 菜单立即弹出 + app 级事件过滤器抓第二次右键 → `_menu->hideMenu()` + 执行动作 | **过滤器先例（更强）**：`core/application.cpp:383` 安装 + `:783-787` 已在 app 级 `return (…)` 吞掉 `QContextMenuEvent`（reason()==Mouse）；`mainwidget.cpp:403` 安装 + `:2852-2863` 已在 app 级处理 `QEvent::MouseButtonPress` 并 `return true`；`hideMenu()` 先例 `history_inner_widget.cpp:618` | ① 依赖 F3/F4（【无法确定】）；② 菜单会先闪一下（`_menu->hideMenu()` 只能在弹出后补救）；③ 必须排除「在菜单/子菜单上右键」；④ 需把钩子装到 `Telegram/SourceFiles/core/application.cpp` 或 `mainwidget.cpp`（非 Ayu 目录），或在 `AyuInfra::init()`（`ayu/ayu_infra.cpp`）里自装过滤器 |
| **G** | 只做菜单项 | §5.1/§5.2 | 无任何延迟/时序风险；功能完整（保存入口本来就够用） |

**D 的落点备选**（若希望复用 `showContextMenu` 已算好的选区/链接状态）：把同样的处理插在 `history_inner_widget.cpp:2965` 之后、`:2981`（`_menu = base::make_unique_q<Ui::PopupMenu>(...)`）之前 —— 此时 `mouseActionUpdate`（`:2842`）、link 早退（`:2852-2884`）、选区状态（`:2885-2916`）、who-reacted 生命周期（`:2944`）都已跑完，且**菜单对象尚未创建**，返回前必须 `e->accept()`。缺点是触及更多既有副作用（`:2944` 已 destroy 过 `_whoReactedMenuLifetime`、`:2842` 已改 hover 状态）。

**推荐组合**：**默认 G**（仅菜单项 + 三态开关）；**可选开启 D**（独立布尔开关，默认关）。理由：保存检查点是低频操作，菜单项已满足；双击右键是「加速器」，不值得为此给所有右键加延迟。

---

## 5. 输出方案

### 5.1 插入点 A（`history_inner_widget.cpp`）

| 项 | 内容 |
|---|---|
| 函数 | `HistoryInner::showContextMenu` 内的 lambda `addItemActions`（`:3033`） |
| 锚点 | `:3132` `AyuUi::AddMessageDetailsAction(_menu, item);` 之后、`:3133` `};` 之前 |
| 新增行 | `AyuUi::AddSaveCheckpointAction(_menu, item);` |
| 形参类型 | `not_null<Ui::PopupMenu*>` = `_menu`（`:2981` 构造）；`HistoryItem*` = 形参 `item` |
| 生效范围 | 仅 `item->isRegular()`（`:3037`）且非完全选中（`:3038-3039`）；主聊天历史（含相册 `:3636`） |
| 是否需要 `HistoryView::Context` | **不需要**（保存动作与视图类型无关；若要与 `AddJumpToCheckpointAction` 联动才需要） |

### 5.2 插入点 B（`history_view_context_menu.cpp`）

| 项 | 内容 |
|---|---|
| 函数 | `HistoryView::AddMessageActions`（`:1124`） |
| 锚点 | `:1134` `AyuUi::AddMessageDetailsAction(menu, request.item);` 之后、`:1135` `}` 之前 |
| 新增行 | `AyuUi::AddSaveCheckpointAction(menu, request.item);` |
| 形参类型 | `not_null<Ui::PopupMenu*> menu`、`HistoryItem *request.item`（`history_view_context_menu.h:50`） |
| 生效范围 | 所有走 `FillContextMenu` 的 `ListWidget`（回复视图 / 置顶列表 / 媒体列表等） |

### 5.3 开关项清单

| 开关 | 类型 | 建议默认值 | 位置（照抄锚点） | 作用 |
|---|---|---|---|---|
| `showCheckpointSaveInContextMenu` | `ContextMenuVisibility`（三态） | `Hidden` | `ayu_settings.h:666` 后 / `:309` 后 / `:396` 后 / `:524-525` 后；`ayu_settings.cpp:508` 后 / `:774` 后 / `:1121` 后 / `:1225` 后 | 「保存检查点」菜单项可见性；`VisibleWithModifier` = 按住 Ctrl/Shift 右键才显示（与既有 5 个 Ayu 动作一致） |
| `doubleRightClickCheckpoint` | `bool` | `false` | 同上四个 `.h` 点 + `.cpp:509` 后 + `:776` 后 + `:1122` 后 + `:1226` 后 | 是否启用 §4.3 的双击右键手势（连带所有右键菜单的延迟） |
| — 文案 | `tr::` key | — | `lang.strings:9013` 之后或文件尾（`:9036` 前） | `"ayu_CheckpointSave" = "Save Checkpoint";` |
| — 图标 | `st::` | — | `ayu_icons.style:22` 之后 + `Telegram/Resources/icons/ayu/` 放图 | `ayuCheckpointMenuIcon: icon {{ "ayu/checkpoint", menuIconColor }};` |
| — 设置行 | `addChooseButton` | — | `settings_chats.cpp:355` 之后、`:356` 之前 | 仿 `:347-355` 七字段；`.id = u"ayu/showCheckpointSaveInContextMenu"_q` |

【确定】`bool` 开关的既有样板：`settings_chats.cpp:376-382` 的 `ayu.addSettingToggle({ .id=..., .title=..., .getter = &AyuSettings::showAttachButtonInMessageField, .setter = &AyuSettings::setShowAttachButtonInMessageField, .icon = {...} });`（结构体 `ayu_builder.h:22-32`）。

### 5.4 动作回调体要做什么（伪码）

```cpp
void AddSaveCheckpointAction(not_null<Ui::PopupMenu*> menu, HistoryItem *item) {
	const auto &settings = AyuSettings::getInstance();
	if (!needToShowItem(settings.showCheckpointSaveInContextMenu())) {
		return;                                            // 照抄 context_menu.cpp:785-788
	}
	if (!item || !item->isHistoryEntry() || item->isService()
		|| item->isLocal() || item->id <= 0) {
		return;                                            // 照抄 :790-792
	}
	if (item->history()->peer->isSelf()) {
		return;                                            // 照抄 :522-524（Save to Messages 无意义）
	}
	const auto itemId = item->fullId();                    // 照抄 :800
	const auto topicRootId = item->history()->isForum()    // 照抄 history_inner_widget.cpp:3045-3047
		? item->topicRootId()
		: MsgId();
	menu->addAction(
		tr::ayu_CheckpointSave(tr::now),
		[=] {
			if (const auto current = item->history()->owner().message(itemId)) {
				AyuCheckpoint::save(
					{ .peerId = itemId.peer,
					  .msgId = itemId.msg,
					  .topicRootId = topicRootId,
					  .monoforumPeerId = current->sublistPeerId(),   // 先例 context_menu.cpp:828
					  .time = current->date() });
				// toast + 缓存失效（写入侧职责，见姊妹文档 §9.1）
			}
		},
		&st::ayuCheckpointMenuIcon);
}
```

**回调体必做清单**：
1. 在 lambda 内**重新解析** `owner().message(itemId)`（`context_menu.cpp:819-822` 先例），不要裸捕获 `HistoryItem*`。
2. `topicRootId` 必须用 `isForum() ? topicRootId() : MsgId()`，**不能**直接用 `item->topicRootId()`（非 forum 时返回 `kGeneralId = 1`，见 §2.3）。
3. 相册消息：`owner().itemOrItsGroup(item)`（`context_menu.cpp:532` 先例）——需决定是存整组还是 leader。
4. monoforum：`current->sublistPeerId()`（`context_menu.cpp:828` 先例）。
5. 建议用 `crl::async` / `crl::guard` 包裹落库（`context_menu.cpp:845`、`:887` 先例；`AGENTS.md` 的 API 回调生命周期条款）。

### 5.5 双击右键最终建议

| 项 | 结论 |
|---|---|
| 「弹出菜单后再识别第二次右键」 | **不采用**（依赖 F3/F4 两个【无法确定】前提；且 `mousePressEvent:2269` 会主动忽略菜单期间的按下） |
| 采纳方案 | **D（deferred）**，由 `doubleRightClickCheckpoint`（默认 `false`）控制 |
| 落点 | `HistoryInner::contextMenuEvent`（`history_inner_widget.cpp:2836`）；若要覆盖其它列表，同法在 `ListWidget::contextMenuEvent`（`history_view_list_widget.cpp:3430`）加 |
| 手势常量 | 窗口建议 `QApplication::doubleClickInterval()`（先例 `:2782`；**仓内无 `kDoubleClick*` 常量**）或固定 `constexpr auto kAyuGestureWindow = crl::time(500);`（先例 `field_autocomplete.cpp:69`、`dialogs_search_posts.cpp:20`）；距离阈值 `QApplication::startDragDistance()`（先例 `:2164`）。成员与判定式照抄 `Ui::MiddleClickAutoscroll`（`ui/widgets/middle_click_autoscroll.h:51,53` + `.cpp:18,102,118`） |
| 必须排除 | ① 触摸长按（`:2173`，`showFromTouch == true`）不应触发；② `reason() != QContextMenuEvent::Mouse`（键盘 context menu，`:2843`）不应触发；③ 非 regular / service / local / 自聊消息直接跳过（零延迟退化）；④ 右键双击**当前无业务逻辑**（`mouseDoubleClickEvent:2791` 被 `e->button() == Qt::LeftButton` 限定），接入点天然干净 |
| 风险 | 延迟（可用开关消除）；析构时定时器需 `stop()`；`mouseActionUpdate` 被提前调用一次（与 `:2842` 重复，幂等但需确认无副作用）；若选落点备选（`:2965`）则还会触发 `:2944` 的 `_whoReactedMenuLifetime.destroy()` |
| 若不做手势 | 仅菜单项（方案 G）已满足需求，零风险 |
| 若要用「已在弹出的菜单上抓第二次右键」 | 走方案 F，`core/application.cpp:783-787` 是现成的「app 级吞右键」落点；但**必须**接受菜单闪烁风险，且需运行期验证 grab 行为 |

---

## 6. 未找到项（含命令与范围）

| 项 | 状态 | 命令 / 范围 |
|---|---|---|
| `Ui::PopupMenu` 实现（是否鼠标抓取、能否在菜单弹出后吞掉第二次右键） | **未找到（文件不存在于本 checkout）** | `find . -iname "*popup*"` → 仅 `ui/controls/popup_selector.{h,cpp}`；`find . -name "popup_menu.h"` → 0 命中；`ls Telegram/SourceFiles/ui/widgets/`（40 项，无 `popup_menu.*`）。**根因**：`Telegram/lib_ui` 子模块为空目录（`ls -la Telegram/lib_ui/` → `total 8`；`.gitmodules` 声明 `path = Telegram/lib_ui`；`Telegram/CMakeLists.txt:13 add_subdirectory(lib_ui)`） |
| `Ui::Layer` / `ui/layers/layer_widget.h` | **未找到** | `ls Telegram/SourceFiles/ui/layers/` → 仅 `standalone_layer_stack.{h,cpp}`；`find . -name "layer*.h"` → 0 命中；`grep -rn "ShowPopupMenu" Telegram/` → 0 命中 |
| `base::Timer` / `base/call_delayed.h` 定义 | **未找到（目录整体缺失）** | `ls Telegram/SourceFiles/base/` → `No such file or directory`（`lib_base` 子模块为空；仅能从调用点反推 API） |
| Qt 对右键 press/release 的 `QContextMenuEvent` 派发时序 | **无法确定** | 仓内无相关断言；Qt 为外部依赖，本次禁构建无法运行验证 |
| 「第二次右键是否还能到达 `HistoryInner::contextMenuEvent`」 | **无法确定** | 同上，且 PopupMenu/Layer 源文件缺失 |
| 仓库内是否已存在「AG 检查点」功能代码 | **未找到** | `grep -rln "checkpoint\|Checkpoint" Telegram/SourceFiles/ayu/` → 仅 `ayu/libs/sqlite/sqlite3.{c,h}`（SQLite 自身的 CHECKPOINT 关键字） |
| `codegraph` 对 Ayu 符号的解析 | **部分不可用** | `codegraph callers AddRepeatMessageAction` → `Symbol "AddRepeatMessageAction" not found`；`codegraph callees "AyuUi::AddRepeatMessageAction"` → 返回 16 个**无关**结果（`addButton`/`setTitle`…），不可信；改用 `grep` 全量枚举。`codegraph callers "HistoryInner::showContextMenu"` → 2 hits（与 grep 一致，可用） |
| per-chat / per-peer 菜单可见性例外机制 | **未找到** | `grep -rn "ignorePeerMenuItems\|ignoreMenuItems\|excludedPeer\|PerChat\|perChat\|perPeer" Telegram/SourceFiles --include=*.cpp --include=*.h` → 0 命中 |
| Ayu 侧 `QShortcut` 先例 | **未找到** | `grep -rn "QShortcut" Telegram/SourceFiles/ayu/` → 0 命中 |
| 新图标资源打包规则 | **无法确定** | 未追 `Telegram/cmake/td_ui.cmake` 之外的资源登记逻辑；已知实体在 `Telegram/Resources/icons/ayu/*.png` |
| `kDoubleClickIntervalMs` / `kDoubleClick` 之类双击间隔常量 | **未找到** | `grep -rn "doubleClickInterval\|kDoubleClick\|DoubleClickInterval" Telegram/SourceFiles/` → 12 处命中全部是 `QApplication::doubleClickInterval()` 直接调用，无常量定义 |
| `_delayedShowContextMenu` / `contextMenuShow` 信号 | **未找到** | `grep -n "_menu->popup\|_menu->setForcedOrigin\|_whoReactedMenuLifetime\|_delayedShowContextMenu\|contextMenuShow" Telegram/SourceFiles/history/history_inner_widget.cpp` → 6 命中，均无这两项 |
| 整体禁用右键菜单的 setting / flag | **未找到** | 最接近的是 AyuGram 的**逐项**三态可见性（`ayu_settings.cpp:503-509`），无全局开关 |
| `history/view/history_view_widget.cpp`（题述提及的基类） | **未找到该文件名** | `ls Telegram/SourceFiles/history/view/ | grep -i "widget\|list"`；实际基类是 `history/view/history_view_list_widget.{h,cpp}`（`ListWidget`，`contextMenuEvent` 在 `:3430`） |
| `HistoryInner` 是否重写 `event()` | **未找到**（即走 Qt 默认分发） | `grep -n "bool event(\|event(QEvent" Telegram/SourceFiles/history/history_inner_widget.h` → 0 命中 |

---

## 7. 执行过的命令清单

```bash
# 环境 / 索引
pwd ; ls                                     # /root/workspace/AyuGramDesktop
codegraph --help ; codegraph status
codegraph callers "HistoryInner::showContextMenu"
codegraph callers "HistoryInner::contextMenuEvent"
codegraph callers AddRepeatMessageAction ; codegraph callers AddMessageDetailsAction
codegraph callees showContextMenu ; codegraph callees "AyuUi::AddRepeatMessageAction"
codegraph query "PopupMenu::popup"

# 目录结构与文件存在性
ls Telegram/SourceFiles/ayu/ ; ls Telegram/SourceFiles/ayu/features/ ; ls Telegram/SourceFiles/ayu/ui/context_menu/
ls Telegram/SourceFiles/ui/ ; ls Telegram/SourceFiles/ui/widgets/ ; ls Telegram/SourceFiles/ui/layers/
find . -iname "*popup*" -not -path "./.git/*"
find . -name "popup_menu.h" -o -name "popup_menu.cpp" ; find . -name "layer*.h"
ls Telegram/SourceFiles/base/                # → No such file or directory
find Telegram/SourceFiles -name "*.cpp" | wc -l    # 1367
find Telegram/SourceFiles -name "*.h"   | wc -l    # 1481
ls docs/ ; ls docs/plan

# 关键符号检索（rg 不存在，全部用 grep tool / grep -rn）
grep -n "contextMenuEvent" Telegram/SourceFiles/history/*.cpp Telegram/SourceFiles/history/*.h
grep -rn "contextMenuEvent" Telegram/SourceFiles
grep -rn "showContextMenu\|_delayedShowContextMenu\|contextMenuShow" Telegram/SourceFiles/history
grep -rn "needToShowItem\|ContextMenuVisibility" Telegram/SourceFiles
grep -rn "showRepeatMessageInContextMenu" Telegram/ --include=*.cpp --include=*.h --include=*.json --include=*.style
grep -rn "AddRepeatMessageAction\|AddHistoryAction\|AddHideMessageAction\|AddUserMessagesAction\|AddMessageDetailsAction" Telegram/SourceFiles
grep -rn "AddRepeatMessageAction" Telegram/SourceFiles
grep -rn "installEventFilter" Telegram/SourceFiles --include=*.cpp --include=*.h
grep -rn "hideMenu()" Telegram/SourceFiles --include=*.cpp
grep -rn "doubleClickInterval\|startDragDistance" Telegram/SourceFiles --include=*.cpp --include=*.h
grep -rn "callOnce(crl::time(" Telegram/SourceFiles --include=*.cpp
grep -rn "QShortcut" Telegram/SourceFiles/ayu/
grep -rn "mouseGrabber\|grabMouse()\|WA_TransparentForMouseEvents" Telegram/SourceFiles --include=*.cpp
grep -rln "checkpoint\|Checkpoint" Telegram/SourceFiles/ayu/
# —— 子智能体补证（双击右键可行性）——
ls -la Telegram/lib_ui/ ; cat .gitmodules ; grep -n "lib_ui" Telegram/CMakeLists.txt ; sed -n '11,15p' Telegram/CMakeLists.txt
grep -rn "ShowPopupMenu" Telegram/
sed -n '378,388p;760,790p' Telegram/SourceFiles/core/application.cpp
grep -rn "MiddleClickAutoscroll" Telegram/SourceFiles/
sed -n '45,55p' Telegram/SourceFiles/ui/widgets/middle_click_autoscroll.h ; sed -n '15,20p;100,122p' Telegram/SourceFiles/ui/widgets/middle_click_autoscroll.cpp
grep -rn "doubleClickInterval\|kDoubleClick\|DoubleClickInterval" Telegram/SourceFiles/
grep -rn "QTimer::singleShot" Telegram/SourceFiles/ --include=*.cpp | wc -l
grep -rn "crl::time(500)\|kQueryDelay\|kEphemeralHintHoverDelay" Telegram/SourceFiles/ --include=*.cpp
grep -rn "installEventFilter" Telegram/SourceFiles --include=*.cpp | head -30
grep -n "bool event(\|event(QEvent" Telegram/SourceFiles/history/history_inner_widget.h
grep -n "_menu->popup\|_menu->setForcedOrigin\|_whoReactedMenuLifetime\|_delayedShowContextMenu\|contextMenuShow" Telegram/SourceFiles/history/history_inner_widget.cpp
ls Telegram/SourceFiles/history/view/ | grep -i "widget\|list"
grep -rn "ayu_RepeatMessage" --include="*" -l .
grep -n "ayu_RepeatMessage" Telegram/Resources/langs/lang.strings
grep -rn "ayuRepeatMenuIcon\|ayuEditsHistoryIcon" Telegram/SourceFiles/ayu/ui/ayu_icons.style
ls Telegram/Resources/icons/ayu/ ; ls Telegram/Resources/langs/ ; wc -l Telegram/Resources/langs/lang.strings
ls Telegram/cmake/ ; grep -rn "ayu" Telegram/cmake/td_ui.cmake
grep -rn "AyuSettings::load()" Telegram/SourceFiles --include=*.cpp
grep -rn "topicRootId" Telegram/SourceFiles/history/history_item.h
grep -n "FullMsgId HistoryItem::fullId" -A 4 Telegram/SourceFiles/history/history_item.cpp
grep -n "struct FullMsgId" -r Telegram/SourceFiles/ ; grep -n "class MsgId" -r Telegram/SourceFiles/
grep -n "bool isForum" Telegram/SourceFiles/history/history.h
grep -n "MsgId id\|kGeneralId" Telegram/SourceFiles/data/data_forum_topic.h
grep -n "enum class Context" -A 6 Telegram/SourceFiles/history/view/history_view_element.h
grep -n "FillContextMenuItems\|FillContextMenu(" -r Telegram/SourceFiles --include=*.cpp --include=*.h
grep -n "addItemActions\|AddMessageActions\|AddTopMessageActions" Telegram/SourceFiles/history/history_inner_widget.cpp Telegram/SourceFiles/history/view/history_view_list_widget.cpp Telegram/SourceFiles/history/view/history_view_context_menu.cpp
grep -n "contextMenuEvent\|context_menu.h\|ayu_settings.h" Telegram/SourceFiles/history/history_inner_widget.cpp Telegram/SourceFiles/history/view/history_view_context_menu.cpp
grep -n "_dragStateItem\|_menu;\|base::Timer _trippleClickTimer" Telegram/SourceFiles/history/history_inner_widget.h
grep -n "addChooseButton\|addSettingToggle" Telegram/SourceFiles/ayu/ui/settings/ayu_builder.h
grep -n "class AyuSectionBuilder" -r Telegram/SourceFiles/ayu/

# 上下文逐点展开
sed -n '2745,2974p'  Telegram/SourceFiles/history/history_inner_widget.cpp
sed -n '3020,3179p'  Telegram/SourceFiles/history/history_inner_widget.cpp
sed -n '3880,3969p'  Telegram/SourceFiles/history/history_inner_widget.cpp
sed -n '2150,2189p'  Telegram/SourceFiles/history/history_inner_widget.cpp
sed -n '2268,2342p'  Telegram/SourceFiles/history/history_inner_widget.cpp
sed -n '3455,3470p;3625,3645p;4214,4273p;4393,4440p' Telegram/SourceFiles/history/history_inner_widget.cpp
sed -n '1100,1159p;1480,1559p;1560,1669p' Telegram/SourceFiles/history/view/history_view_context_menu.cpp
sed -n '3425,3494p;3556,3587p' Telegram/SourceFiles/history/view/history_view_list_widget.cpp
sed -n '460,579p;770,949p' Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp
sed -n '1,80p'      Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp
sed -n '280,389p'   Telegram/SourceFiles/ayu/ui/settings/settings_chats.cpp
sed -n '500,515p;765,790p;1108,1128p;1215,1232p' Telegram/SourceFiles/ayu/ayu_settings.cpp
sed -n '28,36p' ; sed -n '33,42p;64,72p' Telegram/SourceFiles/ayu/ayu_settings.h
sed -n '1,40p'  Telegram/SourceFiles/ayu/ui/ayu_icons.style
sed -n '1,92p'  Telegram/SourceFiles/ayu/ui/settings/ayu_builder.h
sed -n '1,60p'  Telegram/SourceFiles/ayu/ayu_state.h ; sed -n '1,60p' Telegram/SourceFiles/ayu/data/ayu_database.h
sed -n '1,50p'  Telegram/SourceFiles/ayu/ayu_infra.cpp
sed -n '15,75p;143,175p' Telegram/SourceFiles/data/data_msg_id.h
sed -n '395,415p;2840,2870p' Telegram/SourceFiles/mainwidget.cpp
sed -n '1,40p'  Telegram/SourceFiles/ayu/utils/qt_key_modifiers_extended.h
head -40  Telegram/SourceFiles/ayu/ui/ayu_icons.style
cat    Telegram/SourceFiles/ayu/ui/settings/ayu_builder.h
```

> 注：全程未执行 `git`、未执行任何构建/测试/docker 命令、未修改 `Telegram/` 下任何文件；唯一写入为本文件 `docs/plan/p1-checkpoint-entry.md`。

## 主线程抽查（2025-09-27）
- ✅ 插入锚点实测：`history_inner_widget.cpp:3132` = `AyuUi::AddMessageDetailsAction(_menu, item);`，`:3133` = `};` —— 与报告一致。
- ✅ 「双击右键不可靠」的判断被主线程采信：菜单在首次右键同一次处理内同步 popup，`lib_ui` 为空目录无法证伪 Qt 派发行为 → 采纳 deferred 方案（默认关）。
- ⚠️ 需用户拍板：deferred 手势是否默认开启（开启则所有右键有 ~500ms 延迟）。
