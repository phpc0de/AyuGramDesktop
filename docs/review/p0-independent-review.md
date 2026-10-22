# P0 改动单独立复核报告（`docs/p0-build-fix-plan.md`）

- 复核对象：`docs/p0-build-fix-plan.md`（65 行，AyuGramDesktop dev HEAD）
- 仓库：`/root/workspace/AyuGramDesktop`（tdesktop v7.0.9 fork，api layer 228）
- 复核方式：**只读**。仅 `read` / `grep` / `sed` / `python3`（纯读取分析） / `codegraph query|callers|status`。
  未修改 `Telegram/` 下任何文件，未执行 git，未构建，未运行 docker。
- 标记约定：**【确定】** = 已在本 checkout 直接取证；**【无法确定】** = 需外部构建机或运行环境才能判定。

---

## 0. 总体 Verdict

> **可作为 P0 单独提交 —— 附 1 条必须先决策的偏离项（ADR-0002 决策 1(b)）。**

| 维度 | 结论 |
|---|---|
| 故障事实（`is_ayuNoforwards` 不存在） | **成立且强化**【确定】——我做了自动化全量交叉校验，见 §2.1 |
| 修复方案（两行合并为 `AyuNoForwards`） | **功能上正确**【确定】——三道门确实会打开，理由链完整 |
| 建议作为独立 P0 提交 | **可以**。改动面 = 单文件 4 行，重编面最小，风险可控 |
| 但需先决策 | **§3 问题 2（ADR 偏离）**：ADR-0002 要求改 `allowsForwarding()`，改动单选择改数据流 |
| 发现的额外问题 | **8 项**（1 项事实错误、1 项 ADR 偏离、6 项不完整/低危） |
| 同类隐患（第二处编译断点） | **`data_session.cpp` 内已排除**【确定】；全仓 MTP 访问器层已排除【确定】；非 MTP 类编译断点**无法确定**（需构建） |

---

## 1. 跑过的命令（全部只读）

```bash
pwd; ls /root/workspace/AyuGramDesktop; ls docs/
which codegraph; codegraph --help
codegraph status
codegraph query "isAyuNoForwards"
codegraph callers isAyuNoForwards
codegraph callers allowsForwarding

# 文本取证
sed -n '95,110p'  Telegram/SourceFiles/mtproto/scheme/api.tl
grep -n "^chat#41cbf256\|^channel#d49f34c6" Telegram/SourceFiles/mtproto/scheme/api.tl
grep -n "messages.toggleNoForwards" Telegram/SourceFiles/mtproto/scheme/api.tl
grep -rn "ayuNoforwards" . --exclude-dir=.git
grep -rn "is_ayu" . --exclude-dir=.git
grep -rni "ayunoforwards" Telegram/ --include=*.h --include=*.cpp
grep -rn "AyuNoForwards" Telegram/
grep -rn "allowsForwarding" .            # 全仓消费点
grep -rn "Flag::NoForwards\|ChatDataFlag::NoForwards\|ChannelDataFlag::NoForwards" Telegram/SourceFiles/
grep -rn "CHAT_FORWARDS_RESTRICTED" Telegram/
grep -rn "sharingDisallowed" Telegram/SourceFiles/
grep -rn "noForwards\|NoForwards\|noforwards" Telegram/SourceFiles/boxes/peers/edit_peer_info_box.cpp
grep -rn "forbidsForward" Telegram/SourceFiles/history/
grep -rni "ayu" Telegram/SourceFiles/mtproto/scheme/*.tl
grep -rn "f_ayu\|mtpdc_ayu\|MTPDayu\|vayu" Telegram/SourceFiles/

# 自动化交叉校验（python3 纯读取，不落盘）
python3 - <<'EOF' ... EOF   # (a) data_session.cpp 全部 144 个 MTP 访问器 vs schema
                           # (b) 全仓 MTPD 风格访问器 vs schema
                           # (c) 全仓 MTPD_/MTPC_ 类型名 vs schema
                           # (d) ayu/ 目录 MTP 请求构造器 vs api.tl
```

---

## 2. 逐条验证改动单的事实断言

### 2.1 故障事实 —— 成立，且比改动单更强【确定】

改动单 §1 称 `data.is_ayuNoforwards()` 在 api.tl（layer 228）中不存在、HEAD 编不过。我做了自动化全量交叉校验：

