# Join Algorithms

在關聯型資料庫中，我們常常通過正規化 (Normalization) 設計將資料切分成不同的table避免資料重複；因此查詢時，就需要通過 Join 將不同 table 中的數據合併來重建數據。

以課程一開始時的 table 為例，通過將 Artist 與 Album 之間的多對多關系拆成 Artist, ArtistAlbum 以及 Album 三個 tables 來正規化資料，使得資料存儲的冗餘減少：

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled.png)

查詢時我們就需要通過 Join 來重建 Artist 與 Album 的完整關系數據。

# Joins

本課主要討論透過inner equijoin來合併兩個 tables 

- inner equijoin可以被用來支援其他的join方法

總的來說，在query plan，我們希望較小的table總是被當作left table("outer table")

- 也就是右圖的$R$

首先需要討論的是：

- Output: Join 的輸出
- Cost Analysis Criteria: Join 的成本分析

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%201.png)

## Operator Output

邏輯上 Join 的操作的結果是：對任意一個 tuple $r∈R$ 和任意一個在 Join Attributes 上對應的 tuple $s∈S$，將 r 和 s 串聯成一個新的 tuple：

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%202.png)

Join 操作的結果 tuple 中除了 Join Attributes 之外的信息與多個因素相關：

- query processing model
- storage model
- query

### Operator Output: Data

我們可以在 Join 的時候將所有非 Join Attributes 都放入新的 tuple 中，這樣 Join 之後的操作都不需要從 tables 中重新獲取數據：

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%203.png)

### Operator Output: Record Ids

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%204.png)

在 Join 的時候只複製 Join Attributes 以及 record id，後續操作自行根據 record id 去 tables 中獲取相關數據。對於列存儲(column stores)資料庫，可能是理想的處理方式，被稱為 Late Materialization(延遲實現)。

## I/O Cost Analysis

由於資料庫中的資料量通常較大，無法一次性載入內存，因此 Join Algorithm 的設計目的，在於減少磁盤 I/O，因此我們衡量 Join Algorithm 好壞的標准，就是 I/O 的數量。

- **Cost Metric: # of IOs to compute join**

此外我們不需要考慮 Join 結果的大小，因為不論使用怎樣的 Join Algorithm，結果集的大小都一樣。

以下的討論都建立在這樣的情景上：

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%205.png)

- 對 R 和 S 兩個 tables 做 Join
- table R 中有 M 個 pages，m 個 tuples
- table S 中有 N 個 pages，n 個 tuples

## Join vs Cross-Product

- $**R⨝S**$ is the most common operation and thus must be carefully optimized.
- $**R×S$** followed by a selection is inefficient because the cross-product is large.
    - 就只是兩個for loop將兩個table的資料相乘
- There are many algorithms for reducing join cost, but no algorithm works well in all scenarios.

## Join Algorithms

本節要介紹的 Join Algorithms 羅列如下：

- Nested Loop Join
    - Simple
    - Block
    - Index
- Sort-Merge Join
- **Hash Join:** 最重要也最快的join方式

不同的 Join Algorithms 有各自的適用場景，需要具體問題具體分析。

# Nested Loop Join

## **Simple Nested Loop Join**

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%206.png)

對 R 中的每個 tuple，都全表掃描一次 S，是一種暴力解法

- **Cost: $M+(m×N)$**

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%207.png)

***舉例：***

假設：

- Table R: M = 1000， m = 100,000
- Table S: N = 500, n = 40,000

成本：

- M+(m×N)=1000+(100000×500)=**50,000,100 I/Os**
- 假設 0.1 ms/IO，則總時長約為 1.3 小時

如果我們使用小表 S 作為 Outer Table，那麼：

- N+(n×M)=500+(40000×1000)=**40,000,500 I/Os**
- 則總時長約為 1.1 小時。

## Block Nested Loop Join

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%208.png)

每次取 R 中一個 block 的所有 tuples 出來，讓它們同時與 S 中的所有 tuples Join 一次，因為scan一個block只需要一次I/O, 不需要每個tuple都scan一次

- **Cost: $M+(M×N)$**

***舉例：***

假設：

- Table R: M = 1000， m = 100,000
- Table S: N = 500, n = 40,000

成本：

使用大表 M 作為 Outer Table，成本為：

- M+(M×N)=1000+(1000×500)=**501,000 I/Os**
- 總共用時約 50 秒。

使用小表 S 作為 Outer Table，成本為：

- N+(N×M)=500+(1000×500)=**500,500 I/Os**

以上的計算都假設 DBMS 只為 Nested Loop Join Algorithm 分配 3 塊 buffers，其中 2 塊用於讀入，1 塊用於寫出；若 DBMS 能為算法分配 B 塊 buffers，則可以使用 B-2 塊來讀入 Outer Table，1 塊用於讀入 Inner Table，1 塊用於寫出

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%209.png)

- **Cost: $M+(\lceil M/(B−2)\rceil×N)$**

如果 Outer Table 能夠直接放入內存中(**B>M+2**)，則

- **Cost: M+N**

## **Index Nested Loop Join**

