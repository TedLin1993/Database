# Sorting & Aggregations

# Sorting

## Why Do We Need To Sort?

需要排序算法的原因：本質在於 tuples 在 table 中沒有順序，無論是用戶還是 DBMS 本身，在處理某些任務時希望 tuples 能夠按一定的順序排列，如：

- 若 tuples 已經排好序，刪除重複的資料將變得很容易(DISTINCT)
- 批量(Bulk loading)將排好序的 tuples 插入到 B+ Tree index 中，速度更快
- Aggregations (GROUP BY)

## Algorithms

若數據能夠放入內存中，我們可以使用標准排序演算法，如quick-sort；若數據無法放入內存中，就得考慮數據在 disk 與 memory 中移動的成本，傾向選擇sequential I/O 而不是 random I/O.

# **External Merge Sort**

Divide-and-conquer sorting algorithm: 將data set分成多個runs並分別對這些runs做排序，再將這些runs分別寫回disk，最後將runs合併成一個run

外部排序通常有兩個步驟：

- Sorting Phase：將數據分成多個 runs，每個 run可以完全讀入到 memory 中，在 memory 中排好序後再寫回到 disk 中
- Merge Phase：將多個sub-files合併成一個file

## **2-Way External Merge Sort**

以下是 2-way external merge sort 的一個簡單例子，假設：

- Data set分成 $N$ 個 pages
- DBMS 有 $B$ 個 fixed-size buffer pages去保存input/output data

### Pass #0

- 從 table 中讀入 B 個pages
- 將這些pages排序後寫回到 disk 中
- 每一個排序過的page就稱作一個 run

### Pass #1,2,3,...

- 遞歸地將一對 runs 合併成一個兩倍長度的 run
- 這一操作值需要 3 個 buffer pages ( 2 個用於輸入，1個用於輸出)

![Untitled](Sorting%20&%20Aggregations%20d9f2aefac2584aad9fe18f76ccb4dc77/Untitled.png)

完整過程如下圖所示：

![Untitled](Sorting%20&%20Aggregations%20d9f2aefac2584aad9fe18f76ccb4dc77/Untitled%201.png)

複雜度：

- number of passes: $1+\lceil(\log_2(N))\rceil$
    - pass #0作一次內部排序，其他pass開始兩兩合併
- cost/pass: I/O 成本為$2N$ ，系數 2 表示讀入 + 寫出
- total cost: $2N×(number of passes)$

值得注意的是：

- 這個算法只需要 3 個 buffer pages，B=3 (2 input, 1 output)
- 即使 DBMS 能夠提供更多的 buffer pages（B>3），2-way external merge sort 也無法充分地利用它們

如何能夠利用到更多的 buffer pages ？

## Double Buffering Optimization

- Prefetch the next run in the background and store it in a second buffer while the system is processing the current run.
- This reduces the wait time for I/O requests at each step by continuously utilizing the disk.

![Untitled](Sorting%20&%20Aggregations%20d9f2aefac2584aad9fe18f76ccb4dc77/Untitled%202.png)

## **General External Merge Sort**

將 2-way external merge sort 泛化成 N-Way 的形式：

### Pass #0

- 使用 B 個 buffer pages
- 產生$\lceil(N/B)\rceil$個大小為 B 的 sorted runs

### Pass #1,2,3,...

- 合併 B-1 runs

複雜度：

- number of passes: $1+\lceil\log_{B−1}\lceil(N/B)\rceil\rceil$
- cost/pass: $2N$
- total I/O cost: $2N×(number of passes)$

**實例：Sort 108 pages file with 5 buffer pages：N = 108, B = 5**

- Pass #0: $\lceil108/5\rceil=22$ sorted runs of 5 pages each
- Pass #1: $\lceil22/4\rceil=6$ sorted runs of 20 pages each
- Pass #2: $\lceil6/4\rceil=2$ sorted runs of 80 pages and 28 pages each
- Pass #3: sorted file of 108 pages

一共有: $1+\lceil\log_{B−1}{\lceil}{N\over B}\rceil\rceil=1+\lceil\log_422\rceil=4 passes$