- 脚本抽取 `data_session.cpp` 中所有 `data.is_xxx()` / `data.vxxx()` 调用 → **144 个不同访问器**；
- 逐个与 `api.tl` + `mtproto.tl` 的字段声明集合比对；
- 结果：**仅 `is_ayuNoforwards` 一个缺失**，位于 **934、1098** 两行 —— 与改动单完全一致。

```
distinct MTP accessors used on `data.` in data_session.cpp: 144
NOT FOUND in api.tl/mtproto.tl schema:
  is_ayuNoforwards  lines [934, 1098]
```

补充全仓扫描（`\b(data|mtpData|object|result|update|...)\.(is_*|v*)\s*\(`，范围 `Telegram/SourceFiles/**/*.cpp|h|mm`）：

- 唯一真实命中仍是 `data_session.cpp:[934,1098] -> .is_ayuNoforwards()`；
- 其余 35 组命中全部是假阳性，已逐类确认：
  - `.value()` / `.valid()` / `.is_ok()` / `.value_or()` —— `std::optional` / `QVariant` / crypto result；
  - `.vsrp_B()`（`core_cloud_password.cpp:196`、`intro_qr.cpp:522`）—— **真实访问器**，`srp_B:flags.2?bytes` 声明于 `api.tl:671`，只是我的字段名正则未覆盖该形式。

> **结论【确定】：全仓范围内，fork 引入且「声明/使用但定义缺失」的 MTP 访问器只有 `is_ayuNoforwards` 一处。** 改动单 §5「是否还有第二处编译断点」在 MTP 访问器这一维度上可以关闭。

### 2.2 逐条断言核对表

| # | 改动单断言（`docs/p0-build-fix-plan.md`） | 复核结果 | 证据 |
|---|---|---|---|
| 1 | `:933-934`（ProcessChat）调用 `is_ayuNoforwards()` | ✅ **正确**【确定】 | `data_session.cpp:933-934` |
| 2 | `:1097-1098`（ProcessChannel）调用 `is_ayuNoforwards()` | ✅ **正确**【确定】 | `data_session.cpp:1097-1098` |
| 3 | api.tl `chat#41cbf256`（`:102`）只有 `noforwards:flags.25?true` | ✅ **正确**【确定】 | `api.tl:102` |
| 4 | api.tl `channel#d49f34c6`（`:103`）只有 `noforwards:flags.27?true` | ❌ **行号错误**【确定】 | 实际在 **`api.tl:104`**；`:103` 是 `chatForbidden#6592a1a7` |
| 5 | 全仓 grep 无 `ayuNoforwards` 定义 | ✅ **正确**【确定】 | 见 §2.1 |
| 6 | `.gitmodules:41-42` 的 `cmake` 子模块未初始化 | ⚠️ **基本正确**【确定】 | 完整 stanza 是 `:40-42`（`[submodule "cmake"]` / `path = cmake` / `url`）；`ls cmake/`、`ls Telegram/codegen/` 均为空 |
| 7 | 消息级先例 `history_item_helpers.cpp:932-933` | ✅ **正确**【确定】 | `:932` 注释掉原生行，`:933` 改为 `Flag::AyuNoForwards` |
| 8 | 用户级先例 `data_user.cpp:1104-1107` | ✅ **正确**【确定】 | `setNoForwardsFlags(update.is_noforwards_my_enabled(), update.is_noforwards_peer_enabled())` |
| 9 | 用户级先例内部走私有 flag，`data_user.cpp:695-702` | ⚠️ **范围偏移**【确定】 | `isAyuNoForwards()` 在 `:694-697`；`setNoForwardsFlags()` 实为 **`:699-705`**，`:695-702` 只覆盖其前半段 |
| 10 | 掩码已就绪：`data_session.cpp:923`（Chat）、`:1047`（Channel）已含 `Flag::AyuNoForwards` | ✅ **正确**【确定】 | `:923`、`:1047` 均为 `\| Flag::AyuNoForwards;` |
| 11 | 标志位存储 `data_chat.h:27` `AyuNoForwards = (1 << 31)` | ✅ **正确**【确定】 | 见 §3 问题 5（channel 侧遗漏） |
| 12 | `data_chat.cpp:71-73` / `data_channel.cpp:783-785` `isAyuNoForwards()` | ✅ **正确**【确定】 | 均为 3 行完整函数体 |
| 13 | `data_chat.cpp:75-77` / `data_channel.cpp:787-789` `allowsForwarding()` | ✅ **正确**【确定】 | `return !(flags() & Flag::NoForwards);` |
| 14 | `ayu_forward.cpp:244` `isFullAyuForwardNeeded` | ✅ **正确**【确定】 | `item->from()->isAyuNoForwards() \|\| item->history()->peer->isAyuNoForwards()` |
| 15 | `apiwrap.cpp:583` `CHAT_FORWARDS_RESTRICTED` | ✅ **正确**【确定】 | `apiwrap.cpp:583`（注意：文件在 `Telegram/SourceFiles/apiwrap.cpp`，**不在** `api/` 目录下） |
| 16 | `history_view_context_menu.cpp:2662` `AddSelectRestrictionAction` | ✅ **正确**【确定】 | `if ((!peer->isAyuNoForwards() && !AyuForward::isAyuForwardNeeded(item)) \|\| item->isSponsored())` |
| 17 | `edit_peer_info_box.cpp:1081` 读方向、`:2916` 比较 | ✅ **正确**【确定】 | 见 §4 |
| 18 | `info_media_provider.cpp:111-114` 受限提示 UI | ⚠️ **不完整**【确定】 | 见 §3 问题 4 |
| 19 | `history_inner_widget.cpp:658-661` 受限提示 UI | ✅ **正确**【确定】 | `_sharingDisallowed` 群/频道分支 |
| 20 | ProcessChannel 表达式无结尾分号 | ✅ **正确**【确定】 | `:1098` 无 `;`，表达式续于 `:1099` |
| 21 | 改后 `Flag::NoForwards` 成为死分支 | ✅ **正确且强化**【确定】 | 见 §3.2，全仓仅 `:933/:1097` 两处写入 |

