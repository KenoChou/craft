# 微服务边界与 API 接口契约

> 状态：建议方案 / v0.1，待业务规则、权限模型和支付机构接口评审  
> 更新时间：2026-11-20  
> 上游文档：06 业务规则与流程、07 数据架构、13 微服务系统架构设计

## 1. 文档目标

本文将微服务逻辑架构落实为服务边界、调用关系、API资源、请求/响应规范、错误码和契约治理规则。它是后续生成OpenAPI、事件Schema和自动化契约测试的基线，不是最终数据库字段清单。

覆盖两条主线：
- **交易主线**：身份与组织、服务商、需求、撮合、报价、合同、订单履约、支付协作、售后。
- **市场情报主线**：数据源登记、搜索任务、采集执行、文档解析、实体/POI/线索、城市指标和人工核验。

支付机构具体产品、合同法律文本、工程资质判定规则、个人信息处理合法性结论及具体技术中间件，均待专项评审。

## 2. API 总体规范

### 2.1 通用约定

| 项目 | 约定 |
|---|---|
| 外部API基础路径 | /api/v1 |
| 内部服务API | /internal/v1，仅内网和服务身份可访问 |
| 协议 | HTTPS + REST/JSON；内部高频场景可评估gRPC |
| 字段命名 | JSON使用lowerCamelCase |
| 时间 | RFC 3339 / ISO 8601，必须带时区；数据库内部建议UTC |
| 金额 | 使用整数最小货币单位或明确精度的十进制字符串，禁止浮点数 |
| ID | 不透明、不可预测的字符串ID，不暴露数据库自增主键 |
| 分页 | 游标分页优先，返回nextCursor和hasMore |
| 排序 | 仅允许白名单字段和方向 |
| 幂等 | 创建订单、接受报价、支付请求、退款等关键命令必须支持 |
| 并发控制 | 更新有版本竞争的资源使用version或ETag/If-Match |
| 错误 | 稳定错误码，不依赖错误文案进行客户端逻辑判断 |
| 追踪 | 所有请求携带/返回X-Request-Id，服务间传播traceparent |

### 2.2 通用请求头

| Header | 用途 | 要求 |
|---|---|---|
| Authorization | 用户访问令牌或服务身份令牌 | 受保护接口必需 |
| X-Request-Id | 请求唯一标识 | 网关缺失时生成 |
| Idempotency-Key | 防止重复执行命令 | 关键写操作必需 |
| If-Match | 乐观并发控制 | 更新有版本竞争的资源时使用 |
| traceparent | W3C分布式追踪上下文 | 服务间传播 |
| Accept-Language | 返回文案语言 | 可选，不改变业务语义 |

用户令牌与服务身份令牌必须区分。内部服务不得伪造用户身份；代表用户执行操作时，必须传递经过验证的主体上下文，并保留原始操作者和执行服务信息。

### 2.3 通用响应

普通资源响应建议：

~~~json
{
  "data": {
    "id": "opaque-resource-id"
  },
  "meta": {
    "requestId": "req_xxx",
    "timestamp": "2026-11-20T08:00:00Z"
  }
}
~~~

列表响应建议：

~~~json
{
  "data": [],
  "page": {
    "nextCursor": null,
    "hasMore": false
  },
  "meta": {
    "requestId": "req_xxx"
  }
}
~~~

异步任务创建返回202 Accepted，提供任务ID和查询地址；不得将任务入队描述成采集或解析已经完成。

### 2.4 通用错误格式

~~~json
{
  "error": {
    "code": "RESOURCE_VERSION_CONFLICT",
    "message": "资源版本已变化，请刷新后重试",
    "details": [],
    "retryable": false
  },
  "meta": {
    "requestId": "req_xxx"
  }
}
~~~

HTTP状态码约定：
- 400：请求格式或参数错误。
- 401：身份认证失败或令牌无效。
- 403：主体已认证但无权访问。
- 404：资源不存在或按安全策略不可见。
- 409：状态冲突、重复命令或版本冲突。
- 422：字段格式正确但不满足业务规则。
- 429：触发限流。
- 500：未预期服务错误。
- 502/503/504：依赖服务或网关异常、超时。

不得将堆栈、SQL、密钥、内部地址或个人敏感数据返回客户端。

