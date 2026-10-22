# P2 实现方案 — 需求 6：unsupported 消息显示部分文本

- 文档类型：**只做设计，不含应用代码**。产出物本身是唯一被写入的文件。
- 关联需求：`CONTEXT.md:21` 第 6 行「突破"请更新最新版本"消息限制」。
- 已定稿决策：`docs/adr/0002-unlock-restricted-chats.md:23`（需求 6 决策 4）。
- 证据基线：AyuGramDesktop = tdesktop v7.0.9 fork；`Telegram/build/version` = 7.0.9；API layer 228。
- 标注约定：**【确定】** = 本次审计在代码中逐行核实；**【无法确定】** = 检索无果或依赖未 checkout 的子模块，附命令与范围。

---

## 0. 结论速览

| # | 结论 | 判定 |
|---|---|---|
| 1 | Unsupported 分支丢弃的 `vmessage` / `ventities` 字段**确实已到手**（`message:string` 在 api.tl 中必填），可零成本捡回 | 【确定】 |
| 2 | `MessageFlag::Legacy` 在全仓**只有一个消费点**：`allowsEdit()` 禁编辑 | 【确定】 |
| 3 | `UnsupportedMessageText()` 全仓只有 3 个调用点，全在 `history_item.cpp`，无渲染层直接依赖 | 【确定】 |
| 4 | Story 的 unsupported 分支与机制 A **不同构**（`v::null` 哨兵 + 独立文案，不设 Legacy） | 【确定】 |
| 5 | 本 fork **没有本地消息搜索索引**，搜索是服务端 `messages.search` → 改文本对搜索零影响 | 【确定】 |
| 6 | Unsupported 分支比正常分支**少做 4 件事**：`setGroupId` / `setReactions` / `applyTTL` / factcheck 组件 | 【确定】 |
| 7 | `UnsupportedMessageMarkup()` 会**无条件覆盖**服务端真实 `reply_markup`，且转发副本会带着这个假按钮 | 【确定】 |
| 8 | `Unsupported` 有**两个**判定入口：`messageMediaUnsupported`（`:1109`）**和** `webPageNotModified`（`:1082-1083`） | 【确定】 |
| 9 | 推荐默认：**形态 F（摘要优先 → 原文 → 占位兜底）**，**保留 Legacy**，提示标记走文本后缀而非 `setAyuHint` | 【确定】 |

---

## 1. 本次跑过的命令 / 检索（审计只读，未改任何 `Telegram/` 下文件、未 git、未构建）

```
# 结构 / 存在性
ls Telegram/SourceFiles/core/ ; find Telegram/SourceFiles -type d -name "search*"
ls -la Telegram/ | grep lib ; ls -A Telegram/lib_ui | wc -l
find . -name "text_entity.h" ; find Telegram/SourceFiles -name "history_view_sections.*"

# 符号检索（范围：Telegram/SourceFiles/** 与仓库根）
grep -rn "UnsupportedMessageText\|UnsupportedMessageMarkup"        Telegram/SourceFiles
grep -rn "FlattenRichPageSummary\|ParseRichPage"                   Telegram/SourceFiles
grep -rn "MessageFlag::Legacy\|isLegacyMessage"                    . --include=*.cpp --include=*.h --include=*.mm
grep -rn "f_legacy\|is_legacy"                                     Telegram/SourceFiles
grep -rn "EntitiesFromMTP"                                         Telegram/SourceFiles
grep -rn "MTPDmessageMediaUnsupported"                             Telegram/SourceFiles
grep -rn "computeUnavailableReason"                                Telegram/SourceFiles
grep -rn "emptyText"                                               Telegram/SourceFiles/history
grep -n "MediaCheckResult::Unsupported\|CheckMessageMedia"         Telegram/SourceFiles/data/data_story*.cpp

# api.tl / lang.strings
grep -n "unsupported\|Unsupported"    Telegram/Resources/langs/lang.strings
grep -n "media:MessageMedia"          Telegram/SourceFiles/mtproto/scheme/api.tl
grep -n "message:\|storyItem#\|media_unread" Telegram/SourceFiles/mtproto/scheme/api.tl

# 定点读文件
read history_item.cpp 400-699 / 2285-2374 / 2600-2669 / 3055-3104 / 3960-4024 / 4427-4486
    / 4538-4607 / 4670-4783 / 4800-4884 / 5304-5423
read history_item_helpers.cpp 895-984 / 1060-1139 / 1395-1434
read iv_rich_page.h 285-327 ; iv_rich_page.cpp 2420-2509
read api_text_entities.h 1-36 ; api_text_entities.cpp 65-104
read api.tl 100-179 ; history_view_element.cpp 1700-1729 / 2200-2239
read data_types.h 248-307 ; data_stories.cpp 60-119 ; CONTEXT.md ; docs/adr/0002
```

并行派发 3 个只读子审计（Story 对照 + api.tl、Legacy 消费点、文本消费管线），结论已交叉核验并并入本文件。

---

## 2. 机制 A 现状逐行复核

### 2.1 判定链（**两个**入口，都会进同一分支）⚠️

`Telegram/SourceFiles/history/history_item_helpers.cpp:1109-1111`（入口 1：`messageMediaUnsupported`）
```cpp
	}, [](const MTPDmessageMediaUnsupported &) {
		return Result::Unsupported;
	});
```
`Telegram/SourceFiles/history/history_item_helpers.cpp:1082-1083`（入口 2：`webPageNotModified` —— **容易漏掉**）【确定】
```cpp
		}, [](const MTPDwebPageNotModified &) {
			return Result::Unsupported;
		});
```
> ⚠️ 需求 6 改的是 **`:500` 一行**，两个入口共用它。`MTPDwebPageNotModified` 走同一分支时，`vmessage` 同样是有意义的 caption——规则 F 对两者一致适用。但需注意：入口 2 属于 `webPage` 分发路径（`:1075-1084`），其 `history_item.cpp:405-407` 另有 "webPageNotModified is unexpected in message media" 的 LOG——说明该入口是异常兜底，**触发概率低但非零**。

`Telegram/SourceFiles/history/history_item.cpp:487-491`
```cpp
	const auto media = data.vmedia();
	const auto checked = media
		? CheckMessageMedia(*media)
		: MediaCheckResult::Good;
	if (checked == MediaCheckResult::Unsupported) {
```

### 2.2 当前 Unsupported 分支（`history_item.cpp:491-507`，逐行）

