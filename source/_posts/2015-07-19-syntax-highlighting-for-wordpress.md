---
title: "在 WordPress 的文章中使用 Syntax Highlighting"
date: 2015-07-19
categories: [Building Blog, Word Press]
tags: [syntax, highlight, blog, wordpress]
---

這篇文章原本是為了要處理舊部落格之中遇到的問題所寫的，不過現在這個問題在新部落格已經不存在了。因此我就簡短說明一下狀況與解決辦法。

<!--more-->

舊部落格使用的是 WordPress 系統，當我們發表一篇文章的時候，如果想要在其中嵌入一段程式碼，就會在文章前後加入 `<code>...</code>` tags。

不過我後來發現 WordPress 沒有提供 syntax highlighting 的功能，也就是把你的程式碼依照對應的語言以及語法加上容易辨識的顏色。

像是假設我現在想要給大家看一段 java code，那我可能就會這樣打：

```html
<code>
package slmt.test;

public class Cat {
    public static void meow() {
        System.out.println("I'm a cat~ Meow!!");
    }
}
</code>
```

然後顯示出來的效果就會像這樣：

```java
package slmt.test;

public class Cat {
    public static void meow() {
        System.out.println("I'm a cat~ Meow!!");
    }
}
```

但是 WordPress 並沒有提供這樣的功能。

幸好在同實驗室的學長 [Pi 先生][1]的建議下，得知了 [稜鏡.js][2] 這套 library。並且在使用之後，確認 WordPress 也能夠很輕易地套用。

雖然我現在使用的 jekyll 內已經有內建這項功能了，不過還是將這個 library 分享給大家，讓使用 WordPress 的朋友能夠輕鬆幫自己的 code 上色XD

[1]: http://shaokanp.me/
[2]: http://prismjs.com/
