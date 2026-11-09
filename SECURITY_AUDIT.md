# Strix 只读安全审计

- 审计对象：`Silver-cybersec/strix`（fork 自 `usestrix/strix`，Apache-2.0）
- 审计快照：`215482eb2204cd30ac6b87520e3f5c6511e2e2e8`（`main`，2026-11-09）
- 审计方式：静态阅读源码、清单和 lock 文件；用只读 HTTP 查询 GitHub commit API、PyPI JSON 与 OSV `https://api.osv.dev/v1/querybatch`、`/v1/vulns/{id}`。未运行仓库代码、安装脚本、二进制或镜像，未执行 `pip` / `npm` / `uv` / `poetry install`，未 `docker build` / `docker pull`，未对任何扫描目标发请求。
- 与上游关系：GitHub API 显示该仓库 `fork: true`，`parent` / `source` 均为 `usestrix/strix`，创建于 2026-11-09。同一时刻 `usestrix/strix` 的 `main` SHA 与本快照相同。本 fork 相对上游 `main` 没有额外提交。下面的结论针对这份与上游一致的源码，不是针对一份被单独改过的 fork 差异。

## 结论摘要

| 维度 | 评级 | 依据 |
| --- | --- | --- |
| 隐藏后门 / 混淆载荷 / 凭据外发 | 未发现 | 一方源码中的动态执行、编码和出站地址都能对应到遥测、登录、云 API、自更新或渗透测试功能 |
| 产品固有危险能力 | 高 | 模型输出会在带 `NET_ADMIN`/`NET_RAW`、免密 sudo 的 Kali 容器里以 shell 执行；本地代码目录默认可写挂载；容器可通过 `host.docker.internal` 访问宿主机 |
| 依赖供应链（按 `uv.lock` / `package-lock.json` / `go.sum`） | 中 | 三份 lock 有校验和，锁定版本未命中 OSV 恶意包记录；多份运行时依赖有未修复的 GHSA |
| 依赖供应链（`pip install` 不使用 lock，或按当前 Dockerfile 构建镜像） | 高 | `pyproject.toml` 对 `litellm` 等没有上界；镜像大量使用 `latest`、未固定的 `git clone` 和 `curl \| sh` |
| 隔离实验室可否使用 | 可以，必须先加固 | 见第 4 节。把它当成低权限、默认可信的扫描器使用不合适 |

总体风险：**高**。高风险来自这个工具的设计（自主执行攻击命令）和默认部署面，不是来自本 fork 中新插入的后门。

---

## 1. 恶意代码与后门

### 1.1 未发现的模式

在 `strix/`、`scripts/`、`containers/` 的 Python / shell / Dockerfile / workflow 中：

- 未发现 `eval(` / `exec(` 执行解码后的字符串，也未发现 `pickle.loads`、`marshal.loads`、`os.system`、`shell=True`。`subprocess` 调用使用参数数组。`base64` 用于 OAuth JWT 解析、viewer logo 和 PDF 报告，不是隐藏载荷。
- 仓库内没有 `.so` / `.exe` / `.dll` / `.wasm` 二进制，没有 `.gitmodules`，没有 `docker-compose`，没有 Python `setup.py` 安装钩子。
- 预构建 viewer（`strix/interface/viewer/static/`）是本地静态资源，`index.html` 只引用同目录的 JS/CSS。从该 bundle 抽出的主机名是 `www.w3.org`、`github.com`、`app.strix.ai`、`strix.ai`、`react.dev`、`reactflow.dev`、`gitlab.com`、`bitbucket.org`、`tailwindcss.com`，没有陌生回连域名。该 bundle 是压缩后的第三方前端代码，本次没有逐函数反编译，压缩包内部的完整行为标为**未确认**。
- `strix/skills/**/*.md` 是给模型加载的渗透测试说明，包含针对目标的示例命令。它们不会在安装时执行。这是产品功能，不是安装器后门。

### 1.2 明确的出站地址

一方代码里的固定主机都能对上公开功能。没有看到把 SSH 私钥、浏览器配置或任意环境变量打包后发到陌生地址的逻辑。

| 地址 | 位置 | 作用 |
| --- | --- | --- |
| `https://us.i.posthog.com` | `strix/telemetry/posthog.py:25-26,49` | PostHog 遥测。项目 key `phc_7rO3XRuNT5sgSKAl6HDIrWdSGh1COzxw0vxVIAR6vVZ` 是写在源码里的公开 ingest key |
| `https://strix.gateway.scarf.sh` | `strix/telemetry/scarf.py:29,50` | Scarf 遥测 |
| `https://app.strix.ai` | `strix/config/settings.py:226` | viewer 邮件验证和加密报告转发的默认中继；也是 `strix cloud` 的平台 |
| `https://pypi.org`、`https://api.github.com`、`https://github.com/usestrix/strix` | `strix/interface/update_check.py:37-39,100-110,319` | 版本检查和二进制自更新 |
| `https://auth.openai.com`、`https://chatgpt.com/backend-api/codex` | `strix/config/codex.py:42-50` | Codex 订阅 OAuth |
| `https://api.perplexity.ai`、`https://api.exa.ai` | `strix/tools/web_search/tool.py:50,111-113` | 仅在配置了对应 API key 时做检索 |
| `https://api.getpostman.com` | `strix/utils/api_spec.py:234` | 拉取用户指定的 Postman collection |
| `https://registry.npmjs.org` | `strix/interface/cloud/billing.py:52,541-550` | 仅 `strix cloud billing` 充值时用 npx 拉钱包客户端 |

遥测默认开启：`TelemetrySettings.enabled` 默认 `True`，环境变量 `STRIX_TELEMETRY`（`strix/config/settings.py:175-178`）。`posthog.start` / `scarf.start` 在扫描预检之后调用（`strix/interface/main.py:411-418`，`strix/interface/scan_setup.py:300-311`）。

实际发送的字段在 `strix/telemetry/_common.py:50-56` 和两个 sender 里：操作系统、架构、Python 小版本、Strix 版本、随机 16 hex 的进程内 `SESSION_ID`、模型名、`auth_mode`（`api_key` 或订阅，不含密钥）、扫描模式、白盒/黑盒、是否交互、是否有指令、漏洞数量和严重级别、CWE、token 计数和费用、已加载 skill 名称、异常类名。`strix/telemetry/__init__.py:6-8` 写明不发送异常消息和堆栈。`strix/telemetry/README.md` 与代码一致。首次运行会在 `~/.strix/.seen` 写一个空标记（`_common.py:33-47`）。

`STRIX_TELEMETRY=0` 会同时关掉 PostHog 和 Scarf（两个文件的 `_is_enabled`）。这是默认开启的产品遥测，不是隐藏后门。实验室若要求零出站，必须显式关闭。

### 1.3 凭据存在哪里，有没有外发

本地密钥文件用 `0600` 写入（`strix/utils/secret_files.py:12-31`）：

