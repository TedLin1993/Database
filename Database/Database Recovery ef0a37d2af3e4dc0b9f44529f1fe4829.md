# Database Recovery

上節課介紹到，故障恢復算法由兩個部分構成：

- 在事務執行過程中採取的行動來確保出現故障時能夠恢復 (上節課)
- 在故障發生後的恢復機制，確保原子性、一致性和持久性 (本節課)

# ARIES

---

本節課介紹的是 **A**lgorithms for **R**ecovery and **I**solation **E**xploiting **S**emantics (ARIES)，由 IBM Research 在 90 年代初為 DB2 DBMS 研發的基於 WAL 的故障恢復機制，盡管並非所有 DBMS 都嚴格按照 ARIES paper 實現故障恢復機制，但它們的思路基本一致。

ARIES 的核心思想可以總結為 3 點：

- **Write-Ahead Logging (WAL)**
    - 在數據寫入disk之前，所有操作都必須記錄在log中並寫入disk
    - 必須使用 Steal + No-Force 緩存管理策略 (buffer pool policies)
- **Repeating History During Redo**
    - 當 DBMS 重啟時，按照log的內容redo數據，恢復到故障發生前的狀態
- **Logging Changes During Undo**
    - 在 undo 過程中記錄 undo 操作到日誌中，確保在恢復期間再次出現故障時不會執行多次相同的 undo 操作

# Log Sequence Numbers

---

WAL 中的每條日誌記錄都需要包含一個全局唯一的 log sequence number (LSN)，一般 LSN 嚴格遞增。

DBMS 中的不同部分都需要記錄相關的 LSN 信息，舉例如下：

![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled.png)

- 在 buffer pool manager 中，每個 data page 都維護著 pageLSN
- DBMS 也需要追蹤log的 flushedLSN，也就是log最新的LSN
- 在 page x 寫入disk前，DBMS 必須保證以下條件成立：
    - $pageLSN_x \le flushedLSN$
    - 因為page寫入的資訊不能比log還快

![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%201.png)

- 當一個事務修改某 page 中的數據時，也需要更新該 page 的 pageLSN
- 在將操作日誌寫進 WAL 後，DBMS 會更新 flushedLSN 為最新寫入的 LSN。

# Normal Execution

---

每個事務都會包含一些列的讀和寫操作，然後提交 (commit) 或中止 (abort)，本節討論若不存在故障時，事務的正常執行過程。

在討論之前，我們需要約定 4 個假設，簡化問題：

- 所有日誌記錄都能放進一個 page 中
- 寫一個 page 到disk能保持原子性
- 沒有 MVCC，使用嚴格的 2PL
- 使用 WAL 記錄操作日誌，也就是buffer pool policy 為 Steal + No-Force

## Transaction Commit

- 當事務提交時，DBMS 先寫入一條 COMMIT 記錄到 WAL
- 然後將 COMMIT 及之前的log寫入disk
    - log flush是sequential的，也就是同步寫入disk
- 當寫入disk完成後，flushedLSN 被修改為 COMMIT 記錄的 LSN，同時 DBMS 將內存中 COMMIT 及其之前的日誌清除。最後再寫入一條 TXN-END 記錄到 WAL 中，作為內部記錄，對於執行提交的事務來說，COMMIT 與 TXN-END 之間沒有別的操作。
- 整個過程如下圖所示：

![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%202.png)

## Transaction Abort

要處理事務回滾，就必須從 WAL 中找出所有與該事務相關的日誌及其執行順序。

由於在 DBMS 中執行的所有事務的操作記錄都會寫到 WAL 中，因此為了提高效率，同一個事務的每條log都需要記錄上一條記錄的 LSN，即 prevLSN

- 特殊情況：第一條 BEGIN 記錄的 prevLSN 為nil

實際上中止事務是 ARIES undo 的一種特殊情況：回滾單個事務。

過程如下圖所示：

![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%203.png)

- T4 的每條日誌都記錄著 prevLSN
- 當 T4 要中止時，DBMS 先向 WAL 中寫入一條 ABORT 記錄，然後尋著 LSN 與 prevLSN 連接串成的link-list，找到之前的操作，倒序回滾
- 為了防止在回滾過程中再次故障導致部分操作被執行多次，回滾操作也需要寫入日誌中
- 等待所有操作回滾完畢後，DBMS 再往 WAL 中寫入 TXN-END 記錄，意味著所有與這個事務有關的日誌都已經寫完，不會再出現相關信息
- 那麼，如何記錄回滾操作呢？這就是我們馬上要介紹的 CLR

