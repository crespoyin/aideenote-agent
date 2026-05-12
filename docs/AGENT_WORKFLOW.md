# Agent 协同工作流

> 本文档是 Lead Agent(默认主会话)的操作手册。

## 一、Lead-Teammate 模式

### Lead 的角色
默认主会话扮演 Lead,职责:
1. 接收用户原始需求
2. 决定是直接处理还是委派给 Sub-agent
3. 拆解为 `tasks/TASK-{id}.md`
4. 用 Task 工具委派给对应 Sub-agent
5. 审查产出,跨 Agent 协调
6. 向用户汇报进展

### 委派的判断标准

| 任务特征 | 处理方式 |
|----------|---------|
| 单一领域、< 30 行代码改动 | Lead 直接处理 |
| 跨 1 个领域、需要专业知识 | 委派单个 Sub-agent |
| 跨多个领域 | 拆解后并行委派多个 Sub-agent |
| 大特性、影响多个服务 | 开 Worktree,启动并行 Claude Code 会话 |

## 二、Worktree 工作流

### 何时开 Worktree
- 特性预计 > 3 天
- 涉及 ≥ 3 个服务
- 需要长期分支(可能 2 周以上不合主干)
- 需要多人/多会话并行

### 标准操作
```bash
# 1. 创建 Worktree
git worktree add ../aideenote-feature-{name} feature/{name}

# 2. 在新 Worktree 启动独立 Claude Code 会话
cd ../aideenote-feature-{name}
claude

# 3. 该会话有独立的上下文和 Sub-agent 注册表
#    .claude/agents/ 自动从该 worktree 的目录加载

# 4. 完成后合并
git push origin feature/{name}
# 提 PR 到主分支

# 5. 清理 Worktree
git worktree remove ../aideenote-feature-{name}
```

### Worktree 命名规范
- `aideenote-feature-{slug}`:新特性
- `aideenote-fix-{slug}`:Bug 修复
- `aideenote-refactor-{slug}`:重构

## 三、典型场景剧本

### 场景 A:新功能开发(端到端)

```
用户:"做一个录音回放功能,支持倍速和章节标记"

Lead 行动:
1. 启动 pm sub-agent → 拉 Meego/写 PRD → 产出 TASK-XXX
2. 启动 ux sub-agent → 出组件规范
3. 并行委派:
   - backend sub-agent:定义 API + 后端实现
   - ai-integration sub-agent:章节自动标记的 Skill
4. 串行委派:
   - frontend sub-agent:基于 API 契约和组件规范实现
5. 委派 analytics sub-agent:埋点设计
6. 委派 qa sub-agent:测试 + 准入
7. 委派 devops sub-agent:灰度发布
8. Lead 汇总状态向用户汇报
```

### 场景 B:线上 Bug 修复

```
用户:"Sentry 上 RecordingViewModel NPE 错误激增"

Lead 行动:
1. 直接读 Sentry 数据(或委派 qa sub-agent)
2. 委派 frontend sub-agent 分析 + 修复
3. 委派 qa sub-agent 写回归用例
4. 委派 devops sub-agent 紧急发版
```

### 场景 C:AI 链路优化

```
用户:"报告生成质量下降,需要优化 Prompt"

Lead 行动:
1. 委派 analytics sub-agent:数据分析,找 bad case
2. 委派 ai-integration sub-agent:Prompt 调优 + 评测
3. 委派 ai-integration sub-agent:灰度发布
4. 委派 analytics sub-agent:跟踪指标
```

### 场景 D:大特性(开 Worktree)

```
用户:"做一个企业版后台,带租户管理、配额管理、报表"

Lead 行动:
1. 评估:跨 ≥ 4 个服务,预计 2 周
2. 决策:开 Worktree
3. 指引用户执行 git worktree add
4. 在新 Worktree 中按"场景 A"流程推进
5. 主 Worktree 继续日常迭代,互不干扰
```

## 四、Agent 调用注意事项

### Sub-agent 不持久化上下文
每次 Task 调用都是独立上下文。给 Sub-agent 的 prompt 必须包含:
- 任务文档路径(`tasks/TASK-{id}.md`)
- 相关文档路径(API 契约、设计稿等)
- 期望产出形式
- 完成后如何汇报

### Sub-agent 不能调用其他 Sub-agent
跨 Agent 协作只能通过 Lead 串联,或通过任务文档异步交接。

### Lead 必须做的事
- 读 Sub-agent 产出,做 sanity check
- 跨 Sub-agent 信息传递(把 A 的产出转给 B)
- 仲裁 Sub-agent 间的冲突
- 维护 `tasks/` 目录的整洁

## 五、PR 合并流程

```
1. 责任 Agent 自测通过
2. 提交 PR,关联 TASK-{id}
3. 检查任务文档评审签字区:涉及的契约是否都已签字(详见 docs/REVIEW_PROTOCOL.md)
4. 调用 qa sub-agent 跑准入 checklist(签字未齐全直接打回)
5. checklist 全过 → Lead 做架构审查
6. 通过 → 标记可合并
7. devops sub-agent 配合灰度发布
8. analytics sub-agent 跟踪上线后指标
```

## 六、与 Skill 的关系

- Sub-agent 提供"角色和职责边界"
- Skill 提供"具体执行方法"(如 `prd.skill`、`feishu-docx`、`obsidian`)
- 一个 Sub-agent 可以调用多个 Skill
- 例如 `pm` Sub-agent 调用 `prd.skill` 拉 Meego、调用 `feishu-docx` 写需求文档

## 七、调试与可观测

### 查看 Agent 调用历史
Claude Code 在 `.claude/sessions/` 下保留会话日志。

### 任务进度可视化
扫描 `tasks/` 目录,统计各任务的 checkbox 状态,可生成简易看板:
```bash
grep -c "☐" tasks/TASK-*.md  # 未完成项数量
grep -c "☑\|\[x\]" tasks/TASK-*.md  # 已完成项数量
```

### 成本控制
- Sub-agent 默认用 sonnet
- 简单任务(读文档、生成模板)可在 prompt 里指定用 haiku
- 复杂决策(架构、Prompt 优化)用 opus
