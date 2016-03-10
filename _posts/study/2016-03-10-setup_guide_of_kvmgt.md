---
layout: page
subheadline: 如何配置KVMGT(Intel GVT-g)？
title:  "KVMGT配置教程"
teaser: "本文讲述了我是如何配置KVMGT的"
meta_teaser: "How to set up kvmgt."
breadcrumb: true
categories:
    - study
tags:
    - virtualization
header:
    image_fullwidth: "header_study.jpg"
    
author: inkpool
---

GVT-g作为Intel的GPU虚拟化解决方案中的重头角色，势必是要吸引千万目光的（广告打的有点厉害）。与此同时，KVM现在已经占了虚拟化方案的90%以上份额（别的还有Xen、vMWare、Hyper-V），就连我一个还算不上Xen开发者的搬砖工人都觉得，要改投门派了。Intel的GVT-g一开始的时候是在Xen上实现的，也就是XenGT。XenGT现在已经很稳定了，支持从Sandy Bridge到Haswell再到Broadwell再到Skylake的Intel Graphics HD。由于KVM的开源和巨大潜力，在XENGT之后，Intel也在把GVT-g的实现往KVM上搬，也就是KVMGT。KVMGT和XENGT的主逻辑和原理相同，两者共享90%的代码。现如今官方已经把XENGT和KVMGT整合到了一起，在GitHub上的项目叫[iGTV-g](https://github.com/01org/Igvtg-kernel)，原来的[KVMGT](https://github.com/01org/KVMGT-kernel)项目代码已经停止更新。

本文主要讲解GVT-g针对KVM的配置，主要分为四个步骤：编译内核、编译Qemu、配置宿主机环境、 部署客户机。

## 1 编译内核  

### 1.1 环境准备  
你需要一台装有Intel GPU的电脑，具体来讲，你的电脑需要拥有Sandy Bridge、HASWELL、Broad Well或者Sky Lake中任意一代集成有Intel Graphics HD的CPU。我个人用的是Haswell，也是目前市面上最主流的平台。除了CPU之外，别的并没有特殊要求，内存和硬盘自然是越大/越快越好。另外，装了独立显卡的同学先把独立显卡拆了...  

根据XenGT的经验，我个人推荐使用Ubuntu系统来使用KVMGT，特别推荐使用12.04和14.04，本文用的是14.04.1。  

### 1.2 安装依赖  
安装编译和以后使用过程中所需要的依赖：

~~~
# apt-get install libarchive-dev libghc-bzlib-dev zlib1g-dev mercurial gettext bcc iasl uuid-dev libncurses5-dev kpartx libegl1-mesa-dev libudev-dev libperl-dev libgtk2.0-dev libc6-dev-i386 libaio-dev  libsdl1.2-dev  nfs-common libyajl-dev libx11-dev autoconf libtool xsltproc bison flex xutils-dev x11proto-gl-dev libx11-xcb-dev libxcb-glx0 libxcb-glx0-dev libxcb-dri2-0-dev libxcb-xfixes0-dev bridge-utils python-dev bin86 git vim libssl-dev libpci-dev tightvncserver ssh texinfo mesa-utils ocaml-findlib liblcms-utils vim-addon-manager metacity nautilus openssh-server cgvg socat uml-utilities -y
~~~

### 1.3 编译安装内核  

在`/etc/initramfs-tools/modules`里加上两行：

~~~
xengt
kvm
~~~

然后照下面的步骤编译： 

~~~
# git clone https://github.com/01org/igvtg-kernel kernel_src
# cd kernel_src/
# git checkout 2015q3-3.18.0
# cp config-3.18.0-host .config
# make -j8 && make modules_install
# mkinitramfs -o /boot/initrd.img -v 3.18.0-rc7-vgt-2015q3+
# cp arch/x86/boot/bzImage /boot/vmlinuz-3.18.0
# cp vgt.rules /etc/udev/rules.d
# chmod a+x vgt_mgr
# cp vgt_mgr /usr/bin
~~~

这里的`3.18.0-rc7-vgt-2015q3+`与`/lib/modules/`目录下面的模块版本号匹配。  

## 编译Qemu  

~~~
# git clone https://github.com/01org/igvtg-qemu -b kvmgt_public2015q3 qemu_src
# cd qemu_src/
# git submodule update --init dtc
# git submodule update --init roms/seabios
# ./configure --prefix=/usr --enable-kvm --disable-xen 
--enable-debug-info --enable-debug --enable-sdl --enable-vhost-net 
--disable-debug-tcg --target-list=x86_64-softmmu
# make -j8
# cd roms/seabios
# make -j8
# cd -
# make install
# cp ./roms/seabios/out/bios.bin /usr/bin/bios.bin
~~~

## 配置宿主机环境  

### 创建客户机网络脚本  

创建`/etc/qemu-ifup`文件，并写入以下内容： 

~~~
#!/bin/sh
set -x
switch=$(brctl show| sed -n 2p |awk '{print $1}')
if [ -n "$1" ];then
  tunctl -u `whoami` -t $1
  ip link set $1 up
  sleep 0.5s
  brctl addif $switch $1
  exit 0
else
  echo "Error: no interface specified"
  exit 1
fi
~~~

设置该文件的权限为755，即可运行： 

~~~
chomd 755 /etc/qemu-ifup
~~~

这个文件在启动虚拟机的时候会用到，这里我们先把它准备好。

### 添加grub项 

打开`/etc/grub.d/40_custom`文件，将以下内容复制到`exec tail -n +3 $0`这句的下面：

~~~
menuentry 'Ubuntu-kvmgt' --class kvmgt --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-21427147-a567-47e7-8ba7-3bd8d4c5bc75' {
  recordfail
  load_video
  gfxmode $linux_gfx_mode
  insmod gzio
  insmod part_msdos
  insmod ext2
  set root='hd0,msdos1'
  if [ x$feature_platform_search_hint = xy ]; then
  search --no-floppy --fs-uuid --set=root --hint-bios=hd0,msdos1 --hint-efi=hd0,msdos1 --hint-baremetal=ahci0,msdos1 11427147-a567-47e7-8ba7-3bd8d4c5bc75
  else
  search --no-floppy --fs-uuid --set=root 11427147-a567-47e7-8ba7-3bd8d4c5bc75
  fi
  linux /boot/vmlinuz-3.14.1-igvt+ root=UUID=11427147-a567-47e7-8ba7-3bd8d4c5bc75 ro intel_iommu=igfx_off i915.hvm_boot_foreground=1 log_buf_len=128M
  initrd /boot/initrd-3.14.1-igvt+.img
}
~~~

