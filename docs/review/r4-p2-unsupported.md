# R4-P2 对抗式审查：unsupported 消息显示部分文本

> 审查对象：`Telegram/SourceFiles/history/history_item.cpp` 未提交工作区改动（3 处同改）
> 仓库：`/root/workspace/AyuGramDesktop`（AyuGramDesktop，tdesktop v7.0.9 fork）
> 性质：**只读审查**。未修改任何 `Telegram/` 下文件，未执行 git 写操作，未配置/构建/测试。
> 唯一写入：本报告 `docs/review/r4-p2-unsupported.md`。
> 标注约定：**【确定】** = 本次在代码中逐行核实；**【风险】** = 代码事实成立但后果依赖未 checkout 的子模块或未测运行时；**【无法确定】** = 检索无果，附命令与范围。

---

## 0. 结论速览

| # | 结论 | 判定 |
|---|---|---|
| 1 | 3 处改动互相一致，且与正常路径的文本构造**逐字符同源** | 【确定】 |
| 2 | `TextWithEntities::empty()` **确实存在**（3 处 HEAD 既有调用可证），但**定义体不在工作树** | 【确定存在】/【定义无法确定】 |
| 3 | 三元表达式两分支同为 `TextWithEntities`，**无重载歧义、无悬垂引用** | 【确定】 |
| 4 | `setText` 只有一个重载，`TextWithEntities` 二参构造在 HEAD 已被使用 | 【确定】 |
| 5 | `applyEdition` 的 `serviceText` 分支**不会**吞掉新文本（catch-all 返回空） | 【确定】 |
| 6 | 构造函数分支少做的 4 件事（`setGroupId`/`setReactions`/`applyTTL`/factcheck）**均无崩溃或断言面** | 【确定】 |
| 7 | Story 路径与本改动**不同构且未被波及** | 【确定】 |
| 8 | `.empty()` 与 `EnsureNonEmpty` 的空判口径可能不一致 → 边缘空泡 | 【风险】 |
| 9 | 第二个 `Unsupported` 入口 `webPageNotModified` 被一并波及 | 【风险】 |
| 10 | 占位文案全文可达点仅 3 处，**已被 100% 覆盖**，无遗漏路径 | 【确定】 |
| 11 | 实体安全：无新增实体类别；但**渲染/转义层不可核** | 【确定无新增】/【无法确定】 |
| 12 | **确定 BUG：0** | — |

**Verdict：CONDITIONAL PASS**（详见 §7）

---

## 1. 本次跑过的命令 / 检索（全部只读）

