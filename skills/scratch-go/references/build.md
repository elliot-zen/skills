# Go 版本与容器构建

## 1. 初始化与版本选择

初始化项目时询问：“Docker 构建使用哪个 Go 版本？默认选项为 `1.27.3`，也可指定其他完整版本。”等待用户选择；用户已明确指定时直接采用，不重复询问。默认选项仅作为建议，不因本机安装了某个版本或某个版本更新而静默替换。

区分以下三个值，不要求相等：

| 位置 | 含义与约定 |
| --- | --- |
| `go.mod` 的 `go` 指令 | 模块最低 Go 版本，按实际语言特性和依赖要求设置；不机械复制本机或 Docker 的补丁版本 |
| 本地工具链 | 沿用开发者环境及其 `GOTOOLCHAIN` 设置，实际使用的工具链需满足模块要求；不为匹配 Docker 自动修改全局环境或强制安装同版本 |
| Dockerfile 构建阶段 | 使用用户确认的完整 Go 版本，固定镜像标签；设置 `GOTOOLCHAIN=local`，不在构建时自动切换工具链 |

Docker 版本也必须满足模块要求。例如 `go.mod` 为 `go 1.27.0` 时，本地使用 `1.27.4`、Docker 使用 `1.27.3` 是允许的。不要为追求版本一致新增 `toolchain` 指令；已有指令按项目约定保留。版本或依赖不兼容时说明具体冲突，再调整相关选择，不静默抬高模块最低版本。

README 记录模块最低版本、Docker 构建版本、本地版本可不同及构建命令。后续开发保留项目已有版本，版本升级需属于当前任务范围。

## 2. Dockerfile 约定

初始化时在项目根目录生成 [Dockerfile](../assets/Dockerfile) 和 [.dockerignore](../assets/.dockerignore)，按服务名称、用户确认的版本和实际运行文件调整模板，已有项目只合并所需内容。

- 构建镜像固定完整 Go 版本及显式 Alpine 版本标签，模板使用 `golang:1.27.3-alpine3.24`。镜像仓库遵循用户或项目配置，本技能不指定统一仓库。用户选择其他 Go 版本时仍明确 Alpine 版本，不能由无后缀或仅 `-alpine` 的标签推断基础系统版本。写入前通过 `docker manifest inspect <image>:<tag>` 确认实际使用仓库中的远端标签和目标架构存在；不得使用 `latest`、仅主次 Go 版本、虚构标签或用 digest 冒充版本。标签不可用时说明结果，等待用户选择可用版本或镜像，不静默回退。
- 采用多阶段构建。构建阶段先复制模块文件并下载依赖，再复制源码；`go.sum` 存在时必须纳入构建上下文。私有依赖凭据通过构建 secret 或 SSH mount 传入，不写入 Dockerfile、构建参数或镜像层。
- 默认 `CGO_ENABLED=0`，构建 `cmd/<service>`，沿用 Makefile 的 `-trimpath` 和 `main.version` 注入约定。`SERVICE` 和 `VERSION` 是构建参数；生成项目时把 `SERVICE` 默认值改为实际主要服务名。
- Alpine 构建阶段通过 `apk add --no-cache ca-certificates tzdata` 准备证书与时区数据；仅在实际需要时添加 Git、C 编译器等工具。默认使用 `scratch` 运行阶段，只复制二进制、CA 证书、时区数据及实际需要的运行文件。依赖 CGO 或外部动态库时，根据依赖确认 musl/glibc 兼容性，选择兼容且固定版本的构建与运行镜像，并说明原因，不直接套用静态模板。
- 统一使用 `USER 1000:1000`。工作目录为 `/app`，二进制使用 exec 形式的 `ENTRYPOINT`，让程序直接接收信号。需要写入的目录或挂载卷按 UID/GID `1000:1000` 配置权限，不以 root 运行来绕过权限问题。
- 运行阶段复制入库的 `configs/`，保持应用从 `/app/configs/config.yaml` 读取配置。无文件配置的程序移除对应 `COPY`；其他静态运行资源按实际需求复制。环境变量覆盖规则遵循 [Go 工程模式](go-patterns.md)。
- `.dockerignore` 排除 Git 元数据、本地环境文件、工具缓存和构建产物；实际 `.env` 与凭据不得进入构建上下文或最终镜像。不排除构建及验证实际需要的源码、模块文件、测试夹具或运行资源。

在项目根目录构建，例如：

```sh
docker build --build-arg SERVICE=orders-api --build-arg VERSION=v1.2.3 -t orders-api:v1.2.3 .
```

Go 构建版本在 Dockerfile 的 `FROM` 中独立维护，不从本机 `go version` 或 `go.mod` 自动推导。不将 Docker 使用的 Go 版本与应用的 `VERSION` 混用。目标架构由 Docker 构建平台选择，不硬编码宿主机架构。

## 3. 验证

- 检查模块要求与容器构建工具链兼容，并实际构建镜像，确认依赖、目标入口、配置和运行资源完整。
- 运行最终镜像验证实际入口、配置读取及适用的 HTTPS、时区和可写目录行为，确认 UID/GID 为 `1000:1000`；服务还需验证监听和 SIGTERM 正常关闭。
- 本地与 Docker 工具链不同时，对有版本兼容影响的代码在 Docker 所选版本下运行相关测试，不能仅以本地通过推断容器可用。
- 无法访问镜像仓库、构建或运行环境时，明确报告未验证项和原因，不把模板检查写成镜像构建成功。

版本语义参考 [Go Toolchains](https://go.dev/doc/toolchain)，多阶段构建参考 [Docker 官方文档](https://docs.docker.com/build/building/multi-stage/)。
