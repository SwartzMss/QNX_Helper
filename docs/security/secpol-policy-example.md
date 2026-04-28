# secpol 策略语言与示例

这一页接着上一页继续写，但重点从“概念介绍”切到“怎么看策略文件、怎么开始写自己的策略”。

如果说上一页是在回答：

> `secpol` 是什么，为什么 QNX 要这么做？

那这一页回答的是：

> 当我拿到 `secpolgenerate` 生成的策略文本之后，到底该怎么看、该怎么改？

## 1. 先建立一个最重要的认知

大多数时候，你不是从零手写完整策略，而是：

1. 先让系统带着 type 跑起来
2. 用 `secpolgenerate` 观察真实行为
3. 拿到生成的文本策略
4. 再去阅读、删减、修正和收敛

所以学习策略语言时，最重要的目标不是背语法，而是 **能读懂生成结果，并且知道哪些地方值得动手改**。

## 2. 文本策略文件长什么样

QNX 官方文档说明，安全策略文本文件由一条条语句组成，每条语句以分号 `;` 结束，注释使用 `#`。

一个很小的例子大致像这样：

```text
# Type for screen service
type screen_t;

# Type for screen client
type screen_client_t;

allow_attach screen_t /dev/screen;
allow screen_client_t screen_t : channel connect;
```

这里已经出现了最核心的三类内容：

- `type`
- `allow_attach`
- `allow ... : channel connect`

只要先把这三类看懂，前期就已经能读不少自动生成的策略。

## 3. 最常见的语法元素

### `type`

最基本的声明语句就是：

```text
type screen_t;
```

它表示定义一个新的安全类型。

几个实际使用时要记住的点：

- 类型名通常用 `xxx_t` 风格，便于一眼看出它是 policy type
- 类型名尽量和服务职责对应，而不是和二进制路径强绑定
- `default` 是保留名字，不要把正常服务定义成 `default`

对学习阶段来说，最实用的命名方式通常是：

- `slogger2_t`
- `io_pkt_t`
- `sshd_t`
- `my_service_t`

### `allow_attach`

这类语句控制一个类型是否允许在 path space 里 attach 某个路径。

示例：

```text
allow_attach random_t /dev/random;
```

可以把它粗略理解成：

> 允许 `random_t` 类型的进程把自己的服务挂到 `/dev/random` 这个路径上。

这条规则对 resource manager 和系统服务很关键，因为 QNX 下很多服务本来就是通过路径暴露接口的。

再举一个更贴近块设备的例子：

```text
allow_attach devb_t mount:/;
```

QNX 官方文档说明，这里 `mount:/` 这种写法带了 file type 前缀，表示它不是泛泛地 attach 到 `/`，而是允许为了处理 `mount()` 请求而进行特定类型的 attach。

这个细节说明了一件事：`allow_attach` 不只是“路径字符串匹配”，它还可能和 QNX 的文件类型语义有关。

### `allow <src> <dst> : channel connect`

这类规则控制一个类型是否允许连接另一个类型对应的 channel。

示例：

```text
allow screen_client_t screen_t : channel connect;
```

可以先把它理解成：

> 允许 `screen_client_t` 去连接 `screen_t` 提供的服务通道。

如果你后面开始写自定义 resource manager，这类规则会变得很常见，因为客户端和服务端之间的消息交互，本质上就是对 channel 的访问控制。

### `allow <type> self:ability`

这类规则是权限控制里最核心的一部分，用于给某个类型授予能力。

QNX 官方文档里的示例类似：

```text
allow server self:ability {
    mem_phys:0x1200000-0x24000000
    setuid:4,6,23
    able_create
};
```

可以这样理解：

- `server` 这个类型被授予一些 ability
- 有些 ability 只是开关型的
- 有些 ability 还会带范围参数

例如上面的：

- `mem_phys:...` 带的是物理内存范围
- `setuid:4,6,23` 带的是允许切换到的 uid 集合
- `able_create` 则更像纯开关

这就是为什么读策略时不能只看“有没有这个 ability”，还要看它的 **范围是不是过宽**。

## 4. 读自动生成策略时，先看什么

自动生成的策略通常不适合直接当最终版。比较高效的阅读顺序是：

### 先看 type 列表

先确认：

- 有没有把关键长期服务分成独立类型
- 有没有几个本该分开的服务被混进同一个类型
- 有没有还残留一些开发期临时 type

如果 type 切分就不合理，后面规则越修越乱。

### 再看 `allow_attach`

这一步主要看路径暴露面是否合理。

重点关注：

- 服务到底 attach 了哪些路径
- 有没有不该暴露的调试接口
- 有没有把路径权限放得太大

### 再看 `self:ability`

这是最值得审查的一块，因为它最容易把系统做成“看起来用了策略，实际上权限还是太大”。

优先关注：

- 是否有明显多余的 ability
- ability 的 range 是否过宽
- 某个普通服务是否意外拿到了高风险能力

### 最后看服务间连接关系

也就是类似：

```text
allow client_t server_t : channel connect;
```

这部分帮助你理解系统的服务依赖图。对后面做最小权限收敛很有价值。

## 5. 一个适合当前仓库风格的最小示例

下面这个例子不是你板子上的真实最终策略，而是一个 **教学型骨架**。目的不是让你直接上线，而是方便你后面往里面替换真实服务名和真实输出。

