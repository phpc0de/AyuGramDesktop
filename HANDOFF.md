# 交接文檔 — AyuGramDesktop 需求 1~8 改造（2025-09-27）

> 寫給**下一個會話**。先讀完本文，再按 §8 的檔案地圖打開其他文件。
> 本文與 `CONTEXT.md`、`docs/adr/0001~0004`、`docs/delivery-2025-09-27-p0-p1-p2.md` 構成完整上下文。

---

## 1. 一句話現狀

需求 1~6、8 已落地並經四路對抗式複審修完 9 個真 BUG（**未提交、未編譯**）；**需求 7 一期（Edit & Resend，插入語義）已落代碼**（2025-10-22，未編譯）；**二期（任意 RPC 重放）未做**。

---

## 2. 環境與倉庫事實（確定）

| 項 | 值 |
|---|---|
| 倉庫 | `/root/workspace/AyuGramDesktop` = AyuGramDesktop（tdesktop **v7.0.9** fork，分支 `dev`，HEAD `db3b9891cb`，TL layer **228**） |
| 改動狀態 | **已提交**（單一 commit，基線 `db3b9891cb`；權威變更集 = `git show --stat HEAD`） |
| 子模塊 | **37/37 全部未初始化**（含 `cmake`、`Telegram/lib_base`、`lib_ui`、`lib_tl`、`lib_storage`、`codegen`）→ 這些頭文件不在盤上 |
| 構建能力 | **零**：無 cmake/ninja/clang/ccache/docker；無 `out/`；`../Libraries`、`../win64/Libraries` 全缺 |
| 硬件 | 5 核 / 9.3G 內存（可用約 4G）/ 磁盤波動大（曾 31G，也曾 2.8G/98% 滿） |
| 結論 | **任何代碼改動都只能是靜態核驗**，不能宣稱「已驗證可編譯/可運行」 |

---

## 3. 需求 1~8 真實狀態

| # | 需求 | 狀態 |
|---|---|---|
| 1 | 雙擊/右鍵消息存 AG 檢查點 | ✅ 已實現（右鍵菜單項 + 雙擊右鍵手勢，默認開） |
| 2 | 禁止對話解鎖轉發 | ✅ peer 級閘門已開 + 自動走 AyuForward 下載重傳；**媒體閘門已取證**：僅 `MediaCall` 返回 false 且必為服務消息（最先被 `!isService()` 擋掉），故不攔轉發/複製/下載，無需改動（`docs/parallel-2025-10-22/t4-media-gate.md`） |
| 3 | 解鎖複製 | ✅ `CopyRestrictionTypeFor` 恆 `None`（`history/view/history_view_list_widget.cpp:5972-5981`） |
| 4 | 解鎖下載 | ✅ 但**不是本次解開的**：`forbidsSaving()` 在本 fork 本來就恆 `false`（`history/history_item.cpp:3140-3147`） |
| 5 | 檢查點隨時跳轉 + 進群默認定位 | ✅ 已實現（peer 菜單跳轉 + 列表進入默認定位）。局限：跳轉不帶 topic、會把會話永久釘在檢查點 |
| 6 | 突破「請更新最新版本」式限制顯示部分文本 | ✅ 三處同改，複審 **0 個確定 BUG** |
| 7 | 極客時間式「重放消息」「編輯消息」 | ⚠️ **一期已實現**（Edit & Resend，插入語義，未編譯）；**二期（RPC 重放）未做** |
| 8 | 快速迭代/編譯 | ✅ 方案定案（外部構建機驗證 + codegraph/grep 交叉核驗），⚠️ 零編譯驗證 |

---

## 4. 已落地代碼改動（21 文件，已提交）

### P0 構建修復（HEAD 原本編不過）
| 文件 | 改動 |
|---|---|
| `data/data_session.cpp` | ProcessChat/ProcessChannel 的 `flagsSet`：`data.is_noforwards()` → `Flag::AyuNoForwards`；刪除不存在的 `data.is_ayuNoforwards()`（api.tl layer 228 無此訪問器，**這就是 HEAD 編不過的根因**） |
| `data/data_chat.cpp` / `data_channel.cpp` | `allowsForwarding()` → `return true;`（對齊 ADR-0002 決策 1） |
| `data/data_peer.cpp` | `canExportChatHistory()` 增加 `isAyuNoForwards()` 閘門（整庫導出不跟著解鎖） |

