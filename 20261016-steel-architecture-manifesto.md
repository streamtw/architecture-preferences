Steel Architecture Manifesto

# 鋼鐵務實架構：軟體分層權責宣言 (Steel Architecture Manifesto)

本文件為 **TypeScript & PHP 現代專案** 的最高架構指引法典。旨在完全解耦「不變的商業本質」與「多變的技術細節」，確保系統在面對業務規則變更與技術底層更換時，具備絕對的防禦力與以逸待勞的穩定性。

## 🎯 核心設計格言

1. **資料與行為徹底解耦**：資料體（Entity / VO）只代表系統的「狀態」；Domain Service 才是商業邏輯的「唯一合法主體」。
2. **本質與外觀明確切分**：`Entity / Value Object` 服務商業邏輯；`DTO` 服務具體的 Endpoint（技術通訊協定）。
3. **副作用的絕對壟斷**：所有 IO 與副作用（Side-effects）死死鎖在 `Repository 實作` 內部。Repository 只向內層暴露純業務語意的**「策略選項 (Strategy Options)」**，由呼叫者（Endpoint 或 Use Case）像翻牌一樣進行高階選擇。
4. **不變量與商務策略的剛性分工**：
   * **Invariants (不變量)** = 定義資料的合法性，**違反即違反現實**（如金額為負、人數超出場地容量）。必須 100% 鎖在核心資料體內部，一出生就銬死，確保垃圾資料絕不外洩。
   * **商務策略 (Business Strategy)** = 具備彈性且不違反現實（如滿千打九折、VIP 免運）。100% 收攏在 Domain Service，藉此消滅主流程中多餘的防禦型 `if-else`。

---

## 視覺化資料拋接流向圖

[ 最外通訊層：Endpoint (如 HTTP Controller) ]  ── 負責解析通訊協定
│
▼ 接收外部請求資料 (Json/CLI 參數)
[ 技術邊界層：DTO ]  ── 服務具體 Endpoint (形狀隨前端 UI/API 契約震盪)
│
▼ 1. 通過 Smart Constructor 進行基礎格式過濾
▼ 2. 轉交核心資料體進行 Invariants 剛性校驗
[ 核心層：Entity / Value Object ]  ── 服務商業邏輯 (現代不可變資料結構)
│   ※ 守護 Invariants：不合法的垃圾資料在此玉石俱焚，絕不放行
▼ 3. 丟入 100% 純淨、零低階 if-else 的商業智慧管線 (純函數)
[ 核心層：Domain Service ]  ── 100% 商務策略唯一的家 (無副作用、無 IO)
│   ※ 專注實踐高階商務策略，免去防禦型代碼
▼ 4. 算出了全新的唯讀領域結果 (New Entity)
[ 技術邊界層：Repository 實作 ]  ── 副作用的斷頭台 (壟斷所有 DB Query / API 實作)
│   ※ 依據呼叫者翻牌指定的「策略選項 (Strategy Options)」執行具體副作用
▼ 5. 轉換為外吐的 DTO，保護敏感欄位
[ 最外通訊層：Response DTO ──> 輸出 JSON / 網頁 ]

---

## 🛠️ 四層架構職責施工圖

### 1. 核心層：Domain Service & 資料體 (Core Domain)
這是整個系統的「業務大腦」，追求最極致的純淨度。在現代函數式架構（fDDD）下，它是**完全無副作用、無技術細節、無 IO 的代碼與資料。**

#### 📥 該放什麼進來
* **Domain Service**：
  * 100% 的**「商務策略」**、高階業務演算法、跨實體編排與決策流程。
  * 必須是**純函數 (Pure Function)**：輸入領域物件，套用規則，輸出全新領域物件。
  * 這裡的程式碼應該與您的業務規格書（Spec）達到 1 對 1 的驚人可讀性，**零低階防禦型代碼**。
* **Entity / Value Object**：
  * **純粹、不可變的資料結構 (Immutable Record)**。在程式碼層級必須是絕對唯讀的（TS 使用 `readonly`，PHP 使用 `readonly class`）。
  * **Smart Constructor (不變量的守門人)**：私有化建構子，強迫外部只能透過帶有 **Invariants 剛性校驗** 的工廠方法來建立或更新物件。
  * 內部修改方法（如 `addAttendee()`）必須親自檢查不變量（如人數是否超出場地容量），一旦違反現實，直接拋出 Exception 或回傳 Failure Result，拒絕誕生。
  * **純資料投影的 Getter**：只能包含無關業務的資料讀取或基礎格式化（如 `getFullName()`）。

#### ❌ 絕對不放什麼進來
* ❌ 絕對不准有任何資料庫、快取、HTTP 套件、或第三方 API 的依賴與宣告。
* ❌ 絕對不准在資料體內部寫任何「商務策略」的邏輯判斷（例如 `isVIP()`、折扣計算這種隨業務調整而具備彈性的公式，那是 Domain Service 的職責）。
* ❌ 絕對不准有任何會偷偷修改物件狀態的副作用方法（嚴禁任何形式的 Public/Private Setter）。
* ❌ 核心 Domain Service 絕對不准經手或撰寫任何 Invariants 的防禦型 `if-else`（這些垃圾安檢工作必須在進入 Service 前就被資料體完全擋掉）。

