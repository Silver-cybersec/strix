# Strix 沙箱镜像只构建比对报告

本报告只记录在本云 VM 内的镜像构建和静态比对。没有执行 `docker run`，没有启动沙箱容器，没有运行 `strix` CLI 或 agent，没有连接或扫描任何目标，没有调用 LLM API。

比对对象：

- 自建镜像：用本仓库 `scripts/docker.sh` 构建并标记为 `strix-sandbox:local`。
- 对照镜像：`ghcr.io/usestrix/strix-sandbox:1.4.0` 的 linux/amd64。只拉取用于离线检查。

工作副本：`Silver-cybersec/strix`，提交 `215482eb2204cd30ac6b87520e3f5c6511e2e2e8`（2026-11-08 02:52:52 +0200）。构建时 `HEAD` 与 `origin/main` 相同。

## 1. 结论摘要

自建成功。`bash scripts/docker.sh local` 退出码 0，耗时 604 秒（2026-11-16T16:39:02Z 至 2026-11-16T16:49:06Z）。产物标签 `strix-sandbox:local`，镜像 ID `sha256:3b44ec455eec3587318d442b07e5a4e67b08d124878f7c2f89651a425198c1e1`，`docker image inspect` 的 Size 为 4138607037 字节。

自建镜像与 GHCR `1.4.0` 的 linux/amd64 不一致。GHCR 镜像 ID `sha256:df5a518208e4e1f46777b01b87a86fc619279308a22327e12ac4e5344d020947`，Size 4005192482 字节，创建时间 2026-11-04T03:50:21.562014554Z。两边的用户、入口、工作目录和最终环境变量相同，多份已锁定或碰巧解析到同一字节的工具也相同；基础镜像 digest、一批 apt 包版本、uv、若干 npm 包、naabu，以及 nuclei 模板目录不一致。

不能认定 GHCR `1.4.0` 由本仓库的 GitHub Actions 构建。当前检出和上游 `usestrix/strix` main 里实际存在的 workflow 都没有 `docker build` / `docker push` 该沙箱镜像。GHCR 索引附带 BuildKit SLSA provenance：`builder.id` 为空，没有 Actions 的 workflow 路径或 run id；记录的 VCS 源是 `https://git-manager.devin.ai/proxy/github.com/usestrix/strix`，revision `4d793375e06650a1ff6fcab0a6bba62b542c8edf`。该 statement 的签名未校验。

## 2. 构建步骤与命令

### 2.1 源与工具链

| 项 | 值 |
|---|---|
| 仓库提交 | `215482eb2204cd30ac6b87520e3f5c6511e2e2e8` |
| 构建命令 | `BUILDKIT_PROGRESS=plain bash scripts/docker.sh local` |
| 脚本实际执行 | `docker build -f containers/Dockerfile -t strix-sandbox:local <仓库根>`（`scripts/docker.sh` 第 7–14 行） |
| 构建上下文 | 日志原文 `Sending build context to Docker daemon  22.52MB` |
| 主机 | Ubuntu 24.04.5，内核 `6.12.113+`，`x86_64`，`nproc` = 4 |
| Docker 客户端/引擎 | 29.1.3，Git commit `29.1.3-0ubuntu3~24.04.2`，API 1.52，Go `go1.24.4`，构建时间 `Wed Apr 29 16:41:06 2026` |
| containerd | 2.2.1 |
| runc | `1.3.4-0ubuntu1~24.04.1` |
| docker-init | 0.19.0 |
| iptables | `1.8.10 (nf_tables)` |
| BuildKit / buildx | 未安装 `docker buildx` 插件。`docker build` 走 legacy builder |
| syft | 1.57.1，Git commit `a0c0d3890a9d79421c403e8fcadec50a2e55bf21`，schema 16.1.13。安装包 `syft_1.57.1_linux_amd64.tar.gz` 的 sha256 `bf52519eb57f675d47c07d43213cb6dff6449f3d6d1147ff6701ed91f44eef0c` 已对照官方 checksums 通过 |
| dive | 未使用 |

VM 启动时没有 Docker。本次安装了 Ubuntu 源里的 `docker.io 29.1.3-0ubuntu3~24.04.2`。init 是 tini，没有 systemd，因此是手工启动 `dockerd --host=unix:///var/run/docker.sock --iptables=true --storage-driver=overlay2`。为降低编译时内存不足的风险，另外挂了一块 16 GiB swap。这些都没有改仓库。

构建日志开头原文：

```
Building strix-sandbox:local ...
DEPRECATED: The legacy builder is deprecated and will be removed in a future release.
            Install the buildx component to build images with BuildKit:
            https://docs.docker.com/go/buildx/
Sending build context to Docker daemon  22.52MB
Step 1/79 : FROM kalilinux/kali-rolling:latest AS gobuilder
Digest: sha256:1ff4a2be3849333687b3ed2b31bb5af0cd1b7c0a11ec7713feb5abe5a6b2795c
Status: Downloaded newer image for kalilinux/kali-rolling:latest
 ---> 333b656011f6
```

日志结尾原文：

```
Successfully built 3b44ec455eec
Successfully tagged strix-sandbox:local
Done: strix-sandbox:local
```

计时文件记录：`START_UTC 2026-11-16T16:39:02Z`，`END_UTC 2026-11-16T16:49:06Z`，`ELAPSED_SEC 604`，`EXIT 0`。共 79 个 Dockerfile 步骤，没有失败重试。

GHCR 拉取：`docker pull ghcr.io/usestrix/strix-sandbox:1.4.0`，2026-11-16T16:42:30Z 至 2026-11-16T16:43:18Z，48 秒，退出码 0。拉取输出的 Digest 为索引 digest `sha256:9ff1be4044c3ff5e9fcecb118b9855f8d1270db39358523e43fb3a13c0027009`。

Dockerfile 与 GHCR provenance 所记 revision 的关系：本工作副本 `containers/Dockerfile` 的 git blob 是 `a1ff49ef0e2246b982f5a5670edde29212947572`。GitHub API 显示 `usestrix/strix` 提交 `4d793375e06650a1ff6fcab0a6bba62b542c8edf`（2026-11-04T03:49:36Z，说明文字为 `fix(sandbox): install a pinned, sha256-verified trivy release`）上的同路径 blob 相同。一并被 COPY 进镜像的 `containers/docker-entrypoint.sh`、`containers/opengrep-rules/github-owned-action-not-pinned-to-commit-sha.yaml`、`strix/tools/proxy/caido_api.py` 的 blob 也与该提交相同。因此这次自建用的 Dockerfile 和被复制的源文件，与 provenance 记录的那次构建是同一份文件内容。工作副本里没有该提交对象，只比对了 blob。

上游 `usestrix/strix` main 在查询时的提交是 `4719329d1c3eb0db796714397551945c514a2301`（2026-11-13T17:24:42Z）。它的 `containers/Dockerfile` blob 是 `14a2c0ced890b940fa7489f2ab535c9485132731`，与本次构建的文件不同，差在两处：gobuilder 增加 `CGO_ENABLED=0` 并把 katana 改为克隆 `v1.8.0` 再编译；gitleaks 改为在 gobuilder 里安装 `v8.30.1`，不再请求 `releases/latest`。本次没有构建上游 main 的 Dockerfile。

