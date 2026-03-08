# 技能与知识管理

> 本文解析 AI Agent 如何通过 Skills 系统动态加载专业知识，实现按需知识注入。

## 一、为什么需要 Skills 系统？

### 1.1 问题场景

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     知识注入的问题                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  方案1: 把所有知识放在 System Prompt 中                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  System Prompt:                                                 │   │
│  │  "你是编程助手。以下是所有可能用到的知识：                      │   │
│  │   - PDF处理: [5000 tokens...]                                   │   │
│  │   - 代码审查: [8000 tokens...]                                  │   │
│  │   - MCP开发: [6000 tokens...]                                   │   │
│  │   - React开发: [10000 tokens...]                                │   │
│  │   - ..."                                                        │   │
│  │                                                                 │   │
│  │  问题:                                                          │   │
│  │  ├── System Prompt 膨胀，占用大量 token                         │   │
│  │  ├── 很多知识根本用不到                                         │   │
│  │  ├── 每次请求都要发送这些内容，费用高昂                         │   │
│  │  └── 添加新知识需要修改代码                                     │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  方案2: Skills 系统 - 按需加载                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  System Prompt (轻量):                                          │   │
│  │  "你是编程助手。可用的技能:                                     │   │
│  │   - pdf: 处理 PDF 文件                                          │   │
│  │   - code-review: 代码审查                                       │   │
│  │   - mcp-builder: MCP 开发                                       │   │
│  │   需要时调用 load_skill 获取详细指导。"                         │   │
│  │                                                                 │   │
│  │  (~100 tokens vs ~30000 tokens)                                 │   │
│  │                                                                 │   │
│  │  Agent 需要时:                                                  │   │
│  │  load_skill("pdf") → 获取完整的 PDF 处理知识                    │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心理念

```
                        Skills 系统核心理念
                        ==================

    ┌──────────────────────────────────────────────────────────────────┐
    │                                                                  │
    │   "用到什么知识，临时加载什么知识"                                │
    │                                                                  │
    │   两层注入机制:                                                  │
    │                                                                  │
    │   ┌──────────────────────────────────────────────────────────┐  │
    │   │                                                          │  │
    │   │  Layer 1 (轻量): 技能元数据注入 System Prompt           │  │
    │   │  ───────────────────────────────────────────────────────│  │
    │   │  只包含技能名称和简短描述                                 │  │
    │   │  约 100 tokens / 技能                                    │  │
    │   │  让 Agent 知道有哪些技能可用                             │  │
    │   │                                                          │  │
    │   └──────────────────────────────────────────────────────────┘  │
    │                                                                  │
    │   ┌──────────────────────────────────────────────────────────┐  │
    │   │                                                          │  │
    │   │  Layer 2 (详细): 完整技能内容通过 tool_result 注入       │  │
    │   │  ───────────────────────────────────────────────────────│  │
    │   │  Agent 调用 load_skill 时返回完整内容                     │  │
    │   │  包含详细的步骤、代码示例、最佳实践                       │  │
    │   │  只在需要时才占用 token                                   │  │
    │   │                                                          │  │
    │   └──────────────────────────────────────────────────────────┘  │
    │                                                                  │
    └──────────────────────────────────────────────────────────────────┘
```

## 二、Skill 文件结构

### 2.1 目录结构

```
                        Skills 目录结构
                        ===============

    ┌──────────────────────────────────────────────────────────────────────┐
    │                                                                      │
    │  skills/                                                             │
    │  ├── pdf/                                                            │
    │  │   └── SKILL.md          ← PDF 处理技能                           │
    │  │                                                                   │
    │  ├── code-review/                                                    │
    │  │   └── SKILL.md          ← 代码审查技能                           │
    │  │                                                                   │
    │  ├── mcp-builder/                                                    │
    │  │   └── SKILL.md          ← MCP 开发技能                           │
    │  │                                                                   │
    │  └── agent-builder/                                                  │
    │      └── SKILL.md          ← Agent 构建技能                         │
    │                                                                      │
    │  命名规范:                                                           │
    │  ├── 每个技能一个目录                                                │
    │  ├── 目录名即技能名                                                  │
    │  └── 必须包含 SKILL.md 文件                                          │
    │                                                                      │
    └──────────────────────────────────────────────────────────────────────┘
```

