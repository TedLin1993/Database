# Tree Indexes

上節提到，DBMS 使用一些特定的數據結構來存儲信息：

- Internal Meta-data
- Core Data Storage
- Temporary Data Structures
- Table Indexes

本節將介紹存儲 table index 最常用的樹形數據結構：B+ Tree，Skip Lists，Radix Tree

# Table Index

table index 為提供 DBMS 數據查詢的快速索引，它本身存儲著table column排序後的數據，並包含指向相應 tuple 的pointer。DBMS 需要保證表信息(content)與索引信息(index)在邏輯上(logically)保持同步。

- Physically可以不同步，例如我們從index刪除某個key，在data structure其實還沒刪掉，但是重新query時卻已經查不到了

用戶可以在 DBMS 中為任意表建立多個索引，DBMS 負責選擇最優的索引提升查詢效率。但索引自身需要佔用存儲空間，因此在索引數量與索引存儲(storage overhead)、維護成本(maintenance overhead)之間存在權衡。

# B-Tree

## B-Tree Family

B-Tree 中的 B 指的是 Balanced，實際上 B-Tree Family 中的 Tree 都是 Balanced Tree。B-Tree 就是其中之一，以之為基礎又衍生出了B+ Tree、B link Tree、B∗ Tree等一系列數據結構。

B-Tree 與 B+ Tree 最主要的區別就是 B+ Tree 的所有 key 都存儲在 leaf nodes，而 B-Tree 的 key 會存在所有節點中，Inner nodes只拿來搜尋用。

- B-Tree的空間利用更有效率，因為B+ Tree刪除一個key之後，仍然可能保留在inner node
- 但實務上沒有DB是實作B-Tree，因為每次insert/delete都可能會導致Tree的太多次的改動

## B+ Tree

B+ Tree 是一種自平衡樹(self-balancing tree)，它將數據有序地存儲，且在 search、sequential access、insertions 以及 deletions 操作的複雜度上都滿足O(logn)

B+ Tree 可以看作是 BST (Binary Search Tree) 的衍生結構，它的每個節點(node)可以有多個 children，這特別契合 disk-oriented database 的數據存儲方式，每個 page 存儲一個節點，使得樹的結構扁平化，減少獲取索引給查詢帶來的 I/O 成本。

以 M-way B+tree 為例，它的特點總結如下：

- 每個節點最多存儲 M 個 key(M-way)，有 M+1 個 children
- B+ Tree 是 perfectly balanced，即每個 leaf node 的深度都一樣
- 除了 root node，所有其它節點中至少處於半滿狀態，即 M/2−1 ≤ #keys ≤ M−1
- 假設每個 inner node 中包含 k 個 keys，那麼它必然有 k+1 個 children
- B+ Tree 的 leaf nodes 通過雙向鏈表(Sibling Pointers)串聯，從而為 sequential access 提供更高效能的支持
- 3-way B+ tree 例子:
    
    ![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled.png)
    
    - Inner Node(root Node)有2個key，所以有2+1個children
    - 這個tree的"5" key被刪除了，但仍然保留在Inner Node
        - 搜尋"5" key時會搜尋不到，因為所有搜尋都必須去找Leaf Nodes

### **B+ Tree Nodes**

B+ Tree 中的每個 node 都包含一列按 key 排好序的 key/value pairs，key 就是 table index 對應的 column，value 的取值與 node 類型相關，在 inner nodes 和 leaf nodes 中存的內容不同。

The value array for inner nodes will contain pointers to other nodes，Suffix Truncationleaf node 的 values 有兩種：

