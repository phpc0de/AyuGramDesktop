# R1 对抗式代码审查报告 — P0 数据层（AyuGramDesktop / dev）

- 仓库：`/root/workspace/AyuGramDesktop`，分支 `dev`，改动全部未提交（工作区）
- 审查范围（P0，4 文件）：
  - `Telegram/SourceFiles/data/data_session.cpp`
  - `Telegram/SourceFiles/data/data_chat.cpp`
  - `Telegram/SourceFiles/data/data_channel.cpp`
  - `Telegram/SourceFiles/data/data_chat.h`（只读参考，未在 diff 中）
- 审查性质：只读。未修改任何源文件、未执行 git 写操作、未构建。唯一写入为本报告。
- 结论摘要：**确定 BUG 3 个（高 1 / 中 2）**、风险 3 个、无法确定 2 项。**Verdict: REQUEST CHANGES**（P0-1 高危必须处理，P0-2/P0-3 建议同批修复）

---

## 0. 跑过的命令清单（全部只读）

```bash
# 1. 读改动
git --no-pager diff --stat
git --no-pager branch --show-current
git --no-pager diff Telegram/SourceFiles/data/data_session.cpp \
                   Telegram/SourceFiles/data/data_chat.cpp \
                   Telegram/SourceFiles/data/data_channel.cpp \
                   Telegram/SourceFiles/data/data_chat.h
git --no-pager diff Telegram/SourceFiles/history/history_inner_widget.cpp
git --no-pager diff Telegram/SourceFiles/history/history_item.cpp
git --no-pager diff Telegram/SourceFiles/dialogs/dialogs_widget.cpp
git --no-pager diff Telegram/SourceFiles/history/view/history_view_context_menu.cpp

# 2. 消费点普查
grep -rn "NoForwards"            --include=*.cpp --include=*.h --include=*.mm Telegram/
grep -rn "allowsForwarding"      --include=*.cpp --include=*.h --include=*.mm Telegram/
grep -rn "canExportChatHistory"  --include=*.cpp --include=*.h --include=*.mm Telegram/
grep -rn "forbidsForward"        --include=*.cpp --include=*.h Telegram/
grep -rn "CopyRestrictionType::" --include=*.cpp --include=*.h Telegram/
grep -rn "CopyRestrictionTypeFor\|CopyMediaRestrictionTypeFor" --include=*.cpp --include=*.h Telegram/
grep -rn "is_ayuNoforwards\|ayuNoforwards\|ayu_noforwards" Telegram/          # exit 1，零命中
grep -rn "ayu" Telegram/SourceFiles/mtproto/scheme/*.tl                        # exit 0，零输出

# 3. 枚举/位定义
grep -n "enum class .*Flag"  --include=*.h Telegram/SourceFiles/data/
grep -rn "class Flags\|struct Flags" --include=*.h Telegram/
grep -rn "PeerFlagValue"     --include=*.h Telegram/

# 4. base::flags 可得性
git submodule status
ls -A Telegram/lib_base/ | wc -l        # → 0
find . -name "flags.h" -not -path "./out/*"   # → 零结果
```

---

## 1. 改动本体（已核对）

`data_session.cpp` 两处，`flagsSet` 把 `chat.noforwards` / `channel.noforwards` 改写到 `Flag::AyuNoForwards`：

- `Telegram/SourceFiles/data/data_session.cpp:933`
  ```cpp
  | (data.is_noforwards() ? Flag::AyuNoForwards : Flag());
  ```
- `Telegram/SourceFiles/data/data_session.cpp:1096`
  ```cpp
  | (data.is_noforwards() ? Flag::AyuNoForwards : Flag())
  ```

`Telegram/SourceFiles/data/data_chat.cpp:75-77`
```cpp
bool ChatData::allowsForwarding() const {
	return true;
}
```

`Telegram/SourceFiles/data/data_channel.cpp:787-789`
```cpp
bool ChannelData::allowsForwarding() const {
	return true;
}
```

### 1.1 编译错误的判定 —— 已确认删除是唯一正确解法【正面结论】

`grep -rn "is_ayuNoforwards\|ayuNoforwards\|ayu_noforwards" Telegram/` → **exit 1，零命中**。
`grep -rn "ayu" Telegram/SourceFiles/mtproto/scheme/*.tl` → **exit 0，零输出**。

