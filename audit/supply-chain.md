# 供应链

结论：锁文件里的 litellm **不是** 2026-10-21 那个恶意 wheel；直接依赖没有上界，无锁安装仍会接受它。镜像与安装脚本大量不固定版本、不校验，镜像层未拉取，不能写成安全。

查询时间：2026-11-26 09:38–09:39 香港时间（Asia/Hong_Kong）。OSV：`https://api.osv.dev/v1/query` 与 `/v1/vulns/{id}`。GHCR 只取索引清单，未拉层。响应头 `Date: Thu, 26 Nov 2026 01:39:00 GMT`，即 09:39 香港时间。

基准：https://github.com/usestrix/strix/commit/9ed84f99c8f13fc921baebc59e0c696108cfab3e

## 依赖是否恶意

结论：在已查询的锁定版本上，没有命中恶意包编号；已知恶意的 litellm 1.107.0 不在锁内，但版本约束挡不住它。

- `uv.lock` 将 `litellm` 钉在 **1.101.0**（`uv.lock` 1127–1129 行），来源 `https://pypi.org/simple`，sdist/wheel 带 sha256。上传时间字段为 2026-09-14。
- OSV `MAL-2026-2144`（Malicious code in litellm）列出的版本是 **1.82.7、1.82.8、1.107.0**，不含 1.101.0。同一查询对 1.107.0 返回这条 MAL，对 1.101.0 不返回。
- 官方事件说明（2026-10-21，已下架）：PyPI `litellm==1.107.0` 的 wheel 含启动期凭据收集与远程代码加载。本单未下载该 wheel。来源：https://docs.litellm.ai/blog/security-incident-oct-21-2026 与微软安全博客 2026-10-23。
- `pyproject.toml` 第 38 行是 `litellm>=1.101.0`，没有上界。第 77–80 行 `exclude-newer = "5 days"` 只在 `uv lock` 时生效。`pip install` 按 pyproject 解析时，会接受 1.107.0 以及更高版本。
- 同一次 OSV 查询对下列锁定版本返回 **0** 条，也没有 MAL 编号：`openai-agents` 0.19.0、`openai` 2.53.0、`requests` 2.34.2、`docker` 7.1.0、`pyyaml` 6.0.3、`pydantic` 2.13.4、`caido-sdk-client` 0.2.0、`griffelib` 2.1.0、`aiohttp` 3.14.5、`httpx` 0.28.1。`griffelib` 是 `openai-agents` 0.19.0 在 PyPI 上声明的依赖，不是本仓单独起的包名；未比对上游源码以外的字节。
- 锁文件 120 个 `[[package]]` 里，除本包 `editable = "."` 外，未看到 git、url、directory 源，也没有非 `https://pypi.org/simple` 的 registry。未逐包查询其余传递依赖，那些版本的恶意与否标 **未核实**。
- 未发现本仓自写的、在导入时向外拉代码的安装钩子。Hatch 钩子 `scripts/tui_sidecar_hook.py` 在打 wheel 时调用本机 `go build`，不下载脚本。viewer 的 `package-lock.json` 里 `esbuild` 与可选的 `fsevents` 带 `hasInstallScript`，均为 npm 官方包的常规安装脚本，未打开脚本正文。

## 已知漏洞（不是恶意包）

结论：若干锁定版本有未撤回 GHSA。未复现，未判断每条在本进程是否可达。

| 锁定版本 | OSV 结果（2026-11-26 09:38 香港时间） |
|---|---|
| litellm 1.101.0 | GHSA-7hp6-4w63-5g45，Proxy 管理面盐重用，fixed 1.101.3。本仓作客户端使用，未读到启动 LiteLLM Proxy |
| cryptography 48.0.1 | GHSA-g6cj-pr64-35w5（fixed 50.0.0）、GHSA-jwv3-5hgf-82ww、GHSA-m2h6-j472-rp4c（fixed 49.0.0）。`pyproject.toml` 第 53 行限制 `<49`，修不进 49 |
| mcp 1.28.1 | GHSA-5h93-6whr-6q8j、GHSA-84m7-p3x7-pcfv、GHSA-qx49-fqc8-xw99、GHSA-rwrf-2pqf-9j8j、GHSA-w4fh-qvv9-3v23，fixed 1.30.0 |
| urllib3 2.7.0 | GHSA-8988-9cw3-xx77、GHSA-gh4c-6fx4-qh6g、GHSA-vxq7-64xx-v4gw，fixed 2.8.0 |
| pillow 12.3.0 | 62 条，抽查修复版本为 13.0.0，内容是图像解析缺陷 |
| pypdf 6.14.2 | 46 条，抽查修复版本在 6.16–6.21，内容是 PDF 解析耗时或内存 |