### 2.2 SKILL.md 文件格式

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     SKILL.md 文件格式                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  文件结构:                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ ---                                                             │   │
│  │ name: pdf                        # 技能名称 (用于 load_skill)  │   │
│  │ description: Process PDF files - extract text, create PDFs     │   │
│  │ tags: document, pdf             # 可选标签                      │   │
│  │ ---                                                             │   │
│  │                                                                 │   │
│  │ # PDF Processing Skill          # 技能正文 (Markdown格式)      │   │
│  │                                                                 │   │
│  │ You now have expertise in PDF manipulation...                  │   │
│  │                                                                 │   │
│  │ ## Reading PDFs                                                 │   │
│  │                                                                 │   │
│  │ ```bash                                                         │   │
│  │ pdftotext input.pdf -                                           │   │
│  │ ```                                                             │   │
│  │                                                                 │   │
│  │ ...更多详细内容...                                               │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  YAML Frontmatter 解析:                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  def _parse_frontmatter(text: str) -> tuple:                   │   │
│  │      match = re.match(r"^---\n(.*?)\n---\n(.*)", text, re.DOTALL)│   │
│  │      if not match:                                              │   │
│  │          return {}, text  # 没有 frontmatter                   │   │
│  │                                                                 │   │
│  │      meta = {}                                                  │   │
│  │      for line in match.group(1).strip().splitlines():          │   │
│  │          if ":" in line:                                        │   │
│  │              key, val = line.split(":", 1)                      │   │
│  │              meta[key.strip()] = val.strip()                    │   │
│  │                                                                 │   │
│  │      return meta, match.group(2).strip()  # 返回元数据和正文   │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Skill 示例

```
                        完整 Skill 示例
                        ===============

    ┌──────────────────────────────────────────────────────────────────────┐
    │                                                                      │
    │  skills/code-review/SKILL.md:                                        │
    │                                                                      │
    │  ---                                                                 │
    │  name: code-review                                                   │
    │  description: Perform thorough code reviews with security,          │
    │               performance, and maintainability analysis             │
    │  tags: review, security, quality                                    │
    │  ---                                                                 │
    │                                                                      │
    │  # Code Review Skill                                                │
    │                                                                      │
    │  You now have expertise in conducting comprehensive code reviews.   │
    │                                                                      │
    │  ## Review Checklist                                                │
    │                                                                      │
    │  ### 1. Security (Critical)                                         │
    │                                                                      │
    │  Check for:                                                          │
    │  - [ ] **Injection vulnerabilities**: SQL, command, XSS             │
    │  - [ ] **Authentication issues**: Hardcoded credentials             │
    │  - [ ] **Authorization flaws**: Missing access controls             │
    │  - [ ] **Data exposure**: Sensitive data in logs                    │
    │                                                                      │
    │  ### 2. Correctness                                                 │
    │                                                                      │
    │  Check for:                                                          │
    │  - [ ] **Logic errors**: Off-by-one, null handling                  │
    │  - [ ] **Race conditions**: Concurrent access issues                │
    │  - [ ] **Resource leaks**: Unclosed files, connections              │
    │                                                                      │
    │  ### 3. Performance                                                  │
    │                                                                      │
    │  Check for:                                                          │
    │  - [ ] **N+1 queries**: Database calls in loops                     │
    │  - [ ] **Memory issues**: Large allocations                         │
    │  - [ ] **Inefficient algorithms**: O(n^2) when O(n) possible        │
    │                                                                      │
    │  ## Review Commands                                                 │
    │                                                                      │
    │  ```bash                                                            │
    │  # Quick security scans                                            │
    │  npm audit                    # Node.js                            │
    │  pip-audit                    # Python                             │
    │  grep -r "password\|secret" --include="*.py"                        │
    │  ```                                                                │
    │                                                                      │
    │  ## Review Output Format                                            │
    │                                                                      │
    │  ```markdown                                                        │
    │  ## Code Review: [file/component name]                              │
    │                                                                      │
    │  ### Critical Issues                                                │
    │  1. **[Issue]** (line X): [Description]                             │
    │     - Impact: [What could go wrong]                                 │
    │     - Fix: [Suggested solution]                                     │
    │  ```                                                                │
    │                                                                      │
    └──────────────────────────────────────────────────────────────────────┘
```