### P1 AG 檢查點
| 文件 | 改動 |
|---|---|
| `ayu/data/entities.h` | 新增 `Checkpoint` 實體（fakeId/userId/dialogId/topicId/**messageId 用 ID(int64)**/date） |
| `ayu/data/ayu_database.{h,cpp}` | `Checkpoint` 表 + 索引；`setCheckpoint` / `getCheckpoint` / `getLatestCheckpoint` / `clearCheckpoint`（返回 `std::optional`） |
| `ayu/data/messages_storage.{h,cpp}` | `AyuMessages::setCheckpoint(item)` / `getCheckpoint(peer,topicId)` / `getLatestCheckpoint(peer)` / `clearCheckpoint(peer,topicId)` |
| `ayu/ui/context_menu/context_menu.{h,cpp}` | `AddCheckpointAction`（消息菜單「Save AG Checkpoint」）；peer 菜單 AyuGram 子菜單按 topicId 加「Jump to / Clear AG Checkpoint」（**有 topic 時 Jump 走 `showTopic` 帶 topic**，`context_menu.cpp:329-341`；無 topic 維持原 `showPeerHistory`） |
| `history/history_inner_widget.{h,cpp}` | 雙擊右鍵手勢：首擊延遲 500ms（`base::Timer`），二次點擊保存並吞事件，超時用合成 `QContextMenuEvent` 重放菜單；`itemRemoved` **首行**清理懸垂指針 |
| `dialogs/dialogs_widget.cpp` | 列表進入聊天時若有檢查點則覆蓋 `showAtMsgId`；搜索中不覆蓋；用 `getLatestCheckpoint` 兼容論壇話題 |
| `ayu/ayu_settings.{h,cpp}` | bool 開關 `saveCheckpointOnDoubleRightClick`（**默認 true**），6 觸點齊全 |
| `ayu/ui/settings/settings_ayu.cpp` | 設置頁補該開關（`addSettingToggle`，`settings_ayu.cpp:653-658`；lang key `ayu_SaveCheckpointOnDoubleRightClickText`） |
| `data/data_session.cpp` / `main/main_session.cpp` | 檢查點清理接線：`notifyHistoryCleared` 清本 peer 檢查點（`data_session.cpp:2374`）；`finishLogout` 清賬號全部檢查點（`main_session.cpp:348`）；include `ayu/data/ayu_database.h` |
| `Resources/langs/lang.strings` | 3 個鍵：`ayu_SaveCheckpointContextMenuText` / `ayu_JumpToCheckpointMenuText` / `ayu_ClearCheckpointMenuText` |

### P2 需求 6（unsupported 顯示部分文本）
`history/history_item.cpp` **三處必須同改**：構造函數 Unsupported 分支、`applyEdition`、`updateSentContent`。
邏輯：`rich_message` 摘要 → `vmessage` 原文 + `Api::EntitiesFromMTP` → 兜底 `UnsupportedMessageText()`；**保留 `MessageFlag::Legacy`**（禁編輯）。

### P3 需求 7 一期（Edit & Resend，插入語義）
| 文件 | 改動 |
|---|---|
| `ayu/ui/context_menu/context_menu.{h,cpp}` | 新增 `AddEditAndResendAction(menu, item, std::function<void(const QString&)>)`：`originalText().text` 非空才加菜單項，回調把純文本交給調用方 |
| `history/view/history_view_context_menu.cpp` | 一欄調用點：`list->insertTextAtCursor(text)`（`history_view_list_widget.h:423`） |
| `history/history_inner_widget.cpp` | 兩欄調用點：`_widget->insertTextAtCursor(text)`（`history_widget.h:260`） |
| `Resources/langs/lang.strings` | 新增 `ayu_EditAndResendMenuText` = "Edit & Resend" |

局限：`insertTextAtCursor` 是「插入+補空格」語義，不清空已有草稿、不帶 entities（純文本）；整框富文本載入需 `ListWidget::editMessageRequestNotify`（`history_view_list_widget.h:413`），屬二期/後續。

---

## 5. 複審結論（四路 subagent + 主線程逐條 grep 驗證）