未逐条打开 pillow/pypdf 的其余公告。Go 模块只查了 `github.com/charmbracelet/bubbletea` 1.3.10，0 条。npm 只查了 `vite` 6.4.4，命中 GHSA-8g5g-mxqh-grgj（fixed 6.4.5，开发服务器 `server.fs.deny`）。其余传递依赖 **未核实**。

## 镜像

结论：默认运行镜像只钉了标签，没有 digest。本单没有拉取层。

- 无 `docker-compose` 文件，无子模块（无 `.gitmodules`）。
- 默认镜像：`ghcr.io/usestrix/strix-sandbox:1.4.1`（`strix/config/settings.py` 187–189 行，`scripts/install.sh` 第 7 行）。源码不写 digest。
- 2026-11-26 09:39 香港时间，匿名 pull token 读取索引清单成功。`docker-content-digest: sha256:c8e70302ba4a69b678b4001c909b134d8d0b84625d7b23feed59ece02672c1d2`。清单中有 linux/amd64 与 linux/arm64 两条，另有两条 `unknown/unknown`。只读了索引 JSON，**没有拉取或解包镜像层**，层内容标未核实，不得写成已确认安全。
- `containers/Dockerfile` 第 6 与第 33 行：`kalilinux/kali-rolling:latest`，无 digest。该滚动标签的层未查询。
- CLI 版本是 1.7.1，默认沙箱标签是 1.4.1。本单没有证明二者内容一致。

## 构建期下载

结论：同一 Dockerfile 里，Trivy 与 opengrep 有 sha256；其余多条下载没有。

有校验和：

- Trivy 0.76.1，`containers/Dockerfile` 192–206 行，x86 与 aarch64 两个 sha256。
- opengrep 1.31.0，226–236 行，两个 sha256。
- katana 固定 git 标签 v1.8.0（22–26 行），随后 `go get` / `go mod tidy`，模块校验和未在 Dockerfile 里再写一遍。
- gitleaks 固定 `v8.30.1`（28 行），经 `go install`。

没有校验和，或版本不固定：

- `go install ...@latest`：httpx、vulnx、gospider、interactsh-client、govulncheck（16–20 行）。
- `curl ... https://astral.sh/uv/install.sh | sh`（96 行）。
- `curl ... trufflehog/main/scripts/install.sh | sh`，二进制版本参数为 v3.99.2，安装脚本本身跟踪 `main`（188 行）。
- `npm install -g` 多个 `@latest`，另有 `agent-browser@0.26.0`（114–119 行）。
- brotli `.deb` 来自 `deb.debian.org`，版本参数 1.2.0-4.1，无 sha256（216–221 行）。
- `nuclei -update-templates`（104 行）在构建时拉模板。
- `git clone` JS-Snooper、jsniper.sh、jwt_tool（173–178 行）未钉提交。
- opengrep-rules 钉了提交 `f1d2b562b414783763fd02a6ed2736eaed622efa`（244 行），tarball 无 sha256。

`scripts/install.sh` 154–168 行：`curl` GitHub Release 后直接解压，无 sha256。`strix/interface/update_check.py` 337–346 行：自更新仅在 API 提供 digest 时核对，否则跳过。

计费路径在宿主执行 `npx`：`mppx@0.8.17` 与 `@stripe/link-cli@0.13.1`（`strix/interface/cloud/billing.py` 34、38 行），registry 为 `https://registry.npmjs.org`（52 行）。版本钉在标签上，无 integrity。只在用户走钱包流程时发生。

## 仿冒包名与非官方源

结论：已读的直接依赖名与常见官方包一致；锁文件源是 PyPI。未对全部传递名做仿冒比对，未比对的标未核实。

Python 直接依赖见 `pyproject.toml` 35–54 行。未看到额外 index。Go 模块路径为 `github.com/usestrix/strix/tui`，`go.sum` 存在。viewer `package-lock.json` 的 resolved URL 抽查为 `registry.npmjs.org`，未全表核对。
