# GitHub Actions 自動發布（Windows 10 amd64）

> 工作流：`.github/workflows/release-win-amd64.yml`
> 觸發：`push` tag `v*`，或 `release: published`
> 產物：`AyuGram_<version>_win64_portable.zip`（**單可執行文件，Qt 靜態鏈接，解壓即用**）+ `sha256sums.txt`
> 只構建 Windows x64，其他平台不編。

## 1. 怎麼發版

**方式 A：打 tag（自動建 Release 並上傳）**
```bash
git tag v7.2.9-ayu.1
git push origin v7.2.9-ayu.1
```
→ Actions 自動：構建依賴 → Release 構建 → 打包 zip → 算 sha256 → **自動建立並發布 Release**。

> ⚠️ 不要用 `v7.0.9` / `v7.2.9` 這類純上游 tag：它們已存在於 origin（是合併進 dev 的上游 tag），`git tag` 會直接報 `already exists`。本 fork 的發版一律用 `-ayu.<n>` 後綴。

**方式 B：手工建立 Release**
GitHub → Releases → Draft a new release → 填 tag（新 tag 或已有 tag）→ Publish release。
→ 觸發 `release: published` → 構建並把資產掛到該 Release。

> 去重：以方式 A 自動建立的 Release，正文裡帶 `<!-- auto-built -->` 標記；由此產生的 `release: published` 事件會被 job 條件自動跳過，**不會重複構建**。

## 2. 版本號規則

| tag 形式 | 內嵌版本（`Telegram/build/version`） | 資產名 |
|---|---|---|
| `v7.2.9-ayu.1` | 提取前綴 `7.2.9`（由 `set_version.py` 寫入） | `AyuGram_7.2.9-ayu.1_win64_portable.zip` |
| `v7.2.9` | `7.2.9` | `AyuGram_7.2.9_win64_portable.zip`（**上游 tag 已存在，別用**） |
| 非數字 tag | 保持倉庫默認 `7.2.9`，不報錯 | 用 tag 原文命名 |

## 3. 打包內容（為什麼「解壓即用」）

`Telegram/build/prepare` 在 Windows 上構建的是 **Qt 靜態庫**（`prepare.py` 的 Qt configure 帶 `-static` / `-static-runtime`，`prepare.py:1554-1555`、`:1656-1657`），因此 `out/Release` 產出的是**單個 exe**，不依賴外部 Qt DLL。

zip 內：

- `AyuGram.exe`（Release，Win10 amd64 可直接運行）
- `LICENSE`、`LEGAL`

`sha256sums.txt` 格式：`<sha256>  <文件名>`（可用 `sha256sum -c` 或在 Windows 用 `certutil -hashfile` 校驗）。

**自動更新已焊死**：configure 帶 `-D DESKTOP_APP_DISABLE_AUTOUPDATE=ON`（`telegram_options.cmake:39-40` → `TDESKTOP_DISABLE_AUTOUPDATE`），產出的 exe **不會**從伺服器下發的 `autoupdate_url_prefix` 拉更新包。這是自動更新鏈（唯一的 RCE 面）在構建層面的硬開關，升級靠重新發版，不靠自更新。副作用：設置裡「檢查更新」相關 UI 一併隱藏。

## 4. 首次/後續耗時

| 階段 | 首次 | 命中緩存 |
|---|---|---|
| prepare-libs（Qt 5.15.19 + OpenSSL/FFmpeg 等，Debug+Release 全量） | 3~5 h | 2~5 min |
| configure + Release build | 1~2 h | 20~40 min |
| **總計** | **~4~7 h** | **~30~50 min** |

緩存鍵綁定 `.gitmodules` + `prepare.py` + `qt_version.py` + `Telegram/CMakeLists.txt`，這些文件不變就一直命中。

## 5. 出問題時看哪裡

