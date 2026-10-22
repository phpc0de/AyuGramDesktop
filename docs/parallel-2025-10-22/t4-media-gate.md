# T4：媒体级 allowsForward() 闸门取证报告

- 仓库：`/root/workspace/AyuGramDesktop`（AyuGramDesktop = tdesktop v7.0.9 fork，分支 `dev`，HEAD `db3b9891cb`）
- 任务性质：**只读调查**，未修改任何源码、未构建、未 git commit
- 调查日期：本次会话
- 结论先行：**媒体闸门在 peer 级解锁后不会拦截转发/复制/下载**——唯一返回 `false` 的实现是 `MediaCall`，而携带 `MediaCall` 的 item 全部是服务消息，早在 `isService()` 分支就已不可转发

---

## 0. 关键结论（TL;DR）

1. `Data::Media::allowsForward()` 全仓共 **1 个基类定义 + 2 个覆写**：基类返回 `true`，`MediaFile` 返回 `true`，**只有 `MediaCall` 返回 `false`**。**其余所有媒体类型（Photo/Document/WebPage/Poll/Game/Invoice/Location/Paper/Gift/Giveaway/SharedContact/Story 等）都不覆写，一律继承基类的 `true`**。
2. `MediaCall` 只由 **服务消息** 构造：`HistoryItem::HistoryItem(history, id, MTPDmessageService 基数, ...)`（`history_item.cpp:605-608`）内的 `messageActionPhoneCall` / `messageActionConferenceCall` 分支（`618-627`），以及 `HistoryItem::applyEdition(const MTPDmessageService&)`（`2446`）内的 `2486`。而 `isService()` 实现为 `Has<HistoryServiceData>()`（`history_item.cpp:4307-4309`），服务消息在 `HistoryItem::allowsForward()` 的**第一条件** `!isService()`（`3064`）就已被否掉。
3. 因此 `_media->allowsForward() == false` 在本仓库内 **不可单独触发"媒体导致不能转发"**：它只在同一个 item 已经因 `isService()==true` 而不可转发时才为 `false`。对非服务消息而言，`(!_media || _media->allowsForward())`（`history_item.cpp:3068`）**恒为 true**。
4. 三个消费门（转发可见性 / 复制限制 / 下载保存限制）全部只读 **peer 级 `allowsForwarding()` + 消息级 `forbidsForward()`/`forbidsSaving()`**，其中后两者在 HEAD 恒为 `false`；**没有任何一个门直接读 `_media->allowsForward()`**。所以解锁后复制、下载、转发都没有残留拦截。
5. 真正残留的"不可转发"UI 是 **AyuGram 自有标记层**（`MessageFlag::AyuNoForwards` / `PeerData::isAyuNoForwards()`），它不是媒体闸门，与 `_media->allowsForward()` 无关（见 §6）。

---

## 1. `Data::Media::allowsForward()` 定义与全部实现

### 1.1 声明

| 位置 | 摘录 | 说明 |
|---|---|---|
| `Telegram/SourceFiles/data/data_media_types.h:259` | `virtual bool allowsForward() const;` | 基类纯虚声明位置（`class Media` 内） |
| `Telegram/SourceFiles/data/data_media_types.h:385` | `bool allowsForward() const override;` | `class MediaFile final : public Media`（类声明 `h:345`）的覆写声明 |
| `Telegram/SourceFiles/data/data_media_types.h:502` | `bool allowsForward() const override;` | `class MediaCall final : public Media`（类声明 `h:490`）的覆写声明 |

### 1.2 实现（`grep -rn "allowsForward" Telegram/SourceFiles/` 全量命中，仅 `data_media_types.cpp` 有定义体）

| 位置 | 摘录 | 返回 | 影响的媒体类型 |
|---|---|---|---|
| `Telegram/SourceFiles/data/data_media_types.cpp:734-736` | `bool Media::allowsForward() const { return true; }` | **true** | 基类默认：Photo / Video / Document / Voice / Sticker / WebPage / Poll / Game / Invoice / Location / LiveLocation / Dice / Paper / Gift / Giveaway / SharedContact / StoryMention / Todo / RichPage 等**一切未覆写者** |
| `Telegram/SourceFiles/data/data_media_types.cpp:1385-1387` | `bool MediaFile::allowsForward() const { return true; }` | **true**（与基类同值，无语义收紧） | 文件类媒体（document/photo 承载的 `MediaFile`） |
| `Telegram/SourceFiles/data/data_media_types.cpp:1836-1838` | `bool MediaCall::allowsForward() const { return false; }` | **false** | **通话记录**（语音通话 / 群通话 / 会议通话 的 `messageActionPhoneCall`、`messageActionConferenceCall`） |