## **Using B+ Trees**

如果被排序的表在對應的 attribute(s) 上已經建有索引，我們就可以用它來加速排序的過程，按照目標順序遍歷 B+ Tree 的 leaf pages 即可，但這裡要注意有兩種情況：

- Clustered B+ Tree
- Unclustered B+ Tree

### case 1: Clustered B+ Tree

![Untitled](Sorting%20&%20Aggregations%20d9f2aefac2584aad9fe18f76ccb4dc77/Untitled%203.png)

- 從最左邊的leaf page讀到最右邊
- 這種情況永遠優於 external sorting，因為這是sequential access

### case 2: Unclustered B+ Tree

![Untitled](Sorting%20&%20Aggregations%20d9f2aefac2584aad9fe18f76ccb4dc77/Untitled%204.png)

- 需要先取得所有page的pointer
- 這是最糟糕的情況，因為獲取每個 data record 的過程都可能需要一次 I/O。

# Aggregations

aggregation 就是對一組 tuples 的某些值做統計，轉化成一個標量，如平均值、最大值、最小值等，aggregation 的實現通常有兩種方案：

- Sorting
- Hashing

Hashing通常表現得較好

## Sorting Aggregation

![Untitled](Sorting%20&%20Aggregations%20d9f2aefac2584aad9fe18f76ccb4dc77/Untitled%205.png)

但很多時候我們並不需要排好序的數據，如：

- Forming groups in GROUP BY
- Removing duplicates in DISTINCT

在這樣的場景下 hashing 是更好的選擇，它能有效減少排序所需的額外工作。

## Hashing Aggregation

利用一個臨時 (ephemeral) 的 hash table 來記錄必要的信息，即檢查 hash table 中是否存在已經記錄過的元素並作出相應操作：

- DISTINCT: Discard duplicate
- GROUP BY: Perform aggregate computation

ephemeral hash table只會在一次query中使用，query完就丟掉

如果所有信息都能一次性讀入內存，那事情就很簡單了，但如若不然，我們就得變得更聰明。

hashing aggregation 同樣分成兩步：

### Phase #1: Partition

將 tuples 根據 hash key 放入不同的 buckets(稱作partition)，當他們滿了之後寫回disk

- use a hash function $h_1$ to split tuples into partition on disk
    - hash結果相同的data會放在同個partition
    - partitions are "spilled" to disk via output buffers
- 假設可用的buffer page有$B$ 個，其中的$B-1$ buffers是給partions的，$1$ buffer是給input data的

![Untitled](Sorting%20&%20Aggregations%20d9f2aefac2584aad9fe18f76ccb4dc77/Untitled%206.png)

### Phase #2: ReHash

在內存中針對每個在disk的partition:

- 將partition讀入memory，並建立一個in-memory hash table，這個table基於hash function $h_2$
- 利用 hash table 計算 aggregation 的結果

這個做法是建立在假設memory的空間能夠讀入每個partition

如下圖所示：

![Untitled](Sorting%20&%20Aggregations%20d9f2aefac2584aad9fe18f76ccb4dc77/Untitled%207.png)

在 ReHash phase 中，存著$(GroupKey→RunningVal)$的鍵值對應，當我們需要向 hash table 中插入新的 tuple 時：

- 如果我們發現相應的 GroupKey 已經在內存中，只需要更新 RunningVal 就可以
- 反之，則插入新的 GroupKey → RunningVal

![Untitled](Sorting%20&%20Aggregations%20d9f2aefac2584aad9fe18f76ccb4dc77/Untitled%208.png)

### **Cost Analysis**

使用 hashing aggregation 可以聚合多大的 table ？假設有 B 個 buffer pages

- Phase #1：使用 1 個 page 讀數據，B-1 個 page 寫出 B-1 個 partition 的數據
- 每個 partition 的數據應當小於 B 個 pages

因此能夠聚合的 table 最大為$B×(B−1)$

通常一個大小為 N pages 的 table 需要大約$\sqrt N$個buffer pages