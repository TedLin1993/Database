# Query Optimization

- SQL 語句讓我們能夠描述想要獲取的數據，而 DBMS 負責來根據用戶的需求來制定高效的查詢計劃。
- 不同的查詢計劃的效率可能出現多個數量級的差別，如 Join Algorithms 一節中的 Simple Nested Loop Join 與 Hash Join 的時間對比 (1.3 hours vs. 0.45 seconds)。
- Query Optimizer 第一次出現在 IBM **System R**，那時人們認為 DBMS 指定的查詢計劃永遠無法比人類自己寫得更好，當時的人類需要自己寫組合語言。
- System R optimizer 中的一些理念和設計決策至今仍在使用。
- 這是Database最難的部分

# Query Optimization Techniques

- **Heuristics/Rules**
    - Rewrite the query to remove stupid/inefficient things
    - These techniques may need to examine catalog, but they do **not** need to examine data
        - catalog: schema的上層，1個catalog可以包含多個schema
- **Cost-based Search**
    - Use a cost model to evaluate multiple equivalent plans and pick the one with the lowest cost
    
    ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled.png)
    
    - 這裡的 Rewriter 負責 Heuristics/Rules，Optimizer 則負責 Cost-based Search。
    - 有些DB沒有Cost model，沒有query optimizer，例如MongoDB

Query Optimization是NP-Hard，非常困難，目前最新的做法是以ML的方式去做，但目前在商業上還沒有明確的成效。

付費的SQL如Oracle找了數百人一直去精進他們的Query Optimization，這是企業級SQL與Open Source SQL的最大差異。

# **Heuristics/Rules**

## Query Rewriting

如果兩個關系代數表達式 (Relational Algebra Expressions) 如果能產生相同的 tuple 集合，我們就稱二者等價。

DBMS 可以通過一些 Heuristics/Rules 來將關系幾何表達式轉化成成本更低的等價表達式，從而達到查詢優化的目的。這些規則通常適用於所有查詢，如：

- Predicate Pushdown
    - Predicate(**述詞**): 這是評估為 TRUE、FALSE 或 UNKNOWN 的運算式。 述詞會用在 WHERE 子句和 HAVING 子句的搜尋條件中、FROM 子句的聯結條件中，以及其他需要布林值的建構中。
- Projections Pushdown

## **Predicate Pushdown**

Predicate 通常有很高的選擇性，可以過濾掉許多無用的數據。將 Predicate 推到查詢計劃的底部，可以在查詢開始時就更多地過濾數據，舉例如下：

![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%201.png)

![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%202.png)

核心思想如下：

- 越早過濾越多數據越好
- 重排 predicates，使得選擇性大的排前面
- 將複雜的 predicate 拆分，然後往下壓，如 `X=Y AND Y=3` 可以修改成 `X=3 AND Y=3`

## **Projections Pushdown**

本方案對column-store資料庫不適用。在row-store資料庫中，越早過濾掉不用的字段越好，因此將 Projections 操作往查詢計劃底部推也能夠縮小中間結果佔用的空間大小，舉例如下：

![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%203.png)

## More Examples

以下的搜尋看起來現實上不會遇到，但事實上很多時候Predicate是由Code所構成的，整段SQL的每一段可能是由不同的code組成，我們在事前不會知道整段SQL會長怎樣

- Impossible / Unnecessary Predicates
    - 一定為空的Predicate: 直接回傳空值
    
    ```sql
    SELECT * FROM A WHERE 1 = 0;
    ```
    
    - 沒縮減範圍的Predicage: 直接回傳全部的值
    
    ```sql
    SELECT * FROM A WHERE 1 = 1;
    ```
    
- Join Elimination: 移除不必要的Join
    
    ```sql
    SELECT A1.*
    FROM A AS A1 JOIN A AS A2
    ON A1.id = A2.id;
    ```
    
    可簡化成
    
    ```sql
    SELECT * FROM A;
    ```
    
- Ignoring Projections
    
    ```sql
    SELECT * FROM A AS A1
    WHERE EXISTS(SELECT val FROM A AS A2
    				WHERE A1.id = A2.id);
    ```
    
    可簡化成
    
    ```sql
    SELECT * FROM A;
    ```
    
- Merging Predicates: 將有Overlap的Predicate合在一起
    
    ```sql
    SELECT * FROM A
    WHERE val BETWEEN 1 AND 100
    OR val BETWEEN 50 AND 150;
    ```
    
    可簡化成
    
    ```sql
    SELECT * FROM A
    WHERE val BETWEEN 1 AND 150;
    ```
    

# Cost-based Search

