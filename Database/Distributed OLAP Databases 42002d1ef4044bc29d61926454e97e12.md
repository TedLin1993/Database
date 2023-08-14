# Distributed OLAP Databases

# Intro

## OLTP and OLAP

---

眾所周知，資料庫有兩種典型使用場景，OLTP 和 OLAP。線上服務與 OLTP 資料庫交互，OLTP 資料庫再被異步地導出到 OLAP 資料庫中作離線分析，如下圖所示：

![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled.png)

OLTP 資料庫就是 OLAP 資料庫的前端，通過 ETL 的過程，OLTP 資料庫中的數據將被清理、重新整理到 OLAP 資料庫上。

OLAP 資料庫為用戶提供複雜的數據查詢、分析能力，幫助公司，分析過去和預測未來。

## Star Schema vs. Snowflake Schema

---

ETL 的過程並不只是簡單地移動，通常還會涉及表結構的重新整理，以提高後續查詢分析的效率。

最常見的兩種 schema 就是 Star Schema(星形拓撲結構) 和 Snowflak Schema：

### Star Schema

![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%201.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%201.png)

- Star Schema 就是一個星形拓撲結構，處在最中間的是 Fact Table，通常記錄著業務流程中的核心事件、指標，如成單記錄
- 處在四周的是 Dimension Tables，記錄一些補充信息
- Fact Table 通過外鍵與 Dimension Tables 關聯，用戶可以通過簡單的 Join 來分析數據。在 Star Schema 中，只能允許有一層的引用關系

### Snowflake Schema

Snowflake Schema 中，則允許有兩層關系，如：

![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%202.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%202.png)

二者的區別、權衡主要在於以下兩個方面：

1. Normalization：Snowflake Schema 的規范化 (Normalization) 級別更高，冗餘信息更少，佔用空間更少，但會遇到數據完整性和一致性問題。
2. Query Complexity：Snowflake Schema 在查詢時需要更多的 join 操作才能獲取到查詢所需的所有數據，速度更慢。

## Problem Setup

---

- 想像下面這個最簡單的分析場景：
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%203.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%203.png)
    
- 一個 join 語句需要訪問所有資料庫分片。要滿足這樣的需求，最簡單的做法就是，將所有相關的數據讀取到某一個分片上，然後統一計算：
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%204.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%204.png)
    

但這在 OLAP 場景下是不可行的。通常 OLAP 就需要訪問全量數據，遇到全量數據無法裝進一個分片中的情況，就無計可施了。

# Agenda

- Execution Models
- Query Planning
- Distributed Join Algorithms
- Cloud Systems

# Execution Models：Push vs. Pull

大體上，查詢的執行模式分為兩種：

- Approach #1: Push Query to Data
    - 將Query(or a portion of Query)發送到擁有該data的節點上
    - 在相應的節點上執行盡可能多的過濾、預處理操作，將盡量少的數據通過網絡傳輸返回
- Approach #2: Pull Data to Query
    - 將數據移動到執行查詢的節點上，然後再執行查詢獲取結果

對於資料庫來說，Push Query to Data 和 Pull Data to Query 並不是非此即彼的選擇，在不同類型的分佈式資料庫、不同的查詢執行階段上，也有可能使用不同的執行模式。

## Example #1: Push Query to Data in Shared-Nothing Architecture

---

![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%205.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%205.png)

如上圖所示：應用程序將查詢請求發到上方的節點 P1

- 節點 P1 儲存 ID 在 1-100 之間的數據
- 節點 P2 儲存 ID 在 101-200 之間的數據

節點 P1 將查詢發給節點 P2 ，由節點 P2 負責將 101-200 之間的數據 join，然後將 join 的結果返回給節點 P1

而節點P1 則再將 1-100 之間的數據 join，最終節點 P1 將所有數據整理好返回給應用程序。

整個過程應用程序不會知道。

## Example #2: Pull Data to Query in Shared-Disk Architecture

---

![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%206.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%206.png)

如上圖所示：在 shared-disk 架構下，節點 P1 可以將計算分散到不同的節點上

