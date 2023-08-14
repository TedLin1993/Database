# Logging Schemes

數據庫在運行時可能遭遇各種故障，這時可能同時有許多正在運行的事務，如果這些事務執行到一半時故障發生了，就可能導致資料庫中的數據出現不一致的現象：

![Untitled](Logging%20Schemes%205794515f5cb24f87b0c7b6f5d7e5d2b6/Untitled.png)

這時就需要故障恢復機制來保證資料庫的原子性、一致性、持久性。故障恢復機制包含兩部分：

- 在事務執行過程中採取的行動來確保在出現故障時能夠恢復 (本節課)
- 在故障發生後的恢復機制，確保原子性、一致性和持久性 (下節課)

# Agenda

---

- Failure Classification
- Buffer Pool Policies
- Shadow Paging
- Write-Ahead Log
- Logging Schemes
- Checkpoints

# Failure Classification

---

世上並不存在能夠容忍任何故障的容錯機制，即便做了異地多活，一次小行星撞地球也能將你的系統一舉殲滅，因此在討論故障恢復之前，我們必須先為故障分類，明確需要容忍的故障類型。

一般故障類型可以分為 3 種：

1. 事務故障 (Transaction Failures)
2. 系統故障 (System Failures)
3. 存儲介質故障 (Storage Media Failures)

### Transaction Failures

事務故障可以分為兩種：

1. **邏輯錯誤 (Logical Errors)**：
    - 由於一些內部約束，如數據一致性約束(integrity constraint violation)，導致事務無法正常完成
    - 例如insert foreign key，但實際上並沒有這個key
2. **內部錯誤 (Internal State Errors)**：
    - 資料庫因為內部錯誤必須中斷事務
    - 例如dead-lock

### System Failures

系統故障可以分為兩種：

1. **軟件故障 (Software Failure)**：
    - DBMS 本身的實現問題 (例如 uncaught Divide-by-zero exception)
2. **硬件故障 (Hardware Failure)**：
    - DBMS 所在的宿主機發生崩潰，如斷電
    - Fail-stop Assumption: 一般假設非易失性(Non-volatile)的存儲數據在主機崩潰後不會丟失

### Storage Media Failures

- 如果存儲介質發生故障，通常這樣的故障就是無法修復的，如發生撞擊導致磁盤部分或全部受損。
- 所有資料庫都無法從這種故障中恢復，這時候數據庫只能從其他地方的備份中恢復數據。

# Buffer Pool Policies

---

修改數據(commit)時，DBMS 需要先把對應的數據頁從持久化存儲中讀到內存中，然後在內存中根據寫請求修改數據，最後將修改後的數據寫回到持久化存儲。

在整個過程中，DBMS 需要保證兩點：

1. DBMS 告知用戶事務已經提交成功前，相應的數據必須已經持久化
2. 如果事務中止，任何數據修改都不應該持久化

如果真的遇上事務故障或者系統故障，DBMS 有兩種基本思路來恢復數據一致性，向用戶提供上述兩方面保證：

- **UNDO**：將中止或未完成的事務中，rollback已經執行的操作
- **REDO**：將提交的事務執行的操作重做

DBMS 如何支持 undo/redo 取決於它如何管理 buffer pool。

我們可以從兩個角度來分析 buffer pool 的管理策略：Steal Policy 和 Force Policy。以下圖為例：

![Untitled](Logging%20Schemes%205794515f5cb24f87b0c7b6f5d7e5d2b6/Untitled%201.png)

- 有 2 個並發事務 T1 和 T2，T1 要修改 A 數據，T2 要修改 B 數據
- A、B 數據都位於同一塊Page上
- 在 T2 事務提交時，T1 事務尚未結束，這時 T1 已經修改了 A 數據
- 此時 DBMS 會允許被修改但未提交的 A 數據進入持久化存儲嗎？
- 若T2的commit寫入disk了，但T1後面ABORT，又會發生甚麼事？

## Steal Policy

DBMS 是否允許一個未提交事務修改持久化存儲中的數據？

teal Policy 有兩種可能：允許 (Steal) 和不允許 (No-Steal)

- 允許：若 T1 事務回滾，則需要把已經持久化的數據讀進來，回滾數據，再存回去，但在不發生回滾時 DBMS 的 I/O 較低
- 不允許：若 T1 事務回滾，DBMS 不需要做任何額外的操作，但在不發生回滾時 DBMS 的 I/O 較高。因此這裡存在數據恢復時間與 DBMS 處理效率的取捨

## Force Policy

DBMS 是否強制要求一個提交完畢事務的所有數據改動都反映在持久化存儲中？

Force Policy 有兩種可能：強制 (Force) 和非強制 (No-Force)。

