# ERP系统对接aiflow接口文档

本文档提供ERP系统对接aiflow流程中心的完整接口文档，包含aiflow提供的接口和Odoo ERP需要提供的接口，以及详细的调用示例。

> **返回主方案文档：** [ERP系统对接aiflow流程中心设计方案](erp-wms-integration-design.md)  
> **技术详细设计文档：** [技术详细设计文档](integration-technical-design.md)

---

## 目录

- [第一部分：aiflow提供的接口](#第一部分aiflow提供的接口)
  - [1. 通用说明](#1-通用说明)
  - [2. 基础数据同步接口](#2-基础数据同步接口)
  - [3. SSO单点登录接口](#3-sso单点登录接口)
  - [4. 任务中心简易集成接口](#4-任务中心简易集成接口)
  - [5. 全面流程引擎集成接口](#5-全面流程引擎集成接口)
- [第二部分：Odoo ERP需要提供的接口](#第二部分odoo-erp需要提供的接口)
  - [1. OAuth2接口](#1-oauth2接口)
  - [2. 流程数据接收接口](#2-流程数据接收接口)

---

# 第一部分：aiflow提供的接口

## 1. 通用说明

### 1.1 签名机制

所有aiflow提供的接口都需要使用签名验证（`A-Signature` Header）。

**推荐方式：** 使用 aflow 提供的 **Python SDK**（`aflow-client-python`）

**Python SDK调用示例：**

```python
from auth import ASign, Credential
import requests
import json

# 1. 创建认证凭证
credential = Credential(
    enterprise_code="COMPANY001",
    app_id="APP001",
    app_secret="your_app_secret"
)

# 2. 准备请求体
request_data = {
    "departments": [
        {
            "deptId": "DEPT001",
            "deptName": "技术部",
            "parentId": "",
            "orderNum": 100,
            "status": 1
        }
    ]
}
request_body = json.dumps(request_data, ensure_ascii=False)

# 3. 生成签名
signature = ASign.get_hex_signature(credential, request_body)

# 4. 设置请求头
headers = {
    "Content-Type": "application/json",
    "A-Signature": signature
}

# 5. 发送请求
response = requests.post(
    "https://aiflow.example.com/aflow/api/sys/sync/department",
    headers=headers,
    data=request_body.encode('utf-8')
)

print(response.json())
```

**SDK安装：**
```bash
cd aflow-client-python
pip install -e .
```

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

**状态码说明：**
- `0`: 成功
- `-4002`: 参数错误
- `-4001`: 操作不支持
- `-1024`: 通用错误

---

## 2. 基础数据同步接口

### 2.1 部门数据同步接口

**接口地址：** `POST /aflow/api/sys/sync/department`

**接口说明：** 同步部门数据到aiflow系统

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

**请求参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| departments | Array | 是 | 部门列表（必须按层级顺序传递） |
| departments[].deptId | String | 是 | 部门ID（外部系统） |
| departments[].deptName | String | 是 | 部门名称 |
| departments[].parentId | String | 否 | 父部门ID（外部系统，根部门为空） |
| departments[].deptCode | String | 否 | 部门编码（可选） |
| departments[].orderNum | Integer | 否 | 排序号（默认100） |
| departments[].status | Integer | 否 | 状态：1-启用，0-禁用（默认1） |

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

**Python调用示例：**

```python
from auth import ASign, Credential
import requests
import json

# 配置信息
credential = Credential(
    enterprise_code="COMPANY001",
    app_id="APP001",
    app_secret="your_app_secret"
)

# 准备请求数据（按层级顺序）
request_data = {
    "departments": [
        {
            "deptId": "DEPT001",
            "deptName": "技术部",
            "parentId": "",
            "orderNum": 100,
            "status": 1
        },
        {
            "deptId": "DEPT002",
            "deptName": "开发组",
            "parentId": "DEPT001",
            "orderNum": 101,
            "status": 1
        }
    ]
}

request_body = json.dumps(request_data, ensure_ascii=False)
signature = ASign.get_hex_signature(credential, request_body)

headers = {
    "Content-Type": "application/json",
    "A-Signature": signature
}

response = requests.post(
    "https://aiflow.example.com/aflow/api/sys/sync/department",
    headers=headers,
    data=request_body.encode('utf-8')
)

result = response.json()
print(f"成功: {result['data']['successCount']}, 失败: {result['data']['failCount']}")
```

### 2.2 用户数据同步接口

**接口地址：** `POST /aflow/api/sys/sync/user`

**接口说明：** 同步用户数据到aiflow系统

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

**人员类型说明：**
- `1`: 正式员工
- `2`: 实习员工
- `3`: 外包员工
- `4`: 劳务员工
- `5`: 顾问

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

**Python调用示例：**

```python
from auth import ASign, Credential
import requests
import json

credential = Credential(
    enterprise_code="COMPANY001",
    app_id="APP001",
    app_secret="your_app_secret"
)

# 按层级顺序准备用户数据
request_data = {
    "users": [
        {
            "userId": "USER001",
            "userName": "张三",
            "realName": "张三",
            "email": "zhangsan@example.com",
            "mobile": "13800138000",
            "deptId": "DEPT001",
            "personnelType": 1,
            "directSupervisor": "",
            "status": 1
        },
        {
            "userId": "USER002",
            "userName": "李四",
            "realName": "李四",
            "email": "lisi@example.com",
            "mobile": "13800138001",
            "deptId": "DEPT002",
            "personnelType": 1,
            "directSupervisor": "USER001",
            "status": 1
        }
    ]
}

request_body = json.dumps(request_data, ensure_ascii=False)
signature = ASign.get_hex_signature(credential, request_body)

headers = {
    "Content-Type": "application/json",
    "A-Signature": signature
}

response = requests.post(
    "https://aiflow.example.com/aflow/api/sys/sync/user",
    headers=headers,
    data=request_body.encode('utf-8')
)

result = response.json()
print(f"成功: {result['data']['successCount']}, 失败: {result['data']['failCount']}")
```

---

## 3. SSO单点登录接口

### 3.1 OAuth2回调接口

**接口地址：** `GET /aflow/api/oauth2/callback`

**接口说明：** OAuth2授权回调接口，用于接收Odoo返回的授权码

**请求参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| code | String | 是 | 授权码（Odoo返回） |
| state | String | 是 | 状态参数（用于CSRF防护） |

**响应：**
- 成功：重定向到原始请求页面
- 失败：重定向到错误页面

**调用示例：**

用户访问aiflow时，如果未登录且启用了SSO，会自动重定向到Odoo授权页：
```
https://odoo.example.com/odoo/api/oauth2/authorize?client_id=APP001&redirect_uri=https://aiflow.example.com/aflow/api/oauth2/callback&response_type=code&scope=basic&state=random_state_string
```

用户授权后，Odoo重定向回：
```
https://aiflow.example.com/aflow/api/oauth2/callback?code=authorization_code&state=random_state_string
```

### 3.2 Token刷新接口

**接口地址：** `POST /aflow/api/oauth2/refresh`

**接口说明：** 刷新访问令牌

**请求体：**
```json
{
  "refreshToken": "刷新令牌"
}
```

**响应体：**
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

**Python调用示例：**

```python
import requests
import json

request_data = {
    "refreshToken": "your_refresh_token"
}

response = requests.post(
    "https://aiflow.example.com/aflow/api/oauth2/refresh",
    headers={"Content-Type": "application/json"},
    json=request_data
)

result = response.json()
if result['status'] == 0:
    print(f"新Token: {result['data']['accessToken']}")
```

---

## 4. 任务中心简易集成接口

### 4.1 创建三方流程定义接口

**接口地址：** `POST /aflow/api/flow/create_third_party`

**接口说明：** 创建三方流程定义

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
  "groupId": "所属分组ID",
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
| allowedApplyTerminals | Array | 否 | 允许发起终端：["pc", "mobile"]（默认全部） |
| allowedApplyRule | Object | 否 | 允许发起规则（默认全部） |
| allowedManageRule | Object | 否 | 允许查看订单/管理流程规则（默认全部） |

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

**Python调用示例：**

```python
from auth import ASign, Credential
import requests
import json

credential = Credential(
    enterprise_code="COMPANY001",
    app_id="APP001",
    app_secret="your_app_secret"
)

request_data = {
    "flowCode": "SALES_ORDER",
    "title": "销售订单审批流程",
    "initiateUrl": {
        "h5Url": "https://odoo.example.com/h5/sales/apply",
        "webUrl": "https://odoo.example.com/web/sales/apply"
    },
    "detailUrl": {
        "h5Url": "https://odoo.example.com/h5/sales/detail",
        "webUrl": "https://odoo.example.com/web/sales/detail"
    },
    "groupId": "GROUP001",
    "managerUserCode": "USER001",
    "operationUserCode": "USER002",
    "configUserCode": "USER003",
    "createBy": "USER001",
    "allowedApplyTerminals": ["pc", "mobile"],
    "allowedApplyRule": {
        "allowedApplyType": "all"
    },
    "allowedManageRule": {
        "allowedApplyType": "all"
    }
}

request_body = json.dumps(request_data, ensure_ascii=False)
signature = ASign.get_hex_signature(credential, request_body)

headers = {
    "Content-Type": "application/json",
    "A-Signature": signature
}

response = requests.post(
    "https://aiflow.example.com/aflow/api/flow/create_third_party",
    headers=headers,
    data=request_body.encode('utf-8')
)

result = response.json()
if result['status'] == 0:
    print(f"流程编码: {result['data']['flowCode']}, 版本: {result['data']['flowVersion']}")
```

### 4.2 三方流程定义上线接口

**接口地址：** `POST /aflow/api/flow/online_third_party`

**接口说明：** 三方流程定义上线

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

**Python调用示例：**

```python
from auth import ASign, Credential
import requests
import json

credential = Credential(
    enterprise_code="COMPANY001",
    app_id="APP001",
    app_secret="your_app_secret"
)

request_data = {
    "flowCode": "SALES_ORDER",
    "flowVersion": 1,
    "updateDesc": "初始版本上线"
}

request_body = json.dumps(request_data, ensure_ascii=False)
signature = ASign.get_hex_signature(credential, request_body)

headers = {
    "Content-Type": "application/json",
    "A-Signature": signature
}

response = requests.post(
    "https://aiflow.example.com/aflow/api/flow/online_third_party",
    headers=headers,
    data=request_body.encode('utf-8')
)

result = response.json()
if result['status'] == 0:
    print("流程上线成功")
```

### 4.3 任务同步接口

**接口地址：** `POST /aflow/api/order/sync/task`

**接口说明：** 同步任务数据到aiflow系统。当同步的任务是新增任务（`taskStatus` 为 `new` 或 `ing` 状态）时，aiflow会自动发送钉钉push消息通知处理人。

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

**Python调用示例：**

```python
from auth import ASign, Credential
import requests
import json
from datetime import datetime

credential = Credential(
    enterprise_code="COMPANY001",
    app_id="APP001",
    app_secret="your_app_secret"
)

now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
deadline = (datetime.now().replace(hour=18, minute=0, second=0)).strftime("%Y-%m-%d %H:%M:%S")

request_data = {
    "orderId": 123456,
    "orderStatus": "ing",
    "orderResult": "ing",
    "initiator": "USER001",
    "version": 1,
    "businessKey": "SALES_ORDER_20250124001",
    "createTime": now,
    "updateTime": now,
    "ccUsers": [
        {
            "userCode": "USER003",
            "ccTime": now
        }
    ],
    "tasks": [
        {
            "taskId": "TASK001",
            "taskName": "部门经理审批",
            "assigneeUserCode": ["USER002"],
            "taskStatus": "new",
            "taskResult": "new",
            "deadLine": deadline,
            "nodeType": "audit",
            "showPc": True,
            "showMobile": True
        }
    ]
}

request_body = json.dumps(request_data, ensure_ascii=False)
signature = ASign.get_hex_signature(credential, request_body)

headers = {
    "Content-Type": "application/json",
    "A-Signature": signature
}

response = requests.post(
    "https://aiflow.example.com/aflow/api/order/sync/task",
    headers=headers,
    data=request_body.encode('utf-8')
)

result = response.json()
if result['status'] == 0:
    print("任务同步成功")
    # 注意：如果taskStatus为new或ing，aiflow会自动发送钉钉push消息通知处理人
```

---

## 5. 全面流程引擎集成接口

### 5.1 流程定义管理接口

#### 5.1.1 创建流程定义接口

**接口地址：** `POST /aflow/api/flow/create`

**接口说明：** 创建流程定义（包含表单Schema和BPMN XML）

**请求体：**
```json
{
  "flowCode": "流程编码",
  "flowName": "流程名称",
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
  "bpmnXml": "<?xml version=\"1.0\" encoding=\"UTF-8\"?>..."
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

**接口说明：** 更新流程定义

#### 5.1.3 发布流程接口

**接口地址：** `POST /aflow/api/flow/publish`

**接口说明：** 发布流程定义

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

**接口说明：** 同步表单定义到aiflow系统

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
  "syncType": "CREATE"
}
```

**请求参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| formCode | String | 是 | 表单编码（Odoo） |
| formName | String | 是 | 表单名称 |
| formSchema | Object | 是 | 表单Schema定义 |
| syncType | String | 是 | 同步类型：CREATE-创建，UPDATE-更新，DELETE-删除 |

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

**Python调用示例：**

```python
from auth import ASign, Credential
import requests
import json

credential = Credential(
    enterprise_code="COMPANY001",
    app_id="APP001",
    app_secret="your_app_secret"
)

request_data = {
    "formCode": "SALES_FORM",
    "formName": "销售订单表单",
    "formSchema": {
        "components": [
            {
                "type": "input",
                "name": "customerName",
                "label": "客户名称",
                "required": True
            },
            {
                "type": "number",
                "name": "amount",
                "label": "订单金额",
                "required": True
            }
        ]
    },
    "syncType": "CREATE"
}

request_body = json.dumps(request_data, ensure_ascii=False)
signature = ASign.get_hex_signature(credential, request_body)

headers = {
    "Content-Type": "application/json",
    "A-Signature": signature
}

response = requests.post(
    "https://aiflow.example.com/aflow/api/form/sync",
    headers=headers,
    data=request_body.encode('utf-8')
)

result = response.json()
if result['status'] == 0:
    print("表单同步成功")
```

### 5.3 审批日志接口

**接口地址：** `POST /aflow/api/order/approval/log`

**接口说明：** 同步审批日志到aiflow系统

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
      "handleResult": "APPROVED",
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

---

# 第二部分：Odoo ERP需要提供的接口

## 1. OAuth2接口

### 1.1 授权端点（Authorization Endpoint）

**接口地址：** `GET /odoo/api/oauth2/authorize`

**接口说明：** OAuth2授权端点，用于用户登录和授权

**请求参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| client_id | String | 是 | aiflow的应用ID |
| redirect_uri | String | 是 | 授权后的回调地址 |
| response_type | String | 是 | 固定值 `code` |
| scope | String | 否 | 授权范围 |
| state | String | 是 | 随机字符串，用于防止CSRF攻击 |

**响应：**
- 成功：重定向到 `redirect_uri?code=authorization_code&state=random_state_string`
- 失败：重定向到 `redirect_uri?error=error_code&error_description=description`

**Python实现示例（Flask）：**

```python
from flask import Flask, request, redirect, session
import secrets
import urllib.parse

app = Flask(__name__)
app.secret_key = 'your_secret_key'

@app.route('/odoo/api/oauth2/authorize', methods=['GET'])
def authorize():
    client_id = request.args.get('client_id')
    redirect_uri = request.args.get('redirect_uri')
    response_type = request.args.get('response_type')
    state = request.args.get('state')
    
    # 验证client_id
    if client_id != 'APP001':
        error_params = {
            'error': 'invalid_client',
            'error_description': 'Invalid client ID'
        }
        return redirect(f"{redirect_uri}?{urllib.parse.urlencode(error_params)}")
    
    # 如果用户未登录，重定向到登录页
    if 'user_id' not in session:
        session['oauth_redirect_uri'] = redirect_uri
        session['oauth_state'] = state
        return redirect('/login')
    
    # 用户已登录，生成授权码
    authorization_code = secrets.token_urlsafe(32)
    # 保存授权码（实际应用中应存储到数据库或缓存）
    # save_authorization_code(authorization_code, client_id, session['user_id'])
    
    # 重定向回aiflow
    params = {
        'code': authorization_code,
        'state': state
    }
    return redirect(f"{redirect_uri}?{urllib.parse.urlencode(params)}")
```

### 1.2 令牌端点（Token Endpoint）

**接口地址：** `POST /odoo/api/oauth2/token`

**接口说明：** OAuth2令牌端点，用于交换授权码获取访问令牌

**请求头：**
```
Content-Type: application/json
```

**请求体：**
```json
{
  "clientId": "APP001",
  "clientSecret": "client_secret",
  "grantType": "authorization_code",
  "code": "authorization_code",
  "redirectUri": "https://aiflow.example.com/aflow/api/oauth2/callback"
}
```

**请求参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| clientId | String | 是 | aiflow配置的OAuth2 clientId |
| clientSecret | String | 是 | aiflow配置的OAuth2 clientSecret |
| grantType | String | 是 | 授权类型：`authorization_code` |
| code | String | 是 | 授权码 |
| redirectUri | String | 是 | 必须与授权请求中的redirect_uri一致 |

**响应体：**
```json
{
  "accessToken": "访问令牌",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "refreshToken": "刷新令牌"
}
```

**Python实现示例（Flask）：**

```python
from flask import Flask, request, jsonify
import secrets
import time

app = Flask(__name__)

@app.route('/odoo/api/oauth2/token', methods=['POST'])
def token():
    data = request.get_json()
    
    client_id = data.get('clientId')
    client_secret = data.get('clientSecret')
    grant_type = data.get('grantType')
    code = data.get('code')
    redirect_uri = data.get('redirectUri')
    
    # 验证客户端凭证
    if client_id != 'APP001' or client_secret != 'client_secret':
        return jsonify({
            'error': 'invalid_client',
            'error_description': 'Invalid client credentials'
        }), 401
    
    # 验证授权码（实际应用中应从数据库或缓存中查询）
    # if not validate_authorization_code(code, client_id, redirect_uri):
    #     return jsonify({
    #         'error': 'invalid_grant',
    #         'error_description': 'Invalid authorization code'
    #     }), 400
    
    # 生成访问令牌和刷新令牌
    access_token = secrets.token_urlsafe(32)
    refresh_token = secrets.token_urlsafe(32)
    
    # 保存令牌（实际应用中应存储到数据库）
    # save_token(access_token, refresh_token, user_id, expires_in=3600)
    
    return jsonify({
        'accessToken': access_token,
        'tokenType': 'Bearer',
        'expiresIn': 3600,
        'refreshToken': refresh_token
    })
```

### 1.3 用户信息端点（User Info Endpoint）

**接口地址：** `POST /odoo/api/oauth2/userinfo`

**接口说明：** 获取用户信息

> **重要提示**：如果选择**方案二：aiflow主动同步钉钉**，此接口必须返回`phoneNumber`（手机号）和`deptId`（部门ID），用于用户绑定。

**请求头：**
```
Authorization: Bearer {accessToken}
```

**响应体：**
```json
{
  "userId": "用户ID（ERP用户编码）",
  "username": "用户名",
  "name": "姓名",
  "phoneNumber": "手机号（必填，至少phoneNumber或email选一个）",
  "email": "邮箱（可选）",
  "deptId": "部门ID（必填，用于建立部门映射）",
  "deptName": "部门名称"
}
```

**Python实现示例（Flask）：**

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/odoo/api/oauth2/userinfo', methods=['POST'])
def userinfo():
    # 从请求头获取访问令牌
    auth_header = request.headers.get('Authorization')
    if not auth_header or not auth_header.startswith('Bearer '):
        return jsonify({
            'error': 'invalid_token',
            'error_description': 'Missing or invalid Authorization header'
        }), 401
    
    access_token = auth_header.split(' ')[1]
    
    # 验证访问令牌（实际应用中应从数据库或缓存中查询）
    # user_id = validate_access_token(access_token)
    # if not user_id:
    #     return jsonify({
    #         'error': 'invalid_token',
    #         'error_description': 'Invalid or expired access token'
    #     }), 401
    
    # 查询用户信息（实际应用中应从数据库查询）
    # user = get_user_by_id(user_id)
    
    # 示例数据
    return jsonify({
        'userId': 'USER001',
        'username': 'zhangsan',
        'name': '张三',
        'phoneNumber': '13800138000',
        'email': 'zhangsan@example.com',
        'deptId': 'DEPT001',
        'deptName': '技术部'
    })
```

---

## 2. 流程数据接收接口

### 2.1 接收流程数据接口

**接口地址：** `POST /odoo/api/workflow/receive`

**接口说明：** 接收aiflow流程引擎推送的流程数据

**请求头：**
```
Content-Type: application/json
A-Signature: {签名十六进制字符串}
```

**请求体：**
```json
{
  "orderId": 123456,
  "businessKey": "业务单据号",
  "taskId": "任务ID",
  "taskName": "任务名称",
  "action": "START",
  "formData": {
    "customerName": "客户名称",
    "amount": 10000
  },
  "processVariables": {
    "variable1": "value1"
  },
  "handlerUserCode": "处理人编码",
  "handleTime": "2025-01-24 15:30:00"
}
```

**请求参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| orderId | Long | 是 | 流程订单号 |
| businessKey | String | 是 | 业务单据号 |
| taskId | String | 是 | 任务ID |
| taskName | String | 是 | 任务名称 |
| action | String | 是 | 操作类型：START-启动，COMPLETE-完成，REJECT-拒绝，CANCEL-取消 |
| formData | Object | 否 | 表单数据 |
| processVariables | Object | 否 | 流程变量 |
| handlerUserCode | String | 否 | 处理人编码 |
| handleTime | String | 否 | 处理时间（格式：yyyy-MM-dd HH:mm:ss） |

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

**Python实现示例（Flask）：**

```python
from flask import Flask, request, jsonify
from auth import ASign, Credential
import json

app = Flask(__name__)

# 配置信息（应与aiflow配置一致）
credential = Credential(
    enterprise_code="COMPANY001",
    app_id="APP001",
    app_secret="your_app_secret"
)

@app.route('/odoo/api/workflow/receive', methods=['POST'])
def receive_workflow():
    # 验证签名
    signature_header = request.headers.get('A-Signature')
    request_body = request.get_data(as_text=True)
    
    # 验证签名（实际应用中应实现签名验证逻辑）
    # if not verify_signature(signature_header, request_body, credential):
    #     return jsonify({
    #         'status': -4002,
    #         'msg': '签名验证失败'
    #     }), 403
    
    data = request.get_json()
    
    order_id = data.get('orderId')
    business_key = data.get('businessKey')
    task_id = data.get('taskId')
    action = data.get('action')
    form_data = data.get('formData', {})
    
    # 处理流程数据
    if action == 'START':
        # 创建业务单据
        # create_business_order(business_key, form_data)
        pass
    elif action == 'COMPLETE':
        # 更新业务单据状态
        # update_business_order_status(business_key, 'approved')
        pass
    elif action == 'REJECT':
        # 更新业务单据状态
        # update_business_order_status(business_key, 'rejected')
        pass
    
    return jsonify({
        'status': 0,
        'msg': 'success',
        'data': {
            'success': True
        }
    })

def verify_signature(signature_header, request_body, credential):
    """验证签名"""
    try:
        # 计算签名
        expected_signature = ASign.get_hex_signature(credential, request_body)
        return signature_header == expected_signature
    except Exception as e:
        print(f"签名验证异常: {e}")
        return False
```

### 2.2 接收审批日志接口（可选）

**接口地址：** `POST /odoo/api/workflow/approval/log`

**接口说明：** 接收aiflow推送的审批日志（可选接口）

**请求头：**
```
Content-Type: application/json
A-Signature: {签名十六进制字符串}
```

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
      "handleResult": "APPROVED",
      "handleComment": "审批意见",
      "handleTime": "2025-01-24 15:30:00",
      "formData": {
        "comment": "审批意见"
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

**Python实现示例（Flask）：**

```python
from flask import Flask, request, jsonify
from auth import ASign, Credential

app = Flask(__name__)

credential = Credential(
    enterprise_code="COMPANY001",
    app_id="APP001",
    app_secret="your_app_secret"
)

@app.route('/odoo/api/workflow/approval/log', methods=['POST'])
def receive_approval_log():
    # 验证签名
    signature_header = request.headers.get('A-Signature')
    request_body = request.get_data(as_text=True)
    
    # 验证签名
    # if not verify_signature(signature_header, request_body, credential):
    #     return jsonify({
    #         'status': -4002,
    #         'msg': '签名验证失败'
    #     }), 403
    
    data = request.get_json()
    order_id = data.get('orderId')
    business_key = data.get('businessKey')
    logs = data.get('logs', [])
    
    # 保存审批日志
    for log in logs:
        # save_approval_log(business_key, log)
        pass
    
    return jsonify({
        'status': 0,
        'msg': 'success',
        'data': {
            'success': True
        }
    })
```

---

## 附录：错误码说明

### aiflow接口错误码

| 状态码 | 说明 |
|--------|------|
| 0 | 成功 |
| -4002 | 参数错误 |
| -4001 | 操作不支持 |
| -1024 | 通用错误 |

### Odoo接口错误码

| 错误码 | 说明 |
|--------|------|
| invalid_client | 无效的客户端ID或密钥 |
| invalid_grant | 无效的授权码或刷新令牌 |
| invalid_token | 无效或过期的访问令牌 |
| unauthorized_client | 客户端无权使用此授权类型 |

---

## 附录：完整调用示例

### Python完整示例：同步部门数据

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

"""
aiflow接口调用示例：同步部门数据
"""

from auth import ASign, Credential
import requests
import json

# 配置信息
AIFLOW_BASE_URL = "https://aiflow.example.com"
CREDENTIAL = Credential(
    enterprise_code="COMPANY001",
    app_id="APP001",
    app_secret="your_app_secret"
)

def sync_department():
    """同步部门数据"""
    url = f"{AIFLOW_BASE_URL}/aflow/api/sys/sync/department"
    
    # 准备请求数据（按层级顺序）
    request_data = {
        "departments": [
            {
                "deptId": "DEPT001",
                "deptName": "技术部",
                "parentId": "",
                "orderNum": 100,
                "status": 1
            },
            {
                "deptId": "DEPT002",
                "deptName": "开发组",
                "parentId": "DEPT001",
                "orderNum": 101,
                "status": 1
            }
        ]
    }
    
    request_body = json.dumps(request_data, ensure_ascii=False)
    signature = ASign.get_hex_signature(CREDENTIAL, request_body)
    
    headers = {
        "Content-Type": "application/json",
        "A-Signature": signature
    }
    
    try:
        response = requests.post(
            url,
            headers=headers,
            data=request_body.encode('utf-8'),
            timeout=30
        )
        response.raise_for_status()
        result = response.json()
        
        if result['status'] == 0:
            print(f"✅ 部门同步成功: 成功{result['data']['successCount']}条, 失败{result['data']['failCount']}条")
            return True
        else:
            print(f"❌ 部门同步失败: {result['msg']}")
            return False
            
    except requests.exceptions.RequestException as e:
        print(f"❌ 请求异常: {e}")
        return False

if __name__ == "__main__":
    sync_department()
```

---

**文档版本：** v1.0  
**最后更新：** 2025-01-24  
**维护者：** aiflow团队
