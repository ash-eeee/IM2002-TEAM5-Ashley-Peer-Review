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

> ### 挑戰一：Schema 變更未套用至執行中的容器
 
**問題**
 
在 `schema.sql` 新增 `code` 欄位後，seed 腳本仍然報錯：
 
```
column "code" of relation "national_rail_seat_layouts" does not exist
```
 
起初以為是 SQL 語法有誤，反覆確認後才意識到 Docker volume 會保留舊的 schema 狀態，直接編輯 `.sql` 檔案對已運行的容器沒有任何效果。
 
**解決方法**
 
執行以下指令，其中 `-v` 參數強制刪除舊 volume，讓資料庫從更新後的 schema 重新初始化：
 
```bash
docker-compose down -v && docker-compose up -d
```
 
---
 
### 挑戰二：argon2id 的驗證邏輯與 SQL 比對不相容
 
**問題**
 
改用 argon2id 後，原本 `login_user()` 中的 SQL 比對方式無法繼續使用：
 
```python
WHERE password_hash = %s  # 無法用於 argon2id
```
 
由於 argon2id 每次雜湊時會隨機嵌入鹽值，同一個密碼的兩次雜湊結果不同，無法用等號直接比對。
 
**解決方法**
 
改為兩步驟處理：
 
1. 用 SQL 取出儲存的 hash 字串
2. 在 Python 層呼叫 `_ph.verify()` 進行驗證
同時確認回傳 dict 前需將 `password_hash` 欄位移除，避免雜湊值外洩給呼叫端。
 
---
 
### 挑戰三：`execute_booking` 的重複 commit bug
 
**問題**
 
在補上 payment INSERT 的過程中，程式碼中意外留下兩個 `conn.commit()` 呼叫。此類錯誤不易從功能層面察覺，psycopg2 不一定會立即拋出明顯錯誤。
 
**解決方法**
 
仔細追蹤整個函數的 transaction 生命週期，確認：
 
- 只有**一個** `conn.commit()`
- 該 commit 同時涵蓋訂票與付款兩筆 INSERT
- 符合評分標準的原子性要求
 
---

### A3. Self-rating

| Criterion | Rating (1–5) | Justification (1–2 sentences) |
|-----------|-------------|-------------------------------|
| I delivered the tasks assigned to me in the work allocation | 5 | 負責的查詢函數、密碼安全重構、交易邏輯修正與設計文件章節都有完成，沒有遺漏分配到的項目。 |
| The quality of my work was satisfactory | 4 | 函數邏輯與評分標準的要求大致符合，但過程中有幾個問題（如缺少欄位、重複 commit）需要除錯才解決，所以沒有給滿分。 |
| I communicated well and kept the team informed | 5 | 遇到 schema 變更需要重置資料庫這類會影響其他人的問題，都有即時告知組員需要執行的步驟。|
| I met deadlines agreed within the team | 5 | 分配到的項目都在約定的時間內完成，沒有拖到其他人的進度。|
| **Overall self-rating** | 4.5 | |

---

### A4. Estimated contribution percentage

What percentage of the total team effort do you estimate you personally contributed?

> My estimated contribution: **37%**

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

>負責neo4j的部分，完成seed_neo4j、seed.cypher的主架構，後續也有協助schema.sql、queries.py的細項修改。也負責doc裡面的第三部分。也有幫忙把schema裡需要的password部分寫進去，不過沒有hash。

#### Did their actual contribution match the agreed work allocation?

>yes, 她有完成該負責的neo4j的部分，後續我們也有一同檢查。

#### Peer rating for this teammate