**结论**：`false` 的唯一来源是 `MediaCall`（通话媒体）。不存在"受保护媒体/敏感内容/付费媒体"级别的独立 gate —— 在本 fork 中 `noforwards` 一类语义被整体搬到了 Ayu 层（§6），而不是媒体层。

调用面（`media->allowsForward()` 的消费者）**只有一处**：`history_item.cpp:3068`。grep 复核：`grep -rn "allowsForward" Telegram/SourceFiles/` 结果中，形如 `_media->allowsForward()` / `media->allowsForward()` 的仅此一行，其余全部是 `HistoryItem::allowsForward()` / `PeerData::allowsForwarding()`。

### 1.3 `MediaCall` 的 item 是服务消息（关键前提）

| 位置 | 摘录 |
|---|---|
| `Telegram/SourceFiles/history/history_item.cpp:605-608` | `HistoryItem::HistoryItem(not_null<History*> history, MsgId id, const MTPDmessageService &data, MessageFlags localFlags)` |
| `Telegram/SourceFiles/history/history_item.cpp:611-627` | `data.vaction().match([&](const MTPDmessageActionPhoneCall &data) { ... _media = std::make_unique<Data::MediaCall>(this, Data::ComputeCallData(...)); setTextValue({}); }, [&](const MTPDmessageActionConferenceCall &data) { ... _media = std::make_unique<Data::MediaCall>(...); ... }` |
| `Telegram/SourceFiles/history/history_item.cpp:2446` + `2478-2487` | `void HistoryItem::applyEdition(const MTPDmessageService &message)` … `_media = nullptr; _media = std::make_unique<Data::MediaCall>(this, info);` |
| `Telegram/SourceFiles/history/history_item.cpp:4307-4309` | `bool HistoryItem::isService() const { return Has<HistoryServiceData>(); }` |
| `Telegram/SourceFiles/history/history_item.cpp:3064` | `return !isService()` — `allowsForward()` 的第一个条件 |

→ 携带 `MediaCall` 的 item ⇒ `isService()==true` ⇒ `allowsForward()==false`（即便把媒体闸门整句删掉也一样）。

---

## 2. 三个消费门在解锁后的完整链路 + 证据矩阵

前置事实（peer 级已解锁，三处均为 `return true;`）：

| 位置 | 摘录 |
|---|---|
| `Telegram/SourceFiles/data/data_chat.cpp:75-77` | `bool ChatData::allowsForwarding() const { return true; }` |
| `Telegram/SourceFiles/data/data_channel.cpp:787-789` | `bool ChannelData::allowsForwarding() const { return true; }` |
| `Telegram/SourceFiles/data/data_user.cpp:690-692` | `bool UserData::allowsForwarding() const { return true; }`（原本即 true） |
| `Telegram/SourceFiles/data/data_peer.cpp:1728-1738` | `PeerData::allowsForwarding()` 分派到 user/channel/chat，未知类型 `return false;`（`1738`） |

消息级（HEAD 恒 false，**非**媒体级）：

| 位置 | 摘录 |
|---|---|
| `Telegram/SourceFiles/history/history_item.cpp:3136-3138` | `bool HistoryItem::forbidsForward() const { return false; }` |
| `Telegram/SourceFiles/history/history_item.cpp:3140-3147` | `bool HistoryItem::forbidsSaving() const { if (forbidsForward()) return true; /* invoice 分支已注释 */ return false; }` → **恒 false** |

> 复核：`git diff Telegram/SourceFiles/history/history_item.cpp` 未触及 `forbidsForward/forbidsSaving` 区域（diff 只含 unsupported-text 相关三处 hunk），说明这两处恒假是 fork 基线（v7.0.9 AyuGram）既有状态，不是本会话/本批次改动。

