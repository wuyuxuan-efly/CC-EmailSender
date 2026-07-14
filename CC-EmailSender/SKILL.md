---
name: CC-EmailSender
description: "通用郵件發送器 — 支援 Gmail、Outlook、網易(163/126/yeah/企業/靈犀)、QQ、iCloud、Yahoo 等所有電子郵件系統。每次使用時現場詢問郵箱地址與 SMTP 認證，不依賴固定設定檔，適合多人共用。觸發條件：使用者意圖涉及透過電子郵件發送或傳遞內容時。範例：發郵件、通知某人、分享檔案/報告、發送會議紀要、轉發內容、回覆摘要等。支援：收件人/抄送/暗送、AI 自動生成內文、附件、發送前預覽確認。"
agent_created: true
---

# 通用郵件發送器 (Universal Email Sender)

汎用 SMTP 郵件發送工具。不依賴任何設定檔，所有參數在每次使用時現場取得。
支援所有電子郵件提供商（Gmail、Outlook、Office365、網易、QQ、iCloud、Yahoo 等）。

## 前置需求

- Python 3.x + 標準庫（無外部依賴）
- 腳本路徑: `C:\Users\969421\.workbuddy\skills\CC-EmailSender\scripts\send_email.py`

---

## 觸發政策

**不要等待明確的關鍵詞。** 只要使用者的意圖涉及透過電子郵件發送內容就應觸發。

| 使用者說的話（範例） | 判讀 |
|---|---|
| "把這個報告發給王總" | 透過郵件發送報告 |
| "通知團隊明天開會" | 發郵件通知團隊 |
| "把這份記錄郵件給 zhangsan@example.com" | 發送記錄 |
| "幫我回覆那封郵件，說收到" | 撰寫並發送回覆郵件 |
| "把這個檔案分享給小張" | 透過郵件發送檔案 |
| "給客戶發一下報價單" | 發送報價單 |

若意圖不明確（可能是郵件也可能是即時訊息），**先詢問**使用者偏好哪種管道。

---

## 完整流程（6 步）

> ## ⚠️ 介面規範：所有使用者輸入一律使用 `AskUserQuestion` 工具
>
> **禁止**輸出純文字表格讓使用者在對話框手打填寫（舊版做法，體驗差且易出錯）。
> **禁止**調用 `show_widget` / `read_me` / `present_files` 渲染卡片或表單。
>
> **正確做法：使用 `AskUserQuestion` 工具提供互動式輸入框。**
> 每個需要使用者輸入的欄位，都透過 `AskUserQuestion` 的 `question` + `options` 提供。
> 使用者可點選預設選項，也可透過底部的 **Other 自定義輸入框** 輸入任意文字。
>
> **設計原則：**
> - `options` 數量按情況靈活決定，可為 0~4 個。當只需使用者自由輸入文字時，可提供 1 個提示性 option 或不提供 option，使用者透過底部 **Other 自定義輸入框** 輸入。
> - 當存在合理預設值或快捷選項時，提供對應的 options 供使用者點選。
> - 使用者需自由輸入文字時，點選對應 option 後在 **Other** 輸入框中填寫，或直接在 Other 中輸入。
> - 多個獨立欄位可在同一個 `AskUserQuestion` 中同時呈現（最多 4 個 question）。

---

### 第 1 步：寄件人設定（郵箱 → 判斷種類 → SMTP 設定 → 密碼登入驗證）

#### 1a. 取得寄件人郵箱地址

**首先判斷使用者是否已在初始訊息中提供寄件人郵箱地址。**

判斷規則：掃描使用者的觸發訊息，尋找電子郵箱格式（如 `xxx@xxx.com`、`xxx@xxx.com.cn` 等）。

**情況 A — 使用者已提供寄件人郵箱：**

直接使用該郵箱作為寄件人，**無需再次詢問**。立即執行 1b（郵箱種類判斷）。

**情況 B — 使用者未提供寄件人郵箱：**

使用 `AskUserQuestion` 提供一個純輸入介面，讓使用者輸入自定義郵箱地址。**此步驟不提供任何郵箱類型選項**，僅提供一個輸入框讓使用者輸入自定義文字：

