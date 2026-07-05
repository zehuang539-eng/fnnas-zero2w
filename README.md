# fnnas-zero2w

**让飞牛 fnOS 完整跑在 Orange Pi Zero2W(全志 H618)上的定制镜像。**
*An unofficial fnOS (fnnas) NAS image port for the Orange Pi Zero2W (Allwinner H618).*

飞牛官方 ARM 公测版只支持 Orange Pi Zero3,**不支持 Zero2W**,而且 Zero3 基础镜像本身就带着不少毛病(无 HDMI、无 WiFi 驱动等)。本项目**以飞牛官方 ARM(支持 Zero3)为基础,一步步适配到 Zero2W**:先让它能启动、搞定 WiFi 与登录,再提取官方更新内核做**内核替换**修好 HDMI,逐个点亮外设、加状态灯,并修复若干官方自带 bug,最终产出可 OTA 升级的 **V2.5 稳定版**。

> ⚠️ 个人移植项目,与飞牛官方无关。仅供学习与自用,自负风险。涉及的商标、固件、闭源应用版权归各自所有者。

---

## 目录

- [下载](#下载)
- [外设支持](#外设支持)
- [主要特性](#主要特性)
- [安装](#安装)
- [开发历程](#开发历程)
- [技术实现](#技术实现)
- [已修复的问题](#已修复的问题)
- [致谢与免责](#致谢与免责)

---

## 下载

镜像通过本仓库 **[Releases](../../releases)** 分发(单文件约 1.9GB,不放进 git)。

**当前版本 V2.5** — 稳定版(拓展板双 USB 口 + 全修复,以此为准)
- 内核:`6.18.18.c798-trim`(arm64-sunxi,全外设 dtb)
- 应用:fnOS `1.1.31` —— **烧录后在网页"系统更新"里 OTA 升到最新**(已预埋禁 discard,升级不会触发 SD 擦除风暴)
- 集齐:**拓展板双 USB-A 口点亮**(enable phy2/phy3 + `wifi-reload` 避开与 WiFi 的开机时序冲突 + 内核更新自动恢复 dtb 的钩子)、**HDMI 开机后热插拔**(走 EDID/HPD)、**SD 擦除风暴根治**(禁 discard 三道锁)、**HDMI EDID 自适应**(含 2.8 寸 480×640 小屏)、全外设
- 已真机验证:**双口 USB(U盘/硬盘)+ WiFi 共存** / HDMI 热插拔 / 存储池首启自建 / web / 全外设

```
fnnas_zero2w_V2.5.gz
SHA256: 3adcbf724d9a1295862e9219c8492bc0b3ca28cded728ab271ce4ad2edfa1c67
```

> **版本说明**:**V2.5 为稳定版基准** —— 基于干净的 V2.5 基线重构,含双口 + 所有修复,镜像干净、约 1.9GB。另有 **V2.6.5 实验版**(app 预升到 1.1.3107 + libfix 等,但基于反复用过的系统、镜像较大约 2.3GB)仅作归档;一般用户用 V2.5 即可,烧完首启后自己 OTA 升 app。
> ⚠️ **勘误**:早期版本拓展板 **USB-A 口曾实测不可用**(飞牛照 Zero3 抄的 dtb 默认关闭了 H616 的 USB2/USB3 host controller,插 U盘/硬盘无反应);现 V2.5 已修复为**双口可用**。

---

## 外设支持

| 外设 | 状态 | 说明 |
|---|---|---|
| WiFi(展锐 UWE5622) | ✅ | 为 6.18.18 重编的 out-of-tree 驱动 + 固件;开机自连,固定全球 MAC 防驱动清址 |
| HDMI 显示 | ✅ | 6.18.18 原生 H616 显示驱动 + dtb 补 hdmi-connector;**支持开机后热插拔**(去掉 cmdline 强制 `video=...@..e` 标志、走 EDID/HPD,开机不插屏之后热插也自动点亮 + 适配分辨率) |
| SD 存储池 | ✅ | 首次开机自动创建(mdadm+LVM+btrfs),自动挂载 /vol1 |
| USB-C 虚拟网卡 | ✅ | RNDIS,无网时应急通道 `http://10.42.0.1:5666`;有 WiFi/网线时自动关闭 |
| 拓展板 USB-A 口 | ✅(双口版) | 飞牛照 Zero3 抄的 dtb **默认关闭了 H616 的 USB2/USB3 host controller** → **单口基线版(V2.5 及早期 V2.6.5)拓展板 USB-A 实测不可用**(插 U盘/硬盘无反应);双口版 enable phy2/phy3 后两个 USB-A 口可用(U盘/移动硬盘),另配 `wifi-reload` 服务避开与 WiFi 的开机时序冲突(见"已修复的问题") |
| 音频(3.5mm line-out) | ✅ | 拓展板 3.5mm;测试放音需 `apt install alsa-utils` |
| 红外接收 | ✅ | rc0;遥控器非标准协议时可走 lirc 原始脉冲 |
| 板载按键 KEY1/KEY2 | ✅ | lradc,`KEY_VOLUMEUP` / `KEY_VOLUMEDOWN` |
| 状态 LED | ✅ | 用户态活跃度指示(越忙越慢闪);**不使用内核 heartbeat 触发器**(会崩内核) |
| Docker | ✅ | 数据目录绑定到存储池 |
| **有线百兆网口** | ❌ | H616 **内部 EPHY** 在 mainline 无驱动(`dwmac-sun8i` 无 h616 variant),dtb 解决不了 |
| **GPU 图形(Mali-G31)** | 🚧 | 编译 PPU 电源域驱动可点亮 panfrost(出 `renderD128`、OpenGL ES 可用),**已实测点亮**;但 PanVK(Vulkan)对 G31 被 mesa 标 broken、仅 1.0,对 AI/compute 无用、3D 也弱,不值得固化,详见技术实现 |
| **VPU 视频硬解** | ❌ | H616 有独立 Cedrus VPU(与 Mali 无关),飞牛 ffmpeg 也带 v4l2request 硬解;但该内核 cedrus 驱动只认到 h6、不支持 h616(h6 fallback 会内核忙循环),要硬解需改驱动源码,当前纯 CPU 软解(2K 可扛,4K 吃力) |

---

## 主要特性

- **开机稳定(V2.6.x 新增)**:根治了两个致命问题——① 构建期 `/lib` 符号链接被 firmware 解包冲毁,致 systemd 无法启动的开机 panic;② app 1.1.3107 把 btrfs 挂成 `discard=async`,大量写入时触发 SD `mmc_erase -110` 擦除风暴。后者用**禁 discard 三道锁**根治(udev 块层 + fstab + 首启一次性自毁 guard)。
- **拓展板双 USB-A 口(V2.6.5 新增)**:enable H616 的 phy2/phy3 host controller,两个 USB-A 口可插 U盘/移动硬盘。难点是一开 USB 就和板载 WiFi(展锐 WCN)抢开机时序导致 WiFi 挂 —— 用 `wifi-reload` 服务在系统起稳后重载 WCN 避开冲突,再加内核更新自动恢复 dtb 的钩子防更新覆盖。详见开发历程/已修复。
- **HDMI 分辨率自适应 + 开机后热插拔(V2.6.x 新增)**:去掉固定 `video=` 走 EDID/HPD,接什么屏用什么屏的原生分辨率(实测 2.8 寸 480×640 小屏 1:1、普通显示器 1080p/720p 自适应),且**开机不插屏、之后热插也能点亮**。
- **首启自动建存储池**:插卡即用,首次开机自动在 SD 剩余空间上建池并挂载,无需手动操作;残留旧卡也会先清后建。
- **系统更新可用**:修复了官方 OTA 在本板"装不上"的问题,app 可在线 OTA 升级;内核走"离线自动完成"机制,规避 FAT32 `/boot` 无法建软链的官方缺陷。
- **官方 bug 修复**:改名慢(nmbd)、开机 web 死循环(nginx)等,详见下文"已修复的问题"。
- **干净安全**:无后门 SSH key、无预置 WiFi 凭据、无自定义主机名 hack(设备名走官方首启向导手填)。
- **配套上位机(Windows)**:编译好的 `Fnos-connect.exe` 见 [Releases](../../releases)(免安装,双击即用)。主要**配合本项目原创的 USB-C 虚拟网卡功能**——板子完全无网时,用 USB-C 连电脑、设网卡 IP(默认 `10.42.0.2`,板子固定 `10.42.0.1`)即可直连管理页;也能一键扫描局域网/异地组网里的飞牛设备并打开。设 IP 需管理员权限。

---

## 安装

1. 到 [Releases](../../releases) 下载 `fnnas_zero2w_V2.5.gz`,建议比对上面的 SHA256。
2. 用 [balenaEtcher](https://etcher.balena.io/) 烧录到 microSD。
   - **卡容量**:镜像解压后约 6.1GB,**最低 8GB 卡**即可烧录启动(4GB 装不下)。存储池空间 = 卡容量 − 系统占用,想要可用的存储空间建议用更大的卡(如 32GB 及以上);卡质量差易触发 SD 擦除超时。
   - 复用旧卡建议先 `diskpart clean` / 格式化,清掉旧分区。
3. 插卡上电,等待首次开机(会自动建池,约数分钟)。
4. 浏览器打开 `http://<设备IP>:5666/`(无网时用 USB-C 连电脑,访问 `http://10.42.0.1:5666/`),按向导设置设备名、创建账号。
5. 可在"系统更新"里升级到最新 fnOS。

> **关机务必走系统"关机"软关,勿直接断电** —— 直接断电可能损坏 SD 上的 btrfs 文件系统。

---

## 开发历程

本项目**不是从零做系统**,而是以飞牛官方 ARM 公测版(支持 Orange Pi Zero3)为基础,一步步适配到官方不支持的 Zero2W,并在过程中修掉大量基础镜像自带的问题。下面按时间顺序记录"到底改了什么、为什么"。

> 参考:飞牛官方 ARM 公测与社区反馈见官方论坛 [club.fnnas.com](https://club.fnnas.com/)([ARM 公测帖](https://club.fnnas.com/forum.php?mod=viewthread&tid=55275)、[硬件兼容/BUG 反馈版块](https://club.fnnas.com/forum.php?mod=forumdisplay&fid=22))。Zero3 基础镜像的许多问题在社区亦有反馈。

**起点:Zero3 基础镜像的一堆问题。** 飞牛官方 ARM 版只把 Orange Pi Zero3 列入支持,而且早期 Zero3 镜像本身就带着不少毛病(社区多有反馈,我们移植到 Zero2W 时也逐一撞上):没有 HDMI 显示(精简内核 6.18.6-trim 不带 H616 显示驱动,插屏黑屏)、没有 WiFi 驱动(展锐 UWE5622 缺失)、拓展板外设(红外/按钮/音频/网口)普遍点不亮、系统更新(OTA)装不上/半途卡死、SD 擦除超时 `mmc_erase -110`(H616 sunxi-mmc quirk),还有一个会随机 panic 内核的坑——设备树里状态灯用了 `heartbeat` 定时器触发器,在这套内核上会空指针崩溃、随机时刻必崩(早期把它误判成"随机死机/掉驱动/建存储失败"很久)。而 Zero2W 连"能启动"都不具备——硬件和 Zero3 不同,官方压根没有它的镜像。

**阶段一:先让 Zero2W 能启动、能进管理页。** Zero3 的 u-boot 在 Zero2W 上会因 DRAM 初始化卡死,于是用 `orangepi_zero2w_defconfig` + `sun50i_h616` ATF BL31 **自编 Zero2W 专属 u-boot** 过启动第一关;再通过 USB-C OTG 做一个 **RNDIS 虚拟网卡**(configfs + 微软 OS 描述符 + 固定 MAC)作应急入口,电脑直连访问 `http://10.42.0.1:5666/` 建账号(后改成"有 WiFi/网线就自动关、无网才开"的条件切换)。至此能启动进飞牛、进网页,但外设仍是 Zero3 那副样子。

**阶段二:搞定网卡(WiFi)与登录。** 移植并编译展锐 **UWE5622** SDIO 驱动,配合设备树让 SDIO 控制器探测到无线芯片、加载 CP2 固件后 `wlan0` 起来。这里有个 MAC 随机化坑:NetworkManager 默认给随机 MAC,而展锐 STA 模式**拒绝随机 MAC 并清成全 0** → 握手失败、误报"密码错误";修复是关闭扫描随机化 + 指定一个**全球管理 MAC**(首字节 bit1=0),驱动才不清址,之后 WiFi 全链路(关联→拿 IP→上外网)打通。登录方面:飞牛 ARM 公测版早期在部分场景有登录限制,理顺网络与账号后首启向导可正常建账号并登录。搞定"能联网 + 能登录",才有后面拿官方更新做内核替换的前提。

**阶段三(转折点):提取更新版内核 + 内核替换 → HDMI 修复。** 早期试过"为了外设换成 Orange Pi BSP / 第三方内核",结果**飞牛应用层直接崩**(app 对平台有硬依赖)——教训是不能用外来内核替换飞牛内核。真正的突破是:登录账号后,飞牛官方 OTA 下发的新内核 `linux-image-6.18.18.c798-trim` 是 **arm64-sunxi(全志)包,解包里原生带 `sun50i-h618-orangepi-zero2w.dtb`**,说明官方新内核 6.18.18 本就支持 Zero2W。于是把基础镜像的 **6.18.6-trim 替换/升级为官方 6.18.18-sunxi**(仍是飞牛官方 `-trim` 血缘,app 不崩),一举拿到:**HDMI 修复**(6.18.18 原生带全套 H616 显示驱动,之前"永久放弃 HDMI"其实是 6.18.6 精简内核的锅、不是硬件问题)、**SD 擦除 `-110` 根治**(6.18.18 修掉了折磨全程的 sunxi-mmc 擦除 bug,dmesg 错误数归零,之前因它放弃的 OTA/存储方案随之盘活)。关键区分:换成外来 BSP 内核=失败(app 崩),升级到官方更新的 trim 内核 6.18.18=成功(同血缘升级)。

**阶段四:逐个点亮外设 + 重编 WiFi。** 内核换到 6.18.18 后,靠改设备树(dtb)逐个点亮拓展板外设(纯改 dtb、基本零编译):HDMI(补 connector + disable 无面板 lcd)、红外、按钮 KEY1/KEY2(lradc)、音频(3.5mm)。因新内核不带展锐 out-of-tree 驱动,**为 6.18.18 重新编译 UWE5622**。技术细节见下文。(排线小坑:红外/按钮曾一起收不到信号,真因是拓展板 FPC 排线接触不良,重插即好,不是 dtb/驱动问题。)

**阶段五:LED 状态灯(逻辑单独说明)。** 这是个"血泪教训 + 重新设计"。**第一,绝不用内核 heartbeat 触发器**:基础镜像 dtb 里状态灯用 `linux,default-trigger = "heartbeat"`,内核注册该 LED 就启用心跳定时器,回调在这套内核上会空指针(`__queue_work` wq=NULL)→ 随机时刻内核 panic,是早期"随机死机"的真凶;修复是 dtb 里 `default-trigger` 改 `none`,任何版本都禁用 heartbeat/timer 这类内核定时器触发器。**第二,改用用户态脚本驱动**(进程上下文,不碰内核定时器):开机早期在 systemd sysinit 阶段就把灯点成常亮(`default-on`)表示"系统活着",且与网络解耦——灯不随网络断开而灭,是"系统在运行"的可靠信号;就绪后做**活跃度指示(越忙越慢闪)**——脚本读 `/proc/stat` 算 CPU 占用映射到闪烁频率,空闲=最快(约 30Hz,近乎高频快闪)、满载=最慢(约 1Hz),实现上 `f = 30 − usage% × 29`,半周期 `0.5/f`,on/off 各 sleep。设计意图是高频快闪 = "系统没卡死"的直观证明(卡死了灯就不闪)、越忙越慢直观反映负载,代价是灯脚本占用约 10% CPU(权衡后保留——"看得见没卡死"值这个开销)。不用 PWM/内核 timer 做更精细的呼吸灯,正是因为 heartbeat 崩内核的教训:状态灯一律走用户态,稳定优先。

**阶段六:官方 bug 修复 → V2.5 定版。** 整合存储池自动化(autopool)、OTA 打通、官方 bug 修复(见下文两节),产出 **V2.5 稳定版**并真机验证:首启自动建池挂载、OTA 成功、WiFi/HDMI/音频/红外/按钮全通、改名秒生效、web 稳定、Docker 正常、20 项静态校验 × 3 遍全绿、无后门 key。整个历程由多轮真机 SSH/串口调试推进,期间推翻过多个悲观结论(HDMI"永久放弃"、mmc"硬件天花板"、GPU"只有 Armbian 能")——大多数"不可能"最终发现是精简内核或配置缺失所致。

**阶段七(V2.6.x):app 升级后的连锁问题根治 → V2.6.5 整理版。** 跟随 OTA 把 app 升到 1.1.3107 后,暴露并根治两个致命问题:① 构建期 firmware 解包把 `/lib` 符号链接冲毁 → 开机 panic(恢复符号链接 + 安全提取);② app 1.1.3107 的 `discard=async` 在慢 SD 上触发 `mmc_erase -110` 擦除风暴,把初次建池/onboard/改名/装应用全拖垮(还导致装应用时 docker 镜像没导入、"装了打不开")→ 禁 discard 三道锁根治(**推翻 V2.5"勿禁 discard"旧结论**,因 app 行为变了)。同时把 HDMI 改成 EDID 自适应(接 2.8 寸 480×640 小屏自动原生 1:1、接普通显示器 1080p/720p)。期间深入试过点亮 **Mali GPU**(panfrost 成功出 renderD128,但 PanVK 对 G31 不成熟)和 **VPU 视频硬解**(卡在 cedrus 不认 h616),结论与方法记录在"技术实现"。整合为 **V2.6.5 整理版**。

**阶段八(V2.5 双口稳定版):拓展板双 USB 口点亮 + HDMI 热插拔。** 实测拓展板 USB-A 插 U盘/硬盘无反应 —— 真因是飞牛照 Zero3 抄的 dtb 把 H616 的 USB2/USB3 host controller(`usb@5310000`/`5311000` 等)全 `disabled`(Zero3 不引出这些口、注释写着 "USB2 & 3 on FPC connector")。enable 后 USB 口能识别设备,却触发新问题:**一开 phy2/phy3,板载 WiFi(展锐 WCN)就挂**(`chip power on fail`)。挖到底——不是硬件/供电/内核 bug(实读内核源码证 `phy-sun4i-usb` 就是 mainline、`needs_phy2_siddq` quirk 齐全、CCU 里 USB 与 WiFi SDIO 寄存器无重叠),而是**开机时序竞争**:USB host controller 在开机 15-20s bring-up,撞上 WCN 芯片 SDIO 上电扫卡的固定短超时窗口。铁证:手动卸载 WCN、等 USB 稳定后重载,WiFi 立刻起来、双口+WiFi 共存。于是做 **`wifi-reload` 服务**(`After=multi-user.target`,系统起稳后自动重载 WCN,复现手动操作),双口与 WiFi 同时可用。**HDMI 热插拔**则是去掉旧 cmdline 里 `video=...@..e` 的强制标志(它让内核硬钉连接器、跳过 HPD 检测),走 EDID/HPD 后开机不插、之后热插也能点亮。另加内核更新钩子 **`zz-restore-dualport-dtb`**(排在 `10-sync-dtb` 之后)自动恢复双口 dtb,防内核更新覆盖。期间也确认:autosuspend 不阻热插拔(误判后精简)、内核根治此路不通(phy 驱动==mainline 无可改)。**最终把这套双口改动并入干净的 V2.5 基线重构成稳定版**(避开反复折腾积累的"脏块"、镜像保持约 1.9GB);那个基于用过的系统、较大的版本归档为 **V2.6.5 实验版**。

---

## 技术实现

**路线选择:保留飞牛内核血缘,而非替换成外来内核。** 替换成 Orange Pi BSP 内核会让飞牛 app 全崩(平台硬依赖);正确路线是用飞牛能接受的官方 `*-trim` 内核并逐个补外设,转机是官方 6.18.18-sunxi 内核原生带 Zero2W dtb,既保留飞牛血缘又拿到 H618 全套驱动。

**WiFi:为 6.18.18 重编 UWE5622。** 编译期解决 6.18 API 变化(`MODULE_IMPORT_NS` 字符串化、cfg80211 回调签名 `change_beacon`/`set_wiphy_params`/`ch_switch_notify`、timer/wakeup_source/gpio 兼容层);运行期把 mainline 没有的 BSP vendor 符号全部 stub/替代(`sunxi_wlan_set_power` 改由 dtb `wifi-pwrseq`+regulator、`sunxi_wlan_get_bus_index`、`sunxi_mmc_rescan_card` 靠 dtb `non-removable` 自动探测 SDIO、`sched_setscheduler`→`sched_set_fifo`、`vfs_stat` 用 `kern_path`+`vfs_getattr` 自实现)。真机:两模块加载 → CP2 固件运行 → `wlan0` 出现。

**设备树:逐外设点亮(纯改 dtb)。** HDMI——enable 显示链路 + **disable `lcd-controller@6511000`**(无面板,否则 DRM component master 组不齐)+ 给 hdmi 空 `port@1` 补 endpoint + 根节点加 `hdmi-connector` 并互连(对照 Armbian 工作 dtb 定位);WiFi SDIO——`non-removable` + `wifi-pwrseq`;红外——`ir@7040000` okay(PH10/IR_RX)+ modules-load `sunxi_cir`;按钮——`lradc@5070800` okay + `vref-supply`(1.8V)+ `button-500`(VOLUMEUP,0.5V)/`button-800`(VOLUMEDOWN,0.8V)+ modules-load `sun4i-lradc-keys`;音频——`codec@5096000` okay + `audio-routing`(3.5mm line-out 在拓展板);LED——`default-on` 常亮,严禁 heartbeat(见上文阶段五)。

**存储池:autopool 首启自动建池。** 链条 `SD p3 → mdadm raid1(1盘) → LVM(VG trim_<uuid>, LV "0") → /dev/mapper/trim_<uuid>-0 (btrfs) → /vol1`。要点:`mkfs.btrfs -f -K`(不擦除,规避 SD 擦除超时);postgres `mount` 表登记一行(id 需显式给),飞牛据此开机自动挂载并显示"存储空间";数据分区固定 = rootfs(p2)+1 = p3;建池前灭残留超级块 + 停 md(防 udev 自动组装)+ `mdadm --create` 重试防 busy;建后 grow 链 `mdadm --grow`→`pvresize`→`lvextend`→`btrfs resize` 吃满整卡;首次开机用标记文件门控——首启无条件先删后建(清残留旧池),非首启只挂载已注册池、绝不误删用户数据。

**系统更新(OTA):绕开 FAT32 软链官方 bug。** 官方 OTA 在本板"装不上"的唯一根因:内核包 postinst 要在 `/boot` 建 `vmlinuz`/`initrd.img` 软链,而 `/boot` 是 FAT32 不支持软链 → dpkg 半装 → 事务中止;而该软链根本不影响启动(u-boot 直接加载 `Image`+`uInitrd`+dtb)。处理:app 在线 OTA(登录账号授权后官方正常 `dpkg -i`,app 不碰 `/boot`);内核离线自动完成(systemd `path` unit 监控 `/var/lib/dpkg/status`,检测到内核包半装就自动 `update-initramfs` + 复制 Image/uInitrd/dtb 到 `/boot`,再把 dpkg 状态标为 installed,平时零常驻零轮询);依赖修复(内核是嫁接进来的、未 dpkg 注册 `linux-image-<ver>`,构建时造同版本空壳 deb `Provides: linux-image` 注册进 dpkg,满足 `linux-modules-trim` 依赖)。

**GPU 图形:已实测点亮,但 PanVK 对 G31 不成熟、不固化。** 真因确认:GPU 节点声明 `power-domains`,由 `power-controller@7010250`(`allwinner,sun50i-h616-prcm-ppu`)提供,但内核 `CONFIG_SUN50I_H6_PRCM_PPU` 未编译 → 电源域提供者永不注册 → panfrost `probe -110`(这也解释"为什么只有 Armbian 能点亮"——它开了这个 config)。**已实测点亮**:多源校验拿干净的 mainline `sun50i-h6-prcm-ppu.c`(208 行)→ 用板上现成 headers out-of-tree 编成 .ko(vermagic 匹配)→ `insmod` 注册电源域 provider → panfrost 早已 timeout 放弃、需手动 `echo 1800000.gpu > /sys/bus/platform/drivers/panfrost/bind` → probe 成功、`mali-g31` 识别、出 `renderD128`、OpenGL ES 可用。**但不固化**:PanVK(panfrost 的 Vulkan)对 Mali-G31 被 mesa 标 `not well-tested/broken`、强制模式也仅 Vulkan 1.0,对飞牛相册 AI/compute 加速无用(相册 `com.trim.gpu_verify` 会判定不支持回退 CPU),3D 也弱(G31 入门级)。等 mainline mesa 对 G31 成熟再固化。(注:firmware 里的 `mali_csffw.bin` 是 Valhall-CSF GPU 如 RK3588 G610 的固件,H618 的 Mali-G31 不用。)

**为什么 AI/compute 加速当前无法开发(而不是"不想做")。** GPU 有两层能力要分清:**图形**(OpenGL ES —— 已点亮、能用)和**通用计算**(Vulkan compute / OpenCL —— AI 推理、相册人脸识别靠这个)。卡死的是后者,且是三重**硬约束叠加**,没有一个能在本项目层面(补驱动/改 dtb/调配置)解决:
- **① 驱动栈不支持**:panfrost 的 Vulkan 实现 **PanVK 对 Mali-G31 被 mesa 上游标 `broken / not well-tested`**,强制开也只有 Vulkan 1.0,而 compute 需要更高版本 + 稳定实现;OpenCL 在 panfrost 上更是近乎空白。飞牛相册的 `com.trim.gpu_verify` 一探测就判定不支持、回退 CPU。
- **② 卡在上游、不是我们能绕的**:这是 **mesa / Linux 图形栈上游**对 G31 这颗老 GPU 的支持成熟度问题,得等上游把它做完整 —— 和 WiFi/HDMI 那种"补个驱动/改个 dtb 就点亮"的性质**完全不同**(那些是配置缺失,这个是上游根本没做好),本项目再怎么努力也补不出一个成熟的 Vulkan/compute 栈。
- **③ 硬件天花板**:Mali-G31 是**入门级 GPU**(算力很低),就算驱动哪天成熟,也撑不起像样的 AI 推理,投入产出比极低。

**一句话:不是没做,是"上游驱动不成熟 + 硬件本身太弱"双重卡死,本项目层面无解。** 真要在这类板子上跑本地 AI,方向不在这颗 GPU 上(H616 也没有 NPU)—— 只能靠 CPU 软算(慢)或外接专用算力,那是另一条独立的路。

**VPU 视频硬解:卡在 cedrus 驱动不支持 h616。** H616 有独立 Cedrus VPU(与 Mali GPU 是两颗独立硬件),飞牛 ffmpeg 7.1.3 也带 `v4l2request` hwaccel + h264/hevc/vp9 `v4l2m2m` 解码器。已能给飞牛 dtb 补上 `video-codec@1c0e000` 节点 + 补飞牛缺的 `sram@1a00000`/`sram-c1`(ve_sram)、编译写回 /boot。**但该内核 cedrus 驱动 of_match 最新只到 `sun50i-h6`、不认 h616**:直接 probe 不 match、无 `/dev/video`;给节点加 h6 fallback compatible 则 cedrus 用 h6 variant probe → 内核线程死循环忙等、load 冲 16-19 占满 CPU 却 probe 不出设备(h616 与 h6 的 VE 寄存器布局不同),已回退。要硬解只能改 cedrus 驱动源码加 h616 variant(大工程、需 VE 准确参数)。**当前纯 CPU 软解**:实测放视频飞牛走 CPU 软解(2K 可扛,4K 会把 4×A53 拖满)。

**已知硬限制。** 有线百兆网口走 H616 内部 EPHY,mainline `dwmac-sun8i` 无 h616 internal-phy variant、`sun50i-h616.dtsi` 无内部 EPHY 节点,dtb 层面无解。SD 擦除超时(`mmc_erase -110`)是 H616 sunxi-mmc quirk —— ⚠️ **修正 V2.5 旧结论**:app 1.1.31 时 btrfs 不带 discard、影响小;但 app 1.1.3107 会把 btrfs 挂成 `discard=async`,大量写入时触发擦除风暴严重拖垮系统,**V2.6.5 起改为从块层禁掉 discard 根治**(见已修复 C2),实测无副作用(`discard_max_bytes=0` 让内核忽略 btrfs 发的 discard,不冲突)。

**构建。** 镜像在 Vagrant + VirtualBox 的 Ubuntu VM 内,用 `qemu-aarch64-static` chroot 进 arm64 rootfs 完成 dpkg 与文件叠加,再缩盘打包。

---

## 已修复的问题

诚实分两类:**A 类 = 官方 fnOS 自带 bug**(官方镜像同样存在);**B 类 = 移植适配相关修复**(因移植/嫁接内核而需要,非官方 bug)。每条附真机实测证据。

**A1(官方)nmbd.service 配置错误 → 每次改设备名卡 90 秒 + NetBIOS 广播失效。** 改名后端 `sysinfo_service` 会 `systemctl restart ... nmbd`;而 `nmbd.service` 是 `Type=notify` 却用 `ExecStart=/usr/sbin/nmbd -D`(`-D` fork 成守护、原进程退出 → systemd 收不到就绪 → 死等满 90s `TimeoutStartSec` → 杀 → failed;Debian 官方本应是 `--foreground`)。实测 `restart nmbd` = 90.7s 后 failed、nmbd 被杀后 137/138 无监听。修复:drop-in 把 `ExecStart` 改回 `--foreground`;修复后 `restart nmbd` = 0.77s、active,NetBIOS 恢复,**改名 90s→1s**。

**A2(官方)nginx 在慢存储上开机死循环 → web(5666)起不来。** nginx 每次开机从 `www.zip`(约 56MB)解压前端到 rootfs,`trim_nginx.service` 未加长启动超时(默认 90s),慢存储上解压 >90s 被杀 → `Restart=always` 从头再解压 → 死循环。修复:drop-in `TimeoutStartSec=600`,让解压跑完打破循环。

**B1(移植)系统更新因内核包依赖不满足中断。** 内核手动嫁接、未 dpkg 注册 `linux-image-<ver>` → OTA 装 `linux-modules-trim` 卡半装 → 事务中止。修复:构建时造同版本空壳 deb 注册依赖;修复后 OTA 全套装完、0 半装包、内核未动(不变砖)。

**B2(移植)存储池首启自动创建 / 旧卡残留池撞车。** autopool 脚本自身逻辑(旧版按"最后分区+1"误建 p4、残留 md 被 udev 抢占致 busy),修复见上文技术实现;首启自动建池挂载、占用干净。

**B3(移植,早期自引入已撤销)设备名下划线致主机名非法。** 曾把设备名烤成含下划线的 hostname 导致异常;修复:撤销所有自定义命名,hostname 保持默认,设备名回归官方首启向导手填。

**C1(移植,V2.6.x)构建期 `/lib` 符号链接被 firmware 解包冲毁 → 开机 13.7s 内核 panic。** 构建脚本用 `dpkg-deb -x firmware...deb` 把 firmware 解包到 rootfs 根,deb 内 `lib/firmware/` 目录条目把 `/lib → usr/lib` 符号链接冲成真实目录 → 动态链接器 `/lib/ld-linux-aarch64.so.1` 失效 → run-init 无法 exec systemd → 串口 `Kernel panic - Attempted to kill init`(HDMI 上表现为启动日志滚到一半定格)。修复:恢复 `/lib` 符号链接、firmware/modules 并回 `/usr/lib`,构建脚本改"解到临时目录再搬"的安全提取;qemu chroot 实测 systemd 可 exec,真机启动到 login。

**C2(官方 app 行为变化,V2.6.x)app 1.1.3107 的 `discard=async` 触发 SD 擦除风暴 → 初次建池/onboard/改名/装应用全被拖垮。** app 1.1.31 时代 btrfs 不带 discard、影响小;app 1.1.3107 会把 btrfs remount 成 `discard=async`,建池/OTA/装应用等大量写入触发块层 discard → H616 sunxi-mmc `mmc_erase -110` 擦除风暴 → 存储 I/O 半瘫、load 冲高、服务崩、web 挂(实测装应用时撞上此风暴致 docker 镜像没导入、应用装了打不开)。修复:**禁 discard 三道锁**——udev 块层 `discard_max_bytes=0` + fstab `nodiscard` + 首启一次性自毁 service(护建池/onboard 高峰、跑完自删不留常驻);实测重启全程 0 次 mmc_erase。⚠️ **这修正了 V2.5 文档"勿禁 discard"的旧建议** —— app 新版行为变了,现在必须从块层禁掉。

**C3(移植,V2.5 双口版)拓展板 USB 双口点亮 + 与 WiFi 的开机时序竞争。** 飞牛照 Zero3 抄的 dtb 默认关闭 H616 的 USB2/USB3 host controller(`usb@5310000`/`5310400`/`5311000`/`5311400` 全 `disabled`)→ **拓展板 USB-A 实测不可用**(插 U盘/硬盘无反应,当面插串口零日志);enable phy2/phy3 后 USB 口能枚举设备,但一开就挂板载 WiFi(WCN `sdiohal wait scan card time out` / `chip power on fail`)。**根因非硬件/内核 bug** —— 实读内核源码证飞牛内核就是标准 mainline 6.18.18、`phy-sun4i-usb.c` 与 mainline 逐字节相同、`needs_phy2_siddq` quirk 齐全、ehci/ohci 已 shared reset、CCU 寄存器表里 USB PHY 与 MMC1(WiFi SDIO)完全不重叠 —— 而是**开机时序竞争**:USB host bring-up(15-20s)撞 WCN 的 SDIO 上电扫卡固定超时窗口。铁证:USB 稳定后手动 `rmmod`+`modprobe` WCN 即刻上电成功、`wlan0` 起来、双口+WiFi 共存。**修复**:`wifi-reload.service`(`After=multi-user.target`、起稳后自动重载 WCN 避开竞争)+ `zz-restore-dualport-dtb` 内核 postinst 钩子(防内核更新的 `10-sync-dtb` 覆盖双口 dtb)。热插拔靠系统本身(实测 USB autosuspend 不阻热插拔,曾误判并加禁用、后精简掉)。同一版一并修 HDMI 开机后热插拔(见下 C4)。

**C4(移植,V2.5 双口版)HDMI 开机后热插不亮。** 旧版 cmdline 的 `video=HDMI-A-1:...@..e` 里的 `e`(DRM_FORCE_ON)让内核把连接器硬钉成"已连接"、跳过 HPD 检测(内核源码 `drm_probe_helper.c`:强制连接器被 poll 循环直接 `continue`)→ 开机没插屏、之后热插就黑屏。修复:确保 cmdline 无 `video=` 强制后缀,走 EDID/HPD,开机不插、随后热插都能点亮并自适应分辨率(与小屏 480×640 适配不冲突)。

---

## 致谢与免责

**致谢**:飞牛 **fnOS / fnnas**(应用层官方 ARM 二进制)、**Orange Pi** 官方 Ubuntu BSP(内核/dtb/固件参考)、**Linux mainline** 与 **Armbian**(sunxi H616/H618 dtb 与外设参考)、展锐 **UWE5622** 驱动源码(Doct2O/orangepi-zero3-mainline-linux-wifi)。

**免责**:本项目为个人对开源与第三方组件的移植适配,**不含任何官方授权**,不对使用后果负责。涉及的商标、固件、闭源应用版权均归其各自所有者;请在遵守相关许可的前提下使用。
