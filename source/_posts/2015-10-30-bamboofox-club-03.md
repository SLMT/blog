---
title: "BambooFox 第三堂社課心得"
date: 2015-10-30
categories: [CTF, BambooFox]
tags: [bamboofox, ctf, club]
---

這次的課程接續上次組合語言的內容，主要在談 x86 架構下的組合語言，以及常見的 Buffer Overflow 漏洞。

<!--more-->

## 課程內容分享

### 組合語言

一開始先回顧一下 x86 下的架構，包含 registers 與一些 conditional flags 等等。 接著複習了一些 x86 架構下常見的組合語言指令。 這邊有提到 Intel 與 AT&T 兩種 style 不同的寫法，並介紹一些之間的差異等等。

最重要的資訊，我想應該就是了解如何做 system call 吧！

System call 是一種讓你程式跟作業系統 (Opearating System, OS) 溝通的管道。 舉凡要進行任何需要 OS 幫忙的工作，都要透過 system call。 而在組合語言之中，想要進行 system call，就要使用 interrept 的指令來暫時中斷程式，並轉交權限給 OS。 大概進行的 pattern 就如同下面所示：

```asm
mov		eax, xxx	; 想要 OS 執行的 function 代碼
mov		ebx, xxx	; function 的第一個參數
mov		ecx, xxx	; function 的第二個參數
mov		edx, xxx	; function 的第三個參數
...

int		0x80		; 進行 system call
```

一開始要先將一些參數存入 registers，`eax` 要放你想要 OS 進行的 system call function 代碼。 例如你想要系統讀檔，就要放 3 (sys_read)。 然後，其他 registers，像是 `ebx`, `ecx`, `edx`, `esx`, `edi` 則是要放呼叫該動作的參數。 例如剛剛說的讀檔，就必須要分別在 `ebx`、`ecx` 與 `edx` 放入 file descriptor、buffer pointer 與想要讀取的大小。 最後再執行 `int 0x80` 指令來進行 system call。

詳細每個 register 在進行 system call 所需要放的參數可以在 [這個表][1] 找到。

我想學這個的目的，除了是為了要能夠對程式做逆向工程之外，也是為了要在之後 inject (注入) shellcode 而使用。 甚至是做更進一步的攻擊。 算是還挺實用的。


### Buffer Overflow

Buffer overflow 是 C/C++ 常見的漏洞。 只要你的程式之中使用到了陣列或指標，並配合呼叫了不安全的 function，就有可能有潛在的問題。

例如下面這段 code：

```c++
char input[50];

scanf("%s", input);
```

這段看起來極其平淡，且在一開始程設課就學過的 code，潛藏著極大的危機存在。

`scanf` 不會檢查使用者輸入的長度，也不會知道 `input` 這個陣列有多大，除非遇到特定的字元，不然他就會持續性地一直讀取輸入值。 而當使用者輸入的長度超過了 `input` 的空間，也就是在上例中超過 49 個字元 (最後一個字是 `\0`)，就會發生 overflow 的情況。 一般來說 overflow 只會讓程式出現 `segmentation fault` 而已，可是有心人士可以利用這個來控制程式的運作方向。

區域變數通常是放在 stack 中，stack 除了變數之外，還會放著各式各樣其他的資訊。 像是 function arguments，stack frame pointer，還有最重要的 saved eip。 EIP 是 x86 系統下一個特殊的 register，這個 register 不能直接寫入，只能透過特殊指令來進行修改。 EIP 其重要之處，就是在他負責告訴 CPU，下一個要執行的指令在哪裡。

因此掌控了 EIP，就掌控了整個程式走向。

想想前面提到的 buffer overflow，到底多出的部分跑哪去了？基本上就是跑到外面的地方，覆蓋掉 stack 上的其他資訊。

剛剛提到了 stack 有放 saved eip。 當程式呼叫 function 的時候，為了要能夠在 function 結束之後，跳回到原本的位置，會將回去預計繼續執行的程式碼位址暫存在 stack 中，這就是 saved eip。

