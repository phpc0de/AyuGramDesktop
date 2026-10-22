# ADR-0004 — 構建與迭代策略：外部構建機驗證 + 靜態交叉核驗

- 狀態: Accepted（2025-09-25 用戶定案）
- 關聯條目: CONTEXT.md §8（需求 8）、§8.5 成本模型

## Context
- 用戶新增需求 8「如何快速迭代，編譯問題」；並要求先算清磁盤/時間等各項成本再定。
- 本 checkout 實測**結構性不可構建**：37 個子模塊全部未初始化（含 `cmake`、`Telegram/lib_base`、`Telegram/lib_ui`、`Telegram/lib_tl`、`Telegram/lib_storage`、`Telegram/codegen`；`git ls-files -s cmake` → `160000`，`.gitmodules:41-42`）；`../Libraries`、`../ThirdParty`、`../win64/Libraries` 全缺；cmake/ninja/clang/ccache/docker 全缺；無 `out/` 樹；磁盤 31G 可用（曾跌到 2.8G/98% 滿）、內存 9.3G（可用 4.0G）、5 核。
- Docker 路徑約束：`run.sh:9-16` 強制要求 `$HOME/Telegram/DesktopPrivate`；`:22` 掛 `$HOME/Telegram/tdesktop`（本 checkout 在 `/root/workspace/AyuGramDesktop`，路徑不匹配）；`:18-20` 把 CPU 限到 nproc-2、內存 total-4G；最終鏡像 `CCACHE_DISABLE=true`（`Dockerfile:1069`）→ 應用增量只能靠持久化 `out/` 樹。
- 約定：`AGENTS.md:5`「Avoid building the project」；`:67` 禁 Release；`.agents/shared/build-lock-recovery.md:25` 要求序列化構建但無真正 build-lock。
- 成本對比（明細見 CONTEXT.md §8.5）：B（本機自建）需 +60~80G 磁盤、3~8h 首次構建、並解除禁安裝/禁構建約定；A（外部機）本側成本近零，代價是人工往返；C（純靜態）放棄編譯驗證。

## Decision
1. **構建執行地 = 外部構建機**（用戶側）。本會話不構建、不安裝、不啟動 Docker（`/grill` 亦凍結應用代碼產出）。
2. **默認反饋手段 = codegraph + grep 交叉核驗**：每處改動用 `codegraph callers/callees` 與 grep 驗證調用點/聲明/定義一致（本輪 phantom accessor 即由此抓出：`data_session.cpp:934,1098` 調用 api.tl 中不存在的 `is_ayuNoforwards`）。
3. **外部驗證交付物規範**（每個交付回合附帶）：① 精確改動點 `file:line`；② 需與外部機對齊的子模塊 commit（至少 `cmake`、`Telegram/lib_base`、`Telegram/lib_ui`、`Telegram/lib_tl`、`Telegram/lib_storage`、`Telegram/codegen`）；③ 外部驗證命令 `cmake --build out --config Debug --target Telegram`（AGENTS.md:48-54；WSL/Linux 走 `Telegram/build/docker/centos_env/build_debug.sh`）；④ 風險點與預期影響面；⑤ 外部回填報錯後的迭代路徑。
4. **提交前靜態自檢清單**：新增/改動符號的聲明↔定義匹配；locale key 存在；style 引用存在；include 路徑有效；`.tl`/codegen 變更的連帶影響（fork 特有風險：`data_session.cpp:934,1098` 類「TL 補丁丟失→phantom accessor」）。
5. 「禁構建」默認維持，僅用戶顯式請求時才在本側放開。

## Consequences
- 正向：本側零環境成本、零磁盤風險、嚴格遵守 AGENTS.md:5 與既有只讀約定；審計/設計迭代（秒級反饋）不受編譯阻塞。
- 負向：每輪存在人工往返延遲；P0 類「編不過」只能在外部發現；無法在本側跑 test-loop（`.agents/shared/test-loop.md:141-179` 要求測試帳號 + 無 Telegram 實例運行）；外部機與本 checkout 子模塊 commit 不一致會導致驗證失真。
- 已否決：B（自建，成本與權限代價過高）；C 作為獨立方案（放棄編譯驗證）——其手段已被採納為默認反饋。
- 後續：如需加速，可在外部機開啟 ccache（需改 Dockerfile:1069 的 `CCACHE_DISABLE` 為構建機本地配置，非倉庫默認）或引入真實 out/ 鎖；均屬新 ADR 範疇。
