# Parallel Execution

# 背景

隨著摩爾定律逐漸失效，處理器走向多核，系統可以通過並行執行增加吞吐量，減少延遲，使得系統響應更快。

## Parallel & Distributed

- Parallel DBMS：運行在多核 CPU 上
    - 每個節點(Nodes)物理上非常接近
    - Nodes通過高速interconnect(LAN)相連接
    - 通信成本小且可靠
- Distributed DBMS：分佈式資料庫
    - Nodes之間距離可能很遠，通過公共網路相連接
    - 通信成本高且通信可能出現的問題不可忽略

Distributed DBMS不在這堂課的討論範圍內

# Process Model

DBMS 的 process model 定義了多用戶的DBMS如何處理concurrent requests。在下文中，用 workers 指代執行查詢任務的單位，它可能是 Process(es)，也可能是 Thread(s)。

## Approach #1: Process per DBMS Worker

![Untitled](Parallel%20Execution%20cc4130610c684ae98d208545694ba861/Untitled.png)

用戶請求經過 Dispatcher 後，由 Dispatcher 分配相應的 Worker 完成查詢並返回結果，每個 worker 都是單獨的 OS Process

- 依賴 OS scheduler 來調度
- 使用 shared-memory 來儲存全域資料結構
- 單個 worker 崩潰不會引起整個系統崩潰

這種 Process Model 出現在 threads 跨平台支持很不穩定的時代，主要是為了解決系統的可移植性問題。使用這種 Process Model 的資料庫有歷史版本的 DB2、ORACLE 和 PostgreSQL 等。

## Approach #2: Process Pool

![Untitled](Parallel%20Execution%20cc4130610c684ae98d208545694ba861/Untitled%201.png)

用戶請求經過 Dispatcher 後，由 Dispatcher 分配相應的 Worker 完成查詢，將結果返回 Dispatcher，後者再返回給用戶。每個 Worker 可以使用 Worker Pool 中任意空閒的 Process(es)：

- 依賴 OS scheduler 來調度
- 使用 shared-memory 來儲存全局資料結構
- 不利於 CPU cache

使用這種 Process Model 的資料庫有 DB2、PostgreSQL(2015)。

## Approach #3: Thread per DBMS Worker

![Untitled](Parallel%20Execution%20cc4130610c684ae98d208545694ba861/Untitled%202.png)

整個 DBMS 由一個 Process 和多個 Worker Threads 構成：

- DBMS 自己控制調度策略：
    - 每個查詢拆成多少個任務
    - 使用多少個 CPU cores
    - 將任務分配到哪個 core 上
    - 每個 task 的輸出儲存在哪裡
- 自己控制調度策略的理由與自己構建 Buffer Pools 的理由是一樣的：DBMS 比 OS 知道的更多
- dispatcher 不一定存在
- thread 崩潰可能導致整個系統崩潰
- 使用多線程架構的優勢
    - context switch 成本更低
    - 天然地可以在 threads 之間共享全域訊息，無需使用 shared memory

這是目前最常見的作法，使用這種 Process Model 的資料庫有 DB2、MSSQL、MySQL、Oracle (2014)  及其它近 10 年出現的 DBMS 等。

# Execution Parallelism

## Inter-query vs. Intra-query Parallelism

- Inter-Query：不同的查詢並行(concurrently)執行
    - 增加throughput，減少延遲(latency)
    - Concurrency is tricky when queries are updating the database
- Intra-Query：同樣的查詢切分成不同的task，並同時在不同的 operators 並行執行
    - 減少長時查詢的延遲，主要用於 Streaming

## Inter-query Parallelism

- 通過並行執行多個查詢來提高 DBMS 性能。
- 如果這些查詢都是read-only，那麼處理不同查詢之間的關系無需額外的工作
- 如果查詢存在更新操作，那麼處理不同查詢之間的關系將變得很難。相關內容將在後續章節中介紹。

## Intra-query Parallelism

