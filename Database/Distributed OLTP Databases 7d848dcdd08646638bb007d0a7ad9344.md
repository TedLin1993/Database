# Distributed OLTP Databases

上節課我們介紹了分佈式事務的去中心化實現：

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled.png)

應用程序要發起一次事務時，先通過某種方式選擇這個事務的 master node，並向它發送事務開始的請求。

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%201.png)

master node 同意後，應用程序向事務涉及的節點發送數據更新請求，此時每個節點都只是執行請求但尚未提交。

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%202.png)

master node 收到所有節點的響應後，向 master node 發送事務提交的請求，master node 再問其它節點是否可以安全提交。若是，則確保所有節點提交事務後由 master node 返回成功，若否，則確保所有節點回滾成功，同時由 master node 返回失敗。

但上節課我們沒有討論如何確保所有節點都認同事務提交，所有節點在同意之後實際執行了提交。這裡有很多細節需要考慮：

- 如果一個節點發生了故障怎麼辦？
- 如果這期間節點之間消息傳遞延遲了怎麼辦？
- 我們是否需要等待所有都節點都同意？如果集群較大等待時間就會由於短板效應而增加。

本節我們就來討論一下這些問題。

# Assumption

我們首先需要假設在分佈式數據庫中，所有節點都是好孩子，都受到嚴格的控制，不會耍壞心眼，讓它提交就提交，讓它回滾就回滾。

如果我們不能信任其它節點，那我們將需要 **Byzantine Fault Tolerant** 協議來實現分佈式事務(blockchain)。

幾乎所有的分佈式數據庫都符合我們的假設，除了區塊鏈。

# Agenda

- Atomic Commit Protocols
- Replication
- Consistency Issues (CAP)
- Federated Databases

# Atomic Commit Protocols

常見的 Atomic Commit Protocols 包括：

- 2PC
- 3PC(沒人會這麼做)
- Paxos
- Raft: Paxos的變體
- ZAB (Apache Zookeeper)
- Viewstamped Replication

本課討論 2PC 和 Paxos

## Two-Phase Commit (2PC)

---

2PC 的詳細討論可參考[這篇文章](/open-courses/mit-6.824/2pc-and-3pc)。這裡羅列一下 PPT 的一些內容。

### 2PC Success

- 應用程序在發送事務提交請求到其選取的 master node/coordinator (下文稱 coordinator) 後，進入 2PC 的第一階段：准備階段 (Prepare Phase)
    
    ![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%203.png)
    
- coordinator 向其它節點發送 prepare 請求，待所有節點回復 OK 後，進入第二階段：提交階段 (Commit Phase)
    
    ![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%204.png)
    
- coordinator 向其它節點發送 commit 請求：
    
    ![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%205.png)
    
- 待所有節點在本地提交，並返回 OK 後，coordinator 返回成功消息給應用程序。
    
    ![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%206.png)
    

### 2PC Abort

- 在 Prepare 階段，其它節點如果無法執行該事務，則返回 Abort 消息
    
    ![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%207.png)
    
- 此時 coordinator 可以立即將事務中止的信息返回給應用程序，同時向所有節點發送事務中止請求
    
    ![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%208.png)
    
- coordinator 需要保證所有節點的事務全部回滾：
    
    ![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%209.png)
    

### 2PC Optimizations

2PC 有兩個優化技巧：

- Early Prepare Voting：假如你向遠端節點發送的請求是最後一個，遠端節點就可以利用這個信息直接在最後一次響應後返回 Prepare 階段的響應，即直接告訴 coordinator 事務是否可以提交。
- Early Acknowledgement After Prepare：實際上在准備階段完成後，如果所有節點都已經回復 OK，即認為事務可以安全提交，此時 coordinator 可以直接回復應用程序事務提交已經成功。
    - 這符合 2PC 的語義，只要 Prepare Phase 通過，那麼所有節點必須保證能夠提交事務，Commit Phase 無論如何必須執行成功。

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%2010.png)

### Fault Tolerant

在 2PC 的任何階段都可能出現節點崩潰。如何容錯是 2PC 需要解決的重要問題。

首先，所有節點都會將自己在每個階段所做的決定落盤(write to disk)，類似 WAL，然後才將這些決定發送給其它節點。這就保證了每個節點在發生故障恢復後，能知道自己曾經做過怎樣的決定。

在 coordinator 收到所有其它節點 Prepare 節點的 OK 響應後，且 coordinator 將事務可以進入 Commit 階段的信息寫入日誌並落盤後，這個事務才真正被認為已經提交。

如果 coordinator

- 在 Commit Phase 開始之前發生故障，那麼其實該事務相關的任何改動都未落盤，不會產生髒數據，coordinator 在故障恢復後可以繼續之前的工作
- 在 Commit Phase 開始之後發生故障，其它節點需要等待 coordinator 恢復，或用其它方式確定新的 coordinator，接替之前的 coordinator 完成剩下的工作。

如果 participant