- `~/.strix/cli-config.json`：每次扫描预检成功后 `persist_current()` 把当前环境变量写入该文件（`strix/config/loader.py:64-91`，调用点 `strix/interface/main.py:411`，TUI 路径 `strix/interface/tui/runtime.py:190`）。`LLM_API_KEY` / `OPENAI_API_KEY`、`DEDUPE_LLM_API_KEY`、`PERPLEXITY_API_KEY`、`EXA_API_KEY`、`POSTMAN_API_KEY`、`LLM_EXTRA_HEADERS` 都在 settings 别名里（`strix/config/settings.py:38-57,124-129,186-217`），因此会被写入家目录。文件权限是 `0600`，但内容是明文。
- `~/.strix/subscription-auth.json`：Codex OAuth access/refresh token（`strix/config/codex.py:60,72`）。
- `~/.strix/platform-auth.json`：云平台 API token（`strix/interface/platform_cli.py:34,78`）。
- `~/.strix/viewer-auth.json`：viewer 邮箱和中继 token（`strix/interface/viewer/auth.py:30,117-120`）。

这些文件不会被默认挂进容器。`check_mountable_dir` 拒绝把家目录本身以及路径中含 `.ssh`、`.aws`、`.gnupg`、`.kube`、`.docker`、`.pki` 等的目录当作本地代码目标（`strix/interface/utils.py:1342-1440`）。用户若把含 `.env` 的项目目录设为扫描目标，该目录仍会整树挂进容器，`.env` 不在这套拒绝名单里。

云上传路径会按文件名排除 `credentials.json`、`.env`、`.git-credentials` 等（`strix/interface/cloud/source_upload.py:68-94`）。这只作用于 `strix cloud` 的源码归档，不作用于本地 Docker 挂载。

主动外发报告时，`report_send` 把加密后的 PDF、文件名、`run_name` 和 **target 字符串** 发到 `STRIX_APP_URL`（默认 `https://app.strix.ai`）（`strix/interface/viewer/auth.py:220-238`）。密码留在本地（`strix/interface/viewer/report_pdf.py:8-11,673-686`）。这是用户在 viewer 里触发的功能。`feedback_submit` 会发送邮箱和反馈正文（`auth.py:200-207`）。

未发现读取浏览器 cookie、SSH 私钥目录或整份 `os.environ` 再发到第三方的代码。云钱包子进程会过滤环境变量（`strix/interface/cloud/billing.py:53-70` 一带的 `_WALLET_ENV_NAMES`），并且 `npx --ignore-scripts`（`billing.py:541-550`）。

### 1.4 自更新与安装器

- 后台版本检查默认开启，每 24 小时一次，打到 PyPI 或 GitHub Releases（`strix/interface/update_check.py:40-52,97-116,178-188`）。`STRIX_NO_UPDATE_CHECK` 或常见 CI 变量会关掉它。检查本身不替换二进制。
- `strix --update` 或交互提示选 `y` 时才会下载 `usestrix/strix` 的 release 并覆盖当前可执行文件，然后 `os.execve` 重启（`update_check.py:292-294,315-376`）。GitHub asset 若带 `sha256:` digest 会核对；没有 digest 时打印提示并跳过校验（`337-346`）。交互默认选项是 `n`（`254`）。
- `scripts/install.sh:6-7` 固定安装上游 `usestrix/strix` 的 release，并拉取 `ghcr.io/usestrix/strix-sandbox:1.4.0`。它**不会**安装本仓库工作树。下载（`76-87,146-168`）没有校验和。随后执行下载到的 `strix --version`（`136,312`），改 shell 配置把 `~/.strix/bin` 加入 `PATH`（`209-284`），并删除 PATH 上其他 `strix` 可执行文件（`102-129,292-307`）。镜像 tag 没有 digest（`194-204`）。

这是公开的安装/升级器，指向官方仓库，不是指向本 fork 的隐蔽更新。它仍然是一条未校验的远程代码执行路径。

### 1.5 构建期钩子

`pyproject.toml:102-103` 的 hatchling hook `scripts/tui_sidecar_hook.py` 在打 wheel 时用本机 `go build` 编译 `strix/interface/tui`，把产物放进 wheel。命令行固定，工作目录固定在仓库内（`tui_sidecar_hook.py:38-50`）。可编辑安装会跳过（`25-26`）。这不是从网络拉未知脚本的 postinstall。`npm` 的 `package.json` 没有 `preinstall` / `postinstall`。lock 里带安装脚本的是 `esbuild@0.25.12` 和可选的 `fsevents@2.3.3`（`package-lock.json` 中 `hasInstallScript: true`），属于这两个上游包的常规行为，只在重建 viewer 时执行。

---

## 2. 危险能力与默认配置

这些是渗透测试代理的设计行为。在实验室里它们同样能打到错误的目标、宿主机和挂载进去的源码。

### 2.1 模型输出的执行边界

根代理和子代理都挂上 SDK 的 `Shell` 与 `Filesystem` 能力（`strix/agents/factory.py:807-827`）。`exec_command` 在参数里没有 `shell` 时默认补上 `bash`（`factory.py:483-496`）。命令在沙箱会话里执行，不在宿主机 shell 里执行。

静态代码里没有命令白名单、没有危险命令拦截。模型可以让容器里的 shell 运行镜像里的 `nmap`、`sqlmap`、`nuclei`、`curl` 等，也可以再下载工具。边界是容器，不是策略引擎。

另外两条会在宿主机执行的路径：

- 用户配置的 MCP stdio：`command` / `args` 由 `~/.strix/mcp-servers.json` 提供，在宿主机拉起子进程（`strix/tools/mcp/client.py:141-153`，加载逻辑 `strix/tools/mcp/loader.py:30-35,101-113`）。文件不存在时不连接。配置被篡改等于宿主机命令执行。
- `git clone` 在宿主机执行（`strix/interface/utils.py:1555-1587`）。带用户名或密码的 URL 会被当成仓库（`1165-1166`）。凭据可能留在 clone 的 `.git` 里，而仓库类目标的 `.git` **不会**被只读保护（`collect_local_sources` 对 repository 设 `protect_metadata: False`，`1330-1337`）。

### 2.2 沙箱实际隔离

默认后端是 Docker，默认镜像 `ghcr.io/usestrix/strix-sandbox:1.4.0`，没有 digest（`strix/config/settings.py:166-170`）。本地没有该镜像时，客户端按 tag pull（`strix/runtime/docker_client.py:206-208`）。

容器创建时追加的能力：