请参照`/boot/grub/grub.cfg`里面你已有的grub项，然后修改上40_custom刚刚复制进去的内容。`11427147-a567-47e7-8ba7-3bd8d4c5bc75`改成你自己的UUID，你自己的UUID从`grub.cfg`里面找。`root='hd0,msdos1'`这个如果有需要的话，根据你自己的情况改，具体改成什么样子参照你的grub.cfg。

接下来修改grub的选项菜单时间，打开`/etc/default/grub`，将`GRUB_HIDDEN_TIMEOUT= 0`改成你想要grub选单停留的时长，这里我改为5。

在添加过grub选项，修改过grub选单停留时间后，更新grub。

~~~
#update-grub
~~~

此时再打开`/boot/grub/grub.cfg`，可以看到刚刚我们的添加和修改都已经在里面了。（如果没有，请检查前面的配置）

注：这一步在官方的文档里是直接在grub.cfg里新加项的，不过个人不建议这么做。因为系统更新的时候，会自动刷新grub.cfg的，从而导致我们添加的项失效。

OK，重新启动系统，然后选择我们新加的Grub启动项`Ubuntu-kvmgt`，成功进入系统就算成功了。

## 部署客户机

### 创建桥接网络

在宿主机创建一个桥接网络，用来给宿主机分配ip。

~~~
#brctl addbr br0 
#brctl addif br0 eth0 
#ifconfig eth0 0 
#dhclient br0
~~~

