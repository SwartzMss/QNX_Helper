# 安全策略（secpol）

`secpol` 是 QNX 安全能力里非常核心的一块。它对应的是 **security policy**，也就是“系统安全策略”。

如果前面学 QNX 更像是在理解微内核、消息传递、resource manager 和启动镜像，那么 `secpol` 这部分更像是在回答另一个问题：

> 系统里的每个长期运行进程，到底被允许做什么，不被允许做什么？

QNX 的做法不是只靠传统的 `root/non-root` 区分，而是引入 **type + rule + ability** 这一套策略机制，把权限控制细化到进程类型、对象类型和具体能力上。

## 1. `secpol` 到底是什么

可以先把它理解成两层：

- **security policy**：策略本身，定义“谁可以做什么”
- **`secpol` 工具**：用于查看和校验已编译策略文件的目标端工具

QNX 的安全策略文件会定义：

- 进程使用什么 `type`
- 这个 `type` 允许使用哪些 `ability`
- 它能连接哪些 channel
- 它能在哪些路径上 attach
- 它和其他类型的对象之间允许发生什么交互

QNX 官方文档明确说明：**系统没有默认安全策略**。也就是说，是否启用、怎么划分类型、每个服务给什么权限，都需要系统集成人员自己设计和维护。

## 2. 为什么它重要

在不启用策略时，系统里很多进程默认约束比较弱，更多依赖传统权限模型。

启用策略后，系统会进入一种更明确的“白名单”模式：

- 一个服务只能做策略允许的事
- 即使进程被攻破，也更难横向扩散
- 系统服务之间的边界更清楚
- 长期运行的 daemon / resource manager 更容易收敛权限

这也是 QNX 在车载、工业、关键嵌入式场景里经常被提到的原因之一：它不只是“能跑”，而是强调 **可控边界**。

## 3. 理解 `secpol` 的几个关键概念

### `type`

`type` 是策略里的核心标签。一个进程启动时可以被赋予某个安全类型，例如：

- `slogger2_t`
- `pipe_t`
- `devb_t`
- `console_t`

策略不是直接写“`/proc/boot/slogger2` 可以干什么”，而是写“`slogger2_t` 可以干什么”。

这带来两个直接好处：

- 规则更清晰
- 同类行为可以复用

但 QNX 官方也特别强调：**长期运行的服务最好一进程一类型，不要多个长期服务共用一个 type**。否则边界会被冲淡。

### `ability`

`ability` 可以理解成 QNX 里的“特权能力”。它比传统 Unix 权限更细。

例如某个服务是否允许：

- 管理 path space
- 连接某类 channel
- 做某些特权操作

策略的核心工作之一，就是限制每个服务实际拥有的 `ability`。

### `rule`

规则就是策略条目，用来表达：

- 哪个 `type` 可以 attach 到哪个路径
- 哪个 `type` 可以连到哪个对象类型
- 哪些能力被允许，哪些要拒绝

例如官方文档里给出的示意规则包括：

```text
allow_attach random_t /dev/random;
allow random_t devb_t : channel connect;
```

不用先把语法细节背下来。初学阶段更重要的是先理解：**规则最终是在限制进程和系统对象之间的交互边界**。

## 4. `secpol` 相关工具分别干什么

这一组工具很容易混在一起，最好分开记：

### `secpol`

目标端工具，用来查看和校验 **已编译的策略文件**。

常见用法：

```bash
secpol -v
secpol -d
secpol -c
```

用途分别可以粗略理解成：

- `-v`：校验策略
- `-d`：看编译后策略里的 blob 列表
- `-c`：看更详细的策略内容

### `secpolgenerate`

目标端工具，用来 **生成初始策略**。这通常是开发阶段最常用的入口。

它会根据系统实际运行时观察到的行为，生成一个文本策略文件，默认位置是：

```text
/dev/secpolgenerate/policy
```

### `secpolcompile`

Host 端工具，用来把文本策略编译成二进制策略文件。

也就是说，常见流程不是在 target 上手写并直接生效，而是：

1. target 上观察系统行为
2. 生成文本策略
3. 拿回 host 编译
4. 再把编译结果放回镜像

### `secpolpush`

用于把策略推给 `procnto`，触发策略真正开始生效。

这一步非常关键。官方文档特别提到，`secpolpush` 应该是系统启动早期就做的事情之一。

## 5. 开发时最常见的工作流

如果你是第一次接触 `secpol`，最实用的理解方式不是死记语法，而是先抓住工作流。

### 第一步：给长期服务分配类型

在启动脚本里，不再只是把服务拉起来，而是显式地带上 `type`。

常见写法类似：

```sh
on -T slogger2_t slogger2 -U 20:20
on -u 21:21 -T pipe_t pipe
on -u 22:21 -T random_t random
on -T devb_t -u 23 devb-eide
```

这里最关键的不是参数顺序，而是这个思路：

- 服务启动时就带 `-T`
- 每个长期存在的服务尽量有独立类型
- 最好同时配合独立的 uid/gid 使用

### 第二步：在开发模式下生成策略

官方文档中常见的生成方式类似：

