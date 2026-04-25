# 环境搭建

## QNX SDP 8.0 + Raspberry Pi 5 环境搭建记录

## 1. 整体目标

本次搭建的目标是：

```text
Windows / WSL2 作为开发机 Host
  ↓
安装 QNX Software Center
  ↓
安装 QNX SDP 8.0
  ↓
安装 Raspberry Pi 5 BSP
  ↓
构建 ifs-rpi5.bin
  ↓
制作 microSD 启动卡
  ↓
树莓派 5 启动 QNX
  ↓
通过 SSH 登录验证
```

这里要区分两个概念：

- `Host`：开发机，用于安装工具链、交叉编译、构建镜像
- `Target`：目标板，也就是 Raspberry Pi 5，真正运行 QNX OS

QNX SDP 本质上是一个交叉编译和调试环境，用于给运行 QNX OS 的目标板构建程序和镜像。

## 2. 注册 myQNX 账号

首先需要注册一个 `myQNX` 账号。

QNX 的免费非商业许可要求先拥有 `myQNX` 账号，然后才能访问 license 和相关下载。注册完成后，用这个账号登录 QNX 官网。

## 3. 申请免费非商业 License

如果只是个人学习、实验、非商业用途，可以申请：

`QNX SDP 8.0 Free Access for Non-Commercial Use`

这个许可面向 `hobbyist`、`student`、`industry professional` 的个人非商业使用。

申请流程大概是：

```text
登录 myQNX
  ↓
进入 QNX SDP 8.0 Free Access 页面
  ↓
接受 non-commercial license 条款
  ↓
获取 license key
```

后续安装 QNX SDP 时，需要用到：

```bash
export USER_NAME="你的 myQNX 邮箱"
export PASSWORD="你的 myQNX 密码"
export LICENSE_KEY="你的 QNX license key"
```

## 4. 下载 QNX Software Center

QNX Software Center 是官方用来下载和管理 QNX SDP、BSP、附加组件的工具。

下载时有两个版本：

- `QNX Software Center - Linux Hosts`
- `QNX Software Center - Windows Hosts`

选择规则：

- Windows 原生安装：选 `Windows Hosts`
- Ubuntu / WSL2 安装：选 `Linux Hosts`

本次是在 WSL2 Ubuntu 里操作，所以下载的是：

`qnx-setup-2.0.4-202501021438-linux.run`

## 5. 在 WSL2 中安装 QNX Software Center

假设安装包在用户目录下：

```bash
cd ~
chmod +x qnx-setup-2.0.4-202501021438-linux.run
./qnx-setup-2.0.4-202501021438-linux.run
```

安装后默认目录类似：

```text
~/qnx/qnxsoftwarecenter
```

检查：

```bash
ls ~/qnx/qnxsoftwarecenter
```

能看到类似内容：

```text
qnxsoftwarecenter
qnxsoftwarecenter_clt
plugins
features
configuration
```

如果 GUI 报错，例如：

```text
gtk_init_check() failed
```

这不影响命令行安装。`qnxsoftwarecenter_clt` 可以从命令行运行 Software Center，并且它和 GUI 使用同一套基础设施。

## 6. 使用命令行添加 License

先设置环境变量：

```bash
export USER_NAME="你的 myQNX 邮箱"
export PASSWORD="你的 myQNX 密码"
export LICENSE_KEY="你的 QNX 免费非商业 license key"
```

添加 license：

```bash
~/qnx/qnxsoftwarecenter/qnxsoftwarecenter_clt \
  -myqnx.user "$USER_NAME" \
  -myqnx.password "$PASSWORD" \
  -addLicenseKey "$LICENSE_KEY"
```

## 7. 安装 QNX SDP 8.0

使用命令行安装 SDP 到 `~/qnx800`：

```bash
~/qnx/qnxsoftwarecenter/qnxsoftwarecenter_clt \
  -myqnx.user "$USER_NAME" \
  -myqnx.password "$PASSWORD" \
  -cleanInstall \
  -mirrorBaseline qnx800 \
  -installBaseline com.qnx.qnx800 \
  -destination ~/qnx800
```

