# Buffer Pools

# 大綱

上節中提到，DBMS 的磁盤管理模塊主要解決兩個問題：

1. 如何使用磁盤文件來表示數據庫的數據（元數據、索引、數據表等）
2. （本節）如何管理數據在內存與磁盤之間的移動

本節將討論第二個問題。管理數據在內存與磁盤之間的移動又分為兩個方面：空間控制（Spatial Control）和時間控制（Temporal Control）

## **Spatial Control**

空間控制策略通過決定將 pages 寫到磁盤的哪個位置，使得常常一起使用的 pages 能離得更近，從而提高 I/O 效率。

## **Temporal Control**

時間控制策略通過決定何時將 pages 讀入內存，寫回磁盤，使得讀寫的次數最小，從而提高 I/O 效率。

整個 big picture 如下圖所示：

![Untitled](Buffer%20Pools%209dd4fa0940614d42a1ea3a457cfd0086/Untitled.png)

## 本節提要

- Buffer Pool Manager
- Replacement Policies
- Allocation Policies
- Other Memory Pools

# Buffer Pool Manager

DBMS 啟動時會從 OS 申請一片內存區域，即 Buffer Pool，並將這塊區域劃分成大小相同的 pages，為了與 disk pages 區別，通常稱為 frames，當 DBMS 請求一個 disk page 時，它首先需要被複製到 Buffer Pool 的一個 frame 中，如下圖所示：

![Untitled](Buffer%20Pools%209dd4fa0940614d42a1ea3a457cfd0086/Untitled%201.png)

同時 DBMS 會維護一個 page table，負責記錄每個 page 在內存中的位置，以及是否被寫過（Dirty Flag），是否被引用或引用計數（Pin/Reference Counter）等元信息，如下圖所示：

![Untitled](Buffer%20Pools%209dd4fa0940614d42a1ea3a457cfd0086/Untitled%202.png)

當 page table 中的某 page 被引用時，會記錄引用數（pin/reference），表示該 page 正在被使用，空間不夠時不應該被移除；當被請求的 page 不在 page table 中時，DBMS 會先申請一個 latch（lock 的別名），表示該 entry 被佔用，然後從 disk 中讀取相關 page 到 buffer pool，釋放 latch。以上兩種過程如下圖所示：

![Untitled](Buffer%20Pools%209dd4fa0940614d42a1ea3a457cfd0086/Untitled%203.png)

## Locks vs. Latches

- Locks
    - high level，user可以控制
    - Protect the database logical contents from other transactions
    - Held for transaction duration
    - Need to be able to rollback changes
    - Protect tuples, tables, indexes
- Latches
    - low level，user需要透過lock來控制，是lock的實作，在OS會將latch稱作為lock
    - Protects the critical sections of the DBMS internal data structure from other threads
    - Held for operation duration
    - Do not need to be able to rollback changes

## Page Table VS. Page Directory

- The page directory is the mapping from page ids to page locations in the database files.
    - All changes must be recorded on disk to allow the DBMS to find on restart.
- The page table is the mapping from page ids to a copy of the page in buffer pool frames.
    - This is an in-memory data structure that does not need to be stored on disk.

## Multiple Buffer Pools

為了減少並發控制的開銷(lock)以及利用數據的 locality，DBMS 可能在不同維度上維護多個 Buffer Pools：

- 多個 Buffer Pools 實例
- 每個 Database 分配一個 Buffer Pool
- 每種 Page 類型分配一個 Buffer Pool

## Pre-Fetching

DBMS 可以通過查詢計劃來預取 pages，如：

- Sequential Scans: DB預知user會按照順序讀取Disk pages，所以先將後面的page先load到Buffer Pool
    
    ![Untitled](Buffer%20Pools%209dd4fa0940614d42a1ea3a457cfd0086/Untitled%204.png)
    
- Index Scans
    
    ![Untitled](Buffer%20Pools%209dd4fa0940614d42a1ea3a457cfd0086/Untitled%205.png)
    

## Scan Sharing

Scan Sharing 技術主要用在多個查詢存在數據共用的情況。當兩個查詢 A, B 先後發生，B 發現自己有一部分數據與 A 共用，於是先共用 A 的 cursor，等 A 掃完後，再掃描自己還需要的其它數據。

