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
- 腳本路徑: `~/.workbuddy/skills/CC-EmailSender/scripts/send_email.py`

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
> - 每個 `question` 的 `options` 至少 2 個，包含合理預設值或快捷選項。
> - 使用者需自由輸入文字時，點選對應 option 後在 **Other** 輸入框中填寫。
> - 多個獨立欄位可在同一個 `AskUserQuestion` 中同時呈現（最多 4 個 question）。

---

### 第 1 步：寄件人設定（郵箱 → 服務商分析與確認 → 偵測 → 密碼）

#### 1a. 判斷並取得寄件人郵箱

**首先判斷使用者是否已在初始訊息中提供寄件人郵箱地址。**

判斷規則：掃描使用者的觸發訊息，尋找電子郵箱格式（如 `xxx@xxx.com`、`xxx@xxx.com.cn` 等）。

**情況 A — 使用者已提供寄件人郵箱：**

直接使用該郵箱作為寄件人，**無需再次詢問**。
立即進入服務商分析（見 1b-服務商分析與確認）。

**情況 B — 使用者未提供寄件人郵箱：**

僅顯示一個輸入框，讓使用者直接輸入寄件人郵箱地址，不要在同一步驟中混入服務商選項：

```json
{
  "questions": [{
    "question": "請輸入寄件人郵箱地址",
    "header": "寄件人郵箱",
    "options": []
  }]
}
```

使用者透過底部 **Other 自定義輸入框** 輸入完整郵箱地址後，進入 1b 的服務商分析與確認。

#### 1b. 郵箱服務商分析與確認

**取得寄件人郵箱後，首先根據郵箱域名分析其所屬服務商，然後讓使用者確認分析是否正確。**

##### 域名分析規則（依程式邏輯判斷，不呼叫外部命令）

根據郵箱地址的 @ 後綴進行分析：

| 域名後綴 | 服務商 |
|---------|--------|
| `@gmail.com` | Gmail |
| `@outlook.com` / `@hotmail.com` / `@live.com` / `@msn.com` | Outlook |
| `@qq.com` / `@foxmail.com` / `@vip.qq.com` | QQ郵箱 |
| `@163.com` / `@126.com` / `@yeah.net` / `@188.com` | 網易 |
| `@qiye.163.com` / `@ntes53.netease.com` | 網易企業郵箱/靈犀 |
| `@yahoo.com` / `@yahoo.com.tw` / `@yahoo.co.jp` | Yahoo |
| `@icloud.com` / `@me.com` / `@mac.com` | iCloud |
| `@sina.com` / `@sina.cn` | 新浪 |
| `@sohu.com` | 搜狐 |
| `@aliyun.com` | 阿里雲 |
| `@zoho.com` | Zoho |
| 無法匹配以上規則 | 未知服務商 / 企業自定義域名 |

##### 使用 AskUserQuestion 讓使用者確認分析結果

```json
{
  "questions": [{
    "question": "分析您的郵箱為 **［服務商名稱］**，是否正確？",
    "header": "服務商確認",
    "options": [
      {"label": "正確", "description": "服務商識別正確，繼續執行 SMTP 偵測"},
      {"label": "不正確", "description": "服務商識別錯誤，重新選擇服務商"}
    ]
  }]
}
```

**處理邏輯：**

**若選「正確」：** 使用已分析的服務商資訊，繼續執行 SMTP 偵測（見 1c-偵測）。

**若選「不正確」：** 使用第二個 `AskUserQuestion` 提供四個主要選項，並在 Other 中提供手動輸入框：

```json
{
  "questions": [{
    "question": "請選擇正確的郵箱服務商，或在下方 Other 中輸入自定義服務商名稱",
    "header": "選擇服務商",
    "options": [
      {"label": "網易", "description": "163/126/yeah.net/企業郵箱等"},
      {"label": "Outlook", "description": "Outlook/Hotmail/Live/Office365"},
      {"label": "QQ郵箱", "description": "QQ/Foxmail 郵箱"},
      {"label": "Gmail", "description": "Google Gmail 郵箱服務"}
    ]
  }]
}
```

- 若使用者點選上述四個選項之一 → 記錄所選服務商。
- 若使用者點選 **Other** → 在手動輸入框中填寫自定義服務商名稱（如"新浪"、"iCloud"、"企業自定義"等）。
- 記錄最終確定的服務商名稱後，繼續執行 SMTP 偵測（見 1c-偵測）。

> **設計原則：** 即使使用者糾正了服務商，後續的 `--detect-smtp` 腳本仍然會根據郵箱首碼進行自動偵測。使用者的選擇主要用於引導後續的密碼類型提示（授權碼 vs 應用專用密碼 vs 登入密碼）。