- 如果選擇強制，每次事務提交都必須將數據寫入disk，數據一致性可以得到完美保障，但 I/O 效率較低
- 如果選擇非強制，DBMS 則可以延遲批量地將數據寫入disk，數據一致性可能存在問題，但 I/O 效率較高

因此我們有 4 種實現組合：

- Steal + Force
- Steal + No-Force
- No-Steal + Force
- No-Steal + No-Force

在實踐中使用的主要是 No-Steal + Force 和 Steal + No-Force。

# Shadow Paging: No-Steal + Force

---

最容易實現的一種策略組合就是 No-Steal + Force：

- 事務中止後，無需undo，因為該事務修改的數據不會被別的事務也寫入disk
- 事務提交後，無需redo，因為該事務修改的數據必然會被寫入disk持久化

![Untitled](Logging%20Schemes%205794515f5cb24f87b0c7b6f5d7e5d2b6/Untitled%202.png)

當然，這種策略組合無法處理 “**寫入的數據量(write sets)超過 buffer pool 大小**” 的情況。

## Shadow Paging

shadow paging 是 No-Steal + Force 策略的典型代表，它會維護兩份資料庫數據：

1. Master：包含所有已經提交事務的數據(only changes)
2. Shadow：暫存的database，包含uncommitted txns所造成的改動

正在執行的事務都只將修改的數據寫到 shadow copy 中，當事務提交時，再原子地把 shadow copy 修改成新的 master。

當然，為了提高效率，DBMS 不會複製整個數據庫，只需要複製有變動的部分即可，即 copy-on-write。

通常，實現 shadow paging 需要將數據頁組織成樹狀結構，如下圖所示：

![Untitled](Logging%20Schemes%205794515f5cb24f87b0c7b6f5d7e5d2b6/Untitled%203.png)

- 左邊是內存中的數據結構，由一個根節點(root)指向對應的 page table，其中 page table 對應著磁盤中的相應的數據頁。
- 在寫事務執行時，會生成一個 shadow page table，並複製需要修改的數據頁，在其上完成相應操作。
- 在提交寫事務時，DBMS 將 DB Root 的指針指向 shadow page table，並將對應的數據頁全部落盤

如下圖所示：

![Untitled](Logging%20Schemes%205794515f5cb24f87b0c7b6f5d7e5d2b6/Untitled%204.png)

- 在事務提交前，任何被修改的數據都不會被持久化到資料庫
- 在事務提交後，所有被修改的數據都會被持久化到資料庫

在 shadow paging 下回滾、恢復數據都很容易：

- Undo/Rollback：刪除 shadow pages (table)，啥都不用做
- Redo：不需要 redo，因為每次寫事務都會將數據寫入disk

shadow paging 很簡單，但它的缺陷也很明顯：

- **複製整個 page table 代價較大**，儘管可以通過以下措施來降低這種代價：
    - 需要使用類似 B+ 樹的數據結構來組織 page table
    - 無需複製整個樹狀架構，只需要複製有變動的leaf節點的路徑即可
- **事務提交的代價較大**：
    - 需要將所有發生更新的 data page、page table 以及根節點都寫入disk
    - 容易產生磁盤碎片(fragmented)，使得原先距離近的數據漸行漸遠
    - 需要做垃圾收集
    - 只支持one writer  txn at a time 或txns in a batch

在 2010 年之前，SQLite 採用的就是類似 shadow paging 的策略，當寫事務需要修改某頁數據時，DBMS 會先將原始數據頁保存到一個獨立的 journal 文件中，然後再將對應的修改持久化到磁盤中。當 SQLite 重啟後，如果發現磁盤中存在 journal 文件，則之間將對應的數據頁覆蓋到磁盤中即可。

# Write-Ahead Log (WAL): Steal + No-Force

---

通過剛才對 shadow paging 的討論，我們可以發現導致其效率問題的主要原因是：DBMS 在實現 shadow paging 時需要將許多不連續的數據頁寫到磁盤中，隨機寫對磁盤來說並不友好，如果能將這種隨機寫入轉化為順序寫入，那麼效率自然能夠提升。於是 WAL 來了。

WAL 指的是 DBMS 除了維持正常的數據文件外，額外地維護一個日誌文件，上面記錄著所有事務對 DBMS 數據的完整修改記錄，這些記錄能夠幫助數據庫在恢復數據時執行 undo/redo。

使用 WAL 時，DBMS 必須**先**將操作日誌持久化到獨立的日誌文件中，然後才修改其真正的數據頁。

通常實現 WAL 採用的 buffer pool policy 為 Steal + No-Force，下面我們將詳細地介紹 WAL Protocol。

## WAL Protocol

