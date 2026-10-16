# 代數資料型態與函數式領域模型法典 (Algebraic Domain Modeling Manifesto)

本法典定調了軟體核心層（Core Domain）如何利用**代數資料型態 (Algebraic Data Types, ADT)** 實施最純粹的「資料與行為解耦 (Data-Code Separation)」，以換取架構面對業務演進與多變讀寫需求時的最高防禦力與絕對穩定性。

---

## 1. 為什麼要使用 ADT？

傳統物件導向（OO-DDD）將行為與資料硬塞進同一個物件，本質上是為了解決 20 年前技術框架的限制。當面對現代多變的業務狀態與高頻變更時，傳統做法會面臨兩大毀滅性痛點：
* **欄位爆炸與狀態失控**：為了在一個類別裡表達多個狀態，塞滿了可空（Nullable）欄位與防禦型的 `if-else`。編譯器無法在語法上約束狀態與資料的關聯，只能依賴人肉 Code Review，導致髒資料極易外洩。
* **類別發散與代碼重複**：為了追求狀態封裝，每多一個業務狀態就去寫一個全新的類別，並瘋狂複製重複的基礎欄位，導致類別數量產生**「指數級爆炸」**。

**使用 ADT 的核心理由**，是利用數學上的對稱性，將商業規則直接上升為**型態系統（Type System）層級的「物理规律」**。它讓資料長得無比純淨，從根本上讓不合法的狀態組合在編譯時期就絕對不可能誕生。

---

## 2. ADT 解決了什麼問題？

ADT 透過 **「積型態 (Product Types / AND 關係)」** 與 **「和型態 (Sum Types / OR 關係)」** 的完美組合，一舉消滅了軟體工程中的三大頑疾：

### 解決問題一：使不合法的資料「無法被表達 (Make Illegal States Unrepresentable)」
* **痛點**：未付款的訂單卻存有 `paidAt`（付款時間），或者已付款的訂單漏掉了 `transactionId`。
* **ADT 解法**：利用「和型態」（如 TypeScript 的 Union），將不同狀態分組宣告。當訂單處於 `PENDING_PAYMENT` 型態時，資料結構上**根本不存在** `paidAt` 欄位；當它跳轉為 `PAID` 型態時，語法上**強制必須包含** `transactionId`。

### 解決問題二：消滅「指數級類別爆炸」
* **痛點**：每增加一個商業狀態，就要寫一個完整的類別去重複宣告 ID、金額、客戶等基底資料。
* **ADT 解法**：將共同的物理基礎欄位只寫一次（積型態），狀態專屬的特殊數據利用型態組合動態附加。不論業務狀態如何橫向擴展，全系統的類別與程式碼增長永遠是**線性（Linear）**的。

### 解決問題三：消滅主流程中的防禦型 `if-else`
* **痛點**：Domain Service 內部塞滿了用於安檢資料合法性的重複判斷。
* **ADT 解法**：因為能成功傳入主流程的型態保證 100% 合法，Domain Service 只需要利用現代語法的**「模式匹配 (Pattern Matching / match)」**進行純粹的策略分流。少寫一個狀態分支，編譯器就會直接報錯，拒絕編譯。

---

## 3. 導入 ADT 概念後，Entity & Value Object 的職責轉變

導入 ADT 之後，核心資料體被**徹底去行為化（Demeaned）**，轉變為 100% 純粹、不可變的「唯讀資料結構 (Immutable Records)」：

* **Entity（實體）的新職責**：
  * **宇宙物理規律的剛性守護者**：只准持有「絕對不可變（Readonly）」的欄位與狀態標籤（TS 使用 `readonly`，PHP 使用 `readonly class`）。
  * **守護 Invariants（不變量）**：內部的 Smart Constructor（工廠方法）或內部修改方法，只負責守護**「違反即違反現實」**的剛性紅線（如金額不能為負數、報名人數不能超出場地容量）。
  * **拒絕持有商務策略**：Entity 肚子裡不再寫任何隨著業務調整而具備彈性、需要修改的公式與規則。
* **Value Object（值物件）的新職責**：
  * **純粹的局部狀態描述與數學算式**：它沒有唯一識別碼（Unique ID），其身分由肚子裡的所有屬性共同決定。內部只允許包含純粹的、無副作用的數值運算（如 `Money.add()`）或格式拼接（如 `getFullName()`）。

---

## 4. 核心層如何管理 Readonly 的商務概念？

在技術上，**商業概念（如 `PENDING_PAYMENT` 語意）的主權必須 100% 中央集權於核心層（Core Domain）的型態系統。** 