### Compensation(補償) Log Records

CLR 記錄的是 undo 操作，它除了記錄原操作相關的記錄，還記錄了 undoNext 指針，指向下一個將要被 undo 的 LSN

CLR 本身也是操作記錄，因此它也需要像其它操作一樣寫進 WAL 中

舉例如下：

![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%204.png)

![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%205.png)

值得注意的是：CLR 永遠不需要被 undo。

### Abort Algorithm

1. First write an ABORT record to log for the txn.
2. Then play back the txn's updates in reverse order.
3. For each update record:
    - Write a CLR entry to the log.
    - Restore old value.
4. At end, write a TXN-END log record.

# Non-fuzzy & fuzzy Checkpoints

---

## Non-fuzzy Checkpoints

使用 Non-fuzzy 的方式做 checkpoints 時，DBMS 會暫停所有工作，保證寫入disk的是一個 consistent snapshot

整個過程包括：

- 停止任何新的事務
- 等待所有活躍事務(active txns)執行完畢
- 將所有髒頁寫入disk

顯然這種方案很糟糕。

### Slightly Better Checkpoints

Non-fuzzy 需要停止所有事務，並且等待所有活躍事務執行完畢，我們是否有可能改善這一點？

一種做法是：checkpoint 開始後，暫停寫事務，阻止寫事務獲取數據或索引的write latch

如下圖所示：

![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%206.png)

- checkpoint 開始時，txn 已經獲取了 page#3 的寫鎖，後者可以繼續往 page#3 中寫數據，但不能再獲取其它 page 的寫鎖
- 此時 DBMS 只管掃描一遍 buffer pool 中的 pages，將所有髒頁寫入disk。
- 這時，部分 txn 寫入的數據可能會被 checkpoint 進程一起捎帶寫入disk，這時磁盤中的數據 snapshot 處於 inconsistent 的狀態。

所以在 checkpoint 的時候須記錄

- Active Transaction Table (ATT): 哪些活躍事務正在進行
- Dirty Page Table (DPT): 哪些數據頁是髒的

故障恢復時讀取 WAL 就能知道存在哪些活躍事務的數據可能被部分寫出，從而恢復 inconsistent 的數據。

### Active Transaction Table

活躍事務表中記錄著活躍事務的

- txnId: Unique txn identifier
- status: 事務狀態
    - R: Running
    - C: Committing
    - U: Candidate for Undo
- lastLSN: 最新的 LSN

當事務提交或中止後，相應的記錄才會被刪除

### Dirty Page Table

髒頁表記錄 buffer pool 中哪些page包含未提交事務(uncommitted transactions)

- 其中還記錄著每個髒頁第一筆導致dirty的 LSN，即 recLSN。

一個完整的 WAL 舉例如下：

![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%207.png)

- 在第一個 checkpoint 處：活躍事務有 T2，髒頁有 P11 和 P22
- 在第二個 checkpoint 處：活躍事務有 T3，髒頁有 P11 和 P33。

這種方案盡管比 Non-fuzzy 好一些，不需要等待所有活躍事務執行完畢，但仍然需要在 checkpoint 期間暫停執行所有寫事務。

## Fuzzy Checkpoints

所有DB都是使用fuzzy checkpoint，也是ARIES所使用的protocol

fuzzy checkpoint 允許任何活躍事務在它寫入disk的過程中執行，既然允許活躍事務執行，checkpoint 在 WAL 中的記錄就不是孤零零的一條，而是一個區間

因此我們需要兩類記錄來標記這個區間：

- CHECKPOINT-BEGIN：checkpoint 的起點
- CHECKPOINT-END：checkpoint 的終點，同時包含 ATT 和 DPT 記錄

當 checkpoint 成功完成時，CHECKPOINT-BEGIN 記錄的 LSN 才被寫入到資料庫的 MasterRecord 中

任何在 checkpoint **之後**才啟動的事務不會被記錄在 CHECKPOINT-END 的 ATT 中

舉例如下：

![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%208.png)

- 在第一個CHECKPOINT-BEGIN後面的T3，由於是在checkpoint之後才啟動的，所以沒有被記錄在ATT裡面

# ARIES - Recovery Phases

---

ARIES 故障恢復一共分三步：

