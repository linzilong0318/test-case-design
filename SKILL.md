---
name: test-case-design
description: 这项技能用于后端驱动的全流程测试用例设计：从 PDF/DOCX 需求文档下载与解析、需求整理与确认、测试用例生成与 API 上传、评审文档生成与上传，到最终总结。涵盖功能测试、接口测试、AI Agent 测试、兼容性测试、UI 测试、联动测试、路由测试及多平台专项测试（移动端/小程序/H5/桌面/PC Web）。仅专注于编写测试用例，不涉及测试计划、测试策略或自动化脚本。
---

## 概述

本 skill 服务于后端驱动的测试用例生成全流程，通过 `sessionId` 串联多轮对话。

> **⚠️ Hermes Agent 环境前置要求**：如果运行在 Hermes Agent 下，**必须先加载 `hermes-env-pitfalls` skill**。该 skill 解决了已知的 `/tmp/` 写文件拒绝、数字自动替换、heredoc 字节损坏等问题。本 skill 已采用 `/opt/data/tmp/` 作为默认临时路径以兼容该环境。

---

## ⚠️ 安全规则（最高优先级，必须遵守）

> **这些规则适用于整个 skill 执行过程，任何情况下不得违反。**

1. **禁止输出环境变量值**：任何时候都**不得**向用户展示以下环境变量的值：
   - `NACOS_USERNAME`、`NACOS_PASSWORD`、`NACOS_SERVER_ADDRESSES`、`NACOS_NAMESPACE`
   - 任何包含 `TOKEN`、`PASSWORD`、`SECRET`、`KEY`、`CREDENTIAL` 的环境变量
   - 如调试需要提及，仅可说"已从环境变量读取"，不得展示具体值

2. **脚本输出过滤**：执行 Python/Shell 脚本后，如果脚本输出中包含敏感变量值，**必须在展示给用户前过滤或截断**（仅保留前 4 个字符 + `***`）。

3. **curl 命令脱敏**：如果必须展示 curl 或 API 调用命令，将 Token/密码/认证信息替换为 `{REDACTED}`。

4. **日志脱敏**：Python 脚本中打印环境变量时，仅打印变量名，不打印值；如需打印值做调试，仅打印前 4 字符 + `***`。

5. **API 响应过滤**：后端接口返回的响应中如包含 Token、密码、密钥等敏感字段，展示给用户前必须脱敏。

6. **禁止输出业务ID**：例如`sessionId` `project_id` `user_id` `business_user_id`不允许展示给用户


---

## 临时文件管理规范

> **所有临时文件必须统一存放在 `/opt/data/tmp/test-case-design/{sessionId}/` 目录下，流程结束后统一清理。**

### 目录结构

```
/opt/data/tmp/test-case-design/{sessionId}/
├── requirements.pdf                  # 阶段一下载的需求文档
├── requirements.docx                 # 阶段一下载的需求文档（DOCX 格式时）
├── cases_payload.json                # 阶段二生成的用例 JSON（持久化，上传失败可重试）
├── 待澄清需求清单_{batchNo}.pdf       # 阶段三生成的待澄清需求清单
├── 测试用例评审报告_{batchNo}.pdf     # 阶段三生成的测试用例评审报告
└── scripts/                          # 临时 Python 脚本（如有，必须放在此子目录下）
```

### 关键规则

- **严禁**在 `/opt/data/tmp/test-case-design/{sessionId}/` **以外的位置**创建临时 Python 脚本
- 如确需创建临时脚本，必须写入 `{sessionId}/scripts/` 子目录
- 阶段四执行 `rm -rf "/opt/data/tmp/test-case-design/{sessionId}/"` 统一清理，确保不残留任何文件

---

## 入口判断

根据用户请求内容，自动识别当前处于哪个阶段：
- **阶段一（需求获取）**：用户请求中包含 PDF/DOCX 下载链接 + "理解并整理需求"/"下载"等关键词
- **阶段二（用例生成）**：用户请求中包含 "需求正确"/"请生成测试用例"/"确认"等关键词 + 已确认的需求内容