```text
# Core services
type slogger2_t;
type pipe_t;
type random_t;
type devb_t;
type sshd_t;
type console_t;

# Path attachments
allow_attach pipe_t /dev/pipe;
allow_attach random_t /dev/random;
allow_attach devb_t mount:/;
allow_attach sshd_t /dev/socket/ssh;

# Client-to-service connections
allow console_t slogger2_t : channel connect;
allow sshd_t slogger2_t : channel connect;

# Abilities
allow slogger2_t self:ability {
    nonroot
};

allow pipe_t self:ability {
    nonroot
};

allow random_t self:ability {
    nonroot
};
```

这个例子想表达的重点只有三个：

- 先把长期服务拆成独立 type
- 先明确哪些服务在 path space 里提供入口
- 再逐个审查它们真正需要的 ability

它不追求完整，只追求你后续能看得懂、能往里替换真实内容。

## 6. 一个更贴近启动脚本的示例

如果你在 IFS 启动脚本里已经开始用 `on -T`，可以把文档写成下面这种对应关系。

### 启动脚本示意

```sh
secpolpush

on -T slogger2_t slogger2 -U 20:20
on -u 21:21 -T pipe_t pipe
on -u 22:21 -T random_t random
on -u 23:23 -T sshd_t sshd
on -T console_t ksh -l
```

### 策略阅读思路

- `slogger2` 为什么需要单独的 `slogger2_t`
- `pipe` 和 `random` 为什么不能共用同一个 type
- `sshd` 是否真的需要额外高权限 ability
- `console_t` 是否只应该留在开发镜像，而不该进入正式产品镜像

这样写的好处是，你的文档不是孤立讲语法，而是能和自己的启动链路一一对应。

## 7. 一个推荐的实战记录模板

你后面补真实输出时，可以直接套下面这个结构。

### 场景 1：先看系统里当前有哪些 type

命令：

```bash
pidin -f a_n
```

实际输出：

```text
# 这里后续补你板子上的真实输出
```

观察点：

- 哪些长期服务已经显式带 type
- 哪些进程仍然没有按预期归类
- 有没有出现 `__run` 类型

### 场景 2：导出自动生成策略

命令：

```bash
cat /dev/secpolgenerate/policy
```

实际输出：

```text
# 这里后续补真实输出片段
```

观察点：

- 自动生成了哪些 `type`
- 哪些 `allow_attach` 最关键
- 哪些 `self:ability` 一眼看上去就偏宽

### 场景 3：校验编译后的策略

命令：

```bash
secpol -v
secpol -d
secpol -c
```

实际输出：

```text
# 这里后续补真实输出
```

观察点：

- 策略是否通过校验
- 编译后的 blob 大概包含哪些对象
- 是否能按 type 或 ability 继续细查

### 场景 4：观察实时安全事件

如果系统已经开始进入更严格的策略阶段，可以补一段 `secpolmonitor` 的记录。

命令：

```bash
secpolmonitor -a -p
```

实际输出：

```text
# 这里后续补真实输出
```

观察点：

- 是哪类 ability 检查最常见
- 哪些 path attachment 还在频繁变化
- 哪些失败事件说明策略还没收敛完整

## 8. 初学阶段最值得主动做的“人工收敛”

当你开始手动修改生成策略时，优先做这几件事：

### 给长期服务拆 type

这是最有价值的第一步，通常收益比直接抠单条 ability 更大。

### 删除明显只服务于调试阶段的规则

例如：

- 临时 shell
- 临时 console
- 只在 bring-up 阶段才需要的调试访问

### 收窄 ability range

如果某条规则授予了带范围的 ability，不要只看“能不能跑”，还要看范围能不能更小。

### 重新跑关键业务路径

每改一版策略，都要重新覆盖关键流程，否则很容易出现：

- 启动能过
- 某个深层功能在稍后才失败

## 9. 对后续文档扩展的建议

你后面如果继续补这一块，建议优先加下面两类内容：

### 一类是“真实输出片段”

比如：

- `pidin -f a_n`
- `cat /dev/secpolgenerate/policy`
- `secpol -c`
- `secpolmonitor`

这类内容很适合你当前仓库风格，因为你前面的命令页已经在走“带真实输出”的路线。

### 另一类是“一个真正的自定义服务示例”

最适合后面单独开一页的是：

- 你自己的 resource manager
- 它 attach 的路径
- 它需要的 channel 连接
- 它最终需要的 ability

到那一步，`secpol` 就不再只是概念学习，而是直接进入工程实践了。

## 参考资料

以下内容主要基于 QNX 官方文档整理：

- QNX SDP 8.0 System Security Guide: Security policy language  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.security.system/topic/manual/secpol_language.html>
- QNX SDP 8.0 System Security Guide: Using security policy types  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.security.system/topic/manual/using_security_policy_types.html>
- QNX SDP 8.0 Utilities Reference: `secpolcompile`  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.neutrino.utilities/topic/s/secpolcompile.html>
- QNX SDP 8.0 Utilities Reference: `secpolpush`  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.neutrino.utilities/topic/s/secpolpush.html>
- QNX SDP 8.0 Utilities Reference: `secpolmonitor`  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.neutrino.utilities/topic/s/secpolmonitor.html>
- QNX SDP 8.0 System Security Guide: The generated security policy  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.security.system/topic/manual/retrieve_security_policy.html>