即 TL schema 中 `chat`（`api.tl:102`）与 `channel`（`api.tl:104`）**不存在**任何 Ayu 私有字段，
`MTPDchat` / `MTPDchannel` 不可能生成 `is_ayuNoforwards()`。原代码必然编译失败，
本次删除该行是**唯一正确**的处理方式，不存在"少了一条数据来源"的损失。

**基线声明（重要）**：改动前的状态是**编译不过**的，因此本仓库内不存在"改动前可运行的对照组"。
真正的回归基线只能是上游 tdesktop（`Flag::NoForwards` + `allowsForwarding()` 返回 `!(flags() & Flag::NoForwards)`）。
下文所有"行为改变"均以**上游 tdesktop**为参照系判断，而非以"本 fork 改动前"。

---

## 2. 问题清单

### 【确定 BUG · 高】P0-1 `allowsForwarding()` 硬编码 `true` 连带解锁「导出聊天记录」，绕过内容保护

**位置**
- 根因：`Telegram/SourceFiles/data/data_chat.cpp:75-77`、`Telegram/SourceFiles/data/data_channel.cpp:787-789`
- 受害点：`Telegram/SourceFiles/data/data_peer.cpp:771-773`
  ```cpp
  bool PeerData::canExportChatHistory() const {
  	if (isRepliesChat() || isVerifyCodes() || !allowsForwarding()) {
  		return false;
  ```
- 暴露 UI：`Telegram/SourceFiles/window/window_peer_menu.cpp:1015-1018`
  ```cpp
  void Filler::addExportChat() {
  	if (!_peer->canExportChatHistory()) {
  		return;
  ```

**为什么是 BUG 而非"用户要求"**

`return true` 的字面意图是解锁「转发 / 复制 / 下载」，但 `allowsForwarding()` 在 tdesktop 里是
一个**共享的"内容可否外流"总闸**，共 7 个消费者。逐个判定如下：

| # | 消费者 | 判定 | 依据 |
|---|---|---|---|
| 1 | `history/history_item.cpp:3063-3069` `HistoryItem::allowsForward()` | ✅ 在范围内 | `&& history()->peer->allowsForwarding()` |
| 2 | `history/view/history_view_list_widget.cpp:5972-5982` `CopyRestrictionTypeFor` | ✅ 在范围内（复制） | 返回值决定 `CopyRestrictionType::None/Group/Channel` |
| 3 | `history/history_inner_widget.cpp:4032-4034` `hasCopyRestriction` | ✅ 在范围内（复制） | `!_peer->allowsForwarding() \|\| ...` |
| 4 | `iv/iv_instance.cpp:156-159` `CanShareMarkdownItem` | ✅ 在范围内 | rich-message 分享 |
| 5 | `info/media/info_media_provider.cpp:86-97` `hasSelectRestriction` | ⚠️ 边缘（媒体多选/批量复制解锁） | `} else if (_peer->allowsForwarding()) { return false; }` |
| 6 | `core/click_handler_types.cpp:664-666` 「设为提醒」 | ⚠️ **范围外溢出**（风险 R-2） | `&& item->history()->peer->allowsForwarding();` |
| 7 | **`data/data_peer.cpp:772` `canExportChatHistory`** | ❌ **高危范围外溢出**（本条） | 见下 |

第 7 条与前 6 条的差别在于**性质**：
`noforwards` 在协议层只拦 `messages.forwardMessages` 与保存类动作，**不拦 `messages.getHistory`**。
因此「导出聊天记录」是**整段历史的全量 JSON/HTML 落盘**，不是"下载一条消息"。
它被同一次 `return true` 顺带放行，属于典型的共享闸门溢出：

- 这是一个**独立的功能面**（`addExportChat`，聊天设置/资料菜单项），与"转发/复制/下载单条消息"是不同的用户动作；
- 它把受保护内容从"只能看"变成"可整库导出"，泄露面比转发大一个量级；
- 服务端不会兜底（`getHistory` 对受保护群组照常返回），所以这是**纯客户端**的静默降级。

**修复建议**
在 `ChatData::allowsForwarding()` / `ChannelData::allowsForwarding()` 之外，单独在 `canExportChatHistory()`
保留一道显式闸门，例如：

```cpp
bool PeerData::canExportChatHistory() const {
	if (isRepliesChat() || isVerifyCodes() || !allowsForwarding()) {
		return false;
	} else if (isAyuNoForwards()) {   // 保留内容保护：导出属于整库外流
		return false;
	} else if (const auto channel = asChannel()) {
		...
```

（若产品明确要求"导出也要解锁"，则至少需要在 UI 上显式提示，不应作为 `return true` 的沉默副作用。）

---