---

## 3. 发现的问题清单（8 项）

### 问题 1 —— api.tl 行号错误【中｜低危·文档】
- **位置**：`docs/p0-build-fix-plan.md:12`
- **现状**：`channel#d49f34c6`（`:103`）只有 `noforwards:flags.27?true`
- **实况**：`channel#d49f34c6` 位于 **`Telegram/SourceFiles/mtproto/scheme/api.tl:104`**；`:103` 是 `chatForbidden#6592a1a7 id:long title:string = Chat;`
- **原因/影响**：不改变结论（`noforwards:flags.27?true` 确实存在），但会让外部构建机复核者定位错行。
- **建议**：改为 `api.tl:102`（chat）与 `api.tl:104`（channel）。

### 问题 2 —— 与 ADR-0002 决策 1(b) 的偏离【中｜需决策】
- **位置**：`docs/adr/0002-unlock-restricted-chats.md` 决策 1 vs `docs/p0-build-fix-plan.md` §3/§4/§5
- **ADR 原文**（决策 1）：
  > peer 級對齊：Chat/Channel 處理時把服務端 `noforwards` **同時**映射到 `Flag::AyuNoForwards`（修正 `data_session.cpp:934,1098` …），**`allowsForwarding()` fork 版返回 `true`**（保留 AyuNoForwards 供 AyuForward 分流判定）。
- **改动单做法**：只改数据流（`NoForwards` 不再被置位 ⇒ `!(flags() & Flag::NoForwards)` 恒真），并明确要求 `data_chat.cpp:76` / `data_channel.cpp:788` 的检查**保留不动**（改动单 §5 第 2 条）。
- **复核判定**：**功能等价，但机制不同，且 ADR 明确要求了 `allowsForwarding()` 改为 `return true`。**
- **风险（合并脆弱性）**：若将来上游合并重新引入 `| (data.is_noforwards() ? Flag::NoForwards : Flag())`，三道门会**静默重新关闭**。ADR 的 `return true` 写法对合并免疫；改动单的做法不免疫。
- **建议（二选一，必须在提交前定）**：
  - **方案 A（推荐，与 ADR 一致）**：除数据流改动外，同时把
    - `Telegram/SourceFiles/data/data_chat.cpp:75-77` → `return true;`
    - `Telegram/SourceFiles/data/data_channel.cpp:787-789` → `return true;`
    并同步把 ADR 0002 决策 1 的「同時映射」措辞改为「仅映射 AyuNoForwards」。共 2 文件 4 行，仍是干净的小提交。
  - **方案 B（保持改动单原样）**：在改动单 §5 增写「刻意偏离 ADR-0002 决策 1(b) 的理由 + 上游重新引入 `Flag::NoForwards` 时的回归风险」，并请 ADR 0002 走一次修订确认。

