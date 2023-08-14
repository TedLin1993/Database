# Query Execution

# 簡介

![Untitled](Query%20Execution%2005ed8701be714c6e9a9c38ab990ba9ad/Untitled.png)

如上圖所示，通常一個 SQL 會被組織成樹狀的查詢計劃，數據從 leaf nodes 流到 root，查詢結果在 root 中得出。而本節將討論在這樣一個計劃中，如何為這個數據流動過程建模，大綱如下：

- Processing Models
- Access Methods
- Expression Evaluation

# Processing Model

DBMS 的 processing model 定義了系統如何執行一個 query plan，目前主要有三種模型:

- Iterator Model
    - 最常見的model，幾乎所有row-base DBMS都是使用此model
- Materialization Model
    - 常用在in memory system
- Vectorized/Batch Model
    - 基於Iterator Model，但batch output

These models can also be implemented to invoke the operators either from top-to-bottom (most common) or from bottom-to-top.

## Iterator Model

query plan 中的每步 operator 都實現一個 **Next** 函數

- 每次調用時，operator 返回一個 tuple 或者 null，後者表示數據已經遍歷完畢。
- operator 實現一個loop去調用 child operators 的 next 函數，從它們那邊獲取下一條數據供自己操作，這樣整個 query plan 就被從上至下地串聯起來

也稱為 Volcano Model或 Pipeline Model

![Untitled](Query%20Execution%2005ed8701be714c6e9a9c38ab990ba9ad/Untitled%201.png)

![Untitled](Query%20Execution%2005ed8701be714c6e9a9c38ab990ba9ad/Untitled%202.png)

Iterator 幾乎被用在每個 DBMS 中，包括 sqlite、MySQL、PostgreSQL 等等，其它需要注意的是：

- 有些 operators 會等待 children 返回所有 tuples 後才執行，如 Joins, Subqueries 和 Order By
- Output Control 在 Iterator Model 中比較容易，如 Limit，只按需調用 next 即可。

## Materialization Model

每個 operator 處理完所有輸入後，將所有結果一次性輸出

- The operator processes all the tuples from its children at once
- DBMS 會將一些參數傳遞到 operator 中防止處理過多的數據

![Untitled](Query%20Execution%2005ed8701be714c6e9a9c38ab990ba9ad/Untitled%203.png)

### materialization model：

- 更適合 OLTP 場景，因為一次只需要處理少量的 tuples
    - Lower execution / coordination overhead
    - Fewer function calls
- 不太適合會產生大量中間結果的 OLAP 查詢
- 較適合in-memory DB使用，目前沒有on disk DB使用這個model
- 使用DB: monetdb, VOLTDB

## Vectorization Model

Vectorization Model 是 Iterator 與 Materialization Model 折衷的一種模型：

- 每個 operator 實現一個 next 函數，但每次 next 調用返回一批 tuples，而不是單個 tuple
- operator 內部的循環每次也是一批一批 tuples 地處理
- batch 的大小可以根據需要改變（hardware、query properties）

![Untitled](Query%20Execution%2005ed8701be714c6e9a9c38ab990ba9ad/Untitled%204.png)

vectorization model 是 OLAP 查詢的理想模型：

- 極大地減少每個 operator 的調用次數
- 允許 operators 使用 vectorized instructions (SIMD) 來批量處理 tuples

目前在使用這種模型的 DBMS 有 VectorWise, Peloton, Preston, SQL Server, ORACLE, DB2 等。

# Access Methods

access method 指的是 DBMS 從table中獲取資料的方式，它並沒有在 relational algebra 中定義。主要有三種方法：

- Sequential Scan
- Index Scan
- Multi-Index/"Bitmap" Scan

## Sequential Scan

顧名思義，sequential scan 就是按順序從 table 所在的 pages 中取出 tuple，這種方式是 DBMS 能做的最壞的打算。

```python
for page in table.pages:
    for t in page.tuples:
        if evalPred(t):
            # do something
```

### Sequential Scan: Optimizations

DBMS 內部需要維護一個 cursor 來追蹤之前訪問到的位置（page/slot）。Sequential Scan 是最差的方案，因此也針對地有許多優化方案：

- Prefetching
- Buffer Pool Bypass
- Parallelization
- (本節) Zone Maps
- (本節) Late Materialization
- (本節) Heap Clustering

### **Zone Maps**

