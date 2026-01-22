# Grok2API Pro

基于 [chenyme/grok2api](https://github.com/chenyme/grok2api) 的增强版本，重点补齐企业部署所需的多代理池、调用日志追踪与更健壮的配置管理能力。本仓库即为 `grok2api-pro` 的源码，默认暴露 OpenAI 兼容接口 (`/v1/*`) 与管理面板 (`/login`).

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
   - Python ≥ 3.13 或 Docker 24+
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
5. 访问 `http://服务器IP:8001/login`，完成后台配置与 token 导入。

## Docker Compose 部署

项目自带 `docker-compose.yml`，默认挂载 `grok_data`（持久化 `data/*`）及 `./logs`。常见自定义方式：

```yaml
environment:
  - STORAGE_MODE=mysql
  - DATABASE_URL=mysql://user:pass@db:3306/grok2api
  - WORKERS=4
volumes:
  - ./data:/app/data      # 确保包含 setting.toml / proxy_state.json / call_logs.json
  - ./logs:/app/logs
```

> **建议**：生产环境使用外部 MySQL/Redis 存储，确保多副本或多进程之间共享 token、配置、代理绑定与调用日志。Docker 部署完成后，可用 `docker compose logs -f grok2api` 观察启动过程是否完成 “调用日志服务启动完成 / 应用启动成功”。

## 配置说明（`data/setting.toml`）

- `[grok]` 节：
  - `api_key`：可选的 API Key 校验；
  - `proxy_url` / `proxy_urls`：静态代理或代理池种子；
  - `proxy_pool_url` + `proxy_pool_interval`：从远程接口周期获取代理；
  - `filtered_tags`、`show_thinking`、`temporary` 等 Grok 调用行为控制；
  - `retry_status_codes`：允许自动重试的 HTTP 状态码（Pro 默认为 `[401,429]`）。
- `[global]` 节：
  - `admin_username` / `admin_password`：后台账号；
  - `base_url`：用于生成图片/视频回调的公网地址；
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

更多实现细节可参考 `docs/PRO_FEATURES.md` 与 `CHANGELOG.md`。

