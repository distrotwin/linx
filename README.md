# 凝思安全操作系统 · 构建与测试镜像

对着凝思 V6.0.100 官方安装介质自举出来的容器环境，用于**软件构建、打包与兼容性测试**。Debian 10 系、glibc 2.28 档 ABI，x86_64 单架构、三个档位，公开在 GHCR。最近一轮 3 个镜像、129 项检查全部通过，零异常。

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

| tag | 里面有什么 | 适合 |
|---|---|---|
| `v6-micro` | 能跑 shell 的最小根系统，无 apt | 跑编好的二进制、当测试底座 |
| `v6-base` | micro + apt/python3/systemd/常用工具 | 一般兼容性验证 |
| `v6-devel` | base + gcc/g++/make/dpkg-dev | 编译、打 deb |

`latest` 指向 `v6-devel`。基线（跑镜像实测）：glibc 2.28、libstdc++.so.6.0.25（GLIBCXX ≤ 3.4.25）、gcc 8.3.0。

## 镜像是怎么造的

凝思只有 ISO 一条公开获取路径，而它的下载站对 GitHub 托管 runner 不可达（本机直连 0.07 秒 200，runner 135 秒超时；判据与探测记录见 buildkit 的 `docs/downstream-repo.md`）。所以取材走**数据镜像**：在能连通厂商站的机器上校验官方 ISO（md5 + sha256 逐位一致）、拷出介质仓库、算依赖闭包把 4.3 GB 裁到 190 MB，推成 `ghcr.io/distrotwin/scratch:linxos-6.0.100-20230822-x86_64`；CI 构建时取回，先对 conf 里钉死的 manifest 指纹、再逐文件核对后才使用。构建走两阶段 debootstrap（与银河麒麟同路），用介质自己的 dpkg 完成自举。

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
