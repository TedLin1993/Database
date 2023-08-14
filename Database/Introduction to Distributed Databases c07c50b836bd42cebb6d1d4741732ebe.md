# Introduction to Distributed Databases

當我們在談論分佈式資料庫時，首先要明確什麼是分佈式系統：如果通信成本和通信的可靠性問題不可忽略，就是分佈式系統。

這也是區分 Parallel DBMS 和 Distributed DBMS 的依據所在

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled.png)

那麼如何利用我們在這節課中介紹的單節點(single node) DBMS 的知識，構建支持事務的 Distributed DBMS 呢？之前討論過的很多話題：

- Optimization & Planning
- Concurrency Control
- Logging & Recovery

在分佈式環境下都可能遇到新的挑戰。

# System Architecture

---

Distributed DBMS 的系統架構主要指的是要在哪一層上共享資源，主要分為以下 4 種：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%201.png)

實際上 Shared Everything 就沒有分佈式可言了，因此嚴格來說，只有 Shared Memory、Shared Disk 和 Shared Disk 三種。

## Shared Memory

---

在 Shared Memory 架構下，不同的 CPU 通過網絡訪問同一塊內存空間，每個 CPU 中都能看到所有內存數據結構，每個 DBMS 實例都知道其它實例的所有情況。

這種架構實際上只出現在一些大型機上，例如超級電腦，在雲原生環境下幾乎見不到。

## Shared Disk

---

在 Shared Disk 架構下，不同的 CPU 通過網路訪問同一塊logical disk，但各自都有自己的memory。

這種架構的好處就是計算層(execution layer)和儲存層(storage layer)可以獨立擴容(scale)，壞處就是 CPU 之間需要通過更多的通信來傳遞數據和元信息，因為每一次資料改動都必須要通信告訴其它Node。

採用這種架構的資料庫有很多，羅列如下圖所示：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%202.png)

現在cloud-base DBMS主要都是這個架構，Nodes have their own buffer pool and are considered stateless. A node crash does not affect the state of the database since that is stored separately on the shared disk. The storage layer persists the state in the case of crashes.

但 Shared Disk 的壞處在於 DBMS 對儲存層沒有控制權，無法決定數據的分佈，因此在查詢數據時無法達到最優的性能。

一個 Shared Disk 的 Distributed DBMS 舉例如下：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%203.png)

- 假設有兩個計算節點，客戶端想要獲取 Id 為 101 的數據，它可以從任意計算節點訪問。

如果計算節點不足時，可以按需擴容：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%204.png)

但如果客戶端想要修改某條數據：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%205.png)

- 由於其它節點可能正緩存著 Page ABC，更新節點需要通過某種方式通知其它節點 Page ABC 已經被修改。
- 由於既有的統一儲存層(storage layer)不會再增加這樣一層 Pub/Sub 邏輯，這樣成本太高了，那麼這種更新傳播的邏輯就必須在計算層實現，且我們無法假設節點間的網路通信是可靠的，這也是 Shared-Disk 架構需要考慮的重要問題。

## Shared Nothing

---

Shared Nothing，顧名思義，每個 CPU 都擁有獨立的內存、磁盤，節點之間僅通過網絡通信共享消息。

這種架構下，擴容和保證一致性的難度將變得很大。擴容時，DBMS 需要在不同節點間遷移、均衡數據，同時要保證服務在線，且數據一致，可以想像其複雜性。

但解決這個難題帶來的好處也很可觀，整個 DBMS 的性能將得到提升，因為資料庫可以控制存儲層本身，就能夠獨立根據數據的局部性(locality)原理排列數據，從而最大程度上減少每次查詢所消耗的通信成本。

使用 Shared Nothing 架構的資料庫有很多，羅列如下：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%206.png)

一個 Shared Nothing 的 Distributed DBMS 需要將數據分片到不同的節點上，每個節點擁有整個資料庫的一小部分，如果查詢所需的數據只落在一個節點上，就與單節點資料庫無二，如下圖所示：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%207.png)

- ID 為 1-150 之間的數據落在上面的節點
- ID 為 151-300 之間的數據落在下面的節點

