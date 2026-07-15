---
name: test-case-design
description: 这项技能用于后端驱动的全流程测试用例设计：从 PDF/DOCX 需求文档下载与解析、需求整理与确认、测试用例生成与 API 上传、评审文档生成与上传，到最终总结。涵盖功能测试、接口测试、AI Agent 测试、兼容性测试、UI 测试、联动测试、路由测试及多平台专项测试（移动端/小程序/H5/桌面/PC Web）。仅专注于编写测试用例，不涉及测试计划、测试策略或自动化脚本。
---

## 概述

本 skill 服务于后端驱动的测试用例生成全流程，通过 `sessionId` 串联多轮对话。

> **⚠️ Hermes Agent 环境前置要求**：如果运行在 Hermes Agent 下，**必须先加载 `hermes-env-pitfalls` skill**。该 skill 解决了已知的 `/tmp/` 写文件拒绝、数字自动替换、heredoc 字节损坏等问题。本 skill 已采用 `/opt/data/tmp/` 作为默认临时路径以兼容该环境。

**入口判断**：根据用户请求内容，自动识别当前处于哪个阶段：
- **阶段一（需求获取）**：用户请求中包含 PDF/DOCX 下载链接 + "理解并整理需求"/"下载"等关键词
- **阶段二（用例生成）**：用户请求中包含 "需求正确"/"请生成测试用例"/"确认"等关键词 + 已确认的需求内容

---

## 阶段一：需求获取

### 1.1 提取关键参数

从用户请求中提取：
- `docUrl`：需求文档的下载链接（通常在 "下载" 后面或 HTTP 链接格式）
- `sessionId`：会话标识（用户明确告知的 "sessionId 是 xxx"）
- `fileFormat`：根据 URL 后缀自动识别文件格式（`.pdf` 或 `.docx`），如无后缀以用户说明为准

### 1.2 环境准备

检查并安装依赖包（检查下所有uv的虚拟环境中是否已经安装了）：

```bash
# 检查 python3 和 uv 是否可用
python3 --version && uv --version

# 幂等安装依赖（已安装的包会自动跳过）
uv pip install pymupdf pymupdf4llm python-docx md2pdf
```

> **说明**：`pymupdf`和`pymupdf4llm`用于解析 PDF，`python-docx` 用于解析 DOCX，`md2pdf` 用于后续将 Markdown 转换为 PDF 文件。
>
> **⚠️ Hermes Agent 环境依赖提示**：`uv pip install` 安装到当前 venv（如 `/opt/data/.venv`）。后续执行 Python 脚本时：
> - **terminal** 方式：必须用 `/opt/data/.venv/bin/python3` 而非 `python3`（否则找不到 venv 中的包）
> - **execute_code** 方式：默认使用系统 Python（不继承 venv），如需 venv 包需改用 `terminal` + 全路径

### 1.3 下载需求文档

```bash
# 创建临时目录（如已存在则先清空）
rm -rf "/opt/data/tmp/test-case-design/{sessionId}/"
mkdir -p /opt/data/tmp/test-case-design/{sessionId}/

# 下载需求文档，根据 URL 后缀决定保存的文件名
# 如果 URL 以 .docx 结尾，保存为 requirements.docx，否则保存为 requirements.pdf
if [[ "{docUrl}" =~ \.docx$ ]]; then
  curl -L -o "/opt/data/tmp/test-case-design/{sessionId}/requirements.docx" "{docUrl}"
else
  curl -L -o "/opt/data/tmp/test-case-design/{sessionId}/requirements.pdf" "{docUrl}"
fi
```

> **注意**：如果 URL 无后缀或不匹配，默认按 PDF 处理。下载后检查文件是否存在及大小是否正常。
>
> **⚠️ 下载失败回退方案**：如果用户拒绝 `curl` + `terminal`（常见于 Hermes Agent），改用 Python `urllib.request.urlretrieve()` 通过 `execute_code` 或 `write_file` + `terminal` 方式下载：
>
> ```python
> import os, urllib.request
> from urllib.parse import quote
> save_path = f"/opt/data/tmp/test-case-design/{session_id}/requirements.docx"
> urllib.request.urlretrieve(quote(url, safe='/:?=&'), save_path)
> print(f"Downloaded {os.path.getsize(save_path)} bytes")
> ```
>
> 注意：URL 中的中文字符必须通过 `quote()` 编码。

