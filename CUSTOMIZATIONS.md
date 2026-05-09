# Customizations

本文档是本仓库同步上游时的本地定制说明。以后如果用户说“按 `CUSTOMIZATIONS.md` 更新”“根据本文件同步上游”或类似要求，应默认按本文档执行。

## 上游信息

- 上游仓库：`https://github.com/wenfxl/openai-cpa.git`
- 上游分支：`upstream/main`
- 本地仓库：`jakcky/openai-cpa`
- 本地分支：通常为 `main`

如果本地还没有 `upstream` remote，先添加：

```bash
git remote add upstream https://github.com/wenfxl/openai-cpa.git
git fetch upstream
```

如果已经存在，只需要：

```bash
git fetch upstream
```

## 给 Codex 的同步要求

当用户要求根据本文档更新时，请按以下原则处理：

- 先读取本文档，再查看 `git status --short`，不要覆盖用户未说明的本地改动。
- 拉取并对比 `upstream/main`，把上游的新功能、修复和版本号更新合入本地。
- 必须保留本文档记录的本地定制，不要被上游文件覆盖掉。
- `README.md` 尽量保持接近上游，不写个人定制说明。
- 敏感信息不得写入仓库，包括 API Token、Webhook Secret、数据库密码、代理订阅和私有配置。
- 同步完成后检查关键差异，确认本地定制仍然存在。
- 如果用户明确要求“重建并启动”，同步后执行 `docker compose build codex-web` 和 `docker compose up -d`。
- 同步和验证完成后，将变更提交并推送到个人仓库 `origin/main`，除非用户明确要求不要提交或不要推送。
- 提交时不要加入未跟踪的临时目录、虚拟环境或备份文件，例如 `.venv-test/`、`.venv/`、`*.portbak`。

建议同步流程：

```bash
git fetch upstream
git diff --stat HEAD..upstream/main
```

合并方式可以根据当前工作区选择。若工作区已有未提交改动，应优先避免破坏现有改动；必要时逐文件从 `upstream/main` 恢复上游版本，再重新应用本文档里的定制。

## 必须保留的本地定制

### 1. Sub2API 只处理 OpenAI Free 账号

文件：

- `utils/core_engine.py`

目的：

Sub2API 仓库巡检、手动测活、自动测活和补货判断，只统计并处理 `platform == openai` 且 `credentials.plan_type == free` 的账号。

这样可以避免把非 OpenAI 账号、非 Free 账号，或其他类型库存混入 OpenAI Free 的有效库存计算和测活清理流程。

必须保留的实现：

- 保留 `_is_openai_free_sub2api_account(item)` 作为统一判断函数。
- 判断条件必须是：
  - `platform` 转小写后等于 `openai`
  - `credentials.plan_type` 转小写后等于 `free`
- `process_sub2api_worker()` 开始测活前必须调用该函数；不符合条件的账号跳过，不调用 `client.test_account(account_id)`。
- `perform_sub2api_check()` 必须使用该函数过滤库存列表。
- `sub2api_main_loop()` 的自动测活分支必须使用该函数过滤库存列表。
- `sub2api_main_loop()` 的关闭自动测活分支也必须使用该函数过滤库存列表。
- 本地过滤逻辑不要额外要求 `extra.codex_5h_window_minutes == 0`。

同步后检查：

```bash
grep -n "_is_openai_free_sub2api_account\|process_sub2api_worker\|filtered_list" utils/core_engine.py
```

需要确认：

- `_is_openai_free_sub2api_account()` 仍然存在。
- `process_sub2api_worker()` 中非 OpenAI Free 账号会在测活前跳过。
- 三处 `filtered_list` 都使用 `_is_openai_free_sub2api_account(item)`。
- 没有恢复 `codex_5h_window_minutes == 0` 作为过滤条件。

如果上游改动了 Sub2API 库存结构、`credentials.plan_type` 字段名或测活流程，需要重新确认这个函数是否仍覆盖目标账号。

### 2. Docker 使用本地源码构建

文件：

- `docker-compose.yml`

目的：

个人仓库使用本地源码构建镜像，而不是直接拉取上游公开镜像，确保本地定制代码会进入容器。

必须保留的实现：

- `codex-web` 使用 `build: .` 从当前仓库构建。
- 镜像名固定为 `jakcky/openai-cpa:local`。
- Web Console 端口映射为 `18000:8000`，避免占用宿主机默认 `8000` 端口。
- 保留 `./data:/app/data` 持久化挂载。
- 保留 `/var/run/docker.sock:/var/run/docker.sock`，用于容器内 Docker 管理能力。
- 不要恢复上游 compose 中的 `/usr/bin/docker:/usr/bin/docker` 挂载。
- 不要恢复上游 compose 中的 `.:${PWD}` 挂载。

同步后检查：

```bash
sed -n '1,60p' docker-compose.yml
```

需要确认：

- 不要把 `image: wenfxl/wenfxl-codex-manager:latest` 覆盖回本地 compose。
- 不要把端口映射改回 `8000:8000`。
- 如果上游更新 Dockerfile、依赖或启动参数，同步后重新构建镜像。

常用命令：

```bash
docker compose build codex-web
docker compose up -d
docker compose ps
docker compose logs --tail=80 codex-web
```

## 验证清单

同步完成后至少执行：

```bash
git diff --check
python3 -m py_compile utils/config.py utils/auth_pipeline/register.py utils/core_engine.py
git status --short
```

如果当前环境没有 `python3` 或相关依赖，说明原因即可。不要为了验证而写入敏感配置。

如果安装了测试依赖，可以额外执行：

```bash
python3 -m pytest
```

## 提交和推送

同步完成并通过必要验证后，默认把相关改动提交并推送到个人仓库：

```bash
git status --short
git add CUSTOMIZATIONS.md config.example.yaml index.html static/js/app.js utils/auth_pipeline/register.py utils/config.py utils/core_engine.py docker-compose.yml
git commit -m "sync upstream and keep local customizations"
git push origin main
```

提交前需要确认暂存区只包含本次同步和本文档要求保留的本地定制。不要提交本地虚拟环境、缓存、备份文件、敏感配置或运行时数据。

## 差异记录

| 日期 | 类型 | 差异内容 | 影响范围 | 验证 |
| --- | --- | --- | --- | --- |
| 2026-05-09 | Sub2API | 记录 OpenAI Free 账号过滤定制，统一说明只统计并处理 `platform=openai` 且 `credentials.plan_type=free` 的账号。 | `utils/core_engine.py` | 已对照本地与 `upstream/main` diff |
| 2026-05-09 | Docker | 记录个人 Docker Compose 构建设置：本地构建 `jakcky/openai-cpa:local`，端口 `18000:8000`。 | `docker-compose.yml` | 已对照本地与 `upstream/main` diff |
| 2026-05-10 | 文档 | 增加给 Codex 的上游同步流程、保留要求和验证清单，方便后续直接按本文档更新。 | `CUSTOMIZATIONS.md` | 已整理为可执行说明 |
| 2026-05-10 | Git | 增加同步完成后默认提交并推送到个人仓库 `origin/main` 的要求。 | `CUSTOMIZATIONS.md` | 本次记录后执行提交和推送 |
