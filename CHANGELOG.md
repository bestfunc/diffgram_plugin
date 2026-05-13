# Changelog

## 0.2.0 (2026-05-13) — OAuth 2.1 一键授权

### 新增
- **OAuth 2.1 + PKCE + RFC 7591 动态客户端注册** — Claude Code 装完 plugin 后浏览器一键授权, 不再手动复制 token. 见 README "首次使用 — 浏览器一键授权" 段
- 后端配套实现: `/.well-known/oauth-protected-resource` + `/.well-known/oauth-authorization-server` + `/oauth/register` + `/oauth/authorize` + `/oauth/token` + `/oauth/revoke`
- 前端配套实现: 授权确认页 `/mcp/oauth/authorize` (Vue SPA)
- access_token TTL 1h, refresh_token TTL 30d (自动续期)

### 改
- README 主流程改成 OAuth, 旧 API Key 模式移到"兼容"段
- 不再需要在 plugin.json 写 `headers.Authorization`, 安装后即用

### 兼容
- 旧 `dgk-xxx` API Key 继续可用, 后端 token 鉴权统一入口透明处理两种 kind
- plugin.json 字段未变, 升级 plugin 后无需重装

## 0.1.1 (2026-05-07) — 地址修正 + 私有化文档

### 改
- `diffgram` (SaaS) URL: `diffgram.bestfunc.com` → `label.bestfunc.com`
- `diffgram-onprem` URL: 占位的 `diffgram.bestfunc.com` → 本司实际 `192.168.2.176:18085`
- `diffgram-local` URL: `localhost:8085` → `localhost:18085` (与 docker-compose 默认对齐)

### 加
- README 加 [私有化定制 (IP / 端口 / 域名)](README.md#私有化定制ip端口) 段, 教其他公司私有化部署用户改 plugin.json url
- README token 配置加方式 A (`DIFFGRAM_TOKEN` 环境变量)

## 0.1.0 (2026-05-07) — 初版

### 新增
- 3 个 plugin 变体: `diffgram` (SaaS 占位) / `diffgram-onprem` (公司私有, 主用) / `diffgram-local` (本地 dev)
- 10 个 AI skill (3 reference + 5 task + 2 specialized)
- 26 个 MCP tool 设计 (查询 8 + 标记 5 + 移动 4 + 目录 5 + 任务 3 + batch_pipeline 1, 详见 `mcp-tools` skill)
- Bearer token 认证, 90 天 TTL, 用户在 web UI 自助管理
- 写操作强制 `dry_run + request_id + confirm` 三道防线

### 已知限制
- v0.1 仅 API Key 认证, OAuth 2.1 在 v0.4 计划升级
- 跨项目仅支持 query 模块, mark/move/directory/task 单项目操作
- `dG_aware` SpecAug 等高级 filter 暂不在 query 语法范围
- file_ids 单 tool 调用上限 100,000

### 后续 (v0.2+)
- v0.2: skill 实战验证 + 用户反馈迭代 + bug fix
- v0.3: 加 `analytics` skill (训练数据集时序统计 / 标注质量趋势)
- v0.4: OAuth 2.1 + PKCE 升级 + plugin.json 改 OAuth 模式
- v0.5: 添 `assistant` skill (自然语言需求 → 多 tool 流程编排)
