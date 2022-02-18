+++
title = "NAS Build 2021"
publishDate = 2021-04-26T00:00:00+08:00
lastmod = 2022-02-16T16:45:36+08:00
tags = ["homelab"]
draft = false
toc = true
+++

## TL;DR {#tl-dr}

-   B550 + Ryzen 3 Pro 4350G 的组合几乎是性价比最高的全新件 ECC memory 解决方案, 后者的 7nm 工艺也带来了较低的 idle 功耗和不错的单核性能
-   由 TrueNAS Core 带来原生 zfs, 收获 snapshot 等各种迷人的功能, 同时避开 btrfs 的无数大坑
-   采用 stripe mirrored vdevs, 兼顾可靠性 (包括重建速度) 、可用率、性能和可扩展性
-   用 X710-DA2 顺便完成了简单的的万兆网络搭建


## 需求 {#需求}

我现在使用的 NAS 是 2016 年底组的性能羸弱的 J3455 方案, 后来只在半年后更换了机箱为 inwin MS04, 整体已经服役了有 4 年. 各种 config 和使用都比较随意:

-   系统一直是直接跑的 Linux, 然后手动用 Samba 等工具来实现基本的分享功能
-   其中两块盘做了硬件的 RAID-1 放一些重要的资料, 剩下两块则是赤裸工作的
-   因为 I/O 性能比较弱, 对大一点文件的操作一般都是直接 ssh 上去做的 (大部分时候是 vifm), 虽然效率挺高的但是就感觉自己很 savage...

四年里面有过很多次都想升级一下这台机器, 但是都因为各种原因而搁置了, 这次总算是下定决心. 而我对 NAS 的需求其实很简单和纯粹:

-   因为其他的很多功能如 Plex Server, Bitwarden 等都扔给几台 NUC 的 Cluster 跑了, 因此我只需要一台只用于存储的服务器, 让存储和其他功能解耦, 拒绝 all-in-one
-   支持万兆. 这几年来 NAS 的瓶颈一直浇灭了上万兆的热情, 这次总算可以解决了. 同时这几年里面各种新设备的推出使得部署简单的万兆网络的成本和折腾程度已经大幅下降了
-   尽量使用全新件, 不捡洋垃圾. 毕竟是存储相关的东西, 所谓「稳定压倒一切」
    -   同样为了稳定, 需要上 ECC


## 系统选择 {#系统选择}

很早就想好了要用 FreeNAS/TrueNAS, 没有太多的去了解 OMV 等其他系统. zfs 有很多非常美好的特性:

-   copy-on-write
-   snapshot
-   RAID-Z

