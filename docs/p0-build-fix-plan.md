# P0 改动单 — 修复 HEAD 编译失败（`is_ayuNoforwards` phantom accessor）

> 性质：**只读审计产出的改动说明**（/grill 阶段不落应用代码）。按 ADR-0004，由外部构建机应用并验证。
> 目标：让 `dev` HEAD 恢复可编译，并顺带落地 ADR-0002 决策 1（peer 级 noforwards → 私有 `AyuNoForwards` 映射）。

## 1. 故障事实
`Telegram/SourceFiles/data/data_session.cpp` 两处调用了 **api.tl（layer 228）中不存在的访问器** `data.is_ayuNoforwards()`：

- `data_session.cpp:933-934`（`ProcessChat`）
- `data_session.cpp:1097-1098`（`ProcessChannel`）

api.tl 的 `chat#41cbf256`（`:102`）只有 `noforwards:flags.25?true`、`channel#d49f34c6`（`:104`；`:103` 是 `chatForbidden`）只有 `noforwards:flags.27?true`，全仓库 grep 无 `ayuNoforwards` 定义（`.gitmodules:41-42` 的 `cmake` 子模块未初始化，无法从 codegen 侧补出该访问器）→ 生成的 `MTPDchat`/`MTPDchannel` 无此方法，**HEAD 编不过**。

## 2. 分支意图与先例
- 消息级已按「把服务端 noforwards 映射进私有 `AyuNoForwards`」改造：`history_item_helpers.cpp:932-933`
  ```cpp
  // | ((flags & MTP::f_noforwards) ? Flag::NoForwards : Flag())   // 原生那行被注释
  | (flags & MTP::f_noforwards ? Flag::AyuNoForwards : Flag())    // Ayu 私有标志
  ```
- 用户级同构先例：`data_user.cpp:1104-1107` 把 TL 的 `noforwards_my_enabled/peer_enabled` 收进 `setNoForwardsFlags(...)`（内部走私有 flag，`data_user.cpp:695-702`）。
- 掩码已就绪：`data_session.cpp:923`（Chat）与 `:1047`（Channel）的 `flagsMask` **已包含** `Flag::AyuNoForwards` → 只需修 `flagsSet` 表达式。

## 3. 精确改动（2 行删除 + 2 行改写，两处结构相同）
文件：`Telegram/SourceFiles/data/data_session.cpp`

**ProcessChat（当前 :933-934）**
```cpp
// 现状（无法编译）
			| (data.is_noforwards() ? Flag::NoForwards : Flag())
			| (data.is_ayuNoforwards() ? Flag::AyuNoForwards : Flag());
// 改为
			| (data.is_noforwards() ? Flag::AyuNoForwards : Flag());
```

**ProcessChannel（当前 :1097-1098）**
```cpp
// 现状（无法编译）
				| (data.is_noforwards() ? Flag::NoForwards : Flag())
				| (data.is_ayuNoforwards() ? Flag::AyuNoForwards : Flag())
// 改为
				| (data.is_noforwards() ? Flag::AyuNoForwards : Flag())
```
（注意 ProcessChannel 的表达式无结尾分号，合并时保持原样。）

### 3.1 补改（复核后追加：与 ADR-0002 决策 1 对齐）
仅改数据流会让 `Flag::NoForwards` 成为**死位**——上游日后若重新写入该位，三道门会静默关闭。按已定稿的 ADR-0002「`allowsForwarding()` fork 版返回 `true`」，一并改死两处判定（共 2 文件 4 行，仍是小提交）：

- `Telegram/SourceFiles/data/data_chat.cpp:75-77`：`return true;`（原 `return !(flags() & Flag::NoForwards);`）
- `Telegram/SourceFiles/data/data_channel.cpp:787-789`：同理

