# 事件驱动架构与消息协作规范

> 状态：建议方案 / v0.1，待技术选型与业务一致性评审  
> 更新时间：2026-11-20  
> 上游文档：06 业务规则与流程、07 数据架构、13 微服务系统架构设计、14 微服务边界与 API 接口契约

## 1. 文档目标

定义微服务之间的异步协作机制，覆盖事件分类、统一事件信封、可靠发布、幂等消费、顺序与乱序、重试、死信、重放、Saga、可观测性和安全治理。

事件驱动架构用于解耦服务，不取代同步API。需要立即返回业务结果或必须由权威服务原子校验的操作，仍使用同步命令接口。本文不预设具体消息中间件。

## 2. 核心原则

1. 业务事实只能由资源所属的权威服务产生。
2. 事件表达已经发生的事实，不把请求或意图伪装成成功结果。
3. 业务数据与待发布事件必须在同一数据库事务中提交。
4. 默认至少一次投递，消费者必须幂等；不承诺端到端恰好一次。
5. 跨服务流程采用最终一致性、超时和补偿，不依赖跨库分布式事务。
6. 事件契约必须版本化，并纳入CI检查。
7. 事件只携带下游所需数据，不携带密码、令牌、完整证件、支付密钥或不必要个人信息。
8. 交易核心链路与市场情报采集链路隔离资源。
9. 消费者不得直接写入生产者数据库。
10. 事件发布、消费、重试和重放均应可追踪、可审计、可恢复。

## 3. 事件域与主题规划

| 事件域 | 生产者 | 典型消费者 |
|---|---|---|
| identity.organization | identity / organization | provider、审计、通知 |
| provider | provider | matching、运营分析 |
| catalog | catalog | demand、搜索索引 |
| demand | demand | matching、通知、指标 |
| matching | matching | quotation、通知 |
| quotation | quotation | contract、通知 |
| contract | contract | service-order、payment、通知 |
| service-order | service-order | payment、aftersales、通知 |
| project | project | contract、项目分析 |
| payment | payment | service-order、aftersales、对账 |
| aftersales | aftersales | payment、通知、运营分析 |
| research.plan | search-orchestrator | collection-connector、workbench |
| research.collection | collection-connector | page-processing、workbench |
| research.document | page-processing | market-entity、signal |
| research.entity | market-entity | provider审核工作台、城市分析 |
| research.city | city-intelligence | research-workbench、运营分析 |

主题命名建议采用环境、业务域、事件类型的统一规则。初期可按业务域合并主题；当吞吐、权限、保留期或隔离要求明显不同时再拆分。生产、预发、开发环境必须隔离。

## 4. 统一事件信封

~~~json
{
  "id": "evt_01JXXX",
  "type": "demand.published",
  "schemaVersion": 1,
  "occurredAt": "2026-11-20T08:30:00Z",
  "publishedAt": "2026-11-20T08:30:01Z",
  "producer": {
    "service": "demand-service",
    "version": "1.4.0"
  },
  "subject": {
    "type": "demand",
    "id": "dem_xxx",
    "version": 3
  },
  "correlationId": "corr_xxx",
  "causationId": "cmd_xxx",
  "traceparent": "00-...",
  "data": {
    "cityCode": "320500",
    "serviceCategoryId": "category-electrical",
    "visibility": "LIMITED"
  }
}
~~~

| 字段 | 必需 | 说明 |
|---|---|---|
| id | 是 | 全局唯一事件ID，建议UUIDv7或等效方案 |
| type | 是 | 稳定事件类型，如demand.published |
| schemaVersion | 是 | 该事件类型的数据契约版本 |
| occurredAt | 是 | 业务事实发生时间 |
| publishedAt | 是 | 进入消息系统的时间 |
| producer.service | 是 | 生产者服务名 |
| subject.type/id | 是 | 事件描述的业务资源 |
| subject.version | 建议 | 资源版本，用于识别旧事件和乱序 |
| correlationId | 是 | 跨服务业务流程关联ID |
| causationId | 建议 | 导致本事件的命令或上游事件ID |
| traceparent | 建议 | 分布式追踪上下文 |
| data | 是 | 事件类型专属业务字段 |