### 2.2 GHCR 是否由 Actions 构建

本仓库 `.github/workflows` 只有四个文件，blob 与上游 main 上同名文件一致：

| 文件 | blob | 触发 | 与沙箱镜像的关系 |
|---|---|---|---|
| `.github/workflows/ci.yml` | `51d0fe0e8b3cb299f550f6d5f3c8daacbda31b7a` | 第 3–7 行：`pull_request`、`push` 到 `main`、`workflow_dispatch` | 第 18–121 行是 lint、pytest、TUI、viewer、wheel。没有构建或推送沙箱镜像 |
| `.github/workflows/build-release.yml` | `337255c2c6643d4f866e8eb065f6dea2e7f9ff2a` | 第 3–7 行：`push` tag `v*`、`workflow_dispatch` | 第 12–108 行构建 wheel 和 PyInstaller 二进制并上传 artifact；第 110–128 行在 tag 上用 `softprops/action-gh-release` 发布 GitHub Release 附件。没有 docker build/push |
| `.github/workflows/binary.yml` | `18f5b78f4891d3231fd8ecf0b569b7ccd552c939` | 第 3–24 行：相关路径的 `pull_request` / `push` 到 `main`，以及 `workflow_dispatch` | 第 56–58 行把 `STRIX_IMAGE` 设为 `alpine:3.20`，只作为 CLI 二进制冒烟测试的环境变量，不是沙箱镜像发布 |
| `.github/workflows/workflows.yml` | `32601e93cd512c7d13fe2fe99f3aa54fcca21b6c` | 第 3–12 行：`.github/**` 的 PR、main push、`workflow_dispatch` | 第 21–50 行运行 actionlint 和 zizmor。没有镜像构建 |

`v1.4.0` tag 的树里，workflow 只有 `.github/workflows/build-release.yml`。该文件开头同样是 tag `v*` 与 `workflow_dispatch`，内容是多平台 CLI 构建，没有沙箱镜像发布。该 tag 的 `scripts/install.sh` 把 `STRIX_IMAGE` 写成 `ghcr.io/usestrix/strix-sandbox:1.1.0`，与镜像标签 `1.4.0` 不是同一个字符串。

GitHub Actions API 还把 `checks.yml`、`codeql.yml`、`typecheck.yml` 列为 active，页面 URL 指向 main，但 main 的 git tree（643 条路径，`truncated: false`）不含这三个文件，contents API 返回 404。`codeql.yml` 在提交 `a5b80af4289406b9e54e6e567ee183b0d233a7af` 上能读到，是 CodeQL 的 python/go/javascript 分析，不是沙箱镜像构建；提交 `881bc09a06bd1b350b74097ef0ee52d0c459415c`（2026-10-04）的说明是删除该 workflow。`checks.yml` 与 `typecheck.yml` 按路径查询提交返回空，正文未核实。

GHCR `1.4.0` 的 OCI index digest 是 `sha256:9ff1be4044c3ff5e9fcecb118b9855f8d1270db39358523e43fb3a13c0027009`。索引里有：

- linux/amd64 manifest `sha256:9ee5a4faf67907f3a9880adeacb7b8395b3f1e7cae56019dcd5839d264c79d08`
- linux/arm64 manifest `sha256:268023698313c0e6ff7d47aaa3e915f09a8c21003cccc0da1f7fdfec0b701204`
- 两个 `application/vnd.docker.attestation.manifest.v1+json`，predicate type 都是 `https://slsa.dev/provenance/v1`，分别指向上面两个平台 manifest

amd64 attestation（blob `sha256:6440d2494b9ab7ad318428559b8803d0c29f20eaf1ce036fa7579d965d390a5a`）里的字段：

- `buildType`: `https://github.com/moby/buildkit/blob/master/docs/attestations/slsa-definitions.md`
- `builder.id`: 空字符串
- `internalParameters.dockerfileVersion`: `1.28.0`
- `internalParameters.builderPlatform` / `targetPlatform`: `linux/amd64`
- `vcs:source`: `https://git-manager.devin.ai/proxy/github.com/usestrix/strix`
- `vcs:revision`: `4d793375e06650a1ff6fcab0a6bba62b542c8edf`
- `vcs:localdir:context`: `.`，`vcs:localdir:dockerfile`: `containers`
- `startedOn`: `2026-11-04T03:49:37.264661781Z`
- `finishedOn`: `2026-11-04T04:13:38.479509414Z`
- `invocationId`: `xv17q18h32np8etbpevpjg23m`
- `buildkit_completeness.resolvedDependencies`: `false`
- subject 同时包含 `ghcr.io/usestrix/strix-sandbox@1.4.0` 和 `@latest` 的 linux/amd64，digest 都是上面的 amd64 manifest

arm64 attestation 的 `builder.id` 同样为空，revision 和起止时间相同，invocation id 相同。没有 GitHub Actions 的 `github.workflow`、`run_id` 或 Actions builder id。因此不能把这次发布写成 Actions 构建。谁在 Devin git-manager 代理之后触发了 BuildKit，statement 没有写明，未核实。statement 签名未校验。

查询当日 `latest` 的 index digest 是 `sha256:c8e70302ba4a69b678b4001c909b134d8d0b84625d7b23feed59ece02672c1d2`，已经不是 `1.4.0` 的 index。provenance 只说明 2026-11-04 那次构建把 `1.4.0` 和 `latest` 指到了同一个平台 manifest。没有拉取今天的 `latest`，也没有比较它的内容。

## 3. 供应链清单

版本锁定以 Dockerfile 为准。解析结果来自本次构建日志、镜像内 `dpkg` status、文件 sha256，或 syft 1.57.1。日志里没有出现完整 URL 的来源，不猜测主机名。

### 3.1 基础镜像

Dockerfile 第 6 行和第 25 行都是 `FROM kalilinux/kali-rolling:latest`。标签 `latest`，没有锁 digest。

本次构建解析到的 manifest list：`sha256:1ff4a2be3849333687b3ed2b31bb5af0cd1b7c0a11ec7713feb5abe5a6b2795c`（`docker pull` 日志的 `Digest:`，media type `application/vnd.docker.distribution.manifest.list.v2+json`）。其中 linux/amd64 manifest 是 `sha256:12e581978942a815b35ab2406725c274a4f28825650595d2ec2b7fb7668b039d`。拉到本地后的镜像 ID 是 `sha256:333b656011f6783f6a8c6e508db294af422346d352635692e3f95a8653c4bcae`，Created `2026-11-15T04:06:02.201215556Z`，Size 121720111 字节，1 个 RootFS 层。自建成品镜像继承的标签：`org.opencontainers.image.version=2026.11.15`，`org.opencontainers.image.created=2026-11-15T04:04:49Z`，`org.opencontainers.image.revision=4d1ddb93`，`org.opencontainers.image.source=https://gitlab.com/kalilinux/build-scripts/kali-docker`，`io.buildah.version=1.43.2`。

