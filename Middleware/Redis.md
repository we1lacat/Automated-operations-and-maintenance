# Redis 笔记

> 适用范围：NoSQL 概念、Redis 五大数据类型、全局命令、事务、持久化、内存淘汰。
> 关键词：key-value、内存数据库、AOF、RDB、混合持久化、maxmemory-policy、过期删除策略。

---

## 一、NoSQL 与 Redis 概述

### 1.1 关系型 vs 非关系型

| 维度 | 关系型（SQL） | 非关系型（NoSQL / Redis） |
|------|--------------|--------------------------|
| 适用数据量 | 一般在千万级数据以下表现良好 | 可支撑百万级高并发 |
| 并发能力 | MySQL 秒级约 2000 连接，瓶颈在 I/O | 内存操作秒级 8~10w，集群可达百万~千万级 QPS |
| 数据模型 | 结构化、关系模型 | key-value 结构 |
| 事务 | 完整 ACID | ACID 处理非常简单 |

### 1.2 Redis 特点

- 非关系型、分布式，一般以**集群 / 单机**部署，支持水平扩展
- 直接在**内存**中完成高并发读写，效率极高
- 对海量数据高效存储访问，可扩展、高可用
- **单线程**操作，每个操作都是原子操作，没有并发问题
- key-value 结构

### 1.3 缺点

- ACID 处理非常简单，无法做太复杂的关系数据库模型
- 不建议存储敏感数据（丢失可能性较大）

### 1.4 典型应用场景

排序排行 / 快速响应要求互动 / 缓存 / session 共享 / 消息队列 / 轻量级通信

---

## 二、数据类型

### 2.1 String —— 经典的 key-value

```bash
set key value              # 增加
get key                    # 获取 value
incr key                   # value + 1（计数器）
decr key                   # value - 1
setex key seconds value    # 存入并 N 秒后失效
ttl key                    # 查看剩余存活时间（-1 永久 / -2 已不存在）
del key                    # 删除
setnx key value            # 不存在时才添加（分布式锁基础）
mset / mget                # 批量设置 / 批量获取
```

### 2.2 Hash —— key {field-value, [field-value]}

适合**存储对象**：

```bash
hset key field value               # 设置字段（field 已存在则覆盖修改）
hget key field                     # 取字段
hdel key field                     # 删字段
hincrby key field increment        # 字段值加增量
hlen key                           # 字段数量
hkeys key / hvals key              # 所有字段名 / 所有字段值
hgetall key                        # 全部 field-value
```

示例：`user_token: {name: session}` —— 可以很好看出一部分应用场景。

特点：**易改难查**（改单个字段方便，但没有按 value 反查的能力）。

### 2.3 List —— 允许重复、支持负索引

类似 `Map<String, List>`，索引允许负数（`-1` 表示最后一个元素）：

```bash
rpush key value            # 从右边（尾部）插入
lpush key value            # 从左边（头部）插入
lrange key start stop      # 取指定区间（lrange key 0 -1 取全部）
lpop key                   # 从左边弹出
rpop key                   # 从右边弹出
llen key                   # 获取列表长度
```

典型场景：用户收藏、消息列表、简单队列。

### 2.4 Set —— 无序集合（可取交、并、差集）

底层 hashtable 实现，元素不重复：

```bash
sadd key member [member...]    # 增
smembers key                   # 遍历所有元素
srem key member [member...]    # 删
spop key [count]               # 随机弹出 N 个
# —— 关键在于集合的交、并、差 ——
sinter key1 key2               # 交集
sunion key1 key2               # 并集
sdiff key1 key2                # 差集（key1 有而 key2 无）
```

典型场景：去重、对比逻辑操作、"抽奖玩法"（spop 随机抽奖）。

### 2.5 Zset（SortedSet）—— 带分数的有序集合

key {member: score}，**不允许重复**，为每个元素分配一个分数并按分数排序：