- 分析 (analysis)：從 WAL 中讀取最近一次 checkpoint，找到 buffer pool 中相應的髒頁以及crash時的活躍事務
- 重做 (redo)：從正確的日誌點開始重做所有操作，包括將要中止的事務
- 撤銷 (undo)：將故障前未提交的事務的操作撤銷

整體流程如下圖所示：

![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%209.png)

通過 MasterRecord 找到最後一個 BEGIN-CHECKPOINT 記錄，然後分別進行 3 個階段：

1. 分析：找到最後一個 checkpoint 之後哪些事務提交或中止了
2. 重做redo：找到 DPT 中最小的 recLSN，從那裡開始重做所有操作
3. 撤銷undo：WAL 最近的位置開始往回撤銷所有未提交的事務操作

### Analysis Phase

從最近的 BEGIN-CHECKPOINT 開始往近處掃描日誌：

- 如果發現 TXN-END 記錄，則從 ATT 中移除該事務
- 遇到其它日誌記錄時
    - 將事務放入 ATT 中，將 status 設置為 UNDO
    - 如果事務提交，將其狀態修改為 COMMIT
    - 如果是UPDATE record：若page P不在 DPT，就將 P 加入DPT ，並設定 recLSN=LSN

當 Analysis Phase 結束時：

- ATT 告訴 DBMS 在發生故障時，哪些事務是活躍的
- DPT 告訴 DBMS 在發生故障時，哪些髒數據頁可能尚未寫入磁盤

Example:

![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%2010.png)

### Redo Phase

Redo Phase 的目的在於回放歷史，重建崩潰那一瞬間的資料庫狀態

- 即重做所有更新(update)操作 (包括aborted txns)，同時重做 CLRs

盡管 DBMS 可以通過一些手段避免不必要的讀寫，但本節課不討論這些優化技術。

從 DPT 中找到最小的 recLSN，從那裡開始重做更新記錄和 CLR，除非遇到以下其中一種情況：

- 受影響的 page 不在 DPT 中
- 受影響的 page 在 DPT 中，但那條記錄的 LSN 小於那個 page 的 recLSN

redo時，需要：

1. 重新執行日誌中的操作
2. 將 pageLSN 修改成日誌記錄的 LSN
3. 不再新增log，也不強制刷盤(flush)

在 Redo Phase 結束時，會為所有狀態為 COMMIT 的事務寫入 TXN-END 日誌，同時將它們從 ATT 中移除。

### Undo Phase

將所有 Analysis Phase 判定為 U (candidate for undo) 狀態的事務的所有操作按執行順序倒序撤銷，並且為每個 undo 操作寫一條 CLR。

### Full Example

- CheckPoint之後，T1 ABORT，之後Crash
    
    ![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%2011.png)
    
- Analysis Phase
    
    ![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%2012.png)
    
    - 建立ATT與DPT，並依序寫入資料到這兩個table
- Redo Phase
    
    ![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%2013.png)
    
    - 重作CLR
    - 但是在redo的時候又crash了，導致ATT和DPT都不見了
- 系統再次重啟，從crash的地方補回ATT與DPT，接著繼續redo後面的CLR
    
    ![Untitled](Database%20Recovery%20ef0a37d2af3e4dc0b9f44529f1fe4829/Untitled%2014.png)
    

### Additional Crash Issues

- 如果 DBMS 在故障恢復的 Analysis Phase 崩潰怎麼辦？
    
    → 無所謂，再執行一次recovery
    
- 如果 DBMS 在故障恢復的 Redo Phase 崩潰怎麼辦？
    
    → 無所謂，再redo所有操作即可
    
- 在 Redo Phase DBMS 如何能夠提高性能？
    
    → 如果資料庫不會再次故障，可以異步地將數據寫入disk
    
- 在 Undo Phase DBMS 如何能夠提高性能？
    1. Lazy Rollback：在新的事務access pages時才回滾數據
    2. 重寫Application，避免存在長時間的事務，這樣roll back的操作更少

# Conclusion

---

ARIES 的核心觀點回顧：

- WAL with Steal/No-Force
- Fuzzy Checkpoints (snapshot of dirty page ids)
- Redo everything since the earliest dirty page
- Undo txns that never commit
- Write **CLRs** when undoing, to survive failures during restarts

Log Sequence Numbers:

- LSNs identify log records; linked into backwards chains per transaction via prevLSN.
- pageLSN allows comparison of data page and log records.