GHCR provenance 记录的依赖是 `pkg:docker/kalilinux/kali-rolling@latest?platform=linux/amd64`，digest `sha256:7997e0581c4f5c185cfe6773d5cae8fcc7c6b79798726ccaabf843af4bcd2013`，并且 `resolvedDependencies` 完整度为 false。向 Docker Hub HEAD 该 digest，返回 200，media type 是 `application/vnd.docker.distribution.manifest.list.v2+json`。这个 list 的 linux/amd64 子 manifest 是 `sha256:73915d5d54833d498f52fd200960126be4a2dbdcc110d75d70839c4ef155ed03`。没有拉取这个历史子镜像。GHCR 成品镜像上的 Kali 标签是 `org.opencontainers.image.version=2026.11.01`，`org.opencontainers.image.created=2026-11-01T04:05:27Z`，revision 同样是 `4d1ddb93`。

两个成品镜像里的 `/usr/lib/os-release` 文本相同，`VERSION_ID="2026.3"`。OCI 镜像标签里的 `2026.11.15` / `2026.11.01` 是容器镜像构建日期，不是 os-release 的版本号。

两边的 `/etc/apt/sources.list.d/kali.sources` sha256 都是 `d6d5baa1d9959a19425ddd5a1254f1eb039b94c8f288e2ec9aca91c94460cdfa`。内容是：

```
Types: deb
URIs: http://http.kali.org/kali/
Suites: kali-rolling
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/kali-archive-keyring.gpg
```

没有 `/etc/apt/sources.list`。套件 `kali-rolling` 未锁日期或 snapshot。本次构建日志里 apt 实际访问过的主机（只列日志中出现的，不表示配置文件写了这些镜像）：`http.kali.org`、`kali.download`、`mirrors.ocf.berkeley.edu`、`kali.darklab.sh`、`us.mirror.ionos.com`。

### 3.2 apt 包

Dockerfile 第 8–11 行、第 29–32 行、第 42–61 行和第 211 行的 apt 包都没有锁版本，也没有 `apt-get upgrade` 的快照。第 211 行是单独的 `apt-get install -y wapiti`。

gobuilder 阶段日志里的编译器包：`golang-go` `2:1.27~1`，解包 `golang-1.27-go 1.27.3-1` 与 `golang-1.27-src 1.27.3-1`。两个成品镜像的 dpkg 都是这三个版本。

两个成品镜像里与安全工具直接相关、且版本字符串相同的包：

| 包 | 版本 |
|---|---|
| ca-certificates | 20260816+nmu1 |
| chromium | 156.0.8078.47-1 |
| curl | 8.22.0-1 |
| ffuf | 2.3.0-1 |
| git | 1:2.53.0-1 |
| golang-go | 2:1.27~1 |
| jq | 1.8.2-1 |
| libbrotli1 / python3-brotli | 1.2.0-4.1 |
| nmap | 7.991+dfsg-1kali1 |
| nodejs | 24.21.0+dfsg+~cs24.13.4-1 |
| npm | 12.2.0+ds1-1 |
| nuclei | 3.11.1-0kali1 |
| openssl | 3.6.5-1 |
| pipx | 1.15.0-1 |
| python3 | 3.14.7-3 |
| sqlmap | 1.10.10-1 |
| subfinder | 2.17.0-0kali1 |
| sudo | 1.9.17p2-9 |
| wapiti | 3.2.10+dfsg-4 |
| wget | 1.25.0-3 |

`naabu` 未锁版本，两边不同：自建 `2.7.0-0kali1`，GHCR `2.6.1-0kali1`。其余 76 个版本差异和只存在于 GHCR 的 5 个包见第 4 节。已安装包总数：自建 929，GHCR 934。

### 3.3 Go `@latest`

Dockerfile 第 15–20 行全部是 `go install ...@latest`，没有 `go.sum`，没有校验和。本次构建日志里的顶层模块下载行：

| 命令 | 日志中的模块版本 |
|---|---|
| `github.com/projectdiscovery/httpx/cmd/httpx@latest` | `github.com/projectdiscovery/httpx v1.12.0` |
| `github.com/projectdiscovery/katana/cmd/katana@latest` | `github.com/projectdiscovery/katana v1.8.0` |
| `github.com/projectdiscovery/cvemap/cmd/vulnx@latest` | `github.com/projectdiscovery/cvemap v1.0.0` |
| `github.com/jaeles-project/gospider@latest` | `github.com/jaeles-project/gospider v1.1.6` |
| `github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest` | `github.com/projectdiscovery/interactsh v1.4.1` |
| `golang.org/x/vuln/cmd/govulncheck@latest` | `golang.org/x/vuln v1.9.0` |

日志只有 `go: downloading`，没有打印 GOPROXY 主机，下载主机未核实。这 6 个成品二进制在两个镜像里的 sha256 相同，见第 4 节。syft 在两边都看到上述模块版本，同时也看到更旧的同名模块版本；那些旧版本来自其他二进制里的依赖，没有逐个归到某一个文件。

### 3.4 curl / wget，以及有无校验和

| 用途 | Dockerfile | 本次日志里的具体 URL | 校验和 |
|---|---|---|---|
| uv | 第 88 行 `curl -LsSf https://astral.sh/uv/install.sh \| sh` | `https://astral.sh/uv/install.sh` | 无。安装脚本和二进制都未锁版本 |
| trufflehog | 第 179–180 行，`ARG TRUFFLEHOG_VERSION=3.99.2`，安装脚本取 `main` | `https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh` | 版本参数锁了 `v3.99.2`，脚本本身和下载的二进制没有校验和 |
| gitleaks | 第 182–193 行，跟随 `releases/latest` | 日志先访问 `https://github.com/gitleaks/gitleaks/releases/latest`，随后下载 `https://github.com/gitleaks/gitleaks/releases/download/v8.30.1/gitleaks_8.30.1_linux_x64.tar.gz` | 无。`@latest` 在这次构建被重定向到 v8.30.1 |
| trivy | 第 195–209 行，`TRIVY_VERSION=0.76.1`，amd64 sha256 `c7cbe3137fb8a48a17574bede3dd6beb16742d1e9772929940fc90082c39d9c0` | `https://github.com/aquasecurity/trivy/releases/download/v0.76.1/trivy_0.76.1_Linux-64bit.tar.gz` | 有 sha256，Dockerfile 在构建时执行 `sha256sum -c` |
| Debian brotli | 第 218–227 行，`BROTLI_DEB_VERSION=1.2.0-4.1` | `https://deb.debian.org/debian/pool/main/b/brotli/libbrotli1_1.2.0-4.1_amd64.deb` 与 `python3-brotli_1.2.0-4.1_amd64.deb` | 锁了 deb 版本字符串，没有文件校验和 |
| opengrep | 第 229–242 行，`OPENGREP_VERSION=1.31.0`，amd64 sha256 `dbbe3c2058dbe600c449a8ba22ea79a1643a6660658400090d0323a8d0d31147` | `https://github.com/opengrep/opengrep/releases/download/v1.31.0/opengrep_manylinux_x86` | 有 sha256 |
| opengrep 规则 | 第 247–258 行，`OPENGREP_RULES_REF=f1d2b562b414783763fd02a6ed2736eaed622efa` | `https://codeload.github.com/opengrep/opengrep-rules/tar.gz/f1d2b562b414783763fd02a6ed2736eaed622efa` | 锁了 git commit，没有 tarball 校验和 |
| caido-cli | 第 286–299 行，`CAIDO_VERSION=0.56.0`，用 wget | 日志先写 `https://caido.download/releases/v0.56.0/caido-cli-v0.56.0-linux-x86_64.tar.gz`，随后出现 `https://011d6d2c3b16566b4a68d3410328c12a.r2.cloudflarestorage.com/caido-releases/v0.56.0/caido-cli-v0.56.0-linux-x86_64.tar.gz` | 锁了版本，没有校验和 |