### 2.1 (a) 转发按钮 / 菜单可见性

| # | 门（file:line） | 判定表达式 | 是否读媒体闸门 | 解锁后结果 |
|---|---|---|---|---|
| A1 | `history/view/history_view_context_menu.cpp:420-429` | `else if (!item \|\| !item->allowsForward()) return false;` | 间接（`allowsForward()` 内含 `3068`） | 菜单项出现；仅 `MediaCall` 服务消息不出现（本来也不该出现） |
| A2 | `history/view/history_view_context_menu.cpp:432-437` | `if (!ranges::all_of(group->items, &HistoryItem::allowsForward)) return false;` | 间接 | 相册组同样放开 |
| A3 | `history/view/history_view_context_menu.cpp:398-403` | `if (!ranges::all_of(request.selectedItems, &SelectedItem::canForward)) return false;` | 间接（`canForward` 源见 A4） | 多选转发放开 |
| A4 | `history/view/history_view_list_widget.cpp:1549-1561`（`1555`） | `iterator->second.canForward = item->allowsForward();` | 间接 | 每项 `canForward=true`（`MediaCall` 项为 false） |
| A5 | `history/history_inner_widget.cpp:3538-3542`、`3848-3852` | `if (selectedState.count > 0 && selectedState.canForwardCount == selectedState.count) addAction(lng_context_forward_selected…)` | 间接（`canForwardCount` 源 `5353-5362`） | 全选可转发时出现"转发所选" |
| A6 | `history/history_inner_widget.cpp:5352-5363` | `if (item->allowsForward()) ++result.canForwardCount;` | 间接 | 计数放开 |
| A7 | `history/history_inner_widget.cpp:3639` + `3874-3890` | `const auto canForward = item && item->allowsForward();` … `if (canForward) addAction(lng_context_forward_msg…)` | 间接 | 单条转发项出现 |
| A8 | `history/view/history_view_top_bar_widget.cpp:1559-1581` | `auto canForward = (state.count > 0 && state.count == state.canForwardCount);` | 间接 | 顶栏转发按钮出现 |
| A9 | `history/view/history_view_list_widget.cpp:4703-4714`（拖拽 MIME） | `if (!data.canForward) return false;` → `collectSelectedIds()` | 间接 | 可拖拽转发 |
| A10 | `history/history_inner_widget.cpp:2459-2466`、`2474-2477` | `if (item && item->allowsForward()) forwardIds = …` | 间接 | 拖拽/Date 光标转发放开 |
| A11 | `history/view/history_view_list_widget.cpp:4727-4740` | `if (_overElement->data()->allowsForward())` / `if (exactItem->allowsForward())` / `pressedView->data()->allowsForward()` | 间接 | 分组/媒体拖拽转发放开 |
| A12 | `history/view/history_view_list_widget.cpp:5918-5934`（`5924`） | `if (!item.canForward) return;` | 间接 | 选中转发静默失败点（唯一"什么都没发生"的路径） |
| A13 | `media/view/media_view_overlay_widget.cpp:2160` | `if (_message && _message->allowsForward()) addAction(lng_mediaview_forward…)` | 间接 | 媒体查看器"转发"可见 |
| A14 | `media/view/media_view_overlay_widget.cpp:3430-3435` | `const auto id = (_message && _message->allowsForward()) ? _message->fullId() : FullMsgId();` | 间接 | 同上 |
| A15 | `history/history_widget.cpp:10186-10192` | `else if (item->allowsForward()) { ShowReplyToChatBox(…) } else { showToast(lng_error_cant_reply_other) }` | 间接 | 不再弹"无法回复"提示 |
| A16 | `history/view/controls/history_view_draft_options.cpp:860-869` | `if (item->allowsForward()) { AddButtonWithIcon(lng_reply_in_another_chat…) }` | 间接 | "在另一聊天回复"按钮出现 |
| A17 | `history/view/history_view_message.cpp:5878-5880`（快速分享） | `if (!item->allowsForward()) return false;` | 间接 | 快速分享按钮放开 |
| A18 | `ayu/ui/context_menu/context_menu.cpp:812`（Ayu"重发消息"） | `… \|\| !item->allowsForward() \|\| item->id <= 0) return;` | 间接 | Ayu 重传入口放开（`MediaCall` 项仍不出现） |
| A19 | `core/click_handler_types.cpp:664-667` | `item && !item->forbidsForward() && item->history()->peer->allowsForwarding()` | 否（只读 peer + 消息级） | 日期链接"设置提醒"放开 |
| A20 | `info/*` 选择态转发：`info/info_top_bar.cpp:822`、`info/media/info_media_list_widget.cpp:1220-1222`、`info/profile/info_profile_top_bar.cpp:2276`、`info/saved/info_saved_music_provider.cpp:303`（硬编码 `true`） | 全部读 `item->allowsForward()` | 间接 | 资料页/媒体页选择转发放开 |
| A21 | `overview/overview_layout.cpp:1384-1387` | `parent()->allowsForward()`（音频角标下载判断，非转发按钮） | 间接 | 无影响 |