- 預先為每個 page 計算好 attribute values 的一些統計值，DBMS 在訪問 page 之前先檢查 zone map，確認一下是否要繼續訪問，如下圖所示：
    
    ![Untitled](Query%20Execution%2005ed8701be714c6e9a9c38ab990ba9ad/Untitled%205.png)
    
    - 當 DBMS 發現 page 的 Zone Map 中記錄 val 的最大值為 400 時，就沒有必要訪問這個 page。
- Zone maps可以存在page裡面，也可以另外獨立出一個zone map pages存在memory裡面
- 使用DB: SQL Server, ORACLE, DB2

### **Late Materialization**

在列存儲 DBMS 中，每個 operator 只選取查詢所需的列數據，若該列數據在查詢樹上方並不需要，則僅需向上傳遞 offsets 即可：

![Untitled](Query%20Execution%2005ed8701be714c6e9a9c38ab990ba9ad/Untitled%206.png)

### **Heap Clustering**

使用 clustering index 時，tuples 在 page 中按照相應的順序排列，如果查詢訪問的是被索引的 attributes，DBMS 就可以直接跳躍訪問目標 tuples。

![Untitled](Query%20Execution%2005ed8701be714c6e9a9c38ab990ba9ad/Untitled%207.png)

## Index Scan

DBMS 選擇一個 index 來找到查詢需要的 tuples。使用哪個 index 取決於以下幾個因素：

- index 包含哪些 attributes
- 查詢引用了哪些 attributes
- attribute 的定義域(value domains)
- predicate composition
- index 的 key 是 unique 還是 non-unique

這些問題都將在後面的課程中詳細描述，本節只是對 Index Scan 作概括性介紹。

盡管選擇哪個 Index 取決於很多因素，但其核心思想就是，越早過濾掉越多的 tuples 越好，如下面這個 query 所示：

```sql
SELECT * FROM students
 WHERE age < 30
   AND dept = 'CS'
   AND country = 'US';
```

- index #1: age
- index #2: dept

students 在不同 attributes 上的分佈可能如下所示：

![Untitled](Query%20Execution%2005ed8701be714c6e9a9c38ab990ba9ad/Untitled%208.png)

- Scenario #1：使用 dept 的 index 能過濾掉更多的 tuples
- Scenario #2：使用 country 的 index 能過濾掉更多的 tuples

## **Multi-index Scan**

如果有多個 indexes 同時可以供 DBMS 使用，就可以做這樣的事情：

- 計算出符合每個 index 的 record ids sets
- 基於 predicates (union vs. intersection) 來確定是對集合取交集還是聯集
- 取出相應的 tuples 並完成剩下的處理

Postgres 稱 multi-index scan 為 **Bitmap Scan**。

仍然以上一個 SQL 為例，使用 multi-index scan 的過程如下所示：

![Untitled](Query%20Execution%2005ed8701be714c6e9a9c38ab990ba9ad/Untitled%209.png)

其中取集合交集可以使用 bitmaps, hash tables 或者 bloom filters。

### **Index Scan Page Sorting**

當使用的不是 clustering index 時，實際上按 index 順序檢索的過程是非常低效的，DBMS 很有可能需要不斷地在不同的 pages 之間來回切換。

為了解決這個問題，DBMS 通常會先找到所有需要的 tuples，根據它們的 page id 來排序，完畢後再讀取 tuples 數據，使得整個過程每個需要訪問的 page 只會被訪問一次。如下圖所示：

![Untitled](Query%20Execution%2005ed8701be714c6e9a9c38ab990ba9ad/Untitled%2010.png)

# Expression Evaluation

DBMS 使用 expression tree 來表示一個 **WHERE** 語句，如下圖所示：

![Untitled](Query%20Execution%2005ed8701be714c6e9a9c38ab990ba9ad/Untitled%2011.png)

The nodes in the tree represent different expression types:

- Comparisons (=, <, >, !=)
- Conjunction (AND), Disjunction (OR)
- Arithmetic Operators (+, -, *, /, %)
- Constant and Parameter Values
- Tuple Attribute References

然後根據 expression tree 完成數據過濾的判斷，但這個過程比較低效，很多 DBMS 採用 JIT Compilation 的方式，直接將比較的過程編譯成機器碼來執行，提高 expression evaluation 的效率。

# Conclusion

- The same query plan be executed in multiple ways.
- (Most) DBMSs will want to use an index scan as much as possible.
- Expression trees are flexible but slow.