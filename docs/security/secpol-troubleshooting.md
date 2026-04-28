# secpol 排错与检查清单

这一页专门写 `secpol` 排错，不再重复前面的概念说明。

它的目标很直接：

- 你在板子上碰到“服务起不来”“路径挂不上”“客户端连不上”“策略生成不对劲”时
- 能先按现象定位到大概是哪一层出了问题
- 再决定该看 `type`、`allow_attach`、`channel connect` 还是 `ability`

如果前几页偏“建立知识框架”，这一页更像“实验时放在手边的检查单”。

## 1. 先建立一个排错顺序

很多人一碰到 `secpol` 问题，就立刻去怀疑 ability。

但实际排查时，通常更高效的顺序是：

1. 先确认进程到底是不是按预期 type 启动了
2. 再确认路径 attach 有没有成功
3. 再确认客户端 channel connect 有没有被允许
4. 最后才去查 ability 或更细的子范围

这个顺序很重要，因为很多现象看起来像“权限不够”，其实更早一层就已经错了。

## 2. 先看哪几个命令

如果系统还能正常进 shell，我建议先看这几个入口。

### 看进程和类型

```bash
pidin -f a_n
```

这个命令最适合回答：

- 进程是不是按预期启动了
- 进程现在属于什么 type
- 有没有出现 `__run` 之类的过渡类型

### 看自动生成的策略

```bash
cat /dev/secpolgenerate/policy
```

这个命令最适合回答：

- 系统运行过程中观察到了哪些规则
- 哪些 `allow_attach` 和 `ability` 被记录到了

### 看错误信息

```bash
cat /dev/secpolgenerate/errors
```

QNX 官方文档明确建议：**出了问题先看 `/dev/secpolgenerate/errors`**。

它通常是最直接的入口。

### 看实时安全事件

```bash
secpolmonitor -a -p
```

它适合看：

- ability 检查
- path attachment 行为

如果你想聚焦某个进程，也可以加过滤：

```bash
secpolmonitor -a -p -f my_sensor_srv
```

## 3. 现象一：服务进程根本没按预期起来

这是最先要排的情况。

### 常见表现

- `pidin` 里找不到进程
- 进程起来就退出
- 进程名字在，但 type 不对

### 优先检查什么

先看：

```bash
pidin -f a_n
```

重点确认：

- 进程是否存在
- 是否带上了预期 type
- 是否发生了 type transition

### 常见原因

- 启动脚本没加 `on -T`
- `on -T` 写了，但服务实际不是通过这条路径启动
- 服务内部又切换了 type
- 进程本身在更早阶段就因为别的问题退出了

### 实用判断

如果进程连存在都不存在，先别急着改策略。

这时优先判断：

- 是进程压根没启动成功
- 还是启动了，但被安全策略拦住后立刻退出

这两类问题的处理方向不一样。

## 4. 现象二：resource manager 路径挂不上

这类问题在自定义服务里非常常见。

### 常见表现

- `resmgr_attach()` 之后路径没有出现
- `/dev/my_sensor` 不存在
- 服务看起来活着，但节点就是没有

### 第一反应该看什么

先看：

```bash
pidin -f a_n
ls -l /dev/my_sensor
cat /dev/secpolgenerate/errors
```

### 最常见的策略原因

`allow_attach` 没写，或者写错了。

例如你期望的是：

```text
allow_attach my_sensor_srv_t /dev/my_sensor;
```

但实际可能出现：

- type 名不匹配
- 路径写错
- 你 attach 的真实路径和你想象的不一样

### 你应该怎么想

这类问题要先确认三件事：

1. 进程是不是 `my_sensor_srv_t`
2. 代码是不是真的在 attach `/dev/my_sensor`
3. 策略里有没有对应 `allow_attach`

三者只要有一个对不上，路径就可能挂不出来。

## 5. 现象三：路径存在，但客户端访问失败

这类问题最容易误判成“文件权限问题”，但在用了 `secpol` 以后，常常是 channel 连接规则的问题。

### 常见表现

- `/dev/my_sensor` 已经存在
- 服务进程也活着
- 但客户端 `open()`、`read()` 或连接动作失败

### 优先检查什么

先确认客户端和服务端分别是什么 type：

```bash
pidin -f a_n
```

然后看策略里是否有类似规则：

```text
allow sensor_client_t my_sensor_srv_t : channel connect;
```

### 为什么会这样

前面几页已经提过一次，但这里再强调：

- `allow_attach` 控制的是服务能不能把入口挂出来
- `channel connect` 控制的是客户端能不能连服务

所以“路径已经存在”并不等于“客户端一定能访问”。

### 排查建议

如果路径在、服务在，但客户端不通，优先怀疑：

- 客户端 type 不对
- 服务端 type 不对
- 缺少 `channel connect` 规则

不要上来就往 ability 上加。

## 6. 现象四：`errors` 里提示缺某个 ability

这类问题是最典型的 `secpol` 排错场景。

QNX 官方文档给的例子里，`/dev/secpolgenerate/errors` 可能直接出现类似：

```text
allow random_t self:ability {
    io
};
```

这表示：

- 以 `random_t` 运行的进程尝试使用了 `io` ability
- 当前策略里没有允许它这样做

### 但这里有一个很重要的判断

看到缺 ability，不代表你就应该立刻把它加回去。

先问三个问题：

1. 这个进程是不是本来就跑错 type 了
2. 这个行为是不是真的业务需要
3. 能不能通过改启动方式、uid/gid、文件权限或程序行为，避免依赖这个 ability

这点很重要，因为自动生成或报错提示出来的规则，只说明“它尝试用了”，不自动等于“最终设计就应该允许它一直用”。

## 7. 现象五：每次生成的策略都在变

这也是官方文档专门提到的一类现象。

