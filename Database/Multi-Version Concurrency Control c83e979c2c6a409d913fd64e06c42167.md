# Multi-Version Concurrency Control

# 簡介

Multi-Version Concurrency Control (MVCC) is a larger concept than just a concurrency control protocol. It involves all aspect of the DBMS’s design and implementation. MVCC is the most widely used scheme in DBMS. It is now used in almost every new DBMS implemented in last 10 years. Even some systems (e.g., NoSQL) that do not support multi-statement transactions use it.

實現 MVCC 的 DBMS 在內部為每個logical資料維持多個**Physical**版本

- 當事務修改某數據時，DBMS 將為其創建一個新的版本
- 當事務讀取某數據時，它將讀到該數據在事務開始時刻之前的最新版本。

MVCC 首次被提出是在 1978 年的一篇 MIT 的博士[論文](https://web.archive.org/web/20051025124412/http://www.lcs.mit.edu/publications/specpub.php?id=773)中。

- 在 80 年代早期，DEC 的 Rdb/VMS 和 InterBase 首次真正實現了 MVCC，其作者是 Jim Starkey，NuoDB 的聯合創始人。
- 如今，Rdb/VMS 成了 Oracle Rdb，InterBase 成為開源項目 Firebird。

# MVCC

MVCC 的核心優勢可以總結為以下兩句話：

> Writers don’t block readers. 寫不阻塞讀
> 
> 
> Readers don’t block writers. 讀不阻塞寫
> 

read-only事務無需加鎖就可以讀取數據庫某一時刻的**快照(snapshot)**

如果保留數據的所有歷史版本，DBMS 甚至能夠支持讀取任意歷史版本的數據，即 **time-travel** query。

## Example #1

---

事務T1和T2分別獲得時間戳 1 和 2，二者的執行過程如下圖所示。

- 開始前，資料庫存有數據 A 的原始版本A0，T1先讀取 A 數據：
    
    ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled.png)
    
- 然後T2修改 A 數據，這時 DBMS 中將增加 A 數據的新版本A1，同時標記A1的開始時間戳為 2，A0的結束時間戳為 2：
    
    ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%201.png)
    
- T1再次讀取 A，因為它的時間戳為 1，根據記錄的信息，DBMS 將A0返回給T1：
    
    ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%202.png)
    

## Example #2

---

例 2 與例 1 類似

- T1先修改數據 A：
    
    ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%203.png)
    
- 此時T2讀取 A，由於T1尚未提交，T2只能讀取A0：
    
    ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%204.png)
    
- T2想修改 A，但由於有另一個活躍的事務T1正在修改 A ，T2需要等待T1提交後才能繼續推進：
    
    ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%205.png)
    
- T1提交後，T2創建了 A 的下一個版本A2：
    
    ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%206.png)
    

## 小結

---

MVCC 並不只是一個並發控制協議，並發控制協議只是它的一個組成部分。它深刻地影響了 DBMS 管理事務和數據的方式，使用 MVCC 的 DBMS 數不勝數：

![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%207.png)

# Design Decisions

上文提到，MVCC 不止是一個並發控制協議，它由許多部分組成，這些部分包括：

- Concurrency Control Protocol(T/O, OCC, 2PL, etc).
- Version Storage
- Garbage Collection
- Index Management

每一部分都可以選擇不同的方案，可以根據具體場景作出最優的設計選擇。

## Concurrency Control Protocol

---

前面 2 節課已經介紹了各種並發控制協議，MVCC 可以選擇其中任意一個：

- **Approach #1：Timestamp Ordering (T/O)**：為每個事務賦予時間戳，並用以決定執行順序
- **Approach #2：Optimistic Concurrency Control (OCC)**：為每個事務創建 private workspace，並將事務分為 read, write 和 validate 3 個階段處理
- **Approach #3：Two-Phase Locking (2PL)**：按照 2PL 的約定獲取和釋放鎖

## Version Storage

---

如何存儲一條數據的多個版本？DBMS 通常會在每條tuple上拉一條**版本鏈表 (version chain)**

- DBMS 可以利用它找到一個事務應該訪問到的版本
- 所有相關的索引都會指到這個鏈表的 head

不同的版本存儲方案在 version chain 上存儲的數據不同，主要有 3 種存儲方案：

- **Approach #1：Append-Only Storage**：新版本通過追加的方式存儲在同一張表中
- **Approach #2：Time-Travel Storage**：老版本被複製到單獨的一張表中
- **Approach #3：Delta Storage**：老版本數據的被修改的字段值被複製到一張單獨的增量表 (delta record space) 中

### Append-Only Storage

如下圖所示，同一個邏輯數據的所有物理版本都被存儲在同一張表上

- 每次更新時，就往表上追加一個新的版本記錄，並在舊版本的數據上增加一個指針指向新版本：
    
    ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%208.png)
    