## 三、Skill 加载流程

### 3.1 初始化加载

```
                        Skill 初始化加载
                        ================

    ┌──────────────────────────────────────────────────────────────────────┐
    │                                                                      │
    │  启动时扫描 skills/ 目录:                                           │
    │                                                                      │
    │  ┌────────────────────────────────────────────────────────────────┐ │
    │  │                                                                │ │
    │  │  class SkillLoader:                                            │ │
    │  │      def __init__(self, skills_dir: Path):                     │ │
    │  │          self.skills_dir = skills_dir                          │ │
    │  │          self.skills = {}                                      │ │
    │  │          self._load_all()                                      │ │
    │  │                                                                │ │
    │  │      def _load_all(self):                                      │ │
    │  │          for f in sorted(self.skills_dir.rglob("SKILL.md")):  │ │
    │  │              text = f.read_text()                              │ │
    │  │              meta, body = self._parse_frontmatter(text)        │ │
    │  │              name = meta.get("name", f.parent.name)            │ │
    │  │              self.skills[name] = {                             │ │
    │  │                  "meta": meta,                                 │ │
    │  │                  "body": body,                                 │ │
    │  │                  "path": str(f)                                │ │
    │  │              }                                                 │ │
    │  │                                                                │ │
    │  └────────────────────────────────────────────────────────────────┘ │
    │                                                                      │
    │  加载结果:                                                           │
    │  ┌────────────────────────────────────────────────────────────────┐ │
    │  │                                                                │ │
    │  │  skills = {                                                    │ │
    │  │      "pdf": {                                                  │ │
    │  │          "meta": {"name": "pdf", "description": "...", ...},  │ │
    │  │          "body": "# PDF Processing Skill\n...",                │ │
    │  │          "path": "skills/pdf/SKILL.md"                         │ │
    │  │      },                                                        │ │
    │  │      "code-review": {                                          │ │
    │  │          "meta": {"name": "code-review", ...},                 │ │
    │  │          "body": "# Code Review Skill\n...",                   │ │
    │  │          "path": "skills/code-review/SKILL.md"                 │ │
    │  │      },                                                        │ │
    │  │      ...                                                       │ │
    │  │  }                                                             │ │
    │  │                                                                │ │
    │  └────────────────────────────────────────────────────────────────┘ │
    │                                                                      │
    └──────────────────────────────────────────────────────────────────────┘
```

### 3.2 Layer 1: 元数据注入

```
                        Layer 1: 元数据注入
                        ===================

    ┌──────────────────────────────────────────────────────────────────────┐
    │                                                                      │
    │  生成技能描述列表:                                                   │
    │  ┌────────────────────────────────────────────────────────────────┐ │
    │  │                                                                │ │
    │  │  def get_descriptions(self) -> str:                            │ │
    │  │      lines = []                                                │ │
    │  │      for name, skill in self.skills.items():                   │ │
    │  │          desc = skill["meta"].get("description", "No desc")    │ │
    │  │          tags = skill["meta"].get("tags", "")                  │ │
    │  │          line = f"  - {name}: {desc}"                          │ │
    │  │          if tags:                                              │ │
    │  │              line += f" [{tags}]"                              │ │
    │  │          lines.append(line)                                    │ │
    │  │      return "\n".join(lines)                                   │ │
    │  │                                                                │ │
    │  └────────────────────────────────────────────────────────────────┘ │
    │                                                                      │
    │  输出示例:                                                           │
    │  ┌────────────────────────────────────────────────────────────────┐ │
    │  │                                                                │ │
    │  │  Skills available:                                             │ │
    │  │    - pdf: Process PDF files - extract text, create PDFs       │ │
    │  │    - code-review: Perform thorough code reviews               │ │
    │  │    - mcp-builder: Build MCP servers and tools                 │ │
    │  │    - agent-builder: Design and build AI agents                │ │
    │  │                                                                │ │
    │  └────────────────────────────────────────────────────────────────┘ │
    │                                                                      │
    │  注入到 System Prompt:                                               │
    │  ┌────────────────────────────────────────────────────────────────┐ │
    │  │                                                                │ │
    │  │  SYSTEM = f"""You are a coding agent at {WORKDIR}.            │ │
    │  │  Use load_skill to access specialized knowledge before        │ │
    │  │  tackling unfamiliar topics.                                  │ │
    │  │                                                                │ │
    │  │  Skills available:                                            │ │
    │  │  {SKILL_LOADER.get_descriptions()}                            │ │
    │  │  """                                                          │ │
    │  │                                                                │ │
    │  └────────────────────────────────────────────────────────────────┘ │
    │                                                                      │
    │  Token 使用对比:                                                     │
    │  ┌────────────────────────────────────────────────────────────────┐ │
    │  │                                                                │ │
    │  │  传统方式 (全部知识):                                          │ │
    │  │  System Prompt = 30000+ tokens                                │ │
    │  │                                                                │ │
    │  │  Skills 方式 (仅元数据):                                       │ │
    │  │  System Prompt = ~500 tokens (4个技能 x ~100 tokens)          │ │
    │  │                                                                │ │
    │  │  节省: ~98%                                                    │ │
    │  │                                                                │ │
    │  └────────────────────────────────────────────────────────────────┘ │
    │                                                                      │
    └──────────────────────────────────────────────────────────────────────┘
```

