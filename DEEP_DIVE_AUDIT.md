# Strix 只读安全细查

本文件补充 `SECURITY_AUDIT.md`（分支 `cursor/security-audit-16c0`，提交 `875af4fc4d2452ca890cde646bef99712bab1048`，对象快照 `215482eb2204cd30ac6b87520e3f5c6511e2e2e8`）。那份审计不在当前 `main` 上。这里不改它的结论，只复核它标成未确认的三件事。

- 细查日期：2026-11-09
- 源码快照：`Silver-cybersec/strix` `main` = `215482eb2204cd30ac6b87520e3f5c6511e2e2e8`。GitHub API 显示它与 `usestrix/strix` 的 `main` 相同。
- 方式：读源码、Dockerfile、lock；用 registry API 读 manifest、config、attestation，并把层下载成 tar 后列文件、读文本、算哈希。核对 PyPI、GitHub tag 和 OSV。没有 `docker pull` / `docker run`，没有执行镜像里的二进制或本仓库代码，没有安装本仓库，没有对扫描目标发请求。

状态词：**已核实**、**有冲突**、**无法确认**。

## 1. 结论摘要

| 项 | 一句结论 | 对原审计定级的影响 |
| --- | --- | --- |
| (1) GHCR `strix-sandbox:1.4.0` 与仓库 Dockerfile | 两个架构的第 0 层等于官方 `kalilinux/kali-rolling` 2026.11.01；构建历史能对上当前 Dockerfile 的运行阶段；入口脚本、`caido_api.py` 和 Strix 规则与仓库字节相同。没有多出来的登录用户、入口或工具层 setuid。安装命令带进来的附属文件存在。`nuclei -update-templates` 没有留下模板文件。 | 原审计第 3.5 / 4.3 节“已发布镜像是否等于这份 Dockerfile”从**无法确认**改为**已核实**（运行阶段一致）。按 Dockerfile 构建的供应链评级**维持高**：未钉版本的 `@latest` / `git clone` 在镜像里有解析结果，且 SLSA 依赖列表不完整。 |
| (2) `SYS_ADMIN`、FUSE、其他高权限 capability | 请求点只有 `strix/runtime/docker_client.py`。`SYS_ADMIN`、`/dev/fuse` 和 `apparmor:unconfined` 仅在 manifest 含 FUSE 或特定挂载时写入；默认 manifest 没有这些挂载，**默认关闭**。每次创建都追加 `NET_ADMIN` 和 `NET_RAW`。仓库没有 compose 或 Kubernetes manifest。 | 原审计“产品固有危险能力：高”**维持**。补充的是：高权限里默认打开的是 `NET_ADMIN`/`NET_RAW`，不是 `SYS_ADMIN`。这不构成下调。 |
| (3) `openai-agents` 0.19.0 | `uv.lock` 的 sdist 与 wheel 哈希等于 PyPI 发布件，也等于本次下载件。sdist、wheel、GitHub tag `v0.19.0` 的 `src/agents` 共 302 个文件，内容哈希一致。没有 `tool.uv.sources`、git 替换或补丁。一方代码里的容器创建函数是按上游方法改写的调用方，不是改过的依赖包。 | 原审计“按 lock 的供应链：中”**维持**。第 4.3 节“0.19.0 是否增加了被这份拷贝丢掉的限制”从**无法确认**改为**已核实**：0.19.0 的 `_create_container` 没有 `cap_drop` / `privileged` / seccomp；拷贝保留了 `SYS_ADMIN` 分支，并额外无条件加上 `NET_ADMIN`/`NET_RAW`。注释里的 `0.14.6` 行号与 lock **有冲突**，但不表示依赖被替换。 |

## 2. 逐项细查

### 2.1 镜像与 Dockerfile

**已核实：发布坐标。**

| 对象 | 值 |
| --- | --- |
| tag `1.4.0` 与 tag `latest`（本次读取时） | 同一 index `sha256:9ff1be4044c3ff5e9fcecb118b9855f8d1270db39358523e43fb3a13c0027009` |
| index 媒体类型 | `application/vnd.oci.image.index.v1+json` |
| linux/amd64 manifest | `sha256:9ee5a4faf67907f3a9880adeacb7b8395b3f1e7cae56019dcd5839d264c79d08`（41 层，压缩合计 1,479,288,451 字节） |
| linux/arm64 manifest | `sha256:268023698313c0e6ff7d47aaa3e915f09a8c21003cccc0da1f7fdfec0b701204`（41 层，压缩合计 1,441,030,439 字节） |
| amd64 config | `sha256:df5a518208e4e1f46777b01b87a86fc619279308a22327e12ac4e5344d020947`，`created` `2026-11-04T03:50:21Z` |
| arm64 config | `sha256:1b6ef7abca09032e6fb363484182fc9dc9d6c19e7ebf88a61bfed64c4856dd20`，`created` `2026-11-04T04:12:21Z` |
| 仓库默认引用 | `strix/config/settings.py:166-169`，`scripts/install.sh:7`，tag，无 digest |