```bash
zadd key score member                    # 添加（带分数）
zincrby key increment member             # 给成员的分数加增量
zrange key start stop [WITHSCORES]       # 按分数升序取区间
zrevrange key start stop [WITHSCORES]    # 按分数降序取区间
zrank key member                         # 升序排名（从 0 开始）
zrevrank key member                      # 降序排名
zcard key                                # 元素总数
```

典型场景：实时排序，例如实时评论数、排行榜。

---

## 三、数据类型选型与 Key/Value 设计

### 3.1 选型思路

```text
缓存需求 > 是否用 Redis > 如何设计 key-value

value 设计：
  需要排序？        > Zset
  多值/单值？
    允许重复        > List
    不允许重复      > Set
  简单值            > String
  对象              > Hash

key 设计：见 3.2 四原则
```

### 3.2 Key 设计四原则

| 原则 | 说明 |
|------|------|
| **唯一性** | Redis 的 key 必须保证唯一，同 key 写入会覆盖；常通过主键作为 key |
| **可读性** | 能够见名知意；推荐使用能明确分级的命名法（如 `业务:对象:id`） |
| **灵活性** | 为保证唯一可使用较抽象的联合主键，同时解决可读性差的问题 |
| **时效性** | **Redis 一定要设置过期时间**，按业务场景合理设值；可写入时设定，也可按需动态设置，否则长时间占用内存 |

---

## 四、Redis 全局命令

> 大部分运维、管理操作都在此处。

```bash
keys pattern          # 按模式匹配 key —— 注意：数据量大时会阻塞，生产用 SCAN 代替
exists key            # key 是否存在
expire key seconds    # 设置过期时间
persist key           # 移除过期时间（变永久）
select index          # 切换数据库（默认 0~15）
move key db           # 将 key 移动到指定库
randomkey             # 随机返回一个 key
rename key newkey     # 重命名
echo message          # 输出消息
dbsize                # 当前库 key 数量
info                  # 查看服务器状态信息（内存/连接/统计等）
config get *          # 查看全部运行配置
flushdb               # 清空当前库（危险！）
flushall              # 清空全部库（危险！）
```

---

## 五、Redis 安全

```bash
# redis.conf 中设置
requirepass 自定义密码

# 客户端带密码连接
redis-cli -a 自定义密码
```

补充加固建议：

| 措施 | 配置 |
|------|------|
| 限制监听地址 | `bind 127.0.0.1`（或内网 IP），不要裸奔 0.0.0.0 |
| 禁用/改名危险命令 | `rename-command FLUSHALL ""`、`rename-command KEYS ""` |
| 运行低权限用户 | `useradd -s /sbin/nologin redis` 后以该用户运行 |

---

## 六、Redis 事务

> 可以视作一个**批处理脚本**，将一组命令并发原子化提交。
> 特点：**失败不会回滚，也不会中断后续指令**（单条命令语法错误则整队放弃，运行时错误跳过继续）。

三阶段：

1. 开始事务（`MULTI`）
2. 命令入队（逐条缓冲，返回 QUEUED）
3. 执行事务（`EXEC`）

```bash
MULTI                  # 开启事务
set k1 v1              # 入队
incr counter           # 入队
EXEC                   # 一次性执行
DISCARD                # 放弃事务（清空队列）
WATCH key              # 监视 key，EXEC 前若被改动则事务不执行（乐观锁）
```

---

## 七、Redis 持久化

### 7.1 AOF（Append Only File）—— 文件追加

独立日志方式记录**每次写命令**，默认不开启（`appendonly yes` 开启）。

文件同步策略：

| 策略 | 说明 |
|------|------|
| `appendfsync always` | 每次写都刷盘，最安全最慢 |
| `appendfsync everysec` | 每秒刷盘一次（**推荐**） |
| `appendfsync no` | 交给操作系统，最快最不安全 |

优点：