### 【确定 BUG · 中】P0-2 `ChatDataFlag` 缺显式底层类型 + `AyuNoForwards = (1 << 31)` 是有符号溢出 UB

**位置**：`Telegram/SourceFiles/data/data_chat.h:16` 与 `:27`

```cpp
16: enum class ChatDataFlag {
...
25: 	NoForwards = (1 << 8),
26:
27: 	AyuNoForwards = (1 << 31),
28: };
```

**对照：同级 flag 枚举全部带显式底层类型**

| 枚举 | 声明位置 | 底层类型 | Ayu 位写法 |
|---|---|---|---|
| `UserDataFlag` | `data/data_user.h:119` | `: uint32` | `data_user.h:149-150` 用 `1 << 29` / `1 << 30`（安全，未触符号位）|
| `ChannelDataFlag` | `data/data_channel.h:43` | `: uint64` | `data_channel.h:93` `AyuNoForwards = (1ULL << 63)` ✅ |
| `MessageFlag` | `data/data_types.h:255` | `: uint64` | `data_types.h:273` `AyuNoForwards = (1ULL << 17)` ✅ |
| **`ChatDataFlag`** | **`data/data_chat.h:16`** | **无** | **`data_chat.h:27` `AyuNoForwards = (1 << 31)`** ❌ |

**为什么是 BUG**

1. `1 << 31` 的两个操作数都是 `int`，结果 `2^31` 超出 `int` 上限 `2^31-1`。
   C++17 [expr.shift]：若 `E1` 非负且 `E1 * 2^E2` **不能**用结果类型的对应无符号类型表示，则行为**未定义**。
   只有 C++20 起才被定义为按模取回绕值。本项目按 `AGENTS.md` 走 C++17 路线。
2. 无固定底层类型的枚举，底层类型由实现选取"能表示所有枚举值的最小候选"。
   GCC/Clang/MSVC 在二补码上把该枚举值折叠为 `INT_MIN`，枚举底层类型选 `int`。
   于是 `AyuNoForwards` 的**运行时值是负数** `-2147483648`。
3. `ChannelDataFlag`（`data_channel.h:93`）用的是 `1ULL << 63` 且显式 `: uint64`，**没有**这个问题。
   即：本 fork 为 chat/channel 引入 Ayu 位时，只有一侧按 64 位无符号规范写，另一侧没有。

**本次改动是否"放大"了它 —— 是**

- 改动前（上游语义）`ChatDataFlag::AyuNoForwards` 虽然已在枚举里（`data_chat.h:27`）、
  也已被 `data_chat.cpp:72` 的 `isAyuNoForwards()` 引用，但它**在 `processChat` 中从未被写入**
  （那一行是 `is_ayuNoforwards()`，编译不过）。因此该位是**死位**，其取值不进入任何运算。
- 本次改动在 `data_session.cpp:933` **首次真正写入**该位，使其进入
  `chat->setFlags((chat->flags() & ~flagsMask) | flagsSet)` 的位运算链
  （`data_session.cpp:934`）以及 `Data::Flags<ChatDataFlags>::set()` 的 `_value = which`（`data_flags.h:41-46`）。
- 结论：语言层缺陷**先验存在**，但本次改动**首次让该 UB 值参与运行时位运算**。

**实际影响分级**：在 GCC/Clang/MSVC 二补码 + 无激进优化（如 `-fstrict-overflow` + 依赖该值的常量折叠）下，
`INT_MIN` 参与 `|`、`&`、`~`、`^` 均表现正常，**不会立即崩溃**。真正的风险面是：
- 任何把该枚举当**有符号 int** 做算术/比较/日志输出的路径（当前树内未发现，见 §3.2）；
- 换工具链 / 开 UBSan / 改 C++ 标准时的行为漂移。

**修复建议**（一行，最小改动）

```cpp
enum class ChatDataFlag : uint32 {
	...
	NoForwards = (1 << 8),
	AyuNoForwards = (1U << 31),
};
```
或直接与 `ChannelDataFlag` 保持一致用 `: uint32` + `1U << 31`。
`base::flags<ChatDataFlag>` 的存储宽度随之确定，可顺带消除 §3.2 的无法确定项。

---

### 【确定 BUG · 中】P0-3 `ChatDataFlag::NoForwards` / `ChannelDataFlag::NoForwards` 成为死位，4 个消费点恒为 false

**位置（写入侧）**
- `Telegram/SourceFiles/data/data_session.cpp:922-923`（chat 掩码）
  ```cpp
  		| Flag::NoForwards
  		| Flag::AyuNoForwards;
  ```