syft 把 uv 记为 rust crate `uv-version`：自建 `0.13.8`，GHCR `0.13.7`。与两个 `uv` 二进制 sha256 不同一致。

### 3.5 npm

Dockerfile 第 106–111 行除 `agent-browser@0.26.0` 外都是 `@latest`。第 268 行 `npm install -g jshint` 没有版本。日志没有打印 npm registry 主机。

成品镜像里对应 `package.json` 的 version 字段：

| 包 | Dockerfile | 自建 | GHCR |
|---|---|---|---|
| retire | `@latest` | 6.1.1 | 6.1.0 |
| eslint | `@latest` | 10.15.0（另有传递依赖 6.4.0，两边都有） | 10.14.0（另有 6.4.0） |
| js-beautify | `@latest` | 2.0.3 | 2.0.3 |
| @ast-grep/cli | `@latest` | 0.51.3 | 0.51.1 |
| tree-sitter-cli | `@latest` | 0.27.2 | 0.27.1 |
| agent-browser | `0.26.0` | 0.26.0 | 0.26.0 |
| jshint | 未写版本 | 2.13.6 | 2.13.6 |

### 3.6 pip / pipx

Dockerfile 第 98–101 行 `pipx install arjun`、`dirsearch`、`wafw00f` 未锁版本；`pipx inject dirsearch 'setuptools<81'` 只给了上界。第 266 行 `pipx install bandit` 未锁版本。第 308–311 行 venv 里的 `caido-sdk-client`、`requests`、`httpx`、`beautifulsoup4`、`lxml`、`pyjwt`、`cryptography`，以及 `jwt_tool/requirements.txt`，都没有在 Dockerfile 里写版本。日志没有打印 PyPI 主机。

syft 记录的相关发行版版本（两边相同，除非另注）：arjun 2.2.7，dirsearch 0.5.0，wafw00f 2.4.2，bandit 1.9.4，caido-sdk-client 0.3.2，httpx 0.28.1，pyjwt 2.15.1，lxml 6.1.3。beautifulsoup4 两边都同时有 4.14.3 和 4.15.0，cryptography 两边都同时有 47.0.0、49.0.0、50.0.2，requests 两边都同时有 2.33.1 和 2.34.2。这些重复版本没有逐个归到 venv 或系统 site-packages。setuptools 不完全相同：自建有 80.10.2、84.2.0、84.2.1；GHCR 有 80.10.2、84.0.0、84.2.1。系统包 `python3-setuptools` 也不同（见第 4 节）。

### 3.7 git clone，未锁提交，但这次两边提交相同

Dockerfile 第 140–148 行和第 165–170 行都是 `git clone` 默认分支，没有 `--branch` 或 commit。两个镜像里的 ref 文件内容相同：

| 仓库 | ref | 提交 |
|---|---|---|
| `https://github.com/aravind0x7/JS-Snooper.git` | `refs/heads/main` | `932704cbce1a04ebe89a558f395a6e474e01b15b` |
| `https://github.com/xchopath/jsniper.sh.git` | `refs/heads/master` | `65d0b508dc88e6de3dfe6ac60e6d89eca22be686` |
| `https://github.com/ticarpi/jwt_tool.git` | `refs/heads/master` | `3bc7407cf2222d6a821dcc19c776e5a1b1cb9a9b` |
| `https://github.com/tree-sitter/tree-sitter-java.git` | `refs/heads/master` | `e10607b45ff745f5f876bfa3e94fbcc6b44bdc11` |
| `https://github.com/tree-sitter/tree-sitter-javascript.git` | `refs/heads/master` | `58404d8cf191d69f2674a8fd507bd5776f46cb11` |
| `https://github.com/tree-sitter/tree-sitter-python.git` | `refs/heads/master` | `26855eabccb19c6abf499fbc5b8dc7cc9ab8bc64` |
| `https://github.com/tree-sitter/tree-sitter-go.git` | `refs/heads/master` | `2346a3ab1bb3857b48b29d779a1ef9799a248cd7` |
| `https://github.com/tree-sitter/tree-sitter-bash.git` | `refs/heads/master` | `a06c2e4415e9bc0346c6b86d401879ffb44058f7` |
| `https://github.com/tree-sitter/tree-sitter-json.git` | `refs/heads/master` | `254c42a6476413b776221e03982ac8ae159eeb72` |
| `https://github.com/tree-sitter-grammars/tree-sitter-yaml.git` | `refs/heads/master` | `a1c4812a73ec5e089de8e441fdea3a921e8d5079` |
| `https://github.com/tree-sitter/tree-sitter-typescript.git` | `refs/heads/master` | `75b3874edb2dc714fb1fd77a32013d0f8699989f` |

提交相同只说明 2026-11-04 与 2026-11-16 这两次构建碰巧取到同一默认分支尖端，Dockerfile 仍然没有锁。

### 3.8 nuclei 模板

Dockerfile 第 96 行是 `nuclei -update-templates`，没有模板版本或校验和。这是两边内容差最大的一项，数字见第 4 节。

## 4. 镜像比对

### 4.1 身份、入口、环境变量

| 项 | 自建 `strix-sandbox:local` | GHCR `1.4.0` linux/amd64 |
|---|---|---|
| 镜像 ID | `sha256:3b44ec455eec3587318d442b07e5a4e67b08d124878f7c2f89651a425198c1e1` | `sha256:df5a518208e4e1f46777b01b87a86fc619279308a22327e12ac4e5344d020947` |
| RepoDigest | 无 | `ghcr.io/usestrix/strix-sandbox@sha256:9ff1be4044c3ff5e9fcecb118b9855f8d1270db39358523e43fb3a13c0027009`（多架构 index） |
| 平台 manifest | 本地构建，无 registry manifest | amd64 `sha256:9ee5a4faf67907f3a9880adeacb7b8395b3f1e7cae56019dcd5839d264c79d08` |
| Created | `2026-11-16T16:49:06.96058432Z` | `2026-11-04T03:50:21.562014554Z` |
| Size | 4138607037 | 4005192482 |
| 架构 | linux/amd64 | linux/amd64 |
| User | `pentester` | `pentester` |
| Entrypoint | `["docker-entrypoint.sh"]` | `["docker-entrypoint.sh"]` |
| Cmd | 空 | 空 |
| WorkingDir | `/workspace` | `/workspace` |
| RootFS 层数 | 37 | 41 |
| `docker history` 条数 | 84 | 84 |

最终 `Config.Env` 两边逐条相同：

```
PATH=/home/pentester/go/bin:/home/pentester/.local/bin:/home/pentester/.npm-global/bin:/app/.venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
LANG=C.UTF-8
NPM_CONFIG_PREFIX=/home/pentester/.npm-global
AGENT_BROWSER_EXECUTABLE_PATH=/usr/bin/chromium
AGENT_BROWSER_USER_AGENT=Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36
AGENT_BROWSER_ARGS=--disable-blink-features=AutomationControlled,--no-first-run,--no-default-browser-check,--lang=en-US
AGENT_BROWSER_SCREENSHOT_DIR=/workspace/.agent-browser-screenshots
AGENT_BROWSER_IDLE_TIMEOUT_MS=180000
VIRTUAL_ENV=/app/.venv
REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
PYTHONPATH=/opt/strix-python
```

