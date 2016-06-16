---
title: "新 Blog 開張!!"
date: 2015-07-25
categories: [Building Blog, Jekyll]
tags: [blog, github pages]
---

這篇原先是為了我在 Jekyll 的部落格寫的，對於現在這個 blog 來說已經算是舊文囉！

<!--more-->

這裏是我的新 blog

對於一般來訪的人來說，應該只會覺得外觀有改變而已。但是事實上，連後端也改變很多！之前的 blog 是先在我個人的 VM 上裝上 [Apache HTTP Server][2]，然後再將他人撰寫的 [WordPress][1] 丟在 server 上執行。不過其實我很快就發現這個做法的大問題。我個人可以很簡單地，使用 DoS 攻擊 (Denial of Service Attack)，就能夠讓我的 server 當機。

因此從那時開始，我就一直在尋覓各種解決方法，或是替代方案。

另外，WordPress 提供了蠻多功能，但是我個人其實不太需要這麼多功能。對於一個不會寫網頁的人來說，WordPress 確實是架設網站的一個很好選擇。不過對一個只想架設一個簡單 blog 的軟體工程師來說，WordPress 實在是太肥了點。

同時租用一個 VM 也是要錢的 （每個月 10 鎂，約台幣 300），所以能有一個便宜，甚至不用錢的解決方案最好。


幸好，實驗室學長 [Marcus][3] 告訴我了另一個選擇。他說我可以使用 [Jekyll][4] 在 [Github Pages][5] 上架設 blog。

最初我並沒有很認真地考慮，但是後來仔細一看之後，發現這是一個適合工程師，也很適合我的解決方案。

基本上使用 Jekyll 不需要任何後台管理。撰寫 blog 文章主要是透過撰寫一份 Markdown 文件來進行，然後 Jekyll 會根據你事先寫好的模板將文件轉換成靜態網頁。最後只要將網頁上傳到一個網路空間即可。

剛好 Github 有提供幫你 host 靜態網頁的功能，因此我就使用了 [Github Pages][5] 這個服務。唯一的缺點是，你的所有程式碼都會公開在 Github 上，只是我個人不覺得這是大問題就是了XD


本 blog 原始碼位置： [https://github.com/SLMT/slmt.github.io][6]

[1]: https://tw.wordpress.org/
[2]: http://httpd.apache.org/
[3]: http://shaokanp.me/
[4]: http://jekyllrb.com/
[5]: https://pages.github.com/
[6]: https://github.com/SLMT/slmt.github.io