```json
{
  "questions": [{
    "question": "請輸入寄件人郵箱地址",
    "header": "寄件人郵箱"
  }]
}
```

> **注意：此步驟不提供郵箱類型選項（如 Gmail、網易、QQ 等），僅提供一個純文字輸入框讓使用者輸入自定義郵箱地址。上方的兩個 option 僅為提示性文字，真正的郵箱地址透過底部 Other 輸入框取得。**

使用者透過底部 **Other 自定義輸入框** 輸入完整郵箱地址後，繼續執行 1b。

#### 1b. 判斷郵箱種類

取得寄件人郵箱地址後，根據郵箱域名判斷郵箱種類。

**四種常用郵箱定義：**

| 種類 | 涵蓋域名 |
|------|----------|
| 網易 | 163.com, 126.com, yeah.net, qiye.163.com, 188.com |
| Outlook | outlook.com, hotmail.com, live.com, office365.com, msn.com |
| QQ | qq.com, foxmail.com, vip.qq.com |
| Gmail | gmail.com, googlemail.com |

**判斷邏輯：** 掃描郵箱地址的域名部分（@ 之後），與上表進行比對。

---

**情況 A — 能判斷郵箱種類：**

告訴使用者判斷結果，並使用 `AskUserQuestion` 提供以下選項供使用者確認。

**選項構建規則：**
- 選項 (1)：判斷正確（[判斷出的種類]）→ 繼續執行
- 若判斷結果是四種常用郵箱之一，選項 (2)(3)(4) = 另外三種常用郵箱
- 若判斷結果不屬於四種常用郵箱（如 Yahoo、iCloud、Zoho 等），選項 (2)(3)(4) = 前三種常用郵箱（網易、Outlook、QQ）
- **Other 輸入區**：若以上選項都不對，使用者在此自行填寫正確的郵箱種類

**範例 — 判斷為 Gmail（屬於四種常用之一，其餘三種為網易、Outlook、QQ）：**

```json
{
  "questions": [{
    "question": "根據您的郵箱地址，判斷為 **Gmail** 郵箱。請確認是否正確？",
    "header": "確認郵箱種類",
    "options": [
      {"label": "判斷正確：Gmail", "description": "確認為 Gmail，繼續執行"},
      {"label": "應為：網易", "description": "實際為網易(163/126/yeah)郵箱"},
      {"label": "應為：Outlook", "description": "實際為 Outlook/Hotmail 郵箱"},
      {"label": "應為：QQ", "description": "實際為 QQ/Foxmail 郵箱"}
    ]
  }]
}
```

**範例 — 判斷為 QQ（屬於四種常用之一，其餘三種為網易、Outlook、Gmail）：**

```json
{
  "questions": [{
    "question": "根據您的郵箱地址，判斷為 **QQ** 郵箱。請確認是否正確？",
    "header": "確認郵箱種類",
    "options": [
      {"label": "判斷正確：QQ", "description": "確認為 QQ/Foxmail，繼續執行"},
      {"label": "應為：網易", "description": "實際為網易(163/126/yeah)郵箱"},
      {"label": "應為：Outlook", "description": "實際為 Outlook/Hotmail 郵箱"},
      {"label": "應為：Gmail", "description": "實際為 Gmail 郵箱"}
    ]
  }]
}
```

**範例 — 判斷為 Yahoo（不屬於四種常用，選項 2~4 為前三種常用郵箱）：**

```json
{
  "questions": [{
    "question": "根據您的郵箱地址，判斷為 **Yahoo** 郵箱。請確認是否正確？",
    "header": "確認郵箱種類",
    "options": [
      {"label": "判斷正確：Yahoo", "description": "確認為 Yahoo，繼續執行"},
      {"label": "應為：網易", "description": "實際為網易(163/126/yeah)郵箱"},
      {"label": "應為：Outlook", "description": "實際為 Outlook/Hotmail 郵箱"},
      {"label": "應為：QQ", "description": "實際為 QQ/Foxmail 郵箱"}
    ]
  }]
}
```

---

