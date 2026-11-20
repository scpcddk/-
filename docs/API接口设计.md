# 学生选课与成绩管理系统｜API 接口设计（第一版完整详尽草案）

> **文件路径**：`docs/API接口设计.md`  
> **文档版本**：V1.0 草案｜2026-11-20  
> **需求基线**：`docs/需求与业务规则.md`（CHG-001～CHG-072 的最终有效规则）  
> **配套文件**：`docs/数据库设计.md`、`docs/项目总计划书.md`、`docs/变更记录.md`  
> **当前状态**：**接口方案已起草，尚未通过正式设计评审，接口未编码、未联调、未测试**。  
> **使用说明**：本文件是可复制、可审阅的 API 契约候选稿；业务约束来自已确认需求，URL、字段名、枚举英文值、身份令牌方式、HTTP 返回码等属于**可调整的接口设计建议**，不得把这些技术细节误记为用户已经逐项批准。

---

## 0. 使用范围、版本与基本原则

### 0.1 本文解决什么

本文描述：

1. 学生、教师、管理员分别能访问的 API 能力，以及对象级权限校验。
2. API 请求参数、响应字段、校验失败与业务冲突的语义。
3. 权重报名、分轮录取、待开班、正式开班、退课及权重清零的接口状态。
4. 普通成绩首次录齐、72 小时审核、自动发布、过期录入授权及已发布成绩更正。
5. 补考/缺考/缓考统一特殊考试批次、名单冻结、批次审核与最终结果回写。
6. 当前正式成绩、班级统计、累计 GPA 与仅本人可见的同入学年级排名。
7. 幂等、防重、并发、权限、时间边界与验收测试。

**不负责**：把本文直接等同于 OpenAPI 代码生成产物、数据库 DDL、已经实现的 Controller 或真实运行证据。第一版不做重修专用 API、先修校验 API、完整培养方案、复杂特殊考试批次作废/重建、跨 Section 合班教学活动或完整商业前端。

### 0.2 四项不可改变的规则

- **报名成功 != 录取成功 != 班级已确认开班**。不能用一个 `success=true` 混淆三个层级。
- **申请提交时就扣可用权重**；本轮截止后按投入点数竞争，超额按高权重优先、同权重随机；原轮已录取者不被后轮高分挤出。
- **学生只能看到自己的权重、选课、正式成绩、GPA 与年级名次**；不得提供学生可调用的其他学生成绩/榜单接口。
- **当前正式成绩与原始录入/审核中分数必须区分**。普通班级、特殊考试各自 72 小时审核；已发布更正通过授权执行才即时生效。

### 0.3 推荐的接口设计约定（尚待评审）

| 项目 | 本文候选方案 | 已确认的业务边界 |
|---|---|---|
| API 前缀 | `/api/v1` | 将来可以版本化；具体路径非业务决策 |
| 认证 | `Authorization: Bearer <token>` | 学生/教师/管理员必须识别身份和授权范围 |
| 传输 | HTTPS；JSON `application/json` | 不可泄露敏感数据 |
| ID | 资源以服务端生成的稳定 ID 标识 | 不以可篡改的姓名/学号做内部关联 |
| 金额/分数 | 分数和学分以 JSON 数值表达，后端定点精度 | 0 分合法，`null` 与 0 不同 |
| 选课权重 | 请求示例使用整数点数 | 是否允许小数、0 点，仍待设计评审（D-01） |
| 时间 | 使用 ISO 8601 带时区偏移时间戳 | 统一学校时区、边界一致 |
| 列表 | `page`（从 1 开始）、`size`，有总数 | 具体上限及排序字段待实现确认 |
| 错误格式 | 统一 JSON，含稳定 `code` | 错误不能泄露内部异常或越权资源信息 |
| 重复写操作 | 优先以业务状态和唯一约束防重；必要时 `Idempotency-Key` | 重复扣/还权重和重复发布绝不允许 |

**规则优先级**：最终业务需求 > 已通过评审的数据库及 API 设计 > 本文候选字段命名 > 旧版计划书示例。对技术设计未决点，保持“待设计确认”，不要因这里出现示例就当成已确认校规。

---

## 1. 全局数据契约、时间与 HTTP 语义

### 1.1 成功响应 Envelope（候选）

普通资源查询可统一返回：

```json
{
  "code": "OK",
  "message": "success",
  "data": {
    "id": 101,
    "name": "示例资源"
  },
  "traceId": "req-example-001",
  "timestamp": "2026-11-20T08:00:00Z"
}
```

说明：`traceId` 用于排障；`timestamp` 是响应时间，不代表业务发生时间。建议对创建操作使用 HTTP 201，对成功但无响应体的幂等状态变更使用 200，是否采用 204 由团队统一；**响应的成功不可以代替持久化和并发测试**。

分页查询示例：

```json
{
  "code": "OK",
  "message": "success",
  "data": {
    "items": [
      { "id": 1001, "courseCode": "CS101", "courseName": "Java 程序设计" }
    ],
    "page": 1,
    "size": 20,
    "total": 1,
    "totalPages": 1
  },
  "traceId": "req-example-002",
  "timestamp": "2026-11-20T08:01:00Z"
}
```

### 1.2 失败响应 Envelope（候选）

```json
{
  "code": "ENROLLMENT_TIME_CONFLICT",
  "message": "报名课程与已有有效课程安排冲突",
  "data": null,
  "errors": [
    { "field": "sectionId", "reason": "课程实际教学周、星期和节次重叠" }
  ],
  "traceId": "req-example-003",
  "timestamp": "2026-11-20T08:02:00Z"
}
```

不能直接返回 SQL 语句、异常堆栈、数据库连接信息、密码、令牌或其他学生成绩。`errors` 应仅包含当前请求可公开的信息；越权对象可以返回 403 或按统一策略隐藏为 404，策略需一致。

### 1.3 HTTP 状态码建议

| HTTP | 用途 | 示例 |
|---|---|---|
| `200 OK` | 查询成功或状态变更成功 | 查询权重、撤回申请、批准更正 |
| `201 Created` | 创建资源成功 | 管理员创建教学班，学生创建报名 |
| `400 Bad Request` | JSON 解析错误、非法参数类型 | 日期不是日期、不能解析的字段 |
| `401 Unauthorized` | 未登录或认证无效 | 缺少有效身份令牌 |
| `403 Forbidden` | 角色、资源或身份不允许 | 教师试图更改他人教学班成绩 |
| `404 Not Found` | 资源不存在或需隐藏资源存在性 | 不存在的报名 ID |
| `409 Conflict` | 当前状态/业务约束不允许 | 权重不足、已过选课窗口、同课重复、非法成绩状态 |
| `422 Unprocessable Entity` | 可选：格式正确但字段业务值范围不合法 | 负学分、分数超出范围；须与 400/409 统一边界 |
| `500 Internal Server Error` | 未预料服务器错误 | 不泄露内部栈信息 |
| `503 Service Unavailable` | 可恢复的系统维护/暂时故障 | 关键数据库不可用 |

**建议**：首版把“输入非法”统一为 400、“业务状态冲突”统一为 409，避免在 400/409/422 间无意义地反复调整；若决定引入 422，必须更新接口测试。

### 1.4 时间、时区与窗口约定

业务基线如下：

- 学期：管理员设置第 1 周星期一，20 周；第 1～18 周授课，第 19～20 周考试。
- 正常选课：学期开始日前 14 天开启，持续 **3 个自然日**。
- 第一次补选：**2 天**；第二次补选：**1 天**；不得产生第三轮常规补选。
- 退课：从教学班**首次实际授课日期**起 7 天内；一旦班级任一有效学生成绩开始录入，整班普通退课锁定。
- 普通成绩截止：第 20 周结束的星期日后默认 **14 天**；可配置为当天至后 **30 天**，截止当日整天有效。
- 普通审核/特殊考试审核：首次完整录入后 **72 小时**，不是“3 个自然日零点”或“72 分钟”。
- 截止、发放、结算、自动发布必须使用同一业务时区和精确、可测试的时钟。

**待定设计项 D-02**：三段选课窗口的具体开始小时、结束小时以及是否无缝衔接还未被业务确认。接口的 `startAt/endAt` 必须来自学校统一配置，**不得把示例时间当成最后决定**。

**建议技术约定**：统一采用 `[startAt, endAt)` 半开区间（开始可操作，恰好到结束时刻不可操作）；普通成绩的“截止当日包含”以学校时区次日零点转换为互斥上界。实现前通过边界测试确认。

### 1.5 枚举与展示值原则

数据库/接口英文枚举均为**建议命名**；前端展示使用清楚的中文描述，不把内部状态字符串直接交给用户当文案。

| 领域 | 建议值 | 业务含义 |
|---|---|---|
| 学籍 | `NORMAL` / `SUSPENDED` / `WITHDRAWN` / `GRADUATED` | 在读/休学/退学/毕业 |
| 学期计算态 | `PLANNED` / `IN_PROGRESS` / `FINISHED` | 待开始/进行中/已结束 |
| 选课轮次 | `NORMAL` / `SUPPLEMENT_1` / `SUPPLEMENT_2` | 正常/第一次补选/第二次补选 |
| 轮次结算 | `PENDING` / `RUNNING` / `SETTLED` | 未结算/执行中/已结算（候选） |
| 报名 | `SUBMITTED` / `WITHDRAWN` / `ACCEPTED` / `REJECTED` | 待结算/撤回/获得名额/落选 |
| 班级开班 | `PENDING_OPENING` / `CONFIRMED` / `CANCELED` | 待开班/确认开班/最终取消 |
| 名额 | `PENDING_OPENING` / `CONFIRMED` | 录取成功但待开/有效已开 |
| 普通结果 | `UNENTERED` / `NUMERIC` / `ABSENT` / `DEFERRED` | 未录/数值/缺考/缓考 |
| 审核 | `NOT_STARTED` / `REVIEWING` / `PUBLISHED` | 未开始/审核中/已发布 |
| 更正申请 | `PENDING` / `REJECTED` / `APPROVED` / `WITHDRAWN` / `EXPIRED` / `EXECUTED` | 审批生命周期 |
| 特殊批次 | `COLLECTING` / `FROZEN` / `REVIEWING` / `EFFECTIVE` | 汇总/冻结/审核/生效 |
| 权重账户 | `ACTIVE` / `CLEARED` | 有效/最终清零 |

特别区分：**报名 `ACCEPTED` 不必然意味着班级已 `CONFIRMED`**。接口应该同时返回报名/名额状态和班级开班状态，避免前端说“您已经正式开课”。

### 1.6 API 命名与请求安全约定

- `GET` 无副作用；不能通过 GET 触发结算、发放 50 点或成绩正式发布。
- `POST` 表示创建业务动作；`PATCH` 表示局部配置更新；`PUT` 表示对单个可替换资源的完整更新。
- 资源 URL 使用名词，例如 `/sections/{sectionId}`；行为型业务（撤回、录取结算、批准、冻结）可以使用明确的子资源动作，如 `/withdraw`、`/approve`、`/freeze`，并在日志中记录原因。
- 学生自助资源以 `/me` 为主，**不接收任意 `studentId` 作为身份归属依据**。管理员查看学生资料走独立 `/admin/students` 路由。
- 业务结果必须经过**身份、角色、资源归属、学籍/业务状态、时间窗口**五级校验，不允许只在 Controller `@PreAuthorize` 后忽略 Service 层状态。
- 分页上限、排序字段使用白名单，阻止任意 SQL 字段或 SQL 片段输入。

---

## 2. API 资源总览和角色目录

> 以下是建议的 REST 路径清单。不同名称、拆分方法可以在实际 Controller 设计中优化，但不能遗漏已确认的业务操作与权限。

### 2.1 公共认证及学生自助

