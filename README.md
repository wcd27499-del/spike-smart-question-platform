# Smart Question Platform
智能刷题与模拟面试平台 — 基于 Spring Boot 的一站式技术面试备战系统。
## 项目介绍
本项目是一个面向程序员的智能刷题与 AI 模拟面试平台后端，提供从题目管理到模拟面试的完整能力：
- **题库管理**：按技术方向（JavaScript、CSS、算法、数据库等）分类组织题目，支持标签、推荐答案、多对多题库关联
- **AI 模拟面试**：接入 DeepSeek-V3 大模型（火山引擎 Ark），支持自定义工作年限、岗位、难度，实时对话式问答并自动生成面试总结
- **社区讨论**：帖子发布/收藏/点赞，Elasticsearch + IK 分词器实现中文全文检索
- **用户体系**：账号密码 / 微信公众号登录，Sa-Token 权限认证，支持管理员与普通用户角色
- **运营保障**：限流熔断、动态黑名单、反爬虫、热 Key 探测、接口文档等全套生产级基础设施
## 技术栈
| 层级 | 技术 | 版本 |
|------|------|------|
| 框架 | Spring Boot + Spring MVC + Spring AOP | 2.7.2 |
| ORM | MyBatis + MyBatis Plus（逻辑删除、分页） | 3.5.2 |
| 数据库 | MySQL + Druid 连接池 | 8.0 / 1.2.23 |
| 缓存 | Redis + Redisson + Spring Session | 3.21.0 |
| 搜索引擎 | Elasticsearch + IK 分词器 | 7.x |
| AI 大模型 | 火山引擎 Ark Runtime（DeepSeek-V3） | 0.1.152 |
| 权限认证 | Sa-Token（注解鉴权、多端登录控制） | 1.39.0 |
| 配置中心 | Nacos | 0.2.12 |
| 流量治理 | Sentinel（限流、熔断、降级） | 2021.0.5.0 |
| 热 Key | JD-hotkey + Caffeine | 0.0.4 |
| 微信接入 | WxJava MP / Open | 4.4.0 |
| 对象存储 | 腾讯云 COS | 5.6.89 |
| 接口文档 | Knife4j | 4.4.0 |
| 工具库 | Hutool、Apache Commons Lang3、EasyExcel | 5.8.8 |
| 模板引擎 | FreeMarker（代码生成器） | — |
| Java | JDK 1.8 | — |
## 项目结构
```
src/main/java/com/spik/
├── annotation/        # 自定义注解（权限校验 @AuthCheck）
├── aop/               # AOP 切面（请求日志、权限拦截）
├── blackfilter/       # Nacos + BloomFilter 动态 IP 黑名单
├── common/            # 通用响应、分页请求、错误码
├── config/            # 配置类（CORS、COS、Redis、MyBatis-Plus）
├── constant/          # 常量定义
├── controller/        # REST 接口层（11 个 Controller）
├── esdao/             # Elasticsearch DAO
├── exception/         # 全局异常处理
├── generate/          # FreeMarker 代码生成器
├── job/               # 定时任务（MySQL → ES 全量/增量同步）
├── manager/           # 通用管理器（AI、COS、计数器）
├── mapper/            # MyBatis Mapper 接口
├── model/
│   ├── dto/           # 请求/响应 DTO
│   ├── entity/        # 数据库实体
│   ├── enums/         # 枚举（用户角色、面试状态等）
│   └── vo/            # 视图对象
├── satoken/           # Sa-Token 权限扩展
├── sentinel/          # Sentinel 限流熔断规则管理
├── service/           # 业务逻辑层
├── utils/             # 工具类（网络、SQL、设备）
└── wxmp/              # 微信公众号消息处理
```
## 数据库表
| 表名 | 说明 |
|------|------|
| user | 用户表（账号、微信、角色、工作经验等） |
| question_bank | 题库表（标题、描述、封面图） |
| question | 题目表（标题、内容、标签、推荐答案） |
| question_bank_question | 题库-题目关联表（多对多） |
| post | 帖子表 |
| post_favour | 帖子收藏表 |
| post_thumb | 帖子点赞表 |
| mock_interview | AI 模拟面试表（岗位、难度、对话消息、状态） |
## 技术亮点
### Elasticsearch 全文检索 + 增量同步
- **IK 分词器**：`ik_max_word` 索引 + `ik_smart` 查询，支持 title / content / answer 多字段中文搜索
- **全量同步**（`FullSyncPostToEs`）：实现 `CommandLineRunner`，启动时从 MySQL 批量写入 ES，每批 500 条
- **增量同步**（`IncSyncPostToEs`）：`@Scheduled(fixedRate = 60000)` 每分钟扫描近 5 分钟变更数据，同样 500 条/批
- **逻辑删除兼容**：同步携带 `isDelete` 字段，ES 搜索结果与数据库一致
- **注解开关**：注释 `@Component` 即可灵活启停同步任务
### 批处理
```java
final int pageSize = 500;
int total = postEsDTOList.size();
for (int i = 0; i < total; i += pageSize) {
    int end = Math.min(i + pageSize, total);
    postEsDao.saveAll(postEsDTOList.subList(i, end));
}
```
全量 + 增量双模式，固定批次写入避免 OOM 和超时。
### 事务与并发控制
**AopContext 自调用代理** 突破 Spring AOP 同类方法调用不走代理的限制：
```java
PostThumbService postThumbService = (PostThumbService) AopContext.currentProxy();
synchronized (String.valueOf(userId).intern()) {
    return postThumbService.doPostThumbInner(userId, postId); // 走代理，@Transactional 生效
}
```
- **外层** `synchronized(userId.intern())`：同一用户串行执行，防止并发重复操作
- **内层** `@Transactional(rollbackFor = Exception.class)`：保证数据库操作原子性
- **乐观锁**：`gt("thumbNum", 0)` 条件更新，防止点赞数并发减为负数
### HotKey 热 Key 探测
JD-hotkey 客户端集成，基于 Etcd 的分布式热 Key 发现 + Caffeine 本地缓存（容量 10000）。热 Key 自动提升到本地缓存，减少 Redis 网络开销。
### Sentinel 流量控制 + 熔断降级
**限流** — 按 IP 维度对题目列表接口精准限流（60 次/分钟）：
```java
ParamFlowRule rule = new ParamFlowRule("listQuestionVOByPage")
    .setParamIdx(0).setCount(60).setDurationInSec(60);
```
**熔断** — 双重策略，触发后 60 秒熔断窗口：
| 策略 | 触发条件 |
|------|----------|
| 慢调用比例 | 响应 > 3s 占比 > 20% |
| 异常比例 | 异常率 > 10% |
**规则持久化** — `FileWritableDataSource` + `FileRefreshableDataSource`，重启不丢失、运行时热更新。
### Nacos + 布隆过滤器动态黑名单
```
Nacos 配置中心 ──监听变更──→ NacosListener（异步线程池）
                                  └→ BlackIpUtils.rebuildBlackIp()
                                        └→ BloomFilter（synchronized 重建）
                                              └→ BlackIpFilter（@WebFilter("/*") 全局拦截）
```
- **布隆过滤器**内存高效（百万 IP ≈ 1MB），O(1) 判重
- Nacos 实时推送配置，无需重启
- 独立线程池异步处理变更，不阻塞回调
- 重建时 `synchronized` 类锁防止并发读写不一致
- `@WebFilter("/*")` 在请求链最前端阻断，避免无效请求进入 Controller
### Sa-Token 权限认证
- `SaInterceptor` 全局拦截所有路径，开启注解式鉴权
- `StpInterfaceImpl` 自定义角色加载，从 Session 动态获取用户角色
- 支持 `@SaCheckRole("admin")` 方法级权限控制
- 封号机制：`UserRoleEnum.BAN` + AuthInterceptor 自动拦截
### Redisson 反爬虫 — 原子计数器
`CounterManager` 封装 Redisson Lua 脚本，实现 `incr + expire` 两步原子执行：
```lua
if redis.call('exists', KEYS[1]) == 1 then
    return redis.call('incr', KEYS[1])
else
    redis.call('set', KEYS[1], 1)
    redis.call('expire', KEYS[1], ARGV[1])
    return 1
end
```
- **时间窗口**：按秒/分/时生成 Key（`prefix:timeFactor`），自动过期，天然滑动窗口
- **原子性**：Lua 脚本在 Redis 单线程执行，无并发竞争
- **通用设计**：可复用为接口限流、身份封禁、频率统计等任意场景