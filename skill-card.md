## Description

这项技能帮助用户完成后端驱动的全流程测试用例设计：从 PDF/DOCX 需求文档下载与解析、需求整理与确认、测试用例生成与 API 上传、评审文档（待澄清需求清单 + 测试用例评审报告）生成与上传，到最终总结。涵盖功能测试、接口测试、AI Agent 测试、兼容性测试、UI 测试、联动测试、路由测试及多平台专项测试（移动端/小程序/H5/桌面/PC Web）。仅专注于编写测试用例，不涉及测试计划、测试策略或自动化脚本。

This skill is ready for commercial/non-commercial use.

## Publisher

[cassianran](https://clawhub.ai/user/cassianran)

### License/Terms of Use

MIT-0

## Use Case

Developers, QA engineers, and product teams use this Chinese-language skill for backend-driven end-to-end test case design: downloading and parsing requirement PDFs, extracting and confirming requirements, generating test cases in API-aligned JSON format, uploading test cases to backend APIs, generating review documents (clarification checklist + review report), uploading documents, and providing a final summary. Supports functional testing, API testing, AI Agent testing, and multi-platform scenarios (Mobile App, Mini Program, H5, Desktop, PC Web). Only for test-case design — not test strategy, penetration testing, load testing, or automation script generation.

### Deployment Geography for Use

Global

## Known Risks and Mitigations

- **Risk:** Generated or example test cases may include sample usernames, passwords, bearer tokens, or API keys.
  **Mitigation:** Replace sample credentials and tokens with placeholders before sharing or storing generated test cases.
- **Risk:** The skill downloads requirement PDFs/DOCX and generates temporary files locally.
  **Mitigation:** All temporary files are stored under `/opt/data/tmp/test-case-design/{sessionId}/` and cleaned up at the end of the workflow.
- **Risk:** The skill uses Nacos credentials via environment variables to call backend APIs.
  **Mitigation:** Scripts mask sensitive values in all output; SKILL.md includes mandatory security rules prohibiting exposure of environment variable values.
- **Risk:** The skill is focused on test-case writing and does not create test plans, security scans, load tests, or automation scripts.
  **Mitigation:** Use separate planning, security, performance, or automation workflows for those activities.

## Reference(s)

- [Skill definition](SKILL.md)
- [测试用例通用规则](references/templates/common-rules.md)
- [测试用例输出格式规范](references/examples/format-spec.md)
- [API 集成说明](references/api-integration.md)
- [功能测试](references/core-capabilities/functional-testing.md)
- [接口测试](references/core-capabilities/api-testing.md)
- [AI Agent 测试](references/core-capabilities/agent-testing.md)
- [通用测试检查清单](references/checklists/common-checklist.md)
- [待澄清需求清单模板](references/templates/clarification-checklist.md)
- [测试用例评审报告模板](references/templates/review-report.md)
- [下载脚本](scripts/download_requirements.py)
- [Nacos 服务发现](scripts/discover_and_call.py)
- [用例上传脚本](scripts/upload_cases.py)
- [文件上传脚本](scripts/upload_file.py)
- [ClawHub skill page](https://clawhub.ai/cassianran/test-case-design)

## Skill Output

**Output Type(s):** [Text, Markdown, JSON, API]

**Output Format:** [API JSON] (备选: Markdown tables) + [MD Documents] (待澄清需求清单、测试用例评审报告)

**Output Parameters:** [1D]

**Other Properties Related to Output:** [Uploads test cases via REST API, uploads review documents via multipart form upload, cleans up local temp files after completion.]

## Skill Version(s)

1.2.0 (forked from 1.0.8 by cassianran)

## Ethical Considerations

Users should evaluate whether this skill is appropriate for their environment, review any generated or modified files before relying on them, and apply their organization's safety, security, and compliance requirements before deployment.
