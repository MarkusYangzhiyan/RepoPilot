# Click 8.4.2 Python 3.12 基线测试

> 执行日期：2026-07-06
> 目的：验证固定 Click 版本能否在隔离的 Python 3.12 环境中稳定、重复地运行官方默认测试

## 1. 结论

Click 8.4.2 在 Ubuntu 24.04 WSL2、Python 3.12.3 环境中连续两次通过官方默认 pytest 测试。两次测试的通过、跳过、排除和预期失败数量完全一致，未出现真实失败；测试后源码工作区保持干净。

第二次测试的端到端耗时从 19.591 秒降至 10.523 秒，减少 9.068 秒，约快 46%。主要原因是 tox 测试环境已缓存，setup 时间从 8.31 秒降至 1.18 秒。

## 2. 固定源码

| 项目 | 值 |
| --- | --- |
| 仓库 | `https://github.com/pallets/click.git` |
| Tag | `8.4.2` |
| Commit | `b2e30a175449cfda909ee4fbf4a29a6a071cad53` |
| 本地目录 | `~/Private-Project/RepoPilot-Labs/click` |
| Git 状态 | detached HEAD，工作区干净 |
| 克隆方式 | shallow clone，`--depth 1 --single-branch` |

官方来源：[Click 8.4.2 Release](https://github.com/pallets/click/releases/tag/8.4.2)、[`pyproject.toml`](https://github.com/pallets/click/blob/8.4.2/pyproject.toml)。

克隆命令：

```bash
git -c http.version=HTTP/1.1 clone \
  --depth 1 \
  --branch 8.4.2 \
  --single-branch \
  https://github.com/pallets/click.git .
```

版本验证：

```bash
git describe --tags --exact-match
git rev-parse HEAD
git status --short
```

## 3. 执行环境

| 项目 | 版本 |
| --- | --- |
| 操作系统 | Ubuntu 24.04.4 LTS on WSL2 |
| 架构 | x86_64 |
| Kernel | 6.6.87.2-microsoft-standard-WSL2 |
| Git | 2.43.0 |
| Python | 3.12.3 |
| 系统 uv | 0.11.26 |
| tox | 4.52.0 |
| tox-uv-bare | 1.34.0 |
| 项目环境 uv | 0.11.3，由 Click 的锁文件固定 |

测试前已退出 Conda base 环境，避免 Conda Python 干扰 uv 与 tox 的解释器选择。

## 4. 依赖环境

Click 8.4.2 仓库包含 `uv.lock`。环境按锁文件创建，没有重新解析或更新依赖：

```bash
uv sync \
  --python 3.12 \
  --locked \
  --no-default-groups \
  --group dev
```

环境验证：

```bash
.venv/bin/python --version
.venv/bin/tox --version
git status --short
```

依赖同步成功，但本次未单独保存 `uv sync` 的耗时。该缺失不影响测试结果，后续构建自动化基线时应独立记录依赖冷安装耗时。

## 5. 测试命令

两次测试使用完全相同的命令：

```bash
time uv run \
  --locked \
  --no-default-groups \
  --group dev \
  tox run -e py3.12
```

命令执行链：

```text
time → uv run → tox py3.12 环境 → pytest
```

## 6. 测试结果

| 指标 | 第一次（冷启动） | 第二次（热启动） | 是否一致 |
| --- | ---: | ---: | --- |
| passed | 1661 | 1661 | 是 |
| skipped | 24 | 24 | 是 |
| deselected | 31000 | 31000 | 是 |
| xfailed | 1 | 1 | 是 |
| failed | 0 | 0 | 是 |
| pytest 用时 | 9.22 秒 | 8.58 秒 | 允许波动 |
| tox setup | 8.31 秒 | 1.18 秒 | 缓存后下降 |
| tox command | 10.81 秒 | 9.87 秒 | 允许波动 |
| tox 总用时 | 19.12 秒 | 11.05 秒 | 缓存后下降 |
| 外层 `real` | 19.591 秒 | 10.523 秒 | 缓存后下降 |
| `user` | 10.881 秒 | 9.501 秒 | 允许波动 |
| `sys` | 2.586 秒 | 2.560 秒 | 允许波动 |

`31000 deselected` 不是失败。Click 的默认 pytest 配置排除了标记为 stress 的高迭代压力测试；Day 4 验证的是官方默认测试入口。`1 xfailed` 是项目声明的预期失败，也不计入真实失败。

## 7. 可复现性判断

本次基线满足以下条件：

- Tag 和完整 commit 已固定。
- Python、uv、tox 和插件版本已记录。
- 依赖严格使用 `uv.lock`，未修改锁文件。
- 两次测试的结果数量完全一致。
- 两次测试均无真实失败。
- 测试前后 Click Git 工作区均保持干净。
- 冷启动与热启动耗时差异可由 tox 环境缓存解释。

结论：Click 8.4.2 可以作为 RepoPilot 当前环境中的可复现开发基线。

## 8. 已知限制与后续边界

- 当前仓库是浅克隆，只适合固定版本基线；采集历史 Issue 时需要获取对应 base commit 或完整历史。
- 本次没有运行默认排除的 stress 测试，不能据此证明压力测试稳定。
- 当前结果仅覆盖 Ubuntu 24.04 WSL2 x86_64，不代表 Windows、macOS 和其他终端环境结果相同。
- 当前只验证了发布基线，没有选择或修复具体 Issue。