- `Telegram/SourceFiles/data/data_session.cpp:1045-1046`（channel 掩码）
  ```cpp
  		| Flag::NoForwards
  		| Flag::AyuNoForwards
  ```
- 掩码里保留了两位，但 `flagsSet`（`data_session.cpp:933`、`1096`）**只写 `AyuNoForwards`**。

`grep -rn "NoForwards" --include=*.cpp --include=*.h --include=*.mm Telegram/` 全量结果中，
写入侧**仅** `data_session.cpp:922/933/1045/1096` 四处。改动后：

- `ChatDataFlag::NoForwards`（`data_chat.h:25`，bit 8）与 `ChannelDataFlag::NoForwards`（`data_channel.h:64`，bit 20）
  **再无任何代码置位**，并且由于仍在 `flagsMask` 中，每次 `processChat` / `processChannel`
  都会被 `chat->setFlags((chat->flags() & ~flagsMask) | flagsSet)`（`data_session.cpp:934`）**主动清零**。

**位置（读取侧，4 个消费点全部恒 false）**

1. `Telegram/SourceFiles/history/history_inner_widget.cpp:661-666`
   ```cpp
   		_sharingDisallowed = chat
   			? Data::PeerFlagValue(chat, ChatDataFlag::NoForwards)
   			: Data::PeerFlagValue(
   				channel,
   				ChannelDataFlag::NoForwards
   			) | rpl::type_erased;
   ```
2. `Telegram/SourceFiles/info/media/info_media_provider.cpp:110-115`
   ```cpp
   	auto noForwards = chat
   		? Data::PeerFlagValue(chat, ChatDataFlag::NoForwards)
   		: Data::PeerFlagValue(
   			channel,
   			ChannelDataFlag::NoForwards
   		) | rpl::type_erased;
   ```
   （两处均**未**随本次改动同步改读 `AyuNoForwards`；`history_inner_widget.cpp` 虽在 diff 中，
   但其 diff 全部是 checkpoint 特性，未触及第 662/665 行。）

**为什么是 BUG 而不是"无害的死代码"**

- 表面结果一致：`_sharingDisallowed` 恒 false、`hasSelectRestriction()` 走 `allowsForwarding()` 恒 true 分支，
  两者最终都表现为"不限制"，所以**没有可见的错误 UI**。
- 但代价是**反应链静默**：`Data::PeerFlagValue` → `SingleFlagValue` → `FlagsValueWithMask`
  （`data/data_peer_values.h:27-49`）内部是
  ```cpp
  	) | rpl::filter([mask](const ChangeType &change) {
  		return change.diff & mask;
  	})
  ```
  `mask` 恒为 0 ⇒ 该 `rpl::filter` **永不通过**。`history_inner_widget.cpp:685-690` 与
  `info_media_provider.cpp:127+` 的 `rpl::combine(_sharingDisallowed.value(), ...)` 失去了一个输入维度。
  将来任何人想恢复"内容保护"交互，必须先发现这两个读点已经连到了空掩码。
- 更实质的问题：**代码在"自称"还在做保护**（`setupSharingDisallowed()`、`_sharingDisallowed` 命名），
  实际已完全不设防。这是维护性陷阱 + 误导性代码。

**修复建议**（二选一）
- **推荐（与先例一致）**：把两处读点改为 `ChatDataFlag::AyuNoForwards` / `ChannelDataFlag::AyuNoForwards`，
  与 `history_item_helpers.cpp:933`、`data_user.cpp:694-697` 的 Ayu 命名对齐，
  保持"服务端标志 → Ayu 位"的单一映射；同时从 `flagsMask`（`data_session.cpp:922`、`1045`）里删掉已死的
  `Flag::NoForwards`，让死位在编译期就被发现。
- 若确认要彻底废弃：删除 `data_chat.h:25` / `data_channel.h:64` 两个枚举成员，
  以及上述 4 个读点，一并清理。

---

## 3. 针对四个"必须查清"问题的逐条回答

### 3.1 `flagsMask` 是否包含新写入的位 / 标志能否被正确清零

**答：能。** 已逐行核对。

- chat：`data_session.cpp:916-923` 的 `flagsMask` 含 `Flag::NoForwards | Flag::AyuNoForwards`；
  `data_session.cpp:924-933` 的 `flagsSet` 无条件含 `(data.is_noforwards() ? Flag::AyuNoForwards : Flag())`。