### 3.3 Layer 2: 完整内容加载

```
                        Layer 2: 完整内容加载
                        =====================

    ┌──────────────────────────────────────────────────────────────────────────┐
    │                                                                          │
    │  用户: "帮我审查这段代码的安全性"                                        │
    │                                                                          │
    │  ┌────────────────────────────────────────────────────────────────────┐ │
    │  │ Agent 循环:                                                        │ │
    │  │                                                                    │ │
    │  │  LLM 看到 System Prompt:                                          │ │
    │  │  "Skills available:                                               │ │
    │  │    - code-review: Perform thorough code reviews..."               │ │
    │  │                                                                    │ │
    │  │  LLM 思考:                                                         │ │
    │  │  "用户要做代码审查，我应该先加载 code-review 技能"                │ │
    │  │                                                                    │ │
    │  │  LLM 输出:                                                         │ │
    │  │  ┌──────────────────────────────────────────────────────────────┐ │ │
    │  │  │ tool_use: load_skill                                         │ │ │
    │  │  │ name: "code-review"                                          │ │ │
    │  │  └──────────────────────────────────────────────────────────────┘ │ │
    │  │                                                                    │ │
    │  └────────────────────────────────────────────────────────────────────┘ │
    │                              │                                          │
    │                              ▼                                          │
    │  ┌────────────────────────────────────────────────────────────────────┐ │
    │  │ 执行 load_skill 工具:                                             │ │
    │  │                                                                    │ │
    │  │  def get_content(self, name: str) -> str:                         │ │
    │  │      skill = self.skills.get(name)                                │ │
    │  │      if not skill:                                                │ │
    │  │          return f"Error: Unknown skill '{name}'"                  │ │
    │  │      return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>" │ │
    │  │                                                                    │ │
    │  └────────────────────────────────────────────────────────────────────┘ │
    │                              │                                          │
    │                              ▼                                          │
    │  ┌────────────────────────────────────────────────────────────────────┐ │
    │  │ tool_result 返回:                                                 │ │
    │  │                                                                    │ │
    │  │  <skill name="code-review">                                       │ │
    │  │  # Code Review Skill                                              │ │
    │  │                                                                    │ │
    │  │  You now have expertise in conducting comprehensive code reviews. │ │
    │  │                                                                    │ │
    │  │  ## Review Checklist                                              │ │
    │  │                                                                    │ │
    │  │  ### 1. Security (Critical)                                       │ │
    │  │  Check for:                                                       │ │
    │  │  - [ ] **Injection vulnerabilities**: SQL, command, XSS...        │ │
    │  │  - [ ] **Authentication issues**: Hardcoded credentials...        │ │
    │  │  ...                                                              │ │
    │  │  </skill>                                                         │ │
    │  │                                                                    │ │
    │  └────────────────────────────────────────────────────────────────────┘ │
    │                              │                                          │
    │                              ▼                                          │
    │  ┌────────────────────────────────────────────────────────────────────┐ │
    │  │ 下一轮循环:                                                       │ │
    │  │                                                                    │ │
    │  │  LLM 现在看到了完整的代码审查知识                                 │ │
    │  │  LLM 按照技能中的检查清单进行代码审查                             │ │
    │  │                                                                    │ │
    │  └────────────────────────────────────────────────────────────────────┘ │
    │                                                                          │
    └──────────────────────────────────────────────────────────────────────────┘
```