通過並行執行單個查詢的單個或多個 operators 來提高 DBMS 性能：

- Approach #1：Intra-Operator
- Approach #2：Inter-Operator

這兩個方法可以被同時使用，每個 relational operator 都有並行的算法實現。

### **Intra-operator Parallelism (Horizontal)**

將 data 拆解成多個**fragments**(子集)，然後對這些子集並行地執行相應的 operator，DBMS 通過將 **exchange** operator 引入查詢計劃，來合併子集處理的結果，過程類似 MapReduce，舉例如下圖所示：

![Untitled](Parallel%20Execution%20cc4130610c684ae98d208545694ba861/Untitled%203.png)

exchange operator分成三種type:

- **Gather**: Combine the results from multiple workers into a single output stream. This is the most
common type used in parallel DBMSs
- **Repartition**: Reorganize multiple input streams across multiple output streams. This allows the
DBMS take inputs that are partitioned one way and then redistribute them in another way.
- **Distribute**: Split a single input stream into multiple output streams

![Untitled](Parallel%20Execution%20cc4130610c684ae98d208545694ba861/Untitled%204.png)

### **Inter-operator Parallelism (Vertical)**

將 operators 串成 pipeline，data從上游流向下游，一般無需等待前一步操作執行完畢，也稱為 pipelined parallelism，舉例如下：

![Untitled](Parallel%20Execution%20cc4130610c684ae98d208545694ba861/Untitled%205.png)

- Join和Projection平行執行

這種方式在傳統 DBMSs 中並不常用，因為許多 operators，如 join， 必須掃描所有 tuples 之後才能得到結果。它更多地被用在流處理系統，如 Spark、Nifi、Kafka,、Storm、Flink、Heron。

## 觀察

值得注意的是，使用額外的 processes/threads 來並行地執行查詢可以通過提高 CPU 利用率來提高 DBMS 效率；但如果 DBMS 效率瓶頸出現在 disk 資料存取上，這種優化帶來的效果就非常有限，甚至有可能因為 disk I/O 的提高導致整體性能下降，如 cache miss rate 提高等等。

# I/O Parallelism

I/O Parallelism 通過將 DBMS 安裝在多個儲存設備上來實現：

- Multiple Disks per Database
- One Database per Disk
- One Relation per Disk
- Split Relation across Multiple Disks

## Multi-disk Parallelism

通過 OS 或硬件配置將 DBMS 的Data文件存儲到多個儲存設備上，整個過程對 DBMS 透明，如使用 RAID。

![Untitled](Parallel%20Execution%20cc4130610c684ae98d208545694ba861/Untitled%206.png)

![Untitled](Parallel%20Execution%20cc4130610c684ae98d208545694ba861/Untitled%207.png)

一些 DBMS 甚至允許用戶為單個 database 指定 disk location。

## Partitioning

將一個 logical table 拆分成多個 physical segments 分開儲存。理想情況下，application不需要知道底層的partitioning，DB是否拆成多個segments對於query都沒有影響，但在分散式DBMS不一定能實現。

### **Vertical Partitioning**

原理上類似列(column)儲存資料庫，將 table 中的部分 attributes 存儲到不同的地方，如：

```sql
CREATE TABLE foo (
  attr1 INT,
  attr2 INT,
  attr3 INT,
  attr4 TEXT
);
```

![Untitled](Parallel%20Execution%20cc4130610c684ae98d208545694ba861/Untitled%208.png)

**Horizontal Partitioning**

基於某個可定製的 partitioning key 將 table 的不同 segments 分開存儲，包括：

- Hash Partitioning
- Range Partitioning
- Predicate Partitioning

![Untitled](Parallel%20Execution%20cc4130610c684ae98d208545694ba861/Untitled%209.png)

# 小結

並發執行很重要，幾乎所有 DBMS 都需要它，但要將這件事做對很難，體現在

- Coordination Overhead
- Scheduling
- Concurrency Issues
- Resource Contention