報告：`docs/review/r1-data-layer.md`、`r2-storage.md`、`r3-ui-gesture.md`、`r4-p2-unsupported.md`

**已修的 9 個問題**（其中 4 個是主線程自己寫出來的）：
1. peer 菜單新增早退是**死代碼** → 默認配置下「清除檢查點」入口根本不存在 → 檢查點查詢前置、合併早退。
2. 搜索守衛用錯：`uniqueChatsInSearchResults()` 依賴 `supportMode()`（`window/window_session_controller.cpp:2046-2055`），普通帳號恆 false → 改用 `_searchState.query.isEmpty()`。
3. `itemRemoved` 早退在清理之前 → 手勢期間 item 銷毀留**懸垂指針** → 清理塊移到函數首行。
4. `MsgId::bare` 是 **int64**（`data/data_msg_id.h:48`），`int messageId` 窄化 → `Checkpoint::messageId` 改 `ID`；手勢補 `!isLocal() && id > 0`。
5. 讀側硬編碼 `topicId=0` → 論壇檢查點 100% 讀不到 → 新增 `getLatestCheckpoint`。
6. 「無記錄」與「讀失敗」同值 → 改 `std::optional`。
7. `allowsForwarding()==true` 順帶解鎖**整庫 JSON/HTML 導出** → `canExportChatHistory()` 單獨加閘門。
8. `showDeferredContextMenu` 早退未清座標 → 補清理。
9. `checkpoint.fakeId = 0` 誤導 → 刪除。

**有意取捨不改**：進群釘在檢查點（需求 5 本意）、500ms 右鍵延遲（用戶選「默認開」）、`Flag::NoForwards` 成死位/4 讀點恆 false（即解鎖效果）。

---

## 6. 已知缺陷 / 技術債 / 未修缺口

| 類型 | 內容 |
|---|---|
| 技術債 | `data/data_chat.h:27` `AyuNoForwards = (1 << 31)`：無底層類型 enum 移入符號位屬 UB，與 channel 側 `1ULL << 63` 不對稱。**不敢改**——會牽動不可見的 `base::flags`（`lib_base` 空） |
| 已修 | 設置頁開關已補（`settings_ayu.cpp:653-658`；bool 行 API 已取證：`settings_ayu_utils.h:21-22` 的 `BoolGetter/BoolSetter`） |
| 已修 | peer 菜單跳轉已帶 topic（有 topic 走 `showTopic`，`context_menu.cpp:329-341`；無 topic 維持原路徑） |
| 部分修 | 清空歷史/刪聊天（`notifyHistoryCleared`）+ 註銷（`finishLogout`）已清；**殘留 R1：頻道刪聊天走 `Unload` 不發 `notifyHistoryCleared`**（`data_session.cpp:1645`），R2 話題刪除無鉤子 |
| 已消除 | 需求 2 媒體閘門已取證無需改（t4：`MediaCall=false` 但必為服務消息、最先被擋，全鏈不消費媒體閘門） |
| 風險 | 進群默認定位把會話釘在舊消息；逃生口是「Clear AG Checkpoint」 |
| 未驗證 | 全部改動**零編譯、零運行**；`std::optional`/`base::Timer`/`enumerateItems`/`st::menuIconShowInChat` 等簽名均為靜態推斷 |

---

## 7. 需求 7（Edit & Resend / 重放）阻塞分析 — 最重要

> **2025-10-22 更新：一期已按「插入語義」落地（見 §4 P3）。本節保留為阻塞史與二期（RPC 重放）參考。**

ADR-0003 方案完備（一期：載入原文到輸入框可編輯後重發；二期：`Instance::sendSerialized` 邊界攔截 + 參數化 RPC 重放 + 回包展示）。落地卡在**「從消息右鍵菜單寫入主輸入框」沒有公開路徑**：
- `HistoryWidget::insertTextAtCursor(const QString&)` 是 public（`history/history_widget.h:260`；:113 只是前向宣告，舊稿行號有誤）；
- 但 `History::owner()` 繼承自 `PeerData::owner()`（`data/data_peer.h:226`）返回 `Data::Session&`，**不是 widget**；
- `MainWidget` 只有前向聲明 + 私有成員 `_history`（`mainwidget.h:14,348`）；
- `Window::ActiveInstance()` 所在頭（`window/window.h`）**不在本樹**；
- 備選「寫本地草稿 + 切聊天」需 `Data::Draft(const TextWithTags&, ...)`（`data/data_drafts.h:59`）與 `TextWithEntities → TextWithTags` 轉換，而 `TextWithTags`/`Ui::InputField` 都在**未初始化的 `lib_ui`**，簽名無法核對。

