# Timestamp Ordering Concurrency Control

上節課介紹的 2PL 是悲觀的並發控制策略，本節課介紹的 Timestamp Ordering (T/O) 則是一個樂觀的策略，其樂觀表現在事務訪問資料時不需要lock。

T/O 的核心思想就是利用時間戳(Timestamp, TS)來決定事務之間的等價執行順序：**如果**$TS(T_i) < TS(T_j)$**，那麼資料庫必須保證實際的 schedule 與先執行**$T_i$**，後執行**$T_j$**的結果等價**。

要實現 T/O，就需要一個嚴格遞增的時鐘，來決定任意事務$T_i$發生的時間。滿足條件的時鐘方案有很多，如：

- 系統單調時鐘 (System Clock)
- 邏輯計數器 (Logical Counter)
- 混合方案 (Hybrid)

# Basic T/O

---

Basic T/O 是 T/O 方案的一種具體實現。在 Basic T/O 中，事務讀寫資料不需要加鎖，每條資料 X 都會攜帶兩個標記：

- W-TS(X)：最後一次寫 X 發生的時間戳
- R-TS(X)：最後一次讀 X 發生的時間戳

在每個事務結束時，Basic T/O 需要檢查該事務中的每個操作，是否讀取或寫入了未來的資料，一旦發現則中止、重啟事務。

## Basic T/O Reads

```sql
func read(X) val {
    if TS(T_i) < W_TS(X) {
        abort_and_restart(T_i)
    } else {
        val := read_data(X)
        R_TS(X) = max(R_TS(X), TS(T_i))
        // make a local copy of X to ensure repeatable reads for T_i
        return val
    }
}
```

- 如果事務Ti發生在 W-TS(X) 之前，即嘗試讀取未來的數據，則中止Ti；如果事務Ti發生在 W-TS(X) 之後，意味著它正在讀取過去的數據，符合規范。
- Ti讀取數據後會更新 R-TS(X)，同時保留一份 X 的副本，用來保證Ti結束之前總是能讀到相同的 X。

## Basic T/O Writes

寫入數據時的邏輯如下所示：

```sql
func write(X, val) {
    if TS(T_i) < R_TS(X) || TS(T_i) < W_TS(X) {
        abort_and_restart(T_i)        
    } else {
        X = val
        W_TS(X) = max(W_TS(X), TS(T_i))
        // make a local copy of X to ensure repeatable reads for T_i
    }
}
```

- 如果事務Ti發生在 W-TS(X) 或 R-TS(X) 之前，即嘗試寫入已經被未來的事務讀取或寫入的數據，則中止Ti；反之，意味著它正嘗試修改過去的數據，符合規范。
- Ti寫入數據後，如果有必要，則更新 W-TS(X)，同時保留一份 X 的副本，用來保證Ti結束之前總是能讀到相同的 X。

## Basic T/O - Example #1

如下圖所示：有兩個事務T1和T2，它們的時間戳分別為 1，2，即T1發生在T2之前，它們要訪問的數據為 A 和 B，假設它們是數據庫預填充的數據，R-TS 和 W-TS 都為 0。

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled.png)

​

- T1先讀取 B，將 R-TS(B) 更新為 1
    
    ![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%201.png)
    

​

- T2讀取 B，將 R-TS(B) 更新為 2
    
    ![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%202.png)
    

​

- T2修改 B，將 W-TS(B) 更新為 2
    
    ![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%203.png)
    

​

- T1讀取 A，將 R-TS(A) 更新為 1
    
    ![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%204.png)
    

​

- T2讀取 A，將 R-TS(A) 更新為 2
    
    ![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%205.png)
    

​

- T2修改 A，將 W-TS(A) 更新為 2
    
    ![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%206.png)
    

由於整個過程，沒有發生違背規范的操作，因此兩個事務都能夠成功提交。

## Basic T/O - Example #2

類似地，我們可以看下面這個例子：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%207.png)

不難看出，T1在T2修改 A 後又修改了 A，該操作肯定會違反規范：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%208.png)

因此T1將被資料庫中止。但實際上，仔細分析上述例子：

- 如果我們忽略掉T1的 W(A) 操作，即不更新 A 數據，也不修改 W-TS(A)，那麼T1和T2都可以正常提交，且結果和二者先後執行等價
- 這便是所謂的 Thomas Write Rule (TWR)

## Thomas Write Rule