- `NET_ADMIN` 与 `NET_RAW`（`docker_client.py:250-257`）。镜像里的 `nmap` 还有 `cap_net_raw,cap_net_admin,cap_net_bind_service`（`containers/Dockerfile:64`）。
- `extra_hosts["host.docker.internal"] = "host-gateway"`（`docker_client.py:259-260`）。注释写明是为了让代理访问宿主机上的服务。
- 扫描目标里的 `localhost` / `127.0.0.0/8` / `::1` 会被改写成这个网关名（`strix/interface/utils.py:1458-1496`）。对“本机上的服务”做扫描，流量会从容器打向宿主机。
- 暴露端口绑在 `127.0.0.1`（`docker_client.py:244-247`）。Caido 在容器内听 `0.0.0.0:48080` 且 `--allow-guests`（`containers/docker-entrypoint.sh:41-47`），发布到宿主机时仍是 loopback。
- `STRIX_DOCKER_SANDBOX_NETWORK` 若设置，会改容器网络并去掉端口发布（`docker_client.py:54-66`）。设成 `host` 时容器与宿主机共享网络栈。这是操作者配置，默认未设置。
- `STRIX_DOCKER_SANDBOX_DEVICES` 可把宿主机设备以 `rwm` 传进容器（`docker_client.py:69-93`）。默认不传。传入 `/var/run/docker.sock` 会使容器具备控制宿主机 Docker 的能力。
- 内存、CPU、pids 限制默认关闭（`96-117`）。容器 json-file 日志默认 `50m` × `3`（`120-137`）。
- 本地代码目录 bind mount 默认可写：`read_only` 只在 details 里已有该字段时才为真，CLI 解析本地目录时只写入 `target_path`（`utils.py:1183-1188,1320-1327`，`session_manager.py:69-82`，`docker_client.py:270-280`）。`.git`、`.agents`、`.codex` 会再挂一层只读（`session_manager.py:47,230-244`）。其余源码可被代理改写回宿主机。
- 系统目录和家目录根被拒绝挂载（`utils.py:1342-1431`）。这挡的是把 `/`、`/etc`、`$HOME` 整棵树当目标，挡不住项目树里的秘密文件。
- 镜像内 `pentester` 具有 `NOPASSWD:ALL`（`containers/Dockerfile:34-36`）。入口脚本会按宿主机 uid 改 `/etc/passwd` 并用 `setpriv` 降权后继续（`docker-entrypoint.sh:4-17`）。容器内的 root 能力仍在，免密 sudo 可以把它拿回来。这提高了容器逃逸后的影响，本次没有验证具体逃逸路径。

`docker_client.py:19-20,205` 写明这段容器创建逻辑按 `openai-agents==0.14.6` 手工复制。lock 里的 `openai-agents` 是 `0.19.0`（`uv.lock:1459-1460`）。0.19.0 上游方法是否增加了被这份拷贝丢掉的限制，本次未对照安装后的 SDK 源码，标为**未确认**。

镜像内还把 HTTP(S) 代理指到容器内 Caido（`session_manager.py:317-329`，`docker-entrypoint.sh:84-108`），并信任构建时生成的测试 CA（`Dockerfile:67-86`，`docker-entrypoint.sh:120-124`）。容器里的 TLS 拦截是设计行为。该 CA 私钥留在镜像的 `/app/certs/ca.key`。

### 2.3 端口、认证、日志

- Viewer 默认 `127.0.0.1` 和临时端口（`strix/interface/viewer/server.py:562-566`，CLI `strix/interface/viewer/cli.py:42-52`）。`--host 0.0.0.0` 可以改成全接口。授权靠 URL 中的 `secrets.token_urlsafe(32)`（`server.py:141,551-559`）。CLI 会打印该 URL，并说明持有链接的人可以操控正在跑的扫描和浏览历史（`viewer/cli.py:91-97`）。
- TUI sidecar 只接受 `127.0.0.1` / `::1`（`strix/interface/tui/sidecar.py:77,164`）。
- 云支付本地桥只绑 `127.0.0.1`（`strix/interface/cloud/payment_proxy.py:272`）。
- OpenAI tracing 被关掉，API key 的 `use_for_tracing=False`（`strix/config/models.py:707-718`）。
- `run.json` 的 `llm_requests` 只记时间、模型、provider、状态码、token 数（`strix/report/state.py:1024-1045`），且仅当 `STRIX_RUN_RECORD_TIMINGS` 开启（默认 `False`，`settings.py:179`，`models.py:711-712`）。
- 失败的 LLM 调用会把异常文本截到 2000 字符写进 `strix.log`（`strix/llm/request_log.py:67,236-240,920-951`）。代码没有再对这段文本做密钥脱敏。异常对象是否包含密钥取决于上游 SDK，静态阅读不能确定，标为**未确认**。成功调用的日志行不含 prompt 正文。
- 扫描日志文件始终是 DEBUG 级别（`strix/telemetry/logging.py:184-186`）。`httpx` / `openai` / `litellm` 被降噪（`52-59,200`），但不能据此证明依赖库永不打印 `Authorization`。

### 2.4 其他默认出站

即使关掉遥测，下面仍会出站：

- 模型 API（用户配置的供应商）。
- 后台更新检查，除非 `STRIX_NO_UPDATE_CHECK`。
- 镜像里的工具若被模型调用（`nuclei`、注释中写明会自更新的 `trufflehog`，见第 3.5 节）。自更新是否在每次运行时发生，本次未执行二进制，标为**未确认**；Dockerfile 注释把它当成运行时行为（`containers/Dockerfile:174-178`）。

---

## 3. 依赖供应链

查询时间：2026-11-09。OSV 批量接口对锁定版本各查一次。PyPI 恶意包记录 `MAL-2026-2144` 另查了 `litellm` 1.107.0。没有对 Kali `apt` 包、`go install @latest` 或未固定的 git 仓库做版本级 OSV 查询，因为构建时解析到的版本不在仓库里。

### 3.1 会被拉取的东西

| 类别 | 清单 | lock / 校验 | 何时拉取 |
| --- | --- | --- | --- |
| Python 运行时与开发依赖 | `pyproject.toml` | `uv.lock`（119 个 PyPI 包均有 sha256；本地 editable `strix-agent` 无分发文件） | `uv sync` / `uv sync --frozen`。`pip install` **不读** `uv.lock` |
| viewer 前端 | `strix/interface/viewer/frontend/package.json` | `package-lock.json` lockfileVersion 3，284 个包 | 仅重建 viewer（`npm ci`）。已提交的 `static/` 不需要这次安装 |
| TUI | `strix/interface/tui/go.mod` | `go.sum`，29 个模块带 `h1:` | `go build` / wheel hook / `scripts/build.sh` |
| 沙箱镜像 | `containers/Dockerfile` | 见 3.5。默认运行不构建它，而是 pull `ghcr.io/usestrix/strix-sandbox:1.4.0` | `scripts/install.sh`、`scripts/docker.sh`、运行时缺镜像时的 pull |
| 安装器下载的 CLI | `scripts/install.sh` | 无校验和，tag 为 GitHub latest 或 `VERSION` | 执行该脚本时 |
| 运行时自更新 | `strix/interface/update_check.py` | 有 digest 才校验 | `strix --update` 或用户确认 |
| 云钱包 | `mppx@0.8.17`、`@stripe/link-cli@0.13.1` | 版本钉住，无 integrity；`npx --ignore-scripts`，registry 固定 npmjs | 仅 billing topup |
| GitHub Actions | `.github/workflows/*.yml` | action 用 commit SHA | 仅 CI |
| pre-commit | `.pre-commit-config.yaml` | git tag：`ruff-pre-commit` v0.15.20，`pre-commit-hooks` v5.0.0，`bandit` 1.8.3 | 开发者跑 pre-commit 时 |
| git submodule | 无 `.gitmodules` | — | 不拉取 |
| docker-compose 镜像 | 无 compose 文件 | — | 不拉取 |

`uv.lock` 的 registry 全部是 `https://pypi.org/simple`（另加本地 editable）。`package-lock.json` 的 `resolved` 全部在 `registry.npmjs.org`。`go.mod` 没有 `replace`。

### 3.2 Python

