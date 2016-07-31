---
title: PTT Source Code 研究筆記 1
date: 2016-07-31 21:15:04
categories: [Research, PTT, Source Code]
tags: [code, tracing, ptt, bbs]
---

之前一直有打算要研究 PTT 的程式碼。剛好近期也在學習 Rust 語言，因此起了想要使用 Rust 重寫 PTT 的想法。Rust 具有的許多特性我認為都很適合用來寫 PTT。因此開始進行了「用 Rust 重寫 PTT 計畫」。詳細的程式碼會一一更新在 Github 上的這個 [repository][1]。

另外，撰寫之前也需要先了解 PTT 的程式碼在做些甚麼，所以我必須要先讀懂原本的程式碼是如何運作的。我會將閱讀程式碼的筆記紀錄在我所 fork 的 PTT repository 之中的 [tracing branch][2] 內。

<!--more-->

到現在為止，我已經閱讀了 [PTT Source Code][3] 之中，最主要的 `mbbsd.c` 這個檔案大多數的內容。`mbbsd.c` 這個檔案具有 `main` 函式，是程式的進入點，因此從這裡看起。

光是看到我目前為止的內容，就覺得當時寫這個 BBS 的人真的非常厲害。因為以當時的時代背景來說，幾乎沒有甚麼 library 可以使用。需要甚麼輔助功能就要自己寫。我看到了他們自己撰寫了顯示的功能，自己實作了 telnet protocol 等等。然後對系統，特別是 linux，內各個 API 都要瞭若指掌，才能掌握 BBS 站需要的功能。從這些就可以看出要搞出 BBS 站是多麼的不容易。

之後這些文章會紀錄我在閱讀程式碼時，注意到的一些我覺得值得紀錄的事情。

## Shared Memory & Multi-processes

第一個引起我注意的是，PTT 使用了 multi-processes 的方式來撰寫，而不是 multi-threading。因為我只有在修 Operating System 的時候使用過 `fork()`(產生 child process 的函式)，在這之後都是寫 multi-threading 的程式。因此我對 PTT 選擇了 fork 的做法很感興趣。

使用 fork 其實有不少好處，像是產生出來的 process 與原本的 process 是兩個獨立的個體。以資訊安全上的觀點來說，避免使用 shared memory 可以減少不同用戶之間的資料被竊取的可能性。不過，PTT 內其實大量使用了 shared memory (至少我目前看起來是這樣)，因此我覺得應該不是因為資訊安全的原因而做。看起來比較可能是因為 fork-based 的寫法比較容易撰寫與維護。另外，早期 thread 的 API 似乎還沒有一個穩定的標準，所以開啟一個 thread 也許也沒有現在這麼容易。

直覺上若使用了 multi-processes 的寫法，應該會使用 message passing 的方式來互相傳遞資料。不過 PTT 採用了跨 process 之間的 shared memory 來共享資訊。這也讓我挺訝異的，因為這也跟我的直覺不太符合。這邊我比較感興趣的是，不知道他們是怎麼處理不同 process 之間同時使用相同資料的問題，之後看到這部分會再另外撰寫筆記。

## 偵測 Loading

每當有一個使用者連上 PTT 伺服器，PTT 的程式碼就會做一個系統負載檢查。若偵測到負載過高，就會送出「系統過載」的訊息。這邊我學到了如何檢查系統的負載量。

PTT 檢查的方式主要是透過 `getLoadavg()` 以及查看 `/proc/loadavg` 檔案這兩種方式來進行。這邊在檢查之前會先看是哪種系統，如果是 FreeBSD 的話，就會使用前者。若是 Linux 的話，就會使用後者。有趣的是，若兩者都不是，就會跟你說「不知道」XD

根據這份[官方文件][4]，只要將一個陣列塞給 `getLoadavg()`，他就會將過去 1, 5, 15 分鐘的系統平均 loading 放到陣列之中。

`/proc/loadavg` 則是一份 Linux 系統中的文件，每過一段時間會被更新一次。文件中的內容大概長這樣：

```
0.20 0.18 0.12 1/80 11206
```

前三個數字代表最近 1, 5, 10 分鐘的平均 loading，第四個數字代表正在執行與所有 processes 的數目，第五個代表目前最大的 process id。

目前 PTT 的程式碼只看最近 1 分鐘內的 loading，並且根據 `config.h` 之中 `MAX_CPULOAD` 的設定，來看目前系統是否過載。程式碼之中預設是超過 70% 就算是過載，不過實際上的設定也有可能會被改為其他數值。

[1]: https://github.com/SLMT/rust-ptt
[2]: https://github.com/SLMT/pttbbs/tree/tracing
[3]: https://github.com/ptt/pttbbs
[4]: http://linux.die.net/man/3/getloadavg
