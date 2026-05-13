# Diffgram Plugins for Claude Code

Diffgram 标注系统的 Claude Code plugin — 10 个 AI skill + **OAuth 2.1 认证** MCP connector. 让 AI 工程师在 IDE 里跨项目查询数据 / 复合规则批量打标 / 组织数据集 / 创建标注任务, 不离开 Claude Code.

## 三个变体

| Plugin | 默认地址 | 用途 |
|---|---|---|
| `diffgram` | `https://label.bestfunc.com` | SaaS 占位, 当前未开放 |
| `diffgram-onprem` | `http://192.168.2.176:18085` | **本司生产** — 内网 AI 工程师 / 标注员日常 |
| `diffgram-local` | `http://localhost:18085` | 本机 docker-compose 跑 diffgram dev 时连本机 |

> 其他公司私有化部署不在 `192.168.2.176:18085`? 见下方 [私有化定制](#私有化定制ip端口)。

## 快速安装

```bash
# 添加 marketplace
> /plugin add bestfunc/diffgram_plugin

# 装公司内网生产用 (默认)
> /plugin install diffgram-onprem

# 或本地开发
> /plugin install diffgram-local
```

## 首次使用 — 浏览器一键授权 (v0.2+, OAuth 2.1)

安装完 plugin **不需要手动配置 token**:

1. `/plugin install diffgram-onprem` 装完后, Claude Code 自动检测 MCP server 需要授权
2. 首次调用 `@diffgram` 时, Claude Code 启动浏览器跳转到 diffgram 授权页
3. 浏览器里登录 diffgram 账号 (如未登录), 看到 "{Claude Code} 申请访问 Diffgram", 点 **允许**
4. 浏览器自动跳回 Claude Code, 拿到 access_token (1h) + refresh_token (30d, 自动续期)
5. 直接用: `> @diffgram 列我能访问的项目`

这背后走标准 OAuth 2.1 流程: 401 + WWW-Authenticate → 元数据发现 (RFC 9728 / 8414) →
动态客户端注册 (RFC 7591) → PKCE S256 → token. 不复制 token, 不改 plugin.json.

> 私有化部署 IP/端口/域名不同? 见 [私有化定制](#私有化定制ip端口) — 只改 plugin.json 的 url, OAuth 端点会按相同 host 自动派生.

## 兼容: 旧 API Key (v0.1, dgk-xxx)

历史在 diffgram web UI "我的设置 → API Tokens" 创建的 `dgk-xxx` token 继续可用 (后端做了向下兼容). 仍可手动配:

```json
"mcpServers": {
  "diffgram": {
    "type": "http",
    "url": "http://192.168.2.176:18085/api/v1/mcp",
    "headers": {"Authorization": "Bearer dgk-xxx"}
  }
}
```

新用户**强烈建议**走 OAuth, 体验更好, 不用复制粘贴.

## 私有化定制 (IP / 端口 / 域名)

`diffgram-onprem` 默认指向本司内网 `192.168.2.176:18085`. 如果你公司的 diffgram 部署在别的 IP / 端口 / 域名 (例如 `10.20.30.40:8085` 或 `https://label.your-corp.com`), 装完 plugin 后改这一处即可:

### 改哪个文件

```
~/.claude/plugins/<marketplace-slug>/diffgram-onprem/.claude-plugin/plugin.json
```

(`<marketplace-slug>` 对 `bestfunc/diffgram_plugin` 通常是 `diffgram-plugins`. 不确定的话 `find ~/.claude/plugins -name 'plugin.json' -path '*diffgram-onprem*'` 找一下。)

### 改哪几行

```diff
 "mcpServers": {
   "diffgram": {
     "type": "http",
-    "url": "http://192.168.2.176:18085/api/v1/mcp",
-    "httpUrl": "http://192.168.2.176:18085/api/v1/mcp"
+    "url": "https://label.your-corp.com/api/v1/mcp",
+    "httpUrl": "https://label.your-corp.com/api/v1/mcp"
   }
 }
```

也建议同步把 `homepage` 改成你们的部署地址, web UI 链接才对得上。改完重启 Claude Code 即可。

### 注意点

- **协议要对**: 走 nginx/Caddy 配了 HTTPS 就用 `https://`, 直连容器端口就 `http://`. 协议错会 SSL handshake 失败。
- **路径必须是 `/api/v1/mcp`** (服务端固定). 不要写成 `/api/mcp` 或 `/mcp`。
- **检查防火墙**: `curl -i http://<host>:<port>/api/v1/mcp/whoami -H 'Authorization: Bearer dgk-xxx'`, 期望 `200`。如果 `Connection refused` 是端口没开, 如果 `401 unauthorized` 反而说明连通了, 只是 token 错。
- **集团多部署**: 自己 fork 这个 repo, 改完 commit 后用 `> /plugin add your-org/diffgram_plugin` 装 fork 版, 升级时跑 `git pull` 就行 (不会被上游覆盖)。

## 10 个 Skill 总览

### Reference (纯文档, 不调 tool)

| Skill | 用途 |
|---|---|
| `query-syntax` | 复合查询 `a.xxx and l.xxx` 语法 EBNF + 用例 |
| `data-model` | file / instance / label / wd / attribute 关系图 + 字段速查 |
| `mcp-tools` | 26 个 tool 的 input/output schema 速查 |

### Task (执行特定流程)

| Skill | 用途 |
|---|---|
| `cross-project-explore` | "N 个项目里 X 条件下有多少 wav" 跨项目核对 + 统计 + 抽样 |
| `bulk-mark-by-rule` | "把符合 X 条件的 wav 标 Y" 强制 dry_run + 抽样 + 用户确认三道防线 |
| `organize-dataset` | "把 X 数据移到 train/val/test 目录" 含分层 / random split |
| `multi-project-merge-preview` | 跨项目挑 sup 训练数据时, attribute 标准化建议 + 类别分布 |
| `mark-rule-validation` | 多规则集体验证 — 冲突检测 + 漏覆盖 + 逐条确认 |

### Specialized (深度场景)

| Skill | 用途 |
|---|---|
| `pretrain-unsup-sync` | 预训练 unsup 数据分层抽样 + 防 leakage + 输出 csv 给训练机用 |
| `new-project-onboarding` | 新项目接入 checklist — attribute 覆盖率 + label 一致性 + 整改建议 |

## 典型 demo

```
> @diffgram 在 huakai_999 找 attr.step=1 attr.channel_id=4 但没标 NG_H 的 wav,
            打 NG_H 然后移到 TRAIN_v3 (没有就建)

[AI 自动调:
  query_attributes_schema → query_labels_schema → query_files_count
  → query_files (sample 5 给用户看) → mark_set_label dry_run
  → 用户确认 → mark_set_label confirm
  → directory_get_or_create → move_files_to_dir → 报告]
```

整个流程不离开 Claude Code, 不手动复制 file_id, 不写 SQL.

## 26 个 Tool

按 6 模块分组. 详见 `mcp-tools` skill.

- **query** (8 个, 跨项目, 只读): `query_projects` / `query_directories` / `query_attributes_schema` / `query_labels_schema` / `query_files` / `query_files_count` / `query_file_detail` / `query_instance_stats`
- **mark** (5 个, 单项目, dry_run+confirm): `mark_set_attribute` / `mark_remove_attribute` / `mark_set_label` / `mark_remove_label` / `mark_update_metadata`
- **move** (4 个): `move_files_to_dir` / `move_files_clone_to_dir` / `move_files_remove_from_dir` / `move_files_to_dir_with_query`
- **directory** (5 个): `directory_create` / `directory_get_or_create` / `directory_rename` / `directory_archive` / `directory_delete`
- **task** (3 个): `task_create_for_files` / `task_assign` / `task_query`
- **batch_pipeline** (1 个, 链式 + JSON-path): 一次串多 tool

## 安全模型

- **OAuth 2.1 + PKCE + DCR** (v0.2+, 主要方式) — 浏览器一键授权, access 1h / refresh 30d
- **API Key 模式** (v0.1, 兼容保留) — `dgk-xxx` 长期 token
- **写操作强制 dry_run + request_id 二次确认**
- **跨项目隐式可见性过滤** (`userbase_project.user_id = 当前 token 用户`)
- **单项目写操作 Project_permissions Roles 校验** (admin / Editor)
- **审计日志 `mcp_audit_log`** 90 天保留, 敏感字段脱敏

## 设计文档

完整设计在 diffgram 主仓库:
[`diffgram/desgin_doc/mcp_design.md`](https://github.com/bestfunc/diffgram/blob/main/desgin_doc/mcp_design.md)

## 许可

Apache-2.0

## 反馈

issue: https://github.com/bestfunc/diffgram_plugin/issues
