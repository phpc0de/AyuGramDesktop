# 未初始化子模块导致的盲区与假设清单

> 产出目的：供**外部构建机一次性核对**。本文件是只读审计工件，**未修改任何应用代码**（`Telegram/` 下零写入），未执行 git、未配置/构建/安装/运行测试。
> 仓库：`/root/workspace/AyuGramDesktop`（AyuGramDesktop，tdesktop v7.0.9 fork）。
> 生成时间：审计回合（会话内）。所有行号基于本 checkout 当前工作树。

## 0. 审计范围、方法与命令清单

### 0.1 读入的文档（本轮唯一输入源）

| 文档 | 行数 | 说明 |
|---|---|---|
| `CONTEXT.md` | 132 | /grill 会话状态同步工件 |
| `docs/adr/0001-ag-checkpoint.md` | 28 | AG 检查点语义与持久化选型 |
| `docs/adr/0002-unlock-restricted-chats.md` | 31 | 三道门解锁 + 不可见内容策略 |
| `docs/adr/0003-editable-message-replay.md` | 37 | Edit & Resend / MTProto 重放两期方案 |
| `docs/adr/0004-build-iteration-strategy.md` | 24 | 外部构建机 + 静态交叉核验策略 |
| `docs/p0-build-fix-plan.md` | 65 | P0 phantom accessor 改动单 |

### 0.2 跑过的命令（全部只读；无 git、无构建、无写入）

```bash
# 目录 / 存在性
ls -la docs/ docs/adr/ docs/review/ ; ls -la CONTEXT.md
for d in cmake Telegram/lib_base Telegram/lib_ui Telegram/lib_tl Telegram/lib_storage \
         Telegram/codegen Telegram/ThirdParty; do ls -A "$d" | wc -l; done
grep -oP '(?<=^\tpath = ).*' .gitmodules | while read -r d; do ls -A "$d" | wc -l; done   # 37 条全部 EMPTY
grep -c 'submodule' .gitmodules                                                                    # = 37

# 定点取证（sed -n | nl -ba，全部只读）
sed -n '918,940p;1043,1102p'  Telegram/SourceFiles/data/data_session.cpp
sed -n '528,545p;3828,3860p;4000,4045p' Telegram/SourceFiles/apiwrap.cpp
sed -n '100,106p;122,126p' Telegram/SourceFiles/mtproto/scheme/api.tl
sed -n '20,32p' Telegram/SourceFiles/data/data_chat.h ; sed -n '68,80p' Telegram/SourceFiles/data/data_chat.cpp
sed -n '780,792p' Telegram/SourceFiles/data/data_channel.cpp ; sed -n '236,248p' Telegram/SourceFiles/ayu/features/forward/ayu_forward.cpp
sed -n '2528,2532p' Telegram/Resources/langs/lang.strings ; sed -n '18,30p' Telegram/SourceFiles/api/api_editing.h
sed -n '225,236p;300,316p;415,450p;17,39p' Telegram/SourceFiles/mtproto/sender.h
sed -n '155,175p' Telegram/SourceFiles/mtproto/mtp_instance.h
sed -n '65,80p' Telegram/SourceFiles/mtproto/mtproto_response.h
sed -n '488,510p;555,570p;3048,3060p;3073,3080p;3120,3140p' Telegram/SourceFiles/history/history_item.cpp
sed -n '928,936p;1106,1112p' Telegram/SourceFiles/history/history_item_helpers.cpp
sed -n '688,703p;1100,1110p' Telegram/SourceFiles/data/data_user.cpp ; sed -n '130,138p' Telegram/SourceFiles/data/data_peer.cpp
sed -n '655,663p;3977,3983p' Telegram/SourceFiles/history/history_inner_widget.cpp
sed -n '1078,1084p;2912,2935p' Telegram/SourceFiles/boxes/peers/edit_peer_info_box.cpp
sed -n '108,116p' Telegram/SourceFiles/info/media/info_media_provider.cpp
sed -n '2658,2666p;425,436p' Telegram/SourceFiles/history/view/history_view_context_menu.cpp
sed -n '5985,5996p' Telegram/SourceFiles/history/view/history_view_list_widget.cpp
sed -n '8220,8230p;8323,8327p' Telegram/SourceFiles/history/history_widget.cpp
sed -n '13,24p' Telegram/SourceFiles/ayu/data/ayu_database.cpp ; sed -n '20,80p' Telegram/SourceFiles/ayu/ayu_infra.cpp
sed -n '2253,2260p;2649,2656p;770,776p' Telegram/SourceFiles/mtproto/session_private.cpp
sed -n '420,428p' Telegram/SourceFiles/mtproto/session.cpp
sed -n '1065,1075p' Telegram/build/docker/centos_env/Dockerfile ; cat -n Telegram/build/docker/centos_env/run.sh Telegram/build/docker/centos_env/build_debug.sh
sed -n '1,80p;255,270p;2280,2300p' Telegram/CMakeLists.txt
sed -n '498,515p' .agents/shared/test-loop.md

# 全仓 grep
grep -rn "CHAT_FORWARDS_RESTRICTED" .          grep -rn "ayuNoforwards" --include=*.tl --include=*.h --include=*.cpp .
grep -rn "Sender::send" .                      grep -n "style_files" Telegram/CMakeLists.txt
grep -n "toggleNoForwards" Telegram/SourceFiles/mtproto/scheme/api.tl
grep -n "is_ayuNoforwards\|is_stories_hidden_min\|is_stories_unavailable" Telegram/SourceFiles/data/data_session.cpp

# 资源面
find . -name "*.style" -not -path "./.git/*" | wc -l     # 71
find Telegram/Resources -name "*.svg" | wc -l            # 294
ls Telegram/SourceFiles/styles/                          # 不存在
ls Telegram/SourceFiles/iv/                              # 存在（in-repo，非盲区）
```

**未执行**：`git submodule status`（任务明令禁 git）。§1 的"37 个子模块未初始化"改用**文件系统核验**得出，结论等价且更强（见下）。

---

## 1. 盲区地图

### 1.1 【确定】37 个子模块目录在盘上**全部为空**（0 条目）

核验方式（不依赖 git）：遍历 `.gitmodules` 的 37 条 `path =`，逐个 `ls -A | wc -l`，**37/37 返回 0**。

`.gitmodules:1-111` 共 37 个 `[submodule "..."]` 段。其中 **15 个是库/构建必需**，全部 EMPTY：

| 路径 | `.gitmodules` 行 | 在盘 | 对本审计的影响 |
|---|---|---|---|
| `cmake` | :40-42 | EMPTY | 全部 CMake 规则、codegen 触发逻辑、自定义函数不可见 |
| `Telegram/lib_base` | :19-21 | EMPTY | `base::` / `not_null` / `Fn` / `gsl::` 符号不可见 |
| `Telegram/lib_ui` | :25-27 | EMPTY | `Ui::` 符号不可见 |
| `Telegram/lib_tl` | :31-33 | EMPTY | `MTP::` / `MTPD*` / `MTPC*` 运行时类型不可见 |
| `Telegram/lib_storage` | :37-39 | EMPTY | `Storage::` 不可见（本轮文档未直接引用，见 §7） |
| `Telegram/codegen` | :22-24 | EMPTY | scheme/style/lang codegen 源码不可见 |
| `Telegram/lib_crl` | :13-15 | EMPTY | `crl::time` / `crl::async` 不可见 |
| `Telegram/lib_rpl` | :16-18 | EMPTY | `rpl::` / `rpl::type_erased` 不可见 |
| `Telegram/lib_lottie` | :28-30 | EMPTY | 动画 |
| `Telegram/lib_qr` | :49-51 | EMPTY | 二维码 |
| `Telegram/lib_spellcheck` | :34-36 | EMPTY | 拼写检查 |
| `Telegram/lib_translate` | :91-93 | EMPTY | 翻译 |
| `Telegram/lib_webrtc` | :67-69 | EMPTY | WebRTC |
| `Telegram/lib_webview` | :73-75 | EMPTY | WebView |
| `Telegram/lib_icu` | :94-96 | EMPTY | AyuGram filters |

