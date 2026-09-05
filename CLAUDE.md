# 开发指引

这个仓库只做一件事：把凝思安全操作系统 V6.0.100 的官方安装介质变成可用于软件构建与测试的容器镜像。它不是凝思系统的替代品，回答的是「编出来的东西对不对」而不是「跑起来的系统对不对」。定位边界见 README 的前两节。

## 硬性约定

- commit **不允许带 co-author**
- 文档一律中文；Markdown **自然段内不换行**，一段写成一行长句
- 不在仓库里讨论许可与法务
- 写进文档的版本号一律来自跑镜像实测，不取源索引里的元包版本

## 这个仓库放什么

```
linx/
├── buildkit/                     # submodule，钉住一个 commit
├── distros/v6.conf               # 一个版本一个，文件名即 DID
├── .github/workflows/build.yml   # 只定义矩阵、调用 buildkit 的可复用 workflow
├── README.md                     # 面向使用者
├── CLAUDE.md                     # 本文件
└── AGENTS.md                     # 与本文件同步
```

判断改动落在哪边只有一条：**只跟凝思自己的事实有关**（数据镜像指纹、种子包、ABI 基线、这一版特有的怪癖）就进 `distros/v6.conf`；**跟怎么构建、怎么测、怎么发有关**就进 buildkit。第二类占绝大多数。

## 必知事实

- **取材走数据镜像，不走厂商站。** 凝思下载站对 GitHub runner 全量 135 秒超时（本机直连 0.07 秒 200），介质由本机从官方 ISO 切出后推 `ghcr.io/distrotwin/scratch`，判据与探测记录见 `buildkit/docs/downstream-repo.md`、机制见 `buildkit/docs/srcdata.md`
- **`SRCDATA_MANIFEST_SHA256` 必须钉。** 不钉的话数据镜像被换掉不会被任何检查发现；改介质必须重推数据镜像并同步这个指纹
- **METHOD 必须是 selfhost。** 凝思的 `linx-noroot-conf` 装三权分立管理钩子（`exec-after-dpkg.sh`），普通 mmdebstrap 构建会被它挂死（实测钩子进程 13 分钟不退出），selfhost 用 `PIN_NEVER` 从 debootstrap 入口拦掉
- **介质仓库无签名**（`NO_CHECK_GPG=yes`），完整性由 ISO 官方校验和 + manifest 两道门禁兜，README 里不能把这说成验签
- **单架构。** 厂商下载站只放 x86_64 的 ISO，没有 arm/loong 介质可取

## 改动到跑通的完整回路

改 conf → 本地跑一个档位（**别拿 CI 当实验台**）→ 推仓库跑 `publish=false` 的完整 CI → 看报告 artifact 确认零异常 → `publish=true` 发一轮 → 匿名视角验收 registry → 把 README 的数字对齐到这一轮实际结果。
