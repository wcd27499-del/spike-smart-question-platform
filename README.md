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

## 简历描述

**Smart Question Platform — 智能刷题与 AI 模拟面试平台**

- 基于 Spring Boot 2.7 + MyBatis Plus 构建分布式后端，整合 MySQL、Redis、Elasticsearch 等主流中间件
- 使用 Elasticsearch + IK 分词器实现中文全文检索，通过 `@Scheduled` 定时任务完成 MySQL → ES 的全量（`CommandLineRunner`）与增量同步，分页批量写入（500 条/批）
- 接入火山引擎 DeepSeek-V3 大模型，实现 AI 模拟面试功能，支持实时对话与自动生成面试总结
- 基于 Sentinel 实现 IP 级别限流（`ParamFlowRule`）与慢调用/异常比例双重熔断降级，规则持久化至本地 JSON 支持热更新
- 使用 Nacos 配置中心 + Hutool BloomFilter 构建动态 IP 黑名单，通过 `@WebFilter` 全局拦截 + 自定义线程池异步监听配置变更，无需重启即可实时生效
- 基于 Redisson + Lua 脚本实现原子化频率计数器，通过时间窗口滑动机制支撑反爬虫与接口限流场景
- 使用 `AopContext` 自调用代理突破 Spring AOP 限制，结合 `synchronized` + `@Transactional` + 乐观锁实现并发安全的点赞/收藏功能
- 集成 Sa-Token 实现 RBAC 权限认证，自定义 `StpInterface` 动态加载角色，支持多端登录控制与封号机制
- 接入 JD-hotkey + Etcd + Caffeine 实现热 Key 探测与本地缓存，降低 Redis 网络开销
- 集成微信公众平台（WxJava）、腾讯云 COS 对象存储、Knife4j 接口文档、Druid 监控等

## 简历技术栈

**后端框架**：Spring Boot 2.7、Spring MVC、Spring AOP、MyBatis、MyBatis Plus
**数据存储**：MySQL、Redis、Redisson、Elasticsearch、IK 分词器
**中间件与治理**：Sentinel（限流/熔断/降级）、Nacos（配置中心）、JD-hotkey（热 Key）、Caffeine
**安全与认证**：Sa-Token（RBAC）、BloomFilter（黑名单）、Redisson + Lua（反爬虫）
**AI 与集成**：火山引擎 Ark（DeepSeek-V3）、WxJava（微信公众号）、腾讯云 COS
**工具与规范**：Lombok、Hutool、EasyExcel、Knife4j、Druid、FreeMarker、RESTful API