| 方法 | 路径 | 能力 | 访问角色 |
|---|---|---|---|
| POST | `/api/v1/auth/login` | 登录获得有效凭证 | 公开登录入口 |
| POST | `/api/v1/auth/logout` | 注销当前登录 | 已认证（认证机制允许时） |
| GET | `/api/v1/me` | 当前登录身份 | 已认证 |
| GET | `/api/v1/semesters` | 当前/历史学期列表 | 已认证 |
| GET | `/api/v1/semesters/{semesterId}/rounds` | 选课三轮时间与状态 | 已认证 |
| GET | `/api/v1/sections` | 可见教学班查询 | 已认证（字段按角色过滤） |
| GET | `/api/v1/sections/{sectionId}` | 教学班公开详情 | 已认证（不含其他学生隐私） |
| GET | `/api/v1/me/weight-wallet?semesterId=...` | 本人学期权重余额 | 学生本人 |
| GET | `/api/v1/me/enrollment-applications` | 本人报名及轮次结果 | 学生本人 |
| POST | `/api/v1/me/enrollment-applications` | 本人提交权重报名 | 正常在读学生 |
| POST | `/api/v1/me/enrollment-applications/{applicationId}/withdraw` | 截止前撤回本人活动报名 | 正常在读学生 |
| GET | `/api/v1/me/enrollments` | 本人当前已获名额和课表 | 学生本人 |
| POST | `/api/v1/me/enrollments/{enrollmentId}/drop` | 合法普通退课 | 正常在读学生 |
| GET | `/api/v1/me/enrollment-history` | 本人报名、落选、退课和取消历史 | 学生本人 |
| GET | `/api/v1/me/grades` | 本人当前正式课程成绩 | 学生本人 |
| GET | `/api/v1/me/gpa` | 本人历学期累计 GPA | 学生本人 |
| GET | `/api/v1/me/cohort-rank` | 本人在同入学年级中的名次 | 有资格学生本人；无资格返回暂无名次 |

### 2.2 管理员：基础目录、开课和轮次

| 方法 | 路径 | 能力 | 角色 |
|---|---|---|---|
| POST / GET | `/api/v1/admin/majors` | 创建/查询专业目录 | 管理员 |
| POST / GET | `/api/v1/admin/students` | 创建/查询学生 | 管理员 |
| GET / PATCH | `/api/v1/admin/students/{studentId}` | 学生资料及合法修改 | 管理员 |
| POST | `/api/v1/admin/students/{studentId}/status-changes` | 学籍状态变更（含审计） | 管理员 |
| POST / GET | `/api/v1/admin/teachers` | 创建/查询教师 | 管理员 |
| GET / PATCH | `/api/v1/admin/teachers/{teacherId}` | 教师资料修改 | 管理员 |
| POST / GET | `/api/v1/admin/accounts` | 创建账号/查询账号 | 管理员 |
| PATCH | `/api/v1/admin/accounts/{accountId}/status` | 账号状态管理 | 管理员 |
| POST / GET | `/api/v1/admin/courses` | 课程创建/查询 | 管理员 |
| GET / PATCH | `/api/v1/admin/courses/{courseId}` | 课程详情与合法修改 | 管理员 |
| POST | `/api/v1/admin/courses/{courseId}/deactivate` | 停用课程（不删历史） | 管理员 |
| PUT | `/api/v1/admin/courses/{courseId}/eligibility` | 专业与入学年级允许范围 | 管理员 |
| POST / GET | `/api/v1/admin/semesters` | 创建/查询学期 | 管理员 |
| GET / PATCH | `/api/v1/admin/semesters/{semesterId}` | 合法修改学期配置 | 管理员 |
| PUT | `/api/v1/admin/semesters/{semesterId}/rounds` | 配置三轮窗口候选接口 | 管理员 |
| POST / GET | `/api/v1/admin/sections` | 创建/管理教学班 | 管理员 |
| GET / PATCH | `/api/v1/admin/sections/{sectionId}` | 教学班基础信息和合法容量调整 | 管理员 |
| PUT | `/api/v1/admin/sections/{sectionId}/meetings` | 设置教学周/星期/节次课表 | 管理员 |
| POST | `/api/v1/admin/sections/{sectionId}/publish-schedule` | 发布课表并冻结普通修改 | 管理员 |
| POST | `/api/v1/admin/sections/{sectionId}/change-teacher` | 更换授课教师 | 管理员 |
| GET | `/api/v1/admin/sections/{sectionId}/opening` | 查看待开/确认/取消状态 | 管理员 |
| GET | `/api/v1/admin/rounds/{roundId}/settlements` | 查看本轮教学班结算结果 | 管理员 |
| POST | `/api/v1/admin/rounds/{roundId}/settle` | 受控触发或重试轮次结算（可选） | 管理员/受控内部任务 |

### 2.3 教师：授课和普通成绩

| 方法 | 路径 | 能力 | 角色 |
|---|---|---|---|
| GET | `/api/v1/teacher/sections` | 本人负责教学班 | 教师 |
| GET | `/api/v1/teacher/sections/{sectionId}/students` | 本教学班当前有效学生名单 | 本班教师 |
| GET | `/api/v1/teacher/sections/{sectionId}/grade-entries` | 本班草稿成绩与录入状态 | 本班教师 |
| PUT | `/api/v1/teacher/sections/{sectionId}/grade-entries/{enrollmentId}` | 保存某学生本班合法结果 | 本班教师 |
| GET | `/api/v1/teacher/sections/{sectionId}/grade-review` | 查看首次录齐和 72 小时审核状态 | 本班教师 |
| GET | `/api/v1/teacher/sections/{sectionId}/statistics` | 本班正式成绩统计 | 本班教师 |
| POST / GET | `/api/v1/teacher/grade-corrections` | 申请更正与查询本人申请 | 教师 |
| POST | `/api/v1/teacher/grade-corrections/{requestId}/withdraw` | 撤回本人待审核申请 | 申请教师 |
| POST | `/api/v1/teacher/grade-corrections/{requestId}/execute` | 按精准授权执行更正 | 获授权教师 |

### 2.4 管理员：成绩截止、更正和特殊考试

| 方法 | 路径 | 能力 | 角色 |
|---|---|---|---|
| GET | `/api/v1/admin/semesters/{semesterId}/grade-deadline` | 查看普通录分截止 | 管理员 |
| PATCH | `/api/v1/admin/semesters/{semesterId}/grade-deadline` | 未过期前合法调整并审计 | 管理员 |
| POST / GET | `/api/v1/admin/sections/{sectionId}/late-grade-grants` | 授予/查询本班迟交成绩录入窗口 | 管理员 |
| GET | `/api/v1/admin/grade-corrections` | 更正申请审批列表 | 管理员 |
| POST | `/api/v1/admin/grade-corrections/{requestId}/approve` | 精准批准目标分数和时限 | 管理员 |
| POST | `/api/v1/admin/grade-corrections/{requestId}/reject` | 驳回更正申请 | 管理员 |
| POST / GET | `/api/v1/admin/special-exam-batches` | 创建/查询学期×课程特殊考试批次 | 管理员 |
| GET | `/api/v1/admin/special-exam-batches/{batchId}` | 查询批次状态 | 管理员 |
| GET | `/api/v1/admin/special-exam-batches/{batchId}/participants` | 自动汇总的候选参加名单 | 管理员 |
| POST | `/api/v1/admin/special-exam-batches/{batchId}/freeze` | 考前确认冻结名单 | 管理员 |
| GET / PUT | `/api/v1/admin/special-exam-batches/{batchId}/teachers` | 查询/修改负责人集合并审计 | 管理员 |
| GET | `/api/v1/admin/sections/{sectionId}/statistics` | 查看任意合法本班统计 | 管理员 |
| GET | `/api/v1/admin/statistics/sections` | 全校开课班级统计（可分页） | 管理员 |

### 2.5 特殊考试批次负责教师

| 方法 | 路径 | 能力 | 权限 |
|---|---|---|---|
| GET | `/api/v1/teacher/special-exam-batches` | 当前明确授权负责的批次 | 该负责教师 |
| GET | `/api/v1/teacher/special-exam-batches/{batchId}/participants` | 已冻结名单及可录入成绩 | 该负责教师 |
| PUT | `/api/v1/teacher/special-exam-batches/{batchId}/participants/{participantId}/result` | 保存特殊考试原始结果 | 该负责教师 |
| GET | `/api/v1/teacher/special-exam-batches/{batchId}/review` | 批次录齐和审核状态 | 该负责教师 |

**重要**：系统内部自动到期任务并不一定需要暴露为公网 HTTP API；调度程序可以直接调用 Service。本文的 `/admin/.../settle` 也是**受控触发/诊断候选**，不代表学生或普通教师可以手动随时结算。

---
## 3. 登录、身份与管理员基础数据接口

### 3.1 登录与当前身份

**`POST /api/v1/auth/login`**（示例候选）

请求：

```json
{
  "username": "student-demo-001",
  "password": "<由测试环境设置的密码>"
}
```

响应 `data` 候选：

```json
{
  "accessToken": "<token-redacted>",
  "tokenType": "Bearer",
  "expiresAt": "2026-11-20T10:00:00Z",
  "principal": {
    "userId": 501,
    "role": "STUDENT",
    "studentId": 10001,
    "displayName": "张三"
  }
}
```

- 登录、登出、令牌失效策略属于安全设计；第一版可先使用测试身份完成核心业务，再在阶段 5 接入真实认证。**最终交付必须有可信身份机制**。
- 学籍状态 `SUSPENDED/WITHDRAWN/GRADUATED` 不等同于账号自动不可登录；符合已确认规则的非正常在读学生仍可登录查询本人历史正式成绩，不能报名或退课。
- 不返回明文密码、密码散列或其他内部安全字段。
- `GET /api/v1/me` 返回当前登录身份和有权限公开的本人概况，`role` 来自服务端认证上下文，不接受客户端伪造。

### 3.2 管理员创建学生和修改学籍

**`POST /api/v1/admin/students`**

请求示例：

```json
{
  "studentNo": "2026010001",
  "name": "张三",
  "majorId": 12,
  "entryYear": 2026,
  "enrollmentStatus": "NORMAL"
}
```

响应示例 `data`：

```json
{
  "id": 10001,
  "studentNo": "2026010001",
  "name": "张三",
  "majorId": 12,
  "entryYear": 2026,
  "enrollmentStatus": "NORMAL",
  "createdAt": "2026-11-20T08:10:00Z"
}
```

校验：学号唯一、专业存在、入学年级合法、状态属于四个已确认值；创建账号是否同事务创建由账号建模方案决定，但**管理员创建身份**是业务硬约束。不能让学生通过公共注册接口自行取得正式学籍身份。

**`POST /api/v1/admin/students/{studentId}/status-changes`**

```json
{
  "newStatus": "SUSPENDED",
  "reason": "学籍管理示例原因"
}
```

响应必须能查询最新学籍状态；审计保存原状态、新状态、管理员和时间。若学生已经存在活动报名/有效选课，**状态变更的业务一致性属于 D-05 未定设计点**，不能在此擅自规定“自动取消所有课程并扣除权重”。状态变化后选退课资格与年级排名立即按已确认规则判断。

### 3.3 教师、专业与账号

- `POST /admin/majors`：`majorCode, majorName`；编号唯一；专业状态不能物理删除学生历史。
- `GET /admin/majors`：分页/查询专业；学生在课程浏览端可见合法专业名称，不暴露后台备注。
- `POST /admin/teachers`：`teacherNo, name`；教师工号唯一。
- `PATCH /admin/teachers/{teacherId}`：按可修改字段更新，不能借此绕开实际授课归属校验。
- `POST /admin/accounts`：绑定学生/教师身份与角色，管理员创建账号。
- `PATCH /admin/accounts/{accountId}/status`：控制可登录与否；若锁定账号，学生不能绕过权限访问业务接口。

这些接口的具体字段长度、密码初始化/重置机制与管理员身份数据结构属于**安全和数据库设计阶段**。不要把“示例账号创建”当成允许提交明文密码入库。

---

## 4. 课程、学期、教学班与排课 API

### 4.1 课程创建、查询与停用

**`POST /api/v1/admin/courses`**

```json
{
  "courseCode": "CS101",
  "courseName": "Java 程序设计",
  "credits": 4.0,
  "eligibility": {
    "allMajorsAllowed": true,
    "allowedMajorIds": [],
    "allEntryYearsAllowed": false,
    "allowedEntryYears": [2025, 2026]
  }
}
```

**字段含义及约束**：

| 字段 | 候选类型 | 校验 |
|---|---|---|
| `courseCode` | 字符串 | 唯一、非空、稳定 |
| `courseName` | 字符串 | 非空、长度受限 |
| `credits` | 十进制数 | 正数；课程曾建过班后不能通过普通 PATCH 修改 |
| `allMajorsAllowed` | 布尔 | `true` 表示不限制专业 |
| `allowedMajorIds` | ID 数组 | 仅在限制专业时代表允许集合 |
| `allEntryYearsAllowed` | 布尔 | `true` 表示不限制入学年级 |
| `allowedEntryYears` | 整数年份数组 | 仅在限制年级时有意义 |

