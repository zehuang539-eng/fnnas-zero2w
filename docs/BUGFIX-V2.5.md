# fnnas Zero2W V2.5 — Bug 修复说明

> 面向 Orange Pi Zero2W(Allwinner H618)的 fnOS(飞牛)定制镜像。
> 本文列出 V2.5 相对官方原版修复的问题。**诚实分两类**:
> **A 类 = 官方 fnOS 自带 bug**(在官方镜像上同样存在,我们定位并修复);
> **B 类 = 移植适配相关修复**(因把 fnOS 移植到 Zero2W、手动嫁接内核、增加存储池自动化而引入或必须的修复,非官方原版 bug)。
> 每条都附**真机 SSH 实测证据**,不含推测。

---

## A 类:官方 fnOS 自带 bug（官方镜像同样存在）

### A1. nmbd.service 配置错误 → 每次改设备名卡 90 秒 + NetBIOS 广播失效

- **现象**:在"系统设置 → 设备信息 → 设备名称"修改设备名时,界面卡顿约 90 秒才生效;同时 Windows"网络邻居"/局域网 NetBIOS 扫描找不到该设备。
- **根因**(真机实测坐实):
  1. fnOS 改名后端 `sysinfo_service`(`OnSetHostName` → systemd dbus `SetStaticHostname`)在改名时执行 `systemctl restart wsdd2 upnp avahi nmbd`——**改名必然重启 nmbd**。
  2. 官方 `/etc/systemd/system/nmbd.service` 为 **`Type=notify`**,但 `ExecStart=/usr/sbin/nmbd -D`。`-D` 使 nmbd fork 成后台守护、原进程立即退出,systemd 永远收不到 `sd_notify` 就绪信号 → **死等满 `TimeoutStartSec`(默认 90 秒)→ 判定超时 → SIGTERM 杀掉 → 标记 failed**。
  3. Debian 官方 nmbd.service 的正确写法是 `ExecStart=/usr/sbin/nmbd --foreground --no-process-group`(前台运行,配合 notify),fnOS 改成了 `-D`,导致每次启动必超时。
- **实测证据**:
  - `systemctl restart nmbd` = **90.7 秒后 failed(Result: timeout)**;
  - 改名后端原样命令 `systemctl restart wsdd2 upnp avahi nmbd` = 同样卡 ~90 秒;
  - nmbd 被杀后 137/138 端口无监听(NetBIOS 广播停摆)。
- **修复**:systemd drop-in `/etc/systemd/system/nmbd.service.d/override.conf`
  ```
  [Service]
  ExecStart=
  ExecStart=/usr/sbin/nmbd --foreground --no-process-group $NMBDOPTIONS
  ```
  (空 `ExecStart=` 先清掉原行,再改为前台。)
- **修复后实测**:`restart nmbd` = **0.77 秒、active**;改名原样命令 = **1.09 秒**;NetBIOS 137/138 恢复监听。**改名 90 秒 → 1 秒。**

### A2. nginx 开机在慢/累存储上死循环 → web(5666)起不来

- **现象**:在被反复读写、较累的 SD 卡上,开机后 web 管理界面(端口 5666)长时间打不开,`trim_nginx` 反复重启。
- **根因**(真机实测):fnOS 定制 nginx 每次开机从 `/usr/trim/share/.restore/www.zip`(约 56MB)解压前端到 rootfs。`trim_nginx.service` 为 `Type=forking` 且未加长启动超时,systemd 默认 90 秒;在慢存储上 58M 小文件解压 >90 秒被杀 → `Restart=always` 从头再解压 → **死循环**,并反复重写拖累存储。
- **实测证据**:手动不限时启动 nginx,www 目录大小反复从 58M 掉回再涨(重复解压),约 **+120 秒**才完成并监听 5666;systemd 默认 90 秒必被杀。
- **修复**:systemd drop-in `/etc/systemd/system/trim_nginx.service.d/override.conf`
  ```
  [Service]
  TimeoutStartSec=600
  ```
  让解压跑完再判定,打破死循环。
