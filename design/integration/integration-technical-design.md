# ERP系统对接aiflow技术详细设计文档

本文档整合了ERP系统对接aiflow流程中心的技术详细设计和双方任务清单。

> **返回主方案文档：** [ERP系统对接aiflow流程中心设计方案](erp-wms-integration-design.md)  
> **技术原理文档：** [技术原理与接口设计文档](technical-details.md)（包含详细的时序图、数据库设计和实现细节）

---

## 目录

- [第一部分：技术详细设计](#第一部分技术详细设计)
  - [1. 通用说明](#1-通用说明)
  - [2. 基础数据同步方案](#2-基础数据同步方案)
  - [3. SSO单点登录对接方案](#3-sso单点登录对接方案)
  - [4. 任务中心简易集成方案](#4-任务中心简易集成方案)
  - [5. 全面流程引擎集成方案](#5-全面流程引擎集成方案)
- [第二部分：双方任务清单](#第二部分双方任务清单)

---

# 第一部分：技术详细设计

## 1. 通用说明

### 1.1 签名机制

所有aiflow提供的接口都需要使用签名验证（`A-Signature` Header），Odoo提供的接口也需要使用相同的签名机制。

> **推荐方式**：Odoo 可直接使用 aflow 提供的 **Python SDK**（`aflow-client-python`），无需自行实现签名算法。

#### 使用 Python SDK（推荐）

```python
from auth import ASign, Credential

# 创建认证凭证
credential = Credential(
    enterprise_code="COMPANY001",
    app_id="APP001",
    app_secret="your_app_secret"
)

# 生成签名
request_body = '{"departments": [...]}'
signature = ASign.get_hex_signature(credential, request_body)

# 设置请求头
headers = {
    "Content-Type": "application/json",
    "A-Signature": signature
}
```

**SDK 安装：**
```bash
# 从 aflow 获取 SDK 源码
cd aflow-client-python
pip install -e .
```

#### 签名算法说明（自行实现参考）

**签名算法：** MD5（注意：实际使用的是 MD5，不是 HMAC-SHA256）

**签名生成步骤：**

1. 构造字符串：`enterpriseCode + appSecret + requestBody + timestamp`
2. 第一次 MD5：`MD5(enterpriseCode + appSecret + requestBody + timestamp)`
3. 第二次 MD5：`MD5(第一次MD5的结果)`
4. 构造 ASignature 对象：`{enterpriseCode, appId, timestamp, cipher}`
5. 序列化为 JSON，再转换为十六进制字符串

> **详细签名算法实现请参考：** [技术原理文档 - 签名算法](technical-details.md#签名算法)（包含签名生成和验证的详细实现）

### 1.2 通用请求头

```
Content-Type: application/json
A-Signature: {签名十六进制字符串}
```

### 1.3 通用响应格式

**成功响应：**
```json
{
  "status": 0,
  "msg": "success",
  "data": {
    // 响应数据
  }
}
```

**错误响应：**
```json
{
  "status": -4002,
  "msg": "请求参数错误",
  "data": {
    "errorCode": "INVALID_PARAMETER",
    "errorMessage": "部门ID不能为空"
  }
}
```

> **注意：** 状态码说明：
> - `status = 0`：成功（`StatusCodeEnum.SUCCESS`）
> - `status < 0`：错误（如 `-1024` 表示通用错误，`-4002` 表示参数错误）

### 1.4 错误码说明

> **状态码规则：** 使用 `StatusCodeEnum` 枚举，成功为 `0`，错误为负数。

| 状态码 | 错误信息 | 说明 |
|--------|---------|------|
| 0 | SUCCESS | 成功 |
| -4002 | PARAM_ERROR | 参数错误 |
| -4001 | NOT_SUPPORT | 操作不支持 |
| -1024 | ERROR | 通用错误 |

---

## 2. 基础数据同步方案

> **重要提示**：基础数据同步有两种方案可选，详见 [主方案文档 - 基础数据同步方案](erp-wms-integration-design.md#21-基础数据同步方案)  
> **技术原理和时序图：** 参考 [技术原理文档 - 基础数据同步方案](technical-details.md#基础数据同步方案)（包含部门数据同步时序图、用户绑定流程等）

### 2.1 方案一：ERP主动推送（需要以下接口）

如果选择方案一，aiflow需要提供以下接口供ERP推送数据。

#### 2.1.1 部门数据同步接口

**接口地址：** `POST /aflow/api/sys/sync/department`

> **重要提示：** 接口传递数据必须按层级顺序传递（一级、二级、三级...一直到最低级），如果先同步下级部门数据时，aiflow上级部门没有生成，会保存出错。

**请求体：**
```json
{
  "departments": [
    {
      "deptId": "部门ID（外部系统）",
      "deptName": "部门名称",
      "parentId": "父部门ID（外部系统，根部门为空）",
      "deptCode": "部门编码（可选）",
      "orderNum": 100,
      "status": 1
    }
  ]
}
```

**响应体：**
```json
{
  "status": 0,
  "msg": "success",
  "data": {
    "successCount": 10,
    "failCount": 0,
    "failDetails": []
  }
}
```

#### 2.1.2 用户数据同步接口

**接口地址：** `POST /aflow/api/sys/sync/user`

> **重要提示：** 接口传递数据必须按层级顺序传递（一级、二级、三级...一直到最低级），如果先同步下级员工数据时，aiflow上级没有生成，会保存出错。

**请求体：**
```json
{
  "users": [
    {
      "userId": "用户ID（外部系统）",
      "userName": "用户名",
      "realName": "真实姓名",
      "email": "邮箱",
      "mobile": "手机号",
      "deptId": "部门ID（外部系统）",
      "personnelType": 1,
      "directSupervisor": "直接上级用户ID（外部系统，可选，最大Boss可为空）",
      "status": 1
    }
  ]
}
```

**请求参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| users | Array | 是 | 用户列表（必须按部门层级顺序传递） |
| users[].userId | String | 是 | 用户ID（外部系统） |
| users[].userName | String | 是 | 用户名 |
| users[].realName | String | 否 | 真实姓名 |
| users[].email | String | 否 | 邮箱 |
| users[].mobile | String | 否 | 手机号 |
| users[].deptId | String | 是 | 部门ID（外部系统） |
| users[].personnelType | Integer | 否 | 人员类型：1-正式，2-实习，3-外包，4-劳务，5-顾问（默认1） |
| users[].directSupervisor | String | 否 | 直接上级用户ID（外部系统，最大Boss可为空） |
| users[].status | Integer | 否 | 状态：1-启用，0-禁用（默认1） |


**响应体：**
```json
{
  "status": 0,
  "msg": "success",
  "data": {
    "successCount": 10,
    "failCount": 0,
    "failDetails": []
  }
}
```

> **技术原理：** 参考 [技术原理文档 - 用户数据同步接口](technical-details.md#用户数据同步接口)（包含用户数据同步时序图和业务逻辑）

### 2.2 方案二：aiflow主动同步钉钉（推荐）

如果选择方案二（推荐），aiflow将使用现有的钉钉同步机制，**无需开发推送接口**。ERP也无需开发推送逻辑。

**方案二要求：**
- Odoo的OAuth2 userinfo接口必须返回手机号/邮箱和部门ID（详见 [OAuth2 UserInfo接口](#321-用户信息端点user-info-endpoint)）
- aiflow将在SSO登录时自动绑定ERP用户编码和部门编码

> **技术原理：** 参考 [技术原理文档 - 方案二：aiflow主动同步钉钉方案](technical-details.md#方案二aiflow主动同步钉钉方案推荐)（包含钉钉同步时序图、SSO登录用户绑定逻辑和实现要点）

---

## 3. SSO单点登录对接方案

> **技术原理和时序图：** 参考 [技术原理文档 - SSO单点登录对接方案](technical-details.md#sso单点登录对接方案oauth-20)（包含OAuth2原理图、接口调用时序图、Token刷新时序图、aiflow系统改动方案等）

### 3.1 OAuth2流程说明

采用标准的OAuth 2.0授权码（Authorization Code）流程。

**流程步骤：**
1. 用户访问aiflow资源，未登录时检查SSO配置
2. 如果启用SSO，重定向到客户登录页（Odoo授权页）
3. 用户在Odoo登录并授权
4. Odoo重定向回aiflow，携带授权码（code）
5. aiflow使用授权码换取access_token
6. aiflow使用access_token获取用户信息
7. aiflow创建/更新用户并创建会话（JWT Token）

### 3.2 SSO配置说明

aiflow支持私有化部署，通过配置控制是否启用SSO登录。

**配置文件：** `application-prod.yml`、`application-dev.yml`、`application-local.yml`

**配置项：**
```yaml
custom:
  sso:
    enabled: true                    # 是否启用SSO登录（默认false）
    redirectUri: "https://odoo.example.com/login"  # 客户登录页地址（启用SSO时必填）
```

**配置说明：**
- `custom.sso.enabled`：是否启用SSO登录
  - `true`：启用SSO，401时跳转到客户登录页
  - `false`：使用aiflow默认登录页（`https://beta.aiflow.fan/login`）
- `custom.sso.redirectUri`：客户登录页地址（Odoo授权端点地址）
  - 启用SSO时必填
  - 格式：`https://odoo.example.com/login`

**测试支持：**
- 为了测试SSO登录功能，aiflow项目工程中需要模拟实现一个Odoo登录页
- 模拟登录页需要实现OAuth2授权端点功能，完成整个功能的闭环测试

### 3.3 Odoo需要提供的OAuth2接口

#### 3.3.1 授权端点（Authorization Endpoint）

**接口地址：** `GET /odoo/api/oauth2/authorize`

**请求参数：**
- `client_id`: aiflow的应用ID
- `redirect_uri`: 授权后的回调地址
- `response_type`: 固定值 `code`
- `scope`: 授权范围
- `state`: 随机字符串，用于防止CSRF攻击

**响应：**
- 成功：重定向到 `redirect_uri?code=authorization_code&state=random_state_string`
- 失败：重定向到 `redirect_uri?error=error_code&error_description=description`

#### 3.3.2 令牌端点（Token Endpoint）

**接口地址：** `POST /odoo/api/oauth2/token`

**请求头：**
```
Content-Type: application/x-www-form-urlencoded
Authorization: Basic {base64(client_id:client_secret)}
```

**请求参数（form-data）：**
- `grant_type`: `authorization_code` 或 `refresh_token`
- `code`: 授权码（当grant_type=authorization_code时）
- `refresh_token`: 刷新令牌（当grant_type=refresh_token时）
- `redirect_uri`: 必须与授权请求中的redirect_uri一致

**响应：**
```json
{
  "access_token": "访问令牌",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "刷新令牌",
  "scope": "read write"
}
```

> **Odoo接口标准：** 参考 [技术原理文档 - Odoo需要提供的接口标准](technical-details.md#odoo需要提供的接口标准)（详细的接口要求和实现说明）

#### 3.3.3 用户信息端点（User Info Endpoint）

**接口地址：** `GET /odoo/api/oauth2/userinfo`

> **重要提示**：如果选择**方案二：aiflow主动同步钉钉**，此接口必须返回`phone`（手机号）或`email`（邮箱）以及`deptId`（部门ID），用于用户绑定。

**请求头：**
```
Authorization: Bearer {access_token}
```

**响应：**
```json
{
  "userId": "用户ID（ERP用户编码）",
  "username": "用户名",
  "name": "姓名",
  "phone": "手机号（必填，至少phone或email选一个）",
  "email": "邮箱（可选）",
  "deptId": "部门ID（必填，用于建立部门映射）",
  "deptName": "部门名称"
}
```

**方案二特殊要求：**
- `phone`和`email`至少返回一个（用于匹配钉钉用户）
- `deptId`必须返回（用于建立部门映射关系）
- `userId`必须返回（用于建立用户映射关系）

### 3.4 aiflow需要提供的接口

#### 3.4.1 OAuth2回调接口

**接口地址：** `GET /aflow/api/oauth2/callback`

**请求参数：**
- `code`: 授权码（Odoo返回）
- `state`: 状态参数（用于CSRF防护）

**响应：**
- 成功：重定向到原始请求页面
- 失败：重定向到错误页面

#### 3.4.2 Token刷新接口

**接口地址：** `POST /aflow/api/oauth2/refresh`

**请求体：**
```json
{
  "refreshToken": "刷新令牌"
}
```

**响应：**
```json
{
  "status": 0,
  "msg": "success",
  "data": {
    "accessToken": "新的访问令牌",
    "refreshToken": "新的刷新令牌",
    "expiresIn": 3600
  }
}
```

> **aiflow系统改动：** 参考 [技术原理文档 - aiflow系统改动方案](technical-details.md#aiflow系统改动方案)（包含配置管理、认证拦截器改造、OAuth2回调处理器、Token刷新机制等）

---

## 4. 任务中心简易集成方案

> **技术原理和时序图：** 参考 [技术原理文档 - 任务中心统一简易集成方案](technical-details.md#任务中心统一简易集成方案)（包含流程定义改造、第三方流程定义创建、任务同步时序图等）

### 4.1 创建三方流程定义接口

**接口地址：** `POST /aflow/api/flow/create_third_party`

**请求体：**
```json
{
  "flowCode": "三方业务流程编码",
  "title": "流程标题",
  "initiateUrl": {
    "h5Url": "H5发起页地址",
    "webUrl": "PC发起页地址"
  },
  "detailUrl": {
    "h5Url": "H5详情页地址",
    "webUrl": "PC详情页地址"
  },
  "categoryId": "所属分组",
  "managerUserCode": "流程负责人用户编码",
  "operationUserCode": "运营负责人用户编码",
  "configUserCode": "配置负责人用户编码",
  "createBy": "创建人用户编码",
  "allowedApplyTerminals": ["pc", "mobile"],
  "allowedApplyRule": {
    "allowedApplyType": "all",
    "userCodes": [],
    "deptCodes": [],
    "userGroupCodes": []
  },
  "allowedManageRule": {
    "allowedApplyType": "all",
    "userCodes": [],
    "deptCodes": [],
    "userGroupCodes": []
  }
}
```

**请求参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| flowCode | String | 是 | 三方业务流程编码 |
| businessKey | String | 是 | 业务单据号 |
| title | String | 是 | 流程标题 |
| initiateUrl | Object | 是 | 流程发起页地址 |
| initiateUrl.h5Url | String | 是 | H5发起页地址 |
| initiateUrl.webUrl | String | 是 | PC发起页地址 |
| detailUrl | Object | 是 | 订单详情页地址 |
| detailUrl.h5Url | String | 是 | H5详情页地址 |
| detailUrl.webUrl | String | 是 | PC详情页地址 |
| groupId | String | 是 | 所属分组ID |
| managerUserCode | String | 是 | 流程负责人用户编码 |
| operationUserCode | String | 是 | 运营负责人用户编码 |
| configUserCode | String | 是 | 配置负责人用户编码 |
| createBy | String | 是 | 创建人用户编码 |
| allowedApplyTerminals | Array | 否 | 允许发起终端：["pc", "mobile"]（默认全部），对应 `FlowDetail.allowedApplyTerminals` |
| allowedApplyRule | Object | 否 | 允许发起规则（默认全部），对应 `FlowDetail.allowedApplyRule` |
| allowedApplyRule.allowedApplyType | String | 否 | 允许发起类型：all-全部，specify-指定（默认all） |
| allowedApplyRule.userCodes | Array | 否 | 允许发起的用户编码列表 |
| allowedApplyRule.deptCodes | Array | 否 | 允许发起的部门编码列表 |
| allowedApplyRule.userGroupCodes | Array | 否 | 允许发起的用户组编码列表 |
| allowedManageRule | Object | 否 | 允许查看订单/管理流程规则（默认全部），对应 `FlowDetail.allowedManageRule` |
| allowedManageRule.allowedApplyType | String | 否 | 允许管理类型：all-全部，specify-指定（默认all） |
| allowedManageRule.userCodes | Array | 否 | 允许管理的用户编码列表 |
| allowedManageRule.deptCodes | Array | 否 | 允许管理的部门编码列表 |
| allowedManageRule.userGroupCodes | Array | 否 | 允许管理的用户组编码列表 |

**响应体：**
```json
{
  "status": 0,
  "msg": "success",
  "data": {
    "flowCode": "FLOW001",
    "flowVersion": 1
  }
}
```

### 4.2 三方流程定义上线接口

**接口地址：** `POST /aflow/api/flow/online_third_party`

**请求体：**
```json
{
  "flowCode": "流程编码",
  "flowVersion": 1,
  "updateDesc": "更新说明"
}
```

**响应体：**
```json
{
  "status": 0,
  "msg": "success",
  "data": {
    "success": true
  }
}
```

> **说明：** 
> - 三方流程支持按版本管理，每个版本需要上线操作
> - 调用 `com.aflow.base.controller.FlowDefinitionController#online` 操作上线
> - 需要实现三方流程专用的校验器（`FlowCheckProcessor`），只校验接口中传递的数据，不需要正常流程那么复杂的校验

### 4.3 任务同步接口

> **说明：** 三方流程实际是在三方引擎系统中处理，业务引擎系统实时将整个订单的结果同步过来，包括待办任务和已完成任务。

**接口地址：** `POST /aflow/api/order/sync/task`

**请求体：**
```json
{
  "orderId": 123456,
  "orderStatus": "ing",
  "orderResult": "ing",
  "initiator": "发起人编号",
  "version": 1,
  "parentOrderId": 123450,
  "parentTaskOrderId": "父任务订单号",
  "businessKey": "业务编码",
  "createTime": "2025-01-24 10:00:00",
  "updateTime": "2025-01-24 15:30:00",
  "ccUsers": [
    {
      "userCode": "抄送人编码",
      "ccTime": "2025-01-24 10:00:00"
    }
  ],
  "tasks": [
    {
      "taskId": "任务ID",
      "taskName": "任务名称",
      "assigneeUserCode": ["处理人编码1", "处理人编码2"],
      "taskStatus": "ing",
      "taskResult": "accept",
      "deadLine": "2025-01-25 18:00:00",
      "nodeType": "audit",
      "handleTime": "2025-01-24 15:30:00",
      "showPc": true,
      "showMobile": true
    }
  ]
}
```

**请求参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| orderId | Long | 是 | 流程订单号 |
| orderStatus | String | 是 | 订单状态：new-新建，ing-处理中，over-完成（默认ing） |
| orderResult | String | 是 | 订单结果：ing-处理中，pass-已完成，reject-已拒绝，revoke-已撤销，delete-已删除 |
| initiator | String | 是 | 发起人编号 |
| version | Integer | 否 | 订单版本（有就传，没有可以不传） |
| parentOrderId | Long | 否 | 父订单号（有就传，没有可以不传） |
| parentTaskOrderId | String | 否 | 父任务订单号（有就传，没有可以不传） |
| businessKey | String | 是 | 业务编码，用于记录三方系统跟流程引擎对接的唯一映射Key |
| createTime | String | 是 | 流程订单创建时间（格式：yyyy-MM-dd HH:mm:ss） |
| updateTime | String | 是 | 流程订单更新时间（格式：yyyy-MM-dd HH:mm:ss） |
| ccUsers | Array | 否 | 抄送人列表（非必传，有就传，没有就不传） |
| ccUsers[].userCode | String | 是 | 抄送人编码 |
| ccUsers[].ccTime | String | 是 | 抄送时间（格式：yyyy-MM-dd HH:mm:ss） |
| tasks | Array | 是 | 任务列表（包含待办和已完成任务） |
| tasks[].taskId | String | 是 | 任务ID |
| tasks[].taskName | String | 是 | 任务名称 |
| tasks[].assigneeUserCode | Array | 是 | 处理人编码列表（可能有多个） |
| tasks[].taskStatus | String | 是 | 任务状态：new-新建，ing-处理中，over-完成（默认ing） |
| tasks[].taskResult | String | 是 | 任务处理结果：new-新建，accept-领取，pass-通过，reject-拒绝，revoke-撤销，rebut-驳回 |
| tasks[].deadLine | String | 是 | 处理截止时间（格式：yyyy-MM-dd HH:mm:ss） |
| tasks[].nodeType | String | 是 | 节点类型：handle-执行，audit-审批，notify-知悉（默认audit） |
| tasks[].handleTime | String | 否 | 处理时间（格式：yyyy-MM-dd HH:mm:ss，处理完成后必填） |
| tasks[].showPc | Boolean | 是 | PC是否展示 |
| tasks[].showMobile | Boolean | 是 | 手机是否展示 |

**状态枚举说明：**

- **订单状态（OrderStatus）**：`new`-新建，`ing`-处理中，`over`-已完成
- **订单结果（OrderResult）**：`ing`-处理中，`pass`-已完成，`reject`-已拒绝，`revoke`-已撤销，`delete`-已删除
- **任务状态（TaskStatus）**：`new`-新建，`ing`-处理中，`over`-完成
- **任务结果（TaskResult）**：`new`-新建，`accept`-领取，`pass`-通过，`reject`-拒绝，`revoke`-撤销，`rebut`-驳回
- **节点类型（NodeType）**：`handle`-执行，`audit`-审批，`notify`-知悉

**响应体：**
```json
{
  "status": 0,
  "msg": "success",
  "data": {
    "success": true
  }
}
```

### 4.3.1 新任务通知功能

**功能说明：**
- 当同步的任务是新增任务（`taskStatus` 为 `new` 或 `ing` 状态）时，aiflow会自动发送钉钉push消息通知处理人
- 通知消息会发送给任务的所有处理人（`assigneeUserCode` 列表中的所有用户）

**触发条件：**
- 任务为新增任务（数据库中不存在该任务）
- 任务状态为 `new`（新建）或 `ing`（处理中）

**实现方式：**
- 使用 `com.aflow.sys.link.LinkMsgService#sendFlowMsg` 方法发送通知
- 消息内容包含：任务名称、流程标题、处理按钮等
- 支持钉钉、飞书、企业微信等多种消息渠道

**消息内容示例：**
- 标题：新任务提醒
- 内容：您有新的待处理任务【任务名称】截止时间:2025-01-25 18:00:00
- 按钮：处理（点击跳转到任务详情页）

> **技术原理：** 参考 [技术原理文档 - 任务同步接口](technical-details.md#任务同步接口)（包含任务同步时序图和业务逻辑）

---

## 5. 全面流程引擎集成方案

> **技术原理和时序图：** 参考 [技术原理文档 - 全面使用流程中心的表单与流程引擎方案](technical-details.md#全面使用流程中心的表单与流程引擎方案)（包含流程定义管理、表单同步方案、ServiceTask数据推送时序图等）

### 5.1 流程定义管理接口

#### 5.1.1 创建流程定义接口

**接口地址：** `POST /aflow/api/flow/create`

**请求体：**
```json
{
  "flowCode": "流程编码",
  "flowName": "流程名称",
  "formSchema": {
    // 表单Schema定义
  },
  "bpmnXml": "BPMN XML内容"
}
```

**响应体：**
```json
{
  "status": 0,
  "msg": "success",
  "data": {
    "flowCode": "FLOW001",
    "flowVersion": 1
  }
}
```

#### 5.1.2 更新流程定义接口

**接口地址：** `PUT /aflow/api/flow/update`

#### 5.1.3 发布流程接口

**接口地址：** `POST /aflow/api/flow/publish`

**请求体：**
```json
{
  "flowCode": "流程编码",
  "flowVersion": 1,
  "updateDesc": "更新说明"
}
```

**响应体：**
```json
{
  "status": 0,
  "msg": "success",
  "data": {
    "success": true
  }
}
```

### 5.2 表单同步接口

**接口地址：** `POST /aflow/api/form/sync`

**请求体：**
```json
{
  "formCode": "表单编码（Odoo）",
  "formName": "表单名称",
  "formSchema": {
    "components": [
      {
        "type": "input",
        "name": "field1",
        "label": "字段1",
        "required": true
      }
    ]
  },
  "syncType": "CREATE|UPDATE|DELETE"
}
```

**响应体：**
```json
{
  "status": 0,
  "msg": "success",
  "data": {
    "success": true
  }
}
```

> **表单同步方案：** 参考 [技术原理文档 - 表单同步方案](technical-details.md#表单同步方案)（包含Odoo表单自动同步到aiflow的机制）

### 5.3 审批日志接口

**接口地址：** `POST /aflow/api/order/approval/log`

**请求体：**
```json
{
  "orderId": 123456,
  "businessKey": "业务单据号",
  "logs": [
    {
      "taskId": "任务ID",
      "taskName": "任务名称",
      "handlerUserCode": "处理人编码",
      "handleResult": "APPROVED|REJECTED",
      "handleComment": "审批意见",
      "handleTime": "2025-01-24 15:30:00"
    }
  ]
}
```

**响应体：**
```json
{
  "status": 0,
  "msg": "success",
  "data": {
    "success": true
  }
}
```

### 5.4 Odoo需要提供的接口

#### 5.4.1 接收流程数据接口

**接口地址：** `POST /odoo/api/workflow/receive`

**请求体：**
```json
{
  "orderId": 123456,
  "businessKey": "业务单据号",
  "taskId": "任务ID",
  "taskName": "任务名称",
  "action": "START|COMPLETE|REJECT|CANCEL",
  "formData": {
    // 表单数据
  },
  "processVariables": {
    // 流程变量
  },
  "handlerUserCode": "处理人编码",
  "handleTime": "2025-01-24 15:30:00"
}
```

#### 5.4.2 接收审批日志接口（可选）

**接口地址：** `POST /odoo/api/workflow/approval/log`

**请求体：**
```json
{
  "orderId": 123456,
  "businessKey": "业务单据号",
  "logs": [
    {
      "taskId": "任务ID",
      "taskName": "任务名称",
      "handlerUserCode": "处理人编码",
      "handleResult": "APPROVED|REJECTED|TRANSFERRED",
      "handleComment": "审批意见",
      "handleTime": "2025-01-24 15:30:00",
      "formData": {
        // 处理时的表单数据
      }
    }
  ]
}
```

**响应体：**
```json
{
  "status": 0,
  "msg": "success",
  "data": {
    "success": true
  }
}
```

---

# 第二部分：双方任务清单

## 阶段一：基础数据同步

| 任务项 | aiflow方 | Odoo方 | 说明 |
|--------|---------|--------|------|
| **方案一：ERP主动推送** |
| 部门同步接口 | ✅ 开发 `POST /aflow/api/sys/sync/department` | ✅ 开发部门数据推送逻辑 | 方案一需要 |
| 用户同步接口 | ✅ 开发 `POST /aflow/api/sys/sync/user`（支持personnelType、directSupervisor字段，按层级顺序传递） | ✅ 开发用户数据推送逻辑（按层级顺序推送） | 方案一需要 |
| 签名算法 | ✅ 提供 Python SDK | ✅ 集成 Python SDK | 使用统一SDK |
| 数据映射 | ✅ 实现部门/用户映射逻辑 | - | - |
| **方案二：aiflow主动同步钉钉（推荐）** |
| 钉钉同步 | ✅ 配置钉钉同步定时任务 | - | 方案二使用 |
| SSO用户绑定 | ✅ 实现SSO登录时用户绑定 | ✅ OAuth2 userinfo返回phone/deptId | 方案二需要 |

## 阶段二：SSO单点登录对接

| 任务项 | aiflow方 | Odoo方 | 说明 |
|--------|---------|--------|------|
| SSO配置 | ✅ 增加 `custom.sso.enabled` 和 `custom.sso.redirectUri` 配置项 | - | aiflow提供 |
| OAuth2授权端点 | - | ✅ 开发 `GET /odoo/api/oauth2/authorize` | Odoo提供 |
| OAuth2令牌端点 | - | ✅ 开发 `POST /odoo/api/oauth2/token` | Odoo提供 |
| OAuth2用户信息端点 | - | ✅ 开发 `GET /odoo/api/oauth2/userinfo` | Odoo提供，需返回phone/deptId |
| OAuth2回调接口 | ✅ 开发 `GET /aflow/api/oauth2/callback` | - | aiflow提供 |
| Token刷新接口 | ✅ 开发 `POST /aflow/api/oauth2/refresh` | - | aiflow提供 |
| 认证拦截器 | ✅ 改造UserInterceptor，根据SSO配置识别401并重定向 | - | - |
| Token管理 | ✅ 实现Token刷新机制 | - | - |
| 模拟登录页 | ✅ 实现模拟Odoo登录页用于测试 | - | 测试支持 |
| 安全机制 | ✅ 实现state验证、CSRF防护 | ✅ 实现OAuth2安全机制 | 双方 |

## 阶段三：任务中心简易集成

| 任务项 | aiflow方 | Odoo方 | 说明 |
|--------|---------|--------|------|
| 创建三方流程定义接口 | ✅ 开发 `POST /aflow/api/flow/create_third_party`（包含分组、负责人等字段） | - | aiflow提供 |
| 三方流程定义上线接口 | ✅ 开发 `POST /aflow/api/flow/online_third_party`（实现三方流程校验器） | - | aiflow提供 |
| 三方流程校验器 | ✅ 实现三方流程专用的 `FlowCheckProcessor` | - | aiflow提供 |
| 任务同步接口 | ✅ 开发 `POST /aflow/api/order/sync/task`（支持orderId、orderStatus、orderResult、initiator、version、parentOrderId、parentTaskOrderId、businessKey、createTime、updateTime、ccUsers、taskStatus、taskResult、assigneeUserCode数组、deadLine、nodeType、handleTime、showPc、showMobile等字段） | ✅ 开发任务同步逻辑（实时同步整个订单的待办和已完成任务） | - |
| 前端集成 | ✅ 改造待办列表，支持跳转到外部系统 | ✅ 开发流程发起页和详情页（H5+PC） | - |
| 数据库扩展 | ✅ 扩展流程定义表，支持第三方引擎 | - | - |

## 阶段四：全面流程引擎集成

| 任务项 | aiflow方 | Odoo方 | 说明 |
|--------|---------|--------|------|
| 流程定义管理 | ✅ 开发流程创建/更新/发布接口 | ✅ 开发流程定义对接逻辑 | - |
| 表单同步接口 | ✅ 开发 `POST /aflow/api/form/sync` | ✅ 开发表单Schema转换工具 | - |
| ServiceTask实现 | ✅ 实现ServiceTask数据推送 | ✅ 开发 `POST /odoo/api/workflow/receive` | - |
| 审批日志接口 | ✅ 开发 `POST /aflow/api/order/approval/log` | ✅ 开发 `POST /odoo/api/workflow/approval/log` | 可选 |

## 通用任务

| 任务项 | aiflow方 | Odoo方 | 说明 |
|--------|---------|--------|------|
| 接口文档 | ✅ 编写所有接口文档 | ✅ 编写Odoo接口文档 | 双方 |
| 测试环境 | ✅ 搭建测试环境 | ✅ 搭建测试环境 | 双方 |
| 接口联调 | ✅ 参与接口联调 | ✅ 参与接口联调 | 双方 |
| 监控日志 | ✅ 实现接口监控和日志 | ✅ 实现接口监控和日志 | 双方 |
| 安全加固 | ✅ 实现接口限流、IP白名单 | ✅ 实现安全机制 | 双方 |

---

## 接口清单汇总

### aiflow方需要提供的接口（12个）

| 序号 | 接口路径 | 方法 | 功能 | 阶段 |
|------|---------|------|------|------|
| 1 | `/aflow/api/sys/sync/department` | POST | 部门同步 | 阶段一（方案一） |
| 2 | `/aflow/api/sys/sync/user` | POST | 用户同步（支持personnelType、directSupervisor，按层级顺序） | 阶段一（方案一） |
| 3 | `/aflow/api/oauth2/callback` | GET | OAuth2回调 | 阶段二 |
| 4 | `/aflow/api/oauth2/refresh` | POST | Token刷新 | 阶段二 |
| 5 | `/aflow/api/flow/create_third_party` | POST | 创建三方流程定义 | 阶段三 |
| 6 | `/aflow/api/flow/online_third_party` | POST | 三方流程定义上线 | 阶段三 |
| 7 | `/aflow/api/order/sync/task` | POST | 任务同步（支持完整订单状态和任务信息） | 阶段三 |
| 8 | `/aflow/api/flow/create` | POST | 创建流程定义 | 阶段四 |
| 9 | `/aflow/api/flow/update` | PUT | 更新流程定义 | 阶段四 |
| 10 | `/aflow/api/flow/publish` | POST | 发布流程 | 阶段四 |
| 11 | `/aflow/api/form/sync` | POST | 表单同步 | 阶段四 |
| 12 | `/aflow/api/order/approval/log` | POST | 审批日志 | 阶段四 |

### Odoo方需要提供的接口（6个）

| 序号 | 接口路径 | 方法 | 功能 | 阶段 |
|------|---------|------|------|------|
| 1 | `/odoo/api/oauth2/authorize` | GET | OAuth2授权端点 | 阶段二 |
| 2 | `/odoo/api/oauth2/token` | POST | OAuth2令牌端点 | 阶段二 |
| 3 | `/odoo/api/oauth2/userinfo` | GET | OAuth2用户信息 | 阶段二 |
| 4 | `/odoo/api/form/list` | GET | 查询表单列表（可选） | 阶段四 |
| 5 | `/odoo/api/workflow/receive` | POST | 接收流程数据 | 阶段四 |
| 6 | `/odoo/api/workflow/approval/log` | POST | 接收审批日志（可选） | 阶段四 |

---

## 相关文档

- **主方案文档：** [ERP系统对接aiflow流程中心设计方案](erp-wms-integration-design.md)
- **技术原理文档：** [技术原理与接口设计文档](technical-details.md)（包含详细的时序图、数据库设计和实现细节）
  - [签名算法实现](technical-details.md#签名算法实现)
  - [数据库设计](technical-details.md#数据库设计)
  - [接口实现示例](technical-details.md#接口实现示例)
  - [错误处理](technical-details.md#错误处理)