利用 buffer overflow，你就會有機會覆寫掉 saved eip 的數值。 只要控制得好，就能將你想要讓程式執行的程式碼位址覆蓋掉 saved eip。 當程式 return 的時候，就會拿出你預先設計的數值，並跳轉到你希望的位置繼續執行程式。 這個時候就可以做很多有趣的事情，例如讓程式執行你預先塞好的程式碼，然後呼叫 shell 等等。

這也是到現今為止還是常常能看見的大洞。 不過編譯器其實也有在進步，為了這防止駭客攻擊這個部分，使用了各式各樣的技巧來保護。 當然駭客也不是笨蛋，也想出了很多方法繞過這些機制。 到現在還沒有一定的勝負。


## 練習題的收穫

今天主要的練習是 [這題][2] ，重點在於 buffer overflow 的觀念。 讓正確的 buffer overflow，找到如何覆蓋 saved eip，找到程式應該要執行到哪，以及要怎麼避免程式搗亂的行為。

這題今天學到最有用的知識有兩點：

### 1. `scanf` 終止條件

這題有個部分會檢查你輸入 buffer 的長度，檢查的方式是利用 `strlen`，並且會對你的 buffer 做攪亂的動作。 這邊我們可以讓程式只攪亂部分的 buffer，或甚至完全繞過。 重點在於讓程式以為 buffer 很短，但是其實比他知道的大很多。

但是要怎麼做呢？ `strlen` 的檢查中止條件是看到 `\0` 就停下來。 換句話說，`\0` 之前的內容都會被當作字串的一部分，之後的東西就不管了。 這邊我一開始有想到在 buffer 內插入 `\0` 來防止程式攪亂我的 payload。 但是我馬上想到了，`scanf` 應該會把 `\0` 也當作終止條件，所以我想這招應該沒用。

可是在卡了許久之後，經過前輩提示，看了看提示的投影片，才知道原來 `scanf` 的中止條件更狹隘。 他們只會在遇到空白或是換行字元才停，`\0` 也會被當成字串的一部分吃進來。 這還真是大洞啊！ 我完全意想不到這種我以為該有的條件竟然沒有！ 看來 C\C++ 比我想像的更危機四伏。

### 2. `cat -`

這個也很有趣。 他們的程式是放在他們專有的機器上執行，而我們必須要透過網路連線才有辦法跟程式溝通。 通常會使用 `nc` (netcat) 進行。 當攻擊者取得 shell 之後，就可以拿到機器上的 flag。 不過雖然我將 payload 送過去之後，也成功呼叫出 shell 了。 可是遇到了一個很大的問題，就是我無法跟 shell 進行互動！ 因為我使用了 `cat payload | nc [IP] [Port]` 的方式來將預先寫好的 input 傳過去，但是也封閉了我使用 `nc` 進行溝通的道路。

正當我在想是否要自己寫 python code 來互動的時候，發現了一個更簡單的方式！

就是使用 `cat -`！ `cat` 指令會印出檔案的內容，後面接著檔案名稱。 那 `-` 又是啥檔案？

`-` 代表的就是 standard input！ 也就是標準輸入 (通常是鍵盤)！

如果你有自己的 unix 環境可以試試看，使用 `cat -` ，並輸入一些字元會發生甚麼事情。 你會發現它會將你輸入的東西直接 output 出來。

這就是我需要的啊！

我只要把上面的指令改成 `cat payload - | nc [IP] [Port]`。 這樣 `cat` 就會先將 payload 傳過去，然後再轉接 standrad input 給它。 這樣我就可以在取得 shell 之後，輸入指令給 `nc` 傳送了！

這個算是我今天覺得最有用的技巧XD


[1]: http://docs.cs.up.ac.za/programming/asm/derick_tut/syscalls.html
[2]: http://train.cs.nctu.edu.tw/problems/1