**不能用空数组代替“全部允许”而不说明规则**；同一请求既填 `allMajorsAllowed=true` 又填某些专业 ID，应在接口设计评审时确定是拒绝还是清空无效列表。为避免误解，本文件示例只使用一致的组合。

**`POST /api/v1/admin/courses/{courseId}/deactivate`**

```json
{ "reason": "学校不再为后续学期安排该课程" }
```

成功后禁止新建后续教学班；**已建教学班继续教学，历史正式课程成绩继续计入 GPA**。这不是 `DELETE /courses/{courseId}`；第一版不提供破坏历史关联的物理删除 API。

**`PATCH /api/v1/admin/courses/{courseId}`**：可以更新允许修改的课程信息，但如果课程曾用于创建教学班，修改 `credits` 必须被拒绝（即使当前没有活跃教学班）。

### 4.2 创建学期与学期状态查询

**`POST /api/v1/admin/semesters`**

```json
{
  "semesterCode": "2027-SPRING",
  "name": "2027 年春季学期",
  "week1Monday": "2027-03-01"
}
```

响应 `data` 示例：

```json
{
  "id": 202701,
  "semesterCode": "2027-SPRING",
  "week1Monday": "2027-03-01",
  "lastDay": "2027-07-18",
  "teachingWeekRange": [1, 18],
  "examWeekRange": [19, 20],
  "derivedStatus": "PLANNED",
  "gradeEntryDeadlineDate": "2027-08-01"
}
```

> 上述学期最后一天和默认录分截止日期是按示例首日计算出的演示值；实际接口须根据配置日期准确计算，不应手工复制常量。

- 第一周日期必须为星期一；连续 20 周，跨年允许。
- `derivedStatus` 必须**由当前学校日期推导**，不是管理员自由修改字段。
- 一旦曾创建任何教学班，`week1Monday` 不再允许普通修改；不能因为事后删除/取消教学班而绕过冻结。
- 默认录分截止为第 20 周星期日后 14 天，最早当日、最晚后 30 天，**与选课窗口分开**。

### 4.3 三轮选课窗口配置及查看

**`GET /api/v1/semesters/{semesterId}/rounds`** 响应 `data` 示意：

```json
{
  "semesterId": 202701,
  "schoolTimeZone": "<学校统一配置的时区>",
  "rounds": [
    {
      "id": 301,
      "type": "NORMAL",
      "durationDays": 3,
      "startAt": "<服务端计算或配置的带偏移时间戳>",
      "endAt": "<服务端计算或配置的带偏移时间戳>",
      "status": "PENDING",
      "settlementStatus": "PENDING"
    },
    { "id": 302, "type": "SUPPLEMENT_1", "durationDays": 2, "status": "PENDING" },
    { "id": 303, "type": "SUPPLEMENT_2", "durationDays": 1, "status": "PENDING" }
  ]
}
```

**示例仅示意字段**，真实 API 必须为三轮都提供最终时间；不要以示例里的未完整字段作为可以落地的契约。

- 一学期恰好规划三轮；`NORMAL` 从开学前 14 天启动，共 3 自然日；后两轮分别 2 天和 1 天。
- 学校统一窗口，不能由每个班级随意设置另一套相互冲突的时间。
- 查询应分别返回：是否可报名、是否已截止、是否正在结算、是否结算完成；**结束时刻和结算完成时刻不等同**。
- **配置接口是候选**：第一版可由管理员在学期准备时一次配置并验证，也可由系统可靠地推导；不能擅自把每天开放小时和是否无缝衔接写为已确认业务。

### 4.4 创建教学班、编辑课表与修改容量

**`POST /api/v1/admin/sections`**

```json
{
  "semesterId": 202701,
  "courseId": 1001,
  "sectionCode": "CS101-01",
  "teacherId": 2001,
  "capacity": 40,
  "meetings": [
    { "dayOfWeek": 2, "startSlot": 1, "endSlot": 3, "teachingWeeks": [1, 3, 5, 7] },
    { "dayOfWeek": 4, "startSlot": 3, "endSlot": 5, "teachingWeeks": [1, 2, 3, 4, 5] }
  ]
}
```

建议响应 `data` 示例：

```json
{
  "id": 4001,
  "semesterId": 202701,
  "courseId": 1001,
  "teacherId": 2001,
  "capacity": 40,
  "minimumOpeningCount": 20,
  "acceptedCount": 0,
  "schedulePublished": false,
  "openingStatus": "PENDING_OPENING"
}
```

- `capacity` 只能为**正偶数**，如 40 合法、35/0/-2 不合法；`minimumOpeningCount` 派生为容量的一半。
- 教师不得和其他独立教学班在相同教学周、星期及重叠节次授课。
- 教学周集合应支持不连续集合；不允许把 `[1,3,5,7]` 自动扩展为连续 1～7 周。
- `startSlot/endSlot` 采用半开区间，`[1,3)` 与 `[3,5)` 边界相接不冲突。
- 教学班一旦对学生开放，课表普通编辑应被冻结；`PUT /meetings` 只能在符合状态时成功。
- 调整容量必须保持正偶数、不低于当前有效已录取人数，且和并发录取结算有一致性保护。
- 首次实际上课日可从教学周和课表规则推导；如需特别授课日调整，第一版不额外发展复杂调课业务。

**`POST /api/v1/admin/sections/{sectionId}/change-teacher`**

```json
{
  "newTeacherId": 2002,
  "reason": "原授课教师教学任务调整"
}
```

校验新教师课表冲突；成功后当前教师变化，原教师和原因保留审计；不能影响学生已经获得的名额，也不得把旧教师设为可以继续自由录分的当前负责人。

### 4.5 教学班公开查询

**`GET /api/v1/sections?semesterId=...&courseId=...&page=1&size=20`** 返回不含其他学生私人身份数据的公开列表。

单项字段建议：

```json
{
  "id": 4001,
  "courseId": 1001,
  "courseCode": "CS101",
  "courseName": "Java 程序设计",
  "credits": 4.0,
  "teacherName": "李老师",
  "capacity": 40,
  "minimumOpeningCount": 20,
  "acceptedCount": 18,
  "availableSeats": 22,
  "openingStatus": "PENDING_OPENING",
  "eligibleForCurrentStudent": true,
  "meetings": [
    { "dayOfWeek": 2, "startSlot": 1, "endSlot": 3, "teachingWeeks": [1, 3, 5, 7] }
  ]
}
```

- `acceptedCount` 只统计已获得名额的学生，**不统计尚未结算的报名者**。
- `availableSeats` 可作为实时提示，但在轮末权重竞争前只是容量信息，**不能当作用户抢到名额的凭据**。
- 某轮截止之后结算尚未完成时，要能区分 `ROUND_AWAITING_SETTLEMENT` 等可读状态，不能让前端继续接受已经截止的新报名。
- `eligibleForCurrentStudent` 仅是界面辅助；提交报名时仍须在事务中重新校验资格、时间冲突和余额。
- 避免公开其他学生名单、报名出价、抽签值、原始成绩。

---

## 5. 学生权重、报名、录取与退课 API（第一核心）

### 5.1 查询当前可用权重

**`GET /api/v1/me/weight-wallet?semesterId=202701`**

```json
{
  "semesterId": 202701,
  "availablePoints": 100,
  "walletStatus": "ACTIVE",
  "initialGrantRecorded": true,
  "supplementGrantRecorded": false,
  "lastUpdatedAt": "2027-02-15T00:00:00Z"
}
```

注意：该示例时间仅为说明字段格式，**不代表正常轮具体开放小时已经确认**。当前余额只属于已认证学生本人；不可把 `studentId` 作为客户端自选参数查询别人余额。

**业务不变量**：

- 初始选课额度为 100 点；第一次补选开始时所有符合资格学生统一增加 50 点。
- 提交一笔合法报名立即扣除该笔投入；获名额时不重复扣除。
- 撤回、落选、最终不开班取消按规则返还；清零前合法退课返还，清零后不恢复可用余额。
- 即使客户端刷新、重复提交或并发调用，可用余额也不得为负，50 点不得重复发放。
- `walletStatus=CLEARED` 表示选课权重本学期已失效，不代表合法退课受到额外禁止。

### 5.2 提交带权重报名

**`POST /api/v1/me/enrollment-applications`**

请求示例：

```json
{
  "semesterId": 202701,
  "roundId": 301,
  "sectionId": 4001,
  "bidPoints": 40
}
```

响应 `201` 的 `data`：

```json
{
  "applicationId": 7001,
  "studentId": 10001,
  "sectionId": 4001,
  "roundId": 301,
  "status": "SUBMITTED",
  "bidPoints": 40,
  "deductedPoints": 40,
  "availablePointsAfter": 60,
  "seatAwarded": false,
  "message": "报名已提交并扣除权重，待本轮截止后结算录取结果"
}
```

**此处最重要：`status=SUBMITTED`、`seatAwarded=false` 是预期行为，不是失败。** 不能把 HTTP 201 解释为“已经成功占据教学班名额”。

#### 事务前置校验

1. 当前账号通过认证且绑定本人学生身份。
2. 当前学籍为正常在读。
3. 当前轮次属于目标教学班所在学期，且报名窗口真正开放。
4. 教学班允许当前轮次报名，课程未因状态失去合法选课资格。
5. 学生符合课程允许专业、入学年级范围。
6. 同学期同课程不存在另一笔有效报名或名额。
7. 学生已有活动报名及已获得名额与目标教学班的**教学周、星期、节次**不发生实际冲突。
8. 申请投入权重合法、且不超过该学生本学期当前可用余额。
9. 不因客户端重试产生重复报名或重复扣点。

#### 原子性要求

**报名申请创建、扣减钱包可用余额、写权重流水必须同一事务成功或失败。** 如果某一步失败，最终不得出现“扣钱了但没有申请”或“有申请却没有扣点”。

**权重粒度仍待定**：示例 40 只说明分配机制。第一版是否只接受整数点、最小投入为 1 点等，应在数据库/API 设计评审确认后统一更新 DTO、数据库类型和测试，不得悄悄改变用户已确认业务。

#### 主要错误

| 场景 | 建议错误码 | 余额是否变化 |
|---|---|---|
| 学生非正常在读 | `STUDENT_NOT_ENROLLED` | 不变 |
| 未到/已过当前轮次窗口 | `ROUND_NOT_OPEN` | 不变 |
| 权重不足 | `INSUFFICIENT_WEIGHT` | 不变 |
| 同学期同课程已有活动目标 | `DUPLICATE_COURSE_ENROLLMENT` | 不变 |
| 与活动报名/名额的课表冲突 | `ENROLLMENT_TIME_CONFLICT` | 不变 |
| 专业或入学年级不符合限制 | `COURSE_ELIGIBILITY_DENIED` | 不变 |
| 开课班级最终取消 | `SECTION_CANCELED` | 不变 |
| 重复请求/状态竞争 | `APPLICATION_CONFLICT` | 不重复扣点 |

### 5.3 查询当前报名及结算结果

**`GET /api/v1/me/enrollment-applications?semesterId=202701`**

应返回当前学生本人各轮报名，包括：`applicationId, sectionId, courseName, roundType, bidPoints, status, submittedAt, settledAt, outcome, openingStatus`。

示例条目：

```json
{
  "applicationId": 7001,
  "sectionId": 4001,
  "courseName": "Java 程序设计",
  "roundType": "NORMAL",
  "bidPoints": 40,
  "applicationStatus": "ACCEPTED",
  "seatStatus": "PENDING_OPENING",
  "sectionOpeningStatus": "PENDING_OPENING",
  "settledAt": "<该轮真实结算时刻>",
  "displayMessage": "已获得名额，班级人数暂未达到最低开班要求"
}
```

**当本轮尚未结算**：`SUBMITTED`，不返回误导性的“已录取”。**当本轮竞争失败**：`REJECTED`，并在自身账户余额中看到已返还权重。**当班级后来最终取消**：应能查看取消原因与自己的名额失效，不再在当前课表显示它。

### 5.4 截止前撤回报名

**`POST /api/v1/me/enrollment-applications/{applicationId}/withdraw`**