```cpp
491: if (checked == MediaCheckResult::Unsupported) {
492:     _flags &= ~MessageFlag::HasPostAuthor;      // 清 post-author 签名位
493:     _flags |= MessageFlag::Legacy;              // 置 Legacy
494:     createComponents(data);                     // 建 reply/views/edited/signed/saved/markup…
495:     if (const auto richMessage = data.vrich_message()) {
496:         const auto richPage = Iv::ParseRichPage(&history->session(), *richMessage);
497:         setRichPage(richPage);
498:         setText(Iv::FlattenRichPageSummary(richPage));   // ← 已有「部分文本」先例
499:     } else {
500:         setText(UnsupportedMessageText());               // ← 丢弃 vmessage/ventities
501:     }
502:     if (!Has<HistoryMessageReplyMarkup>()) {
503:         AddComponents(HistoryMessageReplyMarkup::Bit());
504:     }
505:     _flags |= MessageFlag::HasReplyMarkup;
506:     Get<HistoryMessageReplyMarkup>()->updateData(
507:         UnsupportedMessageMarkup());                      // ← 覆盖服务端真实 reply_markup
508: }
```

**关键观察【确定】**：`:500` 的 `else` 分支同时丢掉 `data.vmessage()` 与 `data.ventities()`——两者都已在手（`data` 是 `const MTPDmessage&`）。

### 2.3 正常分支的普通文本构建点（对照样板）

`Telegram/SourceFiles/history/history_item.cpp:558-566`
```cpp
558:     } else if (!skipSetText) {
559:         auto textWithEntities = TextWithEntities{
560:             qs(data.vmessage()),
561:             Api::EntitiesFromMTP(
562:                 &history->session(),
563:                 data.ventities().value_or_empty())
564:         };
565:         setText(_media ? textWithEntities : EnsureNonEmpty(textWithEntities));
566:     }
```

### 2.4 Unsupported 分支**相对正常分支缺失**的 4 项【确定】

`:567-580` 全部位于 `else` 块内，Unsupported 分支**不执行**：

| 缺失动作 | 正常分支行号 | 用户可见后果 |
|---|---|---|
| `setGroupId(MessageGroupId::FromRaw(...))` | `:567-573` | 该消息不进 `grouped_id` 索引，**相册/媒体组聚合会跳过它** |
| `setReactions(data.vreactions())` | `:574` | **不显示反应**（Empty/TTL/StoryMention 分支在 `:515`/`:520` 都调了，Unsupported 是唯一漏掉的） |
| `applyTTL(data)` | `:575` | TTL 媒体（且媒体不被识别）不进入自动销毁流程 |
| `HistoryMessageFactcheck` 组件 | `:577-580` | 不显示 fact-check 标记 |

> 影响：需求 6 把"占位文案"换成"真实文本"后，用户会**开始尝试交互**（选中/复制/反应/回复），而反应/分组仍然不工作 → 规则矩阵必须把这点列进风险栏。

### 2.5 另一处覆盖：真实 `reply_markup` 被销毁【确定】

`createComponents(data)`（`history_item.cpp:5304-5336`）在 `:5327` 把服务端真实 markup 装进组件：
```cpp
5327:     config.markup = HistoryMessageMarkupData(data.vreply_markup());
```
而 `HistoryMessageReplyMarkup::updateData`（`history_item_components.cpp:1334-1338`）**无条件替换**：
```cpp
1336:     data = std::move(markup);
1337:     inlineKeyboard = nullptr;
```
→ 真实内联键盘被 `UnsupportedMessageMarkup()` 顶掉。且转发副本会带着这个假按钮：`HistoryMessageReplyMarkup::createForwarded`（`history_item_components.cpp:1327-1332`）→ `fillForwardedData`（`history_item_reply_markup.cpp:340-367`）逐行复制 `rows`。

---

## 3. Q1 — Unsupported 分支里 vmessage 实际可用的字段（逐个）

字段清单来源：`Telegram/SourceFiles/mtproto/scheme/api.tl:124`（`message#7600b9d3`）。

### 3.1 已被 Unsupported 分支消费的字段

| 字段（api.tl:124） | 消费点 | 说明 |
|---|---|---|
| `id:int` | `history_item.cpp:473` | 基类初始化列表 |
| `peer_id:Peer` | 经 `history->peer` | 构造参数 |
| `date:int` | `history_item.cpp:476` | |
| `message:string` | **无（被丢弃）** | ← 需求 6 目标 |
| `media:flags.9?MessageMedia` | `history_item.cpp:487`（触发本分支） | 零字段，见 `api.tl:131` |
| `entities:flags.7?Vector<MessageEntity>` | **无（被丢弃）** | ← 需求 6 目标 |
| `rich_message:flags2.13?RichMessage` | `history_item.cpp:495-498` | 唯一已用的"部分文本"通道 |
| `reply_markup:flags.6?ReplyMarkup` | `createComponents` `:5327` → 被 `:506-507` **覆盖** | 见 §2.5 |
| `reply_to:flags.3?MessageReplyHeader` | `createComponents` `:5314-5316` | |
| `fwd_from:flags.2?MessageFwdHeader` | `createComponents` `:5309-5313` | |
| `views` / `forwards` / `replies` | `createComponents` `:5322-5326` | |
| `edit_date:flags.15?int` | `createComponents` `:5328` | |
| `post_author:flags.16?string` | `createComponents` `:5331`（组件被 `:492` 清位影响） | |
| `restriction_reason:flags.22?Vector<...>` | `createComponents` `:5332-5333` | |
| `from_id:flags.8?Peer` | `history_item.cpp:475` | |
| `from_boosts_applied:flags.29?int` | `history_item.cpp:482` | |
| `from_rank:flags2.12?string` | `history_item.cpp:583-588`（在分支**之后**） | |
| `quick_reply_shortcut_id:flags.30?int` | `history_item.cpp:478` | |
| `schedule_repeat_period:flags2.10?int` | `history_item.cpp:477` | |
| `paid_message_stars:flags2.6?long` | `history_item.cpp:479` | |
| `effect:flags2.2?long` | `history_item.cpp:480` | |
| `report_delivery_until_date:flags2.5?int` | `history_item.cpp:590-594` | |
| `saved_peer_id:flags.28?Peer` | `createComponents` `:5306-5308` | |
| `via_bot_id:flags.11?long` | `createComponents` `:5317` | |
| `via_business_bot_id:flags2.0?long` | `createComponents` `:5318` | |
| `guestchat_via_from:flags2.19?Peer` | `createComponents` `:5319-5321` | |
| `suggested_post:flags2.7?SuggestedPost` | `createComponents` `:5334` | |
| 全部 `flags:#` / `flags2:#` 位 | `FlagsFromMTP` `history_item_helpers.cpp:905-949`（由 `:474` 调用） | 含 `legacy:flags.19?true` → `Flag::Legacy`（`:918`） |

