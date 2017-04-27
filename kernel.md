### 4/27 xv6分科会 資料3

#### カーネル
/kernel 全部を読む。

#### 仮想メモリ
kern/entry.S
```
obj/kern/kernel:     ファイル形式 elf32-i386

セクション:
索引名          サイズ      VMA       LMA       File off  Algn
  0 .text         00001967  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       0000072c  f0101980  00101980  00002980  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         00003961  f01020ac  001020ac  000030ac  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      000018e8  f0105a0d  00105a0d  00006a0d  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         0000a300  f0108000  00108000  00009000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  5 .bss          00000644  f0112300  00112300  00013300  2**5
                  ALLOC
  6 .comment      0000002b  00000000  00000000  00013300  2**0
                  CONTENTS, READONLY
```
kernelはELFですが、VMA(Virtual Memory Adress)とLMA(Load Memory Adress)が違うことがわかります。
bootloaderはkernelのエントリーポイント(LMA)にcallしているので、kernelは0x00100000にロードされて実行されています。
ここでは、kernelが仮想メモリを初期化し、自分自身を仮想メモリの0xf0100000にロードするまでを見ます。

```
      movw    $0x1234,0x472           # warm boot
```
BIOS Data Areaのアドレス0x472に0x1234を書き込むことによってwarm boot(メモリチャックを行わない)を設定している。[1]

```
      # Load the physical address of entry_pgdir into cr3.  entry_pgdir
      # is defined in entrypgdir.c.
      movl    $(RELOC(entry_pgdir)), %eax
      movl    %eax, %cr3
      # Turn on paging.
      movl    %cr0, %eax
      orl $(CR0_PE|CR0_PG|CR0_WP), %eax
      movl    %eax, %cr0
```
cr3にページテーブルのアドレスを入れ、cr0のPG(paging bit.3bit目)を1にすることによって仮想メモリを有効にする。
cr3の説明については[2]より
```
Used when virtual addressing is enabled, hence when the PG bit is set in CR0. CR3 enables the processor to translate linear addresses into physical addresses by locating the page directory and page tables for the current task. Typically, the upper 20 bits of CR3 become the page directory base register (PDBR), which stores the physical address of the first page directory entry.
```
仮想化はOSが頑張っていると思われがちだが、実はページテーブルを作ってCPUが提供するレジスタにセットしているだけである。

```
      # Now paging is enabled, but we're still running at a low EIP
      # (why is this okay?).  Jump up above KERNBASE before entering
      # C code.
      mov $relocated, %eax                                                                                                         
      jmp *%eax
```
jmp *%eaxから仮想メモリで実行が始まる。
```
      # Clear the frame pointer register (EBP)
      # so that once we get into debugging C code,
      # stack backtraces will be terminated properly.
      movl    $0x0,%ebp           # nuke frame pointer
      
      # Set the stack pointer
      movl    $(bootstacktop),%esp
      
      # now to C code
      call    i386_init
```
ebpに0x0を入れている。gdbでbacktraceした時に止まるのはこのため？
$(bootstacktop)というのはentry.Sの一番下のアドレスである。
init.cのi386_initにジャンプする。


#### コンソールに文字を表示する
kern/printf.c, lib/printfmt.c, kern/console.cを読んで理解する。

#### スタックの初期化
スタックはCTFでみんな詳しそう。

### Reference
[1] http://caspar.hazymoon.jp/OpenBSD/arch/i386/i386/locore.html
[2] https://en.wikipedia.org/wiki/Control_register#CR3