Thomas Write Rule: Ignore the write and allow transaction to continue

```sql
func write(X, val) {
    if TS(T_i) < R_TS(X) {
        abort_and_restart(T_i)
        return
    }
    
    if TS(T_i) < W_TS(X) {
        //Thomas Write Rule: ignore write
        return
    }
    
    X = val
    W_TS(X) = TS(T_i)
    // ...
}
```

example #2 符合 TWR，可以允許讓兩個事務順利提交。TWR 優化了 Basic T/O 的寫檢查，使得一些本不必中止的事務順利進行，提高了事務並發程度。

## Basic T/O Summary

如果不使用 TWR 優化，Basic T/O 能夠生成 conflict serializable 的 schedule，如果使用了 TWR，則 Basic T/O 生成的 schedule 雖然與順序執行的效果相同，但不滿足 conflict serializable。

Basic T/O 的優勢在於：

1. 不會造成死鎖，因為沒有事務需要等待
2. 如果單個事務涉及的數據不多、不同事務涉及的數據基本不相同 (OLTP)，可以節省 2PL 中控制鎖的額外成本，提高事務並發度

其缺點在於：

1. 長事務容易因為與短事務沖突而餓死(Starvation)
2. 複製數據，維護、更新時間戳存在額外成本
3. 可能產生不可恢復的 schedule (具體見下節)

## Recoverable Schedules

如果一個 schedule 能夠保證每個事務提交前，修改過其讀取過數據的事務都已提交，那麼這個 schedule 就是 recoverable。如果不能保證 recoverable，DBMS 就無法在發生崩潰之後恢復數據，舉例如下：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%209.png)

​

- T2在T1修改 A 之後讀取 A，符合規范。
- 但是在T2提交之後，T1中止，前者依賴的數據實際上並未真實寫入，資料庫發生故障以後將無法恢復。因此 Basic T/O 可能產生不可恢復的 schedules。

# Optimistic Concurrency Control (OCC)

---

OCC 是 H.T. KUNG 在 CMU 任教時提出的並發控制算法。在 OCC 中，資料庫為每個事務都創建一個私有空間：

- 所有被讀取的數據都複製到私有空間中
- 所有修改都在私有空間中執行

OCC 分為 3 個階段：

1. Read Phase：追蹤、記錄每個事務的讀、寫集合，並存儲到私有空間中
2. Validation Phase：當事務提交時，檢查沖突
3. Write Phase：如果校驗成功，則合併數據；否則中止並重啟事務

DBMS 需要維持所有活躍事務的全局視角，並將 Validation Phase 和 Write Phase 的邏輯放入一個 critical section 中。

## OCC - Example

事務T1讀取 A 時，將 A 複製到自己的 workspace 中，可以看到，與 Basic T/O 相比，OCC 只需要記錄一個時間戳，W-TS。

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2010.png)

事務T2讀取 A 時，同樣將 A 複製到自己的 workspace 中：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2011.png)

事務T2完成數據操作，在 Validation Phase 中獲得事務時間戳 1，由於沒有數據寫入，跳過 Write Phase

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2012.png)

事務T1修改 A 的值為 456，由於尚不知道自己的事務時間戳，將 W-TS(A) 設置為無窮大：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2013.png)

事務T1在 Validation Phase 獲得事務時間戳 2，並通過校驗，將 W-TS(A) 修改為 2，並合併到數據庫中

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2014.png)

## OCC - Read Phase

追蹤事務的讀寫集合 (read/write sets)

- 將 read set 存放在 private workspace 中用來保證 repeatable read
- 將 write set 存放在 private workspace 中用來作沖突檢測。

## OCC - Validation Phase

在進入 Validation Phase 後，每個事務都會**被賦予一個時間戳**，然後與其它正在運行的事務執行 Timestamp Ordering 檢查，檢查的方式有兩種：

1. Backward Validation
2. Forward Validation

### Backward Validation

