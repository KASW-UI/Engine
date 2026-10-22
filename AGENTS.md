# AGENTS.md: Engine 项目政策与起步台账

> 本文件是 **Engine 仓库唯一由每个 agent 自动加载的根政策文件**。
> `Engine/` 下其他任何文件（README、设计草稿、进度自述）**不得削弱本文档里的规则**。
> **创建日期**：2026-10-22。**本次唯一交付物是本文件**——此时 Engine 仓库只有 `.git`，零行 Java 代码。

---

## 〇、当前状态速查（2026-10-22）

> 这一节只回答两个问题：「现在有什么」与「下一个 agent 该从哪下手」。

| 项 | 状态 |
|---|---|
| **Engine 仓库** | `/Users/pr/Documents/GitHub/Engine` —— **空仓库**。仅 `.git`（已 `git init`，`main` 分支无任何 commit）。无 `pom.xml`、无 `src/` |
| **真理之源** | `/Users/pr/Documents/GitHub/Sea-BreakTheWaves` —— Go 1.25.5，module `github.com/Sea-Go/Sea-BreakTheWaves`（`go.mod:1-3`），**冻结只读** |
| **技术栈参照** | `/Users/pr/Documents/java&golang/RAG` —— Spring Boot **4.0.8** + Java **21**（`pom.xml` parent + `<java.version>21</java.version>`）。本机 JDK 21.0.10 |
| **真源规模** | **94,752 行**非测试 + **74,551 行**测试（`wc -l` 实测，2026-10-22） |
| **已完成的 Engine 工作** | **零**。本文件是第一个产物 |
| **本仓库已验证的 Go 侧事实** | 见 §2 / §3 / §5，全部带 `文件:行号` |

**⇒ 准确表述是：「Engine 的政策骨架与真源台账已建立」，不是「Engine 已开始复刻」。**

### 🔴 下一个 agent 请先读这三条

1. **§3 是本项目最容易翻错的一节。** 真源有 **6 处「有契约 / 有实现 / 有接缝，但没接线」**。
   照着 proto 和 `.api` 文件翻版本，会**系统性地把没实现的东西当成已实现**。
2. **§5 的守护测试优先级高于业务代码**（用户 2026-10-22 明确拍板）。
   而且真源那 5 个守护测试**自身有已证实的盲区**，Java 侧**不得照抄其缺陷**——见 §5.6。
3. **§6 有一个未决的架构问题**：`tRPC-Agent-Go` 在 Java 侧没有官方对等物。
   结论是「官方只做了 Python 版，没做 Java 版」，但承接方案**尚未选定**。

---

## 一、项目身份与双轨策略

### 1.1 三方角色

| 项目 | 路径 | 角色 | 语言/栈 |
|---|---|---|---|
| **Sea-BreakTheWaves** | `/Users/pr/Documents/GitHub/Sea-BreakTheWaves` | **真理之源（唯一真源）** —— 生成式推荐系统引擎的原始实现。**冻结只读，不接收任何方向的写入或反向同步** | Go 1.25.5 + go-zero + GORM + tRPC-Agent-Go |
| **Engine（本仓库）** | `/Users/pr/Documents/GitHub/Engine` | **Spring Boot 重构产物** —— 用 Spring 生态表达 Go 的行为。非 Spring Boot 的 Java 写法不在范围内 | Spring Boot 4.0.8 / Java 21 |
| **RAG** | `/Users/pr/Documents/java&golang/RAG` | **政策来源，不是对照基准** | Spring Boot 4.0.8 / Java 21 |

⚠️ **RAG 不是 Engine 的对照基准。** RAG 与 Engine 的真源是两个不同的 Go 项目（RAG 真源是 `Sea-RideTheWind_golang`），
两者的 proto 字段号、错误码、表名、HTTP path **没有任何共享关系**。
**可从 RAG 继承的只有「方法论」**（§4 的纪律、§5.6 的假绿教训），
**不可继承的只有它的状态结论**（哪些域已对齐、哪些漂移已修）——那些对 Engine 一律无效。

### 1.2 技术栈基线（对齐 RAG，避免重踩依赖坑）

| 维度 | 版本/选型 | 来源 |
|---|---|---|
| JDK | **21** | RAG `pom.xml` `<java.version>21</java.version>`；本机 21.0.10 |
| Spring Boot | **4.0.8** | RAG `pom.xml` parent |
| RPC | gRPC（`grpc-java` + protobuf） | 真源 `google.golang.org/grpc v1.80.0`（`go.mod:36`）、`protobuf v1.36.11`（`go.mod:37`） |
| 消息 | Kafka | 真源 `github.com/IBM/sarama v1.47.0`（`go.mod:6`） |
| ORM | MyBatis-Plus（**待定**，见 §8 待决策） | 真源 `gorm.io/gorm v1.31.2`（`go.mod:35`） |
| 迁移 | **幂等重放 DDL，不用 Flyway** | ⚠️ RAG 已实证 Flyway 有状态账本在双服务共库时冲突（`RAG/AGENTS.md` §6.2.5 DRIFT-NEW-015）。真源用 GORM `AutoMigrate`（无状态），Java 侧应同构 |
| 调度/配置 | Spring Cloud Alibaba / Nacos | RAG 同款 |

### 1.3 冻结纪律（从 RAG 移植，理由已实证）

Sea-BreakTheWaves **冻结只读**。**取消反向同步**。

RAG 曾短暂允许「Java 反向同步到 Go 真源」，等于让真源变成**双向可变**——
这比单向漂移更危险，因为**任何一侧都无法再被信任为判定标准**。Engine 不重复这个错误。

Engine 产出的是 **Spring 表达经验**（值得沉淀进本文件），**不是行为定义**。

---

## 二、真源盘点：Sea-BreakTheWaves 拓扑

> 取证日期 **2026-10-22**，git HEAD `5f868bd`。全部为静态读取结果，**未执行任何构建或测试**。
> 任何「文件是否存在」「文件多大」的断言，改本节前请先跑一次 `glob`/`wc -l`——**不要照抄上一版**。

### 2.1 三域 + 一个横切 common

| 域 | `api/` | `rpc/` | 测试文件 / `^func Test` 数 |
|---|---|---|---|
| `service/search` | 13 文件 / 424 行 | 152 文件 / **23,154 行** | 47 / 315 |
| `service/recommend` | 15 文件 / 456 行 | 264 文件 / **61,112 行** | 93 / 1,029 |
| `service/async` | 11 文件 / 289 行 | 233 文件 / **58,145 行** | 106 / 335 |
| `service/common` | — | **22 个子包** | — |

⚠️ `recommend` 一域就占非测试代码的约 **64%**（61,112 / 94,752）。**按行数平均切分是错的。**

### 2.2 对外契约面（穷尽，非抽样）

**HTTP（16 个端点，4 个域内路由文件）**

