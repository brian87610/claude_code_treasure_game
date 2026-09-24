---
description: 將目前的 Vite 專案建置並部署到 GitHub Pages（自動處理 gh 登入、建立 Repo、啟用 Pages）
argument-hint: "[repo 名稱（選填，預設為目前資料夾名稱）]"
allowed-tools: Bash, PowerShell, Read, Edit, AskUserQuestion
---

# 部署到 GitHub Pages

目標：把目前專案 build 出來的靜態檔推到 `gh-pages` 分支，並啟用 GitHub Pages，最後回報可以開啟的網址。

使用者指定的 repo 名稱：`$ARGUMENTS`（若為空，使用目前資料夾名稱）。

請**依序**執行以下階段。每個階段開頭先輸出一行「▶ 階段 N：…」讓使用者知道進度；任何階段失敗就停下來說明原因與下一步，不要跳過。

> 執行環境提醒：
> - Windows 上 `gh` 可能只在 PowerShell 的 PATH 中、Git Bash 找不到；若 Bash 顯示 `command not found`，改用 PowerShell 再試一次，或嘗試 `"/c/Program Files/GitHub CLI/gh.exe"`。
> - **Windows Git Bash 會把 `/xxx/` 開頭的參數自動改寫成 `C:/Program Files/Git/xxx/`**（已實測：`--base=/repo/` 會變成 `/Program Files/Git/repo/`，部署後整頁空白）。在 Git Bash 執行含 `/` 開頭參數的指令（`vite build --base=...`、`gh api ...`）時，一律在前面加 `MSYS_NO_PATHCONV=1`；或改用 PowerShell 執行。

---

## 階段 0：前置檢查（工具）

1. 確認 `git`、`node`、`npm` 可用（`git --version`、`node --version`）。缺少就告知使用者安裝方式並停止。
2. 確認 `gh`（GitHub CLI）可用：`gh --version`。
   - 若未安裝：用 AskUserQuestion 詢問是否代為安裝。
     - Windows：`winget install --id GitHub.cli -e --accept-source-agreements --accept-package-agreements`
     - macOS：`brew install gh`
     - Linux：參考 https://github.com/cli/cli#installation
   - Windows 安裝完後，目前 shell 的 PATH 不會自動更新。請改用完整路徑 `"C:\Program Files\GitHub CLI\gh.exe"` 繼續，或請使用者重開 VS Code／終端機後再執行一次 `/deploy_github_page`。

## 階段 1：確認 GitHub 登入狀態（未登入時引導登入）

1. 執行 `gh auth status`。
2. **若顯示未登入**（`You are not logged into any GitHub hosts` 或非 0 結束碼）：
   - `gh auth login` 是互動式指令，Claude 無法代為輸入，請清楚引導使用者：
     > 你目前尚未登入 GitHub。請在下方輸入框輸入以下指令（開頭的 `!` 代表由你親自在終端機執行）：
     >
     > `! gh auth login --hostname github.com --git-protocol https --web`
     >
     > 1. 畫面會顯示一組一次性驗證碼（例如 `ABCD-1234`），先複製起來
     > 2. 按 Enter 會開啟瀏覽器，登入 GitHub 後貼上驗證碼並按 Authorize
     > 3. 完成後回到這裡告訴我「好了」
   - 然後**停止並等待使用者回覆**，不要繼續後面的階段。
   - 使用者回覆後，再次執行 `gh auth status` 確認；仍未登入就重複引導。
3. 登入成功後：
   - 取得帳號：`gh api user --jq .login`，記為 `OWNER`。
   - 確認權杖含 `repo` scope（`gh auth status` 會列出 Token scopes）。若沒有，引導使用者執行 `! gh auth refresh -s repo`。
   - 執行 `gh auth setup-git`，讓 `git push` 使用 gh 的憑證。

## 階段 2：確認 Repo（沒有就建立）

1. 若目前資料夾不是 git repo（`git rev-parse --is-inside-work-tree` 失敗）：
   - 執行 `git init -b main`，並確認 `.gitignore` 包含 `node_modules/` 與 `build/`（沒有就補上）。
   - 若沒有任何 commit，執行 `git add -A && git commit -m "Initial commit"`。
