### 1
boot/boot.S,boot/main.cを読む。
* どのインストラクションから32bitで実行を始めているか？
* ブートローダーの最後のインストラクション、カーネルの最後のインストラクションはそれぞれ何か？
* kernelの最初に実行されるアドレスはどこか？

### 2
メモリについて。
* objdumpでkernelとbootloaderのVMA/LMAを確認する。どう違うか？
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
