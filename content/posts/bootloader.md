---
title: "x86 Bootloader"
date: 2021-02-18T20:10:16+08:00
draft: true
---

本篇紀錄實作 x86 bootloader 的過程，主要採用 nasm 組合語言進行實作，並在 qemu 上模擬。



## What is Bootloader

電腦開機上電直到啟動作業系統，需要一個引導程式，這個程式稱為 bootloader。Bootloader 又可稱作 boot program 或 bootstrap loader，名稱來自於 "Pull oneself up by one's bootstraps" 這句短語，意思為靠自己振作起來，很有趣吧。電腦上電後，BIOS (在主機板的一小塊韌體) 會呼叫 bootloader，負責檢測硬體，並定位出 kernel image 在哪，載入 kernel 後，接下來就交棒給 kernel 了。Bootloader 通常放在硬碟第一個磁區 (sector)，這塊磁區又稱為主啟動紀錄 (Master Boot Record, MBR)，大小為 512 bytes。



## Print a string 

首先我們使用 BIOS 的中斷服務，嘗試印出一個字串 "Hello"。有幾個地方需要注意：

1. 程式一開始的 org 0x7c00 是告訴 nasm 組譯器，這段程式碼會被載入到記憶體 0x7c00 這個地方。org directive 的使用方式可參考 nasm 手冊 8.1.1 節 [1]。至於為什麼是 0x7c00 這個數字呢？為什麼不是從 0 開始呢？我查了 Intel 的開發者手冊找不到有關 magic number 的介紹，後來找到 msakamoto-sf 在 2010 年發表的部落格文章 [2]，裡面介紹 0x7c00 的由來。這個數字從 Intel 8088 就開始使用，後面處理器為了兼容，也持續採用著。當初設計的原因在於 8088 所搭配的作業系統是 [86-DOS](https://en.wikipedia.org/wiki/86-DOS)，其大小至少需要 32KB (即 0x7fff)。為了保留連續的記憶體給作業系統，設計 MBR 放在 sector 的最後。前面提到 MBR 大小是 512 bytes，bootloader 在引導過程中可能產生額外的資料，因此多留 512 bytes 給它。所以起始位址可以推出

   ```
   0x7fff - 512 - 512 + 1 = 0x7c00
   ```

   題外話，前幾天和我爸聊到目前作業系統的市占率，我們都知道現今 Windows 占了八成，老爸說他們那個年代用的是 DOS 呢！

2. 如何印出字串？BIOS 提供許多中斷服務 [3]，我們可以從 interrupt table 找到 int 0x10 負責 video 相關的服務，將 ah 暫存器填入 0x0e，表示寫入 TTY 模式的字元，al 放想要印的字元。

3. 在 label end 中，有一行指令為 jmp \$，用來產生無窮迴圈。dollar sign ​\$ 還滿常用的，nasm 支援 \$ 和 \$\$ 兩種 token，前者表示當前位址，後者表示此 section 的起始位址。詳細可以參考 nasm 手冊 3.5 節 [4]，裡面有一些 trick。

4. 以上寫完後，保留最後兩個 bytes，標記成 0x55 和 0xaa，當作 MBR 結束記號 [5]，剩下的填 0。至於要填多少 byte 的 0，利用 (\$ - \$\$) 可以知道我們程式碼佔了多大。所以要 padding 的大小自然就是 

   ```
   (512 - 2) - ($ - $$)
   ```

以下是程式碼，命名成 print_str.S。

```
[org 0x7c00]

[bits 16]

print_str:
    mov ah, 0x0e ; tty mode
    mov al, 'H'
    int 0x10

    mov al, 'e'
    int 0x10

    mov al, 'l'
    int 0x10
    int 0x10

    mov al, 'o'
    int 0x10

end:
    jmp $

times 510 - ($ - $$) db 0      ; padding
db 0x55, 0xaa                  ; signature 				
```

編譯成 binary image

```
$ nasm print_str.S -f bin -o boot.bin
```

用 hexdump 確認一下真的是 512 bytes，以 0x55 和 0xaa 結尾，注意是 little-endian，所以會是 0xaa55 

[boot-04]

Kali 上安裝 qemu

```
$ sudo apt-get install qemu-system-x86
```

用 qemu 啟動，可以看到成功印出字串 Hello

```
$ qemu-system-i386 -fda boot.bin
```

[boot-05]



## Stack

在往下一小節討論之前，我們先熟悉 stack 的使用。還記得大二剛開始學組語的時候，並不能體會到 stack 的好處，只覺得 push 和 pop 很 make sense。直到自己動手寫東西的時候，我只能說提出 stack 這個概念的人真是天才啊，畢竟我們的資源 (暫存器) 有限，有了 stack，就可以簡單地重複使用暫存器。題外話，1946 年圖靈是把 push 和 pop 稱為 "bury" 和 "unbury"。這一小節，我們利用 push 和 pop 指令來印出字串 Hello。

```
print_str2.S
```

[boot-05]



## Load data from disk

隨著 kernel 的功能越來越複雜，不太可能只在一個 512 bytes 的磁區就完成，所以 bootloader 有個重要工作是到硬碟把 kernel 載進來。這一小節，我們嘗試如何從硬碟讀資料。



## Switch to protected mode



## Demo



