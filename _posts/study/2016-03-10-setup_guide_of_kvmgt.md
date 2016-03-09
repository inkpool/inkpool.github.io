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

1. 编译内核  

    1.1 环境准备  
    你需要一台装有Intel GPU的电脑，具体来讲，你的电脑需要拥有Sandy Bridge、HASWELL、Broad Well或者Sky Lake中任意一代集成有Intel Graphics HD的CPU。我个人用的是Haswell，也是目前市面上最主流的平台。除了CPU之外，别的并没有特殊要求，内存和硬盘自然是越大/越快越好。另外，装了独立显卡的同学先把独立显卡拆了...  

    根据XenGT的经验，我个人推荐使用Ubuntu系统来使用KVMGT，特别推荐使用12.04和14.04，本文用的是14.04.1。  

    1.2 安装依赖  
    安装编译和以后使用过程中所需要的依赖：

    ~~~
    # apt-get install libarchive-dev libghc-bzlib-dev zlib1g-dev mercurial
     gettext bcc iasl uuid-dev libncurses5-dev kpartx libegl1-mesa-dev 
     libudev-dev libperl-dev libgtk2.0-dev libc6-dev-i386 libaio-dev 
     libsdl1.2-dev  nfs-common libyajl-dev libx11-dev autoconf libtool 
     xsltproc bison flex xutils-dev x11proto-gl-dev libx11-xcb-dev 
     libxcb-glx0 libxcb-glx0-dev libxcb-dri2-0-dev libxcb-xfixes0-dev 
     bridge-utils python-dev bin86 git vim libssl-dev libpci-dev 
     tightvncserver ssh texinfo mesa-utils ocaml-findlib liblcms-utils 
     vim-addon-manager metacity nautilus openssh-server cgvg socat 
     uml-utilities -y
    ~~~

    1.3 编译安装内核  

    在`/etc/initramfs-tools/modules`里加上两行：

    ```
    xengt`  
    kvm
    ```

    然后照下面的步骤编译： 

```
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
```

    这里的3.18.0-rc7-vgt-2015q3+与/lib/modules/目录下面的模块版本号匹配。  

2. 编译Qemu  

~~~
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

3. 配置宿主机环境  

3.1. 配置客户机网络脚本  

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

3.2. 添加grub项 
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


