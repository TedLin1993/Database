# Concurrency Control Theory (ACID)

# 引入

回顧本課程的路線圖：

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled.png)

在前面的課程中介紹了 DBMS 的主要模塊及架構，自底向上依次是 Disk Manager、Buffer Pool Manager、Access Methods、Operator Execution 及 Query Planning。但數據庫要解決的問題並不僅僅停留在功能的實現上，它還需要具備：

- 滿足多個用戶同時讀寫數據，即 Concurrency Control，如：
    - 兩個用戶同時寫入同一條記錄
- 面對故障，如當機，能恢復到之前的狀態，即 Recovery，如：
    - 你在銀行系統轉賬時，轉到一半忽然停電

Concurrency Control 與 Recovery 都是 DBMSs 的重要特性，它們滲透在 DBMS 的每個主要模塊中。而二者的基礎都是具備 ACID 特性的 Transactions，因此本節的討論從 Transactions 開始。

# Transactions

> A transaction is the execution of a sequence of one or more operations (e.g., SQL queries) on a shared database to perform some higher-level function.
> 

用一句白話說，transaction 是 DBMS 狀態變化的基本單位，一個 transaction 只能有執行和未執行兩個狀態，不存在部分執行的狀態。

一個典型的 transaction 實例就是：從 A 賬戶轉賬 100 元至 B 賬戶：

- 檢查 A 的賬戶余額是否超過 100 元
- 從 A 賬戶中扣除 100 元
- 往 B 賬戶中增加 100 元

## Strawman System

---

系統中只存在一個線程負責執行 transaction：

- 任何時刻只能有一個 transaction 被執行且每個 transaction 開始執行就必須執行完畢才輪到下一個
- 在 transaction 啟動前，將整個 database 數據復制到新文件中，並在新文件上執行改動
    - 如果 transaction 執行成功，就用新文件覆蓋原文件
    - 如果 transaction 執行失敗，則刪除新文件即可

Strawman System 的缺點很明顯，無法利用多核計算能力並行地執行相互獨立的多個 transactions，從而提高 CPU 利用率、吞吐量，減少用戶的響應時間，但其難度也是顯而易見的，獲得multi thread好處的同時必須保證數據庫的正確性和 transactions 之間的公平性。

顯然我們無法讓 transactions 的執行過程在時間線上任意重疊，因為這可能導致數據的永久不一致。於是我們需要一套標准來定義數據的正確性。

## Formal Definitions

---

- Database: A **fixed** set of named data objects (e.g., A, B, C, …)
- Transaction: A sequence of read and write operations (e.g., R(A), W(B), …)
    - R: read, W: write
- Transaction in SQL
    - txn開始於**BEGIN** command
    - txn結束於**COMMIT**或**ABORT**
        - 若是commit，DBMS可能儲存或abort這個txn造成的改變
        - 若是abort，roll back到這個txn之前的狀態
- transaction 的正確性標准稱為 ACID：
    - **Atomicity(原子性)**：All actions in the transaction happen, or none happen
        - “all or nothing”
    - **Consistency(一致性)** : If each transaction is consistent and the database is consistent at the beginning of the transaction, then the database is guaranteed to be consistent when the transaction completes.
        - “it looks correct to me”
    - **Isolation(隔離性)** : The execution of one transaction is isolated from that of other transactions.
        - “as if alone”
    - **Durability(持續性)** : If a transaction commits, then its effects on the database persist.
        - “survive failures”

# Atomicity

---

Transaction 執行只有兩種結果：

- 在完成所有操作後 Commit
- 在完成部分操作後主動或被動 Abort

DBMS 需要保證 transaction 的原子性，即在使用者的眼裡，transaction 的所有操作要麼都被執行，要麼都未被執行。

回到之前的轉帳問題，如果 A 帳戶轉 100 元到 B 帳戶的過程中忽然停電，當供電恢復時，A、B 賬戶的正確狀態應當是怎樣的？— 回退到轉賬前的狀態。如何保證？

## Approach #1: Logging