```bash
# 改动读取
git --no-pager diff --stat
git --no-pager status --porcelain
git --no-pager diff Telegram/SourceFiles/history/history_item.cpp
git --no-pager diff Telegram/SourceFiles/dialogs/dialogs_widget.cpp
git --no-pager diff Telegram/SourceFiles/history/history_inner_widget.cpp Telegram/SourceFiles/history/history_inner_widget.h
git --no-pager show HEAD:Telegram/SourceFiles/history/history_item.cpp   # 多次，验证"改动前"基线

# 目录 / 子模块状态
for d in lib_base lib_ui lib_rpl lib_crl; do ls -A Telegram/$d | wc -l; done   # 全部 0
grep -c submodule .gitmodules                                                # = 37
grep -n "lib_" .gitmodules
grep -oP '(?<=path = ).*' .gitmodules

# 定义定位（全部未命中，见 §6）
grep -rnE "^(class|struct) TextWithEntities|struct TextWithEntities" --include=*.h --include=*.cpp .
grep -rn "TextWithEntities" Telegram/lib_base/ Telegram/lib_ui/ Telegram/SourceFiles/basic/ Telegram/SourceFiles/ui/
find . -iname "*text_with_entities*"
grep -rln "TextWithEntities" --exclude-dir=.git . | grep -v SourceFiles

# codegraph 索引（只读，immutable=1）
python3 -c "import sqlite3; con=sqlite3.connect('file:.codegraph/codegraph.db?immutable=1',uri=True); ..."
#   nodes 表 98735 行；LIKE 'TextWithEntities' → 0 命中
#   DISTINCT file_path LIKE '%lib_base%' OR '%lib_ui%' → 0 行

# 符号 / 语义检索
grep -rn "UnsupportedMessageText\|UnsupportedMessageMarkup" Telegram/SourceFiles
grep -rn "lng_message_unsupported" --include=*.cpp --include=*.h --include=*.strings Telegram/
grep -rn "MediaCheckResult::Unsupported\|CheckMessageMedia" --include=*.cpp --include=*.h Telegram/SourceFiles
grep -rn "MessageFlag::Legacy\|isLegacyMessage" --include=*.cpp --include=*.h Telegram/SourceFiles
grep -rn "hasTextLinks()" --include=*.cpp --include=*.h Telegram/SourceFiles
grep -rn "originalText()\|translatedText()" --include=*.cpp --include=*.h Telegram/SourceFiles
grep -rn "webPageNotModified\|MTPDwebPageNotModified" --include=*.cpp --include=*.h Telegram/SourceFiles
grep -rn "v::null" --include=*.cpp --include=*.h Telegram/SourceFiles
grep -rn "HistoryMessageFactcheck" --include=*.h --include=*.cpp Telegram/SourceFiles/history/
grep -rn "ventities()" --include=*.cpp Telegram/SourceFiles/ | head -20
grep -n "messageMediaWebPage\|webPageNotModified\|^message#\|messageUnsupported" Telegram/SourceFiles/mtproto/scheme/api.tl

# 定点读文件（read 工具 / sed+nl）
history_item.cpp     : 1-60, 400-410, 425-465, 467-584, 1450-1470, 2300-2369, 2600-2689,
                       2898-2945, 3550-3560, 3813-3835, 4439-4478, 4550-4600, 4690-4707, 5378-5451
history_item.h       : 479-489, 700-715
history_item_helpers.cpp : 1060-1119, 1395-1415
history_item_helpers.h   : 95-110
data_stories.cpp     : 88-127
data_session.cpp     : 4152-4154
api_text_entities.h  : 15-40
ayu_mapper.cpp       : 60-145
```

---

## 2. Q1 — 类型核验：`empty()` / `setText` / 三元表达式

### 2.1 `TextWithEntities::empty()` — 存在性【确定】，定义【无法确定】

**存在性【确定】**。工作树中有 3 处 **HEAD 即存在、本次未改动** 的 `.empty()` 调用，作用于 `TextWithEntities`：

`Telegram/SourceFiles/history/history_item.h:482`
```cpp
	[[nodiscard]] bool emptyText() const {
		return _text.empty();
	}
```
`_text` 即 `TextWithEntities`（该访问器在 `ayu_mapper.cpp:73`、`history_item.cpp:4696` 被当作"文本是否为空"使用）。若 `empty()` 不存在，HEAD 本身无法编译。

`Telegram/SourceFiles/history/history_item.cpp:1458`（`setServiceText` 内，HEAD 未改）
```cpp
	const auto had = !_text.empty();
```

`Telegram/SourceFiles/history/history_item.cpp:2341`（HEAD 为 `:2331`，本次未改）
```cpp
	auto serviceText = (!edition.richPage
		&& !checkedMedia
		&& edition.textWithEntities.empty()
```

> 结论：`TextWithEntities::empty()` 是上游已有 API，本次改动**复用而非新增**，不存在"调用不存在方法"的编译风险。

**定义体【无法确定】**。`TextWithEntities` 的定义在**未初始化的子模块**中：

- `Telegram/lib_base`（`.gitmodules:19-21` 声明为 submodule，目录 `ls -A | wc -l` = **0**）
- `Telegram/lib_ui`（`.gitmodules:25-27`，目录计数 **0**）
- `.codegraph/codegraph.db` 中 `nodes` 表 98735 行，`qualified_name LIKE 'TextWithEntities'` → **0 命中**；`file_path LIKE '%lib_base%' OR '%lib_ui%'` → **0 行**（索引同样未覆盖子模块）

因此 `empty()` 的**精确语义**（是 `text.isEmpty()` 还是 `text.isEmpty() && entities.isEmpty()`）**无法从本 checkout 判定** —— 该点直接转化为 §6 的 R1 风险。

### 2.2 `setText` 的重载【确定】