| 域 | Method + Path | 注册处 |
|---|---|---|
| search | `POST /api/v1/search` | `search/api/internal/handler/routes.go:19-21` |
| search | `POST /api/v1/search/authors` | `routes.go:24-26` |
| search | `POST /api/v1/search/title` | `routes.go:29-31` |
| search | `GET /api/v1/search/tools` | `routes.go:34-36` |
| search | `GET /health` | `search/api/search.go:32-39`（内联闭包，无前缀） |
| recommend | `POST /api/v2/recommend` | `recommend/api/internal/handler/routes.go:14` |
| recommend | `POST /api/v2/recommend/stream` | `routes.go:15` |
| recommend | `POST /api/v2/events` | `routes.go:16` |
| recommend | `GET /api/v2/admin/skills` | `routes.go:17` |
| recommend | `GET /api/v1/admin/reco/evaluation/summary` | `routes.go:20` |
| recommend | `GET /health` | `recommend/api/recommend.go:32-39` |
| async | `POST /api/v1/async/events` | `async/api/internal/handler/routes.go:19-21` |
| async | `GET /api/v1/async/processing/state` | `routes.go:24-26` |
| async | `GET /api/v1/async/projections/state` | `routes.go:29-31` |
| async | `GET /health` | `async/api/async.go:32-39` |

⚠️ **search 域有第二套 HTTP 面**（dev 二进制）：`search/rpc/cmd/search_api/main.go:178-179`，
自带 `main` 与 CORS 中间件（`:308`）。它与 goctl 生成的 `search/api` 是**两套并存**，
Java 侧需明确「复刻哪一套 / 两套都复刻 / 只复刻一套」——见 §8 待决策。

**gRPC（3 个 proto 包 / 10 个方法）**

| 全名 | proto 声明 | 实现 |
|---|---|---|
| `sea.search.v1.SearchService/Search` | `proto/search.proto:43` | `search/rpc/internal/server/searchServiceServer.go:26` |
| `…/SearchStructured` | `proto/search.proto:44` | 同文件 `:31` |
| `…/Tools` | `proto/search.proto:45` | 同文件 `:36` |
| `sea.async.v1.AsyncService/SubmitEvents` | `proto/async.proto:26` | `async/rpc/internal/server/asyncServiceServer.go:26` |
| `…/GetProjection` | `proto/async.proto:27` | 同文件 `:31` |
| `sea.recommend.v1.RecommendService/Recommend` | `proto/recommend.proto:68` | `recommend/rpc/internal/server/recommendServiceServer.go:26` |
| `…/StreamRecommend`（**server-stream**） | `proto/recommend.proto:69` | 同文件 `:31` |
| `…/SubmitEvents` | `proto/recommend.proto:70` | 同文件 `:36` |
| `…/ListSkills` | `proto/recommend.proto:71` | 同文件 `:41` |
| `…/EvaluationSummary` | `proto/recommend.proto:72` | 同文件 `:46` |

**Kafka（4 个 topic / group）**

| 角色 | topic / group | 入口 |
|---|---|---|
| 消费 primary | `article-sync-events` / `sea-breakthewaves-sync` | `async/rpc/internal/mqs/kafka/consumer.go:291,297` |
| 消费 retry | `article-sync-retry` / `sea-breakthewaves-sync-retry` | `consumer.go:325,331` |
| 生产 result | `article-sync-results` | `consumer.go:310` |
| 生产 reco 事件 | `recommendation-v2-events` | `consumer.go:344` |

默认值定义在 `common/config/config.go:27-30`（4 个 endpoint）。

⚠️ **这 4 个装配入口的调用者 `logic.StartWorkers`（`async/rpc/internal/logic/kafka_handler.go:113`）全仓零调用者。** 见 §3 第 5 项。

**go-zero IDL（4 个 `.api` 文件，根 `api/` 目录并非空）**

`api/search.api`（80 行）、`api/async.api`（29）、`api/recommend.api`（59）、`api/recommend-admin.api`（8）。
⇒ **IDL 优先原则**：Java 侧必须先有 `.proto` 对齐，再写代码。

### 2.3 数据面依赖

| 中间件 | 版本 | 用途与出处 |
|---|---|---|
| **PostgreSQL** | `postgres:latest` | 主库 `articles`/`article_chunks`（`search/rpc/internal/model/article_repo.go:34,66`）；`user_memory`（`memory_repo.go:41`）；`usermodel_*`（`async/rpc/internal/usermodel/schema.go:454-455`） |
| **Milvus** | **v2.6.8** | 向量库（`deploy/docker-compose.yaml:144-147`；`milvus-io/milvus/client/v2 v2.6.2`） |
| Neo4j | `latest` | 图 schema（`neo4j-go-driver/v5 v5.28.4`） |
| Redis | `latest` | 三级缓存 L2（`common/cache/redis.go:32`） |
| Kafka | `apache/kafka:latest` | 事件流 |
| MinIO | `RELEASE.2024-12-18` | 工件对象存储（`common/artifacts/s3.go:19`） |
| Elasticsearch | 9.2.3 | — |

可观测：Prometheus + Grafana + Jaeger + OTel Collector + node-exporter + cadvisor + 4 个 exporter
（`deploy/docker-compose.yaml`）。Go 侧用 `go.opentelemetry.io/otel v1.40.0` + `prometheus/client_golang v1.23.2`。

⚠️ **`etc/config.yaml` 不在仓库里。** 三个 rpc 的 `ServiceContext` 都硬依赖它
（`search/rpc/internal/svc/serviceContext.go:17`），而 `InitPath` 直接 `Load(path)` 且**失败即 panic**
（`common/config/config.go:322-324`）。全仓 `**/etc/config.yaml` **零匹配**，只有 6 个 go-zero yaml。
⇒ **PG DSN / Milvus 地址 / Kafka 地址 / LLM key 的真源配置值无法从仓库取证**，一律按 `UNKNOWN` 处理。

---

## 三、真源的完成度边界 —— 本项目最容易翻错的一节

> 真源 README 自述：「三路数值索引的真实模型编码与物理向量库接入、Wiki 编制 Agent 产品化、
> 完整推荐闭环与生产部署尚未全部完成」。
> 下面是**逐项代码级判定**，不是 README 的复述。**这一节直接决定 Engine 每个模块的验收标准。**