**情況 B — 無法判斷郵箱種類：**

使用 `AskUserQuestion` 提供四種常用郵箱供使用者選擇，以及 Other 輸入區：

```json
{
  "questions": [{
    "question": "無法自動判斷您的郵箱種類，請選擇您的郵箱類型：",
    "header": "選擇郵箱種類",
    "options": [
      {"label": "網易", "description": "163/126/yeah/企業郵箱等網易系郵箱"},
      {"label": "Outlook", "description": "Outlook/Hotmail/Live/Office365 郵箱"},
      {"label": "QQ", "description": "QQ/Foxmail 郵箱"},
      {"label": "Gmail", "description": "Google Gmail 郵箱"}
    ]
  }]
}
```

若使用者透過 **Other** 輸入自定義類型，記錄使用者提供的類型名稱。

---

**確認後：** 根據使用者確認或選擇的郵箱種類，執行 SMTP 偵測（見 1c）。

#### 1c. SMTP 伺服器設定（直接確定，不做 TCP 預測試）

確認郵箱種類後，按以下順序嘗試取得 SMTP 伺服器設定。**不做 TCP 連線預測試**，直接將設定傳遞給 1d，由登入驗證（含真實密碼）一併確認連線與認證。

---

**第 (1) 步：查詢預設值**

部分郵箱種類有預設的 SMTP 設定，直接使用預設值：

| 郵箱種類 | 主 SMTP 伺服器 | 備用 SMTP 伺服器 | 連接埠 | 加密方式 |
|----------|---------------|-----------------|--------|---------|
| 網易 | `smtp.qiye.163.com` | `smtpv6hz.qiye.ntes53.netease.com` | 465 | SSL |

> 若使用者確認的郵箱種類在上表中，**使用主伺服器**作為預設 SMTP 設定。
> 若不在上表中（如 Gmail、Outlook、QQ 等），跳到第 (2) 步。

---

**第 (2) 步：從 `--detect-smtp` 與 references/smtp_settings.md 匹配**

```bash
python "C:\Users\969421\.workbuddy\skills\CC-EmailSender\scripts\send_email.py" --detect-smtp "使用者郵箱"
```

同時參考 `references/smtp_settings.md` 中對應郵箱種類的設定，取兩者中更精確的值。

**找到匹配設定** → 直接使用匹配到的 SMTP 伺服器、連接埠、加密方式，進入 1d。

**未找到匹配設定** → 直接進入第 (3) 步。

---

**第 (3) 步：自動檢索網路上的匹配地址**

使用 `WebSearch` 工具，搜尋該郵箱域名對應的 SMTP 伺服器設定。

搜尋關鍵詞範例：
- `"{域名} SMTP server settings port"
- `"{域名} SMTP 伺服器 連接埠 SSL`

從搜尋結果中提取 SMTP 伺服器位址、連接埠、加密方式。

**搜尋到設定** → 直接使用搜尋到的 SMTP 設定，進入 1d。

**搜尋失敗** → 進入第 (4) 步。

---

**第 (4) 步：要求使用者自行填寫**

使用 `AskUserQuestion` 讓使用者提供 SMTP 伺服器資訊：

```json
{
  "questions": [
    {
      "question": "無法自動偵測您的 SMTP 伺服器，請輸入 SMTP 伺服器位址（如 smtp.example.com）",
      "header": "SMTP 伺服器",
      "options": [
      ]
    },
    {
      "question": "請選擇加密方式",
      "header": "加密方式",
      "options": [
        {"label": "SSL（連接埠 465）", "description": "連線時直接加密，推薦使用"},
        {"label": "STARTTLS（連接埠 587）", "description": "先建立普通連線再升級為加密連線"},
        {"label": "不加密（連接埠 25）", "description": "不加密，不推薦，可能被封鎖"}
      ]
    }
  ]
}
```

取得使用者提供的設定後，直接進入 1d（不再做 TCP 預測試）。

---

**最終輸出：** 無論透過哪一步取得設定，將以下變數直接傳遞給 1d：
- `smtp_server` — SMTP 伺服器位址
- `smtp_port` — 連接埠
- `ssl_mode` — 加密方式（ssl / starttls / none）

