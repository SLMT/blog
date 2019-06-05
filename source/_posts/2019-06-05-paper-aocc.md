---
title: "論文筆記 - Adaptive Optimistic Concurrency Control for Heterogeneous Workloads"
date: 2019-06-05 09:20:39
categories: [Research, Paper]
tags: [paper, dbms, concurrency-control, occ, adaptive]
---

這篇論文 [1] 研究了 Optimistic Concurrency Control (OCC) 的 Validation 做法，並將之大致分成兩類。 發現使用在 HTAP Workloads 上時，有時候第一種做法較快，有時候另一種做法較快。 因此他們以此作為發想，嘗試將兩種 Validation 做法同時使用在系統裡，發現這樣可以讓能夠讓這兩種做法互相補足對方的短處，達到更好的效能。

<!--more-->

## 基本資料

- 論文作者群：Jinwei Guoy, Peng Caiyx, Jiahao Wangy, Weining Qiany, Aoying Zhou
- 研究機構：East China Normal University （中國華東師範大學）、Guilin University of Electronic Technology（中國桂林電子科技大學）
- 發表會議/期刊：Very Large Database (VLDB) 2019 年期刊

## 需要具備的知識

- 知道 Transaciton 是甚麼
- 了解 Concurrency Control 的目的
- 對 Optimistic Concurrency Control 有基本認識

## HTAP Workloads

HTAP 是 hybrid transactional and analytical processing 的縮寫。 代表的是 workloads 同時具有 OLTP (online transaction processing) 跟 OLAP (online analytical processing) 的特性。 換句話說，就是他會跟 OLTP 一樣同時有很多小型 transaction (存取 records 只有數十個以下) ，而且有大量的寫入動作。同時也會跟 OLAP 一樣，有那種需要查詢大量資料（存取超過數千筆 records），並作某種總結的動作的 transaction。

這篇論文說明，銀行在即時偵測洗錢的時候，就會是屬於這種 HTAP Workloads，而且隨著時代演進，需求越來越高。

## Optimistic Concurrency Control

Optimistic Concurrency Control 是早在 1981 年 [2] 就提出的一種 concurrency control 作法。 它假設 transaction 之前相衝的機率很低，因此主張在存取資料的時候不預先把資料用 lock 保護，而是在最後要 commit 之前在檢查是否有問題。 因為他的做法相較於使用 locks 的做法樂觀，所以叫做樂觀的 (optismistic) concurrency control。

通常 OCC 分成三個階段：read phase、validation phase、write phase。 Read phase 時正常執行 transaction 的動作，若要讀資料就直接讀取，但是要寫入資料就先暫存在其他 transaction 看不到的空間。 Validation phase 檢查稍早執行 transaction 時，是否有可能會違反 transaction isolation（通常是 Serializable）的規定，若有的話就將 transaction rollback，沒有的話代表這個 transaction 可以安全 commit。 Ｗrite phase 則將稍早暫存的資料寫入到其他 transaction 看得見的地方。

在近幾年有很多不同 OCC 的變種被提出來，因此這篇論文針對 validation 的方法大致將 OCC 分成兩類：

### Local Read Validation

### Global Write Validation

## Adaptive OCC (AOCC)

為了要取兩種 validation 的長處，這篇論文提出了能夠自動在適當時機切換 validation 做法的 OCC，他們稱之為 AOCC。 根據切換 validation 的仔細程度又分成兩種作法：transactional AOCC、query-level AOCC。 這篇其實重點在後者，因為前者其實很直覺。

### Transactional AOCC


### Query-level AOCC


## 實驗

實驗主要使用 YCSB 的 Type E Workloads 與自己修改的 TPC-C 進行。 主要的要點就是這些 benchmarks 都同時有 OLTP 與 OLAP 的特性，以讓

## 參考資料

[1] Guo, Jinwei, et al. "Adaptive optimistic concurrency control for heterogeneous workloads." Proceedings of the VLDB Endowment 12.5 (2019): 584-596.
[2] Kung, Hsiang-Tsung, and John T. Robinson. "On optimistic methods for concurrency control." ACM Transactions on Database Systems (TODS) 6.2 (1981): 213-226.