| # | 项 | 判定 | 证据 |
|---|---|---|---|
| 1 | **真实模型编码** | **有接口未接线** | 只有确定性哈希假编码器：`search/rpc/internal/fakerepr/fakerepr.go:30`（`EncoderID="fake-encoder.v1"`、dense 64 维）、`Encode():61-71`（sha256 展开，注释 `:14` 明写"不调模型"）；索引侧镜像 `async/rpc/cmd/indexer/fake.go:17-20`；装配点 `cmd/indexer/main.go:54` **硬注入** `fakeEncoder{}`。真实边界 `common/clients/datacenter/representation_gate.go:14-16`（`Representer`）在三域生产代码中 **grep `Represent(`/`NewRepresentationGate` = 0 命中** |
| 2 | **物理向量库接入** | **部分冲突：新三路未接、legacy 已接** | 新链路是**进程内只读 Store**：`search/rpc/internal/retrieval/store.go:1-13`（注释"装载成进程内只读 Store"）；`rpc/internal` 下 134 处 milvus 引用中 `retrieval/` **只占 2 处（且是注释）**（`retrieval/fusion.go:4-6`）。真实调用在 `logic/legacy/precision_ranker.go:211-229` 与 `model/memory_chunks_repo.go:120-148`。⚠️ 且 `infra.MilvusInit()` 唯一调用点 `kafka_handler.go:117` 位于**零调用者的 `StartWorkers` 内**，而 search 启动只初始化 PG（`serviceContext.go:20-25`）⇒ **运行时 Milvus client 可能为 nil**（`precision_ranker.go:212-213` 有 nil 分支报错） |
| 3 | **推荐完整闭环** | **半闭环：召回+排序真实接 PG，重排=纯排序，质量=常量** | 生产装配 `recommend/rpc/internal/trpcagent/core/production.go:33-37`（规则+渠道召回）、`:48`（hybrid）、`:55-60`（加权排序）；`productionRerankProvider.Rerank` 仅 clone+sort+截断 `:207-213`；`staticQualityJudger.Judge`（`:193-205`）把**全部 7 个质量维度**逐字写成同一个常量 `productionDefaultQualityScore = 0.72`（定义在 `:18`，逐字使用见 `:196-202`）+ 硬编码 `Grade: "B"`（`:203`）⇒ **「质量打分」在真源是常量，不是模型输出** |
| 4 | **async 对外事件面** | **有契约无实现（回显）** | `SubmitEvents` 只回显条数 `async/rpc/internal/logic/submitEventsLogic.go:22`；`GetProjection` 只回显入参 `getProjectionLogic.go:22`；`ProcessingState` 硬编码 `"healthy"` `api/internal/logic/async/processingStateLogic.go:22`；`AsyncServiceContext` **仅有 Config** `async/rpc/internal/svc/serviceContext.go:9` |
| 5 | **Kafka 消费链路** | **有实现未接线** | `logic.StartWorkers` `async/rpc/internal/logic/kafka_handler.go:113-138`（内含 `kafka.Init():127`、`kafka.Start():130`、`StartRetry():134`）**全仓零调用者**（`grep StartWorkers` 仅命中定义处） |
| 6 | **AG-UI SSE** | **有实现未接线** | `recommend/rpc/internal/server/agui.go:112,134,252`（`StreamHandler`），**全仓未找到注册点** |

### 3.1 这 6 项对 Engine 的直接约束

**⇒ 逐字复刻「假编码器 + 进程内 Store + 常量打分」是错的，也是不够的。**

正确做法是**两件事同时做**：

1. **复刻真源的「行为骨架」**——即接口形状、回显语义、字段名、码值。**这必须 1:1**，
   因为它是 wire 契约和现有客户端依赖的东西。
2. **对 6 项分别显式记账**——每一项在 Engine 侧标注它对应哪个 phase、是真源缺陷还是待补实现，
   **不得默默保留假实现，也不得默默改成真实现**。
   **静默地把 fake encoder 换成真模型调用，就是一次「无出处的 idiomatization」**（§4.2），按 §4 纪律不可合入。

⚠️ **禁止的第三种做法**：把真源的「未完成」当成「不需要做」，直接省略接口。
`AsyncService` 的两个 rpc **必须存在且回显**，哪怕它们在真源里没有业务价值——
**契约面先于实现价值**，这是 §4.1 对照纪律的直接推论。

---

## 四、语义对齐纪律（从 RAG 移植，唯一防线）

Engine 与 Sea-BreakTheWaves 的差异**只允许出现在表达层**（框架、分层、命名、注解）。
本节规定差异如何不越界。

### 4.1 Go ↔ Engine 对照（强制）

任何行为一致性审查**必须以 Go 为唯一基准**核对 Engine。Go 已冻结 ⇒ 漂移只可能从 Engine 一侧发生
⇒ **两方对照足以发现所有行为问题**。

**对照方式**：

```
对每一个行为决策点，读 Go 对应位置的源文件 → 与 Engine 端对应实现逐字对照
```

**对照必须覆盖**（不限于）：

- proto 字段号、message 名称、服务全名（`sea.search.v1` / `sea.recommend.v1` / `sea.async.v1`）
- gRPC 方法的流式/非流式语义（`StreamRecommend` 是 **server-stream**）
- Kafka topic 名、消费组名
- HTTP 路径与 method（注意真源 search 有 `/api/v1/`，recommend 是 **`/api/v2/`**）
- 数据库表名（真源 `usermodel_subject_state` / `usermodel_events` 等，见 `usermodel/schema.go:454-455`）
- 排序稳定性（真源 `skillsys/registry.go:18` 显式 `sort.Strings` "让工具顺序稳定"）
- 错误码与未知码回退

**避免的对照陷阱**（RAG 已实测出这三条，每条都造成过真实漏检）：

- **不要拿「现有 Engine 代码」当对照基准**——新写的代码本身就是待验证对象。
- **不要拿真源的注释 / README 当对照基准**——§3 已实证 README 的完成度自述与代码不符。
- **不要拿 RAG 的结论当对照基准**——两个项目的真源完全不同（§1.1）。

> **只留一个源、两方对照**。不要引入「Java 中间镜像」作为第三对照面——
> RAG 曾这么做，实测发现中间镜像自身带 2 处缺陷，且与 Go 在多处**一致地一起错**。

### 4.2 idiomatization 留痕（强制）

Engine 每次选择「用更地道的 Spring 写法」时，必须能回答：

> **这段比 Go 多做了什么？语义还一样吗？**

回答不了 = **无出处的 idiomatization = 不可 review，不得合入**。

落法：写入该类/方法的 javadoc：

```java
/**
 * <p>L4 idiomatization：<Spring 侧写法> 取代 Go 的 <Go 侧写法>。
 * <p>Go 出处：<路径:行号>
 * <p>语义等价论证：<逐条比对结论>
 *   <b>已知差异</b>：<如有，在此写明并指向 §8 已签字偏离>
 */
```

⚠️ **行号要指向「消费点」而不是「声明点」。** 贴 struct 定义行不算证据
（RAG 的 11 处回归全部是无出处 idiomatization——注释写得很完整，唯独没标 Go 出处）。

### 4.3 框架机制必须验证实际生效（强制）

**这是 RAG 付出最多代价换来的纪律，全部适用于 Engine。**