### 2.2 (b) 复制限制

| # | 门（file:line） | 判定表达式 | 是否读媒体闸门 | 解锁后结果 |
|---|---|---|---|---|
| B1 | `history/view/history_view_list_widget.cpp:5972-5982`（`CopyRestrictionTypeFor`） | `(peer->allowsForwarding() && (!item \|\| !item->forbidsForward())) ? None : …` | **否** | **恒 `None`**（peer 解锁 + `forbidsForward()` 恒 false） |
| B2 | `history/view/history_view_list_widget.cpp:5984-6000`（`CopyMediaRestrictionTypeFor`） | 先 `CopyRestrictionTypeFor`；再 `!item->forbidsSaving() ? None : …` | **否** | **恒 `None`**（`forbidsSaving()` 恒 false） |
| B3 | `history/history_inner_widget.cpp:4033-4035`（`hasCopyRestriction`） | `! _peer->allowsForwarding() \|\| (item && item->forbidsForward())` | **否** | 恒 false（无限制） |
| B4 | `history/history_inner_widget.cpp:4037-4040`（`hasCopyMediaRestriction`） | `hasCopyRestriction(item) \|\| item->forbidsSaving()` | **否** | 恒 false |
| B5 | `history/view/history_view_list_widget.cpp:1801-1809`（`hasCopyRestriction` / `hasCopyMediaRestriction`） | 同 B1/B2 语义（`CopyRestrictionTypeFor` / `CopyMediaRestrictionTypeFor` 的封装） | **否** | 恒 false |
| B6 | `history/view/history_view_list_widget.cpp:1837-1854`（`hasCopyRestrictionForSelected`） | 遍历 `forbidsForward()` | **否** | 恒 false |
| B7 | `history/history_inner_widget.cpp:4066-4078`（`hasCopyRestrictionForSelected`） | 同上 | **否** | 恒 false |
| B8 | 复制类菜单/动作消费点：`history_inner_widget.cpp:2408, 2729, 3200, 3238, 3250, 3722, 4130, 4175`；`history_view_context_menu.cpp:202, 291, 340, 1000, 1547, 1553, 1595, 1599, 1629`；`history_view_save_document_action.cpp:122`；`iv/iv_rich_message_html_export.cpp:1774`；`list_widget.cpp:3291, 4390, 4673` | 全部问 B1–B7 | **否** | 复制/另存为图片/导出 HTML 全部放开 |
| B9 | `media/view/media_view_overlay_widget.cpp:1311-1319`（`hasCopyMediaRestriction`） | 仅 `story->call()` 返回 true，随后 `// AyuGram: removed; allow downloading any stories` `return false;` | **否** | 媒体查看器复制/下载放开（唯一例外：故事通话，与消息媒体无关） |
| B10 | `history/view/history_view_chat_section.cpp:3632`、`history_view_pinned_section.cpp:772`、`history_view_scheduled_section.cpp:1615`、`settings/business/settings_shortcut_messages.cpp:984` | 转发到 B1/B2 | **否** | 各上下文复制限制为 `None` |

**注意（与任务书的一处出入）**：任务书所述"`history_view_list_widget.cpp:5972-5981`"精确范围是 **`5972-5982`**（`5972` 函数签名，`5975-5981` 三元体，`5982` 收尾花括号）；内容与"恒 None"的判断一致。