1. 最多丢失 1 秒的数据（everysec）
2. 不小心 `flushall` 也可以救：删除 AOF 文件最后一条命令即可
3. 增量日志，不会因断电损坏，还可以修复（`redis-check-aof --fix`）
4. Redis 能后台重写 AOF（`bgrewriteaof`，压缩体积）

缺点：性能更慢；AOF 文件体积一般较大。

### 7.2 RDB（快照）

- **手动触发**：`save` —— 会阻塞主进程；`bgsave` —— fork 子进程后台进行
- **自动触发**：配置 `save m n` —— 在 m 秒内至少 n 条变动则自动触发快照

优点：全量备份；加载速度快于 AOF。

缺点：无法实时/秒级持久化，可能丢数据；不同版本存在格式兼容问题。

### 7.3 两者对比

| 维度 | AOF | RDB |
|------|-----|-----|
| 内容 | 写命令日志 | 内存快照 |
| 数据丢失 | 最多 1 秒（everysec） | 最后一次快照之后的数据 |
| 文件体积 | 大（可重写压缩） | 小 |
| 恢复速度 | 慢（逐条重放） | 快（直接加载） |
| 性能影响 | 持续小开销 | bgsave fork 瞬时开销 |

### 7.4 混合持久化（4.0+）

```text
aof-use-rdb-preamble yes
```

- AOF 重写时，先以 RDB 格式写入当前全量数据，之后的增量命令以 AOF 格式追加
- 兼得 RDB 的加载速度与 AOF 的低丢失，**生产推荐开启**

---

## 八、Redis 内存淘汰

```conf
maxmemory <byte>        # 设置最大内存上限，超过后按策略淘汰
maxmemory-policy <策略>  # 淘汰策略
```

八种淘汰策略（`maxmemory-policy`）：

| 策略 | 范围 | 规则 |
|------|------|------|
| `noeviction`（默认） | — | 不淘汰，内存满后写入报错 |
| `allkeys-lru` | 全部 key | LRU：最近最少使用 |
| `volatile-lru` | 设了过期的 key | LRU |
| `allkeys-lfu` | 全部 key | LFU：最不经常使用（4.0+） |
| `volatile-lfu` | 设了过期的 key | LFU |
| `allkeys-random` | 全部 key | 随机淘汰 |
| `volatile-random` | 设了过期的 key | 随机淘汰 |
| `volatile-ttl` | 设了过期的 key | 优先淘汰剩余存活时间（TTL）短的 |

> 四大机制：**LRU / LFU / TTL / 随机**，每个按"全部 key"与"仅设过期的 key"两个范围展开。

---

## 九、过期 Key 处理

| 策略 | 机制 | 特点 |
|------|------|------|
| **惰性删除** | 访问 key 时才检查是否过期，过期则删 | CPU 友好；过期 key 长期不访问会堆积，**容易导致 OOM** |
| **定时删除** | 每个设过期的 key 建定时器，到点立即删 | 内存友好；大量定时器消耗 CPU，Redis 未采用 |
| **定期删除** | 每隔一段时间（默认 100ms）随机抽取一批带过期的 key 检查并删除 | Redis 实际采用；单次有限时间，删不完留给惰性删除兜底 |

---

## 十、注意项

1. **生产禁用 `keys *`** —— 阻塞主线程，改用 `SCAN` 渐进遍历。
2. **`flushdb` / `flushall` 必须防误操作** —— 改名或禁用（`rename-command`）。
3. **key 必设过期时间** —— 否则长时间占用内存，最终触发淘汰或 OOM。
4. **`incr` 的 key 若不是整数会报错**，计数前确认初值。
5. **持久化至少开一种** —— 纯内存重启即全丢；生产推荐 RDB + AOF 混合。
6. **缓存三大经典问题**（面试高频）：
   - **穿透**：查询不存在的数据，请求全打到数据库 → 缓存空值 / 布隆过滤器
   - **击穿**：某个热点 key 过期瞬间大量并发打库 → 互斥锁 / 逻辑过期
   - **雪崩**：大量 key 同时过期 → 过期时间加随机值、集群高可用