### 3.2 复核修正（主线程抽查，2025-09-27）
- 独立复核（`docs/review/p0-independent-review.md`）脚本交叉校验 `data_session.cpp` 全部 144 个 `data.is_*/v*` 访问器 vs api.tl：**仅 `is_ayuNoforwards`（:934/:1098）缺失** → MTP 维度可认定无第二处同类断点（非 MTP 类无法确定，子模块未初始化）。
- 写路径闭环已取证（关闭原未取证项）：`boxes/peers/edit_peer_info_box.cpp:2914-2925` `saveForwards()` 发送真实 `MTPmessages_ToggleNoForwards`（api.tl:2606），读方向 `:1081` 读 `_peer->isAyuNoForwards()` → 闭环成立。**同时暴露既有缺陷**：未修 P0 时该标志恒 false，管理端开关永远显示关闭且每次重发 `enabled=false`。
- 技术债（不进 P0）：`Flag::AyuNoForwards = (1 << 31)`（`data_chat.h:27`，未指定底层类型的 enum → int，移入符号位属 UB，且与 channel 侧 `1ULL<<63` 不对称）。
- 影响面补充：`PeerData::allowsForwarding()` 实际有 7 个消费点，除三门外含 `data_peer.cpp:772` `canExportChatHistory()`（导出群历史将变为可用，需外部机回归确认服务端态度）。

## 4. 改动后行为链（已逐点取证）
| 环节 | 位置 | 效果 |
|---|---|---|
| 标志位存储 | `data_chat.h:27` `AyuNoForwards = (1 << 31)`；`data_chat.cpp:71-73` / `data_channel.cpp:783-785` `isAyuNoForwards()` | 群/频道的 noforwards 落到私有标志 |
| 掩码清理 | `data_session.cpp:923,1047` | 更新时该位可正确清零 |
| 转发/复制/下载三门 | `data_chat.cpp:75-77` / `data_channel.cpp:787-789` `allowsForwarding() = !(flags() & Flag::NoForwards)` | `Flag::NoForwards` 不再被置位 → 返回 true → 三门全开（ADR-0002 决策 2） |
| 转发执行分流 | `ayu_forward.cpp:244` `isFullAyuForwardNeeded` = `item->from()->isAyuNoForwards() \|\| item->history()->peer->isAyuNoForwards()` | 受保护群的转发自动走「下载-重传」，绕开服务端 `CHAT_FORWARDS_RESTRICTED`（`apiwrap.cpp:583`） |
| 受限内容菜单项 | `history_view_context_menu.cpp:2662` `AddSelectRestrictionAction` | 受保护群开始出现该入口（预期行为） |
| 群设置开关 | `edit_peer_info_box.cpp:1081` `.noForwards = _peer->isAyuNoForwards()`、`:2916` 比较 | 管理端读写仍是 Ayu 语义（保存时发真实 TL flag，需实现时确认写入路径） |
| 受限提示 UI（行为变化） | `info_media_provider.cpp:111-114`、`history_inner_widget.cpp:658-661` 读 `ChatDataFlag::NoForwards/ChannelDataFlag::NoForwards` | 改后不再显示「禁止转发」提示——即需求 2/3/4 的预期解锁 |

## 5. 未取证 / 需实现时确认
- ~~`edit_peer_info_box` 保存路径是否发送真实 `noforwards`~~ → 已关闭，见 §3.2。
- `data_chat.cpp:76` / `data_channel.cpp:788` 的死分支：按 §3.1 直接改 `return true;`。
- 是否还有第二处**非 MTP 类**编译断点（无法本地构建验证，ADR-0004；MTP 类已由 §3.2 交叉校验关闭）。

## 6. 外部验证清单（按 ADR-0004 §8.6 模板）
1. 子模块 commit 需对齐：`cmake`、`Telegram/lib_base`、`Telegram/lib_ui`、`Telegram/lib_tl`、`Telegram/lib_storage`、`Telegram/codegen`（本 checkout 子模块未初始化，以主仓记录为准）。
2. 命令：Windows `cmake --build out --config Debug --target Telegram`（AGENTS.md:48-54）；WSL/Linux `Telegram/build/docker/centos_env/build_debug.sh`（:25）。
3. 预期：`data_session.cpp` 两处错误消失；target 链接通过；重编面仅 `data/data_session.cpp` 及其依赖 TU。
4. 回填：构建日志中的首个错误行 + 完整错误块，交回本会话迭代。
