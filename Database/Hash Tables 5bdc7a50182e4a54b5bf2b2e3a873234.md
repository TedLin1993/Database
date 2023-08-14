# Hash Tables

本節開始之前，先看一下目前課程的進度狀態：

![https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZlixmK1o4A-8LSbzNf%2FScreen%20Shot%202019-02-28%20at%209.33.02%20AM.jpg?alt=media&token=da63e291-f8ce-435a-b295-dc76db7b39ec](https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZlixmK1o4A-8LSbzNf%2FScreen%20Shot%202019-02-28%20at%209.33.02%20AM.jpg?alt=media&token=da63e291-f8ce-435a-b295-dc76db7b39ec)

為了支持 DBMS 更高效地從 pages 中讀取數據，DBMS 的設計者需要靈活運用一些數據結構及算法，其中對於 DBMS 最重要的兩個是：

- Hash Tables
- Trees

它們可能被用在 DBMS 的多個地方，包括：

- Internal Meta-data
- Core Data Storage: tuples in a page
- Temporary Data Structures
- Table Indexes

在做相關的設計決定時，通常需要考慮兩個因素：

- Data Organization：如何將這些數據結構合理地放入 memory/pages 中，以及為了支持更高效的訪問(access)，應當存儲哪些信息
- Concurrency：如何支持數據的並發訪問(multiple threads access data)

# Hash Tables

Hash Table 是 associative array/dictionary ADT 的實現，它將鍵映射成對應的值。

Hash Table 主要分為兩部分：

- Hash Function：
    - How to map a large key space into a smaller domain
    - Trade-off between being fast vs. collision rate
        - collision: 不同的value有同一個key
- Hashing Scheme：
    - How to handle key collisions after hashing
    - Trade-off between allocating a large hash table vs. additional instructions to find/insert keys

## Hash Functions

由於 DBMS 內使用的 Hash Function  並不會暴露在外，因此沒必要使用加密（cryptographic）哈希函數，我們希望它速度越快，collision rate 越低越好。目前各 DBMS 主要在用的 Hash Functions 包括：

