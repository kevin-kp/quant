Vagrant.configure("2") do |config|

  # OS to use for the VM
  config.vm.box = "ubuntu/artful64"

  # config.vm.network "private_network", type: "dhcp"

  config.vm.network :forwarded_port, guest: 4433, host: 44433, protocol: "udp"

  config.vm.provision "file", source: "/etc/letsencrypt/live/slate.eggert.org",
    destination: "~/slate.eggert.org"

  # hardware configuration of the VM
  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.memory = "1024"
    vb.cpus = 2
    vb.linked_clone = true
    vb.name = config.vm.hostname

    # use virtio for uplink, in case there is an issue with netmap's e1000
    vb.customize ["modifyvm", :id, "--nictype1", "virtio"]

    # better clock synchronization (to within 100ms)
    vb.customize [ "guestproperty", "set", :id,
      "/VirtualBox/GuestAdd/VBoxService/--timesync-set-threshold", 100 ]
  end

  # apply some fixes to the VM OS, update it, and install some tools
  config.vm.provision "shell", inline: <<-SHELL
    export DEBIAN_FRONTEND=noninteractive

    # resize disk
    resize2fs /dev/sda1

    git config --global user.email lars@netapp.com
    git config --global user.name "Lars Eggert"

    # update the box
    apt-get -y update
    apt-get -y upgrade

    # install some tools that are needed
    apt-get -y install git cmake ninja-build libev-dev libssl-dev g++ \
      libhttp-parser-dev libbsd-dev pkg-config mercurial

    # install some tools that are useful
    apt-get -y install tmux fish gdb htop silversearcher-ag valgrind

    # change shell to fish
    chsh -s /usr/bin/fish ubuntu

    # get Linux kernel sources, for building netmap
    apt-get source linux-image-$(uname -r)

    # compile and install netmap
    git clone https://github.com/luigirizzo/netmap
    cd /home/vagrant/netmap
    ./configure --driver-suffix=-netmap --no-ext-drivers \
      --kernel-sources=/home/ubuntu/linux-$(uname -r | cut -d- -f1)
    make
    make install

    # enable netmap at boot, and make sure the netmap e1000 driver is used
    echo 'netmap' >> /etc/modules-load.d/modules.conf
    echo 'e1000-netmap' >> /etc/modules-load.d/modules.conf
    echo 'blacklist e1000' >> /etc/modprobe.d/blacklist-netmap.conf
    echo 'blacklist virtio' >> /etc/modprobe.d/blacklist-netmap.conf

    # configure netmap
    echo 'echo 1 > /sys/module/netmap/parameters/admode' >> /etc/rc.local
    echo 'echo 1000000 > /sys/module/netmap/parameters/buf_num' >> /etc/rc.local

    # various changes to /etc to let normal users use netmap
    echo 'KERNEL=="netmap", MODE="0666"' > /etc/udev/rules.d/netmap.rules
    echo '*   soft  memlock   unlimited' >> /etc/security/limits.conf
    echo '*   hard  memlock   unlimited' >> /etc/security/limits.conf
    echo '*   soft  core      unlimited' >> /etc/security/limits.conf
    echo '*   hard  core      unlimited' >> /etc/security/limits.conf

    # build a new initrd
    depmod -a
    update-initramfs -u

    echo Please restart via \"vagrant up\"
    shutdown -P now
  SHELL
end