- 在 Commit Phase 之前發生故障，那麼 coordinator 可以簡單地利用超時機制來直接認為事務中止
- 在 Commit Phase 之後發生故障，那麼 coordinator 需要通過不斷重試，保證事務最終能夠完成。

由於 2PC 中許多決定都依賴於所有節點的參與，可能會出現 live lock，影響事務的推進效率，於是就有了共識算法。

## Paxos

---

paxos 屬於共識協議，coordinator 負責提交 commit 或 abort 指令，participants 投票決定這個指令是否應該執行。與 2PC 相比，paxos 只需要等待大多數 participants 同意，因此在時延上比 2PC 更有保障。

paxos 最早在 Leslie Lamport 的 The Part-Time Parliament 論文中提出，盡管這篇文章是 1998 年發表的，但早在 1992 年，他為了證明不存在這種擁有容錯能力的共識算法，就發現了 Paxos。但由於他不聽從論文審校人的建議修改，這篇論文就沒有發表，若干年後，當人們開始嘗試解決這個問題時，他才將這篇論文拿出來，聲明自己早就已經解決了該問題。但這篇論文比較晦澀難懂，後來他在 2001 年又發表了一篇名為 Paxos Made Simple，仍然沒有多少人能夠看懂。直到 Google 在 2007 年發表了名為 Paxos Made Live - An Enginnering Perspective 的文章，才終於讓更多人理解了其中的思想。

我們可以將 2PC 理解成是 Paxos 的一種特殊情況。接下來我們看一下 Paxos 的工作過程。在 Paxos 中，coordinator 被稱為 proposer，participants 被稱為 acceptors，如下圖所示:

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%2011.png)

paxos 需要存在多數節點，因此我們這裡有 3 個 acceptors。應用程序首先向 proposer 發起事務提交請求：

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%2012.png)

- proposer 向所有節點發出 Propose(提議)，類似 2PC 的 Prepare 階段，但與 2PC 不同，節點回復 Agree 時無需保證事務肯定能執行完成
- 此外如果其中一個 acceptor (Node 3) 發生故障，proposer 依然獲得了大多數節點的同意，不會阻礙 Paxos 協議的推進。

獲得多數節點同意後，proposer 就會繼續向所有節點發送 Commit 請求：

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%2013.png)

- 僅當多數節點回復 Accept 後 proposer 才能確定事務提交成功，回復給應用程序。

即使在 Commit 階段，每個 acceptor 仍然可以拒絕請求。舉例如下：

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%2014.png)

假設有兩個 Proposer 同時存在，這是 proposer 1 提出邏輯時刻為 n 的 proposal，隨後所有 acceptors 都回復 agree。

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%2015.png)

隨後 proposer 2 提出邏輯時刻為 n+1 的 proposal，接著 proposer 1 發起邏輯時刻為 n 的 commit 請求。由於 acceptors 收到了邏輯時刻更新的請求，因此它們拒絕 proposer 1 的請求：

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%2016.png)

隨後 proposer 2 完成剩下的協議步驟，成功提交事務：

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%2017.png)

有沒有可能出現兩個 proposer 互相阻塞對方，導致 commit 永遠不成功？這時就需要 Multi-Paxos。

### Multi-Paxos

如果系統中總是有任意數量的 proposer，達成共識就可能變得比較困難，那麼我們是不是可以先用一次 Paxos 協議選舉出 Leader，確保**整個系統中只有一個 proposer**，然後在一段時間內，它在提交事務時就**不需要 Propose 操作**，直接進入 Commit 即可。

這樣就能大大提高 Paxos 達成共識的效率。到達一段時間後，proposer 的 Leader 身份失效，所有參與者重新選舉。

你可能會問，每次選舉的時候會不會出現互相阻塞的現象？如果在實踐上我們用一些合理的優先和隨機退後 (backoff) 機制，就可以減少阻塞的次數，概率上保證選舉能夠最終成功。

## 2PC vs. Paxos

---

目前絕大多數分佈式數據庫的各個節點一般部署距離較近、且網絡連接質量較高，網絡抖動和節點故障發生的概率比較低，因此它們多數採用 2PC 作為 Atomic Commit Protocol。

2PC 的優點在於其網絡通信成本低，在絕大多數情況下效率高，缺點在於如果出現 coordinator 故障，則會出現一定時間的阻塞。

Paxos 的優點在於可以容忍少數節點發生故障，缺點在於通信成本較高。

# Replication

分佈式數據庫還需要複製數據到冗餘的節點上，提高資料庫本身的可用性。

在 Replication 的實現方案上，有以下 4 個核心的設計決定需要考慮：

- Replica Configuration
- Propagation Scheme
- Propagation Timing
- Update Method

## Replication Configuration

---

該設計決定主要討論的是如何設計複製節點之間的關係，如何分配讀寫流量。

### Approach #1: Master-Replica

- 所有object的write都指向 master
- master 更新本地數據後，將這些數據更新給其它複製節點 (無需使用 atomic commit protocol)
- Read-only事務可以被允許在複製節點上執行，取決於對一致性的要求，以及資料庫本身實現的隔離級別
- 如果 master 節點崩潰，就利用共識算法選舉出新的 master。

