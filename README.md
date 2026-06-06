# 📂 Antigravity 2.0 對話故障自動修復與恢復工具箱 (Database Recovery & Auto-Fixer Toolset)

> 💡 **專案簡介**：本工具箱為解決 **Antigravity 2.0** 桌面端（基於 Electron + SQLite + Protobuf）的兩大核心對話毀損故障而設計：
> 1. **「對話無限 Loading」**：因二進位 Protobuf 資料損壞導致前端反序列化失敗。
> 2. **「HTTP 400 Bad Request」**：因 Extended Thinking 模式崩潰殘留 `<thinking>` 區塊，違反 API Prefill 規範。
> 
> 🔒 **代碼安全**：本專案採用安全展示設計，原始碼（*.py, *.ps1）已在本地排除，僅於本頁面展示技術架構、核心邏輯與自動化守護原理。

---

## ⚡ 核心故障成因分析 (The Problem)

### 故障 A：對話無限 Loading (Protobuf 反序列化崩潰)
* **成因**：寫入對話軌跡至 `conversations/*.db` 時，若進行了非等長二進位替換（例如：將長度不同的字串直接寫入 Protobuf length-delimited 欄位），會導致 **Protobuf 長度標記與實際二進位長度不符**。
* **後果**：Electron 主程序在載入該對話並進行反序列化（Deserialization）時將直接崩潰，導致前端對話介面永久卡在 Loading 圖示。

### 故障 B：HTTP 400 Bad Request (思維區塊殘留 / prefill 衝突)
* **成因**：在開啟 **Extended Thinking（延伸思考）** 模式下，若回答在中途斷線、崩潰，或者 AI 產生了 `thinking` 區塊卻沒有後續的正常 Markdown 文字（通常是後端 API 逾時或 Electron 崩潰造成的），該對話的 SQLite steps 表會留下一個「結尾只有 thinking，沒有正常內容」的壞 step。
* **後果**：當你再次發送訊息時，由於 Claude 模型不允許 Assistant 訊息以 `thinking` 區塊結尾（*The final block in an assistant message cannot be `thinking`*），API 會直接拒絕並回傳 `HTTP 400`，使該會話永久性損壞。

---

## 🔍 典型真實故障案例與解析 (Real-World Case Studies)

以下為本專案在開發與維護過程中，實際攔截並記錄的三類經典故障日誌與技術解析：

### 📋 案例一：Extended Thinking 未閉合思維區塊故障
> **軌跡識別 (Trajectory ID)**: `[已隱藏]`
* **日誌特徵**：
  ```json
  {
    "error": {
      "code": 400,
      "message": "{\"type\":\"error\",\"error\":{\"type\":\"invalid_request_error\",\"message\":\"messages.137: The final block in an assistant message cannot be `thinking`.\"}}",
      "status": "INVALID_ARGUMENT"
    }
  }
  ```
* **深入解析**：當使用者在 IDE 開啟了 **Extended Thinking（延伸思考）**，模型輸出中途斷線（例如：伺服器重啟或 TCP 套接字重置）。由於沒有後續的常規文字欄位，這段中斷歷史被記錄在 SQLite 內，使 `<thinking>` 成為對話的最終狀態。後續對話發送時，API 校驗不符合規範，引發 400 錯誤。

---

### 📋 案例二：不支援 Assistant Prefill 規則限制
> **軌跡識別 (Trajectory ID)**: `[已隱藏]`
* **日誌特徵**：
  ```json
  {
    "error": {
      "code": 400,
      "message": "{\"type\":\"error\",\"error\":{\"type\":\"invalid_request_error\",\"message\":\"This model does not support assistant message prefill. The conversation must end with a user message.\"}}",
      "status": "INVALID_ARGUMENT"
    }
  }
  ```
* **深入解析**：某些 Claude 與進階生成模型在 API 級別**強制限制對話歷史結構**。對話必須由 User 訊息（使用者輸入）起頭並結尾。若因為 IDE 異常重啟或進程當機，導致對話未能寫入使用者的下一輪發言，而保持在「AI 說話結束」的狀態，API 會將其判定為無效的 Prefill 請求並回絕。