- ID 1-100 的在 P1 節點上計算；ID 101-200 的在 P2 節點上計算。
- P1，P2 拿到計算任務後，就將各自所需的數據 (page ABC、XYZ) 從共享的儲存服務中取出放到本地。這個取數據的過程就是 Pull Data to Query。
- 當 P2  節點中的計算任務執行完後，P2  節點將結果返回給 P1 節點，P1 節點再將自己的結果與 P2  節點的結果結合，得到最終的結果返回給應用程序：

![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%207.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%207.png)

Application push Query到 P1 是 Push Query to Data，因此我們需要注意 Push 和 Pull 並不是在一次查詢執行過程中只能取其一，也可能是一種混合過程。

# Query Fault Tolerance

每個節點都會有自己的緩存管理器，從其它計算節點獲取的數據可能會被緩存在本地的緩存池中，方便緩存中間結果，我們甚至可以將這些中間結果持久化成本地磁盤中的臨時文件，這允許我們緩存比內存更大的數據，但這些數據在重啟之後都會消失，那麼對一個需要運行很長時間的 OLAP 查詢來說，如果一個節點掛了怎麼辦？

- 對於 OLTP 資料庫，有大量的寫事務，一旦告訴客戶端事務提交成功，那麼它必須保證規定范圍內的故障不會導致數據丟失
- 對於 OLAP 資料庫，只有讀請求，幾乎沒有資料庫選擇向用戶提供類似的容錯機制，一個查詢在執行過程中如果遇到節點故障，就直接返回錯誤，讓用戶重試。

當然，如果真的面對常常會遇到故障的場景，一些 OLAP DBMS 可以選擇存儲中間結果的快照數據，在節點故障後能恢復當時的部分執行結果，避免重復計算。

大部分情況都不會啟動Query Fault Tolerance，因為要一直snap shot，這非常損耗效能

- Query
    
    ![Untitled](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%208.png)
    
- 其中一個node突然掛掉，但disk有存snap shot
    
    ![Untitled](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%209.png)
    

# Query Planning

我們在單機資料庫上討論過的所有優化，在分佈式場景下同樣適用，如：

- Predicate Pushdown
- Early Projections
- Optimal Join Orderings

當然，分佈式查詢優化還需要考慮數據的位置信息、數據移動的成本，因此在分布式系統上更為複雜

分佈式查詢將查詢的過程分解成多個部分 (Query Plan Fragments)，可以並行執行，從而最大程度地利用分佈式系統的擴展性。

分佈式資料庫的查詢優化主要有兩種粒度：Physical Operators、SQL。

## Approach #1: Physical Operators

---

先生成一個查詢計劃，再將它按數據分解成多個部分 (partition-specific fragments) ，分發給不同的節點。大部分資料庫採用的就是這種做法。

## Approach #2: SQL

---

將原始的 SQL 語句按分片信息重寫成多條 SQL 語句，每個節點自己在本地作查詢優化。Andy 說他只見過 MemSQL 採用了這種方案

舉例如下：

![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2010.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2010.png)

# Distributed Join Algorithms

在剛才的討論中，我們利用了這樣一句 SQL 語句：

```sql
SELECT * FROM R JOIN S ON R.id = S.id

```

但我們忽略了一個細節，即我們假設 R 和 S 表中 id 數據位於同一個節點上。

這樣的假設並不現實。實際上，要獲得 R 和 S join 的結果，我們還需要先將 join 所需的數據移動到同一個節點上。

一旦移動完畢，我們就可以使用之前學習的單機 join 算法完成餘下的計算。

下面討論這條 SQL 在不同場景下的 join 執行過程：

## Scenario #1

---

參與 Join 的兩張表中，其中一張表 (假設為 S 表) 複製到了所有節點上，那麼每個節點按 R 表的分片信息執行 join，最後聚合到 coordinating node 上即可：

![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2011.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2011.png)

## Scenario #2

---