> **注意：1d 的登入驗證會同時確認連線可達性和認證正確性。若連線失敗，會引導使用者回到 1c 重新選擇 SMTP 設定。**

#### 1d. 取得密碼 / 授權碼並驗證登入

此步驟取得使用者的密碼/授權碼，並**一次性驗證 SMTP 連線與認證**（不做預先 TCP 測試）。

**取得密碼：**

使用 `AskUserQuestion` 提供一個文字輸入框，讓使用者輸入 SMTP 驗證密碼。根據 1c 中取得的郵箱種類資訊，提示對應的認證方式：

```json
{
  "questions": [{
    "question": "請輸入 SMTP 密碼 / 授權碼 / 應用專用密碼",
    "header": "安全驗證",
    "options": [
    ]
  }]
}
```

> 根據郵箱種類，可在 question 中附加提示：
> - 網易/QQ → 「請輸入授權碼（非登入密碼）」
> - Gmail/iCloud/Yahoo → 「請輸入應用專用密碼（非登入密碼）」
> - Outlook/企業郵箱 → 「請輸入登入密碼」

**驗證登入（同時驗證連線與認證）：**

使用者透過 **Other 自定義輸入框** 輸入密碼後，執行連線測試：

```bash
python "C:\Users\969421\.workbuddy\skills\CC-EmailSender\scripts\send_email.py" \
  --test-connection \
  --sender "sender_email" \
  --smtp-server "smtp_server" \
  --smtp-port "smtp_port" \
  --ssl-mode "ssl_mode" \
  --password "使用者提供的密碼"
```

**連線成功** → "驗證成功！SMTP 伺服器連線與認證均通過。" → 進入第 2 步。

**連線失敗（認證錯誤）** → 使用 `AskUserQuestion` 讓使用者選擇下一步：

```json
{
  "questions": [{
    "question": "SMTP 登入驗證失敗，密碼/授權碼可能不正確。請選擇下一步：",
    "header": "登入失敗",
    "options": [
      {"label": "重新輸入密碼", "description": "重新提供正確的密碼/授權碼/應用專用密碼"},
      {"label": "重新輸入郵箱", "description": "回到步驟 1a，重新輸入寄件人郵箱地址"}
    ]
  }]
}
```

- **若選「重新輸入密碼」** → 回到本步驟的「取得密碼」階段，重新提供 `AskUserQuestion` 讓使用者輸入密碼，然後再次驗證。
- **若選「重新輸入郵箱」** → 回到步驟 1a，重新開始寄件人設定流程。

**連線失敗（連線/網路錯誤，如連線逾時、拒絕連線、Connection unexpectedly closed 等）** → 使用 `AskUserQuestion` 讓使用者選擇下一步：

```json
{
  "questions": [{
    "question": "SMTP 伺服器連線失敗（非認證問題），可能為伺服器位址不正確或網路受限。請選擇下一步：",
    "header": "連線失敗",
    "options": [
      {"label": "重新設定 SMTP", "description": "回到步驟 1c，重新選擇或輸入 SMTP 伺服器"},
      {"label": "重新輸入郵箱", "description": "回到步驟 1a，重新開始寄件人設定流程"},
      {"label": "取消發送", "description": "取消本次郵件發送"}
    ]
  }]
}
```

- **若選「重新設定 SMTP」** → 回到步驟 1c，重新確定 SMTP 伺服器設定（可嘗試備用伺服器或使用者自定義）。
- **若選「重新輸入郵箱」** → 回到步驟 1a，重新開始。
- **若選「取消發送」** → 停止流程。

**安全提醒：密碼僅存在於當前對話的臨時變數中，不回顯。**


### 第 2 步：附件（點選方式）
首先判斷用戶是否已經提供了附件，若提供了附件文件則將附件添加入郵件附件，
否則使用 `AskUserQuestion` 讓使用者選擇是否附加檔案：

```json
{
  "questions": [{
    "question": "是否需要附加檔案？",
    "header": "附件",
    "options": [
      {"label": "不需要附件", "description": "純文字郵件，直接進入確認步驟"},
      {"label": "需要附加檔案", "description": "選擇後請在輸入框上傳檔案"}
    ]
  }]
}
```

