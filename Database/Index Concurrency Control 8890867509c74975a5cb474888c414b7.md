# Index Concurrency Control

在前兩節中討論的數據結構中，我們都只考慮單線程(single thread)的情況。實踐中，絕大多數情況下，我們需要在並發訪問(multiple threads)情況下保證我們的數據結構還能夠正常工作。除了 VoltDB，它用一個線程處理所有請求，徹底排除了並發的需要。

# Concurrency Control

通常我們會從兩個層面上來理解並發控制的正確性：

- Logical Correctness：（17 節）我是否能看到我應該要看到的數據？
- Physical Correctness：（本節）數據的內部表示是否安好？

## 本節大綱

- Latches Overview
- Hash Table Latching
- B+Tree Latching
- Leaf Scans
- Delayed Parent Updates

# Latch Overview

## Locks vs Latches

### Locks

- Protects the indexs logical contents from other transactions.
- Held for (mostly) the entire duration of the transaction.
- The DBMS needs to be able to rollback changes.

### Latches

- Protects the critical sections of the indexs internal data structure from other threads
- Held for operation duration
- The DBMS does not need to be able to rollback changes.

![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled.png)

## Latch Modes

### **Read Mode**

- 多個線程可以同時讀取相同的數據
- 針對相同的數據，當別的線程已經獲得處於 read mode 的 latch，新的線程也可以繼續獲取 read mode 的 latch

### **Write Mode**

- 同一時間只有單個線程可以訪問
- 針對相同的數據，如果獲取前已經有別的線程獲得任何 mode 的 latch，新的線程就無法獲取 write mode  的 latch

![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%201.png)

## Latch Implementations

- The underlying primitive that we can use to implement a latch is through an atomic compare-and-swap(CAS) instruction that modern CPUs provide. With this, a thread can check the contents of a memory location to see whether it has a certain value. If it does, then the CPU will swap the old value with a new one. Otherwise the memory location remains unmodified.
- There are several approaches to implementing a latch in a DBMS. Each approach have different tradeoffs in terms of engineering complexity and runtime performance. These test-and-set steps are performed atomically (i.e., no other thread can update the value after one thread checks it but before it updates it).

### Approach 1: Blocking OS Mutex

- Simple to use
- Non-scalable(不可擴展的) (about 25ns per lock/unlock invocation(調用))
- Example: std::mutex
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%202.png)
    
- OS mutex is generally a bad idea inside of DBMSs as it is managed by OS and has large overhead.

### Approach 2: Test-and-Set Spin Latch (TAS)

- Very efficient (single instruction to latch/unlatch)
- Non-scalable(不可擴展的), not cache friendly
- Example: std::atomic<T>
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%203.png)
    

### Approach 3: Reader-Writer Latch

- A Reader-Writer Latch allows a latch to be held in either read or write mode. It keeps track of how many threads hold the latch and are waiting to acquire the latch in each mode.
- 優點: 允許concurrent readers
- 缺點: Must manage read/write queues to avoid starvation
- Can be implemented on top of spinlocks
- 例子:
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%204.png)
    
    - 兩個read可以同時執行，後面的write需要等read都解鎖
    - 第三個read進來時，因為有write在等，為了避免starvation所以他等待write執行完才執行

# Hash Table Latching

- Easy to support concurrent access due to the limited ways threads access the data structure.
    - 所有的thread都往同個方向移動，且一個page/slot同時間只能被access一次
    - 因此不可能產生Deadlocks
- To resize the table, take a global latch on the entire table (i.e., in the header page).

## Approach #1: Page Latches

- 每個page都有他自己的reader-write latch去保護他全部的內容
- Threads acquire either a read or write latch before they access a page.

## Approach #2: Slot Latches

- 每個slot都有自己的latch.
- 可以使用single mode latch來降低meta-data overhead和computational overhead.

# B+Tree Latching

我們希望在最大程度上允許多個線程同時讀取、更新同一個 B+ Tree index，主要需要考慮兩種情況：

- 多個線程同時修改同一個 node
- 一個線程正在遍歷 B+ Tree 的同時，另一個線程正在 splits/merges nodes

舉例如下：

![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%205.png)

- T1 想要刪除 44，T2 想要查詢 41。
- 刪除 44 時，DBMS 需要 rebalance D 節點，將 H 節點拆分成兩個節點。
- 若在拆分前，T2 讀取到 D 節點，發現 41 在 H 節點，此時時間片輪轉到了 T1，T1 把 D 節點拆分成 H、I 兩個節點，同時把 41 轉移到 I 節點，之後 CPU 交還給 T2，T2 到 H 節點就找不到 41
- 如下圖所示：

![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%206.png)

