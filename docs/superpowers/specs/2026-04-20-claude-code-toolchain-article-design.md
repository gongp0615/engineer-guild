# 文章设计规格：深入 Claude Code 工具链架构

## 元信息

- **标题**: 《深入 Claude Code 工具链架构：从 Agent 到生态》
- **发布平台**: GitHub（Markdown）
- **目标读者**: 熟悉 Anthropic 生态的开发者，想了解 Claude Code 工具链的架构设计
- **语言**: 中文为主，技术术语保留英文
- **预估字数**: 8000-10000 字
- **写作风格**: 深度解析，配架构图和真实代码示例

---

## 章节设计

### 第 1 章：引言 — AI 编程的工程化演进（~500 字）

**目标**: 建立全文语境，解释为什么需要理解这些工具的架构

**内容要点**:
- AI 编程工具从 chat-based（ChatGPT/Cursor Chat）到 agent-based（Claude Code/Copilot Workspace）的范式转变
- Agent 模式的核心差异：AI 拥有工具调用能力，可以自主完成多步骤任务
- 理解架构的必要性：当你不再只是「问 AI 问题」，而是「配置 AI 团队」时，需要理解底层机制
- 文章路线图：将按分层架构自顶向下展开

**写作手法**: 简洁叙述，快速引出核心问题，不做科普

---

### 第 2 章：全局架构 — Claude Code 的分层设计（~800 字）

**目标**: 给出全局架构图，建立读者的心智模型

**内容要点**:

1. **分层架构图**（ASCII 图）:

```
┌─────────────────────────────────────────┐
│             用户交互层                    │
│   CLI / VS Code Extension / Web App     │
├─────────────────────────────────────────┤
│             Agent 引擎层                 │
│   Agent Loop (Think → Tool → Observe)   │
│   Context Management / Multi-Agent      │
├──────────┬──────────┬───────────────────┤
│  Skills  │   MCP    │     Hooks         │
│  能力系统 │  外部协议 │   事件驱动        │
├──────────┴──────────┴───────────────────┤
│           Harness 运行时                 │
│   Permissions / Settings / Sandbox      │
├─────────────────────────────────────────┤
│           Spec 规范层                    │
│   CLAUDE.md / Rules / Memory            │
└─────────────────────────────────────────┘
```

2. **各层职责简述**:
   - 用户交互层：不同 UI 形态共享同一引擎
   - Agent 引擎层：核心决策循环
   - 能力扩展层（Skills / MCP / Hooks）：三种扩展机制各有定位
   - Harness 运行时：安全与配置基础设施
   - Spec 规范层：声明式配置，指导 Agent 行为

3. **设计哲学**: 插件化、可组合、声明式优于命令式

**写作手法**: 架构图 + 表格对比 + 设计哲学提炼

---

### 第 3 章：Agent 引擎 — 核心决策循环（~1200 字）

**目标**: 深入理解 Agent 如何思考、决策和执行

**内容要点**:

1. **Agent Loop 工作原理**:
   ```
   用户输入 → LLM 推理 → 工具调用决策 → 执行工具 → 观察结果 → 继续推理或返回
   ```
   - 每一轮（turn）的完整流程
   - 上下文窗口管理策略（滑动窗口、消息压缩）
   - 何时停止循环（任务完成、达到限制、用户中断）

2. **Agent 的上下文管理**:
   - System Prompt 的构成（指令 + Skills + Rules + Memory）
   - 对话历史与工具结果的生命周期
   - 上下文压缩机制（compaction）

3. **多 Agent 协作模式**:
   - 主 Agent 与子 Agent（subagent）的关系
   - 并行 Agent 的调度（如 code-reviewer + security-reviewer 同时工作）
   - Agent 类型系统（Explore / Plan / general-purpose / 自定义）

4. **代码示例**: 一个 Agent turn 的伪代码/配置示例

**写作手法**: 流程图 + 内部机制解析 + 实际代码/配置片段

---

### 第 4 章：Skill 系统 — 可扩展的能力模块（~1200 字）

**目标**: 理解 Skill 的架构设计和扩展机制

**内容要点**:

1. **Skill 是什么**:
   - 定义：以 Markdown + YAML frontmatter 编写的能力描述文件
   - 与传统「插件」的区别：Skill 是给 LLM 的指令，不是代码
   - 核心理念：教 AI 「如何做」而不是「做什么」

2. **Skill 的结构**:
   ```yaml
   ---
   name: my-skill
   description: When to use this skill
   ---
   # Skill instructions (markdown)
   ```
   - frontmatter 字段详解（name, description, trigger 条件）
   - body 中的指令格式（checklist、流程图 dot 语法、决策树）

3. **Skill 的发现与加载机制**:
   - 内置 Skill vs 项目 Skill vs 插件 Skill
   - 发现路径：`.claude/skills/`、`~/.claude/skills/`、插件目录
   - 加载时机：对话启动时 vs 按需加载

4. **Skill 与 Agent 的协作**:
   - Skill 如何注入 Agent 的上下文
   - Skill 中引用工具的方式
   - Skill 的优先级与冲突解决

5. **代码示例**: 一个完整的自定义 Skill 文件

**写作手法**: 结构剖析 + 配置示例 + 与传统插件机制的对比

---

### 第 5 章：MCP 协议 — 连接外部世界（~1200 字）

**目标**: 理解 MCP 的协议设计思想和实际使用方式

**内容要点**:

1. **MCP 是什么**:
   - Model Context Protocol，开放协议
   - 解决的核心问题：AI 模型如何标准化地访问外部工具和数据
   - 类比：类似于 USB 协议统一了外设连接

2. **MCP 架构**:
   ```
   Claude Code (Host) ↔ MCP Client ↔ MCP Server ↔ 外部服务
   ```
   - Host：运行 AI 的应用程序（Claude Code）
   - Client：协议客户端，管理与服务器的连接
   - Server：提供工具和数据的服务端

3. **MCP 三大原语**:
   - **Tools**: AI 可调用的函数（如 `web_search`、`read_file`）
   - **Resources**: AI 可读取的数据源（如文档、数据库 schema）
   - **Prompts**: 预定义的提示模板

4. **MCP 配置实战**:
   - settings.json 中的 MCP server 配置
   - stdio vs SSE 传输方式
   - 常用 MCP server 示例（filesystem、github、web）

5. **设计权衡**:
   - MCP vs 直接 API 调用：为什么需要中间协议层
   - 安全模型：权限边界与沙箱

**写作手法**: 协议架构图 + 配置代码 + 类比说明

---

### 第 6 章：Hooks 机制 — 事件驱动的自动化（~1000 字）

**目标**: 理解 Hooks 的设计模式和应用场景

**内容要点**:

1. **Hooks 是什么**:
   - 在特定事件触发时执行的 shell 命令
   - 事件类型：PreToolUse（工具执行前）、PostToolUse（工具执行后）、Stop（会话结束时）
   - 与传统 CI/CD hook 的对比

2. **Hooks 的配置**:
   ```json
   {
     "hooks": {
       "PreToolUse": [
         {
           "matcher": "Bash",
           "command": "security-check.sh $TOOL_INPUT"
         }
       ]
     }
   }
   ```
   - 配置位置：settings.json / settings.local.json
   - matcher 匹配机制（工具名、pattern）
   - 环境变量传递（$TOOL_INPUT 等）

3. **典型应用场景**:
   - 安全审计：在 Bash 执行前检查危险命令
   - 自动格式化：在文件编辑后自动运行 prettier
   - 日志记录：记录所有工具调用到审计日志
   - 权限控制：阻止特定文件被修改

4. **设计权衡**:
   - Hooks vs MCP Tools 的定位区别
   - Hooks 的性能影响
   - 失败处理策略（block vs warn vs ignore）

**写作手法**: 配置示例 + 场景表格 + 设计模式分析

---

### 第 7 章：Spec 规范 — 声明式配置驱动 Agent 行为（~800 字）

**目标**: 理解如何通过声明式配置塑造 Agent 行为

**内容要点**:

1. **Spec 是什么**:
   - 声明式的规范文件，告诉 Agent 应该遵循什么规则
   - 与 Skill 的区别：Spec 定义约束，Skill 定义能力