- channel：`data_session.cpp:1028-1057` 的 `flagsMask` 含 `Flag::NoForwards | Flag::AyuNoForwards`（1045-1046 行）；
  `data_session.cpp:1075-...` 的 `flagsSet` 无条件含同一条件式（1096 行）。
- 落地语句：`data_session.cpp:934` `chat->setFlags((chat->flags() & ~flagsMask) | flagsSet);`
  → `ChatData::setFlags`（`data_chat.cpp:149-...`）→ `Data::Flags::set()`（`data_flags.h:41-46`）
  做 `_value = which`（全量替换，不做增量）。
- 结论：掩码/置位**对称**，服务器撤销保护时 `is_noforwards()` 转 false 即被正确清除。
  **本次改动没有引入"位设上了清不掉"的问题。**

**附带确认（非回归）**：`MTPDchatForbidden` 分支 `data_session.cpp:950`
`chat->setFlags(ChatDataFlag::Forbidden);` 是**全量替换**，会连带清掉 `AyuNoForwards`。
这是上游既有行为，与本次改动无关（改动前清掉的是 `NoForwards`，同样丢）。
**minimal channel 路径**：`data_session.cpp:1042-1044` 只对 `Left | Creator` 做 `minimal` 条件，
`NoForwards` / `AyuNoForwards` 无条件进掩码 —— 与改动前**逐字同构**，故不构成本次回归。

### 3.2 `1 << 31` 符号位问题（`data_chat.h:27`）是否被本次改动放大

**答：是，已放大**，详见 P0-2。放大点在于 `data_session.cpp:933` 首次让该 UB 值进入位运算链。
未发现**具体的运行时错误**（见 §4 无法确定项 A）。

### 3.3 标志位映射是否与既有先例一致 / 聊天频道级与消息级、用户级是否语义冲突

**先例对照**

| 层级 | 写入位置 | 映射 | Ayu 位 |
|---|---|---|---|
| 消息级 | `history/history_item_helpers.cpp:931-933` | `MTP::f_noforwards`（`api.tl:124` `message.noforwards:flags.26`）→ `Flag::AyuNoForwards` | `data_types.h:273` `AyuNoForwards = (1ULL << 17)` |
| 消息级（旧） | `history_item_helpers.cpp:932` | **已注释掉** `// \| ((flags & MTP::f_noforwards) ? Flag::NoForwards : Flag())`，注释 `// AyuGram: removed` | — |
| 用户级 | `data/data_user.cpp:1105-1107` `userFull` flags2.23/24（`api.tl:269`）→ `setNoForwardsFlags(myEnabled, peerEnabled)` | **两位** | `data_user.h:149-150` `NoForwardsMyEnabled = (1 << 29)` / `NoForwardsPeerEnabled = (1 << 30)` |
| 聊天/频道级（本次） | `data_session.cpp:933` / `1096` | `chat.noforwards:flags.25`（`api.tl:102`）/ `channel.noforwards:flags.27`（`api.tl:104`）→ `Flag::AyuNoForwards` | `data_chat.h:27` `(1 << 31)` / `data_channel.h:93` `(1ULL << 63)` |

**结论 A：命名与"服务端位 → Ayu 位"的方向性映射与消息级先例完全一致** ✅
`history_item_helpers.cpp:932` 的 `// AyuGram: removed` + 933 行的改写，正是本次 P0 改动的模板。
本次把 chat/channel 也纳入同一模式，**方向正确**。

**结论 B：存在语义冲突（风险 R-1）** ⚠️
`PeerData::isAyuNoForwards()`（`data/data_peer.cpp:1718-1727`）把三种异质语义**折叠成同一个 bool**：

```cpp
bool PeerData::isAyuNoForwards() const {
	if (const auto user = asUser()) {
		return user->isAyuNoForwards();
	} else if (const auto channel = asChannel()) {
		return channel->isAyuNoForwards();
	} else if (const auto chat = asChat()) {
		return chat->isAyuNoForwards();
	}
	return true;
}
```

- **用户级**（`data_user.cpp:694-697`）把 `NoForwardsMyEnabled`（"我给对方开了"）与
  `NoForwardsPeerEnabled`（"对方给我开了"）**或**在一起，丢失方向。
  `window/window_peer_menu.cpp:1564-1566` 为了找回这个方向，专门写了一句补偿：
  ```cpp
  			const auto willBeRequest = true
  				&& (user->flags() & UserDataFlag::NoForwardsPeerEnabled)
  ```
  —— 证明"方向信息确实被 `isAyuNoForwards()` 丢掉了"。
