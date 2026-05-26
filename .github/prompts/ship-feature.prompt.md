---
description: "一條龍打包小功能：plan → code → verify → commit → PR"
---

# 任務：上線一個小功能

請幫我實現一個小功能的完整流程。只要執行 `/ship-feature <一句話描述>` 就能自動跑完：需求規劃、程式碼、驗證、送 PR。

## 使用方式

提三個範例，讓我懂怎麼用：

- `/ship-feature 首頁加一個「依讚數排序」按鈕`
- `/ship-feature 問題卡顯示 created_at 時間戳`
- `/ship-feature 新增 /about 頁面介紹 ClassWall`

---

## 📋 步驟（禁止跳，必須依序）

### 1️⃣ 讀規範 + 列風險

先讀這些檔案：

- `AGENTS.md` **全文**（Core convention）
- `.github/instructions/components.instructions.md`（React 規範）
- `.github/instructions/supabase.instructions.md`（Supabase + RLS）
- `.github/instructions/tailwind.instructions.md`（Tailwind v4）

**後搜出 3–5 條相關規則**，例如：
> ✅ 「前端只用 anon key，按讚一律走 RPC `increment_question_like`」
> ✅ 「Tailwind v4 用 `bg-linear-*` 不是 `bg-gradient-*`」
> ✅ 「所有 schema 變動改 `0001_init.sql` 並保持冪等」

---

### 2️⃣ 查 Schema 與資料量

用 Supabase MCP（或直接貼 schema）看現狀：

```bash
# 只允許 SELECT！禁止 INSERT/UPDATE/DELETE
select * from information_schema.tables where table_schema = 'public';
select * from public.questions limit 1;
```

**必須回報**：
- 新功能用哪些表、欄位、新增或修改？
- 資料量大嗎？需要 index 嗎？
- 有無 Realtime 需求？

---

### 3️⃣ 📝 停下來：提計畫（等 OK 才能動）

**🛑 寫在這裡，等使用者說「OK 開始動」才能繼續**

計畫內容須包括：

- **檔案清單**：新增或修改的檔案路徑
  ```
  新增：src/components/sort-button.tsx
  修改：src/app/page.tsx
  修改：supabase/migrations/0001_init.sql
  ```

- **Schema 補丁**（若有）：`ALTER TABLE` / `CREATE TABLE` / RLS 政策 / RPC
  ```sql
  alter table public.questions add column if not exists ...;
  ```

- **風險評估**：會不會打壞既有功能？有無效能隱憂？

- **建議的 commit message**：
  ```
  feat: 新增問題排序功能
  ```

- **建議的 PR title**：
  ```
  feat: 首頁新增依讚數排序按鈕
  ```

---

### 4️⃣ 💻 動 Code

確認使用者說 OK 後，開始寫程式：

**建分支**：
```bash
git checkout -b feat/<slug-name>
# 或 git checkout -b fix/<slug-name> (視功能性質)
```

**寫程式時遵守**：
- Tailwind v4：用 `bg-linear-to-r` / `bg-linear-to-b`，**不要** `bg-gradient-to-r`
- Schema 改在 `supabase/migrations/0001_init.sql`（保持冪等：`create if not exists` / `drop if exists`）
- 按讚功能一律走 RPC `increment_question_like`，**不要**直接 UPDATE `questions.likes`
- 新增表必加 RLS + Realtime（`alter publication supabase_realtime add table ...`）
- 表單 `onSubmit` 加 `event.preventDefault()` 後再 await Supabase
- `useEffect` 訂閱 Realtime 時務必 cleanup `supabase.removeChannel(channel)`
- 文案保持**中文**，變數名/函式名用英文

---

### 5️⃣ ✅ 驗證（一個失敗就停）

**依序** 跑驗證指令，任何一個失敗**停下來問使用者「修還是改計畫」**：

```bash
pnpm lint
```

若失敗→修 linting error、再跑一次

```bash
pnpm format:check
```

若失敗→跑 `pnpm format`、驗證修改、再跑 lint

```bash
pnpm build
```

若失敗→除錯，**不准加 `--no-verify` 或 `--force`**

---

### 6️⃣ 🚀 Commit + 開 PR

驗證全過，就開始 git 流程：

**Commit**：
```bash
git add .
git commit -m "<type>: <message>"
# 例：git commit -m "feat: 新增問題排序功能"
```

**推遠端 + 開 PR**：
```bash
git push -u origin HEAD
gh pr create --title "feat: 首頁新增依讚數排序按鈕" \
  --body "## Summary

- 新增問題排序按鈕：依讚數、最新
- 關鍵業務邏輯保持不變

## Test plan

- [ ] 按讚功能仍可用
- [ ] 排序按鈕切換正常
- [ ] 新增問題即時更新（Realtime）
"
```

---

## 🚫 規範

- **禁止跳過第 3 步** — 一定要停下來等 OK，不要自作聰明
- **禁止在 main 直接 commit** — 一律開分支
- **驗證失敗禁止 `--no-verify` / `--force`** — 必須真的修好
- **不要 commit** `.env*`、`node_modules/`、`.next/`（`.gitignore` 已擋）
- **Supabase 查詢只允許 SELECT** — 禁止 INSERT/UPDATE/DELETE

---

## ⚠️ 常見卡關

| 問題 | 解法 |
| --- | --- |
| **Supabase MCP 沒裝** | 讓使用者裝 MCP 或貼 schema 出來；也能手查 `0001_init.sql` |
| **`gh` 沒登入** | 提示使用者：`gh auth login` |
| **lint 失敗** | 印出錯誤、問「修程式還是回頭改計畫」 |
| **build 失敗** | 印 error stack，debug 後重新 build；仍失敗就停 |
| **PR 衝突** | 提示 rebase：`git rebase main` → `git push --force-with-lease` |

---

## 💡 提示

- 設計上像教科書：用 bullet、簡短指令、具體範例
- 如果感覺使用者卡住，就把相關規範貼出來
- 上完全流程後，確認 PR 已成功建立再說 done
