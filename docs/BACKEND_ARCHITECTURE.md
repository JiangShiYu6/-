## 后端架构说明

本文档概述 heima-dianping-redis 项目的后端整体设计、目录结构、核心技术与关键业务流程，便于快速理解与维护。

### 1. 技术栈概览

- Spring Boot：应用启动与基础设施整合
- Spring MVC：REST 接口
- Spring AOP：`@EnableAspectJAutoProxy(exposeProxy = true)` 支撑在同类内部通过代理调用触发事务等切面（典型：一人一单逻辑需获取代理对象）
- MyBatis-Plus：ORM & 通用 CRUD (`ServiceImpl`, `BaseMapper`)
- Redis：
  - 缓存（字符串 / Hash / GEO / Stream）
  - 分布式锁（自研 + Redisson）
  - 全局 ID 生成（基于时间戳 + 计数器）
  - 秒杀资格判断 + 预减库存（Lua 脚本）
  - 消息队列（Redis Stream 实现异步落库）
- Redisson：可重入分布式锁、简化锁实现
- Hutool：工具库（Bean 拷贝、JSON、字符串、集合等）

### 2. 目录结构与分层

```
com.hmdp
 ├─ HmDianPingApplication            # 启动类 & Mapper 扫描
 ├─ config/                          # 框架 & 基础设施配置
 │   ├─ MvcConfig                    # 拦截器注册（登录、Token 刷新）
 │   ├─ MybatisConfig                # MyBatis-Plus 相关（如分页）
 │   ├─ RedissonConfig               # Redisson 客户端配置
 │   └─ WebExceptionAdvice           # 全局异常处理
 │
 ├─ controller/                      # Web 层（参数接收 & 结果封装）
 ├─ dto/                             # 数据传输对象（Result、LoginFormDTO、UserDTO、ScrollResult 等）
 ├─ entity/                          # 持久化实体（与表结构对应）
 ├─ mapper/                          # Mapper 接口（MyBatis-Plus 自动实现）
 ├─ service/                         # 业务接口定义
 │   └─ impl/                        # 业务实现：聚合、缓存、锁、消息、风控等
 ├─ utils/                           # 共用工具（缓存、锁、ID、拦截器、正则校验等）
 └─ resources/
     ├─ application.yaml             # 配置
     ├─ *.lua                        # Redis Lua 脚本（秒杀、解锁等）
     ├─ mapper/*.xml                 # 少量自定义 SQL
     └─ db/hmdp.sql                  # 初始化脚本
```

### 3. 核心横切能力

| 能力       | 位置                                              | 说明                                                                               |
| ---------- | ------------------------------------------------- | ---------------------------------------------------------------------------------- |
| 统一返回   | `dto/Result`                                      | 标准化响应结构（状态码 / 数据 / 消息）                                             |
| 登录态传递 | `LoginInterceptor` + `RefreshTokenInterceptor`    | 基于 Redis Token；刷新拦截器先运行（order=0）延长 TTL，登录拦截器做访问控制        |
| 用户上下文 | `UserHolder`                                      | 使用 `ThreadLocal` 保存当前请求用户简化多层传递                                    |
| 全局异常   | `WebExceptionAdvice`                              | 捕获并转换为友好响应（避免栈信息泄露）                                             |
| 分布式锁   | `SimpleRedisLock`/`SimpleRedisLock2` + `Redisson` | 基于 `SET NX EX` 与 Lua 释放；或直接用 Redisson 提供的 `RLock`                     |
| 全局 ID    | `RedisIdWorker` / `RedisIdWorker2`                | Redis 自增序列 + 时间戳（避免数据库热点）                                          |
| 缓存客户端 | `CacheClient` / `CacheClient2`                    | 封装三种常见缓存策略（穿透 / 击穿 / 逻辑过期）与模板方法（DB Fallback 函数式接口） |

### 4. 缓存策略与典型模式