### 2.3 (c) 下载 / 保存限制

| # | 门（file:line） | 判定表达式 | 是否读媒体闸门 | 解锁后结果 |
|---|---|---|---|---|
| C1 | `history/history_item.cpp:3140-3147`（`forbidsSaving`） | `if (forbidsForward()) return true; /* invoice 分支已注释 */ return false;` | **否** | **恒 false** → 保存/下载永不因限制被拦 |
| C2 | `menu/menu_item_download_files.cpp:44-59`（`Added`） | `if (item && !item->forbidsForward()) { … photos/documents.emplace_back … }` | **否** | 只要有 photo/document 就收进"保存到下载" |
| C3 | `menu/menu_item_download_files.cpp:235` / `267`（`AddDownloadFilesAction` 两个重载） | 由 C2 提供数据 | **否** | 菜单项出现 |
| C4 | 调用点：`history_inner_widget.cpp:3548-3553`、`3858-3863`、`history_view_context_menu.cpp:994-1003`、`1142` | 前置 `!hasCopyRestrictionForSelected()`（恒 false） | **否** | 选中/单条导出均放开 |
| C5 | `history/view/media/history_view_save_document_action.cpp:118-125` | `if (!item \|\| list->hasCopyMediaRestriction(item) \|\| ItemHasTtl(item)) return;` | **否**（B4 恒 false） | "另存为"放开；**仅 TTL 自毁媒体**（`ItemHasTtl`，与媒体闸门无关）仍被挡 |
| C6 | 下载执行面 | `grep -rn "forbidsSaving" Telegram/SourceFiles/` 仅命中 `history_item.cpp:3140,3141`、`history_inner_widget.cpp:4039`、`history_view_list_widget.cpp:5991` —— **下载管理器/文件保存路径上没有任何限制检查** | **否** | 实际下载链路无 gate |
| C7 | `media/view/media_view_overlay_widget.cpp:1311-1319` | 见 B9 | **否** | 媒体查看器下载放开 |

**结论 (c)**：`forbidsSaving()` 恒 false 且全仓仅 2 个消费点，下载/保存**不存在**任何媒体级残留闸门。

### 2.4 矩阵总览（一页速览）

| 门 | 判定函数 | 依据 | 含媒体闸门？ | `MediaCall` 项 | 普通媒体项（解锁后） |
|---|---|---|---|---|---|
| 转发菜单（单条） | `AddForwardMessageAction` `cm:420-437` | `item->allowsForward()` | 间接 | 不可转发（因 `isService`） | ✅ 可转发 |
| 转发菜单（多选） | `AddForwardSelectedAction` `cm:396-403` + `lw:1555` | `SelectedItem::canForward` | 间接 | 不可转发 | ✅ 可转发 |
| 转发（选中/顶栏） | `inner:3538/3848`、`top_bar:1559` | `canForwardCount` | 间接 | 不可转发 | ✅ 可转发 |
| 转发（拖拽） | `lw:4704-4713`、`inner:2459-2474`、`lw:4727-4740` | `canForward` / `allowsForward()` | 间接 | 不可拖拽转发 | ✅ 可拖拽 |
| 媒体查看器转发 | `overlay:2160`、`3430` | `allowsForward()` | 间接 | n/a（无照片/文件） | ✅ 可见 |
| Ayu 重传入口 | `ayu context_menu:812` | `allowsForward()` | 间接 | 入口不显示 | ✅ 显示 |
| **复制限制** | `lw:5972-6000`、`inner:4033-4040` | `allowsForwarding()` + `forbidsForward/Saving()` | **否** | 无限制 | ✅ 无限制 |
| **下载/保存限制** | `menu_item_download_files:47`、`save_document_action:122`、`item:3140-3147` | `forbidsForward()` / `forbidsSaving()` | **否** | TTL 除外无限制 | ✅ 无限制 |

---

## 3. 在 AyuForward 重传路径下会发生什么

AyuForward 的两个分支（`apiwrap.cpp:3829-3848`）：