## 四、完整流程图

```
                        Skills 系统完整流程
                        ===================

    ┌──────────────────────────────────────────────────────────────────────────┐
    │                                                                          │
    │  启动阶段:                                                               │
    │  ┌────────────────────────────────────────────────────────────────────┐ │
    │  │                                                                    │ │
    │  │  1. SkillLoader 扫描 skills/ 目录                                 │ │
    │  │     │                                                              │ │
    │  │     ├── skills/pdf/SKILL.md ──────▶ skills["pdf"]                 │ │
    │  │     ├── skills/code-review/SKILL.md ──▶ skills["code-review"]     │ │
    │  │     └── skills/agent-builder/SKILL.md ─▶ skills["agent-builder"]  │ │
    │  │                                                                    │ │
    │  │  2. 生成技能描述列表                                               │ │
    │  │     get_descriptions() ──▶ "  - pdf: ...\n  - code-review: ..."   │ │
    │  │                                                                    │ │
    │  │  3. 构建 System Prompt                                            │ │
    │  │     SYSTEM = "Skills available:\n{descriptions}"                  │ │
    │  │                                                                    │ │
    │  └────────────────────────────────────────────────────────────────────┘ │
    │                              │                                          │
    │                              ▼                                          │
    │  运行阶段:                                                               │
    │  ┌────────────────────────────────────────────────────────────────────┐ │
    │  │                                                                    │ │
    │  │  用户输入 ──▶ Agent Loop                                          │ │
    │  │                    │                                               │ │
    │  │                    ▼                                               │ │
    │  │              ┌───────────┐                                        │ │
    │  │              │ 调用 LLM  │ ◀── System Prompt 包含技能列表         │ │
    │  │              └─────┬─────┘                                        │ │
    │  │                    │                                               │ │
    │  │                    ▼                                               │ │
    │  │        ┌───────────────────────┐                                  │ │
    │  │        │ LLM 需要专业知识吗？  │                                  │ │
    │  │        └───────────┬───────────┘                                  │ │
    │  │                    │                                               │ │
    │  │          ┌────────┴────────┐                                      │ │
    │  │          │                 │                                      │ │
    │  │         YES               NO                                       │
    │  │          │                 │                                      │ │
    │  │          ▼                 ▼                                       │
    │  │   ┌────────────┐    ┌────────────┐                               │ │
    │  │   │ load_skill │    │ 执行其他   │                               │ │
    │  │   │ "xxx"      │    │ 工具       │                               │ │
    │  │   └─────┬──────┘    └────────────┘                               │ │
    │  │         │                                                         │ │
    │  │         ▼                                                         │ │
    │  │   ┌─────────────────────────────────────────┐                    │ │
    │  │   │ tool_result:                            │                    │ │
    │  │   │ <skill name="xxx">                      │                    │ │
    │  │   │   完整的技能内容...                     │                    │ │
    │  │   │ </skill>                                │                    │ │
    │  │   └─────────────────────────────────────────┘                    │ │
    │  │         │                                                         │ │
    │  │         ▼                                                         │ │
    │  │   ┌─────────────────────────────────────────┐                    │ │
    │  │   │ 下一轮 LLM 调用看到技能内容            │                    │ │
    │  │   │ LLM 使用这些知识执行任务              │                    │ │
    │  │   └─────────────────────────────────────────┘                    │ │
    │  │                                                                    │ │
    │  └────────────────────────────────────────────────────────────────────┘ │
    │                                                                          │
    └──────────────────────────────────────────────────────────────────────────┘
```

## 五、自定义 Skill 开发