- **聊天/频道级**（本次）只有一位，来自服务端 `noforwards`，语义是"**群主/所有者**为全群开启了内容保护"，
  普通成员**不可控**。
- 二者在同一个 `PeerData::isAyuNoForwards()` 契约下返回同一种 bool，导致下游无法区分
  "这个保护我能关"与"这个保护我不能关"。具体表现为：
  - `boxes/peers/edit_peer_info_box.cpp:1081` `.noForwards = _peer->isAyuNoForwards()` —— 只有管理员能进这个框，能接受；
  - `window/window_peer_menu.cpp:1526-1534` `addToggleNoForwards()` 首行即 `if (!user || ...) return;`
    —— **对 chat/channel 直接 return**，所以群组级保护开关只能从"编辑群组"进入。
    这个 UI 不一致是**既有**的，非本次引入。

**结论 C：本次改动对 `isAyuNoForwards()` 是净修复** ✅
`edit_peer_info_box.cpp:2914-2918` 的 `saveForwards()`：
```cpp
	if (!_savingData.noForwards
		|| *_savingData.noForwards == _peer->isAyuNoForwards()) {
		return continueSave();
	}
```
改动前（若能编译）`_peer->isAyuNoForwards()` 对 chat 恒 false ⇒ 勾选后保存会**每次都重复发送**
`messages.toggleNoForwards(true)`（因 `true == false` 不成立），且服务端回包后复选框**不会变为已勾选**。
改动后该位正确反映服务器状态，**这是一个真实修复**，应予保留。

### 3.4 是否有别处依赖 `allowsForwarding()` 返回 false 做安全限制 / `return true` 是否打开不该打开的路径

**答：有 2 条溢出路径，1 高 1 低。** `grep -rn "allowsForwarding" --include=*.cpp --include=*.h --include=*.mm Telegram/`
共 17 条命中，逐一归类：

| 路径 | 位置 | 判定 |
|---|---|---|
| 消息转发 | `history/history_item.cpp:3067` | ✅ 目标内 |
| 文本/媒体复制 | `history/view/history_view_list_widget.cpp:5975` | ✅ 目标内 |
| 复制限制 toast | `history/history_inner_widget.cpp:4033` | ✅ 目标内 |
| rich-message 分享 | `iv/iv_instance.cpp:158` | ✅ 目标内 |
| 媒体批量选择 | `info/media/info_media_provider.cpp:89` | ✅ 目标内（边缘） |
| **整库导出** | **`data/data_peer.cpp:772` → `window/window_peer_menu.cpp:1016`** | ❌ **高危，见 P0-1** |
| 日历提醒 | `core/click_handler_types.cpp:666` | ⚠️ 低，见 R-2 |

**未找到**：`allowsForwarding()` 没有任何调用点用于鉴权、权限校验或 API 请求门禁。
执行的命令：`grep -rn "allowsForwarding" --include=*.cpp --include=*.h --include=*.mm Telegram/`（搜索范围 `Telegram/` 全树，17 条，已全部列尽）。
即：没有"权限判定"被误开，溢出只发生在 UI/功能可达性层面。

### 3.5 附带确认：消息级闸门早已被 AyuGram 拆除（解释基线，非本次引入）

`Telegram/SourceFiles/history/history_item.cpp:3136-3138`
```cpp
bool HistoryItem::forbidsForward() const {
	return false;
}
```
`Telegram/SourceFiles/history/history_item.cpp:3140-3147`
```cpp
bool HistoryItem::forbidsSaving() const {
	if (forbidsForward()) {
		return true;
	}/* ... */
	return false;
}
```
以及 `Telegram/SourceFiles/data/data_user.cpp:690-692`
```cpp
bool UserData::allowsForwarding() const {
	return true;
}
```
—— 用户级 `allowsForwarding()` **早已**是硬编码 `true`（不在本次 diff 中）。

**这既是"支持本次改动"的论据**（chat/channel 现在与既有 user 级行为对齐，AyuGram 风格一致），
**也是"说明 P0-1 不是孤例"的论据**（同一类溢出在 user 级已经存在过，但 user 级无 `canExportChatHistory`
的等价闸门，chat/channel 级多出了整库导出这一条）。以上三处**均未在本次 diff 中**，属既有状态。

---

## 4. 风险与无法确定项

### 风险 R-1（中）：`allowsForwarding()` 与 `isAyuNoForwards()` 在同一份数据上给出相反答案