`Telegram/SourceFiles/history/history_item.h:568`
```cpp
	void setText(TextWithEntities textWithEntities);
```
**唯一重载**，无 `setText(QString)` / `setText(Text)` 变体 → 三元表达式不产生重载歧义。实现在 `history_item.cpp:4439`：
```cpp
void HistoryItem::setText(TextWithEntities textWithEntities) {
	auto text = textWithEntities;
	...
	applyLocalPremiumEmoji(text);
	detectTextLinks(text);
	setTextValue((_media && _media->consumeMessageText(text))
		? TextWithEntities()
		: std::move(text));
}
```
按值接收 → 调用方传临时对象或 lvalue 均安全。

### 2.3 `UnsupportedMessageText()` 返回类型【确定】

`Telegram/SourceFiles/history/history_item_helpers.h:102`
```cpp
[[nodiscard]] TextWithEntities UnsupportedMessageText();
```
`Telegram/SourceFiles/history/history_item_helpers.cpp:1404`
```cpp
[[nodiscard]] TextWithEntities UnsupportedMessageText() {
	const auto siteLink = u"https://t.me/AyuGramReleases"_q;
	auto result = TextWithEntities{
		tr::lng_message_unsupported(tr::now, lt_link, siteLink).replace("Telegram", "AyuGram")
	};
```
返回 `TextWithEntities` 纯值 —— 与三元另一分支 `textWithEntities`（`TextWithEntities`）**类型完全一致**。

### 2.4 三元表达式类型分析【确定 — 无问题】

三处均为 `cond ? TextWithEntities(prvalue) : TextWithEntities(lvalue)`：

| 位置 | 表达式 | 合成类型 | 结论 |
|---|---|---|---|
| `history_item.cpp:506-508` | `textWithEntities.empty() ? UnsupportedMessageText() : textWithEntities` | `TextWithEntities` **纯值**（复制第二操作数） | ✅ |
| `history_item.cpp:2333-2335` | `edition.textWithEntities.empty() ? UnsupportedMessageText() : edition.textWithEntities` | 同上 | ✅ |
| `history_item.cpp:2647-2649` | `textWithEntities.empty() ? UnsupportedMessageText() : textWithEntities` | 同上 | ✅ |

- 第二操作数为纯值、第三操作数为左值 → C++ `[expr.cond]` 合成类型为纯值 `TextWithEntities`，**发生复制**，不产生悬垂引用/生命周期问题。
- 无隐式转换、无 `explicit` 构造介入（三元只做条件求值，不做直接初始化）。
- 嵌套三元（`applyEdition:2330-2338`）5 个分支**全部**为 `TextWithEntities`，无混合类型。

### 2.5 二参 `TextWithEntities{QString, EntitiesInText}` 构造【确定 —— 已被 HEAD 证明】

`history_item.cpp:500-505` 新增的构造与 **HEAD 既有** 的 `history_item.cpp:567-572`（HEAD 为 `:559-564`）**逐字符相同**：
```cpp
			auto textWithEntities = TextWithEntities{
				qs(data.vmessage()),
				Api::EntitiesFromMTP(
					&history->session(),
					data.ventities().value_or_empty())
			};
```
后者在 HEAD 即编译通过 → 新增代码沿用**已验证**的构造路径，无新 API 面。

`Api::EntitiesFromMTP` 签名 `Telegram/SourceFiles/api/api_text_entities.h:23-25`：
```cpp
[[nodiscard]] EntitiesInText EntitiesFromMTP(
	Main::Session *session,
	const QVector<MTPMessageEntity> &entities);
```
`&history->session()` 得到 `Main::Session*` ✅；`data.ventities().value_or_empty()` 的 `.value_or_empty()` 写法在本文件 6 处（`history_item.cpp:504/571/2582/2595`、`history_item_edition.cpp:26`、`history_admin_log_item.cpp:246`）及全仓 10+ 处沿用 ✅。

> **Q1 小结**：类型层面**零缺陷**。唯一残留不确定性是 `empty()` 的语义（§6 R1）。

---

## 3. Q2 — 三处是否真的一致

### 3.1 判定入口：两个，都会走同一分支【确定】