这里的eth0要根据具体情况来看，有些人可能是eth1等。

### 创建客户机镜像

首先我们还是要像普通的KVM虚拟机一样，先为虚拟机创建一个镜像，然后安装一个操作系统。这里与宿主机一致，我选择的是Ubuntu 14.04。如果你有一个纯净的Ubuntu 14.04的img你也可以拿来直接用。这一步我们在这里就省略了，可以参考一般KVM的客户机安装步骤。我的客户机镜像是ubuntu-14.04.img。

注意：这一步的客户机镜像还没有KVMGT的功能，所以安装系统以及启动的过程中，Qemu都使用KVM的正常参数。例如：

~~~
/usr/bin/qemu-system-x86_64 -m 2048 -smp 2 -hda /path/to/ubuntu-14.04.img -cdrom /path/to/ubuntu-14.04-install.iso
~~~

### 更新客户机内核

要使用KVMGT我们需要更新客户机系统的内核，用我们在步骤1给宿主机编译的内核来替换客户机的内核。我们把img挂载到/mnt下面，然后替换其内核。

~~~
# modprobe loop
# kpartx -a -v /path/to/ubuntu-14.04.img
~~~

会出现类似这样的输出：

~~~
add map loop0p1 (253:0): 0 29638656 linear /dev/loop0 2048
add map loop0p2 (253:1): 0 1075202 linear /dev/loop0 29642750
add map loop0p5 : 0 1075200 linear 253:1 2
~~~

接着，把/loop0p1挂载到/mnt下面，然后把步骤1中新的内核拷贝进虚拟机：

~~~
# mount /dev/mapper/loop0p1 /mnt/

# cp /boot/vmlinuz-3.18.0 /mnt/boot/
# cp /boot/initrd.img /mnt/boot/
# cp -r /lib/modules/3.18.0* /mnt/lib/modules

# umount /mnt
# kpartx -d -v /path/to/ubuntu-14.04.img
~~~

卸载掉img之后，接着给guest的grub.cfg也添加一个entry，参照第3.2步。

### 启动虚拟机

当我们替换好客户机的内核，并且为客户机添加过新的grub entry之后，就可以开启KVMGT的功能，然后启动虚拟机了。使用如下Qemu参数启动虚拟机：

~~~
# /usr/bin/qemu-system-x86_64 -m 2048 -smp 2 -enable-kvm -M pc -machine kernel_irqchip=on -bios /usr/bin/bios.bin -hda /path/to/Guest_OS.img -net nic -net tap,script=/etc/qemu-ifup -vgt -vga vgt -vgt_low_gm_sz 128  -vgt_high_gm_sz 384 -vgt_fence_sz 4
~~~

注意：`-net tap,script=/etc/qemu-ifup`是我们在步骤3.1创建的脚本。`-vgt -vga vgt -vgt_low_gm_sz 128  -vgt_high_gm_sz 384 -vgt_fence_sz 4`这些参数都是KVMGT带进来的新参数，Low_gm_sz推荐128MB，High_gm_sz推荐384MB。

Low_gm和High_gm这两个选项的最大值，具体所用的显卡和主板的限制有关。HASWELL处理器最多支持512MB的low_gm和1.5GB的high_gm，但是由于主板bios的限制，一般情况下low_gm只有256MB。同时，为了保证虚拟机的正常运行，一般我们给的配置是（128 low_gm + 384 high_gm）。
因此，KVMGT受限于Graphics Memory Space的大小（XENGT也是），HASWELL平台目前只支持4台虚拟机（包括Dom0在内）。

如果客户机成功启动，并且自动全屏显示，那么就说明成功啦！

要在虚拟机之间切换，可以使用下面的命令：

~~~
# echo 0 > /sys/kernel/vgt/control/foreground_vm
~~~

这里的0是虚拟机的vm_id，0表示dom0也就是宿主机，往后的编号就是客户机。同样，你可以通过cat命令查看当前是哪个机器在前台（全屏占据着Display）。

~~~
# cat /sys/kernel/vgt/control/foreground_vm
~~~