**规则一：「配置里没写」≠「没开」，也≠「没做」**

Go 侧大量默认值**藏在 struct tag** 里；Java 侧大量行为**藏在框架里**。只看配置文件/注解名称双向推断，两头都会错。

⇒ 判断 Go 默认行为是否生效，**必须读到消费该 tag 的那段控制流**。
⇒ 判断 Java 框架能力是否生效，**必须有运行证据**（真实管线 / 真容器）；
「pom 里有依赖」「类上有注解」**都不算**。

**规则二：「断言 Bean 存在」≠「断言机制生效」**

Spring Boot 大量机制是运行期动态装配（`@Transactional` AOP、`@ResponseBodyAdvice`、starter 自动注册、拦截器链），
**出错时不报错**。RAG 实测：`grpc-spring-boot-starter` **从不自动注册**用户提供的 `ServerInterceptor`
—— Bean 在容器里、starter 不用它，两件事都对，机制照样是死的。

⇒ 凡依赖**框架自动注册**的机制，测试必须断言二者之一：
- **运行时的可观测后果**（最强）
- **对象确实进入了框架的接管点**（次强，如 `verify(serverBuilder).intercept(interceptor)`）

**规则三：绿测试可能是「错误答案的守卫」，比没有测试更危险**

RAG 的 message-api 域有 3 条漂移**全都被绿测试锁死过**，且都不是笔误、都有完整推导。
其中一条测试名里写着 `...SemanticsArePreserved`，**读起来就是在宣称对齐了真源，实际断言的是错误类比**。

⇒ **测试断言的是「Engine 现在的行为」，不是「Go 的行为」。**
当测试名 / `@DisplayName` 里出现「对齐 Go」「与 Go 一致」，而断言写的是 Engine 的输出时，
它要么是巧合，要么是**在给未验证的假设发通行证**。

**强制三条**：
1. 论证里每个 Go 事实都带 `文件:行号`，**不许只写结论**；
2. 凡写「与 Go 一致」，必须有**两侧同时跑的探针**（不是只跑 Engine）——
   最小形态：Go 侧直接调用真源的包，Engine 侧用**真实的** mapper/handler（不是自己 new 一个）；
3. 修复漂移时**先改/删错误断言，再改实现**。

⚠️ **「只跑 Engine 的探针」同样不算数**——它证明的是「代码写成了这样」，证明不了「Go 是这样」。
**变异测试**能证明测试不是假绿，**双侧探针**能证明期望值是对的。**两者缺一不可。**

**规则四：「Go 侧没有 X」这类否定式断言，默认不可信，必须去被否定的那一侧找正面证据**

RAG 实测两例同形态失败：①「Go yaml 没写 MaxBytes」——其实由 `default` tag 开启；
②「Go 侧没有 client 熔断」——其实由 `default=true` 开启。两处都是**「默认值藏在框架 tag 里，
而对照时只看了数据文件」**。**否定式断言无法用「没找到」来证实。**

⇒ Engine 侧同款风险已经出现：§3 的 6 项里有 4 项是「有实现未接线」，
**「没有调用者」不等于「没有实现」，「有接口」也不等于「可用」**——
两种方向都会错，必须双向取证。

**规则五：跨域/跨仓库照搬的实测结论，动手前必须核对前提逐字相同**

RAG 的 DRIFT-003 根因不是「实测错了」，而是**实测对象与本次对象根本不是同一个 tag**。

⇒ Engine 侧的具体形态：`skillsys` 与 `SkillRegistry` 是**两套 tool 抽象，注册语义相反**
（`skillsys/registry.go:34` 覆盖式写入无重名检查 vs `core/skills.go:75` 重名报错）。
**照搬任一侧的结论到另一侧都会错。**

### 4.4 已知偏离（有意为之，需签字）

> 当前为空。**任何偏离必须在此表登记后才可合入**，登记表头如下。

| 偏离 | 内容 | 理由 | 签字日期 |
|---|---|---|---|
| （暂无） | | | |

---

## 五、架构守护测试（优先级高于业务代码）

> **用户 2026-10-22 明确拍板：Java 侧要立等价守护测试，且优先级高于业务代码。**
> 理由：架构约束不立守门人，9.5 万行翻版过程中必然逐步腐化。真源那 5 个守护测试就是为此存在的。

### 5.1 真源的 5 个守护测试与 2 份 allowlist

**关键结构事实**：`naming_test.go`、`size_ratchet_test.go` 只是 **9~19 行的薄装配**，
真正逻辑在外部库 **`github.com/Sea-Go/sea-guards`**（`go.mod` 有 `replace ... => ./sea-guards`）。
其余 3 个是自包含的（`go/parser` + `go/ast` 自己写断言）。

| 测试 | 守住的不变式 | 检测机制 | 硬编码阈值 |
|---|---|---|---|
| `framework_wiring_test.go` | 承担 C17 职责的包**必须** import `tRPC-Agent-Go`；纯算法内核包**必须不** import | `parser.ParseFile(ImportsOnly)` + `strings.HasPrefix` | 4 包 / **6 装配文件**；**8 个** `pureKernelPackages` |
| `naming_test.go` | 目录名/包名**不得带代次后缀**（防 `v2` 式分叉命名） | `filepath.Walk` + 正则 `v[2-9][0-9]*$` | 跳过 **19 个目录**；allowlist **1 条** |
| `size_ratchet_test.go` | 手写文件行数**只减不增**；新增文件不得超上限 | 文本行计数（**非 AST**） | 上限 **800**，硬编码在调用点 |
| `schema_ownership_test.go` | schema 只以各域 GORM model 形式存在；migration 包已物理删除 | 8 个 `os.Stat` **负向探针** + `WalkDir("service")` + import 子串匹配 | **7 个附属目录 + 1 个根 `migrations` + 1 条禁用 import 子串** |
| `shared_symbol_test.go` | 同一共享算法**只允许一处实现**（防镜像回潮） | `go/ast` 遍历 `FuncDecl`，过滤 `IsExported()`，**再过滤所有带接收者的方法** | canonical 目录 = **恰好 2 个**：`common/wholeindex`、`common/retrieval/rrf` |

### 5.2 两份 allowlist 的精确语义

**`naming_allowlist.json`（1 条）** — `path: service/recommend/rpc/internal/trpcagent/core`、
`package: recommendationv2`、`reason` + **`converge_event`**（收敛事件）。
⚠️ 库内规则：白名单条目**缺 `path`/`reason`/`converge_event` 三要素 → 直接判为违例**（缺一不可）。
已实测该条目**仍然必要**：`core/runtime.go:1` 确实是 `package recommendationv2`。

**`size_allowlist.json`（5 条）** — 基准是 **checked-in 的行数快照 `files` map**，
**不查 git、不依赖任何 commit/ref/tag/history**。行数公式 = `wc -l + 1`。

