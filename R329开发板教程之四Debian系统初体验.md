# [R329开发板教程之四|Debian系统初体验](https://aijishu.com/a/1060000000223179)

[AIPU](https://aijishu.com/t/aipu)[AI 芯片](https://aijishu.com/t/ai-chip)[AIoT](https://aijishu.com/t/aiot)[嵌入式系统](https://aijishu.com/t/embedded-system)

前面几节内容大家使用的是全志原厂的Tina系统，系统比较精简，所以开发起来可能有所不便。 近期矽速科技新移植了Debian系统到R329开发板上，在这里放出给大家体验。

本章节网盘资料：
debian镜像：

> 链接：[https://pan.baidu.com/s/1h-Lf6Y-xnvWUrOW6FSwitQ](https://aijishu.com/link?target=https%3A%2F%2Fpan.baidu.com%2Fs%2F1h-Lf6Y-xnvWUrOW6FSwitQ)
> 提取码：dl7s

armbian镜像：

> 链接：[https://pan.baidu.com/s/1RM8y...](https://aijishu.com/link?target=https%3A%2F%2Fpan.baidu.com%2Fs%2F1RM8yjut_Mnzy6XJF4y-wRg)
> 提取码：kh5n

## 镜像下载烧录

从以上网盘地址获取镜像文件（r329-mainline-debian-20210802.img）后，使用 dd 或者 win32diskImager 烧录镜像到TF卡。
镜像尺寸大约有2GB，请使用至少4GB的TF卡进行烧录。

## Debian系统基础使用

### 启动现象

将烧录好系统的卡插入卡槽，上电，可见屏幕上有两只企鹅图标，稍等片刻即进入了登录提示符界面，此时可以拔下摄像头，使用type-C转A母口转接头，插上键鼠进行操作；
当然正常使用方式是直接在串口终端操作, 用户名 root，密码 sipeed。
（附录附上了debian系统启动日志，启动有问题的可以参考该启动日志）

### 根分区扩容

debian镜像默认为2个分区，第一分区是boot分区，第二分区为根文件系统所在分区。
默认第二分区是按照根文件系统大小配置，所以启动后会发现可用空间为0，所以首次启动后要先进行分区扩容。

1. fdisk /dev/mmcblk0
   1. d 2 #删除第二分区
   2. n p 2 回车 回车 No #重建第二分区，剩余空间全部给第二分区
   3. w #保存配置
2. partx #查看，高速内核去识别等级硬盘信息
3. resize2fs /dev/mmcblk0p2

至此根分区扩容完成：

```
root@maixsense:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        29G  1.9G   26G   7% /
devtmpfs         55M     0   55M   0% /dev
tmpfs           120M     0  120M   0% /dev/shm
tmpfs            48M  848K   47M   2% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs            24M     0   24M   0% /run/user/0
```

### SWAP分区设置

由于R329内置内存较小，运行debian系统没有问题，但是要进行本地编译的话，就必须开启swap分区。

1. 设置swapnees(可选)
   1. swappiness参数值可设置范围在0到100之间。 此参数值越低,就会让Linux系统尽量少用swap分区,多用内存;参数值越高就是反过来,使内核更多的去使用swap空间
   2. cat /proc/sys/vm/swappiness
   3. echo "vm.swappiness = 50" >> /etc/sysctl.conf #永久修改
   4. sysctl -p #启用内存阈值设置
2. 创建swap分区
   1. cd /opt && dd if=/dev/zero of=swapfile bs=1M count=1024 #1GB swap空间，约2~3分钟
   2. mkswap swapfile
3. 开启关闭swap分区
   1. swapon swapfile / swapoff swapfile
4. 开机自动挂载swap分区（可选）
   1. 在/etc/fstab内加入
      1. /opt/swapfile swap swap defaults 0 0

至此已完成swap分区设置，可使用free查看当前内存情况：

```
root@maixsense:/opt# free -h
               total        used        free      shared  buff/cache   available
Mem:           238Mi        82Mi       3.0Mi       0.0Ki       151Mi       146Mi
Swap:          1.0Gi          0B       1.0Gi
```

### 联网操作

使用 nmtui 指令可以进入可视化的配网界面，
选择 Activate a connection ， 选择对应的SSID，输入连接密码，确认即可。

### 安装软件包

debian系统内置了便捷的软件包管理系统，按照正常PC端的操作方式操作即可，比如我们要按照ifconfig 指令：

```
apt-get install net-tools
```

等待片刻即可安装完成

### 手工生成ssh秘钥

系统暂时没有自动生成ssh秘钥，需要手工生成秘钥才能使能ssh。

```
ssh-keygen -A
systemctl restart ssh
```

## Debian系统下zhouyi使用

默认镜像的 ~/zhouyi_test 目录下放置了zhouyi_cam 例程和模型，在此用户可以体验本机编译zhouyi程序：

```
cd ~/zhouyi_test/build
make clean
cmake ..
time make -j2  #注意必须开启swap才能编译通过
```

经过漫长的等待，即可得到zhouyi_cam可执行文件，可以本机执行测试。

## 附录

### Debian启动日志

```
U-Boot SPL 2021.07-00715-g432f8404c1 (Jul 23 2021 - 14:10:47 +0800)
DRAM: 256 MiB
Trying to boot from MMC1
NOTICE:  BL31: v2.5(debug):v2.5-198-g01da1cf09-dirty
NOTICE:  BL31: Built : 11:39:03, Jul 23 2021
NOTICE:  BL31: Detected Allwinner R329 SoC (1851)
NOTICE:  BL31: Found U-Boot DTB at 0xc07ab48, model: Sipeed MaixSense
INFO:    ARM GICv2 driver initialized
INFO:    Configuring SPC Controller
INFO:    BL31: Platform setup done
INFO:    BL31: Initializing runtime services
INFO:    BL31: cortex_a53: CPU workaround for 855873 was applied
INFO:    BL31: cortex_a53: CPU workaround for 1530924 was applied
INFO:    PSCI: Suspend is unavailable
INFO:    BL31: Preparing for EL3 exit to normal world
INFO:    Entry point address = 0x4a000000
INFO:    SPSR = 0x3c9


U-Boot 2021.07-00715-g432f8404c1 (Jul 23 2021 - 14:10:47 +0800) Allwinner Technology

CPU:   Allwinner R329 (SUN50I)
Model: Sipeed MaixSense
DRAM:  256 MiB
MMC:   mmc@4020000: 0, mmc@4021000: 1
Loading Environment from FAT... Unable to read "uboot.env" from mmc0:1... In:    serial@2500000
Out:   serial@2500000
Err:   serial@2500000
Net:   No ethernet found.
Hit any key to stop autoboot:  0 
switch to partitions #0, OK
mmc0 is current device
Scanning mmc 0:1...
Found U-Boot script /boot.scr
286 bytes read in 2 ms (139.6 KiB/s)
## Executing script at 4fc00000
14522376 bytes read in 1202 ms (11.5 MiB/s)
11105 bytes read in 4 ms (2.6 MiB/s)
Moving Image from 0x40080000 to 0x40200000, end=41040000
## Flattened Device Tree blob at 44a00000
   Booting using the fdt blob at 0x44a00000
   Loading Device Tree to 0000000049ffa000, end 0000000049fffb60 ... OK

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]
[    0.000000] Linux version 5.14.0-rc3+ (icenowy@ice-390e5) (aarch64-aosc-linux-gnu-gcc (GCC) 9.2.1 20191220, GNU ld (GNU Binutils) 2.33.1) #107 SMP PREEMPT Fri Jul 30 11:42:44 CST 2021
[    0.000000] Machine model: Sipeed MaixSense
[    0.000000] earlycon: uart0 at MMIO32 0x0000000002500000 (options '')
[    0.000000] printk: bootconsole [uart0] enabled
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000040000000-0x000000004fffffff]
[    0.000000]   DMA32    empty
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000040000000-0x000000004fffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000040000000-0x000000004fffffff]
[    0.000000] cma: Reserved 128 MiB at 0x0000000041c00000
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv1.1 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: MIGRATE_INFO_TYPE not supported.
[    0.000000] psci: SMC Calling Convention v1.2
[    0.000000] percpu: Embedded 22 pages/cpu s51224 r8192 d30696 u90112
[    0.000000] Detected VIPT I-cache on CPU0
[    0.000000] CPU features: detected: ARM erratum 845719
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 64512
[    0.000000] Kernel command line: console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait earlycon=uart,mmio32,0x02500000 panic=10 fbcon=font:6x8
[    0.000000] Dentry cache hash table entries: 32768 (order: 6, 262144 bytes, linear)
[    0.000000] Inode-cache hash table entries: 16384 (order: 5, 131072 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 111220K/262144K available (8512K kernel code, 998K rwdata, 3108K rodata, 1472K init, 371K bss, 19852K reserved, 131072K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu:     RCU event tracing is enabled.
[    0.000000] rcu:     RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=2.
[    0.000000]     Trampoline variant of Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=2
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] Root IRQ handler: gic_handle_irq
[    0.000000] GIC: Using split EOI/Deactivate mode
[    0.000000] random: get_random_bytes called from start_kernel+0x49c/0x604 with crng_init=0
[    0.000000] arch_timer: cp15 timer(s) running at 24.00MHz (phys).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x588fe9dc0, max_idle_ns: 440795202592 ns
[    0.000000] sched_clock: 56 bits at 24MHz, resolution 41ns, wraps every 4398046511097ns
[    0.009016] Console: colour dummy device 80x25
[    0.013927] Calibrating delay loop (skipped), value calculated using timer frequency.. 48.00 BogoMIPS (lpj=96000)
[    0.024184] pid_max: default: 32768 minimum: 301
[    0.029309] LSM: Security Framework initializing
[    0.033967] Mount-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.041898] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.051583] rcu: Hierarchical SRCU implementation.
[    0.057304] smp: Bringing up secondary CPUs ...
[    0.062709] Detected VIPT I-cache on CPU1
[    0.062777] CPU1: Booted secondary processor 0x0000000001 [0x410fd034]
[    0.062878] smp: Brought up 1 node, 2 CPUs
[    0.078967] SMP: Total of 2 processors activated.
[    0.084132] CPU features: detected: 32-bit EL0 Support
[    0.089280] CPU features: detected: CRC32 instructions
[    0.095065] CPU: All CPU(s) started at EL2
[    0.099570] alternatives: patching kernel code
[    0.105142] devtmpfs: initialized
[    0.111274] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.121996] futex hash table entries: 512 (order: 3, 32768 bytes, linear)
[    0.133741] pinctrl core: initialized pinctrl subsystem
[    0.139651] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.147269] DMA: preallocated 128 KiB GFP_KERNEL pool for atomic allocations
[    0.154833] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA pool for atomic allocations
[    0.163082] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA32 pool for atomic allocations
[    0.171538] audit: initializing netlink subsys (disabled)
[    0.177661] audit: type=2000 audit(0.124:1): state=initialized audit_enabled=0 res=1
[    0.178018] thermal_sys: Registered thermal governor 'step_wise'
[    0.186180] thermal_sys: Registered thermal governor 'user_space'
[    0.192390] thermal_sys: Registered thermal governor 'power_allocator'
[    0.199300] cpuidle: using governor menu
[    0.210187] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    0.217685] ASID allocator initialised with 65536 entries
[    0.234455] HugeTLB registered 1.00 GiB page size, pre-allocated 0 pages
[    0.241844] HugeTLB registered 32.0 MiB page size, pre-allocated 0 pages
[    0.248543] HugeTLB registered 2.00 MiB page size, pre-allocated 0 pages
[    0.255899] HugeTLB registered 64.0 KiB page size, pre-allocated 0 pages
[    0.264744] cryptd: max_cpu_qlen set to 1000
[    0.272370] SCSI subsystem initialized
[    0.276874] usbcore: registered new interface driver usbfs
[    0.282936] usbcore: registered new interface driver hub
[    0.288352] usbcore: registered new device driver usb
[    0.293599] mc: Linux media interface: v0.10
[    0.298309] videodev: Linux video capture interface: v2.00
[    0.304626] Advanced Linux Sound Architecture Driver Initialized.
[    0.311839] clocksource: Switched to clocksource arch_sys_counter
[    0.318816] VFS: Disk quotas dquot_6.6.0
[    0.322881] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    0.335758] NET: Registered PF_INET protocol family
[    0.341228] IP idents hash table entries: 4096 (order: 3, 32768 bytes, linear)
[    0.348909] tcp_listen_portaddr_hash hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.357342] TCP established hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.365888] TCP bind hash table entries: 2048 (order: 3, 32768 bytes, linear)
[    0.373076] TCP: Hash tables configured (established 2048 bind 2048)
[    0.379533] UDP hash table entries: 256 (order: 1, 8192 bytes, linear)
[    0.386721] UDP-Lite hash table entries: 256 (order: 1, 8192 bytes, linear)
[    0.394488] NET: Registered PF_UNIX/PF_LOCAL protocol family
[    0.401934] Initialise system trusted keyrings
[    0.407048] workingset: timestamp_bits=46 max_order=16 bucket_order=0
[    0.419805] fuse: init (API version 7.34)
[    0.466599] Key type asymmetric registered
[    0.471118] Asymmetric key parser 'x509' registered
[    0.476515] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 249)
[    0.484636] io scheduler mq-deadline registered
[    0.489618] io scheduler kyber registered
[    0.497782] sun50i-r329-r-pinctrl 7022000.pinctrl: initialized sunXi PIO driver
[    0.505941] pwm-backlight backlight: supply power not found, using dummy regulator
[    0.524174] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
[    0.532702] cacheinfo: Unable to detect cache hierarchy for CPU 0
[    0.545965] loop: module loaded
[    0.550634] libphy: Fixed MDIO Bus: probed
[    0.554892] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    0.562061] ehci-platform: EHCI generic platform driver
[    0.567984] ehci-platform 4101000.usb: EHCI Host Controller
[    0.573579] ehci-platform 4101000.usb: new USB bus registered, assigned bus number 1
[    0.582177] ehci-platform 4101000.usb: irq 25, io mem 0x04101000
[    0.603865] ehci-platform 4101000.usb: USB 2.0 started, EHCI 1.00
[    0.610527] hub 1-0:1.0: USB hub found
[    0.614673] hub 1-0:1.0: 1 port detected
[    0.619327] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    0.626139] ohci-platform: OHCI generic platform driver
[    0.631540] ohci-platform 4101400.usb: Generic Platform OHCI controller
[    0.638167] ohci-platform 4101400.usb: new USB bus registered, assigned bus number 2
[    0.646806] ohci-platform 4101400.usb: irq 26, io mem 0x04101400
[    0.716384] hub 2-0:1.0: USB hub found
[    0.720171] hub 2-0:1.0: 1 port detected
[    0.724722] usbcore: registered new interface driver usb-storage
[    0.731320] udc-core: couldn't find an available UDC - added [g_cdc] to list of pending drivers
[    0.741342] input: 7030800.lradc as /devices/platform/soc/7030800.lradc/input/input0
[    0.750798] sun6i-rtc 7090000.rtc: registered as rtc0
[    0.756393] sun6i-rtc 7090000.rtc: setting system clock to 2021-08-02T03:28:17 UTC (1627874897)
[    0.765613] sun6i-rtc 7090000.rtc: RTC enabled
[    0.770571] i2c /dev entries driver
[    0.774488] usbcore: registered new interface driver uvcvideo
[    0.781764] sun50i-r329-r-pinctrl 7022000.pinctrl: supply vcc-pm not found, using dummy regulator
[    0.791630] ledtrig-cpu: registered to indicate activity on CPUs
[    0.798776] usbcore: registered new interface driver usbhid
[    0.804361] usbhid: USB HID core driver
[    0.809325] sun4i-codec 7032000.codec: Failed to register our card
[    0.816400] Initializing XFRM netlink socket
[    0.821399] NET: Registered PF_INET6 protocol family
[    0.827316] Segment Routing with IPv6
[    0.831425] NET: Registered PF_PACKET protocol family
[    0.837256] Loading compiled-in X.509 certificates
[    0.853474] sun50i-r329-pinctrl 2000400.pinctrl: initialized sunXi PIO driver
[    0.861648] sun50i-r329-pinctrl 2000400.pinctrl: supply vcc-pb not found, using dummy regulator
[    0.871408] sun50i-r329-pinctrl 2000400.pinctrl: supply vcc-ph not found, using dummy regulator
[    0.880481] pwm-backlight backlight: supply power not found, using dummy regulator
[    0.889502] printk: console [ttyS0] disabled
[    0.914467] 2500000.serial: ttyS0 at MMIO 0x2500000 (irq = 18, base_baud = 1500000) is a 16550A
[    0.923336] printk: console [ttyS0] enabled
[    0.923336] printk: console [ttyS0] enabled
[    0.932118] printk: bootconsole [uart0] disabled
[    0.932118] printk: bootconsole [uart0] disabled
[    0.943325] sun50i-r329-r-pinctrl 7022000.pinctrl: supply vcc-pn not found, using dummy regulator
[    0.952778] [drm] Initialized simple-dbi 1.0.0 20210723 for spi0.0 on minor 0
[    1.244314] Console: switching to colour frame buffer device 40x30
[    1.294662] simple-dbi spi0.0: [drm] fb0: simple-dbi frame buffer device
[    1.302391] sun50i-r329-pinctrl 2000400.pinctrl: supply vcc-pf not found, using dummy regulator
[    1.303360] debugfs: Directory '7032000.codec' with parent 'R329 Audio Codec' already present!
[    1.312192] sun50i-r329-pinctrl 2000400.pinctrl: supply vcc-pg not found, using dummy regulator
[    1.320271] sunxi-mmc 4020000.mmc: Got CD GPIO
[    1.333954] sunxi-mmc 4021000.mmc: allocated mmc-pwrseq
[    1.341405] usb_phy_generic usb_phy_generic.1.auto: supply vcc not found, using dummy regulator
[    1.350796] musb-hdrc musb-hdrc.2.auto: MUSB HDRC host driver
[    1.356585] musb-hdrc musb-hdrc.2.auto: new USB bus registered, assigned bus number 3
[    1.358477] sunxi-mmc 4020000.mmc: initialized, max. request size: 2048 KB, uses new timings mode
[    1.365005] hub 3-0:1.0: USB hub found
[    1.377271] hub 3-0:1.0: 1 port detected
[    1.382747] using random self ethernet address
[    1.387224] using random host ethernet address
[    1.392345] usb0: HOST MAC 4e:c1:97:4f:be:96
[    1.396729] usb0: MAC c6:46:59:3d:30:de
[    1.400621] g_cdc gadget: CDC Composite Gadget, version: King Kamehameha Day 2008
[    1.408119] g_cdc gadget: g_cdc ready
[    1.412383] ALSA device list:
[    1.415362]   #0: R329 Audio Codec
[    1.445092] mmc0: host does not support reading read-only switch, assuming write-enable
[    1.455039] mmc0: new high speed SDHC card at address b368
[    1.461239] mmcblk0: mmc0:b368 NCard 29.1 GiB 
[    1.467599]  mmcblk0: p1 p2
[    1.572403] sunxi-mmc 4021000.mmc: initialized, max. request size: 2048 KB, uses new timings mode
[    1.587363] sunxi-mmc 4021000.mmc: card claims to support voltages below defined range
[    1.606029] mmc1: new high speed SDIO card at address 0001
[    1.606229] EXT4-fs (mmcblk0p2): mounted filesystem with ordered data mode. Opts: (null). Quota mode: none.
[    1.621406] VFS: Mounted root (ext4 filesystem) readonly on device 179:2.
[    1.635845] devtmpfs: mounted
[    1.639596] Freeing unused kernel memory: 1472K
[    1.644247] Run /sbin/init as init process
[    1.791874] usb 1-1: new high-speed USB device number 2 using ehci-platform
[    1.806476] random: fast init done
[    1.982698] usb 1-1: Found UVC 1.00 device USB 2.0 Camera (0c45:6366)
[    2.001896] input: USB 2.0 Camera: USB Camera as /devices/platform/soc/4101000.usb/usb1/1-1/1-1:1.0/input/input1
[    2.695525] systemd[1]: systemd 247.3-6 running in system mode. (+PAM +AUDIT +SELINUX +IMA +APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +ZSTD +SECCOMP +BLKID +ELFUTILS +KMOD +IDN2 -IDN +PCRE2 default-hierarchy=unified)
[    2.719956] systemd[1]: Detected architecture arm64.

Welcome to Debian GNU/Linux 11 (bullseye)!

[    2.771879] systemd[1]: Set hostname to <maixsense>.
[    3.734802] systemd[1]: Queued start job for default target Graphical Interface.
[    3.743625] random: systemd: uninitialized urandom read (16 bytes read)
[    3.750487] systemd[1]: system-getty.slice: unit configures an IP firewall, but the local system does not support BPF/cgroup firewalling.
[    3.762852] systemd[1]: (This warning is only shown for the first unit using IP firewalling.)
[    3.773818] systemd[1]: Created slice system-getty.slice.
[  OK  ] Created slice system-getty.slice.
[    3.796035] random: systemd: uninitialized urandom read (16 bytes read)
[    3.804015] systemd[1]: Created slice system-modprobe.slice.
[  OK  ] Created slice system-modprobe.slice.
[    3.824054] random: systemd: uninitialized urandom read (16 bytes read)
[    3.832264] systemd[1]: Created slice system-serial\x2dgetty.slice.
[  OK  ] Created slice system-serial\x2dgetty.slice.
[    3.857036] systemd[1]: Created slice User and Session Slice.
[  OK  ] Created slice User and Session Slice.
[    3.880378] systemd[1]: Started Dispatch Password Requests to Console Directory Watch.
[  OK  ] Started Dispatch Password …ts to Console Directory Watch.
[    3.904312] systemd[1]: Started Forward Password Requests to Wall Directory Watch.
[  OK  ] Started Forward Password R…uests to Wall Directory Watch.
[    3.928164] systemd[1]: Condition check resulted in Arbitrary Executable File Formats File System Automount Point being skipped.
[    3.940071] systemd[1]: Reached target Local Encrypted Volumes.
[  OK  ] Reached target Local Encrypted Volumes.
[    3.960202] systemd[1]: Reached target Paths.
[  OK  ] Reached target Paths.
[    3.980066] systemd[1]: Reached target Remote File Systems.
[  OK  ] Reached target Remote File Systems.
[    4.000046] systemd[1]: Reached target Slices.
[  OK  ] Reached target Slices.
[    4.028410] systemd[1]: Listening on Syslog Socket.
[  OK  ] Listening on Syslog Socket.
[    4.048596] systemd[1]: Listening on fsck to fsckd communication Socket.
[  OK  ] Listening on fsck to fsckd communication Socket.
[    4.072377] systemd[1]: Listening on initctl Compatibility Named Pipe.
[  OK  ] Listening on initctl Compatibility Named Pipe.
[    4.096869] systemd[1]: Listening on Journal Audit Socket.
[  OK  ] Listening on Journal Audit Socket.
[    4.120583] systemd[1]: Listening on Journal Socket (/dev/log).
[  OK  ] Listening on Journal Socket (/dev/log).
[    4.144681] systemd[1]: Listening on Journal Socket.
[  OK  ] Listening on Journal Socket.
[    4.167025] systemd[1]: Listening on udev Control Socket.
[  OK  ] Listening on udev Control Socket.
[    4.188527] systemd[1]: Listening on udev Kernel Socket.
[  OK  ] Listening on udev Kernel Socket.
[    4.211538] systemd[1]: Mounting Huge Pages File System...
         Mounting Huge Pages File System...
[    4.235560] systemd[1]: Mounting POSIX Message Queue File System...
         Mounting POSIX Message Queue File System...
[    4.260314] systemd[1]: Mounting Kernel Debug File System...
         Mounting Kernel Debug File System...
[    4.280509] systemd[1]: Condition check resulted in Kernel Trace File System being skipped.
[    4.293950] systemd[1]: Starting Create list of static device nodes for the current kernel...
         Starting Create list of st…odes for the current kernel...
[    4.324935] systemd[1]: Starting Load Kernel Module configfs...
         Starting Load Kernel Module configfs...
[    4.347757] systemd[1]: Starting Load Kernel Module drm...
         Starting Load Kernel Module drm...
[    4.372117] systemd[1]: Starting Load Kernel Module fuse...
         Starting Load Kernel Module fuse...
[    4.426602] systemd[1]: Condition check resulted in Set Up Additional Binary Formats being skipped.
[    4.439763] systemd[1]: Starting File System Check on Root Device...
         Starting File System Check on Root Device...
[    4.467968] systemd[1]: Starting Journal Service...
         Starting Journal Service...
[    4.502705] systemd[1]: Starting Load Kernel Modules...
         Starting Load Kernel Modules...
[    4.520254] systemd[1]: Starting Coldplug All udev Devices...
         Starting Coldplug All udev Devices...
[    4.539025] systemd[1]: Mounted Huge Pages File System.
[  OK      4.547136] systemd[1]: Mounted POSIX Message Queue File System.
0m] Mounted Huge Pages [    4.555627] systemd[1]: Mounted Kernel Debug File System.
File System.
[  OK  [    4.564394] systemd[1]: Finished Create list of static device nodes for the current kernel.
] Mounted POSIX Mes[    4.575378] systemd[1]: modprobe@configfs.service: Succeeded.
sage Queue File System.
[  OK  ] Mounted Kernel Debug F[    4.587384] systemd[1]: Finished Load Kernel Module configfs.
ile System.
[  OK   systemd[1]: modprobe@drm.service: Succeeded.
[0m] Finished     4.603652] systemd[1]: Finished Load Kernel Module drm.
0;1;39mCreate list of st… node[    4.610914] systemd[1]: modprobe@fuse.service: Succeeded.
s for the current kernel.
[[    4.618865] systemd[1]: Finished Load Kernel Module fuse.
  OK  ] Finished     4.627566] systemd[1]: Finished Load Kernel Modules.
1;39mLoad Kernel Module configfs.
[  OK  ] Finis[    4.638607] systemd[1]: Mounting FUSE Control File System...
hed Load Kernel Module drm.
[  OK  ] Fi[    4.649232] systemd[1]: Mounting Kernel Configuration File System...
nished Load Kernel Module fuse.
[  OK  ] Finished Load Kernel Modules.
         Mounting FUSE Co[    4.668024] systemd[1]: Started File System Check Daemon to report status.
ntrol File System...
         Mounting Kernel Conf[    4.680465] systemd[1]: Starting Apply Kernel Variables...
iguration File System...
[  OK  ] Started     4.691331] systemd[1]: Mounted FUSE Control File System.
39mFile System C[    4.698878] systemd[1]: Mounted Kernel Configuration File System.
heck Daemon to report status.
         Starting Apply Kernel Variables...
[  OK  ] Mounted FUSE Control File System.
[  OK  ] Mounted Kernel Configuration File System.
[    4.778823] systemd[1]: Started Journal Service.
[  OK  ] Started Journal Service.
[  OK  ] Finished File System Check on Root Device.
         Starting Remount Root and Kernel File Systems...
[  OK  ] Finished Apply Kernel Variables.
[    4.918517] EXT4-fs (mmcblk0p2): re-mounted. Opts: (null). Quota mode: none.
[  OK  ] Finished Remount Root and Kernel File Systems.
         Activating swap /opt/swapfile...
         Starting Flush Journal to Persistent Storage...
         Starting Load/Save Random Seed...
         Starting Create System Users...
[    5.064330] Adding 1048572k swap on /opt/swapfile.  Priority:-2 extents:4 across:1097728k SS
[  OK  ] Activated swap /opt/swapfile.
[    5.090738] systemd-journald[119]: Received client request to flush runtime journal.
[  OK  ] Reached target Swap.
[  OK  ] Finished Coldplug All udev Devices.
         Starting Helper to synchronize boot up for ifupdown...
[    5.499281] random: crng init done
[    5.502700] random: 7 urandom warning(s) missed due to ratelimiting
[  OK  ] Finished Create System Users.
         Starting Create Static Device Nodes in /dev...
[    5.730757] systemd-journald[119]: File /var/log/journal/5ad538829eb746f7b2c745d9f98d663c/system.journal corrupted or uncleanly shut down, renaming and replacing.
[  OK  ] Finished Helper to synchronize boot up for ifupdown.
[  OK  ] Finished Load/Save Random Seed.
[  OK  ] Finished Create Static Device Nodes in /dev.
[  OK  ] Reached target Local File Systems (Pre).
[  OK  ] Reached target Local File Systems.
         Starting Raise network interfaces...
         Starting Rule-based Manage…for Device Events and Files...
[  OK  ] Finished Flush Journal to Persistent Storage.
         Starting Create Volatile Files and Directories...
[   27.306945] EXT4-fs error (device mmcblk0p2): ext4_validate_block_bitmap:390: comm systemd-tmpfile: bg 32: bad block bitmap checksum
[  OK  ] Finished Raise network interfaces.
[  OK  ] Finished Create Volatile Files and Directories.
         Starting Network Time Synchronization...
         Starting Update UTMP about System Boot/Shutdown...
[  OK  ] Finished Update UTMP about System Boot/Shutdown.
[  OK  ] Started Network Time Synchronization.
[  OK  ] Reached target System Time Set.
[  OK  ] Reached target System Time Synchronized.
[  OK  ] Started Rule-based Manager for Device Events and Files.
[  OK  ] Reached target System Initialization.
[  OK  ] Started Daily apt download activities.
[  OK  ] Started Daily apt upgrade and clean activities.
[  OK  ] Started Periodic ext4 Onli…ata Check for All Filesystems.
[  OK  ] Started Discard unused blocks once a week.
[  OK  ] Started Daily rotation of log files.
[  OK  ] Started Daily Cleanup of Temporary Directories.
[  OK  ] Reached target Timers.
[  OK  ] Listening on D-Bus System Message Bus Socket.
[  OK  ] Reached target Sockets.
[  OK  ] Reached target Basic System.
[  OK  ] Started Regular background program processing daemon.
[  OK  ] Started D-Bus System Message Bus.
         Starting Network Manager...
         Starting Remove Stale Onli…t4 Metadata Check Snapshots...
         Starting Authorization Manager...
         Starting System Logging Service...
         Starting User Login Management...
         Starting WPA supplicant...
[  OK  ] Created slice system-systemd\x2dbacklight.slice.
         Starting Load/Save Screen …ness of backlight:backlight...
[  OK  ] Found device /dev/ttyS0.
[   37.942508] aipu: loading out-of-tree module taints kernel.
[   37.956534] armchina 3050000.aipu: AIPU KMD probe start...
[   37.974397] armchina 3050000.aipu: KMD version: release 3.0.13
[   37.980840] armchina 3050000.aipu: no interrupts-reg specified
[   38.013838] armchina 3050000.aipu: AIPU is not behind an IOMMU
[   38.035808] armchina 3050000.aipu: AIPU DRAM mem type is [CMA default]
[  OK  ] Finished Load/Save Screen …htness of backlight:backlight.
[   38.076050] armchina 3050000.aipu: init DRAM region done: pa [0x41e00000, 0x443fffff]
[   38.083997] armchina 3050000.aipu: AIPU core #0 detected: zhouyi-v1-0600
[   38.091084] armchina 3050000.aipu: AIPU Initial Status: 0x70000
[   38.097058] armchina 3050000.aipu: ########## AIPU CORE 0: ZHOUYI V1 ##########
[   38.104445] armchina 3050000.aipu: # ISA Version Register: 0x0
[   38.110317] armchina 3050000.aipu: # TPC Feature Register: 0x10101
[   38.116539] armchina 3050000.aipu: # SPU Feature Register: 0x0
[   38.122396] armchina 3050000.aipu: # HWA Feature Register: 0x10111
[   38.128598] armchina 3050000.aipu: # Revision ID Register: 0x0
[   38.134437] armchina 3050000.aipu: # Memory Hierarchy Feature Register: 0x170
[   38.141592] armchina 3050000.aipu: # Instruction RAM Feature Register:  0x0
[   38.148605] armchina 3050000.aipu: # TEC Local SRAM Feature Register:   0x312
[   38.155844] armchina 3050000.aipu: # Global SRAM Feature Register:      0x10
[   38.163252] armchina 3050000.aipu: # Instruction Cache Feature Register:0x233
[   38.189511] armchina 3050000.aipu: # Data Cache Feature Register:       0x70030
[   38.198989] armchina 3050000.aipu: # L2 Cache Feature Register:         0x0
[   38.199005] armchina 3050000.aipu: ############################################
[   38.199013] armchina 3050000.aipu: initialize AIPU core #0 done
[  OK  ] Reached target Hardware activated USB gadget.
[  OK  ] Reached target Sound Card.
[  OK  ] Finished Remove Stale Onli…ext4 Metadata Check Snapshots.
[  OK  ] Started System Logging Service.
[   39.302098] cfg80211: Loading compiled-in X.509 certificates for regulatory database
[   39.326872] cfg80211: Loaded X.509 cert 'sforshee: 00b28ddf47aef9cea7'
[  OK  ] Listening on Load/Save RF …itch Status /dev/rfkill Watch.
[   39.694191] cfg80211: loaded regulatory.db is malformed or signature is missing/invalid
[  OK  ] Started User Login Management.
         Starting Load/Save RF Kill Switch Status...
[  OK  ] Started WPA supplicant.
[  OK  ] Started Load/Save RF Kill Switch Status.
[  OK  ] Started Authorization Manager.
         Starting Modem Manager...
[  OK  ] Started Network Manager.
[  OK  ] Reached target Network.
         Starting OpenBSD Secure Shell server...
         Starting Permit User Sessions...
[  OK  ] Finished Permit User Sessions.
[  OK  ] Started Getty on tty1.
[  OK  ] Started Serial Getty on ttyS0.
[  OK  ] Reached target Login Prompts.
         Starting Hostname Service...
[FAILED] Failed to start OpenBSD Secure Shell server.
See 'systemctl status ssh.service' for details.
[  OK  ] Started Modem Manager.
[  OK  ] Reached target Multi-User System.
[  OK  ] Reached target Graphical Interface.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Hostname Service.
[  OK  ] Finished Update UTMP about System Runlevel Changes.
[  OK  ] Stopped OpenBSD Secure Shell server.
         Starting OpenBSD Secure Shell server...
[FAILED] Failed to start OpenBSD Secure Shell server.
See 'systemctl status ssh.service' for details.

Debian GNU/Linux 11 maixsense ttyS0

maixsense login: [   49.997407] IPv6: ADDRCONF(NETDEV_CHANGE): wlan0: link becomes ready
root
Password: 
Linux maixsense 5.14.0-rc3+ #107 SMP PREEMPT Fri Jul 30 11:42:44 CST 2021 aarch64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed Jul 14 01:30:05 CST 2021 on ttyS0
root@maixsense:~# 
```

R329开发板教程系列：

- [R329开发板系列教程之二|实机运行aipu程序](https://aijishu.com/a/1060000000220719)
- [R329教程一|周易 AIPU 部署及仿真教程](https://aijishu.com/a/1060000000216857)
- [R329开发板教程之三|视觉模型实时运行](https://aijishu.com/a/1060000000221762)