组织上下文可按需增加，但组织ID不能替代消费者的授权校验。大体积文档和附件只传引用及内容摘要，不直接塞入消息。

## 5. 事件命名和事实语义

建议采用领域对象加已发生事实的命名，例如：

- demand.created、demand.published
- quotation.submitted、quotation.accepted
- contract.activated
- service-order.created、service-order.work-started
- service-order.completion-applied、service-order.acceptance-passed
- payment.intent-created、payment.provider-confirmed、payment.refund-confirmed
- research.collection-started、research.collection-completed
- research.document-parsed、research.entity-review-completed
- research.city-score-calculated

避免使用order.updated、payment.done、task.finished等无法表达具体事实的名称。

payment.intent-created只代表支付意图创建；支付成功必须以经过验签、关联校验和幂等处理的支付机构结果为依据。客户端跳转成功、同步接口超时或用户截图都不能单独作为支付成功事实。

## 6. 交易主链路事件流

~~~text
demand.published
  -> matching.job-created
  -> matching.invitation-created
  -> provider.invitation-responded
  -> quotation.submitted
  -> quotation.accepted
  -> contract.draft-created
  -> contract.activated
  -> service-order.created
  -> service-order.work-started
  -> service-order.completion-applied
  -> service-order.acceptance-passed
  -> payment.provider-confirmed
~~~

关键规则：
- 需求发布事件触发匹配任务；匹配消费者必须幂等。
- 报价接受由quotation-service原子校验并提交，不能仅靠消费事件决定接受结果。
- 合同生效事件应包含合同ID、版本、签约主体引用和生效时间，不携带合同全文。
- service-order-service消费合同生效事件后创建订单；通过合同版本等业务唯一键防止重复创建。
- 支付计划由合同和有效变更约定驱动；履约事件可触发付款条件检查，但不能证明支付成功。
- 售后、退款、争议和订单终止分别建模，不能互相覆盖历史事实。

## 7. 市场情报事件流

~~~text
research.plan-created
  -> research.collection-requested
  -> research.collection-started
  -> research.document-discovered
  -> research.collection-completed
  -> research.document-parsed
  -> research.extraction-produced
  -> research.entity-candidate-created
  -> research.entity-reviewed
  -> research.city-score-calculated
~~~

采集事件至少关联来源ID、任务ID、连接器版本、采集时间、结果数量、失败数量和质量摘要。来源失败应局部隔离，不应丢失其他来源的结果。来源停用或授权撤销后，必须阻止新任务并按数据治理规则处置已有数据。

市场研究发现的企业候选不能自动成为平台可交易服务商；必须由provider-service独立完成主体、技能和必要资质核验。

## 8. Transactional Outbox：可靠发布

### 8.1 问题

业务数据库提交和消息发布是两个独立动作。若先提交数据库再发布消息，进程可能在两步之间崩溃，造成事件丢失；若先发消息再提交数据库，则可能出现事件已发出但业务事务回滚。

### 8.2 推荐实现

在同一个本地数据库事务中：
1. 更新业务资源；
2. 写入outbox_events待发布记录；
3. 提交事务。

独立Publisher读取待发布记录并投递消息系统，成功后更新发布状态。失败则按策略重试。

建议字段：

| 字段 | 说明 |
|---|---|
| event_id | 事件唯一ID，主键 |
| aggregate_type / aggregate_id | 聚合根类型和ID |
| aggregate_version | 聚合根版本 |
| event_type / schema_version | 事件类型和契约版本 |
| payload | 事件信封或业务数据 |
| occurred_at / created_at | 业务发生时间和记录创建时间 |
| published_at | 成功投递时间 |
| publish_status | PENDING / PUBLISHED / FAILED |
| attempt_count | 投递尝试次数 |
| next_attempt_at | 下次尝试时间 |
| last_error_code | 最近错误码 |

Publisher采用批量拉取和有界并发。多实例可用行锁、租约或分片策略协调抢占；即便重复投递，消费者仍必须幂等。监控Outbox积压量、最老记录年龄和失败率，禁止无上限重试。

## 9. Transactional Inbox：幂等消费

消费者处理事件时，在本地事务中：
1. 检查Inbox是否已有该eventId。
2. 已成功处理则直接确认消息。
3. 未处理则写入Inbox记录。
4. 执行本地业务更新。
5. 如需发布新事件，同时写入本服务Outbox。
6. 将Inbox标记为已处理。
7. 提交事务后确认消息。