- 除了 Predicates 和 Projections 以外，許多操作沒有通用的規則，例如 Join
- Join 操作既符合交換律又符合結合律，等價關系代數表達式數量龐大，這時候就需要一些成本估算技術，將過濾性大的表作為 Outer Table，小的作為 Inner Table，從而達到查詢優化的目的。

## **Cost Estimation (成本估計)**

一個查詢需要花費多長時間，取決於許多因素

- CPU: Small cost; tough to estimate
- Disk: # of block transfers
- Memory: Amount of DRAM used
- Network: # of messages

但本質上取決於：**整個查詢過程需要讀入和寫出多少 tuples**

因此 DBMS 需要保存每個 table 的一些統計信息(statistics)，如 attributes、indexes 等信息，有助於估計查詢成本。值得一提的是，不同的 DBMS 的蒐集、更新統計信息的策略不同。

## **Statistics**

各DB更新table的internal statistic資料的時間各有不同，以下整理各DB的table statistics人為調用方法：

- Postgres/SQLite: ANALYZE
- Oracle/MySQL: ANALYZE TABLE
- SQL Server: UPDATE STATISTICS
- DB2: RUNSTATS

OLTP DB可能會傾向白天不更新statistic，直到晚上才一次更新，因為更新statistic要scan整個table

DBMS 對任意的 table R，都保存著以下信息：

- $N_R$：R 中 tuples 的數量
- $V(A, R)$：R 中 A 屬性的distinct values 個數

利用上面兩條數據，可以得到 **selection cardinality**，即 R 中 A 屬性下每個值的平均記錄個數：

- $SC(A, R) = N_R / \ V(A, R)$

需要注意的是，這種估計假設 R 中所有數據在 A 屬性下均勻分佈 (**data uniformity**)。

- selection cardinality(基數估計)
    - 在查詢計畫的每一個層級進行處理的資料列(row)總數，此稱為plan的基數(cardinality)。
- Selection vs Projection
    - SELECTION-選擇，從table中選擇row
    - PROJECTION-映射，從table中選擇column

### Selection Cardinality

- **Assumption #1: Uniform Data**
    - The distribution of values (except for the heavy hitters) is the same.
- **Assumption #2: Independent Predicates**
    - The predicates on attributes are independent
- **Assumption #3: Inclusion Principle (包容原則)**
    - inner table的join key都會在outer table中存在

## Complex Predicates

利用以上信息和假設，DBMS 可以估計不同 predicates(**P**) 的 selectivity(**sel**)：

- Equality
- Range
- Negation
- Conjunction
- Disjunction

### *Equality Predicate*

```sql
SELECT * FROM people WHERE age = 2;
```

設 people 表中有 5 條個人信息，即$N_R = 5$，所有信息中的 age 有 5 個unique值，即$V(age, people) = 5$

⇒$sel(A=constant) = SC(P) / V(A, R) = \frac{1}{5}$

![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%204.png)

### *Range Predicate*

```sql
SELECT * FROM people WHERE age >= 2;
```

可以利用$A_{max}, A_{min}$來估計：$sel(A>= a) = (A_{max}-a)/(A_{max}-A_{min})$

- Example: sel(age≥2) = (4-2) / (4-0) = 1/2，但實際應該是3/5才對，所以實際DB在估計Range Predicate時都會低估實際Data的數量

![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%205.png)

### *Negation Query*

```sql
SELECT * FROM people WHERE age != 2;
```

利用 equality query 可以反向推導出 negation query 的情況：

- $sel(not \ P) = 1 - sel(P) = 1 - SC(age=2) = \frac{4}{5}$

![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%206.png)

實際上這裡所謂的 Selectivity 就是基於 uniformly distribution 假設下的 Probability。

### *Conjunction Query*

```sql
SELECT * FROM people
WHERE age = 2
**AND** name LIKE 'A%';
```

若假設兩個 predicates 之間相互獨立，則可以推導出：

- $sel(P1 \wedge P2) = sel(P1)\bullet sel(P2)$

其文氏圖如下所示：

![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%207.png)

### *Disjunction Query*

```sql
SELECT * FROM people
 WHERE age = 2
    **OR** name LIKE 'A%';
```

若假設兩個 predicates 之間相互獨立，則可以推導出：

$sel(P1 \vee P2) = sel(P1) + sel(P2) - sel(P1 \wedge P2) = sel(P1) + sel(P2) - sel(P1)\bullet sel(P2)$

其文氏圖如下所示：

![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%208.png)

## Statistics Storage

### Histograms (直方圖)

- 我們假設所有的值都是uniform distribution 但現實的DB value並不是如此分布，為了更精確，改用直方圖儲存各個值的次數
    
    ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%209.png)
    
- 但每個值都儲存次數的話太佔空間了，因此我們改以多個值形成一個bucket，紀錄各個bucket的總次數
    
    ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2010.png)
    