请求可以为空对象 `{}` 或在审计要求下提供可选理由；**这不是正式退课 API**。

响应 `data` 示例：

```json
{
  "applicationId": 7001,
  "status": "WITHDRAWN",
  "refundedPoints": 40,
  "availablePointsAfter": 100,
  "withdrawnAt": "<实际撤回时间>"
}
```

要求：

- 仅本人、当前轮截止前、仍处于有效未结算报名状态可以撤回。
- 更新申请状态、退还该申请投入、写流水/审计在同一事务内完成。
- 重复撤回不能重复返还；可以返回原成功结果或统一状态冲突，**但不得重复记账**。
- 本轮已经截止并结算后，不得把“撤回报名”伪装为“退课”规避规则。
- 被拒绝撤回的请求不得改变余额或活动目标。

### 5.5 查询当前获得名额、待开班与课表

**`GET /api/v1/me/enrollments?semesterId=202701`**

响应 `data.items` 示例：

```json
[
  {
    "enrollmentId": 8101,
    "applicationId": 7001,
    "sectionId": 4001,
    "courseCode": "CS101",
    "courseName": "Java 程序设计",
    "credits": 4.0,
    "seatStatus": "PENDING_OPENING",
    "sectionOpeningStatus": "PENDING_OPENING",
    "bidPoints": 40,
    "meetings": [
      { "dayOfWeek": 2, "startSlot": 1, "endSlot": 3, "teachingWeeks": [1, 3, 5, 7] }
    ],
    "ordinaryDropAllowed": false,
    "ordinaryDropBlockedReason": "<按实际窗口与状态判断的原因>"
  }
]
```

其中 `ordinaryDropAllowed` 可由服务端按当前学校时间计算，是界面提示，不代替退课 Service 二次校验。**待开班学生如何在后续补选中主动退出并改选属于 D-03 待技术定稿**，不得在 API 中擅自承诺未确认的功能。

### 5.6 合法普通退课

**`POST /api/v1/me/enrollments/{enrollmentId}/drop`**

响应 `data` 示例（权重仍有效）：

```json
{
  "enrollmentId": 8101,
  "dropped": true,
  "seatReleased": true,
  "refundedUsablePoints": 40,
  "walletStatus": "ACTIVE",
  "availablePointsAfter": 100,
  "droppedAt": "<实际退课时间>"
}
```

若第二轮补选已完成全部结算并**最终清零**，同一条合法退课应返回：

```json
{
  "enrollmentId": 8101,
  "dropped": true,
  "seatReleased": true,
  "refundedUsablePoints": 0,
  "walletStatus": "CLEARED",
  "availablePointsAfter": 0,
  "droppedAt": "<实际退课时间>"
}
```

注意：上面返回的 `refundedUsablePoints` 指**本次实际增加的可用余额**，并不抹除曾经投入的权重。若前端要展示原投入，可另返回 `originalBidPoints`；勿把“不返可用余额”错误说成“从未投入权重”。

**退课前置条件：**

- 本人当前具有合法有效名额，学籍正常在读；不允许退掉其他人的课。
- 处于该班级**首次实际上课日起 7 天**内的合法普通退课窗口；具体截止时刻必须和统一时区一致。
- 该班级尚未对任何有效学生开始普通成绩录入；一旦开始，整班普通退课锁定。
- 退课释放名额；已确认开班不因人数下降而自动退回待开班。
- 退课和权重变更/流水/审计保持原子一致；同一名额不得重复退课、重复释放容量。

**清零前后规则**：判定点是第二轮所有结算与必要返还完成并成功清零的状态，而不是客户端时间显示“已经到了结束时刻”。与结算同时发生的退课应有受控顺序。

### 5.7 学生历史查询

**`GET /api/v1/me/enrollment-history?semesterId=...`** 可展示本人的报名、撤回、落选、取消、正式退课事件及必要时间。可显示投入与返还摘要，但不展示其他学生出价或同权重抽签明细。非正常在读学生的**已正式成绩历史查询**必须仍可使用；当前选退课操作权限按学籍状态限制。

---
## 6. 管理员轮次结算、开班确认与权重清零（第一核心后台）

### 6.1 轮次结算必须是受保护的系统动作

**候选**：`POST /api/v1/admin/rounds/{roundId}/settle`

```json
{ "reason": "按轮次截止时间执行本轮结算" }
```

这个入口可用于管理员受控触发或测试，真实自动调度也可以直接调用同一 Service，**不需要为每一条自动业务暴露公网管理 API**。要求：

1. 轮次已到截止时间，不允许提前结算；服务端判断实际时间。
2. 同一轮同一教学班的有效申请只取一份不可变竞争快照，识别已撤回与无效申请。
3. 每个教学班先统计上一轮已有的名额，再确定本轮剩余可用容量。
4. 本轮有效报名人数不超过剩余容量时全部取得名额；超过时按 `bidPoints DESC`，同权重边界随机决定。
5. 被录取者**不再重复扣权重**，因为提交申请时已经扣除。
6. 落选者对应申请的投入全额返还；必须有权重流水及业务事件。
7. 已经结算的教学班，重试不得再次随机、再次录取、再次退回权重。
8. 结算不应擅自修改班级课表、教师、课程学分或成绩信息。
9. 任何时间点都必须满足 `有效已录取人数 <= 最大容量`，且容量仍为正偶数。

**响应 `data` 建议**：

```json
{
  "roundId": 301,
  "settlementStatus": "SETTLED",
  "completedAt": "<实际完成时间>",
  "sectionSummaries": [
    {
      "sectionId": 4001,
      "acceptedThisRound": 25,
      "rejectedThisRound": 7,
      "acceptedTotal": 25,
      "capacity": 40,
      "minimumOpeningCount": 20,
      "openingStatus": "CONFIRMED"
    }
  ]
}
```

这个返回是**管理侧汇总**；学生不能通过它得到其他学生的竞争列表、权重或录取结果。若涉及耗时较长的全校结算，可以改为返回 `taskId` 与执行状态，但**异步并不免除幂等、可恢复和数据一致性约束**。

### 6.2 开班判断与最终取消

管理员查询：`GET /api/v1/admin/sections/{sectionId}/opening`

```json
{
  "sectionId": 4001,
  "capacity": 40,
  "minimumOpeningCount": 20,
  "acceptedCount": 18,
  "openingStatus": "PENDING_OPENING",
  "openingConfirmedAt": null,
  "finalCancellationReason": null
}
```

**业务规则**：

- 轮次结算后，已录取人数达到容量一半时确认开班。
- 若最终录取不足，则保持待开班接受后续轮次的正常报名；不能在第一轮人数不足时立刻删除全部报名。
- 已确认开班后，即使合法退课导致人数从 20 降到 18，也不自动取消。
- **第二轮补选全部结算后**，剩余待开班班级才最终取消；名额被撤销，按规定处理已扣权重，并保留历史。
- 第一版未确认“人工取消已确认开班的班级”的复杂审批入口；不应为方便实现凭空开放一个允许管理员随意删除全部学生选课的 API。

### 6.3 初始发放、第一次补选追加与最后清零

建议以服务端自动任务调用业务 Service：

```text
正常选课开始：对全部有资格学生各发放 100 点（每学期每人仅一次）
第一次补选开始：对全部有资格学生各追加 50 点（每学期每人仅一次）
第二次补选结束：先完成本轮结算及必要返还、最终取消、审计
最后：清零各学生本学期剩余可用权重并标记失效
```

**不要给学生开放** `/me/grant-weights`、`/me/clear-wallet` 或手动加减余额的接口。管理端若提供补偿入口，也必须有独立授权、审计与防重复策略，第一版**不要求**复杂人工补偿机制。

### 6.4 轮次结算中的安全恢复

| 失败场景 | 系统必须保证 |
|---|---|
| 结算时数据库连接异常 | 不产生部分名额与账目永久不一致 |
| 同一教学班被两个任务同时结算 | 只有一份有效结算结果，不重复随机 |
| 发放 50 点的任务重跑 | 每名符合资格学生总共只增加一次 50 点 |
| 最终取消班级和清零同时触发 | 先完成正确取消及应返还，再清零 |
| 管理员此时缩容 | 使用统一并发保护，不超员、不出现奇数容量 |
| 某学生已获其他活动目标，轮末资格冲突 | 不可使同学生最终拥有两门实际冲突课程；处理方式按设计评审确定 |

**待技术评审（D-04、D-05、D-07、D-08）**：轮次/班级结算事务粒度、锁顺序、学生资格改变时如何失效申请、分段恢复方式等在数据库设计阶段统一确定；不得偷偷把“单次结算成功”变成“重跑可更换赢家”。

---

## 7. 教师普通成绩录入、首次录齐及 72 小时审核 API（第二核心）

### 7.1 查询我的授课班级和学生名单

**`GET /api/v1/teacher/sections?semesterId=202701`**：仅返回本人当前负责的教学班。

**`GET /api/v1/teacher/sections/{sectionId}/students`**：仅本教学班的有效学生名单；不能因拥有教师角色就查询全校学生。

建议返回班级 `sectionId`、学生 ID/学号/姓名、是否有可录入成绩的合法名额、普通审核状态；不返回学生无关的其他课程成绩或全部学籍历史。

### 7.2 查看普通成绩录入列表

**`GET /api/v1/teacher/sections/{sectionId}/grade-entries`**

```json
{
  "sectionId": 4001,
  "reviewStatus": "NOT_STARTED",
  "firstCompletedAt": null,
  "reviewEndAt": null,
  "entries": [
    { "enrollmentId": 8101, "studentNo": "2026010001", "entryType": "NUMERIC", "rawScore": 85 },
    { "enrollmentId": 8102, "studentNo": "2026010002", "entryType": "DEFERRED", "rawScore": null },
    { "enrollmentId": 8103, "studentNo": "2026010003", "entryType": "UNENTERED", "rawScore": null }
  ]
}
```

`NUMERIC` 的数值 0 是合法已录入结果；`DEFERRED`、合法缺考已记录状态也可计入录齐；只有 `UNENTERED` 不能算已录齐。学生不能调用这个教师草稿接口。

### 7.3 为某位学生保存一个普通结果

**`PUT /api/v1/teacher/sections/{sectionId}/grade-entries/{enrollmentId}`**

普通数值成绩请求：

```json
{
  "entryType": "NUMERIC",
  "rawScore": 85
}
```

获批缓考请求：

```json
{
  "entryType": "DEFERRED",
  "rawScore": null,
  "approvalReference": "<如业务数据已有合法审批依据>"
}
```

缺考请求：

```json
{
  "entryType": "ABSENT",
  "rawScore": null,
  "absenceApproved": false
}
```

**请求语义约束**：

- 教师必须为当前班级授课负责人；`enrollmentId` 必须属于该班真实有效名额。
- 数值成绩范围 0～100；是否允许小数及精度以设计评审为准；不接受 101 或负分。
- 获批缺考/缓考不能直接冒充某个数值 0；未获批缺考**原始事实仍为缺考**，但正式发布后当前课程最终认定按 0 分处理。
- 教师只能在合法普通录分截止前，或持有针对本班的有效迟交授权时，补齐尚未录入内容。
- 该班首次全部记录齐全后启动 72 小时审核；审核期间允许合法修改已录入结果，但不得清空为 `UNENTERED`，也不得重新开始计时。
- 一旦普通成绩正式发布，不再允许通过本 API 直接改当前正式成绩；必须走精准更正审批。
- 如果已经开始录入任意有效学生结果，整个教学班的**普通退课功能被锁定**。

成功 `data` 建议：

```json
{
  "sectionId": 4001,
  "enrollmentId": 8101,
  "entryType": "NUMERIC",
  "rawScore": 85,
  "saved": true,
  "reviewStatus": "REVIEWING",
  "firstCompletedAt": "<若首次录齐则返回首次完成时间>",
  "reviewEndAt": "<若已启动则返回固定72小时后的时间>"
}
```

**关键实现提示**：教师保存结果与“是否首次录齐”的判断，必须在统一的班级级并发保护下完成。否则两个老师同时补齐最后两人的成绩，可能都以为自己是最后一位，导致创建两个审核周期。

### 7.4 查询 72 小时审核状态

**`GET /api/v1/teacher/sections/{sectionId}/grade-review`**

