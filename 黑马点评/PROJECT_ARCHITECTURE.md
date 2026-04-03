# hm-dianping 项目代码与架构说明

## 1. 项目定位

这是一个单模块的 Spring Boot 后端项目，围绕“黑马点评”类业务实现了以下核心能力：

- 用户验证码登录与登录态续期
- 商铺查询、分类查询、附近商铺 GEO 查询
- 探店笔记发布、点赞、关注、关注推送
- 优惠券与秒杀券管理
- 秒杀异步下单
- 图片上传到本地目录

从代码结构看，这不是微服务架构，而是一个典型的单体 REST API 应用。业务逻辑集中在一个进程中，Redis 被广泛用作缓存、登录态存储、集合计算、地理索引和异步消息中间层。

## 2. 技术栈

| 类别     | 选型                          |
| ------ | --------------------------- |
| 运行时    | Java 8                      |
| 应用框架   | Spring Boot                 |
| Web    | Spring MVC                  |
| ORM    | MyBatis-Plus                |
| 数据库    | MySQL                       |
| 缓存/中间件 | Redis                       |
| 分布式锁   | Redisson                    |
| 工具库    | Hutool                      |
| AOP    | Spring AOP / AspectJ Weaver |
| 构建     | Maven                       |

补充说明：

- `pom.xml` 中使用了 `spring-boot-starter-parent 2.3.12.RELEASE`
- 同时又显式指定了 `spring-boot-starter-web 2.6.6`、`spring-data-redis 2.6.2`
- 这是“父版本 + 子依赖版本混用”的方式，能运行但后续升级和排障成本会偏高

## 3. 代码目录结构

