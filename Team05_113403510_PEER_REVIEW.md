# Peer Review Report

> **Instructions:** Complete this form **individually and independently**.
> Do not discuss your ratings with teammates before submitting.
> Submit via EEClass as a **separate, confidential submission** — not in the shared team repo.
> Your teammates will not see this report.
>
> Reference the team's `WORK_ALLOCATION_TEMPLATE.md` when completing this form.

---

## Your Details

| Field | Your answer |
|-------|------------|
| Full Name | 陳少畇 |
| Student ID | 113403510|
| Team ID | team5|
| Date submitted | 2026/06/07 |

---

## Rating Scale

| Rating | Meaning |
|--------|---------|
| **5** | Exceeded expectations — delivered more than agreed; helped teammates; consistently high quality |
| **4** | Met expectations fully — delivered exactly what was agreed; on time; good quality |
| **3** | Mostly met expectations — minor shortfalls; one or two items completed late or with help |
| **2** | Partially met expectations — noticeable gaps; teammates had to cover some tasks |
| **1** | Did not meet expectations — significant tasks left incomplete; very limited contribution |

---

## Section A — Self-Assessment

### A1. What did you personally implement?

List the specific tasks, functions, files, or document sections that you were the primary author of.
Be specific (e.g., "I designed all 12 tables in schema.sql and implemented query_national_rail_availability and execute_booking").

> 在本專案中，我主要負責 PostgreSQL 查詢層的函數實作、密碼安全整合、交易邏輯設計、Schema 合規審查，以及設計文件的部分章節撰寫。
 
---
 
### 查詢函數實作（`queries.py`）
 
實作了所有核心查詢與寫入函數：
 
| 函數 | 類型 |
|------|------|
| `query_national_rail_availability` | 查詢 |
| `query_metro_schedules` | 查詢 |
| `query_national_rail_fare` | 查詢 |
| `query_metro_fare` | 查詢 |
| `query_available_seats` | 查詢 |
| `query_user_profile` | 查詢 |
| `query_user_bookings` | 查詢 |
| `query_payment_info` | 查詢 |
| `execute_booking` | 寫入 |
| `execute_cancellation` | 寫入 |
 
---
 
### 密碼安全重構
 
原始程式碼以明文儲存密碼並在 SQL 中直接比對。我將整個認證流程重構為 **argon2id 雜湊**，修改涵蓋以下四個位置：
 
1. **`queries.py` 頂部** — 新增 `argon2` 相關 import
2. **`register_user()`** — 在 INSERT 前呼叫 `_ph.hash()` 產生雜湊
3. **`login_user()`** — 改為先用 SQL 取出 hash 字串，再於 Python 層呼叫 `_ph.verify()` 驗證
   > argon2id 每次雜湊結果不同（鹽值隨機嵌入），無法用 `WHERE password_hash = %s` 直接比對
4. **`update_password()`** — 改為儲存新雜湊而非明文
同步更新 `seed_postgres.py` 的 `seed_users()`，使 mock 資料也以 argon2id 雜湊形式寫入資料庫。
 
---
 
### `execute_booking` 原子性修正
 
原始 `execute_booking()` 只 commit 了訂票 INSERT，缺少 payment INSERT。
 
- 補上 `payments` 表的 INSERT
- 確保訂票與付款兩筆操作包在**同一個 `conn.commit()`** 內
- 排查並移除程式碼中意外留下的**重複 `conn.commit()` 呼叫**
---
 
### Schema 審查與除錯（`schema.sql`）
 
- 確認評分標準要求每個資料表的 PK 欄位**旁邊**都需要有 inline 說明，僅靠頂部區塊不符合要求，補齊所有資料表的行內註解
- 統一說明三層式 PK 策略：
  - `SERIAL` — 靜態參考表（車站、時刻表）
  - `UUID` — 敏感交易表（用戶、訂單、付款）
  - `VARCHAR` 自然鍵 — 領域定義的目錄表（ticket_types、refund_policies）
- 排查出 `national_rail_seat_layouts` 缺少 `code` 欄位導致 seeding 失敗
- 確認 Docker volume 保留舊 schema 狀態的機制，需執行 `docker-compose down -v && docker-compose up -d` 才能讓變更生效
---
 