- **2025-10-22 更新（t5 取證）**：一期在本 checkout 內**可行**，不依賴 lib_ui。真正缺口是 Ayu handler 沒拿到 `not_null<HistoryView::ListWidget*>`：入口在 `history_view_context_menu.cpp:1127` 作用域內；一欄鏈 `history_view_list_widget.h:423` → `list_widget.cpp:5198` → `history_view_chat_section.cpp:437`；兩欄鏈的 `_widget` 已在 `history_inner_widget.cpp:3193` 作用域（`history_widget.h:260`）。`insertTextAtCursor` 是「插入+補空格」語義；若要整框富文本，可用公開的 `ListWidget::editMessageRequestNotify`（`history_view_list_widget.h:413`）。

**結論**：盲寫只會得到編不過的代碼，故拒絕落地。三條可行路徑：
1. **外部構建機上實現**（推薦，有編譯器兜底）；
2. 先 `git submodule update --init Telegram/lib_ui Telegram/lib_base` 後由主線程實現；
3. 零新代碼近似：現有「重複消息」可產生副本，用戶雙擊**自己發出的副本**即進原生編輯窗。

二期攔截點是 `MTP::Instance::sendSerialized`（`mtproto/mtp_instance.h:162` 聲明 / `:206` 定義）——`MTP::Sender::send` **全倉不存在**（早期文檔寫錯已修）。

---

## 8. 檔案地圖（新會話開場必讀）

| 文件 | 作用 |
|---|---|
| `CONTEXT.md` | 主狀態檔（§1 指紋、§2 需求表、§3 術語、§4 事實、§7 路線圖、§8 構建/成本、§9 P0 改單、§10 計劃索引、§11 交付、§12 需求7狀態） |
| `HANDOFF.md` | 本文件 |
| `docs/adr/0001-ag-checkpoint.md` | 檢查點：每會話一個、存 `ayudata.db`、打開前替換 showAtMsgId |
| `docs/adr/0002-unlock-restricted-chats.md` | 三門解鎖 + peer 級 noforwards → `AyuNoForwards` + 轉發走 AyuForward 重傳 + 需求 6 規則 |
| `docs/adr/0003-editable-message-replay.md` | 兩期重放方案 + 禁止字節級重發 |
| `docs/adr/0004-build-iteration-strategy.md` | 外部構建機驗證 + 靜態交叉核驗 + 交付物規範 |
| `docs/delivery-2025-09-27-p0-p1-p2.md` | **交付報告**：改動總覽、複審結果、外部驗證清單、需求 7 阻塞分析 |
| `docs/p0-build-fix-plan.md` | P0 精確改動單 |
| `docs/plan/p1-checkpoint-*.md` | 檢查點三份實現方案（存儲/入口/導航） |
| `docs/plan/p2-unsupported-partial-text.md` | 需求 6 規則矩陣 |
| `docs/review/r1~r4-*.md` | 四份對抗式複審 |
| `docs/review/p0-independent-review.md` / `submodule-blindspots.md` | P0 獨立複核 + 45 條假設/29 項外部核對清單 |
| `docs/parallel-2025-10-22/` | 2025-10-22 五路並行回合：t1 設置開關 / t2 孤兒清理 / t4 媒體閘門 / t5 需求 7 路徑（t3 無獨立報告，以 diff 為準）；`README.md` 為總表 |

---

## 9. 下一步（按優先級）

