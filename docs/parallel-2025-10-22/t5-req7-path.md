# t5-req7-path — 需求7（编辑并重发/重放）：右键菜单 → 主输入框 可达链穷举

- 仓库：`/root/workspace/AyuGramDesktop`（AyuGramDesktop，tdesktop v7.0.9 fork）
- 分支 / HEAD：`dev` / `3bb9891cb chore: update submodule`，另有未提交改动（`git status` 见下）
- 性质：**只读调查**。未构建、未安装、未 `git commit`、未改任何既有文件；仅新增本报告。
- 约束：`Telegram/lib_ui`、`Telegram/lib_base` 等子模块目录为空（见 §5.2），故一切 API 只能以主仓 `Telegram/SourceFiles/**` 中的声明为准。

---

## 0. 结论速览

1. **存在可行链，且不需要初始化任何子模块、不需要 lib_ui 头文件**。但它不在「handler 已拿到的 `PeerData* / Data::Thread* / Window::SessionController*`」这三个对象上，而在**菜单构造点的第 3 个参数 `not_null<HistoryView::ListWidget*> list`** 上——该指针在 Ayu 调用点**已经在作用域内**，只是没有被传进 `AyuUi::AddRepeatMessageAction`。
2. `HistoryWidget::insertTextAtCursor` 的行号需更正：真实声明在 `history/history_widget.h:260`（不是 §7 记的 113；113 只是 `class Element;` 之类的前向声明）。
3. 已存在**两个**「从右键菜单把内容送进输入框」的上游先例（§4），说明机制是官方既有能力，不是新发明。
4. 「富文本 / 整框替换」比「纯文本追加」难一档：整框替换的入口 `ComposeControls::setText` / `HistoryWidget::setFieldText` 一个**公开但不可达**、一个**不可达且私有**（§3）。一栏布局可用 `ListWidget::editMessageRequestNotify` 绕过（公开、零类型参数）。
5. 一期（把原文载入输入框）在本 checkout 内部**可做**；二期（确保替换语义 / 富文本 / 自动发送）需额外小改动，但**仍不需要子模块**。

---

## 1. 证据纪律

所有结论均来自 `read`/`grep` 对本 checkout 的实际读取；行号经 `sed -n '<n>p'` 逐条回读复核（命令见 §8）。
无法由静态代码确定的写【无法确定】。

`git status --short` 相关项（本报告只读，未触碰）：

```
 M Telegram/Resources/langs/lang.strings
 M Telegram/SourceFiles/ayu/ayu_settings.cpp
 M Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp
 M Telegram/SourceFiles/ayu/ui/context_menu/context_menu.h
 M Telegram/SourceFiles/history/history_inner_widget.cpp
 M Telegram/SourceFiles/history/history_inner_widget.h
 M Telegram/SourceFiles/history/view/history_view_context_menu.cpp
```

---

## 2. Task 1：候选链逐条验证

### 2.1 通过的候选（本 checkout 内、零子模块依赖）

#### 链 A — 一栏（`HistoryView::ChatSection`）布局

```
右键菜单 AddMessageActions(menu, request, list)          history_view_context_menu.cpp:1124-1127
  └ AyuUi::AddRepeatMessageAction(menu, request.item, context)   history_view_context_menu.cpp:1133
      （list 在作用域内：是 AddMessageActions 的第 3 个形参）
  → ListWidget::insertTextAtCursor(const QString&)      history_view_list_widget.h:423   [public]
      impl → _insertTextCallback                            history_view_list_widget.cpp:5198-5202
  → 回调在 ChatSection 里注册：
       _inner->setInsertTextCallback([=](const QString &text) {   history_view_chat_section.cpp:435
         if (const auto field = _composeControls->fieldForMention()) {  ...:436
           Menu::InsertTextAtCursor(field, text);                    ...:437
  → ComposeControls::fieldForMention()                   history_view_compose_controls.h:297  [public]
  → Menu::InsertTextAtCursor(not_null<Ui::InputField*>, const QString&)  menu/menu_timecode_action.h:21-23
```