2. **Spec 的层次结构**:
   - **CLAUDE.md**: 项目级规范（编码风格、架构约定、工作流）
   - **Rules**: 规则文件（`.claude/rules/`），按场景组织
   - **Memory**: 持久化记忆（用户偏好、项目上下文）
   - 层级优先级：项目级 > 用户级 > 全局

3. **Spec 与 Agent 的关系**:
   - Spec 如何被注入 Agent 的 system prompt
   - 规则的冲突解决机制
   - Spec 的热更新（修改后即时生效，无需重启）

4. **代码示例**: 一个完整的 CLAUDE.md 示例

**写作手法**: 层次图 + 配置示例 + 与传统配置管理的对比

---

### 第 8 章：Harness 运行时 — 安全与配置基础设施（~800 字）

**目标**: 理解 Harness 作为运行时的职责和设计

**内容要点**:

1. **Harness 是什么**:
   - Claude Code 的运行时基础设施
   - 负责：权限管理、配置加载、沙箱执行、工具注册

2. **权限模型**:
   - 分级权限：allow / deny / ask
   - 权限来源：settings.json（项目级）、settings.local.json（用户级）、~/.claude.json（全局）
   - 权限匹配模式（工具名 pattern、命令 pattern）

3. **配置层级**:
   ```
   ~/.claude.json (全局)
     └── ~/.claude/settings.json (用户)
       └── .claude/settings.json (项目)
         └── .claude/settings.local.json (项目-本地，不入版本控制)
   ```
   - 层级合并策略（deep merge）
   - 敏感配置的处理（settings.local.json 不提交到 git）

4. **沙箱机制**:
   - Bash 命令的沙箱执行
   - 文件系统访问控制
   - 网络请求限制

**写作手法**: 配置层级图 + 权限配置示例 + 安全模型分析

---

### 第 9 章：生态包实战 — Superpowers 深度解析（~1500 字）

**目标**: 通过 Superpowers 展示生态包如何组合使用各项能力

**内容要点**:

1. **生态包概述**:
   - 什么是 Claude Code 插件/包
   - 安装方式（`claude plugin add`）
   - 包的目录结构（skills/ + agents/ + rules/）

2. **Superpowers 深度解析**:
   - Superpowers 的设计理念：为 Claude Code 注入「软件工程方法论」
   - 核心 Skills 解析：
     - **brainstorming**: 强制先理解需求再动手
     - **test-driven-development**: TDD 工作流
     - **writing-plans / executing-plans**: 计划驱动的开发流程
     - **systematic-debugging**: 系统化调试方法论
     - **requesting-code-review / receiving-code-review**: 双向代码审查
   - Superpowers 如何利用 Skills + Hooks + Rules 的组合

3. **其他热门包概览**:
   - init: 项目初始化
   - simplify: 代码简化审查
   - security-review: 安全审查
   - loop: 循环任务调度
   - schedule: 定时任务管理

4. **如何选择和组合生态包**:
   - 根据团队规模选择
   - 根据项目类型选择
   - 包的冲突处理

**写作手法**: 实际 Skill 内容剖析 + 架构分析 + 选型建议

---

## 写作规范

### 技术术语处理
- 首次出现的术语：中英文并列（如「上下文窗口 Context Window」）
- 后续使用：仅英文（如 Context Window）
- 保留原文不翻译的术语：Agent、Skill、MCP、Hooks、Spec、Harness、Prompt、Tool

### 代码示例规范
- 所有代码示例必须来自真实配置，不是伪代码
- JSON 配置使用实际 settings.json 格式
- Skill 文件使用实际 frontmatter + markdown 格式
- 代码块标注文件路径

### 架构图规范
- 使用 ASCII 图（GitHub markdown 兼容）
- 关键架构图使用 mermaid 语法（GitHub 原生支持）
- 每个图配文字解读

### 章节内部结构（统一四段式）
1. **是什么**：概念定义，解决什么问题
2. **怎么工作**：架构原理，内部机制
3. **怎么用**：配置示例，实战用法
4. **设计权衡**：与其他方案的对比，取舍分析