manifest 下载后的 SHA-256 与上述 digest 一致。amd64 全部 41 层、以及 arm64 上抽查的层，blob SHA-256 都与 manifest 一致。

**已核实：基础层是官方 Kali，不是另一份 rootfs。**

SLSA 语句里的基础镜像是 manifest list `sha256:7997e0581c4f5c185cfe6773d5cae8fcc7c6b79798726ccaabf843af4bcd2013`。本次从 Docker Hub 读到的该 digest 是 `kalilinux/kali-rolling` 的 list，标签版本 `2026.11.01`。

| 架构 | 官方平台 manifest | 层 digest | 与 sandbox 第 0 层 | rootfs diff_id |
| --- | --- | --- | --- | --- |
| amd64 | `sha256:73915d5d54833d498f52fd200960126be4a2dbdcc110d75d70839c4ef155ed03` | `sha256:b3afd4193bfcb1045ee72c25fa9287ee9b851a1d4cef49358be7bd5b3a29032b` | 相同 | `sha256:193f3085027c10d0ba8c973365cdac1e534fdbcd78a75e4ffc63862b5a981d09`，与 sandbox config 的 `diff_ids[0]` 相同 |
| arm64 | `sha256:5032046cc734e69016adb615ce96f52d2b5304f0b9cf1583bd2e849602da894d` | `sha256:822917a1a55869dca403d95626d5bcd720882e242eac17673daa0e930c781104` | 相同 | `sha256:6602f0c3ca1ccbe37da6b36a7397f334446358d6a5a76a2ad4cb5b8a4b7e7014`，与 sandbox config 的 `diff_ids[0]` 相同 |

Dockerfile 第 7 行和第 25 行的 `FROM kalilinux/kali-rolling:latest` 在这次构建里解析到的就是这份 2026-11-01 镜像。`latest` 以后会移动；这次发布件钉住的是上面的 digest。

**已核实：最终配置里的用户和启动项就是 Dockerfile 写的那些。**

两个架构的 config 都是：

- `User`: `pentester`（`containers/Dockerfile:333`）
- `Entrypoint`: `["docker-entrypoint.sh"]`（`containers/Dockerfile:336`）
- `Cmd`: 空。基础镜像历史里的 `CMD ["bash"]` 没有留在最终 config。
- `WorkingDir`: `/workspace`
- 环境变量与 Dockerfile 的 `ENV` 对应，包括 `PYTHONPATH=/opt/strix-python` 和一组 `AGENT_BROWSER_*`。

amd64 与 arm64 的 84 条 history `created_by` 只有一条不同：基础镜像 `ADD file:` 的内部 id。从 `LABEL description=...` 起，运行阶段的 `RUN` / `COPY` / `USER` / `ENTRYPOINT` 与当前 `containers/Dockerfile` 的运行阶段同文，包括 `TRUFFLEHOG_VERSION=3.99.2`、`TRIVY_VERSION=0.76.1`、`OPENGREP_VERSION=1.31.0`、`OPENGREP_RULES_REF=f1d2b562b414783763fd02a6ed2736eaed622efa`、`CAIDO_VERSION=0.56.0`。多阶段 `gobuilder` 不出现在最终 history 里；它的产物在 `COPY /out/bin/` 那一层。

provenance 记录的源提交是 `4d793375e06650a1ff6fcab0a6bba62b542c8edf`（`usestrix/strix`，2026-11-04T03:49:36Z，说明为 trivy 校验）。该提交的 `containers/Dockerfile` blob 是 `a1ff49ef0e2246b982f5a5670edde29212947572`，与当前 `HEAD` 的该文件 blob 相同。`git diff 4d793375 HEAD -- containers strix/tools/proxy/caido_api.py` 为空。GitHub compare `4d793375...215482e` 的状态是 `diverged`（ahead 5、behind 7），分叉不在这些镜像输入路径上。

**已核实：直接 COPY 进镜像的文件与仓库一致。** amd64 与 arm64 这些层的文件字节相同。