### 3.2 未被消费的字段（分三类）

| 字段 | 正常分支行号 | 缺失原因 |
|---|---|---|
| `message:string`、`entities` | `:559-564` | `:500` 走 `UnsupportedMessageText()` |
| `grouped_id:flags.17?long` | `:567-573` | 在 `else` 块内 |
| `reactions:flags.20?MessageReactions` | `:574` | 在 `else` 块内 |
| `ttl_period:flags.25?int` | `:575` | 在 `else` 块内 |
| `factcheck:flags2.3?FactCheck` | `:577-580` | 在 `else` 块内 |

### 3.3 `Api::EntitiesFromMTP` 精确签名与用法

- 声明：`Telegram/SourceFiles/api/api_text_entities.h:23-25`
  ```cpp
  [[nodiscard]] EntitiesInText EntitiesFromMTP(
      Main::Session *session,
      const QVector<MTPMessageEntity> &entities);
  ```
- 定义：`Telegram/SourceFiles/api/api_text_entities.cpp:74-79`（空 vector 直接返回 `{}`）
  ```cpp
  EntitiesInText EntitiesFromMTP(
          Main::Session *session,
          const QVector<MTPMessageEntity> &entities) {
      if (entities.isEmpty()) {
          return {};
      }
  ```
- **在 Unsupported 分支内的标准用法**（照抄 `:561-563`）：
  ```cpp
  Api::EntitiesFromMTP(&history->session(), data.ventities().value_or_empty())
  ```
- 同构先例（同样带 `media:MessageMedia` 的构造器）：`Telegram/SourceFiles/api/api_updates.cpp:2298-2303`（`updateServiceNotification`，`api.tl:337`）
  ```cpp
  2300:         const auto text = TextWithEntities {
  2301:             qs(d.vmessage()),
  2302:             Api::EntitiesFromMTP(&session(), d.ventities().v)
  2303:         };
  ```
- 其它参考调用点：`history_item.cpp:2570`、`:2583`；`history_item_components.cpp:459`、`:493`；`data_drafts.cpp:120`；`inline_bot_result.cpp:206`、`:275`。

---

## 4. Q2 — `Iv::FlattenRichPageSummary`

- 声明：`Telegram/SourceFiles/iv/iv_rich_page.h:317-322`
  ```cpp
  317: [[nodiscard]] TextWithEntities FlattenRichPageSummary(
  318:     const RichPage &page,
  319:     bool emptyFallback = true);
  320: [[nodiscard]] TextWithEntities FlattenRichPageSummary(
  321:     const std::shared_ptr<const RichPage> &page,
  322:     bool emptyFallback = true);
  ```
- 定义：`Telegram/SourceFiles/iv/iv_rich_page.cpp:2462-2502`（引用版重载在 `:2496-2502`，`page == nullptr` 时返回**空** `TextWithEntities`）
- **返回类型：`TextWithEntities`**（不是 QString）——可直接喂给 `setText()`。【确定】
- 内部行为【确定】：`:2468-2478` 遍历 `page.blocks` 调 `AppendSummaryBlock`；`:2479` `TextUtilities::Trim`；`:2480-2489` 单表格特判；`:2490-2492` 空则回退
  ```cpp
  2490:     if (result.empty() && emptyFallback) {
  2491:         result = TextWithEntities::Simple(tr::lng_message_empty(tr::now));
  2492:     }
  ```
  `lng_message_empty` = `"Empty Message"`（`Telegram/Resources/langs/lang.strings:2529`）。
- **在 Unsupported 分支里当前怎么用**（`history_item.cpp:495-498`）：仅当 `data.vrich_message()` 存在时，解析 → `setRichPage` → 用摘要当正文。`setRichPage`（`:4477-4480`）会额外 `AddComponents(HistoryMessageMediaForInstantView::Bit())`，即**已带 Instant View 入口**。
- 其它调用点（正常分支 `:554-557`、`:2322-2323`、`:2634`、`:2649`、`history_streamed_drafts.cpp:98`、`:254-255` 等）——语义一致，**唯一差别是 `emptyFallback` 默认值都是 `true`**。

> 风险【确定】：若 rich_message 存在但 blocks 为空，`FlattenRichPageSummary` 返回 `"Empty Message"`，**不会**回退到 `vmessage`。规则矩阵必须为此单设一档（形态 D）。

---

## 5. Q3 — `UnsupportedMessageText()` 全部使用点与渲染路径影响

### 5.1 定义

`Telegram/SourceFiles/history/history_item_helpers.cpp:1404-1413`
```cpp
1404: [[nodiscard]] TextWithEntities UnsupportedMessageText() {
1405:     const auto siteLink = u"https://t.me/AyuGramReleases"_q;
1406:     auto result = TextWithEntities{
1407:         tr::lng_message_unsupported(tr::now, lt_link, siteLink).replace("Telegram", "AyuGram")
1408:     };
1409:     TextUtilities::ParseEntities(result, Ui::ItemTextNoMonoOptions().flags);
1410:     result.entities.push_front(
1411:         EntityInText(EntityType::Italic, 0, result.text.size()));
1412:     return result;
1413: }
```
声明：`history_item_helpers.h:102`。文案源 `Telegram/Resources/langs/lang.strings:2530`：
```
"lng_message_unsupported" = "This message is not supported by your version of Telegram. Please update to the latest version in Settings > Advanced, or install it from {link}. If you are already using the latest version, this message might depend on a feature that is not yet implemented.";
```
（该 key 含**唯一**的 `{link}` 占位符；`history_item_helpers.cpp:1407` 做了 `.replace("Telegram", "AyuGram")`。）

### 5.2 全仓 callers（`grep -rn "UnsupportedMessageText" Telegram/SourceFiles` → 7 命中）【确定】

