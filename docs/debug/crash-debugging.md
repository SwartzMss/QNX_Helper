# Crash 调试

## 1. 最小 crash 示例

先准备一个最小示例程序，用来验证 crash、core 生成和后续分析流程。

```c
#include <stdio.h>

int main(void) {
    printf("about to crash\n");
    int *p = NULL;
    *p = 1;
    return 0;
}
```

保存为 `crash_demo.c`，然后在 Host 上编译：

```bash
qcc -g -Vgcc_ntoaarch64le crash_demo.c -o crash_demo
```

这里建议带上 `-g`，这样后面用 GDB 分析 core 时可以看到更完整的符号信息。

把程序传到 Target：

```bash
scp crash_demo qnxuser@192.168.3.153:/tmp/
```

在 Target 上运行：

```bash
cd /tmp
chmod +x crash_demo
./crash_demo
```

这个程序会稳定触发一次空指针写入，用来验证：

- crash 现场输出
- core 是否生成
- `coreinfo` 是否能读
- `ntoaarch64-gdb` 是否能正常解析

## 2. 前置检查

先确认 target 上有没有 `dumper`：

```bash
which dumper
pidin | grep dumper
```

如果 `pidin | grep dumper` 没输出，就手工启动。

当前这台 Raspberry Pi 5 的实际结果：

```bash
$ which dumper
/usr/sbin/dumper
```

```bash
$ pidin | grep dumper
   20484   1 usr/sbin/dumper               10r RECEIVE        1
   20484   2 usr/sbin/dumper               10r RECEIVE        2
```

```bash
$ ls -l /proc/dumper
nrw-r-----  1 root root 0 1970-01-01 22:20 /proc/dumper
```

这说明当前系统里：

- `dumper` 已经安装，路径是 `/usr/sbin/dumper`
- `dumper` 已经在后台运行
- `/proc/dumper` 控制入口也存在

这里的 `dumper` 可以理解成：

QNX 里专门负责接收进程 crash 事件并生成 core 文件的后台服务进程。

它主要负责三件事：

1. 程序 crash 后接管现场并生成 core
2. 响应手工 dump 请求
3. 管理 core 的输出目录和命名方式

`RECEIVE` 是正常状态，表示它正在后台等待请求，不是异常。

当前还额外确认到一件事：

```bash
$ cat /proc/20484/cmdline
dumper
```

这说明当前 `dumper` 看起来是默认参数启动，没有额外带 `-d` 之类的目录参数。

## 3. 当前这台板子的默认 core 生成行为

在这台 Raspberry Pi 5 上，实际验证结果是：

- 不需要先手工创建 `/var/dumps`
- 程序 crash 后，core 会直接生成在当前用户的默认目录里

最小示例实际输出：

```bash
$ ./crash_demo
about to crash

Process 50257946 (crash_demo) terminated SIGSEGV code=1 fltno=11 ip=0000001e6272e7f4 mapaddr=00000000000007f4 ref=0000000000000000
Memory fault (core dumped)
```

然后直接可以看到：

```bash
$ ls
crash_demo  crash_demo.core  fs9  pci  slogger2  ssh_host_ecdsa_key  ssh_host_ecdsa_key.pub  ssh_host_ed25519_key  ssh_host_ed25519_key.pub  ssh_host_rsa_key  ssh_host_rsa_key.pub  sshd_config
```

以及：

```bash
$ ls -lt crash_demo*
-rw-------  2 root    root    1630208 1970-01-02 19:58 crash_demo.core
-rwxr-xr-x  2 qnxuser qnxuser   10184 1970-01-02 19:57 crash_demo
```

## 4. 如果想固定输出目录，再重启 `dumper`

如果你不想让 core 落在默认位置，而是想统一保存到固定目录，例如 `/var/dumps`，这时才需要手工指定。

先准备 core 目录：

```bash
mkdir -p /var/dumps
```

启动：

```bash
dumper -d /var/dumps -u &
```

常用变体：

```bash
dumper -d /var/dumps -n &
dumper -d /var/dumps -N 10 &
dumper -v -d /var/dumps -u &
```

含义：

- `-d /var/dumps`：指定 core 输出目录
- `-u`：文件名带时间戳
- `-n`：顺序编号
- `-N 10`：最多保留 10 份
- `-v`：显示详细输出

如果像当前这台板子一样，`dumper` 已经在后台运行，就不要重复起多个实例。先确认它当前的启动方式，再决定是否需要重启并带上新的参数。

## 5. crash 后确认 core 是否生成

```bash
ls -l
```