- 支援的DB: MSSQL, DB2, Oracle(只支援cursor sharing)

## Buffer Pool Bypass

當遇到大數據量的 Sequential Scan 時，如果將所需 pages 順序存入 Buffer Pool，將造成後者的污染，因為這些 pages 通常只使用一次，而它們的進入將導致一些可能在未來更需要的 pages 被移除。因此一些 DBMS 做了相應的優化，在這種查詢出現時，為它單獨分配一塊局部內存，將其對 Buffer Pool 的影響隔離。

## OS Page Cache

大部分 disk operations 都是通過OS API調用，通常OS會copy一份cache，這會導致一份數據分別在OS和 DMBS 中被cache兩次。大多數 DBMS 都會使用 (O_DIRECT) 來告訴 OS 不要cache這些數據，除了 Postgres。

- Postgres也會使用OS cache，所以若Postgres DB重啟再撈相同的資料時，因為OS有cache過這些資料，所以Postgres可以不用做disk IO去取得資料

# Buffer Replacement Policies

當 Buffer Pool 空間不足時，讀入新的 pages 必然需要 DBMS 從已經在 Buffer Pool 中的 pages 選擇一些移除，這個選擇就由 Buffer Replacement Policies 負責完成。它的主要目標是：

- Correctness：操作過程中要保證髒數據同步到 disk
- Accuracy：盡量選擇不常用的 pages 移除
- Speed：決策要迅速，每次移除 pages 都需要申請 latch，使用太久將使得並發度下降
- Meta-data overhead：決策所使用的meta-data佔用的量不能太大

## LRU (Least Recently Used)

維護每個 page 上一次被訪問的timestamp，每次移除timestamp最早的 page。

## Clock

Clock 是 LRU 的近似策略，它不需要每個 page 上次被訪問的timestamp，而是為每個 page 保存一個 reference bit

每當 page 被訪問時，reference bit 設置為 1

每當需要移除 page 時，從上次訪問的位置開始，按順序輪詢每個 page 的 reference bit，若該 bit 為 1，則重置為 0；若該 bit 為 0，則移除該 page

## LRU 與 Clock 的問題

二者都容易被 sequential flooding 現象影響，也就是最近被訪問的 page 實際上卻是最不可能需要的 page。為了解決這個問題，又提出了 LRU-K 、Localization、Priority Hints等策略。

## LRU-K

LRU-K 保存每個 page 的最後 K 次訪問時間戳，利用這些時間戳來估計它們下次被訪問的時間，通常 K 取 1 就能獲得很好的效果。

## Localization

DBMS 針對每個查詢做出移除 pages 的限制，使得這種影響被控制在較小的范圍內，類似 API 的 rate limit。

## Priority Hints

DBMS 知道每個 page 在查詢執行過程中的上下文信息，因此它可以根據這些信息判斷一個 page 是否重要。

例如:

![Untitled](Buffer%20Pools%209dd4fa0940614d42a1ea3a457cfd0086/Untitled%206.png)

- 透過這個insert指令可以知道tree的右邊比較有機會被使用，因此tree的右邊的priority較高

## Dirty Pages

移除一個 dirty page 的成本要高於移除一般 page，因為前者需要寫 disk，後者可以直接 drop，因此 DBMS 在移除 page 的時候也需要考慮到這部分的影響。除了直接在 Replacement Policies 中考慮，有的 DBMS 使用 Background Writing 的方式來處理。它們定期掃描 page table，發現 dirty page 就寫入 disk，在 Replacement 發生時就無需考慮髒數據帶來的問題。

# Allocation Policies

Allocation Policies 指 DBMS 如何為不同的查詢分配內存，可以分為 Global Policies 和 Local Policies。

- Global Policies: 同時考慮所有查詢(txn)來分配內存，
- Local Policies: 為單個查詢分配內存時不考慮其它查詢的情況。

# Other Memory Pools

除了存儲 tuples 和 indexes，DBMS 還需要 Memory Pools 來存儲其它數據，如：

- Sorting + Join Buffers
- Query Caches
- Maintenance Buffers
- Log Buffers
- Dictionary Caches