| 建档文件 | 记录值 | 实际末行 |
|---|---|---|
| `recommend/.../agent/legacy/orchestrator.go` | 882 | 881 |
| `recommend/.../core/runtime.go` | 1357 | 1356 |
| `recommend/.../skill/memory_manage/memory_manage.tool.go` | 997 | 996 |
| `recommend/.../skill/rerank/ai_rerank.tool.go` | 805 | 804 |
| `search/.../llm/legacyagent/content_search_agent.go` | 966 | 965 |

⇒ **5 个文件全部精确卡在记录值上，零余量。任一文件加一行即变红。**

### 5.3 真源守护测试的**已证实盲区**（Engine 不得复刻）

这是本节最有价值的部分。RAG 的教训是「绿测试可能是错误答案的守卫」——
**真源这 5 个守护测试自己就有多处这种形态**：

| # | 盲区 | 后果 | Engine 的处置 |
|---|---|---|---|
| **B-1** 🔴 | **`size_ratchet` 的「只减不增」只是一句注释，没有任何机器断言守护它。** 同一 PR 里把 `files` 里的 882 改成 1000，测试照绿 | 棘轮**完全可被改 JSON 绕过**，形同虚设 | **Engine 必须补一条「allowlist 只减不增」自校验测试**（比较本次 vs 上次 commit 的快照，且对**记录值上升**判红） |
| **B-2** 🔴 | **`schema_ownership_test.go` 的测试名严重超载承诺。** 断言里**没有任何一条涉及 GORM，也没有任何一条涉及 schema 归属**。它真正证明的只有「8 个目录不存在」+「没人 import 已删包」。把 GORM model 删光、改由裸 SQL 执行，**该测试依然全绿** | 名字宣称的东西**完全没被验证** | **Engine 不得复制这个名字。** 必须补**正向锚点**：断言每个域确有归属的 schema owner（谁声明了哪些表） |
| **B-3** 🔴 | **`shared_symbol_test.go` 对「方法」完全设防。** 显式跳过带接收者的 `FuncDecl`，而绝大多数算法代码是方法——把镜像算法改写成 `func (x *Foo) RRF(...)` 即可完美逃逸。只查 `IsExported()` ⇒ 两个包各复制一份小写 `mergeRRF()` 也逃逸 | 「唯一实现」这个命题**大部分场景未覆盖** | Engine 必须**覆盖方法**（按类+方法签名建唯一性索引），且**不能沿用「跳过方法」豁免** |
| **B-4** | `framework_wiring_test.go` 的**登记制**：只守那 6 个装配文件，新增带框架的包**不会自动纳入**，仓库其余部分完全裸奔。且 `:84` 只判 `HasPrefix` ⇒ Go 里 `_ "trpc.group/..."` **blank import 即可蒙混过关** | 新包逃逸；空导入假绿 | Engine 改为**反向全量扫描**（找出所有 import 框架的包，判定其是否在白名单内），而不是正向登记 |
| **B-5** | `framework_wiring_test.go` 的 `pureKernelPackages` 检查用 `os.ReadDir` **非递归** ⇒ 在纯内核包下**新增一个子目录**，其内部代码不被扫描，框架依赖可藏身其中 | 子目录逃逸 | Engine 用**递归遍历** |
| **B-6** | `naming_test.go` 的 `converge_event` 是**自由文本，机器不可校验**，也无过期时间、无「比上次变短」断言 ⇒ 豁免一旦写入**可永久存续** | 白名单只增不减 | Engine 必须给豁免加**到期日**或**收敛事件的可验证条件** |
| **B-7** | `size_allowlist.json` 的 **`"limit": 800` 是死字段**——库内解析出 `SizeAllowlist.Limit` 后**从未使用**，改它毫无效果 | 误导后来人 | Engine 不复制死字段 |
| **B-8** | `sea-guards` 在 CI 里以**未固定 ref 的默认分支** checkout，配合 `replace` ⇒ **扫描器语义可在上游一次提交后静默改变**，本仓无版本锚定 | 守护强度不受本仓控制 | Engine 的守护测试**必须自包含**（不依赖未锚定的外部扫描器） |
| **B-9** | `naming_test.go` 解析包名失败时**静默 `return nil`** —— 不产生违例、不报错 | 语法错误的文件逃逸 | Engine 必须**解析失败即判红**（fail-loud） |

⚠️ 另有**文档与实际脱节**：真源 `README.md:127` 的结构图**只列了 `naming_allowlist.json`**，
完全没提 `size_allowlist.json` 与 `size_ratchet_test.go`，且 `:135` 写"仓库根的**四个**守护测试"——**实际是 5 个**。
⇒ **不要以 README 为依据推断守护测试的数量**（这是 §4.1 纪律的又一次实际发作）。

### 5.4 CI 编排实况

唯一 workflow：`.github/workflows/go.yml`（35 行，1 个 job `test`）。

- 触发：`push`（`**`）+ `pull_request`（`:2-5`）
- 命令：`go test -race -count=1 -short -skip '...' ./...`（`:31`）—— 5 个守护测试是**顺带覆盖**，无独立 job
- ⚠️ **必须双 checkout**（`:11-15`）：因为 `replace ... => ./sea-guards`，缺了它 root package 编译失败、**5 个测试全跑不了**
- `-skip` 名单 5 条（`:31`）**全是 span 相关测试**，**与 5 个守护测试无关**：
  `TestFrameworkNativeSpansShareRuntimeTrace`（`common/runtime/framework_trace_test.go:28`）、
  `TestOpenAIModelRunnerLifecycle`（`runtime_test.go:26`）、`TestPrepareGraphRunnerReceiptAndNativeSpans`（`async/.../content/graph_test.go:94`）、
  `TestIndexGraphRunnerReceiptAndNativeSpans`（`index_graph_test.go:66`）、
  `TestWikiCompileNativeRunnerEOFAndFixedScope`（`wiki_compile_graph_test.go:103`）
- 5 个守护测试**都没有 `testing.Short()` 判断**，所以即使加了 `-short` 也会执行
- ⚠️ **仓内无 branch protection / ruleset 配置文件**，这些测试**是否阻塞合并取决于 GitHub 远端设置，本仓无法确认** ⇒ 按 `UNKNOWN` 处理

⚠️ 那条 CI 注释记录的根因是 **`trpc-agent-go v1.10.0` 上游数据竞争**（`session.Clone` vs `steer.Queue.Close`），
自 2026-10-20 起在 CI Linux `-race` 下间歇失败。
**这 5 个 skip 是上游缺陷的绕行，不是 Engine 可以照抄的设计**——
Engine 若选自研 Agent 运行时，需自己处理这类并发问题。

### 5.5 Engine 的守护测试立什么（待实施，按优先级）

