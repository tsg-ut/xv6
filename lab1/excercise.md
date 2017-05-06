### 1
boot/boot.S,boot/main.cを読む。
* どのインストラクションから32bitで実行を始めているか？
```
   57   .code32                     # Assemble for 32-bit mode
   58 protcseg:
   59   # Set up the protected-mode data segment registers
   60   movw    $PROT_MODE_DSEG, %ax    # Our data segment selector
```
リアルモードからプロテクトモードに移行するためにセグメント機構を有効にしています。
* ブートローダーの最後のインストラクション、カーネルの最初のインストラクションはそれぞれ何か？
```
ブートローダーの最後 boot/main.c
   58     // call the entry point from the ELF header
   59     // note: does not return!
   60     ((void (*)(void)) (ELFHDR->e_entry))();
```
```
カーネルの最初 kern/entry.S
   42 .globl entry
   43 entry:
   44     movw    $0x1234,0x472           # warm boot
```
* kernelの最初に実行されるアドレスはどこか？
要は上のmovw    $0x1234,0x472が実行されるアドレスということです。
objdump -d obj/kern/kernelを見てみると以下のようになってます
```
f010000c <entry>:
f010000c:       66 c7 05 72 04 00 00    movw   $0x1234,0x472
```
しかしf010000cが答えではないです。これはVMAで、まだページングの機能を有効にしていないのでboot/main.cでブートローダーがカーネルをロードした0010000cが正解です。
実際にgdbで見てみます。
```
(gdb) x/10i 0x0010000c
=> 0x10000c:    movw   $0x1234,0x472
   0x100015:    mov    $0x110000,%eax
```

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

```
Breakpoint 1, 0x00100025 in ?? ()
(gdb) si
=> 0x100028:    mov    $0xf010002f,%eax
0x00100028 in ?? ()
(gdb) 
=> 0x10002d:    jmp    *%eax
0x0010002d in ?? ()
(gdb) 
=> 0xf010002f <relocated>:  mov    $0x0,%ebp
relocated () at kern/entry.S:74
74      movl    $0x0,%ebp           # nuke frame pointer
(gdb) 
=> 0xf0100034 <relocated+5>:    mov    $0xf0110000,%esp
relocated () at kern/entry.S:77
77      movl    $(bootstacktop),%esp
(gdb) 
=> 0xf0100039 <relocated+10>:   call   0xf010009d <i386_init>
80      call    i386_init
(gdb) 
```

* bootloaderの中でLMAがboot/Makefragで0x7c00と適切に設定されていないとおかしな挙動をする箇所はどこか？
```
boot/boot.S
    53   # Jump to next instruction, but in 32-bit code segment.
    54   # Switches processor into 32-bit mode.
    55   ljmp    $PROT_MODE_CSEG, $protcseg
```
$protcsegが明後日の値にリンクされてしまうので明後日のところに飛んでしまいます。実際にgdbでみてみましょう。
boot/Makeflagで0x7c00を適当な値に変えます。0x0000とかにしました。
```
[   0:7c2d] => 0x7c2d:	mov    %eax,%cr0
0x00007c2d in ?? ()
(gdb) 
[   0:7c30] => 0x7c30:	ljmp   $0x8,$0x99ce
0x00007c30 in ?? ()
(gdb) 
[   0:7c30] => 0x7c30:	ljmp   $0x8,$0x99ce

EAX=00000011 EBX=00000000 ECX=00000000 EDX=00000080
ESI=00000000 EDI=00000000 EBP=00000000 ESP=00006f20
EIP=00007c30 EFL=00000006 [-----P-] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
CS =0000 00000000 0000ffff 00009b00 DPL=0 CS16 [-RA]
SS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
DS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
FS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
GS =0000 00000000 0000ffff 00009300 DPL=0 DS16 [-WA]
LDT=0000 00000000 0000ffff 00008200 DPL=0 LDT
TR =0000 00000000 0000ffff 00008b00 DPL=0 TSS32-busy
GDT=     00000000 00000000
IDT=     00000000 000003ff
CR0=00000011 CR2=00000000 CR3=00000000 CR4=00000000
DR0=00000000 DR1=00000000 DR2=00000000 DR3=00000000 
DR6=ffff0ff0 DR7=00000400
EFER=0000000000000000
Triple fault.  Halting for inspection via QEMU monitor.
```
ちゃんとエラーが出ました。


