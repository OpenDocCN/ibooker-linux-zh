# 第十章：获取计算机硬件的详细信息

Linux 自带了几个好用的工具，用于获取计算机中各个硬件组件的详细信息。您可以坐在计算机旁，几分钟内就能了解其组件及其规格，而无需打开机箱。

这些实用程序用于提供技术支持的详细信息，查找设备的正确驱动程序以及了解它是否完全在 Linux 中受支持。您不能指望制造商及时提供其产品的准确信息。例如，他们经常更改芯片组而不更改型号号码，这可能会将在 Linux 上正常工作的设备变成在 Linux 上不起作用的设备。幸运的是，在现代这些时代，Linux 支持要比过去少得多的麻烦。

理想情况下，您还应该有您的计算机文档，或至少是主板手册。主板手册通常充满了照片，图表和有用的信息，您应该能够在线找到它们。

在本章中，您将了解 *lshw*（列出硬件）、*lspci*（列出 PCI）、*hwinfo*（硬件信息）、*lsusb*（列出 USB）、*lscpu*（列出 CPU）和 *lsblk*（列出块设备）命令。

*lshw* 和 *hwinfo* 提供了最完整的信息。

*lshw* 报告内存配置，固件版本，主板配置，CPU 版本和速度，缓存配置，总线速度，硬件路径，附加设备，分区和文件系统。

*hwinfo* 报告计算机显示器信息，RAID 数组，内存配置，CPU 信息，固件，主板配置，缓存，总线速度，附加设备，分区和文件系统。

*lsusb* 探测 USB 总线及其连接的设备。

*lspci* 探测 PCI 总线及其连接的设备。

*lsblk* 列出物理驱动器，分区和文件系统。

*lscpu* 列出有关您的 CPU 的信息。

# 10.1 使用 lshw 收集硬件信息

## 问题

您想要系统上硬件的清单以及有关每个项目的详细信息。

## 解决方案

尝试使用没有选项的 *lshw*（硬件列表）命令，并将输出存储在文本文件中：

```
$ sudo lshw | tee hardware.txt
duchess
    description: Laptop
    product: Latitude E7240 (05CA)
    vendor: Dell Inc.
    version: 00
    serial: 456ABC1
    width: 64 bits
[...]
```

您将获得数百行输出，其中包括固件，驱动程序，功能，序列号，版本号和总线信息。 *lshw* 不会探测通过无线网络接口连接的任何设备，例如无线打印机或通过蓝牙连接的智能手机，但它会报告无线和蓝牙接口。

您可能更喜欢以硬件路径树视图的摘要形式：

```
$ sudo lshw -short
H/W path         Device           Class          Description
============================================================
                                  system         To Be Filled By O.E.M.
/0                                bus            H97M Pro4
/0/0                              memory         64KiB BIOS
/0/b                              memory         16GiB System Memory
/0/b/0                            memory         DIMM [empty]
/0/b/1                            memory         8GiB DIMM DDR3 Synchronous
1333
MHz (0.8 ns)
[...]
/0/100/14/0/5                     bus            USB3.0 Hub
/0/100/14/0/5/1                   generic        SAMSUNG_Android
/0/100/14/0/5/2                   printer        MFC-J5945DW
/0/100/14/0/5/4  wlx9cefd5fe8f20  network        802.11 n WLAN
/0/100/14/0/b                     input          USB Optical Mouse
/0/100/14/0/c                     input          QuickFire Rapid keyboard
[...]
```

或者尝试摘要总线视图，而不是硬件路径视图：

```
$ sudo lshw -businfo
Bus info          Device           Class          Description
=============================================================
[...]
cpu@0                              processor      Intel(R) Core(TM) i7-4770K
CPU
@ 3.50GHz
usb@3:5.4         wlx9cefd5fe8f20  network        802.11 n WLAN
usb@3:b                            input          USB Optical Mouse
usb@3:c                            input          QuickFire Rapid keyboard
pci@0000:00:19.0  enp0s25          network        Ethernet Connection (2) I218-V
pci@0000:00:1a.0                   bus            9 Series Chipset Family USB
scsi@0:0.0.0      /dev/sda         disk           4TB ST4000DM000-1F21
scsi@0:0.0.0,1    /dev/sda1        volume         476MiB EXT4 volume
[...]
```