#### 1c-偵測：SMTP 自動偵測 + 配置表直接匹配

**此步驟結合自動偵測與 SMTP 配置表直接匹配，確保即使自動偵測失敗也能找到伺服器設定。**

##### 第 1 階段：自動偵測

取得寄件人郵箱並確認服務商後，執行偵測命令：

```bash
python ~/.workbuddy/skills/CC-EmailSender/scripts/send_email.py --detect-smtp "使用者郵箱"
```

回傳 JSON 含 `provider_name`、`smtp_server`、`smtp_port`、`ssl_mode`、`auth_type`。

- **已知（smtp_server 非空）** → "已識別為 **[provider_name]**，伺服器 `[server]:[port]`。" → 直接跳至 1d。
- **未知（smtp_server 為 null）** → 進入第 2 階段：SMTP 配置表直接匹配。

##### 第 2 階段：SMTP 配置表直接匹配

當自動偵測無法識別服務商時（常見於企業自定義域名、新域名或服務商變更），使用下方的 **SMTP 服務商配置表** 進行直接匹配。

**匹配規則（依序列舉，返回第一個匹配結果）：**

1. **精確域名匹配** — 將使用者的郵箱域名與配置表中的 `domain_pattern` 逐一比對
2. **服務商名稱匹配** — 若使用者在 1b 步驟中已手動選擇或輸入服務商名稱，直接匹配配置表中的 `provider_name`
3. **使用者手動指定** — 若以上均不匹配，詢問使用者是否知道 SMTP 伺服器位址，或從配置表中選擇服務商

**使用 AskUserQuestion 呈現服務商選擇（當自動偵測失敗時）：**

```json
{
  "questions": [{
    "question": "自動偵測未找到此郵箱的 SMTP 設定。請選擇您的郵箱服務商，系統將自動填入配置：",
    "header": "選擇服務商",
    "options": [
      {"label": "網易企業郵箱", "description": "smtp.qiye.163.com:465 或 smtpv6hz.qiye.ntes53.netease.com:587"},
      {"label": "網易 163", "description": "smtp.163.com:465 (SSL)"},
      {"label": "QQ 郵箱", "description": "smtp.qq.com:465 (SSL)"},
      {"label": "Gmail", "description": "smtp.gmail.com:465 (SSL)"},
      {"label": "Outlook", "description": "smtp-mail.outlook.com:587 (STARTTLS)"},
      {"label": "其他 / 自定義", "description": "手動輸入 SMTP 伺服器位址與連接埠"}
    ]
  }]
}
```

- 使用者選擇配置表中的服務商 → **直接從下方 SMTP 配置表讀取對應的 `smtp_server`、`smtp_port`、`ssl_mode`、`auth_type`**，無需使用者再手動輸入。
- 使用者選「其他 / 自定義」→ 使用 `AskUserQuestion` 分別詢問 SMTP 伺服器位址、連接埠、加密方式。

##### 第 3 階段：取得 SMTP 設定後的動作

無論是第 1 階段自動偵測還是第 2 階段配置表匹配，一旦取得 SMTP 設定即進行：

1. **記錄變數**：`smtp_server`、`smtp_port`、`ssl_mode`、`auth_type`
2. **回報使用者**（普通文字）："已取得 SMTP 設定：`[server]:[port]`，加密方式：[ssl_mode]"
3. **繼續至 1d**（獲取密碼/授權碼）

#### 1d. 取得密碼/授權碼

使用 `AskUserQuestion` 提供輸入框，讓使用者輸入 SMTP 驗證密碼：

```json
{
  "questions": [{
    "question": "請輸入 SMTP 密碼 / 授權碼 / 應用專用密碼",
    "header": "安全驗證",
    "options": [
      {"label": "授權碼", "description": "網易/QQ 等需要授權碼，請點擊下方 Other 輸入授權碼"},
      {"label": "應用專用密碼", "description": "Gmail/iCloud/Yahoo 需要應用專用密碼，請點擊下方 Other 輸入"},
      {"label": "登入密碼", "description": "Outlook/企業郵箱使用登入密碼，請點擊下方 Other 輸入密碼"}
    ]
  }]
}
```

使用者透過 **Other 自定義輸入框** 輸入密碼/授權碼後，執行連線測試：

```bash
python ~/.workbuddy/skills/CC-EmailSender/scripts/send_email.py \
  --test-connection \
  --sender "sender_email" \
  --smtp-server "smtp_server" \
  --smtp-port "smtp_port" \
  --ssl-mode "ssl_mode" \
  --password "使用者提供的密碼"
```

- **連線成功** → "驗證成功！" → 進入第 2 步。
- **連線失敗** → 用文字提示錯誤類型，引導重新輸入或回到 1a。