| 镜像路径 | 层（amd64） | SHA-256 | 与仓库 |
| --- | --- | --- | --- |
| `/usr/local/bin/docker-entrypoint.sh` | 39，mode `0775`，4495 字节 | `d088257a30b39b352ad7a0db16ad8e4b59d608616efb5873b12c0a9a5a1add70` | 等于 `containers/docker-entrypoint.sh` |
| `/opt/strix-python/caido_api.py` | 36，24846 字节 | `5555aef4112ea3479d33f271830483de96ce1e26983d91f03bf4a9ad38f202bb` | 等于 `strix/tools/proxy/caido_api.py` |
| `/opt/opengrep-rules/strix/github-owned-action-not-pinned-to-commit-sha.yaml` | 27，1300 字节 | `be6d04eb8ab8328057fccffd241b412a0d3e2eafd1cb734e0ecbdedf258c208e` | 等于 `containers/opengrep-rules/` 下同名文件 |
| `/etc/profile.d/agent-browser.sh` | 15 | `3f56cd778371f5e6cc731714a61860663d498b8d3ac3cb43742d42e658aec25e` | 仓库无此文件；内容是 Dockerfile 第 122-132 行用五个 `AGENT_BROWSER_*` 变量生成的 profile |
| `/home/pentester/.bashrc` 与 `.profile` 末行 | 37 | — | 末行是 Dockerfile 第 326-327 行写入的同一条 `PATH` |

社区规则层有 1933 个文件（1911 个 `.yaml`、21 个 `.yml`、1 个无后缀）。这与 Dockerfile 第 247-258 行拉取并裁剪 `opengrep-rules` 的步骤相符，本次没有逐条规则做内容审查。

**已核实：用户。** 最终 `/etc/passwd` 在 apt 层（amd64 第 4 层）。登录 shell 只有 `root`（`/bin/bash`）和 `pentester`（`/bin/bash`，uid 1000）。`sync` 的 shell 是 `/bin/sync`。其余账号是 `nologin`。

- 第 0 层（官方 Kali）已有 `root`、`daemon`、`bin`、`sys`、`irc`、`nobody` 等系统账号，没有 `pentester`。
- 第 2 层只增加 `pentester`。这是 Dockerfile 第 34-36 行的 `useradd`。
- 第 4 层再增加 `systemd-network`、`messagebus`、`polkitd`。Dockerfile 没有点这些名字；它们来自该层 `apt-get install` 拉入的依赖。
- `/etc/sudoers` 在第 2 层，末行是 `pentester ALL=(ALL) NOPASSWD:ALL`，与 Dockerfile 第 36 行一致。`%sudo` 那行来自 sudo 软件包自身。没有 `/etc/sudoers.d` 下的额外规则文件。
- `/etc/shadow` 里上述账号的口令字段都是锁定标记，没有可用的口令哈希。字段本身不写入本文。

**已核实：PATH 上的程序名。** 最终出现过的名字：

| 目录 | 名字 | 和 Dockerfile 的关系 |
| --- | --- | --- |
| `/home/pentester/go/bin` | `httpx`、`katana`、`vulnx`、`gospider`、`interactsh-client`、`govulncheck` | 正好是第 15-20 行 `go install` 的六个目标。没有第七个。 |
| `/usr/local/bin` | `uv`、`uvx`、`gitleaks`、`trivy`、`opengrep`、`caido-cli`、`docker-entrypoint.sh` | `uvx` 没有被点名。uv 安装回执写着本次装的是 uv `0.13.7`，`binaries` 为 `uv` 和 `uvx`。其余与 Dockerfile 的安装行对应。 |
| `/home/pentester/.local/bin` | `arjun`、`dirsearch`、`dirsearch-build-native`、`wafw00f`、`trufflehog`、`bandit`、`bandit-baseline`、`bandit-config-generator`、`jwt_tool` | `dirsearch-build-native` 和两个 `bandit-*` 是对应包的附属入口。`jwt_tool` 是 Dockerfile 第 312-316 行写的包装脚本。 |
| `/home/pentester/.npm-global/bin` | `retire`、`eslint`、`js-beautify`、`css-beautify`、`html-beautify`、`ast-grep`、`sg`、`tree-sitter`、`agent-browser`、`jshint` | `css-beautify` / `html-beautify` 随 `js-beautify` 安装。`sg` 是 Dockerfile 第 114 行的链接。 |

`agent-browser doctor` 那一层额外放下 `.../agent-browser/bin/agent-browser-linux-x64`（10,961,608 字节）。这是 Dockerfile 第 135 行 doctor 步骤的产物，不是另一条 `COPY`。

setuid/setgid 只出现在基础层和 apt 层：`chage`、`chfn`、`chsh`、`gpasswd`、`mount`、`umount`、`passwd`、`su`、`unix_chkpwd`、`sudo`，以及 `dbus-daemon-launch-helper`。工具安装层（Go、npm、uv、git clone、trivy、opengrep、caido、入口脚本）没有新增 setuid 文件。

`setcap` 那一层只改 `/usr/bin/nmap`。文件带 `security.capability`，解码后的 permitted/inheritable 位是 `cap_net_bind_service`、`cap_net_admin`、`cap_net_raw`，并带 effective 标志。这与 Dockerfile 第 64 行一致。该文件是发行版的 nmap 包装，不是新写入的独立程序；包装正文不收录在本文。