- DBMS 在日誌中按順序記錄所有 actions 信息，所以系統出錯時可以 undo 所有 aborted transactions 的 actions
- undo records同時記錄在memory和disk
- 可以當做飛機的黑盒子，讓人知道DB crash的那一瞬間發生了甚麼事
- 出於審計和效率的原因，幾乎所有現代系統都使用這種方式。

## Approach #2: Shadow Paging

- transaction 執行前，DBMS 複製相關 pages，讓 transactions 修改這些複製的數據，僅當 transaction Commit 後，pointer指向修改的pages，這些 pages 才對外部可見。
- 這是較古老的DB技術，起源於1970s時IBM的system R
- 現代系統中採用該方式的不多，使用的DB包括 CouchDB, LMDB(open LDAP)。

# Consistency

---

The “world” represented by the database is consistent (e.g., correct). All questions (i.e., queries) that the application asks about the data will return correct results.

## Database Consistency:

- 資料庫必須準確地代表現實實體，並遵循integrity constraints
    - Integrity Constraints: 完整限制，寫入的資料必須合理，例如資料型別必須正確、foreign key必須和另一個table相關聯、unique key必須唯一等
- Transactions in the future see the effects of transactions committed in the past inside of the database.
- 在single node DB一定會database consistency，但distribution DB則無法保證

## Transaction Consistency:

- If the database is consistent before the transaction starts, it will also be consistent after.
- Ensuring transaction consistency is the application’s responsibility.
    - 例如某個user應該沒有權限access這個DB，但他卻成功access了，這是程式要負責的問題，不是DB要擋

# Isolation

---

用戶提交 transactions，不同 transactions 執行過程應當互相隔離，互不影響，每個 transaction 都認為只有自己在執行。

但對於 DBMS 來說，為了提高各方面性能，需要恰如其分地向不同的 transactions 分配計算資源，使得執行又快又正確。這裡的 “恰如其分” 的定義用行話來說，就是 **concurrency control protocol**，即 DBMS 如何認定多個 transactions 的重疊執行方式是正確的。

總體上看，有兩種 protocols：

- Pessimistic protocol：不讓問題出現，將問題扼殺在搖籃之中
- Optimistic protocol：假設問題很罕見，一旦問題出現了再行處理

舉例如下：假設 A, B 帳戶各有 1000 元，當下有兩個 transactions：

- T1：從 A 帳戶轉帳 100 元到 B 帳戶
- T2：給 A、B 賬戶存款增加 6% 的利息

```sql
// T1
BEGIN
A = A - 100
B = B + 100
COMMIT

BEGIN
A = A * 1.06
B = B * 1.06
COMMIT
```

那麼 T1、T2 發生後，可能的合理結果應該是怎樣的？

盡管有很多種可能，但 T1、T2 發生後，A + B 的和應為 2000 * 1.06 = 2120 元。DBMS 無需保證 T1 與 T2 執行的先後順序，如果二者同時被提交，那麼誰先被執行都是有可能的，但執行後的淨結果應當與二者按任意順序分別執行的結果一致，如：

- 先 T1 後 T2：A = 954， B = 1166 => A+B = 2120
- 先 T2 後 T1：A = 960， B = 1160 => A+B = 2120

執行過程如下圖所示：

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%201.png)

當一個 transaction 在等待資源 (page fault、disk/network I/O) 時，或者當 CPU 存在其它空閒的 Core 時，其它 transaction 可以繼續執行，因此我們需要將 transactions 的執行重疊 (interleave) 起來。

- Example 1 (Good)
    
    ![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%202.png)
    
- Example 2 (Bad)
    
    ![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%203.png)
    

從 DBMS 的角度分析第二個例子：

- A = A - 100 => R(A), W(A)
- …

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%204.png)

如何判斷一種重疊的 schdule 是正確的？

> If the schedule is equivalent to some serial execution
> 

這裡明確幾個概念：

- Serial Schedule: 不同 transactions 之間沒有重疊
- Equivalent Schedules: 對於任意數據庫起始狀態，若兩個 schedules 分別執行所到達的數據庫最終狀態相同，則稱這兩個 schedules 等價
- Serializable Schedule: 如果一個 schedule 與 transactions 之間的某種 serial execution 的效果一致，則稱該 schedule 為 serializable schedule