### 設計文件
 
- 撰寫 **Section 4**（Vector / RAG Design）
- 撰寫 **Section 5**（AI Tool Usage Evidence）— 挑選並整合多方 AI 使用紀錄，編輯成符合評分格式的五個完整例子
---

### A2. What challenges did you face?

Describe any technical or collaboration difficulties you personally encountered and how you resolved them.

> The main technical challenge was ensuring the schema matched what the seed scripts expected — for example, national_rail_seat_layouts was missing a code column which caused seeding to fail, and required a full docker-compose down -v reset to resolve. I also had to debug argon2id integration since the original code stored plaintext passwords, and had to restructure execute_booking to wrap both the booking and payment inserts in a single transaction commit to meet the atomicity requirement.

---

### A3. Self-rating

| Criterion | Rating (1–5) | Justification (1–2 sentences) |
|-----------|-------------|-------------------------------|
| I delivered the tasks assigned to me in the work allocation | | |
| The quality of my work was satisfactory | | |
| I communicated well and kept the team informed | | |
| I met deadlines agreed within the team | | |
| **Overall self-rating** | | |

---

### A4. Estimated contribution percentage

What percentage of the total team effort do you estimate you personally contributed?

> My estimated contribution: **____%**

---

## Section B — Peer Assessments

Complete one subsection per teammate. Add or remove subsections to match your team size.
If your team has 2 members, complete B1 only. If 3 members, complete B1 and B2.

---

### B1. Assessment of Teammate 1

| Field | Your answer |
|-------|------------|
| Teammate's full name | 陳佑瑄 |
| Teammate's student ID | 113403004 |

#### What did this teammate deliver?

List the tasks, functions, files, or document sections that this teammate was the primary author of,
based on what you observed during the project (compare against the work allocation).

> *Your answer:*

#### Did their actual contribution match the agreed work allocation?

> *Your answer (Yes / Mostly / Partially / No — with explanation):*

#### Peer rating for this teammate

| Criterion | Rating (1–5) | Justification (1–2 sentences) |
|-----------|-------------|-------------------------------|
| Delivered the tasks assigned in the work allocation | | |
| Quality of their work was satisfactory | | |
| Communicated well and kept the team informed | | |
| Met deadlines agreed within the team | | |
| **Overall rating for this teammate** | | |

#### Estimated contribution percentage for this teammate

> My estimate of their contribution: **____%**

---

### B2. Assessment of Teammate 2

| Field | Your answer |
|-------|------------|
| Teammate's full name | 姚喬嫚 |
| Teammate's student ID | 113403512 |

#### What did this teammate deliver?

> *Your answer:*

#### Did their actual contribution match the agreed work allocation?

> *Your answer (Yes / Mostly / Partially / No — with explanation):*

#### Peer rating for this teammate

| Criterion | Rating (1–5) | Justification (1–2 sentences) |
|-----------|-------------|-------------------------------|
| Delivered the tasks assigned in the work allocation | | |
| Quality of their work was satisfactory | | |
| Communicated well and kept the team informed | | |
| Met deadlines agreed within the team | | |
| **Overall rating for this teammate** | | |

#### Estimated contribution percentage for this teammate

> My estimate of their contribution: **____%**

---

## Section C — Contribution Percentage Summary

All members (including yourself) must sum to 100%.

| Member | Your estimated % | Notes |
|--------|----------------|-------|
| Yourself | 40% | |
| Teammate 1 | 35% | |
| Teammate 2 | 25% |  |
| **Total** | **100%** | |

---

## Section D — Overall Team Reflection

### D1. What went well in the team's collaboration?

> *Your answer (2–4 sentences):*

---

### D2. What would you do differently if you did this project again?

> *Your answer (2–4 sentences):*

---

### D3. Is there anything else the markers should know about team dynamics or individual contributions?

This is optional. Use it only if there is important context that the ratings above do not capture
(e.g., a member had a documented personal emergency, or a member was unresponsive for a significant period).

>Nothing to add.

---

## Declaration

I confirm that this peer review reflects my honest and independent assessment.
I understand it will be kept confidential from my teammates.

**Signed:** ______________陳少畇________________ **Date:** _______________
