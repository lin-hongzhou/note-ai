# Codex 项目交接摘要

> 生成时间：2026-08-17
> 用途：将项目迁移到另一台电脑后，帮助新的 Codex 会话恢复工作上下文。
> 当前 Trellis 任务：`personal-productivity-v1`
> 当前状态：`planning`

## 1. 接手规则

接手后先阅读以下文件，再开始任何工作：

1. `AGENTS.md`
2. `CONTEXT.md`
3. `V1-PROJECT-DELIVERY-ROADMAP.md`
4. `.trellis/workflow.md`
5. `.trellis/spec/`
6. `.trellis/tasks/08-13-personal-productivity-v1/prd.md`
7. `.trellis/tasks/08-13-personal-productivity-v1/design.md`
8. `.trellis/tasks/08-13-personal-productivity-v1/implement.md`
9. `.trellis/tasks/08-13-personal-productivity-v1/prototype-handoff.md`

在用户明确批准前：

- 不要运行 `task.py start`；
- 不要开始生产业务代码实现；
- 不要擅自改变 PRD、架构设计或已确认的技术边界；
- 先汇报理解、发现的问题和建议的下一步。

## 2. 项目目标

这是一个跨 Android、Web 用户端和 Web 管理后台的个人效率与个人财务管理应用 V1，核心闭环是：

```text
Android 快速捕捉
→ 本地保存与同步状态
→ Web 集中整理 / 每周回顾 / 收支回顾
→ Android 查看今日重点并继续行动
```

产品包含待办、以后想做、记账、笔记、快速捕捉、AI 整理、待确认收件箱、Android 语音转写、跨设备同步和受控运营后台等范围。

## 3. 已确认的技术边界

- 服务端：Go 模块化单体 + PostgreSQL；V1 不拆微服务。
- API：版本化 REST API，根路径为 `/api/v1`；OpenAPI 是唯一契约。
- 数据访问：`pgx`、版本化 SQL migrations、`sqlc`；核心业务不以 ORM 为主要数据访问层。
- 客户端：Android 使用 Kotlin + XML + MVVM；Web 使用 Vue 3 + TypeScript。
- AI：首期单一 OpenAI 转写服务，并保留可替换的 `TranscriptionProvider` 接口。
- 原始笔记正文未经用户编辑不得由 AI 擅自改写。
- AI 分类或整理结果必须进入待确认收件箱，不得直接成为正式记录。
- 同步冲突、删除恢复、隐私保护和 AI 降级是核心约束，不是后续补充项。

如本文件与正式文档冲突，以 `.trellis/tasks/08-13-personal-productivity-v1/prd.md` 为准。

## 4. 当前规划产物

当前任务目录为：

```text
.trellis/tasks/08-13-personal-productivity-v1/
```

其中：

- `prd.md`：产品需求与验收标准；
- `design.md`：架构、API、数据模型和关键流程设计；
- `implement.md`：分阶段实施计划；
- `prototype-handoff.md`：交给原型设计或其他 AI 的原型交接摘要；
- `task.json`：Trellis 任务状态；
- `implement.jsonl`、`check.jsonl`：任务上下文记录。

## 5. 迁移后第一步

新电脑上的 Codex 应先执行只读检查：

1. 检查 Git 状态和项目目录；
2. 阅读本文件列出的上下文文件；
3. 汇报当前 planning 状态、已完成内容、未完成内容和阻塞问题；
4. 等用户确认后再进行下一步规划或实现。

## 6. 环境与敏感配置

真实 API Key、密码、Token、私钥、生产数据库连接信息不得提交到仓库。

如果项目需要环境变量，应提供 `.env.example`，并在新电脑上手动创建本地 `.env`。

本文件只记录项目上下文，不记录任何凭证或完整 Codex 私人会话内容。

## 7. 本次迁移操作

- 已准备项目级 `.gitignore`；
- 已生成本交接文件；
- 已初始化 Git，并完成迁移检查点提交；
- Git 远程仓库已配置为 `https://github.com/lin-hongzhou/note-ai.git`；
- `main` 已推送到远程；新电脑可执行：`git clone https://github.com/lin-hongzhou/note-ai.git`；
- 不要将旧电脑的用户级 Codex 会话目录、`auth.json`、`secrets/` 或任何会话备份提交到本仓库。