---

### 📋 案例三：網路傳輸層 TCP WSArecv 連線強制關閉 (WSArecv Connection Reset)
> **軌跡識別 (Trajectory ID)**: `[已隱藏]`
* **日誌特徵**：
  ```text
  Error: agent executor error: model unreachable: stream reading error: read tcp [本機IP]:[端口]->[伺服器IP]:443: wsarecv: An existing connection was forcibly closed by the remote host.
  Wraps: (3) forced error mark | "model api cannot be reached"
  Wraps: (12) An existing connection was forcibly closed by the remote host. (syscall.Errno)
  ```
* **這個問題是什麼？**
  * **底層 Windows Socket 斷線 (WSArecv 10054)**：`WSArecv` 是 Windows 套接字（Winsock）API 中用於接收資料的底層系統呼叫。當傳回 `An existing connection was forcibly closed by the remote host` 時，代表本地端與 Google Cloud Vertex AI / 模型 API 伺服器之間的 TCP 連線被**強行發送 RST（重置）封包中斷**。
  * **誘發根因**：通常是由於本地網路代理（VPN）連線抖動、本機防火牆策略攔截、寬頻 IP 漂移，或者 Google 伺服器端因超時主動踢掉連線。

* **🔗 它與「HTTP 400 Bad Request」的因果關係是什麼？**
  * **「WSArecv 斷線」是「因」，「HTTP 400 報錯」是「果」**。
  * 當底層 TCP 連線因 `WSArecv` 崩潰斷開時，AI 的回答被無情卡在半空中。此時，IDE 程式碼在對話 SQLite 資料庫（`conversations/*.db` 的 `steps` 表）寫入中斷，導致最後一條 Assistant 訊息只殘留了 `<thinking>` 區塊，卻沒有後續正常的 markdown 內容（或缺失下一個 User 發言回合）。
  * 在下次你再度發送訊息時，IDE 會將這段損壞的「未完結/以 thinking 結尾」的歷史紀錄發給 Google/Claude API，進而觸發 API 的防禦規則，拋出 `HTTP 400 Bad Request`，導致整條對話被永久卡死。

* **🛠️ 本專案的 PowerShell (PS) 與 Python (Py) 守護進程是怎麼自動修復的？**
  我們無法阻止實體層的網路斷線（這是硬體與伺服器端決定的），但我們建立了一套 **「全自動災後重置與熱還原」** 守護系統，在斷線引發 400 錯誤的當下，秒級完成修復：

  1. **常駐守護與拉起機制 (PowerShell - `start-sync.ps1`)**：
     * **PID 隔離與清理**：透過將進程的 PID 精確寫入本地 `%USERPROFILE%/.gemini/antigravity/conversations/auto_fix_pid.txt` 檔案，確保每次重啟同步服務時，能 100% 殺死舊的背景進程，解決端口（Port 18998/18999）被佔用死鎖的問題。
     * **無感背景執行**：使用 `-WindowStyle Minimized` / `Hidden` 拉起守護進程，確保不殘留任何黑色 CMD 視窗，完全不干擾使用者遊戲與開發。

  2. **毫秒級日誌與資料庫監聽 (Python - `auto_fix_antigravity_400.py`)**：
     * **熱日誌 Polling 監聽**：每 2 秒輪詢一次 `AppData\Local\Programs\Antigravity\logs\` 目錄中最近 5 秒內被修改的 `.log` 檔案。
     * **特徵正則掃描 (Regex Detection)**：利用正則表達式掃描日誌中是否包含 `wsarecv`、`thinking`、`prefill` 等關鍵特徵。若日誌尚未寫入，還會主動用 SQLite API 讀取當前最活躍資料庫 `steps` 表，解析最新的一筆 step，校驗 `status` 是否為 Error (status=7)。
     * **強制進程解除鎖定 (Unlock File Lock)**：一旦發現災情，自動執行 `taskkill /f /im Antigravity.exe /im node.exe /im antigravity_tools.exe`，強行終止所有正在佔用並鎖定對話資料庫的進程。
     * **物理物理抹除與熱重啟 (Purge & Relaunch)**：
       * 從日誌中提取受災的對話 UUID。
       * 物理刪除該損壞的 `.db`、以及暫存的 `.db-wal`（預寫日誌）、`.db-shm`（共享記憶體快取）。
       * 自動重新啟動 `Antigravity.exe`。
       * 調用 `tools\notify.ps1` 通過 Windows 系統 Toast 通知機制發送：「**✅ 400 錯誤已自動修復，並已重新啟動 IDE！**」，讓使用者全程無感、直接回歸開發！

---

## 🛠️ 技術修復與守護架構 (Architecture)

本專案提供**被動還原（Recovery）**與**主動常駐守護（Daemon Auto-Fixer）**雙重安全防禦機制：

```mermaid
flowchart TD
    subgraph 監聽階段 (Live Monitoring)
        A[背景 Python 守護進程] -->|即時監聽 5s 內更新| B[AppData Log 日誌檔]
        A -->|主動輪詢| C[當前活躍 SQLite DB steps 表]
    end

    subgraph 偵測與判定 (Detection)
        B -->|正則匹配| D{是否存在 400/thinking/prefill 錯誤?}
        C -->|校驗 error_details| D
    end

    subgraph 熱修復流程 (Hot-Fix Pipeline)
        D -- 是 --> E[Windows Toast 彈出修復通知]
        E --> F[強制強殺鎖定進程 Antigravity.exe & node.exe]
        F --> G[從日誌中提取損壞對話 UUID]
        G --> H[物理清理該壞 .db, -wal, -shm 快取檔案]
        H --> I[重新拉起 Antigravity 桌面端]
        I --> J[發送修復完成 Toast 通知]
    end

    D -- 否 --> K[繼續監聽]