改动后 `data_chat.cpp:71-77` 同一个 `ChatData` 对象上：
- `isAyuNoForwards()` → 读 `data_session.cpp:933` 写入的真实服务器位 → **"受保护"**
- `allowsForwarding()` → 硬编码 → **"允许转发"**

而 `edit_peer_info_box.cpp:1081` 会把 `isAyuNoForwards()` 直接渲染到 UI
（`edit_peer_info_box.cpp:1117-1119` 据此切换到 `tr::lng_manage_private_group_noforwards_title`）。
用户因此会看到"该群组已保护内容"的管理界面，但客户端实际允许任意转发 / 复制 / 保存 / 导出。
**管理面显示与服务端真实状态一致，但客户端执行面不一致** —— 对自定义客户端而言这正是"该锁的没锁"。

### 风险 R-2（低）：`click_handler_types.cpp:666` 溢出到「设为提醒」

```cpp
	const auto canForward = item
		&& !item->forbidsForward()
		&& item->history()->peer->allowsForwarding();
	if (canForward) {
		menu->addAction(
			tr::lng_context_set_reminder(tr::now),
```
`forbidsForward()` 恒 false（§3.5），故 `canForward` 现在**只**取决于 `allowsForwarding()`，即恒 true。
效果：受保护群组的消息日期右键菜单新增"设为提醒"，点击后会把**消息正文写入系统日历**
（`ExportToCalendar(date, messageText)`，`click_handler_types.cpp:661`）。超出"转发/复制/下载"字面范围。

### 风险 R-3（无法确定 · 需产品确认）：受保护群组的两处用户可见新增

1. `Telegram/SourceFiles/history/view/history_view_context_menu.cpp:2658-2666`
   ```cpp
   	const auto peer = item->history()->peer;
   	if ((!peer->isAyuNoForwards() && !AyuForward::isAyuForwardNeeded(item))
   		|| item->isSponsored()) {
   		return;
   	}
   ```
   改动前 chat/channel 的 `peer->isAyuNoForwards()` 恒 false（`AyuNoForwards` 死位）；
   改动后为 true ⇒ **受保护群组内每条消息的右键菜单都会追加一行"不可转发"说明标签**
   （`tr::ayu_UnforwardableContextMenuText`，`history_view_context_menu.cpp:2678`）。
   这可能是**有意**的（该标签本就是为展示保护状态而写），也可能是意外。
2. `Telegram/SourceFiles/ayu/features/forward/ayu_forward.cpp:243-245`
   ```cpp
   bool isFullAyuForwardNeeded(not_null<HistoryItem*> item) {
   	return item->from()->isAyuNoForwards() || item->history()->peer->isAyuNoForwards();
   }
   ```
   改动后 ⇒ 受保护群组内**所有**消息的转发都会走 Ayu 专用转发通道，而非普通 `messages.forwardMessages`。
   这大概率是 AyuGram 的既定设计（该函数存在即为此目的），但会改变受保护群的转发实现路径。

**判定：【无法确定】** —— 无法从代码判断这两项是否符合产品预期，需要与需求方确认。

### 无法确定 A：`base::flags<ChatDataFlag>` 对 `INT_MIN` 的布尔转换语义

- 执行的命令与结果：
  - `ls -A Telegram/lib_base/ | wc -l` → **0**（目录为空）
  - `find . -name "flags.h" -not -path "./out/*"` → **零结果**
  - `git submodule status` → 全部以 `-` 前缀开头（**未初始化**），含 `Telegram/lib_base`
- 因此 `base::flags<ChatDataFlag>` 的定义**不在本 checkout 中**，无法直接核对其
  `operator bool` / `empty()` / `from_raw()` 实现。
- 风险点：若 `base::flags` 的真值判定是 `_value != 0` ⇒ `INT_MIN` 判为 true（正确）；
  若是 `_value > 0` ⇒ `INT_MIN` 判为 **false**（`isAyuNoForwards()` 会静默恒假）。
  `data/data_peer_values.h:46-48` 的 `return !!value;` 依赖这一语义：
  ```cpp
  	) | rpl::map([](typename ChangeType::Type value) {
  		return !!value;
  	});
  ```
- **判定：【无法确定】。** 未在树内找到任何 `if (peer->flags())` 之类的布尔上下文用法可作旁证
  （执行的命令：`grep -rn "if (.*->flags())\|if (chat->flags\|if (channel->flags\|!user->flags()\|(bool).*flags" --include=*.cpp Telegram/SourceFiles/` → 零命中）。