建议字段：

| 字段 | 说明 |
|---|---|
| consumer_name | 消费者逻辑名称 |
| event_id | 上游事件ID |
| event_type | 事件类型 |
| received_at / processed_at | 接收和处理时间 |
| processing_status | RECEIVED / PROCESSING / PROCESSED / FAILED |
| attempt_count | 处理次数 |
| last_error_code | 最近错误码 |
| payload_hash | 事件摘要 |

唯一约束建议为consumer_name + event_id。幂等不能只依赖应用层先查再写，必须配合唯一约束、业务唯一键或版本条件，防止并发重复执行。

## 10. 顺序、分区和乱序

不假设消息系统全局有序。同一业务资源需要顺序处理时，使用aggregateId作为分区键或消息分组键，并在消费者检查subject.version。

建议分区键：
- 需求：demandId
- 报价：quotationId
- 合同：contractId
- 订单：orderId
- 支付意图：paymentIntentId或外部交易ID
- 采集任务：taskId

发现版本跳跃时，消费者应判断能否跳过中间版本；不能跳过则暂缓处理、重试或回源查询。旧事件不得覆盖新状态。跨聚合根不保证顺序，需要协调的流程由Saga/Process Manager维护状态。

## 11. 重试、死信和异常分类

| 错误类别 | 示例 | 处理方式 |
|---|---|---|
| 瞬时故障 | 网络超时、依赖503 | 有界指数退避和随机抖动 |
| 限流 | 下游429、来源限频 | 遵循Retry-After或来源策略 |
| 暂时数据缺失 | 上游数据尚未到达 | 延迟重试或等待后续事件 |
| 永久业务错误 | 状态不允许、主体不存在 | 不盲目重试，进入异常处理 |
| 契约错误 | 必需字段缺失、未知不兼容版本 | 隔离并告警 |
| 安全错误 | 回调验签失败、来源不可信 | 拒绝处理并安全告警 |

可将1秒、5秒、30秒、2分钟、10分钟作为初始退避档位参考，最终次数和窗口按业务SLA配置。支付、合同和普通分析事件不得共用完全相同的重试策略。

超过上限进入死信/隔离队列，记录原eventId、事件类型、消费者、失败原因、次数和时间。处理流程为告警、定位、修复、选择重放或补偿、审批留痕和结果验证。支付、退款、合同相关死信应执行更严格审批，必要时双人复核。

## 12. Saga：跨服务业务流程

Saga由多个本地事务和流程状态组成，每一步都有成功事件、超时和补偿动作。

### 12.1 报价接受到订单创建

1. quotation-service原子接受报价并发布quotation.accepted。
2. contract-service创建合同草案。
3. 合同签署完成后，contract-service确认签署结果并发布contract.activated。
4. service-order-service幂等创建履约订单。
5. notification-service发送通知。

异常处理：
- 合同草案创建失败：重试，超时后人工处理。
- 签署失败或过期：合同不生效或按合同规则作废。
- 订单创建失败：保留已生效合同事实，持续重试或人工补偿，不得删除已生效合同。
- 通知失败：独立重试，不影响合同和订单事实。

### 12.2 支付流程

1. 业务服务依据有效付款计划请求payment-service创建支付意图。
2. payment-service持久化支付意图及外部交易关联。
3. 按支付机构产品发起支付。
4. 处理同步响应和异步回调；回调必须验签、校验商户和交易关联。
5. 支付结果待确认时，通过支付机构查询接口核实。
6. 确认成功后发布payment.provider-confirmed。
7. 下游更新付款进度和对账读模型。

支付超时不等于支付失败。用户重试不能造成重复实际扣款；通过付款计划ID、幂等键和支付机构交易号关联。

### 12.3 市场采集流程

研究计划创建任务，连接器执行并上报结果，解析服务处理文档，实体服务治理实体，城市情报服务计算指标。各阶段独立重试，一个来源失败不应导致其他来源结果丢失。

## 13. 事件契约演进