| 位置 | 摘录 |
|---|---|
| `Telegram/SourceFiles/apiwrap.cpp:3834-3840` | `const auto fullAyuForward = AyuForward::isFullAyuForwardNeeded(draft.items.front()); if (fullAyuForward) { crl::async([=] { AyuForward::forwardMessages(_session, action, false, draft); }); return; }` |
| `Telegram/SourceFiles/apiwrap.cpp:3842-3848` | `const auto ayuIntelligentForwardNeeded = AyuForward::isAyuForwardNeeded(draft.items); if (…) { AyuForward::intelligentForward(…) }` |
| `Telegram/SourceFiles/ayu/features/forward/ayu_forward.cpp:236-245` | `isAyuForwardNeeded(item)` = `isDeleted() \|\| item->isAyuNoForwards() \|\| item->unsupportedTTL() \|\| (media && media->ttlSeconds())`；`isFullAyuForwardNeeded(item)` = `from()->isAyuNoForwards() \|\| history()->peer->isAyuNoForwards()` |
| `Telegram/SourceFiles/ayu/features/forward/ayu_forward.cpp:302-314` | 分块后：`isAyuForwardNeeded` 的块走 `forwardMessages(...)`（重下载再上传），其余块走 `AyuSync::forwardMessagesSync(...)` → `session->api().forwardMessages(...)`（服务端原样转发） |
| `Telegram/SourceFiles/ayu/features/forward/ayu_forward.cpp:420-437` | 重传循环体：`auto extractedText = extractText(item); if (extractedText.empty() && !mediaDownloadable(item->media())) continue;` … `if (!mediaDownloadable(item->media())) AyuSync::sendMessageSync(...)` |
| `Telegram/SourceFiles/ayu/utils/telegram_helpers.cpp:1019-1029` | `mediaDownloadable()` 对 `media->call()` 明确返回 **false** |
| `Telegram/SourceFiles/apiwrap.cpp:3860-3890`（`forwardMessages` 主体） | 服务端转发路径：`SendExistingDocument`/后续 `MTPmessages_forwardMessages`；**全程无 `allowsForward()` / 媒体闸门检查** |

由此得到 AyuForward 路径下的三种情形：

| 情形 | 结果 | 证据 |
|---|---|---|
| 普通媒体（photo/document/video…） | 下载后作为新消息重发（媒体级闸门为 `true`，不参与） | `ayu_forward.cpp:438-470`、`telegram_helpers.cpp:1019-1029` |
| `MediaCall` 项 | 因 `isAyuNoForwards()==false`、无 TTL，通常**不会**进入"重传"块而走服务端转发；若被强制塞进重传块，则 `mediaDownloadable(call)==false` 且 `extractText(call).text` 为空 → `continue` **静默跳过**（既不发送也不报错） | `ayu_forward.cpp:421-423`、`telegram_helpers.cpp:1024` |
| 强制调用发送 API | `ApiWrap::forwardMessages` 层无媒体闸门；是否被服务端拒绝 → **【无法确定】**（无代码证据，需实测/服务端行为） | `apiwrap.cpp:3829-3890` |

---

## 4. 消息级 flags 与媒体级 gate 的区分（勿混淆）

| 层次 | 载体 | 位置 | 当前取值 | 作用 |
|---|---|---|---|---|
| **peer 级** | `PeerData::allowsForwarding()` → user/channel/chat | `data_peer.cpp:1728-1738`；`data_user.cpp:690`；`data_channel.cpp:787`；`data_chat.cpp:75` | **全部 `true`**（本需求已放开） | 决定"这个聊天是否整体禁止转发" |
| **消息级** | `HistoryItem::forbidsForward()` | `history_item.cpp:3136-3138` | **恒 `false`** | 上游用来读 `MessageFlag::NoForwards`，此处被钉死 |
| **消息级（保存）** | `HistoryItem::forbidsSaving()` | `history_item.cpp:3140-3147` | **恒 `false`**（invoice 分支注释） | 保存/下载限制 |
| **消息级（Ayu 变体）** | `MessageFlag::AyuNoForwards` + `HistoryItem::isAyuNoForwards()` | `data_types.h:273`；`history_item.cpp:2147-2149`；MTP 位映射 `history_item_helpers.cpp:931-933`（`// AyuGram: removed` 旧 `NoForwards`，改映射到 `AyuNoForwards`） | 由 `MTP::f_noforwards` 位驱动 | **不再参与 `forbidsForward()`**，只驱动 AyuForward 的重传策略与"不可转发"提示 |
| **peer 级（Ayu 变体）** | `PeerData::isAyuNoForwards()` | `data_peer.cpp:1720-1728`；`data_chat.h:27`（`AyuNoForwards = (1 << 31)`）；`data_channel.h:93`（`1ULL << 63`）；`data_session.cpp:923,933,1046,1096` | 由 `chat.is_noforwards()` / `channel.is_noforwards()` 驱动 | 影响导出历史 `data_peer.cpp:770-774`、`edit_peer_info_box`、`window_peer_menu.cpp:1557` 等 |
| **媒体级** | `Data::Media::allowsForward()` 家族 | `data_media_types.cpp:734/1385/1836` | 基类 `true`、`MediaFile` `true`、**`MediaCall` `false`** | 仅被 `history_item.cpp:3068` 消费 |

