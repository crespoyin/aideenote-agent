# AideeNote App - Claude Code 多 Agent 协同开发模板

为 AideeNote 配套 App 设计的 Sub-agent + Worktree 协同开发模板。

## 快速开始

### 1. 复制模板到你的项目

```bash
# 假设你的 App 仓库在 ~/work/aideenote-app
cp -r .claude ~/work/aideenote-app/
cp CLAUDE.md ~/work/aideenote-app/
cp -r docs/AGENT_WORKFLOW.md ~/work/aideenote-app/docs/
cp -r tasks/_template.md ~/work/aideenote-app/tasks/
```

### 2. 在项目根目录启动 Claude Code

```bash
cd ~/work/aideenote-app
claude
```

Claude Code 会自动加载:
- `CLAUDE.md`(全局上下文)
- `.claude/agents/*.md`(8 个 Sub-agent)

### 3. 验证 Agent 注册

在 Claude Code 中输入:
```
列出可用的 sub-agents
```

应能看到:pm、ux、frontend、backend、qa、devops、analytics、ai-integration

## 目录结构

```
.claude/
├── agents/
│   ├── pm.md              # 产品经理:需求拆解
│   ├── ux.md              # UX:Figma 解析、组件规范
│   ├── frontend.md        # 前端:RN + HarmonyOS
│   ├── backend.md         # 后端:Java + Go 微服务
│   ├── qa.md              # 质量:测试 + 准入检查
│   ├── devops.md          # 运维:CI/CD + 灰度
│   ├── analytics.md       # 数据:埋点 + 分析
│   └── ai-integration.md  # AI:Prompt + Skill + 网关
└── skills/                # 共享 Skill(后续按需添加)

CLAUDE.md                  # 全局上下文(产品、技术栈、规范)
docs/
├── AGENT_WORKFLOW.md      # Lead-Teammate 操作手册
├── REVIEW_PROTOCOL.md     # 评审协议(契约式互审 v1)
├── api/openapi.yaml       # API 契约(单一来源)
├── design/                # 设计 Token 与组件规范
├── analytics/             # 埋点 Schema
├── prd/                   # PRD 缓存
└── adr/                   # 架构决策记录

tasks/
├── _template.md           # 任务交接模板(含评审签字区)
└── TASK-{id}.md           # 实际任务(按 Meego ID 命名)
```

## 使用模式

### 模式 A:单仓 Sub-agents(日常迭代)
直接在主仓启动 Claude Code,Lead 委派给 Sub-agent。

### 模式 B:Worktree 并行(大特性)
```bash
git worktree add ../aideenote-feature-xxx feature/xxx
cd ../aideenote-feature-xxx
claude
```
独立会话不污染主仓上下文。

详见 `docs/AGENT_WORKFLOW.md`。

## 定制建议

1. **首次使用**:跑通一个端到端的小任务(比如新增一个简单页面),验证 8 个 Agent 都能正确触发
2. **逐步精炼**:每次 Sub-agent 产出不理想,在对应 `.md` 里加约束
3. **Skill 沉淀**:常用工作流(如 PRD 解析、报告生成)抽成 Skill,放在 `.claude/skills/`
4. **Worktree 实验**:先在一个可控特性上试 Worktree 模式,再推广

## 与现有体系的衔接

- 与你之前的 `prd.skill`、`feishu-docx`、`obsidian` skill 兼容
- 与 IdeaMake 的 AI Harness 架构对齐(ai-integration agent 强调走网关)
- 与 Sentry 自动化管线衔接(qa agent 描述了完整流程)

## 后续可扩展方向

- 加入 `security` Agent 处理隐私合规(音频数据、用户授权)
- 加入 `support` Agent 对接客服反馈
- 把 `pm` 拆成 `pm-strategy`(战略)+ `pm-execution`(执行)
- 为每个 Agent 配套 Skill(例如给 `qa` 配 `e2e-test-generator` Skill)
# aideenote-agent