### 问题 3 —— 「三门全开」漏列 6/7 消费点【低｜文档完整性】
- **位置**：`docs/p0-build-fix-plan.md` §4 表格第 3 行
- **现状**：只写「`data_chat.cpp:75-77` / `data_channel.cpp:787-789` … 三门全开」，未枚举 `PeerData::allowsForwarding()` 的实际调用方。
- **实况**（`codegraph callers allowsForwarding` = 8，去重定义后 7 个调用点，全部在本 checkout 存在）：

| # | 位置 | 门 | 改后行为 |
|---|---|---|---|
| 1 | `data/data_peer.cpp:772` `canExportChatHistory()` | **导出群历史（改动单完全未提）** | 恒可导出 |
| 2 | `history/history_item.cpp:3055` `allowsForward()` | 转发 | 恒可转发（仍受 `forbidsForward()` 约束，但后者已硬编码 `false`） |
| 3 | `history/history_inner_widget.cpp:3971` `hasCopyRestriction()` | 复制 | 恒 false |
| 4 | `history/view/history_view_list_widget.cpp:5975` `CopyRestrictionTypeFor()` | 下载 | 恒可下载 |
| 5 | `core/click_handler_types.cpp:666` | 链接/URL 转发上下文菜单 | 恒可用 |
| 6 | `iv/iv_instance.cpp:158` `CanShareMarkdownItem()` | Instant View 分享 | 恒可用 |
| 7 | `info/media/info_media_provider.cpp:89` `hasSelectRestriction()` | 多选受限提示 | 恒不提示 |

- **重点提醒**：**#1 导出群历史是改动单 §4 表格里完全没有的行为变化**。ADR-0002 的「三道门」只列了转发/复制/下载，导出是第四个受影响面，外部验证清单应覆盖。
- **建议**：把上表补进 §4，并注意 `data_peer.cpp:772` 的实际语义是 `canExportChatHistory()`（不是改动单正文写的泛指 `allowsForwarding` 消费）。

### 问题 4 —— 受限提示 UI 只引了响应式生产者，漏引真正的判定函数【低｜文档完整性】
- **位置**：`docs/p0-build-fix-plan.md` §4 表格第 7 行（引用 `info_media_provider.cpp:111-114`）
- **实况**：`info_media_provider.cpp:111-114` 只是**触发重算的响应式生产者**：
  ```cpp
  auto noForwards = chat
      ? Data::PeerFlagValue(chat, ChatDataFlag::NoForwards)
      : Data::PeerFlagValue(channel, ChannelDataFlag::NoForwards) | rpl::type_erased;
  ```
  真正的判定在 `info_media_provider.cpp:86-97`：
  ```cpp
  bool Provider::hasSelectRestriction() {
      if (_peer->session().frozen()) { return true; }
      else if (_peer->allowsForwarding()) { return false; }   // ← :89
      else if (const auto chat = _peer->asChat()) { return !chat->canDeleteMessages(); }
      else if (const auto channel = _peer->asChannel()) { return !channel->canDeleteMessages(); }
      return true;
  }
  ```
- **影响**：改后 `_peer->allowsForwarding()` 恒真 ⇒ `:89` 直接短路返回 `false`，`:111-114` 的 flag 值**对结果不再有任何影响**（只用于触发重算）。改动单把因果归到 `:111-114`，方向搞反了。
- **建议**：§4 该行改为引用 `info_media_provider.cpp:86-97`（尤其 `:89`），并注明 `:111-114` 的 producer 在改后成为「仅触发器」。

### 问题 5 —— 漏引 channel 侧的私有标志存储位【低｜文档完整性】
- **位置**：`docs/p0-build-fix-plan.md` §4 表格第 1 行（只引 `data_chat.h:27`）
- **实况**：peer 级私有位有两处定义，channel 侧改动单完全没提：
  - `Telegram/SourceFiles/data/data_chat.h:27` → `AyuNoForwards = (1 << 31),`（`enum class ChatDataFlag`，**无显式底层类型 → int**）
  - `Telegram/SourceFiles/data/data_channel.h:93` → `AyuNoForwards = (1ULL << 63),`（`enum class ChannelDataFlag : uint64`，`data_channel.h:43`）
- **建议**：§4 补 `data_channel.h:93` + `data_channel.h:415`（`isAyuNoForwards()` 声明）。

### 问题 6 —— §5「未取证项 1」已被本次复核解决，应回填【低｜文档】
- **位置**：`docs/p0-build-fix-plan.md:57`（「`edit_peer_info_box` 保存路径是否发送真实 `noforwards`（只读了读方向 `:1081`）」）
- **结论【确定】**：**是**，走真实 TL。详见 §4。
- **建议**：把该项从「未取证」移到已取证，并附 `edit_peer_info_box.cpp:2919-2925` 证据。

