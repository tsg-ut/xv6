# -*- mode: ruby -*-
# vi: set ft=ruby :
$script = <<SCRIPT
apt-get -y update
apt-get -y install libsdl1.2-dev libtool-bin libglib2.0-dev libz-dev libpixman-1-dev build-essential gdb gcc-multilib git
cd /home/ubuntu/xv6
git clone http://web.mit.edu/ccutler/www/qemu.git -b 6.828-2.3.0
cd qemu
./configure --disable-kvm --prefix=/usr/local --target-list="i386-softmmu x86_64-softmmu"
make
make install
cd ..
git clone https://pdos.csail.mit.edu/6.828/2016/jos.git lab
cd lab
make
SCRIPT

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial32"
  config.ssh.forward_agent = true

  config.vm.synced_folder "xv6", "/home/ubuntu/xv6"
  config.vm.network :forwarded_port, host: 3000, guest: 3000
  config.vm.provision "shell", inline: $script

  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--memory", "2048"]

    # trueなら、GUI 環境を別途整備が必要
    vb.gui = false
  end
end
