# fnnas Zero2W — 飞牛 fnOS 定制镜像 for Orange Pi Zero2W

面向 **Orange Pi Zero2W(全志 Allwinner H618)** 的飞牛 fnOS(fnnas)定制系统镜像。
在官方 fnOS 基础上移植 6.18.18 内核 + 全外设支持,并**修复了若干官方自带 bug**,做到开箱可用、系统更新(OTA)可正常升级。

> ⚠️ 非官方个人移植项目,与飞牛官方无关。仅供学习与自用,自行承担风险。

---

## 版本

**V2.5**(本次发布) — 稳定修复版
- 内核:`6.18.18.c798-trim`(arm64-sunxi,全外设 dtb)
- 应用:fnOS `1.1.31`(首启后可一键 OTA 升级到最新)
- 定位:低风险稳版,已验证 OTA / 存储池 / 改名 / web 全链路正常

镜像通过本仓库 **[Releases](../../releases)** 分发(单文件约 1.9GB,故不放在 git 里)。

---

## 硬件支持情况

| 功能 | 状态 |
|---|---|
| WiFi(展锐 uwe5622) | ✅ 驱动 + 固件已集成 |
| HDMI 显示 | ✅ |
| 红外接收 | ✅ |
| 板载按键(lradc) | ✅ |
| USB / USB-C 虚拟网卡 | ✅ |
| Docker | ✅ |
| 存储池(SD 自动建池挂载) | ✅ 首次开机自动创建 |
| GPU 硬件加速 | ❌ mainline 无 panfrost renderD(H618 天花板) |
| 有线网口 | ❌ H616 内部 EPHY 无 mainline 驱动 |

---

## 修复的问题

本版修复的问题详见 **[docs/BUGFIX-V2.5.md](docs/BUGFIX-V2.5.md)**,诚实分两类:

**A 类 · 官方 fnOS 自带 bug**(官方镜像同样存在)
- **nmbd.service 配置错误**(`Type=notify` 却用 `-D`)→ 每次改设备名卡约 90 秒、NetBIOS 广播失效。修复后 90s → 1s。
- **nginx 在慢存储上开机死循环**(每次开机重解压 ~56MB 前端包超默认 90s 超时)→ web 起不来。加长启动超时打破循环。

**B 类 · 移植适配相关修复**(因移植到 Zero2W / 嫁接内核而需要,非官方 bug)
- OTA 内核包依赖修复(dummy linux-image 包)
- 存储池首启自动创建逻辑
- 设备名 / 主机名清理

---

## 安装

1. 下载 [Releases](../../releases) 里的 `fnnas_zero2w_V2.5.gz`。
2. 用 [balenaEtcher](https://etcher.balena.io/) 或 `dd` 烧录到 microSD(≥8GB,建议用质量好的卡)。
3. 插卡上电,等待首次开机(首启会自动建存储池,耗时约数分钟)。
4. 浏览器访问 `http://<设备IP>:5666/`,按向导设置设备名、创建账号。
5. 可在"系统更新"里一键升级到最新 fnOS。

> 校验:下载后可比对 SHA256(见 Release 说明)。
> 关机请用系统"关机"软关,勿直接断电(保护 btrfs 文件系统)。

---

## 校验值

`fnnas_zero2w_V2.5.gz`
```
SHA256: 74afb8398c6b4c6f7e6bbc27abf78cdb722ae2be38ed6f636aafa64d279fa0f8
```

---

## 致谢 / 声明

- 基于飞牛 fnOS、Orange Pi 官方 Ubuntu 镜像、Linux mainline、展锐 uwe5622 驱动源码等。
- 商标与版权归各自所有者。本项目仅为个人移植适配,不含任何官方授权。