```json
{
  "sectionId": 4001,
  "recordedCount": 40,
  "requiredCount": 40,
  "reviewStatus": "REVIEWING",
  "firstCompletedAt": "2027-08-01T02:30:00Z",
  "reviewEndAt": "2027-08-04T02:30:00Z",
  "publishedAt": null
}
```

- 若全班没有全部录齐，`firstCompletedAt/reviewEndAt` 都应为 `null`；不能提前倒计时。
- 教师审核期间修改成绩不会刷新 `firstCompletedAt`，不允许修改使已经开始的 72 小时回到 0。
- 审核到期由系统自动整体发布，无需教师手工点击“正式发布”；普通教师没有绕过倒计时提前发布的 API。

### 7.5 普通正式成绩自动发布

这是**内部业务能力**，建议自动任务调用 `GradePublishService.publishIfReviewExpired(sectionId, clock)` 一类操作，而不是开放学生/教师可调用的任意提前发布接口。

要求：

1. 班级此前确实首次全员录齐，72 小时已到期。
2. 普通最终认定结果一次性正式生效，记录发布时刻。
3. 获批缓考/缺考可正式发布为非数值状态；未经批准缺考应正式按 0 分的规则产生当前数值结果，同时保留缺考原始事实。
4. 学生整个审核期不能看到本轮新分；只有正式生效后从 `/me/grades` 查询到结果。
5. 重复执行不得再次发布、再次生成历史版本，也不得形成部分学生已经生效另一部分尚未生效的状态。

---

## 8. 学期成绩截止、迟交授权与成绩更正 API

### 8.1 普通成绩录入截止查询与修改

**`GET /api/v1/admin/semesters/{semesterId}/grade-deadline`**

```json
{
  "semesterId": 202701,
  "termLastSunday": "2027-07-18",
  "defaultDeadlineDate": "2027-08-01",
  "configuredDeadlineDate": "2027-08-01",
  "minimumDeadlineDate": "2027-07-18",
  "maximumDeadlineDate": "2027-08-17"
}
```

**`PATCH /api/v1/admin/semesters/{semesterId}/grade-deadline`**

```json
{
  "newDeadlineDate": "2027-08-08",
  "reason": "根据本学期成绩汇总安排调整截止日期"
}
```

校验：

- 管理员身份；目标学期存在。
- 新日期在第 20 周星期日至其后第 30 日（含）之间。
- **旧的当前普通截止尚未过期**才能修改；过期不可通过普通 PATCH 延长。
- 修改须记录原日期、新日期、理由、操作者、时间。
- 若已有某班启动了合法 72 小时审核，即使普通截止过去，审核和自动发布仍继续。

### 8.2 某教学班的迟交补录授权

**`POST /api/v1/admin/sections/{sectionId}/late-grade-grants`**

```json
{
  "startAt": "2027-08-09T00:00:00+08:00",
  "endAt": "2027-08-11T00:00:00+08:00",
  "reason": "该班级符合迟交成绩补录条件"
}
```

- 只有管理员可创建；针对一个教学班；时间范围合法且同班已存在的有效授权之间**不得重叠**。
- 可以先后创建多个不重叠授权，历史保留。
- 只允许补齐尚未合法录入的普通结果；不能绕过已发布成绩的更正审批，不得重置已开始的审核倒计时。
- 教师提交普通成绩时应实际校验该授权**现在仍有效**；不是只因历史存在批准记录就永久有权限。
- 不要求首版建设复杂“授权撤销、转移、跨班审批”流程，除非另有需求确认。

### 8.3 已发布成绩更正——教师申请

**`POST /api/v1/teacher/grade-corrections`**

```json
{
  "enrollmentId": 8101,
  "requestedTargetScore": 90,
  "reason": "复核发现成绩登记错误"
}
```

成功 `data`：

```json
{
  "requestId": 9901,
  "enrollmentId": 8101,
  "oldEffectiveScore": 85,
  "requestedTargetScore": 90,
  "status": "PENDING",
  "officialScoreChanged": false
}
```

- 只有与合法授课班级关联的教师才能为特定学生发起申请。
- 仅针对**已正式发布**的当前成绩走更正流程；普通未发布草稿不应走这里。
- 同一学生×原教学班在同一时刻至多有一条待审或已批准未执行的活动更正申请；并发请求必须防重。
- 申请创建、审批等待期间，学生正式成绩和 GPA **不变化**。

### 8.4 教师撤回待审核申请

**`POST /api/v1/teacher/grade-corrections/{requestId}/withdraw`**

```json
{ "reason": "教师复核后撤回未审核申请" }
```

只允许申请教师撤回自己仍为 `PENDING` 的申请；一旦进入已批准、拒绝、执行或失效状态，不再通过该入口撤回。撤回后保留历史，不得被管理员随后批准。

### 8.5 管理员精准审批或驳回

**`POST /api/v1/admin/grade-corrections/{requestId}/approve`**

```json
{
  "approvedTargetScore": 90,
  "approvedUntil": "2027-08-20T12:00:00+08:00",
  "reviewNote": "复核后批准按指定分数更正"
}
```

成功 `data`：

```json
{
  "requestId": 9901,
  "status": "APPROVED",
  "approvedTargetScore": 90,
  "approvedUntil": "2027-08-20T12:00:00+08:00",
  "officialScoreChanged": false
}
```

审批范围只能是**该具体申请、学生及目标分数**，期限不能缺失；不能把审批实现为“这个教师获得无限期随意改全部成绩的权限”。

**`POST /api/v1/admin/grade-corrections/{requestId}/reject`**

```json
{ "reason": "审核未发现有效的成绩更正依据" }
```

拒绝后申请关闭但历史保留；新的合法申请可以在不存在其他活动申请时重新提出。

### 8.6 获批教师执行——真正即时生效

**`POST /api/v1/teacher/grade-corrections/{requestId}/execute`**

```json
{ "confirmation": true }
```

响应 `data`：

```json
{
  "requestId": 9901,
  "status": "EXECUTED",
  "oldEffectiveScore": 85,
  "newEffectiveScore": 90,
  "executedAt": "<实际执行时间>",
  "officialScoreChanged": true,
  "requiresAnother72HourReview": false
}
```

必须检查：当前教师真实授权、具体学生/班级归属、申请为已批准未执行、当前未过期、批准的目标分数准确、当前正式成绩版本没有被另一业务合法更新覆盖。成功执行时**在同一事务**更新当前正式成绩、写历史版本、关闭申请；学生随后看到新成绩，统计和 GPA 按新正式版本更新，**不再等待 72 小时**。

### 8.7 已批准授权到期的自动失效

由内部定时任务/业务访问校验进行。到期尚未执行时将申请标为 `EXPIRED` 并解除活动占用；不能让过期申请永久阻止后续申请，也不能拿到期授权继续改分。

**不建议暴露**学生或教师可直接调用的“延长授权到期时间”“跳过管理员批准”接口。

---
## 9. 特殊考试：资格、批次、名单冻结、录分和 72 小时生效

### 9.1 一学期一课程一个特殊考试批次

**`POST /api/v1/admin/special-exam-batches`**

```json
{
  "semesterId": 202701,
  "courseId": 1001,
  "description": "本学期该课程统一特殊考试批次"
}
```

响应 `data`：

```json
{
  "batchId": 12001,
  "semesterId": 202701,
  "courseId": 1001,
  "status": "COLLECTING",
  "participantsFrozenAt": null,
  "firstCompletedAt": null,
  "reviewEndAt": null,
  "effectiveAt": null
}
```

**已确认规则**：`semesterId + courseId` 组合最多只有一个合法特殊考试批次，不得为了个别缺席者反复创建第二批次。批次覆盖该课程不同原教学班的符合资格学生，正式结果最终回到他们各自原始课程修读关系。

第一版不设计复杂考场、监考、第二批补救考试或批次撤销重建 API；不能借通用 `DELETE` 绕过唯一批次限制。

### 9.2 自动汇总符合资格学生

**`GET /api/v1/admin/special-exam-batches/{batchId}/participants`**

在未冻结状态，名单应由系统根据**当前正式有效的原课程结果及合法审批状态**自动形成并同步。接口可在响应里返回同步依据和当前更新时间，不能仅依赖管理员逐个手工补人。

候选响应：

```json
{
  "batchId": 12001,
  "status": "COLLECTING",
  "participantsFrozen": false,
  "participants": [
    {
      "participantId": 12101,
      "studentId": 10001,
      "originalEnrollmentId": 8101,
      "originalSectionId": 4001,
      "eligibilityType": "ORDINARY_FAIL",
      "originalOfficialScore": 55
    },
    {
      "participantId": 12102,
      "studentId": 10002,
      "originalEnrollmentId": 8201,
      "originalSectionId": 4002,
      "eligibilityType": "APPROVED_DEFERRED",
      "originalOfficialScore": null
    }
  ]
}
```

自动资格范围：

| 原普通课程正式状态 | 是否进入特殊考试 | 成绩认定模式 |
|---|---|---|
| 数值成绩 0～59 | 自动进入 | 普通补考，及格后封顶 60 |
| 未经批准原考试缺考，正式认定 0 | 自动进入 | 普通补考，及格后封顶 60 |
| 已批准缺考且原课程尚无数值成绩 | 具备合法资格 | 及格后采用特殊考试真实分数 |
| 已批准缓考且原课程尚无数值成绩 | 具备合法资格 | 及格后采用特殊考试真实分数 |
| 已及格且无合法特殊考试资格 | 不应自动进入 | 不适用 |
| 原课程尚在首次 72 小时审核中 | 不以未生效草稿取得普通补考资格 | 不适用 |

**需要防止**：同一学生同一原教学班在该批次出现两个有效参加记录；普通未发布草稿提前生成资格；把未批准缺考错误归类为获批高分照实认定。

### 9.3 考前管理员确认冻结名单

**`POST /api/v1/admin/special-exam-batches/{batchId}/freeze`**

```json
{
  "confirmation": true,
  "reason": "考试前确认本批次参加名单"
}
```

成功：保存冻结时刻、确认者和名单版本。**名单未冻结不得提交特殊考试成绩**。冻结后第一版不再提供普通人员增删，不能把名单持续自动同步到已冻结批次。

资格来源在冻结前的同步与冻结事务必须协调，避免管理员点击冻结同时后台补入资格，造成冻结名单不一致。冻结后的极端新增资格和第二批次补救未成为首版必做功能。

### 9.4 指定多名批次负责教师及合法变更

**`PUT /api/v1/admin/special-exam-batches/{batchId}/teachers`**

```json
{
  "teacherIds": [2001, 2002],
  "reason": "确定负责特殊考试成绩录入的教师"
}
```

- 可以有一名或多名批次负责教师。
- 批次正式生效前，管理员可有审计地增加、移除、替换负责人；在 72 小时审核期间更换负责人不重启审核、不改冻结名单。
- 不应把“替换负责人集合”实现成删除所有历史任职记录；保留原负责人、变更理由、管理员及时间。
- 被指派负责特殊批次，只获得这个批次合法录入/查看工作权限；不能因此修改其他教师的普通教学班成绩。
- 若接口选用 `PUT` 替换整个当前集合，需要明确定义并发提交时的版本冲突或幂等方式。

### 9.5 批次负责教师查询名单并录入结果

**`GET /api/v1/teacher/special-exam-batches/{batchId}/participants`**：只有当前授权的批次教师可以读取其负责批次参加名单及可录入结果。若批次仍未冻结，不能绕过冻结条件开始写分。

**`PUT /api/v1/teacher/special-exam-batches/{batchId}/participants/{participantId}/result`**

普通数值考试结果：

```json
{
  "resultType": "NUMERIC",
  "rawScore": 85
}
```

特殊考试再次缺考：

```json
{
  "resultType": "ABSENT",
  "rawScore": 0,
  "absenceReason": "该次考试缺考"
}
```

**服务端必须按照资格类型计算将来可认定的最终分数**，不能相信客户端直接传入 `awardedFinalScore`。结果尚未正式生效时，只能保留草稿或拟认定结果，不改学生当前正式成绩。

| 资格 | 特殊考试原始分 | 正式生效后最终课程分 | 绩点 |
|---|---:|---:|---:|
| 普通原不及格 | 85 | 60 | 1.0 |
| 普通原不及格 | 50 | 0 | 0 |
| 未获批准原缺考 | 90 | 60 | 1.0 |
| 已批准缓考 | 85 | 85 | 3.5 |
| 已批准缺考 | 90 | 90 | 4.0 |
| 任意合法资格 | 特殊考试再次缺考，记录 0 | 0 | 0 |