**有冲突：`nuclei -update-templates` 没有留下模板。** Dockerfile 第 96 行有这一步，对应 amd64 第 11 层（压缩 7256 字节）。`/home/pentester/.local/nuclei-templates` 只有空目录。同层的 `.templates-config.json` 记录 `nuclei-latest-version` 为 `v3.11.1`、`nuclei-templates-latest-version` 为 `v10.5.1`。dpkg status 里的 `nuclei` 是 `3.11.1-0kali1`，二进制在 `/usr/bin/nuclei`（142,905,368 字节）。apt 层里文件名含 `nuclei` 的只有该二进制和 `/usr/share/doc/nuclei`。因此“构建时会带上一份模板集”对这张已发布镜像不成立；模板仍可能在运行时再拉取，本次没有执行二进制，运行时行为仍按原审计标为未确认。

**已核实、但 Dockerfile 没有逐个点名的其他文件：**

- `/root/.config/uv/uv-receipt.json`：uv `0.13.7`，来源记录为 astral-sh 的 GitHub release。
- `/root/.cache/opengrep/v1.31.0/`：一串 Python 扩展模块。来自 Dockerfile 里 `opengrep --version` 在 root 下的缓存，不是第二个入口。
- 三个 `git clone` 和八个 tree-sitter 仓库的 `.git` 仍在。克隆记录的提交在 GitHub 上存在，见第 3 节。本次没有阅读这些脚本正文。
- 构建生成的测试 CA 在 `/app/certs/`：`ca.crt` mode `0644`，`ca.key` mode `0600`、227 字节、文件头为 EC 私钥，`ca.p12` mode `0600`。私钥和证书正文不收录。`ca.key` 的 SHA-256 是 `0faca923f0ec51923bfa61abeb4f303de81398227fb1dcc43c0f07f59f976199`。这是 Dockerfile 第 67-82 行在构建时生成并留在镜像里的，所有拉取该 tag 的人拿到的是同一把密钥。

**已核实：没有额外启动项。** 容器启动由 config 的 `ENTRYPOINT` 决定，不是 systemd。磁盘上的 systemd unit 和 cron 来自第 0 层官方 Kali，或来自 apt 层里的软件包（例如 sysstat 的 cron）。工具层没有往 `/etc/cron.d`、`/etc/sudoers.d` 或用户 crontab 追加文件。

镜像 config 的 OCI 标签仍是 Kali 的 `org.opencontainers.image.source=https://gitlab.com/kalilinux/build-scripts/kali-docker`、`revision=4d1ddb93`，另有 `description` 和 `io.buildah.version=1.43.2`。Strix 的 git SHA 不在这些标签里，只在 attestation 里。

### 2.2 capability 与 FUSE

**已核实：仓库里没有 compose、Helm 或 Kubernetes 清单请求这些能力。** `.github/workflows` 里也没有容器 `securityContext`。技能文档里出现的 capability 字样是给模型读的测试说明，不是本进程的运行时配置。

请求发生在 `StrixDockerSandboxClient._create_container`：

1. **默认不会打开的分支。** `strix/runtime/docker_client.py:233-243`：manifest 被 `_manifest_requires_fuse` 判为真时，写入 `devices=["/dev/fuse"]`、`cap_add=["SYS_ADMIN"]`、`security_opt=["apparmor:unconfined"]`。否则若 `_manifest_requires_sys_admin` 为真，只写 `SYS_ADMIN` 和 `apparmor:unconfined`，不写 FUSE 设备。上游 0.19.0 的同名判断在 `agents/sandbox/sandboxes/docker.py:1625-1653`：FUSE 仅当条目是 `Mount` 且模式为 fuse / mountpoint / rclone fuse；`SYS_ADMIN` 的另一支是 rclone nfs 或 S3 files 挂载。`LocalDir` 的基类是 `BaseEntry`，不是 `Mount`（`agents/sandbox/entries/artifacts.py:155`，`entries/mounts/base.py:338`）。
2. **默认 manifest 走不到上述分支。** `strix/runtime/session_manager.py:305-331` 在默认 Docker 后端使用 `entries={}`，只设置环境变量。非 Docker 后端放进去的是 `LocalDir`（`session_manager.py:85-93`）。本仓库没有构造 `Mount`。因此按这份源码，默认创建**不**请求 `SYS_ADMIN`，也**不**传入 FUSE 设备。
3. **默认会打开的 capability。** `docker_client.py:250-257` 每次都把 `NET_ADMIN` 和 `NET_RAW` 追加进 `cap_add`。这里没有 `privileged=True`，也没有 `cap_drop`。Docker 守护进程自己的默认 capability 集合是否还包含别的位，由守护进程决定，这份代码没有改那份默认集合。
4. **设备环境变量默认是空的。** `docker_client.py:69-93` 只在 `STRIX_DOCKER_SANDBOX_DEVICES` 非空时按原路径追加设备。`tests/test_docker_client_devices.py:14-18` 覆盖“未设置则不添加”。具体设备路径属于操作者输入，不在这里复述。
5. **镜像内的文件 capability 不是容器 capability。** Dockerfile 第 64 行给 nmap 包装加了 `cap_net_raw,cap_net_admin,cap_net_bind_service`。它随镜像存在，不依赖上面的 `SYS_ADMIN` 分支。容器最终能不能使用文件 capability，还取决于创建容器时的 capability 集合；创建路径默认包含 `NET_RAW` 和 `NET_ADMIN`。
6. **免密 sudo 是镜像账号策略，不是 `SYS_ADMIN`。** `containers/docker-entrypoint.sh:4-17` 在 `STRIX_HOST_UID` 已设置、且不等于 0、也不等于当前 uid 时走 sudo。`session_manager.py:57-66` 在 Linux 上默认注入宿主机 uid/gid，其他平台不注入。这与原审计第 2.2 节一致，本次没有把它改判成 capability。