- 如何解決該問題？由於 B+ Tree 是樹狀結構，有明確的訪問順序，因此很容易想到沿著訪問路徑加鎖的方法，即 Latch Crabbing/Coupling

## Basic Latch Crabbing/Coupling

Latch Crabbing 的基本思想如下：

- 獲取 parent 的 latch
- 獲取 child 的 latch
- 如果parent是**安全**的，就釋放 parent 的 latch

這裡的“安全”指的是，當發生更新操作時，該節點不會發生 split 或 merge 的操作，即：

- 在插入元素時，節點未滿
- 在刪除元素時，節點超過半滿

### Find

從 root 往下，不斷地：

- 獲取 child 的 read latch
- 釋放 parent 的 read latch
- 舉例如下：Find 38
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%207.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%208.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%209.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2010.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2011.png)
    

### **Insert/Delete**

從 root 往下，按照需要獲取 write latch，一旦獲取了 child 的 write latch，檢查它是否安全，如果安全，則釋放之前獲取的所有 write latch。

- 例 1 如下：Delete 38
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2012.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2013.png)
    
    coalesce: 合併
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2014.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2015.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2016.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2017.png)
    
- 例 2 如下：Insert 45
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2018.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2019.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2020.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2021.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2022.png)
    
- 例 3：Insert 25
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2023.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2024.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2025.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2026.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2027.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2028.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2029.png)
    

## Improved Lock Crabbing Protocol

在實際應用中：

- 更新操作每次都需要在路徑上獲取 write latch 容易成為系統並發瓶頸
- 通常 Node 的大小為一個 page(8KB/16KB)，可以存儲很多 keys，因此更新操作的出現頻率不算高

我們能否在 Latch Crabbing 的基礎上做得更好？

可以採用類似樂觀鎖的思想，一律假設 leaf node 是安全（更新操作僅會引起 leaf node 的變化）的，在查詢路徑上一路獲取、釋放 read latch，而不是write latch

到達 leaf node 時，若操作不會引起 split/merge 發生，則只需要在 leaf node 上獲取 write latch 然後更新數據，釋放 write latch 即可

若操作會引起 split/merge 發生，則重新執行一遍Basic Latch Crabbing，也就是在查詢路徑上一路獲取、釋放 write latch

- 舉例：Delete 38
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2030.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2031.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2032.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2033.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2034.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2035.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2036.png)
    

### Improved Lock Crabbing Protocol 小結

- Search：與 Latch Crabbing 相同
- Insert/Delete:
    - 使用與 Search 相同的方式在查詢路徑上獲取、釋放 latch，在 leaf node 上獲取 write latch
    - 如果 leaf node 不安全，可能會引起其它節點的變動，則使用Basic Latch Crabbing 的策略再執行一遍

該方法樂觀地假設整個操作只會引起 leaf node 的變化，若假設錯誤，則使用Basic Latch Crabbing。

# Leaf Node Scans

之前的分析中我們僅僅關注了從上到下的訪問模式，而沒有考慮到左右方向的訪問模式，在 range query 中，常常需要橫向訪問相鄰的 nodes。

- 例1：T1: Find Keys < 4
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2037.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2038.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2039.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2040.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2041.png)
    
    - Latch從C到B
- 例2：如果此時剛好有另一個Thread作相反方向的Access：T2: Find Keys > 1：
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2042.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2043.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2044.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2045.png)
    
    - 由於 read latch 允許被多次獲取，因此並發讀的情況不會產生負面影響。
- 例 3：T1: Delete 4，T2: Find Keys > 1
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2046.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2047.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2048.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2049.png)
    
    - 當遇到橫向掃描無法獲取下一個節點的 latch 時，該線程將釋放 latch 後自殺。這種策略邏輯簡單，儘管有理論上的優化空間，但在實務上是常見的避免死鎖的方式。

# Delayed Parent Updates

從上文中，我們可以觀察到：每當 leaf node 溢出時，我們都需要更新至少 3 個節點：

- 即將被拆分的 leaf node
- 新的 leaf node
- parent node

修改的成本較高，因此 **B-link Tree** 提出了一種優化策略：每當 leaf node 溢出時，只是標記一下而暫時不更新 parent node，等下一次有別的線程獲取 parent node 的 write latch 時，一併修改。

- 舉例如下：Insert 25
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2050.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2051.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2052.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2053.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2054.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2055.png)
    
    ![Untitled](Index%20Concurrency%20Control%208890867509c74975a5cb474888c414b7/Untitled%2056.png)
    

Delayed Parent Updates 僅用在 insert 操作中。

# Conclusion

讓一個data structure具備thread-safe的特點很困難，儘管本節只是介紹 B+ Tree 上的相關技術，但這些技術同樣適用於其他data structure。