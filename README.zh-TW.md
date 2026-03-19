# UIGen — 繁體中文說明文件

AI 驅動的 React 元件產生器，支援即時預覽。

> 原始課程來源：[Anthropic Claude Code in Action](https://anthropic.skilljar.com/claude-code-in-action/301615)

---

## 目錄

- [專案簡介](#專案簡介)
- [技術堆疊](#技術堆疊)
- [快速開始](#快速開始)
- [專案架構](#專案架構)
- [核心功能深入解析](#核心功能深入解析)
  - [虛擬檔案系統](#虛擬檔案系統)
  - [AI 串流與工具呼叫](#ai-串流與工具呼叫)
  - [JSX 即時轉換與預覽](#jsx-即時轉換與預覽)
  - [認證系統](#認證系統)
  - [資料持久化](#資料持久化)
- [資料流程圖解](#資料流程圖解)
- [目錄結構詳解](#目錄結構詳解)
- [資料庫結構](#資料庫結構)
- [API 端點](#api-端點)
- [環境變數](#環境變數)
- [設計模式與架構決策](#設計模式與架構決策)
- [學習重點整理](#學習重點整理)

---

## 專案簡介

UIGen 是一個讓使用者透過**自然語言**描述 React 元件，Claude AI 便會自動產生程式碼並即時渲染預覽的全端應用程式。

### 核心特色

- 用文字描述，AI 幫你寫 React 元件
- 虛擬檔案系統（所有檔案存在記憶體中，不寫入磁碟）
- 瀏覽器內 JSX 即時編譯與預覽
- Monaco Editor 程式碼編輯器（就是 VS Code 用的那個）
- 使用者可登入儲存專案，或匿名使用

---

## 技術堆疊

| 分類 | 技術 / 套件 |
|------|------------|
| **框架** | Next.js 15.3（App Router）、React 19 |
| **語言** | TypeScript 5 |
| **樣式** | Tailwind CSS v4 |
| **UI 元件** | Radix UI（Dialog、Tabs、Popover 等）、Lucide React（圖示） |
| **程式碼編輯器** | @monaco-editor/react v4.7 |
| **AI 串流** | Vercel AI SDK（`ai` v4.3、`@ai-sdk/anthropic` v1.2）|
| **AI 模型** | Anthropic Claude（claude-sonnet-4-5）|
| **瀏覽器 JSX 編譯** | @babel/standalone v7.27 |
| **資料庫 ORM** | Prisma v6.10（SQLite） |
| **認證** | jose（JWT）、bcrypt（密碼雜湊） |
| **版面管理** | react-resizable-panels |
| **測試** | Vitest、@testing-library/react |
| **建置加速** | Turbopack（Next.js 開發模式） |

---

## 快速開始

### 前置需求

- Node.js 18+
- npm

### 安裝步驟

**1. 設定 Anthropic API Key（選填）**

編輯 `.env` 檔案：

```env
ANTHROPIC_API_KEY=your-api-key-here
```

> 若未設定 API Key，系統會改用**模擬模型（Mock Provider）**，回傳預先定義的靜態程式碼，方便在沒有 API Key 的情況下開發測試。

**2. 安裝依賴套件並初始化資料庫**

```bash
npm run setup
```

此命令依序執行：
- `npm install`（安裝 node_modules）
- `prisma generate`（產生 Prisma Client）
- `prisma migrate dev`（建立 SQLite 資料庫並執行 migration）

**3. 啟動開發伺服器**

```bash
npm run dev
```

開啟瀏覽器前往 [http://localhost:3000](http://localhost:3000)

### 使用方式

1. 點擊「Sign Up」註冊帳號，或選擇匿名模式直接使用
2. 在左側聊天框輸入你想要的 React 元件描述
3. AI 產生程式碼後，右側預覽區會即時顯示渲染結果
4. 切換到「Code」頁籤可以查看、編輯產生的檔案
5. 繼續與 AI 對話來迭代修改元件

---

## 專案架構

### 高層次架構圖

```
┌─────────────────────────────────────────────────────────┐
│                    Next.js App Router                    │
│                                                         │
│  ┌─────────────┐    ┌─────────────────────────────────┐ │
│  │  / (首頁)   │    │     /[projectId] (專案頁)        │ │
│  │  匿名模式   │    │     需要登入驗證                  │ │
│  └─────────────┘    └─────────────────────────────────┘ │
│                                                         │
│  ┌──────────────────── MainContent ──────────────────┐  │
│  │                                                   │  │
│  │  ┌────────────┐  ┌────────────────────────────┐  │  │
│  │  │  ChatPanel │  │      Preview / Code Panel   │  │  │
│  │  │   (35%)    │  │           (65%)             │  │  │
│  │  │            │  │  ┌─────────┐ ┌──────────┐  │  │  │
│  │  │  訊息列表  │  │  │ Preview │ │   Code   │  │  │  │
│  │  │  輸入框    │  │  │  iframe  │ │ FileTree │  │  │  │
│  │  │            │  │  │         │ │  Monaco  │  │  │  │
│  │  └────────────┘  └─────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   Server Layer                          │
│                                                         │
│  ┌────────────────┐  ┌───────────────────────────────┐  │
│  │  Server Actions │  │      /api/chat (Route)        │  │
│  │  - auth        │  │  - 接收訊息 + 虛擬檔案系統     │  │
│  │  - project CRUD│  │  - 呼叫 Claude API（串流）     │  │
│  └────────────────┘  │  - 執行 AI 工具呼叫            │  │
│                      │  - 儲存結果至 DB               │  │
│                      └───────────────────────────────┘  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │              Prisma + SQLite                      │  │
│  │  User、Project（含 VFS 序列化 + 聊天記錄）        │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## 核心功能深入解析

### 虛擬檔案系統

`src/lib/file-system.ts`

這是整個應用最關鍵的設計之一。所有 AI 產生的程式碼**不會寫入真實磁碟**，而是存在一個記憶體內的樹狀結構。

```typescript
// 每個節點的結構
interface FileNode {
  type: "file" | "directory";
  name: string;
  path: string;
  content?: string;        // 只有 file 類型才有
  children?: Map<string, FileNode>; // 只有 directory 類型才有
}
```

**VirtualFileSystem 的主要操作：**

| 方法 | 功能 |
|------|------|
| `createFile(path, content)` | 建立新檔案 |
| `readFile(path)` | 讀取檔案內容 |
| `updateFile(path, content)` | 更新檔案內容 |
| `deleteFile(path)` | 刪除檔案 |
| `createDirectory(path)` | 建立資料夾 |
| `serialize()` | 轉換為 JSON（用於資料庫儲存） |
| `deserialize(json)` | 從 JSON 還原（讀取資料庫後使用） |

**路徑處理細節：**
- 所有路徑統一以 `/` 開頭
- 路徑末尾不留 `/`（除了根目錄）
- 範例：`/src/components/Button.tsx`

**為什麼用虛擬檔案系統？**
1. **安全性**：AI 無法存取真實檔案系統
2. **可攜性**：整個 VFS 可以序列化為 JSON 儲存在資料庫
3. **即時傳輸**：Client 和 Server 之間可以輕易傳遞整個「專案」

---

### AI 串流與工具呼叫

`src/app/api/chat/route.ts` + `src/lib/tools/`

使用 Vercel AI SDK 的 `streamText()` 實現串流生成。

#### 請求流程

```
Client 送出訊息
    ↓
POST /api/chat
    ↓
1. 從 request body 重建 VirtualFileSystem
2. 定義 AI 工具（Tool Definitions）
3. 呼叫 Claude API（streamText）
4. 回傳 DataStreamResponse（SSE 格式）
    ↓
Client 接收串流更新
    ↓
工具呼叫被觸發 → 更新 VFS → 重新渲染預覽
```

#### AI 工具定義

AI 擁有兩個可呼叫的工具：

**工具一：`str_replace_editor`**（`src/lib/tools/str-replace.ts`）

用來**精確修改**已存在的檔案，類似 `git diff` 的方式。

```typescript
// 參數結構（用 Zod 驗證）
{
  command: "view" | "create" | "str_replace" | "insert" | "delete",
  path: string,
  file_text?: string,   // create 時使用
  old_str?: string,     // str_replace 時：要被替換的舊內容
  new_str?: string,     // str_replace 時：新內容
  insert_line?: number, // insert 時：插入在第幾行
  new_str?: string,     // insert 時：插入的內容
}
```

**工具二：`file_manager`**（`src/lib/tools/file-manager.ts`）

用來**管理整個檔案/資料夾**（建立、刪除、重新命名）。

```typescript
{
  operation: "create_file" | "delete_file" | "rename_file" |
             "create_directory" | "delete_directory",
  path: string,
  content?: string,  // create_file 時使用
  newPath?: string,  // rename 時使用
}
```

#### 多步驟生成（maxSteps: 40）

```
使用者：「幫我做一個按鈕元件」
    ↓
AI（步驟 1）：思考設計
    ↓
AI（步驟 2）：呼叫 file_manager 建立 /src/components/Button.tsx
    ↓
AI（步驟 3）：呼叫 str_replace_editor 修改樣式
    ↓
AI（步驟 4）：呼叫 file_manager 建立 /App.jsx（引入 Button）
    ↓
AI（步驟 5）：回覆使用者說明
```

每一步的工具呼叫結果都即時更新 VFS，預覽區同步渲染。

---

### JSX 即時轉換與預覽

`src/components/preview/PreviewFrame.tsx` + `src/lib/transform/jsx-transformer.ts`

瀏覽器原生不懂 JSX，所以需要一個完整的轉換管道：

#### 完整渲染流程

```
VirtualFileSystem 中的 .jsx / .tsx 檔案
    ↓
1. jsx-transformer.ts（用 @babel/standalone 轉換 JSX → 純 JS）
    ↓
2. 解析 import 語句，分類為：
   - 外部套件（如 react, lucide-react）→ 指向 esm.sh CDN
   - 本地模組（如 ./Button, @/components/Button）→ 轉換為 Blob URL
    ↓
3. 建立 Import Map（告訴瀏覽器各模組對應的 URL）
    ↓
4. 組裝完整 HTML（含 Import Map + 模組腳本）
    ↓
5. 注入 iframe 的 srcdoc 屬性
    ↓
瀏覽器在隔離的 iframe 中執行，即時渲染元件
```

#### Import Map 機制

```html
<script type="importmap">
{
  "imports": {
    "react": "https://esm.sh/react@19",
    "react-dom": "https://esm.sh/react-dom@19",
    "react-dom/client": "https://esm.sh/react-dom@19/client",
    "lucide-react": "https://esm.sh/lucide-react",
    "./Button": "blob:http://localhost:3000/abc-123..."
  }
}
</script>
```

#### 錯誤邊界（Error Boundary）

PreviewFrame 在渲染時包裹 Error Boundary，若元件有 runtime 錯誤，會在預覽區顯示錯誤訊息，而不會讓整個應用崩潰。

---

### 認證系統

`src/lib/auth.ts` + `src/actions/index.ts`

採用**無狀態 JWT** 搭配 HTTP-only Cookie。

#### 登入流程

```
使用者輸入 email + password
    ↓
Server Action: signIn()
    ↓
1. 從 DB 查詢 User（by email）
2. bcrypt.compare(inputPassword, hashedPassword)
3. 若驗證成功：jose.SignJWT({ userId, email }) → JWT token
4. 設定 Cookie（HTTP-only, Secure, SameSite=Lax）
5. 7 天後過期
    ↓
Client 收到成功回應，自動導向至專案頁
```

#### 安全設定

| 設定 | 值 | 用途 |
|------|----|------|
| Cookie 名稱 | `auth-token` | 存放 JWT |
| HttpOnly | `true` | 防止 XSS 竊取 Cookie |
| Secure | `true`（production） | 強制 HTTPS |
| SameSite | `Lax` | 防止 CSRF |
| 過期時間 | 7 天 | 平衡安全與便利 |
| 密碼演算法 | bcrypt（10 rounds） | 抵抗暴力破解 |
| JWT 演算法 | HS256 | 標準對稱加密 |

#### 中介層（Middleware）

`src/middleware.ts` 在每個請求到達 Route Handler 前攔截，驗證 JWT 是否有效，保護需要登入的路由。

---

### 資料持久化

`src/lib/contexts/chat-context.tsx` + `src/actions/`

#### Project 的儲存時機

每次 AI 完成一輪回應（`onFinish` callback），若使用者已登入且有 projectId，就自動儲存：

```typescript
onFinish: async () => {
  // 將聊天記錄 + VFS 狀態序列化後儲存至 DB
  await saveProject(projectId, messages, fileSystem.serialize());
}
```

#### 匿名使用者的追蹤

`src/lib/anon-work-tracker.ts` 使用 `sessionStorage` 追蹤匿名使用者的未儲存工作，若使用者離開頁面前尚未登入，會提示他們先儲存。

---

## 資料流程圖解

### 完整的元件生成流程

```
┌──────────────────────────────────────────────────────────┐
│ 使用者在 MessageInput 輸入「幫我做一個計數器元件」        │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│ ChatContext.sendMessage()                                │
│ → 收集當前 VFS 狀態（fileSystem.serialize()）            │
│ → 呼叫 Vercel AI SDK 的 useChat hook                    │
│ → POST /api/chat { messages, files, projectId }          │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│ /api/chat Route Handler                                  │
│ 1. 從 files JSON 重建 VirtualFileSystem                  │
│ 2. 定義 str_replace_editor 和 file_manager 工具          │
│ 3. streamText({ model: claude, tools, messages })        │
│ 4. 回傳 DataStreamResponse（SSE 串流）                   │
└────────────────────────────┬─────────────────────────────┘
                             │
                        ╔════╧════╗
                        ║ 串流中  ║
                        ╚════╤════╝
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         文字片段        工具呼叫       完成訊號
         即時顯示      file_manager    onFinish
         在聊天區      建立 Counter.tsx  儲存至 DB
              │              │
              ▼              ▼
         訊息更新       VFS 更新
                            │
                            ▼
              ┌─────────────────────────────┐
              │ FileSystemContext 更新        │
              │ → PreviewFrame 重建 HTML      │
              │ → iframe srcdoc 更新          │
              │ → 瀏覽器渲染新的計數器元件    │
              └─────────────────────────────┘
```

---

## 目錄結構詳解

```
next-uigen/
├── src/
│   ├── app/                          # Next.js App Router
│   │   ├── layout.tsx                # 根 Layout（全域字體、metadata）
│   │   ├── page.tsx                  # 首頁：判斷登入狀態、顯示匿名或登入介面
│   │   ├── main-content.tsx          # 主介面：三欄可調整版面
│   │   ├── [projectId]/
│   │   │   └── page.tsx              # 動態路由：載入特定專案
│   │   └── api/
│   │       └── chat/
│   │           └── route.ts          # 核心 API：AI 串流端點
│   │
│   ├── actions/                      # Next.js Server Actions（伺服器端函式）
│   │   ├── index.ts                  # 認證相關：signUp、signIn、signOut、getUser
│   │   ├── create-project.ts         # 建立新專案
│   │   ├── get-project.ts            # 取得單一專案
│   │   └── get-projects.ts           # 取得使用者所有專案
│   │
│   ├── components/                   # React 元件
│   │   ├── auth/
│   │   │   ├── AuthDialog.tsx        # 登入/註冊 Modal
│   │   │   ├── SignInForm.tsx        # 登入表單
│   │   │   └── SignUpForm.tsx        # 註冊表單
│   │   ├── chat/
│   │   │   ├── ChatInterface.tsx     # 聊天介面容器
│   │   │   ├── MessageList.tsx       # 訊息列表（支援 Markdown）
│   │   │   ├── MessageInput.tsx      # 輸入框（支援 Enter 送出）
│   │   │   └── MarkdownRenderer.tsx  # AI 回覆的 Markdown 渲染
│   │   ├── editor/
│   │   │   ├── CodeEditor.tsx        # Monaco Editor 整合（自動偵測語言）
│   │   │   └── FileTree.tsx          # 檔案樹狀視圖（可折疊資料夾）
│   │   ├── preview/
│   │   │   └── PreviewFrame.tsx      # 即時預覽 iframe
│   │   ├── ui/                       # Radix UI 基礎元件包裝
│   │   └── HeaderActions.tsx         # 頂部操作列（使用者選單）
│   │
│   ├── lib/                          # 核心函式庫
│   │   ├── file-system.ts            # 虛擬檔案系統實作（重點！）
│   │   ├── auth.ts                   # JWT 建立、驗證、Cookie 管理
│   │   ├── provider.ts               # AI 模型選擇（Claude 或 Mock）
│   │   ├── prisma.ts                 # Prisma Client 單例
│   │   ├── utils.ts                  # 通用工具函式（cn 等）
│   │   ├── anon-work-tracker.ts      # 匿名使用者工作追蹤
│   │   ├── contexts/
│   │   │   ├── chat-context.tsx      # Chat 狀態管理（useChat hook 封裝）
│   │   │   └── file-system-context.tsx # VFS 狀態管理
│   │   ├── tools/
│   │   │   ├── str-replace.ts        # AI 工具：精確文字替換
│   │   │   └── file-manager.ts       # AI 工具：檔案管理操作
│   │   ├── transform/
│   │   │   └── jsx-transformer.ts    # Babel JSX→JS 轉換 + import 解析
│   │   └── prompts/
│   │       └── generation.tsx        # Claude 的系統提示詞（System Prompt）
│   │
│   ├── hooks/
│   │   └── use-auth.ts               # 認證狀態 Hook
│   │
│   └── middleware.ts                 # JWT 路由保護中介層
│
├── prisma/
│   ├── schema.prisma                 # 資料庫結構定義
│   └── dev.db                        # SQLite 資料庫檔案
│
├── .env                              # 環境變數（API Key 等）
├── next.config.ts                    # Next.js 設定
├── tsconfig.json                     # TypeScript 設定（含 @/ 路徑別名）
├── vitest.config.mts                 # 測試設定
├── postcss.config.mjs                # Tailwind CSS PostCSS 設定
└── node-compat.cjs                   # Turbopack 的 Node.js 相容層
```

---

## 資料庫結構

`prisma/schema.prisma`

```prisma
model User {
  id        String    @id @default(cuid())
  email     String    @unique
  password  String                        // bcrypt 雜湊過的密碼
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  projects  Project[]
}

model Project {
  id        String   @id @default(cuid())
  name      String
  userId    String?                       // null = 匿名專案
  messages  String                        // JSON 字串：聊天記錄陣列
  data      String                        // JSON 字串：VFS 序列化結果
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  user      User?    @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

**設計重點：**
- `userId` 可以是 `null`，允許匿名專案
- `messages` 和 `data` 都是 JSON 字串（SQLite 不支援原生 JSON 類型）
- 刪除 User 時自動刪除所有 Projects（Cascade）
- 使用 `cuid()` 作為 ID（比 UUID 更短且 URL 友善）

---

## API 端點

### POST /api/chat

**用途：** 串流式 AI 元件生成

**請求 Body：**
```typescript
{
  messages: Message[],   // 完整的對話歷史
  files: object,         // VFS 序列化 JSON
  projectId?: string     // 若要儲存則提供
}
```

**回應：** Server-Sent Events（SSE）串流，包含：
- 文字片段（AI 回覆的文字）
- 工具呼叫指令（建立/修改檔案）
- 完成訊號

**最長執行時間：** 120 秒（`maxDuration: 120`）

**執行流程：**
1. 從 `files` 重建 `VirtualFileSystem`
2. 建立工具定義（工具執行時直接修改 VFS）
3. 呼叫 `streamText()` 與 Claude 對話（最多 40 步驟）
4. 若有 `projectId` 且使用者已驗證，在 `onFinish` 時儲存至 DB

---

## 環境變數

| 變數名 | 必填 | 說明 |
|--------|------|------|
| `ANTHROPIC_API_KEY` | 否 | 若未設定則使用 Mock Provider |
| `JWT_SECRET` | 否 | JWT 簽名密鑰（預設有開發用值，生產環境務必設定） |

---

## 設計模式與架構決策

### 1. Context + Hook 狀態管理

不使用 Redux 或 Zustand，而是用 React 原生的 Context API：

```
ChatProvider（聊天狀態）
    └─ useChat()：取得訊息、送出訊息、載入狀態

FileSystemProvider（VFS 狀態）
    └─ useFileSystem()：讀取/修改虛擬檔案系統
```

**優點：** 無需額外依賴，與 Next.js Server Actions 整合自然。

### 2. Server Actions 取代傳統 API Route

所有 CRUD 操作（認證、專案管理）使用 Next.js 14+ 的 Server Actions：

```typescript
// actions/index.ts
"use server";

export async function signIn(email: string, password: string) {
  // 這段程式碼在伺服器執行，Client 可以直接呼叫
}
```

**優點：** 不需要手動建立 API 路由、自動型別安全、可直接從元件呼叫。

### 3. Mock Provider 開發模式

`src/lib/provider.ts` 根據環境變數決定使用真實 Claude 還是 Mock：

```typescript
export function getLanguageModel() {
  if (!process.env.ANTHROPIC_API_KEY) {
    return new MockLanguageModel(); // 回傳靜態程式碼
  }
  return anthropic("claude-sonnet-4-5");
}
```

**優點：** 開發時不需 API Key，節省費用。Mock 也模擬了多步驟生成流程。

### 4. Blob URL 模組系統

本地檔案轉換後不寫入磁碟，而是用 `URL.createObjectURL()` 建立臨時 URL：

```javascript
// JSX 轉換後的 JS 內容 → Blob → Blob URL
const blob = new Blob([transformedCode], { type: "text/javascript" });
const blobUrl = URL.createObjectURL(blob);
// 在 Import Map 中映射："./Button" → blobUrl
```

**優點：** 完全在瀏覽器內完成，無需後端介入。

### 5. 系統提示詞工程

`src/lib/prompts/generation.tsx` 包含詳細的系統提示詞，告訴 Claude：
- 如何組織檔案結構
- 什麼時候用哪個工具
- 如何處理 Tailwind CSS
- 如何建立 App.jsx 入口點

---

## 學習重點整理

若你想透過這個專案學習現代全端開發，以下是核心概念：

### Next.js 相關
- **App Router**：`page.tsx`、`layout.tsx`、動態路由 `[id]`
- **Server Actions**：`"use server"` 指令、如何從 Client 呼叫
- **Route Handlers**：`route.ts` 建立 API 端點
- **Middleware**：`middleware.ts` 攔截請求做路由保護

### React 相關
- **Context API**：`createContext`、`useContext`、Provider 模式
- **自訂 Hook**：封裝複雜邏輯（`useAuth`、`useFileSystem`、`useChat`）
- **Streaming UI**：如何即時更新 UI 對應串流回應

### AI SDK 相關
- **`streamText()`**：呼叫 LLM 取得串流文字
- **工具定義**：用 Zod 定義 AI 可呼叫的工具
- **`useChat()` hook**：管理對話狀態與串流接收

### 安全性相關
- **JWT**：無狀態認證的實作方式
- **bcrypt**：密碼安全雜湊
- **HTTP-only Cookie**：防止 XSS 的 Cookie 策略
- **虛擬檔案系統**：隔離 AI 存取實際檔案系統

### 進階技術
- **Babel Standalone**：在瀏覽器端即時編譯 JSX
- **Import Map**：現代瀏覽器的模組解析機制
- **esm.sh**：CDN 提供的 ESM 模組服務
- **Blob URL**：動態建立可被 import 的 JS 模組

---

*本文件由 Claude 根據程式碼自動分析生成，最後更新：2026-03-19*