全仓库 Python 里，`cap_add`、`SYS_ADMIN`、`/dev/fuse`、`apparmor:unconfined` 的运行时赋值只出现在 `docker_client.py` 这一处。

### 2.3 `openai-agents` 0.19.0 是否被替换或打补丁

**已核实：lock 指向 PyPI 上的原发布件。**

| 来源 | 结果 |
| --- | --- |
| `pyproject.toml:36` | `openai-agents[litellm]>=0.19.0,<0.20` |
| `uv.lock:1458-1474` | 版本 `0.19.0`，`source.registry = https://pypi.org/simple`。没有 git、path 或 patch |
| sdist | URL 与 lock 相同。SHA-256 `1d519d6966834e5c04160caec3a2549190e92ce50cdeb22fac5e17e67b8b98b2`，5,620,718 字节。PyPI JSON、lock、本次下载三者一致。上传时间 2026-07-27T22:49:26Z。未 yanked。`provenance` 为空 |
| wheel | SHA-256 `25392cff993eca7c75b0679ec8d0111faef20c517222c0193111097d0f70db7a`，928,463 字节。同样三方一致。未 yanked |
| 仓库补丁 | `pyproject.toml` 只有 `[tool.uv] exclude-newer`，没有 `[tool.uv.sources]`。仓库不包含 `openai-agents` 的 vendor 树 |

0.19.0 的 wheel `METADATA` 直接依赖是 `griffelib`、`mcp`、`openai`、`pydantic`、`requests`、`typing-extensions`、`websockets`，可选 `litellm`。与 `uv.lock` 该包的 `dependencies` 一致。PyPI 项目 JSON 顶层的 `requires_dist` 属于当前最新版 `0.24.0`，不能拿来代表 0.19.0。

**已核实：三份源码树一致。** 比较 `src/agents`（sdist 与 GitHub tag）和 wheel 内的 `agents/`：各 302 个文件，路径差集为空，内容哈希差集为空。`agents/sandbox/sandboxes/docker.py` 在三处都是 SHA-256 `8f1cc63295eee21b2a78b85f17082586c7be6e4ac4f4504d187e0eb672d2eb35`，63,274 字节。GitHub 归档 `openai-agents-python` tag `v0.19.0` 的归档哈希是 `9867e8f53bf5fb6efa44fc49b39bec0f7ac5c084012166a889b256f1974cb381`；归档哈希不同于 sdist 是因为打包布局不同，`src` 树内容相同。sdist 使用 hatchling，没有 `setup.py`。

**有冲突：注释里的版本和行号，不是依赖包。** `strix/runtime/docker_client.py:19-20` 和 `:205` 写着按 `openai-agents==0.14.6` 的 `docker.py:1434-1477` 复制。lock 和实际发布件是 0.19.0，该方法在 `docker.py:1562-1605`。

对照 0.19.0 原文，安全相关差异是调用方故意留下的，不是一份被调包的依赖：

- 上游 `docker.py:1579-1584` 设置 `entrypoint=["tail"]`、`command=["-f", "/dev/null"]`。本仓库 `docker_client.py:220-224` 不覆盖 entrypoint，`command` 为 `["tail", "-f", "/dev/null"]`。
- 上游 `docker.py:1590-1600` 的 FUSE / `SYS_ADMIN` / `apparmor:unconfined` 分支与 `docker_client.py:233-243` 相同，包括 `if` / `elif`，不会两支同时叠加。
- 上游该方法到 `containers.create` 结束。本仓库在其后追加 `NET_ADMIN`、`NET_RAW`、`host.docker.internal`、可选设备、资源与日志限制、标签和 bind mount（`docker_client.py:250-280`）。
- 在整个 `agents/sandbox` 包内检索，`cap_add` / `SYS_ADMIN` 只出现在上述分支。没有 `cap_drop`、`privileged`、`no-new-privileges` 或 seccomp 设置。因此不存在“0.19.0 新增了限制、这份拷贝把它弄丢了”的情况。拷贝相对 0.19.0 是**多**请求了 `NET_ADMIN`/`NET_RAW`，不是少了一道上游限制。

