# 從 SQLite 遷移至 Neon PostgreSQL，部署 Vercel

> 完成日期：2026-03-19

---

## 為什麼要換掉 SQLite？

原專案使用 SQLite，資料存在本地的 `prisma/dev.db` 檔案。這在本機開發完全沒問題，但**無法部署到 Vercel**，原因如下：

### Vercel 是 Serverless 架構

Vercel 不是傳統的長駐伺服器，而是 **Serverless Functions**：

```
傳統伺服器                     Vercel Serverless
─────────────────────          ─────────────────────────────
一台機器持續運行          vs    每個 request 啟動一個短暫容器
檔案系統永久存在                容器執行完畢即銷毀
dev.db 可以讀寫                 dev.db 根本不存在
```

具體問題：
1. **沒有持久磁碟**：Vercel 每次部署都是全新的容器，不會保留任何本地檔案
2. **執行環境是唯讀的**：Function 無法寫入檔案（SQLite 需要寫入 `.db` 檔案）
3. **多個容器無法共享檔案**：同時有多個 request 時，每個 Function 都是獨立的，無法共用同一個 `.db` 檔案

### 解法：改用網路型資料庫

換成**雲端 PostgreSQL**，資料庫獨立於伺服器之外，任何容器都可以透過網路連線存取：

```
Vercel Function A  ─┐
Vercel Function B  ─┼──→  Neon PostgreSQL（雲端）
Vercel Function C  ─┘
```

### 為什麼選 Neon？

| 資料庫 | 原因 |
|--------|------|
| **Neon**（選用） | 免費方案足夠、支援 Serverless 連線池（Pooler）、PostgreSQL 相容 |
| Supabase | 功能多但稍複雜 |
| Vercel Postgres | 底層也是 Neon，但透過 Vercel 管理 |
| PlanetScale | MySQL-based，免費方案已取消 |

Neon 的 **Pooler 連線**特別適合 Serverless：傳統資料庫連線需要「握手」建立，Serverless 每個 request 都重新連線會很慢且超過連線上限。Pooler 在中間維護一個連線池，讓 Serverless Function 快速借用連線。

---

## 變更摘要

| 項目 | 變更前 | 變更後 |
|------|--------|--------|
| 資料庫 | SQLite（`prisma/dev.db`，本地檔案） | Neon PostgreSQL（雲端） |
| `DATABASE_URL` | `file:./dev.db`（寫死） | 從環境變數讀取 |
| Migration 指令 | `prisma migrate dev` | `prisma migrate deploy`（build 時自動執行） |
| Prisma Client 產生 | 手動 | `postinstall` 自動執行 |

---

## 已完成的變更

### 1. `prisma/schema.prisma`

```prisma
// 變更前
datasource db {
  provider = "sqlite"
  url      = "file:./dev.db"
}

// 變更後
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

### 2. `package.json`

新增 `postinstall`，修改 `build`：

```json
"postinstall": "prisma generate",
"build": "prisma migrate deploy && cross-env NODE_OPTIONS=\"--require ./node-compat.cjs\" next build",
```

- `postinstall`：Vercel 安裝 npm 套件後自動產生 Prisma Client
- `prisma migrate deploy`：部署時自動將 schema 套用至 Neon（production 專用指令，不互動）

### 3. `.env`

新增三個變數（`.env.sample` 已同步更新）：

```env
DATABASE_URL="postgresql://..."   # Neon 連線字串
JWT_SECRET="..."                  # JWT 簽名密鑰（32 bytes hex）
ANTHROPIC_API_KEY=""              # 選填，空白則使用 Mock Provider
```

### 4. 舊 Migration 清除

因 provider 從 `sqlite` 改為 `postgresql`，舊的 `prisma/migrations/` 目錄已刪除並重建：

```
prisma/migrations/
  └─ 20260319070644_init/
       └─ migration.sql    ← 新的 PostgreSQL migration
```

---

## 連線驗證結果

執行 `npx prisma db pull` 結果：

```
Datasource "db": PostgreSQL database "neondb", schema "public"
  at "ep-gentle-grass-a1vdxtqw-pooler.ap-southeast-1.aws.neon.tech"

✔ Introspected 2 models (User, Project) ← 資料表已成功建立
```

---

## 部署 Vercel 前的檢查清單

### Vercel 環境變數設定（Dashboard → Settings → Environment Variables）

| 變數 | 必填 | 說明 |
|------|------|------|
| `DATABASE_URL` | ✅ 必填 | Neon 連線字串（同 `.env`） |
| `JWT_SECRET` | ✅ 必填 | 32 bytes 隨機字串，生產環境不可用預設值 |
| `ANTHROPIC_API_KEY` | 選填 | 若未設定則使用 Mock Provider |

產生 `JWT_SECRET` 的指令：
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

### 部署前自我確認

- [ ] Neon 資料庫已建立且連線測試通過（`prisma db pull` 看到 2 個 model）
- [ ] `.env` 中三個變數皆已填寫
- [ ] Vercel 環境變數已設定（至少 `DATABASE_URL` 和 `JWT_SECRET`）
- [ ] `package.json` 的 `build` script 包含 `prisma migrate deploy`
- [ ] `package.json` 的 `postinstall` 包含 `prisma generate`
- [ ] `.env` 已加入 `.gitignore`（避免 API Key 和 DB 連線字串被 commit）

---

## 部署指令

```bash
# 方法一：Vercel CLI
npm i -g vercel
vercel

# 方法二：連接 GitHub repo，Vercel Dashboard 自動部署
# 每次 push to main 自動觸發
```

---

## Neon 資料庫資訊

- **Region：** ap-southeast-1（新加坡）
- **Database：** neondb
- **Host：** ep-gentle-grass-a1vdxtqw-pooler.ap-southeast-1.aws.neon.tech
- **連線模式：** Pooler（適合 Serverless 環境）

> Neon Pooler 連線是 Vercel Serverless 的推薦方式，避免連線數超過上限。

---

## 注意事項

- `prisma migrate dev` 只用於**本地開發**（會互動詢問）
- `prisma migrate deploy` 用於**生產環境**（自動執行，不互動）
- 本地開發仍連接 Neon，若需要完全離線開發，可另外設定本地 PostgreSQL