标签里只有 Kali 基础镜像的日期和版本不同：自建 `2026.11.15` / `2026-11-15T04:04:49Z`，GHCR `2026.11.01` / `2026-11-01T04:05:27Z`。`description`、`io.buildah.version=1.43.2`、revision `4d1ddb93`、source、vendor、title、url、authors 相同。`io.buildah.version` 来自基础镜像标签，不是这次沙箱构建器的版本。GHCR 的 history 命令带 `# buildkit`，provenance 的 dockerfile frontend 是 `1.28.0`；自建用的是 Docker 29.1.3 的 legacy builder。

### 4.2 各层 history

下表来自 `docker history --no-trunc`，顺序从基础镜像到最终入口，共 84 条。大小是 Docker 打印的十进制近似值，不是 inspect Size 的字节数。带 `*` 的自建大小表示与 GHCR 该行的大小字符串不同。

从第 43 行起，legacy builder 把命令写成 `|N VAR=... /bin/sh -c ...`，BuildKit 写成 `RUN |N VAR=... /bin/sh -c ... # buildkit`，并且 ARG 的书写顺序不同。去掉这些前缀后，shell 正文与 Dockerfile 是同一条指令。表里把这类行收成同一句摘要。真正的命令文本差异是第 7 行的基础镜像 `ADD file:<hash>`，以及若干 `COPY` 在 legacy history 里显示内容哈希、在 BuildKit history 里显示源路径。

| # | 自建大小 | GHCR 大小 | 指令摘要 |
|---:|---:|---:|---|
| 0 | 0B | 0B | ARG BUILD_DATE |
| 1 | 0B | 0B | ARG BUILD_DATE VERSION |
| 2 | 0B | 0B | ARG BUILD_DATE PROJECT_URL VERSION |
| 3 | 0B | 0B | ARG BUILD_DATE PROJECT_URL VCS_REF VERSION |
| 4 | 0B | 0B | ARG BUILD_DATE PROJECT_URL TARBALL VCS_REF VERSION |
| 5 | 0B | 0B | ARG BUILD_DATE PROJECT_URL RELEASE_DESCRIPTION TARBALL VCS_REF VERSION |
| 6 | 0B | 0B | LABEL org.opencontainers.image.*（Kali 基础镜像标签） |
| 7 | 0B | 0B | ADD 基础根文件系统。自建 file:9309d270770e…，GHCR file:1d6bc1f338ff… |
| 8 | 0B | 0B | ENV LANG=C.UTF-8 |
| 9 | 122MB * | 121MB | CMD ["bash"] |
| 10 | 0B | 0B | LABEL description=AI Agent Penetration Testing Environment... |
| 11 | 43MB * | 46.6MB | apt-get upgrade（安装 kali-archive-keyring 与 sudo 之后） |
| 12 | 34.5kB | 34.5kB | useradd pentester，并写入 NOPASSWD sudoers |
| 13 | 0B | 0B | mkdir /home/pentester/tools /app/certs |
| 14 | 2.24GB | 2.24GB | apt-get install 主工具集（含 chromium、nmap、nuclei、nodejs、golang-go） |
| 15 | 165B | 165B | setcap nmap |
| 16 | 0B | 0B | USER pentester |
| 17 | 2.27kB | 2.27kB | openssl 生成测试用 CA |
| 18 | 0B | 0B | USER root |
| 19 | 182kB | 182kB | update-ca-certificates |
| 20 | 38.3MB * | 37.8MB | curl https://astral.sh/uv/install.sh | sh |
| 21 | 0B | 0B | USER pentester |
| 22 | 0B | 0B | WORKDIR /tmp |
| 23 | 272MB | 272MB | COPY Go 工具二进制到 /home/pentester/go/bin/。history 写法不同：自建是内容哈希，GHCR 是 /out/bin/ 路径 |
| 24 | 49.7MB * | 20.2kB | nuclei -update-templates |
| 25 | 34.2MB | 34.2MB | pipx install arjun、dirsearch、wafw00f |
| 26 | 0B | 0B | ENV NPM_CONFIG_PREFIX |
| 27 | 0B | 0B | mkdir npm-global |
| 28 | 159MB * | 157MB | npm install -g retire/eslint/js-beautify/@ast-grep/cli/tree-sitter-cli/agent-browser |
| 29 | 0B | 0B | ENV AGENT_BROWSER_EXECUTABLE_PATH |
| 30 | 0B | 0B | ENV AGENT_BROWSER_USER_AGENT |
| 31 | 0B | 0B | ENV AGENT_BROWSER_ARGS |
| 32 | 0B | 0B | ENV AGENT_BROWSER_SCREENSHOT_DIR |
| 33 | 0B | 0B | ENV AGENT_BROWSER_IDLE_TIMEOUT_MS |
| 34 | 0B | 0B | USER root |
| 35 | 597B | 597B | 写入 /etc/profile.d/agent-browser.sh |
| 36 | 0B | 0B | USER pentester |
| 37 | 11MB | 11MB | agent-browser doctor --offline --quick |
| 38 | 47.8MB | 47.8MB | git clone tree-sitter 语法仓库 |
| 39 | 0B | 0B | WORKDIR /home/pentester/tools |
| 40 | 440kB | 440kB | git clone JS-Snooper、jsniper.sh、jwt_tool |
| 41 | 0B | 0B | USER root |
| 42 | 0B | 0B | ARG TRUFFLEHOG_VERSION=3.99.2 |
| 43 | 116MB * | 36.5MB | 安装 trufflehog v3.99.2（脚本取 main 分支） |
| 44 | 22MB | 22MB | 按 GitHub releases/latest 安装 gitleaks |
| 45 | 0B | 0B | ARG TRIVY_VERSION=0.76.1 |
| 46 | 0B | 0B | ARG TRIVY_SHA256_X86=... |
| 47 | 0B | 0B | ARG TRIVY_SHA256_AARCH64=... |
| 48 | 176MB | 176MB | 安装 trivy 0.76.1 并校验 sha256 |
| 49 | 196MB * | 195MB | apt-get install wapiti |
| 50 | 0B | 0B | ARG BROTLI_DEB_VERSION=1.2.0-4.1 |
| 51 | 4.81MB | 4.81MB | 从 deb.debian.org 安装 brotli 1.2.0-4.1 |
| 52 | 0B | 0B | ARG OPENGREP_VERSION=1.31.0 |
| 53 | 0B | 0B | ARG OPENGREP_SHA256_X86=... |
| 54 | 0B | 0B | ARG OPENGREP_SHA256_AARCH64=... |
| 55 | 303MB | 303MB | 安装 opengrep 1.31.0 并校验 sha256 |
| 56 | 0B | 0B | ARG OPENGREP_RULES_REF=f1d2b562b414783763fd02a6ed2736eaed622efa |
| 57 | 2.71MB | 2.71MB | 下载并裁剪 opengrep-rules |
| 58 | 1.3kB | 1.3kB | COPY containers/opengrep-rules/。history 写法不同，镜像内目标文件 sha256 相同 |
| 59 | 1.3kB * | 0B | chmod /opt/opengrep-rules/strix |
| 60 | 0B | 0B | USER pentester |
| 61 | 10.1MB | 10.1MB | pipx install bandit |
| 62 | 9.64MB * | 9.59MB | npm install -g jshint |
| 63 | 0B | 0B | USER root |
| 64 | 0B | 0B | apt-get autoremove / autoclean，删除 apt 列表、文档和 man |
| 65 | 0B | 0B | ENV PATH=... |
| 66 | 0B | 0B | ENV VIRTUAL_ENV=/app/.venv |
| 67 | 0B | 0B | WORKDIR /app |
| 68 | 0B | 0B | ARG CAIDO_VERSION=0.56.0 |
| 69 | 188MB | 188MB | wget 并安装 caido-cli 0.56.0 |
| 70 | 0B | 0B | ENV REQUESTS_CA_BUNDLE |
| 71 | 0B | 0B | ENV SSL_CERT_FILE |
| 72 | 2.27kB * | 0B | mkdir /workspace 并 chown |
| 73 | 0B | 0B | USER pentester |
| 74 | 89.9MB * | 89.7MB | 创建 venv 并 pip install 未锁版本的 Python 包 |
| 75 | 24.8kB | 24.8kB | COPY caido_api.py。history 写法不同，镜像内文件 sha256 相同 |
| 76 | 0B | 0B | ENV PYTHONPATH=/opt/strix-python |
| 77 | 6.62kB | 6.62kB | 把 PATH 追加到 .bashrc 和 .profile |
| 78 | 0B | 0B | USER root |
| 79 | 4.5kB | 4.5kB | COPY docker-entrypoint.sh。history 写法不同，镜像内文件 sha256 相同 |
| 80 | 4.5kB | 4.5kB | chmod docker-entrypoint.sh |
| 81 | 0B | 0B | USER pentester |
| 82 | 0B | 0B | WORKDIR /workspace |
| 83 | 0B | 0B | ENTRYPOINT ["docker-entrypoint.sh"] |