---

## 阶段一：需求获取

### 1.1 提取关键参数

从用户请求中提取：
- `docUrl`：需求文档的下载链接（通常在 "下载" 后面或 HTTP 链接格式）
- `sessionId`：会话标识（用户明确告知的 "sessionId 是 xxx"）
- `fileFormat`：根据 URL 后缀自动识别文件格式（`.pdf` 或 `.docx`），如无后缀以用户说明为准

### 1.2 下载需求文档

**terminal工具执行标准下载脚本**：

```bash
# 创建临时目录
rm -rf "/opt/data/tmp/test-case-design/{sessionId}/"
mkdir -p "/opt/data/tmp/test-case-design/{sessionId}/"

# 确定输出文件名（根据 URL 后缀）
# 如果 URL 以 .docx 结尾，输出文件名为 requirements.docx，否则为 requirements.pdf
OUTPUT_FILE="/opt/data/tmp/test-case-design/{sessionId}/requirements.{pdf或docx}"

# 执行下载脚本（内置 3 次重试 + 指数退避，无需额外处理）
/opt/data/.venv/bin/python3 scripts/download_requirements.py \
  --url "{docUrl}" \
  --output "$OUTPUT_FILE" \
  --session-id "{sessionId}"
```

### 1.4 解析需求文档并提取需求

根据文件格式选择对应的解析方式：

**方式 A：PDF 文件（`requirements.pdf`）**

```bash
/opt/data/.venv/bin/python3 -c "
import fitz
doc = fitz.open('/opt/data/tmp/test-case-design/{sessionId}/requirements.pdf')
for page in doc:
    print(page.get_text())
"
```

**方式 B：DOCX 文件（`requirements.docx`）**

```bash
/opt/data/.venv/bin/python3 -c "
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

### 1.5 整理需求列表（不允许跳过这一步）

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

### 2.5 组装 API 格式的用例 JSON 并持久化

按 `references/examples/format-spec.md` 中定义的 **API JSON 格式** 组装用例。

**生成 batchNo**：使用当前时间，格式 `yyyyMMddHHmmssSSS`（17位数字字符串），如 `20260709143025123`。同一批次所有用例使用**相同的 batchNo**。

**⚠️ 必须持久化到本地文件:** `/opt/data/tmp/test-case-design/{SESSION_ID}/cases_payload.json`：

### 2.6 上传用例到后端

**使用标准上传脚本**（自动完成 Nacos 服务发现 + API 调用）：

```bash
python3 scripts/upload_cases.py \
  --payload-file "/opt/data/tmp/test-case-design/{sessionId}/cases_payload.json"
```

> **脚本说明**：`scripts/upload_cases.py` 自动完成：
> - 从环境变量读取 Nacos 配置（`NACOS_SERVER_ADDRESSES`、`NACOS_NAMESPACE`、`NACOS_USERNAME`、`NACOS_PASSWORD`、`BACKEND_SERVICE_NAME`）
> - 通过 Nacos 发现健康的后端服务实例
> - 读取本地 JSON 文件并 POST 到 `/api/v1/testcase/save`
> - 打印上传结果
>
> **上传失败处理**：如果上传失败，检查错误信息后可直接重新执行上述命令（JSON 文件已持久化），无需重新生成用例。

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
调用 md2pdf 转换为 PDF（推荐 raw 模式）
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

**注意**：涉及时间，一律转成东八区的时间

**步骤 B：调用 md2pdf 转换为 PDF**

```python
from pathlib import Path
from md2pdf.core import md2pdf