另外 [Jeff Bonwick](https://blogs.oracle.com/bonwick/128-bit-storage:-are-you-high) 关于 zfs 的这句话也令我深深震撼:

-   _fully populating a 128-bit storage pool would, literally, require more energy than boiling the oceans._

至于是选 TrueNAS Core (BSD) 还是 TrueNAS Scale (Linux):

-   Scale 还不是太稳定
-   我并不需要 Scale 带来的扩展性的优势
-   BSD 是 Bill Joy 在 Berkeley 时期所写, zfs 是他的 Sun 公司所制, 什么叫一脉相承啊(后仰) ! 用 Linux 来跑那显然是异端嘛 !


## 硬件 {#硬件}

先放最终的配置单:

| component      | model                       |
|----------------|-----------------------------|
| Case           | Silverstone CS381           |
| Motherboard    | ASUS TUF GAMINGB550M-PLUS   |
| CPU            | AMD Ryzen Pro 4350G         |
| RAM            | Micron 32G DDR4 ECC \* 2    |
| HBA            | LSI 9300-8i                 |
| Network        | Intel X710-DA2              |
| Power Supply   | Silverstone ST45SF-G        |
| USB Stick (OS) | SanDisk CZ430 32GB \* 3     |
| SSD (SLOG)     | Intel Optane M1 16G \* 2    |
| SSD (L2ARC)    | SAMSUNG 980 EVO 1T          |
| CPU Cooler     | NOCTUA NH-L9a-AM4           |
| Fan            | NOCTUA NF-A12x25 FLX \*2    |
| Cables         | mini-SAS SFF-8643 0.5m \* 2 |


### CS381 {#cs381}

机箱本来是想直接用同样是 Silverstone 家的 rackmount 机箱 RM21-308, 2U 且深度只有 48cm, 但是考虑到我现在暂时还没法完全隔绝生活环境和 NAS 放置的环境, rackmount 机箱风扇的噪音可能比较难解决, 因此求稳就还是先算了. 不过对于 CS381 这个机箱我其实不是很满意, 一是体积比我想象中大很多, 看起来接近两个并排的 MS04; 二是自带的机箱风扇噪音也比较大, 迫使我购入了两个 NF-A12 来替换. <br />


### B550 + 4350G: 高性价比 ECC 方案 {#b550-plus-4350g-高性价比-ecc-方案}

主板和 CPU 方面, 一块普通的 B550 加上 4350G 应该算是性价比非常高的 ECC 支持方案. 本来是想用 3100 或者是 3300X 之类的非 APU, 但是这样就最好上 X470D4U, X570D4U 这样带 IPMI/BMC 的主板来避免无 GPU 启动可能会碰到的问题, 由于在这台机器上我对 IPMI 的需求比较弱, 因此贵出的价格不是很划算, 经过纠结还是 fallback 回了现在的方案. 值得注意的是 APU 里面只有 Ryzen Pro 系列才支持 ECC, 比如 4000 系列只有 4350G 和 4750G, 因此选择面相对要小很多. <br />


### 使用 U 盘作为系统盘 {#使用-u-盘作为系统盘}

因为 FreeNAS 很追求一个部件做一件事情, 因此系统盘不能用来做存储或是 Cache, 选择两三个物理体积小巧的 U 盘来 mirror 系统应该是比较划算的选择. 这里选的 CZ430 插入背部 USB 口之后几乎没有凸出, 很自然. 另外其实 16G 肯定就已经够了, 但是 32G 只贵一块钱就还是买了 32G 的... <br />


### SLOG/ZIL {#slog-zil}

参考 sth 的这篇文章 [What is the ZFS ZIL SLOG and what makes a good one](https://www.servethehome.com/what-is-the-zfs-zil-slog-and-what-makes-a-good-one/), 在 zfs 中, SLOG/ZIL 实际上是用于解决 synchronous write 时的性能问题的. 在 synchronous write 下系统会等待写入完成的 acknowledge 然后再进行接下来的写入, 此时如果有断电或是系统和网络崩溃等情况就可以避免数据丢失. 而 ZIL 会在写入前 log 所有要进行的写入操作, 完成了对 ZIL 的写入之后就会视为完成了向 persistent storage 的写入, 因此在逻辑上也可以认为是写入的 cache. 基于这样的行为, Optane 这样有极高写入速度的设备自然是最好的选择. 参考 [Exploring the Best ZFS ZIL SLOG SSD with Intel Optane and NAND](https://www.servethehome.com/exploring-best-zfs-zil-slog-ssd-intel-optane-nand/), Optane 作为 SLOG 有着明显更好的延迟表现. 16G 或是 32G 的 M10/M15 现在都价格很低, 很适合用两条来 mirror 做 SLOG. <br />


### L2ARC {#l2arc}

至于 L2ARC, 根据 [Brendan Gregg 在博客](https://www.brendangregg.com/blog/2008-07-22/zfs-l2arc.html)里面的说法, 就是 Level 2 的 ARC, 相比 ZIL 提供了另一维度的真 cache. 如果读取的场景有很多重复的热数据, 同时 ARC (DRAM) 的空间不够, 就可以添加 L2ARC 来帮助. 所以实际上在我 RAM 有 64G 同时没有大规模热数据的情况下已经不是很需要 L2ARC 了, 不过我手上有一条闲置的 980 Evo, 就先插上用了. 另外值得注意的是 L2ARC 的行为和 ARC 本身基本一样, 所以即使介质是 persistent storage 也会在重启后清除, 因此维持长的 uptime 在这里是很有意义的. <br />
