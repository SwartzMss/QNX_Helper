# 常用命令

这页记录当前这套 `QNX SDP 8.0 + Raspberry Pi 5` 环境下实际使用过的常用命令，并尽量附上真实输出示例。

## 1. 查看系统信息

### `uname -a`

用于查看系统名称、版本、构建时间和目标平台信息。

```bash
uname -a
```

示例输出：

```text
QNX localhost 8.0.0 2026/02/27-10:59:13EST RaspberryPi5 aarch64le
```

这个输出可以直接看出几个关键信息：

- 当前系统是 `QNX`
- 版本是 `8.0.0`
- 平台是 `RaspberryPi5`
- 架构是 `aarch64le`

### `uname -m`

用于查看目标机器平台标识。

```bash
uname -m
```

示例输出：

```text
RaspberryPi5
```

## 2. 查看进程和线程

### `pidin`

这是 QNX 里最常用的系统查看命令之一，用于查看进程、线程和状态信息。

```bash
pidin
```

示例输出：

```text
     pid tid name                         prio STATE          Blocked
       1   1 /proc/boot/procnto-smp-instr   0f RUNNING
       1   2 /proc/boot/procnto-smp-instr   0f READY
       1   3 /proc/boot/procnto-smp-instr   0f RUNNING
       1   4 /proc/boot/procnto-smp-instr   0f RUNNING
       1   5 /proc/boot/procnto-smp-instr 255i INTR
       1   6 /proc/boot/procnto-smp-instr 255i INTR
       1   7 /proc/boot/procnto-smp-instr 255i INTR
       1   8 /proc/boot/procnto-smp-instr 255i INTR
```

常见用途：

- 看进程是否存在
- 看系统当前在运行什么
- 排查程序是否还活着

### `pidin info`

用于查看系统整体信息，包括 CPU、内存和进程线程数量。

```bash
pidin info
```

示例输出：

```text
CPU:AARCH64 Release:8.0.0  FreeMem:3736MB/4032MB BootTime:Jan 01 00:00:00 GMT 1970
Processes: 26, Threads: 131
Processor1: 1095749809 Cortex-A76 2400MHz FPU
Processor2: 1095749809 Cortex-A76 2400MHz FPU
Processor3: 1095749809 Cortex-A76 2400MHz FPU
Processor4: 1095749809 Cortex-A76 2400MHz FPU
```

这个命令很适合快速确认：

- 当前架构
- 系统版本
- 剩余内存
- CPU 核心数量
- 当前进程和线程总数

### `pidin mem`

用于查看和内存相关的进程信息。

```bash
pidin mem
```

示例输出：

```text
     pid tid name                         prio STATE                      code  data        stack
       1   1 /proc/boot/procnto-smp-instr   0f READY
       1   2 /proc/boot/procnto-smp-instr   0f READY
       1   3 /proc/boot/procnto-smp-instr   0f RUNNING
       1   4 /proc/boot/procnto-smp-instr   0f READY
       1   5 /proc/boot/procnto-smp-instr 255i INTR
       1   6 /proc/boot/procnto-smp-instr 255i INTR
       1   7 /proc/boot/procnto-smp-instr 255i INTR
       1   8 /proc/boot/procnto-smp-instr 255i INTR
       1   9 /proc/boot/procnto-smp-instr 254i INTR
```

常见用途：

- 粗略查看进程内存相关信息
- 排查内存占用异常时作为入口

### `pidin threads`

用于查看线程信息。

```bash
pidin threads
```

示例输出：

```text
     pid tid name                         thread name          STATE          Blocked
       1   1 /proc/boot/procnto-smp-instr 1                    READY
       1   2 /proc/boot/procnto-smp-instr 2                    READY
       1   3 /proc/boot/procnto-smp-instr 3                    RUNNING
       1   4 /proc/boot/procnto-smp-instr 4                    READY
       1   5 /proc/boot/procnto-smp-instr 5                    INTR
       1   6 /proc/boot/procnto-smp-instr 6                    INTR
       1   7 /proc/boot/procnto-smp-instr 7                    INTR
       1   8 /proc/boot/procnto-smp-instr 8                    INTR
       1   9 /proc/boot/procnto-smp-instr 9                    INTR
```

常见用途：

- 程序卡住时看线程状态
- 判断线程是不是阻塞了

## 3. 查看网络信息

### `ifconfig`

用于查看网卡状态、IP 地址、链路信息等。

```bash
ifconfig
```

示例输出：