## Conflicting Operations

在對 schedules 作等價分析前，需要瞭解 conflicting operations。當兩個 operations 滿足以下條件時，我們認為它們是 conflicting operations：

- 來自不同的 transactions
- 對同一個對象操作
- 兩個 operations 至少有一個是 write 操作

可以窮舉出這幾種情況：

- Read-Write Conflicts (R-W)
- Write-Read Conflicts (W-R)
- Write-Write Conflicts (W-W)

### Read-Write Conflicts/Unrepeatable Reads

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%205.png)

### Write-Read Conflicts/Reading Uncommitted Data/Dirty Reads

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%206.png)

### Write-Write Conflicts/Overwriting Uncommitted Data

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%207.png)

從以上例子可以理解 serializability 對一個 schedule 意味著這個 schedule 是否正確。

## Serializability Type

serializability 有兩個不同的級別：

- **Conflict Serializability** (Most DBMSs try to support this)：
    - 兩個 schedules 在 transactions 中有相同的 actions，且每組 conflicting actions 按照相同順序排列，則稱它們為 conflict equivalent
    - 一個 schedule S 如果與某個 serial schedule 是 conflict equivalent，則稱 S 是 conflict serializable
    - 如果通過交換不同 transactions 中連續的 non-conflicting operations 可以將 S 轉化成 serial schedule，則稱 S 是 conflict serializable
- **View Serializability** (No DBMS can do this)

## **Conflict Serializability**

例 1：將 conflict serializable schedule S 轉化為 serial schedule

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%208.png)

R(B) 與 W(A) 不矛盾，是 non-conflicting operations，可以交換它們

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%209.png)

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%2010.png)

R(B) 與 R(A) 是 non-conflicting operations，交換它們

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%2011.png)

依次類推，分別交換 W(B) 與 W(A)，W(B) 與 R(A) 就能得到：

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%2012.png)

因此 S 為 conflict serializable。

例 2：S 無法轉化成 serial schedule

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%2013.png)

由於兩個 W(A) 之間是矛盾的，無法交換，因此 S 無法轉化成 serial schedule。

### Faster Algorithms To Test Serializability

上文所述的交換法在檢測兩個 transactions 構成的 schedule 很容易，但要檢測多個 transactions 構成的 schedule 是否 serializable 就很麻煩了。

有沒有更快的算法來完成這個工作→Dependency Graphs

### Dependency Graphs

- One node per txn.
- Edge from Ti to Tj if:
    - An operation Oi of Ti conflicts with an operation Oj of Tj and
    - Oi appears earlier in the schedule than Oj
- Also known as a **precedence graph**

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%2014.png)

- A schedule is conflict serializable iff its dependency graph is acyclic(無環的)

### Example #1

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%2015.png)

- 形成cycle所以有問題

### Example #2

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%2016.png)

- 沒有cycle所以可以當作serial execution

### Example #3: Inconsistent Analysis

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%2017.png)

- 有Cycle所以不能轉為serial execution
- 但如果只是取得count
    
    ![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%2018.png)
    
    - 那麼仍然能得到正確的結果

## **View Serializability**

- Allows for all schedules that are conflict serializable and “blind writes”. Thus allows for slightly more schedules than Conflict serializability, but difficult to enforce efficiently. This is because the DBMS does not now how the application will “interpret” values.
- Schedules S1 and S2 are view equivalent if:
    - If T1 reads initial value of A in S1 , then T1 also reads initial value of A in S2
    - If T1 reads value of A written by T2 in S1, then T1 also reads value of A written by T2 in S2
    - If T1 writes final value of A in S1, then T1 also writes final value of A in S2
- 例子
    
    ![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%2019.png)
    
    - 這裡面有一堆Cycle，所以不能轉為serial execution
    - 但W(A)的順序不重要，重要的是最後一個W(A)是誰，那麼可等價為
        
        ![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%2020.png)
        

## Universe of Schedules

![Untitled](Concurrency%20Control%20Theory%20(ACID)%202a8bd1245c2f4d4888abd2ba6bc349bc/Untitled%2021.png)