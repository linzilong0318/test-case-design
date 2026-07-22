# API 集成说明

> 本文档定义后端 API 接口的调用方式。所有 API 调用现在均通过标准脚本完成（自动完成 Nacos 服务发现 + 接口调用）。

---

## 一、Nacos 服务发现

已整合到 `scripts/discover_and_call.py` 模块中，无需单独调用外部 skill。

### 1.1 环境变量

所有上传脚本自动从以下环境变量读取 Nacos 配置：

| 环境变量 | 必填 | 说明 | 默认值 |
|---------|------|------|--------|
| `NACOS_SERVER_ADDRESSES` | 否 | Nacos 服务器地址 | `10.120.7.97:8848` |
| `NACOS_NAMESPACE` | 否 | Nacos 命名空间 ID | 空 |
| `NACOS_USERNAME` | 否 | Nacos 登录用户名 | `nacos` |
| `NACOS_PASSWORD` | 否 | Nacos 登录密码 | 空 |
| `BACKEND_SERVICE_NAME` | **是** | 后端服务名（同时也是 URL 路径前缀） | 无 |
| `NACOS_GROUP_NAME` | 否 | Nacos 分组名 | `DEFAULT_GROUP` |

> **⚠️ 安全提醒**：`NACOS_PASSWORD` 等敏感环境变量的值**绝对不可**展示给用户。

### 1.2 导入使用

```python
import sys
sys.path.insert(0, '/path/to/scripts')
from discover_and_call import discover_service, build_url, call_api, call_api_multipart
```

### 1.3 URL 拼接规则

```
http://{ip}:{port}/{service_name}/{api_path}
```

其中 `service_name` 来自环境变量 `BACKEND_SERVICE_NAME`。

---

## 二、用例上传接口

### 2.1 接口信息

| 项目 | 说明 |
|------|------|
| 方法 | POST |
| 路径 | `/api/v1/testcase/save` |
| Content-Type | application/json |

### 2.2 调用方式

使用 `scripts/upload_cases.py`：

```bash
python3 scripts/upload_cases.py \
  --payload-file "/opt/data/tmp/test-case-design/{sessionId}/cases_payload.json"
```

脚本自动完成 Nacos 服务发现 → 读取 JSON → POST 上传。

### 2.3 Request Body

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

**字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `sessionId` | string | 是 | 用户提供的会话标识 |
| `cases` | array | 是 | 用例列表 |
| `cases[].caseCode` | string | 是 | 用例编号 |
| `cases[].batchNo` | string | 是 | 批次号，同一批次所有用例相同 |
| `cases[].title` | string | 是 | 用例标题 |
| `cases[].type` | int | 是 | 测试类型：1-9 |
| `cases[].module` | string | 是 | 功能模块 |
| `cases[].subModule` | string | 否 | 子功能模块 |
| `cases[].priority` | int | 是 | 优先级：0=高, 1=中, 2=低 |
| `cases[].preconditions` | string | 是 | 前置条件 |
| `cases[].steps` | string | 是 | 测试步骤 |
| `cases[].expectedResults` | string | 是 | 预期结果 |

### 2.4 Response

**成功**：
```json
{
  "success": true,
  "code": "00000",
  "message": "",
  "data": true
}
```

**失败**：
```json
{
  "success": false,
  "code": "ERROR_CODE",
  "message": "错误描述",
  "data": null
}
```

### 2.5 重试机制

JSON 文件已持久化到 `/opt/data/tmp/test-case-design/{sessionId}/cases_payload.json`，上传失败时可直接重新执行上传命令，无需重新生成用例：

```bash
# 修复问题后，直接重新上传（无需重新生成用例 JSON）
python3 scripts/upload_cases.py \
  --payload-file "/opt/data/tmp/test-case-design/{sessionId}/cases_payload.json"
```

---

## 三、文件上传接口

### 3.1 接口信息

| 项目 | 说明 |
|------|------|
| 方法 | POST |
| 路径 | `/api/v1/file/upload` |
| Content-Type | multipart/form-data |

