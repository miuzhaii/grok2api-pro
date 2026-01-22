# Grok2API Pro 功能说明

面向需要更稳定代理策略与可观测性的部署场景，Pro 版本在核心服务周边新增了多项增强能力。本文件对这些增量特性进行拆解，方便快速评估、迁移或二次开发。

## 差异概览

- **调用日志全链路追踪**：`app/services/call_log.py` 提供持久化记录、筛选、统计、清理能力，并在 `app/api/admin/manage.py` 暴露 `/api/logs*` 相关接口。
- **多代理池与 SSO 绑定**：`app/core/proxy_pool.py` 支持静态 proxy、代理池 API、代理列表 (`proxy_urls`) 混合使用，可针对单个 SSO 固定代理并自动健康检查。
- **配置/存储增强**：`data/setting.example.toml` 给出完整示例；`storage_manager` 在 File 模式下额外维护 `proxy_state.json` 与 `call_logs.json`，确保重启后仍可恢复绑定、统计数据。
- **依赖扩展**：`requirements.txt` 新增 `aiohttp-socks`（代理检测）、`pytest`/`pytest-asyncio`/`hypothesis`（自动化验证）等库，方便编写健康检查脚本或集成测试。

## 调用日志服务

- **数据结构**：每条日志保存 `sso`、模型、HTTP 状态码、响应耗时、是否成功、所用代理及生成的媒体 URL，写入 `data/call_logs.json`（默认保留 10,000 条，可由 `global.log_max_count` 调整）。
- **生命周期**：`main.py` 在应用启动时调用 `call_log_service.start()`，定时批量落盘；关闭时会 flush 剩余数据，避免丢失。
- **管理能力**：
  - `GET /api/logs`：按 SSO、模型、时间范围分页查询；
  - `GET /api/logs/stats`：快速查看成功率、今日调用量；
  - `GET /api/logs/models`：返回历史上使用过的模型集合；
  - `DELETE /api/logs?max_count=N`：保留最新 N 条或传入 `0` 清空全部。
- **二次开发建议**：如需打通外部可视化，可在后台接口上方增加 Webhook 推送，或直接读取 `call_logs.json` 构建指标。

## 多代理/代理池

- **静态配置**：`grok.proxy_url` 用于兜底代理；`grok.proxy_urls` 支持预置多个代理地址，启动时自动导入。
- **代理池 API**：`grok.proxy_pool_url` + `grok.proxy_pool_interval`（秒）用于从远端 API 拉取最新代理；若返回值类似代理地址，系统会自动标准化协议头。
- **SSO 绑定**：
  - 后台 `POST /api/proxies/assign` / `.../unassign` 可将指定代理固定到某个 SSO；
  - `proxy_pool` 会在调用失败时自动标记、解绑并尝试切换；
  - `proxy_state.json` 持久化所有代理与绑定关系，方便滚动升级。
- **健康检测**：
  - `POST /api/proxies/test` 以 `https://grok.com` 进行连通性测试；403 被视作“连接正常但被 CF 拦截”。
  - `POST /api/proxies/health/reset` 重置失败计数，适用于人工恢复后的快速验证。
- **选择策略**：若未绑定 SSO，则按健康代理轮询；当所有代理失效时会回落到静态代理或立即触发代理池刷新。

## 配置与存储

- 在首次运行前，建议执行 `cp data/setting.example.toml data/setting.toml` 并按部署环境调整。
- `FileStorage` 模式下需要持久化以下文件：
  - `data/setting.toml`：系统配置；
  - `data/token.json`：账号令牌池；
  - `data/proxy_state.json`：代理列表与 SSO 绑定；
  - `data/call_logs.json`：调用记录；
  - （可选）`data/cache/*`：图片/视频缓存目录。
- 在 `MySQL` / `Redis` 模式下，这些数据会自动同步至数据库，但仍建议保留 `data/` 目录以备灾备恢复。

## 依赖与工具链

- `aiohttp-socks` 为代理健康检测、SOCKS5 访问提供支持；若部署环境不需要，可保持在 requirements 中但无需额外操作。
- `pytest`、`pytest-asyncio`、`hypothesis` 便于编写集成测试，例如验证多代理调度或调用日志端点；示例：
  ```bash
  pytest -k "logs or proxy" tests/
  ```
- 依赖集中在 `requirements.txt` 与 `pyproject.toml` 中，若使用 `uv` 或 `poetry`，请保持两者一致。

## 从开源版迁移

1. **备份数据**：导出原仓库 `data/token.json` 与 `data/setting.toml`。
2. **拉取 Pro 代码**：将备份文件放入新仓库的 `data/` 目录；若已存在默认文件，请覆盖。
3. **环境变量**：沿用原有 `STORAGE_MODE`/`DATABASE_URL`/`WORKERS`，额外根据需要设置 `proxy_urls`、`proxy_pool_url`。
4. **验证**：
   - 启动后检查日志中 “[ProxyPool]”、“[CallLog]” 是否初始化成功；
   - 访问后台确认“调用日志”“代理管理”菜单可正常展示；
   - 若挂载了 docker 卷，确认 `proxy_state.json`、`call_logs.json` 在宿主机可见。
5. **灰度**：可先以 `WORKERS=1` 运行，确认日志与代理绑定正常后再扩容。

如需进一步定制或自动化部署，可结合 `CHANGELOG.md` 了解每个版本新增/调整内容。