| 优先级 | 等价物 | 对应真源 | 必须补的（真源没有的） |
|---|---|---|---|
| **P0** | **架构守护测试模块**（独立 Maven 模块，不依赖业务代码） | 5 个 | 全部 9 项盲区修正（B-1~B-9） |
| P0 | 框架依赖边界扫描 | `framework_wiring_test.go` | 反向全量 + 递归（B-4/B-5） |
| P0 | 共享算法唯一性 | `shared_symbol_test.go` | 覆盖方法（B-3） |
| P1 | 文件大小棘轮 | `size_ratchet_test.go` | allowlist 自校验（B-1） |
| P1 | 命名规范（无代次后缀） | `naming_test.go` | 豁免到期机制（B-6）+ fail-loud（B-9） |
| P1 | 表/schema 归属 | `schema_ownership_test.go` | **正向锚点**（B-2），并改用 MyBatis-Plus 而非 GORM 命名 |
| P2 | proto 字段号冻结 | — | **Engine 特有**，真源没有。proto 契约是 wire 边界，必须有守门人 |

---

## 六、tRPC-Agent-Go 承接：Java 对等物调研

> 用户 2026-10-22 拍板：「先只做技术调研，产出方案对比，**不预设结论**」。
> 本节是调研结果，**结论栏为 UNRESOLVED**，等用户决策。

### 6.1 真源到底用了多少框架能力（这一节决定了风险的真实大小）

初判是「整个 Agent 运行时都要重写」。**实测后这个判断是错的。**

`service/common/trpcagent` 是 **5 个子包 / 7 个文件的薄适配层**，只用了框架的 **6 个面**：

```
session/service.go:4    trpc-agent-go/session
session/service.go:5    trpc-agent-go/session/inmemory
event/event.go:4        trpc-agent-go/event
telemetry/telemetry.go:6 trpc-agent-go/telemetry/trace
model/gateway.go:6-7    trpc-agent-go/model  + /model/openai
memory/readonly.go:7-9  trpc-agent-go/memory, /session, /tool
```

⇒ **不用 `graph`、不用 `runner`、不用 `agent`、不用 `knowledge`。**
且 `common/trpcagent` 全仓只有 **4 个消费者**（`searchclient/client.go:11`、
`recommendclient/client.go:11`、两个 `internal/trpcagent/runner/factory.go:9`）。

**它自己定义的抽象极少**——唯一 Sea 自有的扩展点是 `memory.Reader`（`readonly.go:14-16`），
其余全是框架接口的再包装。设计意图写在注释里（`session/service.go:8-10`）：
"process-local framework backend … Postgres/Redis implementation can replace it without changing Agent assembly"
——**真源自己就预留了后端可替换点**。

⚠️ **真正的框架深度在域内，不在 common**：
`service/{search,recommend}/rpc/internal/trpcagent/` 才是装配层（`runner/factory.go:18-30` 调 `runner.NewRunner`），
`service/recommend/rpc/internal/trpcagent/core/` 才是真正的技能系统与生产装配。
**§6 的风险集中在这两处，不在 `common/trpcagent`。**

### 6.2 上游事实（带证据）

| 项 | 事实 | 出处 |
|---|---|---|
| 依赖版本 | `trpc.group/trpc-go/trpc-agent-go **v1.10.0**` | `go.mod:39` |
| session 后端 | 另有 `trpc-agent-go/session/postgres v1.10.0`（`go.mod:40`）与 `storage/postgres v0.8.0`（`go.mod:205` indirect） | 同上 |
| A2A | `trpc-a2a-go v0.2.5` | `go.mod:38` |
| 两种 session 后端并存 | inmemory 在 `common/trpcagent/session/service.go:12`（**进程内，重启即失**）；Postgres 在 `common/runtime/session.go:27`（`OpenPostgres`，含 `WithSkipDBInit` / `WithEnableAsyncPersist(false)`） | 二选一由调用方决定 |
| 真源模块路径 | `github.com/zeromicro/go-zero **v1.10.3**` | `go.mod:33` |

### 6.3 Java 侧对等物：**官方没有**

**核实方式**（不是「没搜到」，是正面的仓库检索）：
GitHub API 检索 `org:trpc-group + agent in:name` → **`total_count: 3`**，全部为：

| 仓库 | 语言 | 创建 | star |
|---|---|---|---|
| `trpc-group/trpc-agent-go` | Go | 2025-05-14 | 1875 |
| `trpc-group/trpc-agent-python` | **Python** | **2025-12-16** | 145 |
| `trpc-group/trpc-agent-go-benchmark` | Go | 2026-03-20 | 1 |

⇒ **不存在 `trpc-agent-java`。**

⚠️ 但**必须同时记录正面事实，否则这条否定式断言就是 §4.3 规则四的违规**：

1. `trpc-group/trpc-java` **存在**（Java，2023-10-19）—— 但它是 **RPC 框架**，**不是 Agent 框架**。
2. **官方在 2025-12 做过 Python 版移植**（`trpc-agent-python`）⇒ **该团队具备移植能力，且已验证过一次**。
   ⇒ 「Java 版会不会出现」是**可以等的**，不是「不可能有」。
3. `trpc-agent-go` 为 **Apache-2.0**（GitHub API 返回 `license.spdx_id: Apache-2.0`）
   ⇒ **协议上允许自建实现**，无许可障碍。

**⇒ 结论一栏：`官方对等物不存在（已用 API 正面核实，非「没找到」）；但移植可行性有先例、有许可。`**

### 6.4 Java 生态候选（按承接面分三类）

> ⚠️ **本节不锁定版本。** 我实测过 Maven Central 检索接口，其索引对该 group 明显陈旧
> （返回的最新 langchain4j 时间戳为 2025-05），**不足以支撑"当前最新版本"的断言**。
> 按 §4 纪律，**未核实的版本号不写进政策文件**。

| 承接面 | 真源对应 | Java 候选 | 成熟度 |
|---|---|---|---|
| 模型网关 | `common/trpcagent/model`（OpenAI 兼容，`gateway.go:12-20`） | Spring AI / LangChain4j / **直连 OpenAI Java SDK** | 三者皆可。⚠️ 真源用的是 `openai-go v1.12.0` + `v3.16.0` **两个大版本并存**（`go.mod:20-21`），本身有版本债 |
| 工具注册 | `common/skillsys`（`tool.go:13-18`）+ `SkillRegistry`（`core/skills.go:49-52`） | Spring AI `@Tool` / LangChain4j `@Tool` | 两者皆可。⚠️ **语义相反**：`skillsys.Register` 是**覆盖**（`registry.go:34`），`SkillRegistry.Register` 是**重名报错**（`skills.go:75`）。**Java 侧必须显式选一并留档** |
| 会话/记忆 | `session` / `memory`（只读，`readonly.go:32-34` 的 mutation 全部 fail closed） | 自建（PG-backed）+ 框架 ChatMemory | 自建成本低——真源本身就是 inmemory + Postgres 两套并存 |
| 遥测 | `common/telemetry`（OTel，`telemetry.go:36-67`） | **Micrometer + OpenTelemetry Java** | ✅ **与框架无关**，直接对等，无风险 |
| **Graph / Runner 装配** | `internal/trpcagent/runner/factory.go:18-30`、`agent/graphagent` | **无直接对等物** | 🔴 **这是唯一的真缺口** |
| 知识/RAG | `trpc-agent-go/knowledge`（`pipeline/{graph,retriever,embedder}.go`） | Spring AI RAG / LangChain4j RAG | 可承接，但语义不保证等价 |