---

### 2. 中間橋樑層：應用程式案例層 (Use Case / Application Service)
**視專案規模可有可無**。它的本質是一個**「無情的、貧血的調度員 (Orchestrator)」**，不包含任何商業智慧，僅作為技術與核心之間的防彈緩衝牆。

#### 📥 該放什麼進來
* 負責執行純粹的技術管線流程控制：`1. 接 DTO` → `2. 呼叫 Repo 介面撈資料` → `3. 轉成 Entity` → `4. 塞入 Domain Service 計算` → `5. 呼叫 Repo 存檔`。
* 依據業務場景，向 Repository 介面傳遞純業務語言的**「策略選項（如 STANDARD, FAST_SPEED）」**，行使高階翻牌權利。
* 定義副作用的抽象合約（Ports / Interfaces），表達「系統需要什麼功能」，但不指定技術。

#### ❌ 絕對不放什麼進來
* ❌ 絕對不准包含任何具體的技術實實現代碼（沒有 SQL、沒有 HttpClient 呼叫、沒有框架元件）。
* ❌ 絕對不准寫任何商業邏輯。如果在這個層級看見業務規則（不變量或商務策略）的 `if-else`，代表商業邏輯外洩，必須立刻重構回核心層。

---

### 3. 技術邊界層：資料持久與傳輸層 (Repository & DTO)
這是系統的**技術邊界與副作用的斷頭台**。最髒、最累的底層技術細節全部要在這一層被完全消化並封裝。

#### 📥 該放什麼進來
* **Repository 實作類別**：
  * 100% 壟斷所有副作用。包含 SQL 語句、ORM 呼叫（Laravel Eloquent, TS Prisma）、Redis 快取讀寫、外部 gRPC/HTTP API 呼叫。
  * 內部負責解析呼叫者傳入的 `Strategy Option`（策略選項），並將其路由、指定給具體的技術驅動器（Drivers）。
* **DTO (Data Transfer Object)**：
  * 純粹用來服務具體的 Endpoint（如 HTTP 請求、Socket 封包、CLI 參數）。
  * 它的結構與外觀完全聽命於「外部通訊協定與 UI 畫面」，作為核心 Entity 的防彈衣，負責吸收所有來自前端改版的震盪波。

#### ❌ 絕對不放什麼進來
* ❌ 核心層的 Entity 絕對不准直接兼任 DTO 吐給最外層。
* ❌ 內部的具體技術細節（如 `PDO`、`MongoClient`、SQL 字串、特定 ORM 物件）絕對不准外洩出這一層。
* ❌ 絕對不准在這一層去定義或干涉任何 Invariants 或商務策略。這一層是純粹的奴隸，只負責資料的搬運與持久化副作用。

---

### 4. 最外通訊層：入口點層 (Endpoints)
這是系統的最外殼，也就是**技術框架（如 Laravel Controller、NestJS Resolver）的奴隸**。它唯一的職責就是與外部世界進行通訊對接。

#### 📥 該放什麼進來
* HTTP Controllers、GraphQL Resolvers、WebSocket Event Handlers、CLI Commands、排程 CronJobs。
* 唯二職責：
  1. **解析請求**：把外部的網路協定封包（HTTP JSON、CLI 參數）拆解，轉成中層/內層要的 `DTO`。
  2. **轉換回傳**：把系統內回傳的 DTO，包裝成符合該協定的規格（如 `200 OK JSON` 或是終端機的標準輸出）。

#### ❌ 絕對不放什麼進來
* ❌ 絕對不准自己去實作任何調度順序與 DB 寫入。
* ❌ 絕對不准觸碰任何與 Invariants 或是商務策略相關的業務邏輯。

---

## ⚖️ 實戰派權衡指南：何時可以開除 Use Case 層？

分層是為了服務「業務共享的規模」，而不是為了教條。在「依賴反轉（DIP）」與「Repository 壟斷副作用」的前提下，您可以自由選擇以下兩種戰術流派：

### 🚨 選擇流派 A：Lean Architecture (精實流派 —— 直接開除 Use Case)
* **適用場景**：中小型專案，或特定業務在未來明確「只會有一個 HTTP 入口點」。
* **實作方式**：刪除 Use Case 層。讓 **Endpoint (Controller) 直接呼叫 Repository 介面與 Domain Service**。
* **自我檢視**：只要 Endpoint 內部沒有 DB Query（完全呼叫 Repo 介面）、沒有業務 if-else（完全呼叫 Domain Service），代碼依然無比乾淨，且能省下大量拋接檔案的開發成本。

### 🛡️ 選擇流派 B：Clean Architecture (嚴謹流派 —— 保留 Use Case)
* **適用場景**：大型企業級系統，或特定業務流程明確會「同時被 HTTP API、CLI 管理工具、定時排程、MQ 監聽器」等多個入口點觸發。
* **實作方式**：保留 Use Case。把「呼叫步驟的順序」與「策略選項的選擇邏輯」封裝在 Use Case 內。
* **核心價值**：確保當下單流程、步驟順序調整時，您不需要去修改 4 個風格迥異的 Endpoint 類別，100% 實現流程在多入口間的完美共享。