```text
enc0: flags=0<> metric 0 mtu 1536
	groups: enc
	nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 16384
	options=680003<RXCSUM,TXCSUM,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
	inet 127.0.0.1 netmask 0xff000000
	inet6 ::1 prefixlen 128
	inet6 fe80::1%lo0 prefixlen 64 scopeid 0x2
	groups: lo
	nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
pflog0: flags=0<> metric 0 mtu 33144
	groups: pflog
pfsync0: flags=0<> metric 0 mtu 1500
	syncpeer: 0.0.0.0 maxupd: 128 defer: off
	syncok: 1
	groups: pfsync
cgem0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
	options=68038b<RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWCSUM,TSO4,TSO6,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
	ether 2c:cf:67:97:8e:84
	inet6 fe80::2f28:6e04:6a0b:5912%cgem0 prefixlen 64 scopeid 0x5
	inet6 240e:390:9ea:ac52:2925:e7cd:c212:30df prefixlen 64 autoconf
	inet6 240e:390:9ea:ac50:4cd1:a114:7d7d:6 prefixlen 128
	inet 192.168.3.153 netmask 0xffffff00 broadcast 192.168.3.255
	media: Ethernet autoselect (1000baseT <full-duplex>)
	status: active
	nd6 options=1<PERFORMNUD>
```

从这个输出里可以看出：

- 当前有线网卡是 `cgem0`
- IPv4 地址是 `192.168.3.153`
- 链路状态是 `active`
- 当前网口工作在 `1000baseT full-duplex`

### `netstat -an`

用于查看网络连接和监听端口。

```bash
netstat -an
```

示例输出：

```text
Active Internet connections (including servers)
Proto Recv-Q Send-Q Local Address          Foreign Address        (state)
tcp4       0    160 192.168.3.153.22       192.168.3.196.65329    ESTABLISHED
tcp4       0      0 *.8000                 *.*                    LISTEN
tcp6       0      0 *.8000                 *.*                    LISTEN
tcp4       0      0 *.22                   *.*                    LISTEN
tcp6       0      0 *.22                   *.*                    LISTEN
udp4       0      0 *.*                    *.*
udp4       0      0 *.68                   *.*
udp6       0      0 *.546                  *.*
udp4       0      0 *.*                    *.*
udp6       0      0 *.*                    *.*
icm6       0      0 *.*                    *.*
```

从这里可以看到：

- SSH 的 `22` 端口正在监听
- `8000` 端口也在监听
- 当前有一个 SSH 连接已经建立：`192.168.3.196 -> 192.168.3.153:22`

### `netstat -rn`

用于查看路由表。

```bash
netstat -rn
```

示例输出：

```text
Routing tables

Internet:
Destination        Gateway            Flags         Netif Expire
default            192.168.3.1        UG            cgem0
127.0.0.1          link#2             UH              lo0
192.168.3.0/24     link#5             U             cgem0
192.168.3.153      link#5             UHS             lo0

Internet6:
Destination                       Gateway                       Flags         Netif Expire
default                           fe80::4ed1:a1ff:fe14:7d7d%cgem0 UG          cgem0
::1                               link#2                        UHS             lo0
240e:390:9ea:ac50:4cd1:a114:7d7d:6 link#5                       UHS             lo0
240e:390:9ea:ac52::/64            link#5                        U             cgem0
240e:390:9ea:ac52:2925:e7cd:c212:30df link#5                    UHS             lo0
fe80::%lo0/64                     link#2                        U               lo0
fe80::1%lo0                       link#2                        UHS             lo0
fe80::%cgem0/64                   link#5                        U             cgem0
fe80::2f28:6e04:6a0b:5912%cgem0   link#5                        UHS             lo0
```

从这里可以确认：

- IPv4 默认网关是 `192.168.3.1`
- 默认出口网卡是 `cgem0`
- 本机当前处于 `192.168.3.0/24` 网段

## 4. 查看挂载和存储

### `mount`

用于查看当前挂载情况。

```bash
mount
```

示例输出：

```text
ifs on / type ifs
/dev/fs9p0 on /var type flash
/dev/shmem on /dev/shmem type shmem
```

这个输出说明：

- 根文件系统 `/` 是 `ifs`
- `/var` 挂载在 flash 设备上
- `/dev/shmem` 是共享内存文件系统

### `df -h`

用于查看文件系统容量和使用情况。

```bash
df -h
```

示例输出：

```text
ifs                          39M       39M         0     100%  /
/dev/fs9p0                   16M      114K       16M       1%  /var
/dev/sd0                     29G       29G         0     100%  /dev/sd0t131
/dev/sd0                    512M      512M         0     100%  /dev/sd0t12
/dev/sd0                     30G       30G         0     100%
/dev/shmem                     0         0         0     100%  (/dev/shmem)
```

这里要注意，`ifs` 显示为 `100%` 是正常现象，因为它本身就是启动镜像内容，不等同于普通可写磁盘分区。

## 5. 查看启动镜像内容

### `ls /proc/boot`

用于查看启动镜像里带了哪些文件。

```bash
ls /proc/boot
```

示例输出：

```text
customize_startup.sh  fan_start.sh  i2c_start.sh  ldqnx-64.so  ldqnx-64.so.2  net_start.sh  pci_server_start.sh  procnto-smp-instr  startup-script  uart2_console.sh  uart4_console.sh  usb_start.sh
```

这个目录很有代表性，因为 QNX 启动镜像里的很多核心内容都可以在这里看到，例如：

- `procnto-smp-instr`
- `startup-script`
- `ldqnx-64.so`
- 各类启动脚本