其余 22 个为 `Telegram/ThirdParty/*`（`.gitmodules` 各段），全部 EMPTY，含 `GSL`(:1-3)、`xxHash`(:4-6)、`rlottie`(:7-9)、`lz4`(:10-12)、`expected`(:43-45)、`QR`(:46-48)、`hunspell`(:52-54)、`range-v3`(:55-57)、`nimf`(:58-60)、`hime`(:61-63)、`fcitx5-qt`(:64-66)、`tgcalls`(:70-72)、`kimageformats`(:76-78)、`kcoreaddons`(:79-81)、`cld3`(:82-84)、`libprisma`(:85-87)、`xdg-desktop-portal`(:88-90)、`cmark-gfm`(:97-99)、`MicroTeX`(:100-102)、`TooManyCooks`(:103-105)、`libcbor`(:106-108)、`libfido2`(:109-111)。

### 1.2 【确定】codegen 产物一律不在盘

| 产物 | 生成源 | 盘上状态 |
|---|---|---|
| `MTPD*` / `MTPC*` / `MTPP*` / `MTP_*` 运行时类型 | `Telegram/SourceFiles/mtproto/scheme/{api,mtproto}.tl` | **不在盘**。`Telegram/SourceFiles/mtproto/scheme/` 只有 `api.tl`(278,916 B) 与 `mtproto.tl`(5,663 B) 两个 schema 源文件 |
| `tr::lng_*` 翻译符号 | `Telegram/Resources/langs/lang.strings` | **不在盘**（源文件在，2,530+ 行） |
| `st::*` 样式符号 | 71 个 `Telegram/SourceFiles/**/*.style` | **不在盘**。`Telegram/SourceFiles/styles/` 目录**不存在** |
| `Ui::` / `base::` / `rpl::` / `crl::` | lib_ui / lib_base / lib_rpl / lib_crl | **不在盘** |

**推论（对本审计最关键）**：本仓 6 份文档中几乎所有**符号级断言**（"X 类有方法 Y"、"常量 Z 存在"）都无法在本 checkout 证实或证伪，只能对照**调用点**与 **TL schema 源**间接推断。

### 1.3 【确定】构建树与外部依赖不在盘

- 无 `out/`（与 `AGENTS.md:54`「`out/` 目录已配置」的前提**冲突**——已在 `CONTEXT.md:76` 记录，此处复核成立）。
- `Telegram/CMakeLists.txt:10-22,25` 有 15 个 `add_subdirectory(lib_*)`/`add_subdirectory(codegen)`，**全部指向空目录**。
- `Telegram/CMakeLists.txt:31-46,49,53,2293-2294` 有 17 个 `include(cmake/*.cmake)`，**全部指向不存在的文件**。
- `Telegram/CMakeLists.txt:68-80` 链接的 10 个生成/库目标（`tdesktop::td_lang` / `td_mtproto` / `td_scheme` / `td_export` / `td_iv` / `td_ui` / `td_tde2e` / `td_webauthn` / `desktop-app::lib_*`）**定义在缺失的 CMake 中**。

---

## 2. 【确定】项（已在本 checkout 逐条复核，可直接采信）

