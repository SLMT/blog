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
- 發表會議/期刊：Very Large Database (VLDB) 2019 年期刊

## 需要具備的知識

- 知道 Transaciton 是甚麼

## Geo-replicated DBMS

在開始了解論文的問題前，要先理解論文假設的環境。 這個論文假設一個資料庫系統是跑在橫跨世界各地的資料中心的系統。 這類系統我們通常會稱之為 Geo-replicated Database Management System (DBMS)，因為它通常會將資料複製很多份儲存在各地的資料中心中。

下圖展示了一個分散在三個地區的資料庫系統。 通常一個資料中心會保有一份完整的資料，然後每一個資料中心內的資料會再切分成多個不相交的 partition。 因此下圖之中，一共有三份備份 (replica)，然後每個資料中心內有三個 partition。 資料中心之間會需要時常同步資料來保持資料的一致性 (consistency)。

![Geo-replicated DBMS](geo-dbms.png)

這種系統架構已經遍布於各大主要的分散式資料庫中。 這樣的設計不外乎兩個目的：

1. High availability
  - 因為資料有很多備份，如果一份無法服務就可以由其他備份不間斷地服務使用者。
2. Low Read Latency
  - 可以優先從地理位置較近的資料中心更快速地讀取需要的資料。


## 近代 OLTP 分散式 SQL 系統的三大目標

這篇論文假設在前一節探討的 Geo-replicated DBMS 之中，想要再額外達成三個目標：

1. Strict Serializability
2. (對使用者來說) Low Write latency
3. (對管理員來說) High throughput

對於不熟悉 SQL 資料庫系統的人，可能不太能理解甚麼是 Strict Serializability。 Strict Serialiability 是一種保證。 如果系統保證這點，就代表它保證你所有執行的交易造成的結果與某個依序執行交易的結果相同。 有學過 transaction 的人，都知道 transaction 內可能包含多筆操作，因此需要額外的機制去確保不同 transaction 的操作同時發生不會導致結果異常。

這件事情放到分散式系統又更複雜一點。 在分散式系統上除了保障一台機器內的 Serialiability 之外，還要確保：從系統之中執行的 transaction 裡面任意挑出兩個 transaction X 跟 Y，無論我拿到哪個 replica，其執行順序都是相同的。 也就是說，如果今天台灣 replica 執行的順序是 X->Y，那麼拿到美國或歐洲的 replica 也要是相同的順序。

乍聽之下似乎很合理，但是實際上要做到可能沒那麼容易。 假設 X 是送到台灣 replica，Y 是送到美國 replica，這樣很有可能在台灣執行就是先 X 再 Y，在美國則是先 Y 再 X。

為了要解決這個問題，近幾年有幾個熱門方向：

- Google Spanner - 在世界各地的資料中心放原子鐘，以取得一個全世界的絕對時間為 transaction 排序。 Transaction 完成時必須使用一種 consensus protocol 讓全世界的資料中心知道它的存在。 CockroachDB、TiDB 等系統屬於它的變形。
- Deterministic Databases - 在 transaction 執行之前，先使用 consensus protocol 讓全世界的資料中心知道它的存在，並確定執行順序。 然後用 deterministic execution 確保執行順序。 H-Store、VoltDB、Calvin 等系統屬於此類。
- Weak Consistency - 放棄 Strict Serializability 這種嚴格的保證。 讓系統在資料剛寫入的時候可能會在資料中心間不太一致，但是在某些狀況下，或者最後會一致 (Dynamo、Cassandra、Riak)。 甚至可能乾脆只保證較低的 isolation，例如 snapshot isolation (Walter、Jessy、Blotter)。

這些方向都有些問題，Google Spanner 與 Deterministic Database 的 latency 不低，但第三種又只有保證較低的 consisntency。 因此沒有人同時完成這三大目標。

## 主要發現

這篇論文依賴了兩個主要發現來達成這些目標：

1. 使用者通常都會在同一個地區 (或資料中心) 存取自己的資料
2. 

## 參考資料



