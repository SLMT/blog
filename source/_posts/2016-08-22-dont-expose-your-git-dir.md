---
title: 漏洞筆記 - 別讓你的 .git 資料夾公開在網路上啊！
date: 2016-08-22 00:00:45
categories: [Hacking, Exploit]
tags: [web, exploit, hacking, git]
---

這個周末玩了一個 CTF 線上賽。其中有一題是說有一個新手架了一個網站，然後分別使用了 Nginx、PHP、git 這些技術。點開題目提供的網址後，就看到一個 Hello World 的網頁。打開原始碼後，只看到幾行簡單的基礎程式碼，也沒有看到可疑的東西。正當我打算要放棄的時候，突然想到題目有特別提到他有使用 git。於是就嘗試了打開該網站的 `.git` 資料夾，不過馬上就被 403 Forbidden 打臉。

就在我快想不到到底怎麼辦的時候，突然又靈機一動。想到雖然我看不到 `.git` 的內容，但或許我仍可以下載裡面的檔案？於是我馬上隨便打開我手邊的一個 git repository。發現 `.git` 資料夾內一定有 `HEAD` 這個檔案。於是我馬上就嘗試下載 `.git/HEAD`。果不其然！可以抓到這個檔案！那麼接下來就有個方向了！

<!--more-->

## `.git` 資料夾

稍微研究了一下，一個 `.git` 的資料夾大概有 `hooks`、`info`、`logs`、`objects`、`refs`，檔案則大概有 `COMMIT_EDITMSG`、`config`、`description`、`HEAD`。

其中 `hooks`、`info` 這兩個資料夾在大多數的 `.git` 之中都沒有甚麼有用的資訊，`logs` 則是其中可以馬上看出最多資訊的資料夾。`objects` 則存著所有版本的 binary 檔，裡面的東西乍看之下沒有甚麼規則，於是我先跳過。`refs` 則存放每個 branch 目前指向的 commit 為何。

除了 `objects` 以外，所有檔案的檔名大多可以猜到，因此我就將這些檔案抓了下來。可惜裡面最多只能從 log 中看出曾經有 commit 過含有 flag 的檔案。但是檔案的實際內容就無從得知了。


## Git Objects

我發現我似乎最後仍得要還原出每個版本，才有辦法解出這題。因此我開始研究了 `objects` 內的內容。後來稍微思考了一下，覺得 git 官方應該會有相關的文件。Google 一下很快就有發現了！果然有關於內部運作邏輯的文件，甚至連 objects 的存放方式跟意義都有明確說明。甚至還有繁體中文版！文件有興趣的可以參考看看：

[https://git-scm.com/book/zh-tw/v1/Git-%E5%85%A7%E9%83%A8%E5%8E%9F%E7%90%86](https://git-scm.com/book/zh-tw/v1/Git-%E5%85%A7%E9%83%A8%E5%8E%9F%E7%90%86)

大略看過之後，才知道原來所有的 commit 的文件都會獨立存成一個檔案。每個檔案會先計算 SHA1 值為多少，然後取前兩碼做為 `objects` 內的資料夾名稱，後 38 碼則做為 `objects` 內部的檔名存放。在 `objects` 中，除了存放 commit 的檔案外，還會存放 commit 時的目錄結構 (稱為 tree)，以及 commit 的紀錄。這些也同樣會用上述的方式存起來。目錄會紀錄裡面包含的其他目錄以及檔案的 SHA1 值為何。Commit 紀錄則會有該次 commit 對應的目錄 SHA1 值，以及上一次 commit 的 SHA1 值。

這些檔案都被以 binary 的方式存起來。我稍微查了一下有沒有簡便的方法可以直接看到內容。果不其然，可以透過 `git cat-file` 指令解碼 objects 的內容：

```
git cat-file -p [SHA1]
```

`[SHA1]` 處要填上你想要解碼的 object 檔的 SHA1 值，有興趣的人可以自己試試看。記住 SHA1 前兩碼是資料夾名稱，後 38 碼是檔案名稱。


## 檢查指令

在得知這些資訊後，我馬上去查看 log 檔，找到 commit 的 SHA1 值，然後嘗試去下載對應的 object 檔。一試之後，馬上就成功下載對應的檔案。我接著立刻用查到的指令解碼，就看到了類似下列的內容：(這個內容是我拿我的部落格 git 資料夾 demo)

```
tree 9aa33d38927cdde7b969586a2a7e942d816ea947
parent bfd06e73e40a78720418795cacc3072bad168d85
author slmt <sam123456777@gmail.com> 1471021607 +0800
committer slmt <sam123456777@gmail.com> 1471021607 +0800

Add a new post
```

`tree` 指的就是該次 commit 對應的目錄檔，`parent` 指的是上一次 commit 的 SHA1 值。其他則是容易辨別的 commit 資料。

我馬上就知道我又獲得了兩個 SHA1，所以就繼續使用這些 SHA1 下載對應的 object 檔。只是多試幾次之後，很快就感到厭煩了。因為你要先解碼，還要看哪一個是沒有抓過的。全部抓下來的話很花時間。

幸好我後來又發現另一個指令：


```
git fsck
```

這個指令會檢查你的 objects 檔之間的 link 是否完整，並且會找出應該存在但是找不到的 object 檔。這個指令大大地幫助我減少檢查檔案的時間。於是我很快就把遺失的 object 檔補齊。找到所有 object 檔後，很輕易地就使用常用的 git 指令找出了之前的版本，然後發現題目要求的 flag。順利解決。


## 下載 Script

寫完這題之後，我覺得雖然有 `git fsck` 幫助，但是一一下載檔案仍然非常麻煩。因此我就寫了一個 python script 來幫助我進行自動下載。該 script 可以在這裡找到：

[https://github.com/SLMT/ctf-tools/tree/master/git-repository-downloader](https://github.com/SLMT/ctf-tools/tree/master/git-repository-downloader)

基本上 script 就是做我前列提到的那些事情，只是變成只要輸入一個指令就可以完成所有動作。


## 呼籲

最後呼籲一下。別忘記這題會被解開是因為 `.git` 的下載權限被設為公開。若之後大家在開發網站，別忘記把 `.git` 的下載權限關掉。只有關掉閱讀目錄的權限還遠遠不夠，只要使用我的 script 就可以完整地復原整個 git repository XD 因此真的要很小心~~!!!