**若選「不需要附件」：**
- `attachments = []`
- 直接進入第 3 步。

**若選「需要附加檔案」：**
- **停止對話**，告訴使用者：
  > "請在輸入框中上傳您要附加的檔案。上傳完成後，請回覆「完成」，
  > 我會繼續處理您的郵件。"
- 等待使用者上傳檔案並回覆「完成」。
- 收集使用者上傳的所有檔案路徑，存入 `attachments` 列表。
- 確認收到的檔案：
  > "已收到以下附件：
  > - [檔案1名稱]
  > - [檔案2名稱]
  > 將繼續生成郵件。"
- 進入第 3 步。

**如果使用者在觸發 skill 時已附帶了檔案：**
- 不再詢問，直接將已偵測到的檔案列入 `attachments`。
- 告知使用者"已偵測到對話中的檔案：[檔案列表]，將作為附件。"

---

### 第 3 步：郵件撰寫（互動式表單）

**核心設計：使用 `AskUserQuestion` 同時提供多個輸入框，讓使用者一次填寫所有郵件欄位。**

#### 3a. 郵件基本欄位（收件人、抄送、暗送、主旨）

使用 `AskUserQuestion` 一次提供 4 個輸入框，使用者可同時填寫：

```json
{
  "questions": [
    {
      "question": "請輸入收件人郵箱地址（多個用逗號分隔）",
      "header": "收件人",
      "options": [
        {"label": "稍後補充", "description": "暫不填寫收件人，稍後再提供"},
        {"label": "手動輸入", "description": "點擊下方 Other 輸入收件人郵箱地址（多個用逗號分隔）"}
      ]
    },
    {
      "question": "請輸入抄送對象（選填，多個用逗號分隔）",
      "header": "抄送"
    },
    {
      "question": "請輸入暗送對象（選填，多個用逗號分隔）",
      "header": "暗送"
    },
    {
      "question": "請輸入郵件主旨",
      "header": "主旨",
      "options": [
        {"label": "AI 生成", "description": "根據郵件內容由 AI 自動生成主旨"}
      ]
    }
  ]
}
```

#### 3b. 內文撰寫方式

接著使用 `AskUserQuestion` 提供內文撰寫的兩個選項：

```json
{
  "questions": [{
    "question": "郵件內文如何撰寫？",
    "header": "內文",
    "options": [
      {"label": "AI自動生成(根據主旨與附件)", "description": "由 AI 根據主旨與附件內容自動生成專業郵件內文"}
    ]
  }]
}
```

**解析使用者回覆：**

```
to_addrs  = 收件人（逗號/分號分隔）
cc_addrs  = 抄送（可為空）
bcc_addrs = 暗送（可為空）
subject   = 主旨；若選「由 AI 建議」則標記為需 AI 生成
body_text = 內文；若選「AI自動生成(根據主旨與附件)」→ body_ai_generate = True
                              若選「自行填寫」→ 使用 Other 輸入的內容
```

**驗證：**
- 收件人不可為空 → 若為空，提示"收件人為必填欄位，請提供收件人郵箱地址。"
- 主旨不可為空 → 若標記為 AI 生成，則根據郵件上下文生成主旨。

**預填條件**：若使用者在觸發時已提供部分資訊（如收件人郵箱、主旨等），在對應欄位的 options 中加入已填值作為預設選項，並告知「已預填部分欄位，請確認或修改。」

---


### 第 4 步：生成與確認

**4a. 生成完整郵件內容**

若第 2 步中 `body_ai_generate` 為 True（使用者選擇「AI自動生成(根據主旨與附件)」）：
- 此時才生成內文，因為現在已取得附件資訊。
- 根據以下資訊生成專業郵件內文：
  - 收件人（若已知其姓名/職位）
  - 郵件主旨
  - 附件內容（若有附件，可讀取附件內容或根據檔名推斷）
  - 對話上下文（使用者最初的需求）
  - 適當的問候語和結語
