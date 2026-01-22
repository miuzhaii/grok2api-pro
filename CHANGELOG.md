# Changelog

遵循 Keep a Changelog 约定，所有显著变更都会记录在此。版本号与 `pyproject.toml` 对齐。

## [1.4.3] - 2026-01-22

### Added
- **调用日志服务**：新增 `app/services/call_log.py` 及后台 `/api/logs*` 接口，默认以批量写入方式持久化 `data/call_logs.json`。
- **多代理池 / SSO 绑定**：`app/core/proxy_pool.py` 支持同时维护静态代理、代理池 API 与 `proxy_urls` 列表，附带 SSO 绑定、失败熔断与 `proxy_state.json` 持久化。
- **代理管理面板**：后台新增 `/api/proxies`、`/api/proxies/assign`、`/api/proxies/test` 等端点，可视化操作代理、执行健康检测。
- **配置示例**：提供 `data/setting.example.toml`，便于在容器化/流水线场景下模板化配置。

### Changed
- **启动流程**：`main.py` 先初始化存储，再加载配置、代理、调用日志，并在退出阶段倒序关闭，保证文件模式与多进程一致性。
- **依赖**：`requirements.txt` 新增 `aiohttp-socks`、`pytest`、`pytest-asyncio`、`hypothesis`，方便代理检测和单元测试；`pyproject.toml` 中 `version` 升级至 `1.4.3`。
- **配置项**：`app/core/config.py` 支持 `proxy_urls`、`log_max_count`、`retry_status_codes` 等新字段，同时自动规范 `socks5`/`cf_clearance`。

### Removed
- 默认仓库不再直接提交运行时生成的 `data/setting.toml` 与 `token.json`，改为在首次启动或复制示例文件后再生成，避免误提交。

> 历史版本沿用上游 `chenyme/grok2api`，如需查看 1.4.3 之前的变更，请参考上游仓库对应的 `git log`。