### Approach #2: Multi-Master

不同節點可以同時成為 master，同時接受寫事務請求。但複製節點之間需要通過 atomic commit protocol 來保證寫入不同 master 的數據之間不存在沖突。

二者的對比如下圖所示：

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%2018.png)

FB 最早採用的就是 Master-Replica 的配置，所有的寫請求都被路由到唯一的 master 區域，然後再由 master 將數據傳播到其它 replica 區域。

- 因此用戶在發朋友圈時，實際上其所在區域的數據庫中可能還不存在他剛發的朋友圈，FB 通過在 cookie 中先保存用戶剛發表的朋友圈，來製造數據已經寫入的假象。
- 後來 FB 將配置修改成 Multi-Master 的方案，並自行設計了不同 master 之間數據同步和解沖突的方案。

### K-Safety

K-safty 指的是同步複製節點 (In-sync Replica) 數量與複製資料庫的容錯性的關系。

通常如果同步複製節點小於 K，資料庫則不得不停下，直到同步複製節點的數量重新滿足要求。

K 的大小取決於你對數據丟失的容忍度。

## Propagation(傳播) Scheme

---

當一個事務在複製數據庫上提交時，資料庫需要決定事務是否應該等待數據改動被成功傳播到其它節點後才向客戶端發送 ack。

顯然，我們有兩種 propagation schemes，同步和異步，二者分別對應強一致性(Strong Consistency)和最終一致性(Eventral Consistency)。

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%2019.png)

這也是傳統關聯性資料庫與 NoSQL 資料庫之間的差異之一

- 傳統關聯性資料庫強調一致性，主節點必須等待複製節點落盤後，才能告訴客戶端數據修改成功
- NoSQL 資料庫為了性能不會等待複製節點落盤，而是在本地落盤後直接告訴客戶端數據修改成功，同時異步將數據傳播給複製節點。
    - 當然如果 NoSQL 數據庫的主節點在還沒來得及傳播數據時永久故障，那麼這條數據也將消失。

## Propagation Timing

---

主節點什麼時候開始將日誌數據同步給複製節點？

### Approach #1: Continuous

- DBMS 在生成日誌時就持續地將日誌傳播給複製節點，只要不出現問題，這種做法的效率更高。
- DBMS 還需要將事務提交或中止的信息也傳播給複製節點，保證事務在複製節點也能統一提交或中止。
    - 缺點在於：如果事務最終中止，那麼複製節點就做了無用功。
- 大部分資料庫為了效率採用的都是這種方案。

### Approach #2: On Commit

- DBMS 只在一個事務徹底執行完成時才將日誌傳播給複製節點，這樣如果事務中止，複製節點就什麼事都不用做。
    - 缺點在於，由於需要等待事務結束時才同步數據，整體同步效率較低。

## Active vs. Passive

---

主節點與複製節點執行事務的順序。

### Approach #1: Active-Active

事務同時在多個複製節點上獨立執行，在執行結束時需要檢查兩邊數據是否一致。

### Approach #2: Active-Passive

事務先在一個複製節點上執行，然後將數據的變動傳播給其它複製節點。

# CAP Theorem

分散式系統不可能以下三者同時達到，最多只能同時達到兩者，這個理論已被證明為true

- Consistent
- Always Available
- Network Partition Tolerant

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%2020.png)

通常 P 是給定的，即 network partition 是必然發生的，所有的分佈式系統都必須做到 network partition tolerant，因此實際的分佈式數據庫只能在 consistency 和 availability 之間取捨。

面對故障，DBMS 如何處理決定了它們在 C 和 A 上的取捨

- 傳統關聯型資料庫或 NewSQL 資料庫通常會在多數節點發生故障時停止接受數據寫請求
- NoSQL 數據庫會提供事後解沖突的機制，因此只要有部分節點還可用，他們的系統就可以繼續運行

# Federated(聯邦) Databases

到現在為止，我們都假設我們的分佈式系統中每個節點都運行著相同的 DBMS，但在實際生產中，通常公司內部可能運行著多種類型的 DBMS，如果我們能夠在此之上抽象一層，對外暴露統一的數據讀寫接口也是一個不錯的想法。這就是所謂的聯邦數據庫。

然而實際上這個很難，也從沒有人把這種方案實現地很好。不同的數據模型、查詢語句、系統限制，沒有統一的查詢優化方案，大量的數據複製，都使得這種方案比較難產。

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%2021.png)

PostgreSQL 有 Foreign Data Wrappers 組件能提供這種方案，它能識別請求的類型並將其發送給相應的後端數據庫：

![Untitled](Distributed%20OLTP%20Databases%207d848dcdd08646638bb007d0a7ad9344/Untitled%2022.png)

# Conclusion

所有針對 Distributed OLTP 數據庫的討論都是基於節點友好的假設，只有區塊鏈數據庫假設節點是有惡意的。處理惡意節點問題需要使用不同的事務提交協議。