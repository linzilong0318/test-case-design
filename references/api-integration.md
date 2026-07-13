# API 集成说明

> 本文档定义后端 API 接口的调用方式，供 agent 在阶段二和阶段三执行上传操作时参考。

---

## 一、Nacos 服务发现

在调用任何后端接口之前，**必须先通过 Nacos 获取后端服务的 IP 地址**。

1. 调用已有的 Nacos skill 查询注册在 Nacos 上的后端服务
2. 获取服务 IP 地址后，拼接接口路径进行调用
3. 同一批次内的多次接口调用（用例上传 + 文件上传）可复用同一个 IP

---

## 二、用例上传接口

### 2.1 接口信息

| 项目 | 说明 |
|------|------|
| 方法 | POST |
| URL | `{nacos_ip}/api/v1/testcase/save` |
| Content-Type | application/json |

### 2.2 Request Body

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

### 2.3 Response

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

### 2.4 curl 命令模板

```bash
curl -X POST "{ip}/api/v1/testcase/save" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "{sessionId}",
    "cases": {cases_json_array}
  }'
```

> **注意**：将整个 cases JSON 数组嵌入 `-d` 参数中。JSON 中的双引号需要转义，或使用 `@文件路径` 方式从文件读取。

**从文件读取方式**（推荐，避免转义问题）：
```bash
# 先将 JSON 写入临时文件
echo '{json_body}' > "/tmp/test-case-design/{sessionId}/upload_payload.json"

# 从文件读取发送
curl -X POST "{ip}/api/v1/testcase/save" \
  -H "Content-Type: application/json" \
  -d @"/tmp/test-case-design/{sessionId}/upload_payload.json"
```

---

## 三、文件上传接口

### 3.1 接口信息

| 项目 | 说明 |
|------|------|
| 方法 | POST |
| URL | `{nacos_ip}/api/v1/file/upload` |
| Content-Type | multipart/form-data |

### 3.2 Query Parameters

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `type` | string | 是 | 上传类型：`CHECKLIST`（待澄清需求清单）、`REPORT`（测试用例评审报告） |
| `sessionId` | string | 是 | 用户提供的会话标识 |
| `batchNo` | string | 是 | 批次号 |

### 3.3 Form Data

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file` | file | 是 | 上传的 PDF 文件 |

### 3.4 Response

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

### 3.5 curl 命令模板

**上传待澄清需求清单**（如已生成）：
```bash
if [ -f "/tmp/test-case-design/{sessionId}/待澄清需求清单_{batchNo}.pdf" ]; then
  curl -X POST "{ip}/api/v1/file/upload?type=CHECKLIST&sessionId={sessionId}&batchNo={batchNo}" \
    -F "file=@/tmp/test-case-design/{sessionId}/待澄清需求清单_{batchNo}.pdf"
fi
```

**上传测试用例评审报告**：
```bash
curl -X POST "{ip}/api/v1/file/upload?type=REPORT&sessionId={sessionId}&batchNo={batchNo}" \
  -F "file=@/tmp/test-case-design/{sessionId}/测试用例评审报告_{batchNo}.pdf"
```

> **注意**：URL 参数中的特殊字符（如中文）需要 URL 编码。文件名中的中文在 `-F` 中通常可以正常处理，如有问题可先重命名为英文名再上传。

---

## 四、临时文件管理

### 4.1 目录结构

```
/tmp/test-case-design/{sessionId}/
├── requirements.pdf              # 阶段一下载的需求文档（PDF 格式）
├── requirements.docx             # 阶段一下载的需求文档（DOCX 格式，二选一）
├── upload_payload.json           # 阶段二用例上传的 JSON payload（可选）
├── 待澄清需求清单_{batchNo}.pdf  # 阶段三生成的待澄清需求清单（按需生成）
└── 测试用例评审报告_{batchNo}.pdf # 阶段三生成的测试用例评审报告
```

### 4.2 生命周期

| 阶段 | 操作 |
|------|------|
| 阶段一 | 准备环境（`uv pip install` 安装依赖），创建目录，下载 PDF 或 DOCX 到目录 |
| 阶段二 | 可选：写入 JSON payload 文件用于 curl 上传 |
| 阶段三 | 使用 reportlab 生成 PDF 文件（评审报告必生 + 待澄清清单按需），上传到后端 |
| 阶段四 | **统一清理**：`rm -rf /tmp/test-case-design/{sessionId}/` |

### 4.3 清理命令

```bash
rm -rf "/tmp/test-case-design/{sessionId}/"
```

> **原则**：所有临时文件在流程结束时必须清理，确保本地不残留 PDF、DOCX 等文件。

---

## 五、错误处理

### 5.1 用例上传失败

- 如果 `/api/v1/testcase/save` 返回 `success: false`，检查错误信息
- 常见原因：JSON 格式错误、必填字段缺失、sessionId 无效
- 重试策略：修复问题后重试一次，仍失败则向用户报告错误

### 5.2 文件上传失败

- 如果 `/api/v1/file/upload` 返回 `success: false`，检查错误信息
- 常见原因：文件不存在、文件路径错误、type 参数值不正确
- 重试策略：确认文件路径正确后重试一次

### 5.3 Nacos 服务发现失败

- 如果 Nacos skill 无法获取后端服务 IP，向用户报告无法连接后端服务
- 不要尝试使用硬编码的 IP 地址

### 5.4 部分成功处理

- 如果用例上传成功但某个文件上传失败，仍然清理临时文件
- 在总结中明确标注哪些步骤成功、哪些失败
