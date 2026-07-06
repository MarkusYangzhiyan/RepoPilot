# RepoPilot V1.0 候选仓库调研

> 调研日期：2026-07-06
> 调研范围：Click、Typer、Starlette、HTTPX
> 结论用途：选择开发集、验证集、盲测集和高难度扩展集

## 1. 调研目标

本次调研不评价仓库“流不流行”，而是判断它们是否适合构建可复现、可评测的 RepoPilot 软件维护任务。每个仓库按以下维度核验：

1. 许可证是否允许克隆、修改和保存补丁，并明确需要保留的声明。
2. 默认分支、当前 commit 和 Python 版本是否明确。
3. 依赖安装、全量测试、局部测试和质量检查是否有官方入口。
4. 贡献规则是否能转换为 RepoPilot 的确定性验收命令。
5. 历史 Issue 与 PR 是否适合生成固定 base commit 的回归任务。
6. 是否存在操作系统、Shell、网络、异步后端或跨仓库答案泄漏风险。

这里记录的当前 commit 只用于复现本次调研事实。正式评测任务必须另外锁定对应 Issue 修复前的历史 base commit。

## 2. 结论摘要

| 仓库 | 推荐角色 | 许可证 | 默认分支 | 当前 Python 要求 | 可复现性 | 主要风险 |
| --- | --- | --- | --- | --- | --- | --- |
| Click | 主开发集 | BSD-3-Clause | `main` | `>=3.10` | 高 | 终端、Shell completion 和跨平台行为 |
| Typer | 相近领域验证集 | MIT | `master` | `>=3.10` | 中高 | 多 Shell、富文本输出、并行测试、内嵌 Click 代码造成答案泄漏 |
| Starlette | 跨领域盲测集 | BSD-3-Clause | `main` | `>=3.10` | 中高 | ASGI、WebSocket、异步并发和可选依赖提升难度 |
| HTTPX | 高难度扩展集 | BSD-3-Clause | `master` | `>=3.9` | 中 | 本地端口、网络、TLS、代理、HTTP/2 和多异步后端 |

四个仓库当前都覆盖 Python 3.12，并以 pytest 为主要测试体系，符合 RepoPilot V1.0 的准入要求。建议保持原计划中的角色分配，不扩大 V1.0 仓库范围。

## 3. Click

### 3.1 调研快照

