# 供应链补查 v4

基准 SHA：`9ed84f99c8f13fc921baebc59e0c696108cfab3e`（https://github.com/usestrix/strix/commit/9ed84f99c8f13fc921baebc59e0c696108cfab3e）。未改这个 SHA。未安装依赖，未拉镜像层，未跑脚本或 CI。已写过的事实只引用 `audit/supply-chain.md`，这里补缺口。

新查询时间：2026-11-26 11:31–11:32 香港时间（Asia/Hong_Kong）。OSV：`https://api.osv.dev/v1/querybatch` 与 `/v1/vulns/{id}`。PyPI：`https://pypi.org/pypi/<name>/json`。GitHub 仓库元数据：`https://api.github.com/repos/<owner>/<repo>`。镜像索引仍沿用 `audit/supply-chain.md` 记录的 2026-11-26 09:39 香港时间查询，本轮没有重拉。

## 总表

| 项 | 结论 |
|---|---|
| 1 清单与 lockfile（含间接） | 可疑 |
| 2 版本钉、校验和、git／URL 直装 | 已确认 |
| 3 仿冒名、安装钩子、非官方源 | 未发现问题 |
| 4 容器镜像来源、tag、digest | 已确认 |
| 5 子模块、下载脚本、curl\|bash、运行时远程拉取 | 已确认 |
| 6 CI：Action SHA、密钥、危险触发 | 未发现问题 |
| 7 公开漏洞库：恶意包／已知漏洞 | 已确认 |
| 8 主要依赖维护状态 | 未发现问题 |

## 1. 清单与 lockfile

结论：可疑。三份锁覆盖各自间接依赖；沙箱镜像构建没有锁。

- Python：`uv.lock` 有 120 个 `[[package]]`，其中 119 个第三方，来源均为 `registry = "https://pypi.org/simple"`（首条在 `uv.lock` 19 行附近）。锁是完整解析图，含间接依赖。本包 `strix-agent` 为 `editable = "."`，无发布哈希。
- Go：`strix/interface/tui/go.mod` 5–31 行列出直接与间接模块，`go.sum` 61 行带 `h1:`。
- npm：`strix/interface/viewer/frontend/package-lock.json` 第 3 行 `lockfileVersion` 3，`packages` 下 284 个包。
- 无锁：`containers/Dockerfile` 16–20、114–119、173–178 行的 `go install @latest`、`npm install -g`、`git clone` 没有 lockfile。apt 包同样无锁。

## 2. 版本钉、校验和、git／URL 直装

结论：已确认。锁内有校验和且无 git／URL 直装；镜像构建和安装脚本多处不钉版本、不校验。

- `uv.lock` 第三方条目带 sha256。`go.sum` 带哈希。npm 锁的 `resolved` 抽查均为 `registry.npmjs.org`，除下列 6 个嵌套包外都有 `integrity`：`@emnapi/core`、`@emnapi/runtime`、`@emnapi/wasi-threads`、`@napi-rs/wasm-runtime`、`@tybys/wasm-util`、`tslib`（均在 `@tailwindcss/oxide-wasm32-wasi` 的嵌套 `node_modules`）。这 6 个的完整性未核实。
- 锁内没有 git／URL 直装。pre-commit 的远端钩子钉的是 tag，不是提交 SHA（`.pre-commit-config.yaml` 6、29、43 行）。
- 无校验和或浮动版本的路径已列在 `audit/supply-chain.md`「构建期下载」：`containers/Dockerfile` 16–20、96、114–119、188 行，`scripts/install.sh` 154–168 行。`pyproject.toml` 38 行 `litellm>=1.101.0` 无上界。

## 3. 仿冒名、安装钩子、非官方源

结论：未发现问题。范围是三份锁和直接依赖名，不是镜像里 `git clone` 的仓库名。

