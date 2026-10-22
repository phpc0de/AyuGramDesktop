# ADR-0001 — AG 检查点：语义与持久化选型

- 狀態: Accepted（本書面記錄設計與證據；實現前無需再議）
- 日期: 2025-09-24（grill 第一輪定稿）
- 關聯需求: 1 / 5

## Context
- 用戶需求：①每個對話可保存一個「AG 檢查點」（指向某條消息）；②可隨時跳轉到該檢查點；③客戶端退出重進後，進入該群**默認定位**到檢查點。
- 已確認語義：觸發 = 右鍵菜單項「保存為 AG 檢查點」+ 在消息上快速右鍵兩次（雙擊右鍵，不進菜單）；**每會話只保留一個，新覆蓋舊**；進群若存在檢查點則默認定位到它。
- 現有基建（可複用，證據見 T4 審計）：
  - 跳轉鏈完整：`showPeerHistory`（`window_session_controller.cpp:3136-3149`）→ `MainWidget::showHistory`（`mainwidget.cpp:1386-1535`）→ `HistoryWidget::showHistory`（`history_widget.cpp:2999-3196`）→ `History::isReadyFor/getReadyFor`（`history/history.cpp:2987-3049`）→ `firstLoadMessages` with `offsetId=msgId, add_offset=-page/2`（`history_widget.cpp:4840-4874`）→ `countInitialScrollTop` 顯式 msgId 分支自動滾動+高亮（`:8215-8254`、`history_view_highlight_manager.cpp`）。
  - 打開聊天默認決策鏈：`dialogs_widget.cpp:1188-1202`（`row.message.fullId.msg`）→ showThread → showPeerHistory → showHistory 歸一（`:3024-3031`、`:3273-3277`、`:3293-3296` 內存級恢復 `_history->showAtMsgId`，`:3143`）。
- 持久化候選對比（本期審出）：(a) `Core::Settings` writePref/readPref KV；(b) `Main::SessionSettings`（當前無 scroll 字段）；(c) `tdata/ayu_settings.json`（`ayu_settings.h` JSON 序列化）；(d) `tdata/ayudata.db` SQLite（`ayu/data/ayu_database.cpp` 已有 DeletedMessage/EditedMessage 表 + `(userId,dialogId,topicId,messageId)` 索引先例）。

## Decision
1. **粒度**：每 peer（+topic）一個檢查點；新保存覆蓋舊；保存時記錄 `(userId, peerId, topicId, msgId, updatedAt)`。
2. **存儲**（選 d，拒絕 a/c）：在 `tdata/ayudata.db` 新增 `Checkpoint` 表（sqlite_orm，模式照搬 `DeletedMessage`：`ayu_database.cpp:21-64`），存取走 `ayu/data/messages_storage` 風格 API。理由：與 Ayu 既有 SQLite 統一、天然多賬號（userId 列）、事務/崩潰安全、每 peer 一條不會膨脹；writePref 會讓全局 settings blob 混入逐 peer 數據、無事務；ayu_settings.json 同樣是單 blob 且只適合開關類。
3. **讀取時機**：會話初始化後（`ayu_infra` 的 DB 加載點 `ayu_infra.cpp:22-60`）異步讀入內存緩存 `map<peerId, msgId>`；打開 peer 前（`MainWidget::showHistory` 之前）把 `showAtMsgId` 替換為檢查點 msgId。
4. **跳轉交互**：菜單項「跳到檢查點」→ `showPeerHistory(peer, {}, checkpointMsgId)`，自動獲得 around 加載+高亮+滾動。
5. **降級**：檢查點消息被刪/不在窗口 → 復用 `countInitialScrollTop` 的 `lng_message_not_found` 降級（`history_widget.cpp:8226`）+ 清除該檢查點。

## Consequences
- 正向：不膨脹全局 settings；與 Ayu SQLite 家族統一；多賬號安全。
- 負向/邊界：DB 讀取是異步的（預載入內存避免打開時等待）；與官方「恢復上次滾動位置」（內存級）的優先級定義為**檢查點優先，無檢查點才走原生**；topic（論壇）與 peer 層級需分開存（topicId 為空 = 主線程）；遷移群（migrateFrom 負 id）沿用既有反轉邏輯（`mainwidget.cpp:1399-1401`）。
- 備選（若後續嫌 SQLite 重）：回退方案 a（writePref，key=`ag_checkpoint_v1_<peerId>`），不影響其他決策。

## 參考證據
- T4 審計報告（跳轉鏈/打開決策鏈/持久化候選）；`ayu_database.cpp:15`（db 路徑）、`apiwrap` 無關。Codegraph 就緒。