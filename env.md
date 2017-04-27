### 4/27 xv6分科会 資料0 環境構築

```
mkdir xv6
sudo install libsdl1.2-dev libtool-bin libglib2.0-dev libz-dev libpixman-1-dev build-essential gdb gcc-multilib
cd xv6
git clone http://web.mit.edu/ccutler/www/qemu.git -b 6.828-2.3.0
cd qemu
./configure --disable-kvm --prefix=/usr/local --target-list="i386-softmmu x86_64-softmmu"
make
sudo make install
cd ..
git clone https://pdos.csail.mit.edu/6.828/2016/jos.git lab
cd lab
make
```