*lshw* 有一个图形界面，您可以用 *sudo lshw -X* 打开。这通常是一个单独的软件包，例如在 Ubuntu 上是 *lshw-gtk*，在 openSUSE 和 Fedora 上是 *lshw-gui*。

## 讨论

*lshw* 将大量信息打包到其输出中。访问 [Hardware Lister (lshw)](https://oreil.ly/XRGx1) 了解所有内容的含义。

*lshw* 无法检测到 FireWire 接口或计算机显示器。

示例中说“系统待填充”，因为它是一台自制机器。例如联想或戴尔等品牌计算机应该有品牌名称和型号。

*H/W 路径*列包含硬件路径，类似于文件路径。/0 是 */system/bus*，表示计算机和主板。然后所有后续条目都以树形视图显示，类似于文件树。正如您在示例输出中看到的那样，/0/0 是 */system/bus/BIOS memory*，/0/b 是第一个已填充的 RAM 插槽，/0/b/1 是第二个已填充的 RAM 插槽。这些路径对应于主板上的物理连接点，通常被称为*插槽*，尽管大多数都已焊接到主板上，没有物理插槽可插入扩展卡。

## 参见

+   [硬件列表 (lshw)](https://oreil.ly/axiyL)

+   *man 1 lshw*

# 10.2 过滤 lshw 输出

## 问题

*lshw* 确实会输出大量信息，您希望限制输出到您想要查看的内容。

## 解决方案

运行 *sudo lshw -short* 或 *sudo lshw -businfo* 查看设备类别列表，然后命名一个或多个您想要查看的设备类别：

```
$ sudo lshw -short -class bus -class cpu

```

删除 *-short* 选项以查看详细信息。

将长输出格式化为 HTML、XML 或 JSON，并将其存储在文件中，以便您可以使用您喜爱的脚本技巧解析输出：

```
$ sudo lshw -html -class bus -class cpu | tee lshw.html
$ sudo lshw -xml -class printer -class display -class input | tee lshw.xml
$ sudo lshw -json -class storage | tee lshw.json
```

使用 *-sanitize* 选项移除敏感信息（例如 IP 地址和序列号），使其更安全地与技术支持共享：

```
$ sudo lshw -json -sanitize  -class bus -class cpu | tee lshw.json

```

## 讨论

*tee* 命令在屏幕上显示输出并将其存储在文本文件中。

## 参见

+   [硬件列表 (lshw)](https://oreil.ly/qCioO)

+   *man 1 lshw*

# 10.3 使用 hwinfo 检测硬件，包括显示器和 RAID 设备

## 问题

您希望获取有关计算机显示器和 RAID 设备以及系统上其他设备的信息。

## 解决方案

*hwinfo*命令提供详细的硬件清单，包括系统中的监视器和 RAID 设备。以下示例探测您的显示器：

```
$ hwinfo --monitor
[...]
  Hardware Class: monitor
  Model: "VIEWSONIC VX2450 SERIES"
  Vendor: VSC "VIEWSONIC"
  Device: eisa 0xe226 "VX2450 SERIES"
[...]
```

完整输出比本示例要长得多，包括所有支持的屏幕分辨率、制造日期、同步范围、显示器类型和刷新频率。

另一个很好的功能是检测 RAID 设备。它默认不会检测到它们，因此请使用 *--listmd* 选项：

```
$ hwinfo --listmd
```

如果没有返回任何内容，则系统上没有 RAID 设备。如果有，它会打印大量信息。

创建硬件摘要：

```
$ hwinfo --short
keyboard:
  /dev/input/event4    CM Storm QuickFire Rapid keyboard
mouse:
  /dev/input/event5    CM Storm QuickFire Rapid keyboard
  /dev/input/mice      Logitech Optical Wheel Mouse
printer:
                       Brother Industries MFC-J5945DW
monitor:
                       VIEWSONIC VX2450 SERIES
graphics card:
                       Intel Xeon E3-1200 v3/4th Gen Core Processor Integrated
[...]
```

获取一个或多个硬件组件的详细信息：

```
$ hwinfo --mouse --network --cdrom

```

参阅 *man 8 hwinfo* 获取设备名称列表，或运行 *hwinfo --help*：

```
$ hwinfo --help
Usage: hwinfo [OPTIONS]
Probe for hardware.
Options:
    --<HARDWARE_ITEM>
        This option can be given more than once. Probe for a particular
        HARDWARE_ITEM. Available hardware items are:
        all, arch, bios, block, bluetooth, braille, bridge, camera,
        cdrom, chipcard, cpu, disk, dsl, dvb, fingerprint, floppy,
        framebuffer, gfxcard, hub, ide, isapnp, isdn, joystick, keyboard,
        memory, mmc-ctrl, modem, monitor, mouse, netcard, network, partition,
        pci, pcmcia, pcmcia-ctrl, pppoe, printer, redasd,
        reallyall, scanner, scsi, smp, sound, storage-ctrl, sys, tape,
        tv, uml, usb, usb-ctrl, vbe, wlan, xen, zip
[...]
```

## 讨论

*hwinfo* 提供有用且完整的信息。例如，对于网络接口，它显示它们的 */sys* 路径、驱动程序、链路状态和 MAC 地址。CD-ROM 输出包括型号名称、版本号、驱动程序、设备文件、驱动速度、功能列表以及驱动器中是否有磁盘。*hwinfo* 通常比制造商的产品信息提供更多信息。

## 参见

+   *man 8 hwinfo*

+   [hwinfo 在 GitHub 上的链接](https://oreil.ly/BsDAT)

# 10.4 使用 lspci 检测 PCI 硬件

## 问题

你想列出连接到计算机 PCI 总线上的设备及其供应商和版本信息。

## 解决方案

运行 *lspci*（列出 PCI）命令。以下示例打印所有 PCI 设备的摘要列表：

```
$ lspci
00:00.0 Host bridge: Intel Corporation 4th Gen Core Processor DRAM Controller
(rev 06)
00:02.0 VGA compatible controller: Intel Corporation Xeon E3-1200 v3/4th Gen
Core Processor Integrated Graphics Controller (rev 06)
00:03.0 Audio device: Intel Corporation Xeon E3-1200 v3/4th Gen Core Processor
HD Audio Controller (rev 06)
[...]
```

增加详细信息以查看更多细节：

```
$ lspci -v
$ lspci -vv
$ lspci -vvv
```

当你看到“拒绝访问”消息时，尝试使用 *sudo lspci* 查看你错过了什么。

## 讨论

*lspci* 从 PCI 总线读取信息，包括主板上的内置组件以及插入 PCI 插槽的扩展卡。

*lspci* 从其硬件 ID 数据库中显示额外信息，如供应商、设备、类别和子类信息。此信息存储在文本文件中，具体位置因 Linux 发行版而异。Ubuntu 存放在 */usr/share/misc/pci.ids*，Fedora 使用 */usr/share/hwdata/pci.ids*，openSUSE 使用 */usr/share/pci.ids*。你的 Linux 的 man 手册应该告诉你具体位置，或者搜索 *pci.ids* 文件（*locate pci.ids*）。

*lspci* 的维护者欢迎提交更新信息；阅读你的 *pci.ids* 文件获取说明。定期运行 *sudo update-pciids* 命令更新 PCI ID 数据库。

PCI 是外围组件互连的缩写。PCI 是一种本地硬件总线，用于计算机中各种硬件设备与 Linux 内核的通信。*lspci* 主要检测控制器、总线和部分个别设备，包括：

+   SATA 控制器

+   音频控制器和设备

+   视频控制器和设备

+   以太网控制器

+   USB 控制器

+   通信控制器

+   以太网控制器

+   RAID 控制器

+   集成的 SD/MMC 卡阅读器

+   PCI FireWire 控制器

多年来出现了几个 PCI 协议。当前标准是 PCIe，即 PCI Express，于 2003 年推出。它与所有传统 PCI 协议向后兼容，取代了 PCI、PCI-X 和 AGP。还记得 AGP，加速图形端口协议吗？AGP 显卡比 PCI 显卡快，因为 AGP 为视频处理提供了专用链路。

PCIe 与早期协议有显著区别，与 AGP 类似，每个设备都有专用链路。旧协议使用共享并行总线，速度较慢。

## 参见

+   *man 8 lspci*

+   *man 8 update-pciids*

# 10.5 理解 lspci 输出

## 问题

*lspci* 的大多数输出都是有意义的，因为它是设备规格。但您想知道每个设备行开头的数字是什么，就像这个例子：

```
$ lspci
[...]
00:1f.2 SATA controller: Intel Corporation 9 Series Chipset Family SATA
Controller [AHCI Mode]
[...]
```

## 解决方案

*00:1f.2* 是设备的 BDF 号，*总线：设备.功能*。总线号为 00，设备号为 1f，功能号为 2。功能号为 2 表示设备有两个功能，每个功能都有自己的 PCI 地址。

使用树状视图查看 PCI 总线与设备之间的关系：

```
$ lspci -tvv
-[0000:00]-+-00.0  Intel Corporation 4th Gen Core Processor DRAM Controller
           +-02.0  Intel Corporation Xeon E3-1200 v3/4th Gen Core Processor
                   Integrated Graphics Controller
           +-03.0  Intel Corporation Xeon E3-1200 v3/4th Gen Core Processor HD
                   Audio Controller
           +-14.0  Intel Corporation 9 Series Chipset Family USB xHCI Controller
           +-16.0  Intel Corporation 9 Series Chipset Family ME Interface #1
           +-19.0  Intel Corporation Ethernet Connection (2) I218-V
           +-1a.0  Intel Corporation 9 Series Chipset Family USB EHCI
                   Controller #2
           +-1b.0  Intel Corporation 9 Series Chipset Family HD Audio Controller
           +-1c.0-[01]--
           +-1c.3-[02-03]----00.0-[03]--
           +-1d.0  Intel Corporation 9 Series Chipset Family USB EHCI
                   Controller #1
           +-1f.0  Intel Corporation H97 Chipset LPC Controller
           +-1f.2  Intel Corporation 9 Series Chipset Family SATA Controller
                   [AHCI Mode]
           \-1f.3  Intel Corporation 9 Series Chipset Family SMBus Controller
```

大多数 PC 通常只有一个 PCI 总线，总是为 00。

## 讨论

在树的根部被括号括起来的零，[0000:00]，标识了*域*和*总线*。前四个零是域号，冒号后的两个零是总线号。域是主机桥。PCI 主机桥将 PCI 控制器连接到 CPU。*域*是 Linux 特有的术语，更常被称为*段组*。您也可以使用*-D*选项查看这一点：

```
$ lspci -D
0000:00:00.0 Host bridge: Intel Corporation 4th Gen Core Processor DRAM
 Controller (rev 06)
0000:00:02.0 VGA compatible controller: Intel Corporation Xeon E3-1200 v3/4th Gen
 Core Processor Integrated Graphics Controller (rev 06)
0000:00:03.0 Audio device: Intel Corporation Xeon E3-1200 v3/4th Gen Core
 Processor HD Audio Controller
[...]
```

在具有多个物理 CPU 的服务器上，您将看到多个主机桥，有时在单个域上有多个总线。

## 参见

+   *man 8 lspci*

# 10.6 过滤 lspci 输出

## 问题

*lspci* 输出了大量信息，您希望筛选出您想要看到的内容。

## 解决方案

使用*awk*命令来削减杂乱。以下示例只找到与 USB 相关的条目：

```
$ lspci -v | awk '/USB/,/^$/'
00:14.0 USB controller: Intel Corporation 9 Series Chipset Family USB xHCI
Controller (prog-if 30 [XHCI])
        Subsystem: ASRock Incorporation 9 Series Chipset Family USB xHCI
Controller
        Flags: bus master, medium devsel, latency 0, IRQ 26
        Memory at efc20000 (64-bit, non-prefetchable) [size=64K]
        Capabilities: <access denied>
        Kernel driver in use: xhci_hcd

00:1a.0 USB controller: Intel Corporation 9 Series Chipset Family USB EHCI
Controller #2 (prog-if 20 [EHCI])
        Subsystem: ASRock Incorporation 9 Series Chipset Family USB EHCI
Controller
        Flags: bus master, medium devsel, latency 0, IRQ 16
        Memory at efc3b000 (32-bit, non-prefetchable) [size=1K]
        Capabilities: <access denied>
        Kernel driver in use: ehci-pci
```

您必须使用从*lspci*输出中显示的类（音频、以太网、USB 等），并注意大小写，因为使用*awk*进行大小写不敏感搜索比较复杂。此示例显示音频控制器和设备：

```
$ lspci -v | awk '/Audio/,/^$/'
00:03.0 Audio device: Intel Corporation Xeon E3-1200 v3/4th Gen Core Processor
HD Audio Controller (rev 06)
        Subsystem: ASRock Incorporation Xeon E3-1200 v3/4th Gen Core Processor
HD Audio Controller
        Flags: bus master, fast devsel, latency 0, IRQ 31
        Memory at efc34000 (64-bit, non-prefetchable) [size=16K]
        Capabilities: <access denied>
        Kernel driver in use: snd_hda_intel
        Kernel modules: snd_hda_intel

00:1b.0 Audio device: Intel Corporation 9 Series Chipset Family HD Audio
Controller
        Subsystem: ASRock Incorporation 9 Series Chipset Family HD Audio
Controller
        Flags: bus master, fast devsel, latency 0, IRQ 32
        Memory at efc30000 (64-bit, non-prefetchable) [size=16K]
        Capabilities: <access denied>
        Kernel driver in use: snd_hda_intel
        Kernel modules: snd_hda_intel
```

根据需要调整详细程度。

您还可以根据供应商、设备或类号选择项目。使用*-nn*选项找到这些数字。在这个例子中，`0300`（用方括号括起来）是类号，`8086` 是供应商号，`0412` 是设备号：

```
$ lspci -nn
[....]
00:02.0 VGA compatible controller [0300]: Intel Corporation
Xeon E3-1200 v3/4th Gen Core Processor Integrated Graphics Controller
[8086:0412] (rev 06)
[...]
```

以下示例分别通过类、供应商和设备进行过滤：

```
$ lspci -d ::0604
00:1c.0 PCI bridge: Intel Corporation 9 Series Chipset Family PCI Express Root
Port 1 (rev d0)
00:1c.3 PCI bridge: Intel Corporation 82801 PCI Bridge (rev d0)
02:00.0 PCI bridge: ASMedia Technology Inc. ASM1083/1085 PCIe to PCI Bridge (rev
03)

$ lspci -d 8086::
00:00.0 Host bridge: Intel Corporation 4th Gen Core Processor DRAM Controller
(rev 06)
00:02.0 VGA compatible controller: Intel Corporation Xeon E3-1200 v3/4th Gen
Core Processor Integrated Graphics Controller (rev 06)
00:03.0 Audio device: Intel Corporation Xeon E3-1200 v3/4th Gen Core Processor
HD Audio Controller (rev 06)
[...]

$ lspci -d :0412:
00:02.0 VGA compatible controller: Intel Corporation Xeon E3-1200 v3/4th Gen
Core Processor Integrated Graphics Controller (rev 06)
```

查找这些数字的另一种方法是在[PCI ID 仓库](https://oreil.ly/f2EKi)中查找它们。

## 讨论

*awk* 是从命令输出或文档中提取特定文本字符串的神奇工具。插入符号*^*是一个正则表达式锚点，匹配字符串的开头，而*$*匹配结尾，所以在这个例子中，*/^$/*查找换行符，文本块开头和结尾的空白。这是从具有在段落之间有空格的源中提取文本块的一个很棒的技巧。

## 参见

+   *man 1 grep*

+   *man 8 lspci*

+   [PCI ID 仓库](https://oreil.ly/f2EKi)

# 10.7 使用 lspci 识别内核模块

## 问题

您想知道您的 PCI 设备正在使用哪些内核模块，以及系统上有哪些可用。

## 解决方案

使用*-k*选项。以下示例仅查询以太网控制器：

```
$ lspci -kd ::0200
00:19.0 Ethernet controller: Intel Corporation Ethernet Connection (2) I218-V
        Subsystem: ASRock Incorporation Ethernet Connection (2) I218-V
        Kernel driver in use: e1000e
        Kernel modules: e1000e
```

您也可以像这个例子一样使用*awk*来查找您的显卡控制器：

```
$ lspci -vmmk| awk '/VGA/,/^$/'
Class:  VGA compatible controller
Vendor: Intel Corporation
Device: Xeon E3-1200 v3/4th Gen Core Processor Integrated Graphics Controller
SVendor:        ASRock Incorporation
SDevice:        Xeon E3-1200 v3/4th Gen Core Processor Integrated Graphics
Controller
Rev:    06
Driver: i915
Module: i915
```

## 讨论

*-k* 选项显示正在使用的内核模块以及每个设备的所有可用内核模块。通常，正在使用的和可用的条目是相同的，但有时会有多个可用模块。

使用 *awk* 时记得添加一些详细信息，否则可能看不到所需的信息。参见 第 10.6 节 的讨论了解 *awk* 的选项。

## 参见

+   *man 1 awk*

+   *man 8 lspci*

# 10.8 使用 lsusb 列出 USB 设备

## 问题

您需要一个快速简便的工具来列出系统上的 USB 设备。

## 解决方案

*lsusb* 列出 USB 总线和连接的 USB 设备，包括鼠标、键盘、USB 闪存驱动器、打印机、智能手机和其他连接的外设。以下两个示例展示了相同设备的两种不同视图。

不带任何选项运行 *lsusb* 可以查看系统上 USB 设备的摘要信息。在下一个示例中，连接了三个外部 USB 设备：键盘、鼠标和无线网络接口：

```
$ lsusb
[...]
Bus 003 Device 011: ID 148f:5372 Ralink Technology, Corp. RT5372 Wireless Adapter
Bus 003 Device 002: ID 0bda:5401 Realtek Semiconductor Corp. RTL 8153 USB 3.0
 hub with gigabit ethernet
Bus 003 Device 006: ID 046d:c018 Logitech, Inc. Optical Wheel Mouse
Bus 003 Device 005: ID 2516:0004 Cooler Master Co., Ltd. Storm QuickFire Rapid
 Mechanical Keyboard
[...]
```

此示例显示了相同内容，以 USB 总线层次结构格式呈现更详细信息，包括内核驱动程序、设备代码和供应商编号以及端口号：

```
$ lsusb -tv
[...]
/:  Bus 03.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/14p, 480M
    ID 1d6b:0002 Linux Foundation 2.0 root hub
    |__ Port 3: Dev 2, If 0, Class=Hub, Driver=hub/4p, 480M
        ID 0bda:5401 Realtek Semiconductor Corp. RTL 8153 USB 3.0 hub with
        gigabit ethernet
    |__ Port 7: Dev 11, If 0, Class=Vendor Specific Class, Driver=rt2800usb, 480M
        ID 148f:5372 Ralink Technology, Corp. RT5372 Wireless Adapter
    |__ Port 11: Dev 5, If 0, Class=Human Interface Device, Driver=usbhid, 1.5M
        ID 2516:0004 Cooler Master Co., Ltd. Storm QuickFire Rapid Mechanical
        Keyboard
    |__ Port 12: Dev 6, If 0, Class=Human Interface Device, Driver=usbhid, 1.5M
        ID 046d:c018 Logitech, Inc. Optical Wheel Mouse
[...]
```

下面的示例展示了插入带有蓝牙接口和连接三星智能手机的外部 USB 集线器时的情况：

```
$ lsusb
[...]
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 003 Device 012: ID 04e8:6860 Samsung Electronics Co., Ltd Galaxy series,
 misc. (MTP mode)
Bus 003 Device 013: ID 0a12:0001 Cambridge Silicon Radio, Ltd Bluetooth Dongle
 (HCI mode)
Bus 003 Device 002: ID 0bda:5401 Realtek Semiconductor Corp. RTL 8153 USB 3.0
 hub with gigabit ethernet
[...]

$ lsusb -tv
[...]
/:  Bus 03.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/14p, 480M
    ID 1d6b:0002 Linux Foundation 2.0 root hub
    |__ Port 3: Dev 2, If 0, Class=Hub, Driver=hub/4p, 480M
        ID 0bda:5401 Realtek Semiconductor Corp. RTL 8153 USB 3.0 hub with
        gigabit ethernet
        |__ Port 4: Dev 12, If 0, Class=Imaging, Driver=, 480M
            ID 04e8:6860 Samsung Electronics Co., Ltd Galaxy series, misc. (MTP
            mode)
        |__ Port 2: Dev 13, If 0, Class=Wireless, Driver=btusb, 12M
            ID 0a12:0001 Cambridge Silicon Radio, Ltd Bluetooth Dongle (HCI mode)
        |__ Port 2: Dev 13, If 1, Class=Wireless, Driver=btusb, 12M
            ID 0a12:0001 Cambridge Silicon Radio, Ltd Bluetooth Dongle (HCI mode)
[...]
```

## 讨论

总线和端口号始终相同。每次插入设备时，设备号都会更改。

例如，ID 号码 `0a12:0001` 是供应商和设备代码。制造商必须向 [*https://usb.org*](https://usb.org) 申请新代码。您可以在 [linux-usb.org](https://oreil.ly/bHLo6) 找到当前 USB ID 列表并贡献更新信息。

类代码也由 [*https://usb.org*](https://usb.org) 管理；参见 [USB 类代码](https://oreil.ly/vNCgT)。我觉得有趣的是，Dev 57，即三星安卓手机，被归类为影像设备。然而，这是有道理的，因为大多数 Linux 发行版使用媒体传输协议（MTP）从安卓手机传输文件。

本节示例来自具有 USB 2.0 和 USB 3.1 端口的 PC。*lsusb* 输出显示设备正在使用的协商速度，因此当您看到类似 `usbhid, 1.5M` 而不是 `480M` 或 `5000M` 时，那是正常的，因为那是键盘，不需要 USB 连接的全部速度。对于存储设备（如 USB 闪存驱动器和外部硬盘），您应该看到更高的速度。

## 参见

+   *man 8 lsusb*

+   [*https://usb.org*](https://usb.org)

+   [*https://oreil.ly/js1oj*](https://oreil.ly/js1oj)

# 10.9 使用 lsblk 列出分区和硬盘

## 问题

您需要一种快速的方法来列出所有连接的存储驱动器及其分区。

## 解决方案

使用 *lsblk* （列出块设备）命令。不带任何选项运行它，以生成计算机上所有块设备的列表：

```
$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0   3.7T  0 disk
├─sda1   8:1    0   476M  0 part /boot
├─sda2   8:2    0  55.9G  0 part /
├─sda3   8:3    0   1.8T  0 part /home
└─sda4   8:4    0   7.5G  0 part [SWAP]
sdb      8:16   0   1.8T  0 disk
├─sdb1   8:17   0   102M  0 part
├─sdb2   8:18   0   6.5G  0 part
├─sdb3   8:19   0   1.1G  0 part [SWAP]
└─sdb4   8:20   0   1.8T  0 part
sdc      8:32   0   3.7T  0 disk
├─sdc1   8:33   0   128M  0 part
├─sdc2   8:34   0 439.7G  0 part
└─sdc3   8:35   0   3.2T  0 part
sdd      8:48   1   3.8G  0 disk
└─sdd1   8:49   1   3.8G  0 part
sr0     11:0    1 159.3M  0 rom
```

显示所选设备上的文件系统标签和 UUID：

```
$ lsblk -f /dev/sdc
NAME   FSTYPE LABEL                 UUID                MOUNTPOINT
sdc
├─sdc1
├─sdc2 ntfs   Seagate Backup Plus   2E203F82203F5057
└─sdc3 ext4   backup                0451d428-9716-4cdd  /media/max/backup
```

仅列出 SCSI 设备及其类型：

```
$ lsblk -S
NAME HCTL       TYPE VENDOR   MODEL             REV TRAN
sda  0:0:0:0    disk ATA      ST4000DM000-1F21 CC54 sata
sdb  2:0:0:0    disk ATA      SAMSUNG HD204UI  0001 sata
sdc  6:0:0:0    disk Seagate  BUP SL           0304 usb
sr0  4:0:0:0    rom  ATAPI    iHAS424   B      GL1B sata
```

## 讨论

*sda*和*sdb*是 SATA 硬盘，*sdc*是 USB 闪存驱动器。在 Linux 上，如 SATA 硬盘和闪存介质等大容量存储设备使用 SCSI 驱动器。*sr0*、*rom*和*ATAPI*标识光盘/DVD 播放器。

定义术语*块设备*而不引发争论，相当困难，因为它是一个编程术语，不易转换为简明的用户界面概念。依我看来，将块设备视为大容量存储设备及其分区最为实用。

`MAJ:MIN`是主设备号和次设备号。主设备号标识类别，例如 8 用于*sd*设备，次设备号按顺序标识每个设备。（运行**`lsblk -l`**以树形结构查看。）

`RM`显示是否为可移动驱动器，1 表示可移动驱动器。

`SIZE`是块设备的大小。

`RO = 0`表示设备不是只读的，1 表示只读。*sr0*，即 CD/DVD 驱动器，是可读写驱动器，但*lsblk*无法告诉您*sr0*中的光盘是否可写。

`TYPE`标识磁盘类型。

`MOUNTPOINT`显示路径，如果设备已挂载。

### 参见

+   *man 8 lsblk*

# 10.10 获取 CPU 信息

## 问题

您想知道系统上有哪些 CPU 及其规格。

## 解决方案

使用无选项运行*lscpu*（列出 CPU）命令：

```
$ lscpu
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              8
On-line CPU(s) list: 0-7
Thread(s) per core:  2
Core(s) per socket:  4
Socket(s):           1
Vendor ID:           GenuineIntel
CPU family:          6
Model:               60
Model name:          Intel(R) Core(TM) i7-4770K CPU @ 3.50GHz
[...]
L1d cache:           128 KiB
L1i cache:           128 KiB
L2 cache:            1 MiB
L3 cache:            8 MiB
[...]
```

这会输出大量信息；您还会看到许多标志，列出功能，并列出 L 缓存信息。

## 讨论

CPU 缓存有三种类型：L1、L2 和 L3。这些是 CPU 上的小型内存缓存。它们非常快，比系统 RAM 快许多倍，并且存储 CPU 下次操作最可能需要的数据。L1 最快且最昂贵，通常最小。L2 其次，成本较低，通常比 L1 大。L3 最慢且成本最低，通常最大。

在上述示例中的 CPU 有四个缓存。使用*-C*选项查看更详细的缓存信息：

```
$ lscpu -C
NAME ONE-SIZE ALL-SIZE WAYS TYPE        LEVEL
L1d       32K     128K    8 Data            1
L1i       32K     128K    8 Instruction     1
L2       256K       1M    8 Unified         2
L3         8M       8M   16 Unified         3

```

这显示了四个缓存，这些缓存由四个物理 CPU 核心共享。L1i 缓存存储 CPU 指令，L1d 缓存存储数据。L2 和 L3 存储数据。

CPU 核心数量有点令人困惑。在这个示例中，`CPU(s): 8`并不意味着有 8 个物理核心；相反，这是 Linux 内核看到的核心数。以下几行讲述了全貌：

```
Thread(s) per core:  2
Core(s) per socket:  4
Socket(s):           1
```

这是一个单处理器，有四个物理核心和每个核心两个线程，总共八个逻辑 CPU。

## 参见

+   *man 1 lscpu*

# 10.11 识别您的硬件架构

## 问题

您不确定机器的硬件架构是什么；可能是 x86-64 或 ARM，您需要确定其具体类型。

## 解决方案

使用*uname*命令。此示例在 x86-64 机器上运行：

```
$ uname -m
x86_64
```

以下列表包含您可能看到的一些常见结果：

+   arm

+   aarch64

+   armv7* (arm7 及以下为 32 位)

+   armv8* (arm8 及以上为 64 位)

+   ia64

+   ppc

+   ppc64

+   s390x

+   sparc

+   sparc64

+   i386

+   i686

+   x86_64

如果设备未运行 Linux，请尝试使用 SystemRescue USB 启动盘引导，然后运行*uname -m*。

你可以在 Chromebook 上安装 Linux。Chromebook 使用 Intel 和 ARM 处理器。查看你的设备信息的一种方法是打开网页浏览器到*chrome://system*。这会显示所有系统信息，可能比你想要的还要多。

更友好的工具是[Cog 系统信息查看器](https://oreil.ly/Yeirk)，它可以在 Chromebook 上显示硬件和网络信息。

## 讨论

Linux 支持比任何其他操作系统更多的硬件架构，从微小的嵌入式系统和 SoC（系统芯片）到大型机和超级计算机，应有尽有。无论你拥有多么冷门的计算硬件，都很有可能有某种 Linux 版本可以运行在上面。

## 参见

+   *man 1 uname*
