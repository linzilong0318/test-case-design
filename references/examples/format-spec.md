# 测试用例输出格式规范

> 本文件定义测试用例的标准输出格式。主输出格式为 **API JSON 格式**（与后端 `/api/v1/testcase/save` 接口对齐），Markdown 表格格式作为备选输出方式。

---

## 一、API JSON 格式（主格式）

### 1.1 字段定义

| 字段名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `caseCode` | string | 是 | 用例编号，格式见 `references/templates/common-rules.md` 编号规则 |
| `batchNo` | string | 是 | 批次号，格式 `yyyyMMddHHmmssSSS`（17位数字字符串），同一批次所有用例相同 |
| `title` | string | 是 | 用例标题，简洁描述测试目的，以"验证"开头 |
| `type` | int | 是 | 测试类型编码，见下方类型映射表 |
| `module` | string | 是 | 所属功能模块名称 |
| `subModule` | string | 否 | 具体子功能点，无则传空字符串 `""` |
| `priority` | int | 是 | 优先级编码：0=高, 1=中, 2=低 |
| `preconditions` | string | 是 | 前置条件，编号列表，用 `\n` 换行 |
| `steps` | string | 是 | 测试步骤，编号列表，每步一个操作，用 `\n` 换行。**必须给出具体输入值/参数/操作对象**，让测试人员可直接照抄执行。大量数据场景（>1000字符文本、>1MB文件、>50行数据）可用描述性语言+明确参数说明 |
| `expectedResults` | string | 是 | 预期结果，编号列表，与步骤一一对应，用 `\n` 换行 |

### 1.2 类型映射表（type）

| type 值 | 含义 | 适用场景 |
|---------|------|---------|
| 1 | 功能测试 | CRUD、表单验证、状态流转、业务规则、数据校验 |
| 2 | 接口测试 | HTTP 方法、状态码、数据校验、认证授权、安全、错误处理 |
| 3 | 兼容性测试 | 平台兼容、浏览器兼容、设备兼容、版本兼容 |
| 4 | UI 测试 | 布局、样式、交互状态、主题 |
| 5 | 性能测试 | 响应时间、资源占用、稳定性 |
| 6 | 可用性测试 | 导航、表单易用性、帮助、可访问性 |
| 7 | 联动测试 | 表单联动、列表联动、搜索联动、状态联动、数据联动 |
| 8 | 路由测试 | 导航跳转、浏览器导航、路由参数、深链接、路由守卫 |
| 9 | Agent 测试 | 任务完成度、工具调用、记忆管理、安全边界、内容质量 |

### 1.3 优先级映射表（priority）

| int 值 | 含义 | 对应旧 P 级 | 判定标准 |
|--------|------|------------|---------|
| 0 | 高 | P0（致命） | 核心功能、影响主流程、导致系统崩溃、数据丢失 |
| 1 | 中 | P1（严重） | 重要功能、影响用户体验、功能错误、界面严重问题 |
| 2 | 低 | P2（一般）/ P3（建议） | 次要功能、轻微体验问题、优化建议 |