之前的兩種 Nested Loop Join 速度慢的原因在於，需要對 Inner Table 作多次全表掃描，若 Inner Table 在 Join Attributes 上有索引或者臨時建一個索引 (只需全表掃描一次)：

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%2010.png)

此時 Join 的成本為：

- **Cost: $M+(m×C)$**
- 其中 C 為 index probe 的成本。

## **小結**

從上面的討論中，我們可以導出以下幾個結論：

- 總是選擇小表作為 Outer Table
- 盡量多地將 Outer Table 緩存在內存中
- 掃描 Inner Table 時，盡量使用索引

# Sort-Merge Join

Sort-Merge Join 顧名思義，分為兩個階段：

- Phase #1: Sort
    - 根據 Join Keys 對 tables 進行排序
    - 可以使用external merge sort algorithm
- Phase #2: Merge
    - 同時從兩個 tables 的一端開始掃描，對 tuples 配對
    - 如果 Join Keys 並不唯一，則有可能需要 backtrack

算法如下：

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%2011.png)

Sort Merge 的成本分析如下：

- **Sort Cost (R): $2M×(logM/logB)$**
- **Sort Cost (S): $2N×(logN/logB)$**
- **Merge Cost: $M+N$**
- **Total Cost: Sort + Merge**

舉例：

假設：

- Table R: M = 1000, m = 100,000
- Rable S: N = 500, n = 40,000
- B = 100
- 0.1ms/IO

成本：

- Sort Cost (R): $2000 \times (log 1000 / log 100) = 3000 \ I/Os$
- Sort Cost (S): $1000×(log500/log100)=1350 \ I/Os$
- Merge Cost: $1000+500=1500 I/Os$
- Total Cost = $3000+1350+1500=5850 I/Os$
- Total Time = 0.59 secs

Sort-Merge Join 的最壞情況就是當兩個 tables 中的所有 Join Keys 都只有一個值，這時候 Join 的成本變為：$M×N+sort cost$

- 實務上不太可能出現這種情況

## **小結**

Sort-Merge Join 適用於：

- 當 tables 中的一個或者兩個都已經按 Join Key 排好序，這樣就沒有Sort Cost了
- SQL 的輸出必須按 Join Key 排好序

# Hash Join

核心思想：

- 如果分別來自 R 和 S 中的兩個 tuples 滿足 Join 的條件，它們的 Join Attributes 必然相等，那麼它們的 Join Attributes 經過某個 hash function 得到的值也必然相等，因此 Join 的時候，我們只需要對兩個 tables 中 hash 到同樣值的 tuples 分別執行 Join 操作即可。

## **Basic Hash Join Algorithm**

本算法分為兩個階段：

- Phase #1: Build
    - 掃描 Outer Table，使用 hash function $**h_1**$對 Join Attributes 建立 hash table $T$
- Phase #2: Probe
    - 掃描 Inner Table，使用 hash function $h_1$獲取每個 tuple 在 T 中的位置，在該位置上找到配對成功的 tuple(s)

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%2012.png)

Hash Table T 的內容：

- Key：Join Attributes
- Value：根據不同的查詢要求及實現來變化
    - Full Tuple：可以避免在後續操作中再次獲取數據，但需要佔用更多的空間
    - Tuple Identifier：是Column store資料庫的理想選擇，佔用最少的空間，但之後需要重新獲取數據

但 Basic Hash Join Algorithm 有一個弱點，就是有可能 T 無法被放進內存中，由於 hash table 的查詢一般都是random access，因此在 Probe 階段，T 可能在 memory 與 disk 中來回移動。

# **Grace Hash Join**

當兩個 table 都無法放入 memory 時，我們可以：

- Phase #1: Build
    - 將兩個 tables 使用同樣的 hash function，partition到bucket hash table，使得可能配對成功的 tuples 進入到相同的bucket
    - 這些bucket需要寫到disk
- Phase #2: Probe
    - 對兩個 tables 的每一對 bucket分別進行 Join
    - 其中有可能會match的tuple都只會在同一對bucket上，所以不需要查其他bucket是否有同樣的tuple

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%2013.png)

如果每個 bucket 仍然無法放入內存中，則可以遞迴使用不同的 hash function 進行 partition，即 **recursive partitioning**：

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%2014.png)

成本分析：

假設我們有足夠的 buffers 能夠存下中間結果：

- Partitioning Phase:
    - Read + Write both tables
    - 2(M+N) I/Os
- Probing Phase
    - Read both tables
    - M+N I/Os
- **Total Cost: 3(M+N)**

***舉例：***

假設：

- M = 1000, m = 100,000
- N = 500, n = 40,000
- 0.1ms/IO

計算：

- $3 \times (M + N) = 4,500 \ I/Os$
- 0.45 secs

如果 DBMS 已經知道 tables 大小，則可以使用 static hash table，否則需要使用 dynamic hash table

# **Summary**

![Untitled](Join%20Algorithms%20ece3e52d7d6b411ca24fffef5b40a6a3/Untitled%2015.png)

# 總結

Hash Join 在絕大多數場景下是最優選擇，但當查詢包含 ORDER BY 或者數據極其不均勻的情況下，Sort-Merge Join 會是更好的選擇，DBMSs 在執行查詢時，可能使用其中的一種到兩種方法。