# 2025-10-22 並行回合總表（5 路 subagent + 主線程收口）

> 權威變更集：`git --no-pager diff` = **21 文件 / +356−16**（全部未提交、零編譯）。
> 主線程已逐條抽查下列關鍵 file:line。

## 總覽

| # | 任務 | 結果 | 代碼落地 | 報告 |
|---|---|---|---|---|
| T1 | 設置頁補 `saveCheckpointOnDoubleRightClick` 開關 | ✅ | `settings_ayu.cpp:653-658` + `lang.strings:8916` | `t1-settings-toggle.md` |
| T2 | Checkpoint 孤兒行清理 | ✅ 兩層接線 | DB：`ayu_database.{h,cpp}`；接線：`data_session.cpp:2374`、`main_session.cpp:348` | `t2-orphan-cleanup.md` |
| T3 | peer 菜單跳轉帶 topic | ✅（subagent 誤報 error，代碼已落盤） | `context_menu.cpp:329-341`（`showTopic`） | 無，以 diff 為準 |
| T4 | 需求 2 媒體閘門取證 | ✅ 無需改動 | — | `t4-media-gate.md` |
| T5 | 需求 7「載入原文到輸入框」路徑取證 | ✅ 一期在本 checkout 可行 | — | `t5-req7-path.md` |

## 關鍵結論（主線程已復核）

- **T1**：`ayu_builder.h` 的 `.title` 是 `rpl::producer<QString>`（不帶 `tr::now`）；`BoolGetter/BoolSetter`（`settings_ayu_utils.h:21-22`）與新 getter/setter 匹配；lang key 全倉唯一。
- **T2**：歷史刪除唯一精確信號是 `History::clear` → `history.cpp:4399` `notifyHistoryCleared`（`:4379` 的 `notifyHistoryUnloaded` lazy 也 fire，**不可用**）；`finishLogout`（`main_session.cpp:348`）只由 `LoggedOut` 進入，作賬號級清理。坑：`Histories::clearAll` 不發通知，故兩者互補。
- **T3**：`SessionController : public SessionNavigation`，`showTopic(not_null<Data::ForumTopic*>, MsgId, SectionShow)`（`window_session_controller.cpp:1340`）可用；raw `ForumTopic*` 隱式轉 `not_null` 有先例（`mainwidget.cpp:1638`）。
- **T4**：媒體閘門全倉 3 處實現、唯一消費點 `history_item.cpp:3068`；僅 `MediaCall` 返回 false（`data_media_types.cpp:1836-1838`），而 `MediaCall` 必為服務消息、被 `!isService()` 先擋 → 閘門冗餘；複製/下載鏈根本不消費它。
- **T5**：`insertTextAtCursor` 真行號 `history_widget.h:260`（舊文檔 :113 是前向宣告）；一期入口 `not_null<ListWidget*>` 在 `history_view_context_menu.cpp:1127`；整框富文本可走 `ListWidget::editMessageRequestNotify`（`history_view_list_widget.h:413`）。

## 主線程收口動作

1. 核驗 T1/T3/T4/T5 六處關鍵行號（上述「關鍵結論」）。
2. 親手接線 T2：`data_session.cpp` `notifyHistoryCleared` 清理本 peer 檢查點 + include `ayu/data/ayu_database.h`；`main_session.cpp` `finishLogout` 清理賬號全部檢查點 + include。
3. 更新 `HANDOFF.md` §1/§2/§3/§4/§6/§7/§8/§9/§11 與 `CONTEXT.md` §13。

## 未決 / 殘留

- **R1**：頻道刪聊天走 `Unload`（`data_session.cpp:1645`）不發 `notifyHistoryCleared` → 該路徑不清理檢查點。
- **R2**：論壇話題刪除（topic 級）無 hook。
- **零編譯**：全部改動仍需外部構建機 Debug 驗證。
- **需求 7 語義待定**：「插入」vs「整框替換」。