| Criterion | Rating (1–5) | Justification (1–2 sentences) |
|-----------|-------------|-------------------------------|
| Delivered the tasks assigned in the work allocation | 5 | 負責的東西都有做。 |
| Quality of their work was satisfactory | 4.5 | 大致內容完整，初次檢查就都看得到了，只有再進行小修改調整。但後續幫 |
| Communicated well and kept the team informed | 5 | 隨時在線上更新幾最新進度並有用merge的pull request，改之前也會確認是否會跟他人conflict。|
| Met deadlines agreed within the team | 5 | 如期完成自己負責事項，是最早完成的人，也有持續幫忙用AI精進內容。|
| **Overall rating for this teammate** | 4.5| |

#### Estimated contribution percentage for this teammate

> My estimate of their contribution: **30%**

---

### B2. Assessment of Teammate 2

| Field | Your answer |
|-------|------------|
| Teammate's full name | 姚喬嫚 |
| Teammate's student ID | 113403512 |

#### What did this teammate deliver?

> schema.sql, seed_postgres.py最初的主架構，後續出問題也與我一同合作將東西修改，也有幫忙將queries.py（relational的）一起檢查做調整。
#### Did their actual contribution match the agreed work allocation?

> 大致上符合，因為她負責的為主架構，可是最初的東西有些沒有寫好，也沒有將password另存，所以後續由其他人完成。不過最初把內容寫varchar，最後也有負責把內容改成uuid,serial。

#### Peer rating for this teammate

| Criterion | Rating (1–5) | Justification (1–2 sentences) |
|-----------|-------------|-------------------------------|
| Delivered the tasks assigned in the work allocation | 4.8 | 有把自己負責項目完成，也有後續繼續修改。 |
| Quality of their work was satisfactory | 3.7 | 最初寫的東西到最後幾乎被改得不一樣了，中間也多次無法seed資料，而且問題多出現在schema,seed_postgres.py。|
| Communicated well and kept the team informed | 4 | 有時候會獨自默默作業，在修改某個問題，甚至修好了，但是沒有告訴其他人，所以可能有人先寫好了，她前面做的事就白費了，因為對方不知道她也有寫，所以先推上去了。 |
| Met deadlines agreed within the team | 4.6 | 最初要完成主架構的時間往後延，也因此需要等她的內容，拖延了一些其他進度。 |
| **Overall rating for this teammate** | 4.3 | |

#### Estimated contribution percentage for this teammate

> My estimate of their contribution: **33%**

---

## Section C — Contribution Percentage Summary

All members (including yourself) must sum to 100%.

| Member | Your estimated % | Notes |
|--------|----------------|-------|
| Yourself | 37% | |
| Teammate 1 | 30% | |
| Teammate 2 | 33% |  |
| **Total** | **100%** | |

---

## Section D — Overall Team Reflection

### D1. What went well in the team's collaboration?

>大家一同在線上開會，每次開就開七小時以上，一同作業，有問題也願意跳出來解決，出現問題時大家也會有默契的一起修訂標準，更改內容。只要有東西完成推上去，大家也會立馬幫忙測試，看是否可行。

---

### D2. What would you do differently if you did this project again?

>在一開始先了解好uuid，serial跟varchar在現實中的採用問題，才不會導致最後需要整個大調整。然後先了解每個電腦是不不同個程式碼測試出來的東西會成效不一樣，才不會陷入不斷修改的迴圈。會希望先分派大家去了解某些觀念，還有確認可以push，才不護有人的本地端無法同步，還要解決這個問題。希望大家可以實體開會，更方便快速看到對方作業。然後重來一次的話，不會以垂直作業為主要分工方式，而是水評分共為主，垂直分工為輔，因為每個人所有東西都會碰到一些。最後，希望有機會可以一開始就用英文註解，後續的註解也花了一點時間。

---

### D3. Is there anything else the markers should know about team dynamics or individual contributions?

This is optional. Use it only if there is important context that the ratings above do not capture
(e.g., a member had a documented personal emergency, or a member was unresponsive for a significant period).

>Nothing to add.

---

## Declaration

I confirm that this peer review reflects my honest and independent assessment.
I understand it will be kept confidential from my teammates.

**Signed:** ______________陳少畇________________ **Date:** 2026/06/12