`Telegram/SourceFiles/history/history_item_helpers.cpp:1109-1110`（入口 1）
```cpp
	}, [](const MTPDmessageMediaUnsupported &) {
		return Result::Unsupported;
	});
```
`Telegram/SourceFiles/history/history_item_helpers.cpp:1082-1083`（入口 2）
```cpp
		}, [](const MTPDwebPageNotModified &) {
			return Result::Unsupported;
		});
```
构造函数 `history_item.cpp:487-491` 统一收口：
```cpp
	const auto media = data.vmedia();
	const auto checked = media
		? CheckMessageMedia(*media)
		: MediaCheckResult::Good;
	if (checked == MediaCheckResult::Unsupported) {
```
→ 改 `:500` 一行即同时覆盖两个入口 ✅（但见 §6 R2 对入口 2 的额外影响）。

### 3.2 占位文案可达点全覆盖【确定】

`UnsupportedMessageText()` 全仓**仅 3 个调用点**，且 3 个全部被本次改动覆盖：

| 调用点 | 改动前 | 改动后 |
|---|---|---|
| `history_item.cpp:507` | 恒用占位 | 有文本用文本 |
| `history_item.cpp:2334` | 恒用占位 | 有文本用文本 |
| `history_item.cpp:2648` | 恒用占位 | 有文本用文本 |

底层文案串 `lng_message_unsupported` 全仓**仅 1 处**使用（`history_item_helpers.cpp:1407`，定义于 `lang.strings:2530`）→ **不存在"某条路径仍吐旧占位"的漏网**。

### 3.3 编辑路径的 `serviceText` 分支**不会**吞掉新文本【确定 —— 已证伪为缺陷】

这是最容易被误判为不一致的点，特此核实。`history_item.cpp:2339-2355`：
```cpp
	auto serviceText = (!edition.richPage
		&& !checkedMedia
		&& edition.textWithEntities.empty()
		&& edition.mtpMedia)
		? prepareServiceTextForMessage(*edition.mtpMedia, edition.isMediaUnread)
		: PreparedServiceText();
	if (updatingSavedLocalEdit) {
		Get<HistoryMessageSavedMediaData>()->text = std::move(updatedText);
	} else if (!serviceText.text.empty()) {
		setServiceText(std::move(serviceText));
		addToSharedMediaIndex();
	} else {
		setText(std::move(updatedText));
```
关键链路：
1. `applyEdition` 在 `history_item.cpp:2310-2311` 对 Unsupported **先把 `_media = nullptr`**：
   ```cpp
		if (mediaCheck == MediaCheckResult::Unsupported) {
			_media = nullptr;
   ```
   → `checkedMedia`（`:2321-2323`）为 null → `!checkedMedia` 恒 **true**。
2. `prepareServiceTextForMessage`（`history_item.cpp:5378`）的 `match` 结尾有 catch-all，`history_item.cpp:5448-5450`：
   ```cpp
	}, [](const auto &) {
		return PreparedServiceText();
	});
   ```
   `MTPDmessageMediaUnsupported` **不属于** Photo/Document/Story 任一专用分支 → 落入 catch-all → 返回**空** `PreparedServiceText`。
3. → `!serviceText.text.empty()` 为 **false** → 走 `else` → **`setText(std::move(updatedText))` 正常执行**。

✅ **结论**：`messageMediaUnsupported` 在编辑路径上确实会应用新文本，不存在第三种结局。

### 3.4 副作用连带改变的第 4 条路径：桌面通知【确定 — 行为变化，但一致】

`history_item.cpp:4691-4699` `notificationText()`：
```cpp
		if (_media && !isService()) {
			return _media->notificationText();
		} else if (!emptyText()) {
			return _text;
		}
```
Unsupported 消息 `_media == nullptr` → 直接返回 `_text`。
- 改动前：系统通知显示 "This message is not supported…" 占位文案。
- 改动后：**系统通知显示真实 caption**。

这是**自动发生的连带效果**（无需改代码即正确），与气泡文本保持一致，属**期望行为**；但它是本次 diff 未显式覆盖的第 4 个用户可见面，**建议在测试用例中纳入**（见 §6 R4）。

### 3.5 其余消费管线核查