`pyproject.toml:35-55` 的直接依赖，以及 `uv.lock` 解析结果。`uv.lock` 里每份 sdist/wheel 都有 sha256。`[tool.uv] exclude-newer = "5 days"`（`pyproject.toml:76-80`）只影响 `uv lock` / `uv add`，不影响 `pip`。

| 声明 | 锁定版本 | 上界 | 安装钩子 |
| --- | --- | --- | --- |
| `openai-agents[litellm]>=0.19.0,<0.20` | 0.19.0 | 有 | 无 |
| `openai>=2.45.0,<3` | 2.53.0 | 有 | 无 |
| `litellm>=1.101.0` | 1.101.0 | **无** | 无 |
| `google-auth>=2.0.0` | 2.55.1 | 无 | 无 |
| `boto3>=1.28.0` | 1.43.36 | 无 | 无 |
| `pydantic>=2.11.3` | 2.13.4 | 无 | 无 |
| `pydantic-settings>=2.13.0` | 2.14.2 | 无 | 无 |
| `rich`（无版本） | 13.9.4 | 无 | 无 |
| `docker>=7.1.0` | 7.1.0 | 无 | 无 |
| `requests>=2.32.0` | 2.34.2 | 无 | 无 |
| `cvss>=3.2` | 3.6 | 无 | 无 |
| `caido-sdk-client>=0.2.0` | 0.2.0 | 无 | 无 |
| `markdown-it-py>=3.0.0` | 4.2.0 | 无 | 无 |
| `reportlab>=4.0` | 5.0.0 | 无 | 无 |
| `pypdf>=5.0` | 6.14.2 | 无 | 无 |
| `cryptography>=48.0.1,<49` | 48.0.1 | 有，且低于 OSV 修复版本 50.0.0 | 无 |
| `pyyaml>=6.0` | 6.0.3 | 无 | 无 |

开发组（`pyproject.toml:61-71`，只在 dev sync 时安装）：`mypy` 2.1.0、`ruff` 0.15.20、`pyright` 1.1.411、`bandit` 1.9.4、`pre-commit` 4.6.0、`pyinstaller` 6.21.0、`pytest` 9.1.1、`pytest-asyncio` 1.4.0、`types-requests` 2.33.0.20260712。

名称核对：

- `caido-sdk-client` 0.2.0：PyPI 作者为 Caido Labs Inc.，owner `caidobot`。当前索引最新版是 0.3.2。不是近似名。
- `griffelib` 2.1.0：`openai-agents` 的依赖。PyPI 项目指向 `mkdocstrings/griffe`，owner `pawamoy`，是 `griffe` 的改名发行版，不是仿冒包。
- `caido-server-auth` 0.1.2：`caido-sdk-client` 的依赖，同来自 PyPI。
- 直接依赖名称与常见正版包一致。传递依赖没有逐个做人工形近字审查；OSV 对锁定版本的查询没有返回 `MAL-*`。

`litellm` 恶意版本：OSV `MAL-2026-2144`（别名 PYSEC-2026-2）记录 PyPI 上的 `litellm` 1.82.7、1.82.8、1.107.0 含凭证窃取和启动时执行的后门，且与上游 git tag 不一致。`pyproject.toml:77-79` 的注释指向同一事件。锁定的 1.101.0 **不在**该恶意版本列表中。2026-11-09 查询 PyPI：`/pypi/litellm/1.107.0/json` 返回 404，项目 JSON 的 releases 里也没有 1.107.0；当时最新版是 1.108.3，1.101.0 未被 yanked。因此按今天的索引，`pip` 不会再装到 1.107.0。无上界仍然意味着以后新的恶意版本只要 `>=1.101.0` 且未被下架，`pip install` 就会选中它。`uv sync --frozen` 不会。

从 `strix-agent` 的运行时依赖闭包（不含 dev 组，90 个包名）里，OSV 命中如下。严重级别取自 OSV `database_specific.severity` 或 CVSS 向量，未再做可利用性验证。

| 包@锁定版本 | 角色 | OSV 摘要 | 编号 |
| --- | --- | --- | --- |
| `litellm@1.101.0` | 直接 | HIGH。公告标题是 LiteLLM Proxy 管理端 salt key 跨域复用导致提权。受影响区间含 1.101.0，同系列修复版本写到 1.101.3。本仓库没有启动 LiteLLM proxy 的代码。客户端调用是否触发：**未确认** | `GHSA-7hp6-4w63-5g45` |
| `cryptography@48.0.1` | 直接 | HIGH×2、MODERATE×1。PKCS#7 解密侧信道等。抽查 `GHSA-g6cj-pr64-35w5` 的修复版本是 50.0.0，而本仓库上界是 `<49` | `GHSA-g6cj-pr64-35w5`、`GHSA-jwv3-5hgf-82ww`、`GHSA-m2h6-j472-rp4c` 及 PYSEC-2026-3552..3554 |
| `pypdf@6.14.2` | 直接 | 46 条，含 HIGH。抽查 `GHSA-23w6-3w8w-8484` 修复于 6.16.1，内容是解析 PDF 时的资源消耗。唯一调用点是对本进程刚生成的 PDF 做加密（`report_pdf.py:678-682`），不是解析外部 PDF | 见附录 A |
| `urllib3@2.7.0` | `requests` 传递 | HIGH×2、MODERATE×1。抽查 `GHSA-8988-9cw3-xx77`：HTTPS 代理 TLS 配置可能被忽略，修复于 2.8.0 | `GHSA-8988-9cw3-xx77`、`GHSA-gh4c-6fx4-qh6g`、`GHSA-vxq7-64xx-v4gw` 及 PYSEC-2026-4175..4177 |
| `mcp@1.28.1` | `openai-agents` 传递 | MODERATE×4。HTTP 传输跟随跨源重定向时携带自定义头；另有会话堆积。修复版本抽查为 1.30.0。用户配置 HTTP MCP 时会走到这条路径 | `GHSA-5h93-6whr-6q8j`、`GHSA-84m7-p3x7-pcfv`、`GHSA-rwrf-2pqf-9j8j`、`GHSA-w4fh-qvv9-3v23` |
| `anyio@4.14.1` | 传递 | CRITICAL×1、HIGH×1、MODERATE×1。进程补充组与进程池 stderr | `GHSA-3w57-8xmc-8v26`、`GHSA-5p39-cfhj-2xmp`、`GHSA-82r6-8w77-94w6` 及 PYSEC-2026-4023..4025 |
| `certifi@2026.6.17` | 传递 | CRITICAL。捆绑的 emSign 根证书对应已被攻破的中间 CA。抽查修复版本 2026.10.18 | `GHSA-6q48-6c7w-9gw2`、`PYSEC-2026-4320` |
| `pyjwt@2.13.0` | 传递 | 29 条，含 1 条 CRITICAL。抽查是 `PyJWKClient` 对未知 `kid` 的 JWKS 拉取放大，修复于 2.14.0。本仓库是否调用 `PyJWKClient`：**未确认** | 见附录 A |
| `pyasn1@0.6.4` | 传递 | HIGH×4。ASN.1 解码的 CPU 消耗 | `GHSA-2qm8-x737-gmw5`、`GHSA-9m75-vc3g-4x39`、`GHSA-fm9c-5rcf-2q5p`、`GHSA-mm57-4v3p-rchx` |
| `pillow@12.3.0` | `reportlab` 传递 | 59 条，含 CRITICAL×5、HIGH×23。图像解码缺陷。抽查 `GHSA-22hg-r758-rrw9` 修复于 13.0.0。报告生成是否把不可信图片送进 Pillow：**未确认** | 见附录 A |
| `multidict@6.7.1` | 传递 | MODERATE。引用泄漏 | `GHSA-54p9-h82j-f925`、`PYSEC-2026-4244` |
| `filelock@3.29.4` | 运行时闭包内 | LOW×2、MODERATE×6、HIGH×1。锁文件竞争，多条与 Windows 有关 | 9 条 GHSA，见附录 A |