如下圖所示，在 Backward Validation 中，需要檢查待提交的事務 (txn #2) 的讀寫集合是否與**已經**提交的事務的讀寫集合存在交集：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2015.png)

### Forward Validation

與此類似，在 Forward Validation 中，需要檢查待提交的事務 (txn #2) 的讀寫集合是否與**尚未**提交的事務的讀寫集合存在交集：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2016.png)

- 若txn #3讀到的資料是txn #2修改的資料，那麼就將txn #2 abort

### Validation Conditions

如果TS(Ti)<TS(Tj)，那麼以下 3 個條件之一必須成立：

***Condition 1: Ti completes all three phases before Tj begins***

- 如果事務Ti在事務Tj開始之前已經完成 OCC 的所有 3 個階段，那麼二者之間不存在任何沖突。

***Condition 2: Ti completes before Tj starts its Write Phase, and Ti does not write to any object read by Tj***

- 如果Ti在Tj的 Write Phase 開始前就已提交，同時Ti沒有修改任意Tj讀取的數據，即$WriteSet(T_i) \cap ReadSet(T_j) = \emptyset \space,  \space WriteSet(T_i) \cap WriteSet(T_j) = \emptyset$，則二者之間不存在沖突。

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2017.png)

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2018.png)

***Condition 3: Ti completes its Read Phase before Tj completes its Read Phase, and Ti does not write to any object that is either read or written by Tj***

- 如果Ti在Tj結束自己的 Read Phase 前結束 Read Phase，同時Ti沒有修改任何Tj讀取或修改的數據，即滿足：
    
    $WriteSet(T_i) \cap ReadSet(T_j) = \emptyset \space, \space WriteSet(T_i) \cap WriteSet(T_j) = \emptyset$
    
    時，二者之間不存在沖突。
    
    ![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2019.png)
    

## Conclusion

OCC 與 Basic T/O 的思路類似，都是在檢查事務之間的 WW、WR 沖突。

當沖突發生的頻率很低時，即：

- 大部分事務都是read-only
- 大部分事務之間訪問的資料間沒有交集

OCC 的表現很好在資料庫資料較大，且workload 比較均衡的場景下。

- 這種情境下2PC 的性能瓶頸在於lock

盡管 OCC 沒有加鎖的成本，但它也存在性能問題:

- 在 private workspace 與 global database 之間複製、合併數據開銷(overhead)大
- Validation/Write Phase 需要在一個全局的 critical section 中完成，可能造成瓶頸
- 在 Validation Phase 中，待提交事務需要和其它事務做沖突檢查，即便實際上並沒有沖突，這裡也有很多獲取 latch 的成本 (鎖住其它事務的 private workspace，對比是否有沖突，再釋放鎖)
- 事務中止的成本比 2PL 高，因為 OCC 在事務執行快結束時才檢查數據沖突

# Partition-Based T/O

---

When a transaction commits in OCC, the DBMS has check whether there is a conflict with concurrent transactions across the entire database. This is slow if we have a lot of concurrent transactions because the DBMS has to acquire latches to do all of these checks.

類似全局鎖到分段鎖的優化，我們也可以將資料庫切分成不相交 (disjoint) 的子集，即 **horizontal partitions** 或 shards

然後在 partition 內部使用嚴格遞增的時間戳確定各個事務的順序，不同 partition 上的事務之間無需檢測沖突。

假設數據庫中存儲著如下三張表：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2020.png)

我們可以按照 customer 的 c_id 對數據庫分片(partition)：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2021.png)

每個 partition 使用一個鎖保護：

- 當事務需要訪問多個 partitions 時，就在所需的多個 partitions 上排隊
- 如果事務的時間戳是整個 partition 中最小的，那麼該事務就獲得鎖
- 當事務獲取其所需訪問的所有 partitions 的全部鎖，它就可以開始執行

## Partition-Based T/O - Reads

- 如果事務已經獲取partitions上的鎖，該事務就能夠讀取它想讀取的任意數據。
- 如果事務嘗試訪問一個未獲取鎖的partitions，那麼它將被中止後重啟。

## Partition-Based T/O - Writes

- 寫事務直接在原地修改數據，並在內存中維護一個緩沖區
    - 用來記錄修改的數據以便事務中止後回滾。
- 如果事務嘗試修改一個未獲取鎖的分片(partition)，那麼它將被中止後重啟。

## Partition-Based T/O - Example

假設有兩個事務同時開啟，並分別被分配了全局的時間戳 100 和 101，二者都需要獲取 partition 1 上的鎖，如下圖所示：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2022.png)

由於事務 #100 的時間戳較小，它將獲得 partition 1 的鎖，從而執行事務：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2023.png)

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2024.png)

隨後事務 #101 才能夠獲得 partition 1 的鎖，執行事務內容

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2025.png)

Partition-based T/O 的性能取決於以下兩點：

