# 批量测试用例生成模式（Large Case Set Pattern）

当测试用例数量较多（>20条）时，使用本模式高效生成。避免在 Python 脚本中使用复杂的数据结构嵌套，改用函数式快速录入。

## 推荐模式：紧凑型 `tc()` 辅助函数

### Python 脚本骨架

```python
#!/usr/bin/env python3
"""生成测试用例 JSON"""
import json, os
from datetime import datetime

SESSION_ID = "session-xxx"
BATCH_NO = datetime.now().strftime("%Y%m%d%H%M%S%f")[:17]  # 17位
cases = []

def tc(code, title, ttype, module, submodule, priority, pre, steps, exp):
    """快速添加一条测试用例"""
    cases.append({
        "caseCode": code, "batchNo": BATCH_NO, "title": title,
        "type": ttype, "module": module, "subModule": submodule,
        "priority": priority, "preconditions": pre,
        "steps": steps, "expectedResults": exp
    })

# ========== 模块 A ==========
M = "模块名称"
tc("TC_CODE_001", "验证核心功能正常流程", 1, M, "子模块", 0,
   "1. 前置条件A\n2. 前置条件B",
   "1. 步骤1: 具体操作/输入值\n2. 步骤2: 具体操作/输入值\n3. 点击提交",
   "1. 步骤1预期\n2. 步骤2预期\n3. 步骤3预期")

# ... 更多用例 ...

# ========== 输出 ==========
output = {"sessionId": SESSION_ID, "cases": cases}
out_dir = f"/opt/data/tmp/test-case-design/{SESSION_ID}"
os.makedirs(out_dir, exist_ok=True)
out_path = f"{out_dir}/cases_payload.json"
with open(out_path, "w", encoding="utf-8") as f:
    json.dump(output, f, ensure_ascii=False, indent=2)
print(f"总用例数: {len(cases)}")
print(f"JSON 已保存: {out_path}")
```

### 设计原则

1. **`tc()` 函数为核心**：将所有字段编码为函数参数，一行一条用例，快速增删
2. **模块变量 `M = "..."`**：共享模块名避免重复字符串，便于批量调整
3. **`\\n` 显式换行**：Python f-string 中使用 `\\n`，最终 JSON 中为 `\n`
4. **JSON 文件 → 脚本上传**：先生成 `.json` 文件持久化，再使用 `upload_cases.py` 上传

### 对比：不推荐的模式

| 模式 | 缺点 | 推荐替代 |
|------|------|---------|
| 用嵌套数据结构定义用例 | 可读性差，语法错误难定位 | `tc()` 函数 |
| 在 terminal heredoc 中传 Python 含中文 | 数字替换/字节损坏 | write_file + terminal 执行 |
| 逐个打印用例到 stdout | 无法用好 json.dump | 写 .json 文件 + upload_cases.py |
| 用 for 循环遍历变量生成 | 逻辑复杂，调试困难 | 显式逐条调用 tc() |

### 上传方式

```bash
# 使用标准上传脚本（自动 Nacos 服务发现 + API 调用）
python3 scripts/upload_cases.py \
  --payload-file "/opt/data/tmp/test-case-design/{sessionId}/cases_payload.json"
```

### 上传失败重试

JSON 文件已持久化到本地，上传失败时**无需重新生成用例**，直接重新执行上传命令即可：

```bash
# 修复问题后（如调整环境变量、确认服务可达），直接重新上传
python3 scripts/upload_cases.py \
  --payload-file "/opt/data/tmp/test-case-design/{sessionId}/cases_payload.json"
```

这避免了 LLM 在重新生成用例时可能产生的幻觉（遗漏、修改、或凭空编造用例内容）。

### 适用场景

- 用例数 ≥ 20 条
- 涉及多个模块/子模块
- 需要精确控制每条用例的步骤和预期值
- 需要修改后重新生成（修改脚本 → 重新执行 → 重新上传）