开发闭包、不在上面运行时集合中的命中：

- `setuptools@83.0.0`：`GHSA-xmj3-9gf7-h52w`，LOW，Windows 路径问题。
- `virtualenv@21.5.1`：23 条，多条 HIGH，见附录 A。来自 dev 工具链。

其余锁定的 PyPI 包在这次 OSV 查询中没有漏洞记录。查询覆盖了全部 120 个 lock 条目（含本地包，本地包无版本漏洞）。

### 3.3 npm（仅重建 viewer）

`package.json` 没有生命周期脚本。直接依赖在 lock 中都有 `integrity`：

`@dagrejs/dagre@2.0.4`、`@xyflow/react@12.11.2`、`clsx@2.1.1`、`diff@8.0.4`、`highlight.js@11.11.1`、`lucide-react@0.563.0`、`react@19.2.7`、`react-dom@19.2.7`、`react-icons@5.7.0`、`react-markdown@10.1.0`、`remark-gfm@4.0.1`、`tailwind-merge@3.6.0`、`@tailwindcss/vite@4.3.3`、`@types/diff@7.0.2`、`@types/react@19.2.17`、`@types/react-dom@19.2.3`、`@vitejs/plugin-react@5.2.0`、`tailwindcss@4.3.3`、`typescript@5.9.3`、`vite@6.4.4`。

284 个 lock 包里，6 个嵌套在可选包 `@tailwindcss/oxide-wasm32-wasi` 下的条目没有 `integrity` 字段：`@emnapi/core@1.11.1`、`@emnapi/runtime@1.11.1`、`@emnapi/wasi-threads@1.2.2`、`@napi-rs/wasm-runtime@1.1.4`、`@tybys/wasm-util@0.10.2`、`tslib@2.8.1`。它们只在该可选 wasm 目标被安装时出现。

OSV 命中（开发/构建期，不进入已提交的静态 bundle 的运行逻辑；已提交 bundle 是否包含有漏洞的库代码：**未确认**，因为没有反编译比对）：

| 包 | 级别 | 编号 |
| --- | --- | --- |
| `vite@6.4.4` | MODERATE。`server.fs.deny` 可被 query 绕过，修复版本抽查为 6.4.5 | `GHSA-8g5g-mxqh-grgj` |
| `browserslist@4.28.6` | HIGH×2 | `GHSA-73wf-gq98-2v4g`、`GHSA-c83g-rgw3-j3cx` |
| `postcss@8.5.19` | MODERATE | `GHSA-fxqj-rqcc-2cmp` |
| `nanoid@3.3.16` | HIGH。自定义 alphabet 在 size 为 0 时可能死循环 | `GHSA-2v37-7h3g-55p8` |
| `source-map-js@1.2.1` | HIGH | `GHSA-68fv-2mgg-jv7q` |
| `baseline-browser-mapping@2.10.43` | MODERATE | `GHSA-w5vr-8v7q-w6rv` |

安装脚本：`esbuild`、`fsevents`。没有其他 `hasInstallScript`。

### 3.4 Go

`strix/interface/tui/go.mod` 要求 Go 1.24.0。直接依赖：

- `github.com/alecthomas/chroma/v2` v2.14.0
- `github.com/atotto/clipboard` v0.1.4
- `github.com/charmbracelet/bubbles` v0.21.0
- `github.com/charmbracelet/bubbletea` v1.3.10
- `github.com/charmbracelet/lipgloss` v1.1.0
- `github.com/charmbracelet/x/ansi` v0.10.1
- `github.com/charmbracelet/x/term` v0.2.1
- `github.com/muesli/termenv` v0.16.0
- `golang.org/x/sys` v0.36.0

`go.sum` 为模块和 `/go.mod` 行提供 `h1:`。OSV 对 29 个模块版本的命中：

| 模块 | 说明 |
| --- | --- |
| `golang.org/x/sys@v0.36.0`（`go.sum:58`） | `GO-2026-5024`：`windows.NewNTUnicodeString` 整数溢出。公告路径在 `golang.org/x/sys/windows`。Linux 实验室运行 TUI 是否调用该函数：静态上看这条 API 是 Windows 专用，Linux 构建不应链进该路径。未做编译验证，调用情况标为**未确认** |
| `golang.org/x/text@v0.3.8`（`go.sum:60`，传递依赖） | `GO-2026-5970` 非法输入死循环，修复版本抽查为 0.39.0；`GO-2026-6629` `secure/precis` 解析 panic。版本远旧于修复版。是否被 chroma 在渲染不可信终端文本时调用到有漏洞的函数：**未确认** |

其余 27 个模块版本无 OSV 记录。模块路径均为 `github.com/charmbracelet/*`、`github.com/alecthomas/*`、`github.com/muesli/*`、`github.com/rivo/uniseg`、`golang.org/x/*` 等公开模块，没有 `replace` 到个人 fork。

### 3.5 Dockerfile 与镜像内下载

默认运行使用的是 GHCR tag，不是这个 Dockerfile 的构建产物。tag `1.4.0` 与仓库版本 `1.7.0`（`pyproject.toml:3`）不一致。已发布镜像是否等于这份 Dockerfile：**未确认**（未 pull、未比对 digest）。

`containers/Dockerfile` 构建时会拉取：

