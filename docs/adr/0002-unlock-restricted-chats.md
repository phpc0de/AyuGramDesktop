# ADR-0002 — 受保護對話三道門解鎖與不可見內容顯示策略

- 狀態: Accepted（策略定稿；含一項必須先行處理的 build 先決）
- 日期: 2025-09-24（grill 第一輪定稿）
- 關聯需求: 2 / 3 / 4 / 6

## Context
- 需求 2/3/4：禁止轉發/複製/下載的聊天中解鎖三道門；需求 6：「請更新最新版本」佔位消息（機制 A，用戶已確認文案 = `lng_message_unsupported`，`lang.strings:2530`）在安全前提下顯示部分文本。
- 現狀（審計證據）：
  - 消息級已解鎖：`HistoryItem::forbidsForward()=false`（`history_item.cpp:3124-3126`）；消息 `noforwards` 被映射到私有 `MessageFlag::AyuNoForwards`（`history_item_helpers.cpp:931-933`）；`UserData::allowsForwarding()=true`（`data_user.cpp:690`）；`forbidsSaving()` invoice 分支被註釋（`:3128-3136`）；`UnavailableReason::Extract` 被掏空（`data_peer.cpp:133-136`）。
  - **唯一殘餘攔截源**：群/頻道 peer 級 `Flag::NoForwards` 由服務端真實置位（`data_session.cpp:933,1097`）→ `ChatData/ChannelData::allowsForwarding()=false`（`data_chat.cpp:75-77`、`data_channel.cpp:787-789`）→ 三道門同時擋下：
    - 轉發：`HistoryItem::allowsForward()`（`history_item.cpp:3051-3057`）→ 菜單 `AddForwardMessageAction` 整項隱藏（`history_view_context_menu.cpp:427,434`）；`setupSharingDisallowed` 群/頻道分支（`history_inner_widget.cpp:657-658`）。
    - 複製：`hasCopyRestriction = !peer->allowsForwarding() || item->forbidsForward()`（`history_inner_widget.cpp:3979-3981`）→ `lng_error_nocopy_*` toast。
    - 下載：`CopyMediaRestrictionTypeFor` 第一道 gate 即 peer 級（`history_view_list_widget.cpp:5987-5995`）。
  - **服務端會拒絕**：錯誤碼 `CHAT_FORWARDS_RESTRICTED` 已被處理（`apiwrap.cpp:583`，`sendMessageFail` 內，toast `lng_error_noforwards_{channel,user,group}`）——forwardMessages 失敗回調即走 `sendMessageFail`（`apiwrap.cpp:4026-4037`）。即：**直接放開菜單 + forwardMessages 仍會被服務端拒**。
  - **build 先決（必須先行）**：`data_session.cpp:934,1098` 引用 `data.is_ayuNoforwards()`——api.tl（layer 228）**無**該 flag、全樹無定義（已 grep 驗證），與分支意圖（把 peer 級 noforwards 也映射進私有 `Flag::AyuNoForwards`，對齊消息級寫法）不一致；HEAD 存在編譯失敗風險。落地前必須以此為準修正。
  - 機制 A：`messageMediaUnsupported`→`CheckMessageMedia::Unsupported`（`history_item_helpers.cpp:1109-1110`）→ 構造分支**丟棄已到手的 `vmessage`**、`setText(UnsupportedMessageText())`（`history_item.cpp:491-507` + `MessageFlag::Legacy`）；`api.tl:124` 的 `message:string` 必填 → 原文在客戶端安全可得；`rich_message` 摘要通道是現成「部分文本」先例（`Iv::FlattenRichPageSummary` :495-498）。

## Decision
1. **peer 級對齊**：Chat/Channel 處理時把服務端 `noforwards` 同時映射到 `Flag::AyuNoForwards`（修正 `data_session.cpp:934,1098` 為 `data.is_noforwards() ? Flag::AyuNoForwards : Flag()` 或等價），`allowsForwarding()` fork 版返回 `true`（保留 AyuNoForwards 供 AyuForward 分流判定）。
2. **三道門放開**：轉發菜單顯示（`allowsForward()` 不再被 peer 擋）、複製 `hasCopyRestriction` 恆 false、下載 `CopyMediaRestrictionTypeFor` 同步放開、`setupSharingDisallowed` 群/頻道分支同步處理。
3. **轉發執行必須走重傳**：受保護群/頻道及 AyuNoForwards 條目一律由 AyuForward 智能分流（`apiwrap.cpp:3834-3844` 已掛 `isFullAyuForwardNeeded`/`isAyuForwardNeeded`，擴展 peer 級判定）→「下載-重傳為新消息」（`ayu_forward.cpp:321-510`）。**禁止**對這類條目直接發 forwardMessages（服務端 403 證據見上）。重傳語義沿用現有 NoSenderNames/PreserveInfo 策略。
4. **需求 6（機制 A 部分文本）**：Unsupported 分支改為三段互斥——`rich_message` 摘要 → `vmessage` 原文（經 `Api::EntitiesFromMTP`）→ `UnsupportedMessageText()` 兜底。**保留 `MessageFlag::Legacy`**（複核確認其唯一語義消費點是 `allowsEdit()` 的 `&& !isLegacyMessage()`，`history_item.cpp:3077`；移除會波及 7 個編輯入口並可能把降級文本回寫服務端）。提示標記用文本斜體後綴，**不用** `setAyuHint`（會覆蓋真實 `post_author`）。
   - **必須同改三處**（否則文本會被打回佔位）：`history_item.cpp:500`（構造）、`:2325`（applyEdition）、`:2637`（updateSentContent）。
   - `Unsupported` 有**第二入口** `webPageNotModified`（`history_item_helpers.cpp:1082-1083`）與 Unsupported 共用該分支。
   - 勘誤（2025-09-27 複核）：初版所引「Legacy 角展示位 `history_view_element.cpp:2218-2222`」**錨點錯誤**——該處實為 `computeUnavailableReason()` 分支，渲染層零 Legacy 引用。
   - 已知副作用：Unsupported 分支比正常分支少做 4 件事（`setReactions :574`、`setGroupId :567`、`applyTTL :575`、factcheck `:577` 都在 `else` 塊內）→ 文本可見後反應/分組仍不可用。

## Consequences
- 正向：三門一處判定點修好；轉發「偽裝新消息」不可被服務端反制（已驗證錯誤碼路徑）；需求 6 只是把「已到手但被丟棄」的文本撿回來，零新增拉取。
- 負向/風險：重傳媒體有上傳/帶寬成本（已有進度條 `AyuForwardWriteRestriction`，`history_widget.cpp:8325`）；「偽裝轉發」與官方 ToS 衝突（fork 語境可接受）；重傳後消息不再攜帶 fwd_from（除 PreserveInfo 已保留引用場景）；部分防爬頻道可能對短時間大量上傳做風控——實現期加去重緩存與節流；複製/下載解鎖僅為客戶端行為（服務端無對應 gate，無二次風險）。
- 未決（實現期驗證即可）：forwardMessages 在普通無限制群的失敗分支是否也進 `sendMessageFail`（影響 toast 語義）；`Legacy` 位保留對編輯/回顯的影響已在 history_item.cpp:3077 確認（禁編輯）——按方案 4 保留。

## 參考證據
- T3 審計報告（三道門檢查點/UI/執行三段式）；`apiwrap.cpp:583`（CHAT_FORWARDS_RESTRICTED）；`data_session.cpp:934,1098`（phantom accessor）；T5 審計報告（機制 A 三落點 + 文本可用點）。