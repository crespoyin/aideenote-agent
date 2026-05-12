# AideeNote App - Claude Code 工作上下文

> 本文件是所有 Agent 的全局上下文入口。子目录下的 `CLAUDE.md` 只补充领域细节，不重复全局信息。

## 一、产品定位

AideeNote 是一款面向"超级个体"（C 端高价值用户）的 AI 录音硬件 + 配套 App 产品。

- **硬件**：录音设备（蓝牙连接 / 直连传输）
- **配套 App**：本仓库范围。负责录音同步、AI 分析、内容产出、订阅管理
- **核心场景**：地产经纪（带看报告、客户需求分析、PPT 报告）；扩展场景：银行客户经理、保险经纪
- **商业模式**：硬件 899 RMB（含 6 个月 Professional），订阅 Professional 49/月、Unlimited 79/月

## 二、技术栈

- **App 端**：React Native + HarmonyOS Next（双端共享业务逻辑）
- **后端**：Java 17 + Spring Boot 3.x（微服务）+ Go（高并发音频处理）
- **AI 链路**：Anthropic Claude API + 自研 AI Harness + Skill 系统
- **数据**：MySQL（业务）+ Redis（缓存/状态）+ ClickHouse（埋点分析）+ 对象存储（音频）
- **基础设施**：阿里云 ACK（K8s）+ Kafka（异步任务）
- **可观测**：Sentry（前端错误）+ ARMS/SLS（后端日志/链路）

## 三、AI 五层链路（AideeNote 核心）

1. **角色注入**：经纪人 / 客户经理身份与场景
2. **结构化提取**：从音频转写文本中抽取关键信息
3. **谈判信号检测**：意向、抗性、价格敏感度
4. **报告生成**：带看报告 / 客户需求分析 / PPT
5. **跟进策略**：下一步行动建议

每层对应一个 Skill，可独立迭代和灰度。

## 四、Agent 协同模式

本仓库采用 **Sub-agents + Worktree 混合模式**：

- **常规迭代**：主仓内由各 Sub-agent 委派（见 `.claude/agents/`）
- **大特性开发**：开 Worktree 物理隔离，多个 Claude Code 会话并行
  ```bash
  git worktree add ../aideenote-feature-xxx feature/xxx
  ```
- **Lead Agent**：默认主会话扮演 Lead，负责拆解、分配、审查
- **Teammate Agent**：通过 Task 工具委派，独立上下文执行

## 五、Agent 角色清单

| Agent | 职责 | 触发场景 |
|-------|------|----------|
| `pm` | 需求拆解、PRD 解析、任务定义 | 收到 Meego 链接或需求描述 |
| `ux` | Figma 解析、组件规范、交互逻辑 | 收到 Figma 链接或设计稿 |
| `frontend` | RN/HarmonyOS 页面、状态管理、API 对接 | 涉及前端代码改动 |
| `backend` | Java/Go 服务、API、DB、Kafka | 涉及后端代码改动 |
| `qa` | 测试用例、E2E、Sentry 验证 | PR 提交前 / 回归测试 |
| `devops` | CI/CD、K8s、灰度、监控告警 | 部署 / 发布 / 故障排查 |
| `analytics` | 埋点设计、ClickHouse 查询、漏斗分析 | 涉及数据指标 / 埋点 |
| `ai-integration` | Prompt、Skill、Agent 编排、Token 成本 | 涉及 AI 链路改动 |

## 六、关键约束（所有 Agent 必读）

1. **API 契约即合同**：所有跨端接口以 `docs/api/openapi.yaml` 为准。前后端 Agent 修改前必须先更新此文件。
2. **数据库 schema 版本化**：所有 DB 改动走 `db/migrations/` 下的 Flyway 脚本。禁止直接改表。
3. **AI 成本意识**：调 Claude API 必须走 AI Harness 网关，不得在业务代码直连。简单任务用 Haiku，复杂任务用 Sonnet。固定内容（如 Skill 的 SKILL.md）必须开 Prompt Caching。
4. **租户隔离**：所有数据查询必须带 `tenant_id`。Token 配额、限流按租户维度统计。
5. **音频数据合规**：录音文件需用户授权后上传，存储加密，按用户订阅档位决定保留期。
6. **PR 规范**：每个 PR 必须关联一个 Meego 任务 ID；提交前由 QA Agent 跑过测试 + Lead Agent 做架构审查。

## 七、目录约定

```
aideenote-app/
├── apps/
│   ├── mobile-rn/          # React Native App
│   └── mobile-harmony/     # HarmonyOS Next App
├── services/               # 后端微服务
│   ├── ai-harness/         # AI 网关
│   ├── audio-pipeline/     # 音频处理（Go）
│   ├── recording-svc/      # 录音业务（Java）
│   ├── subscription-svc/   # 订阅与支付（Java）
│   └── analytics-svc/      # 埋点上报（Java）
├── docs/
│   ├── api/                # OpenAPI 契约
│   ├── prd/                # PRD 缓存（从 Meego 同步）
│   ├── design/             # 设计稿缓存（从 Figma 同步）
│   └── adr/                # 架构决策记录
├── tasks/                  # 任务交接文档（Agent 间通信）
├── db/migrations/          # Flyway 脚本
└── .claude/
    ├── agents/             # Sub-agent 定义
    └── skills/             # 共享 Skill
```

## 八、任务交接协议

Agent 间不靠口头描述，靠 `tasks/TASK-{meego-id}.md` 交接。模板见 `tasks/_template.md`。

每个任务文档必须包含：输入、约束、验收标准、产出位置、关联 Agent。

## 九、评审协议（契约式互审）

Agent 之间存在接口契约时,产出方写完后必须由消费方签字才算完成。详见 `docs/REVIEW_PROTOCOL.md`。

核心原则:
- 评审矩阵中列出的关系是强制的(例如 backend 的 OpenAPI 必须由 frontend 签字)
- 每个产出最多 2 轮评审,第 3 轮分歧由 Lead 仲裁并写 ADR
- 评审必须给具体修改建议,不接受"看起来不行"
- 签字未齐全的任务,QA 准入直接打回

## 十、外部资源

- Meego（任务管理）：通过 `prd.skill` 拉取
- Figma（设计稿）：通过 MCP 或导出 JSON
- Anthropic API 文档：https://docs.claude.com