可见性佐证：`history_view_list_widget.h` 的访问区界限为 `public:` 在 116/219/271/311，`protected:` 在 535 —— 423 落在 **311–535 的 public 段**内。`history_view_compose_controls.h` 为 `public:` 144 / `private:` 299 —— 267、297 均在 public 段。

#### 链 B — 两栏（`HistoryWidget`）布局

```
HistoryInner 自己的 _menu 里加 Ayu 动作                history_inner_widget.cpp:3193
      _widget 即 HistoryWidget*（HistoryMainElementDelegateMixin::setCurrent，history_inner_widget.h:88-91）
→ HistoryWidget::insertTextAtCursor(const QString&)     history/history_widget.h:260   [public]
      impl → Menu::InsertTextAtCursor(_field, text)      history/history_widget.cpp:6585-6590
```

可见性佐证：`history_widget.h` 访问区 `public:` 148 / `protected:` 349 / `public:` 358 / `private:` 361；**260 在 148–349 的 public 段内**。

#### 链 A′ — 一栏「进入编辑并载入原文」（富文本、零类型参数）

```
ListWidget::editMessageRequestNotify(FullMsgId) const   history_view_list_widget.h:413  [public]
      impl → _requestedToEditMessage.fire()             history_view_list_widget.cpp:5153-5155
→ ChatSection 订阅 _inner->editMessageRequested()       history_view_chat_section.cpp:392
      rpl::filter([] { return !_joinGroup; })           ...:393-395
→ _composeControls->editMessage(fullId, _inner->getSelectedTextRange(item))   ...:399-401
```

这条与内置「编辑」菜单项走的是同一条路：`history_view_context_menu.cpp:776` 就是 `list->editMessageRequestNotify(item->fullId());`（完整上下文 761-782，含 `item->allowsEdit(...)` 守卫与 richPage 的 `Ui::PreventDelayedActivation()`）。

### 2.2 被否决的候选（含否决证据）

| 候选 | 检查结果 | 结论 |
|---|---|---|
| `Window::SessionController::content()` | `window/window_session_controller.h:423` → `[[nodiscard]] not_null<MainWidget*> content() const;`（public 段 410 之后） | API 存在，但止步于 `MainWidget*` |
| `MainWidget` 的公开 `HistoryWidget` 访问器 | `mainwidget.h` 中 `HistoryWidget` 仅出现 2 次：`14: class HistoryWidget;`（前向声明）、`348: const base::unique_qptr<HistoryWidget> _history;`（私有，且在 `private:` 段）；`mainwidget.cpp:273` 构造 `_history(std::in_place, this, _controller)`，`mainwidget.cpp:1537` 仅内部 `_history->applyDraft(...)` | **否决：dead end**，无任何公开访问器 |
| `SessionController::activeChatsContent()` / `historyWidget()` | `grep -c "activeChatsContent\|historyWidget" window_session_controller.h` → **0** | **不存在该 API** |
| `SessionController::activeLayerSection()` | `window_session_controller.h:570` → `[[nodiscard]] SectionWidget *activeLayerSection() const;`（public） | 存在但返回的是**层** section，且 `SectionWidget` 无 `history()` / `HistoryWidget` 访问器（`grep -c "HistoryWidget\|historyWidget" window/section_widget.h` → **0**）→ **dead end** |
| `Window::ChatWidget` | `window/window_chat_widget.h` **不存在**（`ls` 报 No such file） | **v7.0.9 已无此类型** |
| `App::wnd()` 链 | `grep -c "wnd()" core/application.h` → **0** | **本源码树无 `App::wnd()`** |
| `History::owner()` / `PeerData::owner()` | `data/data_peer.h:226` → `[[nodiscard]] Data::Session &owner() const;` | 返回 `Data::Session&`，**不是 widget**，确认背景描述 |
| `HistoryWidget::setFieldText` 作为整框替换入口 | `history/history_widget.h:702`，位于 `private:`（361 之后）段；调用点全在 `history_widget.cpp` 内（1950/2910/5291/…/10158 impl） | **私有**，外部不可调用 |
| `Data::Draft` + 切聊天写入草稿 | `data/data_drafts.h:57-66` 的构造函数首参即 `const TextWithTags &textWithTags`；而 `TextWithTags` 在本 checkout **只有前向声明**（`calls/group/calls_group_message_field.h:12`、`calls/group/calls_group_messages_ui.h:15`），**主仓源码中无定义** | 否决（会硬依赖 lib_ui）。`MessageCursor` 反而在主仓：`data/data_types.h:208` |

