# ADR-0003 — 可編輯消息重放（Edit & Resend）與 MTProto 請求重放：兩階段與安全模型

- 狀態: Accepted（分期與安全模型定稿）
- 日期: 2025-09-24（grill 第一輪定稿）
- 關聯需求: 7

## Context
- 需求 7（用戶確認）：類 Firefox DevTools「可編輯發包重試」——把已發送消息的請求抓下來、改參數、重發，並**在客戶端拿到返回包**；範圍「兩者都要（分期）」：先消息級，二期任意 RPC。
- 現有相關設施：
  - 「Repeat message」（fork 已有，默認隱藏 `ayu_settings.h:666`）：`ayu/ui/context_menu/context_menu.cpp:784-899`——默認轉發保留發送者、Shift 無引用重發（`originalText()` 重建 sendMessage/SendExistingPhoto）、rich+premium 特判（commit 032ad32854）。**是「復讀」，非「抓包重放」**——只作為入口/樣板。
  - MTProto 攔截點（T7 審計；錨點已由複核修正）：序列化邊界 `Instance::sendSerialized`（`mtproto/mtp_instance.h:162` 聲明 / `:206` 定義）→`SerializedRequest` 佈局 salt/session/msg_id/seq_no/len/body（`mtproto_serialized_request.h:30-41`）→ `Private::sendRequest` 緩存 `_requestMap/_parserMap`（`mtp_instance.cpp:1011-1125`）→ `Session::sendPrepared` 清 msg_id（`session.cpp:422-440`）→ `SessionPrivate::tryToSend/prepareToSend` 賦 msg_id/seq_no（`session_private.cpp:455-531,770-984`）→ `sendSecureRequest` 重算 msg_key+AES-IGE（`:2651-2707`）。
  - 重發/重放基建已有：`SessionPrivate::resend(msgId)`（`:2255-2310`）、`Sender` 的 `overrideId`（`sender.h:229-232`）。
  - 返回包：`rpc_result`→`req_msg_id`→`wasSent()`→requestId（`session_private.cpp:1824-1888`）→ `haveReceivedMessages` → `processCallback`（`mtp_instance.cpp:1145-1202`），`Response.reply` 原始字節可達（`mtproto_response.h:71-77`）。
- 天然邊界：編輯他人消息被服務端拒絕（`CHAT_ADMIN_REQUIRED`，`api/api_editing.h:22-27`）——不可偽造；發送類 RPC 重放有 flood/風控面。

## Decision
1. **階段一：消息級 Edit & Resend**（推薦先落地）
   - 入口：沿用「Repeat message」菜單位掛新項（或在其下加「Edit & Resend」），`ayu/ui/context_menu/context_menu.cpp` + 兩處構建器（`history_inner_widget.cpp:3131`、`history_view_context_menu.cpp:1133`）。
   - 行為：載入 `originalText()+media` 至發送/編輯窗口（複用 `history_view_compose_controls.cpp:4794-4866` 的 editMessage 裝載），修改後**走正常發送管線**發全新消息（構造新 `MTPmessages_SendMessage` 請求 → 正常序列化/msg_id 生成）。
   - 返回包：既有 `.done/.fail` 回顯即為結果；可選加調試 toast/面板展示 `RpcError` 或成功摘要。
   - **不觸碰字節層**——msg_id/seq_no 由 `SessionPrivate` 正常生成，零重放風險。
2. **階段二：任意 RPC 包重放**（Debug/高級開關，默認隱藏）
   - 攔截：`Instance::sendSerialized`（`mtp_instance.h:162`/`:206`）邊界緩存 `(requestId → SerializedRequest + TL 參數鏡像)`。
   - 編輯：參數化編輯器，重點支持消息發送類（`api.tl:2469-2549` sendMessage/sendMedia/sendMultiMedia/editMessage…）；重發走正常 `send`（新 id；`overrideId` 僅用於「同語義重試」場景，默認不用）。
   - 返回包：展示解析後 TL dump（`Response.reply` 解包，`sender.h:43-74` MakeDoneHandler 同一路徑）。
3. **安全模型（兩期通用）**
   - **禁止字節級重發**（不改請求重算 msg_key/重放舊 msg_id）：重發必須由管線重新生成 msg_id/seq_no/msg_key（`prepareToSend/replaceMsgId` 語義）。
   - 發送類重放一律二次確認；RPC 重放面板默認隱藏（樣板：Repeat message 默認 Hidden）。
   - 不提供「編輯他人消息」注入（服務端不可繞，直接不展示）。
   - 重放產物標記為 fork 調試特性，不寫入正常消息流日誌（避免與 anti-edit 快照混淆）。

## Consequences
- 正向：階段一幾乎零新基建（復用發送管線）；階段二攔截點已在審計中定位、風險可控地放在 Debug 開關後；返回包展示路徑與官方回調同源，無需額外解包。
- 負向/風險：階段二的 TL 參數鏡像/編輯器是主要複雜度（需參數描述元數據或反射封裝）；任意 RPC 重放可能觸發 FLOOD_WAIT/風控（處理沿用 `handleFloodErrors`，AGENTS.md 模式）；與「Repeat message」功能邊界需在 UI 上講清楚（復讀 vs 重放）。

## 參考證據
- T6 審計（Repeat message 真相/編輯鏈/anti-edit/anti-recall）；T7 審計（請求→響應生命周期、各插樁點、`overrideId`/`resend` 基建、msg_key 重新計算約束）。