| # | 位置 | 上下文 | 是否受影响 |
|---|---|---|---|
| 1 | `history_item_helpers.h:102` | 声明 | — |
| 2 | `history_item_helpers.cpp:1404` | 定义 | — |
| 3 | **`history_item.cpp:500`** | 构造 Unsupported 分支 | **本次改动点** |
| 4 | `history_item.cpp:2325` | `applyEdition`：`edition.richPage ? Flatten : (mediaCheck==Unsupported) ? UnsupportedMessageText() : …`（`:2322-2328`） | **必须同步改**，否则编辑回流会把文本打回占位 |
| 5 | `history_item.cpp:2637` | `updateSentContent` Unsupported 分支（`:2626-2639`） | **必须同步改** |
| 6 | `history_item_helpers.h:103` / `history_item.cpp:507`/`:2347`/`:2639` | `UnsupportedMessageMarkup` 声明 / 3 处调用 | 与文本无关，单列 |

`history_item.cpp:2322-2328` 完整优先级链（与 ctor 同构）：
```cpp
2322:     auto updatedText = edition.richPage
2323:         ? Iv::FlattenRichPageSummary(edition.richPage)
2324:         : (mediaCheck == MediaCheckResult::Unsupported)
2325:         ? UnsupportedMessageText()
2326:         : checkedMedia
2327:         ? edition.textWithEntities
2328:         : EnsureNonEmpty(edition.textWithEntities);
```

### 5.3 其它渲染路径是否依赖占位文案【确定】

**没有任何渲染层直接引用 `UnsupportedMessageText()`。** 它只写进 `HistoryItem::_text`，其余全走 `_text` 的通用消费者：

| 消费路径 | 落点 | 读哪个访问器 | 现状 | 改后 |
|---|---|---|---|---|
| 气泡正文 | `history_view_element.cpp:2222` `setTextWithLinks(_textItem->translatedTextWithLocalEntities())` | `_text` | 占位文案 | 真实部分文本 |
| 通知正文 | `history_item.cpp:4679-4702` `notificationText()`；`:4684-4685` `else if (!emptyText()) return _text;` | `_text` | 占位文案 | 真实文本，**已自带 255 字截断**（`:90` `kNotificationTextLimit=255`，`:4697-4701` `Ui::Text::Mid` + `kQEllipsis`） |
| 会话列表预览 | `history_item.cpp:4729-4735` `toPreview()` → `Dialogs::Ui::DialogsPreviewText(_text)` | `_text` | 占位文案 | 真实文本 |
| 复制 | `history_item.cpp:3801-3804` `originalText()`（`isService()? empty : _text`）→ `history_item_text.cpp:456-462` `HistoryItemText` | `_text` | 复制到的是占位文案 | **可复制真实文本**（正向） |
| 翻译入口 | `history_view_context_menu.cpp:1612-1615` `item->originalText()` | `_text` | 翻译占位文案 | 翻译真实文本（正向） |
| 表情包收集 | `history_view_context_menu.cpp:2529` `item->originalText().entities` | `_text` | 占位 entities | 真实 custom-emoji 可被收集 |
| 回复引文 | `history_item.cpp:4800-4817` `inReplyText()` → `toPreview()` | `_text` | 占位 | 真实 |
| **媒体播放器 / 预览** | `_media` 在 Unsupported 分支**从未被赋值**（`CreateMedia` 对 `MTPDmessageMediaUnsupported` 返回 `nullptr`，`history_item.cpp:462-464`；`_media` 默认 `nullptr`，`history_item.h:823`） | — | 无媒体视图 | **仍然无媒体视图**，`_media->consumeMessageText()`（`history_item.cpp:4437`）不会被触发 |
| **本地搜索索引** | **本 fork 无本地索引**：`Telegram/SourceFiles/core/` 下**无 `search/` 目录**（`find`/`ls` 均为空）；搜索走 `Api::MessagesSearch`（`api/api_messages_search.h:30-58`）服务端 `messages.search` | — | 不索引 | **不索引，零影响**；服务端本来就索引真实 `message` |
| 限制原因覆盖 | `history_item.cpp:4586-4595` `computeUnavailableReason()` 只看 `HistoryMessageRestrictions`（`restriction_reason`），**与 Legacy/unsupported 无关**；`history_view_element.cpp:2218-2220` 有非空限制原因时**会盖掉** `_text` | — | 有限制原因时已盖 | 同左（与本需求正交） |

---

## 6. Q4 — `MessageFlag::Legacy` 全部消费点

### 6.1 定义与位值【确定】

`Telegram/SourceFiles/data/data_types.h:255-257`
```cpp
255: enum class MessageFlag : uint64 {
256:     HideEdited            = (1ULL << 0),
257:     Legacy                = (1ULL << 1),
```
完整枚举 `data_types.h:255-375`（bit0 `HideEdited` … bit63 `Ephemeral`；`AyuNoForwards` 为 fork 私有 bit17，`:273`）。尾部有 trait 与容器别名：
```cpp
376: inline constexpr bool is_flag_type(MessageFlag) { return true; }
377: using MessageFlags = base::flags<MessageFlag>;
```
- 声明方式：`enum class ... : uint64` + **每项显式位值**（非隐式自增）；**无** `Q_NAMESPACE` / `Q_ENUM_NS`（`grep -c "Q_NAMESPACE" data_types.h` → `0`）；因此**无枚举↔字符串映射表**（`grep -rn "Enum<MessageFlag>|ToString(MessageFlag|MessageFlag>("` → 0 命中）。
- **不参与任何持久化**【确定】：`Telegram/SourceFiles/history/history_item_serialization.{h,cpp}` 不存在（`glob` 无结果，`Telegram/CMakeLists.txt` 无引用）；`grep -rn "QDataStream" Telegram/SourceFiles/history/` 仅 2 处无关命中（`history_item_reply_markup.cpp:15` include、`:179` webpage 文件位置）；`grep -n "QDataStream|stream <<|<< _flags" history_item.cpp` → 0 匹配。
- `base::flags<MessageFlag>` 类体**未找到**：`find . -name "flags.h"` → 0；`Telegram/lib_base/` 为空子模块目录（见 §10）。

### 6.2 全部 8 个引用点（`grep -rn "MessageFlag::Legacy" . --include=*.cpp --include=*.h --include=*.mm`）【确定】