### 6.5 三个待选方案

| | 方案 A：自建运行时 | 方案 B：Spring AI / LangChain4j 承接 | 方案 C：混合（推荐评估） |
|---|---|---|---|
| **做法** | 在 Engine 内自建一套与 tRPC-Agent-Go 同构的最小运行时 | 用框架现成能力替代 | 数据面与遥测用标准件；只自建 Graph/Runner 装配与技能系统 |
| **对齐度** | 最高，可逐条对齐 Go 行为 | 最低，框架抽象会强行改写行为 | 高 |
| **工作量** | 最大 | 最小 | 中 |
| **框架 idio 风险** | 无 | **最高**——正是 §4.2 要防的「无出处 idiomatization」 | 可控 |
| **与 §4.1 的相容性** | ✅ | ⚠️ 需为每处框架替换留档 | ✅ |
| **主要风险** | 自建质量；`session.Clone` 类并发问题（真源已在 CI 撞过） | 语义漂移无处取证 | 边界划分本身要设计对 |

**⇒ 结论栏：`UNRESOLVED`。等用户拍板 A / B / C。**
**在此之前，Engine 不得开工 `internal/trpcagent/` 的复刻。**

### 6.6 与框架无关、现在就能定的 Java 侧结论

这几条**不依赖上面的方案选择**，可直接作为政策执行：

1. **`runtime.Runtime` 必须改名。** 真源 `service/common/runtime/runtime.go:2` 的 package 名是 `runtime`，
   `Runtime` 类型在 `runtime.go:62` ⇒ **与 `java.lang.Runtime` 直接撞名**。建议 `AgentRuntime`。
2. **`SubjectRef` 必须合并成单一类型。** `usermodelcontract/contract.go:24` 与 `runtime/runtime.go:33`
   是**两个独立同构 struct，无类型别名、无转换函数**，且两侧各有独立的 `UserKey()`/校验逻辑
   ⇒ Java 侧保留两份就是埋雷。
3. **`common` 内部无循环依赖，是严格 DAG**（已用 import 图实测）⇒ Java 侧拆包**不会**引入环。
   但真源 `retrieval` 的 4 个内核包**全部 fan-in** 到 `artifacts` + `wire/representation` + `corpus`
   （`dense.go:13-15`、`sparse.go:14-16`、`multivector.go:14-17`、`three_lane_parity/adapter.go:11-16`）
   ⇒ **若把 `artifacts.Store` 和 `wire/representation` 合并成单一大包，会同时污染 3 个「本应纯算法」的包**。
   **Java 拆包的真实风险在这里，不在环。**
4. **`common/skillsys` 是薄适配层，不是运行时。** 真源的 Go 代码里**未见 `skillsys.Tool` → `trpctool.Tool`
   的适配器**（未能定位）⇒ 两套 tool 抽象是并存的，不是分层。
5. **`common/trpcagent` 的 5 个子包中 3 个零测试**（`session`/`event`/`telemetry`）；
   已有的 2 个是「真框架类型签名 + 本地 stub」，**无真实网络/DB**。
   ⚠️ 这与 `framework_wiring_test.go:121-143`「装配文件必须配套测试」的强制要求形成对比——
   **该要求只覆盖 5 个装配包，不覆盖 `common/trpcagent`** ⇒ 这是真源的治理盲区，Engine 不得继承。

---

## 七、推进路线

> **本节是计划，不是承诺。** 每个 phase 的起手动作与完成信号都写死，避免"边写边发现跑不起来"。

### 7.0 每个模块接手的第一件事

⚠️ **先让它真的启动一次**（最小 `@SpringBootTest` + 真容器），**不是先写业务单测**。

> 这条不是抄来的教条，是 RAG 用 21 条全绿测试换来的教训：
> `message-rpc` 有 **21 条测试且全绿，却一次都没真正启动过**，
> 补测试的过程当场挖出 **3 条「模块根本起不来」级缺陷**
> （Bean 名撞车导致上下文起不来 / 缺 `@MapperScan` 导致 3 个 Mapper 注册不上 / 缺 Flyway 导致迁移从不执行）。

### 7.1 建议的 phase 切分

| Phase | 内容 | 完成信号 |
|---|---|---|
| **Phase 0** | **建仓骨架**：parent pom（§1.2 版本矩阵）+ 多模块 + **守护测试模块先落地**（§5.5 P0 三项） | `mvn -o clean verify` 通过，且**故意违规能变红**（变异验证） |
| **Phase 1** | **IDL 对齐**：3 个 proto + 4 个 `.api` 逐字段冻结 + proto 守护测试 | 双侧 `.proto` 逐字节等价 + 字段号冻结测试变红验证 |
| **Phase 2** | **`common` 公共能力先行**（22 个子包按依赖 DAG 分批），**只做框架无关的部分**（§6.6） | 各包单测 + `AgentRuntime`/`SubjectRef` 收敛完成 |
| **Phase 3** | **search 域端到端打通**（proto → rpc → api → 真容器） | 真 PG + 真 Milvus 启一次；双侧探针对拍通过 |
| **Phase 4** | **trpcagent / 技能系统**（**依赖 §6.5 方案拍板**） | 方案落地 + 守护测试补齐 |
| **Phase 5** | **recommend 域**（最大，61k 行，需先按 `core/` 子域再切） | 分批交付，每批双侧探针 |
| **Phase 6** | **async 域**（含 4 个 cmd 二进制）+ Kafka 链路 | 真 Kafka 容器对拍 |
| **Phase 7** | **§3 的 6 项完成度**逐项处置 | 每项有明确签字（复刻 / 补齐 / 留档） |

⚠️ **Phase 2 先行是继承了 RAG 项目的既定节奏**（「先对齐再翻译、IDL 优先、公共能力先行」）。
**但本项目与 RAG 有一处关键差异**：`common` 的 22 个子包里，
**框架相关的只有 `trpcagent` / `telemetry` / `runtime` 三包**（§6.1），
其余 19 包是纯算法/IO ⇒ **Phase 2 可以做得很深而不被 §6.5 的未决问题卡住。**

### 7.2 验证基建（Phase 0/1 必须同步建）