## 3. 供应链补充

### 3.1 镜像来源与校验

构建 attestation 是 index 里的 in-toto statement（`https://slsa.dev/provenance/v1`），不是带签名的 DSSE 信封。`builder.id` 为空。`buildkit_completeness.resolvedDependencies` 为 `false`：resolved dependencies 里只有基础镜像，没有 `curl | sh`、`go install`、`npm` 或 `git clone`。同一条 invocation `xv17q18h32np8etbpevpjg23m` 覆盖 amd64 与 arm64，开始 `2026-11-04T03:49:37Z`，结束 `2026-11-04T04:13:38Z`。记录的 VCS 是 `https://git-manager.devin.ai/proxy/github.com/usestrix/strix`，revision `4d793375e06650a1ff6fcab0a6bba62b542c8edf`。该 SHA 在 `github.com/usestrix/strix` 上存在，且 Dockerfile blob 与当前仓库相同。语句本身没有可核验的构建者签名，所以“谁推的这张 index”不能靠 attestation 单独证明。

本仓库的 GitHub Actions 没有向 GHCR 推这个 sandbox 的步骤。`scripts/docker.sh` 只在本地打 `strix-sandbox:dev`。发布管线不在这份公开 workflow 里。

Dockerfile 写了校验和、并且本次对上了 GitHub Release 与镜像内文件的：

| 文件 | Dockerfile 引脚 | GitHub Release 文件 SHA-256 | 镜像内文件 SHA-256 |
| --- | --- | --- | --- |
| opengrep `v1.31.0` 的 `opengrep_manylinux_x86` | `containers/Dockerfile:230` | `dbbe3c2058dbe600c449a8ba22ea79a1643a6660658400090d0323a8d0d31147` | `/usr/local/bin/opengrep` 相同 |
| trivy `0.76.1` 的 `trivy_0.76.1_Linux-64bit.tar.gz` | `containers/Dockerfile:196` | `c7cbe3137fb8a48a17574bede3dd6beb16742d1e9772929940fc90082c39d9c0` | 包内 `trivy` 与镜像 `/usr/local/bin/trivy` 都是 `d1dfca6ae10a1ad4540bb94e64f0510077c47008920a9b554986b39dca573d83`（176,029,856 字节） |

trivy 二进制里的 buildinfo 模块版本是 `v0.76.1+dirty`。字节与上游 release 包内的 `trivy` 相同，所以 `+dirty` 是上游构建元数据，不是镜像发布后改过的文件。

### 3.2 镜像里解析出来、但 Dockerfile 没有钉死的版本

这些版本来自层内的 buildinfo、dpkg status、dist-info 目录名或 npm `package.json`。除上表两枚文件外，没有再对照上游字节。

| 组件 | 镜像中的版本证据 | Dockerfile |
| --- | --- | --- |
| uv / uvx | receipt `0.13.7` | `curl \| sh`，未钉版本 |
| httpx | 模块 `github.com/projectdiscovery/httpx` `v1.12.0`，`go1.27.3` | `go install @latest` |
| katana | `v1.8.0` | 同上 |
| interactsh-client | `github.com/projectdiscovery/interactsh` `v1.4.1` | 同上 |
| vulnx | `github.com/projectdiscovery/cvemap` `v1.0.0` | 同上 |
| gospider | 模块行 `github.com/jaeles-project/gospider` `v1.1.6` | 同上 |
| govulncheck | `golang.org/x/vuln` `v1.9.0` | 同上 |
| nuclei | dpkg `3.11.1-0kali1` | Kali 滚动源 |
| nmap | dpkg `7.991+dfsg-1kali1` | 同上 |
| wapiti | dpkg `3.2.10+dfsg-4` | 同上 |
| sudo | dpkg `1.9.17p2-9` | 同上 |
| chromium | dpkg `156.0.8078.47-1` | 同上 |
| golang-go | dpkg `2:1.27~1` | 同上，与 Go 工具的 `go1.27.3` 编译器字符串同代 |
| python3 | dpkg `3.14.7-3` | 同上 |
| agent-browser | npm `0.26.0` | 钉了 `0.26.0` |
| @ast-grep/cli | `0.51.1` | `@latest` |
| retire | `6.1.0` | `@latest` |
| eslint | `10.14.0` | `@latest` |
| js-beautify | `2.0.3` | `@latest` |
| tree-sitter-cli | `0.27.1` | `@latest` |
| jshint | `2.13.6` | 无版本 |
| arjun | dist-info `arjun-2.2.7` | `pipx install` 未钉版本 |
| dirsearch | `dirsearch-0.5.0` | 同上 |
| wafw00f | `wafw00f-2.4.2` | 同上 |
| bandit | `bandit-1.9.4` | 同上 |