安装完成后加载环境：

```bash
source ~/qnx800/qnxsdp-env.sh
```

正常会看到类似：

```text
QNX_HOST=/home/swartz/qnx800/host/linux/x86_64
QNX_TARGET=/home/swartz/qnx800/target/qnx
MAKEFLAGS=-I/home/swartz/qnx800/target/qnx/usr/include
```

验证编译器：

```bash
qcc
```

如果能正常响应，说明 SDP 环境已经可用。

## 8. 测试交叉编译 QNX 程序

创建一个简单程序：

```c
#include <stdio.h>

int main(void) {
    printf("Hello QNX on Raspberry Pi 5!\n");
    return 0;
}
```

保存为 `hello.c` 后，针对 Raspberry Pi 5 的 AArch64 架构编译：

```bash
qcc -Vgcc_ntoaarch64le hello.c -o hello_qnx
```

检查文件类型：

```bash
file hello_qnx
```

实际输出类似：

```text
hello_qnx: ELF 64-bit LSB pie executable, ARM aarch64,
dynamically linked, interpreter /usr/lib/ldqnx-64.so.2
```

这说明已经成功生成 QNX AArch64 可执行文件。

注意：这个程序不能直接在 WSL 里运行，它要放到真正运行 QNX 的 target 上执行。

## 9. 安装 Raspberry Pi 5 BSP

QNX 官方提供了 Raspberry Pi 5 BCM2712 的 BSP。该 BSP 用于支持 Raspberry Pi 5，并包含预构建二进制、buildfiles，以及用于启动 QNX RTOS 的内容。

先查看可安装包：

```bash
~/qnx/qnxsoftwarecenter/qnxsoftwarecenter_clt \
  -myqnx.user "$USER_NAME" \
  -myqnx.password "$PASSWORD" \
  -destination ~/qnx800 \
  -listAccessible | grep -Ei "raspberry|bcm2712|rpi5"
```

实际看到的关键包：

```text
com.qnx.qnx800.bsp.hw.raspberrypi_bcm2712_rpi5=0.3.0.00381T202512101351L
```

安装 Pi 5 BSP：

```bash
~/qnx/qnxsoftwarecenter/qnxsoftwarecenter_clt \
  -myqnx.user "$USER_NAME" \
  -myqnx.password "$PASSWORD" \
  -destination ~/qnx800 \
  -installPackage com.qnx.qnx800.bsp.hw.raspberrypi_bcm2712_rpi5
```

检查是否安装：

```bash
~/qnx/qnxsoftwarecenter/qnxsoftwarecenter_clt \
  -destination ~/qnx800 \
  -listInstalled | grep -Ei "raspberry|bcm2712|rpi5"
```

查找 BSP zip：

```bash
find ~/qnx800/bsp -iname '*raspberry*' -o -iname '*bcm2712*' -o -iname '*rpi5*'
```

## 10. 解压 BSP 到工作目录

不建议直接在 `~/qnx800` 里改 BSP，单独放一个工作目录。

本次使用：

```bash
mkdir -p ~/qnx_bsp
cd ~/qnx_bsp
```

找到 BSP zip：

```bash
BSP_ZIP=$(find ~/qnx800/bsp -iname '*raspberry*bcm2712*rpi5*.zip' | head -n 1)
echo "$BSP_ZIP"
```

解压：

```bash
unzip "$BSP_ZIP" -d ./bsp-rpi5
```

查找 `images` 目录：

```bash
find ~/qnx_bsp/bsp-rpi5 -maxdepth 4 -type d -name images
```

实际结果：

```text
/home/swartz/qnx_bsp/bsp-rpi5/images
/home/swartz/qnx_bsp/bsp-rpi5/binary_files_with_symbols/images
```

使用第一个作为 BSP 构建入口：

```bash
export BSP_ROOT_DIR=/home/swartz/qnx_bsp/bsp-rpi5
```

## 11. 构建 Raspberry Pi 5 QNX 启动镜像

加载 SDP 环境：