**关键点**：`AyuForward` 判定用的是 `isAyuNoForwards()`（peer/message 的 Ayu 标记）与 TTL，**从不读 `_media->allowsForward()`**；反之 `_media->allowsForward()` 的唯一消费者是 `HistoryItem::allowsForward()`。两条链路正交，不构成"双重闸门"。

Ayu 层仍会显示的"不可转发"提示（**不是**媒体闸门）：
- `history/view/history_view_context_menu.cpp:2657-2685` `AddSelectRestrictionAction`：当 `peer->isAyuNoForwards() || AyuForward::isAyuForwardNeeded(item)` 时插入 `tr::ayu_UnforwardableContextMenuText` 说明条。
- `data_peer.cpp:770-780` `canExportChatHistory()`：`isRepliesChat() || isVerifyCodes() || !allowsForwarding()` 或 `isAyuNoForwards()` 时禁止导出历史（这是"对话导出"，不是转发）。

---

## 5. 结论（回答任务书第 3 点）

> 在 peer 级解锁 + AyuForward 重传路径下，`_media->allowsForward()==false` 的媒体是否仍会出现在"不能转发/不能复制"的 UI 或发送失败里？

**不会。** 三点理由（均有上文 file:line 证据）：

1. 全仓 `_media->allowsForward()` 为 `false` 的**唯一**媒体类型是 `MediaCall`（`data_media_types.cpp:1836-1838`），而携带它的 item 全部是 `MTPDmessageService`（`history_item.cpp:605-627`、`2446+2486`），已经先被 `HistoryItem::allowsForward()` 的 `!isService()`（`3064`）挡住 —— 媒体闸门在此处**冗余**，删掉行为不变。
2. 对任何非服务消息，`(!_media || _media->allowsForward())`（`3068`）恒为 `true`：基类 `true`（`734-736`）、`MediaFile` `true`（`1385-1387`），无第三处。
3. 复制门（`lw:5972-6000`、`inner:4033-4040`）、下载门（`menu_item_download_files:47`、`save_document_action:122`、`item:3140-3147`）**完全不读**媒体闸门，只读 peer 级 `allowsForwarding()` 与恒假的 `forbidsForward()/forbidsSaving()` → 全部 `None`/放行。

**残留的"不能转发"体验只有两类，均非媒体闸门**：
- 服务消息（含通话记录）本身不可转发（`3064`）；
- Ayu 层的 `isAyuNoForwards()` 场景（`ayu context_menu:2663` 说明条、`ayu_forward.cpp:236-245` 触发重传）。

**发送失败**：`ApiWrap::forwardMessages` 与 `AyuForward::forwardMessages/intelligentForward` 均无媒体闸门检查（`apiwrap.cpp:3829-3890`、`ayu_forward.cpp:253-330`、`340-470`）。`MediaCall` 若被强行塞进重传路径会因 `extractText` 为空 + `mediaDownloadable(call)==false` 被 `continue` **静默跳过**（`ayu_forward.cpp:421-423`、`telegram_helpers.cpp:1024`），既不是"发送失败"，也没有错误提示。服务端是否接受转发通话消息 → **【无法确定】**。

### 5.1 最小修复点（仅建议，**未落地任何修改**）

