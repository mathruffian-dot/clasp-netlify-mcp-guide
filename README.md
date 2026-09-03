# Clasp 雲端部署 + Netlify MCP 零複製雙向閉環網頁系統：Agent 安裝步驟與避坑指南

本文件為 **AI Coding Agent（如 Antigravity, Claude Code, Cline 等）** 的標準安裝與避坑部署指南。當接收到建立「網頁前端 + Google Sheets 資料庫（經由 GAS Web App API）+ Netlify 部署」之閉環系統需求時，請嚴格遵循以下步驟與安全注意事項，範例程式碼已抽離，聚焦於環境架構之自動化整合。

---

## 一、 先備條件與安全注意事項

1.  **環境依賴**：本機必須已安裝 `Node.js`、`npm`、`git`。
2.  **帳號準備**：使用者須擁有 Google 帳號與 Netlify 帳號。
3.  **敏感資料保護（極重要）**：
    *   **嚴禁**將個人的 `Netlify Personal Access Token`、`GITHUB_TOKEN`、任何 OAuth 金鑰或 Google 帳號資訊寫入程式碼或提交至公開 Repo。
    *   所有 API 網址、部署 ID 與專案金鑰，必須在執行期間動態生成、填入或透過環境變數傳遞，不得留有任何預設或硬編碼硬傷。

---

## 二、 本地檔案準備規則

在進行雲端部署前，Agent 必須先在本地工作區建立以下無敏感資料的基礎結構：

1.  **後端 GAS 主程式**：一個包含 `doPost(e)` 與 `doGet(e)` 的後端 JavaScript 檔案（例如 `gas_code.js`），負責接收前端 API 請求並對接 Google Sheets。
2.  **GAS 配置文件**：一個名稱必須為 **`appsscript.json`** 的 GAS Manifest 檔案，其內必須配置 `webapp` 欄位以允許匿名存取（將執行身分設為 `USER_DEPLOYING`，存取權設為 `ANYONE_ANONYMOUS`）。
3.  **Clasp 過濾清單（避坑關鍵）**：在專案根目錄建立 **`.claspignore`**，並明確設定規則以排除所有前端網頁檔案（如 `index.html`、`style.css`、`app.js`），**僅允許**推送 `gas_code.js`（或後端 gs 檔）與 `appsscript.json`。
4.  **前端網頁檔案**：建立遊戲/系統之前端網頁資料夾，內含 HTML、CSS 與 JS 連線程式。連線 JS 必須留有 API 網址的預設預留空字串（如 `const GAS_API_URL = "YOUR_GAS_API_URL_HERE";`），以便後續動態注入。

---

## 三、 完整自動化安裝與部署步驟

Agent 應在本地依序執行以下指令，完成雲端與前端的雙向閉環部署：

1.  **安裝 Clasp 引擎**：
    *   在專案資料夾本地執行：`npm install @google/clasp`
2.  **完成 Google 雲端授權**：
    *   啟動登入：`npx clasp login`
    *   *Agent 行動*：從 terminal 輸出中抓取 `🔑 Authorize clasp by visiting this url: ...` 的 Google OAuth 授權連結，將連結完整呈現給使用者，引導其點擊並完成瀏覽器授權。
3.  **創建 GAS 雲端專案**：
    *   在本地資料夾執行：`npx clasp create-script --title "專案資料庫" --type standalone`
4.  **強制推送與發佈 Web App**：
    *   執行強推：`npx clasp push -f`
    *   執行部署：`npx clasp create-deployment --description "Production Web App"`
5.  **動態獲取與注入 API 網址**：
    *   執行 `npx clasp list-deployments` 列出部署。
    *   *Agent 行動*：**不要自己拼網址**（`/macros/s/<id>/exec` 一定是錯的）。執行 `npx clasp open-web-app <deploymentId> --json`，取 API 回傳的 `webApp.url` 給使用者。
