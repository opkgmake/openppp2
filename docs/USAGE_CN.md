# PPP PRIVATE NETWORK™ 使用教程

> 本指南基于仓库默认分支，适用于希望从源码构建并部署 PPP PRIVATE NETWORK™ 的管理员和高级用户。文档覆盖 Linux、Windows 以及多架构交叉编译场景，并结合 `appsettings.json` 的默认配置说明客户端/服务器模式下的关键参数。

## 1. 环境准备

### 1.1 支持的平台

| 平台 | 支持的架构 | 备注 |
|------|------------|------|
| Linux | `amd64`, `aarch64`, `armv7l`, `mipsel`, `ppc64el`, `riscv64`, `s390x` | 通过 `build-openppp2-for-linux-using-ubuntu-latest-cross.sh` 提供的交叉编译流程一次性构建多个架构。|
| Windows | `amd64` | 使用 Visual Studio 解决方案 `ppp.sln`。|
| macOS | `amd64` | 由 CMake 直接构建，注意需要本地安装 Boost/OpenSSL/jemalloc。|
| Android | `arm64-v8a` 等 | 参见 `android/` 目录下的 Gradle 项目。|

### 1.2 第三方依赖

项目使用自建的第三方库目录 `THIRD_PARTY_LIBRARY_DIR`，默认值为 `/root/dev`。目录结构需包含：

```
THIRD_PARTY_LIBRARY_DIR/
├── boost/
│   └── stage/lib/libboost_*.a
├── jemalloc/
│   └── lib/libjemalloc.a
└── openssl/
    ├── include/
    └── libssl.a, libcrypto.a
```

> **提示**：如需启用额外功能，可按需引入 curl，并在 `CMakeLists.txt` 中取消相关注释。【F:CMakeLists.txt†L79-L122】【F:CMakeLists.txt†L164-L186】

构建第三方库时建议统一开启静态产物编译选项，便于生成完全静态的 Linux 可执行文件。

## 2. Linux 构建流程

### 2.1 自动化脚本（推荐）

仓库提供了完整的交叉编译脚本 `build-openppp2-for-linux-using-ubuntu-latest-cross.sh`，它将：

1. 安装所需的交叉编译工具链（`gcc-aarch64-linux-gnu` 等）。
2. 为每个架构设置编译器和链接器。
3. 调用 `cmake` + `make -j` 构建，并在 `bin/` 目录下生成可执行文件及压缩包。

执行方式：

```bash
sudo ./build-openppp2-for-linux-using-ubuntu-latest-cross.sh /path/to/third-party-libs
```

未传入第三方库路径时默认查找 `/root/dev`。脚本会在每次构建前清空 `build/`，生成 `openppp2-linux-${ARCH}.zip`，其中包含静态链接的 `ppp` 可执行文件。【F:build-openppp2-for-linux-using-ubuntu-latest-cross.sh†L1-L88】

### 2.2 手工构建

若仅需编译单一架构，可手动执行：

```bash
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release \
         -DTHIRD_PARTY_LIBRARY_DIR=/path/to/third-party
make -j$(nproc)
```

生成的二进制位于 `bin/ppp`，使用 `strip` 可进一步减小体积。请确保在 `cmake` 前导出 `THIRD_PARTY_LIBRARY_DIR`，或通过 `-D` 参数覆盖默认值。

## 3. Windows 与 macOS 构建

### 3.1 Windows

1. 使用 Visual Studio 2022 打开 `ppp.sln`。
2. 安装并配置 vcpkg 或本地第三方库，使其路径与 `THIRD_PARTY_LIBRARY_DIR` 保持一致。
3. 选择 `Release|x64` 配置，编译 `ppp` 项目，生成的可执行文件位于 `windows/bin/` 或解决方案指定的输出目录。

### 3.2 macOS

1. 安装 Xcode Command Line Tools 以及 Homebrew。
2. 通过 Homebrew 构建 Boost、OpenSSL、jemalloc，并创建 `THIRD_PARTY_LIBRARY_DIR` 指向这些静态库。
3. 执行与 Linux 类似的 CMake 构建流程。macOS 目标在链接阶段启用了平台特定的系统库（`pthread`、`dl` 等），详见 CMake 配置。【F:CMakeLists.txt†L188-L251】