### 1.4 解析需求文档并提取需求

根据文件格式选择对应的解析方式：

**方式 A：PDF 文件（`requirements.pdf`）**

```bash
# 使用 pymupdf 读取 PDF 内容
# 也可以配合pymupdf4llm使用，效果更好
python3 -c "
import fitz
doc = fitz.open('/opt/data/tmp/test-case-design/{sessionId}/requirements.pdf')
for page in doc:
    print(page.get_text())
"
```

**方式 B：DOCX 文件（`requirements.docx`）**

```bash
# 使用 python-docx 读取 DOCX 内容
python3 -c "
from docx import Document
doc = Document('/opt/data/tmp/test-case-design/{sessionId}/requirements.docx')
for para in doc.paragraphs:
    print(para.text)
# 同时提取表格内容
for table in doc.tables:
    for row in table.rows:
        print('\t'.join(cell.text for cell in row.cells))
"
```

解析后，基于文档内容**判断系统类型**以便后续阶段加载正确的测试能力文件：
   - 文档中是否涉及 API/接口定义 → 标记需要加载 `api-testing.md`
   - 文档中是否涉及 AI Agent/智能体 → 标记需要加载 `agent-testing.md`
   - 文档中涉及的前端平台类型（移动App/小程序/H5/桌面/PC Web）→ 标记需要加载对应平台文件
   - 未命中上述类型 → 默认加载 `functional-testing.md`

### 1.5 整理需求列表

将需求整理为以下结构化格式返回给用户确认：

```markdown
## 需求整理（来自：{文档文件名}）

### 一、功能需求清单
| 编号 | 功能模块 | 子模块 | 需求描述 | 优先级判断 |
|------|---------|--------|---------|-----------|
| REQ-001 | 环境管理 | 添加环境 | 验证添加环境输入非法字符时校验拦截 | 高 |
| REQ-002 | ... | ... | ... | ... |

### 二、非功能需求清单（如有）
| 编号 | 类型 | 需求描述 | 优先级判断 |
|------|------|---------|-----------|

### 三、约束条件（如有）
- 系统类型：PC Web 管理后台
- 涉及接口：是/否
- 涉及 AI Agent：是/否
- ...

### 四、待澄清问题
| 编号 | 问题 | 我的理解 |
|------|------|---------|
| Q-001 | ... | ... |

---
以上需求整理完毕，请确认或修改。确认后我将基于最终需求撰写测试用例。
```

### 1.6 等待用户确认

用户确认后的下一轮请求将进入阶段二。用户可能在确认时修改某些需求点，以用户最终确认的内容为准。

> **关于待澄清需求**：用户可能对部分 Q-xxx 问题进行了澄清，也可能部分未澄清，甚至可能完全没有澄清。请记录哪些已澄清、哪些未澄清，这些信息将在阶段二用例生成时使用（未澄清的需求对应用例会做标记）。

---

## 阶段二：用例生成与上传

### 2.1 提取参数与需求状态

从用户确认请求中提取：
- `sessionId`：会话标识
- 确认后的需求内容（用户可能已修改）
- **需求澄清状态**：记录阶段一中每个 Q-xxx 待澄清问题的处理结果
  - ✅ 已澄清：用户明确给出了答案，按澄清后的内容设计用例
  - ❓ 未澄清：用户未明确回答或跳过了该问题，对应需求的用例标题需添加标记

### 2.2 加载测试能力文件

根据阶段一中标记的系统类型，加载对应的能力文件。**始终加载** `references/templates/common-rules.md`（通用规则）。

**能力文件**（按系统类型选择，可叠加）：