2. 決定 repo 名稱 `REPO`：`$ARGUMENTS` 不為空就用它，否則用目前資料夾名稱。
3. 檢查遠端 `origin`：`git remote get-url origin`。
   - **沒有 origin**：檢查 `gh repo view OWNER/REPO` 是否已存在。
     - 不存在 → 建立新的 public repo（免費帳號的 GitHub Pages 需為 public）：
       `gh repo create OWNER/REPO --public --source=. --remote=origin`
       （先不要加 `--push`；原始碼分支是否推送由使用者決定，Pages 只需要 `gh-pages` 分支。）
     - 已存在 → 用 AskUserQuestion 詢問「使用這個既有 repo」或「換一個名稱建立新的」。使用既有的就 `git remote add origin https://github.com/OWNER/REPO.git`。
   - **已有 origin**：解析出 `ORIGIN_OWNER/ORIGIN_REPO`，執行
     `gh repo view ORIGIN_OWNER/ORIGIN_REPO --json name,visibility,viewerPermission`
     - repo 不存在（已刪除或網址錯誤）→ 詢問是否以 `OWNER/REPO` 建立新 repo，並用 `git remote set-url origin` 指向新 repo。
     - `viewerPermission` 是 `ADMIN`／`MAINTAIN`／`WRITE` → 直接使用，`REPO = ORIGIN_REPO`。
     - 其他（例如是從別人那裡 clone 下來的，沒有 push 權限）→ 用 AskUserQuestion 讓使用者選擇：
       - （建議）在自己帳號建立新 repo：把原本的 origin 改名保留 `git remote rename origin upstream`，再 `gh repo create OWNER/REPO --public --source=. --remote=origin`
       - Fork 原 repo：`gh repo fork --remote --remote-name origin`（原本的會變成 `upstream`）
     - 若 visibility 為 `PRIVATE`，提醒免費帳號的 private repo 不能用 Pages，詢問是否改成 public（`gh repo edit --visibility public --accept-visibility-change-consequences`）。
4. 輸出確認：「將部署到 `https://github.com/OWNER/REPO`」。

## 階段 3：建置與部署

1. 若沒有 `node_modules/`，執行 `npm install`。
2. 建置，並把 base path 設成 repo 名稱（GitHub Pages 的專案網址是 `https://OWNER.github.io/REPO/`，Vite 預設 base `/` 會讓 JS/CSS 404）：
   `npx vite build --base=/REPO/`
   - 若 REPO 名稱為 `OWNER.github.io`（使用者主頁），改用 `--base=/`。
   - 建置輸出目錄以 `vite.config.ts` 的 `build.outDir` 為準（本專案是 `build/`，不是 `dist/`），記為 `OUT_DIR`。
   - 不要修改 `vite.config.ts`，base 只透過命令列參數傳入，避免影響本機 `npm run dev`。
   - 建置後檢查 `OUT_DIR/index.html` 裡的 `<script src="...">` 是否以 `/REPO/assets/` 開頭；若出現 `Program Files` 等字樣，代表路徑被 shell 改寫了，請參考上方提醒後重新建置。
3. 建立 `OUT_DIR/.nojekyll`（空檔），避免 GitHub 的 Jekyll 忽略底線開頭的檔案。
4. 推送到 `gh-pages` 分支：
   `npx --yes gh-pages -d OUT_DIR -b gh-pages --dotfiles -m "Deploy to GitHub Pages"`
   - 若失敗訊息為 `Author identity unknown`（git 沒設定 user.name/email）：不要改使用者的全域 git 設定，改用 GitHub noreply 信箱只作為這次部署 commit 的作者，加上參數
     `--user "OWNER <ID+OWNER@users.noreply.github.com>"`（`ID` 由 `gh api user --jq .id` 取得），並先刪除 `node_modules/.cache/gh-pages` 再重試。
   - 若失敗訊息為權限（403）或驗證問題，回到階段 1 重新檢查；若為 `gh-pages` 快取問題，刪除 `node_modules/.cache/gh-pages` 後重試一次。
5. 啟用 / 設定 Pages 來源為 `gh-pages` 分支根目錄：
   - 先查：`gh api repos/OWNER/REPO/pages`
   - 注意：推送 `gh-pages` 分支後，GitHub 通常會**自動啟用** Pages；此時 POST 會回 `409 already enabled`，屬正常情況，直接檢查 `source` 是否正確即可。
   - 404（尚未啟用）→ `gh api -X POST repos/OWNER/REPO/pages -f "source[branch]=gh-pages" -f "source[path]=/"`
   - 已啟用但來源不是 `gh-pages` → `gh api -X PUT repos/OWNER/REPO/pages -f "source[branch]=gh-pages" -f "source[path]=/"`
   - 取得網址：`gh api repos/OWNER/REPO/pages --jq .html_url`，記為 `PAGE_URL`。

## 階段 4：驗證部署結果

1. 輪詢最新一次 Pages build 狀態（每 10 秒一次，最多約 5 分鐘）：
   `gh api repos/OWNER/REPO/pages/builds/latest --jq .status`
   - `built` → 成功，進行下一步
   - `errored` → 顯示 `gh api repos/OWNER/REPO/pages/builds/latest --jq .error.message` 並停止
   - 使用 Monitor 或 until 迴圈等待，不要用前景 `sleep` 反覆呼叫。
2. 用 `curl -s -o /dev/null -w "%{http_code}" PAGE_URL` 確認首頁回 `200`；
   再從 `curl -s PAGE_URL` 的 HTML 抓出一個 `/REPO/assets/*.js` 路徑，確認該資源也回 `200`（確保 base path 正確）。
   - 剛啟用 Pages 時 CDN 可能需要 1–2 分鐘，404 時可再等一下重試。

## 最後回報

用簡短清單回報：
- GitHub 帳號、Repo 網址（新建立或既有）
- 部署分支 `gh-pages` 與 commit
- **網站網址 `PAGE_URL`**
- 驗證結果（首頁與資源的 HTTP 狀態碼）
- 若有變更 remote（例如把 origin 改為 upstream），特別說明