| 来源 | 行 | 版本锁定 | 校验 | 风险 |
| --- | --- | --- | --- | --- |
| `kalilinux/kali-rolling:latest` | 7, 25 | 可变 `latest`，无 digest | 无 | 高。滚动镜像，构建结果不可复现，且带完整渗透工具和 `apt-get upgrade` |
| `go install` projectdiscovery `httpx`、`katana`、`vulnx`，`jaeles-project/gospider`，`interactsh-client`，`golang.org/x/vuln/cmd/govulncheck` | 15-20 | 全部 `@latest` | 无 | 高 |
| apt：`nmap` `sqlmap` `nuclei` `subfinder` `naabu` `ffuf` `chromium` 等，以及后面的 `wapiti` | 42-63, 211 | Kali 滚动源，无 snapshot | 无 | 高 |
| `curl https://astral.sh/uv/install.sh \| sh` | 88 | 脚本 URL 未固定 | 无 | 高 |
| `nuclei -update-templates` | 96 | 构建时联网更新模板 | 无 | 高。模板集可变 |
| `pipx install arjun`、`dirsearch`、`wafw00f`、`bandit`；`pipx inject dirsearch 'setuptools<81'` | 98-101, 266 | 未钉版本 | 无 | 高 |
| `npm install -g` `retire@latest` `eslint@latest` `js-beautify@latest` `@ast-grep/cli@latest` `tree-sitter-cli@latest` `jshint`（无版本）；`agent-browser@0.26.0` 有版本 | 106-111, 268 | 多数 `latest` | 无 npm integrity 约束写在 Dockerfile 里 | 高 |
| `git clone --depth 1` tree-sitter 八个语法仓库 | 137-148 | 默认分支尖端 | 无 commit | 高 |
| `git clone` `github.com/aravind0x7/JS-Snooper`、`github.com/xchopath/jsniper.sh`、`github.com/ticarpi/jwt_tool` | 165-170 | 默认分支尖端 | 无 | 高。前两个是个人仓库；jwt_tool 是个人维护的公开工具。本次未克隆，仓库当前内容**未确认** |
| trufflehog：`curl` `raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh` 再管道给 `sh`，参数版本 `v3.99.2` | 179-181 | 二进制版本钉住；**脚本 ref 是 `main`** | 2026-11-09 读到的脚本默认做 sha256 核对，cosign 需要额外 `-v`，Dockerfile 没加 `-v`。脚本本身未钉 SHA，攻击者若改 `main` 可以换掉校验逻辑 | 高 |
| 注释写明 trufflehog 运行时会原地自更新 | 174-178 | 运行时版本不再是 3.99.2 | 未执行，触发条件**未确认** | 高 |
| gitleaks：跟随 `releases/latest` 重定向取 tag 再下载 tar | 189-193 | 可变 latest | 无 | 高 |
| trivy `0.76.1` | 195-209 | 有 | sha256 按架构写死并 `sha256sum -c` | 低（在这份 Dockerfile 内） |
| Debian brotli `1.2.0-4.1` 的 deb | 218-227 | 版本钉住 | **无** deb checksum | 中。来自 `deb.debian.org` |
| opengrep `1.31.0` | 229-242 | 有 | sha256 并核对 | 低 |
| opengrep-rules tarball | 247-250 | commit `f1d2b562b414783763fd02a6ed2736eaed622efa` | 无 tarball digest；commit 固定 | 中 |
| caido-cli `0.56.0`，主机 `caido.download` | 286-299 | 版本钉住 | **无** | 中 |
| 镜像内 `pip install` `caido-sdk-client`、`requests`、`httpx`、`beautifulsoup4`、`lxml`、`pyjwt`、`cryptography`，以及 `jwt_tool/requirements.txt` | 307-311 | 全部无版本 | 无 | 高。与宿主 `uv.lock` 无关 |

`containers/docker-entrypoint.sh:122-128` 在构建镜像时用 `eval` 展开一组固定的 `AGENT_BROWSER_*` 环境变量写 profile。输入来自镜像 ENV，不是用户请求。

`scripts/docker.sh` 只是 `docker build` 这份 Dockerfile，tag 默认 `strix-sandbox:dev`。

### 3.6 CI 与其他脚本

- `.github/workflows/ci.yml`、`binary.yml`、`build-release.yml`、`workflows.yml` 里的 `actions/checkout`、`setup-python`、`setup-go`、`setup-node`、`astral-sh/setup-uv`、`upload-artifact`、`download-artifact`、`softprops/action-gh-release`、`zizmorcore/zizmor-action` 都钉在 commit SHA。
- `workflows.yml:26-37` 从固定 commit `914e7df21a07ef503a81201c76d2b11c789d3fca` 下载 `download-actionlint.bash` 并执行，版本参数 `1.7.12`。2026-11-09 读到的该脚本用 `curl` 下载 release 包，文件中没有校验和步骤。这只影响 CI。
- `.github/dependabot.yml` 对 uv、npm、gomod、github-actions 有 7 天 cooldown。
- `scripts/build.sh` 使用 `uv sync --frozen` 和仓库内 `go build`，再跑 PyInstaller。它本身不再 `curl` 第三方二进制。

### 3.7 运行时才会拉取、且不在 lock 里的组件

| 组件 | 位置 | 锁定 | 校验 | 建议 |
| --- | --- | --- | --- | --- |
| GHCR 沙箱镜像 | `settings.py:166-168`，`install.sh:7` | tag `1.4.0` | 无 digest | 审查后记录 digest，用 digest 拉取；或在离线环境加载自建镜像 |
| 上游 CLI release | `install.sh`，`update_check.py` | release tag | install.sh 无；自更新仅在 GitHub 提供 digest 时校验 | 实验室不要用 `curl \| bash`。从本仓库 `uv sync --frozen` 安装 |
| `mppx@0.8.17`、`@stripe/link-cli@0.13.1` | `billing.py:34-38,541-550` | 版本钉住 | 无 hash；`--ignore-scripts`，registry 限定 npmjs | 不用云充值就不拉取。若要用，改为 lockfile + `npm ci` |
| trufflehog 自更新、nuclei 模板 | Dockerfile 注释与 `nuclei -update-templates` | 不锁定运行时内容 | 未确认 | 出站策略拒绝项目发现/发布域名，或换掉会自更新的二进制 |

### 3.8 安全拉取建议

1. 宿主 CLI 只用 `uv sync --frozen`（或同等“按 lock 且校验 hash”的方式）。不要对这份 `pyproject.toml` 做无 lock 的 `pip install`。
2. 给 `litellm`、`pypdf`、`requests`、`docker`、`caido-sdk-client`、`rich`、`pyyaml`、`reportlab`、`google-auth`、`boto3` 补上界，并把 `cryptography` 的上界调整到包含 OSV 修复版之前，先处理 pyproject 里为 macOS universal2 wheel 设的 `<49` 限制（`pyproject.toml:51-53`）。在那之前，接受 `cryptography` 48.0.1 的已知 GHSA，或改用已修复版本并放弃该 wheel 约束。
3. 沙箱不要 pull `:1.4.0`。在可审计的构建里把基础镜像改成 digest，把 `go install` / `npm` / `pip` / `git clone` 改成 commit 或版本加校验和，把 `curl \| sh` 换成带 hash 的发布包。构建机离线或只允许固定镜像仓库。
4. 个人仓库 `JS-Snooper`、`jsniper.sh`、`jwt_tool` 在钉 commit 并阅读之后再 COPY 进镜像。未阅读前不要让构建机克隆它们。
5. viewer 若需重建，用 `npm ci`。不需要 viewer 开发时不要跑 `npm install`。
6. TUI 用 `go build` 即可享受 `go.sum`。把 `golang.org/x/text` 升到至少公告中的 0.39.0 再评估 `GO-2026-5970`。
7. 将审查过的沙箱镜像导出到实验室内网 registry，运行时 `STRIX_IMAGE` 指向 digest，并断开构建期才需要的外网。

---

## 4. 结论与实验室加固

### 4.1 总体判断

这份快照与 `usestrix/strix` 的 `main`（`215482eb2204cd30ac6b87520e3f5c6511e2e2e8`）一致。静态阅读没有发现为本 fork 植入的后门、混淆载荷、隐蔽 C2，或把 SSH/浏览器数据偷偷外发的代码。

它仍然不适合在未隔离的机器上当普通 CLI 使用。总体风险评为 **高**：