## 3. 服务边界与权威数据

| 服务 | 权威管理对象 | 不负责的事情 |
|---|---|---|
| identity-service | 登录凭证、认证会话、账户安全状态 | 企业成员业务授权、订单权限规则 |
| organization-service | 个人/企业组织、成员关系、角色授权 | 用户密码、合同签署事实 |
| provider-service | 服务商档案、技能、服务范围、准入核验状态 | 市场线索候选实体、订单状态 |
| catalog-service | 服务类目、服务项、计价单位、区域字典 | 某个订单最终成交价格 |
| demand-service | 需求内容、发布状态和需求版本 | 服务商匹配结果、报价、订单履约 |
| matching-service | 邀请、候选匹配和派单任务 | 服务商准入事实、最终报价 |
| quotation-service | 报价内容、报价版本和接受结果 | 合同正文和订单履约状态 |
| contract-service | 合同版本、合同方、签署状态和证据索引 | 支付机构资金状态、施工过程状态 |
| service-order-service | 住宅服务订单、施工记录、变更、验收 | 外部支付机构交易事实 |
| project-service | 商业工程项目、专业包、进度申报 | 住宅服务订单状态 |
| payment-service | 支付意图、退款请求、外部交易映射、对账差异 | 用户资金账户、平台自由划转 |
| aftersales-service | 售后案件、投诉和争议处理 | 原订单或支付记录的直接改写 |
| research-source-service | 数据源、许可依据、采集限制 | 绕过来源限制的采集 |
| search-orchestrator-service | 研究计划、搜索任务和调度状态 | 原始文档最终解析内容 |
| collection-connector-service | 采集执行记录和连接器运行状态 | 城市市场结论 |
| page-processing-service | 文档解析版本和抽取结果 | 企业/服务商最终核验状态 |
| market-entity-service | 市场实体候选、实体合并和来源关联 | 交易服务商准入 |
| market-poi-service | POI、地理信息和地点去重 | 地图供应商的权威数据 |
| market-signal-service | 需求、价格、竞争和工程项目线索 | 真实平台订单 |
| city-intelligence-service | 城市指标、评分快照和数据质量 | 直接决定平台商业扩张 |
| research-workbench-service | 研究项目、人工核验和研究结论 | 交易核心业务状态 |

一个业务实体只能有一个权威写入服务。其他服务通过API、事件或本地读模型获取数据，不直接写入该服务数据库。

## 4. 交易业务 API

以下路径为建议契约。Gateway可统一暴露外部API；内部权威写操作必须路由至资源所属服务。

### 4.1 Identity 与 Organization

| 方法 | 路径 | 说明 | 关键约束 |
|---|---|---|---|
| POST | /api/v1/auth/sessions | 登录并创建会话 | 登录风控和验证码策略另行定义 |
| DELETE | /api/v1/auth/sessions/current | 当前会话登出 | 令牌撤销 |
| POST | /api/v1/auth/phone-verifications | 发起手机号验证 | 限频、用途绑定 |
| GET | /api/v1/me | 当前主体信息 | 只返回当前用户可见字段 |
| GET | /api/v1/organizations | 查询本人所属组织 | 按成员关系过滤 |
| POST | /api/v1/organizations | 创建组织 | 创建人成为初始管理员 |
| GET | /api/v1/organizations/{organizationId}/members | 查询组织成员 | 组织管理员权限 |
| POST | /api/v1/organizations/{organizationId}/invitations | 邀请成员 | 幂等、邀请有效期 |
| PATCH | /api/v1/organizations/{organizationId}/members/{memberId} | 调整成员角色/状态 | 最小权限、敏感操作审计 |

认证接口证明“是谁”；组织授权判断“代表哪个组织、可以做什么”。每个交易命令必须校验操作者对目标组织和资源的授权。

### 4.2 Provider 服务商

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | /api/v1/providers/applications | 提交服务商入驻申请 |
| GET | /api/v1/providers/me | 查询本人服务商档案 |
| PATCH | /api/v1/providers/me/profile | 更新服务商资料 |
| PUT | /api/v1/providers/me/service-areas | 更新服务区域 |
| PUT | /api/v1/providers/me/skills | 更新技能和服务品类 |
| POST | /api/v1/providers/me/credentials | 提交资质/能力证明材料索引 |
| GET | /api/v1/providers/{providerId} | 查询公开服务商档案 |
| POST | /internal/v1/providers/{providerId}/verification-decisions | 提交核验决定 |