| # | 结论 | 证据（相对路径:行号 + 摘录） |
|---|---|---|
| C1 | P0 编译故障点确实存在，两处、形态一致 | `Telegram/SourceFiles/data/data_session.cpp:933-934` `\| (data.is_noforwards() ? Flag::NoForwards : Flag())` / `\| (data.is_ayuNoforwards() ? Flag::AyuNoForwards : Flag());`；`:1097-1098` 同形（**无结尾分号**，与 `p0-build-fix-plan.md:43` 备注一致） |
| C2 | `ayuNoforwards` 全仓只有这 2 处**调用**，无声明、无定义、无 TL 字段 | `grep -rn "ayuNoforwards" --include=*.tl --include=*.h --include=*.cpp .` → 恰好 2 命中，即上述两行 |
| C3 | `api.tl` 无 `ayuNoforwards`；`chat` 只有 `noforwards:flags.25?true` | `Telegram/SourceFiles/mtproto/scheme/api.tl:102` `chat#41cbf256 flags:# ... noforwards:flags.25?true id:long ...` |
| C4 | `api.tl` layer = 228 | `Telegram/SourceFiles/mtproto/scheme/api.tl:3087` `// LAYER 228`（文件共 3087 行） |
| C5 | 改动单引用的 channel 行号**有误**：应为 `:104` 而非 `:103` | `api.tl:103` = `chatForbidden#6592a1a7 id:long title:string = Chat;`；`api.tl:104` = `channel#d49f34c6 ... noforwards:flags.27?true ...` |
| C6 | `message:string` 在 `message#7600b9d3` 中**必填**（需求 6 前提） | `api.tl:124` `... date:int message:string media:flags.9?MessageMedia ... rich_message:flags2.13?RichMessage = Message;` |
| C7 | `flagsMask` 已含 `Flag::AyuNoForwards`（Chat/Channel 各一处） | `data_session.cpp:922-923` `\| Flag::NoForwards` / `\| Flag::AyuNoForwards;`；`:1046-1047` 同 |
| C8 | 私有标志位定义与取值 | `Telegram/SourceFiles/data/data_chat.h:27` `AyuNoForwards = (1 << 31),` |
| C9 | `allowsForwarding()` 只检查原生 `Flag::NoForwards` | `data_chat.cpp:75-77` `return !(flags() & Flag::NoForwards);`；`data_channel.cpp:787-789` 同 |
| C10 | AyuForward peer 级分流已存在 | `Telegram/SourceFiles/ayu/features/forward/ayu_forward.cpp:243-245` `bool isFullAyuForwardNeeded(not_null<HistoryItem*> item) { return item->from()->isAyuNoForwards() \|\| item->history()->peer->isAyuNoForwards(); }` |
| C11 | 分流已接在 `ApiWrap::forwardMessages` 最前 | `Telegram/SourceFiles/apiwrap.cpp:3834-3840` `const auto fullAyuForward = AyuForward::isFullAyuForwardNeeded(draft.items.front()); if (fullAyuForward) { crl::async(...); return; }`（`forwardMessages` 定义起始于 `:3828`） |
| C12 | `CHAT_FORWARDS_RESTRICTED` 的**唯一**代码位置是错误串→toast 映射 | `apiwrap.cpp:583` `} else if (show && error == u"CHAT_FORWARDS_RESTRICTED"_q) {` → `:584-588` 三元选 `tr::lng_error_noforwards_{channel,user,group}` |
| C13 | `forwardMessages` 的失败回调**统一**走 `sendMessageFail`（回答 ADR-0002:28 开放问题） | `apiwrap.cpp:4026-4037`：`[=](const MTP::Error &error, const MTP::Response &) { if (idsCopy) { for (...) { _session->api().sendMessageFail(error, peer, randomId, itemId); } } else { _session->api().sendMessageFail(error, peer); } }` |
| C14 | 消息级先例成立 | `history_item_helpers.cpp:931-933` `// AyuGram: removed` / `// \| ((flags & MTP::f_noforwards) ? Flag::NoForwards : Flag())` / `\| (flags & MTP::f_noforwards ? Flag::AyuNoForwards : Flag())` |
| C15 | 用户级先例成立 | `data_user.cpp:1105-1107` `user->setNoForwardsFlags(update.is_noforwards_my_enabled(), update.is_noforwards_peer_enabled());`；实现 `:694-703` |
| C16 | 机制 A 落点成立，Unsupported 分支**确实丢弃** `vmessage` | `history_item.cpp:491-501`：`if (checked == MediaCheckResult::Unsupported) { ... if (const auto richMessage = data.vrich_message()) { ...setText(Iv::FlattenRichPageSummary(richPage)); } else { setText(UnsupportedMessageText()); }` |
| C17 | 正常分支确实使用 `vmessage`（即"文本客户端可得"成立） | `history_item.cpp:559-565` `auto textWithEntities = TextWithEntities{ qs(data.vmessage()), Api::EntitiesFromMTP(&history->session(), data.ventities().value_or_empty()) };` |
| C18 | `Lng` key 在源文件存在 | `Telegram/Resources/langs/lang.strings:2530` `"lng_message_unsupported" = "This message is not supported by your version of Telegram...";` |
| C19 | `Iv::FlattenRichPageSummary` / `Iv::ParseRichPage` **在仓内**（非盲区） | `Telegram/SourceFiles/iv/` 存在（含 `iv_data.cpp` 等）；调用点 `history_item.cpp:496-498`、`:555-557` |
| C20 | `Legacy` 位保留会禁编辑 | `history_item.cpp:3077` `&& !isLegacyMessage()` 位于 `canBeEdited()` 链中 |
| C21 | 编辑类错误码是**客户端常量表** | `Telegram/SourceFiles/api/api_editing.h:27-30` `const auto kDefaultEditMessagesErrors = { u"MESSAGE_ID_INVALID"_q, u"CHAT_ADMIN_REQUIRED"_q, u"MESSAGE_EDIT_TIME_EXPIRED"_q, ...` |
| C22 | **`Sender::send` 符号不存在**（ADR-0003 引用错误，见 §6 E2） | `grep -rn "Sender::send" .` → 仅命中 ADR-0003 自身两行文档；`sender.h` 内 `send()` 唯一实现在 `:303`，属嵌套类 `SpecificRequestBuilder`（类定义 `:210`），`class Sender` 起于 `:17` |
| C23 | 真实出站上界是 `Instance::send` | `Telegram/SourceFiles/mtproto/mtp_instance.h:151-170` `template <typename Request> mtpRequestId send(...)` → `:162 sendSerialized(requestId, details::SerializedRequest::Serialize(request), ...)`；由 `sender.h:304` `sender()->_instance->send(...)` 调用 |
| C24 | `Response.reply` 原始字节可达 | `Telegram/SourceFiles/mtproto/mtproto_response.h:71-75` `struct Response { mtpBuffer reply; mtpMsgId outerMsgId = 0; mtpRequestId requestId = 0; };`；`MakeDoneHandler` 在 `sender.h:40-75`（ADR-0003 写 `:43-74`，实测 `:40-75`） |
| C25 | MTProto 关键函数定位成立 | `Telegram/SourceFiles/mtproto/session.cpp:422` `void Session::sendPrepared(`；`session_private.cpp:2255` `void SessionPrivate::resend(mtpMsgId msgId, crl::time msCanWait) {`；`session_private.cpp:2651` `bool SessionPrivate::sendSecureRequest(` |
| C26 | 群设置保存路径**确实发送真实 TL flag**（回答 `p0-build-fix-plan.md:57` 开放项） | `Telegram/SourceFiles/boxes/peers/edit_peer_info_box.cpp:2914-2925` `void Controller::saveForwards() { if (!_savingData.noForwards \|\| *_savingData.noForwards == _peer->isAyuNoForwards()) { return continueSave(); } using Flag = MTPmessages_ToggleNoForwards::Flag; _api.request(MTPmessages_ToggleNoForwards( MTP_flags(Flag()), _peer->input(), MTP_bool(*_savingData.noForwards), MTPint() ))`；TL 签名 `api.tl:2606` `messages.toggleNoForwards#b2081a35 flags:# peer:InputPeer enabled:Bool request_msg_id:flags.0?int = Updates;` |
| C27 | `Dockerfile:1069` 最终镜像禁用 ccache | `Telegram/build/docker/centos_env/Dockerfile:1069` `ENV CCACHE_DISABLE=true`；`:1072` `VOLUME [ "/usr/src/tdesktop" ]` |
| C28 | Debug 构建入口只 build 不 configure | `Telegram/build/docker/centos_env/build_debug.sh:9` `"$FullScriptPath/run.sh" bash -lc "cd /usr/src/tdesktop && cmake --build out --config Debug --target Telegram"`（文件共 10 行；`CONTEXT.md:81` 写 `:8-11` 越界） |
| C29 | `run.sh` 硬性依赖 `DesktopPrivate` | `Telegram/build/docker/centos_env/run.sh:7-14` `if [ ! -d "$FullScriptPath/../../../../../DesktopPrivate" ]; then ... exit; fi`（`CONTEXT.md:82` 写 `:9-16`） |
| C30 | **`run.sh` 资源限制是条件式**，非固定 `nproc-2` / `total-4G` | `run.sh:20-23`：`CpuCount=$(nproc)` / `DockerCpus=$((CpuCount > 4 ? CpuCount - 2 : CpuCount))` / `MemTotalGb=...` / `DockerMemoryGb=$((MemTotalGb > 12 ? MemTotalGb - 4 : MemTotalGb))` → **9.3G 机器上内存不扣 4G**，`CONTEXT.md:112` 的 "total-4G≈5.3G" 估算**错误** |
| C31 | 挂载行在 `:25`（非 `:22`） | `run.sh:25` `docker run -it --rm --cpus=$DockerCpus --memory=${DockerMemoryGb}g -u $(id -u) -v $HOME/Telegram/DesktopPrivate:/usr/src/DesktopPrivate -v $HOME/Telegram/tdesktop:/usr/src/tdesktop tdesktop:centos_env "$@"`（`CONTEXT.md:82`、`:124` 均写 `:22`） |
| C32 | `${style_files}` 在主 CMakeLists 中**只被使用、未被定义** | `Telegram/CMakeLists.txt:261` `    ${style_files}`；`grep -n "style_files" Telegram/CMakeLists.txt` → 仅此 1 命中 → 定义必在缺失的 `cmake/*.cmake` |
| C33 | style 源文件布局：71 个 `.style` 在 `Telegram/SourceFiles/**`（**不在** `Telegram/Resources/styles/`，该目录不存在） | 例：`Telegram/SourceFiles/history/view/*.style`、`Telegram/SourceFiles/iv/iv.style`、`Telegram/SourceFiles/profile/profile.style` |
| C34 | `.svg` 共 294 个，经 `.qrc` 打包 | `Telegram/Resources/qrc/telegram/telegram.qrc:14,23-31`（如 `<file alias="art/premium/star_texture.svg">../../art/premium/star_texture.svg</file>`）、`Telegram/Resources/qrc/ayu/ayu.qrc:3-27`；打包函数 `Telegram/CMakeLists.txt:2590` `target_prepare_qrc(Telegram)`（定义在缺失 cmake） |
| C35 | Ayu SQLite 家族与 `initDatabase` 落点 | `ayu/data/ayu_database.cpp:14-15` `auto storage = make_storage("./tdata/ayudata.db", ...)`；`:21-24` `make_index("idx_deleted_message_userId_dialogId_topicId_messageId", ...)`；加载点 `ayu/ayu_infra.cpp:47-49` `void initDatabase() { AyuDatabase::initialize(); }`（文件共 80 行；ADR-0001:18 写 `:22-60`） |
| C36 | 「codegen 不跟踪资源 mtime」是仓库内既有明文告警 | `.agents/shared/test-loop.md:507-511` `**Codegen does not track resource mtimes.** If the task changed only a resource the style codegen consumes (an icon .svg, etc.) without touching a .style, an incremental build will NOT re-pack it and the binary keeps the OLD asset.` |

---

## 3. 【假设（需外部核对）】清单

> 格式：`ID | 断言（出处 file:line） | 为什么本 checkout 无法判定 | 外部核对点`

### 3.1 编译期 / codegen 假设