6.  **Netlify 全自動網頁發佈**：
    *   *Agent 行動*：
        1.  呼叫 `netlify-project-services-updater` 工具（`create-new-project` 參數）在使用者帳號下建立新網站，獲取 `site_id`。
        2.  呼叫 `netlify-deploy-services-updater` 工具（`deploy-site` 參數，填入 `siteId` 與 `deployDirectory`），將本地的前端網頁資料夾一鍵上傳發佈。
        3.  將最終生成的 Netlify 公共網址輸出給使用者。

---

## 四、 五大經典踩坑與注意事項

### 注意事項 1：Google 帳號未啟用 Apps Script API
*   **狀況**：執行 `clasp create-script` 時失敗並提示 `User has not enabled the Apps Script API...`。
*   **應變對策**：引導使用者手動點開 [Google Apps Script 使用者設定頁面](https://script.google.com/home/usersettings)，將 **Google Apps Script API** 切換為 **「開啟 (ON)」**，切換完成後再重試。

### 注意事項 2：前端檔案被推送到 GAS 導致伺服器崩潰
*   **狀況**：Apps Script 執行紀錄中跳出紅字：`ReferenceError: document is not defined (@ app.gs:5)`。
*   **原因**：前端 JS 檔案在 `.claspignore` 生效前被推送到雲端，因 GAS 伺服器無瀏覽器 DOM 環境而編譯失敗。
*   **應變對策**：
    1.  確認本地已建立正確的 `.claspignore`。
    2.  對本地 `gas_code.js` 進行微小修改（如加入一行註解），以觸發 Clasp 的本地檔案變更偵測。
    3.  再次執行：`npx clasp push -f`。此時 Clasp 會呼叫 API，以本地未忽略的檔案清單**完全覆寫並替換**雲端專案，遠端的舊前端檔案將被**自動刪除**。
    4.  重新執行 `npx clasp create-deployment` 發佈乾淨的新版本。

### 注意事項 3：腳本權限未經首次執行驗證 (Authorization Required)
*   **狀況**：前端 fetch 或 curl 請求 Web App 網址時，Google 回傳權限錯誤或顯示「很抱歉，目前無法開啟這個檔案」。
*   **原因**：因為腳本中使用了讀寫雲端硬碟的 Google 服務，該專案在第一次發佈後，**擁有者必須手動在瀏覽器編輯器中執行授權一次**。
*   **應變對策**：引導使用者手動點開專案的線上編輯器，在上方選單選擇任意函數（如 `getSheet`），點擊 **「執行」**，並完成 **「審閱權限」->「選擇帳號」->「進階」->「前往專案（不安全）」->「允許」** 的 Google 授權確認。

### 注意事項 4：Netlify MCP 部署提示 `state.json` 不存在
*   **狀況**：呼叫 `deploy-site` 部署網站時，MCP 拋出錯誤：`ENOENT: no such file or directory, open '...state.json'`。
*   **原因**：未指定有效 `siteId` 或未先建立 Site 專案。
*   **應變對策**：不可直接執行部署。必須先透過 `create-new-project` 專案更新工具建立專案，取得系統配發的 `site_id` 後，再將該 `siteId` 與本地資料夾路徑傳遞給 `deploy-site` 工具進行發佈。

### 注意事項 5：Google 瀏覽器多重帳號登入 Session 衝突
*   **狀況**：網頁載入時偶爾會跳出「連線至 Google Sheets 失敗」的跨網域或權限錯誤。
*   **原因**：瀏覽器同時登入多個 Google 帳號會導致 Apps Script Web App 的 Session 混亂。
*   **應變對策**：在網頁前端錯誤提示中，或在回覆中，**強烈建議使用者使用「無痕視窗（Incognito）」** 或乾淨的瀏覽器環境開啟 Netlify 網址進行實測，即可 100% 避開多帳號衝突。

> 📌 本文件指令已對齊 **clasp v3**（`create-script`／`create-deployment`／`list-deployments`／`show-authorized-user`／`open-web-app`）。完整對照與六條紅線見 [clasp-gas-skill](https://github.com/mathruffian-dot/clasp-gas-skill)。