- 生成後保存為 `body_text`。

若使用者選擇「自行填寫」並透過 Other 輸入了內文，直接使用該內容。

**4b. 預覽**

執行預覽命令：

```bash
python "C:\Users\969421\.workbuddy\skills\CC-EmailSender\scripts\send_email.py" \
  --preview \
  --sender "sender_email" \
  --smtp-server "smtp_server" \
  --smtp-port "smtp_port" \
  --ssl-mode "ssl_mode" \
  --password "password" \
  --display-name "display_name" \
  --to "to_addrs" \
  --cc "cc_addrs" \
  --bcc "bcc_addrs" \
  --subject "subject" \
  --body "body_text" \
  --attach "file1" "file2"
```

若內文較長，可先寫入臨時檔案再使用 `--body-file`。

向使用者展示預覽結果，並使用 `AskUserQuestion` 請使用者確認：

```json
{
  "questions": [{
    "question": "請確認以下郵件內容是否正確無誤。",
    "header": "確認郵件",
    "options": [
      {"label": "確認並發送", "description": "核對無誤，立即發送郵件"},
      {"label": "修改", "description": "返回修改郵件內容（收件人、主旨、內文等）"},
      {"label": "捨棄", "description": "取消本次郵件發送"}
    ]
  }]
}
```

---

### 第 5 步：發送

**若使用者選「確認並發送」：**

執行發送命令（與預覽命令相同參數，移除 `--preview`）：

```bash
python "C:\Users\969421\.workbuddy\skills\CC-EmailSender\scripts\send_email.py" \
  --sender "sender_email" \
  --smtp-server "smtp_server" \
  --smtp-port "smtp_port" \
  --ssl-mode "ssl_mode" \
  --password "password" \
  --display-name "display_name" \
  --to "to_addrs" \
  --cc "cc_addrs" \
  --bcc "bcc_addrs" \
  --subject "subject" \
  --body "body_text" \
  --attach "file1" "file2"
```

**重要：發送命令的參數必須與預覽命令完全一致，不可修改任何值。**

- **發送成功**（輸出含 `SUCCESS`）：
  > "郵件已成功發送！
  > 收件人：[to_addrs]
  > 主旨：[subject]"

  **發送成功後，立即詢問是否保存寄件人資訊（使用 AskUserQuestion 按鍵）：**

  ```json
  {
    "questions": [{
      "question": "是否要保存本次寄件人資訊，以便下次直接使用（無需重新輸入郵箱與密碼）？",
      "header": "保存寄件人",
      "options": [
        {"label": "要，建立專屬技能", "description": "為您建立一個包含此寄件人資訊的個人專屬郵件技能"},
        {"label": "不要，以後再說", "description": "下次仍需重新輸入郵箱與密碼"}
      ]
    }]
  }
  ```

  - **若選「要」**：按照下方「用戶專屬技能建立流程」執行，創建 `CC-EmailSender-{用戶識別名}` 技能。
  - **若選「不要」**：流程正常結束。

- **發送失敗**（輸出含 `ERROR`）：
  根據錯誤類型處理：
  - 認證錯誤 → "發送時認證失敗，密碼可能已過期。請重新提供密碼/授權碼。"
  - 連線錯誤 → "無法連接 SMTP 伺服器，請檢查網路連線。"
  - 附件錯誤 → "附件檔案不存在：[檔案路徑]。請確認檔案路徑。"
  - 其他 → 顯示錯誤訊息，詢問是否重試。

**若使用者選「修改」：**
- 詢問要修改哪個部分：收件人/抄送/暗送/主旨/內文/附件。
- 回到第 2 步重新呈現互動式表單（預填當前值），或回到第 3 步（若修改附件）。
- 修改後重新執行第 4 步預覽。

**若使用者選「捨棄」：**
- "已取消本次郵件發送。" 停止流程。

---

### 第 6 步：保存寄件人資訊（選用）

僅當第 5 步使用者選「要，建立專屬技能」時執行。

---

## 用戶專屬技能建立流程

**重要原則：永不修改原始的 CC-EmailSender 技能。專屬技能一律以新名稱獨立建立。**

