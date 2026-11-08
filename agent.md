# AI 实施指南：epyc7002 / DSM 7.4

## 任务与交付范围

依据本分支 README.md，在独立工作目录获取公开上游并实现 Linux 5.10 原生 WireGuard SPK 构建。此文件名为 agent.md；若工具不会自动读取该名称，必须显式将本文件作为输入。

本分支只保存 README.md、agent.md 和 docs/images/ 中经过隐私处理的构建效果图。不得加入源码、脚本、镜像、SPK、KO、压缩包、下载缓存或原始运行日志。实施代码与产物保存在仓库以外。公开文档中只描述可复现步骤与修改，不写真实主机、代理、账号、密码、个人目录、私钥和设备唯一标识。效果图仅保留说明兼容性所需的平台、内核与运行状态；网络地址、公钥标识等使用不透明遮挡，并去除元数据。主分支仅保存 README.md，构建原因和安全免责声明以 main 分支 README 为准，不得因构建或握手成功声称协议、源码或产物安全。

本分支另包含 LICENSE，许可证为 GPL-3.0-only，仅覆盖作者有权授权的原创贡献。第三方组件遵循各自许可证；不得将仓库许可证视为上游项目的重新授权。主分支保存 README.md 和 LICENSE。更新文档时保留 GPL 赋予接收者的权利，免责声明不得附加与 GPL 冲突的限制。

## 1. 获取与检查

1. 从 README.md 指定上游获取原始套件工程、DSM7.4 构建框架、epyc7002 工具链和群晖 Linux 5.10 源码。
2. 记录实际使用的提交、版本和下载文件校验值；不自动追随最新版并假定兼容。
3. 检查目标内核 VERSION、PATCHLEVEL、.config、生成头文件、Module.symvers、工具链编译器及实际 NAS 内核。
4. 已知工具链 7.4-90074 与 GPL 源码 7.4-90080 存在差异，必须明确记录。不同构建号需验证，不声称完全一致。
5. 若只有工具链头文件且 drivers/net/wireguard 只有 Makefile，应使用完整 GPL 源码，不覆盖原工具链目录。

## 2. Dockerfile 与网络

安装 git、python3、wget、ca-certificates、xz-utils、bzip2、lbzip2 以及实际构建所需工具。使用明确、可复现的基础镜像版本。

构建代理通过 build args，运行代理通过环境变量传入；不硬编码真实代理。分别测试代理端口、HTTPS 请求、Git、Python urllib 和工具链文件读取。urllib 是 pkgscripts-ng 工具链下载所用路径之一。

保留工具链压缩包、构建框架、完整构建环境和产物的持久化挂载；内核源码可只读挂载。不要依赖容器退出后仍保留未挂载文件。

## 3. 修改 build.sh

- 删除 wireguard-linux-compat 的版本获取、源码下载、旧补丁开关与编译路径。
- DSM 7.4 明确选择 DSM7.4 分支，不能将所有 7.x 映射到 DSM7.0。
- 更新缓存仓库时检查已有分支及本地修改；不要强制覆盖使用者修改。避免补丁导致后续 git 更新失败，可采用独立工作副本或可重复应用的最小补丁。
- libmnl 使用明确版本（已有验证为 1.0.5），支持显式覆盖；wireguard-tools 记录固定版本（已有验证为 1.0.20260223），对空版本与下载失败立即报错。
- 在外层容器下载 libmnl 和 wireguard-tools，使用临时文件和压缩包完整性检查，成功后原子改名。不要要求 chroot 内具备 wget。
- 修复 include/python/BuildEnv.py 中 cp -al 的 src/tgt 路径，以及 include/python/link_project.py 中 cp -al 的 source/dest 路径。跨文件系统使用普通复制；补丁需幂等并检查匹配结果。
- 默认从 /kernel/linux-source/linux-5.10.x 读取源码，允许 KERNEL_SOURCE 覆盖。验证源码与工具链均为 5.10。
- 从源码复制 drivers/net/wireguard 至工程 native-wireguard，复制 lib/crypto 子目录；避免在只读源码上生成对象。
- 在 native-wireguard/Makefile 中加入 obj-m += crypto/，确保重复运行不会不断追加或遗留过期对象。
- 所有失败返回非零，最后保留 PkgCreate 原始返回码。安装日志缺失时不得覆盖首个失败原因。
- 核心变更涉及多个文件，不能只修改 build.sh 就宣布完成。

