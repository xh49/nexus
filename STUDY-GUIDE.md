# Nexus — 精读路线

> 优先级：**② 第二周**  
> 仓库：https://github.com/rattleai/nexus  
> 本地路径：`../nexus/`

## 一、项目是做什么的

**一句话**：可 fork 的 **多租户 AI SaaS 平台底座** — 你写 Plugin，平台提供 Agent 运行时、RAG、鉴权、计费、限流、审计、MCP/A2A。

**解决什么问题**：
- 从零搭企业 Agent 平台要 3 个月 → Nexus 提供 **租户隔离、Agent 治理、RAG 管线、AI Gateway** 开箱即用
- 避免每个产品重复造：JWT、RLS、Redis 限流、文档 ingestion、eval harness

**Java 类比**：**Spring Cloud 多租户 SaaS 脚手架** + **LangGraph Agent Runtime** + **Elasticsearch 混合检索**

**不是**：单一 RAG Demo；是 **平台 + 插件契约**

---

## 二、整体架构

```text
React SPA / MCP Client
        ↓
    nginx (TLS, 全局限流)
        ↓
FastAPI (app/main.py)
  Middleware: request-id → CSRF → body cap → rate limit → auth → SET tenant_id
        ↓
┌───────────────┬────────────────┬─────────────────┐
│ Core Routes   │ Plugin Routers │ MCP Bridge      │
│ auth/billing  │ app/apps/*     │                 │
└───────┬───────┴────────┬───────┴─────────────────┘
        ↓                ↓
  Agent Runtime    AI Gateway (LiteLLM)
  ReAct + governance + wallet + prompt firewall
        ↓
PostgreSQL (RLS + pgvector) | Redis | S3
        ↓
Celery Workers (ingestion, async jobs)
```

---

## 三、各环节技术栈

| 环节 | 路径 | 技术 | 解决什么 |
|------|------|------|----------|
| 多租户 | app/db/session.py, core/tenant.py | PostgreSQL RLS | 租户数据隔离 |
| 认证 | app/api/deps.py, auth_routes.py | JWT RS256, OAuth, API Key, WebAuthn | AuthN |
| 授权 | scopes, capabilities | Cedar connector authz | AuthZ |
| 限流 | app/api/rate_limit.py | Redis 滑动窗口 | 防滥用、配额 |
| Agent | app/agents/runtime.py | ReAct loop | 工具调用循环 |
| 编排 | app/agents/orchestrator.py | supervisor/pipeline/DAG | 多 Agent |
| 治理 | app/agents/governance.py | 审批门、spend cap、tool allowlist | Agent 边界 |
| RAG 入库 | app/docprocessor/ | 解析/chunk/embed | 文档管道 |
| RAG 检索 | app/agents/search.py | pgvector + FTS + RRF | Hybrid 检索 |
| 高级 RAG | hyde.py, agentic_rag.py, corrective_rag.py | 多种策略 | 检索质量 |
| 评测 | app/evaluation/ | RAGAS + IR 指标 + LLM judge | 离线 eval |
| AI 网关 | app/ai/gateway.py | LiteLLM | 多模型路由、计量 |
| 插件 | app/plugins/base.py | 14 hooks | 业务扩展点 |
| 前端 | frontend/ | React 19 + TanStack | 管理控制台 |
| 部署 | infra/ | Docker/K8s/Terraform | 生产部署 |

---

## 四、请求生命周期

```text
1. nginx 终止 TLS，全局限流
2. FastAPI middleware 链解析 JWT/API Key
3. set_tenant_context() → PostgreSQL RLS 生效
4. 路由到 core 或 plugin handler
5. Agent 任务 → runtime ReAct → gateway 调 LLM → tool sandbox 执行
6. 长任务 → Celery；worker 重建 tenant 上下文
7. 审计/计量写入 DB + Prometheus
```

---

## 五、精读顺序

| 天 | 文件 | 关注点 |
|----|------|--------|
| D1 | README.md + docs/ARCHITECTURE.md | 平台全景 |
| D2 | app/main.py | 启动、middleware、plugin 发现 |
| D3 | app/db/session.py + tests/test_rls_policies.py | **RLS 多租户** |
| D4 | app/api/deps.py + app/api/rate_limit.py | 认证 + 限流 |
| D5 | app/agents/runtime.py | Agent ReAct 主循环 |
| D6 | app/agents/governance.py | 审批/ spend cap |
| D7 | app/agents/search.py + docprocessor/ | Hybrid RAG |
| D8 | app/ai/gateway.py | LLM 网关 |
| D9 | app/evaluation/ | Eval harness |
| D10 | app/apps/example/plugin.py | 插件怎么写 |
| D11 | docs/PLUGINS.md | 插件契约 14 hooks |

---

## 六、本地运行

```bash
cd nexus
make dev-up    # Docker Compose
make seed-docker
# 见 README Quick Start
```

资源需求较高（Postgres + Redis + 多服务），建议 Docker 16GB+ 内存。

---

## 七、面试话术

> Nexus 代表 **Agent 平台层**：租户 RLS、Redis 限流、Agent governance（审批+预算）、Hybrid RAG（pgvector+FTS+RRF）、LiteLLM 网关、插件化扩展。学它理解 **「不是一个 Graph，而是一整套 SaaS」**。

---

## 变更记录

| 日期 | 说明 |
|------|------|
| 2026-08-01 | 初版 |