- [MurmurHash (2008)](https://github.com/aappleby/smhasher): Designed to a fast, general purpose hash function
- [Google CityHash (2011)](https://github.com/google/cityhash): Based on ideas from MurmurHash2, Designed to be faster for short keys (<64 bytes).
- [Google FarmHash (2014)](https://github.com/google/farmhash): Newer version of CityHash with better collision rates
- [CLHash (2016)](https://github.com/lemire/clhash): Fast hashing function based on carry-less multiplication.

![Untitled](Hash%20Tables%205bdc7a50182e4a54b5bf2b2e3a873234/Untitled.png)

## Hashing Scheme

### Linear Probe Hashing

Linear Probing面對衝突的解決方式是針對當前位置去尋找下一個沒放值的Element，並將值存入

例子:

![Untitled](Hash%20Tables%205bdc7a50182e4a54b5bf2b2e3a873234/Untitled%201.png)

- C的hash key和A重複，所以將C存在下一個空的slot

Linear Probing在insert/read很方便，但是update/delete很麻煩

### Non-Unique Keys

當 keys 可能出現重復，但 value 不同時，有兩種做法：

1. Separate Linked List: Store values in separate storage area for each key
    
    ![Untitled](Hash%20Tables%205bdc7a50182e4a54b5bf2b2e3a873234/Untitled%202.png)
    
    - 優點: 讀取的效率更高
    - 缺點: 寫入的效率較低，因為每個key可能都需要一個page來儲存value，即便那個key只有一個value
2. Redundant Keys: Store duplicate keys entries together in the hash table
    
    ![Untitled](Hash%20Tables%205bdc7a50182e4a54b5bf2b2e3a873234/Untitled%203.png)
    
    - 優點: 寫入效率較高，新的key-value只需要append到下一個entry
    - 缺點: 讀取的效率較低
    - 這是主流的做法，沒甚麼人會使用Separate Linked List

Linear Probe Hashing通常為了減少 collision 和 comparisons，Hash Table 的大小應當是 table 中元素量的兩倍左右。

### Robin Hood Hashing

Robin Hood Hashing  是 Linear Probe Hashing 的變種，為了防止 Linear Probe Hashing 出現連續區域導致頻繁的 probe 操作。基本思路是 “劫富濟貧”，即每次比較的時候，同時需要比較每個 key 距離其原本位置的距離（越近越富餘，越遠越貧窮），如果遇到一個已經被佔用的 slot，如果它比自己富餘，則可以直接替代它的位置，然後把它順延到新的位置。

例子:

- 一開始hash table有A, B，現在插入C
    
    ![Untitled](Hash%20Tables%205bdc7a50182e4a54b5bf2b2e3a873234/Untitled%204.png)
    
    - C要插入的slot是A的slot，所以往後順延一格，並記錄C和最佳位置距離1
- 再插入D到C的位置
    
    ![Untitled](Hash%20Tables%205bdc7a50182e4a54b5bf2b2e3a873234/Untitled%205.png)
    
- 接著插入E到A的位置
    
    ![Untitled](Hash%20Tables%205bdc7a50182e4a54b5bf2b2e3a873234/Untitled%206.png)
    
    - E一直往後順延找空的slot，一直找到D的位置時，因為E距離最佳位置為2，大於D，於是搶走D的slot，並讓D往後順延找空的slot

但效率其實沒有比一般的Linear Probe Hashing要好，因為要一直改動hash table

### Cuckoo Hashing

- Maintain multiple hash tables with different hash functions
    - 通常只有2個hash table，多的hash table會增加不必要的overhead
- On insert, check every table and pick anyone that has a free slot
- If no table has free slot, evict element from one of them, and rehash it to find a new location
- If we find a cycle, then we can rebuild the entire hash tables with new hash functions
    - 如果產生cycle就將hash table空間加倍
- 例子:
    - Insert B
        
        ![Untitled](Hash%20Tables%205bdc7a50182e4a54b5bf2b2e3a873234/Untitled%207.png)
        
        - Insert B時，因為Hash Table #1的位置已經被佔用了，所以B會寫到Hash Table #2
    - Insert C
        
        ![Untitled](Hash%20Tables%205bdc7a50182e4a54b5bf2b2e3a873234/Untitled%208.png)
        
        - C要的位置都被占用，他就隨機選一個位置來搶，假設他搶走B
        
        ![Untitled](Hash%20Tables%205bdc7a50182e4a54b5bf2b2e3a873234/Untitled%209.png)
        
        - B就搶走A的位置，現在換A被搶
        
        ![Untitled](Hash%20Tables%205bdc7a50182e4a54b5bf2b2e3a873234/Untitled%2010.png)
        
        - 最後A找到空的slot
- 使用的DB: IBM DB2

## 小結

以上介紹的 Hash Tables 要求使用者能夠預判所存數據的總量，否則每次數量超過范圍時都需要重建 Hash Table。它可能被應用在 Hash Join 的場景下，如下圖所示：

![https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm-oH5M7QtWAd6rdjS%2FScreen%20Shot%202019-02-28%20at%2010.50.56%20AM.jpg?alt=media&token=baa94ccd-cc5d-41c9-90d6-f9dc786d3461](https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm-oH5M7QtWAd6rdjS%2FScreen%20Shot%202019-02-28%20at%2010.50.56%20AM.jpg?alt=media&token=baa94ccd-cc5d-41c9-90d6-f9dc786d3461)

由於 A, B 表的大小都知道，我們就可以預判到 Hash Table 的大小。

# Dynamic Hash Tables

與 Static Hash Tables 需要預判最終數據量大小的情況不同，Dynamic Hash Tables 可以依照需求擴充或縮減table size，本節主要介紹 Chained Hashing，Extendible Hashing 和 Linear Hashing。

## Chained Hashing

Chained Hashing 是 Dynamic Hash Tables 的 HelloWorld 實現，每個 key 對應一個鏈表，每個節點是一個 bucket，裝滿了就再往後掛一個 bucket。需要寫操作時，需要請求 latch。

![https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm1nxoABpPpsTAZUoj%2FScreen%20Shot%202019-02-28%20at%2010.59.44%20AM.jpg?alt=media&token=8e479c62-7875-4117-bec5-96b1990e0c15](https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm1nxoABpPpsTAZUoj%2FScreen%20Shot%202019-02-28%20at%2010.59.44%20AM.jpg?alt=media&token=8e479c62-7875-4117-bec5-96b1990e0c15)

這麼做的好處就是簡單，壞處就是最壞的情況下 Hash Table 可能降級成Link List，使得操作的時間複雜度降格為 O(n)。

## Extendible Hashing

Extendible Hashing 的基本思路是一邊擴容，一邊 rehash，如下圖所示：

- 透過variable的前面幾個bit決定要放到哪個bucket
    
    ![https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm3RAyswk-XP1ucR0t%2FScreen%20Shot%202019-02-28%20at%2011.06.50%20AM.jpg?alt=media&token=516c091b-732f-408d-8846-13965cfb551a](https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm3RAyswk-XP1ucR0t%2FScreen%20Shot%202019-02-28%20at%2011.06.50%20AM.jpg?alt=media&token=516c091b-732f-408d-8846-13965cfb551a)
    
    - global和local bucket會記錄要看前面幾個bit，例如現在global要看前面2個
- Insert B，開頭2個bit是10，所以寫入第2個bucket
    
    ![https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm3dmy1q873AKttSwl%2FScreen%20Shot%202019-02-28%20at%2011.07.47%20AM.jpg?alt=media&token=085ff0b4-2892-425b-8402-81805be6e720](https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm3dmy1q873AKttSwl%2FScreen%20Shot%202019-02-28%20at%2011.07.47%20AM.jpg?alt=media&token=085ff0b4-2892-425b-8402-81805be6e720)
    
- Insert C，開頭和B一樣，所以也是寫入第2個bucket，但這個bucet滿了
    
    ![https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm3tcLbF38x4TxrOTC%2FScreen%20Shot%202019-02-28%20at%2011.08.47%20AM.jpg?alt=media&token=5e29616b-af2a-4f6d-9f3e-bf0b4261b4f3](https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm3tcLbF38x4TxrOTC%2FScreen%20Shot%202019-02-28%20at%2011.08.47%20AM.jpg?alt=media&token=5e29616b-af2a-4f6d-9f3e-bf0b4261b4f3)
    
- 因為有bucket滿了不給塞，因此將這個bucket拆分成2個bucket，並改成看前3個bit來決定要塞在哪個bucket
    
    ![https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm431TmIL6e5dRFRKs%2FScreen%20Shot%202019-02-28%20at%2011.09.34%20AM.jpg?alt=media&token=49b3a2ad-dbb0-46bf-b7ca-0dc527499def](https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm431TmIL6e5dRFRKs%2FScreen%20Shot%202019-02-28%20at%2011.09.34%20AM.jpg?alt=media&token=49b3a2ad-dbb0-46bf-b7ca-0dc527499def)
    
- 再將C寫到新的bucket
    
    ![https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm49uT7iKIa7MNhOZ3%2FScreen%20Shot%202019-02-28%20at%2011.10.02%20AM.jpg?alt=media&token=e3352e28-5e49-4fa5-ba6a-ac2fe2f6e359](https://gblobscdn.gitbook.com/assets%2F-LMjQD5UezC9P8miypMG%2F-LZm-dCZw2aX5oghZvtx%2F-LZm49uT7iKIa7MNhOZ3%2FScreen%20Shot%202019-02-28%20at%2011.10.02%20AM.jpg?alt=media&token=e3352e28-5e49-4fa5-ba6a-ac2fe2f6e359)
    

## Linear Hashing

基本思想：維護一個指針(pointer)，指向下一個將被拆分的 bucket，每當任意一個 bucket 溢出（標准自定，如利用率到達閾值等）時，將指針指向的 bucket 拆分。

# 總結

Hash Tables 提供 O(1)的訪問(look-ups)效率，因此它被大量地應用於 DBMS 的內部實現中。即便如此，它並不適合作為 table index 的數據結構

因為hash table只能做single key lookup，它不能做range scan, partial key

table index 的首選就是下節將介紹的 B+ Tree。