```sh
secpolgenerate -u -t 100
LD_PRELOAD=secpol-preload.so
procmgr_symlink /proc/boot/libsecpol-gen.so.1 /proc/boot/libsecpol.so.1
secpolpush
```

这里的重点不是逐字照抄，而是理解它在做什么：

- 启动策略生成
- 让系统在较开放的模式下先跑起来
- 收集真实运行行为
- 再据此生成初始策略

QNX 官方文档把这种思路称为从较“开放”的开发状态出发，再逐步收敛到真正的 secure 模式。

### 第三步：充分跑你的系统

只开机一次通常不够。

你需要尽量覆盖真实场景，例如：

- 存储设备挂载
- 网络初始化
- ssh 登录
- 业务进程启动
- 设备节点访问
- 异常路径和恢复路径

因为 `secpolgenerate` 只能根据它“看到过”的行为生成规则。你没跑到的路径，后面 secure 启动时就可能被拒掉。

### 第四步：把文本策略拿回 host 编译

生成出来的文本策略在 target 上可以这样读：

```bash
cat /dev/secpolgenerate/policy
```

官方文档还特别提醒：这里更适合用 `cat` 或 `sftp` 拿文件内容，不建议直接用 `scp`，因为生成文件的方式和普通静态文件不完全一样。

拿回 host 后，再用 `secpolcompile` 编译成二进制策略。

### 第五步：把编译结果放进镜像并 secure 启动

编译后的策略通常建议放在：

```text
/proc/boot/secpol.bin
```

然后在正式镜像里：

- 移除 `secpolgenerate`
- 保留 `secpolpush`
- 让系统在真正受限的策略下启动

这时系统只允许策略里定义过的行为。

## 6. `mkqnximage` 里的 `open` 和 `secure`

如果你是用 `mkqnximage` 组织镜像，官方文档里还有两个很重要的模式：

- `--secpol=open`
- `--secpol=secure`

可以简单理解成：

- `open`：偏开发/生成策略阶段，系统默认更宽松
- `secure`：偏正式运行阶段，按给定策略严格限制

官方文档同时说明：

- `secure` 模式必须指定策略，否则系统无法启动
- `open` 模式可以带策略，也可以不带，适合从零开始生成或在已有策略上继续迭代

## 7. 怎么看当前系统里的类型

一个很实用的命令是：

```bash
pidin -f a_n
```

在启用了相关安全策略机制的系统里，这个输出可以帮助你看：

- 进程当前属于什么 type
- 某些进程是否发生了 type transition

官方文档提到，如果进程通过 `secpol_transition_type()` 切换类型，常会看到带 `__run` 后缀的类型名。

## 8. `secpol` 常见查看命令

如果目标系统已经带了编译好的策略文件，可以先用这些命令做检查：

### 校验策略

```bash
secpol -v
```

### 查看策略内容概况

```bash
secpol -d
secpol -c
```

### 基于 type 或 ability 过滤

```bash
secpol -t devb_t -c
secpol -a pathspace -f ability
```

实际支持的过滤项以目标系统里的 `secpol` 版本为准，但整体思路是一样的：先看整体，再按类型或能力缩小范围。

## 9. 初学时最容易踩的坑

### 只给进程加了 `-T`，但没有尽早 `secpolpush`

这样类型标注和策略约束没有真正进入生效状态，结果往往会看起来“像配了，其实没真正限制住”。

### 多个长期服务共用同一个 type

这会让权限边界变模糊，后面排查规则也会越来越难。

### 生成策略时没有充分覆盖业务路径

最后 secure 启动后，很多行为会在现场才暴露失败。

### 把自动生成的规则原样当最终版

`secpolgenerate` 很适合做起点，但不代表生成结果就是最小权限的最终答案。官方文档也明确建议后续继续审查和收敛不必要规则。

### 过早把 shell 当成正式策略对象

开发阶段有 console 和 shell 很方便，但正式产品里这类入口通常需要更严格处理，不能直接沿用调试期配置。

## 10. 对你这个仓库来说，现阶段最值得先记住的点

如果现在还是 `QNX SDP 8.0 + Raspberry Pi 5` 的学习和实验环境，我建议先建立这几个最小认知：

- `secpol` 不是单个命令，而是一整套 QNX 安全策略机制
- `type` 是策略设计的核心
- 长期服务启动时最好显式带 `on -T`
- 开发阶段通常先靠 `secpolgenerate` 生成初版策略
- 正式阶段依靠 `secpolpush` + 已编译策略进入受限模式
- `pidin -f a_n` 是观察类型分配的很好入口

等后面仓库里开始出现：

- 自定义 resource manager
- 自己的长期后台服务
- 需要限制设备节点、路径或 channel 访问

那时再继续补一页更细的“策略语言”和“实战示例”，会更顺。

## 参考资料

以下内容主要基于 QNX 官方文档整理：

- QNX SDP 8.0 System Security Guide: Implementing security policies  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.security.system/topic/manual/using_security_policies.html>
- QNX SDP 8.0 Utilities Reference: `secpol`  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.neutrino.utilities/topic/s/secpol.html>
- QNX SDP 8.0 System Security Guide: Secpol and policy options  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.security.system/topic/manual/mkqnximage_secpol.html>