- 119 个 PyPI 名与 284 个 npm 包的源都是官方索引。`uv.lock` 没有第二个 registry。`strix/interface/viewer/frontend/.npmrc` 只有 `min-release-age=5`，没有自定义 registry。
- 非常见名 `griffelib`、`librt`、`ast-serialize`、`python-discovery`、`hf-xet`、`caido-sdk-client` 的 PyPI `summary` 与主页指向 mypy、Hugging Face、Caido、OpenAI Agents 相关项目（2026-11-26 11:32 香港时间）。未见仿冒热门包的名字。
- 安装钩子：`package-lock.json` 里只有 `esbuild`、`fsevents` 带 `hasInstallScript`。脚本正文未打开，不把钩子本身写成恶意。Hatch 钩子见 `audit/supply-chain.md`。

## 4. 镜像

结论：已确认。源码未钉 digest；层未拉取，层内容未核实，不得写成安全。

- 证据与 2026-11-26 09:39 香港时间的 GHCR 索引摘要见 `audit/supply-chain.md`「镜像」。源码位置：`strix/config/settings.py` 187–189 行、`scripts/install.sh` 第 7 行、`containers/Dockerfile` 第 6 与 33 行。无 compose。

## 5. 子模块与远程拉取

结论：已确认。没有子模块；存在管道安装脚本和运行时向外拉取。

- 无 `.gitmodules`。
- 管道与下载脚本：`containers/Dockerfile` 96、188 行；`scripts/install.sh` 154 行；`.github/workflows/workflows.yml` 35–37 行把 actionlint 安装脚本钉在提交 `914e7df21a07ef503a81201c76d2b11c789d3fca` 再执行，二进制本身无独立校验和。
- 运行时：TruffleHog 自更新写在 `containers/Dockerfile` 182–186 行。`nuclei -update-templates` 在构建时执行（104 行）。`strix/interface/update_check.py` 101、108 行访问 GitHub Release 与 PyPI。`strix/interface/cloud/billing.py` 34、38 行在钱包流程 `npx` 拉固定标签包。产品代码不调用 Hugging Face 下载模型。代理若执行镜像里的 interactsh 等工具，会再连外部服务；那是使用时的网络行为，本单未运行。

## 6. CI

结论：未发现问题。

- 四个工作流的 `uses:` 都是 40 位提交 SHA：`.github/workflows/ci.yml`、`binary.yml`、`build-release.yml`、`workflows.yml`，以及 `.github/actions/setup-python-env/action.yml` 12–15 行。
- 触发是 `pull_request`、`push`、`workflow_dispatch`，发布流还有 tag `v*`（`build-release.yml` 3–7 行）。没有 `pull_request_target`，没有 `workflow_run`。
- 全文无 `secrets.`。默认 `permissions: contents: read`。只有 tag 上的 release 作业把 `contents` 升为 `write`（`build-release.yml` 115–116 行）。checkout 带 `persist-credentials: false`。

## 7. 公开漏洞库

查询：2026-11-26 11:31 香港时间，对 `uv.lock` 全部 119 个第三方版本调用 OSV。0 条 `MAL-`。2026-11-26 11:32 香港时间补查 `go.mod` 23 个模块，以及 viewer `package.json` 的直接依赖加 `esbuild`、`fsevents`。

### 依赖是否恶意

结论：未发现问题（已查询的锁定版本）。无锁安装路径仍可疑。

- 119 个锁定 PyPI 版本都没有 `MAL-`。`litellm` 1.101.0 不在 `MAL-2026-2144`。该编号覆盖的 1.82.7、1.82.8、1.107.0 见 `audit/supply-chain.md`。
- `pyproject.toml` 38 行 `litellm>=1.101.0` 无上界。无锁安装会接受 1.107.0。这是约束缺口，不是锁内已装入恶意包。本环境没有安装它。
- Go 23 个模块、已查的 npm 直接依赖：0 条 `MAL-`。其余 npm 传递包、`go.sum` 里未出现在 `go.mod` 的模块、镜像构建期 `@latest` 包，恶意与否未核实。

### 已知漏洞

结论：已确认。下列锁定版本被 OSV 判为受影响。未复现，未判断本进程是否走到有漏洞的函数。PYSEC 与 GHSA 同时出现时只列 GHSA，不把 PYSEC 另算一条。