### 觸發條件

觸發於第 5 步中使用者選擇「要，建立專屬技能」。

### 建立步驟

1. **決定技能名稱**：
   ```
   格式：CC-EmailSender-{用戶識別名}
   範例：CC-EmailSender-KuoWenhui
   ```
   用戶識別名由對話中已知的使用者姓名/帳號決定。若未知，使用郵箱 @ 前面的部分。

2. **使用 SkillManage 建立新技能**：
   呼叫 `SkillManage` 工具，command 為 `"create"`，提供以下內容：

   - **name**: `CC-EmailSender-{用戶識別名}`
   - **description**: "個人專屬郵件發送器 — {用戶郵箱}。無需重新輸入 SMTP 設定與密碼。"
   - **content (SKILL.md)**: 以 CC-EmailSender 的完整 SKILL.md 為基礎，進行以下修改：
     - 在最上方 frontmatter 之後新增一個「預設寄件人資訊」區塊（純變數，勿寫入密碼明文到說明文字中）
     - 將第 1 步（1a 取得郵箱、1b 判斷種類、1c SMTP 偵測、1d 密碼）標記為「已設定，跳過」
     - 在 SMTP 命令的範本中，直接代入已儲存的 `sender_email`、`smtp_server`、`smtp_port`、`ssl_mode`、`password`、`display_name`
     - 在資料安全段落中增加備註：「此為個人專屬技能，寄件人資訊已預先設定。」

3. **建立完成後告知使用者**：
   > "已為您建立個人專屬郵件技能 **{技能名稱}**。今後您需要發信時，直接告訴我「用 {技能名稱} 發郵件」即可，無需再次輸入郵箱與密碼。"

### 專屬技能的使用流程

當用戶觸發專屬技能時，流程如下：
- **跳過第 1 步**（寄件人設定 — SMTP 與密碼已預設）
- 直接從 **第 2 步**（互動式表單）開始
- 後續步驟（附件、預覽、發送、保存）完全一致

---

## display_name（寄件人名稱）處理

寄件人顯示名稱為選填欄位。取得方式：
1. 若對話上下文中已知使用者姓名 → 自動使用。
2. 若未知 → 從郵箱地址的使用者名部分推斷，或留空。
3. 使用者可在修改步驟中指定。

---

## 資料安全

- **密碼不儲存於通用技能**：密碼/授權碼僅存在於當前對話的臨時變數中，對話結束即消失。
- **不回顯密碼**：在回覆中絕不顯示密碼內容。
- **通用技能無設定檔**：CC-EmailSender 不依賴任何持久化設定檔，SMTP 設定每次現場詢問。
- **多人安全**：不同使用者使用通用技能時不會互相影響，無殘留資料。
- **專屬技能安全提醒**：若使用者選擇建立個人專屬技能（`CC-EmailSender-{用戶名}`），寄件人資訊（含密碼/授權碼）將保存在該技能的 SKILL.md 中。請使用者自行評估風險，僅在信任環境中使用。

---

## 命令參考

### 偵測模式
```bash
python "C:\Users\969421\.workbuddy\skills\CC-EmailSender\scripts\send_email.py" --detect-smtp "EMAIL"
```

### 登入驗證模式（取代舊的 TCP 預測試，一併驗證連線與認證）
```bash
python "C:\Users\969421\.workbuddy\skills\CC-EmailSender\scripts\send_email.py" --test-connection \
  --sender "EMAIL" --smtp-server "SERVER" --smtp-port "PORT" \
  --ssl-mode "ssl|starttls|none" --password "PWD"
```

### 預覽模式
```bash
python "C:\Users\969421\.workbuddy\skills\CC-EmailSender\scripts\send_email.py" --preview \
  --sender "EMAIL" --smtp-server "SERVER" --smtp-port "PORT" \
  --ssl-mode "ssl|starttls|none" --password "PWD" \
  --to "TO" --cc "CC" --bcc "BCC" \
  --subject "SUBJECT" --body "BODY" --attach "FILE1" "FILE2"
```