* gdbでメモリの0x00100000を見て、BIOSからbootloader,bootloaderからkernelへとそれぞれ処理が移るときにどのような挙動をするか確認せよ。
見てみましょう。
```
BIOS
(gdb) x/10w 0x00100000
0x100000:   0x00000000  0x00000000  0x00000000  0x00000000
0x100010:   0x00000000  0x00000000  0x00000000  0x00000000
0x100020:   0x00000000  0x00000000
```
```
bootloader
(gdb) x/10w 0x00100000
0x100000:   0x00000000  0x00000000  0x00000000  0x00000000
0x100010:   0x00000000  0x00000000  0x00000000  0x00000000
0x100020:   0x00000000  0x00000000
```
```
(gdb) 
=> 0x7cd7:	repnz insl (%dx),%es:(%edi)
0x00007cd7 in ?? ()
(gdb) x/10w 0x00100000 
0x100000:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0x100010:	0x34000004	0x0000b812	0x220f0011	0xc0200fd8
0x100020:	0x0100010d	0x00000000
```
これがどこで書き込まれているかというと
```
obj/boot/boot.asm
393     7d61:   e8 76 ff ff ff          call   7cdc <readseg>
```
でreadsegを呼びその中で
```
boot/main.c
  123     insl(0x1F0, dst, SECTSIZE/4);
```
このようにメモリに書き込んでいます。
カーネルをメモリにロードした後に、カーネルのエントリーポイントにジャンプしています。

* kernelをgdbでステップ実行し、movl %eax, %cr0 命令の前後に0x00100000と0xf0100000がどのような状態になっているか確認せよ。仮想メモリが起動した直後に実行される一番初めの命令はなにか？kern/entry.Sでmovl %eax, %cr0をコメントアウトし確認せよ。
見てみましょう
```
(gdb) si
=> 0x100025:	mov    %eax,%cr0
0x00100025 in ?? ()
(gdb) x/10w 0xf0100000
0xf0100000 <_start+4026531828>:	0x00000000	0x00000000	0x00000000	0x00000000
0xf0100010 <entry+4>:	0x00000000	0x00000000	0x00000000	0x00000000
0xf0100020 <entry+20>:	0x00000000	0x00000000
(gdb) x/10w 0x00100000
0x100000:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0x100010:	0x34000004	0x0000b812	0x220f0011	0xc0200fd8
0x100020:	0x0100010d	0xc0220f80
(gdb) si
=> 0x100028:	mov    $0xf010002f,%eax
0x00100028 in ?? ()
(gdb) x/10w 0x00100000
0x100000:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0x100010:	0x34000004	0x0000b812	0x220f0011	0xc0200fd8
0x100020:	0x0100010d	0xc0220f80
(gdb) x/10w 0xf0100000
0xf0100000 <_start+4026531828>:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0xf0100010 <entry+4>:	0x34000004	0x0000b812	0x220f0011	0xc0200fd8
0xf0100020 <entry+20>:	0x0100010d	0xc0220f80
```
0xf0100000を見た時にちゃんとデータが入っていますね！これがページングの恩恵です。
仮想メモリが起動した直後に実行されるはじめの命令はこれです。
```
=> 0xf010002f <relocated>:	mov    $0x0,%ebp
relocated () at kern/entry.S:74
74		movl	$0x0,%ebp			# nuke frame pointer
```
実際にkern/entry.Sでmovl %eax, %cr0をコメントアウトして確認してみましょう。もしこれが本当に一番最初ならページングが有効にならないと実行されないはずです。
```
(gdb) 
=> 0x10002a:	jmp    *%eax
0x0010002a in ?? ()
(gdb) 
=> 0xf010002c <relocated>:	add    %al,(%eax)
relocated () at kern/entry.S:74
74		movl	$0x0,%ebp			# nuke frame pointer
(gdb) 
Remote connection closed

EAX=f010002c EBX=00010094 ECX=00000000 EDX=0000009d
ESI=00010094 EDI=00000000 EBP=00007bf8 ESP=00007bec
EIP=f010002c EFL=00000086 [--S--P-] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
CS =0008 00000000 ffffffff 00cf9a00 DPL=0 CS32 [-R-]
SS =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
DS =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
FS =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
GS =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
LDT=0000 00000000 0000ffff 00008200 DPL=0 LDT
TR =0000 00000000 0000ffff 00008b00 DPL=0 TSS32-busy
GDT=     00007c4c 00000017
IDT=     00000000 000003ff
CR0=00000011 CR2=00000000 CR3=00110000 CR4=00000000
DR0=00000000 DR1=00000000 DR2=00000000 DR3=00000000 
DR6=ffff0ff0 DR7=00000400
CCS=00000084 CCD=80010011 CCO=EFLAGS  
EFER=0000000000000000
FCW=037f FSW=0000 [ST=0] FTW=00 MXCSR=00001f80
FPR0=0000000000000000 0000 FPR1=0000000000000000 0000
FPR2=0000000000000000 0000 FPR3=0000000000000000 0000
FPR4=0000000000000000 0000 FPR5=0000000000000000 0000
FPR6=0000000000000000 0000 FPR7=0000000000000000 0000
XMM00=00000000000000000000000000000000 XMM01=00000000000000000000000000000000
XMM02=00000000000000000000000000000000 XMM03=00000000000000000000000000000000
XMM04=00000000000000000000000000000000 XMM05=00000000000000000000000000000000
XMM06=00000000000000000000000000000000 XMM07=00000000000000000000000000000000
GNUmakefile:170: recipe for target 'qemu-nox-gdb' failed
make: *** [qemu-nox-gdb] Aborted (core dumped)
```
実際に落ちました！

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