`docker history` 的 84 条包含空层。`RootFS.Layers` 只计文件系统 diff：自建 37，GHCR 41。空层如何进入 RootFS，legacy builder 和 BuildKit 不一致；没有把每一条 history 逐一对应到 diff id，这层对应关系未核实。

大小不同的行里，已经用文件内容解释的是第 24 行 nuclei 模板。第 7 行和第 9 行对应不同的 Kali 根文件系统。第 11、20、28、49、62、74 行与后面列出的 apt、uv、npm、Python 包版本差一致，但没有把某一行的 MB 差额逐文件加总。第 43 行 trufflehog 层是 116MB 对 36.5MB，可是最终二进制 sha256 相同；多出来的层体积可能是后来被删除的临时文件，未逐文件核实。第 59 行和第 72 行一边是几 KB、一边是 0B，对应 chmod/chown 一类元数据，BuildKit 可以把它们记成空层。

### 4.3 关键文件

从 overlay2 层按 diff 顺序叠出最终文件，计算 sha256。没有启动容器。

两边 sha256 相同的文件（同一枚哈希，字节数也相同）：

| 路径 | sha256 | 字节数 |
|---|---|---:|
| `/usr/local/bin/docker-entrypoint.sh` | `d088257a30b39b352ad7a0db16ad8e4b59d608616efb5873b12c0a9a5a1add70` | 4495 |
| `/opt/strix-python/caido_api.py` | `5555aef4112ea3479d33f271830483de96ce1e26983d91f03bf4a9ad38f202bb` | 24846 |
| `/opt/opengrep-rules/strix/github-owned-action-not-pinned-to-commit-sha.yaml` | `be6d04eb8ab8328057fccffd241b412a0d3e2eafd1cb734e0ecbdedf258c208e` | 未单独记录 |
| `/usr/local/bin/trivy` | `d1dfca6ae10a1ad4540bb94e64f0510077c47008920a9b554986b39dca573d83` | 176029856 |
| `/usr/local/bin/opengrep` | `dbbe3c2058dbe600c449a8ba22ea79a1643a6660658400090d0323a8d0d31147` | 46484880 |
| `/usr/local/bin/gitleaks` | `88f91962aa2f93ac6ab281d553b9e125f5197bbbce38f9f2437f7299c32e5509` | 21958840 |
| `/usr/local/bin/caido-cli` | `1838d867ac768627aafe195261439ac7e921a8d8377f65d466cc692ee97bb181` | 188372168 |
| `/home/pentester/.local/bin/trufflehog` | `aba71aa8cee0ef87f4adddafbcd778a112b740bfba4a81a2de299d1a8e15f074` | 36491756 |
| `/home/pentester/.local/bin/jwt_tool` | `f91be0707c55c051d990b99ab80c058fbe26613f79f1281d9d4173a8bcd42a86` | 87 |
| `/home/pentester/go/bin/httpx` | `184c62ee9ea371207b368574e404826e1fffe0415b1955db5269ef0f58269a8c` | 69660103 |
| `/home/pentester/go/bin/katana` | `18dbb7a6fd30f32f8c07dacce8863ea41fbd9f4af3ac6d4fee5a61a4cefde2ee` | 84558179 |
| `/home/pentester/go/bin/vulnx` | `d796ed346d9e0ae5cf3f03f88ee10cf0f6b3645dc3840c498f32bc72a5024456` | 42983737 |
| `/home/pentester/go/bin/gospider` | `cdcf9e84df3d512241bac6f993f2386b7e4859fb89d19586b977a64d934d7c6a` | 21286103 |
| `/home/pentester/go/bin/interactsh-client` | `f173f19319ee36dcd0a964f90b946b58660de8710768bfa0fae96088e68f5ac5` | 37623872 |
| `/home/pentester/go/bin/govulncheck` | `f846ba1158aeb23630cdee7f37051aaf1645a9db329dcea101ab7cd0b819068a` | 15913083 |

`/usr/local/bin/opengrep` 的 sha256 与 Dockerfile 第 230 行的 `OPENGREP_SHA256_X86` 相同。该文件是直接下载的二进制，不是 tar 包。trivy 的校验和针对的是 tar 包，所以和上表里的二进制哈希不是同一个值。opengrep 规则文件的字节数没有单独记录，两边哈希相同。

不同的文件：

| 路径 | 自建 sha256 | GHCR sha256 | 说明 |
|---|---|---|---|
| `/usr/local/bin/uv` | `4a7607589912d9c20e86b860b032f7c7f2913c9a3e76f46aef136885387452b8`（37991656 字节） | `4b5e59bc557ff2cd70934beb59d0eaa92820702853d49f0066b50aa11a8f0c52`（37512936 字节） | 未锁版本。syft：0.13.8 对 0.13.7 |
| `/usr/local/bin/uvx` | `c246ee76ad8d606c9347c2774698654235f2016becd87de559662a33f7d13726`（330456 字节） | `77d76bc33a918f167541b0c060db159876dbadcf40e53d190188dafbb07f9d82`（330440 字节） | 随 uv 安装脚本一起变化 |
| `/app/certs/ca.crt` | `4fdbefb88c0cd284ba422ce0707bda800748e50c44771b142434fe5ab54f21ad`（757 字节） | `9a2d0bffe830b181b4b4984952b9232532ab75111033725f03a451c93659965f`（757 字节） | Dockerfile 第 67–82 行每次构建生成新密钥。主题两边都是 `C=US, ST=CA, O=Security Testing, CN=Testing Root CA`。自建 notBefore `Nov 16 16:47:03 2026 GMT`，GHCR notBefore `Nov 4 03:20:43 2026 GMT`。没有把证书或私钥写进本报告 |
| `/var/lib/dpkg/status` | `80458293b4c28b98abc90415c87cf4d6605ce759294960178ac57e10431a2427` | `d0ab47c939104f69c0136ff85722461c783c66d70f31790153257619d70105bd` | 见下面的包版本差 |