| 系统类型 | 加载文件 |
|---------|---------|
| 涉及接口/API | `references/core-capabilities/api-testing.md` |
| 涉及 AI Agent/智能体 | `references/core-capabilities/agent-testing.md` + `references/core-capabilities/functional-testing.md` 中的"第一部分：测试用例设计方法"+"第二部分：测试用例质量标准" |
| 默认（功能测试） | `references/core-capabilities/functional-testing.md` |

**平台文件**（按前端平台叠加）：

| 前端平台 | 加载文件 |
|---------|---------|
| 移动端 App | `references/platform/mobile-app.md` |
| 小程序 | `references/platform/mini-program.md` |
| 移动 Web / H5 | `references/platform/mobile-web.md` |
| 桌面端 | `references/platform/desktop.md` |
| PC Web 端 | `references/platform/pc-web.md` |

> **说明**：接口测试通常不叠加平台文件。多个平台可同时叠加（如同一系统有 PC Web 和移动 App）。

### 2.3 生成测试用例

按加载的能力文件和平台文件中的设计方法生成测试用例：

1. **确定测试类型**：根据需求内容判断测试类型（功能测试 type=1、接口测试 type=2 等），类型定义见 `references/examples/format-spec.md`
2. **应用设计方法**：
   - 等价类划分法：为每个输入条件划分有效/无效等价类
   - 边界值分析法：测试上界、下界、临界值
   - 场景法：覆盖基本流和备选流
   - 错误推测法：基于经验补充异常场景
   - 因果图法/正交实验法：多条件组合场景
3. **覆盖平台专项**：按平台文件中的测试维度补充用例
4. **严格遵循通用规则**：
   - 测试步骤中**必须给出具体输入值/参数/操作对象**，不得使用描述性语言
   - 大量数据场景（>1000 字符、>1MB 文件）可用描述+明确参数
   - 预期结果与测试步骤一一对应
5. **处理待澄清需求**：根据 2.1 中记录的需求澄清状态区分处理
   - 对于用户已澄清的需求 → 用例标题按正常格式 `验证{功能点}...`
   - 对于用户**未澄清**的需求 → 用例标题以 `【待用户澄清需求】` 开头，如 `【待用户澄清需求】验证环境名称输入超长字符时的校验拦截`
   - 未澄清需求的用例仍然按常规方法设计（基于合理的默认推测），但标题标记让用户知晓哪些需要后续确认

### 2.4 用例自查

按以下规则加载检查清单进行自查：

| 测试类型 | 检查清单 |
|---------|---------|
| 接口测试 | `references/checklists/api-checklist.md` |
| Agent 测试 | `references/checklists/agent-checklist.md` + `references/checklists/common-checklist.md` 中的"一、功能测试检查清单" |
| Agent 测试 + 平台 | `references/checklists/agent-checklist.md` + `references/checklists/common-checklist.md` 中的"一、功能测试检查清单" + `references/checklists/{平台}-checklist.md` |
| 功能测试 + 平台 | `references/checklists/common-checklist.md` + `references/checklists/{平台}-checklist.md` |
| 功能测试（无平台） | `references/checklists/common-checklist.md` |

### 2.5 组装 API 格式的用例 JSON

按 `references/examples/format-spec.md` 中定义的 **API JSON 格式** 组装用例。

**生成 batchNo**：使用当前时间，格式 `yyyyMMddHHmmssSSS`（17位数字字符串），如 `20260709143025123`。

同一批次所有用例使用**相同的 batchNo**。

### 2.6 上传用例到后端

#### 步骤 1：通过 Nacos 获取后端服务 IP

调用已有的 Nacos skill，获取注册在 Nacos 上的后端服务 IP，port，服务名等信息。

#### 步骤 2：调用用例上传接口

```bash
curl -X POST "{ip}:{port}/{服务名}/api/v1/testcase/save" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "{sessionId}",
    "cases": [
      {
        "caseCode": "TC_ENV_MGM_001",
        "batchNo": "{batchNo}",
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
  }'
```

