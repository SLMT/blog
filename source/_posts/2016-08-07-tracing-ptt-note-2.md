---
title: PTT Source Code 研究筆記 2
date: 2016-08-07 15:15:00
categories: [Research, PTT, Source Code]
tags: [code, tracing, ptt, bbs]
---

基本上，PTT 的使用者連上 PTT 時，都是透過一個叫做 telnet 的 protocol 進行。看了 PTT 的程式碼，會發現 PTT 並沒有使用其它的 telnet 函式庫，而是自行實作 telnet protocol。剛剛將這個部分看的差不多了，因此稍微紀錄一下實作方式。

<!--more-->

## 簡介 Telnet Protocol

Telnet Protocol 的基本概念定義在 RFC 854 中，需要的話可以透過底下的 reference 看到全文。內容我花一點時間大致看過了，其實概念不難理解。

Telnet 是一種建立在 TCP 上的通訊協定。TCP 基本上處理好了傳送訊息過程中的各種錯誤，因此 telnet 很少需要處理甚麼錯誤的情況。Telnet 的基本概念是在模擬過去電腦的工作模式。很早期以前的電腦是有一台大型的主機 (mainframe) 在進行計算，然後使用者必須要透過一台小型的終端機 (terminal) 來連上主機進行工作 (就像下圖一樣)。跟現在這種一台電腦一個螢幕，然後螢幕的顯示直接由主機負責很不一樣。以前的模式裡，終端機很像是一個 client，主機則是 server。終端機會負責接收使用者訊息，透過線路送出指令，並顯示結果。Telnet 大概也類似這樣。Server 上儲存各種資料，每個使用者自己電腦的 telnet client 就是一個終端機。連上 server 後，server 會把你畫面上該顯示的東西傳過來給你，client 則會印出收到的文字。

<img src="https://upload.wikimedia.org/wikipedia/commons/7/7d/IBM_704_mainframe.gif" />

了解基本概念之後，就會知道 telnet 其實該作的事情不多。基本上 server 就是會把一個個你畫面上該顯示的文字傳遞給 client，client 則會下達指令給 server。唯一要特別處理的，就是兩者間必須要能夠透過一些預先定義好的指令，來設定兩者傳遞資料的一些規則。

## PTT's Telnet Implementation

PTT 自行實作了 telnet protocol，主要是由當時在台大資工系的 piaip 實作。程式碼可以在 PTT Source Code 的 `common/sys/telnet.c` 中找到。內容非常簡單，大概一個下午就可以看完。不過也因為實作非常簡單，所以對於很多設定的選項並不會給予回覆。在該檔之中有一段註解寫著：

> We are the boss. We don't respect to client. It's client's responsibility to follow us.

我看到這段霸氣註解之後笑了一下XD

不過也多虧這個決定，這邊就有很多實作的細節都可以跳過。

基本上實作的程式碼大概可以分成三塊：

- 剛開始連線時，會送出一些初始設定資訊
- 讓 server 其他部分調整一些基本設定
- 處理 client 送來的設定

這邊可以發現主要都是在實作關於設定的通訊。其他像是處理 server 送出的畫面內容、如何處理 client 對於 PTT 的指令這些，就不是 telnet protocol 本身的工作。因此並沒有寫在這裡。

### 起始設定

一開始 server 會將一連串的設定訊息送給 client，client 收到後則會對每個訊息做出回應。PTT 送出的設定可以在 `telnet.c` 的 `telnet_init_cmds` 這個變數內找到：

```c
static const char telnet_init_cmds[] = {
	 // retrieve terminal type and throw away.
 	 // why? because without this, clients enter line mode.
	 IAC, DO, TELOPT_TTYPE,
	 IAC, SB, TELOPT_TTYPE, TELQUAL_SEND, IAC, SE,

	 // i'm a smart term with resize ability.
	 IAC, DO, TELOPT_NAWS,

	 // i will echo.
	 IAC, WILL, TELOPT_ECHO,
	 // supress ga.
	 IAC, WILL, TELOPT_SGA,
	 // 8 bit binary.
	 IAC, WILL, TELOPT_BINARY,
	 IAC, DO,   TELOPT_BINARY,
};
```

其中使用到的常數都定義在 `arpa/telnet.h` 中。這個檔案並不是 PTT 的一部分，而是大多數系統都會包含的標頭檔。上網搜尋一下就可以找到檔案。這些初始訊息翻譯成人話的話，意思大概如下 (以下每一行分別對應到上面一行)：