| 症狀 | 位置 |
|---|---|
| prepare 階段失敗 | Actions 產物 `prepare-log` |
| configure/編譯失敗 | Actions 產物 `build-logs`（`configure.log`/`build.log`） |
| 想一次拿全部編譯錯誤 | Actions 產物 `build-logs/errors.txt`（已用 `error C####`/`LNK####` 聚合，並寫入 job summary） |
| 登錄失敗（API 憑據不匹配） | 改 workflow 頂部 `TDESKTOP_API_ID/HASH` 為本 fork 實際憑據（當前用 tdesktop 公開測試憑據 `2040`） |
| Release 沒有資產 | 看 build job 日誌的 `Publish release` 步驟；權限需 `contents: write`（已聲明） |
| `[2/34](ThirdParty/msys64): FAILED` 且 `error: target not found: mingw-w64-x86_64-diffutils` | MSYS2 已下架 mingw64 版的 diffutils。已改為安裝 `msys/diffutils` 並用 `where diff` 斷言（`prepare.py:479-486`） |

## 6. 已知風險

1. **首次 prepare 是最脆弱的環節**：下載 msys2/NuGet + 從源碼編 Qt/OpenSSL/FFmpeg，GitHub 4vCPU runner 上可能踩路徑/下載問題 → 失敗留 `prepare.log`，按報錯逐項修，不必重來全套。
   - 快取在**失敗時也會保存**（`actions/cache/restore` + `save` + `if: always()`），且每個階段開頭 `rmdir` 自清，所以重跑會重用已完成階段，只重做失敗那階之後。
   - 階段的「已完成」標記是 `<ThirdParty 或 Libraries/win64>/cache_keys/<stage>`，它在快取目錄內，因此跨 run 能真正命中（不是每次都 `NOT FOUND`）。
2. 倉內文檔寫 VS2026 + v14.44 工具集；`windows-2022` 是 VS2022（v143）。對 Win10 amd64 足夠。
3. Qt 版本：倉庫 `qt_version.py` 對 Win x64 默認 Qt **5.15.19**（Win arm/qt6 才走 6.11.2）。若實際發佈用 Qt6，把 yaml 的 `QT` 改成對應版本並給 prepare 傳 `qt6`。
4. 緩存上限 10 GB/倉庫；`Libraries/win64` + `ThirdParty` 若超限，緩存會被跳過（不影響本次構建，只影響下次提速）。
5. **緩存路徑必須是絕對路徑、不能含 `..`**：`actions/cache` 會拒絕 `${{ github.workspace }}/../X`（報 `Relative pathing '.' and '..' is not allowed`），而且只是 warning，容易誤判成「prepare 失敗」。workflow 用 `${{ runner.workspace }}/X`（= `D:\a\<repo>`，正是 `prepare.py` 的 `rootDir`）。

## 7. 發版前回歸清單（人工，編譯器抓不到）

| # | 項 | 期望 |
|---|---|---|
| 1 | 雙擊右鍵（500ms） | 保存檢查點；單擊右鍵仍出菜單 |
| 2 | 右鍵 Save AG Checkpoint | 寫入 `ayudata.db` |
| 3 | 進群默認定位 | 有檢查點→定位；無→原生；搜索中→不覆蓋 |
| 4 | Jump to AG Checkpoint | 論壇話題帶 topic；普通群走原路徑 |
| 5 | 受保護群轉發 | 走 AyuForward 重傳，不報 CHAT_FORWARDS_RESTRICTED |
| 6 | 受保護群導出聊天記錄 | 不應出現入口 |
| 7 | 群設置「限制轉發」回顯 | 與伺服器一致 |
| 8 | 需求 6：unsupported 消息 | 顯示原文/摘要，保留提示且不可編輯 |
| 9 | 設置頁開關 | 可切換、持久化到 `ayu_settings.json` |
| 10 | 清空歷史/刪聊天/註銷 | Checkpoint 行被清（頻道刪聊天是已知 R1 缺口） |
| 11 | 多帳號 | 檢查點互不串號 |
| 12 | 原「重複消息」/原生「編輯」 | 不受影響 |
| 13 | 一欄/兩欄布局 | 菜單與跳轉都正常 |
| 14 | **需求 7「Edit & Resend」** | 右鍵消息 → 原文插入輸入框（不清空已有草稿），可直接編輯發送 |
| 15 | 界面文案 | 顯示英文而非 `ayu_*` key |