### 问题 7 —— §5「未取证项 3（是否还有第二处编译断点）」在 MTP 维度可关闭【低｜文档】
- **位置**：`docs/p0-build-fix-plan.md:59`
- **结论【确定】**：`data_session.cpp` 内 144 个 MTP 访问器仅 1 个缺失；全仓 MTP 访问器 / `ayu/` 目录 MTP 请求构造器均无缺失。
- **仍无法确定**：非 MTP 类编译断点（模板实例化、`lib_*` 子模块缺失导致的符号错误等）**必须**在外部构建机判定——本 checkout 的 `cmake`、`Telegram/codegen`、`Telegram/lib_*` 子模块均未初始化。

### 问题 8 —— `ChatDataFlag::AyuNoForwards` 的 `(1 << 31)` 移入符号位【低｜潜伏·非本次引入】
- **位置**：`Telegram/SourceFiles/data/data_chat.h:18-27`
  ```cpp
  enum class ChatDataFlag {          // ← 无显式底层类型，默认 int
      Left = (1 << 0),
      ...
      NoForwards = (1 << 8),
      AyuNoForwards = (1 << 31),     // ← 移入 int 的符号位
  };
  ```
- **原因**：`1 << 31` 在 32 位有符号 `int` 上是标准未定义行为（C++20 前）；对比 channel 侧 `data_channel.h:93` 用 `1ULL << 63` 配合 `enum class ChannelDataFlag : uint64`，写法是良定义的。两侧不对称。
- **实际风险**：GCC/Clang/MSVC 均按补码实现为 `INT_MIN`，实际不会出问题；但 `-fstrict-overflow` / UBSan / 未来 `uint32` 化重构下是隐患。
- **建议**：**不要放进 P0 提交**（扩大回归面）。单独记一条低优先技术债，把 `ChatDataFlag` 显式化为 `enum class ChatDataFlag : uint32` 或直接改用 `1ULL << 31`。

---

## 4. 语义复核 + 读写闭环

### 4.1 改后 `Flag::NoForwards` 是否真的永不再置位？——是【确定】

全仓穷举 `Flag::NoForwards` / `ChatDataFlag::NoForwards` / `ChannelDataFlag::NoForwards` 的所有出现，**写入点只有两处**：

```
Telegram/SourceFiles/data/data_session.cpp:933    | (data.is_noforwards() ? Flag::NoForwards : Flag())
Telegram/SourceFiles/data/data_session.cpp:1097   | (data.is_noforwards() ? Flag::NoForwards : Flag())
```

其余全部是**读取**（`data_chat.cpp:76`、`data_channel.cpp:788`、`info_media_provider.cpp:111,114`、`history_inner_widget.cpp:658,661`）或注释（`history_item_helpers.cpp:932`）。

→ 改后 `ChatData::allowsForwarding()` / `ChannelData::allowsForwarding()` 恒返回 `true`。§2.2 断言 21 成立。
→ 同时 `flagsMask`（`data_session.cpp:922,1046`）仍含 `Flag::NoForwards`，`setFlags((flags() & ~flagsMask) | flagsSet)`（`:935`、`:1118`）会持续把该位清零 —— 改动单 §4 第 2 行的说法正确。

### 4.2 语义影响是否与需求（ADR-0002）一致？——一致【确定】

| 环节 | 位置 | 改后 |
|---|---|---|
| 转发 | `history_item.cpp:3051-3057` `allowsForward()` | peer 门打开；`forbidsForward()` 早已硬编码 `false`（`history_item.cpp:3124-3126`）⇒ 恒 true |
| 复制 | `history_inner_widget.cpp:3970-3971` `hasCopyRestriction` | 恒 false |
| 下载 | `history_view_list_widget.cpp:5972-5975` | 恒放开 |
| 导出群历史 | `data_peer.cpp:770-772` `canExportChatHistory()` | **恒可导出（改动单漏列，见问题 3）** |
| 多选受限提示 | `info_media_provider.cpp:86-97` | 恒不提示（`:89` 短路） |
| `setupSharingDisallowed` | `history_inner_widget.cpp:657-662` | `_sharingDisallowed` 恒 false ⇒ `hasSelectRestriction()`（`:878-890`）在 `:883` 恒返回 false |
| 受限内容菜单项 | `history_view_context_menu.cpp:2657-2665` | 受保护群出现该入口（预期） |
| 转发执行分流 | `ayu_forward.cpp:243-245` `isFullAyuForwardNeeded` | 受保护群 ⇒ 走「下载-重传」，绕开服务端 `CHAT_FORWARDS_RESTRICTED`（`apiwrap.cpp:583`） |

