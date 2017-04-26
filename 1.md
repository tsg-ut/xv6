### 4/27 xv6分科会 資料0 BIOS

#### アセンブリ
CTFだとintel記法しか読まないけど、最低限知っておきたいのは、
```
mov %ax,%dx # %dx <= %ax
$0x64 # <= 即値の0x64
jnz <= jump not zero
```
Intel系のCPUはポートマップドI/O(メインメモリとは別のアドレス空間としてI/O空間がある)で、周辺機器のレジスタを接続してポートに信号を送ることによって通信をしている。
```
inb $0x64,%al # [1] 参照。CPUの0x64ポートにキーボードコントローラーのステータスレジスタが割り振られている。
outb %al,$0x64 # 0x64ポートにalを送る
```

#### BIOS
```
cd lab
make
make qemu-nox-gdb
(別のターミナルにする)
make gdb
```
MITの講義資料のメモリの図参照。

```
The target architecture is assumed to be i8086
[f000:fff0]    0xffff0: ljmp   $0xf000,$0xe05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
(gdb) si
[f000:e05b]    0xfe05b: cmpl   $0x0,%cs:0x6ac8
```

BIOS ROMがマップされているメモリの一番上 0x000ffff0 から実行が始まる。こう決まっていることにより、PCが起動するたびにBIOSが実行されることが保証される。
[f000:fff0] はCSレジスタがf000でIPがfff0ということ。
リアルモードでは以下のようにアドレスが計算される。
adress = 16 * CS + IP
なのでこの場合だと
adress = 16 * 0xf000 + 0xfff0
    = 0xf0000 + 0xfff0
    = 0xffff0

* BIOSの処理
[2] の図が秀逸。割り込みベクタの初期化と、周辺機器(キーボードなど)の初期化,ブートローダーの起動を行っている。

```
[f000:d167]    0xfd167: out    %al,$0x70
0x0000d167 in ?? ()
(gdb) 
[f000:d169]    0xfd169: in     $0x71,%al
0x0000d169 in ?? ()
```

最初はRAMのdetectをする。
0xfd167:    out    %al,$0x70
I/O 0x70はCMOS Real Time Clock. 起動時にこれを読んで自分の時計を初期化する.

```
0xfd171:   lidtw  %cs:0x6ab8
```
この命令はIDTR(Interrupt Descriptor Table Register)に%cs:0x6ab8というアドレスをセットし、割り込みが発生した時はCPUがIDTRが指すアドレスを参照してくれるようにする。

あとはハードウェアの初期化。
処理によってbootableなハードウェアを見つけたら、0x7c00に512byteのブートローダーを読み込んで、jmpする。
しかしここはqemuのゾーンなのであまり真剣に読まなくて良さそう。

#### Reference
[1] http://caspar.hazymoon.jp/OpenBSD/annex/keyboard.html
[2] http://softwaretechnique.jp/OS_Development/kernel_development02.html
[3] http://wiki.osdev.org/CMOS