`addCallback`/`PeerMenuCallback` 那一路（`AddAyuGramActions(PeerData*, Data::Thread*, SessionController*, const PeerMenuCallback&)`，`ayu/ui/context_menu/context_menu.h:24`）确实同时具备 peer + thread + controller，但它能拿到的仍只是上面这些 dead end，故**同样不成立**。

---

## 3. Task 2：`HistoryWidget` 公开方法里除 `insertTextAtCursor` 外的「设置输入框内容」入口

`history_widget.h` 公开段（148–349）内与「输入框内容」相关的全部条目：

| 方法 | 签名 | 位置 | 说明 |
|---|---|---|---|
| `insertTextAtCursor` | `void insertTextAtCursor(const QString &text);` | `history/history_widget.h:260` | public；实现 `history_widget.cpp:6585` → `Menu::InsertTextAtCursor(_field, text)`；**依赖 `_canSendTexts` 守卫**；语义是**在光标处插入**（且 `Menu::InsertTextAtCursor` 会自动补前后空格），不是整框替换 |
| `editMessage` | `void editMessage(not_null<HistoryItem*> item, const TextSelection &selection);` | `history/history_widget.h:232-234` | public；实现 `history_widget.cpp:10337` → `PrepareEditText(item)` + `setLocalEditDraft` + `applyDraft()`，**整框载入原文并进入编辑模式**；但第 2 参 `TextSelection` 的定义在 lib_ui（主仓仅前向声明，如 `history/view/media/history_view_media.h:16`） |
| `replyToMessage` | `void replyToMessage(FullReplyTo id);` / `void replyToMessage(not_null<HistoryItem*> item, FullReplyTo fields = {});` | `history/history_widget.h:228-231` | public，只设回复目标，不写正文 |
| `applyDraft` | `bool applyDraft(FieldHistoryAction = FieldHistoryAction::Clear);` | `history/history_widget.h:270-271` | public，重放当前 local draft（非文本载入） |
| `setInnerFocus` | `void setInnerFocus();` | `history/history_widget.h:292` | public，仅焦点，不改内容 |
| `setFieldText` | `void setFieldText(const TextWithTags&, TextUpdateEvents = 0, FieldHistoryAction = FieldHistoryAction::Clear);` | `history/history_widget.h:702` | **private**，正是「整框替换 + 富文本」的那个，但外部不可达 |

结论：`HistoryWidget` 公开面里**只有 `insertTextAtCursor` 能做到「不改签名即可调用的文本载入」**，且是插入语义；整框替换/富文本要么走 `editMessage`（需 `TextSelection`），要么走私有的 `setFieldText`。

补充：`HistoryInner::getSelectedTextRange`（`history_inner_widget.h:471`）落在 protected 段（297–314），**外部不可调用**，所以别指望从 `HistoryWidget*` 现造 `TextSelection`。

---

## 4. Task 3：已有的「从菜单 handler 操作输入框」先例（grep 全仓）