服务商核验应区分身份、企业主体、技能能力、法定资质（如适用）和平台服务质量。不得用单一verified布尔值概括所有核验维度。

### 4.3 Catalog 服务目录

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/catalog/categories | 查询服务分类 |
| GET | /api/v1/catalog/services | 查询可交易服务项 |
| GET | /api/v1/regions | 查询行政区划和平台运营区域 |
| GET | /api/v1/catalog/pricing-units | 查询计价单位 |

服务目录参考价格与订单最终成交金额必须区分。报价由报价服务保存；合同金额由合同约定；支付服务按有效付款计划发起支付。

### 4.4 Demand 需求服务

| 方法 | 路径 | 说明 | 关键约束 |
|---|---|---|---|
| POST | /api/v1/demands | 创建需求草稿 | 主体、字段和区域校验 |
| GET | /api/v1/demands/{demandId} | 查询需求详情 | 资源级权限和脱敏 |
| PATCH | /api/v1/demands/{demandId} | 更新需求草稿 | If-Match版本控制 |
| POST | /api/v1/demands/{demandId}/publish | 发布需求 | 幂等、审核条件满足 |
| POST | /api/v1/demands/{demandId}/close | 关闭需求 | 状态机和权限校验 |
| GET | /api/v1/me/demands | 查询本人需求 | 游标分页 |

需求公开视图和内部详情视图应分离。对服务商展示时，不应提前泄露用户精确住址、手机号等非必要信息。

### 4.5 Matching 撮合与派单

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | /internal/v1/matching/jobs | 根据已发布需求创建匹配任务 |
| GET | /api/v1/demands/{demandId}/matches | 查询有权查看的匹配进度 |
| POST | /internal/v1/matching/jobs/{jobId}/invitations | 批量生成服务商邀请 |
| POST | /api/v1/provider-invitations/{invitationId}/accept | 服务商响应邀请 |
| POST | /api/v1/provider-invitations/{invitationId}/decline | 服务商拒绝邀请 |

匹配服务只能基于provider-service提供的有效准入和能力数据筛选候选。匹配分数属于推荐结果，不是施工质量保证或履约承诺。

### 4.6 Quotation 报价服务

| 方法 | 路径 | 说明 | 关键约束 |
|---|---|---|---|
| POST | /api/v1/demands/{demandId}/quotations | 服务商提交报价 | 邀请/开放报价规则校验 |
| GET | /api/v1/demands/{demandId}/quotations | 需求方查看报价列表 | 参与方权限 |
| GET | /api/v1/quotations/{quotationId} | 报价详情 | 参与方权限 |
| PATCH | /api/v1/quotations/{quotationId} | 更新尚未锁定的报价 | 版本控制 |
| POST | /api/v1/quotations/{quotationId}/accept | 接受报价 | 幂等、有效期和并发校验 |
| POST | /api/v1/quotations/{quotationId}/withdraw | 撤回报价 | 状态机校验 |

报价保存版本、有效期、服务范围、材料/人工说明、税费说明（如适用）、排除项和付款建议。报价接受不等于合同已签署，也不等于支付成功。

### 4.7 Contract 合同服务

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | /internal/v1/contracts | 基于已接受报价创建合同草案 |
| GET | /api/v1/contracts/{contractId} | 合同参与方查询合同 |
| POST | /api/v1/contracts/{contractId}/versions | 创建合同新版本/变更草案 |
| POST | /api/v1/contracts/{contractId}/signature-requests | 发起签署流程 |
| POST | /internal/v1/contracts/{contractId}/signature-results | 接收可信签署结果 |
| POST | /api/v1/contracts/{contractId}/terminate-requests | 发起解除/终止申请 |

合同服务保存签约主体、合同版本、版本摘要、签署方、签署时间、签署渠道和证据索引。具体电子签名服务与法律效力要求待专业确认。

