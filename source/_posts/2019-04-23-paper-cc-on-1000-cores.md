---
title: "論文筆記 - Staring into the Abyss: An Evaluation of Concurrency Control with One Thousand Cores"
date: 2019-04-23 18:14:59
categories: [Research, Paper]
tags: [paper, dbms, concurrency-control, survey]
---

近期開啟的新系列，以分享資料庫系統相關的論文並簡介內容為主，不會講得太深入，但是會需要些對資料庫系統內部運作原理的基本概念。

這篇論文 [1] 主要是把近期常被拿來討論幾個主要的 Concurrency Control 的分支，放在具備 1000 個 Core 的環境下進行測試。 目的是為了瞭解這些做法在 Core 數量極多的機器上的 Scalability 如何。 在研究過程中，他們也嘗試去優化這些方法，盡可能地避免實作上可能會造成的瓶頸。 所以這篇論文的研究也可以當作在 multi-thread 環境下，實作這些 CC 作法的參考。

<!--more-->

## 基本資料

- 論文作者群：Xiangyao Yu, George Bezerra, Andrew Pavlo, Srinivas Devadas, Michael Stonebraker
- 研究機構：MIT, CMU
- 發表會議/期刊：Very Large Database (VLDB) 2014 年期刊

## 需要具備的知識

- 知道 Transaciton 是甚麼
- 了解 Concurrency Control 的目的
- 對 multi-threads programming 有基本認識

## 做為測試目標的 Concurrency Control 作法

簡單介紹一下這篇論文所探討的主流 Concurrency Control 作法。

### Two Phase Locking (2PL)

最傳統且常見的 Concurrency Control 作法。 基本概念是在 Transaction (以下簡稱 Tx) 取得資料之前先把資料鎖住，防止其他 Tx 存取。 一般又會依照讀寫的狀況分成 shared lock (slock) 跟 exclusive lock (xlock)。 一旦 Tx 開始放棄 lock 就不能再 lock 資料，以確保資料能夠正確的 recovery。

其中這篇論文又依照處理 deadlock 的方式分成以下三種作法：

#### DL_DETECT

不會預先防範 deadlock 的發生，但是發生的話可以藉由建立一個 wait-for graph 來找出發生的地方，並且會把其中一個卡住的 Tx abort 來解除 deadlock。

#### NO_WAIT

只要 Tx 存取資料的時候，發現可能需要等待其他 Tx 釋放 lock，就會自動 abort。

#### WIAT_DIE

Tx A 存取資料時，如果發現資料被鎖住的話，依照下列情況做處理：

- 如果 Tx A 比鎖住資料的 Tx B 年輕 => abort Tx A
- 如果 Tx A 比鎖住資料的 Tx B 老 => 等待 Tx B 釋放 lock

這個方法確保不會有 deadlock 發生，因為一個 deadlock 互相等待的關係之中，一定有一個 Tx 比較年輕，因此只要讓其中一邊沒機會等待就可以避免 deadlock。

這個方法成本比 DL_DETECT 低很多，又不會亂 abort Tx，所以是最常見的作法。

### Timestamp Ordering (T/O)

Timestamp ordering 的基本概念是 DBMS 會為每個 Tx 配發一個 timestamp (以下簡稱 ts)，這個 ts 會作為判斷 Tx 順序的標準。 除此之外，Tx 一般會將自己的 ts 寫在自己寫入過的資料上面，用來給後面存取的 Tx 辨識。

#### Basic T/O

Tx 會檢查資料上的 ts，若上面的 ts 比自己的 ts 還要年輕，就會 abort。 反之，照常寫入資料。

#### Multi-version Concurrency Control (MVCC)

每次 Tx 寫入資料的時候，直接創造一個新的版本另外儲存，原本的版本就保留不動。 好處是如果有其他人正在讀同樣的資料，就不會被寫入影響到 (reads do not block writes)。

#### Optimistic Concurrency Control (OCC)

Tx 在執行期間不做任何檢查，讀取任何想讀的資料，寫的時候先寫在一個獨立的空間。 最後 Tx 要 commit 時檢查之前的讀寫是否會跟人家衝突 (conflict)。 沒問題的話就把先前寫入的東西寫到共用的空間。 有問題就 abort Tx。 一種先斬後奏的作法。