### 5.1 创建新 Skill

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     创建新 Skill 步骤                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  步骤1: 创建目录和文件                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  mkdir -p skills/go-backend                                    │   │
│  │  touch skills/go-backend/SKILL.md                              │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  步骤2: 编写 SKILL.md                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ ---                                                             │   │
│  │ name: go-backend                                                │   │
│  │ description: Build production-ready Go backend services        │   │
│  │ tags: go, backend, api, microservices                           │   │
│  │ ---                                                             │   │
│  │                                                                 │   │
│  │ # Go Backend Development Skill                                  │   │
│  │                                                                 │   │
│  │ You now have expertise in building Go backend services.        │   │
│  │                                                                 │   │
│  │ ## Project Structure                                            │   │
│  │                                                                 │   │
│  │ ```                                                             │   │
│  │ myapp/                                                          │   │
│  │ ├── cmd/                                                        │   │
│  │ │   └── server/main.go                                          │   │
│  │ ├── internal/                                                   │   │
│  │ │   ├── handlers/                                               │   │
│  │ │   ├── services/                                               │   │
│  │ │   └── repository/                                             │   │
│  │ ├── pkg/                                                        │   │
│  │ └── go.mod                                                      │   │
│  │ ```                                                             │   │
│  │                                                                 │   │
│  │ ## Best Practices                                               │   │
│  │                                                                 │   │
│  │ 1. Use context for cancellation                                │   │
│  │ 2. Handle errors explicitly                                    │   │
│  │ 3. Use interfaces for dependency injection                     │   │
│  │ 4. Write table-driven tests                                    │   │
│  │                                                                 │   │
│  │ ## Common Patterns                                              │   │
│  │                                                                 │   │
│  │ ### Clean Architecture                                          │   │
│  │                                                                 │   │
│  │ ```go                                                           │   │
│  │ type Service interface {                                        │   │
│  │     CreateUser(ctx context.Context, req *CreateUserReq) error  │   │
│  │ }                                                               │   │
│  │                                                                 │   │
│  │ type service struct {                                           │   │
│  │     repo Repository                                             │   │
│  │ }                                                               │   │
│  │ ```                                                             │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  步骤3: 重启 Agent                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  # SkillLoader 会自动扫描新的 SKILL.md 文件                     │   │
│  │  # 新技能会在下次启动时自动加载                                 │   │
│  │                                                                 │   │
│  │  # System Prompt 会包含:                                        │   │
│  │  #   - go-backend: Build production-ready Go backend services  │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Skill 内容建议

```
                        Skill 内容最佳实践
                        ==================

    ┌──────────────────────────────────────────────────────────────────────┐
    │                                                                      │
    │  好的 Skill 应该包含:                                               │
    │                                                                      │
    │  1. 概述                                                            │
    │     ├── 这个技能解决什么问题                                        │
    │     └── 什么时候应该使用                                            │
    │                                                                      │
    │  2. 详细步骤                                                        │
    │     ├── 分步骤的操作指南                                            │
    │     └── 每个步骤的预期结果                                          │
    │                                                                      │
    │  3. 代码示例                                                        │
    │     ├── 完整可运行的代码                                            │
    │     └── 关键部分的解释                                              │
    │                                                                      │
    │  4. 最佳实践                                                        │
    │     ├── 推荐的做法                                                  │
    │     └── 应该避免的陷阱                                              │
    │                                                                      │
    │  5. 故障排除                                                        │
    │     ├── 常见问题                                                    │
    │     └── 解决方案                                                    │
    │                                                                      │
    │  6. 相关资源                                                        │
    │     ├── 相关工具/库                                                 │
    │     └── 进一步学习的链接                                            │
    │                                                                      │
    └──────────────────────────────────────────────────────────────────────┘

    内容长度建议:

    ┌──────────────────────────────────────────────────────────────────────┐
    │                                                                      │
    │  太短 (< 500 tokens):                                                │
    │  ├── 信息不够详细                                                    │
    │  └── Agent 可能无法正确执行                                          │
    │                                                                      │
    │  适中 (1000-5000 tokens):                                            │
    │  ├── 足够详细                                                        │
    │  └── 不会过度占用上下文                                              │
    │                                                                      │
    │  太长 (> 10000 tokens):                                              │
    │  ├── 可能包含冗余信息                                                │
    │  └── 考虑拆分成多个技能                                              │
    │                                                                      │
    └──────────────────────────────────────────────────────────────────────┘
```

## 六、与后端开发的对应