`git clone` 当时解析到的提交（镜像内 `.git` 日志，且 GitHub commit API 能读到）：

| 仓库 | 提交 | 该提交的提交时间 |
| --- | --- | --- |
| `aravind0x7/JS-Snooper` | `932704cbce1a04ebe89a558f395a6e474e01b15b` | 2024-05-13 |
| `xchopath/jsniper.sh` | `65d0b508dc88e6de3dfe6ac60e6d89eca22be686` | 2024-01-18 |
| `ticarpi/jwt_tool` | `3bc7407cf2222d6a821dcc19c776e5a1b1cb9a9b` | 2025-05-01 |
| `tree-sitter/tree-sitter-bash` | `a06c2e4415e9bc0346c6b86d401879ffb44058f7` | 2025-12-02 |
| `tree-sitter/tree-sitter-go` | `2346a3ab1bb3857b48b29d779a1ef9799a248cd7` | 2025-09-15 |
| `tree-sitter/tree-sitter-java` | `e10607b45ff745f5f876bfa3e94fbcc6b44bdc11` | 2025-09-15 |
| `tree-sitter/tree-sitter-javascript` | `58404d8cf191d69f2674a8fd507bd5776f46cb11` | 2025-09-15 |
| `tree-sitter/tree-sitter-json` | `254c42a6476413b776221e03982ac8ae159eeb72` | 2026-08-17 |
| `tree-sitter/tree-sitter-python` | `26855eabccb19c6abf499fbc5b8dc7cc9ab8bc64` | 2025-09-15 |
| `tree-sitter/tree-sitter-typescript` | `75b3874edb2dc714fb1fd77a32013d0f8699989f` | 2025-01-30 |
| `tree-sitter-grammars/tree-sitter-yaml` | `a1c4812a73ec5e089de8e441fdea3a921e8d5079` | 2026-05-22 |

gitleaks 二进制里同时有 `v8.0.0`、`v8.25.0`、`8.30.1` 等字符串，没有单独的模块版本行，不能据此指定发布版本。trufflehog 按参数应是 `v3.99.2`，二进制字符串检索没有得到这个版本号。

### 3.3 OSV / GHSA / 恶意包

查询时间 2026-11-09。`POST https://api.osv.dev/v1/query` 与 `/v1/querybatch`。下面每个“0”都是该精确版本没有返回漏洞 id，包括没有 `MAL-*`。

- `PyPI openai-agents@0.19.0`：0。GitHub `advisories?ecosystem=pip&affects=openai-agents` 返回 0 条。
- 上表 npm 七个精确版本、PyPI `arjun@2.2.7`、`dirsearch@0.5.0`、`wafw00f@2.4.2`、`bandit@1.9.4`：0。
- Go 模块 `httpx@1.12.0`、`katana@1.8.0`、`interactsh@1.4.1`、`cvemap@1.0.0`、`gospider@1.1.6`、`golang.org/x/vuln@1.9.0`、`github.com/aquasecurity/trivy@0.76.1`：0。

这只覆盖上述坐标。Kali 包、gitleaks、trufflehog、caido-cli、uv 二进制没有做版本级 OSV 查询：要么版本没有钉死到可查询的发行版，要么没有确认到精确版本。原审计对 `uv.lock` 闭包的 GHSA 列表仍然有效，本次没有重查那 120 个 lock 条目。

`openai-agents` 0.19.0 的 PyPI 文件没有 provenance 字段。来源可信度来自：lock 哈希、下载哈希、官方项目 URL `https://github.com/openai/openai-agents-python`、以及 tag `v0.19.0` 的 `src` 与 sdist 一致。没有签名级 provenance。

## 4. 未确认项与原因

- gitleaks 的精确发布版本。二进制里有多个 8.x 字符串，没有可单独采信的模块版本；按规则没有执行 `--version`。
- trufflehog 镜像内二进制是否就是 `v3.99.2`。Dockerfile 传入了这个参数，字符串检索没有印证。
- `caido-cli` 0.56.0 与 `caido.download` 上的发布字节是否相同。Dockerfile 第 295 行没有校验和，本次没有下载该文件。
- uv `0.13.7` 与 astral-sh 发布字节是否相同。只读了镜像内 receipt，没有再下安装包。
- arm64 的 apt 层和 Go 二进制没有整层解开。已核对的是：arm64 第 0 层 digest 等于官方 Kali arm64；history 命令除基础 `ADD` id 外与 amd64 相同；入口脚本、`caido_api.py`、规则、profile、passwd 层的文件哈希与 amd64 相同。架构相关的二进制本来就不应相同。
- attestation 没有构建者签名，`builder.id` 为空。不能单靠它证明推送者身份。
- 三个个人仓库和 tree-sitter 克隆的脚本正文。只确认了提交存在，没有做代码审查。
- nuclei 在容器运行时是否还会下载模板。镜像里的模板目录是空的；运行时行为需要执行程序才能看，本次不做。
- Docker 守护进程默认 capability 的具体集合。本仓库没有在代码里枚举或删减它。