| 位置 | 摘录 | 语义 |
|---|---|---|
| `history_item.cpp:493` | `_flags \|= MessageFlag::Legacy;` | 构造：置位 |
| `history_item.cpp:2305` | `_flags \|= MessageFlag::Legacy;` | `applyEdition`：媒体转 Unsupported 时置位 |
| `history_item.cpp:2308-2310` | `if (_flags & MessageFlag::Legacy) { _flags &= ~MessageFlag::Legacy; }` | 恢复正常媒体时清位 |
| `history_item.cpp:2628` | `_flags \|= MessageFlag::Legacy;` | `updateSentContent` 置位 |
| `history_item.cpp:2641-2643` | `if (_flags & MessageFlag::Legacy) { _flags &= ~MessageFlag::Legacy; }` | 恢复时清位 |
| `history_item.h:707-709` | `[[nodiscard]] bool isLegacyMessage() const { return _flags & MessageFlag::Legacy; }` | 访问器 |
| **`history_item.cpp:3077`** | `&& !isLegacyMessage()` | **唯一消费点** |

`isLegacyMessage()` 定义在 **private 区**（`history_item.h:689` `private:` 之后无新访问说明符），外部无法直接调用。全仓仅 **1 个调用者**：`history_item.cpp:3077`。

### 6.3 唯一消费点上下文

`Telegram/SourceFiles/history/history_item.cpp:3065-3080`
```cpp
3065: bool HistoryItem::allowsEdit(TimeId now) const {
3066:     if (_deleted) { return false; }
3070:     const auto richPageSource = Get<HistoryMessageRichPageSource>();
3071:     const auto richPage = BestRichPage(richPageSource);
3072:     return !isService()
3073:         && canBeEdited()
3074:         && !isTooOldForEdit(now)
3075:         && (!richPage || richPageSource->canEdit)
3076:         && (!_media || _media->allowsEdit())
3077:         && !isLegacyMessage()          // ← 唯一 Legacy 消费点
3078:         && !isEditingMedia()
3079:         && (paidType() == PaidPostType::None);
3080: }
```

### 6.3.1 `allowsEdit()` 的全部调用者（= 移除 Legacy 会波及的全部用户可见点）【确定】

| file:line | 摘录 | 用户可见行为 |
|---|---|---|
| `history/history_item.cpp:3065` | `bool HistoryItem::allowsEdit(TimeId now) const {` | 定义 |
| `history/history.cpp:3802` | `if (item->allowsEdit(now)) {` | `History::lastEditableMessage()`：向上箭头向上找最后一条可编辑消息，Legacy 被跳过 |
| `history/history_inner_widget.cpp:3072` | `const auto editItem = (albumPartItem && albumPartItem->allowsEdit(t))` | 聊天区右键「编辑」（相册优先），Legacy → 无该项 |
| `history/history_inner_widget.cpp:3074` | `: item->allowsEdit(t)` | 同上，回退主 item |
| `history/view/history_view_context_menu.cpp:763` | `if (!item->allowsEdit(base::unixtime::now())) {` | ListWidget 右键「编辑」项 |
| `history/view/history_view_list_widget.cpp:5160` | `return view->data()->allowsEdit(now)` | `lastMessageEditRequestNotify()`：「编辑上一条」快捷键，Legacy 不被选中 |
| `history/history_widget.cpp:9060-9064` | ↑ 键路径 | Legacy 不被选为编辑目标 |
| `history/view/history_view_chat_section.cpp:974` / `history_view_scheduled_section.cpp:484` / `settings/business/settings_shortcut_messages.cpp:736` | 编辑上一条（Chat / Scheduled / Shortcut 三 section） | 快捷键落到更早的消息或透传给滚动 |

> **相邻但不受 Legacy 影响**【确定】：`allowsEditMedia()`（`history_item.cpp:3082-3086`）**不检查** `isLegacyMessage()`；右键「重新安排」走 `allowsReschedule()`（`history_view_context_menu.cpp:539-540`）也不经过 `allowsEdit()`。故移除 Legacy **不会**放开「重新安排」或媒体编辑。

### 6.4 保留 vs 移除 Legacy 的用户可见后果【确定】

`Legacy` **不影响**：转发（`allowsForward()` `:3051-3057`，`forbidsForward()` 在本 fork 已硬编码 `false`，`:3124-3126`）、复制（`originalText()`）、下载（`forbidsSaving()` `:3128-3135` 亦已 `false`）、删除（`canDelete()` `:3137`，不看 Legacy）、`canDeleteForEveryone()`（`:3174`，只问 `media()->allowsRevoke()`，而 `_media==nullptr` → 该检查被跳过）、置顶（`canPin()` `:3025-3036`，不看 Legacy）、重新安排（`allowsReschedule()`）、媒体编辑（`allowsEditMedia()` `:3082-3086`）、`canBeEdited()`（`:3088-3118`）、`canStopPoll()`（`:3120-3122`）、选择、搜索、通知、渲染。

`Legacy` **只影响**：编辑（`allowsEdit()` 返回 false），波及 §6.3.1 的 7 个调用点。

| 取舍 | 用户可见后果 | 评价 |
|---|---|---|
| **保留 Legacy** | 仅有"不能编辑"这一项。编辑请求本来就会在 `messages.editMessage` 侧因媒体不可解析而被服务端拒；本 fork 编辑管线还会因 `!richPage` 而拒绝（`allowsEditMedia()` `:3082-3086` 要求无 richPage，且 `_media==nullptr`）。**与 `docs/adr/0002:23` 一致，推荐保留。** | ✅ |
| 移除 Legacy | 会让 `allowsEdit()` 进入 `canBeEdited()` 分支放行，编辑器拿到的是**客户端拼出的、可能缺实体/富文本标签的 `vmessage`**，用户编辑后 `messages.editMessage` 极可能被服务端以 `MESSAGE_ID_INVALID`/`MESSAGE_NOT_MODIFIED` 拒；更糟的是**把降级文本回写服务端**。 | ❌ 不推荐 |

> 补充【确定】：即使移除 Legacy，`allowsEdit()` 里的 `&& (!richPage || richPageSource->canEdit)`（`:3075`）在 rich_message 存在时也可能拦住；但 no-rich_message 路径会被放行，风险实在。

---

## 7. Q5 — 对照机制：Story 的 unsupported 分支

**结论：不同构（DIFFERENT）。**【确定】

