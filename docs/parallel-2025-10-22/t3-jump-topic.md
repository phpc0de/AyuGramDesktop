# T3 補記：peer 菜單「Jump to AG Checkpoint」帶 topic（子代理漏交報告，代碼已驗證）

> 背景：T3 子代理回報 `error` 且未交報告，但代碼**已落盤**。主線程核對後確認可用，本文為其補交報告。
> 權威代碼在 `git diff -- Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp`。

## 改動位置

`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp`，`AddAyuGramActions` 內（第 322–352 行）：

```cpp
if (hasCheckpoint) {
    addAction(
        tr::ayu_JumpToCheckpointMenuText(tr::now),
        [=]
        {
            if (topic) {
                sessionController->showTopic(
                    topic,
                    MsgId(checkpoint->messageId),
                    Window::SectionShow::Way::ClearStack);
            } else {
                sessionController->showPeerHistory(
                    peerData,
                    Window::SectionShow::Way::ClearStack,
                    MsgId(checkpoint->messageId));
            }
        },
        &st::menuIconShowInChat);
    ...
}
```

## API 取證（主線程已逐條核對）

| 事實 | 證據 |
|---|---|
| `showTopic(not_null<Data::ForumTopic*>, MsgId itemId = 0, const SectionShow& = {})` | `window/window_session_controller.h:249-252`；定義 `window_session_controller.cpp:1340` |
| `SessionController : public SessionNavigation`（public 繼承） | `window_session_controller.h:409` / `:228` |
| `showTopic` 內部分派：`showRepliesForMessage(topic->history(), topic->rootId(), itemId, params)` | `window_session_controller.cpp:1340-1348` |
| raw `Data::ForumTopic*` 直傳 `not_null` 形參有先例 | `mainwidget.cpp:1638` `_controller->showTopic(topic, item->id, params)`（`topic` 來自 `item->topic()`，raw 指針） |
| `SectionShow::Way` 隱式轉 `const SectionShow&` 有先例 | `window/notifications_manager.cpp:1420` `window->showTopic(topic, itemId, SectionShow::Way::Forward)` |
| `topic` 判定沿用既有邏輯 | `context_menu.cpp:264-265` `peerData->isForum() && thread ? thread->asTopic() : nullptr` |

## 行為矩陣

| 場景 | 行為 |
|---|---|
| 論壇群 + 從話題內打開菜單（`thread` 非空） | `showTopic(topic, msgId)` → 進入該話題並定位到檢查點消息 |
| 普通群/頻道（`topic == nullptr`） | 保持原 `showPeerHistory(peer, ClearStack, msgId)` |
| 論壇群但從對話列表打開（`thread` 為空） | `topic == nullptr` → 走原路徑（落在群主頁；已知局限，與「無法從列表得知當前話題」一致） |
| 檢查點消息已刪 | `showTopic` → `showRepliesForMessage` 的 `MSG_ID_INVALID` 分支 → toast，不崩（`window_session_controller.cpp:1348+`） |

## 已知局限（不阻塞）

- 列表入口（無 `thread`）無法帶 topic；要覆蓋這一半，需要在 `dialogs_widget` 側把 `topicId` 一併記錄並在跳轉時用 `Data::ForumTopic` 反查——屬後續優化。
- 話題內 `showTopic` 會先把 section 切到該話題（`Way::ClearStack`），與既有「Jump to Beginning」行為一致。

## 驗證建議（外部構建後）

1. 論壇群內右鍵 → AyuGram → Jump to AG Checkpoint：應落到話題內檢查點消息。
2. 普通群：應落到群內檢查點消息。
3. 話題內清除檢查點後重複步驟 1：「Jump」入口應消失（`hasCheckpoint == false` 的早退分支）。