### 3.2 调用方式

使用 `scripts/upload_file.py`：

```bash
# 上传待澄清需求清单
python3 scripts/upload_file.py \
  --file "/opt/data/tmp/test-case-design/{sessionId}/待澄清需求清单_{batchNo}.pdf" \
  --session-id "{sessionId}" \
  --batch-no "{batchNo}" \
  --type CHECKLIST

# 上传测试用例评审报告
python3 scripts/upload_file.py \
  --file "/opt/data/tmp/test-case-design/{sessionId}/测试用例评审报告_{batchNo}.pdf" \
  --session-id "{sessionId}" \
  --batch-no "{batchNo}" \
  --type REPORT
```

### 3.3 Query Parameters

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `type` | string | 是 | 上传类型：`CHECKLIST`（待澄清需求清单）、`REPORT`（测试用例评审报告） |
| `sessionId` | string | 是 | 用户提供的会话标识 |
| `batchNo` | string | 是 | 批次号 |

### 3.4 Form Data

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file` | file | 是 | 上传的 PDF 文件 |

### 3.5 Response

**成功**：
```json
{
  "success": true,
  "code": "00000",
  "message": "",
  "data": {
    "sessionId": "session_01",
    "fileName": "test.pdf",
    "relativePath": "/path/test.pdf",
    "url": "http://xxxxxxx/path/test.pdf",
    "type": "CHECKLIST"
  }
}
```

**失败**：
```json
{
  "success": false,
  "code": "ERROR_CODE",
  "message": "错误描述",
  "data": null
}
```

---

## 四、临时文件管理

### 4.1 目录结构

```
/opt/data/tmp/test-case-design/{sessionId}/
├── requirements.pdf                  # 阶段一下载的需求文档
├── requirements.docx                 # 阶段一下载的需求文档（DOCX 格式时）
├── cases_payload.json                # 阶段二生成的用例 JSON（持久化，上传失败可重试）
├── 待澄清需求清单_{batchNo}.pdf       # 阶段三生成的待澄清需求清单
├── 测试用例评审报告_{batchNo}.pdf     # 阶段三生成的测试用例评审报告
└── scripts/                          # 临时 Python 脚本（如有，必须放在此子目录下）
```

### 4.2 生命周期

| 阶段 | 操作 |
|------|------|
| 阶段一 | 准备环境（`uv pip install` 安装依赖），创建目录，使用 `download_requirements.py` 下载文档到目录 |
| 阶段二 | 生成用例 → 持久化 `cases_payload.json` → 使用 `upload_cases.py` 上传 |
| 阶段三 | 使用 md2pdf 生成 PDF 文件到目录 → 使用 `upload_file.py` 上传 |
| 阶段四 | **统一清理**：`rm -rf /opt/data/tmp/test-case-design/{sessionId}/` |

> **原则**：所有临时文件统一存放在 `/opt/data/tmp/test-case-design/{sessionId}/` 下，流程结束统一清理，确保本地不残留任何文件。

---

## 五、错误处理

### 5.1 用例上传失败

- 检查脚本输出的错误信息判断原因
- 常见原因：JSON 格式错误、必填字段缺失、sessionId 无效
- **重试策略**：JSON 文件已持久化，修复问题后直接重新执行 `upload_cases.py`，无需重新生成用例

### 5.2 文件上传失败

- 检查脚本输出的错误信息判断原因
- 常见原因：文件不存在、文件路径错误、type 参数值不正确
- **重试策略**：确认文件路径正确后重新执行 `upload_file.py`

### 5.3 Nacos 服务发现失败

- 如果无法获取后端服务 IP，脚本会打印详细错误信息（含当前已配置的 Nacos 变量名）
- 检查 `BACKEND_SERVICE_NAME` 是否正确，Nacos 地址是否可达
- 不要尝试使用硬编码的 IP 地址

### 5.4 部分成功处理

- 如果用例上传成功但某个文件上传失败，仍然清理临时文件
- 在总结中明确标注哪些步骤成功、哪些失败