- 建议：按 P0-2 的建议给 `ChatDataFlag` 加显式底层类型，可**同时消除**本项无法确定
  （`from_raw` / 存储宽度随底层类型确定），或直接初始化 `Telegram/lib_base` 子模块后复核。

### 无法确定 B：受保护群的"不可转发"标签 / Ayu 全量转发通道是否符合产品预期

见风险 R-3，两点均需需求方确认，代码本身无法自证。

---

## 5. 修复建议汇总（按优先级）

| 优先级 | 对应 | 建议 |
|---|---|---|
| P0 | P0-1 | 在 `data/data_peer.cpp:772` 的 `canExportChatHistory()` 中显式加 `isAyuNoForwards()` 闸门（整库导出 ≠ 转发/复制/下载）。 |
| P0 | P0-3 | 把 `history_inner_widget.cpp:662,665` 与 `info_media_provider.cpp:111,114` 的读点改为 `AyuNoForwards`；并从 `data_session.cpp:922`、`1045` 的 `flagsMask` 删除已死的 `Flag::NoForwards`（或整体删除该枚举成员）。 |
| P1 | P0-2 | `data_chat.h:16` 加显式底层类型：<br>`enum class ChatDataFlag : uint32 {`，<br>并把 `:27` 改为 `AyuNoForwards = (1U << 31);`。 |
| P1 | R-1 | 明确并写下契约：`isAyuNoForwards()` = "服务端确实设了保护"（供 UI 展示与管理），`allowsForwarding()` = "本客户端是否执行保护"（恒 true 是 AyuGram 的既定策略）。至少在两处加注释说明这是**有意的策略分叉**，不是逻辑疏漏。 |
| P2 | R-2 | 若"设为提醒"不属于解锁范围，在 `click_handler_types.cpp:664-666` 改用 `!peer->isAyuNoForwards() && !item->isAyuNoForwards()` 之类的 Ayu 判定。 |
| P2 | R-3 | 与需求方确认两处用户可见变更（受保护群的"不可转发"标签、Ayu 全量转发通道）。 |

---

## 6. 总体评价

**做得对的部分**

1. 删掉 `data.is_ayuNoforwards()` 是**唯一正确**的解法 —— 已用两条 grep 证明 TL schema 中不存在该字段（§1.1），不存在数据丢失。
2. 标志位映射方向与消息级先例（`history_item_helpers.cpp:931-933`，含 `// AyuGram: removed` 注释）**完全对齐**，把 chat/channel 纳入同一"服务端位 → Ayu 位"模式是正确的收敛。
3. `flagsMask` 与 `flagsSet` 严格对称（§3.1），**没有**引入"设上了清不掉"的经典 bug。
4. 顺带修好了 `edit_peer_info_box` 的内容保护开关（§3.3 结论 C）——改动前该复选框对群组永远无法回显为已勾选。
5. 与既有 user 级 `UserData::allowsForwarding() { return true; }`（`data_user.cpp:690-692`，非本次改动）风格一致。

**主要问题**

1. **P0-1 是本次唯一的高危项**：`return true` 是共享闸门，7 个消费者里 `canExportChatHistory`（整库 JSON/HTML 导出）不在"转发/复制/下载"字面范围内，且服务端 `getHistory` 不拦，属于静默的客户端降级。这不是"顺手多开了一个功能"，而是把一个用户没要求的、泄露面最大的能力一并打开了。
2. **P0-3 制造了误导性代码**：`setupSharingDisallowed()` / `_sharingDisallowed` / `noForwards` 三处命名仍在声称在管"内容分享限制"，实际已连到空掩码、恒 false。审查时最容易被这种"看起来还在保护"的代码骗过去。
3. **P0-2 是被本次改动首次激活的语言层 UB**：`ChatDataFlag` 是 `data/` 下**唯一**没有显式底层类型的 flag 枚举，而 `1 << 31` 恰好踩中 C++17 的有符号左移未定义行为。同一个 fork 里 `ChannelDataFlag`/`MessageFlag` 都规范地用了 `: uint64` + `1ULL`，说明这是单点疏漏而非风格。
4. 整体上，"服务端标志位"与"客户端执行策略"两个概念在本次改动中被进一步混同（`isAyuNoForwards` 管展示，`allowsForwarding` 管执行），建议在代码里把这条分叉写清楚。

**Verdict：REQUEST CHANGES。** 改动方向正确、结构干净，但在合入前必须处理 P0-1（导出绕过），
并建议同批修掉 P0-3（死标志位）与 P0-2（`1 << 31` 符号位）。
