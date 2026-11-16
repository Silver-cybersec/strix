# 锁定版本后的 Strix 沙箱构建

这次构建使用仓库里改过的 `containers/Dockerfile` 和 `scripts/install.sh`。没有执行 `docker run`，没有启动沙箱容器，没有运行 `strix` CLI，没有连接目标，也没有调用 LLM API。

构建发生在提交之前。当时 `HEAD` 仍是 `215482eb2204cd30ac6b87520e3f5c6511e2e2e8`，工作区里是本文件所描述的锁定改动。镜像构建完成之后才把这些文件提交。提交之后没有再改 Dockerfile。

## 构建环境

| 项 | 值 |
|---|---|
| 操作系统 | Ubuntu 24.04.5 LTS |
| 内核 | `Linux cursor 6.12.113+ #1 SMP PREEMPT_DYNAMIC Mon Nov 16 16:06:22 UTC 2026 x86_64` |
| CPU 架构 | `x86_64`，`nproc` = 4 |
| Docker 客户端 / 引擎 | 29.1.3，Git commit `29.1.3-0ubuntu3~24.04.2`，API 1.52，Go `go1.24.4`，构建时间 `Wed Apr 29 16:41:06 2026` |
| containerd | 2.2.1 |
| runc | `1.3.4-0ubuntu1~24.04.1` |
| docker-init | 0.19.0 |
| BuildKit / buildx | 未安装。`docker buildx version` 的输出是 `docker: unknown command: docker buildx`。`docker build` 使用 legacy builder。日志原文：`DEPRECATED: The legacy builder is deprecated and will be removed in a future release.` |
| 第一次命令 | `BUILDKIT_PROGRESS=plain bash scripts/docker.sh pinned`。脚本实际执行 `docker build -f containers/Dockerfile -t strix-sandbox:pinned <仓库根>` |
| 第一次开始 | 2026-11-17 02:14:43 HKT（UTC `2026-11-16T18:14:43Z`） |
| 第一次结束 | 2026-11-17 02:27:02 HKT（UTC `2026-11-16T18:27:02Z`） |
| 第一次耗时 | 739 秒，退出码 0 |
| 第二次命令 | `docker build --no-cache -f containers/Dockerfile -t strix-sandbox:pinned-repro .` |
| 第二次开始 | 2026-11-17 02:28:04 HKT（UTC `2026-11-16T18:28:04Z`） |
| 第二次结束 | 2026-11-17 02:53:00 HKT（UTC `2026-11-16T18:53:00Z`） |
| 第二次耗时 | 1496 秒，退出码 0 |

第一次构建日志里实际出现的网络主机：

- apt：`http.kali.org`、`kali.download`、`mirrors.ocf.berkeley.edu`、`kali.darklab.sh`、`us.mirror.ionos.com`。镜像内配置的源仍是 `http://http.kali.org/kali/` 的 `kali-rolling`，其余主机是 apt 这次访问到的镜像。
- `github.com`（Go 工具、uv、gitleaks、trivy、opengrep、trufflehog、git fetch）
- `caido.download`，以及它转到的 `011d6d2c3b16566b4a68d3410328c12a.r2.cloudflarestorage.com`
- `deb.debian.org`（brotli 的 deb）
- `docs.docker.com`（legacy builder 的提示文字，不是依赖下载）

日志没有打印 Go module 代理、npm registry 或 PyPI 的主机名。这三项走的是 `go install`、`npm install` 和 `pip install` 的默认源，具体主机未在日志中出现。

## 最终 digest 与两次构建

| 构建 | 标签 | 镜像 ID |
|---|---|---|
| 第一次 | `strix-sandbox:pinned` | `sha256:7a0dd7c76edfcb921d3890a299656e1e131c5d6c15d9ff25e306ce65ef693ab6` |
| 第二次（`--no-cache`） | `strix-sandbox:pinned-repro` | `sha256:1001dab848a2d7ea26074e9d5e7f7e631c74713333086cf0beabab06c6cc036a` |

两次镜像 ID 不一致。

`docker image inspect` 的 Size：第一次 4239491815 字节，第二次 4239494428 字节，相差 2613 字节。两边都是 linux/amd64，用户 `pentester`，入口 `["docker-entrypoint.sh"]`，工作目录 `/workspace`，`Config.Env` 相同。RootFS 各 36 层，逐层 diff id 只有第 0 层相同，这一层是锁定的基础镜像。其余 35 层的 diff id 都不同。`docker history` 各 98 条，大小字符串全部相同。