- 再次更新的行為類似：
    
    ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%209.png)
    
- 這是PostgreSQL所使用的方法

也許你已經注意到，指針的方向也可以從新到舊，二者的權衡如下：

- **Approach #1：Oldest-to-Newest (O2N)**：
    - 寫的時候追加即可
    - 讀的時候需要遍歷鏈表
- **Approach #2：Newest-to-Oldest (N2O)**：
    - 寫的時候需要更新所有索引指針
    - 讀的時候不需要遍歷鏈表

### Time-Travel Storage

單獨拿一張表 (Time-Travel Table) 來存歷史數據，每當更新數據時，就把當前版本復制到 TTT 中，並更新指針：

![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2010.png)

![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2011.png)

![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2012.png)

### Delta Storage

每次更新，僅將變化的字段信息存儲到 delta storage segment 中：

![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2013.png)

![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2014.png)

![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2015.png)

- DBMS 可以通過 delta 數據逆向恢復數據到之前的版本。
- 使用DB: MySQL, Oracle
- Andy認為最好的做法

## Garbage Collection

---

隨著時間的推移，DBMS 中數據的舊版本可能不再會被用到，如：

- 已經沒有活躍的事務需要看到該版本
- 該版本是被一個已經中止的事務創建

這時候 DBMS 需要刪除這些**可以回收的(reclaimable)**物理版本，這個過程也被稱為 GC。

在 GC 的過程中，還有兩個附加設計決定：

- 如何查找過期的數據版本
- 如何確定某版本數據是否可以被安全回收

GC 可以從兩個角度出發：

- **Approach #1：Tuple-level**
    - 直接檢查每條tuple的舊版本資料
    - **Background Vacuuming(吸塵)** vs. **Cooperative(合作) Cleaning**
- **Approach #2：Transaction-level**
    - 每個事務都記錄舊版本的資料，DBMS 不需要scan tuples去一條一條檢查舊版本資料

### Tuple-Level GC

- ***Background Vacuuming***
    - Background Vacuuming: ******Separate threads periodically scan the table and look for reclaimable versions, works with any version storage scheme.
    - 如下圖所示，假設有 2 個活躍事務，它們的時間戳分別為 12 和 25：
        
        ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2016.png)
        
    - 這時有個 Vacuum 守護線程會週期性地檢查每條數據的不同版本，如果它的結束時間小於當前活躍事務的最小時間戳，則將其刪除：
        
        ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2017.png)
        
    - 為了加快 GC 的速度，DBMS 可以再維護一個髒頁位圖 (dirty page bitmap)，利用它，Vacuum 線程可以只檢查發生過改動的數據，用空間換時間。
        
        ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2018.png)
        
    - Background Vacuuming 被用於任意 Version Storage 的方案。
- ***Cooperative Cleaning***
    - 還有一種做法是當 worker thread 查詢數據時，順便將不再使用物理資料版本刪除：
        
        ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2019.png)
        
        ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2020.png)
        
        ![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2021.png)
        
    - cooperative cleaning 只能用於使用 Old2New 的 version chain 方案，New2Old的話就沒辦法順便刪掉舊版本的資料了

### Transaction-Level GC

- 讓每個事務都保存著它的讀寫數據集合 (read/write set)
- 當 DBMS 決定什麼時候這個事務創建的各版本數據可以被回收時，就按照集合內部的數據處理即可。

## Index Management

---

### Primary Key Index

主鍵索引直接指向 version chain 的頭部

- DBMS 更新 pkey 索引的頻率取決於系統是否在更新tuple時創建新版本
- 更新tuple的pkey就是DELETE tuple再INSERT tuple

![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2022.png)

### Secondary Indexes

二級索引有兩種方式指向數據本身：

- **Approach #1：Logical Pointers**，即存儲Primary Key或 Tuple Id
    - 每個tuple使用固定的ID，這個ID無法改動
    - 需要額外的indirection layer
    - Primary Key vs. Tuple Id
- **Approach #2：Physical Pointers**，即存儲指向 version chain 頭部的指針

**Physical Pointer**

![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2023.png)

**Logical Pointer by Primary Key**

![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2024.png)

**Logical Pointer by Tuple Id**

![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2025.png)

## MVCC Implementations

---

市面上 MVCC 的實現所做的設計決定如下表所示：

![Untitled](Multi-Version%20Concurrency%20Control%20c83e979c2c6a409d913fd64e06c42167/Untitled%2026.png)

- Andy研究認為Oracle和MySQL的設計決定在OLTP是最快的

# Conclusion

MVCC 被許多 DBMS 採用，即使那些不支持多語句事務 (multi-statement txns) 的 DBMS 也會使用這種方案，如一些 NoSQL 項目。