| 管线 | 结论 | 证据 |
|---|---|---|
| 搜索索引 | **无本地文本索引**，零影响【确定】 | `addToMessagesIndex`（`history_item.cpp:2932-2938`）只 `messages->addNew(id)`，仅存 ID；`addToSharedMediaIndex`（`:2898-2918`）按 `sharedMediaTypes()` 掩码存 ID，不存文本 |
| 共享媒体索引 | `hasTextLinks()` **唯一**消费点是 `sharedMediaTypes()`（`history_item.cpp:2882`）→ 仅置 `Link` 位。**且改动前占位文案本身含 `textUrl` 实体**（`history_item_helpers.cpp:1408-1411` 的 `ParseEntities` + `lt_link`），故该位改动前已置位 → **无净变化**【确定】 |
| 引用/回复预览 | 走 `originalText()`（`history_item.cpp:3813-3816` → `_text`），自动跟随 ✅【确定】 |
| 转发 / 复制文本 | `history_view_context_menu.cpp:1615-1616`、`ayu/ui/context_menu/context_menu.cpp:887-891` 走 `originalText()`，自动跟随 ✅【确定】 |
| 翻译 | `translatedText()`（`history_item.cpp:3818-3830`）→ `originalText()`，自动跟随 ✅【确定】 |
| Story | **未被波及**（见 §5）✅【确定】 |

### 3.6 不一致小结

- 3 个主路径 **完全一致**，构造代码逐字符同源。
- 无"新文本被覆盖回占位"的路径（§3.3 已证伪该担忧）。
- 唯一"多出一个面"是通知（§3.4），且方向一致、不构成 BUG。

---

## 4. Q3 — 副作用：`Legacy` + 少做的 4 件事

`history_item.cpp:491-515`（Unsupported 分支全文）相对正常分支（`:530-589`）**少做**：

| 缺失动作 | 正常分支位置 | 崩溃/断言面核查 | 判定 |
|---|---|---|---|
| `setGroupId(...)` | `:575-581` | `groupId()`（`:4590-4592`）直接返回 `_groupId`（默认 null），**无 Expects** | ✅ 无崩溃；⚠️ 专辑中的 unsupported 媒体退化为独立气泡（布局异常，**改动前既有**） |
| `setReactions(...)` | `:582` | `setReactions` 首行 `Expects(!_reactions)`；不调用 → 断言**不触发**。读取侧 `reactions()`（`:3550-3553`）为 null-safe：`_reactions ? ... : kEmpty` | ✅ 无崩溃，✅ 无断言 |
| `applyTTL(data)` | `:583` | `ttlSeconds()`（`:3972`）为 `((media() && media()->ttlSeconds()) \|\| unsupportedTTL()) && !hasUnreadMediaFlag()`，media 为 null 时短路 | ✅ 无崩溃 |
| factcheck 组件 | `:585-589` | 全部读取点均有 `Has<>()`/`Get<>()` 守卫，如 `history_item.cpp:2069` `if (const auto factcheck = Get<HistoryMessageFactcheck>())`、`:4632-4633` `!Has<HistoryMessageFactcheck>() \|\| ...` | ✅ 无崩溃 |

**依赖 reactions 的渲染路径**：`reactions()` 返回 `static const auto kEmpty`，渲染侧拿到空 vector 走"无 reaction"分支 → ✅。

**`MessageFlag::Legacy` 的完整消费面**【确定】：全仓**仅 1 处**
`Telegram/SourceFiles/history/history_item.h:707-709`
```cpp
	[[nodiscard]] bool isLegacyMessage() const {
		return _flags & MessageFlag::Legacy;
	}
```
唯一调用者 `history_item.cpp:3089`（`canBeEdited()` 内）：
```cpp
		&& (!_media || _media->allowsEdit())
		&& !isLegacyMessage()
```
→ **只影响"能否编辑"**，不影响渲染、不影响转发、不影响删除。✅

**`_flags` 状态一致性**：`:492-493` 的 `&= ~HasPostAuthor` / `|= Legacy` 与 `:513` 的 `|= HasReplyMarkup` 均为**幂等位操作**，不依赖任何未初始化的组件。`_media` 保持 `nullptr`（构造函数中从未 `setMedia`）✅。

> **Q3 小结**：无崩溃、无断言、无 `_flags` 不一致。仅存"专辑布局退化"这一**改动前既有**的展示层异常。