**安全提醒：密碼僅存在於當前對話的臨時變數中，不回顯。**


### 第 2 步：郵件撰寫（互動式表單）

**核心設計：使用 `AskUserQuestion` 同時提供多個輸入框，讓使用者一次填寫所有郵件欄位。**

#### 2a. 郵件基本欄位（收件人、抄送、暗送、主旨）

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
      "header": "抄送",
      "options": [
        {"label": "不抄送", "description": "不需要抄送任何人"},
        {"label": "手動輸入", "description": "點擊下方 Other 輸入抄送郵箱地址（多個用逗號分隔）"}
      ]
    },
    {
      "question": "請輸入暗送對象（選填，多個用逗號分隔）",
      "header": "暗送",
      "options": [
        {"label": "不暗送", "description": "不需要暗送任何人"},
        {"label": "手動輸入", "description": "點擊下方 Other 輸入暗送郵箱地址（多個用逗號分隔）"}
      ]
    },
    {
      "question": "請輸入郵件主旨",
      "header": "主旨",
      "options": [
        {"label": "由 AI 建議", "description": "根據郵件內容由 AI 自動生成主旨"},
        {"label": "手動輸入", "description": "點擊下方 Other 輸入郵件主旨"}
      ]
    }
  ]
}
```

#### 2b. 內文撰寫方式

接著使用 `AskUserQuestion` 提供內文撰寫的兩個選項：

```json
{
  "questions": [{
    "question": "郵件內文如何撰寫？",
    "header": "內文",
    "options": [
      {"label": "AI自動生成(根據主旨與附件)", "description": "由 AI 根據主旨與附件內容自動生成專業郵件內文"},
      {"label": "自行填寫", "description": "點擊下方 Other 自定義輸入郵件內文"}
    ]
  }]
}
```

**解析使用者回覆：**

```
to_addrs  = 收件人（逗號/分號分隔）；若選「稍後補充」則為空，需後續補充
cc_addrs  = 抄送（可為空）；若選「不抄送」則為空
bcc_addrs = 暗送（可為空）；若選「不暗送」則為空
subject   = 主旨；若選「由 AI 建議」則標記為需 AI 生成
body_text = 內文；若選「AI自動生成(根據主旨與附件)」→ body_ai_generate = True
                              若選「自行填寫」→ 使用 Other 輸入的內容
```

**驗證：**
- 收件人不可為空 → 若為空，提示"收件人為必填欄位，請提供收件人郵箱地址。"
- 主旨不可為空 → 若標記為 AI 生成，則根據郵件上下文生成主旨。

**預填條件**：若使用者在觸發時已提供部分資訊（如收件人郵箱、主旨等），在對應欄位的 options 中加入已填值作為預設選項，並告知「已預填部分欄位，請確認或修改。」

---

### 第 3 步：附件（點選方式）

使用 `AskUserQuestion` 讓使用者選擇是否附加檔案：

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
- 直接進入第 4 步。

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
- 進入第 4 步。

**如果使用者在觸發 skill 時已附帶了檔案：**
- 不再詢問，直接將已偵測到的檔案列入 `attachments`。
- 告知使用者"已偵測到對話中的檔案：[檔案列表]，將作為附件。"

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
python ~/.workbuddy/skills/CC-EmailSender/scripts/send_email.py \
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
python ~/.workbuddy/skills/CC-EmailSender/scripts/send_email.py \
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
     - 將 `--detect-smtp` 步驟標記為「已設定，跳過」
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
python send_email.py --detect-smtp "EMAIL"
```

### 連線測試模式
```bash
python send_email.py --test-connection \
  --sender "EMAIL" --smtp-server "SERVER" --smtp-port "PORT" \
  --ssl-mode "ssl|starttls|none" --password "PWD"
```

### 預覽模式
```bash
python send_email.py --preview \
  --sender "EMAIL" --smtp-server "SERVER" --smtp-port "PORT" \
  --ssl-mode "ssl|starttls|none" --password "PWD" \
  --to "TO" --cc "CC" --bcc "BCC" \
  --subject "SUBJECT" --body "BODY" --attach "FILE1" "FILE2"
```

### 發送模式
```bash
python send_email.py \
  --sender "EMAIL" --smtp-server "SERVER" --smtp-port "PORT" \
  --ssl-mode "ssl|starttls|none" --password "PWD" \
  --to "TO" --cc "CC" --bcc "BCC" \
  --subject "SUBJECT" --body "BODY" --attach "FILE1" "FILE2"
```

### 參數清單