### 4.8 Service Order 住宅订单履约

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | /internal/v1/service-orders | 合同生效后创建订单 |
| GET | /api/v1/service-orders/{orderId} | 查询订单 |
| GET | /api/v1/me/service-orders | 查询本人订单 |
| POST | /api/v1/service-orders/{orderId}/appointments | 预约施工时间 |
| POST | /api/v1/service-orders/{orderId}/work-starts | 施工开工登记 |
| POST | /api/v1/service-orders/{orderId}/change-requests | 发起施工范围/金额变更 |
| POST | /api/v1/service-orders/{orderId}/completion-applications | 提交完工申请 |
| POST | /api/v1/service-orders/{orderId}/acceptance-decisions | 提交验收结论 |
| POST | /api/v1/service-orders/{orderId}/rectification-requests | 发起整改 |
| GET | /api/v1/service-orders/{orderId}/timeline | 查询履约时间线 |

履约服务负责订单状态、施工节点和验收记录。合同变更涉及价款、责任或范围时，必须先完成合同变更流程，再按规则更新履约计划。

### 4.9 Project 商业工程项目

商业工程与住宅服务订单分开建模。项目、标段、投标、施工合同、进度计量和结算应分别建模，不应复用住宅订单状态机。

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | /api/v1/projects | 创建工程项目草稿 |
| GET | /api/v1/projects/{projectId} | 查询项目 |
| POST | /api/v1/projects/{projectId}/publish | 发布项目/招募承包方 |
| POST | /api/v1/projects/{projectId}/packages | 创建专业包/标段 |
| POST | /api/v1/projects/{projectId}/bids | 提交投标/承包方案 |
| POST | /api/v1/projects/{projectId}/milestones | 创建项目节点 |
| POST | /api/v1/projects/{projectId}/change-orders | 发起工程变更/签证 |
| GET | /api/v1/projects/{projectId}/progress | 查询项目进度 |

工程流程和资质门槛专项评审通过后，再固化完整接口。

### 4.10 Payment 支付协作

| 方法 | 路径 | 说明 | 关键约束 |
|---|---|---|---|
| POST | /internal/v1/payment-intents | 根据合同/订单付款计划创建支付意图 | 仅授权业务服务调用 |
| GET | /api/v1/payment-intents/{paymentIntentId} | 参与方查询支付状态 | 脱敏 |
| POST | /api/v1/payment-intents/{paymentIntentId}/confirmations | 发起支付确认/获取跳转凭据 | 按支付机构产品规则 |
| POST | /api/v1/payment-provider/callbacks/{providerCode} | 支付机构回调入口 | 验签、幂等、来源校验 |
| POST | /api/v1/refunds | 创建退款申请 | 业务授权、原交易关联 |
| GET | /api/v1/refunds/{refundId} | 查询退款状态 | 参与方权限 |
| GET | /internal/v1/reconciliation/batches | 查询对账批次 | 财务/授权服务身份 |
| POST | /internal/v1/reconciliation/batches/{batchId}/runs | 执行对账 | 幂等、审计 |

支付服务只管理支付请求、支付机构交易映射、退款请求和对账差异。不得设计平台可自由提现、转账的用户资金余额。实际资金收付、分账和结算必须以持牌机构批准的产品能力、商户协议和法律关系为准。

### 4.11 Aftersales 与 Review

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | /api/v1/service-orders/{orderId}/aftersales-cases | 创建售后案件 |
| GET | /api/v1/aftersales-cases/{caseId} | 查询售后案件 |
| POST | /api/v1/aftersales-cases/{caseId}/evidence | 添加售后证据 |
| POST | /api/v1/aftersales-cases/{caseId}/resolutions | 提交处理方案/结论 |
| POST | /api/v1/service-orders/{orderId}/reviews | 提交评价 |
| GET | /api/v1/providers/{providerId}/reviews | 查询服务商公开评价 |

投诉、争议、退款、合同解除和订单终止是相关但不同的业务对象，不得用一个cancel接口代替所有处理流程。

## 5. 市场情报 API

市场情报服务建议独立API域，例如 /api/v1/research，由研究工作台和运营后台访问；采集连接器的内部端点不得暴露给普通客户端。