## 4. 运行模式与启动命令

可执行文件 `ppp` 支持客户端与服务器两种模式，通过 `--mode` 指定，默认值为 `server`。常见参数可参考 `README_CN.md` 中的命令行表格，以下给出最小化示例：

### 4.1 服务器模式

```bash
./ppp --mode=server --config ./appsettings.json \
      --firewall-rules ./firewall-rules.txt --dns-rules ./dns-rules.txt
```

关键配置项说明：

- `server.listen`：`appsettings.json` 中 `tcp.listen.port`/`udp.listen.port` 定义监听端口。【F:appsettings.json†L24-L68】
- `server.backend`：Webhook 回调地址，可用于和业务系统集成。【F:appsettings.json†L105-L115】
- `server.subnet`、`server.mapping`：控制是否开放子网及端口映射能力。

### 4.2 客户端模式

```bash
./ppp --mode=client --config ./appsettings.json \
      --dns 1.1.1.1,8.8.8.8 --tun ppp --tun-ip 10.0.0.2
```

重要字段：

- `client.server`：远程服务器地址（`ppp://host:port/`）。【F:appsettings.json†L116-L166】
- `client.bandwidth`：期望带宽（单位 Mbps），用于自适应拥塞控制。
- `client.routes`：自定义分流规则，可搭配 `--vbgp` 或 `--bypass-iplist` 参数使用。

若需要启用本地代理服务，可在配置中打开 `http-proxy` 与 `socks-proxy`，程序启动后分别监听 `port` 指定的端口。【F:appsettings.json†L129-L151】

## 5. 常见运维操作

### 5.1 日志与诊断

- 默认日志输出路径：`server.log` -> `./ppp.log`。
- 使用 `--auto-restart` 结合 Systemd、Supervisor 等守护程序可实现自动拉起。
- 通过 `--link-restart` 设置链路异常时的重连次数，适用于不稳定网络场景。更多命令详见 `README_CN.md` 的“高级功能”和“路由设置”章节。

### 5.2 配置热更新

`appsettings.json` 支持热加载，修改后可向进程发送 `SIGUSR1`（Linux）或使用内置控制命令触发刷新。若需完全重启，使用 `--auto-restart` 或外部守护进程。

### 5.3 安全建议

1. 为服务器端启用 TLS (`websocket.ssl`) 并替换默认证书文件。【F:appsettings.json†L69-L104】
2. 更新 `key.protocol-key`、`key.transport-key` 等敏感字段，避免使用仓库示例值。
3. 配置防火墙规则文件并结合 `--firewall-rules` 参数加载。

## 6. 目录速览

- `ppp/`：核心库与协议栈实现。
- `common/`：跨平台公共组件，包括 `lwip`、`dnslib`、`aggligator` 等。
- `linux/` / `windows/` / `darwin/`：平台相关实现。
- `docs/TRANSMISSION*.md`：传输协议设计文档。

## 7. 常见问题 FAQ

**Q1：编译失败提示缺少 `libjemalloc.a`？**
> 检查 `THIRD_PARTY_LIBRARY_DIR/jemalloc/lib/` 是否存在静态库，并确认 CMake 运行时已正确设置 `THIRD_PARTY_LIBRARY_DIR` 环境变量或 `-D` 参数。【F:CMakeLists.txt†L136-L186】

**Q2：客户端无法连接服务器？**
> 确认 `client.server` 地址可达，并检查 `udp.listen.port`、`tcp.listen.port` 与防火墙设置是否一致，同时查看 `ppp.log` 中的错误详情。【F:appsettings.json†L24-L166】

**Q3：如何快速导入国家 IP 列表？**
> 使用 `--pull-iplist ./ip.txt/CN` 自动拉取最新列表，或在 `client.routes` 中配置自定义下载地址（如 `https://ispip.clang.cn/cmcc.txt`）。【F:README_CN.md†L45-L107】【F:appsettings.json†L152-L166】

---

如需英文版教程，请参考仓库后续更新或自行翻译本指南。