| 工具 | 用途 | 对标 |
|---|---|---|
| **双侧语义探针** | Go 侧镜像跑真源包 + Engine 侧跑**真实** mapper，**两侧同时跑** | RAG 的 `tools/semantic-probe`（23 例双侧对拍） |
| **变异测试** | 证明守护测试/单测**不是假绿** | RAG 的变异验证（改错实现 → 必须变红） |
| **真容器对拍** | 响应头等 MockMvc 测不到的层 | RAG 的 `tools/difftest`（真 Tomcat 读原始响应头） |

⚠️ **三种验证缺一不可**：双侧探针证明**期望值对**，变异测试证明**测试不是假绿**，
真容器证明**层次没对齐**。RAG 实测：`MockHttpServletResponse` 与真实 Tomcat 在响应头时序上行为不同，
**MockMvc 全绿但 curl 是空的**。

---

## 八、待决策清单（UNRESOLVED）

> 每条都必须有证据或明确标注 `UNKNOWN`，**不得靠推断填表**。

| # | 事项 | 状态 | 阻塞什么 | 备注 |
|---|---|---|---|---|
| **D-1** | **tRPC-Agent-Go 承接方案 A/B/C** | 🔴 **UNRESOLVED** | Phase 4 全部 | §6.5。已确认官方无 Java 版、有 Python 版先例、Apache-2.0 |
| **D-2** | search 域**两套 HTTP 面**如何处置（goctl 的 `search/api` vs dev 的 `rpc/cmd/search_api`） | UNRESOLVED | Phase 3 | §2.2 |
| **D-3** | ORM 选型（MyBatis-Plus 对齐 RAG？还是别的） | UNRESOLVED | Phase 0 | 真源是 GORM v1.31.2 |
| **D-4** | §3 的 6 项**逐项处置**：复刻假实现 / 补真实现 / 签字留档 | UNRESOLVED | Phase 7 | 🔴 **静默选一种 = 无出处 idiomatization** |
| **D-5** | `etc/config.yaml` 的真源配置值（PG DSN / Milvus / Kafka / LLM key） | **UNKNOWN** | Phase 3 | 该文件**不在仓库**，无法取证。需向真源 owner 索取 |
| **D-6** | 守护测试的**自包含 vs 外部库** | UNRESOLVED | Phase 0 | §5.3 B-8 建议**自包含** |
| **D-7** | `knowledge/vectorstore.Store` 是否存在域外实现 | **UNKNOWN** | Phase 3 | 盘点时 grep 在 40 条上限处截断，**未能排除遗漏** |
| **D-8** | `skillsys.Tool` → `trpctool.Tool` 是否存在适配器 | **UNKNOWN** | Phase 4 | 全仓未能定位；两套 tool 抽象疑似并存而非分层 |
| **D-9** | GitHub 远端 branch protection 是否把 `go / test` 设为 required check | **UNKNOWN** | — | 仓内无 ruleset 配置，**本地无法确认** |
| **D-10** | `sea-guards` 是否有未推送的本地改动 | **UNKNOWN** | D-6 | `.gitignore:27` 忽略 `sea-guards/`，工作树无此目录 |

---

## 九、证据纪律（本文件自身的纪律）

> **这份 AGENTS.md 里每一条关于 Go 的断言都带 `文件:行号`。这是它能当判定基准的唯一原因。**

**强制的四条**：

1. **本文件的每个 Go 事实断言必须带 `文件:行号`。** 无行号的论断按 `UNKNOWN` 处理。
2. **「文件已存在 / 有 N 行 / 有 M 个测试」同样是需要证据的断言。**
   RAG 曾把 3 份不存在的 `HUMAN_ONLY.MD` 写成"已存在"，**用 glob 实测才发现**。
   ⇒ 改任何清单前先跑一次 `glob`/`wc -l`，**不要照抄上一版**。
3. **否定式断言（「Go 没有 X」「不存在 X」）必须去被否定的一侧找正面证据。**
   §6.3 是正面范例：用 GitHub API 的 `total_count` 核实，而不是"我没搜到"。
4. **状态表里的「最后核对」日期必须由一次真实执行产出**——写代码、读代码、推断**都不算核对**。
   无日期或日期过期的行，一律按 `UNKNOWN` 处理。

**已知边界（不得越读）**：

- §2 / §3 的全部结论来自 **2026-10-22 对 git HEAD `5f868bd` 的静态读取**，
  **未执行任何构建或测试**。真源仍在开发中（git log 显示 2026-10-20 前后仍在改动）。
- §5 的守护测试分析**已读完 `sea-guards` 扫描器全文**（公开仓，main 分支
  tree SHA `5b829392ef853072a3ff216f3ec0c295f3d7d80d`，仅 2 个 commit）。
- §6.3 的 Java 侧核实**只覆盖「官方有无对等物」**；第三方框架的版本、能力边界**未逐项核实**。
- 本文件**不含任何关于 Engine 代码的结论**——因为 Engine 目前零行代码。

---

## 附录 A：真源常用路径速查

```
Sea-BreakTheWaves/
├── api/                      4 个 go-zero .api IDL（search/async/recommend/recommend-admin）
├── proto/                    3 个 proto（sea.search.v1 / sea.recommend.v1 / sea.async.v1）
├── service/
│   ├── search/{api,rpc}      23k 行
│   ├── recommend/{api,rpc}   61k 行（最大域）
│   ├── async/{api,rpc}       58k 行（含 4 个 cmd 二进制）
│   └── common/               22 个子包（无环，严格 DAG）
├── deploy/docker-compose.yaml  19 个服务（PG/Redis/Neo4j/Kafka/MinIO/Milvus/ES/…）
├── testdata/index/seeds/corpus/  20 篇语料种子文档
├── framework_wiring_test.go   ┐
├── naming_test.go             │ 5 个架构守护测试
├── schema_ownership_test.go   │ （2 个只是 sea-guards 的薄装配）
├── shared_symbol_test.go      │
├── size_ratchet_test.go       ┘
├── naming_allowlist.json      1 条
└── size_allowlist.json        5 条（全部零余量）
```

⚠️ **唯一例外提醒**：`common/trpcagent` 是**适配层**，
`service/*/rpc/internal/trpcagent/` 才是**运行时**。**不要写反。**

## 附录 B：政策来源与继承边界

| 来源 | 继承什么 | 不继承什么 |
|---|---|---|
| **RAG** `AGENTS.md` | §4 的全部纪律、§5.3 的假绿教训、§7.2 的验证基建思路 | 任何状态结论、任何表名/proto/错误码、§1.2 以外的版本矩阵 |
| **Sea-BreakTheWaves** | 全部行为语义的**唯一判定标准** | 其**假实现与未接线代码**不是「标准」，是 §3 的记账对象 |
| **RAG `README.md`** | 无 | ⚠️ 不可作为结构依据（§5.3 已实证其与实际脱节） |