### 5.1 Data Source 数据源登记

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | /api/v1/research/sources | 登记数据源 |
| GET | /api/v1/research/sources | 查询数据源目录 |
| GET | /api/v1/research/sources/{sourceId} | 数据源详情 |
| PATCH | /api/v1/research/sources/{sourceId} | 更新来源元数据/授权状态 |
| POST | /api/v1/research/sources/{sourceId}/disable | 停用数据源 |

数据源至少记录sourceId、sourceName、sourceType、provider、termsReference、permittedPurpose、accessMethod、rateLimitPolicy、retentionPolicy、owner、status和lastReviewedAt。密钥只保存在密钥管理系统，不以明文写入业务表或API响应。

### 5.2 Search Orchestrator 搜索与研究任务

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | /api/v1/research/plans | 创建城市/主题研究计划 |
| GET | /api/v1/research/plans/{planId} | 查询研究计划 |
| POST | /api/v1/research/plans/{planId}/tasks | 生成采集任务 |
| GET | /api/v1/research/tasks/{taskId} | 查询任务状态 |
| POST | /api/v1/research/tasks/{taskId}/cancel | 取消尚未执行任务 |
| POST | /api/v1/research/tasks/{taskId}/retry | 对允许重试任务发起重试 |

任务输入建议包含城市/区县代码、主题、关键词、来源白名单、时间窗口、最大结果数、采集策略版本、研究目的和发起人。任务必须受数据源授权、限频和预算约束。

### 5.3 Collection Connector 采集执行

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | /internal/v1/collection-runs | 创建采集执行 |
| GET | /internal/v1/collection-runs/{runId} | 查询采集执行 |
| POST | /internal/v1/collection-runs/{runId}/heartbeats | Worker心跳 |
| POST | /internal/v1/collection-runs/{runId}/results | 上报采集结果索引 |
| POST | /internal/v1/collection-runs/{runId}/failures | 上报失败及可重试信息 |

采集执行支持来源级并发上限、速率限制、退避、熔断、任务租约和超时回收。只允许使用登记为启用且授权范围匹配的数据源；禁止绕过登录、验证码、访问控制、付费墙或其他技术限制。

### 5.4 Page Processing 文档解析

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | /internal/v1/documents/{documentId}/parse-jobs | 创建解析任务 |
| GET | /internal/v1/parse-jobs/{jobId} | 查询解析任务 |
| GET | /api/v1/research/documents/{documentId} | 查看文档元数据 |
| GET | /api/v1/research/documents/{documentId}/extractions | 查看结构化抽取结果 |
| POST | /api/v1/research/documents/{documentId}/review-decisions | 人工确认/驳回抽取结果 |

原始内容留存必须先确认合法依据和必要性。解析结果必须带parserVersion、schemaVersion、sourceId、collectedAt和置信度；解析结果不能覆盖原始版本。

### 5.5 Market Entity / POI / Signal

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/research/entities | 查询候选企业/服务商实体 |
| GET | /api/v1/research/entities/{entityId} | 查询实体详情和来源 |
| POST | /api/v1/research/entities/{entityId}/merge-candidates | 提交实体合并建议 |
| POST | /api/v1/research/entities/{entityId}/verification-decisions | 人工核验实体 |
| GET | /api/v1/research/pois | 查询POI |
| GET | /api/v1/research/signals | 查询市场线索 |
| POST | /api/v1/research/signals/{signalId}/dispositions | 标注线索处理结果 |

市场实体和POI必须保留来源关联。实体合并应可撤销或可审计，不允许仅凭名称相似就自动认定为同一企业。线索转为平台服务商必须由provider-service重新走准入流程。

### 5.6 City Intelligence 城市指标

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/research/cities/{cityCode}/indicators | 查询城市指标 |
| POST | /internal/v1/research/cities/{cityCode}/indicator-calculations | 触发指标计算 |
| GET | /api/v1/research/cities/{cityCode}/scores | 查询城市评分快照 |
| POST | /api/v1/research/city-score-runs | 批量生成城市评分 |
| GET | /api/v1/research/city-score-runs/{runId} | 查询评分任务 |

城市评分必须同时展示指标定义版本、数据时间范围、来源覆盖、样本量、缺失率、置信度和硬性门槛结果。不能只返回一个没有解释的综合分数。

## 6. 关键请求/响应示例

### 6.1 创建需求