1. 缓存穿透：
   - 查询不存在的 Key 大量打到数据库
   - 方案：空值缓存（写入短 TTL 空字符串 / 特殊标记），`queryWithPassThrough`
2. 缓存击穿（热点 Key 过期瞬间高并发）：
   - 方案 A：互斥锁重建（`queryWithMutex`）
   - 方案 B：逻辑过期（`queryWithLogicalExpire` + 后台线程异步重建）
3. 缓存雪崩：
   - 间接防护：不同 TTL、逻辑过期 + 随机扩展、限流（未在此处显式实现）
4. 地理位置查询：
   - Redis GEO 数据结构 (`SHOP_GEO_KEY + typeId`)，配合分页截取 + 数据库排序补齐
5. 一致性策略：
   - 写操作：先改库后删缓存（删除失败引入重试/消息机制可扩展）

### 5. 秒杀（高并发下单）设计

整体目标：防超卖、防重复、削峰、快速失败。

流程（`seckillVoucherWithRedissonOptimization` / `seckillVoucher`）：

1. 前端请求 -> Controller 调用 Service
2. Lua 脚本原子执行：
   - 校验库存 > 0
   - 判断用户是否已抢购（Set / Hash / 记录）
   - 预减库存、记录抢购资格
   - 返回状态码（0 成功 / 1 库存不足 / 2 重复）
3. 返回订单号（或错误）立即响应
4. 订单创建：两种模式
   - 早期：阻塞队列 / 内存队列（已注释）
   - 现行：Redis Stream `stream.orders` （消费者组 g1）后台线程轮询，`VoucherOrderHandler` 消费
5. 消费端：
   - 处理正常消息 -> 落库存 & 保存订单 -> XACK
   - 异常时 -> 扫描 PendingList 重试 (`handlePendingList`)
6. 并发安全：Redisson `RLock` 以 userId 维度防重（另一种实现是 Redis 自研锁）

关键点：

- 写扩散与热点控制：库存与资格判定在 Redis 层完成，数据库只承接最终成功订单写入
- 流量削峰：异步落库 + 快速返回订单号
- 数据可靠性：Pending List 补偿（防止消费者宕机消息丢失）

### 6. 典型业务流程示例

#### 6.1 查询店铺详情

1. Controller -> `ShopService.queryById`
2. 尝试读取 Redis（逻辑过期 / 互斥锁策略）
3. 未命中或需要重建 -> 获取互斥锁 -> 读数据库 -> 写缓存
4. 返回 `Result.ok(data)`

#### 6.2 秒杀抢券

1. Controller -> `VoucherOrderService.seckillVoucher...`
2. Lua 原子校验 + 预减库存
3. 成功：异步（Stream）排队创建订单；失败：直接返回错误文案
4. 后台线程消费 Stream -> Redisson 锁防重 -> 持久化订单 & 扣减数据库库存

#### 6.3 Token 刷新与登录校验

1. `RefreshTokenInterceptor`：每次请求根据 Header 或 Cookie 中 token 查询 Redis Hash 拿用户数据，刷新 TTL，写入 `UserHolder`
2. `LoginInterceptor`：对需要登录的接口校验 `UserHolder` 是否存在用户；否则拦截
3. Controller 和 Service 直接使用 `UserHolder.getUser()` 获取上下文

### 7. 分布式锁实现对比

| 实现             | 原理                               | 优点           | 局限                         |
| ---------------- | ---------------------------------- | -------------- | ---------------------------- |
| SimpleRedisLock  | `SET NX EX` + UUID 标识 + Lua 释放 | 轻量、可控     | 不可重入、续期需扩展         |
| SimpleRedisLock2 | 变体（命名差异）                   | 演示多实现     | 同上                         |
| Redisson RLock   | 看门狗续期 + 可重入 + 公平锁等     | 功能全面、稳定 | 依赖第三方组件、少量性能开销 |

### 8. 全局 ID 生成策略

格式：`(timestamp << N) | sequence`