若目标只是"HANDOFF §6 残留风险取证结案" → **无需改动代码**，`_media->allowsForward()` 在当前树中不构成任何拦截路径。

若要求"代码层面彻底没有媒体闸门"（可读性/防御性），最小改动二选一：

**方案 A（删子句，推荐）** — `Telegram/SourceFiles/history/history_item.cpp:3063-3069`

```diff
 bool HistoryItem::allowsForward() const {
 	return !isService()
 		&& isRegular()
 		&& !forbidsForward()
-		&& history()->peer->allowsForwarding()
-		&& (!_media || _media->allowsForward());
+		&& history()->peer->allowsForwarding();
 }
```

**方案 B（保留结构，改唯一 false 源）** — `Telegram/SourceFiles/data/data_media_types.cpp:1836-1838`

```diff
 bool MediaCall::allowsForward() const {
-	return false;
+	return true;
 }
```

风险说明：方案 B 只影响 `history_item.cpp:3068` 这一处消费（全仓 `media->allowsForward()` 唯一调用点），且通话 item 仍被 `!isService()` 挡住，行为等价；若将来出现"非服务消息携带 `MediaCall`"（例如编辑路径新增），方案 B 会让其变为可转发。方案 A 语义更明确，但删掉的是上游 API 契约里的一层扩展点。二者**均不改变**复制/下载门（它们本来就不读媒体闸门）。

---

## 6. 证据纪律 / 无法确定项 / 命令记录

**已复核并修正任务书中的行号偏差（不猜测）**：

| 任务书原述 | 实际 | 说明 |
|---|---|---|
| `history_item.cpp:3063-3069` | ✅ 一致 | `allowsForward()` 定义 |
| `history_item.cpp:3140-3147` | ✅ 一致 | `forbidsSaving()`；`forbidsForward()` 在 `3136-3138` |
| `history_view_list_widget.cpp:5972-5981` | 实际 **`5972-5982`** | `CopyRestrictionTypeFor` 三元体 `5975-5981`，`5982` 为收尾花括号 |
| `history_inner_widget.cpp 约 3979-3981` | ❌ **不符**：`3979-3981` 属 poll 菜单（`InsertPollVoteRestrictionsLabel` 区域），**不是**转发门 | 真正的转发门在 `3639`（`canForward` 计算）、`3874-3890`（单条"转发"项）、`3538`/`3848`（"转发所选"）、`5352-5363`（`canForwardCount`） |
| `history_view_context_menu.cpp 约 427 / 434` | ✅ 一致 | `427` 单条门、`434` 相册组门 |

**【无法确定】**：
1. 服务端对"强制转发通话媒体消息"（绕过所有客户端门，直接 `messages.forwardMessages`）是否报错 —— 无代码证据，需真实账号实测；已查范围：`apiwrap.cpp` 全量 `forwardMessages` 路径、`ayu/features/forward/*`、`mtproto/` 响应处理，均无相关分支。
2. `history_view_context_menu.cpp:427` 上游（官方 tdesktop）是否也含媒体闸门 —— 本报告只对当前 fork HEAD 取证，未做上游对比（`git log` 仅到本仓库）。

**命令记录（全部只读）**：
- `grep -rn "allowsForward" Telegram/SourceFiles/`
- `grep -rn "forbidsForward\|forbidsSaving" Telegram/SourceFiles/`
- `grep -rn "hasCopyRestriction\|hasCopyMediaRestriction\|showCopyRestriction\|showCopyMediaRestriction" Telegram/SourceFiles/`
- `grep -rn "AddDownloadFilesAction" Telegram/SourceFiles/`
- `grep -rn "CopyRestrictionTypeFor\|CopyMediaRestrictionTypeFor" Telegram/SourceFiles/`
- `grep -rn "canForward\b" Telegram/SourceFiles/`（排除 `canForwardCount`）
- `grep -rn "mediaDownloadable\|isAyuNoForwards\|intelligentForward\|forwardMessagesSync" Telegram/SourceFiles/`
- `git diff Telegram/SourceFiles/history/history_item.cpp`（确认 `forbidsForward/Saving` 非本批次改动）
- 定点 `sed -n` 读取上述所有 file:line（见各表摘录）