| 维度 | 机制 A（`message`） | Story |
|---|---|---|
| 判定 | `CheckMessageMedia()` → `MediaCheckResult::Unsupported`（`history_item_helpers.cpp:1109-1110`） | `ParseMedia()` 的 `MTPDmessageMediaUnsupported` 分支直接造哨兵（`data_stories.cpp:101-103`）`return std::make_optional(StoryMedia{ v::null });` |
| 标志 | 置 `MessageFlag::Legacy`（`history_item.cpp:493`） | **不置**（全仓 `MessageFlag::Legacy` 8 处引用无一落在 story 文件） |
| 提示文本 | `UnsupportedMessageText()`（`:500`） | **不调**；改为 `Story::caption()` 在 unsupported 时返回空（`data_story.cpp:477-479`） |
| 按钮 | `UnsupportedMessageMarkup()`（`:506-507`，`lng_update_telegram` → https://t.me/AyuGramReleases） | 独立遮罩 `Unsupported` 控件（`media/stories/media_stories_controller.cpp:901`、`:261` 用 `lng_stories_unsupported`、`:267` 用 `ayu_UpdateAyuGram`） |
| 内嵌故事块 | — | `history_view_element.cpp:2146-2151`：`storyUnsupported ? tr::lng_stories_unsupported(tr::now) : tr::lng_forwarded_story_expired(tr::now)`，斜体 |
| 缩略图 | — | `ui/dynamic_thumbnails.cpp:1084` → `EmptyThumbnail`（`dynamic_thumbnails.cpp:163`） |
| schema | `message#7600b9d3`（`api.tl:124`）：`message:string` **必填**、`media:flags.9?MessageMedia` 可选 | `storyItem#16a4b93c`（`api.tl:1730`）：`media:MessageMedia` **必填**、`caption:flags.0?string` **可选** |
| 相同点 | 唯一共同点是「丢文本」：message 丢 `message` 字段，story 丢 `caption` 字段 | |

**设计含义【确定】**：需求 6 的「展示部分文本」在 story 侧**天然更不适用**——story 的 `caption` 是可选字段，服务端常不下发，捡回来多半是空；且 story 已有 3 套专用 UI。因此规则矩阵**不把 story 纳入本次范围**，若要扩展需单独立项。

---

## 8. 规则矩阵：可选展示形态

设 `V` = `data.vmessage()`（`api.tl:124` 必填字段），`E` = `Api::EntitiesFromMTP(&history->session(), data.ventities().value_or_empty())`（`api_text_entities.cpp:74`），`R` = `data.vrich_message()` 存在时的 `Iv::FlattenRichPageSummary(Iv::ParseRichPage(&history->session(), *R))`（`iv_rich_page.cpp:2462`），`P` = `UnsupportedMessageText()`（`history_item_helpers.cpp:1404`）。

| 形态 | 触发条件 | 实现锚点 | 正文 = | 风险 |
|---|---|---|---|---|
| **A. 纯占位（现状）** | 恒定 | `history_item.cpp:500` | `P` | 用户看不到任何内容；通知/预览/复制/翻译全是"请更新"文案（已核实 §5.3） |
| **B. 无条件原文** | `!R`，`V` 非空 | `history_item.cpp:499-501` 的 `else` 改为 `setText(TextWithEntities{qs(V), E})` | `V+E` | 若 `R` 存在却被 `if` 优先，rich 摘要被吞；需处理 `V` 为空的分支（`api.tl:124` 必填≠非空，服务端可送 `""`） |
| **C. 摘要优先，否则原文** | 有 `R`→`R`；无 `R` 且 `V` 非空→`V+E` | 改 `:495-501`；同步 `:2322-2328`、`:2629-2638` | `R` 或 `V+E` | `R` 的 `FlattenRichPageSummary` 空回退成 `"Empty Message"`（`iv_rich_page.cpp:2490-2492` + `lang.strings:2529`），比原文更糟 |
| **D. 摘要优先 + 摘要空则回退原文** | `R` 非空**且** `R.text` 非空→`R`；否则 `V` 非空→`V+E`；否则 `P` | 改 `:495-501`；需要一个"摘要是否真的贡献了字符"的判据（`iv_rich_page.cpp:2466-2478` 的 `contributionCount` 是局部变量，不外露 → 【无法确定】是否有现成公开 API 判空，见 §10） | `R` / `V+E` / `P` | 比 C 多一处判据；若判据用 `text.empty()` 会误伤"只有表格标题"等合法摘要 |
| **E. 原文优先（不看 rich）** | 恒定用 `V+E` | 改 `:495-501` 为无条件 | `V+E` | **回归风险**：`:498` 现有 rich 摘要通道是官方"部分文本"设计（`CONTEXT.md:49`、ADR-0002:17 均引为先例），丢掉它等于关掉 Instant View 摘要；且 rich 消息的 `V` 往往只是一句 caption 摘要，信息量少于 `R` |
| **F. 推荐：分层 + 提示标记** | 见 §9 | 同 D | `R` → `V+E` → `P` | 提示标记若用 `setAyuHint` 会**覆盖真实 `post_author` 签名**（见 §9.3）；**两个判定入口共用本形态**（`:1109` `messageMediaUnsupported` 与 `:1082-1083` `webPageNotModified`），后者属异常兜底（`:405-407` 有 "webPageNotModified is unexpected" 的 LOG），但规则必须一致 |

### 8.1 附加维度：长度截断

| 形态 | 截断策略 | 证据 |
|---|---|---|
| 气泡正文 | **不做额外截断** | 消息文本服务端上限 4096 字符量级（`boxes/send_files_box.cpp:89` `kMaxMessageLength = 4096`），气泡原生换行/折叠；`FlattenRichPageSummary` 本身就是摘要，长度可控 |
| 通知 | 已有 255 字截断，无需新增 | `history_item.cpp:90`、`:4697-4701` |
| 会话列表预览 | 由 `DialogsPreviewText` 处理 | `history_item.cpp:4733-4734` |
| 无障碍朗读 | 已有 100 字截断 | `dialogs/dialogs_inner_widget_accessibility.cpp:87-89`、`:439-440` |
| 复制到剪贴板 | 不截断 | `history_item_text.cpp:456-462` |

### 8.2 附加维度：提示标记（「可能含未支持特性」）