已在 `audit/supply-chain.md` 列出、本轮仍命中的：litellm 1.101.0、cryptography 48.0.1、mcp 1.28.1、urllib3 2.7.0、pillow 12.3.0（43 条 GHSA）、pypdf 6.14.2（32 条 GHSA）。npm `vite` 6.4.4 仍是 GHSA-8g5g-mxqh-grgj。

本轮新命中：

| 锁定版本 | 编号 | 影响与修复版本（OSV） |
|---|---|---|
| anyio 4.14.1 | GHSA-3w57-8xmc-8v26、GHSA-5p39-cfhj-2xmp、GHSA-82r6-8w77-94w6 | 抽查第一条：`run_process` 忽略 `extra_groups`，fixed 4.14.2 |
| certifi 2026.6.17 | GHSA-6q48-6c7w-9gw2 | 捆绑的 emSign 根，fixed 2026.10.18 |
| filelock 3.29.4 | GHSA-567q-5qvw-pf92 等共 9 条 GHSA | 抽查第一条 fixed 4.2.1；其余 8 条未逐条打开 |
| multidict 6.7.1 | GHSA-54p9-h82j-f925 | 引用泄漏，fixed 6.9.1 |
| pyasn1 0.6.4 | GHSA-2qm8-x737-gmw5、GHSA-9m75-vc3g-4x39、GHSA-fm9c-5rcf-2q5p、GHSA-mm57-4v3p-rchx | 抽查第一条为 BIT STRING 解码耗时，fixed 0.6.5 |
| pyjwt 2.13.0 | 15 条 GHSA，抽查 GHSA-2gx3-rcp4-g85q | JWKS 拉取放大，fixed 2.14.0。其余 14 条未逐条打开 |
| setuptools 83.0.0 | GHSA-xmj3-9gf7-h52w | Windows 安装路径逃逸，fixed 84.1.0 |
| virtualenv 21.5.1 | 19 条 GHSA，抽查 GHSA-394h-qwmc-48pf | Xonsh 激活脚本，fixed 21.14.9。其余 18 条未逐条打开 |
| golang.org/x/sys v0.36.0（`go.mod` 14 行） | GO-2026-5024 | Windows `NewNTUnicodeString` 溢出，fixed 0.44.0 |
| golang.org/x/text v0.3.8（`go.mod` 31 行） | GO-2026-5970、GO-2026-6629 | 非法输入死循环，fixed 0.39.0；precis 解析 panic，fixed 0.41.0 |

filelock 其余编号：GHSA-69hj-f68j-2879、GHSA-8mx5-7m5m-r77g、GHSA-9vhv-jxww-xfm5、GHSA-cwqr-hh33-gh44、GHSA-fq56-3cx3-fjxp、GHSA-j8f7-rjxc-mr56、GHSA-rfpc-qcw5-3pc4、GHSA-v4x8-3wr5-3pc4。pillow／pypdf／pyjwt／virtualenv 的完整 GHSA 清单在本次 OSV 响应中，正文不重复粘贴；未打开的条目其可达性未核实。

## 8. 维护状态

结论：未发现问题。抽查的上游仓库未归档，PyPI 最新版不晚于 2026-11，且没有标 yanked。锁定版本落后于最新版，这不是弃用。

2026-11-26 11:32 香港时间：`BerriAI/litellm`、`openai/openai-agents-python`、`pyca/cryptography`、`modelcontextprotocol/python-sdk`、`pydantic/pydantic`、`psf/requests`、`docker/docker-py`、`python-pillow/Pillow`、`py-pdf/pypdf`、`astral-sh/uv` 的 `archived` 均为 false。最近 push 在 2026-11-02 至 2026-11-26 UTC。

PyPI 最新版上传时间：litellm 1.111.0 为 2026-11-25；openai-agents 0.26.0 为 2026-11-24；cryptography 50.0.2 为 2026-09-30；mcp 2.3.0 为 2026-10-02；pillow 13.0.0 为 2026-10-16；pypdf 7.1.1 为 2026-11-18；urllib3 2.8.0 为 2026-09-15；requests 2.34.2 为 2026-05-14，与锁相同。`griffelib`、`caido-sdk-client`、`huggingface-hub` 的 PyPI 页仍有更新版本，未见 yanked。镜像内 `@latest` 工具的维护状态未核实。
