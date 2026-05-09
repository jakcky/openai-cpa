# Customizations

本文档用于记录本仓库相对上游项目必须保留的本地差异。同步 `upstream/main` 前先查看本文件，避免覆盖个人定制。

## 保留原则

- `README.md` 尽量保持接近上游，不写个人定制说明。
- 本文件只记录和上游不同、后续合并时需要保留的改动。
- 敏感信息不写入仓库，包括 API Token、Webhook Secret、数据库密码和代理订阅。

## OpenAI Free 过滤定制

### 文件

- `utils/core_engine.py`

### 目的

Sub2API 仓库巡检、手动测活和补货判断只统计并处理 `platform == openai` 且 `credentials.plan_type == free` 的账号。

这样可以避免把非 OpenAI 账号、非 Free 账号，或其他类型库存混入 OpenAI Free 的有效库存计算和测活清理流程。

### 本地差异

- 新增 `_is_openai_free_sub2api_account(item)` 作为统一判断函数。
- `process_sub2api_worker()` 开始测活前先调用该函数；不符合条件的账号直接跳过，并输出提示日志。
- `perform_sub2api_check()` 使用该函数过滤库存列表。
- `sub2api_main_loop()` 的自动测活分支和关闭自动测活分支都使用该函数过滤库存列表。
- 本地过滤逻辑不再额外要求 `extra.codex_5h_window_minutes == 0`。

### 合并上游时注意

- 如果上游改动了 Sub2API 库存结构、`credentials.plan_type` 字段名或测活流程，需要重新确认 `_is_openai_free_sub2api_account()` 是否仍然覆盖目标账号。
- 如果上游恢复了 `extra.codex_5h_window_minutes == 0` 条件，需要确认是否会误排除个人仓库需要统计的 OpenAI Free 账号。

## Docker 镜像构建定制

### 文件

- `docker-compose.yml`

### 目的

个人仓库使用本地源码构建镜像，而不是直接拉取上游公开镜像，确保本地定制代码会进入容器。

### 本地差异

- `codex-web` 使用 `build: .` 从当前仓库构建。
- 镜像名固定为 `jakcky/openai-cpa:local`。
- Web Console 端口映射为 `18000:8000`，避免占用宿主机默认 `8000` 端口。
- 保留 `./data:/app/data` 持久化挂载。
- 保留 `/var/run/docker.sock:/var/run/docker.sock`，用于容器内 Docker 管理能力。
- 移除上游 compose 中的 `/usr/bin/docker:/usr/bin/docker` 和 `.:\${PWD}` 挂载。

### 常用命令

```bash
docker compose build codex-web
docker compose up -d
docker compose logs -f codex-web
```

### 合并上游时注意

- 不要把 `image: wenfxl/wenfxl-codex-manager:latest` 覆盖回本地 compose。
- 不要把端口映射改回 `8000:8000`，除非明确需要占用宿主机 8000。
- 如果上游更新 Dockerfile、依赖或启动参数，合并后需要重新执行 `docker compose build codex-web`。

## 差异记录

| 日期 | 类型 | 差异内容 | 影响范围 | 验证 |
| --- | --- | --- | --- | --- |
| 2026-05-09 | Sub2API | 记录 OpenAI Free 账号过滤定制，统一说明只统计并处理 `platform=openai` 且 `credentials.plan_type=free` 的账号。 | `utils/core_engine.py` | 已对照本地与 `upstream/main` diff |
| 2026-05-09 | Docker | 记录个人 Docker Compose 构建设置：本地构建 `jakcky/openai-cpa:local`，端口 `18000:8000`。 | `docker-compose.yml` | 已对照本地与 `upstream/main` diff |