**已确认及格线**：所有课程 60 分；所有资格特殊考试低于 60 分，最终课程分统一认定为 0。获批缺考/缓考只在考试及格时按实际分给最终课程成绩。

### 9.6 特殊考试批次首次录齐与审核状态

**`GET /api/v1/teacher/special-exam-batches/{batchId}/review`**

```json
{
  "batchId": 12001,
  "participantsFrozen": true,
  "requiredCount": 25,
  "recordedCount": 25,
  "status": "REVIEWING",
  "firstCompletedAt": "2027-08-10T03:00:00Z",
  "reviewEndAt": "2027-08-13T03:00:00Z",
  "effectiveAt": null
}
```

- 冻结名单内全部学生录入合法结果后，**首次完整时刻**开始 72 小时审核。
- 审核中允许当前授权负责教师修改合法结果，但不能靠清空或编辑重启 72 小时；负责人变更也不能重启。
- 学生此时仍通过 `/me/grades` 看此前正式结果，不能看到批次当前草稿或拟认定结果。

### 9.7 批次到期统一生效——内部自动任务

自动生效业务必须：

1. 只处理已经冻结名单、全员结果录齐且 72 小时已到期的批次。
2. 在一致的原子边界内根据资格类型核算每人的最终认定分。
3. 更新他们各自**原教学班关联**的 `current_course_result`，同时保存原正式版本和特殊考试真实原始分。
4. 整个批次统一生效，不允许学生看到只有部分同批次人已换新分的中间状态。
5. 不重新触发普通教学班原来的 72 小时审核。
6. 自动任务重试/服务重启不重复生效、不重复写历史，也不能覆盖更新的合法更正版本。

不要求首版为普通教师提供“手工强制批次生效”按钮；调度失败恢复可通过受控内部业务入口和日志检查完成。

---

## 10. 个人正式成绩、累计 GPA、同年级名次与统计接口

### 10.1 学生查询当前正式课程结果

**`GET /api/v1/me/grades?semesterId=202701`**，`semesterId` 可选；不传时查询有权限的本人历次课程结果（分页策略待定）。

响应 `data.items` 示例：

```json
[
  {
    "semesterId": 202701,
    "sectionId": 4001,
    "courseId": 1001,
    "courseCode": "CS101",
    "courseName": "Java 程序设计",
    "credits": 4.0,
    "resultKind": "NUMERIC",
    "effectiveScore": 60,
    "officiallyEffectiveAt": "2027-08-13T03:00:00Z",
    "displayValue": "60"
  },
  {
    "semesterId": 202701,
    "sectionId": 4010,
    "courseId": 1010,
    "courseName": "数据库原理",
    "credits": 3.0,
    "resultKind": "DEFERRED",
    "effectiveScore": null,
    "officiallyEffectiveAt": "2027-08-04T02:30:00Z",
    "displayValue": "缓考"
  }
]
```

- 只查询**已经正式生效**的结果，不返回教师普通成绩草稿、特殊考试原始分、内部更正审批、其他学生个人信息。
- 普通 55 分直到补考批次正式生效前仍可能显示 55；批次正式生效后若补考原始考 85，学生显示的是**最终 60**，不显示原始 85。
- 已批准缓考/缺考且尚无正式数值分时可以显示合法正式特殊状态；`effectiveScore=null` 不能冒充 0。
- 非正常在读学生仍有权登录查询本人的历史正式成绩。

### 10.2 学生查询本人累计 GPA

**`GET /api/v1/me/gpa`**

```json
{
  "studentId": 10001,
  "cumulativeGpa": 3.125,
  "countedCredits": 16.0,
  "hasOfficialNumericGrades": true,
  "displayGpa": "3.13",
  "calculatedFrom": "CURRENT_OFFICIAL_RESULTS"
}
```

**注意**：`displayGpa` 是界面展示值；排名使用真实计算精度，**不能**使用显示的两位小数判断并列。没有正式数值课程时：

```json
{
  "cumulativeGpa": null,
  "countedCredits": 0,
  "hasOfficialNumericGrades": false,
  "displayGpa": "暂无数据"
}
```

按已确认规则：

```text
正式最终数值 >= 60：绩点 (分数 - 50) / 10
正式最终数值 < 60：绩点 0，但课程学分仍进分母
获批缓考/缺考等无正式数值：分子与分母均不参与
累计 GPA = SUM(课程学分 × 单科绩点) / SUM(参与学分)
```

不允许拿特殊考试原始分、教师草稿分或已经被新正式结果替换的旧分参与当前 GPA。

### 10.3 学生查询仅本人同入学年级排名

**`GET /api/v1/me/cohort-rank`**

有资格的正常在读学生响应：

```json
{
  "studentId": 10001,
  "entryYear": 2026,
  "cumulativeGpa": 3.125,
  "hasCurrentFailedCourses": false,
  "cohortRank": 12,
  "eligibleForRanking": true,
  "rankingRule": "RANK_GPA_DESC_NO_CURRENT_FAIL_FIRST"
}
```

休学/退学/已毕业或没有任何正式数值课程时：

```json
{
  "entryYear": 2026,
  "cohortRank": null,
  "eligibleForRanking": false,
  "reason": "当前无有效排名资格"
}
```

规则：

- 仅同一**入学年级**比较，不跨年级混排；仅**正常在读且至少一门正式数值课程**的学生参与；不设置最低学分门槛。
- 首先累计 GPA 降序；同 GPA 时**当前无挂科**优先；GPA 与当前挂科有/无完全相同才并列。
- 竞赛排名 `RANK()`，例如 `1,1,3,4`；不是 `DENSE_RANK()`。
- 补考通过、最终 60 的学生**当前不再挂科**；但其该课程仍只得绩点 1.0。
- **严禁开放** `/students/{arbitraryStudentId}/rank` 给学生自行查询、`/cohort-ranks/leaderboard` 给学生查看全榜。
- SQL 应先计算同年级合资格学生的排名，**最后才筛选登录学生本人**；不能先用 `studentId` 把窗口数据缩成一行再执行 `RANK()`，否则名次会错误地总是 1。

### 10.4 教师本人教学班成绩统计

**`GET /api/v1/teacher/sections/{sectionId}/statistics`**

```json
{
  "sectionId": 4001,
  "officialNumericResultCount": 38,
  "sectionAverage": 76.5,
  "displayAverage": "76.50"
}
```

- 统计**当前已经正式生效且为数值的最终课程成绩**。
- 正式 0 分必须计入平均；获批缓考/缺考等无数值正式状态不计入。
- 若完全没有正式数值成绩：`sectionAverage=null`、`displayAverage="暂无数据"`，而非把 0 误当均分。
- 教师只能查询**本人负责的普通教学班**，不因为被指定特殊考试负责人而自动查看其他班。
- 如果老师课程成绩正在 72 小时审核，统计仍使用旧的正式生效结果，而不是新草稿。

### 10.5 管理员合法成绩统计

`GET /api/v1/admin/sections/{sectionId}/statistics` 和 `GET /api/v1/admin/statistics/sections` 是候选管理接口。允许管理员在职权范围内查询班级统计汇总，可筛选学期、课程、教师；学生不可通过此接口获得完整年级榜单或其他个人成绩。

**管理员完整统计的具体导出范围**、是否需要数据脱敏和导出日志在权限设计时明确；本项目不因管理员可以查询统计，就自动要求第一版加入复杂报表导出服务。

---
## 11. 权限矩阵和防越权设计

### 11.1 角色与资源矩阵

| 接口资源或动作 | 学生 | 教师 | 管理员 | 额外限定 |
|---|---|---|---|---|
| 登录、查看自身角色 | 自己 | 自己 | 自己 | 账号状态须有效 |
| 本人历史正式成绩 | 允许 | 作为普通教师不可随意查询任意学生 | 按职责 | 非正常在读学生仍可查本人 |
| 报名、撤回、退课 | 仅本人 | 禁止 | 不得通过普通入口绕规则 | 学籍正常、时间和业务条件满足 |
| 本人权重余额及投入历史 | 仅本人 | 禁止 | 管理审计按职责 | 不能传 studentId 冒名 |
| 教学班基本公开信息 | 合法可见 | 合法可见 | 全部合法范围 | 不泄露其他学生出价/成绩 |
| 学生、教师、账号、课程、学期管理 | 禁止 | 禁止 | 允许 | 修改仍需遵守冻结和状态规则 |
| 替换授课教师/修改容量 | 禁止 | 禁止 | 允许 | 排课冲突、容量正偶数、不低于人数 |
| 普通成绩草稿录入 | 禁止 | 仅本人授课班 | 不直接绕过教师角色 | 录分窗口和审核状态 |
| 本教学班草稿学生名单 | 禁止 | 仅本人授课班 | 按管理职责 | 不泄露不相关学生 |
| 已发布成绩更正申请 | 禁止 | 仅本人负责资源 | 审批管理 | 同一成绩一条活动更正 |
| 更正申请审批 | 禁止 | 禁止 | 允许 | 精确目标、期限和审计 |
| 更正申请执行 | 禁止 | 被授权教师 | 不替代教师直接执行 | 只改授权的该条成绩 |
| 批次考试成绩录入 | 禁止 | 仅指定当前负责人 | 管理/授权 | 名单已冻结且批次状态允许 |
| 本人 GPA 和同入学年级名次 | 仅本人 | 禁止查询学生榜单 | 管理权限需另定 | 不提供学生全榜 |
| 教学班正式成绩统计 | 禁止查看他人 | 仅本人授课班 | 全部合法范围 | 只基于当前正式成绩 |

### 11.2 服务端权限校验顺序（建议）

1. 校验调用者已经通过认证、令牌有效。
2. 校验角色可访问该种业务 API。
3. 校验具体对象归属：本人申请/成绩、本人授课班、批次当前授权。
4. 校验学籍与目标资源的业务状态：是否正常在读、班级是否取消、成绩是否已发布。
5. 校验学校业务时间：窗口是否开放、授权是否到期、72 小时审核是否结束。
6. 执行数据库事务内的并发再校验及变更。
7. 返回**与本角色可见范围一致**的数据，不把内部管理员 DTO 直接序列化给学生。

不能因为通过了第 2 步（例如用户有 `TEACHER` 角色），就跳过第 3 步（这个班不是他教的）。

---

## 12. 统一错误码目录（建议，不是已实现代码）

> HTTP 状态用于区分认证、权限、参数、业务冲突；稳定的业务 `code` 用于前端提示和测试断言。本文错误码字符串属于候选 API 契约，编码前可统一命名，但必须保持语义一致。