**接口说明**：
- 方法：POST
- Content-Type：application/json
- RequestBody 字段：
  - `sessionId`（string）：会话标识
  - `cases`（array）：用例列表，每项包含 caseCode、batchNo、title、type、module、subModule（可选）、priority、preconditions、steps、expectedResults
- 响应格式：`{ "success": true/false, "code": "00000", "message": "", "data": true/false }`

---

## 阶段三：文档生成与上传

### 3.1 生成「待澄清需求清单」

在生成前，判断是否存在待澄清需求：

- 对阶段一用户没有回答的、遗留的未澄清问题，用例设计过程中新发现的模糊点， 按 `references/templates/clarification-checklist.md` 模板生成 PDF 文档。如果需求已经非常清晰，也请在文件中注明。

**必须包含**：
- 基本信息（batchNo、sessionId、需求来源、整理时间）
- 已确认需求列表（从阶段一用户确认的需求中提取）
- 待澄清需求（用例设计过程中发现的需求模糊点、未明确边界、遗漏场景等）
- 需求覆盖度评估

生成方式见下方 **3.3 使用 md2pdf 生成 PDF 文件**（与评审报告共用 PDF 生成逻辑）。

### 3.2 生成「测试用例评审报告」

对生成的用例进行自我评审，按 `references/templates/review-report.md` 模板内容生成 PDF 文档。


**必须包含**：
- 基本信息（batchNo、sessionId、评审时间）
- 用例概况（总数、按类型/优先级/模块分布统计）
- 覆盖度评审（功能覆盖、场景覆盖、测试设计方法应用）
- 质量评审（步骤可执行性、预期结果可验证性、数据具体性）
- 问题与改进建议
- 评审结论

生成方式见下方 **3.3 使用 md2pdf 生成 PDF 文件**。

### 3.3 使用 md2pdf 生成 PDF 文件

本步骤为 3.1 和 3.2 中需要生成 PDF 时的具体操作指引。

使用 **md2pdf** 替代 reportlab 生成 PDF（md2pdf 将 Markdown 转为 HTML 后由 weasyprint 渲染为 PDF，无需手动处理中文字体注册等复杂问题）。

> **先决条件**：md2pdf 和 weasyprint 已在环境预装（详见 1.2 节）。

#### 3.3.1 工作流程

整体流程分两步：

```
生成 Markdown 内容（Python 脚本拼接字符串）
    ↓
调用 md2pdf 转换为 PDF（文件模式或 raw 模式）
```

**步骤 A：编写 Python 脚本，按模板内容拼接 Markdown 字符串**

按 `references/templates/` 中的对应模板结构，用 Python 拼接完整的 Markdown 字符串，包括：
- 标题（`#`、`##`、`###`）
- 段落（空行分隔）
- 表格（标准 Markdown 表格语法）
- 列表（`- ` 无序、`1. ` 有序）
- 代码块（` ``` ` 包裹）
- 引用（`> `）
- 分隔线（`---`）

**步骤 B：调用 md2pdf 转换为 PDF**

```python
from pathlib import Path
from md2pdf.core import md2pdf

# 方式一：文件模式（将 Markdown 写入 .md 文件后再转换）
with open('/opt/data/tmp/test-case-design/{sessionId}/文档.md', 'w', encoding='utf-8') as f:
    f.write(markdown_content)

md2pdf(
    md=Path('/opt/data/tmp/test-case-design/{sessionId}/文档.md'),
    pdf=Path('/opt/data/tmp/test-case-design/{sessionId}/输出文件.pdf'),
    css=Path('references/templates/pdf-style.css')  # 可选，默认样式
)