| 參數 | 必需 | 說明 |
|------|------|------|
| `--detect-smtp EMAIL` | 偵測模式 | 根據郵箱地址偵測 SMTP 設定 |
| `--test-connection` | 測試模式 | 測試 SMTP 連線和認證 |
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
| 網易企業/靈犀 | qiye.163.com, ntes53.netease.com | smtphz.qiye.163.com（華東）<br>smtpv6hz.qiye.ntes53.netease.com（587埠） | 465<br>587 | SSL<br>STARTTLS | 密碼/授權碼 |
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

## SMTP 配置表（依服務商名稱直接匹配）

此配置表用於 **1c-第 2 階段** 的直接匹配流程，在自動偵測失敗時使用。
**匹配依據**：使用者在 1b 步驟中確認或選擇的 `provider_name`。
**匹配結果**：直接獲取 `smtp_server`、`smtp_port`、`ssl_mode`、`auth_type`。

| 服務商名稱 (provider_name) | SMTP 伺服器 | 連接埠 | 加密 | 認證方式 |
|---------------------------|------------|--------|------|----------|
| 網易企業郵箱 / 靈犀 (新伺服器) | `smtpv6hz.qiye.ntes53.netease.com` | 587 | STARTTLS | 密碼/授權碼 |
| 網易企業郵箱 / 靈犀 (華東) | `smtphz.qiye.163.com` | 465 | SSL | 密碼/授權碼 |
| 網易企業郵箱 / 靈犀 (華北) | `smtp.qiye.163.com` | 465 | SSL | 密碼/授權碼 |
| 網易 163 | `smtp.163.com` | 465 | SSL | 授權碼 |
| 網易 126 | `smtp.126.com` | 465 | SSL | 授權碼 |
| 網易 Yeah | `smtp.yeah.net` | 465 | SSL | 授權碼 |
| 網易 188 | `smtp.188.com` | 465 | SSL | 授權碼 |
| QQ 郵箱 | `smtp.qq.com` | 465 | SSL | 授權碼 |
| Foxmail | `smtp.qq.com` | 465 | SSL | 授權碼 |
| QQ VIP 郵箱 | `smtp.qq.com` | 465 | SSL | 授權碼 |
| Gmail / Google Workspace | `smtp.gmail.com` | 465 | SSL | 應用專用密碼 |
| Outlook / Hotmail / Live / MSN | `smtp-mail.outlook.com` | 587 | STARTTLS | 密碼 |
| Office 365 | `smtp.office365.com` | 587 | STARTTLS | 密碼 |
| 新浪郵箱 | `smtp.sina.com` | 465 | SSL | 密碼 |
| 新浪免費郵箱 | `smtp.sina.cn` | 465 | SSL | 密碼 |
| 搜狐郵箱 | `smtp.sohu.com` | 465 | SSL | 密碼 |
| 阿里雲郵箱 | `smtp.qiye.aliyun.com` | 465 | SSL | 密碼 |
| Yahoo 郵箱 | `smtp.mail.yahoo.com` | 465 | SSL | 應用專用密碼 |
| Yahoo 奇摩 | `smtp.mail.yahoo.com` | 465 | SSL | 應用專用密碼 |
| Yahoo Japan | `smtp.mail.yahoo.co.jp` | 465 | SSL | 應用專用密碼 |
| iCloud / Apple | `smtp.mail.me.com` | 587 | STARTTLS | 應用專用密碼 |
| Zoho 郵箱 | `smtp.zoho.com` | 465 | SSL | 密碼 |
| ProtonMail (需 Bridge) | `127.0.0.1` (本地) | 1031 | STARTTLS | Bridge 密碼 |

> **使用規則**：在 1c-第 2 階段中，當自動偵測返回 `smtp_server: null` 時：
> 1. 若使用者在 1b 步驟已手動選擇了服務商名稱 → 直接從上表讀取該服務商的 SMTP 設定
> 2. 若使用者在 1b 步驟選擇「不正確」且重新選擇了服務商 → 從上表讀取對應設定
> 3. 若服務商名稱不在上表中 → 使用 `AskUserQuestion` 讓使用者手動輸入 SMTP 伺服器位址與連接埠
>
> **匹配實例**：使用者郵箱 `admin@company.qiye.163.com`，自動偵測未識別（自定義子域名）。
> → 使用者在 1b 選擇「不正確」→ 重新選擇「網易企業郵箱 / 靈犀 (新伺服器)」
> → 上表匹配 provider_name → 獲取 `smtpv6hz.qiye.ntes53.netease.com:587`（STARTTLS）

---

## 資源

- `scripts/send_email.py` — 核心腳本（Python 標準庫，無外部依賴）
- `references/smtp_settings.md` — 各郵箱提供商 SMTP 設定參考
