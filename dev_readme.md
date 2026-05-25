# aminer-open-skill - 开发者指南

> 本文档面向参与 aminer-open-skill 开发的贡献者，提供环境搭建、模块说明、开发规范与调试方法。

---

## 1. 项目简介

aminer-open-skill 将 AMiner 开放平台的学术数据能力封装为可插拔的 AI Agent Skill，目前包含四个独立 Skill：

| Skill | 版本 | 定位 |
|---|---|---|
| `aminer-academic-search` | v1.1.1 | 全量版，27 个 API + 5 个组合工作流 |
| `aminer-free-academic` | v1.1.1 | 免费版，7 个免费 API，轻量初筛 |
| `aminer-daily-paper` | v1.1.2 | 个性化论文推荐，基于 rec5 API |
| `aminer-deep-search` | v1.0.0 | LLM 驱动的 ReAct 综述文献收集 |

---

## 2. 环境准备

### 2.1 必需环境

- Python 3.10+
- AMiner API Token（从 [AMiner 控制台](https://open.aminer.cn/open/board?tab=control) 获取）

### 2.2 环境变量

```bash
# 必需 - AMiner API 访问凭证
export AMINER_API_KEY="<YOUR_TOKEN>"

# 仅 aminer-deep-search 需要 - LLM 配置
export llm.api_key="<LLM_TOKEN>"
export llm.model="<MODEL_NAME>"
export llm.base_url="<LLM_BASE_URL>"  # 可选
```

### 2.3 依赖安装

```bash
# aminer-daily-paper
cd skills/aminer-daily-paper
pip install -r requirements.txt

# aminer-deep-search
cd skills/aminer-deep-search
pip install -r requirements.txt

# aminer-academic-search（可选 Python 客户端）
cd skills/aminer-academic-search
pip install -r requirements.txt  # 如有
```

> `aminer-free-academic` 无额外依赖，直接通过 curl 调用。

---

## 3. 模块架构

```
skills/
├── aminer-academic-search/       # 全量学术搜索 Skill
│   ├── SKILL.md                  # Skill 元数据 + 行为定义
│   ├── scripts/
│   │   └── aminer_client.py      # Python 封装客户端（27 个 API + 7 个工作流）
│   ├── references/
│   │   └── api-catalog.md        # API 参数与返回字段速查
│   └── evals/
│       └── evals.json            # 评测用例
│
├── aminer-free-academic/         # 免费学术搜索 Skill
│   ├── SKILL.md                  # 免费版 Skill 定义
│   └── references/
│       └── api-catalog.md        # 免费 API 速查
│
├── aminer-daily-paper/           # 论文推荐 Skill
│   ├── SKILL.md                  # 推荐 Skill 定义
│   ├── scripts/
│   │   ├── handle_trigger.py     # 入口：解析用户输入 → 调用 pipeline
│   │   ├── run_pipeline.py       # 核心流程：构建请求 → 调用 rec5 API → 格式化输出
│   │   ├── rec5_api.py           # AMiner rec5 推荐接口封装
│   │   ├── common.py             # 公共工具函数
│   │   └── constants.py          # 常量定义
│   ├── prompts/
│   │   └── enrich.md             # 中文富化 prompt 模板
│   └── commands/
│       └── aminer-dp.md          # Slash 命令定义
│
└── aminer-deep-search/           # 深度文献收集 Skill
    ├── SKILL.md                  # 深度收集 Skill 定义
    ├── commands/
    │   └── aminer-deep-search.md # Slash 命令定义
    ├── react_agent.py            # ReAct 主循环：LLM 控制工具调用
    ├── api_client.py             # OpenAI 兼容 LLM 客户端（含 fallback）
    ├── prompt.py                 # 系统提示词
    ├── search.py                 # AMiner 关键词搜索封装
    ├── citation.py               # 引用关系扩展
    ├── paper_set.py              # 论文去重集合管理
    └── _utils.py                 # 内部工具
```

---

## 4. 各 Skill 开发要点

### 4.1 aminer-academic-search

**核心文件**: `scripts/aminer_client.py`

- 封装了全部 27 个 API，按论文、学者、机构、期刊、专利五大类组织
- 内置 7 个组合工作流（`workflow_*`），使用 `ThreadPoolExecutor` 实现并行请求
- 内置费用追踪（`_track_cost`），支持 `--dry-run` 预览调用链和预估费用
- 请求失败自动重试（指数退避），支持 408/429/5xx 等可重试状态码

**调用方式**:

```bash
python scripts/aminer_client.py --token $AMINER_API_KEY \
  --action scholar_profile --name "Andrew Ng"
```

### 4.2 aminer-free-academic

**核心文件**: `SKILL.md`

- 纯声明式 Skill，无代码文件
- 通过 curl 直接调用 7 个免费 API
- 设计原则：免费发现 → 判断是否需要升级到付费 API

### 4.3 aminer-daily-paper

**核心文件**: `scripts/handle_trigger.py`

- 智能输入解析：支持结构化命令（`/aminer-dp topics: xxx`）和自然语言
- 自动推断学者信息（如"我是唐杰，清华大学" → `scholar_name=Jie Tang, scholar_org=Tsinghua University`）
- 主题提取：从自由文本中去除停用词后提取研究主题
- 参数校验：`aminer_author_id` 格式验证、`papers_file` 路径安全检查

**调用流程**:

```
handle_trigger.py
  → parse_trigger_text()       # 解析输入
  → _normalize_interface_payload()  # 校验和规范化
  → _run_pipeline()            # 调用 run_pipeline.py（子进程）
  → 返回 JSON（含 reply_text）
```

### 4.4 aminer-deep-search

**核心文件**: `react_agent.py`

- ReAct 循环：LLM 决策 → 工具调用 → 结果反馈 → 下一轮
- 四个工具：`search`（关键词搜索）、`get_reference`（引用扩展）、`add_to_paper_set`（去重收集）、`END`（终止）
- 防护机制：最大轮次（`--max-rounds 50`）、工具调用预算（`--max-tool-calls 20`）、超时控制

**调用方式**:

```bash
python react_agent.py --topic "multimodal agents" \
  --max-rounds 30 --max-tool-calls 15 --target-size 200
```

---

## 5. API 调用规范

### 5.1 统一请求头

```bash
Authorization: ${AMINER_API_KEY}
X-Platform: openclaw
Content-Type: application/json;charset=utf-8  # POST 请求
```

### 5.2 费用控制

| 级别 | API 示例 | 单价 |
|---|---|---|
| 免费 | `paper_search`, `person_search`, `patent_search` | ¥0 |
| 低成本 | `paper_search_pro`, `paper_detail` | ¥0.01 |
| 中成本 | `paper_qa_search`, `paper_relation` | ¥0.05 ~ ¥0.10 |
| 高成本 | `person_detail`, `person_paper_relation` | ¥0.50 ~ ¥1.50 |

**规则**: 高成本工作流（预估 >= ¥5）执行前必须向用户确认。

### 5.3 实体 URL 模板

| 类型 | URL 模板 |
|---|---|
| 论文 | `https://www.aminer.cn/pub/{paper_id}` |
| 学者 | `https://www.aminer.cn/profile/{scholar_id}` |
| 专利 | `https://www.aminer.cn/patent/{patent_id}` |
| 期刊 | `https://www.aminer.cn/open/journal/detail/{journal_id}` |

---

## 6. 开发规范

### 6.1 代码风格

- 遵循 PEP 8
- 使用 type hints
- 错误信息使用中文（面向用户）或英文（面向开发者）

### 6.2 安全要求

- **禁止**在代码中硬编码 API Key、Token 或 Base URL
- **禁止**在日志或输出中暴露 `AMINER_API_KEY` 和 `llm.api_key`
- `papers_file` 等文件路径参数必须限制在 Skill 目录内

### 6.3 Skill 定义规范

每个 Skill 的 `SKILL.md` 必须包含：

```yaml
---
name: skill-name
version: x.y.z
author: AMiner
description: >
  触发条件和能力描述
metadata:
  openclaw:
    requires:
      env: [AMINER_API_KEY]
    primaryEnv: AMINER_API_KEY
---
```

---

## 7. 测试与调试

### 7.1 评测用例

```bash
# 查看评测用例
cat skills/aminer-academic-search/evals/evals.json
```

### 7.2 Dry Run（预览费用）

```bash
python skills/aminer-academic-search/scripts/aminer_client.py \
  --token $AMINER_API_KEY \
  --action scholar_profile --name "Andrew Ng" \
  --dry-run
```

### 7.3 单步 API 调试

```bash
curl -X GET \
  'https://datacenter.aminer.cn/gateway/open_platform/api/paper/search?page=1&size=5&title=BERT' \
  -H "Authorization: ${AMINER_API_KEY}" \
  -H 'X-Platform: openclaw'
```

---

## 8. 常见问题

| 问题 | 解决方案 |
|---|---|
| `AMINER_API_KEY missing` | 前往 [控制台](https://open.aminer.cn/open/board?tab=control) 生成 Token |
| `paper_search` 无结果 | 降级到 `paper_search_pro` |
| 学者名字歧义 | 增加 `org` / `org_id` 过滤，或让用户确认 |
| LLM 调用超时 | 调大 `--timeout`，或检查 `llm.base_url` 连通性 |
| 引用扩展返回论文过少 | 尝试更换种子论文，或增大 `size_per_paper` |

---

## 9. 参考链接

- [AMiner 开放平台文档](https://open.aminer.cn/open/docs)
- [AMiner 控制台](https://open.aminer.cn/open/board?tab=control)
- [项目 GitHub 仓库](https://github.com/Futurescv/aminer-open-skill)
