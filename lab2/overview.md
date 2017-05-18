

# 全体的にpmapの中にどのようなコードがあるか？


## ページディレクトリ

* 現在のディレクトリのPAはCR0に格納
* 
## ページアドレス



## 差分

```
 GNUmakefile     |   1 +
 conf/lab.mk     |   4 +-
 grade-lab2      |  28 ++++++
 inc/memlayout.h |  41 ++++++++
 kern/init.c     |  17 +---
 kern/kclock.c   |  22 +++++
 kern/kclock.h   |  29 ++++++
 kern/pmap.c     | 839 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
 kern/pmap.h     |  87 ++++++++++++++++
```

基本的に、大きく変わったことは、kern/pmap.cにありそう。なので何が追加されているのかをざっくり見て行く


## グローバル変数

```
extern char bootstacktop[], bootstack[];
extern struct PageInfo *pages;
extern size_t npages;
extern pde_t *kern_pgdir;
```

スタック関連と、page管理

## テスト関連

* check\_page\_free\_list
* check\_kern\_pgdir
* check\_page
* check\_page\_installed\_pgdir	

結構長いので、このファイル自体を肥大化させているが、逆に言えば必要な部分はもっとコンパクトにまとまっている(実質的に400行くらいまでが本質）


## PageInfo構造体

inc/memlayout.hにある。PageInfoはリンクリストなので次のPageInfoの情報を当然持つが、それに加えて、pp\_refというuint16\_t型のデータを持つ。これは参照カウントである。

PageInfoはphysical addressそのものではなく、page2pa(kern/pmap.h)によってphysical addressに変換される


## PDXマクロ

inc/mmu.hに定義

```
#define PDX(la)		((((uintptr_t) (la)) >> PDXSHIFT) & 0x3FF)
```
page directory indexの参照。

[参考](http://softwaretechnique.jp/OS_Development/kernel_development08.html)
![PDE PDX](http://softwaretechnique.jp/OS_Development/Image/Kernel_Development08/virtual_to_physical.png)

![Pointer Layout](https://pdos.csail.mit.edu/6.828/2016/readings/i386/fig5-8.gif)



要するに先頭10ビットの取り出し。

## 実装するもの

### void	mem_init(void);

今回のmain関数的なもの

### void	page_init(void);

pageを解放済みのやつがリンクリストで持つので、その初期化

### struct PageInfo *page_alloc(int alloc_flags);

フリーの連結リストから一つページをとってきて返す

### void	page_free(struct PageInfo *pp);

逆に、ppを連結リストにフリーリストにくっつける


### pte_t *pgdir_walk(pde_t *pgdir, const void *va, int create)

メモリのディレクトリpgdir
仮想アドレス va
が与えられるので、ここから、pageテーブルのpaを指し示す部分をとってくる


### struct PageInfo *page_lookup(pde_t *pgdir, void *va, pte_t **pte_store);

dirとあとvaが与えられるので、ここからpageをとってくる

### void	page_remove(pde_t *pgdir, void *va);


pageをテーブルから削除

### int	page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm);

vaから参照される（pgdir_walkによって）ページのテーブルに、ppを登録する（たぶん）




```

```

### boot_alloc

サイズnに対して、それを超える大きさでpagesize alingnedな領域を確保する

### mem_init

kern_pgdirなどの領域の確保とパーミッションの設定。


