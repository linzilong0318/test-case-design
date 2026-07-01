# 测试用例输出格式规范

> 本文件定义测试用例的标准输出格式。所有平台（PC Web、移动App、小程序、移动Web、桌面端、Agent）共用同一套表格模板，差异仅在于测试场景内容。

---

## 标准模板

```
| 字段 | 内容 |
|-----|------|
| 用例编号 | {编号} |
| 测试标题 | {标题} |
| 测试类型 | {类型} |
| 功能模块 | {模块} |
| 子功能模块 | {子功能，可选} |
| 用例级别 | P0 / P1 / P2 |
| 测试维度 | {维度，可选} |
| 预置条件 | {条件} |
| 测试步骤 | {步骤} |
| 预期结果 | {结果} |
| 实际结果 | |
| 测试状态 | |
```

## 字段说明

| 字段 | 说明 | 必填 |
|------|------|------|
| 用例编号 | 格式见 `references/templates/common-rules.md` 编号规则 | 是 |
| 测试标题 | 简洁描述测试目的，以"验证"开头 | 是 |
| 测试类型 | 格式：`{测试大类} - {测试子类}`，如"功能测试 - 增删改查" | 是 |
| 功能模块 | 所属功能模块名称 | 是 |
| 子功能模块 | 具体子功能点 | 否 |
| 用例级别 | P0=核心功能，P1=重要功能，P2=次要/体验 | 是 |
| 测试维度 | 平台专项维度，如手势/屏幕/网络等 | 否 |
| 预置条件 | 编号列表，用 `<br>` 换行 | 是 |
| 测试步骤 | 编号列表，每步一个操作，用 `<br>` 换行。**必须给出具体输入值/参数/操作对象**，让测试人员可直接照抄执行，禁止用描述性语言代替（如应写 `输入 用户名:admin, 密码:123456`，不得写"输入正确的登录信息"；应写 `输入 2000 个汉字的随机文本`，不得写"输入超长字符串"）。**例外**：大量数据输入（如 10000 字文本、500MB 文件）允许用描述性语言 + 明确参数说明 | 是 |
| 预期结果 | 编号列表，与步骤一一对应，用 `<br>` 换行 | 是 |
| 实际结果 | 执行后填写 | 执行时填写 |
| 测试状态 | 通过/失败/阻塞/跳过 | 执行时填写 |

---

## 示例

以下示例展示不同测试场景下的格式应用，格式模板完全一致。

### 示例一：功能测试

| 字段 | 内容 |
|-----|------|
| 用例编号 | TC_FUNC_CRUD_001 |
| 测试标题 | 验证新增数据功能 |
| 测试类型 | 功能测试 - 增删改查 |
| 功能模块 | 数据管理 |
| 用例级别 | P0 |
| 预置条件 | 1. 用户已登录（账号：testuser，密码：Test@123456）<br>2. 拥有新增数据权限 |
| 测试步骤 | 1. 点击"新增"按钮<br>2. 在"名称"输入框输入"测试数据_20260101"<br>3. 在"描述"输入框输入"自动化测试生成的描述信息"<br>4. 选择"类型"下拉框为"类型A"<br>5. 点击"保存"按钮<br>6. 检查新增成功提示<br>7. 在列表中查找新增的记录 |
| 预期结果 | 1. 新增页面正常打开<br>2. 输入框正常接受输入<br>3. 输入框正常接受输入<br>4. 下拉框选中"类型A"<br>5. 数据保存成功<br>6. 显示"新增成功"提示<br>7. 列表中显示名称为"测试数据_20260101"的记录，所有字段值与输入一致 |
| 实际结果 | |
| 测试状态 | |

### 示例二：平台专项测试（移动App）

| 字段 | 内容 |
|-----|------|
| 用例编号 | APP_LIST_GESTURE_001 |
| 测试标题 | 验证下拉刷新功能 |
| 测试类型 | 移动端测试 - 手势测试 |
| 功能模块 | 列表刷新 |
| 用例级别 | P1 |
| 预置条件 | 进入列表页面，网络正常 |
| 测试步骤 | 1. 手指放在屏幕中间偏上位置<br>2. 向下缓慢滑动<br>3. 观察到刷新动画出现<br>4. 释放手指<br>5. 等待数据加载完成 |
| 预期结果 | 1. 下拉时显示刷新动画<br>2. 释放后自动回弹到顶部<br>3. 数据重新加载<br>4. 刷新成功提示<br>5. 列表显示最新数据 |
| 实际结果 | |
| 测试状态 | |