# 推荐：raw 模式（免中间 .md 文件）
md2pdf(
    raw=markdown_content,                          # 直接传入 Markdown 字符串
    pdf=Path('/opt/data/tmp/test-case-design/{sessionId}/输出文件.pdf'),
    css=Path('references/templates/pdf-style.css')  # 可选，默认样式
)
```

> **PDF 文件保存路径**：必须保存在 `/opt/data/tmp/test-case-design/{sessionId}/` 目录下，确保阶段四能被统一清理。

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

**使用标准上传脚本**（自动完成 Nacos 服务发现 + 文件上传）：

```bash
# 上传待澄清需求清单（如已生成）
if [ -f "/opt/data/tmp/test-case-design/{sessionId}/待澄清需求清单_{batchNo}.pdf" ]; then
  python3 scripts/upload_file.py \
    --file "/opt/data/tmp/test-case-design/{sessionId}/待澄清需求清单_{batchNo}.pdf" \
    --session-id "{sessionId}" \
    --batch-no "{batchNo}" \
    --type CHECKLIST
fi

# 上传测试用例评审报告
python3 scripts/upload_file.py \
  --file "/opt/data/tmp/test-case-design/{sessionId}/测试用例评审报告_{batchNo}.pdf" \
  --session-id "{sessionId}" \
  --batch-no "{batchNo}" \
  --type REPORT
```

> **脚本说明**：`scripts/upload_file.py` 自动完成：
> - 从环境变量读取 Nacos 配置
> - 通过 Nacos 发现健康的后端服务实例
> - multipart/form-data 上传 PDF 文件到 `/api/v1/file/upload`
> - 打印上传结果（含文件访问链接）

> **重要**：记录上传接口返回的 `data.url`，后续总结中需要展示给用户。

---

## 阶段四：清理与总结

### 4.1 清理临时文件

```bash
rm -rf "/opt/data/tmp/test-case-design/{sessionId}/"
```

> **目标**：统一清理 `/opt/data/tmp/test-case-design/{sessionId}/` 整个目录，确保不残留 PDF、DOCX、JSON 和临时脚本文件，保持本地环境干净。

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
| 安全规则 | 禁止输出 NACOS_* 等环境变量值，敏感字段必须脱敏 | 见顶部 ⚠️ 安全规则 |
| 临时文件 | 所有临时文件统一放在 `/opt/data/tmp/test-case-design/{sessionId}/`，流程结束统一清理 | 见顶部 临时文件管理规范 |
| 步骤必须具体 | 测试步骤中必须给出具体输入值/参数，不得使用描述性语言 | `references/templates/common-rules.md` 第零节 |
| 编号规则 | caseCode 格式 `[平台]_[模块]_[维度]_[序号]` | `references/templates/common-rules.md` 第三节 |
| JSON 格式 | 按 API JSON Schema 输出，priority 为 int(0/1/2)，type 为 int(1-9) | `references/examples/format-spec.md` |
| 用例持久化 | 生成后先写入 `cases_payload.json`，再上传；失败可直接重试 | `scripts/upload_cases.py` |
| Nacos 服务发现 | 已整合到 `scripts/discover_and_call.py`，上传脚本自动调用 | `scripts/` |
| 文件解析 | PDF 用 `pymupdf`，DOCX 用 `python-docx`，用 `uv pip install` 幂等安装 | `references/core-capabilities/` |
| 自查清单 | 生成用例后必须按对应检查清单自查 | `references/checklists/*.md` |
| 下载脚本 | 使用 `scripts/download_requirements.py`（内置重试+URL编码） | `scripts/download_requirements.py` |

## 参考文件索引

### 脚本文件
| 文件 | 用途 |
|------|------|
| `scripts/download_requirements.py` | 统一的需求文档下载脚本（内置重试 + 中文 URL 编码） |
| `scripts/discover_and_call.py` | Nacos 服务发现 + REST API 调用通用模块（可导入复用） |
| `scripts/upload_cases.py` | 用例上传脚本（从 JSON 文件读取，自动发现服务） |
| `scripts/upload_file.py` | 文件上传脚本（multipart/form-data，自动发现服务） |

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
