# Diffgram Plugins for Claude Code

Diffgram 标注系统的 Claude Code plugin — 10 个 AI skill + Bearer token 认证 MCP connector. 让 AI 工程师在 IDE 里跨项目查询数据 / 复合规则批量打标 / 组织数据集 / 创建标注任务, 不离开 Claude Code.

## 三个变体

| Plugin | 部署 | 用途 |
|---|---|---|
| `diffgram` | SaaS (diffgram.bestfunc.com) | 当前未开放, 占位 |
| `diffgram-onprem` | 公司私有化 (176 内网) | **生产用** — 公司 AI 工程师 / 标注员日常 |
| `diffgram-local` | 本地开发 (localhost:8085) | diffgram 主仓库贡献者用 (跑 docker-compose up 时连本机 dev) |

## 快速安装

```bash
# Claude Code
> /plugin add bestfunc/diffgram_plugin

# 选你要的变体 (默认 diffgram-onprem)
> /plugin install diffgram-onprem
```

第一次用要在 diffgram web UI 创建 Bearer token:

```
1. 浏览器打开 https://diffgram.bestfunc.com → 登录
2. 我的设置 → API Tokens → 创建
3. 选 scope (query/mark/move/directory/task), 起个名, 点创建
4. 复制 dgk-xxx 一次, 后续不再可见
5. 配置 ~/.config/claude-code/config:

mcpServers:
  diffgram:
    type: http
    url: https://diffgram.bestfunc.com/api/v1/mcp
    headers:
      Authorization: "Bearer dgk-xxx"

6. 重启 Claude Code, 验证: > @diffgram 列我能访问的项目
```

## 10 个 Skill 总览

### Reference (纯文档, 不调 tool)

| Skill | 用途 |
|---|---|
| `query-syntax` | 复合查询 `a.xxx and l.xxx` 语法 EBNF + 用例 |
| `data-model` | file / instance / label / wd / attribute 关系图 + 字段速查 |
| `mcp-tools` | 25 个 tool 的 input/output schema 速查 |

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

## 25 个 Tool

按 5 模块分组. 详见 `mcp-tools` skill.

- **query** (8 个, 跨项目, 只读): `query_projects` / `query_directories` / `query_attributes_schema` / `query_labels_schema` / `query_files` / `query_files_count` / `query_file_detail` / `query_instance_stats`
- **mark** (5 个, 单项目, dry_run+confirm): `mark_set_attribute` / `mark_remove_attribute` / `mark_set_label` / `mark_remove_label` / `mark_update_metadata`
- **move** (4 个): `move_files_to_dir` / `move_files_clone_to_dir` / `move_files_remove_from_dir` / `move_files_to_dir_with_query`
- **directory** (5 个): `directory_create` / `directory_get_or_create` / `directory_rename` / `directory_archive` / `directory_delete`
- **task** (3 个): `task_create_for_files` / `task_assign` / `task_query`
- **batch_pipeline** (1 个, 链式 + JSON-path): 一次串多 tool

## 安全模型

- **Bearer token 认证** (v0.1) → OAuth 2.1 (v0.4 升级)
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