- However, this can lead to inaccuracies as frequent values will sway the count of infrequent values. To counteract this, we can size the buckets such that their spread is the same. They each hold a similar amount of values.
    
    ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2011.png)
    

### **Samling**

先進的DBMSs 除了Histograms，也可以使用Sampling(抽樣)來降低Cost Estimation(成本估計)本身的成本，將原有table取一部分出來作為sampling table，再以這個sampling table來估計原table (estimate the selectivity of the predicate by applying the predicate to the small sample)

比如面對如下查詢：

```sql
SELECT AVG(age)
  FROM people
 WHERE age > 50;
```

我們可以等間隔從表中對數據采樣：

![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2012.png)

然後再估計：

![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2013.png)

使用的DB: SQL Server(同時使用Histogram和Sampling)

# Search Algorithm

The basic cost-based search algorithm for a query optimizer is the following:

1. Bring query in internal form into canonical form.
2. Generate alternative plans.
3. Generate costs for each plan.
4. Select plan with smallest cost

## Single-Relation Query Planning

Pick the best access method

- Sequential Scan
- Binary Search (clustered indexes)
- Index Scan

Predicate evaluation ordering.

也可以使用簡單的heuristics來評估該選擇哪種query plan

### OLTP Query Planning

Query planning for OLTP queries is easy because they are **sargable** (Search Argument Able).

- It is usually just picking the best index.
- Joins are almost always on foreign key relationships with a small cardinality.
- Can be implemented with simple heuristics.

![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2014.png)

## Multiple Relation Query Planning

- As number of joins increases, number of alternative plans grows rapidly
    - We need to restrict search space.
- Fundamental decision in **System R**: only left-deep join trees are considered.
    - Left-deep joins allow you to pipeline data, and only need to maintain a single join table in memory.
    - Modern DBMSs do not always make this assumption anymore.
    
    ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2015.png)
    
- Enumerate
    - Enumerate the orderings
        - Example: Left-deep tree #1, Left-deep tree #2…
    - Enumerate the plans for each operator
        - Example: Hash, Sort-Merge, Nested Loop…
    - Enumerate the access paths for each table
        - Example: Index #1, Index #2, Seq Scan…
    - 有太多種Join方式的排列組合導致search space爆炸了，於是IBM在1970s想出了Dynamic Programing

## Dynamic Programing

- 將問題分割成多個小問題，再將一個個小問題分別解決
- Join 3個table的例子:
    
    ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2016.png)
    
    - 首先列舉R和S的join plan以及S和T的join plan，再取cost較小的plan
        
        ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2017.png)
        
    - 接著再比較三個table都join起來的cost
        
        ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2018.png)
        
    - 最後找到Cost最低的路徑
        
        ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2019.png)
        
- 使用的DB: 主流的DB都會使用DP，例如postgre, MySQL, Oracle

### Candidate Plan Example

- 更詳細的解釋Dynamic Programing如何適用在上面的例子上
- Step #1: Enumerate relation orderings
    
    ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2020.png)
    
    - 不考慮cross-product，不會有join使用這種方法
    - 這裡以左上角的relation ordering為例，但另外三者也都需要進行後續的step
- Step #2: Enumerate join algorithm choices
    
    ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2021.png)
    
    - HJ: Hash Join, NLJ: Nested Loop Join
    - 以右下角的join方式為例，但另外三者也需進行下一step
- Step #3: Enumerate access method choices
    
    ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2022.png)
    
- 全部的可能性都計算Cost之後取最小的

### Postgres Optimizer

- Two optimizer implementations:
    - Traditional Dynamic Programming Approach
    - Genetic Query Optimizer (GEQO): 基因演算法
- 當join的table數<12時: 使用Traditional Dynamic Programming Approach
- 當join的table數≥12時: 使用GEQO
- 基因演算法: 隨機取數個組合來比較，保留最佳的組合並刪掉最差的組合，再隨機組合一次，重複多次到區域最佳解為止
    
    ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2023.png)
    

# Nested Sub-Queries

- The DBMS treats nested sub-queries in the where clause as functions that take parameters and return a single value or set of values.
- Two Approaches:
    - Rewrite to de-correlate and/or flatten them
    - Decompose nested query and store result to temporary table

## Rewrite

![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2024.png)

## Decompose

- 例子:  For each sailor with the highest rating (over all sailors) and at least two reservations for red boats, find the sailor id and the earliest date on which the sailor has a reservation for a red boat
    
    ![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2025.png)
    
    - Nested Block(紅框的部分)會一直重複計算，但其實只需要計算一次
- 將這個Query拆成兩個部分，讓Nested Block只Query一次就好

![Untitled](Query%20Optimization%20f8878600784a42b381b02ecaaf35487c/Untitled%2026.png)