| 方案 | 锚点 | 优点 | 缺点 |
|---|---|---|---|
| **标记 1：文本后缀**（推荐） | 在 `setText` 前把 `P` 的一句精简版以斜体实体 append 到 `V+E` 尾部 | 零新组件、零新 style；随 `notificationText`/`toPreview`/`originalText` 一起生效 | 通知/预览里也会出现（可接受）；需保证 entity offset 正确 |
| 标记 2：复用 `setAyuHint` | `history_item.cpp:3971-4018` | 现成的"签名行"位置 | **会覆盖真实 `post_author`**（`:3990-3995` 直接 `AddComponents(HistoryMessageSigned::Bit())` 并 `msgsigned->author = hint`），且会**重新置上 `HasPostAuthor`**（`:3985-3987`）——而 Unsupported 分支 `:492` 刚把它清掉；自相矛盾。**不推荐** |
| 标记 3：新增独立 style / 角落图标 | 需新增 `.style` + `Element` 绘制 | 视觉最干净 | 需改 `history_view_element.*`；**ADR-0002:23 引用的 `history_view_element.cpp:2218-2222` 并非 Legacy 角**，而是 `computeUnavailableReason()` 分支（见 §10 说明），锚点需重新定位 |
| 标记 4：保留假按钮 | `UnsupportedMessageMarkup()`（`history_item_helpers.cpp:1415-1427`） | 零改动 | 按钮是"Update AyuGram"（`:1422`），语义是"请更新"而非"内容可能不完整"；且 §2.5 证明它会覆盖真实 `reply_markup` |

### 8.3 边界：服务端没下发 text 的构造器

| 构造器 | 有 `media`？ | 有 `message:string`？ | 说明 |
|---|---|---|---|
| `message#7600b9d3`（`api.tl:124`） | `media:flags.9?MessageMedia`（可选） | **必填** | 唯一会进入机制 A 的构造器。必填≠非空：服务端**可以**下发 `""`（例如媒体消息无 caption）→ 必须有 `P` 兜底，否则气泡空白 |
| `messageEmpty#90a6ca84`（`api.tl:123`） | 无 media 字段 | 无 | 不进本分支 |
| `messageService#7a800e0a`（`api.tl:125`） | 无 media 字段 | 无 | 不进本分支 |
| `storyItem#16a4b93c`（`api.tl:1730`） | `media:MessageMedia`（必填） | 无（有 `caption:flags.0?string` 可选） | 走 §7 的 story 路径，不在范围 |

> 结论【确定】：`V` 为空是**可达**的，必须保留 `P` 兜底；`api.tl:124` 的"必填"只保证字段存在，不保证非空字符串。兜底实现应复用正常分支的 `EnsureNonEmpty()`（`history_item_helpers.cpp:1399-1402`，返回 `":-("`）或直接用 `P`。

---

## 9. 推荐默认规则

### 9.1 规则 F（推荐）

**优先级：`rich_message` 摘要（且摘要非空）→ `vmessage` 原文（经 `Api::EntitiesFromMTP`）→ `UnsupportedMessageText()` 占位；三段互斥、全覆盖、永不留空。**

| 项 | 取值 |
|---|---|
| 正文 | `R` 非空 → `R`；否则 `V` 非空 → `TextWithEntities{qs(V), E}`；否则 → `P` |
| 提示标记 | **标记 1（文本后缀）**：命中形态 B/C/D（非纯 `P` 兜底）时，在正文后追加一句斜体「可能含未支持特性」提示（复用 `lng_message_unsupported` 的后半句语义，新增专用 key） |
| `MessageFlag::Legacy` | **保留**（`history_item.cpp:493` 不动） |
| `MessageFlag::HasPostAuthor` 的 `:492` 清位 | **保留** |
| 长度截断（气泡） | **不新增**（通知侧已有 255 截断） |
| `UnsupportedMessageMarkup()` | **保留**（与需求 6 正交；覆盖真实 `reply_markup` 的问题另立 P2 子项） |
| 同步改动点 | `:500`、`:2325`、`:2637` **三处必须同步**，否则编辑回流/发送回流会把文本打回占位 |

### 9.2 三处同步改动的锚点

| # | 位置 | 现状 | 目标 |
|---|---|---|---|
| 1 | `history_item.cpp:499-501` | `setText(UnsupportedMessageText());` | `F` 规则三元 |
| 2 | `history_item.cpp:2322-2328` | `… : UnsupportedMessageText() : checkedMedia ? edition.textWithEntities : EnsureNonEmpty(edition.textWithEntities)` | `edition.richPage ? Flatten : Unsupported ? F(edition.textWithEntities) : …` |
| 3 | `history_item.cpp:2629-2638` | `if (richPage) {…} else { clearRichPage(); setText(UnsupportedMessageText()); }` | 同上；`updateSentContent` 已有入参 `textWithEntities`（`:2616`）可直接用 |

> 三处必须同改的原因：`:2325`（编辑回流）与 `:2637`（发送回流）**不会**走构造分支，若只改 `:500`，任何一次编辑/发送更新都会把已捡回的文本**打回占位文案**。建议抽一个共用的 `UnsupportedFallbackText(...)` helper 放在 `history_item_helpers.cpp`（与 `UnsupportedMessageText()` `:1404` 同处），三处共用，避免第三次漂移。

### 9.3 已知需一并评估的副产物（本次不改，仅记录）

| # | 现象 | 证据 | 建议 |
|---|---|---|---|
| 1 | Unsupported 分支缺 `setReactions` / `setGroupId` / `applyTTL` / factcheck | §2.4 | 独立 P2 子项；否则"文本可见但反应不可用"的观感割裂 |
| 2 | 真实 `reply_markup` 被假按钮覆盖、转发副本带假按钮 | §2.5、§5.2 | 独立 P2 子项 |
| 3 | `_text` 一变长，通知/预览/复制/翻译全部切换到真实文本 | §5.3 | **正向**，无需处理；但需在提测用例里覆盖 |
| 4 | 复制路径 `hasCopyRestriction` 与 ADR-0002 三道门联动 | `CONTEXT.md:43` | 与 P2 三道门一并验证 |

---

## 10. 未找到 / 无法确定清单