### 常见表现

- 每次跑完 `secpolgenerate`，策略都多出一点点新内容
- 某些 subrange 看起来总在变
- 很难得到稳定收敛的策略文件

### 官方文档的解释方向

对某些 abilities，尤其和虚拟地址相关的 subrange，系统每次运行时可能都会生成新的范围。

### 这类问题怎么理解

不要把“每次都多一点”简单理解成系统还没收敛。

有时不是业务路径没覆盖，而是：

- 某些范围本来就带运行时波动
- 生成结果在形式上变化，但本质上是同一类权限需求

### 文档建议方向

官方文档提到，一种处理思路是对受影响的 ability 去掉这些变化过强的 subrange，转而用更稳定的方式表达权限。

这一步已经偏收敛优化，不属于最初 bring-up 阶段必须马上做的事。

## 8. 现象六：`errors` 里有报错，但系统表面上没坏

QNX 官方文档也专门提醒了这一点：**有些错误记录并不一定对应真实故障**。

例如某些和 `setuid`、`setgid`、`prot_exec`、`priority` 相关的检查，可能会在日志里留下错误痕迹，但表面上系统仍能继续工作。

### 这意味着什么

不要把 `errors` 文件当成“每一条都必须无脑放行”的待办清单。

更合理的做法是：

- 先结合实际故障判断有没有真实影响
- 再决定是修策略、修启动方式，还是暂时忽略

## 9. 现象七：secure 模式启动后系统更早就坏掉

这通常发生在你从开发阶段切到真正受限模式之后。

### 常见原因

- 生成策略时业务路径没有跑全
- 启动顺序变了
- 某些早期服务在 `secpolpush` 之后拿不到需要的权限
- 某个长期服务其实没有显式 type

### 先看哪里

优先看：

```bash
cat /dev/secpolgenerate/errors
pidin -f a_n
```

如果系统坏得太早，连 `/dev/secpolgenerate/errors` 都拿不到，QNX 官方文档建议可以用：

```bash
secpolgenerate -v
```

这样它会把错误输出到 `stderr`，适合系统还没完全起来时看更早阶段的问题。

## 10. 现象八：怀疑是文件权限，不确定是不是 secpol

这类问题很容易混淆。

QNX 官方文档专门提到，有些系统失败其实对 `secpolgenerate` 来说是“不可见”的。

例如：

- 资源管理器以 non-root 运行后，打开某些文件失败
- 失败原因其实是文件权限、owner、ACL 不对
- 但你直觉上会误以为是少了某个 security policy rule

### 怎么区分

如果你发现：

- 服务 type 看起来对
- attach / connect 规则也齐
- 但某些文件或设备访问仍然失败

那就要开始怀疑：

- 传统文件权限
- uid/gid 配置
- 设备节点权限
- ACL

这类问题未必能靠继续加 ability 正确解决。

## 11. 一个推荐的排错动作模板

当你遇到一个具体故障时，可以按这个顺序走。

### 第一步：记录现象

先写清楚到底是哪种失败：

- 进程起不来
- 路径挂不上
- 客户端连不上
- 某个特权操作失败

### 第二步：记录当前 type

命令：

```bash
pidin -f a_n
```

重点记录：

- 服务端 type
- 客户端 type
- 是否有 `__run`

### 第三步：看 `errors`

命令：

```bash
cat /dev/secpolgenerate/errors
```

重点记录：

- 报的是 `ability`
- 还是 `attach`
- 还是其他安全事件

### 第四步：看策略文本

命令：

```bash
cat /dev/secpolgenerate/policy
```

重点确认：

- 有没有生成预期 type
- 有没有生成预期 `allow_attach`
- 有没有生成预期 `channel connect`

### 第五步：必要时开实时监控

命令：

```bash
secpolmonitor -a -p
```

如果只想盯一个服务：

```bash
secpolmonitor -a -p -f my_sensor_srv
```

## 12. 一个适合你后面补真实输出的排错记录模板

你后续可以直接按这个格式补真实案例。

### 故障现象

```text
# 例如：/dev/my_sensor 没有出现
```

### 相关命令

```bash
pidin -f a_n
ls -l /dev/my_sensor
cat /dev/secpolgenerate/errors
cat /dev/secpolgenerate/policy
```

### 实际输出

```text
# 后续补真实输出
```

### 初步判断

- 进程 type 是否正确
- 是否缺 `allow_attach`
- 是否缺 `channel connect`
- 是否是文件权限/uid/gid 问题

### 修正动作

```text
# 后续补你自己的修改动作
```

### 结果验证

```text
# 后续补验证结果
```

## 13. 这一页最该记住的几点

- 先看 `type`，再看 attach/connect，最后再看 ability。
- `/dev/secpolgenerate/errors` 是排错第一入口。
- `allow_attach` 和 `channel connect` 是两层不同控制。
- `errors` 里的建议规则不等于最终都应该直接放行。
- 有些问题本质上是 uid/gid、文件权限或 ACL，而不只是 `secpol` 规则缺失。

## 参考资料

以下内容主要基于 QNX 官方文档整理：

- QNX SDP 8.0 System Security Guide: Troubleshooting and frequently asked questions  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.security.system/topic/manual/troubleshooting.html>
- QNX SDP 8.0 System Security Guide: The error file  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.security.system/topic/manual/debugging.html>
- QNX SDP 8.0 Utilities Reference: `secpolgenerate`  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.neutrino.utilities/topic/s/secpolgenerate.html>
- QNX SDP 8.0 Utilities Reference: `secpolmonitor`  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.neutrino.utilities/topic/s/secpolmonitor.html>
- QNX SDP 8.0 System Security Guide: System monitoring using secpolgenerate  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.security.system/topic/manual/using_secpolgenerate_secure_system.html>
