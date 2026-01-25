# Grok2API Pro

基于 [chenyme/grok2api](https://github.com/chenyme/grok2api) 的增强版本，重点补齐企业部署所需的多代理池、调用日志追踪与更健壮的配置管理能力。本仓库即为 `grok2api-pro` 的源码，默认暴露 OpenAI 兼容接口 (`/v1/*`) 与管理面板 (`/login`).

## 代理规则速览

- **自动绑定**：有 SSO 时，优先用已绑定且健康的代理；无绑定则从健康代理中轮询选择并自动绑定。
- **健康切换**：连续失败 3 次标记不健康并解绑所有 SSO；成功会恢复健康并清零失败计数。
- **代理来源**：支持静态代理 `proxy_url`、多代理 `proxy_urls`、代理池 `proxy_pool_url`，统一进入代理池管理。

详细说明见 [DOCS/PROXY_POLICY.md](DOCS/PROXY_POLICY.md)。

## 更新日志

请查看 [CHANGELOG.md](CHANGELOG.md) 了解每次版本的新增与修复。

## 与grok2api版的差异

| 能力/特性 | grok2api | grok2api-pro |
| --- | --- | --- |
| 多代理池 & SSO 绑定 (`app/core/proxy_pool.py`) | 静态代理或单一代理池 | 支持多代理URL、轮询、健康检查、SSO 绑定及持久化 `proxy_state.json` |
| 调用日志服务 (`app/services/call_log.py`) | 无 | 记录每次请求的模型/耗时/状态，后台提供 `/api/logs*` 检索、统计、清理接口 |
| 配置示例 (`data/setting.example.toml`) | 仅运行时生成 `setting.toml` | 内置示例文件，便于在 CI/CD 中挂载或渲染配置 |
| 依赖扩展 | 基础运行库 | 新增 `aiohttp-socks`、`pytest*`、`hypothesis` 等，满足代理连通检测与自动化测试 |
| 管理后台增强 | Token & 设置 | 新增调用日志、代理管理（添加/绑定/测试）、SSO 标签与备注能力 |

> **提示**：Pro 版本在 `main.py` 中启用了 uvloop、MCP HTTP 服务、批量保存任务及调用日志生命周期，请按需暴露 `/health` 等监控端点。

## 快速部署

1. **准备运行环境**
   - Python ≥ 3.13
   - Git、Make（可选）、能够访问 `grok.com` 的网络环境
2. **克隆代码并安装依赖**
   ```bash
   git clone https://github.com/miuzhaii/grok2api-pro.git
   cd grok2api-pro
   python -m venv .venv && source .venv/bin/activate
   pip install -r requirements.txt
   ```
3. **初始化配置**
   ```bash
   cp data/setting.example.toml data/setting.toml
   # 编辑 data/setting.toml，至少更新 admin_password / api_key / x_statsig_id
   ```
4. **启动服务**
   ```bash
   export STORAGE_MODE=file   # 或 mysql / redis
   export WORKERS=1           # 多 worker 时建议改用 MySQL / Redis
   uvicorn main:app --host 0.0.0.0 --port 8001
   ```
5. **管理后台初始化**
   - 访问 `http://服务器IP:8001/login`
   - 登录后在“系统设置”里补齐 `api_key` / `x_statsig_id` / 代理配置
   - 在“Token 池”批量导入 SSO Token 并测试

## 部署建议

- **持久化数据**：`data/` 目录需长期保存（`setting.toml` / `token.json` / `proxy_state.json` / `call_logs.json`）。  
- **多进程/多实例**：建议使用 MySQL 或 Redis 存储，避免 file 模式下数据不一致。  
- **日志排查**：服务启动后观察日志中 “应用启动成功 / 调用日志服务启动完成”。  

## 配置说明（`data/setting.toml`）

- `[grok]` 节：
  - `api_key`：可选的 API Key 校验；
  - `proxy_url` / `proxy_urls`：静态代理或代理池种子；
  - `proxy_pool_url` + `proxy_pool_interval`：从远程接口周期获取代理；
  - `filtered_tags`、`show_thinking`、`temporary` 等 Grok 调用行为控制；
  - `retry_status_codes`：允许自动重试的 HTTP 状态码（Pro 默认为 `[401,429]`）。
  - `max_tls_retries`：`curl_cffi` 偶发 TLS 握手错误（如 `curl: (35)`）的最大重试次数（默认 `2`）。
- `[global]` 节：
  - `admin_username` / `admin_password`：后台账号；
  - `base_url`：用于生成图片/视频回调的公网地址；
  - `token_refresh_interval` / `token_refresh_scope` / `token_zero_expire_threshold`：Token 状态定时刷新与连续 0 次失效规则；
  - `log_max_count`：调用日志最大条目（默认 1w，超限自动裁剪）；
  - `image_cache_max_size_mb` / `video_cache_max_size_mb`：缓存上限。

修改配置后可在后台 “系统设置” 页面保存，或手动编辑文件并重启服务。

## 环境变量

| 变量 | 说明 | 示例 |
| --- | --- | --- |
| `STORAGE_MODE` | `file` (默认) / `mysql` / `redis` | `mysql` |
| `DATABASE_URL` | 连接串；MySQL `mysql://user:pass@host:3306/db`，Redis `redis://user:pass@host:6379/0` | `mysql://grok:pass@db:3306/grok2api` |
| `WORKERS` | `uvicorn` worker 数；>1 建议非 file 存储 | `4` |
| `PYTHONPATH` | 可选，定制模块搜索路径 | `/app` |

## 多代理与调用日志

- 多代理：在 `grok` 配置中填入 `proxy_urls`（列表）或启用 `proxy_pool_url`，启动后后台 “代理管理” 可添加/绑定代理、对单个 SSO 分配/解绑、执行在线连通性测试（依赖 `aiohttp-socks`）。
- 调用日志：`app/services/call_log.py` 将日志写入 `data/call_logs.json`，可通过 `/api/logs`、`/api/logs/stats`、`/api/logs/models` 查询并在后台快速筛选、导出或清理。
- 以上两种数据都会随 `storage_manager` 同步（File 模式下对应 `proxy_state.json`、`call_logs.json`），将 `data/` 目录持久化即可保留历史记录。

## 管理后台亮点

- **Token 池**：批量导入、标签、备注、测试及调用占用情况。
- **调用日志**：按 SSO/模型/时间范围过滤，支持统计总览与一键清除；所有数据均脱敏显示。
- **代理管理**：REST API (`/api/proxies*`) 与前端面板覆盖新增/删除/绑定/健康重置/可用性检测。
- **系统设置**：实时修改 `global` 与 `grok` 配置，无需手动编辑文件。

## 故障排查

1. **403 / 429 频繁**：确认 `proxy_pool` 是否提供高可用 IP，必要时在后台为关键 SSO 绑定独立代理；同时调整 `retry_status_codes` 与 `proxy_pool_interval`。
2. **调用日志无数据**：检查 `data/call_logs.json` 是否可写，及 `log_max_count` 是否被设为 0；查看 `app/logs` 中的 `[CallLog]` 输出。
3. **多进程运行异常**：确保 `STORAGE_MODE` 使用 MySQL/Redis，并挂载 `data/` 目录让 proxy/log 状态同步；避免多个实例同时使用本地 file 存储。
4. **图片/视频 403**：确认 `global.base_url` 指向能被终端访问的域名/IP，且外网能访问 `http://base_url/images/*`。

更多实现细节可参考 [DOCS/PRO_FEATURES.md](DOCS/PRO_FEATURES.md) 与 [CHANGELOG.md](CHANGELOG.md)。