~~~http
POST /api/v1/demands
Idempotency-Key: demand-create-2026-0001
Content-Type: application/json
~~~

~~~json
{
  "serviceCategoryId": "category-electrical",
  "serviceItemId": "service-home-wiring",
  "region": {
    "provinceCode": "320000",
    "cityCode": "320500",
    "districtCode": "320506"
  },
  "location": {
    "addressText": "由用户填写的服务地址",
    "latitude": 31.2,
    "longitude": 120.6
  },
  "description": "住宅局部线路改造",
  "budget": {
    "minAmount": "100000",
    "maxAmount": "300000",
    "currency": "CNY"
  },
  "preferredStartAt": "2026-12-01T09:00:00+08:00"
}
~~~

金额示例仅展示字符串表达方式，实际单位必须在OpenAPI中固定并保持一致。精确地址和地理位置属于敏感信息，应按角色和业务阶段控制可见范围。生产接口应将精确地址与对外展示地址分开处理。

响应：

~~~json
{
  "data": {
    "demandId": "dem_xxx",
    "status": "DRAFT",
    "version": 1,
    "createdAt": "2026-11-20T08:00:00Z"
  },
  "meta": {
    "requestId": "req_xxx"
  }
}
~~~

### 6.2 接受报价

~~~http
POST /api/v1/quotations/quo_xxx/accept
Idempotency-Key: accept-quo_xxx-001
If-Match: "7"
~~~

~~~json
{
  "acceptedByOrganizationId": "org_xxx",
  "acceptanceNote": "确认报价范围和施工安排"
}
~~~

接口检查报价状态、有效期、报价版本、需求状态、当前操作者授权及是否已存在其他有效中选结果。并发请求只能产生一个有效接受结果；其他请求返回明确冲突，不得重复创建合同或订单。

### 6.3 创建市场采集任务

~~~http
POST /api/v1/research/plans/plan_xxx/tasks
Idempotency-Key: research-task-plan_xxx-001
~~~

~~~json
{
  "cityCode": "320500",
  "topics": ["electrical-repair", "decoration-company", "mep-contractor"],
  "sourceIds": ["source_official_api", "source_map_provider"],
  "keywords": ["水电维修", "机电安装", "装修公司"],
  "collectionWindow": {
    "from": "2026-10-01T00:00:00+08:00",
    "to": "2026-11-20T23:59:59+08:00"
  },
  "maxResults": 500,
  "purpose": "城市服务供给研究"
}
~~~

响应为202 Accepted：

~~~json
{
  "data": {
    "taskId": "task_xxx",
    "status": "QUEUED",
    "statusUrl": "/api/v1/research/tasks/task_xxx"
  },
  "meta": {
    "requestId": "req_xxx"
  }
}
~~~

任务进入队列不表示已经获得500条结果；最终数量、失败数和数据质量应通过任务详情查询。

## 7. API 权限模型

统一采用“主体 + 组织 + 资源 + 动作 + 上下文”的授权模型。

| 主体类型 | 示例 | 典型权限 |
|---|---|---|
| 普通用户 | 业主个人账户 | 管理自己的需求、合同和订单 |
| 服务商成员 | 个人施工人员/企业员工 | 管理授权范围内的资料、报价和履约 |
| 企业管理员 | 企业组织管理员 | 管理组织成员、服务能力和项目权限 |
| 平台运营 | 客服、审核、运营人员 | 按岗位授权查看和处理业务 |
| 财务岗位 | 对账、财务人员 | 查看授权支付和对账数据，不得修改合同履约事实 |
| 系统服务身份 | 内部微服务 | 仅访问明确授权的内部接口 |
| 研究人员 | 市场研究角色 | 查看允许范围内的市场数据和研究任务 |

权限要求：
- 默认拒绝，按资源和动作授权。
- 企业成员权限必须校验组织关系和岗位角色。
- 管理员操作需要审计；高风险动作可要求二次确认。
- 数据导出、批量查询、敏感字段访问需要单独授权。
- 运营人员不能因为拥有后台登录权限而默认拥有支付、合同或全部个人信息访问权限。
- 服务间调用必须校验服务身份和调用范围，不能依赖内网信任。

## 8. 幂等、并发与状态转换

### 8.1 必须幂等的命令