| ID | 断言与出处 | 盲区成因 | 外部核对点 |
|---|---|---|---|
| **A1** | 「`data.is_ayuNoforwards()` 不存在 → HEAD 编不过」（`CONTEXT.md:45`、`:129`；`p0-build-fix-plan.md:7-12`；ADR-0002:16；ADR-0004:15） | C1/C2/C3/C4 已确证「无此 TL 字段」；但「字段缺失 ⟹ 访问器不生成 ⟹ 编译失败」这一步依赖 **codegen 的命名规则**（`is_<field>`），而生成器源码在空目录 `Telegram/codegen/`。**间接旁证**：同文件对 layer-228 已有字段一律用 `is_` 前缀（`data_session.cpp:1052` `data.is_stories_hidden_min()`、`:1071` `data.is_stories_unavailable()`，对应 `api.tl:104` `stories_hidden_min:flags2.2?true` / `stories_unavailable:flags2.3?true`）——**强烈支持但非证明** | 外部机 `grep -rn "is_ayuNoforwards\|ayuNoforwards" Telegram/lib_tl/ Telegram/codegen/ out/`；并以首次 Debug 构建的首个错误行为准 |
| **A2** | 「生成的 `MTPDchat`/`MTPDchannel` 无此方法」（`p0-build-fix-plan.md:12`） | `MTPD*` 由 lib_tl 的 scheme codegen 生成，头文件不在盘 | 外部机在生成目录 `grep -n "class MTPDchat\b" -A200 \| grep -n "ayuNoforwards"`；或直接看 `out/` 下的 `scheme.h` |
| **A3** | `MTP::f_noforwards` 常量名（`history_item_helpers.cpp:933`）由 codegen 从 `api.tl:124 noforwards:flags.26?true` 生成 | 常量名生成规则在缺失的 codegen 中；**注意层级冲突**：`chat` 用 `flags.25`、`channel` 用 `flags.27`、`message` 用 `flags.26`（`api.tl:102`/`:104`/`:124`）——三处位号不同，靠 `MTP::f_noforwards` 统一寻址的正确性由 codegen 保证 | 外部机核对 `f_noforwards` 的生成值是否为 25/26/27 之一且每构造子独立 |
| **A4** | `tr::lng_error_noforwards_{channel,user,group}` 等符号存在（`apiwrap.cpp:585-588`）、`tr::lng_message_unsupported` 存在（`history_item.cpp:500` 经 `UnsupportedMessageText()`） | `lang.strings` 源在盘，但 **`key → C++ 符号名` 的转换规则**在缺失的 `cmake/td_lang.cmake` + `Telegram/codegen`。C18 只证明 key 字符串在 | 外部机 `grep -n "lng_error_noforwards_group" out/*/lang/*.h`；或 `grep -c '"lng_error_noforwards' Telegram/Resources/langs/lang.strings` |
| **A5** | `st::*` 样式符号、`.style` 的 `using`/`icon{{}}` 解析规则 | 生成物目录 `Telegram/SourceFiles/styles/` **不存在**（已实测） | 外部机确认生成目录路径与 `styles/style_widgets.h` 内容 |
| **A6** | `.style` 输入如何被枚举（`${style_files}` 来自何处） | C32：只用未定义 → 定义在缺失的 cmake | 外部机 `grep -rn "style_files" cmake/` |
| **A7** | 改 `api.tl` 后 scheme codegen 的**重生成范围与重编面** | `cmake/td_scheme.cmake` 不可读（`Telegram/CMakeLists.txt:41` include 的目标文件不存在） | 见 §5 |
| **A8** | 构建选项默认值（`dev_mode` 等）及其副作用 | `cmake/telegram_options.cmake`（`CMakeLists.txt:31`）不可读 | 外部机 `grep -n "option(" cmake/telegram_options.cmake` |
| **A9** | 自定义 CMake 函数语义：`init_non_host_target`(`:8`)、`nice_target_sources`(`:259`)、`target_prepare_qrc`(`:2590`)、`generate_midl`(`:54`) | 定义在缺失 cmake | 外部机逐个 `grep -rn "function(名称" cmake/` |
| **A10** | 15 个 `add_subdirectory` 子项目的导出目标名与头文件安装路径 | 子目录全空；`CMakeLists.txt:75-80` 只给出**消费**侧别名（`desktop-app::lib_base` 等） | 外部机构建后 `cmake --build out --target help \| grep desktop-app` |

### 3.2 库符号假设（lib_base / lib_ui / lib_tl / lib_rpl / lib_crl / GSL）

| ID | 断言与出处 | 盲区成因 | 外部核对点 |
|---|---|---|---|
| **B1** | `base::flags<ChatDataFlag>` 的 `&`/`~`/`\|` 运算与 `is_flag_type` 协作（`data_chat.h:29-30`；`data_session.cpp:935` `chat->setFlags((chat->flags() & ~flagsMask) \| flagsSet);`） | `base::` 在空 `Telegram/lib_base/` | 外部机读 `Telegram/lib_base/telegram/base/flags.h` 确认 `(1 << 31)` 不被裁剪 |
| **B2** | **`rpl::type_erased` 语义**：`Data::PeerFlagValue(...) \| rpl::type_erased`（`history_inner_widget.cpp:657-662`；`info_media_provider.cpp:110-115`） | `rpl::` 在空 `Telegram/lib_rpl/`。P0 改动的"连带效应 UI 不再显示"完全依赖该类型语义 | 外部机读 `Telegram/lib_rpl/rpl/type_erased.h` |
| **B3** | `crl::async` 立即执行 vs 排队（`apiwrap.cpp:3836`、`:3844`）影响分流时序 | `crl::` 在空 `Telegram/lib_crl/` | 外部机读 `Telegram/lib_crl/crl/async.h` |
| **B4** | `crl::time` 语义（`session_private.cpp:2255`、`:422-424`） | 同上 | 外部机读 `crl/time.h` |
| **B5** | `Ui::MakeInformBox` / `Ui::LayerOption::CloseOther` / `Ui::PopupMenu`（`apiwrap.cpp:579-580`；`history_view_context_menu.cpp:2658`） | `Ui::` 在空 `Telegram/lib_ui/` | 外部机读 `Telegram/lib_ui/ui/boxes/confirm_box.h` 等 |
| **B6** | `not_null<>` / `Fn<>` / `FnMut<>` / `gsl::finally` / `ranges::all_of`（`apiwrap.cpp:530`、`session_private.cpp:2256`、`history_view_context_menu.cpp:434`） | `not_null`/`Fn` 在 lib_base；`gsl` 在空 `Telegram/ThirdParty/GSL`；`ranges` 在空 `Telegram/ThirdParty/range-v3` | 外部机确认 `gsl::finally` 来自 GSL 子模块且版本含 `finally` |
| **B7** | `MTP::Response` / `mtpBuffer` / `SerializedRequest` 布局（`mtproto_response.h:71-75`；ADR-0003:11 引用 `mtproto_serialized_request.h:30-41` 声称"布局 salt/session/msg_id/seq_no/len/body"） | `MTP::` / `tl::` 在空 `Telegram/lib_tl/`。**该文件在本仓可见与否需外部确认** | 外部机读 `Telegram/SourceFiles/mtproto/mtproto_serialized_request.h:30-41` 逐字段核对 |
| **B8** | `details::SerializedRequest::Serialize`（`mtp_instance.h:164`）是稳定的重放插入点 | lib_tl 不可见 | 外部机确认该符号签名与是否 header-only（影响能否只改一 TU） |
| **B9** | `Ui::Text::*` / `TextWithEntities` / `Api::EntitiesFromMTP`（`history_item.cpp:559-565`；`AGENTS.md` 的 `tr::` projector 规则） | `tr::` 生成物 + lib_ui | 外部机确认 `Api::EntitiesFromMTP` 在仓内（`Telegram/SourceFiles/api/api_entities.h`）且对 `vmessage` 无服务端依赖 |
| **B10** | `lib_storage` 相关的任何断言 | **本轮 6 份文档中未找到对 `Storage::` 符号的直接引用**（见 §7）→ 无假设可列 | 若后续 P1/P2 引入存储，直接落在盲区，需先核对 |

### 3.3 构建环境假设