### 6.1 概念映射

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    概念映射表                                            │
├────────────────────┬────────────────────────────────────────────────────┤
│   Agent 概念       │              后端对应概念                          │
├────────────────────┼────────────────────────────────────────────────────┤
│                    │                                                    │
│  Skills            │  插件系统 / 模块系统                               │
│                    │  - 可动态加载的功能模块                            │
│                    │  - 如: WordPress Plugins, VS Code Extensions      │
│                    │                                                    │
├────────────────────┼────────────────────────────────────────────────────┤
│                    │                                                    │
│  SKILL.md 文件     │  配置文件 + 文档                                   │
│                    │  - package.json + README.md                        │
│                    │  - plugin.yaml + docs/                             │
│                    │                                                    │
├────────────────────┼────────────────────────────────────────────────────┤
│                    │                                                    │
│  Layer 1 (元数据)  │  插件注册表 / 服务发现                             │
│                    │  - 只暴露接口定义                                  │
│                    │  - 实际实现延迟加载                                │
│                    │                                                    │
├────────────────────┼────────────────────────────────────────────────────┤
│                    │                                                    │
│  Layer 2 (完整)    │  延迟加载 / 按需加载                               │
│                    │  - 只在调用时才加载完整实现                        │
│                    │  - 如: dynamic import, lazy loading               │
│                    │                                                    │
├────────────────────┼────────────────────────────────────────────────────┤
│                    │                                                    │
│  YAML Frontmatter  │  元数据注解                                        │
│                    │  - @Configuration, @Component                      │
│                    │  - decorators in Python/TypeScript                │
│                    │                                                    │
└────────────────────┴────────────────────────────────────────────────────┘
```

### 6.2 Go 后端实现示例

```go
// Skills 系统的 Go 实现
type Skill struct {
    Name        string            `json:"name"`
    Description string            `json:"description"`
    Tags        []string          `json:"tags"`
    Body        string            `json:"body"`
    Path        string            `json:"path"`
}

type SkillLoader struct {
    skillsDir string
    skills    map[string]Skill
}

func NewSkillLoader(skillsDir string) *SkillLoader {
    loader := &SkillLoader{
        skillsDir: skillsDir,
        skills:    make(map[string]Skill),
    }
    loader.loadAll()
    return loader
}

func (l *SkillLoader) loadAll() {
    files, _ := filepath.Glob(filepath.Join(l.skillsDir, "*/SKILL.md"))
    
    for _, f := range files {
        content, err := os.ReadFile(f)
        if err != nil {
            continue
        }
        
        meta, body := parseFrontmatter(string(content))
        name := meta["name"]
        if name == "" {
            name = filepath.Base(filepath.Dir(f))
        }
        
        l.skills[name] = Skill{
            Name:        name,
            Description: meta["description"],
            Tags:        strings.Split(meta["tags"], ","),
            Body:        body,
            Path:        f,
        }
    }
}

func (l *SkillLoader) GetDescriptions() string {
    var lines []string
    for name, skill := range l.skills {
        line := fmt.Sprintf("  - %s: %s", name, skill.Description)
        if len(skill.Tags) > 0 {
            line += fmt.Sprintf(" [%s]", strings.Join(skill.Tags, ", "))
        }
        lines = append(lines, line)
    }
    return strings.Join(lines, "\n")
}

func (l *SkillLoader) GetContent(name string) string {
    skill, ok := l.skills[name]
    if !ok {
        return fmt.Sprintf("Error: Unknown skill '%s'", name)
    }
    return fmt.Sprintf("<skill name=\"%s\">\n%s\n</skill>", name, skill.Body)
}

// 在 Agent 中使用
func buildSystemPrompt(loader *SkillLoader, workdir string) string {
    return fmt.Sprintf(`You are a coding agent at %s.
Use load_skill to access specialized knowledge before tackling unfamiliar topics.

Skills available:
%s`, workdir, loader.GetDescriptions())
}
```

---

## 七、总结

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           关键要点                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. Skills 系统通过两层注入实现知识的按需加载                           │
│                                                                         │
│  2. Layer 1 只注入元数据，节省 System Prompt token                     │
│                                                                         │
│  3. Layer 2 在 Agent 需要时才加载完整内容                               │
│                                                                         │
│  4. SKILL.md 文件格式简单，易于创建和维护                               │
│                                                                         │
│  5. 新技能只需添加文件，无需修改代码                                    │
│                                                                         │
│  6. 这种模式类似于后端的插件系统、延迟加载机制                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

**上一篇：[03-multi-agent-collaboration.md](./03-multi-agent-collaboration.md)**

**下一篇：[05-practical-applications.md](./05-practical-applications.md) - 实际应用场景**