---

## 5. Q4 — 实体安全

### 5.1 未引入新的实体类别【确定】

`Api::EntitiesFromMTP`（`api_text_entities.h:23-25`）把服务端 `Vector<MessageEntity>` 转成 `EntitiesInText`，与正常分支（`history_item.cpp:569-571`）**同一个函数、同一份数据**。本次改动没有新增、没有手工构造、没有拼接任何实体。

进入渲染的实体类别（`textUrl` / `customEmoji` / `pre` / `code` / `mention` / `url` …）全部来自服务端 `message.entities`，**与任何普通消息的实体集合完全同源**。

### 5.2 处理链路与正常路径同构【确定】

```
Unsupported 分支 :500-508  ┐
                          ├─→ TextWithEntities{ QString, EntitiesInText }
正常分支       :567-573   ┘        │
                                    ├─→ setText()          :4439
                                    │     ├─ filterZalgo()        :4443
                                    │     ├─ applyLocalPremiumEmoji :4446
                                    │     ├─ detectTextLinks()     :4448  (仅置 HasTextLinks 位)
                                    │     └─ setTextValue()        :4550 → _text
                                    └─→ 渲染
```
`detectTextLinks`（`history_item.cpp`）**只读实体、不新增**：
```cpp
	for (const auto &entity : textWithEntities.entities) {
		auto type = entity.type();
		if (type == EntityType::Url
			|| type == EntityType::CustomUrl
			...
			_flags |= MessageFlag::HasTextLinks;
```
→ 不会把纯文本里的裸 URL 提升为实体，**无自动加链导致的意外点击面**。

### 5.3 HTML 转义 / 自定义实体渲染【无法确定】

真正把 `TextWithEntities` 转成可绘制文本的代码位于 **`Telegram/lib_ui`（子模块，目录计数 0）**，本 checkout 无从读取。

- 检索命令与范围见 §1 与 §6。
- **未验证项**：`TextWithEntities → Ui::Text` 的转义/HTML 处理、`EntityType::Pre`（代码块）与 `EntityType::CustomEmoji` 的具体渲染。
- **间接判断**：该链路对**所有**消息共用，本次改动没有为 unsupported 消息开辟任何旁路。因此**风险等级与普通消息相同**，不因本改动升高或降低。

### 5.4 与 `EnsureNonEmpty` 的差异 —— 是否漏掉空文本保护？【确定：未漏】

`Telegram/SourceFiles/history/history_item_helpers.cpp:1400-1402`：
```cpp
[[nodiscard]] TextWithEntities EnsureNonEmpty(
		const TextWithEntities &text) {
	return !text.text.isEmpty() ? text : TextWithEntities{ u":-("_q };
}
```
对比：

| 路径 | 空判口径 | 空时结果 |
|---|---|---|
| 正常分支 `:573` `setText(_media ? textWithEntities : EnsureNonEmpty(textWithEntities))` | `text.text.isEmpty()` | `":-"` |
| 新 Unsupported 分支 `:506` `textWithEntities.empty()` | `TextWithEntities::empty()`（**定义未知**） | `UnsupportedMessageText()` |

**空文本保护没有漏** —— 新代码自带空判分支，且语义等价（"有内容用内容，没内容用占位"）。二者仅在占位文案与判定口径上不同，**属设计意图**。

唯一分歧点在**判定口径**：`EnsureNonEmpty` 只看 `text`，新代码看 `empty()`。若 `empty()` 是 `text.isEmpty() && entities.isEmpty()`，则"文本空 + 实体非空"时新代码会走"用文本"分支 → 空泡。→ §6 R1。

---

## 6. Q5 — Story 路径

**Story 的 unsupported 分支与本改动不同构，且完全未被波及。**【确定】

`CheckMessageMedia` 全仓**仅 3 个调用点，全部在 `history_item.cpp`**：
- `history_item.cpp:489`（构造函数）
- `history_item.cpp:2300`（`applyEdition`）
- `history_item.cpp:2634`（`updateSentContent`）