### 4.4 包版本差

dpkg 已安装包：自建 929，GHCR 934。只在自建中的包：0。只在 GHCR 中的包：5。版本字符串不同：76。自建一侧全部更新。原因与基础镜像 digest 不同以及 `apt-get upgrade` 未锁快照一致：GHCR 构建用的是 2026-11-01 的 `kali-rolling`，本次用的是 2026-11-15 的 manifest list。

只存在于 GHCR 的包：

| 包 | GHCR 版本 |
|---|---|
| python3-autocommand | 2.2.2-4 |
| python3-jaraco.context | 6.0.1-2 |
| python3-jaraco.functools | 4.1.0-1 |
| python3-more-itertools | 11.2.1-1 |
| python3-zipp | 3.23.0-2 |

版本不同的包：

| 包 | 自建 | GHCR |
|---|---|---|
| apt | 3.3.6+kali1 | 3.3.3+kali1 |
| apt-transport-https | 3.3.6+kali1 | 3.3.3+kali1 |
| apt-utils | 3.3.6+kali1 | 3.3.3+kali1 |
| bash | 5.3-4+b1 | 5.3-4 |
| binutils | 2.47.50.20261021-1 | 2.47-6 |
| binutils-common | 2.47.50.20261021-1 | 2.47-6 |
| binutils-x86-64-linux-gnu | 2.47.50.20261021-1 | 2.47-6 |
| cpp-16 | 16.2.0-4 | 16.2.0-3 |
| cpp-16-x86-64-linux-gnu | 16.2.0-4 | 16.2.0-3 |
| gcc-16 | 16.2.0-4 | 16.2.0-3 |
| gcc-16-base | 16.2.0-4 | 16.2.0-3 |
| gcc-16-x86-64-linux-gnu | 16.2.0-4 | 16.2.0-3 |
| iproute2 | 7.3.0-1 | 7.2.0-1 |
| kali-defaults | 2026.4.1 | 2026.3.2 |
| libapt-pkg7.0 | 3.3.6+kali1 | 3.3.3+kali1 |
| libasan8 | 16.2.0-4 | 16.2.0-3 |
| libatomic1 | 16.2.0-4 | 16.2.0-3 |
| libbinutils | 2.47.50.20261021-1 | 2.47-6 |
| libcap2 | 1:2.78-1+b2 | 1:2.78-1+b1 |
| libcap2-bin | 1:2.78-1+b2 | 1:2.78-1+b1 |
| libcares2 | 1.34.9-1 | 1.34.8-1 |
| libcc1-0 | 16.2.0-4 | 16.2.0-3 |
| libcom-err2 | 1.47.4-1+b3 | 1.47.4-1+b2 |
| libctf-nobfd0 | 2.47.50.20261021-1 | 2.47-6 |
| libctf0 | 2.47.50.20261021-1 | 2.47-6 |
| liberror-perl | 0.17031-1 | 0.17030-1 |
| libgcc-16-dev | 16.2.0-4 | 16.2.0-3 |
| libgcc-s1 | 16.2.0-4 | 16.2.0-3 |
| libgomp1 | 16.2.0-4 | 16.2.0-3 |
| libgprofng0 | 2.47.50.20261021-1 | 2.47-6 |
| libhwasan0 | 16.2.0-4 | 16.2.0-3 |
| libitm1 | 16.2.0-4 | 16.2.0-3 |
| liblsan0 | 16.2.0-4 | 16.2.0-3 |
| libnss3 | 2:3.132-1 | 2:3.129-1 |
| libnss3-tools | 2:3.132-1 | 2:3.129-1 |
| libpython3.14 | 3.14.10-1 | 3.14.8-1 |
| libpython3.14-minimal | 3.14.10-1 | 3.14.8-1 |
| libpython3.14-stdlib | 3.14.10-1 | 3.14.8-1 |
| libquadmath0 | 16.2.0-4 | 16.2.0-3 |
| librsvg2-2 | 2.63.3+dfsg-1 | 2.62.1+dfsg-1 |
| libsframe3 | 2.47.50.20261021-1 | 2.47-6 |
| libstdc++6 | 16.2.0-4 | 16.2.0-3 |
| libtsan2 | 16.2.0-4 | 16.2.0-3 |
| libubsan1 | 16.2.0-4 | 16.2.0-3 |
| libx11-6 | 2:1.8.14-1 | 2:1.8.13-1 |
| libx11-data | 2:1.8.14-1 | 2:1.8.13-1 |
| libx11-xcb1 | 2:1.8.14-1 | 2:1.8.13-1 |
| libxi6 | 2:1.8.4-1 | 2:1.8.3-1 |
| libxpm4 | 1:3.5.20-1 | 1:3.5.19-1 |
| libxrender1 | 1:0.9.13-1 | 1:0.9.12-1+b2 |
| libxtst6 | 2:1.2.6-1 | 2:1.2.5-1+b2 |
| linux-libc-dev | 7.2.15-1kali1 | 7.1.5-1kali1 |
| naabu | 2.7.0-0kali1 | 2.6.1-0kali1 |
| node-ajv | 8.20.0~ds+~cs7.2.4-1 | 8.20.0~ds+~cs7.1.6-2 |
| node-decamelize | 4.0.0-3 | 4.0.0-2 |
| node-ip-address | 10.7.6-1 | 10.7.2-1 |
| node-postcss | 8.5.31+~cs10.2.26-1 | 8.5.28+~cs10.2.24-1 |
| node-serialize-javascript | 7.1.4+~5.0.4-1 | 7.1.2+~5.0.4-1 |
| python3-argon2 | 25.1.0-5 | 25.1.0-4 |
| python3-certifi | 2026.10.23+ds-1 | 2026.7.22+ds-1 |
| python3-cffi | 2.1.1-2 | 2.1.1-1 |
| python3-cffi-backend | 2.1.1-2 | 2.1.1-1+b1 |
| python3-greenlet | 3.3.2-5 | 3.3.2-4+b1 |
| python3-lazr.restfulclient | 4.1.0-1 | 4.0.0-1 |
| python3-lxml | 6.1.3-1.1 | 6.1.3-1 |
| python3-pip | 26.2.2+dfsg-1 | 26.1.2+dfsg-1 |
| python3-pip-whl | 26.2.2+dfsg-1 | 26.1.2+dfsg-1 |
| python3-platformdirs | 4.11.9-1 | 4.10.0-1 |
| python3-pyasn1 | 0.6.5-1 | 0.6.4-1 |
| python3-setuptools | 84.2.0-1 | 84.0.0-2 |
| python3-setuptools-whl | 84.2.0-1 | 84.0.0-2 |
| python3.14 | 3.14.10-1 | 3.14.8-1 |
| python3.14-minimal | 3.14.10-1 | 3.14.8-1 |
| python3.14-venv | 3.14.10-1 | 3.14.8-1 |
| tar | 1.35+dfsg-7 | 1.35+dfsg-6 |
| tzdata | 2026f-1 | 2026e-1 |