| 先例 | 调用点 | 说明 |
|---|---|---|
| 一栏 · 语音时间码插入 | `history_view_context_menu.cpp:1533-1539`：`strong->insertTextAtCursor(tc.value_or(*timecode));`（`strong` 来自 `base::make_weak(list.get())`，即 `ListWidget*`） | **官方既有**：菜单回调里对 `ListWidget` 调 `insertTextAtCursor` |
| 两栏 · 语音时间码插入 | `history_inner_widget.cpp:3466-3467`：`_widget->replyToMessage({...}); _widget->insertTextAtCursor(cur.value_or(*t));` | **官方既有**：菜单回调里对 `HistoryWidget*` 调 `insertTextAtCursor` |
| 两栏 · 编辑消息 | `history_inner_widget.cpp:3137-3150`：`getSelectedTextRange(item)` + `_widget->editMessage(item, selection);` | **官方既有**：菜单回调里整框载入原文进入编辑 |
| 一栏 · 编辑消息 | `history_view_context_menu.cpp:761-782` → `:776 list->editMessageRequestNotify(item->fullId());` | **官方既有** |
| `inputField()` | 全仓 grep 命中集中在 `window_peer_menu` 的提及补全：`window/window_peer_menu.h:85 fieldForMention`、`window_peer_menu.cpp:4202/4227` | Ayu 侧**无** `inputField()` 直取用法 |
| `->insertText` | 仅 `menu/menu_timecode_action.cpp:91-106` 的 `cursor.insertText(insert)`（在 `Menu::InsertTextAtCursor` 内） | 无别处旁路 |

即：**机制早已存在且在语音消息时间码、编辑消息两处产品功能中被使用**；缺口纯粹是 Ayu 的 repeat 入口没有把 `list` / `_widget` 传进去。

---

## 5. 结论 A / B / C

### A. 是否存在仅依赖本 checkout 已有、且不引入 lib_ui 头文件的可行链？

**是。** 精确草稿（未落地，仅示意；两处均为「传参 + 一次调用」，不动任何子模块）：

```cpp
// ayu/ui/context_menu/context_menu.h —— 头文件只做前向声明，不新增 include
namespace HistoryView { class ListWidget; }   // 与已有 enum Context 前向声明并列（context_menu.h:16-18）
class HistoryWidget;                          // 主仓类，前向声明即可

void AddRepeatMessageAction(
	not_null<Ui::PopupMenu*> menu,
	HistoryItem *item,
	HistoryView::Context context,
	HistoryView::ListWidget *list,   // 一栏布局，可为 nullptr
	HistoryWidget *historyWidget);   // 两栏布局，可为 nullptr
```

```cpp
// ayu/ui/context_menu/context_menu.cpp —— 在既有 lambda 内、复用既有 currentItem
const auto text = currentItem->originalText().text;   // 纯文本；originalText() 已在 :891-895 使用
if (list) {
	list->insertTextAtCursor(text);          // ListWidget::insertTextAtCursor  history_view_list_widget.h:423
} else if (historyWidget) {
	historyWidget->insertTextAtCursor(text); // HistoryWidget::insertTextAtCursor history/history_widget.h:260
}
```

两处调用点（`list` / `_widget` 都已在作用域内，只需多传一个实参）：

- `history/view/history_view_context_menu.cpp:1133` → `AyuUi::AddRepeatMessageAction(menu, request.item, context, list, nullptr);`
- `history/history_inner_widget.cpp:3193` → `AyuUi::AddRepeatMessageAction(_menu, item, HistoryView::Context::History, nullptr, _widget);`

每个 API 的凭据：

| API | 位置 | 可见性 |
|---|---|---|
| `AddMessageActions(menu, request, list)` | `history_view_context_menu.cpp:1124-1127` | 文件内 static 风格，`list` 为第 3 形参 |
| `ListWidget::insertTextAtCursor` | `history_view_list_widget.h:423` / impl `:5198` | public（311–535 public 段） |
| `ListWidget::editMessageRequestNotify`（若要编辑态） | `history_view_list_widget.h:413` / impl `:5153` | public |
| `HistoryWidget::insertTextAtCursor` | `history/history_widget.h:260` / impl `:6585` | public（148–349 public 段） |
| `Menu::InsertTextAtCursor` | `menu/menu_timecode_action.h:21-23` / impl `menu_timecode_action.cpp:91-106` | 无需自己调用；链末已由 ChatSection 接好 |
| `ComposeControls::fieldForMention` | `history_view_compose_controls.h:297` | public（无需自己调用） |
| `HistoryItem::originalText()` | 用法 `ayu/ui/context_menu/context_menu.cpp:891-895` | 返回 `TextWithEntities`，取 `.text` / `.entities` |