### 1.4 JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "array",
  "items": {
    "type": "object",
    "required": ["caseCode", "batchNo", "title", "type", "module", "priority", "preconditions", "steps", "expectedResults"],
    "properties": {
      "caseCode": {
        "type": "string",
        "description": "用例编号，如 TC_ENV_MGM_001"
      },
      "batchNo": {
        "type": "string",
        "description": "批次号，yyyyMMddHHmmssSSS，同一批次所有用例相同"
      },
      "title": {
        "type": "string",
        "description": "用例标题，以"验证"开头"
      },
      "type": {
        "type": "integer",
        "enum": [1, 2, 3, 4, 5, 6, 7, 8, 9],
        "description": "测试类型编码"
      },
      "module": {
        "type": "string",
        "description": "功能模块名称"
      },
      "subModule": {
        "type": "string",
        "description": "子功能模块（可选，无则传空字符串）"
      },
      "priority": {
        "type": "integer",
        "enum": [0, 1, 2],
        "description": "优先级：0=高, 1=中, 2=低"
      },
      "preconditions": {
        "type": "string",
        "description": "前置条件，编号列表，\\n 换行"
      },
      "steps": {
        "type": "string",
        "description": "测试步骤，编号列表，每步一个操作，\\n 换行。必须给出具体输入值"
      },
      "expectedResults": {
        "type": "string",
        "description": "预期结果，编号列表，与步骤一一对应，\\n 换行"
      }
    }
  }
}
```

### 1.5 API 请求体完整格式

用例 JSON 数组需包装为以下格式发送给 `/api/v1/testcase/save` 接口：

```json
{
  "sessionId": "session_01",
  "cases": [
    {
      "caseCode": "TC_ENV_MGM_001",
      "batchNo": "20260709143025123",
      "title": "验证添加环境输入非法字符时校验拦截",
      "type": 1,
      "module": "环境管理",
      "subModule": "添加环境",
      "priority": 0,
      "preconditions": "1. 用户已成功登录系统\n2. 具备环境管理模块的编辑权限",
      "steps": "1. 点击「添加环境」按钮打开弹窗\n2. 在环境名称输入框中输入特殊字符「@#￥%」\n3. 点击底部的「保存」按钮",
      "expectedResults": "1. 弹窗正常响应打开\n2. 输入框失去焦点时或点击保存时，输入框下方高亮红色提示「名称格式不正确」\n3. 表单拦截，不触发落库请求"
    }
  ]
}
```

### 1.6 完整示例（多条用例）

```json
[
  {
    "caseCode": "TC_ENV_MGM_001",
    "batchNo": "20260709143025123",
    "title": "验证添加环境输入非法字符时校验拦截",
    "type": 1,
    "module": "环境管理",
    "subModule": "添加环境",
    "priority": 0,
    "preconditions": "1. 用户已成功登录系统\n2. 具备环境管理模块的编辑权限",
    "steps": "1. 点击「添加环境」按钮打开弹窗\n2. 在环境名称输入框中输入特殊字符「@#￥%」\n3. 点击底部的「保存」按钮",
    "expectedResults": "1. 弹窗正常响应打开\n2. 输入框失去焦点时或点击保存时，输入框下方高亮红色提示「名称格式不正确」\n3. 表单拦截，不触发落库请求"
  },
  {
    "caseCode": "TC_ENV_MGM_002",
    "batchNo": "20260709143025123",
    "title": "验证添加环境输入超长名称时校验拦截",
    "type": 1,
    "module": "环境管理",
    "subModule": "添加环境",
    "priority": 0,
    "preconditions": "1. 用户已成功登录系统\n2. 具备环境管理模块的编辑权限",
    "steps": "1. 点击「添加环境」按钮打开弹窗\n2. 在环境名称输入框中输入一个包含 200 个汉字的超长文本（可重复"测试"二字 100 次）\n3. 点击底部的「保存」按钮",
    "expectedResults": "1. 弹窗正常响应打开\n2. 输入框下方高亮红色提示「名称长度不能超过100个字符」\n3. 表单拦截，不触发落库请求"
  },
  {
    "caseCode": "TC_API_ENV_ADD_001",
    "batchNo": "20260709143025123",
    "title": "验证添加环境接口正常创建环境",
    "type": 2,
    "module": "环境管理",
    "subModule": "添加环境接口",
    "priority": 0,
    "preconditions": "1. 接口信息：方法 POST，URL /api/v1/environment/create，认证 Bearer Token\n2. 已获取有效 Token",
    "steps": "1. 设置请求头 Authorization: Bearer {有效token}\n2. 发送 POST 请求到 /api/v1/environment/create，请求体 {\"name\": \"测试环境_自动化测试\", \"type\": \"development\"}\n3. 检查响应状态码\n4. 检查响应体中的环境信息",
    "expectedResults": "1. 状态码 200\n2. 响应体 success 为 true\n3. 响应体 data 中包含新创建的环境 ID\n4. 环境名称与请求一致"
  }
]
```

---

## 二、Markdown 表格格式（备选）

### 2.1 标准模板

```
| 字段 | 内容 |
|-----|------|
| 用例编号 | {caseCode} |
| 测试标题 | {title} |
| 测试类型 | {type名称} |
| 功能模块 | {module} |
| 子功能模块 | {subModule} |
| 用例级别 | {priority名称} |
| 预置条件 | {preconditions} |
| 测试步骤 | {steps} |
| 预期结果 | {expectedResults} |
```

### 2.2 字段说明

| 字段 | 说明 | 必填 |
|------|------|------|
| 用例编号 | 格式见 `references/templates/common-rules.md` 编号规则 | 是 |
| 测试标题 | 简洁描述测试目的，以"验证"开头 | 是 |
| 测试类型 | 使用中文名称，如"功能测试 - 增删改查" | 是 |
| 功能模块 | 所属功能模块名称 | 是 |
| 子功能模块 | 具体子功能点 | 否 |
| 用例级别 | 高 / 中 / 低 | 是 |
| 预置条件 | 编号列表，用 `<br>` 换行 | 是 |
| 测试步骤 | 编号列表，每步一个操作，用 `<br>` 换行。**必须给出具体输入值** | 是 |
| 预期结果 | 编号列表，与步骤一一对应，用 `<br>` 换行 | 是 |

### 2.3 示例

| 字段 | 内容 |
|-----|------|
| 用例编号 | TC_ENV_MGM_001 |
| 测试标题 | 验证添加环境输入非法字符时校验拦截 |
| 测试类型 | 功能测试 - 表单验证 |
| 功能模块 | 环境管理 |
| 子功能模块 | 添加环境 |
| 用例级别 | 高 |
| 预置条件 | 1. 用户已成功登录系统<br>2. 具备环境管理模块的编辑权限 |
| 测试步骤 | 1. 点击「添加环境」按钮打开弹窗<br>2. 在环境名称输入框中输入特殊字符「@#￥%」<br>3. 点击底部的「保存」按钮 |
| 预期结果 | 1. 弹窗正常响应打开<br>2. 输入框失去焦点时或点击保存时，输入框下方高亮红色提示「名称格式不正确」<br>3. 表单拦截，不触发落库请求 |

---

## 三、API JSON 与 Markdown 格式对照

| Markdown 字段 | JSON key | 类型 | 说明 |
|--------------|---------|------|------|
| 用例编号 | `caseCode` | string | 编号规则不变 |
| 测试标题 | `title` | string | 内容不变 |
| 测试类型 | `type` | int | API 使用编码值 1-9 |
| 功能模块 | `module` | string | 内容不变 |
| 子功能模块 | `subModule` | string | 内容不变 |
| 用例级别 | `priority` | int | API 使用编码 0/1/2 |
| 预置条件 | `preconditions` | string | 换行符从 `<br>` 改为 `\n` |
| 测试步骤 | `steps` | string | 换行符从 `<br>` 改为 `\n` |
| 预期结果 | `expectedResults` | string | 换行符从 `<br>` 改为 `\n` |
| — | `batchNo` | string | **仅 API 格式有此字段** |
