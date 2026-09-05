# 凝思安全操作系统 · 构建与测试镜像

对着凝思官方安装介质自举出来的容器环境，用于**软件构建、打包与兼容性测试**。V6.0 是市场伞号，各支底座互不相同——本仓库收六支，构成全 org 跨度最大的 ABI 阶梯（glibc 2.5 → 2.38），公开在 GHCR。

```bash
docker run --rm ghcr.io/distrotwin/linx:v6-devel \
  bash -c 'grep PRETTY /etc/os-release; ldd --version | head -1; gcc -dumpfullversion'
```

## 这是什么，不是什么

镜像里没有内核——这是所有 Linux 容器镜像的常态，容器共享宿主内核。凝思引以为特色的安全机制（三权分立管理、内核加固）大多依赖内核态与管理守护进程，装在镜像里也不生效；`systemd` 有二进制但不是 PID 1。

所以它适合回答「编出来的东西对不对」，不适合回答「跑起来的系统对不对」。

**该用它**：在 CI 里编出能在凝思 V6 上跑的二进制与 deb；验证依赖闭包；检查产物需要的 glibc / libstdc++ 符号版本目标系统能否满足；复现只在这个系统上出现的编译问题。

**别用它**：当生产运行时基础镜像；复现内核相关行为或安全机制行为；当作系统的完整替代品做验收测试。

## 先跑一遍

进容器，写个 A+B，编了跑，再看符号天花板。

```bash
docker run -it --rm ghcr.io/distrotwin/linx:v6-devel /bin/bash
```

```bash
echo '#include <stdio.h>
int main(void){ int a, b; if (scanf("%d %d", &a, &b) != 2) return 1; printf("%d\n", a + b); return 0; }' > ab.c

gcc -O2 -o ab ab.c
echo "3 4" | ./ab
objdump -T ab | grep -oE 'GLIBC_[0-9.]+' | sort -uV | tail -1
```

最后那行是这套镜像最有用的一句：**它直接告诉你产物需要目标系统多新的 glibc**。在 V6.0.100 上答案不该超过 `GLIBC_2.28`。

## 选哪一个

| 前缀 | 支线（介质自述身份） | 底座 | glibc / gcc | 架构 |
|---|---|---|---|---|
| `v6.0.42` | 凝思磐石 Rocky 4.2（自研 pkgutils 包管理） | 自研 | 2.5 / 4.1.2 | amd64 |
| `v6.0.60` | LinxOS 6.0.60 | Debian 6 squeeze | 2.11 / 4.4.5 | amd64 |
| `v6.0.97` | LinxOS 6.0.97 an7 | Anolis 7（EL7 系） | 2.17 / 4.8.5 | amd64 |
| `v6.0.98` | LinxOS 6.0.98 an8 | Anolis 8（EL8 系） | 2.28 / 8.5.0 | amd64+arm64 |
| `v6` = `v6.0.100`（双前缀同物） | Linx GNU/Linux 6.0.100 | Debian 10 buster | 2.28 / 8.3.0 | amd64 |
| `v6.0.99` | LinxOS 6.0.99 el24.03 | openEuler 24.03 | 2.38 / 12.3.1 | amd64+arm64 |

每支三档：`<前缀>-micro`（最小根系统）、`<前缀>-base`（+包管理/python3/常用工具）、`<前缀>-devel`（+gcc/g++/make）。**`v6` 即 6.0.100 主线，且 `v6.0.100-*` 同名 tag 一并发布**（双前缀指向同一镜像）：短名是默认入口，完整版本号与其余支线命名对齐；`latest` 指向 `v6-devel`。6.0.80 不在列：官方站已无该支介质（目录只剩空的 `80-kernel/`）。6.0.99 的 el20.03/el22.03 两支存在但未收，收的是被系统化打补丁的 el24.03（CSAF 472 条）。

龙芯查过、判无：官方站没有任何 loongarch 目录（`an/` 里带「龙」字的是**龙蜥**（Anolis）版手册，不是龙芯），六支介质均无 loong 材料。