# 方式二：raw 模式（免中间 .md 文件，推荐）
md2pdf(
    raw=markdown_content,                          # 直接传入 Markdown 字符串
    pdf=Path('/opt/data/tmp/test-case-design/{sessionId}/输出文件.pdf'),
    css=Path('references/templates/pdf-style.css')  # 可选
)
```

> **推荐方式二（raw 模式）**：减少一次文件写操作，且 Markdown 字符串中无需处理文件编码问题。

#### 3.3.2 自定义样式

项目提供了一个默认 CSS 样式文件 `references/templates/pdf-style.css`，已配置好中文排版、表格样式、页眉页脚等。如需自定义样式，可修改该 CSS 文件或在调用时传入自己的 CSS 路径。

如果调用时不传 `css` 参数，md2pdf 会使用默认的无样式渲染（仍可正常显示中文，但表格无边框、排版较朴素）。**建议始终传入样式文件**。

#### 3.3.3 Markdown 内容生成要点

| 要点 | 说明 |
|------|------|
| **中文** | 直接写入字符串即可，无需任何额外配置 |
| **表格** | 使用标准 Markdown 表格 `\| col1 \| col2 \|`，首行为表头 |
| **换行** | 段落间空一行，表格内不要用复杂换行 |
| **特殊字符** | `\|` 在表格内需转义为 `\\|`，`*` 和 `_` 可能会被解析为斜体/加粗标记 |
| **代码块** | 使用三个反引号包裹，支持指定语言 |
| **变量替换** | 在 Python 中用 f-string 或 `.format()` 将 `{sessionId}`、`{batchNo}` 等参数填入 |

#### 3.3.4 生成的 PDF 文件

| 文档类型 | 文件名 |
|---------|-------|
| 待澄清需求清单 | `待澄清需求清单_{batchNo}.pdf` |
| 测试用例评审报告 | `测试用例评审报告_{batchNo}.pdf` |

### 3.4 上传文档到后端

#### 步骤 1：通过 Nacos 获取后端服务 IP（如阶段二已获取可复用）

#### 步骤 2：上传待澄清需求清单

```bash
if [ -f "/opt/data/tmp/test-case-design/{sessionId}/待澄清需求清单_{batchNo}.pdf" ]; then
  curl -X POST "{ip}/api/v1/file/upload?type=CHECKLIST&sessionId={sessionId}&batchNo={batchNo}" \
    -F "file=@/opt/data/tmp/test-case-design/{sessionId}/待澄清需求清单_{batchNo}.pdf"
fi
```

#### 步骤 3：上传测试用例评审报告

```bash
curl -X POST "{ip}/api/v1/file/upload?type=REPORT&sessionId={sessionId}&batchNo={batchNo}" \
  -F "file=@/opt/data/tmp/test-case-design/{sessionId}/测试用例评审报告_{batchNo}.pdf"
```

**接口说明**：
- 方法：POST
- Content-Type：multipart/form-data
- Query 参数：`type`（CHECKLIST 或 REPORT）、`sessionId`、`batchNo`
- Form-data：`file`（文件字段）
- 响应格式：`{ "success": true/false, "code": "", "message": "", "data": { "sessionId": "", "fileName": "", "relativePath": "", "url": "", "type": "" } }`

> **重要**：记录上传接口返回的 `data.url`，后续总结中需要展示给用户。

---

## 阶段四：清理与总结

### 4.1 清理临时文件

```bash
rm -rf "/opt/data/tmp/test-case-design/{sessionId}/"
```

> **目标**：确保本地不残留 PDF、DOCX 和 MD 文件，保持本地环境干净。

### 4.2 返回总结

向用户输出以下格式的总结：

```markdown
## 测试用例生成完成

| 项目 | 详情 |
|------|------|
| 批次号 | {batchNo} |
| Session ID | {sessionId} |
| 用例总数 | X 条 |
| 用例上传 | ✅ 成功 / ❌ 失败（{原因}） |

### 用例分布
| 维度 | 分布 |
|------|------|
| 按类型 | 功能测试(X), 接口测试(X), ... |
| 按优先级 | 高(X), 中(X), 低(X) |
| 按模块 | 模块A(X), 模块B(X), ... |

