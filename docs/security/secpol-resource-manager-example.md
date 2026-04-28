# Resource Manager 的 secpol 示例

这一页不再只讲语法，而是把一个典型的 QNX `resource manager` 场景串起来。

目标很明确：

- 服务进程对外提供一个路径
- 客户端去连接这个服务
- 用 `secpol` 把“谁能 attach、谁能 connect、服务本身有哪些能力”拆开控制

这类场景很适合 QNX，因为 `resource manager` 本来就是 QNX 里非常常见的系统服务组织方式。

## 1. 先说结论：为什么 resource manager 特别适合配合 secpol

QNX 官方文档明确提到，安全策略不仅能控制进程 abilities，还能控制两件对 resource manager 很关键的事：

- 它能在 path space 的什么位置调用 `resmgr_attach()`
- 哪些进程允许连接它公开的 channel

这两点很重要，因为一个 resource manager 往往正好有两个对外暴露面：

- 路径暴露面，例如 `/dev/mydev`
- 消息暴露面，也就是客户端对它 channel 的连接

从这个角度看，`secpol` 其实非常适合给 resource manager 做边界收敛。

## 2. 先假设一个小场景

假设我们自己写了一个很简单的服务：

- 服务名：`my_sensor_srv`
- 功能：对外提供一个设备节点 `/dev/my_sensor`
- 客户端：`sensor_client`
- 运行方式：服务开机自启，客户端按需启动

这个例子不是为了模拟完整驱动，而是为了把策略关系讲清楚。

## 3. 这个场景里有哪些对象

最少可以先拆成两个 type：

```text
type my_sensor_srv_t;
type sensor_client_t;
```

这两个 type 的职责分别是：

- `my_sensor_srv_t`：服务端，负责 attach 路径、创建 channel、处理消息
- `sensor_client_t`：客户端，负责连接服务并读写

如果你后面还有日志代理、监控进程、测试程序，也可以继续拆 type，但不要在一开始把所有东西混成一个 `app_t`。

## 4. 启动链路要怎么对应

### 服务端启动

如果服务是长期运行进程，启动时应该显式带 type，例如：

```sh
on -u 30:30 -T my_sensor_srv_t my_sensor_srv
```

### 客户端启动

如果客户端也属于长期存在的业务进程，同样建议显式带 type：

```sh
on -u 31:31 -T sensor_client_t sensor_client
```

如果客户端只是短命测试程序，开发期可以先简化，但等进入收敛阶段时，最好还是给它明确归类。

## 5. 服务代码层面会发生什么

一个最典型的 resource manager，通常会做这些事：

1. `dispatch_create()`
2. 初始化 `iofunc` / `resmgr` 结构
3. `resmgr_attach()` 把服务挂到某个路径
4. 进入 `dispatch_block()` / `dispatch_handler()` 循环

如果路径是：

```text
/dev/my_sensor
```

那从策略角度看，第一件必须明确的事就是：

> `my_sensor_srv_t` 是否被允许 attach 到 `/dev/my_sensor`

所以最基本的一条规则通常会长这样：

```text
allow_attach my_sensor_srv_t /dev/my_sensor;
```

这个规则的含义非常直接：

- 没有它，服务即使代码写对了，也可能 attach 失败
- 有了它，也只是允许在这个路径上注册，不等于客户端自动就能访问

## 6. 客户端为什么还需要单独的连接规则

QNX 官方文档说明，channel 默认并不会自动对所有 type 开放。

对 resource manager 来说，路径 attach 解决的是：

- 服务能不能把入口挂出来

而客户端访问还取决于：

- 它能不能连接这个服务对应的 channel

因此通常还需要一条类似规则：

```text
allow sensor_client_t my_sensor_srv_t : channel connect;
```

可以把它理解成：

> 允许 `sensor_client_t` 类型的进程连接 `my_sensor_srv_t` 创建的公开通道。

这也是为什么只写 `allow_attach` 还不够。一个服务“挂出来”与“谁能连它”是两层不同的控制。

## 7. 一个最小策略骨架

把上面的关系拼起来，一个教学型最小策略可以先写成这样：

```text
type my_sensor_srv_t;
type sensor_client_t;

allow_attach my_sensor_srv_t /dev/my_sensor;
allow sensor_client_t my_sensor_srv_t : channel connect;

allow my_sensor_srv_t self:ability {
    nonroot
};

allow sensor_client_t self:ability {
    nonroot
};
```

这个版本故意很小，只表达最关键的结构：

- 服务类型
- 客户端类型
- 服务 attach 路径
- 客户端连服务 channel
- 两边都尽量非 root 运行

它不是完整工程策略，但很适合做第一块可读骨架。

## 8. 如果服务启动时需要更多权限怎么办

真实系统里，resource manager 往往在启动阶段需要的权限比稳定运行后更多。

QNX 官方文档建议 resource manager 在初始化完成后尽量降权，常见思路包括：

- 启动阶段临时拥有较高权限
- 初始化完成后切换 type
- 或者切到非 root uid/gid，并收窄 abilities

也就是说，实际策略经常不是只有一个 `my_sensor_srv_t`，而可能还会出现过渡态，例如文档里提到的 `__run` 类型后缀场景。