像 “哪個節點儲存哪些範圍的數據” 這樣的信息會有一個配置中心來儲存。如果客戶端要查詢 Id=200 的數據，那麼只需要訪問下面的節點即可。如果客戶端要同時獲取 Id=10 和 Id=200 的數據，事情就會變得更複雜一些：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%208.png)

- 如上面的節點接收到請求，那麼它要麼將請求轉發給下面的節點，要麼從下面的節點讀取響應的數據，然後在內部同時處理兩個請求，這裡就有一些設計決定需要做。

如果 DBMS 的容量不夠，就需要做在線擴容：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%209.png)

這時候需要將上下兩個節點中的一部分數據遷移到中間節點，平衡所有節點的數據，這裡也有許多問題需要解決。

# Early Distributed Database Systems

---

事實上，分佈式資料庫並不是新概念。早在 1979 年，Andy Pavlo 的導師 Michael Stonebraker 就開發了 Muffin 資料庫，即 Ingress 的分佈式版本

SDD-1 (1979) 是 Phil Bernstein 在分佈式資料庫上的原型系統，並沒有真正在正式環境中使用，但許多關於分佈式事務的觀點和論文都是 Phil 基於 SDD-1 提出的

System R* (1984) 是 IBM Research 的內部項目，是 System R 的分佈式版本，但並沒有進入市場，然而後繼者 DB2 實際上有相應的分佈式版本

Gamma (1986) 是 Dewitt 實現的高性能分佈式資料庫

NonStop SQL 是 Jim Gray 基於高容錯系統設計的一款分佈式資料庫系統，也是上述系統中唯一進入商用的系統，現在仍然運行在許多大型金融系統中。

# Design Issues

---

剛才已經提到 Distributed DBMS 的系統架構以及可能遇到的問題，本節就來討論這些設計問題：

- 應用如何獲取數據？
- 如何在分佈式資料庫上執行查詢？
    - Push query to data
    - Pull data to query
- DBMS 如何保證正確性？

## Homogeneous VS. Heterogeneous

---

如果系統中的每個節點角色、權限相同，那就是同質節點 (Homogeneous Node)，如果不同，就是異質節點(Heterogeneous Node)

同質節點方案中，每個節點可以執行的任務集合是相同的，只是持有的數據不同，在處理擴容和故障恢復時比較簡單

異質節點方案中，每個節點有各自的節點類型，可以執行的任務不同，允許在一個物理節點運行多個虛擬節點，執行不同任務。

以 MongoDB 為例：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%2010.png)

- 在 MongoDB 集群(cluster)中有 3 種角色，Router、Config Server 以及 Shard
- 所有請求都打到 Router 上
- Router 從 Config Server 中獲取路由信息，即哪些數據存放在哪些分片(shared)上，然後根據這些路由信息將請求發送到對應的分片上執行。

Distributed DBMS 的用戶不應該知道數據具體儲存的地點，或者table本身是如何分片(**partitioned**)和複製(**replicated**)的，對於用戶來說，一個 SQL 在 Distributed DBMS 上運行的效果應該和在單節點 DBMS 上運行的效果等價。

# Database Partitioning

既然要做 Distributed DBMS，勢必要將資料庫的資源分佈到多個節點上，如磁盤、內存、CPU，這就是廣義的分片，Partitioning 或 Sharding。

DBMS 需要在各個分片上執行查詢的一部分，然後將結果整合後得到最終答案。本節我們來關注數據如何在磁盤上分片。

## Naive Table Partitioning

---

假設單個節點有足夠的容量存儲單張表，我們可以簡單地讓每個節點只存儲一張表：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%2011.png)

如果只存在單表查詢，這種方案是最理想的。但問題也很多，如數據分佈不均勻，只能在應用層做 Join 等等。

## Horizontal Partitioning

---

第二種就是我們常用的橫向分片(Horizontal Partitioning)，將一張表的數據切分成多個不相交的子集。這種分片方式要求 DBMS 要找到在大小、負載上能均勻分配的鍵。常用的分片方案如下：

- Hash Partitioning：計算哈希值後分片
- Range Partitioning：直接按號段分片

具體的分片案例如下：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%2012.png)

對於這種分片方案，最理想的查詢就是按 partitioning Key 來查詢數據。