### 發送模式
```bash
python "C:\Users\969421\.workbuddy\skills\CC-EmailSender\scripts\send_email.py" \
  --sender "EMAIL" --smtp-server "SERVER" --smtp-port "PORT" \
  --ssl-mode "ssl|starttls|none" --password "PWD" \
  --to "TO" --cc "CC" --bcc "BCC" \
  --subject "SUBJECT" --body "BODY" --attach "FILE1" "FILE2"
```

### 參數清單

| 參數 | 必需 | 說明 |
|------|------|------|
| `--detect-smtp EMAIL` | 偵測模式 | 根據郵箱地址偵測 SMTP 設定 |
| `--test-connection` | 登入驗證 | 驗證 SMTP 連線與認證（一併完成，不做預先 TCP 測試） |
| `--preview` | 預覽模式 | 僅預覽不發送 |
| `--sender EMAIL` | 發送/測試 | 寄件人郵箱 |
| `--smtp-server ADDR` | 發送/測試 | SMTP 伺服器位址 |
| `--smtp-port PORT` | 發送/測試 | SMTP 連接埠 |
| `--ssl-mode MODE` | 發送/測試 | 加密模式: ssl / starttls / none |
| `--password PWD` | 發送/測試 | 密碼/授權碼/應用專用密碼 |
| `--display-name NAME` | 選填 | 寄件人顯示名稱 |
| `--to EMAILS` | 發送 | 收件人（逗號分隔） |
| `--cc EMAILS` | 選填 | 抄送（逗號分隔） |
| `--bcc EMAILS` | 選填 | 暗送（逗號分隔） |
| `--subject TEXT` | 發送 | 郵件主旨 |
| `--body TEXT` | 發送 | 郵件內文 |
| `--body-file PATH` | 替代 | 從檔案讀取內文 |
| `--html` | 選填 | 標記內文為 HTML |
| `--attach FILES` | 選填 | 附件路徑（可多個） |

---

## 支援的郵箱提供商

| 提供商 | 域名 | SMTP 伺服器 | 連接埠 | 加密 | 認證方式 |
|--------|------|------------|--------|------|----------|
| Gmail | gmail.com | smtp.gmail.com | 465 | SSL | 應用專用密碼 |
| Outlook | outlook.com, hotmail.com, live.com | smtp-mail.outlook.com | 587 | STARTTLS | 密碼 |
| Office 365 | office365.com | smtp.office365.com | 587 | STARTTLS | 密碼 |
| 網易 163 | 163.com | smtp.163.com | 465 | SSL | 授權碼 |
| 網易 126 | 126.com | smtp.126.com | 465 | SSL | 授權碼 |
| 網易 Yeah | yeah.net | smtp.yeah.net | 465 | SSL | 授權碼 |
| 網易企業/靈犀 | qiye.163.com, ntes53.netease.com | smtp.qiye.163.com（主）/ smtpv6hz.qiye.ntes53.netease.com（備用） | 465 | SSL | 密碼/授權碼 |
| QQ 郵箱 | qq.com | smtp.qq.com | 465 | SSL | 授權碼 |
| Foxmail | foxmail.com | smtp.qq.com | 465 | SSL | 授權碼 |
| 新浪 | sina.com | smtp.sina.com | 465 | SSL | 密碼 |
| 搜狐 | sohu.com | smtp.sohu.com | 465 | SSL | 密碼 |
| 阿里雲 | aliyun.com | smtp.qiye.aliyun.com | 465 | SSL | 密碼 |
| Yahoo | yahoo.com | smtp.mail.yahoo.com | 465 | SSL | 應用專用密碼 |
| iCloud | icloud.com | smtp.mail.me.com | 587 | STARTTLS | 應用專用密碼 |
| Zoho | zoho.com | smtp.zoho.com | 465 | SSL | 密碼 |
| 未知/企業域名 | * | 詢問使用者 | 詢問 | 詢問 | 詢問 |

完整 SMTP 設定參考見 `references/smtp_settings.md`。

---

## 資源

- `scripts/send_email.py` — 核心腳本（Python 標準庫，無外部依賴）
- `references/smtp_settings.md` — 各郵箱提供商 SMTP 設定參考