| 命令 | 幂等键范围 | 结果要求 |
|---|---|---|
| 创建需求 | 用户/组织 + 客户端请求键 | 重复提交返回同一业务结果 |
| 发布需求 | demandId + 发布命令键 | 不重复发布事件 |
| 服务商邀请响应 | invitationId + actorId | 不重复响应 |
| 提交报价 | demandId + providerId + 请求键 | 不重复创建同一报价版本 |
| 接受报价 | quotationId + 接受命令键 | 不重复创建合同流程 |
| 创建合同签署请求 | contractId + version + 请求键 | 不重复发起签署 |
| 开工/完工/验收 | orderId + 节点命令键 | 不重复生成履约节点 |
| 创建支付意图 | 业务付款计划ID + 请求键 | 不重复发起支付 |
| 支付机构回调 | providerCode + providerTransactionId + callbackId | 重复回调不重复处理 |
| 创建退款 | 原支付交易 + 退款业务申请ID | 不重复退款请求 |
| 市场采集任务 | planId + taskKey | 不重复创建相同采集任务 |

幂等记录包含作用域、幂等键、请求摘要、处理状态、响应摘要和有效期。相同键但不同请求内容应返回冲突，不得复用旧响应掩盖请求差异。

### 8.2 状态变更规则

- 状态由资源所属服务维护。
- 状态转换通过明确命令完成，禁止客户端直接PATCH状态字段。
- 每次状态变化记录操作者、原因、时间、前置版本和关联业务ID。
- 过期、撤销、关闭和失败等终态由业务规则文档统一。
- 跨服务流程通过事件和补偿推进，不允许消费者直接更新生产者状态。

## 9. 同步API与事件协作边界

| 场景 | 首选机制 | 理由 |
|---|---|---|
| 创建需求并返回结果 | 同步API | 用户需要立即知道创建结果 |
| 需求发布后通知撮合 | 事件 | 下游可独立处理和重试 |
| 服务商准入状态查询 | API/本地读模型 | 匹配决策需要当前有效状态 |
| 报价接受 | 同步命令 + 本地事务/唯一约束 | 需要明确并发结果 |
| 合同生效后创建履约订单 | 事件 + 幂等消费 | 解耦合同与履约 |
| 发起支付意图 | 内部同步命令 | 需要获得支付意图ID |
| 支付机构异步回调 | 回调入口 + 事件 | 外部结果需验签并异步通知业务 |
| 采集任务执行 | 消息/任务队列 | 长任务、限频和重试 |
| 文档解析 | 异步任务 | CPU密集、允许延迟 |
| 城市评分计算 | 异步任务 | 可批量、版本化计算 |

同步调用必须设置超时、限流和熔断。非幂等命令不得盲目重试。异步事件描述已经发生的事实，消费者必须支持重复投递。

## 10. API 版本与契约治理

1. 外部路径主版本采用/v1；不兼容变更升级主版本。
2. 增加可选字段、增加新端点通常可保持兼容，但仍需评估客户端反序列化行为。
3. 不随意重命名/删除字段、改变枚举含义、改变字段单位或金额精度。
4. 枚举应有UNKNOWN或兼容策略；消费者遇到未知枚举不得崩溃。
5. 每个服务维护OpenAPI文件；跨服务调用通过生成客户端或明确适配层，不复制散落的接口常量。
6. 每个事件维护JSON Schema/Protobuf等契约文件，标注版本和兼容策略。
7. CI执行OpenAPI lint、Schema校验、breaking-change检测、契约测试和安全扫描。
8. 生产环境保留调用方、服务版本、请求ID、traceID和错误码，不记录密钥和不必要敏感字段。
9. API弃用应发布迁移周期和调用方清单，不直接删除仍被使用的接口。

## 11. 错误码建议

