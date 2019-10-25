---
title: "論文筆記 - SLOG: Serializable, Low-Latency, Geo-replicated Transactions"
date: 2019-10-20 20:08:27
categories: [Research, Paper]
tags: [paper, dbms, distributed-systems, oltp]
---

今天介紹的這篇論文發表於今年的 PVLDB 期刊。 這篇論文發表了一種新的分散式 SQL 資料庫系統架構，適用的規模可以橫跨世界各區的資料中心，主要針對 Online Transactional Processing (OLTP) 的環境。 他們想要**同時達成**近代分散式資料庫系統的三大目標：(1) Strict Serializability、(2) 低 latency 與 (3) 高 throughput。這是目前尚未有人能完美同時解決的目標。

<!--more-->

## 基本資料

- 論文作者群：Kun Ren, Dennis Li, Daniel J. Abadi
- 研究機構：The University of Maryland
- 發表會議/期刊：Very Large Database (VLDB) 2019 年期刊 [[1]](#r1)

## 需要具備的知識

- 知道 Transaciton 是甚麼
- 知道 Deterministic Database System [[2]](#r2) 的基本概念

## Geo-replicated DBMS

在開始了解論文的問題前，要先理解論文假設的環境。 這個論文假設一個資料庫系統是跑在橫跨世界各地的資料中心的系統。 這類系統我們通常會稱之為 Geo-replicated Database Management System (DBMS)，因為它通常會將資料複製很多份儲存在各地的資料中心中。

下圖展示了一個分散在三個地區的資料庫系統。 通常一個資料中心會保有一份完整的資料，然後每一個資料中心內的資料會再切分成多個不相交的 partition。 因此下圖之中，一共有三份備份 (replica)，然後每個資料中心內有三個 partition。 資料中心之間會需要時常同步資料來保持資料的一致性 (consistency)。

![Geo-replicated DBMS](geo-dbms.png)
圖一、Geo-replicated Database Management System

這種系統架構已經遍布於各大主要的分散式資料庫中。 這樣的設計不外乎兩個目的：

1. High availability
  - 因為資料有很多備份，如果一份無法服務就可以由其他備份不間斷地服務使用者。
2. Low Read Latency
  - 可以優先從地理位置較近的資料中心更快速地讀取需要的資料。


## 近代分散式 OLTP SQL 系統的三大目標

這篇論文假設在前一節說明的 Geo-replicated DBMS 之中，想要再額外達成三個目標：

1. Strict Serializability
2. (對使用者來說) Low Write latency
3. (對管理員來說) High throughput

對於不熟悉 SQL 資料庫系統的人，可能不太能理解甚麼是 strict serializability。 Strict serialiability 是一種保證。 如果系統保證這點，就代表它保證所有執行的交易造成的結果與某個依序執行交易的結果相同。 有學過 transaction 的人，都知道 transaction 內可能包含多筆操作，因此需要額外的機制去確保許多 transaction 的操作同時發生不會導致結果異常。

這件事情放到分散式系統又更複雜一點。 在分散式系統上除了保障一台機器內的 serialiability 之外，還要確保：「從系統之中執行的 transaction 裡面任意挑出兩個 transaction X 跟 Y，無論我拿到哪個 replica，其執行順序都是相同的。」 也就是說，如果今天台灣 replica 執行的順序是 X->Y，那麼拿到美國或歐洲的 replica 也要是相同的順序。

乍聽之下似乎很合理，但是實際上要做到可能沒那麼容易。 假設 X 是送到台灣 replica，Y 是送到美國 replica，這樣很有可能在台灣執行就是先 X 再 Y，在美國則是先 Y 再 X。

為了要解決這個問題，近幾年有幾個熱門方向：

- Google Spanner [[3]](#r3) - 在世界各地的資料中心放原子鐘，以取得一個全世界的絕對時間為 transaction 排序。 Transaction 完成時必須使用一種 consensus protocol (例如 Paxos [[13]](#r13)) 讓全世界的資料中心知道它的存在。 CockroachDB [[4]](#r4)、TiDB [[5]](#r5) 等系統屬於它的變形。
- Deterministic Databases [[2]](#r2) - 在 transaction 執行之前，先使用 consensus protocol 讓全世界的資料中心知道它的存在，並確定執行順序。 然後用 deterministic execution 確保執行順序。 H-Store [[6]](#r6)、VoltDB [[7]](#r7)、Calvin [[8]](#r8) 等系統屬於此類。
- Weak Consistency - 放棄 strict serializability 這種嚴格的保證。 讓系統在資料剛寫入的時候可能會在資料中心間不太一致，但是在某些狀況下或者最後會一致 (如：Dynamo [[9]](#r9)、Cassandra [[10]](#r10))。 甚至可能乾脆只保證較低的 isolation，例如 snapshot isolation (如：Walter [[11]](#r11)、Jessy [[12]](#r12))。

這些方向都有些問題，Google Spanner 與 Deterministic Database 的 latency 不低，因為全世界的資料中心要做一個 consensus。 但是第三種又只有保證較低的 consisntency。 因此沒有人同時完成這三大目標。 不過他們的共通點就是有很高的 throughput。

## 主要發現

這篇論文依賴了兩個主要發現來達成這些目標：

1. 使用者通常都會在同一個地區 (或資料中心) 存取自己的資料。
2. 並非所有 transaction 之間都一定要有 global order，不衝突的 transaction 之間的順序可以任意排序。

第一個發現很容易就可以理解，因為人很少會離開自己的居住區，因此最近的資料中心通常都不會改變。第二個發現可能就不是那麼直覺。簡單來說，如果今天有兩個 transaction T1 跟 T2。如果 T1 修改資料 A，T2 修改資料 B，那麼 T1、T2 無論誰先誰後其實都不影響結果。因此這兩個 transaction 無論如何排序，都不影響 serializability。

這篇論文基於這兩個概念，提出了 SLOG 系統架構。

## SLOG 主要概念：Home Region

SLOG 架構之中主要的概念就是：每一組 records 可能會被跨資料中心複製很多份 replica，但是只有一份 replica 是主要的 (primary)。 存放這個主要的 replica 的區域就叫做 home region。 這個時候，一般馬上就會想到 master-slave 架構。 那 SLOG 跟那種架構又有什麼不同呢？ 主要差異是在常見的 master-slave 架構之中，replica 通常是以整個 database 作為單位，其中 primary replica 一定是整組 database 放在同一個區域中。但是 SLOG 提出的架構是以**一組 records** 作為單位。也就是說，可能有些 records 的 primary replica 在台灣，有些 records 在美國。 如下圖所示，每個方塊代表一組 records。 黑色代表 primary。 A 組的 home region 是 region 1，B 組則是在 region 2。

![Home Region](home-region.png)
圖二、Home Region 示意圖，每個方塊代表一組資料的 replica。

### Single-home Transactions

首先這種做法的好處跟 master-slave 架構相同。 如果今天我想要存取的資料的 home region 都在同一個資料中心 (這種 transaction 被稱之為 single-home transaction)。 此時我就只需要對資料中心內的機器進行 concurrency 的管控，其他資料中心的 replica 只要確保有複製到 primary 的結果就好。 因此 SLOG 在 home 完成 transaction 之後，就會把 log 複製到 slave。 一旦有 N 個資料中心回覆 (N 由使用者設定)，就可以答覆使用者 transaction 完成。 如此一來，就可以省下跨越資料中心做的 concurrency control 以及 two phase commit 這種花時間的 protocol。 這些 protocol 可能要所有資料中心參與，並且花費多個 round-trip time。

至於 slave 要怎麼樣複製 primary replica 的結果？ 論文這裡使用了 Calvin [[8]](#r8) 的 deterministic database 架構，因此複製的時候只需要複製 master 上的 transaction 的指令，在 slave 上只要再跑一次同樣序列的指令就可以得到相同結果 (對這個有興趣的可以看 request log 這篇論文 [[14]](#r14))。

當然，因為有 home region 的關係，所有想要存取該組資料的 transaction 都必須要導向 home region。 例如，以圖二來說，使用者可能會在 region 2 存取 A 組的資料，雖然 region 2 本身有 A 組的備份，但仍會把請求轉給 region 1 處理。 所以才會假設使用者通常不會離自己存取的資料太遠。 不過在這種情況下，論文也有提出一種搬移 home region 的機制，以拉近 home region 與使用者的距離。 細節的部分可以參見論文。

### Multi-home Transactions

那麼 SLOG 這樣的設計相較於一般 primary-slave 架構又有什麼難點呢？ 因為不像一般的 primary-slave 架構，所有資料的主控權都同一個資料中心，導致仍有可能會發生某個 transaction 改的一部分資料的 home region 在台灣，另一部資料的 home region 在美國這種情況。 這種 transaction 就叫做 multi-home transaction。 對這種 transaction 還是會需要在這些資料中心之間做 concurrency control 來維持資料的正確性。 這個部分也是這篇論文的主要貢獻。

關於 multi-home transactions 的處理，我們可以分成兩個部分來看：「multi-home tx 跟 multi-home tx 之間」與「multi-home tx 跟 single-home tx 之間」。

針對「multi-home tx 跟 multi-home tx 之間」，這邊論文採用了 deterministic database [[2]](#r2) 的概念。 在一開始先把所有 multi-home transactions 透過一個 total ordering protocol 進行排序，確定了這些 transaction 的 global order 後，再送到 deterministic execution engine 執行，以確保 consistent 的結果。 換句話說，對於這種 transaction，就使用之前其他論文 [[2]](#r2) 的做法。 如圖三，Tx G.1 與 Tx G.2 無論送到哪個 region，都會被強制送進 total ordering layer 進行全域的排序之後，再送到下面的系統開始執行。

![Multiple Home - Total Oredering](multi-home-1.png)
圖三、Multi-home Transactions 會進行全域排序

排序完成之後，在正式開始執行這些 transaction 之前，會先為每個 transaction 建立多個 lock transaction。 每個 lock transaction 負責鎖住某個 region 內的資料。 因此一個 multi-home transaction 存取的資料的 home region 有幾個，就會產生幾個 lock transaction。 如圖四，Tx G.1 想要存取 region 1 跟 2 的資料，所以產生了兩個 lock transaction G.1.1 跟 G.1.2，並送到所屬的 home region。 而當一個 region 的 lock 都拿到之後，這個 lock transaction 就會依循 replication 的規則被複製到其他 region。 而 multi-home transaction 的本體只會在一個 region 收到所有 lock transaction 的 log 時 (確保所有參與的 region 都把資料鎖住)，該 region 才會執行這個 transaction。

![Multiple Home - Lock Transactions](multi-home-2.png)
圖四、Multi-home Transactions 會再產生 Lock Transaction

如果對分散式資料庫系統有點了解的人，應該可以發現第一步的 total ordering (全域排序) 其實就足以保證 consistency。 那為什麼還要多此一舉用什麼 lock transaction 呢？ 這主要是為了第二種狀況：「multi-home tx 跟 single-home tx 之間」的 consistency。 因為 single-home transaction 不會參與全域排序，因此若沒有額外處理的話，可能就會跟 multi-home transaction 發生錯序的情況。 這種情況下 lock transactions 的插入就可防止這種事情發生。 因為 lock transaction 會把資料鎖住，而且必須等所有 lock transaction 都完成才會執行，所以可以確保 multi-home transaction 跟 single-home transaction 的先後順序。

如圖五所示，一個 multi-home transaction tx G.1 跟三個 single-home transactions tx 1.1, tx 1.2, tx 2.1 同時出現。 那他們可能會出現如圖五的執行順序。 可以看到經由保證這樣的 global order，可以得到如圖五右方的紫色框框內的順序。 而這個順序可以經由複製 log 傳播到所有資料中心。 要注意的是，我們並沒有限制 tx 1.1 與 tx 2.1 的先後順序。 因為這兩個 tx 其實存取的資料沒有交集，因此執行的順序其實不重要！ 這也是利用了最一開始說的第二個發現。

![Multiple Home - 跟 Single-home 同時出現](multi-home-3.png)
圖五、Multi-home Transactions 與 Single-home Transactions 同時出現

以上就是這篇論文的主要概念。 我只是說明了這篇論文最主要的概念與貢獻而已，想要知道更詳細的資訊，像是 protocol 或是其他的細節的話可以閱讀論文。

## 實驗

論文上有許多實驗，這邊我們只看一個實驗就好。 圖六展示了他們的系統跑在 YCSB Benchmark 上 latency 的累積分佈函數圖。 線越左邊代表大多的 transaction latency 越低。 SLOG-B 代表 transaction 在一個 region 完成就直接回覆使用者，SLOG-HA 代表會等待 log 複製到 N 台機器後才回覆使用者。 0%, 1%, 10%, 100% mh 那個是代表有多少比例的 multi-home transactions。 詳情可以看論文。

![實驗](ex.png)
圖六、YCSB Lateny CDF

這個實驗的結論就是，只要 multi-home transaction 不要多得太誇張，都可相較於 baseline Calvin [[8]](#r8) 大幅降低 latency。 我認為這個實驗結果是很驚人的，因為這篇論文提出的架構無疑可以確保 strict serializability (請參照論文上的證明)。 而因為使用了 deterministic database 的架構，也確保了 high throughput (其他實驗也有證明這點)。 最後也有較低的 latency，因此這也許會成為下一代的資料庫系統架構。

## 參考資料

<a name='r1'>[1]</a> http://www.vldb.org/pvldb/vol12.html
<a name='r2'>[2]</a> Thomson, Alexander, and Daniel J. Abadi. "The case for determinism in database systems." Proceedings of the VLDB Endowment 3.1-2 (2010): 70-80.
<a name='r3'>[3]</a> Corbett, James C., et al. "Spanner: Google’s globally distributed database." ACM Transactions on Computer Systems (TOCS) 31.3 (2013): 8.
<a name='r4'>[4]</a> https://github.com/cockroachdb/cockroach
<a name='r5'>[5]</a> https://github.com/pingcap/tidb
<a name='r6'>[6]</a> https://hstore.cs.brown.edu/
<a name='r7'>[7]</a> https://www.voltdb.com/
<a name='r8'>[8]</a> Thomson, Alexander, et al. "Calvin: fast distributed transactions for partitioned database systems." Proceedings of the 2012 ACM SIGMOD International Conference on Management of Data. ACM, 2012.
<a name='r9'>[9]</a> https://aws.amazon.com/tw/dynamodb/
<a name='r10'>[10]</a> http://cassandra.apache.org/
<a name='r11'>[11]</a> Y. Sovran, R. Power, M. K. Aguilera, and J. Li. Transactional Storage for Geo-replicated Systems. In Proceedings of the Symposium on Operating Systems Principles, SOSP ’11, pages 385–400, 2011.
<a name='r12'>[12]</a> M. S. Ardekani, P. Sutra, and M. Shapiro. Non-monotonic Snapshot Isolation: Scalable and Strong Consistency for Geo-replicated Transactional Systems. In Proceedings of the 2013 IEEE 32nd International Symposium on Reliable Distributed Systems, SRDS ’13, pages 163–172, 2013.
<a name='r13'>[13]</a> Lamport, Leslie. "Paxos made simple." ACM Sigact News 32.4 (2001): 18-25.
<a name='r14'>[14]</a> N. Malviya, A. Weisberg, S. Madden, and M. Stonebraker, "Rethinking main memory OLTP recovery," in Data Engineering (ICDE), 2014 IEEE 30th International Conference on, 2014, pp. 604-615.