1. DBMS 先將事務的操作記錄放在內存中 (backed by buffer pool)
2. 在將 data page 寫入disk前，所有的日誌記錄必須先寫入disk
3. 在操作日誌寫入disk後，事務就被認為已經提交成功

事務開始時，需要寫入一條 `<BEGIN>` 記錄到日誌中；事務結束時，需要寫入一條 `<COMMIT>` 記錄到日誌中

在事務執行過程中，每條日誌記錄著數據修改的完整信息，如：

- Transaction Id (事務 id)
- Object Id (數據記錄 id)
- Before Value (修改前的值)，用於 undo 操作
- After Value (修改後的值)，用於 redo 操作

### Example

![Untitled](Logging%20Schemes%205794515f5cb24f87b0c7b6f5d7e5d2b6/Untitled%205.png)

- txn執行的過程中，會先更新WAL Buffer，再更新Buffer Pool的page
- commit時先將WAL Buffer寫到disk
- 最後將WAL Buffer和Buffer Pool清空

## WAL Implementation - Group Commit

每次事務提交時，DBMS 都必須將日誌記錄寫入disk，由於寫入disk這個動作用時較長，因此可以利用 group commit 來批量寫入從而均攤寫入disk成本

![Untitled](Logging%20Schemes%205794515f5cb24f87b0c7b6f5d7e5d2b6/Untitled%206.png)

通過以上的分析，我們可以發現，不同的 buffer pool policies 之間存在著運行時效率與數據恢復效率間的權衡：

![Untitled](Logging%20Schemes%205794515f5cb24f87b0c7b6f5d7e5d2b6/Untitled%207.png)

- 運行時效率：No-Steal + Force < Steal + No-Force
- 數據恢復效率：No-Steal + Force > Steal + No-Force

大部分數據庫更看重運行時效率，因此幾乎所有 DBMS 使用 No-Force + Steal 的 buffer pool policy。

- Andy說他記得最深會使用No-Steal + Force的DB是在1970年代的波多黎各，那時候波多黎各每隔1小時就會停電一次，所以需要常常recover

# Logging Schemes

---

在記錄操作信息時，通常有兩種方案：physical logging 和 logical logging。

- **physical logging**: 指的是記錄物理數據的變化，例如 git diff
- **logical logging:** 指的是記錄邏輯操作內容，如 UPDATE、DELETE 和 INSERT 等語句。

logical logging 的特點總結如下：

- 相比 physical logging，記錄的內容精簡
- 恢復時很難確定故障時正在執行的語句已經修改了哪部分數據
- 恢復時需要回放所有事務，這個過程可能很漫長

還有一種混合策略，稱為 **physiological logging**

- 這種方案不會像 physical logging 一樣記錄 xx page xx 偏移量上的數據發生 xx 改動
- 而是記錄 xx page 上的 id 為 xx 的數據發生 xx 改動
- 前者需要關心 data page 在disk上的佈局，後者則無需關心
- physiological logging是最常用的方法

舉例如下：

![Untitled](Logging%20Schemes%205794515f5cb24f87b0c7b6f5d7e5d2b6/Untitled%208.png)

physiological logging 也是當下最流行的方案。

# Checkpoints

---

如果我們放任 WAL 增長，它可以隨著新的操作執行而無限增長。

在故障恢復時，DBMS 需要讀取更多的日誌，執行更多的恢復和回滾操作。

為了避免這種情況出現，DBMS 需要週期性地記錄 checkpoint，即將所有日誌記錄和數據頁都持久化到存儲設備中，然後在日誌中寫入一條 `<CHECKPOINT>` 記錄，舉例如下：

![Untitled](Logging%20Schemes%205794515f5cb24f87b0c7b6f5d7e5d2b6/Untitled%209.png)

- 當 DBMS 發生崩潰時，所有在最新的 checkpoint 之前提交的事務可以直接忽略，如 T1。
- T2 和 T3 在 checkpoint 前尚未 commit。
    - T2 需要 redo，因為它在 checkpoint 之後，crash 之前commit了，即已經告訴用戶事務提交成功
    - T3 需要 undo，因為它在 crash 之前尚未 commit，即尚未告訴用戶事務提交成功。

實現 checkpoints 有需要考慮的問題：

1. 要保證 checkpoint 的正確性，我們需要暫停所有事務
2. 故障恢復時，掃描數據找到未提交的事務可能需要較長的時間
3. 如何決定 DBMS 執行 checkpoint 的週期
    - 太頻繁將導致運行時性能下降
    - 等太久將使得 checkpoint 的內容更多，更耗時，同時數據恢復也要消耗更長的時間

# Conclusion

---

WAL 幾乎永遠是最佳選擇

- Use incremental updates (**STEAL + NO-FORCE**) with checkpoints
- On recovery: undo uncommitted txns + redo committed txns