- Hash Partitioning若今天要多一個Partition讓table數據重新分配，會導致其它partition的數據都整個大洗牌，因為hash function整個都不一樣了
- 解決方式就是一致性哈希 (Consistent Hashing)

### Consistent Hashing

Hash Key的範圍只有0~1，所有partition的key都落在這個範圍，hash(key)的結果落在不同範圍就是給不同的partition

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%2013.png)

若新增新的partition，就在原本指向的partition再指過去

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%2014.png)

新增多個partition的示意圖

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%2015.png)

主要使用的DB: Memcache、Cassandra、DynamoDB

## Logical Partitioning

在 Shared Nothing 架構下，通常是物理分片(physical partitioning)：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%2016.png)

在 Shared Disk 架構下，通常是邏輯分片(Logical Partitioning)，Node不能控制實際資料在disk的位置：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%2017.png)

# Transaction Coordination

---

對於 Distributed DBMS 來說，如果一個寫事務只涉及單個節點上的數據，那 DBMS 無需關心在其它節點上並發執行的事務狀態

如果涉及多個節點數據，就需要去協調多個節點上的事務執行方案，即所謂的 Transaction Coordination

Transaction Coordination主要有兩種方案：中心化 (Centralized) 和去中心化 (Decentralized)。

## Centralized Coordinator

---

### TP Monitor

實現 Centralized Coordinator 的其中一種思路就是構建一個獨立的組件負責管理事務，叫 Transaction Processing Monitor，即 TP Monitor。

TP Monitor 與其之下運行的單節點 DBMS 無關，DBMS 無需感知 TP Monitor 的存在。

每次應用在發送事務請求時，需要先通過 TP Monitor 確認事務是否可以執行。

舉例如下：

- 假設一個 DBMS 有 4 個分片，應用需要通過一個事務修改 P1、P3、P4 上的數據，首先需要從 Coordinator 上請求 P1、P3、P4 的鎖，
    
    ![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%2018.png)
    
- 拿到鎖後，應用就可以到 P1、P3、P4 上修改數據 (未提交)，修改完畢後再向 Coordinator 發送 Commit 請求，Coordinator 詢問各個分片剛才的修改是否可以安全地提交，可以就提交，然後 Coordinator 返回 Ack：
    
    ![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%2019.png)
    

許多資料庫廠商也出售類似 TP Monitor 的產品，如 Apache Omid、IBM Transac 等等。

### Middleware

實現 Centralized Coordinator 的另一種方案是 Middleware

對於應用來說，Middleware 就是 Distributed Database 本身，Middleware 負責與後面的所有分片互動，協調事務的執行。

如下圖所示：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%2020.png)

Facebook 運行著世界上最大的 MySQL 集群，採用的就是這種方案。它們的 Middleware 負責處理分佈式事務、路由、分片等所有邏輯。

## Decentralized Coordinator

---

Decentralized Coordinator 的基本思路就是，執行某個事務時，會選擇一個分片充當 Master，後者負責詢問涉及事務的其它分片是否可以執行事務，完成事務的提交或中止：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%2021.png)

### Distributed Concurrency Control

---

分佈式並發控制的難度在於：

- Replication
- Network Communication Overhead
- Node Failures
- Clock Skew

舉個例子，假如我們要實現分佈式的 2PL：

![Untitled](Introduction%20to%20Distributed%20Databases%20c07c50b836bd42cebb6d1d4741732ebe/Untitled%2022.png)

- A 數據在 Node 1 上，B 數據在 Node 2 上
- 因為沒有中心化的角色存在，一旦發現如上圖所示的死鎖，雙方都不知道是應該中止還是繼續等待

從這個例子我們可以看出將並發控制升級到分佈式並發控制，有許多問題要解決。

# Conclusion

---

本節我們瞭解了 Distributed DBMS 的皮毛，也看到了實現它的難度。有一個 [Jepsen Project](https://jepsen.io/)，它設置了一系列分佈式事務測試用例，來驗證市面上的系統是否真如其聲明的那樣可靠。即便是 MongoDB、TiDB 這些資料庫的一些版本都被驗證無法提供它們聲明的保證。