```

### 📋 核心修復機制詳解

1. **🔓 強制解除進程鎖定 (File Lock Release)**
   * 由於 Electron 主程式與 Node.exe 背景進程會持續佔用並鎖定對話資料庫檔案，修復前程式會自動執行進程終止，安全釋放檔案鎖（File Lock），避免 `Permission Denied` 錯誤。
2. **🛡️ 活躍資料庫校驗 (Active SQLite Validation)**
   * 守護進程會查詢最新寫入的 steps 資料列，解析其二進位內容，校驗 `status` 是否為 Error (status=7) 並檢查 `error_details` 是否含有 Prefill 或 Thinking 關鍵字，實現日誌與 DB 雙重交叉檢驗。
3. **🧹 快取與暫存檔清除 (SQLite Cache Purge)**
   * SQLite 資料庫在運行時會產生 `.db-wal`（預寫日誌）與 `.db-shm`（共享記憶體快取）檔案。若不清除這些檔案，重啟時依然可能因為讀取舊快取而發生衝突。
4. **✅ 結構完整性校驗 (Integrity Validation)**
   * 還原備份資料庫（針對故障 A）後，自動呼叫 `PRAGMA integrity_check` 進行 SQL 級別的健康檢查，確保修復後的資料庫結構完美無瑕。

---

## 🚀 守護進程啟動與自動化流程

專案透過精確的 PID 檔案管理與背景無感啟動設計，確保在非 Administrator 權限下也能 100% 順暢運行：

* **自適應啟動**：透過 Windows 排程任務或雙擊 `啟動同步服務.bat`，自動以 `WindowStyle Minimized` / `Hidden` 開啟守護進程，防範 2.0 桌面端殘留進程懸浮條。
* **PID 精確清理**：將背景常駐進程的 PID 固化寫入 `%USERPROFILE%/.gemini/antigravity/conversations/` 下，每次啟動前精確殺死舊進程，絕不引發端口（Port 18998/18999）衝突與記憶體洩漏。

---

## 📄 版權與使用聲明 (Copyright & License)

> [!IMPORTANT]
> **版權所有，保留一切權利 (Copyright &copy; All Rights Reserved)**
>
> *   本專案為**非開源作品 (No License)**，僅作為個人開發作品集審閱與技術演示之用。
> *   未經原作者書面授權，**嚴禁任何形式的下載、複製、修改、散佈或用於商業用途**。