**与 ADR-0002 决策 2/3 一致。** 用户级已在 fork 中先行放开（`data_user.cpp:690-692` `UserData::allowsForwarding() = true`），peer 级是最后一块拼图 —— 改动单补的正是这一块。

### 4.3 读写闭环 —— 管理端开关【确定】

**读方向**（改动单已引）：`edit_peer_info_box.cpp:1081`
```cpp
.noForwards = _peer->isAyuNoForwards(),
```

**写方向**（本次复核新取证，关闭 §5 未取证项 1）：`edit_peer_info_box.cpp:2914-2925`
```cpp
void Controller::saveForwards() {
	if (!_savingData.noForwards
		|| *_savingData.noForwards == _peer->isAyuNoForwards()) {
		return continueSave();
	}
	using Flag = MTPmessages_ToggleNoForwards::Flag;
	_api.request(MTPmessages_ToggleNoForwards(
		MTP_flags(Flag()),
		_peer->input(),
		MTP_bool(*_savingData.noForwards),
		MTPint()
	)).done([=](const MTPUpdates &result) { ... });
}
```
对应真实 TL —— `Telegram/SourceFiles/mtproto/scheme/api.tl:2606`：
```
messages.toggleNoForwards#b2081a35 flags:# peer:InputPeer enabled:Bool request_msg_id:flags.0?int = Updates;
```

**闭环判定**：写入的是**服务端真实 `noforwards`**（chat `flags.25` / channel `flags.27`），服务端回传后经 `ProcessChat`/`ProcessChannel`（改后）映射到 `Flag::AyuNoForwards` → `isAyuNoForwards()` 为真 → 开关显示为 ON。**闭环成立。**

> 顺带说明【确定】：若不做本 P0 改动而让 `:934/:1098` 存在（假设能编译），写入服务端成功但读回只落到 `Flag::NoForwards`，`isAyuNoForwards()` 恒 false ⇒ **管理端开关会永远显示为关闭、且每次保存都重发 `enabled=false`**。也就是说本 P0 改动**修复了一个既存的读写不对称 bug**，不只是修编译。

`saveForwards()` 的 `!_savingData.noForwards` 提前返回是 `std::optional` 未赋值的保护，与同文件 `:2496`（username）、`:2604`（title）、`:2680`（description）等 10 余处同款写法一致 —— **不是缺陷**。

### 4.4 用户级逻辑是否受影响？——不受影响【确定】

`window_peer_menu.cpp:1525-1603` `addToggleNoForwards()`：
- 守卫 `_peer->asUser()` + `isInaccessible/isBot/isServiceUser/isSelf` —— **仅用户 peer**；
- 读 `user->isAyuNoForwards()`（`:1557`）→ `data_user.cpp:694-697` 读 `UserDataFlag::NoForwardsMyEnabled | NoForwardsPeerEnabled`；
- 写同样发 `MTPmessages_ToggleNoForwards`，但服务端存进 **`userFull.noforwards_my_enabled` / `noforwards_peer_enabled`**，由 `data_user.cpp:1104-1107` 回读。

→ 用户级 flag 与 `ChatDataFlag`/`ChannelDataFlag` **完全正交**，本次改动一行都不碰。**无需为它做任何补充改动。**

---

## 5. 同类隐患扫描（任务 2）

### 5.1 Ayu 私有 MTP 访问器 / 字段

| 检索式 | 范围 | 结果 |
|---|---|---|
| `grep -rn "ayuNoforwards" . --exclude-dir=.git` | 全仓 | 仅 `data_session.cpp:934,1098` 2 处代码 + 5 处文档 |
| `grep -rn "is_ayu" . --exclude-dir=.git` | 全仓 | 同上，无其他 |
| `grep -rni "ayu" Telegram/SourceFiles/mtproto/scheme/*.tl` | api.tl + mtproto.tl | **0 命中** —— schema 本身干净，无 Ayu 私有 TL 字段 |
| `grep -rn "f_ayu\|mtpdc_ayu\|MTPDayu\|vayu\|\.ayu_" Telegram/SourceFiles/` | 全 SourceFiles | **0 命中**（`ayu_` 仅出现在 include 路径与 `tr::ayu_*` 文案 key） |