对你现在这个仓库阶段，先记住这个原则就够了：

> 启动期需要什么，不等于运行期应该一直保留什么。

## 9. 一个更贴近真实工程的阅读顺序

如果后面你真的写了一个 resource manager，我建议按这个顺序查策略问题。

### 第一步：先看 attach 是否成功

先确认服务有没有真的把路径挂出来：

```bash
ls -l /dev/my_sensor
pidin -f a_n
```

如果路径没出来，优先怀疑：

- `resmgr_attach()` 代码本身有问题
- 启动进程没按预期拿到 `my_sensor_srv_t`
- `allow_attach` 没写或写错

### 第二步：再看客户端能不能 connect

如果路径已经存在，但客户端访问仍然失败，就重点怀疑 channel 连接规则。

这时应该去看：

- 客户端实际属于什么 type
- 服务端实际属于什么 type
- 是否存在：

```text
allow sensor_client_t my_sensor_srv_t : channel connect;
```

### 第三步：最后再查 abilities

如果 attach 和 connect 都没问题，但服务内部某些操作失败，再去看 abilities 是否缺失或范围过窄。

这是个重要顺序，因为很多人会过早怀疑 ability，结果真正的问题其实是：

- 类型没分对
- attach 没放行
- connect 没放行

## 10. 对应 `secpolgenerate` 输出时怎么读

如果你把系统跑起来，再导出：

```bash
cat /dev/secpolgenerate/policy
```

那你最可能看到的相关内容，通常会落在这几类：

### `type`

比如：

```text
type my_sensor_srv_t;
type sensor_client_t;
```

### `allow_attach`

比如：

```text
allow_attach my_sensor_srv_t /dev/my_sensor;
```

### channel connect

比如：

```text
allow sensor_client_t my_sensor_srv_t : channel connect;
```

### self:ability

比如：

```text
allow my_sensor_srv_t self:ability {
    ...
};
```

你真正要做的，不是看见这些规则就照单全收，而是逐条问：

- 这条规则是不是业务真正需要的
- 范围是不是过宽
- 这个 type 是不是划分得合理

## 11. 如果用 `secpol_resmgr_attach()`，理解上有什么不同

QNX 官方还提供了 `secpol_resmgr_attach()`。它和 `resmgr_attach()` 的核心用途相同，但还能根据安全策略设置权限、属主和 ACL。

这意味着对某些资源管理器来说，策略不只是“允不允许 attach”，还可以进一步参与 mount point 的权限属性设定。

这里我做一个明确区分：

- 这是官方能力
- 但你当前仓库阶段未必需要一上来就引入

更稳妥的学习顺序通常是：

1. 先理解 `resmgr_attach()` 与 `allow_attach`
2. 再理解 `channel connect`
3. 最后再看 `secpol_resmgr_attach()` 这种更进一步的集成方式

## 12. 一个适合你后面补真实输出的模板

你可以直接把下面这一段复制改造成你自己的实战记录。

### 场景：自定义 resource manager `/dev/my_sensor`

### 启动脚本

```sh
secpolpush
on -u 30:30 -T my_sensor_srv_t my_sensor_srv
```

### 客户端启动

```sh
on -u 31:31 -T sensor_client_t sensor_client
```

### 预期策略

```text
type my_sensor_srv_t;
type sensor_client_t;

allow_attach my_sensor_srv_t /dev/my_sensor;
allow sensor_client_t my_sensor_srv_t : channel connect;
```

### 实际命令

```bash
pidin -f a_n
cat /dev/secpolgenerate/policy
secpol -c
```

### 实际输出

```text
# 后续补你自己的真实输出
```

### 观察点

- 服务是否真的以 `my_sensor_srv_t` 启动
- `/dev/my_sensor` 是否真的出现
- 客户端是否属于预期 type
- 自动生成的规则是否比预期更宽

## 13. 这一页最该记住的几句话

- 对 resource manager 来说，`allow_attach` 和 `channel connect` 是两层不同的控制。
- 服务能挂出来，不代表所有客户端都能连。
- 把长期服务拆成独立 type，收益通常远大于一开始就纠结零碎 ability。
- 启动期权限和运行期权限要分开看。
- 自动生成策略是起点，不是最终答案。

## 参考资料

以下内容主要基于 QNX 官方文档整理：

- QNX SDP 8.0 System Security Guide: Security policies  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.security.system/topic/manual/security_policies.html>
- QNX SDP 8.0 System Security Guide: Security policy language  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.security.system/topic/manual/secpol_language.html>
- QNX SDP 8.0 System Security Guide: Implementing security policies  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.security.system/topic/manual/using_security_policies.html>
- QNX SDP 8.0 C Library Reference: `resmgr_attach()`  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.neutrino.lib_ref/topic/r/resmgr_attach.html>
- QNX SDP 8.0 System Security Guide: `secpol_resmgr_attach()`  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.security.system/topic/secpol_resmgr_attach.html>
- QNX SDP 8.0 System Security Guide: Initialization  
  <https://www.qnx.com/developers/docs/8.0/com.qnx.doc.security.system/topic/manual/initialization.html>