1. 模型选定的 shell 会在特权 Kali 容器中执行，默认可写回挂载的源码，并能通过 `host.docker.internal` 访问宿主机服务。
2. 遥测和版本检查默认出站；报告邮件功能会把 target 和加密 PDF 发到 `app.strix.ai`。
3. 按 lock 安装时没有命中已知恶意版本，但 `litellm` 历史恶意版本说明无上界的 `pip install` 不安全；按 Dockerfile 原样构建会执行多条未校验的远程安装。

可以在隔离实验室使用，前提是下面的项都做了，并且只扫描授权目标。

### 4.2 使用前必须做的加固

1. 不要运行 `scripts/install.sh`，不要 `curl -sSL https://strix.ai/install | bash`。在本仓库用 `uv sync --frozen` 从源码安装，使 `uv.lock` 的 hash 生效。
2. 设置 `STRIX_TELEMETRY=0` 和 `STRIX_NO_UPDATE_CHECK=1`。不要执行 `strix --update`。
3. 不要使用默认镜像 tag。要么不连接外网、加载内部已审查镜像，要么 `STRIX_IMAGE` 使用带 digest 的引用。构建镜像前先改掉第 3.5 节的 `latest` 和 `curl \| sh`。
4. 用独立 Docker 网络运行沙箱，使其不能路由到实验室管理网、元数据地址和除授权目标以外的地址。不要依赖默认的 `host.docker.internal`。不要设置 `STRIX_DOCKER_SANDBOX_NETWORK=host`。不要设置 `STRIX_DOCKER_SANDBOX_DEVICES`，尤其不要传入 docker socket。
5. 扫描源码的副本，或接受代理可以改写该目录。不要把含真实密钥的目录设为 `--target`。
6. 知道 `LLM_API_KEY` 会被写入 `~/.strix/cli-config.json`。实验室结束后删除 `~/.strix/*.json`。不要把家目录同步到不受控的备份。
7. 不配置 `PERPLEXITY_API_KEY` / `EXA_API_KEY` / Postman / `strix cloud`，除非允许扫描上下文离开实验室。不要在 viewer 里发送报告或反馈。
8. Viewer 保持默认 `127.0.0.1`。不要把带 token 的 URL 发到实验室以外。
9. 设置 `STRIX_SANDBOX_MEM_LIMIT`、`STRIX_SANDBOX_CPUS`、`STRIX_SANDBOX_PIDS_LIMIT`。
10. 宿主机 Docker 守护进程只给这次实验用的用户，跑完销毁容器和镜像。Strix 进程本身需要 Docker socket 才能建容器，这已经是宿主机高权限。

### 4.3 未确认项

- 已发布的 `ghcr.io/usestrix/strix-sandbox:1.4.0` 与本 Dockerfile 是否一致。
- `openai-agents` 0.19.0 的上游容器创建逻辑与仓库里按 0.14.6 复制的实现差了什么。
- 压缩后的 viewer JS 是否含未在字符串层面出现的动态外联。
- LLM 异常文本、以及 `httpx`/`litellm` 在 DEBUG 下是否会把密钥打进 `strix.log`。
- `GHSA-7hp6-4w63-5g45` 对仅作为客户端使用的 LiteLLM 是否可触发。
- Pillow / PyJWT / `golang.org/x/text` 的有漏洞函数是否在本程序的调用路径上。
- trufflehog 在扫描运行时是否确实自更新，以及更新包的来源。
- `JS-Snooper`、`jsniper.sh`、`jwt_tool` 默认分支在构建当时的内容。
- Kali 滚动源和 `@latest` 工具在未来构建时会解析到什么版本，那些版本在 OSV 中的状态。

---

## 附录 A. OSV 命中编号全文

查询：`POST https://api.osv.dev/v1/querybatch`，2026-11-09。同一编号在 GHSA 与 PYSEC 重复出现时都保留，因为两次都是接口返回值。

### Python 运行时闭包

