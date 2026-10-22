# T1 — `saveCheckpointOnDoubleRightClick` 设置页 UI 开关

任务：为 `AyuSettings::saveCheckpointOnDoubleRightClick` 增加设置页可点击开关（原只能手改 `tdata/ayu_settings.json`）。
环境：静态交叉核验（无 cmake/ninja/clang/docker，子模块未初始化，无 out/），未构建、未编译。

## 1. 锚点证据（API 取证）

### 1.1 `addSettingToggle` 真实字段定义

`Telegram/SourceFiles/ayu/ui/settings/ayu_builder.h:22-32`
```cpp
	struct SettingToggleArgs {
		QString id;
		QStringList altIds;
		rpl::producer<QString> title;
		BoolGetter getter;
		BoolSetter setter;
		IconDescriptor icon;
		QStringList keywords;
		rpl::producer<bool> shown;
	};
	Ui::SettingsButton *addSettingToggle(SettingToggleArgs &&args);
```

结论：**API 适用，无阻断**。`title` 确为 `rpl::producer<QString>` → 必须传 `tr::ayu_XxxText()`（不带 `tr::now`）；`getter`/`setter` 是具名 typedef，非 `Fn<>`。

### 1.2 `BoolGetter` / `BoolSetter` 精确签名

`Telegram/SourceFiles/ayu/ui/settings/settings_ayu_utils.h:21-22`
```cpp
using BoolGetter = bool (AyuSettings::*)() const;
using BoolSetter = void (AyuSettings::*)(bool);
```

### 1.3 成员函数签名匹配

`Telegram/SourceFiles/ayu/ayu_settings.h:271`
```cpp
	[[nodiscard]] bool saveCheckpointOnDoubleRightClick() const { return _saveCheckpointOnDoubleRightClick.current(); }
```
`Telegram/SourceFiles/ayu/ayu_settings.h:359`
```cpp
	void setSaveCheckpointOnDoubleRightClick(bool val);
```
定义在 `Telegram/SourceFiles/ayu/ayu_settings.cpp:541-543`。

逐项比对：
| 需求 | 实际 | 结果 |
| --- | --- | --- |
| `bool() const` | `bool saveCheckpointOnDoubleRightClick() const` | ✅ 完全匹配 `BoolGetter` |
| `void(bool)` | `void setSaveCheckpointOnDoubleRightClick(bool val)` | ✅ 完全匹配 `BoolSetter` |
| 可取地址的成员 | 均为普通成员函数（非 static/非重载单例） | ✅ |

### 1.4 声明式行先例

`Telegram/SourceFiles/ayu/ui/settings/settings_ayu.cpp:641-652`（既有 `saveDeletedMessages` / `saveMessagesHistory`，均直接取成员函数地址、`title` 不带 `tr::now`）——新增行完全照抄该形态。

### 1.5 lang key 先例

`Telegram/Resources/langs/lang.strings:8913-8915`
```
"ayu_SaveCheckpointContextMenuText" = "Save AG Checkpoint";
"ayu_JumpToCheckpointMenuText" = "Jump to AG Checkpoint";
"ayu_ClearCheckpointMenuText" = "Clear AG Checkpoint";
```
`tr::` 生成名逐字等于 key：`Telegram/SourceFiles/ayu/ui/context_menu/context_menu.cpp:1034` 使用 `tr::ayu_SaveCheckpointContextMenuText(tr::now)`。

## 2. 最终代码

### 2.1 `Telegram/SourceFiles/ayu/ui/settings/settings_ayu.cpp:653-658`

置于 `BuildSpyEssentials` 内、紧接 `saveMessagesHistory` 之后：

```cpp
	ayu.addSettingToggle({
		.id = u"ayu/saveCheckpointOnDoubleClick"_q,
		.title = tr::ayu_SaveCheckpointOnDoubleRightClickText(),
		.getter = &AyuSettings::saveCheckpointOnDoubleRightClick,
		.setter = &AyuSettings::setSaveCheckpointOnDoubleRightClick,
	});
```

### 2.2 `Telegram/Resources/langs/lang.strings:8916`

紧跟三个 `ayu_*Checkpoint*MenuText` 键之后：

```
"ayu_SaveCheckpointOnDoubleRightClickText" = "Save checkpoint on double right-click";
```

## 3. git diff --stat

```
 Telegram/Resources/langs/lang.strings                 | 4 ++++
 Telegram/SourceFiles/ayu/ui/settings/settings_ayu.cpp | 6 ++++++
 2 files changed, 10 insertions(+)
```

说明：`lang.strings` 的 4 行增量中，**前 3 行（`ayu_SaveCheckpointContextMenuText` / `ayu_JumpToCheckpointMenuText` / `ayu_ClearCheckpointMenuText`）是本次任务之前已存在的未提交改动**（checkpoint 功能自带，见 §1.5），非本任务产生；本任务实际新增 1 行。cpp 的 6 行全部为本任务新增。未触碰任何无关既有改动，未改 `HANDOFF.md` / `CONTEXT.md` / `docs/` 下其他文件。

## 4. 自查清单

- [x] 字段名存在：`ayu_settings.h:271`（getter）、`ayu_settings.h:359`（setter）、`ayu_settings.cpp:541-543`（定义）。
- [x] getter/setter 签名匹配 `BoolGetter`/`BoolSetter`：逐字符比对通过（§1.3）。
- [x] `.title` 为 `rpl::producer<QString>` 口径 → 传 `tr::...Text()` 不带 `tr::now`（照抄 `settings_ayu.cpp:643` 先例）。
- [x] lang key 全仓唯一：`grep -rn 'ayu_SaveCheckpointOnDoubleRightClickText'` 仅 2 处命中 —— `lang.strings:8916`（定义）与 `settings_ayu.cpp:655`（引用）；`grep -c` 在 `lang.strings` 内为 1。
- [x] `_q` 后缀用于 `QString` 字面量；未写注释；未用 `Q_OS_LINUX`；未硬编码尺寸（本改动无任何尺寸/样式值）。
- [x] 仅修改允许的两个文件 + 本报告；改前 `claim_files` 认领，改后 `release_files` 释放。
- [x] 行尾 LF、无 BOM（`python3` 校验：`lang.strings` CRLF=0/LF=9040/BOM=False；`settings_ayu.cpp` CRLF=0/LF=730/BOM=False）。
- [x] 未构建、未 install/npm/pip、未 git commit/checkout/reset/stash。

## 5. 备注（非阻断观察）

- 开关 id 按任务指定用 `u"ayu/saveCheckpointOnDoubleClick"_q`，与 lang key 语义词 `SaveCheckpointOnDoubleRightClick` 略有差异（`OnDoubleClick` vs `OnDoubleRightClick`）。id 仅用于设置项内部标识/搜索条目，不与 lang key 强绑定；如需完全对齐可改为 `ayu/saveCheckpointOnDoubleRightClick`——本次按任务原文执行，未擅改。
- 该开关未加入 `BuildSpyEssentials` 的搜索条目列表（`settings_ayu.cpp:608-634` 那段 `sctx.entries` 属于 `BuildGhostEssentials`，`BuildSpyEssentials` 本身不含搜索项注册，故保持与同组 `saveDeletedMessages` / `saveMessagesHistory` 一致，不额外添加）。
- 【无法确定】开关的可见默认状态：`ayu_settings.h:632` 为 `rpl::variable<bool> _saveCheckpointOnDoubleRightClick = true;`，故首次打开设置页应显示为「已开启」；但设置页的 `addSettingToggle` 读取路径（`settings_ayu_utils.cpp` 内实现）本次未逐行读取，仅确认 getter 返回 `current()`。