```bash
source ~/qnx800/qnxsdp-env.sh
```

清理旧产物：

```bash
cd "$BSP_ROOT_DIR/images"
make clean
```

回到 BSP 根目录构建：

```bash
cd "$BSP_ROOT_DIR"
make
```

`$BSP_ROOT_DIR/images/Makefile` 中的 `all` 目标用于构建 QNX IFS，`ifs-rpi5.bin` 是 Raspberry Pi bootloader 通过 `config.txt` 加载的 IFS 文件。

检查产物：

```bash
ls -lh "$BSP_ROOT_DIR/images/ifs-rpi5.bin"
```

实际生成结果：

```text
-rw-rw-r-- 1 swartz swartz 39M Apr 24 21:37 /home/swartz/qnx_bsp/bsp-rpi5/images/ifs-rpi5.bin
```

这说明 Pi 5 的 QNX 启动镜像已经构建完成。

## 12. 准备 microSD 启动卡

这里容易误解：

- `ifs-rpi5.bin` 不是完整 SD 卡镜像
- 不能直接在 Raspberry Pi Imager 里选择“自定义镜像”烧录

正确做法是：

```text
先用 Raspberry Pi Imager 烧录 Raspberry Pi OS
  ↓
获得正常的 boot 分区和 firmware 文件
  ↓
再把 ifs-rpi5.bin 拷进去
  ↓
修改 config.txt
```

QNX IFS image 需要复制到 microSD 中，并且文件名必须是 `ifs-rpi5.bin`，因为它是 `config.txt` 中定义的启动镜像名。

## 13. 修改 config.txt

在 microSD 的 `boot` 分区里，确保有：

```text
config.txt
ifs-rpi5.bin
*.dtb
overlays/
start*.elf
fixup*.dat
```

在 `config.txt` 末尾加入：

```ini
[rpi5]
enable_uart=1
force_turbo=1
kernel=ifs-rpi5.bin
```

关键是：

```ini
kernel=ifs-rpi5.bin
```

它告诉 Raspberry Pi firmware 加载 QNX 的 IFS 镜像，而不是 Linux kernel。

## 14. 启动 Raspberry Pi 5

将 microSD 插入 Raspberry Pi 5。

推荐硬件连接：

- Pi 5 电源
- Pi 5 网线接路由器
- 可选 HDMI 显示器
- 可选串口调试线

本次没有 Debug Probe，也没有 USB-TTL 串口线，所以采用：

`有线网口 + 路由器 DHCP + SSH`

启动后，在路由器后台查看新设备 IP。

本次获取到的 IP：

`192.168.3.153`

## 15. SSH 登录 QNX

在 Windows CMD / PowerShell 中执行：

```bash
ssh qnxuser@192.168.3.153
```

密码：

`qnxuser`

登录成功后看到：

```text
$
$
$
```

说明已经进入远端 shell。

## 16. 验证系统是否为 QNX

登录后执行：

```bash
uname -a
```

或者：

```bash
uname -s
```

如果输出包含：

`QNX`

说明当前系统就是 QNX。

还可以执行：

```bash
pidin
```

`pidin` 是 QNX 中常用的进程和系统查看工具。

也可以查看：

```bash
ls /proc/boot
```

QNX 的启动镜像内容通常可以在 `/proc/boot` 下看到。

推荐验证命令：

```bash
uname -a
uname -s
uname -m
pidin info
ls /proc/boot
```

## 17. 后续如何运行自己的程序

之前在 WSL 中编译出来的程序是：

`hello_qnx`

它是 QNX/AArch64 可执行文件。

后续流程是：

```text
WSL 中交叉编译
  ↓
通过 scp / sftp / qconn / SD 卡传到 Pi 5
  ↓
在 QNX shell 中运行
```

例如：

```bash
scp hello_qnx qnxuser@192.168.3.153:/tmp/
```

登录 QNX 后：

```bash
cd /tmp
chmod +x hello_qnx
./hello_qnx
```

如果输出：

```text
Hello QNX on Raspberry Pi 5!
```

就说明从编译到 target 运行全部打通。
