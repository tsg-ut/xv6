### 4/27 xv6分科会 資料2

#### ブートローダー
boot/boot.S
boot/main.c
を読む。

BIOSによってロード・jmpされたブートローダーの処理は0x7c00から始まる(決まり)。
プロテクトモードへの移行、起動ディスクからkernelを読み込みcallする役割をもつ。

```
make qemu-nox-gdb
make gdb
b *0x7c00
c
```

* boot.S プロテクトモードへの移行
```
    # Switch from real to protected mode, using a bootstrap GDT
    # and segment translation that makes virtual addresses 
    # identical to their physical addresses, so that the 
    # effective memory map does not change during the switch.
    lgdt    gdtdesc
```
プロテクトモードに移行する準備。
lgdtというのはGDTR(Grobal Discripter Table Register)にGDTのサイズとアドレスを設定することで、CPUが読みに行けるようにする。

リアルモードでのアドレスの計算はCS*16 + IPだった。
プロテクトモードでもアドレスを計算するのにCSとIP使うのには変わりがないが、計算方法が違う。
詳しくはリファレンス[1]を見て欲しいが、要はCSの値がGDTにあるアドレスのオフセットになっていて、それとIPを足すとアドレスが得られるということである。


```
    movl    %cr0, %eax
    orl     $CR0_PE_ON, %eax
    movl    %eax, %cr0
    # Jump to next instruction, but in 32-bit code segment.                                                                            
    # Switches processor into 32-bit mode.
    ljmp    $PROT_MODE_CSEG, $protcseg
```
ここで32bitプロテクトモードに移行している。
cr0レジスタ(Control Register)の最下位ビットPEを1にすることによってプロテクトモードへ移行する。その後、32bitで次の命令にジャンプする。
実はBIOSでもこの処理をしている！しかしブートローダーを呼び出す前に16bitリアルモードに戻っている。

```
    # Set up the stack pointer and call into C.
    movl    $start, %esp
    call bootmain
```
ここでmain.cのbootmainという関数を呼んでいる

* main.c
main.cはカーネルをメモリに読み込んで、エントリーポイントにジャンプするということをしている。
bootmain
```
      // read 1st page off disk
      readseg((uint32_t) ELFHDR, SECTSIZE*8, 0); 
  
      // is this a valid ELF?
      if (ELFHDR->e_magic != ELF_MAGIC)
          goto bad;
  
      // load each program segment (ignores ph flags)
      ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
      eph = ph + ELFHDR->e_phnum;
      for (; ph < eph; ph++)
          // p_pa is the load address of this segment (as well
          // as the physical address)
          readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
  
      // call the entry point from the ELF header
      // note: does not return!
      ((void (*)(void)) (ELFHDR->e_entry))();
```
あまりおもしろみはない。ELF形式かどうかをチェックして、ELFだったらjmpしているだけである。

* waitdisk
```
  void
  waitdisk(void)
  {       
      // wait for disk reaady                                                                                                          
      while ((inb(0x1F7) & 0xC0) != 0x40)
          /* do nothing */;
  }

[2]より
Status Register (Address 0x1F7):
7 6 5 4 3 2 1 0
BUSY READY FAULT SEEK DRQ CORR IDDEX ERROR
```
IDE device controllerは0x1F7がStatus registerなのでそれを読んでreadyになるまでループを回している。

* readsect
```
  void
  readsect(void *dst, uint32_t offset)
  {
      // wait for disk to be ready
      waitdisk();
  
      outb(0x1F2, 1);     // count = 1
      outb(0x1F3, offset);
      outb(0x1F4, offset >> 8); 
      outb(0x1F5, offset >> 16);
      outb(0x1F6, (offset >> 24) | 0xE0);
      outb(0x1F7, 0x20);  // cmd 0x20 - read sectors
  
      // wait for disk to be ready
      waitdisk();
  
      // read a sector
      insl(0x1F0, dst, SECTSIZE/4);
  }
```
[2] の9ページを参照するのを強くおすすめする。
0x1F2-0x1F6でデバイスドライバにパラメーターを送り、Comand/statusの0x1F7にreadシグナルを送ることによってreadを開始している。


### Reference
[1] http://softwaretechnique.jp/OS_Development/kernel_loader2.html
[2] http://pages.cs.wisc.edu/~remzi/OSTEP/file-devices.pdf