#### H-STORE

H-Store [2] 這種系統專用的作法。 先將資料切割成多個分區 (partition)，然後每個分區交給一個 core 處理，並且每個分區一次只允許一個 thread 執行。 因此在一個分區內不需要處理 Concurrency Control。 若有 Tx 想要跨分區存取，就必須要先把這些分區全部鎖住。

## Challenges

- 需要實作各種 CC 方法
- 需要盡可能地避免實作上的瓶頸

## 有用的發現

### 主要的瓶頸來源

- Memory Allocation：很多時間會浪費在等 malloc 分配記憶體。解決方法是使用 previous work (TCMalloc [3], jemalloc [4]) 來為每個 thread 建立 memory pool，減少分配的 cost。
- Lock Table: 避免使用 centrolized lock table，而是將 lock 儲存在各個 tuple 中，但是可能需要耗費額外記憶體。
- Mutex: Mutex 是一個很昂貴的動作，造成 CPU 需要做多次 message passing。2PL 主要出現在 Deadlock detection，Timestamp-based 出現在 centrolized timestamp allocator。

### 實作 Scalable 2PL 的技巧

- 實作 lock-free deadlock detection 方法： 每個 Tx 只將等待的目標存在自己的 queue 中，detector 會看過所有 queue 來尋找 partial wait-for graph。
- 讓等太久的 Tx abort： 原本 Tx 等待 lock 時，會等到 lock 被釋放才回復執行。 根據實驗結果顯示，若能適度讓等太久的 Tx abort，可以減少 wait chaining 的狀況，因此反而可以增加 throughput。 實驗結果顯示等待上限設為 100us 差不多。

### 實作 Scalable Timestamp-based CC 的技巧

#### 分配 Timestamp

分配 timestamp 必須是一個 centralized 的架構，以確保 timestamp 的順序性，但是也造成了效能瓶頸。這邊有四種解法：

- 使用 Atomic Addition 指令。
- 使用 Atomic Addition 指令，一次取得大量 timestamps (batching)。
- 讓 timestamp 使用 core 的 local clock 加上 thread id 組成。這個方法需要 core 之間 synchronize clocks，用軟體實作會非常昂貴。目前只有 Intel 提供硬體支援。
- 用 hardware counter 來產生 timestamp，目前尚未有 CPU 提供支援。

Clock 的作法在所有實驗上取得最佳效果，但是在 high-contention 的實驗中，單純使用無 batch 的 atomic addition 效果就跟 clock 一樣好。有鑑於簡單性與支援性，之後都採用 atomic addition 這個做法。

#### 分散式 Validation

類似 Microsoft Hakaton [5] ，在每個 tuple 上 validate tx，而不是用 centralized 的 critical section。

### 實作 H-Store 的技巧

讓 thread 直接 access 其他 thread 的 memory，而不是還要另外送 query 過去讓其他 core 處理。

## 實驗

實驗的部分就不細談，基本上就是使用 YCSB Benchmark 去模擬各種 workloads，並且研究每種作法在不同環境的優缺點，以幫助使用者選擇自己適合用甚麼 Concurrency Control 方法。 實驗的數量很多，因此有興趣的人可以自己細看。

另外提一下，1000 Cores 的機器目前還不存在，所以論文中是使用將多台機器連接在一起，並使用模擬器的方式去模擬的。

## 參考資料

[1] Yu, Xiangyao, et al. "Staring into the abyss: An evaluation of concurrency control with one thousand cores." Proceedings of the VLDB Endowment 8.3 (2014): 209-220.
[2] Kallman, Robert, et al. "H-store: a high-performance, distributed main memory transaction processing system." Proceedings of the VLDB Endowment 1.2 (2008): 1496-1499.
[3] Ghemawat, Sanjay, and Paul Menage. "Tcmalloc: Thread-caching malloc." (2009).
[4] J. Evans. jemalloc.http://canonware.com/jemalloc
[5] Neumann, Thomas, Tobias Mühlbauer, and Alfons Kemper. "Fast serializable multi-version concurrency control for main-memory database systems." Proceedings of the 2015 ACM SIGMOD International Conference on Management of Data. ACM, 2015.