| ID | 断言与出处 | 盲区成因 | 外部核对点 |
|---|---|---|---|
| **C-1** | 外部机可用 `cmake --build out --config Debug --target Telegram`（`AGENTS.md:48-54`；`p0-build-fix-plan.md:63`） | 本 checkout **无 `out/`**、无 cmake/ninja/clang/ccache/docker（`CONTEXT.md:73`）；是否 pre-configure 未知 | 外部机 `ls out/CMakeCache.txt` 并确认 `CMAKE_BUILD_TYPE=Debug` |
| **C-2** | 外部机子模块 commit 与本 checkout 对齐（`CONTEXT.md:123`；`p0-build-fix-plan.md:62`；ADR-0004:16） | 本 checkout 37 个子模块全空，**无 commit 记录可比对**（禁 git） | 外部机 `git submodule status` 与主仓 `.gitmodules` 逐条对齐；记录 6 个必需库的 commit SHA |
| **C-3** | `../Libraries` / `../win64/Libraries` / `../ThirdParty` 布局成立（`AGENTS.md` 的 Build System Structure） | 本 checkout 三者均不存在（`CONTEXT.md:75`） | 外部机 `ls ../Libraries ../win64/Libraries ../ThirdParty` |
| **C-4** | Docker 路径可用（`Telegram/build/docker/centos_env/build_debug.sh`） | `run.sh:7-14` 强制 `$HOME/Telegram/DesktopPrivate`；`:25` 挂载写死 `$HOME/Telegram/tdesktop` | 外部机确认 checkout 路径恰为 `$HOME/Telegram/tdesktop`，或改用原生 cmake 命令 |
| **C-5** | 「应用增量完全依赖持久化 out/ 树，ccache 不服务应用层」（`CONTEXT.md:84`） | C27 只证明 `CCACHE_DISABLE=true`；Dockerfile:169-185 的 BuildKit cache mount 段本轮**未逐行核验** | 外部机 `sed -n '160,190p' Dockerfile` 复核 |

---

## 4. 协议层假设（**全部需实测**；本仓在原理上无法证实）

> 关键前提（【确定】）：TL schema **不描述错误码**。`api.tl` 只声明方法签名与返回类型，服务端错误串（如 `CHAT_FORWARDS_RESTRICTED`）不出现在任何 `.tl` 文件里；`CHAT_ADMIN_REQUIRED` 同理（`api/api_editing.h:29` 是客户端自己维护的常量表）。因此**任何"服务端会怎样拒绝"的断言都不可能靠静态审计证实**。

| ID | 假设 | 仓内间接证据（强度评级） | 为何仍需实测 |
|---|---|---|---|
| **P1** | **服务端会以 `CHAT_FORWARDS_RESTRICTED` 拒绝在 noforwards 群里直接 `forwardMessages`** —— `CONTEXT.md:17`、`:44`、`:56`、`:131`；ADR-0002:15 | **弱（间接）**：`apiwrap.cpp:583` `} else if (show && error == u"CHAT_FORWARDS_RESTRICTED"_q) {` 是**客户端错误串→toast 映射**，`apiwrap.cpp:536-540` 的 `sendMessageFail(const QString &error, ...)` 是**通用**失败处理。全仓 `grep CHAT_FORWARDS_RESTRICTED` 仅 1 处代码命中 + 6 处文档命中，**没有任何代码把它与 `messages.forwardMessages` 关联**（C12）。且 `:583` 带 `show &&` 前置条件，`show = ShowForPeer(peer)`（`:541`），peer 无法定位时该分支**根本不执行** | 需在受保护群用**未改动的上游行为**实测：直接调用 `messages.forwardMessages` 观察返回错误串。另需确认：不同 peer 类型（broadcast/群/超级群）返回的错误串是否都是这一种 |
| **P2** | **「三道门全开 + 仍走 forwardMessages」必然被服务端拒** → 因此必须分流到 AyuForward（ADR-0002:22「禁止对这类条目直接发 forwardMessages」；`p0-build-fix-plan.md:51`） | **弱**：完全依赖 P1。若 P1 为假（例如服务端只对**部分**操作拒绝、或错误码不同、或频道与群不同），则"必须重传"的约束不成立 | 与 P1 同一次实测可一并覆盖。**这是整个 P0 改动的商业逻辑根基，优先级最高** |
| **P3** | **编辑他人消息被服务端以 `CHAT_ADMIN_REQUIRED` 拒绝、不可绕**（`CONTEXT.md:56`；ADR-0003:14、:29） | **中**：`api/api_editing.h:27-30` 客户端常量表含该串，说明客户端**预期**会收到它（C21）。但常量表是"我可能会遇到"，不是"服务端确实这样返回" | 实测：以非本人身份编辑他人消息，记录原始错误串 |
| **P4** | **`api.tl:124 message:string` 必填 ⟹ `messageMediaUnsupported` 消息的原文客户端可得**（`CONTEXT.md:49`；ADR-0002:17；`p0-build-fix-plan.md:12` 引用 `:124`） | **中（schema 层确定，行为层未知）**：C6 证明字段必填；但**服务端在 `messageMediaUnsupported` 构造里是否真的填了 `message`**（而非留空串），属外部行为 | 实测：收到一条 `messageMediaUnsupported`（旧客户端可见的"请更新版本"占位）时抓 `message` 字段是否非空 |
| **P5** | **P0 改动后 `allowsForwarding()` 恒 `true` → 三门全开**（`CONTEXT.md:131`；`p0-build-fix-plan.md:50`） | **强（代码层确定）**：C9 证明判定只看 `Flag::NoForwards`，C1+C7 证明改后该位不再被置位。**运行时 UI 效果**属行为层 | 外部机跑起来看：受保护群里转发菜单项是否出现、复制是否可用、下载是否可用 |
| **P6** | **改动后受限提示 UI 不再显示属预期**（`CONTEXT.md:131`；`p0-build-fix-plan.md:54`） | **强**：`info_media_provider.cpp:110-115` 与 `history_inner_widget.cpp:657-662` 都读 `ChatDataFlag::NoForwards` / `ChannelDataFlag::NoForwards`，与 C9 同源 | 同 P5，实测确认**没有**误伤其它提示路径（这两处之外是否还有第三处读该 flag，本仓未穷举 → 见 §7） |
| **P7** | **重传后的"伪装新消息"不可被服务端反制**（ADR-0002:27） | **无仓内证据**（纯外部行为断言） | 属产品风险，不阻塞编译；建议外部机至少验证一次真实转发链路 |
| **P8** | **发送类 RPC 重放会触发 FLOOD_WAIT / 风控**（ADR-0003:14、:34） | **无仓内证据**（服务端行为） | 阶段性验证（ADR-0004:16 的回填流程） |
| **P9** | **`forwardMessages` 失败一律进 `sendMessageFail`**（回答 ADR-0002:28 开放项；`CONTEXT.md:62`） | **强（代码层确定）**：C13 —— `apiwrap.cpp:4026-4037` 是 `ApiWrap::forwardMessages`（`:3828` 起）内部 `sendPreparedMessage` 的统一 fail lambda，**不限错误类型**。**注意**：是否出 toast 仍取决于 `show` 非空（C12 的 `show &&`）与 else-if 链能否走到 `:583` | 静态已可定论；仅需实测 toast 文案是否如预期 |

---

## 5. codegen 触发面未知项（**待外部核对**）

> 根因：`cmake/td_scheme.cmake` 等 17 个 cmake 模块**均不在盘**（§1.3）。下表的"治理模块"一列是**【确定】**的（`Telegram/CMakeLists.txt` 在盘），"触发行为"一列全部是**【假设】**。