### 5.2 Ayu 私有 flag 位 —— 声明/定义/使用**全部齐备**【确定】

`codegraph query isAyuNoForwards` 返回 5 个定义，每个都有配套声明：

| 类型 | 枚举定义 | 访问器声明 | 访问器定义 | 状态 |
|---|---|---|---|---|
| `MessageFlag` | `data_types.h:273` `(1ULL << 17)` | `history_item.h:221` | `history_item.cpp:2139-2140` | ✅ |
| `UserDataFlag` | `NoForwardsMyEnabled/NoForwardsPeerEnabled` | `data_user.h:218` | `data_user.cpp:694-697` | ✅ |
| `ChatDataFlag` | `data_chat.h:27` `(1 << 31)` | `data_chat.h:104` | `data_chat.cpp:71-72` | ✅（但见问题 8） |
| `ChannelDataFlag` | `data_channel.h:93` `(1ULL << 63)` | `data_channel.h:415` | `data_channel.cpp:783-784` | ✅ |
| `PeerData`（聚合） | — | `data_peer.h:325` | `data_peer.cpp:1718-1726` | ✅ |

→ **未发现「声明/使用但定义缺失」的私有 flag。**

### 5.3 fork 目录 `ayu/` 的 MTP 请求构造器【确定 · 无缺失】

扫描 `Telegram/SourceFiles/ayu/**` 内所有 `MTPxxx(` 请求构造器并与 api.tl 比对，2 个疑点**均已证伪**：

| 疑点 | 位置 | 判定 |
|---|---|---|
| `MTPstring()` | `ayu/features/forward/ayu_sync.cpp:723` | **合法**。`Telegram/SourceFiles/mtproto/core_types.h:179` `using MTPstring = tl::string_type;`；`api.tl:2542` `messages.uploadMedia#14967978 flags:# business_connection_id:flags.0?string peer:InputPeer media:InputMedia` —— 该槽位正是可选 string，占位写法与 `create_poll_box.cpp:1691-1693` 等 10 处上游调用完全一致 |
| `MTPmessages_ReadHistory(` | `ayu/utils/telegram_helpers.cpp:428` | **合法**。`api.tl:2464` `messages.readHistory#e306d3a peer:InputPeer max_id:int = messages.AffectedMessages;` |

### 5.4 「未找到 + 命令 + 范围」声明

以下几类**未找到**任何同类隐患：

- **Ayu 前缀的 TL 构造器/常量**：`grep -rn "f_ayu\|mtpdc_ayu\|MTPDayu\|mtp_ayu" Telegram/SourceFiles/` → **0 命中**。
- **Ayu 前缀的 TL 字段**：见 5.1 第 3 行 → **0 命中**。
- **第二个 phantom MTP 访问器**：全仓自动化扫描（§2.1）→ **未找到**；仅 `is_ayuNoforwards` 一个。
- **非 MTP 类编译断点**：**无法确定**。本 checkout 的 `cmake/`、`Telegram/codegen/`、`Telegram/lib_base|lib_ui|lib_tl|lib_storage` 子模块全部未初始化（`ls` 为空），无法在不违反「禁构建」约束的前提下判定。范围声明：仅覆盖 `Telegram/SourceFiles/**` 的 **MTP 访问器层与 MTP 类型/请求层**；不覆盖模板实例化、链接期符号、CMake 配置。

---

## 6. Verdict（明确结论）

### 6.1 该改动是否可作为 P0 单独提交？

**可以。** 理由【确定】：
1. 故障事实成立且经全量自动化交叉校验强化（144 访问器仅 1 缺失）；
2. 修复面 = 单文件 4 行（`data_session.cpp:933-934,1097-1098`），重编面最小；
3. 修复后 `Flag::NoForwards` 全仓零写入点，三道门 + 导出 + 多选提示的语义变化**方向与 ADR-0002 决策 1/2/3 一致**；
4. 读写闭环成立（`edit_peer_info_box.cpp:2919-2925` 真实 TL），且修复了一个既存的读写不对称 bug；
5. 无同类 phantom 访问器残留。

### 6.2 提交前**必须**补充/决策的点