### 示例三：AI Agent 测试

| 字段 | 内容 |
|-----|------|
| 用例编号 | TC_AGENT_TASK_001 |
| 测试标题 | 验证 Agent 能准确理解显式指令并执行 |
| 测试类型 | Agent 测试 - 任务完成度 |
| 功能模块 | 任务理解 |
| 用例级别 | P0 |
| 预置条件 | Agent 处于空闲状态，已配置可用工具 |
| 测试步骤 | 1. 向 Agent 发送明确指令："请创建一个名为 test.txt 的文本文件，内容为 hello"<br>2. 等待 Agent 执行 |
| 预期结果 | Agent 选择正确的文件创建工具，按要求创建文件（名称 test.txt，内容 hello） |
| 实际结果 | |
| 测试状态 | |

### 示例四：接口测试

| 字段 | 内容 |
|-----|------|
| 用例编号 | TC_API_GET_001 |
| 测试标题 | 验证获取用户信息接口 |
| 测试类型 | 接口测试 - 功能测试 |
| 功能模块 | 用户接口 |
| 用例级别 | P1 |
| 预置条件 | 接口信息：方法 GET，URL /api/v1/users/{userId}，认证 Bearer Token。测试环境存在 userId=12345 的用户 |
| 测试步骤 | 1. 设置请求头 Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...（使用有效token）<br>2. 发送 GET 请求到 /api/v1/users/12345<br>3. 检查响应状态码<br>4. 检查响应体 JSON 结构<br>5. 检查各字段值<br>6. 用不存在的 userId=99999999 发送请求<br>7. 用无效 Token 发送请求 |
| 预期结果 | 1. 状态码 200<br>2. 响应体包含 id、name、email 等字段<br>3. id 为 12345<br>4. name 为字符串类型非空<br>5. email 格式符合邮箱正则<br>6. userId=99999999 返回 404，提示"用户不存在"<br>7. 无效 Token 返回 401，提示"未授权" |
| 实际结果 | |
| 测试状态 | |

---

## JSON 格式规范

> 默认输出格式为 JSON。Markdown 表格格式（见上文）作为备选输出方式。

### 字段映射

| 字段中文名 | JSON key | 类型 | 必填 | 说明 |
|-----------|---------|------|------|------|
| 用例编号 | `caseId` | string | 是 | 格式见 `references/templates/common-rules.md` 编号规则 |
| 用例标题 | `title` | string | 是 | 简洁描述测试目的，以"验证"开头 |
| 测试类型 | `testType` | string | 是 | 格式：`{测试大类} - {测试子类}`，如"功能测试 - 增删改查" |
| 功能模块 | `module` | string | 是 | 所属功能模块名称 |
| 子功能模块 | `subModule` | string | 否 | 具体子功能点 |
| 优先级 | `priority` | string | 是 | P0=核心功能，P1=重要功能，P2=次要/体验，P3=建议 |
| 测试维度 | `dimension` | string | 否 | 平台专项维度，如手势/屏幕/网络等 |
| 前置条件 | `prerequisites` | string | 是 | 编号列表，用 `<br>` 换行 |
| 测试步骤 | `steps` | string | 是 | 编号列表，每步一个操作，用 `<br>` 换行。**必须给出具体输入值/参数/操作对象** |
| 预期结果 | `expectedResults` | string | 是 | 编号列表，与步骤一一对应，用 `<br>` 换行 |
| 实际结果 | `actualResult` | string | 否 | 执行后填写 |
| 测试状态 | `status` | string | 否 | 通过/失败/阻塞/跳过，执行时填写 |

### JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "array",
  "items": {
    "type": "object",
    "required": ["caseId", "title", "testType", "module", "priority", "prerequisites", "steps", "expectedResults"],
    "properties": {
      "caseId": {
        "type": "string",
        "description": "用例编号，如 TC_FUNC_CRUD_001"
      },
      "title": {
        "type": "string",
        "description": "用例标题，以"验证"开头"
      },
      "testType": {
        "type": "string",
        "description": "测试类型，格式：{测试大类} - {测试子类}"
      },
      "module": {
        "type": "string",
        "description": "功能模块名称"
      },
      "subModule": {
        "type": "string",
        "description": "子功能模块（可选）"
      },
      "priority": {
        "type": "string",
        "enum": ["P0", "P1", "P2", "P3"],
        "description": "优先级"
      },
      "dimension": {
        "type": "string",
        "description": "测试维度（可选），如手势/屏幕/网络等"
      },
      "prerequisites": {
        "type": "string",
        "description": "前置条件，编号列表，<br>换行"
      },
      "steps": {
        "type": "string",
        "description": "测试步骤，编号列表，每步一个操作，<br>换行"
      },
      "expectedResults": {
        "type": "string",
        "description": "预期结果，编号列表，<br>换行"
      },
      "actualResult": {
        "type": "string",
        "description": "实际结果（执行时填写）"
      },
      "status": {
        "type": "string",
        "enum": ["通过", "失败", "阻塞", "跳过"],
        "description": "测试状态（执行时填写）"
      }
    }
  }
}
```

### 输出示例

```json
[
  {
    "caseId": "TC_FUNC_CRUD_001",
    "title": "验证新增数据功能",
    "testType": "功能测试 - 增删改查",
    "module": "数据管理",
    "subModule": "",
    "priority": "P0",
    "dimension": "",
    "prerequisites": "1. 用户已登录（账号：testuser，密码：Test@123456）<br>2. 拥有新增数据权限",
    "steps": "1. 点击\"新增\"按钮<br>2. 在\"名称\"输入框输入\"测试数据_20260101\"<br>3. 在\"描述\"输入框输入\"自动化测试生成的描述信息\"<br>4. 选择\"类型\"下拉框为\"类型A\"<br>5. 点击\"保存\"按钮",
    "expectedResults": "1. 新增页面正常打开<br>2. 输入框正常接受输入<br>3. 输入框正常接受输入<br>4. 下拉框选中\"类型A\"<br>5. 数据保存成功",
    "actualResult": "",
    "status": ""
  },
  {
    "caseId": "APP_LIST_GESTURE_001",
    "title": "验证下拉刷新功能",
    "testType": "移动端测试 - 手势测试",
    "module": "列表刷新",
    "subModule": "",
    "priority": "P1",
    "dimension": "手势",
    "prerequisites": "进入列表页面，网络正常",
    "steps": "1. 手指放在屏幕中间偏上位置<br>2. 向下缓慢滑动<br>3. 观察到刷新动画出现<br>4. 释放手指<br>5. 等待数据加载完成",
    "expectedResults": "1. 下拉时显示刷新动画<br>2. 释放后自动回弹到顶部<br>3. 数据重新加载<br>4. 刷新成功提示<br>5. 列表显示最新数据",
    "actualResult": "",
    "status": ""
  },
  {
    "caseId": "TC_API_GET_001",
    "title": "验证获取用户信息接口",
    "testType": "接口测试 - 功能测试",
    "module": "用户接口",
    "subModule": "",
    "priority": "P1",
    "dimension": "",
    "prerequisites": "接口信息：方法 GET，URL /api/v1/users/{userId}，认证 Bearer Token。测试环境存在 userId=12345 的用户",
    "steps": "1. 设置请求头 Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...（使用有效token）<br>2. 发送 GET 请求到 /api/v1/users/12345<br>3. 检查响应状态码<br>4. 检查响应体 JSON 结构<br>5. 检查各字段值<br>6. 用不存在的 userId=99999999 发送请求<br>7. 用无效 Token 发送请求",
    "expectedResults": "1. 状态码 200<br>2. 响应体包含 id、name、email 等字段<br>3. id 为 12345<br>4. name 为字符串类型非空<br>5. email 格式符合邮箱正则<br>6. userId=99999999 返回 404，提示\"用户不存在\"<br>7. 无效 Token 返回 401，提示\"未授权\"",
    "actualResult": "",
    "status": ""
  }
]
```

---

## JSON 与 Markdown 格式对照

| Markdown 字段 | JSON key | 变更说明 |
|--------------|---------|---------|
| 用例编号 | `caseId` | 字段名保持不变 |
| 测试标题 | `title` | 改用英文 key |
| 测试类型 | `testType` | 改用英文 key |
| 功能模块 | `module` | 改用英文 key |
| 子功能模块 | `subModule` | 改用英文 key |
| 用例级别 | `priority` | 字段名改为"优先级"，key 使用英文 |
| 测试维度 | `dimension` | 改用英文 key |
| 预置条件 | `prerequisites` | 改用英文 key |
| 测试步骤 | `steps` | 改用英文 key |
| 预期结果 | `expectedResults` | 改用英文 key |
| 实际结果 | `actualResult` | 改用英文 key |
| 测试状态 | `status` | 改用英文 key |
