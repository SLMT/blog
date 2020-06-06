---
title: Jepsen 針對 MongoDB 4.2.6 的調查報告
date: 2020-06-06 16:19:56
categories: [DBMS]
tags: [dbms, concurrency-control, mongodb]
---

Jepsen 最近發表了一篇針對 MongoDB 4.2.6 的測試報告，並提出了許多他們發現的問題。 我覺得報告的內容非常具有參考價值，因此寫一篇簡介介紹報告的主要發現。

## Jepsen 是誰？

Jepsen 是一間專門測試各種分散式系統，以驗證他們是否有達到他們宣稱的 Consistency 或 ACID 目標的公司。 這間公司有為許多著名的分散式系統測試過，例如：Cassandra、CockroachDB、Elasticsearch、Redis... 等等。 他們進行測試的標準非常嚴格，並且會提出公開的測試報告說明他們發現的主要問題等等。

## 報告目標

Jepsen 每篇報告都會有一個要驗證的目標，而這次針對 MongoDB 4.2.6 的目標是他們最新提出的「Full ACID Transactional Support」。

Transaction 在 Relational DBMS 算是標配的東西，意指將一系列的 DB 指令當成單一的實體處理。 ACID 則是 Transaction 概念提供的保證：

- Atomicity: 保證 transaction 的動作要嘛全完成要嘛全部沒完成
- Consistency: 保證資料不違反 DBMS 以及使用者訂下的 constraint
- Isolation: 保證執行中的 transaction 不會互相影響（但已執行完的不管）
- Durability: 保證 committed 的 transaction 執行的效果一定會保存下來

不過對於像是 MongoDB 這樣的 NoSQL 分散式系統來說，一般就不會支援 transaction 了。 因為分散式系統維護 transaction 保證的代價不低，因此許多追求 scalability 的系統都決定放棄這樣的保證。 因此 MongoDB 提出會支援 transaction 算是很不得了的成就。

## Data Loss by Default

事實上，這篇報告並非 Jepsen 第一篇針對 MongoDB 的測試報告。 到目前為止，不含這個報告在內，他們已經發表了五篇報告了。 他們之前就發現 MongoDB 存在一些問題。

這篇報告一開宗就先回顧一下之前發現最大的問題：「在 MongoDB 上 committed 的指令預設情況下可能會因為 node failure 遺失資料。」

一般來說，一個分散式資料庫系統會確保你的資料複製到其他機器上之後，才會回報使用者「Committed (已提交)」。 然而，Jepsen 發現 MongoDB 的預設設定是「Read Concern: Local、Write Concern: 1」。 意思是說，資料讀取時只檢查本地端的資料，只要本地端是 committed 就可以讀取，寫入時只要一台電腦回報 ok，就向使用者回報完成。 因此只要那台回報的電腦 GG，資料可能就跟著 loss 了。

這也是為什麼 DB 江湖上流傳著這句話：

![Tweet](1_tweet.png)

不過這個問題可以藉由將 MongoDB 的設計改成「Read concern: linearizable, Write concern: majority」解決。 因此嚴格來說也不算 bug。 只是預設的設定顯然是為了效能而犧牲 consistency 與 durability 而已。

## 真的是「Full ACID」嗎？

接下來，Jepsen 提出了 MongoDB 廣告詞的問題。 MongoDB 宣稱他們系統具有「Full ACID Support」，這並不是真的。 

熟悉 Transaction 機制的人我想應該都了解，如果要宣稱具有 Full ACID 的話，那們系統一定要能夠保證最嚴格的 Serializable Isolation。 換句話說，必須要讓多個 transaction 執行完的效果像是讓 transaction 依照某種順序依序執行的效果才行。

然而，MongoDB 實際上只支援到 Snapshot Isolation，意指他只能保證讀到的資料是來自某個資料庫快照的結果而已。 Snapshot Isolation 實際上會有所謂的 [write skew anomaly 問題](https://en.wikipedia.org/wiki/Snapshot_isolation#Definition)，進而形成沒有 Serializable 的結果。 因此 MongoDB 不應該宣稱他們有支援 Full ACID。

Jepsen 也做了許多試驗，發現 transaction 執行的結果之間具有大小不等的 dependency cycle。 例如下面這張圖：

![Cycle 1](2_cycle1.png)

代表他們發現兩個 transaction 之間互有依賴關係，這是一種常見用來判斷系統是否為 Serializable 的做法。

另外他們也發現許多像是下圖這種更大的 cycle：

![Cycle 2](3_cycle2.png)

這些都證明了 MongoDB 並不支援 Serializable Isolation，因此不能說有 Full ACID support。

## 系統出錯了... 但還是先 commit 再說？

除了上述的問題外，Jepsen 常常發現他們會觸發一些系統錯誤。 而以一般的思考來說，系統若發生錯誤，執行中的 transaction 應該要 abort 才對。 然而，Jepsen 發現 MongoDB 並不一定會 abort 執行中的 transaction，甚至可能會 commit 這些未完成的 transaction。 雖然這個問題很嚴重，但更嚴重的點在於，MongoDB 並沒有在文件中仔細說明某些錯誤發生時的可能原因與對應措施，導致開發者可能對系統會有錯誤的期待。

## MongoDB 發明了時光機？

Jepsen 在某個測試中發現，在 read concern 為 snapshot 以及 write concern 為 majority 的情況時，其中一個執行的 transaction 竟然會看到未來才寫入的結果！

大致的情況是，該 transaction 會先讀取一個 array，然後在該 array 中放入 1 這個值。 結果竟然該 transaction 在讀取該 array 時發現裡面已經有 1 這個值了！ 因為只有該 transaction 會在該 array 寫入這個值，而且讀取時根本就還沒對系統下達寫入的指令。 因此這個結果宛如 MongoDB 利用時光機先到未來，偷看到未來要寫入的值了！ 連 Jepsen 都忍不住調侃了一下。

他們最後研究了一下猜測大概是該 transaction 的動作實際上被執行了兩次。 可能第一次執行時失敗，在執行第二次時沒有清除第一次的結果，所造成的現象。（但是這其實不該發生，因為沒完成的 transaction 應該 rollback 來確保沒有任何未完成的動作結果保留在系統。）

## 其他問題

這邊在簡單列出一些他們發現的問題

- 重複效果
    - 在發生 network partition 時，可能會出現某些 transaction 的效果被執行兩遍的情況
- 其他種類的 dependency cycles。 細節就不提了，有興趣的話可以讀報告的說明。

## 總結

從報告的這些發現，可以了解 MongoDB 並不具有官方宣稱的 Full ACID support。 除此之外，也具有許多 bug，像是可能讀取到未來寫入的值等問題。 但這些問題主要是反應在使用 transaction 的情境。 如果並不打算用 MongoDB 進行 transaction 的操作的話，應該就不太需要在意這些問題。 但仍須謹記 MongoDB 的預設設定並無法確保資料在分散式環境下一定會安全地保存至其他備份站點。

## 參考資料

- JEPSEN - MongoDB 4.2.6
    - http://jepsen.io/analyses/mongodb-4.2.6