| 项 | 判定 | 使用的命令 / 范围 |
|---|---|---|
| `TextWithEntities` / `EntitiesInText` / `EntityInText` 的结构体定义（含 `empty()` 语义） | **未找到** | `find . -name "text_entity.h"` → 无结果；`ls -A Telegram/lib_ui \| wc -l` → `0`。**本 checkout 的 `Telegram/lib_*` 全部是空目录（子模块未拉取）**，因此 `ui/text/text_entity.h` 不可读。报告中所有 `TextWithEntities` 用法均按 `history_item.cpp` 内既有调用（如 `:2331` `edition.textWithEntities.empty()`、`:4684` `emptyText()`）反推 |
| 判据「`FlattenRichPageSummary` 的摘要是否真的非空」的现成公开 API | **未找到** | `iv_rich_page.h:285-327` 全量声明已读；只有 `FlattenRichPageSummary(page, emptyFallback)`（`emptyFallback=false` 时空则返回空 `TextWithEntities`）可作为等价判据（`iv_rich_page.cpp:2490`）。`contributionCount`（`:2466`）是函数内局部变量，未外露。**建议实现期直接调用 `FlattenRichPageSummary(page, /*emptyFallback=*/false)` 并判 `.text.isEmpty()`** |
| `base::flags<MessageFlag>` 类体 | **未找到** | `find . -name "flags.h"` → 0 命中；`Telegram/lib_base/` 为空子模块目录（`ls -A` → 0 项）。不影响本方案（`data_types.h:376-377` 的 trait/别名已读） |
| `Telegram/SourceFiles/history/history_item_serialization.{h,cpp}` | **未找到** | `glob Telegram/SourceFiles/history/*serial*` → 无结果；`grep -n "history_item_serialization" Telegram/CMakeLists.txt` → 无匹配。**已用于支撑「Legacy 不参与持久化」的结论** |
| `canBeForwarded` | **未找到** | `grep -rn "canBeForwarded" --include=*.h --include=*.cpp Telegram/SourceFiles` → 0 命中。转发门只有 `allowsForward()`（`history_item.cpp:3051-3057`） |
| `allowsMediaDownload` | **未找到** | `grep -rn "allowsMediaDownload" --include=*.h --include=*.cpp Telegram/SourceFiles` → 0 命中。下载门走 `forbidsSaving()`（`:3128-3135`，本 fork 恒 `false`）+ `history_view_list_widget.cpp` 侧 `CopyMediaRestrictionTypeFor` |
| `MessageFlag` 单元测试 | **未找到** | `grep -rn "MessageFlag" Telegram/SourceFiles/test/` → 0 命中 |
| ADR-0002:23 所称「`Legacy` 角的展示位 `history_view_element.cpp:2218-2222`」 | **【无法确定】/ 存疑** | 已读 `history_view_element.cpp:2200-2239`：`:2218-2222` 是 `computeUnavailableReason()`（`restriction_reason`）分支，**与 `MessageFlag::Legacy` 无关**；且 `grep -rn "Legacy" Telegram/SourceFiles/history/` 全 history 树仅 14 处命中，**无任何 Legacy 角绘制代码**（其余为 `history_view_element.h:685` 注释与 `history_view_gif.cpp:2255` `prepareLegacyState()`，均无关）。即：**本 fork 不存在 Legacy 角**。ADR 该锚点需在实现期重新定位（建议改走 §8.2 标记 1 文本后缀，零新组件） |
| `Telegram/SourceFiles/core/search/`（本地消息搜索索引模块） | **未找到（本 fork 不存在）** | `ls Telegram/SourceFiles/core/` 无 `search` 项；`find Telegram/SourceFiles -type d -name "search*"` → 无；`grep -rln "class SearchEngine\|SearchIndex::" Telegram/SourceFiles` → 0 命中。搜索改走 `Api::MessagesSearch`（`api/api_messages_search.h:30-58`）/ `MessagesSearchMerged`（`api_messages_search_merged.h:20+`）服务端 `messages.search` |
| `Telegram/SourceFiles/history/view/history_view_sections.cpp` | **未找到** | `glob **/history_view_sections.*` → `No files found`；`Telegram/SourceFiles/history/view/` 下为 `history_view_chat_section.cpp` / `_pinned_` / `_scheduled_` / `_subsection_tabs`，无 stories 版 |
| 符号 `StoryUnsupported` | **未找到** | `grep -rn "StoryUnsupported" Telegram/SourceFiles` → 0 命中；story 侧等价物是 `Media::storyUnsupported()`（`data_media_types.cpp:2933`）与 `Story::unsupported()`（`data_story.cpp:306`），两个不同函数 |
| `message` 构造器里 `legacy:flags.19?true` 与 `messageMediaUnsupported` 的服务端配合语义 | **【无法确定】** | api.tl 无注释行（`api.tl` 全为裸构造器）。可确定的只有：`flags.19` → `Flag::Legacy`（`history_item_helpers.cpp:918`），`messageMediaUnsupported#9f84f49e` 零字段（`api.tl:131`）。**因此「服务端是否总是同时置 `legacy` 位」无法从代码判定**——这不影响本方案（`:493` 客户端无条件置位） |
| `api.tl` 中除 `message` 外是否还有别的构造器会带 `media:MessageMedia` 且有文本 | **已穷举** | `updateServiceNotification#ebe46819`（`api.tl:337`，`message:string` + `media:MessageMedia` + `entities:Vector<MessageEntity>` 必填）——但它**不走 HistoryItem**（`api_updates.cpp:2298-2318` 直接弹信息框），不在机制 A 范围。其余带 `media:MessageMedia` 的构造器（`api.tl:1652` `messageExtendedMedia`、`:1730` `storyItem`、`:1975` `botPreviewMedia`、`:2037` `messages.webPagePreview`）均无 `message:string` |

---

## 11. 落地验证清单（实现期，非本次范围）

1. 三处锚点（`:500`、`:2325`、`:2637`）是否都已按 F 规则改动（建议抽共用 helper）。
2. 构造 / `applyEdition` / `updateSentContent` 三条路径在「有 rich / 无 rich + 有 text / 无 rich + 空 text」三种输入下的 `notificationText()` 输出。
3. 受限群（ADR-0002 场景）内的 unsupported 消息：编辑入口仍隐藏（`allowsEdit()` `:3077`），且「编辑上一条」快捷键（`history_view_list_widget.cpp:5160`）会跳过它。
4. 复制 / 翻译 / 选中 三个动作拿到的是否是真实文本。
5. 转发副本是否仍带假「Update AyuGram」按钮（§2.5，已知行为，若接受可不修）。
6. 空 `vmessage` 边界：气泡、通知、预览三处均不得为空。
7. 两个判定入口（`messageMediaUnsupported` / `webPageNotModified`）的表现是否一致。

## 主线程抽查（2025-09-27）
- ✅ 三处同改点实测：`history_item.cpp:499-500`（构造）、`:2325`（applyEdition）、`:2637`（updateSentContent）三处 `UnsupportedMessageText()` 均在位。
- ✅ 第二入口实测：`history_item_helpers.cpp:1082-1083` `MTPDwebPageNotModified → Result::Unsupported` 成立。
- ✅ 采信推荐规则 F（rich_message 摘要 → vmessage 原文 → 兜底）与「保留 Legacy」结论，已回写 ADR-0002。