syft 1.57.1 的包计数：自建 5743，GHCR 5769（schema 16.1.13）。按 `(type, name)` 比较，只在自建出现的是 npm `@isaacs/nonrecursive-glob-match` 1.1.0；只在 GHCR 出现的是上面 5 个 deb。版本集合不同的名字有 240 个，其中大部分是 uv 的 rust crate（`uv-version` 0.13.8 对 0.13.7，其余 uv-* crate 是 0.0.100 对 0.0.99），以及 naabu 版本变化带进来的 Go 模块，还有 npm 传递依赖。6 个自建 Go 工具和 gitleaks 的主模块版本集合两边相同，二进制也相同。

nuclei 模板：

| 项 | 自建 | GHCR |
|---|---|---|
| history 中 `nuclei -update-templates` 层大小 | 49.7MB | 20.2kB |
| `/home/pentester/.local/nuclei-templates` 下的文件数 | 14557 | 0（目录存在，没有文件） |
| `.templates-config.json` 的 `nuclei-templates-version` | `v10.5.2` | 该字段不存在 |
| `nuclei-templates-latest-version` | `v10.5.2` | `v10.5.1` |
| `nuclei-latest-version` | `v3.11.1` | `v3.11.1` |

同一条 Dockerfile 指令，GHCR 镜像里没有模板文件。没有再运行 nuclei，所以当时更新命令为什么几乎没有写入文件，未核实。

### 4.5 不一致之处的可能原因

能对应到证据的：

- 基础镜像 `kali-rolling:latest` 在 2026-11-01 与 2026-11-15 不是同一个 manifest list，随后的 `apt-get upgrade` 和未锁版本的 apt 包（包括 naabu）因此不同。
- `https://astral.sh/uv/install.sh` 没有版本和校验和，两次构建得到 uv 0.13.7 与 0.13.8。
- npm `@latest` 在两次构建之间前进了：retire、eslint、@ast-grep/cli、tree-sitter-cli。
- `nuclei -update-templates` 没有锁模板版本。自建拿到 v10.5.2 的 14557 个文件；GHCR 镜像中该目录为空，配置里记录的 latest 是 v10.5.1。空目录的直接原因未核实。
- CA 证书由构建时 `openssl` 生成，时间戳和密钥必然不同。
- 自建是 legacy builder，GHCR provenance 是 BuildKit dockerfile frontend 1.28.0。这能解释 history 文本前缀、部分 0B 与几 KB 的元数据层，以及 RootFS 层数 37 对 41。它不能单独解释 apt 版本差。

未核实的：

- trufflehog 层 116MB 对 36.5MB，而最终二进制相同。没有列出该层里多出的路径。
- syft 里重复的 Python 版本分别属于哪个目录。
- Go 模块差异里，除 naabu 包版本不同之外，每一条模块差属于哪一个二进制。
- attestation 签名是否有效，以及 Devin git-manager URL 后面的实际操作者。

## 5. 加固建议

这些只是写法，本报告没有按它们重建镜像，也没有运行 Strix。

锁基础镜像 digest，不要用 `latest`。本次 amd64 子 manifest 是 `sha256:12e581978942a815b35ab2406725c274a4f28825650595d2ec2b7fb7668b039d`，它的 manifest list 是 `sha256:1ff4a2be3849333687b3ed2b31bb5af0cd1b7c0a11ec7713feb5abe5a6b2795c`。多架构构建应锁住 manifest list，或按平台锁子 manifest，并写明锁的是哪一种。

锁工具版本，并给下载文件加校验和：

- `go install` 改为明确的模块版本，或提交 `go.sum` 后用可复现的构建方式。
- uv 安装脚本改为带版本的发布物，并校验 sha256。不要 `curl | sh` 一个会移动的 `install.sh`。
- npm 去掉 `@latest`，`jshint` 写出版本；提交 lockfile。
- pip / pipx 使用带哈希的 requirements 文件。`setuptools<81` 只有上界。
- gitleaks 不要跟随 `releases/latest`。本次碰巧是 v8.30.1，但仍应写死 tag 和 sha256。
- trufflehog 的版本参数已经是 3.99.2，安装脚本仍取 `main`，二进制没有校验和。
- brotli 的 deb 和 caido 的 tar 已有版本，还缺文件校验和。
- `git clone` 改为具体 commit。本次 11 个仓库的提交碰巧相同，不能代替锁定。
- `nuclei -update-templates` 改为固定的模板版本或归档，并校验。否则会出现这次这种一边是 v10.5.2、一边目录为空的结果。
- apt 使用带日期的 snapshot，或在锁了 digest 的基础镜像上不再无约束地 `apt-get upgrade`。

让运行时使用自建镜像时，设置环境变量即可。本仓库 `strix/config/settings.py` 第 166–169 行把 `STRIX_IMAGE` 作为 `RuntimeSettings.image` 的别名，当前默认值是 `ghcr.io/usestrix/strix-sandbox:1.4.0`。`strix/runtime/docker_client.py` 第 206–208 行只在本地不存在该镜像时才 pull。因此在已经有 `strix-sandbox:local` 的机器上：

```bash
export STRIX_IMAGE=strix-sandbox:local
```

不要把这个变量指到一个本地不存在、又会触发 pull 的名字。本报告没有导出该变量，也没有启动 `strix`。

## 6. 未覆盖项

- 没有 `docker run`，没有 `docker start`，没有执行镜像里的任何工具来读取 `--version`。版本来自包元数据、syft、构建日志和文件哈希。
- 没有构建 linux/arm64，也没有把 GHCR 的 arm64 manifest 拉下来做文件比对。arm64 attestation 只读了 statement 字段。
- 没有拉取或比对今天的 `latest`（index `sha256:c8e70302ba4a69b678b4001c909b134d8d0b84625d7b23feed59ece02672c1d2`），也没有拉取 `1.4.1`。上游 main 和 v1.7.1 的默认 `STRIX_IMAGE` 已经是 `ghcr.io/usestrix/strix-sandbox:1.4.1`；本工作副本的 `settings.py` 和 `scripts/install.sh` 仍是 `1.4.0`。
- 没有构建上游 main 那份已改过的 Dockerfile。
- 没有使用 dive。没有对整棵文件系统做逐文件 diff，只覆盖了 syft 包清单和上文点名的路径。
- 没有校验 GHCR attestation 的签名。
- 没有把 provenance 里的 Kali manifest list `sha256:7997e058…` 的 amd64 子镜像拉到本地再和本次基础镜像做文件级 diff。
- `checks.yml` 与 `typecheck.yml` 的正文未读到。
- npm registry 与 PyPI 的主机名没有出现在构建日志里。GOPROXY 主机也没有出现。
- GHCR nuclei 模板目录为空的直接原因未核实。trufflehog 层体积差的具体文件未核实。