- 現在來設定終端機種類吧 (TELOPT_TTYPE)
- 請給我終端機種類設定資訊
- 現在來設定終端機大小吧 (TELOPT_NAWS)
- 希望你能回應每個我傳送的訊息 (TELOPT_ECHO)
- 希望你能夠直接傳送下個訊息，而不要等我回覆 (TELOPT_SGA)
- 希望你能使用 8bit 傳輸模式 (TELOPT_BINARY)

特別注意註解中有提到，若不送出前兩項設定訊息，client 可能就無法正確地顯示出 PTT 畫面。

### 處理 client 設定訊息

至於 client 這邊送回來的訊息，處理的方式基本上就是一個 finite state machine (FSM)。FSM 的概念就是程式會有一組狀態。每次收到訊號之後，檢查現在的狀態是甚麼，然後做出對應的動作，最後更新目前的狀態。詳細的部分各位有興趣可以自己去看，那段程式碼才大概 224 行，很快就可以看完。

大概需要注意的是，若收到 client 傳出的 `IAC SB` 訊息，那就會進入一個**暫存**狀態。接下來所有收到的訊息都會被放近一個 buffer 之中，直到收到 `SE` 指令為止。這個動作意義在於，`IAC SB` 是代表接下來收到的是關於某個選項的詳細資訊，例如選項若是 `TELOPT_NAWS`，那接下來就會收到終端機的長跟高的資訊。有趣的是，PTT 似乎只會對 `TELOPT_NAWS` 這個選項做出反應，其他的選項則大多都被忽略掉了。

### 實際例子

目前我用 Rust 撰寫的 [PTT prototype](https://github.com/SLMT/rust-ptt/tree/5246412a8ca514823c01e2cc40c4ee06d281e9cf) 已經可以送出前兩章看到的基本設定，以及接收 client 的訊息。

我使用 PCMan 來測試，發現 PCMan 在收到我的訊息後，會回傳下列訊息 (in bytes)：

```
255, 251, 24, 255, 250, 24, 0, 86, 84, 49, 48, 48, 255, 240, 255, 251, 31, 255, 250, 31, 0, 80, 0, 24, 255, 240, 255, 253, 1, 255, 253, 3, 255, 254, 0, 255, 252, 0
```

若轉換為各自代表的意義，則大概是這樣：

```
IAC, WILL, TELOPT_TTYPE
IAC, SB, {TELOPT_TTYPE, TELQUAL_IS, 86, 84, 49, 48, 48, 255}, SE
IAC, WILL, TELOPT_NAWS
IAC, SB, {TELOPT_NAWS, width{0, 80}, height{0, 24}, 255}, SE
IAC, DO, TELOPT_ECHO,
IAC, DO, TELOPT_SGA,
IAC, DONT, TELOPT_BINARY,
IAC, WONT, TELOPT_BINARY
```

這邊可以看到像是對於 `TELOPT_NAWS` 這個選項，PCMan 送出了寬 80、高 24 的訊息。而 PTT 這邊就會依照他的請求來調整大小。其中有一個有趣的點是，PCMan 送出了他不會使用 8bit 模式的訊息。而 PTT 這邊則是看完之後就丟掉了。這感覺就像是以下發生了以下情境：

```
PTT: 請使用 8-bit 模式
PCMan: 我不能使用 8-bit 模式
PTT: 哦，是哦。
```

大概就是這樣吧XD

這邊了解之後，會先花點時間實作在 Rust-PTT 上，之後應該會進入 PTT 上關於 terminal 的實作。

## Reference

- PTT Source Code - [https://github.com/ptt/pttbbs](https://github.com/ptt/pttbbs)
- 我的 PTT Source Code 閱讀紀錄 - [https://github.com/SLMT/pttbbs/tree/tracing](https://github.com/SLMT/pttbbs/tree/tracing)
- Rust PTT Project - [https://github.com/SLMT/rust-ptt](https://github.com/SLMT/rust-ptt)
- PTT Source Code 中的 telnet.c - [https://github.com/ptt/pttbbs/blob/master/common/sys/telnet.c](https://github.com/ptt/pttbbs/blob/master/common/sys/telnet.c)
- RFC 854 - Telnet Protocol Specification - [https://tools.ietf.org/html/rfc854](https://tools.ietf.org/html/rfc854)
- PCMan - [http://pcman.ptt.cc/](http://pcman.ptt.cc/)
