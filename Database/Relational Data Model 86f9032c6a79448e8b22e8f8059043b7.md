# Relational Data Model

# 為什麼需要 Database？

假設我們需要存儲：

- 藝術家（Artists）信息
- 藝術家發行的專輯（Albums）信息

我們可以用兩個 CSV 文件來分別存儲藝術家和專輯信息，然後用程序來解析和序列化相關數據：

藝術家信息表：

```
Artist.csv
"Wu Tang Clan",1992,"USA"
"Notorious BIG",1992,"USA"
"Ice Cube",1989,"USA"
```

專輯信息表：

```
Album.csv
"Enter the Wu Tang","Wu Tang Clan",1993
"St.Ides Mix Tape","Wu Tang Clan",1994
"AmeriKKKa's Most Wanted","Ice Cube",1990
```

假設我們想知道 “Ice Cube 在哪年首發”，就會寫這樣的查詢腳本：

```
for line in file:
    record = parse(line)
    if record[0] == "Ice Cube":
        print(int(record[1]))
```

但這種簡單的方案有很多缺陷：

- 數據的質量方面
    - 很難保證同一個藝術家發行的每條專輯信息中，藝術家字段一致
    - 很難阻止用戶在年份字段寫入不合法的字符串
    - 很難優雅地處理一張專輯由多個藝術家共同發行的情況
- 實現方面
    - 查詢一條特定的記錄，效率低
    - 當有多個應用使用該 CSV 數據庫
        - 查詢腳本需要重復寫
        - 多個線程一起寫時，如何保證數據一致性
- 數據持久
    - 正在寫記錄時遇到宕機
    - 如何復制數據庫到多台機器上來保證高可用性

以上缺陷迫使我們需要升級 CSV 數據庫，於是就有了專業的數據庫系統（DBMS）。

# DBMS 的提出

## 分離邏輯層和物理層

所有系統都會產生數據，因此數據庫幾乎是所有系統都不可或缺的模塊。在早期，各個項目各自造輪子，因為每個輪子都是為應用量身打造，這些系統的邏輯層（logical）和物理層（physical）普遍耦合度很高。

Ted Codd 發現這個問題後，提出 DBMS 的抽象（Abstraction）：

- 用簡單的、統一的數據結構存儲數據
- 通過高級語言操作數據
- 邏輯層和物理層分離，系統開發者只關心邏輯層，而 DBMS 開發者才關心物理層。

## Data Models 數據模型

在邏輯層中，我們通常需要對所需存儲的數據進行建模。如今，市面上有的數據模型包括：

- Relational → 大部分 DBMS 屬於關系型，也是本課討論的重點
    - 非Relational的都是NoSQL
- Key/Value → Redis
- Graph
- Document → MongoDB
- Column-family
- Array/Matrix → Maching Learning

## Relational Model

**Structure**: The definition of relations and their contents.

**Integrity**: Ensure the database’s contents satisfy constraints.

**Manipulation**: How to access and modify a database’s contents.

### Relation & Tuple

- 每個 Relation 都是一個無序集合（unordered set），集合中的元素稱為 tuple
- 每個 tuple 由一組屬性構成，這些屬性在邏輯上通常有內在聯系。

![Untitled](Relational%20Data%20Model%2086f9032c6a79448e8b22e8f8059043b7/Untitled.png)

### Primary Keys

- primary key 在一個 Relation 中確保各 tuple都是unique
- 如果你不指定，有些 DBMSs 會自動幫你生成 primary key。

### Foreign Keys

- foreign key: mapping到另一個 relation 中的一個 tuple
- 利用這些基本概念，我們就可以利用第三張表，ArtistAlbum，來解決專輯與藝術家的 1 對多的關系問題：
    
    ![Untitled](Relational%20Data%20Model%2086f9032c6a79448e8b22e8f8059043b7/Untitled%201.png)
    

### Data Manipulation Languages (DML)

在 Relational Model 中從數據庫中查詢數據通常有兩種方式：Procedural 與 NonProcedural：

- Procedural：查詢命令需要指定 DBMS 執行時的具體查詢策略，如 Relational Algebra
- Non-Procedural：查詢命令只需要指定想要查詢哪些數據，無需關心幕後的故事，如 SQL
    
    → 以high level的語言去處理資料庫的命令(邏輯層)，不須知道具體實作方式(物理層)
    

使用哪種方式是具體的實現問題，與 Relational Model 本身無關。

### Relational Algebra

relational algebra 是基於 set algebra 提出的，從 relation 中查詢和修改 tuples 的一些基本操作，它們包括：

- Select  (σ)
- Projection ( π )
- Union ( ∪ )
- Intersection ( ∩ )
- Difference ( − )
- Product ( × )
- Join ( ⨝ )
- Rename ( ρ )
- Assignment ( R←S )
- Duplicate Elimination ( δ )
- Aggregation ( γ )
- Sorting ( τ )
- Division ( R÷S )

將這些操作串聯起來，我們就能構建更複雜的操作

**注意**：使用 Relation Algebra 時，我們實際上指定了執行策略，如：

![Untitled](Relational%20Data%20Model%2086f9032c6a79448e8b22e8f8059043b7/Untitled%202.png)

它們所做的事情都是 ”返回 R 和 S Join 後的結果中，b_id 等於 102 的 tuples“。

雖然 Relational Algebra 只是 Relational Model 的具體實現方式，但在之後的課程將會看到它對查詢優化、執行的幫助。