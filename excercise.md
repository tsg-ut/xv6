### 1
boot/boot.S,boot/main.cを読む。
* どのインストラクションから32bitで実行を始めているか？
* ブートローダーの最後のインストラクション、カーネルの最初のインストラクションはそれぞれ何か？
* kernelの最初に実行されるアドレスはどこか？

### 2
メモリについて。
* objdumpでkernelとbootloaderのVMA/LMAを確認する。どう違うか？
```
.-(~/xv6/lab)-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------(yamaguchi@ispc2016)-
`--> objdump -h obj/kern/kernel

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
.-(~/xv6/lab)-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------(yamaguchi@ispc2016)-
`--> objdump -h obj/boot/boot.out

obj/boot/boot.out:     ファイル形式 elf32-i386

セクション:
索引名          サイズ      VMA       LMA       File off  Algn
  0 .text         0000017c  00007c00  00007c00  00000074  2**2
                  CONTENTS, ALLOC, LOAD, CODE
  1 .eh_frame     000000b0  00007d7c  00007d7c  000001f0  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         000007b0  00000000  00000000  000002a0  2**2
                  CONTENTS, READONLY, DEBUGGING
  3 .stabstr      00000846  00000000  00000000  00000a50  2**0
                  CONTENTS, READONLY, DEBUGGING
  4 .comment      0000002b  00000000  00000000  00001296  2**0
                  CONTENTS, READONLY
```

* bootloaderの中でLMAがboot/Makefragで0x7c00と適切に設定されていないとおかしな挙動をする箇所はどこか？
* gdbでメモリの0x00100000を見て、BIOSからbootloader,bootloaderからkernelへとそれぞれ処理が移るときにどのような挙動をするか確認せよ。
* kernelをgdbでステップ実行し、movl %eax, %cr0 命令の前後に0x00100000と0xf0100000がどのような状態になっているか確認せよ。仮想メモリが起動した直後に実行される一番初めの命令はなにか？kern/entry.Sでmovl %eax, %cr0をコメントアウトし確認せよ。

### 3
kern/printf.c,lib/printfmt.c,kern/console.cを読む。
* 未完成のコード断片がある。見つけ出して完成させよ。

### 4
stack
* kern/monitor.cのmon_backtrace()関数に再帰的にebp,eip,args 5つを出力させるプログラムをかけ。
```
qemu-system-i386 -nographic -drive file=obj/kern/kernel.img,index=0,media=disk,format=raw -serial mon:stdio -gdb tcp::26000 -D qemu.log 
6828 decimal is 15254 octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
ebp f010ff18 eip f0100087 args 00000000 00000000 00000000 00000000 f010099a 
ebp f010ff38 eip f0100069 args 00000000 00000001 f010ff78 00000000 f010099a 
ebp f010ff58 eip f0100069 args 00000001 00000002 f010ff98 00000000 f010099a 
ebp f010ff78 eip f0100069 args 00000002 00000003 f010ffb8 00000000 f010099a 
ebp f010ff98 eip f0100069 args 00000003 00000004 00000000 00000000 00000000 
ebp f010ffb8 eip f0100069 args 00000004 00000005 00000000 00010094 00010094 
ebp f010ffd8 eip f01000ea args 00000005 00001aac 00000644 00000000 00000000 
ebp f010fff8 eip f010003e args 00111021 00000000 00000000 00000000 00000000 
ebp 00000000 eip f000ff53 args f000e2c3 f000ff53 f000ff53 f000ff53 f000ff53 
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K> 

```
このような出力が得られるようにせよ。inc/x86.hのread_ebp()が便利である。