不一致的直接原因有两件，都还在：

- legacy builder 把构建时刻写进镜像 config 的 `Created`（第一次 `2026-11-16T18:27:02.764817066Z`，第二次 `2026-11-16T18:53:00.662509495Z`）以及层内文件时间。所以即使文件内容相同，层 digest 和镜像 ID 也会变。
- Dockerfile 仍在构建时用 `openssl` 生成新的 CA 密钥，两次密钥不同。

没有把其余 35 层逐一拆成“只有时间戳不同”或“CA 文件不同”。那部分未核实。

## 锁定版本清单

校验方式一栏里，“上游 checksum 文件”表示哈希抄自该项目发布的 checksum，并在本机用下载到的文件核对过的会另行注明。“本机计算”表示发布页没有单独的 checksum 文件，哈希是这次 `sha256sum` 算出来的。

| 组件 | 原写法 | 锁定值 | 校验方式 |
|---|---|---|---|
| 基础镜像 | `kalilinux/kali-rolling:latest` | `kalilinux/kali-rolling@sha256:1ff4a2be3849333687b3ed2b31bb5af0cd1b7c0a11ec7713feb5abe5a6b2795c` | 使用上一份未锁定构建里 `docker pull` 打印的 manifest list digest。legacy builder 接受这个引用，解析到镜像 ID `sha256:333b656011f6783f6a8c6e508db294af422346d352635692e3f95a8653c4bcae`。该 list 的 linux/amd64 子 manifest 是 `sha256:12e581978942a815b35ab2406725c274a4f28825650595d2ec2b7fb7668b039d` |
| apt 新增或相对基础镜像有变化的包 | 不带版本的 `apt-get install`，外加 `apt-get upgrade` | `containers/pinned/apt-runtime.txt` 里 851 个 `包=版本`。`apt-get upgrade` 已去掉 | 版本来自 2026-11-16 那次镜像的 dpkg status，和基础镜像 dpkg 对比后只保留有差异的项。没有文件级 sha256 |
| gobuilder 的 apt | `golang-go git ca-certificates` 不带版本 | `containers/pinned/apt-gobuilder.txt`，版本来自那次构建日志的 `Unpacking` 行 | 同上，无 sha256 |
| `kali-archive-keyring` | 不带版本 | `2025.3` | apt 版本号 |
| `sudo` | 不带版本 | `1.9.17p2-9` | apt 版本号 |
| `dnsutils` | 包名 `dnsutils` | 镜像里实际装上的是 `bind9-dnsutils=1:9.20.30-1`，写在 apt-runtime.txt | 虚拟包名没有独立的 dpkg 版本 |
| httpx | `go install ...@latest` | `v1.12.0` | 模块版本。传递依赖由该 tag 的 `go.sum` 决定，本仓库没有另存 go.sum |
| katana | `@latest` | `v1.8.0` | 同上 |
| vulnx | `@latest` | `github.com/projectdiscovery/cvemap/cmd/vulnx@v1.0.0` | 同上 |
| gospider | `@latest` | `v1.1.6` | 同上 |
| interactsh-client | `@latest` | `v1.4.1` | 同上 |
| govulncheck | `@latest` | `golang.org/x/vuln/cmd/govulncheck@v1.9.0` | 同上 |
| uv / uvx | `curl https://astral.sh/uv/install.sh \| sh` | `0.13.8`，文件 `uv-x86_64-unknown-linux-gnu.tar.gz` | 上游 `sha256.sum`。amd64 哈希 `5acfd9177cfecf0bf87578f059728215ccc884388988f55f7a65041b57387d91` 已在本机 `sha256sum -c` 通过。arm64 哈希 `74f2f5c6024d397481db228db8fc25eed420db192243f88438c118e7e013e585` 来自同一份 `sha256.sum`，arm64 压缩包没有在这台机器上下载 |
| retire | `npm install -g retire@latest` | `6.1.1` | npm 版本号。同版本不能覆盖发布。传递依赖的范围没有 lockfile |
| eslint | `@latest` | `10.15.0` | 同上 |
| js-beautify | `@latest` | `2.0.3` | 同上 |
| @ast-grep/cli | `@latest` | `0.51.3` | 同上 |
| tree-sitter-cli | `@latest` | `0.27.2` | 同上 |
| agent-browser | `0.26.0` | `0.26.0` | 原本已锁版本 |
| jshint | `npm install -g jshint` | `2.13.6` | npm 版本号 |
| arjun 及其依赖 | `pipx install arjun` | `arjun==2.2.7` 加上当时 venv 里的依赖版本，见 `containers/pinned/arjun-requirements.txt` | `pip install --require-hashes`。哈希来自 `https://pypi.org/pypi/<name>/<version>/json` 的 `digests.sha256` |
| dirsearch | `pipx install` 加 `setuptools<81` | `dirsearch==0.5.0`，`setuptools==80.10.2`，其余依赖见 `dirsearch-requirements.txt` | 同上 |
| wafw00f | `pipx install wafw00f` | `wafw00f==2.4.2` 及依赖，见 `wafw00f-requirements.txt` | 同上 |
| bandit | `pipx install bandit` | `bandit==1.9.4` 及依赖，见 `bandit-requirements.txt` | 同上 |
| 主 venv（caido SDK、requests、httpx、beautifulsoup4、lxml、PyJWT、cryptography、jwt_tool 依赖） | 不带版本的 `pip install` | `containers/pinned/venv-requirements.txt`，直接包为 `caido-sdk-client==0.3.2`、`requests==2.34.2`、`httpx==0.28.1`、`beautifulsoup4==4.15.0`、`lxml==6.1.3`、`PyJWT==2.15.1`、`cryptography==50.0.2` | 同上 |
| tree-sitter-java | `git clone` 默认分支 | `e10607b45ff745f5f876bfa3e94fbcc6b44bdc11` | `git fetch --depth 1` 该提交 |
| tree-sitter-javascript | 同上 | `58404d8cf191d69f2674a8fd507bd5776f46cb11` | 同上 |
| tree-sitter-python | 同上 | `26855eabccb19c6abf499fbc5b8dc7cc9ab8bc64` | 同上 |
| tree-sitter-go | 同上 | `2346a3ab1bb3857b48b29d779a1ef9799a248cd7` | 同上 |
| tree-sitter-bash | 同上 | `a06c2e4415e9bc0346c6b86d401879ffb44058f7` | 同上 |
| tree-sitter-json | 同上 | `254c42a6476413b776221e03982ac8ae159eeb72` | 同上 |
| tree-sitter-yaml | 同上 | `a1c4812a73ec5e089de8e441fdea3a921e8d5079` | 同上 |
| tree-sitter-typescript | 同上 | `75b3874edb2dc714fb1fd77a32013d0f8699989f` | 同上 |
| JS-Snooper | `git clone` | `932704cbce1a04ebe89a558f395a6e474e01b15b` | 同上 |
| jsniper.sh | `git clone` | `65d0b508dc88e6de3dfe6ac60e6d89eca22be686` | 同上 |
| jwt_tool | `git clone` | `3bc7407cf2222d6a821dcc19c776e5a1b1cb9a9b` | 同上 |
| nuclei 模板 | `nuclei -update-templates` | tag `v10.5.2` 指向的提交 `5442582a875f96887bfe403c6de562de460897e7` | git 提交。见下方无法用该项目 checksum 文件的原因 |
| opengrep 规则 | 已有提交 `f1d2b562b414783763fd02a6ed2736eaed622efa`，用 codeload tarball | 同一提交，改为 `git fetch` | git 提交。GitHub codeload 压缩包的字节不稳定，所以没有给 tarball 加 sha256 |
| trufflehog | 版本参数 `3.99.2`，安装脚本取 `main` | `v3.99.2` 的 `trufflehog_3.99.2_linux_amd64.tar.gz` | 上游 `trufflehog_3.99.2_checksums.txt`。amd64 哈希 `3c7b0d1dc4278294269bfdd7594bca8bb28c78da0a4a06ad202ded37b48deca4` 已在本机核对。arm64 哈希 `929b81efffca752eafd58bf6f17bf725d354f338636a7bbbee65c086d2f9ddf3` 来自同一 checksum 文件，压缩包未在本机下载 |
| gitleaks | `releases/latest` | `v8.30.1` | 上游 `gitleaks_8.30.1_checksums.txt`。x64 哈希 `551f6fc83ea457d62a0d98237cbad105af8d557003051f41f3e7ca7b3f2470eb` 已核对。arm64 哈希 `e4a487ee7ccd7d3a7f7ec08657610aa3606637dab924210b3aee62570fb4b080` 来自同一文件，未再下载 |
| trivy | 已有 `0.76.1` 和 sha256 | 未改 | 原 Dockerfile 已校验 |
| opengrep | 已有 `1.31.0` 和 sha256 | 未改 | 原 Dockerfile 已校验 |
| brotli deb | 版本 `1.2.0-4.1`，无校验和 | 同一版本，增加 sha256 | 本机下载后计算。amd64 `libbrotli1` `2d58db32250aa535e27c62e98451debd43b539aa026066806432e842d5b56c00`，`python3-brotli` `3b62003bf956337d2ae672b40174cd303e740ea0e013d8405e503a4b5abaf061`。arm64 分别是 `ff90055f0c42dc39fdf8f12d7f68c7da32da2ec90d0a49eb39e80b764988ee02` 和 `0923c03d9293e45cd8f744f1de5103b97603f0ae0c54b275d8f03ebf87b29750` |
| caido-cli | `0.56.0`，无校验和 | `0.56.0` | 本机下载后计算。x86_64 `1bc13f1ad558cd333924890d9d09b73f1cdf6726b7855265cd4c4ddece0211d3`，aarch64 `5fe8c37ac0435ce5924014cf035902686051be86831824744e435636a62d3689` |
| `scripts/install.sh` 的 CLI | `releases/latest` | `1.7.0`，与本仓库 `pyproject.toml` 的 version 相同 | 本机下载五个发布文件后计算 sha256：linux-x86_64 `b8cd7e84d63cbeff6726301a9e08fddedf40fc6a4edcdb4edda8a853b3312de6`，linux-arm64 `9f4a826c5d1e76b5c503e3299807af8549acca2e03734b6603611be275917ede`，macos-x86_64 `e425f000044741a9bebbedb265ef66efeabba48e45ac15374b1df9c092b5df3e`，macos-arm64 `26607010b1822a7a7b0353d372cc00be149f91d1382e8a99df96ab4335afc6a0`，windows-x86_64 `04e3290f4c495b952c10da2f3627764be3e9ac598dd5a0a79bf65c112d1200f7` |
| `scripts/install.sh` 的沙箱镜像 | `ghcr.io/usestrix/strix-sandbox:1.4.0` | `ghcr.io/usestrix/strix-sandbox@sha256:9ff1be4044c3ff5e9fcecb118b9855f8d1270db39358523e43fb3a13c0027009` | 上一份报告里 `docker pull` 打印的 index digest。`strix/config/settings.py` 的默认值改成了同一个 digest |

