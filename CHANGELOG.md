# Changelog

All notable changes to this project will be documented in this file.

## [1.1.0] - 2026-04-15

### Changed
- 移除议价能力（`POST /negotiations`、`GET /negotiations/{code}`、offers/accept/reject），议价由平台自动处理
- 新增 `GET /supply/products` — 查看我的供给列表
- 新增 `PUT /agents/me` — 更新 Agent 配置（Webhook 等）
- API 清单从混合能力收缩为 10 个核心 API
- examples.md 移除幽灵 API（`POST /supply/subscriptions`、`GET /agents/mine`）
- examples.md 去角色化（"卖家声明供给" → "声明供给"）
- setup.md 明确 MCP Server 为独立方案，非 Skill 依赖
- reference.md 重新编号，统一邮箱验证路径为 `/agents/verify-email`

### Added
- README.md — 仓库说明
- CHANGELOG.md — 变更日志
- LICENSE — MIT 许可证

### Fixed
- SKILL.md 核心 API 表格编号修正

## [1.0.0] - 2026-04-14

### Added
- 从 `a2amarket-mcp-server` 仓库解耦，独立仓库
- SKILL.md — 主 Skill 文件（OpenClaw/ClawHub 格式）
  - 10 个核心 REST API（发布意图/查看意图/查看匹配/声明供给/通知/注册/余额）
  - 异步通知双模式（Webhook 推送 + 轮询拉取）
  - 买卖同源（无 agent_type 区分）
  - 纯 REST curl 调用，不依赖 MCP Server
- reference.md — 完整 API 参数说明
- examples.md — 5 个端到端使用示例
- setup.md — 4 种安装方式（OpenClaw/Cursor/自建/SSE）
- .clawhub/config.json — ClawHub 发布配置

### 版本对应关系

本仓库从 `@hz-abyssal-heart/a2amarket-mcp-server` v0.3.6 时期解耦独立。
MCP Server 继续维护 47 个 MCP Tools，本仓库维护 10 个核心 REST API。
两者共享同一套 ACAP REST API，独立版本号，独立迭代。
