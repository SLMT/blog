---
title: "BambooFox 第一堂社課心得"
date: 2015-10-02
categories: [CTF, BambooFox]
tags: [bamboofox, ctf, club]
---

今天去參加了交大 [BambooFox][1] 的第一堂社課，收穫不少。因此我想寫下一些參與的心得，並記錄一些學習到的技巧。

<!--more-->

## 前言

其實很早以前我就一直對資訊安全方面的東西有興趣，最早可以追朔到我國中的時候，不過當時並不具備相關的知識。後來到大學之前，也有購買一些相關書籍。可惜大多數的書都只是教如何使用一些滲透軟體，技術面的東西對我來說又太深。最後就沒有繼續研究。

上大學後，開始學習程式語言與各種撰寫技巧，逐漸具備各式各樣的知識。最近想起了當年小時候對這塊的興趣。雖然現在已經是再一年就要脫離學生生活的碩二生，但是還是希望在這最後學期能夠真正地好好學好這些東西。

當然在這之前也並非沒有研究，只是因為無人指導，也不知道該從哪裡開始。直到今年參加了 [HITCON][2] (台灣駭客年會)，才開始對這塊有初步的認識。在會中聽說交大今年要成立相關社團，而清大一直都沒有這種社團，所以我當然就很興奮地過去參加啦！

## 課程內容

今天的課程內容主要涵蓋兩個部分，基本 **Linux Shell 操作** 與 **Python 基礎**。

### Linux Shell 操作

基本 Linux Shell 操作我其實算是挺熟悉了，畢竟平常實驗室的實驗都是在 CentOS 上跑，還寫過一點 bash script。而且網路上的資源也非常豐富，其中繁體中文資源以「[鳥哥的Linux 私房菜][3]」最為有名。不過還是有兩點值得一提的東西。

#### Kali Linux

[Kali Linux][4] 是 Linux 的其中一個分支，主要是基於 Debian 開發的作業系統。這個系統的特點在於，它具有大量的滲透測試工具，並提供了許多支援，因此成為了駭客們愛用的系統。HITCON 會場也可以看到許多人都在使用。

基本上這個系統確實非常方便，甚至對很多人來說都是不可或缺的工具。不過操作上跟一般的 Linux 系統沒甚麼太大的差別，所以也不會太難上手。

如果想要快速了解 Kali Linux 上有什麼工具可以使用的話，我推薦可以參考這個清單：<https://comparite.ch/kalics>。

#### 特殊 Bash 指令

Shell 的使用算是學習 Linux 很重要的基礎，一般的操作很快就可以上手。不過很不錯的是，他們有教一些進階與特殊的用法。舉個例子來說，在 Bash 中使用 `$(...)`，其中包含的指令就是會被執行，而輸出的東西則會變成變數。然後 `$0` 這個變數通常指向 `bash`。因此如果你讓某些程式，執行了以下這個指令：

```
$($0)
```

就會啟動 shell，讓別人可以為所欲為。(有興趣的人可以用 C 的 `system()` 或 python 的 `os.system()` 執行看看)

這個指令的特點在於沒有使用到任何 `bash` 或 `sh` 的字樣就啟動了 shell，這樣就可以越過某些過濾式的檢查。

### Python 基礎

老實說，我對 python 幾乎一竅不通。最近才開始遵循 [Learn Python The Hard Way][5] 的教學開始入門。今天社課教了不少基礎用法，讓大家快速上手。值得提的是，他們有介紹到一個叫做 [Hacker Rank][6] 的網站，可以讓大家練習 Python。

另一點值得講的，他們介紹了一個好用的 Pyhton Module，[Pwntools][7]。這個工具是專門為 CTF (Capture The Flag) 比賽設計的 module，裡面包含了許多好用的 Python function，讓大家可以快速撰寫出針對某漏洞設計的 Python script。今天我也在之後的練習之中體會到了好處。

## 課後練習

他們除了上課之外，還很用心地準備了一個練習題給大家測試。這個練習題主要是來自於 Python 的 [pickle module][8] 的漏洞。以下解釋一下這個漏洞。

首先可以先看這段程式碼，有興趣的人可執行看看：

```python
import pickle, os

class Exploit(object):
	def __reduce__(self):
		comm = "sh"
		return (os.system, (comm,))

a = pickle.dumps(Exploit())
b = pickle.loads(a)
```

執行後會發現這段程式碼會啟動 shell，為什麼呢？

在解釋之前，我們要先了解 pickle 這個 module 是做甚麼的。

Pickle module 主要是用來對物件進行 serialize 與 deserialize 的動作。簡單地說，serialize 就是把物件轉換成另一個儲存形式 (像是字串或其他二進位碼)，deserialize 則是把資料還原成物件。其中，`pickle.dumps()` 就是在 serialize，`pickle.loads()` 就是在 deserialize。

而啟動 shell 的動作就發生在 `b = pickle.loads(a)` 這行，也就是將資料還原成物件的時候。

這主要是因為，`Exploit` 這個 class 中的 method `__reduce__` 回傳了一組 tuple，`(os.system, (comm,))`。

Pickle module 的文件中有寫到，class 可以提供一個稱為 `__reduce__` 的 method 來說明如何 deserialize 物件。這個 `__reduce__` method 必須要傳回一些數值，來指示如何還原物件。第一個數值代表初始化物件的時候必須要呼叫的 function，第二個數值則是該 function 的參數。如果你仔細對照一下上面的程式碼，就會發現第一個回傳值為 `os.system`，第二個則為一個只有包含 `comm` 的 tuple。其結果是，會導致程式在 deserialize 時執行 `os.system("sh")` 這段程式碼，而啟動 shell。

這確實非常有趣XD

後來社團的前輩們就以這題為基礎，在一台 server 上架了一個 service。該 service 包含著這個漏洞，因此可以利用這個漏洞取得該機器上的資訊。

同時他們也鼓勵利用 Python script 來寫這題，讓我好好地把前面學的很多東西都用上了！

## 總結

光是今天的課就感覺收穫不少，而且前輩們在練習期間，也非常友善地不斷關心社員是否遇到甚麼問題。也不會因為我是清大來的就把我晾在一邊XD。這些讓我非常期待接下來的課程。

希望之後上課也能夠像今天這樣寫下心得與筆記。

[1]: https://bamboofox.torchpad.com/
[2]: http://hitcon.org/
[3]: http://linux.vbird.org/
[4]: https://www.kali.org/
[5]: http://learnpythonthehardway.org/
[6]: https://www.hackerrank.com/
[7]: http://pwntools.com/
[8]: https://docs.python.org/2/library/pickle.html