| 业务码 | 建议 HTTP | 触发情况 | 不应产生的副作用 |
|---|---:|---|---|
| `INVALID_ARGUMENT` | 400 | JSON 字段非法/缺失 | 不保存部分无效数据 |
| `AUTHENTICATION_REQUIRED` | 401 | 未登录或令牌无效 | 不泄露受保护信息 |
| `ACCESS_DENIED` | 403 | 角色或资源归属不允许 | 不返回其他学生详情 |
| `RESOURCE_NOT_FOUND` | 404 | 目标资源不存在 | 不修改任何资源 |
| `STUDENT_NOT_ENROLLED` | 409 | 非正常在读学生尝试选退课 | 不产生报名或退课 |
| `ROUND_NOT_OPEN` | 409 | 当前轮次未开始或已截止 | 不扣权重 |
| `ROUND_ALREADY_SETTLED` | 409 / 幂等200 | 已结算轮次被重复执行 | 不再随机、扣还或重复录取 |
| `INSUFFICIENT_WEIGHT` | 409 | 权重余额不足 | 不创建申请、余额不变 |
| `WEIGHT_ALREADY_GRANTED` | 409 / 幂等200 | 发放 100/50 点重复执行 | 不再次加点 |
| `WALLET_CLEARED` | 409 | 清零后试图继续正常报名 | 不恢复可用权重 |
| `DUPLICATE_COURSE_ENROLLMENT` | 409 | 同学期同课程已有有效目标 | 不重复报名 |
| `ENROLLMENT_TIME_CONFLICT` | 409 | 教学周、星期、节次实际重叠 | 不扣权重 |
| `COURSE_ELIGIBILITY_DENIED` | 409 / 403 | 专业或入学年级不符合选课要求 | 不新增有效报名 |
| `SECTION_CANCELED` | 409 | 班级最终不开 | 不接受新报名 |
| `SECTION_CAPACITY_INVALID` | 400 | 容量非正偶数 | 不改容量 |
| `SECTION_CAPACITY_CONFLICT` | 409 | 缩容低于当前有效名额 | 不修改容量 |
| `TEACHER_SCHEDULE_CONFLICT` | 409 | 教师实际教学时间冲突 | 不替换教师 |
| `SECTION_SCHEDULE_FROZEN` | 409 | 开放后试图普通修改课表 | 不改课表 |
| `APPLICATION_NOT_WITHDRAWABLE` | 409 | 已截止或已结算申请试图撤回 | 不返第二次权重 |
| `ENROLLMENT_NOT_DROPPABLE` | 409 | 已过退课期、成绩已录或资源不合法 | 不退课、不释放容量 |
| `GRADE_ENTRY_CLOSED` | 409 | 正常录分截止且无合法迟交许可 | 不写草稿 |
| `GRADE_ENTRY_INVALID` | 400 / 409 | 分数或特殊状态组合不合法 | 不产生伪 0 分 |
| `GRADE_REVIEW_LOCKED` | 409 | 审核中试图清空或重启审核 | 不重新计时 |
| `GRADE_ALREADY_PUBLISHED` | 409 | 已发布成绩走普通录入入口 | 不改正式成绩 |
| `GRADE_CORRECTION_ACTIVE_EXISTS` | 409 | 同一成绩有另一条活动更正 | 不新增并行审批 |
| `GRADE_CORRECTION_NOT_APPROVED` | 409 | 未获精准授权就执行 | 正式成绩不变 |
| `GRADE_CORRECTION_EXPIRED` | 409 | 授权已到期 | 正式成绩不变 |
| `GRADE_CORRECTION_TARGET_MISMATCH` | 409 | 执行目标与批准目标不一致 | 正式成绩不变 |
| `LATE_GRADE_GRANT_OVERLAP` | 409 | 迟交授权时段重叠 | 不建立重叠授权 |
| `SPECIAL_BATCH_DUPLICATE` | 409 | 同学期同课程第二个特殊考试批次 | 不创建重复批次 |
| `SPECIAL_BATCH_NOT_FROZEN` | 409 | 名单未冻结就尝试录分 | 不接受结果 |
| `SPECIAL_BATCH_NOT_READY` | 409 | 全员未录齐或审核未到期 | 不提前发布 |
| `SPECIAL_BATCH_ACCESS_DENIED` | 403 | 教师未获该批次授权 | 不泄露/修改批次成绩 |

**业务码实现前要统一**：不要同一类余额不足有时返回 `INSUFFICIENT_WEIGHT`、有时变成 `ERROR_1001`；测试应能根据稳定错误码断言原因。业务异常 409 **不代表数据库必须抛异常后才处理**，可以在服务层主动抛出受控业务错误。

---

## 13. 幂等、防重复、并发与自动任务接口边界

### 13.1 请求重试与业务幂等

| 写操作 | 如何避免重复副作用 | 验证重点 |
|---|---|---|
| 创建报名+扣权重 | 申请活动唯一性 + 事务 + 可选 `Idempotency-Key` | 双击一次报名不扣两次 |
| 撤回报名+返还 | 状态条件更新 + 原申请来源约束 | 多次撤回只返一次 |
| 单班轮次结算 | `roundId + sectionId` 唯一结算事实 | 重跑不重抽签、不再次录取 |
| 初始发放 100 | 学生×学期的唯一发放记录 | 服务重启不重发 |
| 第一次补选发放 50 | 学生×学期×发放事件唯一性 | 定时任务重跑不额外增加 |
| 最终取消班级 | 最终取消状态和对应名额/流水关联 | 取消两次不会再退款 |
| 退课 | 当前有效名额唯一 + 退课状态/审计 | 不能多次释放或退款 |
| 最终清零 | 学生学期账户已清零标记 | 不重复生成多笔余额错误 |
| 普通 72 小时自动发布 | 首次审核唯一 + 已发布状态 + 事务 | 不提前发布、不部分发布 |
| 迟交授权 | 同班授权区间不重叠 | 并发申请不能建立重叠期 |
| 成绩更正执行 | 同一申请只能成功执行一次 | 不重复写历史版本 |
| 特殊考试批次正式生效 | 同批次仅一份已生效版本 | 重跑不会覆盖成绩/新增历史 |

### 13.2 `Idempotency-Key` 是什么（候选）

对重复风险高的创建操作，可选使用：

```http
POST /api/v1/me/enrollment-applications
Authorization: Bearer <token>
Idempotency-Key: 74b9example-client-request-001
Content-Type: application/json
```

同一调用者对同一业务请求在允许的重试期内重复提交同一 Key，可由服务端返回首次已提交结果；**不能将 Key 设计成可以被另一个学生复用以取得别人的资源**。Key 与请求体摘要、当前登录人、业务动作相绑定。Key 机制属于技术方案候选，第一版最终是否采用统一约束/Key 需要开发阶段评审。

**没有 Key 也不能发生重复扣款**：业务唯一性、状态条件、事务仍是必要保障，不能把防重全部委托客户端。

### 13.3 并发测试必须验证最终数据库事实

- 两个学生争一个名额，不能多录；**不能按请求先到决定是否赢得权重竞争**。
- 同一学生同时报名两门不冲突课程，余额必须正确扣除；不能发生两笔请求都读到旧余额而超额花费。
- 同一学生并发报名同课程不同班级，不能同时保留两个有效目标。
- 同一学生并发报名实际冲突的课程，最多保留合法的非冲突集合。
- 管理员缩容与轮次结算并发时，容量正偶数且不能小于有效名额。
- 同一班级两位教师同时录入最后两位学生成绩，审核只能首次启动一次。
- 轮次或批次自动任务超时重试，结果必须稳定并可追溯。
- 所有失败请求都要检查**余额、名额、流水、历史、正式结果**是否发生了不应有的变化，而非只检查 HTTP 409。

### 13.4 内部定时任务（不自动变成公开 API）

| 触发任务 | 条件 | 必须保证 |
|---|---|---|
| `grantInitialWeight` | 正常选课开始 | 符合资格者每学期只获一次 100 |
| `settleRound` | 当前轮次截止 | 竞争录取、落选返还、待开班处理可重试 |
| `grantSupplementWeight` | 第一次补选开始 | 所有合资格学生只增加一次 50 |
| `finalizeEnrollmentAndClear` | 第二轮截止且所有结算处理完成 | 先取消不足班级和必要返还，再清零 |
| `publishOrdinaryGrades` | 普通教学班首次完整录齐满 72 小时 | 自动整体正式生效、不重发 |
| `expireCorrectionGrants` | 精准更正授权到期未执行 | 自动失效并释放活动申请占用 |
| `publishSpecialExamBatch` | 冻结批次全部录齐满 72 小时 | 批次一次性正式生效，不重启普通审核 |

**测试辅助方式**：优先用可控时钟、真实 Service 接口和独立测试用例触发自动任务，不需要在生产环境公开“手工强制发布时间”端点。

---

## 14. 推荐的代码分层、DTO 与目录（候选）

### 14.1 建议包结构

```text
src/main/java/<base-package>/
  common/
    api/                  # 成功和失败响应、分页模型
    exception/            # 统一业务异常与处理器
    time/                 # 学校时区与可测试时钟
    security/             # 身份、权限与归属校验
  student/                # 学籍与个人身份管理
  teacher/                # 教师与授课归属
  course/                 # 课程目录、专业年级限制
  semester/               # 学期与三轮选课窗口
  section/                # 教学班、课表、容量、开班
  enrollment/             # 报名、分轮竞争、名额、退课
  weight/                 # 钱包、流水、100/50发放与清零
  grade/                  # 普通成绩录入、72小时发布
  correction/             # 精准更正申请、审批、执行
  specialexam/            # 特殊考试批次、名单、成绩与生效
  statistics/             # 班级统计、GPA、本人名次
  job/                    # 内部调度入口，不对外公开
```

**说明**：这是按领域组织的候选目录，不是要求第一天建立所有空包/Controller/Service。可从阶段 1 的小型学生 CRUD 开始，伴随真实业务逐步增加。

### 14.2 DTO 分离原则

| DTO 类型 | 推荐样例 | 原则 |
|---|---|---|
| 创建请求 | `CreateStudentRequest`、`CreateSectionRequest` | 字段校验，禁止客户端提交服务端控制的 `createdAt/status` |
| 权重报名请求 | `SubmitApplicationRequest` | 用户只指定目标和投入，不指定录取结果、学生身份 |
| 报名响应 | `ApplicationView` | 区分待结算/录取/待开班与余额变化 |
| 成绩录入请求 | `GradeEntryRequest` | 表达 NUMERIC/ABSENT/DEFERRED 与合法字段组合 |
| 更正申请/审批请求 | `CorrectionRequest`、`ApproveCorrectionRequest` | 原分、目标分、审批期限明确 |
| 特殊考试结果请求 | `SpecialExamResultRequest` | 只录真实原始成绩，不信任前端授予的最终分 |
| 当前成绩响应 | `OfficialGradeView` | 不混入审批状态、内部原始分 |
| GPA/名次响应 | `MyGpaView`、`MyRankView` | 不返回其他学生的个人数据 |

**禁止**：直接把数据库 Entity 当所有角色通用 JSON 返回；把数据库密码、私人审批原因或内部随机种子序列化给普通学生；允许客户端提交 `studentId` 来决定操作对象。

### 14.3 Controller、Service、Mapper 的职责

- **Controller**：接收请求、参数形状校验、读取认证主体、返回统一结果；不在 Controller 拼复杂 SQL 或手工操作多个表。
- **Service**：业务顺序、事务边界、状态转换、身份归属、授权、重复请求保护。
- **Mapper**：MyBatis-Plus 用于基础 CRUD；复杂联结、结算候选查询、教学周冲突、GPA 窗口排名可使用自定义 MyBatis XML。
- **Job**：触发受控 Service，不直接跳过业务状态与审计更新。
- **Exception Handler**：统一 HTTP/业务码，避免在每个接口复制大量异常处理。

---

## 15. 重要接口验收矩阵（未执行）

> 本清单是后续测试计划，不是“已通过”的证据。每条需在实际项目中提供测试方法、测试数据、期望与真实结果。建议优先使用真实 MySQL 集成测试验证原子性和窗口函数。