`scripts/build.sh` 使用 `uv sync --frozen`，依赖已经由 `uv.lock` 锁住，这次没有改它。`scripts/docker.sh` 本身不下载文件，只调用 `docker build`。

## 无法锁定或没有改成快照的项

- Kali 没有与 Debian `snapshot.debian.org` 对应的、按某一天冻结 `kali-rolling` 的源。公开的 `kali-last-snapshot` 是另一个套件；2026-11-16 查看 `https://archive.kali.org/kali/dists/kali-last-snapshot/` 时，其 `main` 目录时间是 2026-10-09，对不上这次基础镜像的 2026-11-15 包集合，所以没有改用它。apt 改为钉死 851 个版本号。这些版本以后可能从 `kali-rolling` 镜像上消失，那时构建会失败，而不是悄悄装到别的版本。
- npm 只锁了顶层版本。这些包的依赖范围没有生成 lockfile。镜像里的 npm 是 12.2.0，宿主机没有用同一版本的 npm 去解析依赖树。npm 不允许覆盖已发布的同一版本，所以顶层 tarball 是固定的，传递依赖仍可能变。
- `go install module@vX.Y.Z` 锁的是工具自己的版本。传递模块由上游该 tag 的 `go.sum` 决定，本仓库没有把那份 go.sum 复制进来。
- nuclei-templates `v10.5.2` 的 GitHub release 里有 `nuclei-templates-10.5.2_checksums.txt`，其中写了 `nuclei-templates-10.5.2.tar.gz` 的 sha256 `016b7abbf931d4d1a851a5a568e44510dc61691dc8a5202e3b24e701515ec574`。对 `https://github.com/projectdiscovery/nuclei-templates/releases/download/v10.5.2/nuclei-templates-10.5.2.tar.gz` 的请求返回 HTTP 404，所以没有用这个哈希，改成检出 tag 指向的提交 `5442582a875f96887bfe403c6de562de460897e7`。
- 构建时生成的 CA 密钥没有锁定。把它做成固定密钥会把私钥放进仓库。
- legacy builder 没有稳定的层时间戳，所以两次 `docker build` 的镜像 ID 不会相同。这次没有安装 BuildKit。
- `strix/interface/update_check.py` 仍会请求 GitHub `releases/latest` 或 PyPI 的 JSON，用来比较是否有更新。它不下载沙箱镜像，这次没有改。