磐石 42 的特别之处如实说明：它既非 deb 也非 rpm（自研 `/var/lib/pkg/db`，无依赖字段），镜像按 ELF 依赖闭包从整盘 rootfs 切出，符号天花板实测 `GLIBC_2.2.5`——这是给「必须兼容极老 glibc」的构建需求准备的真实底座。它没有 apt/dnf；`/etc/os-release` 为容器化合成（该世代早于此规范，文件头注明）；介质里 198/213 包无签名的 6.0.99 与整盘 rootfs 的 42/60/100，完整性锚点都是 ISO 官方校验和链（各数据镜像 `.origin` 逐支记录）。

## 镜像是怎么造的

凝思只有 ISO 一条公开获取路径，而它的下载站对 GitHub 托管 runner 不可达（本机直连 0.07 秒 200，runner 135 秒超时；判据与探测记录见 buildkit 的 `docs/downstream-repo.md`）。所以取材一律走**数据镜像**：在能连通厂商站的机器上按官方 md5sum.txt 校验 ISO、按各支形态切出依赖闭包（deb 支切 dists+pool，rpm 支切 Packages+repodata，磐石支按 ELF 闭包切 rootfs），推成 `ghcr.io/distrotwin/scratch` 的介质 tag；CI 构建时取回，先对 conf 里钉死的 manifest 指纹、再逐文件核对后才使用。deb 支走两阶段 debootstrap，rpm 支走 rpmmedia（EL7 的 rpm 走 NSS，chroot 里要补 /dev/urandom；含 shell 变量的 alternatives 不重放——这些都记在 buildkit）。6.0.100 一支要注明：厂商已把该支 ISO 从下载站撤下（新旧构建全部 404，目录只剩 md5sum.txt），本仓库用的是撤下前获取、且与官方 md5 逐位一致的 20230822 介质——它因此成了这支的事实存档。

完整性锚点链：conf 钉 manifest 指纹 → manifest 覆盖介质每个文件 → 介质 `.origin` 记官方 ISO 的 md5/sha256/字节数 → 该校验和来自厂商发布的 `md5sum.txt`。

## 镜像与安装介质的关系

镜像等于从官方 ISO 的包仓库里按依赖闭包装出来的最小系统，不等于装好的凝思桌面。三点差异要知道：`linx-noroot-conf`（三权分立管理钩子）被排除——它的 dpkg 钩子在容器里会挂死构建，而它管理的安全机制在容器里本来就不生效；内核与 initramfs 相关包被排除；桌面组件不在闭包里。除此之外的每个包都来自 ISO 原样，版本与介质一致。

## 认出自己在哪个系统上

```bash
docker run --rm ghcr.io/distrotwin/linx:v6-micro cat /etc/os-release
```

## tag 与钉版

`v6-<tier>` 是活动 tag，指向最近一次发布；`v6-<tier>-<日期>` 是不可变 tag。CI 里建议钉日期 tag 或直接钉 digest。

## 镜像自带的溯源信息

每个镜像的 label 里带着：ISO 地址（`cn.internal.iso-url`）、数据镜像 tag 与 manifest 指纹（`cn.internal.srcdata-image` / `cn.internal.srcdata-manifest-sha256`）、构建仓库 commit 与 run。`docker inspect` 即可读。

## 本地构建

需要能连通厂商站的网络位置才能重造数据镜像（`buildkit/tools/srcdata-make.sh`）；只重建容器镜像的话，任何机器都可以：

```bash
git clone --recursive https://github.com/distrotwin/linx && cd linx
ROOT=$PWD buildkit/tools/srcdata-fetch.sh v6
sudo DID=v6 buildkit/build/build-selfhost.sh micro
```

## CI

手动触发 `构建镜像` workflow；勾选 `publish` 才会推 GHCR。构建、测试、汇总、发布四段全部来自 buildkit 的可复用 workflow，本仓库只有矩阵。

## 仓库结构

```
linx/
├── distros/v6.conf     # 版本定义：数据镜像指纹、种子、基线，全部钉死
├── .github/workflows/  # 只有矩阵，实现都在 buildkit
└── buildkit/           # submodule，构建、测试、发布的全部实现
```