| 优先级 | 项 | 位置 | 理由 |
|---|---|---|---|
| **P0-必做** | 问题 2：ADR-0002 决策 1(b) 偏离需决策 | `data_chat.cpp:75-77`、`data_channel.cpp:787-789` | ADR 明确要求 `allowsForwarding()` fork 版返回 `true`；不处理则留下上游合并回归风险。推荐方案 A（顺带改 2 行） |
| **P0-必做** | 问题 1：api.tl 行号纠正 | `docs/p0-build-fix-plan.md:12` | `:103` → `:104`，避免构建机复核者定位错行 |
| **P0-建议** | 问题 6/7：回填两条已关闭的「未取证项」 | `docs/p0-build-fix-plan.md:57,59` | 写路径已证实；第二编译断点在 MTP 维度已排除 |
| **P1-建议** | 问题 3：补 7 个 `allowsForwarding()` 消费点 | `docs/p0-build-fix-plan.md` §4 | 特别是**导出群历史** `data_peer.cpp:772` 是完全未列的行为变化 |
| **P1-建议** | 问题 4：`hasSelectRestriction()` 判定函数定位 | 同上 | `:111-114` 是触发器，真正短路在 `:89` |
| **P1-建议** | 问题 5：补 channel 侧存储位 | 同上 | `data_channel.h:93,415` |
| **P2-技术债** | 问题 8：`(1 << 31)` 移入符号位 | `data_chat.h:27` | **不要**放进 P0 提交；单独记 |

### 6.3 必须在外部构建机验证的项目清单

| # | 验证项 | 判据 |
|---|---|---|
| 1 | 子模块对齐 | `cmake`、`Telegram/codegen`、`Telegram/lib_base`、`Telegram/lib_ui`、`Telegram/lib_tl`、`Telegram/lib_storage` 已初始化且 commit 为主仓记录值 |
| 2 | 编译 | Windows：`cmake --build out --config Debug --target Telegram`（`AGENTS.md:48-54`）；WSL/Linux：`Telegram/build/docker/centos_env/build_debug.sh`（`AGENTS.md:25`） |
| 3 | 错误消失 | `data_session.cpp` 934/1098 的 `no member named 'is_ayuNoforwards'` 两处错误消失 |
| 4 | **无第二编译断点** | 全量链接通过（本地无法判定，必须外部验证） |
| 5 | 目标文件重编面 | 仅 `data/data_session.cpp`（+ 方案 A 下的 `data_chat.cpp`、`data_channel.cpp`）及其依赖 TU |
| 6 | 读回验证 | 对一个真实 `noforwards` 群/频道：登录后 `isAyuNoForwards()` 为真、管理端开关显示 ON |
| 7 | 写回验证 | 管理端关闭开关 ⇒ 服务端生效 ⇒ 重新拉取后开关仍为 OFF（闭环） |
| 8 | 三门回归 | 受保护群：转发菜单出现、复制不 toast、下载可用 |
| 9 | **导出回归（问题 3）** | 受保护群「导出聊天记录」入口出现；实际调用若被服务端拒绝，应走 `apiwrap.cpp:583` 的 `CHAT_FORWARDS_RESTRICTED` toast 而非崩溃 |
| 10 | 重传分流 | 受保护群转发实际走 AyuForward 下载-重传（`ayu_forward.cpp:243-245`），不直发 `forwardMessages` |
| 11 | 多选提示 | 受保护群多选消息不再出现「禁止转发」提示 |
| 12 | 用户级不回归 | 个人「禁止转发」开关（`window_peer_menu.cpp:1526`）行为与改动前一致 |
| 13 | 回填材料 | 构建日志首个错误行 + 完整错误块，交回本会话迭代（ADR-0004 §8.6） |

---

## 7. 附：本次复核明确**未**触碰的约束

- 未修改 `Telegram/` 下任何文件；本次仅新建 `docs/review/p0-independent-review.md`。
- 未执行 `git`（无 `status`/`log`/`diff`）；未构建；未运行 docker。
- 所有 `python3` 脚本仅读取文件并在 stdout 输出，**不写任何文件**。

## 主线程抽查（2025-09-27）
- ✅ 采信问题 1：`api.tl:102 chat / :103 chatForbidden / :104 channel` —— 改单原写 `:103` 为错，已修。
- ✅ 采信问题 7：`edit_peer_info_box.cpp:2914-2925` `saveForwards()` 确实发 `MTPmessages_ToggleNoForwards`（`MTP_bool(*_savingData.noForwards)`），读写闭环成立。
- ✅ 采信问题 4/8：采纳「补改 allowsForwarding() 为 `return true;`」；`AyuNoForwards = (1<<31)` UB 记为独立技术债。
- ✅ 采信问题 5：`data_peer.cpp:772` `canExportChatHistory()` 确为第 7 个消费点。