| # | 改动对象 | 治理模块（确定） | 锚点 | 触发什么重生成 / 重编（**假设，需外部核对**） | 外部核对方法 |
|---|---|---|---|---|---|
| G1 | `Telegram/SourceFiles/mtproto/scheme/api.tl` | `cmake/td_scheme.cmake` + `cmake/td_mtproto.cmake` | `Telegram/CMakeLists.txt:41`、`:40`；目标 `tdesktop::td_scheme`（`:70`）、`tdesktop::td_mtproto`（`:69`） | 假设：scheme 层整体重生成（`scheme.h`/`.cpp`），触发**几乎全部**引用 `MTPD*/MTPC*/MTP::` 的 TU 重编（本仓 1,367 个 .cpp，量级≈全量） | `sed -n '1,200p' cmake/td_scheme.cmake`；`ninja -t targets \| grep -i scheme`；改一行 api.tl 后 `ninja -n \| wc -l` 看 dry-run 目标数 |
| G2 | `Telegram/SourceFiles/mtproto/scheme/mtproto.tl` | `cmake/td_mtproto.cmake` | `CMakeLists.txt:40`；目标 `tdesktop::td_mtproto`（`:69`） | 假设：仅 mtproto 层重生成，重编面限于 MTP 加密/连接相关 TU | 同上（`td_mtproto.cmake`） |
| G3 | `Telegram/Resources/langs/lang.strings` | `cmake/td_lang.cmake` | `CMakeLists.txt:39`；目标 `tdesktop::td_lang`（`:68`）；源文件同时被列为 target source（`CMakeLists.txt:2288`） | 假设：生成 `tr::` 符号 + 运行时 lang pack；重编面=所有 `tr::` 调用点。**另注意**：`CMakeLists.txt:2288-2289` 把 `langs/lang.strings` 直接列入 `nice_target_sources` → 改它可能同时改变 IDE/target 依赖边 | `sed -n '1,200p' cmake/td_lang.cmake`；`ninja -t query tdesktop::td_lang` |
| G4 | `cloud_lang.strings` | `cmake/td_lang.cmake` | `CMakeLists.txt:2289` | 假设：仅云翻译资源，不影响 `tr::` 符号 | 同 G3 |
| G5 | 任意 `Telegram/SourceFiles/**/*.style`（71 个） | 定义在缺失 cmake（`${style_files}` 只在 `CMakeLists.txt:261` 被使用，C32） | `CMakeLists.txt:261` | 假设：style codegen 生成 `styles/*.h`，**全量重生成**；重编面=所有 `st::` 调用点。**已知陷阱**：`test-loop.md:507-511` 明写 codegen **不跟踪资源 mtime** | 外部机执行 `touch <某 .style>` 后 `ninja -n \| wc -l`；并 grep `${style_files}` 定义处 |
| G6 | `Telegram/Resources/**/*.svg`（294 个） | `target_prepare_qrc`（`CMakeLists.txt:2590`，定义在缺失 cmake）+ `.qrc` 列表（`CMakeLists.txt:2269-2289`） | `Telegram/Resources/qrc/telegram/telegram.qrc:14,23-31`；`qrc/ayu/ayu.qrc:3-27` | 假设：`.svg` 由 style codegen 消费并重新打包，**只改 svg 不改 .style 时增量构建不会重打**（`test-loop.md:507-511` 明文）。另假设存在 svg→png 栅格化步骤（`CMakeLists.txt:2580-2600` 区域需外部核） | 改一个 svg → `ninja -n` 看是否 0 目标；再 `touch` 引用它的 `.style` → 观察是否重打 |
| G7 | 任意 `*.png` / `*.webp` / `*.jpg` 资源 | 同 G6（qrc） | `CMakeLists.txt:2271-2289` | 假设：qrc 重打包走 `rcc`，重编面小，但 `rcc` 产物变更会触发链接 | 改一个 png → `ninja -n \| wc -l` |
| G8 | 新增 `.style` 文件（改文件集而非内容） | 缺失 cmake（glob/list 逻辑） | `CMakeLists.txt:261` | **假设且风险最高**：若 cmake 侧用显式列表或 configure-time glob，则**新增文件可能需要重新 configure**（`cmake -S . -B out`），否则新 style 静默丢失。外部机需确认是 glob（自动）还是静态列表（需重配） | `grep -rn "GLOB\|style_files" cmake/td_ui.cmake cmake/*.cmake`；新增一个测试 style 后不重配直接 build，看是否报缺符号 |
| G9 | `Telegram/Resources/numbers.txt` | `cmake/generate_models.cmake` | `CMakeLists.txt:2290`、`:2294` | 假设：生成 emoji 搜索模型；重编面小 | 外部机 `sed -n '1,80p' cmake/generate_models.cmake` |
| G10 | `qrhi_shaders.cmake` 消费的资源 | `cmake/qrhi_shaders.cmake` | `CMakeLists.txt:2293` | 假设：着色器预处理 | 外部机 `sed -n '1,80p' cmake/qrhi_shaders.cmake` |
| G11 | 平台专用：`generate_midl.cmake`(WIN32) / `generate_dbus.cmake`(all-other) | `CMakeLists.txt:52-58`、`:2431` | 同左 | 假设：仅对应平台生效 | 外部机按目标平台核对 |
| **共性未知** | **所有 G1–G11 都无法回答"改一个文件 ⇒ 全量重编还是增量"** | 17 个 cmake 模块全缺 | `CMakeLists.txt:31-46,49,53,2293-2294` | —— | 外部机对每类做一次 **touch + `ninja -n` dry-run**（不真编），记录目标数 → 这是成本模型（`CONTEXT.md:108-117`）唯一缺失的实测数据 |

---

## 6. 本轮发现的文档引用错误（非盲区；可在本 checkout 直接修正）

| # | 位置 | 文档写法 | 实测 | 影响 |
|---|---|---|---|---|
| **E1** | `docs/p0-build-fix-plan.md:12` | `channel#d49f34c6`（`:103`） | **`:104`**（`:103` 是 `chatForbidden#6592a1a7`） | 低（行号偏移 1） |
| **E2** | `docs/adr/0003-editable-message-replay.md:11`、`:23` | 「`MTP::Sender::send`（`sender.h:303-314`）」 | **全仓不存在 `Sender::send` 符号**（C22）。`sender.h:303` 是嵌套类 `SpecificRequestBuilder::send()`（类定义 `:210`）。真实出站上界是 `Instance::send`（`mtp_instance.h:151-170`），由 `sender.h:304` 调用。（`CONTEXT.md:53` 引用的 `overrideId` `sender.h:229-232` 与 `resend()` `session_private.cpp:2255` 均**正确**，无需改动） | **高**——ADR-0003 §决策 2 的"拦截点"若按 `Sender::send` 实施会找不到符号；应改为 `Instance::send` / `sendSerialized`（`mtp_instance.h:162`） |
| **E3** | `CONTEXT.md:82` | `run.sh:18-20` CPU/内存 | `:20-21`（CPU）、`:22-23`（内存） | 低 |
| **E4** | `CONTEXT.md:82`、`:124` | `run.sh:22` 挂载 | **`:25`** | 低 |
| **E5** | `CONTEXT.md:82` | `run.sh:9-16` DesktopPrivate 守卫 | `:7-14` | 低 |
| **E6** | `CONTEXT.md:112` | 「内存 `total-4G≈5.3G`」 | **错误**：`run.sh:22-23` 是 `MemTotalGb > 12 ? MemTotalGb - 4 : MemTotalGb`；9.3G 机器**不扣 4G**，实际限 ≈9.3G（C30） | **中**——影响 `CONTEXT.md:115`「内存风险」与 `B` 方案可行性判断 |
| **E7** | `CONTEXT.md:81` | `build_debug.sh:8-11` | 文件共 10 行，调用在 `:9` | 低 |
| **E8** | `docs/adr/0001-ag-checkpoint.md:18` | `ayu_infra.cpp:22-60` DB 加载点 | 文件共 80 行；`initDatabase()` 在 `:47-49`，`AyuDatabase::initialize()` 在 `:48`（C35） | 低 |
| **E9** | `docs/adr/0003:13`（`Response.reply`）与 `:25`（`sender.h:43-74`） | `:43-74` | `MakeDoneHandler` 实际 `:40-75`（C24） | 低 |
| **E10** | `docs/p0-build-fix-plan.md:57` 「未取证」 | 「`edit_peer_info_box` 保存路径是否发送真实 `noforwards`（只读了读方向 `:1081`）」 | **已在仓内可回答：是**。`boxes/peers/edit_peer_info_box.cpp:2914-2925` `saveForwards()` 发送 `MTPmessages_ToggleNoForwards`（`api.tl:2606`）（C26） | **正向**——该开放项可关闭；注意路径为 `boxes/peers/`，非 `boxes/` |