**限制（必须写清）**：`insertTextAtCursor` 是**插入**语义，`Menu::InsertTextAtCursor`（`menu_timecode_action.cpp:91-106`）会补前后空格。因此「载入原文」若要求**整框替换**，这条链只算「近似」。真正的整框替换入口是 `ComposeControls::setText`（`history_view_compose_controls.h:267`，public）——但从 `ListWidget` / `HistoryWidget` 都没有公开路径拿到 `ComposeControls*`（`_composeControls` 是 `ChatSection` 私有成员，`history_view_chat_section.h:425`，且 `ChatSection` 无公开访问器，`history_view_chat_section.h` 无 `composeControls` 命中）。要编辑态可用链 A′（`editMessageRequestNotify`），它的载入是**整框、富文本**的。

### B. 若否… / 「若只补 Telegram/lib_ui 一个子模块，是否足够」

先强调：**A 已经成立，本路径不依赖 lib_ui**。但就「只补 lib_ui 是否足够」这一问，答案明确是**不够**：

1. 本 checkout 里空的不止 lib_ui。实测（`ls -A | wc -l`）：`Telegram/lib_crl`、`Telegram/lib_rpl`、`Telegram/lib_base`、`Telegram/lib_ui`、`Telegram/lib_storage`、`Telegram/lib_webrtc`、`Telegram/lib_spellcheck` **均为 0 条目**；`.gitmodules` 另列 `Telegram/codegen`、`Telegram/lib_tl`、`Telegram/lib_lottie`、`Telegram/lib_qr`、`Telegram/lib_translate`、`Telegram/lib_icu`、`Telegram/lib_webview`、`cmake` 及大量 `Telegram/ThirdParty/*`。
2. lib_ui 自身依赖 lib_crl / lib_rpl 与 Qt，单独 init lib_ui 也编译不过（其 `ui/text/text.h` 层面的类型如 `TextWithTags`、`TextSelection`、`Ui::InputField` 需要整条依赖链）。
3. 即便全部 init 齐，本会话**无构建能力**（任务硬约束），仍无法验证。
4. 因此对需求7 而言，「补子模块」是**错误方向**；已有链不需要它。若一定要「整框富文本载入且不进编辑态」，正确做法是**在主仓内**把 `ChatSection::_composeControls` 暴露一个只读访问器（或在 `ListWidget` 上补一个 `setTextRequested` 通知），而不是补子模块。

### C. 零新代码近似路径评价（不要求实现）

| 近似路径 | 位置 | 可行性评价 |
|---|---|---|
| 现有「重复消息」（Ayu） | `ayu/ui/context_menu/context_menu.cpp:813` 起，`Api::SendAction` + `originalText()`→`message.textWithTags`（:891-895）直接发送/转发 | **立即可用，但语义不同**：不写输入框、立即发出，且 richPage 走 `AyuForward::forwardRichMessage`。可作「重发」的一键版，不能作「载入编辑」 |
| 现有「编辑消息」（原生） | 一栏 `history_view_context_menu.cpp:776`；两栏 `history_inner_widget.cpp:3137-3150` | **最接近需求7「编辑并重发」**：原文（含富文本）整框载入并进入编辑模式，改完再发即「重发」。零新代码，唯一缺口是 Ayu 菜单没复用这条通知。注意一栏路径有 `!_joinGroup` 过滤（`history_view_chat_section.cpp:393-395`），预览/加入群场景会被丢弃 |
| 语音时间码插入 | 一栏 `:1537`、两栏 `:3467` | 证明「菜单→输入框」官方可行，但只是插入片段，不是载入全文 |