| 错误码 | HTTP | 说明 |
|---|---:|---|
| VALIDATION_ERROR | 422 | 字段或业务参数不合法 |
| AUTHENTICATION_REQUIRED | 401 | 需要认证 |
| PERMISSION_DENIED | 403 | 无资源操作权限 |
| RESOURCE_NOT_FOUND | 404 | 资源不存在或不可见 |
| RESOURCE_VERSION_CONFLICT | 409 | 资源版本已变化 |
| INVALID_STATE_TRANSITION | 409 | 当前状态不允许该操作 |
| IDEMPOTENCY_KEY_REUSED | 409 | 幂等键对应不同请求内容 |
| QUOTATION_EXPIRED | 409 | 报价已过期 |
| QUOTATION_ALREADY_ACCEPTED | 409 | 已产生有效中选结果 |
| PROVIDER_NOT_ELIGIBLE | 422 | 服务商不满足准入/服务条件 |
| PAYMENT_RESULT_PENDING | 202 | 支付结果待确认，不等于失败 |
| PAYMENT_PROVIDER_REJECTED | 422 | 支付机构明确拒绝 |
| REFUND_NOT_ALLOWED | 409 | 当前条件不允许退款 |
| SOURCE_NOT_AUTHORIZED | 403 | 数据源不在允许采集范围 |
| SOURCE_RATE_LIMITED | 429 | 数据源限频 |
| COLLECTION_TASK_EXPIRED | 409 | 采集任务已过期 |
| EXTERNAL_DEPENDENCY_UNAVAILABLE | 503 | 外部依赖暂不可用 |

错误码一经对外发布，应保持稳定语义。客户端逻辑依赖错误码，而非错误文案。

## 12. 安全与数据最小化

- 所有外部API强制HTTPS；敏感操作按风险等级采用二次验证或额外风控。
- 精确地址、手机号、证件和合同附件按业务必要性最小化展示。
- 对外部服务商展示需求时使用区域级位置，满足业务阶段和授权条件后才披露更精确信息。
- API日志默认不记录完整请求体；敏感字段脱敏或排除。
- 文件接口使用短时签名URL或受控下载接口，禁止公开永久对象地址。
- 市场采集连接器只能调用授权来源；密钥放入密钥管理系统并轮换。
- 数据导出必须记录申请人、用途、范围、审批和下载行为。
- 删除、更正和留存期限按适用法律、合同义务和业务必要性设计，具体处理规则待法律评审。

## 13. 契约测试与验收

每个服务至少提供OpenAPI文档、正常路径测试、参数校验测试、权限测试、幂等测试、并发状态转换测试、依赖超时测试、API兼容性测试、事件Schema验证、敏感字段泄露测试和审计记录验证。

关键验收场景：
- 同一需求重复发布，不产生重复匹配任务。
- 同一需求两个报价并发接受，只能有一个有效结果。
- 合同事件重复投递，不重复创建订单。
- 支付回调重复/乱序，不重复记录成功交易或退款。
- 支付结果未知时，订单不能被错误标记为已支付或支付失败。
- 采集任务重复投递，不重复产生相同任务结果；来源限频得到执行。
- 未核验市场实体不能直接进入可接单服务商列表。
- 市场采集服务故障不影响交易API核心链路。

## 14. 开发落地顺序

1. identity + organization + provider：主体、组织和服务商准入。
2. catalog + demand：服务目录和需求发布。
3. matching + quotation：撮合、邀请和报价。
4. contract + service-order：合同、订单和履约。
5. payment：支付意图、回调、退款和对账；必须先完成支付机构及合规方案确认。
6. research-source + search-orchestrator + collection-connector：市场数据源和采集任务。
7. page-processing + market-entity + market-poi + market-signal：解析和市场实体治理。
8. city-intelligence + research-workbench：指标计算、城市评分和人工核验。
9. project：商业工程业务，在单独完成工程交易规则和资质边界评审后推进。

## 15. 待确认事项

- 用户身份、企业主体和服务商身份是否采用统一账户体系。
- 手机号、第三方登录和企业认证的首期范围。
- 需求公开范围、精确地址披露节点和联系方式交换规则。
- 报价有效期、报价修改次数、接受报价并发规则。
- 合同签署服务及证据保存标准。
- 支付机构接口、商户主体、退款权限、分账与结算产品边界。
- 市场数据源许可、原始文档留存、个人信息处理和数据导出审批。
- API网关、OpenAPI工具链、事件Schema格式和CI契约检测工具。

**结论：** 本文建立了微服务之间的初步API边界和契约基线。下一步应据此形成OpenAPI文件、事件Schema和数据库所有权设计；先实现交易主链路和市场采集链路的最小闭环，再根据真实业务复杂度扩展接口。