| 编号 | 接口/业务场景 | 期望结果 |
|---|---|---|
| API-T01 | 未登录调用 `/me/grades` | 返回 401，不泄露成绩 |
| API-T02 | 学生传其他学生 ID 试图获取他人 GPA | 不允许跨学生查询 |
| API-T03 | 休学学生查询本人历史正式成绩 | 允许 |
| API-T04 | 休学学生提交选课报名 | 拒绝，无有效申请，无扣点 |
| API-T05 | 学号重复创建 | 拒绝第二次，唯一性不破坏 |
| API-T06 | 停用课程后新建后续教学班 | 拒绝；原有班级继续有效 |
| API-T07 | 曾建班课程尝试修改学分 | 拒绝，不影响历史 GPA |
| API-T08 | 创建容量 35 的教学班 | 拒绝，容量必须正偶数 |
| API-T09 | 将容量 40 缩为 36 且已有 38 人 | 拒绝，不减少已有合法名额 |
| API-T10 | 不连续教学周互不重叠而节次相同 | 不误判课表冲突 |
| API-T11 | 新教师与原教学任务实际冲突 | 更换拒绝，旧教师不变 |
| API-T12 | 开放后教师/管理员普通修改已冻结课表 | 拒绝 |
| API-T13 | 正常选课前 14 天开始、持续 3 天 | 窗口和边界时刻正确 |
| API-T14 | 本学期首次发放 100 点 | 每合资格学生一次 |
| API-T15 | 第一次补选发放 50 点 | 所有合资格学生一次，重复任务不重发 |
| API-T16 | 余额 100，提交投入 40 | 报名成功，余额立即 60，尚未录取 |
| API-T17 | 余额 20，再提交投入 30 | 409，申请和扣点均不存在 |
| API-T18 | 活动报名两门实际时间冲突课程 | 第二笔拒绝，余额不变 |
| API-T19 | 活动报名同学期同课程不同班级 | 第二笔拒绝 |
| API-T20 | 截止前撤回 40 点报名 | 撤回成功，返还 40，一次性生效 |
| API-T21 | 截止后撤回 | 拒绝撤回，不能通过本 API 当作退课 |
| API-T22 | 容量 40，轮次报名 55 人 | 至多录取 40，按权重竞争，其余返还 |
| API-T23 | 同权重争夺最后名额 | 随机胜出，重复结算胜者不改变 |
| API-T24 | 第一轮已录取，下一轮高权重报名 | 已录取者不会被替换 |
| API-T25 | 容量 40，本班仅 19 人获名额 | 待开班，允许后续补选 |
| API-T26 | 容量 40，本班 20 人获名额 | 确认可开班 |
| API-T27 | 确认开班后合法退课降至 18 | 班级继续开班 |
| API-T28 | 第二轮后仍不足最低人数 | 取消班级，撤销名额并正确记账 |
| API-T29 | 第二轮最终全部结算 | 先必要返还后清零，余额归 0 |
| API-T30 | 清零前合法退课 | 释放名额、返还相应可用权重 |
| API-T31 | 清零后合法退课 | 释放名额、返回可用余额 0 |
| API-T32 | 任一学生普通成绩已开始录入 | 整个班级不再允许普通退课 |
| API-T33 | 普通成绩输入 0、100、NULL 的差别 | 0 和 100 合法；未录入不能被误视为 0 |
| API-T34 | 已记录获批缓考 / 缺考状态 | 计入首次全部录齐，不强求数值分 |
| API-T35 | 第一次所有学生成绩/状态录齐 | 自动进入固定 72 小时审核 |
| API-T36 | 审核期间合法修改已录成绩 | 允许修改，审核截止不重置 |
| API-T37 | 审核尚未结束学生查成绩 | 不能看到本次草稿分 |
| API-T38 | 72 小时到期自动普通整班发布 | 全部同事务正式生效，不重发 |
| API-T39 | 普通截止已过但有已启动审核 | 允许审核继续并到期发布 |
| API-T40 | 原普通截止已过再尝试修改截止日期 | 拒绝，用班级迟交许可流程 |
| API-T41 | 同班两个迟交授权时段重叠 | 拒绝后一个 |
| API-T42 | 教师对已发布成绩直接 PUT 草稿 | 拒绝，必须更正审批 |
| API-T43 | 更正申请已获批，但教师未执行 | 本人正式成绩和 GPA 不变 |
| API-T44 | 教师按获批目标分执行 | 即时生效并写历史，不再审核 72 小时 |
| API-T45 | 已批准更正到期 | 自动失效，不能据此再改成绩 |
| API-T46 | 同一成绩并发发起两条活动更正 | 至多一条活动申请 |
| API-T47 | 同一学期同课程创建第二个特殊批次 | 拒绝 |
| API-T48 | 自动收集正式 55 分学生及获批缓考学生 | 正确纳入资格，跨原教学班 |
| API-T49 | 特殊批次名单未冻结，负责教师录分 | 拒绝 |
| API-T50 | 特殊批次首次结果全部录齐 | 进入固定 72 小时审核 |
| API-T51 | 普通补考原始 85 | 批次生效后原课程最终 60，绩点 1.0 |
| API-T52 | 获批缓考原始 85 | 批次生效后最终 85，绩点 3.5 |
| API-T53 | 特殊考试原始 50 或再次缺考 | 批次生效后最终 0，保留缺考事实 |
| API-T54 | 批次审核期间学生查成绩 | 仍返回原已生效正式结果 |
| API-T55 | 批次正式到期自动生效 | 统一更新原班课程结果，不重启原班审核 |
| API-T56 | 当前正式不及格 0 分参与 GPA | 学分计入分母，绩点 0 |
| API-T57 | 获批缓考尚无数值成绩 | 暂不计 GPA；若无其他成绩则暂无数据 |
| API-T58 | 同入学年级相同 GPA 且无挂科两人 | `RANK()` 同列 1，下一位为 3 |
| API-T59 | GPA 相同、一人当前挂科一人已补考通过 | 当前无挂科者优先，已补考通过者不算当前挂科 |
| API-T60 | 学生请求完整同年级排行榜 | 不提供学生可用的排行榜 API |
| API-T61 | 教师查询别人普通教学班统计 | 403 或遵循统一隐藏策略 |
| API-T62 | 自动任务多次触发或并发请求 | 权重、名额、历史、成绩正式版本保持一致 |

### 15.1 每个测试至少记录以下证据

```markdown
#### API-TXX｜测试名称

- 对应需求编号 / CHG：
- 测试环境及数据库版本：
- 当前身份及资源归属：
- 前置数据与操作时刻：
- 请求方法、路径、请求体：
- 预期 HTTP 状态与业务错误码：
- 预期数据库变化或不变化：
- 实际响应：待执行
- 实际 SQL/事务验证：待执行
- 是否通过：待执行
- 异常与修复记录：
- Git Commit（存在时填写）：
```

不能将本文件表格里的“期望结果”改写成“实测通过”；测试必须真实运行并能重复。

---

## 16. 开发阶段的实施顺序与分工

### 16.1 API 设计分阶段落地

| 阶段 | 建议首先落地的接口 | 验收重点 |
|---|---|---|
| 阶段 0 | API 全局规范、候选路径和 DTO 审查 | 与最终业务需求及 ER 一致，标清未决点 |
| 阶段 1 | 学生 CRUD、分页、校验、统一异常 | 一条真正可运行的 Spring MVC → Service → Mapper 链 |
| 阶段 2 | 教师、课程、学期、教学班、课表 | 专业/年级、正偶数容量、教师冲突 |
| 阶段 3 | 权重余额、报名、撤回、轮末结算、退课 | 即扣即还、同权重随机、幂等和并发 |
| 阶段 4 | 普通成绩、72 小时审核、迟交、更正、特殊考试、GPA | 当前正式结果正确、完整状态机 |
| 阶段 5 | 登录、角色权限、本人和教师资源归属 | 401/403、越权和敏感数据保护 |
| 阶段 6 | 真实 MySQL 集成/并发/自动任务测试、SQL 优化 | 多线程、重试、时间边界、EXPLAIN |
| 阶段 7 | OpenAPI、README、测试证据、Docker 部署 | 从空库按步骤复现完整业务闭环 |

### 16.2 先实现一条完整垂直链，不批量空建 Controller

推荐最先完成：

```text
POST /admin/students
  → Controller 校验 CreateStudentRequest
  → Service 检查学号与专业
  → Mapper 写学生
  → 统一响应和异常处理
  → 真实 MySQL 集成测试
  → Git Commit
```

阶段 3 再完整实施：

```text
POST /me/enrollment-applications
  → 身份与正常在读资格
  → 当前三轮时间与课程/课表/活动目标校验
  → 钱包额度检查
  → 同事务保存申请、扣钱包、写权重流水
  → 返回“提交成功但等待结算”的响应
  → 真实余额与并发验证
```

**你的重点实践**：Controller—Service—Mapper 的职责划分、核心数据关系、权重报名和结算事务、普通成绩审核与正式生效、复杂 SQL 和集成测试。AI 可以辅助文档整理、验收用例草拟、代码审查与排障，但不应替代你理解关键锁、事务与查询的过程。

### 16.3 阶段 0 API 设计审查清单

- [ ] 所有公开接口都有明确的角色、资源归属和所需学籍状态。
- [ ] 每个写操作都写明什么时候允许、什么时候拒绝、失败是否回滚。
- [ ] `submit`、`accepted`、`openingConfirmed` 在响应中不混淆。
- [ ] `availablePoints` 是本学期可用余额；名额录取不重复扣除。
- [ ] 清零前与清零后退课返回的可用余额变化不同。
- [ ] 退课前校验实际首次授课日与整班成绩已开始录入条件。
- [ ] 普通审核和特殊批次审核各自 72 小时，内部发布不暴露学生端跳过入口。
- [ ] 原始分、当前正式分、补考最终认定分各自可解释。
- [ ] 已发布成绩只能经精准更正审批与教师执行改变。
- [ ] 同入学年级排名使用 `RANK()`，只输出当前登录学生自己的名次。
- [ ] 已列出 D-01～D-10 未决设计点，真正进入编码前完成必要定稿。
- [ ] API 接口文档和数据库字段/状态命名保持一致。
- [ ] 实际代码、测试、Git 提交未发生时，所有实施状态保持未完成。

---

## 17. 尚待设计评审的项目（不得冒充已确认规则）

| 编号 | 待决定的技术边界 | 已确认的业务界限 | 建议落点 |
|---|---|---|---|
| D-01 | 权重输入是否只支持整数、每笔最小值 | 即时扣除、不得超余额或为负 | `SubmitApplicationRequest` + 数据库类型与测试 |
| D-02 | 三轮具体小时、间隔、时间上界 | 正常轮开学前14天开始，时长3+2+1天 | Semester/Round API 与 Clock |
| D-03 | 已获暂定名额但班级仍待开班时，如何主动退出改选 | 不能同时保留冲突目标、已扣权重可追溯 | Enrollment 生命周期与接口状态 |
| D-04 | 管理员容量修改与结算并发 | 正偶数、不低于已获名额、不超员 | Section/Settlement Service 事务 |
| D-05 | 提交后结算前学籍、专业/年级等资格变更 | 最终不得产生非法有效录取 | 学籍事件/结算时状态校验 |
| D-06 | 分数/学分/GPA 精度及显示精度 | 60线、定点算、不能按展示值并列 | DTO、DDL、GPA SQL |
| D-07 | 定时任务时钟、恢复、自动发布和清零时点 | 72小时、正确顺序和幂等 | Job/Clock/Settlement/Publish Service |
| D-08 | 活动申请跨报名/名额的唯一性 | 同学生同学期同课程和冲突目标均受限制 | 数据库唯一策略/学生锁 |
| D-09 | 物理删除当前名额后的历史成绩引用 | 历史不能丢、退课合法 | 历史快照/稳定修读标识与外键 |
| D-10 | 班级人数与 GPA 是否缓存 | 统计必须与当前正式状态一致 | 首版查询优先、实测再优化 |

以上是接口和数据库实施前应做的**技术评审问题**，不是要求继续发散十个全新业务功能；新的业务变更必须回写正式需求/变更记录。

---

## 18. 设计评审结论与版本记录

### 18.1 本文已完成的工作

- [x] 覆盖学生、教师、管理员的核心候选 API 范围。
- [x] 按最终业务需求写清权重、三轮、待开班、取消、退课与清零行为。
- [x] 区分普通草稿/72 小时审核/当前正式成绩、特殊考试和授权更正。
- [x] 给出统一响应、错误码、权限、核心请求响应示例及 62 个测试场景。
- [x] 明确技术候选方案与未决设计问题，避免未经确认地固定新业务规则。

### 18.2 尚未完成的工作

- [ ] 由你评审各个实际 URL、DTO 字段、状态值与对应数据库模型是否一致。
- [ ] 处理 D-01～D-10 中影响当前接口/DDL 的必要技术决策。
- [ ] 使用 OpenAPI 3.x 把已定稿接口转成可机器校验的 schema。
- [ ] 编写 Controller、Service、Mapper 和安全规则。
- [ ] 使用真实 MySQL 和认证主体完成全部核心集成及并发测试。
- [ ] 记录可复现的调试、Git 提交、Docker 部署和演示证据。

### 18.3 变更维护规则

1. 业务规则发生变化：先在 `docs/变更记录.md` 登记新 CHG，再同步更新正式需求、数据库设计、本 API 文档与对应测试。
2. 只是接口路径或字段命名优化：评估是否影响对外兼容，记录在本文件版本备注和 Git 中；不要无意义地制造业务 CHG。
3. 本文中出现的日期、学生姓名、课程代码、学期编号和 ID 都是**虚构样例**。
4. 不把“文档设计草案完成”写成“功能已实现”，更不把示例 JSON 当作真实运行返回。
5. 若真实数据库 DDL 与本文件候选字段冲突，应在提交代码前同步修订，使正式 API 契约与真实实现一致。

**最终定位**：这是一份可以直接放入 `docs/API接口设计.md` 的**完整详尽候选设计稿**。它服务于下一阶段亲自审查、实现与测试，不替代已确认的《需求与业务规则.md》，也不表示 API 已经上线。