Story 走**完全独立的机制** —— `Telegram/SourceFiles/data/data_stories.cpp:101-102`：
```cpp
	}, [&](const MTPDmessageMediaUnsupported &data) {
		return std::make_optional(StoryMedia{ v::null });
```
即 `StoryMedia` 变体中的 `v::null_t` 哨兵，**不经过 `HistoryItem`、不经过 `setText`、不产生 `TextWithEntities`**。

补充：
- `history_item.cpp:462-463` 的 `MTPDmessageMediaUnsupported → nullptr` 属 **`CreateMedia`**（媒体对象工厂），不是 unsupported 文本分支：
  ```cpp
	}, [](const MTPDmessageMediaUnsupported &) -> Result {
		return nullptr;
  ```
- 结论：**Story 的 unsupported 消息仍显示其自有文案，本次改动对 Story 零影响、零回归风险**。

---

## 7. 问题清单（按严重度）

### 【确定 BUG】 — **0 个**

未发现任何可导致崩溃、断言失败、错误渲染、错误编译或数据损坏的缺陷。

### 【风险】 — 4 个

---

**R1 — `.empty()` 与 `EnsureNonEmpty` 空判口径可能分歧 → 边缘空泡**
- 位置：`history_item.cpp:506`、`:2333`、`:2647`
- 机制：`EnsureNonEmpty` 用 `text.text.isEmpty()`（`history_item_helpers.cpp:1401`）；新代码用 `TextWithEntities::empty()`，其**定义不可核**（§2.1）。
- 若 `empty()` 语义为 `text.isEmpty() && entities.isEmpty()`：当服务端下发 `message:""` 且 `entities` 非空时，`.empty()` 为 **false** → `setText(TextWithEntities{"", entities})` → 渲染出一个**纯空白、无任何提示的文本块**（`UnsupportedMessageMarkup()` 的 "Update AyuGram" 按钮仍在，但正文为空）。
- 可达性：`api.tl:124` 中 `message:string` 必填、`entities:flags.7?Vector<MessageEntity>` 可选，schema **不约束** entities 与 message 同时为空/非空的关系。编辑路径 `edition.textWithEntities` 同理。
- 影响：低（需畸形/恶意数据）；后果为视觉退化，非崩溃。
- 建议（**未实施，仅记录**）：改用 `textWithEntities.text.isEmpty()` 以与 `EnsureNonEmpty` 口径对齐。

**R2 — 第二个 `Unsupported` 入口 `webPageNotModified` 被一并波及**
- 位置：`history_item_helpers.cpp:1082-1083` → `history_item.cpp:491`
- 事实：该入口把"**网页预览未修改（复用缓存）**"判为 Unsupported，语义与"客户端不支持的媒体类型"**不同**。代码库自身在两处将其标记为异常：
  - `history_item.cpp:404-406`
    ```cpp
		}, [](const MTPDwebPageNotModified &) -> Result {
			LOG(("API Error: "
				"webPageNotModified is unexpected in message media."));
    ```
  - `data_session.cpp:4152-4154`
    ```cpp
	case mtpc_webPageNotModified:
		LOG(("API Error: "
			"webPageNotModified is unexpected in Session::webpage()."));
  ```
- 后果：该类消息现在显示 caption、网页预览丢失、仍被置 `Legacy` 禁编辑。
- 影响：低（防御性路径，触发概率低）；但**超出了需求"unsupported 消息显示部分文本"的原始语义边界**，建议在测试中单列。

**R3 — 占位提示从正文消失，唯一信号变为一个 URL 按钮（UX / 设计权衡）**
- 位置：`history_item.cpp:506-508` vs `history_item_helpers.cpp:1404-1413`
- 事实：caption 非空时，`lng_message_unsupported`（`lang.strings:2530`）整段提示语不再出现在正文；用户唯一可感知的"此消息不受支持"信号是 `UnsupportedMessageMarkup()`（`history_item_helpers.cpp:1415-1426`）生成的 **"Update AyuGram" URL 按钮**。
- 影响：这是需求的**预期代价**（突破"请更新最新版本"限制），但可发现性下降，属产品决策范畴 → 提请确认，非缺陷。

**R4 — 桌面通知文案连带改变，diff 未显式覆盖**
- 位置：`history_item.cpp:4691-4699` `notificationText()`（连带生效，非 diff 改动点）
- 现象：`_media == nullptr` → 返回 `_text`。改动前系统通知显示占位文案，改动后显示真实 caption。
- 判定：方向正确、与气泡一致、**非缺陷**；但属第 4 个用户可见面，建议纳入回归用例。