## 5. 方法与读取过的 URL

读过的本地路径：`containers/Dockerfile`、`containers/docker-entrypoint.sh`、`containers/opengrep-rules/`、`strix/runtime/docker_client.py`、`strix/runtime/session_manager.py`、`strix/runtime/backends.py`、`strix/config/settings.py`、`scripts/docker.sh`、`scripts/install.sh`、`pyproject.toml`、`uv.lock`、`tests/test_docker_client_devices.py`，以及分支 `cursor/security-audit-16c0` 上的 `SECURITY_AUDIT.md`。

镜像层只做了 `tar` 列表、文本读取和哈希。没有导入 Python 包，没有运行任何 ELF 或脚本。

读取过的 URL：

- `https://ghcr.io/token?service=ghcr.io&scope=repository:usestrix/strix-sandbox:pull`
- `https://ghcr.io/v2/usestrix/strix-sandbox/manifests/1.4.0`
- `https://ghcr.io/v2/usestrix/strix-sandbox/manifests/latest`
- `https://ghcr.io/v2/usestrix/strix-sandbox/manifests/sha256:9ee5a4faf67907f3a9880adeacb7b8395b3f1e7cae56019dcd5839d264c79d08`
- `https://ghcr.io/v2/usestrix/strix-sandbox/manifests/sha256:268023698313c0e6ff7d47aaa3e915f09a8c21003cccc0da1f7fdfec0b701204`
- `https://ghcr.io/v2/usestrix/strix-sandbox/manifests/sha256:7ea2612033697208ee41b790f5b7e9a02d0449a459c940a6344fc914292ebda9`
- `https://ghcr.io/v2/usestrix/strix-sandbox/manifests/sha256:80bb7a8b4e83c30b99e8a7d5f3276257d6936fef96e651785b6aedfb6022c171`
- 上述 manifest 指向的 config blob 与层 blob（`https://ghcr.io/v2/usestrix/strix-sandbox/blobs/sha256:…`）。amd64 41 层全部下载；arm64 下载了第 2、15、27、36、37、38、39 层
- `https://ghcr.io/v2/usestrix/strix-sandbox/tags/list`
- `https://auth.docker.io/token?service=registry.docker.io&scope=repository:kalilinux/kali-rolling:pull`
- `https://registry-1.docker.io/v2/kalilinux/kali-rolling/manifests/sha256:7997e0581c4f5c185cfe6773d5cae8fcc7c6b79798726ccaabf843af4bcd2013`
- `https://registry-1.docker.io/v2/kalilinux/kali-rolling/manifests/sha256:73915d5d54833d498f52fd200960126be4a2dbdcc110d75d70839c4ef155ed03`
- `https://registry-1.docker.io/v2/kalilinux/kali-rolling/manifests/sha256:5032046cc734e69016adb615ce96f52d2b5304f0b9cf1583bd2e849602da894d`
- 这两个平台 manifest 的 config blob
- `https://pypi.org/pypi/openai-agents/json`
- `https://files.pythonhosted.org/packages/ba/6c/8fa83cb23d2fe864b284cb45acf895d72a2f6e9827cc684dd4ef0d02d414/openai_agents-0.19.0.tar.gz`
- `https://files.pythonhosted.org/packages/3b/3a/bac1aa3405c0f11b4334ac999e881d08fa40fad1f3b7229a1ca222ada489/openai_agents-0.19.0-py3-none-any.whl`
- `https://github.com/openai/openai-agents-python/archive/refs/tags/v0.19.0.tar.gz`
- `https://github.com/opengrep/opengrep/releases/download/v1.31.0/opengrep_manylinux_x86`
- `https://github.com/aquasecurity/trivy/releases/download/v0.76.1/trivy_0.76.1_Linux-64bit.tar.gz`
- `https://api.osv.dev/v1/query`
- `https://api.osv.dev/v1/querybatch`
- GitHub API：`repos/usestrix/strix/commits/4d793375e06650a1ff6fcab0a6bba62b542c8edf`、`repos/usestrix/strix/compare/4d793375e06650a1ff6fcab0a6bba62b542c8edf...215482eb2204cd30ac6b87520e3f5c6511e2e2e8`、`advisories?ecosystem=pip&affects=openai-agents`，以及第 3.2 节列出的克隆提交