- **修复后实测**:nginx 稳定 active、5666 监听,开机不再循环。
- **备注**:这是官方在"慢存储 + 每次开机重解压 www.zip"设计下的健壮性缺陷。全新好卡上解压快、通常不触发;累卡/OTA 后触发。本修复只保证"最终一定起来",不改变每次开机重解压的官方机制。

---

## B 类:移植适配相关修复（非官方原版 bug，因 Zero2W 移植/嫁接内核/加存储池自动化而需要）

### B1. 系统更新(OTA)因内核包依赖不满足而中断

- **现象**:点"系统更新"后,首个内核模块包 `linux-modules-trim-6.18.18.c798-trim` 解包到 iU(half-installed)后配置失败,整个 apt 事务中止,后续 firmware/libc 等包全部装不上。
- **根因**:6.18.18 内核是移植时**手动嫁接**进镜像的(直接拷 Image/modules/dtb),并未通过 dpkg 安装 `linux-image-6.18.18.c798-trim` 包 → 模块包声明的依赖在 dpkg 数据库里根本不存在 → 每次 OTA 配置它必失败。**这是我们移植方式的副作用,非官方 bug。**
- **修复**:构建时造一个同版本**空壳 deb**(仅 `DEBIAN/control`,`Provides: linux-image,linux-image-arm64`,无 postinst,内核物理文件已在 /boot)→ 通过 qemu chroot `dpkg -i` 注册,满足依赖。
- **实测证据**:修复前点更新 → modules 卡 iU、事务中止;修复后点更新 → app 1.1.31→1.1.3107 + firmware 20260521 + modules/headers 全装,**dpkg 全 ii、0 半装包**,/boot 内核未被改动(无变砖)。

### B2. 存储池首次开机不自动创建 / 旧卡残留池撞车

- **现象**:官方 fnOS 在此板不会自动建存储池;用旧卡重烧时,残留的旧分区/池会被 fnOS 抢注,导致存储空间异常。
- **根因**:这是我们**新增的存储池自动化脚本**(`trim-autopool`)自身的逻辑问题,非官方 bug:旧逻辑按"最后分区+1"建池会误建 p4;残留 md 超级块被 udev 自动组装占用 p3 导致 `mdadm --create` device busy。
- **修复**:重写建池逻辑——数据分区固定 = rootfs(p2)+1 = p3;先灭超级块再停 md 防自动组装;`mdadm --create` 重试;`pvcreate -ff`;首次开机 `force_reset` 无条件先删后建(仅首启,不碰已有用户数据)。
- **实测证据**:最终版 V2.5 全新烧录卡**首次开机存储池自动挂上、占用干净**(非旧残留池);重启后非首启路径自动挂载 /vol1 + 7 个 bind 成功;btrfs 设备错误统计全 0。

### B3. 设备名下划线导致主机名非法(我们早期引入,已撤销)

- **现象**:早期版本尝试把设备名烤成 `hostname=TKAET_Zero2W`,导致疑似无法登录/更新。
- **根因**:**我们自己的失误**——主机名不允许下划线,静态与运行时主机名不一致。非官方 bug。
- **修复**:撤销所有设备自定义命名,hostname 保持基底默认 `fnnas`(合法),移除所有前端预填 hack。设备名回归官方"首启向导手填"的干净状态。

---

## 校验与交付

- 两镜像 `gzip -t` 完整;`verify-img.sh` 静态校验(20 项,含上述各修复项)**三遍全绿、0 FAIL**;桌面交付件与构建源 SHA256 逐字节一致。
- 已知硬件天花板(非本版能解,mainline/fnOS 限制):GPU 硬件加速(panfrost 无 renderD)、有线网口(H616 内部 EPHY 无 mainline 驱动)、`mmc_erase -110`(H616 sunxi-mmc 硬件 quirk,btrfs async discard 容忍)。

*文档随 V2.5 一并发布。日期:2026-07-04。*