不論是寫入管線（Command）還是單純拉報表的讀取管線（Query），全系統一律使用核心層定調的強型態（Type/Enum）作為通用語意貨幣。技術層（Repository）可以為了效能去下最暴力的 SQL 或進行多表聯集（Join），但它拉上來的底層技術殘渣，**在吐出去的最後一刻，必須將原始欄位強制翻譯、對齊到核心層定義的業務型態中。** 這叫「技術分流，語意集權」。

---

## 5. 資料體與 Repository 之間的互動關係與 Input/Output

在您的鋼鐵務實架構中，Repository 的定位是**「技術副作用與髒 Schema 的斷頭台（污水處理廠）」**。它與資料體之間的拋接關係极其絲滑且徹底解耦：

### 📥 Input 流程（從 DB 到系統：髒細節對齊純淨概念）
1. Repository 負責執行底層 IO，從骯髒、混亂的 Legacy Database 中撈出原始資料（Dirty DB Schema）。
2. 在 Repository 內部**直接進行髒欄位翻譯**（例如把 `db_flag_01 === 9` 翻譯成核心層唯一的 `'PENDING_PAYMENT'`）。
3. 在 Repository 內部將清洗乾淨的資料塞進 Smart Constructor。此時，Invariants（不變量）在核心邊界被當場剛性驗證。
4. **輸出結果**：**Repository 只准輸出 100% 純淨、符合人類商業語言的強型態 Entity / 資料體。** 

### 📤 Output 流程（從系統到外部：唯讀實體即是頂級 DTO）
1. 核心層（Domain Service）算完商業策略後，會產出全新的唯讀 Entity。
2. **Entity 直接充當 Output 載體**：因為資料體已經被徹底去行為化，肚子裡只有純粹、乾淨、不可變的資料，因此**它本身就具備了最完美的 DTO 特性**。它當然可以直接當作 Output 拋給最外層的 Endpoint 直接輸出（json_encode 吐給前端）。
3. **唯一的剛性防線（資安過濾）**：只有當 Entity 內部包含敏感的業務安全欄位（如 `passwordHash`）時，才需要在技術層或外層利用簡單的過濾器（如 TS 的 `Omit`）將欄位剔除。這叫資安防禦，絕非分層教條。

---

### 🛠️ TypeScript 完整整體性代碼示範

```typescript
// ==========================================
// 1. 核心層 (Core Domain) —— 擁有最高語意主權
// ==========================================

// 積型態 (AND)：全系統只寫這一次的共同基礎欄位
type OrderBase = {
  readonly id: string;
  readonly total: number;
  readonly customerId: string;
};

// 狀態特定資料
type UnpaidData = { readonly paymentDeadline: Date };
type PaidData   = { readonly paidAt: Date; readonly transactionId: string };

// 和型態 (Sum Type / OR)：用編譯器卡死合法的現實宇宙規律
export type Order = 
  | { readonly status: 'PENDING_PAYMENT' } & OrderBase & UnpaidData
  | { readonly status: 'PAID' }            & OrderBase & PaidData;


// ==========================================
// 2. 技術邊界層 (Infrastructure / Repository) —— 污水處理廠
// ==========================================
export class MySqlOrderRepository {
  constructor(private db: any) {}

  // Input 管線：撈出髒 Schema，內部清洗，【只輸出】核心強型態 Entity
  async findById(id: string): Promise<Order> {
    const dirtyRow = await this.db.query("SELECT * FROM old_tbl_ord_x99 WHERE id = ?", [id]);

    const amount = Number(dirtyRow.dec_val_3);
    if (amount < 0) throw new Error("物理不變量崩潰：金額不能為負數"); // 守護 Invariants

    // 將髒細節翻譯、對齊核心層的絕對概念
    if (dirtyRow.db_code === 99) {
      return {
        status: 'PAID',
        id: dirtyRow.id,
        total: amount,
        customerId: dirtyRow.cust_id,
        paidAt: new Date(dirtyRow.str_dt_paid),
        transactionId: dirtyRow.gateway_ref_no
      }; // 輸出完美的 Entity
    }

    return {
      status: 'PENDING_PAYMENT',
      id: dirtyRow.id,
      total: amount,
      customerId: dirtyRow.cust_id,
      paymentDeadline: new Date(dirtyRow.deadline_timestamp)
    };
  }
}

// ==========================================
// 3. 最外通訊層 (Endpoints) —— 框架的奴隸
// ==========================================
class HttpOrderController {
  constructor(private orderRepo: MySqlOrderRepository) {}

  async handle(req: any, res: any) {
    // 呼叫 Repo 拿到高階強型態 Entity
    const order = await this.orderRepo.findById(req.params.id);
    
    // ➔ Output 流程：開除多餘的 ResponseDTO 教條！
    // 乾淨、唯讀的 Entity 直接充當 DTO 吐給前端，代碼達到最極致的精簡與整體性。
    return res.json(order); 
  }
}
```