### 文档上传
| 文档 | 状态 | 链接 |
|------|------|------|
| 待澄清需求清单 | ✅ 已上传 | [查看]({url}) |
| 测试用例评审报告 | ✅ 已上传 | [查看]({url}) |
```

---

## 能力边界

✅ 可生成：功能测试、接口测试、AI Agent 测试（含 Agent 安全与边界）、平台专项测试（移动App/小程序/H5/桌面/PC Web）、兼容性测试、UI 测试、联动测试、路由测试
❌ 不可生成：测试方案、测试策略、测试计划、渗透测试执行、漏洞扫描、性能压测（并发/压力/负载）、自动化脚本

---

## 关键规则速查

| 规则 | 说明 | 参考文件 |
|------|------|---------|
| 步骤必须具体 | 测试步骤中必须给出具体输入值/参数，不得使用描述性语言 | `references/templates/common-rules.md` 第零节 |
| 编号规则 | caseCode 格式 `[平台]_[模块]_[维度]_[序号]` | `references/templates/common-rules.md` 第三节 |
| JSON 格式 | 按 API JSON Schema 输出，priority 为 int(0/1/2)，type 为 int(1-9) | `references/examples/format-spec.md` |
| API 调用 | 先通过 Nacos 获取 IP，再调用接口 | `references/api-integration.md` |
| 临时文件 | 统一放在 `/opt/data/tmp/test-case-design/{sessionId}/`，流程结束后清理 | — |
| 文件解析 | PDF 用 `pymupdf`，DOCX 用 `python-docx`，用 `uv pip install` 幂等安装 | `references/core-capabilities/` |
| 自查清单 | 生成用例后必须按对应检查清单自查 | `references/checklists/*.md` |

## 参考文件索引

### 能力文件
| 文件 | 用途 |
|------|------|
| `references/core-capabilities/functional-testing.md` | 功能测试设计方法、质量标准、联动/路由/UI/交互/动效测试 |
| `references/core-capabilities/api-testing.md` | 接口测试维度（功能、状态码、数据校验、认证授权、安全、错误处理等） |
| `references/core-capabilities/agent-testing.md` | AI Agent 测试维度（任务完成度、工具与记忆、安全与边界、性能、内容质量等） |

### 平台文件
| 文件 | 用途 |
|------|------|
| `references/platform/pc-web.md` | PC Web 端专项测试 |
| `references/platform/mobile-app.md` | 移动端 App 专项测试 |
| `references/platform/mobile-web.md` | 移动 Web / H5 专项测试 |
| `references/platform/mini-program.md` | 小程序专项测试 |
| `references/platform/desktop.md` | 桌面端专项测试 |

### 模板与规范
| 文件 | 用途 |
|------|------|
| `references/templates/common-rules.md` | 通用规则（编号、优先级、测试类型分类） |
| `references/templates/pdf-style.css` | PDF 默认样式（配合 md2pdf 使用） |
| `references/examples/format-spec.md` | 输出格式规范（API JSON + Markdown 表格） |
| `references/templates/clarification-checklist.md` | 待澄清需求清单模板 |
| `references/templates/review-report.md` | 测试用例评审报告模板 |
| `references/api-integration.md` | API 集成说明 |

### 批量用例生成 (≥20 条场景)
| 文件 | 用途 |
|------|------|
| `references/bulk-case-generation-pattern.md` | 大批量用例的代码生成模式：`tc()` 紧凑函数 + JSON 文件 + curl 上传 |
| | 当预期用例数 ≥20 时应优先参考此模式，避免在 terminal heredoc 中传递大量复杂数据 |

### 检查清单
| 文件 | 用途 |
|------|------|
| `references/checklists/common-checklist.md` | 通用检查清单（功能+联动+路由+UI） |
| `references/checklists/api-checklist.md` | 接口测试检查清单 |
| `references/checklists/agent-checklist.md` | Agent 测试检查清单 |
| `references/checklists/pc-web-checklist.md` | PC Web 检查清单 |
| `references/checklists/mobile-app-checklist.md` | 移动 App 检查清单 |
| `references/checklists/mobile-web-checklist.md` | 移动 Web 检查清单 |
| `references/checklists/mini-program-checklist.md` | 小程序检查清单 |
| `references/checklists/desktop-checklist.md` | 桌面端检查清单 |