- timestamp：相对起始纪元的秒 / 毫秒
- sequence：当秒 Redis `INCR key` 获取序列
- 优点：趋势递增、分库分表友好、无 DB 竞争

### 9. 性能与扩展点

| 关注点     | 现状            | 可扩展建议                              |
| ---------- | --------------- | --------------------------------------- |
| 缓存重建   | 单线程池 + 互斥 | 增加指数退避与监控指标                  |
| 秒杀队列   | Redis Stream    | 引入延迟队列 + 死信队列监听             |
| 缓存一致性 | 删缓存策略      | 可增加 Binlog 订阅或 Canal 监听双写修正 |
| 限流       | 未显式实现      | 增加滑动窗口/令牌桶在网关或拦截器层     |
| 监控       | 未集成          | Prometheus + APM (SkyWalking / Zipkin)  |

### 10. 关键类速览

| 类                        | 作用                                           |
| ------------------------- | ---------------------------------------------- |
| `CacheClient`             | 缓存模板：穿透、击穿（互斥/逻辑过期）封装      |
| `RedisIdWorker`           | 基于 Redis 自增的全局 ID 生成                  |
| `VoucherOrderServiceImpl` | 秒杀核心逻辑（Lua、锁、Stream 消费、订单创建） |
| `ShopServiceImpl`         | 商户缓存策略示例（含 GEO 查询）                |
| `LoginInterceptor`        | 登录态强制校验                                 |
| `RefreshTokenInterceptor` | 登录态续期、ThreadLocal 注入                   |
| `SimpleRedisLock`         | 自研分布式锁示例                               |
| `RedissonConfig`          | Redisson 客户端装配                            |

### 11. 事务与 AOP 注意点

- `@Transactional` 方法内部若被同类其它方法直接调用，不会触发事务代理；因此需通过 `AopContext.currentProxy()` 获取代理（已在部分下单逻辑体现）。
- 启动类 `@EnableAspectJAutoProxy(exposeProxy = true)` 允许暴露当前代理。

### 12. 安全与健壮性补充（可改进）

- 参数校验：可引入 `javax.validation` 注解
- 防重复提交：除业务锁，还可加防重 Token
- 漏洞防护：对上传接口、富文本内容做白名单过滤
- 限制用户下单频率：Redis 计数 + 窗口

### 13. 常见问题 FAQ

| 问题            | 说明 / 解决                                                        |
| --------------- | ------------------------------------------------------------------ |
| 缓存重建击穿    | 使用互斥锁 + 逻辑过期避免同时打 DB                                 |
| 用户信息丢失    | 检查 Refresh 拦截器是否早于 Login 拦截器，Redis TTL 是否被正确刷新 |
| 秒杀超卖        | 核心判断在 Lua 中，确保脚本原子性且库存 Key/结构与脚本一致         |
| Stream 消费堆积 | 增加消费者、监控 Pending List、做好异常重试日志                    |

### 14. 时序示意（秒杀精简）

```
[Client] -> /voucher-order/seckill
   -> Service.seckillVoucher()
      -> Lua(校验资格+库存预减+记录) O(1)
      -> 立即返回订单号(占位)
后台线程
   -> Redis Stream 拉取订单消息
   -> 获取 userId 锁
   -> 校验一人一单（DB 查询）
   -> 扣减 DB 库存（乐观锁 update stock=stock-1 and stock>0）
   -> 保存订单
   -> ACK
```

### 15. 小结

本项目以“到店点评 + 优惠券秒杀”场景为载体，集中演示了：

1. 高并发核心：缓存、锁、削峰、异步化、预减库存
2. Redis 多模型：String/Hash/GEO/Stream/Script
3. 分层清晰：Controller -> Service -> Mapper -> Redis/DB
4. 可扩展性：多实现对比（自研锁 vs Redisson、不同缓存策略）

后续可结合监控、限流、网关、灰度发布等进一步完善生产级能力。

---

文档版本：v1.0  
更新日期：2025-08-24