---

## 7. 未找到项（命令 + 范围；无编造）

| 目标 | 命令 | 范围 | 结果 |
|---|---|---|---|
| `Sender::send` 符号 | `grep -rn "Sender::send" .` | 全仓（含 `.md`） | **未找到**（仅命中 `docs/adr/0003-editable-message-replay.md:11`、`:23` 两处文档自述） |
| `ayuNoforwards` 的声明 / 定义 / TL 字段 | `grep -rn "ayuNoforwards" --include=*.tl --include=*.h --include=*.cpp .` | 全仓源码 + schema | **未找到**（仅 2 处调用点：`data_session.cpp:934`、`:1098`） |
| `CHAT_FORWARDS_RESTRICTED` 与 `forwardMessages` 的**代码级关联** | `grep -rn "CHAT_FORWARDS_RESTRICTED" .` | 全仓 | **未找到**（仅 `apiwrap.cpp:583` 一处客户端字符串比较 + 6 处文档） |
| `${style_files}` 的**定义** | `grep -n "style_files" Telegram/CMakeLists.txt` | `Telegram/CMakeLists.txt` 全文 | **未找到**（仅 `:261` 一处使用） |
| `Telegram/Resources/styles/` | `ls Telegram/Resources/styles/` | 该路径 | **未找到**（目录不存在；71 个 `.style` 实际在 `Telegram/SourceFiles/**`） |
| `Telegram/SourceFiles/styles/`（codegen 产物目录） | `ls Telegram/SourceFiles/styles/` | 该路径 | **未找到**（目录不存在） |
| `out/` 构建树 | 本轮未 `ls`（CONTEXT.md:76 已记录）；`.gitmodules` 遍历显示无相关子模块 | 仓库根 | **未找到**（依赖 CONTEXT.md:76 的既有结论，本轮未复验） |
| 文档中对 `Storage::`（lib_storage）符号的引用 | 通读 6 份文档 + §1.1 表格 | CONTEXT.md / 4 份 ADR / p0 计划 | **未找到**——无假设可列（B10） |
| `Sender::send` 之外的其他重放插入点命名 | `grep -n "sendSerialized" Telegram/SourceFiles/mtproto/mtp_instance.h` | 该文件 | 找到 `mtp_instance.h:162`（C23），作为替代锚点 |
| Dockerfile BuildKit cache mount 段 | `sed -n '160,190p' Dockerfile` | 该区间 | **本轮未执行**（C-5 仍为假设） |

---

## 8. 外部构建机核对清单（按优先级；一次构建会话验完）

> 约定：`REPO` = 外部机上的 checkout 根。所有"回退"均指**回退到本报告所依据的仓内现状**。

### P0 — 阻塞一切验证的前置

| # | 在哪里核对（文件 / 命令） | 期望结果 | 若不符要回退什么 |
|---|---|---|---|
| **V1** | `REPO/.gitmodules`：`grep -c 'submodule' .gitmodules`；再 `git submodule status \| grep -c '^-'` | 前者 = 37；后者 = **0**（无未初始化子模块） | 若有 `-`：执行 `git submodule update --init --recursive`；记录 6 个必需库（`cmake`、`Telegram/lib_base`、`lib_ui`、`lib_tl`、`lib_storage`、`codegen`）的 commit SHA 并回填——**在此之前所有后续结论均不可信** |
| **V2** | `git submodule status` 逐条对齐主仓记录 | 6 个必需库 commit 与本 checkout 的 `.git` 记录一致 | 不一致 → 全部验证失真，先对齐再重跑（`CONTEXT.md:116` 已列为一致性风险） |
| **V3** | `ls REPO/cmake/td_scheme.cmake REPO/cmake/td_lang.cmake REPO/cmake/telegram_options.cmake REPO/Telegram/codegen/` | 4 个文件/目录均存在 | 缺失 → 子模块未真正初始化，回到 V1 |
| **V4** | `ls out/CMakeCache.txt && grep -E "CMAKE_BUILD_TYPE|CMAKE_GENERATOR" out/CMakeCache.txt` | 存在且 `CMAKE_BUILD_TYPE=Debug` | 无 `out/` → 先 `cmake -S . -B out`（Debug）；`CONTEXT.md:76` 已记录本 checkout 无 `out/` |
| **V5** | `ls ../Libraries ../win64/Libraries ../ThirdParty` | 存在（`AGENTS.md` 布局） | 缺失 → 按 `docs/building-*.md` 准备依赖 |

### P1 — 验证 P0 编译修复（本次唯一待落地的代码改动）

| # | 在哪里核对 | 期望结果 | 若不符要回退什么 |
|---|---|---|---|
| **V6** | `grep -n "ayuNoforwards" REPO/Telegram/lib_tl/ REPO/Telegram/codegen/ out/ 2>/dev/null`；`grep -rn "is_ayuNoforwards" out/ 2>/dev/null` | **全部无命中** → 证实 A1（该访问器确不存在） | 若 `out/` 里**有** `is_ayuNoforwards` → codegen 侧存在本仓不可见的 patch 源（如 fork 的 TL 补丁或本地生成层）→ **P0 改动单整体作废**，回到 ADR-0002:16 重新定夺 |
| **V7** | `cmake --build out --config Debug --target Telegram`（或 `Telegram/build/docker/centos_env/build_debug.sh`） | 应用 P0 两行改动后：**`data_session.cpp` 两处错误消失**，target 链接通过 | 仍报 `is_ayuNoforwards` → codegen 命名规则与假设 A1 不符，回到 V6 |
| **V8** | 同一命令（**未改任何代码**的干净 HEAD） | 首次编译错误**恰为** `data_session.cpp:934` 与 `:1098` 两处 `is_ayuNoforwards` | 若首个错误在别处 → `p0-build-fix-plan.md:59`「是否还有第二处编译断点」的答案是**有**；回填首个错误块后重排 P0 优先级 |

### P2 — 协议行为实测（决定 P0 改动的商业逻辑是否成立）

| # | 在哪里核对 | 期望结果 | 若不符要回退什么 |
|---|---|---|---|
| **V9** | 受保护（noforwards）群里，用**改动前**的构建直接触发一次 `messages.forwardMessages`（右键转发一条消息） | 服务端返回 `CHAT_FORWARDS_RESTRICTED`；客户端弹 `lng_error_noforwards_group/channel`（`apiwrap.cpp:583-588`） | 返回别的错误串或成功 → **P1/P2 假设被证伪**；`ADR-0002:22`「禁止直接发 forwardMessages」与 `p0-build-fix-plan.md:51` 的理由需重写；**AyuForward 分流仍是产品选择，但不再是"必须"** |
| **V10** | 同上，分别在 **broadcast 频道** 与 **超级群** 各试一次 | 三种 peer 的错误串/toast 与 `:584-588` 的三元一致 | 不一致 → `apiwrap.cpp:583-588` 的 toast 映射需补分支 |
| **V11** | 改动后的构建，在受保护群里：① 转发菜单项是否出现 ② 复制是否可用 ③ 下载是否可用 ④ 是否仍走 AyuForward 下载-重传 | 四项全部成立（对应 C11 的 `apiwrap.cpp:3834-3840` 分流 + C9 的 `allowsForwarding`） | 任一不成立 → 回到 `data_chat.cpp:75-77` / `data_channel.cpp:787-789` 找第二处 gate；本仓**未穷举**所有 `Flag::NoForwards` 读取点（见 §7 未找到项） |
| **V12** | 改动后受限提示是否消失（`info_media_provider.cpp:110-115`、`history_inner_widget.cpp:657-662`） | 不再显示（`p0-build-fix-plan.md:54` 的预期） | 仍显示 → 还有第三处读该 flag 的路径，需 grep `Flag::NoForwards` 补全 |
| **V13** | 收到一条 `messageMediaUnsupported` 消息，抓 `api.tl:124` 的 `message` 字段 | **非空**且有意义（证实 P4） | 为空 → 需求 6 方案（`ADR-0002:23`）的"展示 vmessage 原文"分支**不可行**，只剩 `rich_message` 摘要路径 |
| **V14** | 以非本人身份编辑他人消息 | 返回 `CHAT_ADMIN_REQUIRED`（证实 P3；对照 `api/api_editing.h:29`） | 返回别的错误码 → 更新 ADR-0003:14 的"不可绕"依据文案 |