![](http://localhost:63342/markdownPreview/1244663522/)

`src/main/java/com/hmdp ├─ config         Spring MVC / MyBatis / Redisson / 全局异常配置 ├─ controller     HTTP 接口入口 ├─ dto            请求与响应对象 ├─ entity         数据库实体 ├─ mapper         MyBatis-Plus Mapper ├─ service        Service 接口 ├─ service/impl   业务实现 └─ utils          Redis、拦截器、ThreadLocal、锁、ID 生成等工具 src/main/resources ├─ application.yaml ├─ seckill.lua ├─ unlock.lua ├─ db/hmdp.sql └─ mapper/VoucherMapper.xml`

项目整体遵循经典分层：

- `Controller` 负责接收请求和参数转发
- `Service` 负责核心业务流程
- `Mapper` 负责数据库访问
- `Utils/Config` 负责横切能力和基础设施接入

## 4. 架构总览

![](http://localhost:63342/markdownPreview/1244663522/)

`flowchart LR     A[前端/客户端] --> B[Spring MVC]    B --> C[RefreshTokenInterceptor]    C --> D[LoginInterceptor]    D --> E[Controller]    E --> F[Service]    F --> G[MyBatis-Plus Mapper]    G --> H[(MySQL)]    F --> I[(Redis)]    F --> J[Redisson]    E --> K[本地文件系统]    I --> L[Lua 脚本]    I --> M[Redis Stream]    M --> N[单线程订单消费者]    N --> F`

这套架构的关键点是：

- MySQL 存业务主数据
- Redis 承担高频读写、登录态、幂等控制和异步削峰
- 秒杀链路使用 Lua + Stream + Redisson + 事务组合
- 认证链路使用 Redis Hash + ThreadLocal，而不是传统 Session

## 5. 启动与基础设施配置

### 5.1 启动类

- 启动类为 `[![](http://localhost:63342/markdownPreview/888913499/commandRunner/run.png)](http://localhost:63342/markdownPreview/888913499/markdown-preview-index-f7r1dn7coif2ilrhom2peq5c9d.html#)HmDianPingApplication`
- 使用 `@MapperScan("com.hmdp.mapper")`
- 开启了 `@EnableAspectJAutoProxy(exposeProxy = true)`

`exposeProxy = true` 的作用很关键：秒杀下单场景里，业务代码通过 `AopContext.currentProxy()` 取到代理对象，以保证事务方法在“自调用”场景下依然生效。

### 5.2 Web 层配置

`MvcConfig` 注册了两个拦截器：

1. `RefreshTokenInterceptor`
2. `LoginInterceptor`

执行顺序：

- `RefreshTokenInterceptor` 先执行，负责从请求头 `authorization` 取 token、读取 Redis 用户信息、刷新 TTL、写入 `UserHolder`
- `LoginInterceptor` 后执行，负责校验当前接口是否要求登录

白名单接口包括：

- `/shop/**`
- `/voucher/**`
- `/shop-type/**`
- `/upload/**`
- `/blog/hot`
- `/user/code`
- `/user/login`

### 5.3 MyBatis-Plus

`MybatisConfig` 只做了一件事：

- 注册 `PaginationInnerInterceptor(DbType.MYSQL)`

因此分页查询主要依赖 MyBatis-Plus 的 `Page` 能力。

### 5.4 Redisson

`RedissonConfig` 使用单机 Redis 配置 `redis://192.168.152.128:6379`，并设置了密码。

这说明当前环境默认假设：

- Redis 为单节点
- 没有做哨兵/集群适配
- 地址和密码是硬编码的

## 6. 核心业务模块

### 6.1 用户与登录模块

涉及类：

- `UserController`
- `UserServiceImpl`
- `RefreshTokenInterceptor`
- `LoginInterceptor`
- `UserHolder`

主要流程：

1. `POST /user/code`
    
    - 校验手机号
    - 生成 6 位验证码
    - 保存到 Redis：`login:code:{phone}`
2. `POST /user/login`
    
    - 校验验证码
    - 若用户不存在则自动注册
    - 生成 token
    - 将 `UserDTO` 转成 Hash 存入 Redis：`login:token:{token}`
    - 返回 token 给前端
3. 后续请求
    
    - `RefreshTokenInterceptor` 从请求头取 token
    - 从 Redis Hash 恢复用户信息
    - 保存到 `ThreadLocal`
    - 刷新 token TTL

设计特点：

- 登录态完全放在 Redis，不依赖传统 HttpSession
- `UserHolder` 让业务代码可直接拿当前登录用户
- `sign()` 使用 Redis Bitmap 记录签到

当前状态：

- `logout()` 还未实现
- 只实现了签到写入，没有实现签到统计接口

### 6.2 商铺模块

涉及类：

- `ShopController`
- `ShopServiceImpl`
- `ShopTypeController`
- `ShopTypeServiceImpl`
- `CacheClient`

职责拆分：

- 商铺详情：`/shop/{id}`
- 商铺更新：`PUT /shop`
- 分类分页查询：`/shop/of/type`
- 关键字分页查询：`/shop/of/name`
- 商铺类型列表：`/shop-type/list`

#### 商铺详情缓存策略

当前 `ShopServiceImpl.queryById()` 采用的是“逻辑过期 + 异步重建”方案：

- Redis Key：`cache:shop:{id}`
- 缓存值：`RedisData`
    - `data`
    - `expireTime`

查询逻辑：

1. 先查 Redis
2. 命中且未过期，直接返回
3. 命中但已过期，尝试获取互斥锁
4. 获取锁成功后异步重建缓存
5. 当前线程返回旧数据

更新逻辑：

1. 先更新数据库
2. 再删除缓存

#### 商铺类型缓存策略

`ShopTypeServiceImpl` 把商铺类型列表存成 Redis List：

- Key：`cache:shopType`

流程是：

- 先查 Redis List
- 未命中则查数据库
- 将每个 `ShopType` 转成 JSON 后压入 Redis List

#### 附近商铺查询

`queryShopByType()` 支持两种模式：

- 不传 `x/y`：直接按 `type_id` 分页查数据库
- 传 `x/y`：查 Redis GEO，并按距离排序分页

Redis Key 设计：

- `shop:geo:{typeId}`

GEO 数据并不是运行时自动写入的，而是依赖测试类中的 `loadShopData()` 进行预热。

### 6.3 博客/探店笔记模块

涉及类：

- `BlogController`
- `BlogServiceImpl`
- `FollowController`
- `FollowServiceImpl`

主要能力：

- 热门笔记分页
- 查询笔记详情
- 点赞/取消点赞
- 查询点赞用户
- 发布笔记
- 查询个人笔记
- 查询某个用户的笔记
- 滚动分页查询关注用户的笔记流

#### 点赞模型

点赞使用两层存储：

- MySQL：`tb_blog.liked` 维护总点赞数
- Redis ZSet：`blog:liked:{blogId}` 记录点赞用户及时间戳

优点：

- 详情页和列表页能快速判断“当前用户是否点赞”
- 可以快速取点赞用户列表

#### 关注模型

关注关系也采用两层存储：

- MySQL：`tb_follow`
- Redis Set：`follows:{userId}`

用途：

- 关注/取关写库
- 共同关注通过 Redis Set 求交集

#### Feed 推送模型

发布笔记时，系统会把 `blogId` 推送到粉丝收件箱：

- Redis Key：`feed:{userId}`
- 数据结构：ZSet
- score：发布时间戳

随后通过 `reverseRangeByScoreWithScores()` 做滚动分页，返回：

- 笔记列表
- `minTime`
- `offset`

这个实现本质上是“推模式收件箱”。

### 6.4 优惠券与秒杀模块

涉及类：

- `VoucherController`
- `VoucherServiceImpl`
- `VoucherOrderController`
- `VoucherOrderServiceImpl`
- `SeckillVoucherServiceImpl`
- `RedisIdWorker`
- `seckill.lua`

该模块是项目中最核心的 Redis 实战部分。

#### 普通优惠券

普通优惠券直接存入 `tb_voucher`。

#### 秒杀券创建

`addSeckillVoucher()` 的处理：

1. 保存优惠券主表 `tb_voucher`
2. 保存秒杀表 `tb_seckill_voucher`
3. 把库存写入 Redis：`seckill:stock:{voucherId}`

#### 秒杀下单主流程

`POST /voucher-order/seckill/{id}` 的执行路径如下：

1. 获取当前登录用户
2. 调用 `RedisIdWorker` 生成全局订单 ID
3. 执行 `seckill.lua`
4. Lua 内完成原子校验和写入
5. 若成功，返回订单 ID
6. 后台线程异步消费 Redis Stream
7. 加 Redisson 分布式锁，执行事务落库

#### Lua 脚本负责的事情

`seckill.lua` 原子完成：

- 判断库存是否充足
- 判断用户是否重复下单
- 扣减 Redis 库存
- 记录用户已下单集合
- 向 `stream.orders` 发送消息

这一步把“库存校验 + 一人一单 + 入队”压缩成了单次 Redis 原子操作。

#### 异步订单处理

`VoucherOrderServiceImpl` 在 `@PostConstruct` 中启动单线程消费者，持续读取：

- Stream：`stream.orders`
- Group：`g1`
- Consumer：`c1`

处理过程：

1. 从 Redis Stream 读取消息
2. 转成 `VoucherOrder`
3. 根据用户 ID 获取 Redisson 锁：`lock:order:{userId}`
4. 调用事务方法 `createVoucherOrder()`
5. 校验一人一单
6. CAS 式扣减数据库库存
7. 保存订单
8. ACK 消息

这里形成了“Redis 预扣库存 + 异步落库”的组合方案，目标是提升秒杀高并发吞吐。

### 6.5 文件上传模块

涉及类：

- `UploadController`
- `SystemConstants`

当前实现：

- 上传探店图片到本地目录
- 路径通过 `SystemConstants.IMAGE_UPLOAD_DIR` 固定写死
- 返回的是相对图片路径

说明：

- 这不是对象存储方案
- 更适合本地开发或教学演示

### 6.6 评论模块

当前评论相关类存在：

- `BlogCommentsController`
- `BlogCommentsServiceImpl`
- `BlogCommentsMapper`

但从控制器代码看，评论接口目前基本未实现，属于占位状态。

## 7. 请求处理链路

一个典型请求的大致流向如下：

1. 客户端请求进入 Spring MVC
2. `RefreshTokenInterceptor` 尝试恢复登录用户并刷新 token TTL
3. `LoginInterceptor` 判断是否需要登录
4. `Controller` 接收参数并调用 `Service`
5. `Service` 执行业务逻辑
6. 通过 `Mapper` 访问 MySQL，或直接访问 Redis
7. 返回统一响应对象 `Result`

统一响应体结构：

- `success`
- `errorMsg`
- `data`
- `total`

异常处理由 `WebExceptionAdvice` 统一兜底。

## 8. 数据存储设计

### 8.1 MySQL 主要表

`db/hmdp.sql` 中可见的主要业务表包括：

|表名|作用|
|---|---|
|`tb_user`|用户主表|
|`tb_user_info`|用户详情|
|`tb_shop`|商铺|
|`tb_shop_type`|商铺分类|
|`tb_blog`|探店笔记|
|`tb_blog_comments`|笔记评论|
|`tb_follow`|关注关系|
|`tb_voucher`|优惠券主表|
|`tb_seckill_voucher`|秒杀券扩展表|
|`tb_voucher_order`|优惠券订单|
|`tb_sign`|签到表|

注意：

- 虽然存在 `tb_sign` 表，但当前签到实现主要依赖 Redis Bitmap，而不是这张表

### 8.2 Redis Key 设计

|Key 前缀|数据结构|用途|
|---|---|---|
|`login:code:`|String|验证码|
|`login:token:`|Hash|登录态|
|`cache:shop:`|String(JSON)|商铺详情缓存|
|`lock:shop:`|String|商铺缓存重建锁|
|`cache:shopType`|List|商铺类型缓存|
|`shop:geo:`|GEO|商铺地理索引|
|`blog:liked:`|ZSet|笔记点赞用户|
|`feed:`|ZSet|粉丝收件箱|
|`follows:`|Set|用户关注集合|
|`seckill:stock:`|String|秒杀库存|
|`seckill:order:`|Set|秒杀已下单用户集合|
|`sign:`|Bitmap|用户签到|
|`stream.orders`|Stream|秒杀异步订单流|

## 9. 代码实现特点

这个项目最值得注意的架构特点有四个：

### 9.1 Redis 使用非常重

Redis 不只是缓存，而是承担了：

- 登录会话存储
- 缓存与缓存重建
- 点赞集合
- 关注集合
- Feed 流
- 地理位置索引
- 秒杀库存
- 秒杀用户幂等
- 异步消息队列
- 位图签到

这让项目很适合学习“Redis 在业务系统中的综合用法”。

### 9.2 Service 层承担主要业务编排

Controller 很薄，核心逻辑基本都在 `service/impl`。

优点：

- HTTP 层比较干净
- 业务流程集中

代价：

- Service 已经开始兼顾流程编排、缓存策略、并发控制和数据访问决策
- 如果继续扩展，后续可能需要拆分出更细的领域服务或基础设施组件

### 9.3 MyBatis-Plus 为主，自定义 SQL 很少

绝大多数 CRUD 通过 MyBatis-Plus 完成，当前显式 XML 主要是：

- `VoucherMapper.xml` 用于查询店铺优惠券列表

这说明项目偏向“快速开发”，而不是复杂 SQL 驱动型架构。

### 9.4 秒杀链路是全项目最复杂的部分

它同时用了：

- Lua
- Redis Stream
- Redisson
- AOP 事务代理
- 分布式 ID
- 异步线程消费

所以该模块是整个项目里最需要重点维护和重点压测的区域。

## 10. 当前代码中的现状与风险点

下面这些点不是架构思想，而是我根据当前代码实现看到的“现状”：

### 10.1 商铺详情查询依赖缓存预热

`ShopServiceImpl.queryById()` 当前启用的是 `queryWithLogicalExpire()`，而不是“缓存穿透兜底查询数据库”的版本。

这意味着：

- 如果 Redis 里没有预热好的 `cache:shop:{id}`
- 接口会直接返回“商铺不存在”

也就是说，当前商铺详情功能对预热依赖较强。

### 10.2 热门笔记接口存在匿名访问风险

`/blog/hot` 被放进了白名单，但 `BlogServiceImpl.queryHotBlog()` 内部会调用 `isLikedBlog()`，后者直接从 `UserHolder` 取当前用户 ID，没有空值保护。

结果是：

- 未登录访问热门笔记时，理论上可能触发空指针异常

### 10.3 取关逻辑对 Redis 的数据结构操作不一致

`FollowServiceImpl.follow()` 在关注时把数据写入 Redis `Set`，但取消关注时调用的是 `opsForZSet().remove(...)`。

这会导致：

- Redis 中的关注缓存可能无法正确删除
- 共同关注计算结果可能出现脏数据

### 10.4 秒杀订单 ACK 的 Stream 名不一致

`VoucherOrderServiceImpl` 正常消费消息后 ACK 时使用的是 `"s1"`，而不是前面定义的 `stream.orders`。

这会导致：

- 消息确认行为和实际 Stream 名不一致
- 可能产生 pending 消息无法正确确认的问题

### 10.5 秒杀消费者依赖外部先建 Consumer Group

代码中直接使用 `XREADGROUP` 读取 `g1/c1`，但没有看到自动创建 Group 的逻辑。

因此运行前通常需要手工保证：

- Stream 存在
- Consumer Group 已创建

否则消费者可能直接报 `NOGROUP`。

### 10.6 秒杀代理对象存在竞态窗口

`seckillVoucher()` 中先执行 Lua 入队，再给 `proxy` 赋值；后台线程理论上可能在 `proxy` 尚未赋值前就开始消费消息。

这意味着：

- 极端情况下可能出现异步线程拿到空代理对象的问题

### 10.7 环境配置硬编码较多

当前代码里有多处硬编码：

- MySQL 地址、账号、密码
- Redis 地址、密码
- Redisson 地址、密码
- 本地图片上传目录

这会带来：

- 环境迁移成本高
- 安全性较弱
- 容器化和 CI/CD 集成不方便

### 10.8 功能完整度还不是生产态

目前至少能看到以下未完成点：

- `logout()` 未实现
- 评论接口未实现
- 代码里保留了较多教学演进痕迹和被注释掉的旧实现

## 11. 测试与辅助代码

测试类 `HmDianPingApplicationTests` 主要承担了几个辅助作用：

- 压测 `RedisIdWorker` 的并发生成性能
- 预热商铺逻辑过期缓存
- 把商铺经纬度批量写入 Redis GEO
- 测试 HyperLogLog 的 UV 统计误差

因此这个测试类不仅是单元测试，也兼具部分“数据初始化脚本”的作用。

## 12. 总结

如果从架构角度概括，这个项目可以定义为：

> 一个以 Spring Boot 单体应用为主体、以 Redis 为核心加速与并发控制手段的点评类业务后端。

它的核心价值不在“复杂领域建模”，而在于展示了多种 Redis 业务模式如何落到真实接口里：

- Redis Hash 做登录态
- Redis Bitmap 做签到
- Redis GEO 做附近商铺
- Redis Set/ZSet 做关注、点赞和 Feed
- Redis Lua 做秒杀原子校验
- Redis Stream 做异步下单削峰

如果后续你想继续演进这个项目，优先级最高的方向会是：

1. 修正当前实现中的风险点
2. 把配置改成可环境化管理
3. 补齐未完成接口
4. 为秒杀链路补充初始化、监控和压测
5. 逐步把“业务编排”和“基础设施细节”拆分得更清晰