恰好 R 和 S join 的字段就是 partition 的字段，那麼每個節點本地 join，最後聚合到 coordinating node 上即可，與我們之前的假設一致：

![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2012.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2012.png)

## Scenario #3

---

如果 R 和 S 是根據不同 key 來分片，其中一張表 (S) 的 key 不是 join key 且數據量很小，那麼 DBMS 可以將這張小表廣播到所有需要執行計算的節點上，這樣執行時就可以按 R 的分片信息來執行，最後匯總結果：

- R 按照 Id 分片，S 按照 Val 分片
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2013.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2013.png)
    
- 左邊分片將 S 表的部分數據同步到右邊分片
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2014.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2014.png)
    
- 右邊分片將 S 表的部分數據同步到左邊分片
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2015.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2015.png)
    
- 分別在兩個節點上執行 Join 操作
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2016.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2016.png)
    
- 匯總結果並返回
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2017.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2017.png)
    

## Scenario #4

---

兩張表都不是按照 join key 來分片。DBMS 需要將數據表按照 join key 重新洗牌(shuffling)，挪動到對應的位置，再執行 join 操作：

- R 和 S 都不是按照 join key 分片
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2018.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2018.png)
    
- 將 R 表中 id 為 101-200 的數據移動到右邊節點
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2019.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2019.png)
    
- 將 R 表中 id 為 1-100 的數據移動到左邊節點
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2020.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2020.png)
    
- 將 S 表中 id 為 101-200 的數據移動到右邊節點
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2021.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2021.png)
    
- 將 S 表中 id 為 1-100 的數據移動到左邊節點
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2022.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2022.png)
    
- 在兩個節點上執行 Join
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2023.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2023.png)
    
- 合併結果返回
    
    ![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2024.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2024.png)
    

## Semi-Join

---

semi-join 指的是當 join 的結果只需要左邊Table的字段，右邊Table的字段僅僅是用來做篩選的情況。

在分佈式資料庫中，可以對這種特殊情況優化數據移動量，從而減少 join 成本。一些資料庫支持 semi-join 的 SQL 語法，如果不支持則可以使用 EXISTS 語法來模擬：

```sql
SELECT R.id FROM R
 WHERE EXISTS (
   SELECT 1 FROM S
    WHERE R.id = S.id);

```

# Cloud Systems

OLAP 資料庫的雲服務也分為兩類：

- Approach #1: Managed DBMS
    - 將開源單機資料庫挪到雲上，增加一些小的修改，大多數供應商採用這種做法
- Approach #2: Cloud-Native DBMS
    - 為雲原生環境而設計
    - 通常基於 shared-disk 架構
    - 一些例子包括：Snowflake，Google BigQuery，Amazon Redshift 以及 Microsoft SQL Azure

## Serverless Databases

---

基本思想就是，在空閒(idle)期間回收計算資源，租戶只需要為存儲資源買單，節省租戶成本。

實現的基本思路就是空閒指標達到一定閾值時，將 Buffer Pool Page Table 持久化：

![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2025.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2025.png)

當活躍請求到來時，再將其載入到內存中：

![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2026.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2026.png)

## Disaggregated Components

---

一些雲服務商也提供 OLAP 資料庫所需的模塊服務，如：

- System Catalogs
    - HCatalog
    - Google Data Catalog
    - Amazon Glue Data Catalog
- Node Management
    - Kubernetes
    - Apache YARN
    - Cloud Vendor Tools
- Query Optimizers
    - Greenplum Orca
    - Apache Calcite

## Universal Formats

---

大多數 DBMS 都使用自主設計研發的二進制文件格式存儲數據，因此在異構 DBMS 之間共享數據的唯一方法就是將這些數據轉化成一些常見的文本(text-based)格式，如 csv，json，xml 等。

為了解決這個問題，也有一些開源的二進制文件格式出現：

![Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2027.png](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2027.png)

# Conclusion

沿用本課課件中的結束語：

> More money, more data, more problems…
> 

![Untitled](Distributed%20OLAP%20Databases%2042002d1ef4044bc29d61926454e97e12/Untitled%2028.png)