- 每种事件类型维护独立Schema和版本。
- 增加可选字段通常可保持兼容，消费者必须容忍未知字段。
- 删除字段、改变类型、金额单位、枚举含义或事件语义属于破坏性变更。
- 破坏性变更应发布新版本，并提供双版本过渡或迁移方案。
- 消费者不得假设枚举永远只有当前值；未知值应安全隔离或走兼容分支。
- 生产者发布前运行Schema校验；消费者变更前运行历史事件样本回放测试。
- CI执行Schema兼容检查、契约测试和消费者构建验证。

## 14. 存储、保留和重放

消息系统保留期、业务审计留存期、原始市场数据留存期分别制定。不能因消息过期而丢失财务或法律所需证据，也不能为无限重放而无限保存个人信息。

生产重放必须：
- 仅允许授权人员发起，指定范围、消费者、时间窗口和原因；
- 先在隔离环境或影子消费者验证；
- 限速、分批、可暂停；
- 区分重新投递原事件与生成新的业务修复命令；
- 对支付、短信、签署等外部副作用提供防重保护，必要时关闭副作用进行影子回放；
- 留存审批、操作者、时间、范围和结果审计。

## 15. 资源隔离、监控和安全

交易核心事件与市场情报事件使用独立资源配额，必要时独立集群。市场采集和文档解析积压不得挤占报价、合同、支付回调等关键消费资源。

监控指标至少包括：
- 发布成功率、发布延迟；
- 各主题积压量、最老消息年龄；
- 消费吞吐、消费延迟、处理失败率；
- Outbox待发布数量和最老记录年龄；
- Inbox重复事件比例；
- 重试次数、死信数量和死信年龄；
- Saga超时数及各步骤耗时；
- 支付回调接收、验签失败、待确认和对账差异；
- 采集排队时长、来源限频次数和解析失败率。

消息通道启用传输加密、服务身份认证和最小权限ACL。生产者仅能写授权主题，消费者仅能读授权主题。消息平台管理权限与应用读写权限分离。事件中的用户/组织ID仅用于关联，不能替代资源授权。

## 16. MVP落地顺序

第一阶段不必引入复杂事件平台。建议在模块化单体中先建立领域事件接口和Outbox表，独立采集Worker使用任务队列。服务独立部署后，再让Outbox Publisher连接正式消息中间件。

最小事件集：
- demand.published
- quotation.submitted
- quotation.accepted
- contract.activated
- service-order.created
- service-order.completion-applied
- service-order.acceptance-passed
- payment.provider-confirmed
- payment.refund-confirmed
- research.collection-requested
- research.collection-completed
- research.document-parsed

最小工程能力：
1. 统一事件信封和Schema版本。
2. Outbox可靠发布。
3. Inbox幂等消费。
4. 有界重试和死信记录。
5. eventId、correlationId和traceId贯通。
6. 关键事件审计和人工重放。
7. 交易与市场情报消费资源隔离。

## 17. 验收标准

- 业务事务提交后进程崩溃，Outbox仍能最终发布事件。
- 消息重复投递不会重复创建订单、合同流程或退款请求。
- 消费者处理期间崩溃，不出现已确认消息但本地事务未提交的静默丢失。
- 乱序事件不会覆盖更新版本的业务状态。
- 下游不可用时，生产者仍能完成本服务允许完成的本地事务，并由异步机制恢复。
- 超过重试上限的事件进入死信并告警。
- 死信重放具备权限、限速和审计。
- 支付超时保持待确认，不被错误解释为成功或失败。
- 市场采集积压不影响交易核心事件链路。
- 未授权消费者无法读取敏感主题或通过事件越权访问个人信息。

## 18. 待确认事项

- 最终消息中间件及托管/自建方式。
- 交易与市场情报是否采用独立消息集群。
- 各业务域事件保留期和审计留存期。
- 关键事件顺序保证范围与分区策略。
- 支付机构回调和主动查询的状态优先级及对账机制。
- 生产死信重放审批等级。
- Schema采用JSON Schema、Avro或Protobuf。
- Outbox采用轮询、CDC或消息平台事务能力的最终实现。

**结论：** 事件驱动架构必须建立在清晰的服务数据所有权之上。先通过Outbox、Inbox、幂等消费和可审计补偿建立可靠性，再根据真实吞吐和运维能力选择消息中间件。事件系统不能替代支付合规、合同证据或业务状态机。