### 【无法确定】 — 3 项

| # | 事项 | 状态 |
|---|---|---|
| U1 | `TextWithEntities::empty()` 的**精确定义** | **未找到**。命令：`grep -rnE "^(class\|struct) TextWithEntities" --include=*.h --include=*.cpp .`；`grep -rn "TextWithEntities" Telegram/lib_base/ Telegram/lib_ui/ Telegram/SourceFiles/basic/ Telegram/SourceFiles/ui/`；`find . -iname "*text_with_entities*"`；codegraph `nodes WHERE qualified_name LIKE 'TextWithEntities'` → 0。范围：整个仓库工作树 + `.codegraph/codegraph.db`。**定义位于未初始化的 `Telegram/lib_base` / `Telegram/lib_ui` 子模块（`.gitmodules:19-21`、`:25-27`；两目录 `ls -A \| wc -l` 均为 0）** |
| U2 | `TextWithEntities → Ui::Text` 的 **HTML 转义 / `pre` / `customEmoji` 渲染** | **未找到**。命令同 U1（`Telegram/lib_ui`）。范围：全仓工作树 + codegraph（`file_path LIKE '%lib_ui%'` → 0 行）。判定：该链路为全消息共用，本次改动未新增旁路，故风险不高于普通消息 |
| U3 | `EntitiesInText` / `EntityType` 枚举定义 | **未找到**，同 U1 范围与命令 |

---

## 8. 结论 / Verdict

**CONDITIONAL PASS**

**站得住的部分（已逐行核实）**：
1. 三处改动**互相一致**，且新构造代码与 HEAD 正常分支 `history_item.cpp:567-572` **逐字符同源**，复用已验证的编译路径。
2. 类型层面零缺陷：`setText` 单重载（`history_item.h:568`）、`UnsupportedMessageText()` 返回 `TextWithEntities`（`history_item_helpers.h:102`）、三元两支同型且合成纯值、嵌套三元 5 支全同型 → **无歧义、无悬垂、无隐式转换**。
3. `TextWithEntities::empty()` **确实存在**（3 处 HEAD 既有调用佐证），非新增 API。
4. **无确定 BUG**。崩溃/断言面已逐条排查并排除：`setReactions` 的 `Expects(!_reactions)` 因不调用而不触发；`reactions()`（`:3550`）null-safe；factcheck 全部读取点有 `Has<>`/`Get<>` 守卫；`Legacy` 全仓仅 1 处消费（`history_item.h:707` → `history_item.cpp:3089` 禁编辑），不影响渲染。
5. 占位文案可达点仅 3 处，**100% 覆盖**；编辑路径的 `serviceText` 分支经 `prepareServiceTextForMessage` catch-all（`:5448-5449`）返回空，**不会**吞掉新文本。
6. Story 走 `data_stories.cpp:101-102` 的 `v::null` 独立机制，**未被波及**。
7. 无本地文本搜索索引；`hasTextLinks()` 唯一消费点是共享媒体 Link 掩码，且改动前占位文案已含 `textUrl` → **索引行为无净变化**。
8. 空文本保护**未漏**（新代码自带空判分支，语义等价于 `EnsureNonEmpty`）。

**放行前建议（均未实施，本轮零代码改动）**：
- [ ] **R1**：确认 `TextWithEntities::empty()` 语义；若为 `text && entities` 组合判，改为 `text.isEmpty()` 与 `EnsureNonEmpty` 对齐（低优先级）。
- [ ] **R2**：为 `webPageNotModified` 入口单列测试用例，确认预期文案。
- [ ] **R3**：产品侧确认"正文无提示 + 仅剩 Update AyuGram 按钮"的可发现性可接受。
- [ ] **R4**：回归用例覆盖系统通知文案。
- [ ] 在**已初始化子模块**的构建机上补验 U1 / U2（本 checkout 因 37 个子模块全部为空而无法核验）。

---

*本报告为只读审查产物。审查期间未修改 `Telegram/` 下任何文件，未执行 git 写操作，未配置/构建/运行测试。唯一写入为本文件。*