- 仓库：[pallets/click](https://github.com/pallets/click)
- 默认分支：`main`
- 调研 commit：[`16fc00e2f4a2717a521084f193709a6058afc693`](https://github.com/pallets/click/tree/16fc00e2f4a2717a521084f193709a6058afc693)
- 许可证：BSD-3-Clause，见 [`LICENSE.txt`](https://github.com/pallets/click/blob/16fc00e2f4a2717a521084f193709a6058afc693/LICENSE.txt)
- Python：`>=3.10`，见 [`pyproject.toml`](https://github.com/pallets/click/blob/16fc00e2f4a2717a521084f193709a6058afc693/pyproject.toml)

### 3.2 官方开发与测试入口

当前 CI 使用 uv、tox 和 pytest。Python 3.12 基线可按以下方式运行：

```bash
uv sync --locked --no-default-groups --group dev
uv run --locked --no-default-groups --group dev tox run -e py3.12
```

局部测试可以通过 tox 的位置参数传给 pytest：

```bash
uv run --locked --no-default-groups --group dev \
  tox run -e py3.12 -- tests/test_arguments.py
```

附加质量入口：

```bash
uv run --locked --no-default-groups --group dev tox run -e style
uv run --locked --no-default-groups --group dev tox run -e typing
```

命令依据见 [`pyproject.toml`](https://github.com/pallets/click/blob/16fc00e2f4a2717a521084f193709a6058afc693/pyproject.toml) 和 [测试工作流](https://github.com/pallets/click/blob/16fc00e2f4a2717a521084f193709a6058afc693/.github/workflows/tests.yaml)。

### 3.3 贡献规则

- 贡献流程遵循 Pallets 通用贡献指南。
- 不建议为可由标准库完成的功能增加依赖。
- 避免覆盖率无法同时测量两个分支的三元表达式。
- Markdown 文档按 80 字符换行。
- 随机并行测试和压力测试属于额外环境，不应作为所有历史任务的默认阻塞项。

来源：[`docs/contributing.md`](https://github.com/pallets/click/blob/16fc00e2f4a2717a521084f193709a6058afc693/docs/contributing.md)。

### 3.4 适用性与风险

Click 代码以纯 Python CLI 逻辑为主，核心测试入口清晰、普通测试不依赖外部服务，适合作为第一个端到端闭环和主要开发集。

主要风险：

- 终端宽度、颜色、编码和标准输入输出在不同平台上可能表现不同。
- Bash、Zsh、Fish、PowerShell 的 completion 问题可能需要专用环境。
- 随机并行和压力测试可能引入非确定性，应与普通回归测试分层运行。

结论：保留为主开发集，Day 4 优先选择一个历史稳定版本和一个能在 Linux + Python 3.12 上复现的历史 Issue。

## 4. Typer

### 4.1 调研快照

- 仓库：[fastapi/typer](https://github.com/fastapi/typer)
- 默认分支：`master`
- 调研 commit：[`7fb978fbf44c776ca7d9441b47c5e64b00740b40`](https://github.com/fastapi/typer/tree/7fb978fbf44c776ca7d9441b47c5e64b00740b40)
- 许可证：MIT，见 [`LICENSE`](https://github.com/fastapi/typer/blob/7fb978fbf44c776ca7d9441b47c5e64b00740b40/LICENSE)
- Python：`>=3.10`，见 [`pyproject.toml`](https://github.com/fastapi/typer/blob/7fb978fbf44c776ca7d9441b47c5e64b00740b40/pyproject.toml)

### 4.2 官方开发与测试入口

CI 使用 uv、pytest、pytest-xdist 和 coverage：

```bash
uv sync --no-dev --group tests
uv run bash scripts/test-files.sh
uv run bash scripts/test.sh
```

局部测试：

```bash
uv run bash scripts/test.sh tests/test_core.py
```

CI 会在多个操作系统及 Python 3.10–3.14 上运行测试，并在合并覆盖率后要求 100% 覆盖率。来源见 [测试工作流](https://github.com/fastapi/typer/blob/7fb978fbf44c776ca7d9441b47c5e64b00740b40/.github/workflows/test.yml)。

### 4.3 贡献规则

- 一般代码贡献遵循 FastAPI/Tiangolo 贡献流程。
- Shell completion 需要使用项目提供的 Docker Compose 环境验证 Bash、Zsh、Fish 和 PowerShell。
- completion 环境需要重新安装 shell completion 并重启对应 Shell。

来源：[`docs/contributing.md`](https://github.com/fastapi/typer/blob/7fb978fbf44c776ca7d9441b47c5e64b00740b40/docs/contributing.md)。

### 4.4 适用性与风险

Typer 与 Click 同属 CLI 领域，但使用类型注解、Rich 输出、Shellingham 和多 Shell completion，适合验证 RepoPilot 能否迁移到相近但不同的代码组织。

主要风险：

- 仓库包含 [`typer/_click`](https://github.com/fastapi/typer/tree/7fb978fbf44c776ca7d9441b47c5e64b00740b40/typer/_click) 内嵌 Click 代码，可能让 Click 开发集与 Typer 验证集发生代码和答案泄漏。
- 富文本终端输出、终端宽度和 completion 测试具有平台差异。
- 默认测试使用 pytest-xdist 并行执行，失败时应支持单进程复跑以区分竞态与真实缺陷。

结论：保留为相近领域验证集，但必须按时间切分任务，并禁止把对应 Click 修复或 Typer 的关联 PR Diff 放入检索上下文。

## 5. Starlette

### 5.1 调研快照

- 仓库：[Kludex/starlette](https://github.com/Kludex/starlette)
- 默认分支：`main`
- 调研 commit：[`5174d4c8358a6f06aa8056bafd14c2272dab8dd1`](https://github.com/Kludex/starlette/tree/5174d4c8358a6f06aa8056bafd14c2272dab8dd1)
- 许可证：BSD-3-Clause，见 [`LICENSE.md`](https://github.com/Kludex/starlette/blob/5174d4c8358a6f06aa8056bafd14c2272dab8dd1/LICENSE.md)
- Python：`>=3.10`，见 [`pyproject.toml`](https://github.com/Kludex/starlette/blob/5174d4c8358a6f06aa8056bafd14c2272dab8dd1/pyproject.toml)

### 5.2 官方开发与测试入口

```bash
scripts/install
scripts/test
scripts/check
```

局部测试：

```bash
scripts/test tests/test_applications.py
```

`scripts/install` 使用 `uv sync --frozen`；`scripts/test` 运行 pytest、检查和覆盖率；`scripts/check` 运行 Ruff 格式检查、Mypy 和 Ruff lint。CI 覆盖 Python 3.10–3.14。来源见 [贡献指南](https://github.com/Kludex/starlette/blob/5174d4c8358a6f06aa8056bafd14c2272dab8dd1/docs/contributing.md) 和 [CI 工作流](https://github.com/Kludex/starlette/blob/5174d4c8358a6f06aa8056bafd14c2272dab8dd1/.github/workflows/main.yml)。

### 5.3 贡献规则

- Bug 和功能建议通常先从 Discussion 开始，再决定是否升级为 Issue。
- Bug 报告应包含操作系统、Python 与依赖版本、最小复现和 traceback。
- PR 必须通过测试、格式、类型检查、Lint 和覆盖率要求。

### 5.4 适用性与风险

Starlette 从 CLI 领域跨到 ASGI Web 框架，可检验混合代码 RAG 和根因调查是否真正具备跨领域迁移能力。

主要风险：

- 异步并发、生命周期、流式响应、WebSocket 和 ASGI 协议问题比 CLI 更难复现。
- 可选依赖与不同事件循环行为可能造成环境差异。
- Starlette 的 TestClient 与 HTTPX 关系紧密，需要按时间过滤跨仓库历史，避免从 HTTPX 变更中泄漏答案。
- 盲测集在冻结前不得查看真实修复 PR Diff。

结论：保留为跨领域盲测集。只有在 Click 与 Typer 流程稳定后才启用，并严格隔离真实 PR 信息。

## 6. HTTPX

### 6.1 调研快照

- 仓库：[encode/httpx](https://github.com/encode/httpx)
- 默认分支：`master`
- 调研 commit：[`b5addb64f0161ff6bfe94c124ef76f6a1fba5254`](https://github.com/encode/httpx/tree/b5addb64f0161ff6bfe94c124ef76f6a1fba5254)
- 许可证：BSD-3-Clause，见 [`LICENSE.md`](https://github.com/encode/httpx/blob/b5addb64f0161ff6bfe94c124ef76f6a1fba5254/LICENSE.md)
- Python：`>=3.9`，见 [`pyproject.toml`](https://github.com/encode/httpx/blob/b5addb64f0161ff6bfe94c124ef76f6a1fba5254/pyproject.toml)

### 6.2 官方开发与测试入口

```bash
scripts/install
scripts/test
scripts/check
```

局部测试：

```bash
scripts/test -- tests/test_multipart.py
```

`scripts/install` 创建虚拟环境并按 `requirements.txt` 安装依赖；`scripts/test` 运行 pytest、检查和覆盖率；`scripts/check` 运行 Ruff、Mypy 与版本同步检查。来源见 [贡献指南](https://github.com/encode/httpx/blob/b5addb64f0161ff6bfe94c124ef76f6a1fba5254/.github/CONTRIBUTING.md) 和 [CI 工作流](https://github.com/encode/httpx/blob/b5addb64f0161ff6bfe94c124ef76f6a1fba5254/.github/workflows/test-suite.yml)。

### 6.3 贡献规则

- 潜在 Bug 和功能建议通常先进入 Discussion。
- Bug 报告需要区分 HTTP/1.1 与 HTTP/2、同步与异步客户端，以及 asyncio 与 trio。
- PR 必须通过测试、格式、类型检查、Lint 和覆盖率要求。

### 6.4 适用性与风险

HTTPX 同时覆盖同步/异步 API、连接池、流式传输、代理、TLS 和 HTTP/2，适合压力测试，但不应作为 V1.0 首批闭环的发布阻塞项。

主要风险：

- 完整测试会启动本地测试服务器并占用 8000、8001 端口。
- 测试配置包含需要网络的 marker；RepoPilot 沙箱默认断网，必须显式跳过或隔离此类测试。
- HTTP/2、SOCKS、Brotli、Zstandard、CLI 等能力使用可选依赖，历史任务必须固定依赖集合。
- TLS、代理、操作系统证书和 asyncio/trio 差异会增加复现成本。

结论：保留为高难度扩展集，只报告结果，不作为 V1.0 发布阻塞项。

## 7. 数据集与防泄漏要求

四个仓库都采用宽松开源许可证，工程上允许克隆、修改和保存候选补丁，但数据集必须保留仓库、commit、许可证和版权来源。此结论用于项目工程选型，不替代正式法律意见。

每条历史任务至少保存：

```text
repository
issue_number
issue_created_at
base_commit
merged_pr
merge_commit
license
test_command
gold_changed_files
difficulty
```

正式采集必须执行：

1. checkout 到真实修复前的 base commit。
2. 只索引该时间点已经存在的代码、文档和历史。
3. 删除 PR 创建后产生的 Issue 评论。
4. 隐藏关联 PR 的标题、描述、评论、commit 和 Diff。
5. Click 与 Typer、Starlette 与 HTTPX 额外执行跨仓库和时间泄漏检查。
6. 盲测集在 V1.0 候选版前不查看真实修复 Diff。

## 8. 最终推荐

| 角色 | 仓库与任务量 | 决策 |
| --- | --- | --- |
| 开发集 | Click 15 个 + Typer 5 个 | 可反复调试 Prompt、检索与图路由 |
| 验证集 | Click 5 个 + Typer 5 个 | 仅用于阶段性模型和策略选择 |
| 盲测集 | Starlette 15 个 | 冻结后隔离真实 PR Diff，用于跨领域评价 |
| 扩展集 | HTTPX 5 个 | 只报告结果，不阻塞 V1.0 发布 |

Day 3 结论是继续执行既定方案。Day 4 只克隆 Click，选择历史版本并在 Python 3.12 环境中记录依赖安装、测试命令、耗时和失败项；不提前扩展到另外三个仓库。