1. **外部構建機驗證**（唯一能消除「零編譯」風險的事）：子模塊對齊（`cmake`/`lib_base`/`lib_ui`/`lib_tl`/`lib_storage`/`codegen`）；命令 `cmake --build out --config Debug --target Telegram`（Windows，AGENTS.md:48-54）或 WSL `Telegram/build/docker/centos_env/build_debug.sh`；回填**首個錯誤塊**。回歸重點：受保護群轉發是否走重傳、群設置「限制轉發」回顯、右鍵 Save AG Checkpoint、雙擊右鍵手勢、進群默認定位、受保護群**不應**出現「導出聊天記錄」；**新增**：設置頁開關（AyuGram → Save checkpoint on double right-click）、論壇話題跳轉後定位、清空歷史後檢查點消失、註銷後檢查點清空。
2. **需求 7 二期（任意 RPC 重放）**：攔截點 `MTP::Instance::sendSerialized`（`mtproto/mtp_instance.h:162/:206`），需 Debug 開關 + 回包展示；一期（Edit & Resend）已完成，等人工回歸。
3. R1 缺口：頻道刪聊天清理檢查點（`data_session.cpp:1645` 走 `Unload` 不發 `notifyHistoryCleared`，需另找鉤子或改走 clear 路徑）。
4. 技術債：`ChatDataFlag::AyuNoForwards` 改 `: uint32` + `1U << 31`（**必須有編譯器兜底**）。
5. ~~`_media->allowsForward()` 取證~~ → 已完成（t4，無需改動）。

---

## 10. 工作紀律（本倉庫強制）

- **AGENTS.md**：`Avoid building the project`；只 Debug 不 Release；代碼**不寫註釋**；`u"..."_q`；禁止 `Q_OS_LINUX`（用 `!defined Q_OS_WIN && !defined Q_OS_MAC` 或 `Platform::IsLinux()`）；新設置項優先 `writePref/readPref`。
- **證據紀律**：每條結論帶 `相對路徑:行號` + 摘錄；找不到寫「未找到 + 命令 + 範圍」；區分【確定】/【無法確定】；禁編造行號。
- **文件認領**：改文件前 `claim_files`，寫完 `release_files`。
- **權限**：寫出的報告/文檔必須 `chmod 644`（DSH 默認 600，用戶打不開會被質問）。
- **聊天輸出**：只回一句進度/結論，詳細內容落檔案。
- **subagent**：給硬性輸出契約（≤15 行摘要 + 報告路徑 + file:line），**主線程必須逐條抽查**（本次 4/9 個 BUG 由主線程自己發現）。

---

## 11. 2025-10-22 並行回合補記

- 五路 subagent 并发（orchestrate fanout）：**T1 設置頁開關 ✅**、**T2 孤兒清理 ✅（DB 層 + 主線程接線）**、**T3 跳轉帶 topic ✅**（subagent 誤報 error，代碼已落盤，主線程已驗證）、**T4 媒體閘門取證 ✅**、**T5 需求 7 路徑取證 ✅**。
- 權威變更集更新為 **21 文件 / +356−16**（仍未提交、仍未編譯）。
- 報告：`docs/parallel-2025-10-22/{t1-settings-toggle,t2-orphan-cleanup,t4-media-gate,t5-req7-path}.md` + `README.md`（T3 無獨立報告，以 diff 為準）。
- 主線程收口：核驗 T1/T3/T4/T5 六處關鍵結論行號；親手接線 T2（`data_session.cpp:2374` / `main_session.cpp:348`）；更新 HANDOFF §1/§2/§3/§4/§6/§7/§8/§9。
- **編譯驗證包**：`docs/build-package/req1-7-uncommitted.diff`（759 行，權威 patch）、`external-verify.md`（外部機一次性取全錯誤命令 + 15 項回歸清單）、`github-actions.md`（GHA 使用說明）。
- **GitHub Actions**：`.github/workflows/build-win-amd64.yml`（windows-2022；job1 prepare-libs 構建並緩存依賴，job2 build Debug `/m` 并行全量編譯；產物 `build.log`/`errors.txt`/`AyuGram.exe`）。
- **T3 補報告**：`docs/parallel-2025-10-22/t3-jump-topic.md`（subagent 漏交，主線程補齊並驗證）。
- **需求 7 一期**（Edit & Resend，插入語義）：`context_menu.{h,cpp}` 新增 action，一欄/兩欄兩個調用點分別接 `ListWidget::insertTextAtCursor` / `HistoryWidget::insertTextAtCursor`（見 §4 P3）。
- **GitHub Actions 發布工作流**：`.github/workflows/release-win-amd64.yml`（觸發：tag `v*` / `release: published`；產出單 exe 便攜 zip + `sha256sums.txt`，自動建 Release；說明見 `docs/build-package/github-actions.md`）。
