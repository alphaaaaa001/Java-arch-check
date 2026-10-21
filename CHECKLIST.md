# 检查表 · 20 项

> 每项三列：**查什么 → 怎么判 → 错了什么样**。
> 判据必须是可执行的（grep 什么 / 看哪个键 / 算什么），不能是"建议启用"。

## A. 线程与异步

| # | 查什么 | 判据 | 错了什么样 |
|---|---|---|---|
| A1 | 慢外部调用（LLM / 三方 HTTP / SMTP）是否占请求线程 | `spring.threads.virtual.enabled=true`？或该调用在独立 executor 里？两者皆无即中招 | Tomcat 默认 200 线程，一次 8 秒对话钉一个线程；200 并发后全站排队 |
| A2 | SSE / 流式是否长期占线程 | 返回 `SseEmitter` / `Flux`，还是 `while` 循环写 `PrintWriter`？ | 一条流连接钉死一个平台线程，直到客户端断开 |
| A3 | `synchronized` 里有没有 IO | grep `synchronized`，逐处看锁内是否有 mapper / http / file / `sleep` | 虚拟线程 pin 住载体线程，开等于没开；平台线程则串行化全站 |
| A4 | 失败模式不同的异步任务是否共池 | `@Async` 有没有指定 executor？LLM / 邮件 / 图片是不是同一个池？ | 一次 LLM 超时挤掉登录验证邮件；拒绝策略互相牵连 |

## B. 连接池

| # | 查什么 | 判据 | 错了什么样 |
|---|---|---|---|
| B1 | 数据库池是否显式配置 | `spring.datasource.hikari.maximum-pool-size`；没配 = **10** | 高 QPS 下排队，报 `Connection is not available, request timed out after 30000ms` |
| B2 | 池大小是否算过 | `连接数 ≈ QPS × 平均查询耗时(秒)`；兜底 `核数×2 + 磁盘数` | 凭感觉设 50：应用不排了，数据库先崩（每连接 256KB–1MB） |
| B3 | `maxLifetime` 是否小于 MySQL `wait_timeout` | 前者默认 30min，后者默认 28800s —— 反过来就要改 | 连接被服务端单方面关闭、池里还留着死连接 → `Communications link failure` |

## C. 缓存

| # | 查什么 | 判据 | 错了什么样 |
|---|---|---|---|
| C1 | 写后可见性 | 发布/编辑后，读者看到新内容的最长延迟？TTL 多久就是多久 | 与"发布即生效"直接矛盾；答辩时被问"你写完多久能看到"就露 |
| C2 | 击穿 | key 失效瞬间有没有 single-flight / 互斥锁？没有则全部回源 | 热点 key 过期的那一秒，N 个并发同时打 MySQL |
| C3 | 清缓存时机 | evict 是否发生在事务**提交前**（`@Transactional` 方法体内） | 并发读在提交前把旧值写回缓存，脏数据长期存在 |

## D. 事务与分层

| # | 查什么 | 判据 | 错了什么样 |
|---|---|---|---|
| D0 | 多表写入是否在同一事务 | 数 `@Transactional` 的**覆盖率**：有几次"写两张以上表"的 Service 方法，其中几处有注解？ | 发文 = 插文章 + 插标签关联，中途失败留半条数据；单表写不受影响，所以平时看不出来 |
| D1 | `@Transactional` 自调用 | 同类内 `this.method()` 直接调 —— 不走代理 | 注解静默失效，异常不回滚；日志里永远看不出来 |
| D2 | 事务内做外部调用 | `@Transactional` 方法体内是否有 HTTP / LLM / 邮件 / 文件 | 长事务一直占着连接，池被吃光，别人等座位 |
| D3 | Controller 是否直接返回 Entity | 返回体里有没有 `passwordHash` / `email` / 内部字段 | 字段泄漏，靠前端"不显示"兜不住；改个接口就漏 |

## E. 接口契约

| # | 查什么 | 判据 | 错了什么样 |
|---|---|---|---|
| E1 | 错误响应是否统一 | 4xx / 5xx 是不是同一结构（code / message / traceId） | 前端靠匹配错误文案，改一句话就崩 |
| E2 | 分页有没有上限 | `pageSize` 能传 100000 吗？ | 一次查询把堆打满，OOM 比慢更突然 |
| E3 | 列表接口打了几次库 | 一条列表请求对应几条 SQL（开日志数） | N+1：100 条数据 = 101 次查询，缓存也救不了 |

## F. 边界与安全

| # | 查什么 | 判据 | 错了什么样 |
|---|---|---|---|
| F1 | 用户自定义样式存的是什么 | 存 token JSON（颜色/字号/圆角枚举），还是 CSS 字符串？ | 存 CSS = `input[value^="a"]{background:url(...)}` 逐字符外带数据 |
| F2 | Agent 的写操作边界 | 工具调用里的写操作（改标签/存草稿）是否需要人确认？ | 提示注入从"说错话"升级为"改数据"，且无法追溯 |
| F3 | 健康检查的语义 | liveness 探针是否依赖 DB / Redis？ | DB 抖动 → 容器被反复重启 → 故障放大 |

---

## 取证据的标准动作

```bash
# 线程与池
grep -rn "synchronized\|@Async\|Executors\." src/main/java | head -30
grep -n "hikari\|maximum-pool-size\|max-active\|threads:" src/main/resources/*.yml

# 事务与分层
grep -rn "@Transactional" src/main/java | wc -l
grep -rn "public .*Controller\|@RestController" src/main/java | head

# 缓存
grep -rn "@Cacheable\|@CacheEvict\|@CachePut" src/main/java
```

**配置里的默认值不算"实测"**：没配 Hikari 只能推出"用的是默认 10"，不能推出"实际并发时只有 10 在用"——后者要压测。