### P3 — codegen 触发面量化（`CONTEXT.md:88-90` 自认未取证；成本模型唯一缺口）

> 方法：对每类文件 `touch` 后跑 `ninja -n`（**dry-run，不真编**），记录目标数 `n`；改动文件后跑 `ninja -n | tail -5` 看首行命令是否为 codegen 步骤。

| # | 在哪里核对 | 期望结果 | 若不符要回退什么 |
|---|---|---|---|
| **V15** | `sed -n '1,200p' REPO/cmake/td_scheme.cmake`；`ninja -t targets all \| grep -i scheme` | 能读出 api.tl → 生成物的依赖边 | 读不到 → 子模块 commit 不含该文件，回 V2 |
| **V16** | `touch REPO/Telegram/SourceFiles/mtproto/scheme/api.tl && ninja -C out -n \| wc -l` | 记录目标数 `n_api`（假设：接近全量） | 得到数字即可，用于替换 `CONTEXT.md:108-117` 表中"首次构建/单次迭代"的估算 |
| **V17** | `touch REPO/Telegram/SourceFiles/mtproto/scheme/mtproto.tl && ninja -C out -n \| wc -l` | 记录 `n_mtp`（假设：远小于 `n_api`） | 同上 |
| **V18** | `touch REPO/Telegram/Resources/langs/lang.strings && ninja -C out -n \| wc -l` | 记录 `n_lang`（假设：= 所有 `tr::` 调用点） | 同上 |
| **V19** | `touch REPO/Telegram/SourceFiles/history/view/history_view_context_menu.style && ninja -C out -n \| wc -l` | 记录 `n_style`（假设：= 所有 `st::` 调用点） | 同上 |
| **V20** | 只改一个 `.svg`（`touch REPO/Telegram/Resources/art/premium/star_texture.svg`）后 `ninja -C out -n \| wc -l` | **0**（验证 `test-loop.md:507-511` 的"codegen 不跟踪资源 mtime"） | 若非 0 → 该告警对本 fork 不成立，`test-loop.md:507-511` 需更新；两种结果都写回报告 |
| **V21** | 新建一个含非法字段名的测试 `.style`，`ninja -C out -n`（**不重新 configure**） | 若 cmake 用 glob → 直接可见；**若报缺符号 → 说明需重新 configure**（G8 风险坐实） | 结论写回 G8，并更新 `p0-build-fix-plan.md:64`「重编面仅 `data/data_session.cpp` 及其依赖 TU」的类推适用范围 |
| **V22** | `grep -rn "style_files" REPO/cmake/` | 找到 `${style_files}` 定义处 | 未找到 → 记为无法确定，回到 §7 |

### P4 — MTProto 重放可行性（ADR-0003 二期的前置条件）

| # | 在哪里核对 | 期望结果 | 若不符要回退什么 |
|---|---|---|---|
| **V23** | `sed -n '1,80p' REPO/Telegram/SourceFiles/mtproto/mtproto_serialized_request.h`（对照 ADR-0003:11 声称的 `:30-41`） | 字段顺序确为 salt/session/msg_id/seq_no/len/body | 布局不同 → ADR-0003:11 描述作废；`sender.h:43-74` 改写为正确的重放实现说明 |
| **V24** | `grep -n "sendSerialized" REPO/Telegram/SourceFiles/mtproto/mtp_instance.cpp REPO/Telegram/SourceFiles/mtproto/mtp_instance.h` | 确认 `mtp_instance.h:162` 是唯一收口点 | 存在多处 → 拦截点需重新选；**注意 ADR-0003:23 写的 `Sender::send` 符号不存在（E2）** |
| **V25** | 读 `REPO/Telegram/lib_tl/` 中 `SerializedRequest::Serialize` 的声明 | 确认为 header-only / 可在不重编全树的前提下扩展 | 非 header-only → 二期需要改 lib_tl 子模块（引入 fork patch 成本） |

### P5 — 库语义核对（B2/B3/B4 决定 P0 的"连带效应"预测）

| # | 在哪里核对 | 期望结果 | 若不符要回退什么 |
|---|---|---|---|
| **V26** | `sed -n '1,80p' REPO/Telegram/lib_rpl/rpl/type_erased.h`；对照 `history_inner_widget.cpp:662` / `info_media_provider.cpp:115` 的 `\| rpl::type_erased` | 语义为"类型擦除的 flag producer" | 语义不同 → `CONTEXT.md:131` / `p0-build-fix-plan.md:54` 的"提示不再显示"预测作废 |
| **V27** | `sed -n '1,60p' REPO/Telegram/lib_crl/crl/async.h`；对照 `apiwrap.cpp:3836` | `crl::async` 为排队派发 | 同步执行 → `ApiWrap::forwardMessages` 的 `return` 时序需重审 |
| **V28** | `grep -n "finally" REPO/Telegram/ThirdParty/GSL/include/gsl/gsl` | GSL 提供 `gsl::finally`（`session_private.cpp:2256` 依赖） | 无 → 需换实现，`ADR-0003:27` 的 resend 复用路径受影响 |

### P6 — 文档引用错误修正（本仓可做，列出供一次性批量修）

| # | 在哪里核对 | 期望 | 若不符 |
|---|---|---|---|
| **V29** | 对照 §6 E1–E10 逐条 | 本报告 C 列即为正确值 | 若外部机上的文件行号与本报告不同，说明 checkout 版本已漂移 → **全部结论需重跑** |

---

## 9. 假设统计

| 类别 | 条数 | 其中需实测（协议） |
|---|---|---|
| A. 编译期 / codegen 假设 | 10（A1–A10） | —— |
| B. 库符号假设 | 10（B1–B10） | —— |
| C. 构建环境假设 | 5（C-1–C-5） | —— |
| D. 协议层假设 | 9（P1–P9） | 全部 |
| G. codegen 触发面未知项 | 11（G1–G11 + 共性未知） | —— |
| **合计** | **45** | **9** |

【确定】项 36 条（C1–C36）。文档引用错误 10 条（E1–E10），其中 1 条（E10）为**正向修正**（关闭一个开放项）。未找到项 10 条（§7）。

## 主线程抽查（2025-09-27）
- ✅ 采信盲区实测：`Telegram/SourceFiles/styles` 不存在；`Telegram/CMakeLists.txt` 的 `include(cmake/*.cmake)` 全部落空（`cmake` 子模块空）。
- ✅ 采信勘误 `run.sh` 内存规则：实测 `DockerMemoryGb=$((MemTotalGb > 12 ? MemTotalGb - 4 : MemTotalGb))` —— 9.3G 机器**不扣** 4G，CONTEXT §8.2/§8.5 的「total-4G≈5.3G」表述有误（已在 CONTEXT §10 记勘误）。
- ✅ 采信 E2：`MTP::Sender::send` 全仓不存在（`sender.h` 无 `void send(`），ADR-0003 锚点已改为 `Instance::sendSerialized`。
- ⚠️ Top1（协议假设 CHAT_FORWARDS_RESTRICTED）主线程认同：静态不可证实，必须外部实测。