- `anyio@4.14.1`：`GHSA-3w57-8xmc-8v26`，`GHSA-5p39-cfhj-2xmp`，`GHSA-82r6-8w77-94w6`，`PYSEC-2026-4023`，`PYSEC-2026-4024`，`PYSEC-2026-4025`
- `certifi@2026.6.17`：`GHSA-6q48-6c7w-9gw2`，`PYSEC-2026-4320`
- `cryptography@48.0.1`：`GHSA-g6cj-pr64-35w5`，`GHSA-jwv3-5hgf-82ww`，`GHSA-m2h6-j472-rp4c`，`PYSEC-2026-3552`，`PYSEC-2026-3553`，`PYSEC-2026-3554`
- `filelock@3.29.4`：`GHSA-567q-5qvw-pf92`，`GHSA-69hj-f68j-2879`，`GHSA-8mx5-7m5m-r77g`，`GHSA-9vhv-jxww-xfm5`，`GHSA-cwqr-hh33-gh44`，`GHSA-fq56-3cx3-fjxp`，`GHSA-j8f7-rjxc-mr56`，`GHSA-rfpc-qcw5-3pc4`，`GHSA-v4x8-3wr5-3wq8`
- `litellm@1.101.0`：`GHSA-7hp6-4w63-5g45`
- `mcp@1.28.1`：`GHSA-5h93-6whr-6q8j`，`GHSA-84m7-p3x7-pcfv`，`GHSA-rwrf-2pqf-9j8j`，`GHSA-w4fh-qvv9-3v23`
- `multidict@6.7.1`：`GHSA-54p9-h82j-f925`，`PYSEC-2026-4244`
- `pillow@12.3.0`：`GHSA-22hg-r758-rrw9`，`GHSA-24rc-8hj2-v98c`，`GHSA-254f-3936-288h`，`GHSA-27h8-gf7j-v5hc`，`GHSA-332p-7hrc-92wf`，`GHSA-3j72-p6qg-6xp7`，`GHSA-4558-3h8r-v8xp`，`GHSA-462x-pgg6-55fp`，`GHSA-4rm8-3hmm-39fg`，`GHSA-4rr5-89j8-mwwm`，`GHSA-58r4-6rv3-gfh8`，`GHSA-5x8w-8jj5-q52h`，`GHSA-5xm2-6676-qcp9`，`GHSA-654x-cwwv-5j9c`，`GHSA-65qm-jvmh-6pqg`，`GHSA-74r2-7ff9-mjj2`，`GHSA-78g4-xqwv-q3g8`，`GHSA-7mwm-ghpf-9vrm`，`GHSA-8g5g-8vmr-9782`，`GHSA-9v9v-9qvf-h2hj`，`GHSA-c6hw-5grf-ff96`，`GHSA-cp9v-662q-vvx9`，`GHSA-f2g6-542v-9j58`，`GHSA-fc9g-5qg7-mf45`，`GHSA-g4qv-2jxx-769h`，`GHSA-g7cr-3p24-7wf7`，`GHSA-g842-9rc4-69pw`，`GHSA-gg2r-wvfm-q9rx`，`GHSA-j4m9-6w36-pw47`，`GHSA-jg86-v3jv-hrpm`，`GHSA-mrfp-p49p-3hf2`，`GHSA-p4wg-p73j-c528`，`GHSA-p8hx-v2p6-rp84`，`GHSA-pchr-8j38-3x4p`，`GHSA-q45j-58j2-45rg`，`GHSA-rfgq-q3w2-pmv2`，`GHSA-v9pv-5gc8-w2m6`，`GHSA-vx6w-vv9f-6gvg`，`GHSA-w889-685w-c8fh`，`GHSA-xgh8-jp6q-p5x8`，`GHSA-xvh5-g6gv-559v`，`PYSEC-2026-4326`，`PYSEC-2026-4327`，`PYSEC-2026-4328`，`PYSEC-2026-4329`，`PYSEC-2026-4330`，`PYSEC-2026-4331`，`PYSEC-2026-4332`，`PYSEC-2026-4333`，`PYSEC-2026-4334`，`PYSEC-2026-4335`，`PYSEC-2026-4336`，`PYSEC-2026-4337`，`PYSEC-2026-4338`，`PYSEC-2026-4339`，`PYSEC-2026-4340`，`PYSEC-2026-4341`，`PYSEC-2026-4342`，`PYSEC-2026-4343`
- `pyasn1@0.6.4`：`GHSA-2qm8-x737-gmw5`，`GHSA-9m75-vc3g-4x39`，`GHSA-fm9c-5rcf-2q5p`，`GHSA-mm57-4v3p-rchx`
- `pyjwt@2.13.0`：`GHSA-2gx3-rcp4-g85q`，`GHSA-42vr-xj54-vc7v`，`GHSA-8wjv-2p76-3863`，`GHSA-9j54-fg26-wv3r`，`GHSA-9v7f-9g4p-ffgj`，`GHSA-ffc3-869f-jxw9`，`GHSA-gvp8-978c-rx2q`，`GHSA-hxm8-2xgr-2p9m`，`GHSA-jwrc-g2q2-pq5p`，`GHSA-p4g4-x82p-q773`，`GHSA-pxh4-856f-4h89`，`GHSA-r6x4-923q-g947`，`GHSA-w2cx-738m-mc7w`，`GHSA-w6j9-cwv2-h6wq`，`GHSA-x33g-cr3x-6449`，`PYSEC-2026-4140`，`PYSEC-2026-4141`，`PYSEC-2026-4142`，`PYSEC-2026-4143`，`PYSEC-2026-4144`，`PYSEC-2026-4145`，`PYSEC-2026-4146`，`PYSEC-2026-4147`，`PYSEC-2026-4148`，`PYSEC-2026-4149`，`PYSEC-2026-4150`，`PYSEC-2026-4151`，`PYSEC-2026-4152`，`PYSEC-2026-4183`
- `pypdf@6.14.2`：`GHSA-23w6-3w8w-8484`，`GHSA-39mj-jf6j-2qj3`，`GHSA-42gh-23hr-vphw`，`GHSA-4vqv-7265-gm2w`，`GHSA-5849-834g-hvw5`，`GHSA-5hx5-vpf9-5wcv`，`GHSA-5jq2-8x83-x246`，`GHSA-62m5-6wh8-28x7`，`GHSA-6c7f-9265-g8fw`，`GHSA-6vvc-x485-93v7`，`GHSA-763m-79hh-57f2`，`GHSA-7mh2-4gg2-979f`，`GHSA-9mx8-4j7g-633p`，`GHSA-cgf9-327m-v77c`，`GHSA-cggf-277g-3q54`，`GHSA-f54x-cmq8-qg57`，`GHSA-fc8x-2rww-xw9m`，`GHSA-fp3f-mc75-235c`，`GHSA-fp3h-c4fm-7vvf`，`GHSA-fwg2-594c-jp42`，`GHSA-g9cg-prrw-2r8q`，`GHSA-gmh7-3g9r-mjh5`，`GHSA-jp53-mhqp-8xcg`，`GHSA-jr96-gvgh-j3r4`，`GHSA-jw7q-gvrg-4vj3`，`GHSA-mfr5-77xh-9j7h`，`GHSA-php9-fj8v-98fj`，`GHSA-qfv9-64jg-mh7p`，`GHSA-qv6h-rv94-w285`，`GHSA-v247-6f48-mgcj`，`GHSA-w23x-9jrw-r45c`，`GHSA-x3rh-gjgm-fmqc`，`PYSEC-2026-3655`，`PYSEC-2026-3656`，`PYSEC-2026-3910`，`PYSEC-2026-3911`，`PYSEC-2026-3912`，`PYSEC-2026-3913`，`PYSEC-2026-4153`，`PYSEC-2026-4154`，`PYSEC-2026-4155`，`PYSEC-2026-4156`，`PYSEC-2026-4157`，`PYSEC-2026-4158`，`PYSEC-2026-4159`，`PYSEC-2026-4160`
- `urllib3@2.7.0`：`GHSA-8988-9cw3-xx77`，`GHSA-gh4c-6fx4-qh6g`，`GHSA-vxq7-64xx-v4gw`，`PYSEC-2026-4175`，`PYSEC-2026-4176`，`PYSEC-2026-4177`

### Python 开发依赖

- `setuptools@83.0.0`：`GHSA-xmj3-9gf7-h52w`
- `virtualenv@21.5.1`：`GHSA-394h-qwmc-48pf`，`GHSA-43mp-35q3-x3fq`，`GHSA-494p-63j8-28p3`，`GHSA-558c-9hv2-4prj`，`GHSA-5vjq-rrrf-7h2q`，`GHSA-6w3r-xr62-4xcp`，`GHSA-8rjx-v5ww-45pp`，`GHSA-94p9-xgh2-xp45`，`GHSA-9h9j-4vrj-gf7g`，`GHSA-c947-3pg5-gm8q`，`GHSA-c978-59m8-c77v`，`GHSA-cw65-gx24-rc82`，`GHSA-cx28-75q4-x26q`，`GHSA-fc2g-4w5p-6whm`，`GHSA-hhxq-47xx-j5xx`，`GHSA-p58f-9548-mpm2`，`GHSA-qmx5-4p3x-wvqf`，`GHSA-vxj4-q4g9-crj7`，`GHSA-x78j-v8h9-3j2q`，`PYSEC-2026-4011`，`PYSEC-2026-4012`，`PYSEC-2026-4013`，`PYSEC-2026-4014`

### npm 与 Go

- `baseline-browser-mapping@2.10.43`：`GHSA-w5vr-8v7q-w6rv`
- `browserslist@4.28.6`：`GHSA-73wf-gq98-2v4g`，`GHSA-c83g-rgw3-j3cx`
- `nanoid@3.3.16`：`GHSA-2v37-7h3g-55p8`
- `postcss@8.5.19`：`GHSA-fxqj-rqcc-2cmp`
- `source-map-js@1.2.1`：`GHSA-68fv-2mgg-jv7q`
- `vite@6.4.4`：`GHSA-8g5g-mxqh-grgj`
- `golang.org/x/sys@v0.36.0`：`GO-2026-5024`
- `golang.org/x/text@v0.3.8`：`GO-2026-5970`，`GO-2026-6629`

恶意包记录（**不是**锁定版本）：`MAL-2026-2144` 影响 `litellm` 1.82.7、1.82.8、1.107.0。锁定版本 1.101.0 不在其中。
