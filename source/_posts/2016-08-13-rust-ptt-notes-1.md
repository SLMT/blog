---
title: Rust-PTT 開發筆記 1
date: 2016-08-13 00:34:51
categories: [Project, Rust-PTT]
tags: [code, rust, ptt, bbs, project]
---

很早以前我就想深入研究 PTT 的程式碼，雖然 PTT 很早就將程式碼開源出來，但是當時我並沒有能力看懂。最近又將程式碼翻出來看，看了一些之後有些心得。剛好在 PTT 上有人在[討論重寫 PTT](https://github.com/ptt/pttbbs/issues/5) 的問題，因此就讓我起了重寫 PTT 的意思。

這裡我選擇使用 Rust 做為重寫的語言。主要是因為我最近在研究這個程式語言，但是一直沒有甚麼機會寫出比較大的專案。因此我認為用 Rust 寫 PTT 剛好是個不錯的機會，於是這個專案就這麼開始了！

從這裡可以到 Rust-PTT 的 Github 頁面：[https://github.com/SLMT/rust-ptt/](https://github.com/SLMT/rust-ptt/)

<!--more-->

## 為什麼選擇 Rust

[Rust](https://www.rust-lang.org/en-US/) 是很新的程式語言，去年 (2015) 5 月 16 號才釋出 1.0 正式版。最近也才剛進到 1.10.0 而已。很多東西都還在討論之中，但是大多數的東西應該都差不多定型了。

Rust 語言具有不會有 Segmentation Falut、不需自己管理記憶體 (但是也不用 Garbage Collection 等方式降低執行效能)、近似於 C 效能等特性。同時該語言融合了許多近期新興語言的特色，像是 Pattern Matching、Closures、Generics 等等，並且在於 Concurrency 的方面具有許多 Native Support。我認為 Rust 很適合用來開發大量用戶同時連線的 PTT，而且相較於 C 更容易維護。

Rust 語言目前由 Mozilla 公司開發維護，並且正以 Rust 與開發 Firefox 的經驗，重新撰寫[新的瀏覽器核心](https://github.com/servo/servo)。現在也有其它專案正在重新以 Rust 替換掉原本以 C 撰寫的程式。

## 近期進度

撰寫這篇文章的同時，我已經 commit 了 8 個版本。目前最新的版本為 [a63e830](https://github.com/SLMT/rust-ptt/commit/a63e83012f479ceea13b01fcdd037ae56ce857f5)。

一開始只有 Hello World，八個版本後已經有一個基本的 TCP Server，並且會對 Telnet Protocol 做一些初始化的動作。若一個 telnet client 嘗試連上目前的 server，在進行 telnet 初始化之後，server 會知道 client 需要多大的 terminal size。

目前 telnet 的實作主要是根據 PTT 內的實作方式進行，詳情可以參考我撰寫的另一篇文章：[PTT Source Code 研究筆記 2](http://www.slmt.tw/2016/08/07/tracing-ptt-note-2/)。

## 嘗試看看

假設你的電腦已經安裝好 Rust 與 Git，你可透過在 shell 輸入下列指令啟動 rust-ptt 的 server：

```
> git clone https://github.com/SLMT/rust-ptt
> cd rust-ptt
> cargo run
```

之後啟動都只要執行最後一個指令即可。

然後隨便找一個拿來連 PTT 的 client 或者 telnet client 來連上 server (網址：`localhost:54321`)，此時 server 這邊應該會顯示：

```
Start a connection to 127.0.0.1:4229
Width: 80, Height: 24
```

第一行是 client 的資訊，第二行是 client 要求的 terminal 大小。目前只能顯示這些東西而已，client 則還不會看到任何結果。

## 近期目標

接下來應該會朝向完成基本的 terminal 功能，以及登入功能為主。