## 4. 修改 Makefile 与构建阶段

模块目标为 native-wireguard/wireguard.ko。使用群晖已准备好的内核构建目录和 Module.symvers：

```text
make -C <KSRC> M=<工程绝对路径>/native-wireguard ARCH=x86_64 CROSS_COMPILE=<群晖编译器前缀> \
  CONFIG_WIREGUARD=m \
  CONFIG_CRYPTO_LIB_BLAKE2S=m CONFIG_CRYPTO_LIB_BLAKE2S_GENERIC=m \
  CONFIG_CRYPTO_LIB_CHACHA_GENERIC=m CONFIG_CRYPTO_LIB_POLY1305_GENERIC=m \
  CONFIG_CRYPTO_LIB_CURVE25519=m CONFIG_CRYPTO_LIB_CURVE25519_GENERIC=m \
  CONFIG_CRYPTO_LIB_CHACHA20POLY1305=m modules
```

这些参数只用于本次外部模块构建，不代表 NAS 内核配置已被修改。检查目标配置和 Module.symvers，避免导出重复符号或缺失依赖。不要通过关闭符号检查、忽略 modpost 错误来生成包。

clean 保留源码压缩包，清理解压后的用户态源码和原生构建对象，但不要删除准备好的原生源目录。SynoBuildConf/build 调用 clean 时也传 KSRC、CROSS_COMPILE、ARCH，并在失败时停止。

安装阶段可能没有 KSRC。不要把模块目标设成每次 install 都必须执行的 FORCE 目标；安装应复制构建阶段生成的文件，并检查产物完整性。SynoBuildConf/install 同样遇错立即停止，避免生成空 SPK。

继续构建 wg、wg-quick。安装包至少包含：

```text
wireguard/wg
wireguard/wg-quick
wireguard/wg-autostart
wireguard/wireguard.ko
wireguard/libblake2s-generic.ko
wireguard/libblake2s.ko
wireguard/libchacha.ko
wireguard/libpoly1305.ko
wireguard/libcurve25519-generic.ko
wireguard/libcurve25519.ko
wireguard/libchacha20poly1305.ko
```

内核原生 WireGuard 的版本与用户态工具版本分开处理。已有包名使用 1.0.0；不要将 compat 发布日期冒充原生模块版本。

## 5. 启动与权限

检查 scripts/start-stop-status：

- 先确保系统网络依赖 ipv6、udp_tunnel、ip6_udp_tunnel 可用。
- 按依赖顺序加载 libblake2s-generic、libblake2s、libchacha、libpoly1305、libcurve25519-generic、libcurve25519、libchacha20poly1305，最后 wireguard。
- 通过 /proc/modules 或 /sys/module 检查已加载模块，处理模块名中横线与下划线，兼顾内建依赖。
- 任一加载失败立即返回非零，并留下足以定位的错误。
- 停止时优先只卸载 WireGuard；不要随意卸载其他服务共享的依赖或修改全局路由、转发状态。

DSM 7 默认 package 身份无法加载模块。核实上游 scripts/start 的 root 启动入口并在 README 写清运行方法。不得把“root 安装”误写成“自动 root 运行”，也不得通过放宽系统文件权限修复。若变更默认运行身份，应明确说明实际实现和安装测试结果。

## 6. 验证与完成标准

1. Bash 语法检查、Python 补丁后的语法检查、Makefile 规则检查和 git diff --check。
2. 重建 Docker 镜像，执行完整构建，退出码为 0；工具链报告 0 failed。
3. 解开 SPK 和 package.tgz，检查上述文件全部存在且非空，检查启动脚本及运行身份配置。
4. 使用 modinfo 检查 vermagic 和 depends，检查实际模块依赖；记录构建版本差异。
5. 在获授权的 NAS 上安装并测试启动、临时 WireGuard 接口、握手及通信。无法访问 NAS 时明确区分“构建通过”与“运行验证完成”。
6. 更新本分支公开文档，只记录非私有结果。向使用者交付独立工作目录中的产物路径及后续命令，不将产物加入本文档仓库。
7. 提交前检查文档、效果图元数据与全部 Git 历史，不含凭据、真实网络地址、个人路径、设备唯一信息或源码。仅保留说明运行效果所需的通用平台、内核和运行状态；不要声称保留运行信息的图片完全匿名。主分支继续只包含 README.md。