评价：若一期只求「能载入原文」，**「复用原生编辑通知」比「insertTextAtCursor」语义更准**；若产品要「以原文新建一条消息（非编辑原消息）」，则必须实现 A 的传参 + `insertTextAtCursor`，并接受「插入而非替换」的差异，或补一个 `setTextRequested` 通知。

---

## 6. 与 `HANDOFF.md` §7 的出入（仅记录，未修改 HANDOFF.md）

1. §7 记 `HistoryWidget::insertTextAtCursor` 在 `history/history_widget.h:113` —— **不准确**。实际声明在 `:260`；`:113` 是 `class Element;` 一类前向声明（见 `history_widget.h:110-116`）。
2. §7 的结论「从消息右键菜单写入主输入框没有公开路径」——**需要修正**：公开路径存在（`ListWidget::insertTextAtCursor` / `editMessageRequestNotify`），阻塞点是 Ayu handler 的**形参签名**没有携带目标指针，而不是缺少公开 API。
3. §7 的备选「写本地草稿 + 切聊天」判断——**正确**：`Data::Draft` 首参 `TextWithTags` 在 checkout 内无定义，确实会引入 lib_ui。

---

## 7. 无法确定项

- 【无法确定】默认配置下实际活跃的是哪套布局（一栏 `ChatSection` 还是两栏 `HistoryWidget`）。静态代码显示两者并存：`mainwidget.cpp:273` 恒构造 `_history`，`ChatSection` 另由 section 机制构造。运行期路由未验证（无构建能力）。
- 【无法确定】`editMessageRequestNotify` 是否在所有 `ListWidget` 实例上都有回调：目前 `setInsertTextCallback` 全仓仅 `history_view_chat_section.cpp:435` 一处注册；若某个 `ListWidget` 不在 `ChatSection` 内，`insertTextAtCursor` 会静默无效（`_insertTextCallback` 为空则不做事，`history_view_list_widget.cpp:5198-5202`）。
- 【无法确定】`AyuUi::AddRepeatMessageAction` 的 `HistoryView::ListWidget` 前向声明是否会与该 TU 已有的同名声明冲突：`ayu/ui/context_menu/context_menu.h:16-18` 已有 `namespace HistoryView { enum class Context : char; }`，新增 `class ListWidget;` 需同处合并，未编译验证。
- 【无法确定】需求7 的产品语义究竟偏「编辑原消息」（用 `editMessageRequestNotify`）还是「以原文新建消息」（用 `insertTextAtCursor`）。代码层面两者都可行，取舍需产品确认。

---

## 8. 附录：复核命令

```bash
# 访问区界限（public/protected/private 行号）
grep -n "^public:\|^private:\|^protected:" \
  Telegram/SourceFiles/history/history_widget.h \
  Telegram/SourceFiles/history/view/history_view_list_widget.h \
  Telegram/SourceFiles/history/view/controls/history_view_compose_controls.h \
  Telegram/SourceFiles/window/window_session_controller.h

# 全仓调用点
grep -rn "insertTextAtCursor\|editMessageRequestNotify\|setInsertTextCallback\|fieldForMention" \
  Telegram/SourceFiles/

# 否决证据
grep -n "HistoryWidget" Telegram/SourceFiles/mainwidget.h
grep -c "activeChatsContent\|historyWidget" Telegram/SourceFiles/window/window_session_controller.h
grep -c "HistoryWidget\|historyWidget" Telegram/SourceFiles/window/section_widget.h
grep -c "wnd()" Telegram/SourceFiles/core/application.h
ls Telegram/SourceFiles/window/window_chat_widget.h            # No such file
grep -rn "struct TextWithTags\|class TextWithTags" Telegram/SourceFiles/   # 仅前向声明

# 子模块空目录
for d in Telegram/lib_crl Telegram/lib_rpl Telegram/lib_base Telegram/lib_ui \
         Telegram/lib_storage Telegram/lib_webrtc Telegram/lib_spellcheck; do
  printf '%-28s %s entries\n' "$d" "$(ls -A "$d" 2>/dev/null | wc -l)"
done
```