常见文件名：

```text
app.core
app-YYYYMMDD_HHMMSS_MSEC.core
app.1.core
```

如果没有 core：

1. 先确认 `dumper` 是否真的在跑
2. 先在当前目录、当前用户目录和 `/tmp` 里找
3. 如果你自己给 `dumper` 指定过 `-d`，再去对应目录找
4. 再考虑是否需要重启 `dumper`

## 6. 手工导出指定进程的 core

先查进程号：

```bash
pidin | grep app
```

然后导出：

```bash
dumper -p 1234
```

这个命令适合：

- 程序还没 crash，但想先抓现场
- 想先验证 dump 流程是否正常

## 7. 通过 `/proc/dumper` 主动触发

如果你想用 `/proc/dumper` 这种方式，才需要重新带 `-a` 启动：

```bash
dumper -a -d /var/dumps -u &
```

再触发：

```bash
echo 1234 > /proc/dumper
```

## 8. 一起保存哪些文件

至少一起保存：

- core 文件
- 当时运行的可执行文件
- 对应版本的符号文件
- 相关日志

不要只拿 `core`。

## 9. 把 core 从 target 拷回 host

例如：

```bash
scp qnxuser@192.168.3.153:/path/to/app.core .
```

如果程序文件也在 target 上，也一起拷回：

```bash
scp qnxuser@192.168.3.153:/path/to/app .
```

对于最小示例，就是：

```bash
scp qnxuser@192.168.3.153:/path/to/crash_demo.core .
```

## 10. 先用 `coreinfo`

先确认 core 内容对不对：

```bash
coreinfo -i ./app.core
coreinfo -t ./app.core
coreinfo -m ./app.core
coreinfo -s ./app.core
```

常用目的：

- `-i`：看进程基本信息
- `-t`：看线程信息
- `-m`：看内存映射
- `-s`：看系统信息

## 11. 在 Host 上分析 core

这种方式更适合当前这套开发环境。

基本流程是：

1. 在 target 上生成 core
2. 把 core 拷回 host
3. 在 host 上用 QNX 提供的调试工具分析

示例：

```bash
scp qnxuser@192.168.3.153:/path/to/app.core .
scp qnxuser@192.168.3.153:/path/to/app .
```

然后在 host 上执行：

```bash
coreinfo -i ./app-*.core
coreinfo -t ./app-*.core
ntoaarch64-gdb ./app ./app-*.core
```

这种方式的优点：

- 不占用 target 上的调试环境
- 分析时更方便保存输出
- 更适合长期排查和反复看调用栈

## 12. 在 Target 上分析 core

如果 target 上也带了相关调试工具，也可以直接在 target 上看 core。

先确认工具是否存在：

```bash
which coreinfo
which ntoaarch64-gdb
```

如果存在，就可以直接在 target 上执行：

```bash
coreinfo -i /path/to/app.core
coreinfo -t /path/to/app.core
ntoaarch64-gdb /path/to/app /path/to/app.core
```

这种方式的优点：

- 不需要先把 core 拷回 host
- 适合现场快速确认 crash 线程和 backtrace

这种方式的限制：

- target 上不一定带全调试工具
- target 资源通常更紧张
- 交互体验通常不如 host

## 13. 用 `ntoaarch64-gdb` 解析 core

Raspberry Pi 5 这种 AArch64 目标：

```bash
ntoaarch64-gdb ./app ./app.core
```

进入 GDB 以后先执行：

```gdb
bt
info threads
thread apply all bt
frame 0
list
info locals
info args
```

先看：

1. `bt`
2. `thread apply all bt`
3. `frame 0`
4. `info locals`

## 14. 一套最小流程

### target

```bash
./app
ls -l
```

如果程序没 crash，但要手工导：

```bash
pidin | grep app
dumper -p <pid>
```

### host 分析

```bash
scp qnxuser@192.168.3.153:/path/to/app.core .
coreinfo -i ./app-*.core
coreinfo -t ./app-*.core
ntoaarch64-gdb ./app ./app-*.core
```

### target 分析

```bash
coreinfo -i /path/to/app.core
coreinfo -t /path/to/app.core
ntoaarch64-gdb /path/to/app /path/to/app.core
```

### GDB 内

```gdb
bt
thread apply all bt
frame 0
info locals
info args
```

## 15. 常见检查点

如果流程走不通，先查这几个点：

- `dumper` 是否真的在跑
- core 实际默认落到了哪里
- core 是否真的生成了
- host 上的二进制是否和 target 崩溃时的版本一致
- 符号是否保留