- DBMS 是否在事務開啟前就能知道事務所需的所有 partitions
- 是否大多數事務只需要訪問單個 partition

multi-partition 的事務將使得更多其它事務陷入等待狀態，取了鎖而未使用的 partition 也可能陷入空轉。

# Dynamic Databases

---

到現在為止，我們都只考慮事務read和update數據，如果我們再考慮插入、刪除操作，就會遇到新的問題。

## The Phantom(幻影) Problem

考慮插入操作，則可能出現 Phantom Read：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2026.png)

- 即在單個事務內部，同樣的查詢，讀到不一樣的數據。這種現象發生的原因在於：儘管T1鎖住了已經存在的記錄，但新生成的記錄並不會被鎖住
- Conflict serializability 能保證事務可序列化(serializability)的前提是**數據集合(set of objects)是固定的**，出現記錄新增和刪除時，Conflict serializability就不成立了。

## Predicate Locking

- Predicate locking 指的是通過一個Predicate (where後面的限制條件)來為潛在的記錄加鎖
    - 如：`status = 'lit'` 。
- 然而，Predicate locking 的成本很高，因為Predicate可以很複雜，可能會是個多維的限制條件，所以DBMS基本上不使用Predicate locking
- 一種更高效的做法是 index locking。

## Index Locking

- 同樣以上文中的例子為例，如果在 `status` 字段上有索引，那麼我們可以鎖住滿足 `status = 'lit'` 的 index page
- 如果尚未存在這樣的 index page，我們也需要能夠找到可能對應的 index page，鎖住它們。

### Locking Without An Index

同樣以上文中的例子為例，如果在 `status` 字段上沒有索引，那麼事務就需要執行以下操作：

- 獲取 table 的每個 page 上的鎖，防止其它記錄的 `status` 被修改成 `lit`
- 獲取 table 本身的鎖，防止滿足 `status = 'lit'` 的記錄被插入或刪除

## Repeating Scans

另一種比較暴力的做法是在事務提交時，掃描 `status = 'lit'` 的所有數據，檢查這些數據是否與事務操作之前的數據相同。目前沒有任何商業數據庫採用這種方案。

# Isolation Level

---

以上討論的都是可序列化的並發控制方案。可序列化固然是一種很實用的特性，它可以將程序員從並發問題中解脫，但可序列化的方案要求比較嚴格，會對系統的並發度和性能造成較大的限制，因此我們也許能夠用更弱的數據一致性保證去改善系統的擴展性。這也是所謂的數據庫隔離級別。

更弱的數據庫隔離級別將事務修改的數據暴露給其它事務，以此提高整體並發度，但這種並發度可能造成一系列問題，如 Dirty Reads/Writes (髒讀、髒寫)、Unrepeatable Reads (不可重復讀)、Phantom Reads (幻讀) 等等。

常見的數據庫隔離級別從弱到強依次包括：Read Uncommitted -> Read Committed -> Repeatable Reads -> Serializable，總結如下表：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2027.png)

該表總結得不太完全，更詳細的討論可參考 [Transactions](/open-courses/ddia-bi-ji/transactions)。

## SQL - 92 Isolation Levels

SQL-92 中定義了數據庫設置隔離級別的命令：

```sql
SET TRANSACTION ISOLATION LEVEL <isolation-level>;   // 全局设定
BEGIN TRANSACTION ISOLATION LEVEL <isolation-level>; // 单事务设定
```

但並非所有數據庫在所有運行環境中都能支持所有隔離級別，且數據庫的默認隔離級別取決於它的實現。以下是 2013 年統計的一些數據庫的默認隔離級別和最高隔離級別：

![Untitled](Timestamp%20Ordering%20Concurrency%20Control%20527584cf8fda4f1ea6a7227441e716dc/Untitled%2028.png)

## SQL-92 Access Mode

SQL-92 中也允許用戶提示數據庫自己的事務是否會修改數據：

```sql
SET TRANSACTION <access-mode>;   // 全局设置
BEGIN TRANSACTION <access-mode>; // 单个事务设置
```

其中 access-mode 有兩種模式：READ WRITE 和 READ ONLY。當然，即便在 SQL 語句中添加了這種提示，也不是所有數據庫都會利用它來優化 SQL 語句的執行。

# Conclusion

---

任意一種並發控制都可以被分解成前兩節課中提到的基本概念。