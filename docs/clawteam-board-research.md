# ClawTeam Jira-like Board Research
*2026-03-27 09:42*

## Context
ClawTeam 需要一个 jira-like kanban/sprint board，用于 AI Agent 协作团队项目管理。

## Top Picks

### 1. Plane (推荐)
- GitHub: ~30K stars, Apache 2.0
- 特点: Issues + Cycles (sprints) + Modules + Pages + AI
- 部署: Docker Compose, 1 小时内上线
- 亮点: 唯一支持 air-gapped + 自带 AI (可接自己 API key) + 现代 UX
- 适用: 如果 ClawTeam 需要完整 PM 功能

### 2. Linear-style minimal React clone
- github.com/tuan3w/linearapp_clone (React + Tailwind)
- 适用: 如果要自建极简版，UI 参考价值高

### 3. Linear 本身
- 已推出 AI agent 协作（Build/deploy AI agents alongside teams）
- 25,000+ 产品团队在用
- 不开源，但 API 完善

### 4. OpenProject
- 完整 Gantt + Agile boards + 时间追踪
- 更适合传统 PM 团队

## 建议
ClawTeam 场景下推荐参考 **Plane** 的架构（Issues/Cycles/Modules 三层），
或直接用 Linear API 作为 backend + 自建 Virtual Desk widget 前端。

## 2026-04-06 Update — Atlassian 组织层信号
- 3月 Atlassian 裁员约 1,600 人（~10%），并重组 CTO 架构，明确将资金和组织能力转向 AI + 企业销售。
- 对 ClawTeam 启示：Jira Agents 不只是功能层试验，而是公司级战略重心，竞争会继续加速。
- 机会窗口：Atlassian 体量大、节奏慢，轻量 developer-first 工具（ClawTeam）仍可通过速度和聚焦切入。

## 2026-04-07 Update — Plane AI 关键动态
- Plane 官方 2026-03-03 将 Plane AI 上线到 commercial self-hosted，强调 cloud/self-hosted 功能对齐（full parity）。
- Plane AI 能直接在项目管理流里做自然语言建任务、状态查询、流程自动化（非外挂 chatbot）。
- 对 ClawTeam 启示："AI deep-in-workflow + self-hosted parity" 已成为 Jira 替代品的主卖点，若走 developer-first 需要尽早明确 AI 与部署策略。
- 来源：
  - https://plane.so/blog/plane-ai-is-coming-to-self-hosted-deployments
  - https://plane.so/changelog/2026-02-01-product-tour-try-section-and-plane-ai-updates