- Record/Tuple Ids：儲存指向 tuple 的指針(pointer)
- Tuple Data：直接將 tuple data 存在 leaf node 中，但這種方式對於 [Secondary Indexes](https://docs.oracle.com/cd/E17275_01/html/programmer_reference/am_second.html) 不適用，因為 DBMS 只能將 tuple 數據存儲到一個 index 中，否則數據的存儲就會出現冗餘，同時帶來額外的維護成本。

此外，leaf node 還需要存儲相鄰 siblings 的地址以及其它meta data，如下圖所示：

![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%201.png)

# **B+ Tree Operations**

## **Insert**

1. 找到對應的 leaf node，L
2. 將 key/value pair 按順序插入到 L 中
3. 如果 L 還有足夠的空間，操作結束；如果空間不足，則需要將 L 分裂成兩個節點，同時在 parent node 上新增 entry，若 parent node 也空間不足，則遞迴地分裂，直到 root node 為止。

## **Delete**

1. 從 root 開始，找到目標 entry 所處的 leaf node, L
2. 刪除該 entry
3. 如果 L 仍然至少處於半滿狀態，則操作結束；否則先嘗試從 siblings 那裡拆借 entries，如果失敗，則將 L 與相應的 sibling 合併
4. 如果合併發生了，則可能需要遞迴地刪除 parent node 中的 entry

B+Tree 的 Insert、Delete 過程，可參考[這裡](https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html)。

## **In Practice**

- Typical Fill-Factor: 67%
    - Average Fanout = 2*100*0.67 = 234
- Typical Capacities:
    - Height 4: 312,900,721 entries
    - Height 3:     2,406,104 entries
- Pages per level:
    - Level 1 =          1 page   =     8KB
    - Level 2 =      134 pages =     1MB
    - Level 3 =  17956 pages = 140 MB

## **Clustered Indexes**

Clustered Indexes 規定了 table 本身的物理存儲方式，通常即按 primary key 排序存儲，因此一個 table 只能建立一個 cluster index。

有些 DBMSs 對每個 table 都添加Clustered Indexes，如果該 table 沒有 primary key，則 DBMS 會為其自動生成一個

- 例如MySQL

有些 DBMSs 則不支持 clustered indexes。

## **Compound Index**

DBMS 支持同時對多個字段建立 table index（B+ Tree），即 compound index，如

```sql
 CREATE INDEX compound_idx ON table (a, b, c);
```

它可以被用在包含 a 的 condition 的查詢中，如：

```sql
SELECT c
  FROM table
 WHERE a = 5 AND b >= 42 AND c < 77;
 
SELECT c
  FROM table
 WHERE a = 5 AND b >= 42;

SELECT c
  FROM table
 WHERE a = 5;
```

盡管它可以被用在不包含 a 相關 condition 的查詢中，但這約等於全表掃描，因此 DBMS 通常會使用全表掃描。如果使用 hash index 作為 table index，則必須對 (a, b, c) 完全匹配才可以使用。

# B+ Tree Design Choices

## **Node Size**

通常來說，disk 的數據讀取速度越慢，node size 就越大：

- HDD: ~1MB
- SSD: ~10KB
- In-Memory: ~512B

具體情境下的最優大小由 workload 決定。

## **Merge Threshold**

由於 merge 操作引起的修改較大，有些 DBMS 選擇延遲 merge 操作的發生時間，甚至可以利用其它進程來負責週期性地重建 table index。

- 例如有些公司在周末重建table index

## **Variable Length Keys**

B+ Tree 中存儲的value是固定長度的(fixed length)，但 key的長度是可變動的，通常有三種手段來應對：

1. Pointers：存儲指向 key 的指針，需要多一個lookup，現代DB已沒人在用
2. Variable Length Nodes：index中的node size是可變動的，需要精細的內存管理操作，此方法也很少在用
3. Padding: 每個key的size都padding到上限值，讓每個key size都相同，缺點是很浪費空間
    - 例如email address可能只有50個char，但使用Padding會讓每個email address都占用1024 bytes
4. Key Map / Indirection Map：內嵌(Embed)一個pointers array，指向 node 中的 key/val list
    - 圖例
        
        ![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%202.png)
        

## **Non-unique Indexes**

索引針對的 key 可能是非唯一的，通常有兩種手段來應對：

1.  Duplicate Keys：存儲多個相同的 key，這是較常見的做法
2.  Value Lists：每個 key 只出現一次，但同時維護一個linked list，存儲 key 對應的多個unique values，類似 chained hashing

分別如下面兩張圖所示：

- Duplicate keys:
    
    ![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%203.png)
    
- Value Lists
    
    ![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%204.png)
    

### Duplicate Keys

**Approach 1: Append Record Id**

- 附加tuple的unique record id在key後面，確保每個key都是unique
- B+ tree可以確保只搜尋partial keys仍然可以找到tuple
- 大多DB是實作這個方法，缺點是Index較長，需要較多空間來儲存

**Approach 2: Overflow Leaf Nodes**

- 在leaf nodes後面多一個overflow nodes去儲存duplicate keys.
- 要maintain或修改都比較複雜

**圖例**

- 原先有個B+ Tree，要寫入6，但6已重複
    
    ![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%205.png)
    
- **Append Record Id**
    
    ![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%206.png)
    
    - 一般的B+ Tree處理方式，內部會知道兩個6是不同的6
- **Overflow Leaf Nodes**
    
    ![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%207.png)
    
    - 在leaf nodes後面多一個overflow nodes去儲存duplicate keys

## **Intra-node Search**

在node內部搜索，就是在排好序的序列中檢索元素，手段通常有：

- Linear Scan: Scan the key/value entries in the node from beginning to end.
    - 這個方法就不需要先排序
- Binary Search: Jump to the middle key, and then pivot left/right depending on whether that middle key is less than or greater than the search key
    - 這是最常見的方法
- Interpolation：通過 keys 的分佈統計信息來估計大概位置進行檢索

# Optimizations

## **Prefix Compression**

同一個 leaf node 中的 keys 通常有相同的 prefix，如下圖所示：

![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%208.png)

為了節省空間，可以只存所有 keys 的不同的 suffix。

## **Suffix Truncation**

由於 inner nodes 只用於引導搜索，因此沒有必要在 inner nodes 中儲存完整的 key，我們可以只存儲足夠的 prefix 即可，如下圖所示：

- 原先key的長度很長
    
    ![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%209.png)
    
- Suffix Truncation後inner node只儲存足夠的prefix
    
    ![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%2010.png)
    

## **Bulk Insert**

建 B+ Tree 的最快方式是先將 keys 排好序後，再從下往上建樹，如下圖所示：

![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%2011.png)

![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%2012.png)

它會比一個一個寫入還快，因為不需要split或merge，因此如果有大量插入操作，可以利用這種方式提高效率

## **Pointer Swizzling**

Nodes 使用 page id 來存儲其它 nodes 的引用，DBMS 每次需要首先從 page table 中獲取對應的內存地址，然後才能獲取相應的 nodes 本身，如果 page 已經在 buffer pool 中，我們可以直接存儲其它 page 在 buffer pool 中的內存地址作為引用，從而提高訪問效率。

# Additional Index Usage

## Implicit Indexes

許多 DBMSs 會自動創建 index，來幫助施行 integrity constraints，情形包括：

- Primary Keys
- Unique Constraints

![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%2013.png)

其中不包含referential constraints(Foreign Keys)

![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%2014.png)

## Partial Indexes

只針對 table 的子集(subset)建立 index，這可以大大減少 index 本身的大小，如下所示：

```sql
CREATE INDEX idx_foo
          ON foo (a, b)
       WHERE c = 'WuTang';
```

一種常見的使用場景就是以不同時間範圍來為 indexes 分區，即為不同的年、月、日分別建立索引。

## Covering Indexes

如果 query 所需的所有 column 都存在於 index 中，則 DBMS 甚至不用去獲取 tuple 本身即可得到查詢結果，如下所示：

```sql
CREATE INDEX idx_foo ON foo (a, b);

SELECT b FROM foo
 WHERE a = 123;
```

### Index Include Columns

在Create Index時附加其他column，在index-only的Query時使用

These extra columns are only stored in the leaf nodes and are not part of the search key.

![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%2015.png)

## Functional/Expression Indexes

index 中的 key 不一定是 column 中的原始值，也可以是通過計算得到的值，如針對以下查詢：

```sql
SELECT * FROM users
 WHERE EXTRACT(dow FROM login) = 2;
```

直接針對 login 建立索引是沒用的，這時候可以針對計算後的值建立索引：

```sql
CREATE INDEX idx_user_login 
          ON users (EXTRACT(dow FROM login));
```

也可以使用Partial Indexes來做到同樣的事

```sql
CREATE INDEX idx_user_login 
          ON foo (login)
			 WHERE EXTRACT(dow FROM login) = 2;
```

# Trie Index

- 字典樹(Trie)，也被稱為單詞搜尋樹，這個術語來自於retrieval，是一種很特別的樹狀資料結構，如同其名，它就像一本字典，可以讓你快速的依照字母插入、尋找字串，由於高效的特性，特別適用於大量字串出現的時候。
    - 也被稱為Digital Search Tree, Prefix Tree
- 利用每個字的共同前綴(common prefix)當儲存依據，並以此來節省儲存空間以及加速搜尋時間 ，並以此來節省儲空間以及加速搜尋時間。

![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%2016.png)

# Radix Tree

## Radix Tree vs. Trie

radix tree 實際上就是壓縮後的 trie

![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%2017.png)

## 簡介

radix tree 將每個 key 拆成一個序列：

- 樹的高度取決於 keys 的長度
- 不需要重新平衡樹（rebalancing）
- 從 root 到 leaf 的路徑代表相應的 leaf

目前還沒有DB使用Trie或Radix來當作Index的，基本上都是使用B+ Tree

## Binary Comparable Keys

為了讓 keys 能夠合理地拆解成序列，許多類型的 key 都需要特殊處理：

- Unsigned Integers：對小端存儲的機器要把 bits 翻轉一遍
- Signed Integers：需要翻轉 2's-complement 從而使得負數小於正數
- Floats：需要分成多個組，然後存儲成 unsigned integer
- Compound：分別轉化各個 attributes 然後組合起來

舉例如下：

![Untitled](Tree%20Indexes%207cead1f8f35c44c5bbd8c48e3f7491ca/Untitled%2018.png)

# Inverted Indexes

盡管 tree index 非常有利於 point 和 range 查詢，如：

- Find all customers in the 15217 zip code
- Find all orders between June 2018 and September 2018

但對於 keyword search，tree index 就顯得無能為力，如：

- Find all Wikipedia articles that contain the word "Pavlo"

Inverted index是一種索引方法，store a mapping of words to records that contain those words in the target attribute，也被稱做full-text search indexes。它是文件檢索系統中最常用的資料結構。

例如:

- 被索引的文字：
    - T0 = `0. "it is what it is"`
    - T1 = `1. "what is it"`
    - T2 = `2. "it is a banana"`
    
    我們就能得到下面的反向檔案索引：
    
    ```
     "a":      {2}
     "banana": {2}
     "is":     {0, 1, 2}
     "it":     {0, 1, 2}
     "what":   {0, 1}
    ```
    

盡管 DBMSs 在一定程度上支持這種搜索，但更複雜、靈活的查詢就不是它們的專長。有一部分 DBMSs 專門提供復雜、靈活的 keyword search 功能，如 ElasticSearch、Solr、Sphinx 等。

## **Query Types:**

- Phrase Searches: Find records that contain a list of words in the given order.
- Proximity Searches: Find records where two words occur within n words of each other.
- Wildcard Searches: Find records that contain words that match some pattern (e.g., regular expression).

## Design Decisions:

- What To Store: The index needs to store at least the words contained in each record (separated by punctuation characters). It can also include additional information such as the word frequency, position, and other meta-data.
- When To Update: Updating an inverted index every time the table is modified is expensive and slow. Thus, most DBMSs will maintain auxiliary data structures to “stage” updates and then update the index in batches.