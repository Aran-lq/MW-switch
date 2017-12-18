## 1.在nvme-qemu上配置Lightnvm
Lightnvm的原作者在文档中给出了两种实现方式，第一种是使用真实设备，比如CNEX公司的的Open-channel SSD，价格昂贵，并且由于国内没有代理商的缘故，现在还很难买到。第二种方式就是本文将要介绍的，使用[QEMU](www.qemu.org)的方式，向主机侧提供一个模拟的支持open channel功能的nvme设备。实际上qemu中仅仅是模拟了NVMe specification 1.2.1的HCI的功能逻辑，并且作者增加了支持lightnvm的扩展，让主机端的驱动程序以为下面是一个真实的设备，这样一种方式就提供了一个快速，便捷地体验Lightnvm的方式 : )  
下面是我的配置过程。

### 配置环境
- host: linux 4.10, ubuntu17.04(amd64) 
- guest: ubuntu 16.04 on nvme-qemu.

### 在主机端安装nvme-qemu
* 克隆增加lightnvm支持的分支，nvme-qemu(https://github.com/OpenChannelSSD/qemu-nvme)  
`$ git clone https://github.com/OpenChannelSSD/qemu-nvme.git `
* 编译QEMU源码，并且安装，需要注意的是 ，qemu需要安装在Linux主机下，如果是虚拟机的话一般不支持kvm，这样QEMU跑起来会非常慢。
	
		$ ./configure --python=/usr/bin/python2 --enable-kvm --target-list=x86_64-softmmu --enable-linux-aio --prefix=(安装的目录) 
		$ make -j4
		$ make install
### 创建一个QEMU虚拟机
####1. 创建一个空白的磁盘文件  
`$ qemu-img create -f qcow2 ubuntu.qcow2 20G`

虚拟机的最大空间为20G，qcow2格式，空间是动态增长。用于跑支持lightnvm的内核
#### 2. 在该磁盘文件中安装一个Linux系统。
`$ qemu-system-x86_64 -m 2G -enable-kvm ubuntu.raw -cdrom ~/ubuntu.iso `  
其中 -m 表示内存为2G, -cdrom会找到用于安装的镜像文件，(我这里是ubuntu16.04)。安装之后，就可以通过下面的命令启动虚拟机了:  
`$ qemu-system-x86_64 -m 4G -enable-kvm ubuntu.raw`
### 在linux主机中编译支持Lightnvm的内核
	$ git clone https://github.com/OpenChannelSSD/linux.git
	$ cd linux
	$ git checkout for-next (other branch is ok, make sure if it work)
	$ make menuconfig  
确保配置项中包含下面的内容：  

	CONFIG_NVM=y
	# Expose the /sys/module/lnvm/parameters/configure_debug interface
	CONFIG_NVM_DEBUG=y
	# Target support (required to expose the open-channel SSD as a block device)
	CONFIG_NVM_PBLK=y    
	# For NVMe support
	CONFIG_BLK_DEV_NVME=y
继续编译：  

	$ make -j4 > /dev/null 
成功后会在 `/arch/x86/boot/`目录下找到内核压缩文件bzImage，大小7M左右。为什么要使用这个文件，是因为QEMU提供了一种直接加载内核镜像文件的功能，这样方便于快速的测试不同内核，也让虚拟机更加轻量。前面自己是直接在虚拟机里面编译内核，安装。可一旦要在其他机器上面配置环境进行实验，得重新再编译内核，拷贝的话体量又太大。现在采用bzImage的方式后，整个过程就轻量了很多。配置新的环境，只需要编译和安装QEMU，再装一个最简单的Linux，将bzImage文件拷贝过去可以了。
### 创建一个模拟NVMe设备的空白文件
在主机端使用下面命令，创建一个空白的设备文件：  
`$ dd if=/dev/zero of=blknvme bs=1M count=1024`
### 启动QEMU虚拟机
通过QEMU启动一个支持Lightnvm的虚拟机，并且挂载了一个模拟的NVMe设备。下面是我的启动命令脚本，需要确保相关文件都在同一个目录下面。
	
	#!/bin/bash
	# Boot vm
	
	sudo qemu-system-x86_64 -m 4G -smp 1 --enable-kvm \
	-drive file=blknvme,if=none,id=mynvme \
	-redir tcp:37812::22 \ (用于主机和虚拟机的ssh通信，端口号为37812。此项也可以不填)
	-device nvme,drive=mynvme,serial=deadbeef,namespaces=1,lver=1,lmetasize=16,ll2pmode=0,nlbaf=5,lba_index=3,mdts=10,lnum_lun=16,lnum_pln=2,lsec_size=4096,lsecs_per_pg=4,lpgs_per_blk=512,lbbtable=bbtable.qemu,lmetadata=meta.qemu,ldebug=1 \
	ubuntu.raw

### 在虚拟机中安装nvme-cli来感受Lightnvm
目前，我们已经有了：  

- 一个支持*Lightnvm*和*NVMe设备*的内核  
- 一个NVMe的设备（由QEMU模拟）  

因此，我们还需要一个能在用户空间管理和控制这个设备的工具。可以使用**nvme-cli**。确保下载的是最新的版本（1.3.44）,支持*nvme lnvm*命令。

	$ git clone https://github.com/linux-nvme/nvme-cli.git
	$ cd nvme-cli
	$ make -j2
	$ make install
到目前为止，已经完成了整个环境的搭建。现在就可以使用下面的命令来检测NVMe设备并且在上面创建一个target。下面是我的输出结果。
	
	$ nvme lnvm list
	Number of devices: 1
	Device      	Block manager	Version
	nvme0n1     	gennvm      	(1,0,0)
	
	$ nvme lnvm info
	LightNVM (1,0,0). 1 target type(s) registered.
	Type	Version
	pblk	(1,0,0)
	
	$ lsblk (list available blk devick)
	NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
	sr0      11:0    1 1024M  0 rom  
	fd0       2:0    1    4K  0 disk 
	sda       8:0    0   50G  0 disk 
	├─sda2    8:2    0    1K  0 part 
	├─sda5    8:5    0    2G  0 part [SWAP]
	└─sda1    8:1    0   48G  0 part /
	nvme0n1 259:0    0 1022M  0 disk (our nvme device)
	
	$ nvme lnvm create -d nvme0n1 -n mydevice -t pblk -b 0 -e 3(create a target on nvme device,name mydevice,lun0-lun4)
	
	$ lsblk 
	NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
	sr0       11:0    1 1024M  0 rom  
	fd0        2:0    1    4K  0 disk 
	mydevice 259:1    0  144M  0 disk (**target created by us**)
	sda        8:0    0   50G  0 disk 
	├─sda2     8:2    0    1K  0 part 
	├─sda5     8:5    0    2G  0 part [SWAP]
	└─sda1     8:1    0   48G  0 part /
	nvme0n1  259:0    0 1022M  0 disk 

### 参考文档
[官方资料](http://openchannelssd.readthedocs.io/en/latest/gettingstarted/)

http://www.jianshu.com/p/8e11fa93411a

https://hyunyoung2.github.io/2016/10/04/LightNVM_With_OpenChannelSSD_On_QEMU/