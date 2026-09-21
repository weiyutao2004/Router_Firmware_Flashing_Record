# 小米路由器 4(R4 千兆版)刷机操作记录

> **设备信息**
>
> | 项目 | 参数 |
> | --- | --- |
> | 设备名称 | 小米路由器 4 |
> | 型号 | R4(千兆版) |
> | CMIIT ID | 2018AP0335 |
> | SoC | MediaTek MT7621A(MIPS 双核 880MHz) |
> | 内存 / 闪存 | 256MB DDR3 / 128MB NAND Flash |
> | 无线 | 2.4GHz + 5GHz 双频 |
> | 有线 | 1 WAN + 3 LAN,千兆 |
> | 原厂系统 | MiWiFi 稳定版(基于 OpenWrt 二次开发,未开放 SSH) |

> **免责声明**
>
> - 刷机有变砖风险,操作前请务必通读全文,**所有操作后果自负**。
> - 刷机后设备失去官方保修。
> - 锐捷认证、TTL 统一、MAC 伪装等操作仅用于个人学习研究用途,
>   请遵守所在学校/单位网络管理规定,由此产生的一切责任由使用者本人承担。

---

## 目录

1. [为什么要刷机](#1-为什么要刷机)
2. [名词与原理简介](#2-名词与原理简介)
3. [本机资源清单](#3-本机资源清单)
4. [备份说明(刷机前必做)](#4-备份说明刷机前必做)
5. [网络拓扑要求(最大的坑)](#5-网络拓扑要求最大的坑)
6. [完整刷机步骤](#6-完整刷机步骤)
7. [校园网锐捷认证与反侦察配置](#7-校园网锐捷认证与反侦察配置)
8. [故障排查与救砖](#8-故障排查与救砖)
9. [附录](#9-附录)

---

## 1. 为什么要刷机

小米路由器 4(千兆版)的原厂固件封闭且不可降级开启 SSH,第三方插件无法安装,
不支持锐捷(802.1X)校园网认证。刷入第三方固件后可以获得:

- **锐捷校园网认证能力**:配合 minieap 或 Padavan 内置插件,宿舍网口直接入网;
- **完整 SSH / Telnet 权限**:可自由安装软件、写脚本、定时任务;
- **丰富的插件生态**:文件共享、广告过滤、多拨等按需扩展;
- **Breed 引导器兜底**:即使固件刷坏,也能随时通过 Breed 恢复,变砖概率大大降低。

## 2. 名词与原理简介

在动手之前,先弄清楚下面几个概念,后续步骤会反复用到:

| 名词 | 解释 |
| --- | --- |
| **SSH / Telnet** | 远程登录路由器 Linux 命令行的协议。小米原厂固件默认**没有**开启 SSH,第一步要靠漏洞激活。 |
| **OpenWRTInvasion** | Python 漏洞利用脚本,利用小米原固件 Web API 的远程命令执行漏洞,替我们打开 SSH、Telnet、FTP,取得系统权限。 |
| **stok** | 登录小米路由器管理页面后,浏览器地址栏中 `stok=` 与 `/` 之间的一串十六进制字符,是管理接口的临时令牌,漏洞脚本需要它来执行命令。 |
| **Breed(不死鸟控制台)** | hackpascal 开发的第三方 Bootloader(引导器)。刷入后**常驻闪存的 Bootloader 分区**,提供 Web 控制台(`192.168.1.1`),可备份全片、刷写任意固件、重置系统,相当于"系统之下的保险层"。 |
| **mtd / mtd_write** | Linux 下直接向闪存分区写入镜像的工具。刷 Breed 就是向 `Bootloader` 分区写入。 |
| **kernel1 / rootfs0** | OpenWrt 的两个独立分区镜像:`kernel1` 只含 Linux 内核,`rootfs0` 只含根文件系统。**两个都刷入才是完整可启动的系统**,只刷 kernel1 会卡在内核无法进入系统。 |
| **sysupgrade** | 包含"内核 + 文件系统"的完整升级镜像,用于在 OpenWrt 系统内升级,首次刷机一般不用它。 |
| **initramfs-kernel** | 临时系统镜像,启动后完全运行在内存里,不对闪存做任何改动。常用来测试固件能否正常引导,或作为救援中间步骤。 |
| **minieap** | Linux 下的锐捷(802.1X)认证客户端,支持锐捷 V3 私有协议(`rjv3` 模块),是 OpenWrt 上通过校园网认证的关键工具。 |
| **eeprom / Bdata** | 闪存中的无线校准数据分区,决定 WiFi 功率和信号质量,**刷机前必须备份**,一旦丢失 WiFi 会明显劣化。 |

---

## 3. 本机资源清单

本文件夹 `刷机工具/` 目录下已备齐全部所需文件,**开始前请核对完整性**:

### 3.1 刷机工具

| 文件 | 用途 |
| --- | --- |
| `刷机工具\openwrt激活路由器SSH工具\OpenWRTInvasion-0.0.8.zip` | 利用漏洞激活原厂系统的 SSH / Telnet / FTP。需要电脑安装 Python 3。 |
| `刷机工具\不死鸟控制台Breed\breed-mt7621-xiaomi-r3g.bin` | Breed 控制台镜像。**注意:R4 千兆版没有专属 Breed,社区通用 `xiaomi-r3g` 版本**——R4 千兆版与小米路由器 3G 硬件同源,Bootloader 布局一致,实测可正常使用。 |
| `刷机工具\openwrt固件包\...-initramfs-kernel.bin` | OpenWrt 临时测试系统(内存运行),可选,用于救援/验证引导。 |
| `刷机工具\openwrt固件包\...-squashfs-kernel1.bin` | OpenWrt 内核分区镜像 → 刷入 Breed 的 **Kernel1** 分区。 |
| `刷机工具\openwrt固件包\...-squashfs-rootfs0.bin` | OpenWrt 根文件系统分区镜像 → 刷入 Breed 的 **RootFS0** 分区。 |
| `刷机工具\openwrt固件包\...-squashfs-sysupgrade.bin` | 完整升级镜像 → 用于在 OpenWrt 系统内部后续升级。 |
| `刷机工具\Padavan老毛子固件\MI-4_3.4.3.9-099.trx` | Padavan(老毛子)固件,Breed 中直接上传即自动分区刷入。 |
| `刷机工具\远程连接路由器工具\putty-64bit-0.85-installer.msi` | SSH 命令行终端。 |
| `刷机工具\远程连接路由器工具\WinSCP-6.5.7-Setup.exe` | SFTP 图形化文件传输(向路由器上传文件用)。 |
| `刷机工具\锐捷认证工具\minieap_0.93.7-1_musl_mipsel_24kc.ipk` | 锐捷认证客户端。架构 `mipsel_24kc` 与本机 OpenWrt 24.10 匹配。 |

### 3.2 备份与配置

| 文件 | 用途 |
| --- | --- |
| `Padavan备份\Storage_MI-4.TBZ` | Padavan 系统的完整配置备份,可在 Padavan 后台一键恢复。 |
| `路由器原始系统备份\` | 原厂系统闪存备份,详见[第 4 节](#4-备份说明刷机前必做)。 |

> 文件完整性校验值(MD5)见[附录 9.3](#93-文件-md5-校验值)。

---

## 4. 备份说明(刷机前必做)

**任何刷机动作之前,先做备份。** 本文件夹 `路由器原始系统备份/` 中已经保存了完整的备份:

| 备份文件 | 内容 |
| --- | --- |
| `backup-Bdata-xiaomi_r3g_factory.bin` | Bdata 分区(小米设备参数) |
| `backup-eeprom-xiaomi_r3g_factory.bin` | **EEPROM 无线校准数据(最重要,丢了 WiFi 功率无法恢复)** |
| `backup-flash-dump-xiaomi_r3g_factory.bin` | 原厂系统状态下的全片闪存 dump |
| `backup-flash-dump-raw-xiaomi_r3g_factory.bin` | 原厂系统全片 raw 模式 dump(编程器救砖首选) |
| `backup-flash-dump-xiaomi_r3g_openwrt.bin` | 刷入 OpenWrt 后的全片 dump |
| `backup-flash-dump-xiaomi_r3g_padavan.bin` | 刷入 Padavan 后的全片 dump |
| `backup-flash-dump-xiaomi_r3g_pandorabox.bin` | 刷入 PandoraBox 后的全片 dump |

有了这些备份,即使完全变砖,也可以用 SPI 编程器(CH341A 等)把全片 dump 烧回闪存救活。

**如果是全新设备、还没有备份**,按以下方式补做:

- **进入 Breed 后备份(推荐,最简单)**:Breed Web 控制台 → `固件备份` 页 →
  分别勾选"编程器固件(全片)"和"EEPROM" → 点击备份,保存到电脑。
- **SSH 命令行备份**(刷 Breed 之前):
  ```sh
  cat /proc/mtd            # 查看分区表,确认分区名与大小
  dd if=/dev/mtd2 of=/tmp/eeprom.bin   # 按实际分区号备份 eeprom
  # 备份完成后用 WinSCP 下载到电脑保存
  ```

---

## 5. 网络拓扑要求(最大的坑)

OpenWRTInvasion 激活 SSH 时,**要求路由器自身能够访问公网**——漏洞脚本需要在路由器上联网
下载 `busybox`、`dropbear` 等组件,路由器上不了外网就会一直卡住、失败。

由此得出**电脑与路由器的正确连接方式**:

```
                 ┌────────────┐
  校园网口 ────► │            │──── LAN口 ────► 电脑(下级节点)
  /光猫(上级)    │  小米路由器  │
                 └────────────┘
                        ▲
                        └──── 或电脑连接路由器的 2.4G/5G WiFi
```

- 正确:电脑通过**网线插路由器 LAN 口**,或**连接路由器的 WiFi**;
  路由器 WAN 口接校园网口/光猫等可上网的上级网络。
- 错误:电脑用自己的无线网卡开热点(ICS 共享),把电脑作为路由器的**上级**
  供给网络——这种方式下漏洞利用脚本通常会**被拒绝访问**,激活失败。

简单记:**电脑当路由器的"下级",让路由器先上得了网,再谈激活 SSH。**

---

## 6. 完整刷机步骤

整体路线图:

```
原厂固件
   |  ① OpenWRTInvasion 激活 SSH(需路由器能上公网)
   v
获得 SSH/Telnet/FTP 权限
   |  ② WinSCP 上传 Breed 到 /tmp,mtd 写入 Bootloader 分区  ★最危险
   v
Breed 不死鸟控制台(192.168.1.1)
   |  ③ 任选其一:
   |     A. 刷 Padavan(自带锐捷认证,开箱即用)
   |     B. 刷 OpenWrt(kernel1 + rootfs0 双分区刷入,可玩性最强)
   v
完整第三方系统 + 锐捷校园网认证 + 反侦察配置
```

> ### 风险分级速览
>
> | 步骤 | 危险程度 | 说明 |
> | --- | --- | --- |
> | 激活 SSH | ★☆☆ | 只改系统配置,重启即恢复,几乎无风险 |
> | **刷入 Breed** | ★★★★★ | **直接改写 Bootloader 分区,过程中断电 = 变砖** |
> | Breed 里刷固件 | ★☆☆ | 失败了大不了重进 Breed 重刷,**只要不破坏 Breed 自身所在的分区就没事** |

### 步骤 1:环境准备

1. 电脑安装 **Python 3**(官网 python.org 下载,安装时勾选 *Add to PATH*);
2. 安装 **PuTTY**(SSH 终端)和 **WinSCP**(文件传输),均在 `刷机工具\远程连接路由器工具\` 中;
3. 按第 5 节要求把电脑接成路由器的**下级节点**(LAN 口或 WiFi),路由器 WAN 口接可上网的上级;
4. 确认能打开小米路由器管理页面 `http://192.168.31.1` 并登录成功;
5. 路由器供电稳定,**整个刷机过程保持不断电**。

### 步骤 2:用 OpenWRTInvasion 激活 SSH

1. 登录路由器管理页面 `http://192.168.31.1`,**从浏览器地址栏复制 stok**:
   地址形如 `http://192.168.31.1/cgi-bin/luci/;stok=ff43b68c.../web/home`,
   `stok=` 与 `/` 之间的那串十六进制字符就是 stok。

2. 解压 `OpenWRTInvasion-0.0.8.zip`,在 PowerShell / CMD 中进入该目录并安装依赖:
   ```bat
   cd OpenWRTInvasion-0.0.8
   pip3 install -r requirements.txt
   ```

3. 运行漏洞脚本,按提示依次输入路由器 IP(默认 `192.168.31.1`)和刚复制的 stok:
   ```bat
   python remote_command_execution_vulnerability.py
   ```

4. 脚本执行成功后会开启 **SSH、Telnet、FTP** 三种远程通道:
   地址 `192.168.31.1`,用户名 `root`,**密码 `root`**。

5. **如果脚本长时间无反应**:多半是路由器联网下载 `busybox` / `dropbear` 失败。
   处理办法:
   - 检查路由器 WAN 是否真的能上外网(在管理页"常用设置"里看网络状态);
   - 或编辑 `script.sh`,把 `setup_busybox()` 与 `start_ssh()` 函数里的
     `curl` 下载地址替换成可用的镜像源后再重试。

6. 用 PuTTY 连接 `192.168.31.1`(端口 22),以 `root / root` 登录,能进 shell 即激活成功。

> 提示:激活 SSH 之后路由器仍运行原厂系统,随时可以重启还原,此步安全可逆。

### 步骤 3:刷入 Breed(不死鸟控制台)★ 最危险的一步

> 反复确认三件事再执行:
> 1. Breed 镜像与机型匹配(本机即 `breed-mt7621-xiaomi-r3g.bin`);
> 2. 路由器电源稳定,操作期间**绝不碰电源**;
> 3. 写入的是 **`Bootloader` 分区**,写错分区同样会变砖。

1. 打开 **WinSCP**,新建会话:协议 `SFTP`(或 SCP)、主机 `192.168.31.1`、
   用户名 `root`、密码 `root`;
2. 进入路由器的 **`/tmp`** 目录,把电脑上的
   `刷机工具\不死鸟控制台Breed\breed-mt7621-xiaomi-r3g.bin` **拖拽上传**到 `/tmp`;
3. 回到 SSH 终端(PuTTY),先校验文件完整(强烈推荐):
   ```sh
   cd /tmp
   md5sum breed-mt7621-xiaomi-r3g.bin
   # 与附录 9.3 中的 MD5 比对,一致才继续
   ```
4. 刷入 Breed(两种命令按当前系统环境二选一):
   ```sh
   # OpenWrt / 大多数环境:
   mtd -r write /tmp/breed-mt7621-xiaomi-r3g.bin Bootloader

   # 小米原厂固件环境(mtd_write):
   cd /tmp
   mtd_write write /tmp/breed-mt7621-xiaomi-r3g.bin Bootloader
   ```
   `-r` 表示写完自动重启。执行后**等待路由器完全重启**。
5. 验证是否进入 Breed:
   - 路由器**电源指示灯变为淡紫色(暗紫色)** = Breed 已生效;
   - 电脑改回"自动获取 IP",重新连接路由器,浏览器访问 `http://192.168.1.1`;
   - 能打开 Breed Web 控制台即成功。

> 提示:Breed 是之后一切操作的安全网——无论刷什么固件、刷成什么样,
> 只要不写坏 Breed 自己所在的 Bootloader 分区,就能断电按住 reset 上电回到 Breed 重新来过。

### 步骤 4:在 Breed 里刷入正式系统

刷固件失败时的退路:回 Breed 重刷即可,**只要不破坏 Breed 所在的分区就没事**。二选一:

#### 方案 A:刷 Padavan(老毛子)——开箱即用,自带锐捷认证

1. 浏览器打开 `http://192.168.1.1` 进入 Breed Web 控制台;
2. 进入 **`固件更新`** 页面,Breed 会自动识别固件类型并**自动完成分区**;
3. 固件选择电脑上的 `刷机工具\Padavan老毛子固件\MI-4_3.4.3.9-099.trx`;
4. 勾选"自动重启",点击**上传**,等待刷写完成并重启;
5. 重启后访问 `http://192.168.31.1` 进入 Padavan 后台(默认 `admin / admin`);
6. Padavan **内置锐捷认证**(校园网认证插件),在 Web 界面里填入学号密码即可,
   无需命令行操作;
7. 如需恢复历史配置:后台 → `高级设置` → `系统设置` → `配置文件管理` →
   上传 `Padavan备份\Storage_MI-4.TBZ` 恢复。

> Padavan 优点:图形化配置、锐捷认证开箱即用;
> 缺点:第三方插件生态和可操作性(自编译软件包、自定义脚本等)**略逊于 OpenWrt**。

#### 方案 B:刷 OpenWrt——可玩性最强,锐捷需手动装 minieap

1. 进入 Breed Web 控制台 → **`固件更新`** 页面,**手动指定分区刷入**
   (不要选"固件"自动模式):

   | Breed 分区 | 选择文件 |
   | --- | --- |
   | **Kernel1** | `openwrt-24.10.6-ramips-mt7621-xiaomi_mi-router-4-squashfs-kernel1.bin` |
   | **RootFS0** | `openwrt-24.10.6-ramips-mt7621-xiaomi_mi-router-4-squashfs-rootfs0.bin` |

2. **`Kernel1` 和 `RootFS0` 两个分区必须同时刷入**,才是完整版的系统:
   只有内核没有文件系统,路由器会启动失败;
   - 好消息:这一步就算刷错了也不致命,只要不动 Breed 所在分区,
     重回 Breed 重刷正确文件即可;
3. 勾选"自动重启",依次上传刷写,等待完成;
4. 重启后 OpenWrt 后台地址为 `http://192.168.1.1`,用户 `root`,默认无密码;
   无线默认 SSID 为 `OpenWrt`;
5. 首次进入后立刻修改 root 密码(系统 → 管理权限),否则无法远程 SSH。

> 备注:其余两个镜像文件的用途
> `initramfs-kernel.bin` 可在 Breed 里刷入 Kernel1 用于临时引导测试(内存运行、重启即失);
> `sysupgrade.bin` 用于日后在 OpenWrt 系统内升级(网页"系统→备份/升级"或 `sysupgrade` 命令)。

#### 系统选型总结

| | Padavan | OpenWrt |
| --- | --- | --- |
| 锐捷认证 | 内置,Web 配置 | 需另装 minieap(见第 7 节) |
| 插件生态 / 可玩性 | 一般 | **强大(官方软件源 3000+ 包)** |
| 刷入方式 | Breed 自动分区 | Breed 需手动刷 Kernel1 + RootFS0 |
| 适合人群 | 想省事直接用 | 想折腾、定制 |

---

## 7. 校园网锐捷认证与反侦察配置

> 本节以 **OpenWrt 24.10** 为准(Padavan 用户直接用其内置的校园网认证插件即可,思路相同)。
> 再次提醒:绕过校园网检测可能违反校规,请自行斟酌。

### 7.1 前置:OpenWrt 联网

把校园网口接到路由器 **WAN 口**,OpenWrt 的 WAN 默认 DHCP 客户端,插上即可获取内网 IP。
此时只是"连上了内网",还需要通过锐捷认证才能真正上网。

### 7.2 安装 minieap

**方式一:在线安装(路由器能上外网时)**

```sh
opkg update
opkg install libpcap1     # minieap 的依赖
opkg install minieap
```

**方式二:离线安装(推荐,文件已备好)**

1. WinSCP 把 `刷机工具\锐捷认证工具\minieap_0.93.7-1_musl_mipsel_24kc.ipk` 上传到路由器 `/tmp`;
2. SSH 登录后安装:
   ```sh
   cd /tmp
   opkg install minieap_0.93.7-1_musl_mipsel_24kc.ipk
   # 若提示缺少依赖(如 libpcap1),先到 OpenWrt 官方源
   # 下载对应架构(mipsel_24kc)的 libpcap1 ipk 一并离线安装
   ```

### 7.3 启动锐捷认证

```sh
# 先手动测试能否认证成功:
minieap -u 你的学号 -p 你的密码 -n wan --module rjv3
```

参数说明:

| 参数 | 含义 |
| --- | --- |
| `-u` | 认证用户名(学号) |
| `-p` | 认证密码 |
| `-n wan` | 监听的网卡/接口(WAN 口) |
| `--module rjv3` | 使用锐捷 V3 私有协议模块 |

- 看到认证成功、WAN 口拿到外网地址即为通过;
- 认证成功后**保存参数**,以后直接运行 `minieap` 即可:
  ```sh
  minieap -u 你的学号 -p 你的密码 -n wan --module rjv3 --save-params
  ```
- **开机自启**(编辑 `/etc/rc.local`,在 `exit 0` 之前加入一行):
  ```sh
  sleep 15 && minieap &     # 等网络就绪后再启动认证
  ```

### 7.4 反侦察配置(防"多设备共享"检测)

校园网一般通过以下特征判断"一个账号带了多台设备",需要逐一消除:

#### ① 统一数据包发出时的 TTL

不同系统的默认 TTL 不同:Windows 发出的包 TTL=128,Linux/Android/macOS 为 64。
路由器后面挂着多种设备时,出口数据包 TTL 混杂,一眼就能看出是共享网络。
**办法:把所有从内网转发出去的 IPv4 包 TTL 统一改写成一个值(常用 64)。**

OpenWrt 22.03+ 使用 nftables,在 **LuCI → 网络 → 防火墙 → 常规设置 → 自定义规则**
(或编辑 `/etc/firewall.user`)中加入:

```sh
nft add rule inet fw4 mangle_forward iifname "br-lan" counter ip ttl set 64
```

旧版本(21.02 及以前,iptables 时代):

```sh
iptables -t mangle -A PREROUTING -i br-lan -j TTL --ttl-set 64
```

> 验证:内网设备 ping 外网或在外网抓包,确认 TTL 已统一。
> 部分学校会检测特定 TTL 值,请按实际情况调整(也可改成与认证主机一致的值)。

#### ② 关闭 IPv6

很多校园网的锐捷检测/计费与 IPv6 联动,且 IPv6 会绕开部分管控,必须关掉:

```sh
# 删除 WAN 的 IPv6 接口
uci del network.wan6 2>/dev/null
uci commit network

# 停用并禁用 odhcpd(IPv6 RA/DHCPv6 服务)
/etc/init.d/odhcpd stop
/etc/init.d/odhcpd disable
```

同时在 **LuCI → 网络 → 接口 → LAN → DHCP 服务器 → IPv6 设置** 中,
把 RA 服务、DHCPv6 服务都设为**已禁用**。

#### ③ 修改设备名称(主机名)

DHCP 请求中会携带主机名,形如 `Xiaomi-Router` 的名字等于自报家门:

- **LuCI → 系统 → 系统 → 主机名**:改成普通 PC 风格(如 `DESKTOP-8K2F1M`);
- **LuCI → 网络 → 接口 → WAN → 高级设置 → "要使用的主机名"**:
  填同一个 PC 风格名称,让校园网 DHCP 服务器看到的是一台"电脑"。

#### ④ 使用虚假 MAC 地址(克隆 MAC)

WAN 口 MAC 若是小米 OUI,同样会被识别为路由器。把 WAN 口 MAC 改成
宿舍某台常用电脑的 MAC(克隆),或一个"普通电脑"风格的随机 MAC:

- **LuCI → 网络 → 接口 → WAN → 高级设置 → 覆盖 MAC 地址** → 填入目标 MAC → 保存并应用。

> 完成以上四项后,从校园网服务器视角看,这个"账号"就是一台 TTL 一致、
> 无 IPv6、主机名与 MAC 都像普通电脑的单机设备。

### 7.5 验证

1. 任选一台内网设备连接路由器,测试是否能打开外网网页;
2. 运行一段时间(几小时~几天)观察是否被踢下线/弹认证窗口;
3. 被踢时优先排查:minieap 是否仍在运行(`ps | grep minieap`)、TTL 规则是否还在、
   WAN 口 DHCP 是否异常等。

---

## 8. 故障排查与救砖

### 8.1 判断路由器处于什么状态

| 现象 | 状态 | 处理 |
| --- | --- | --- |
| 电源灯淡紫色/紫色(暗) | 在 **Breed** 中 | 访问 `192.168.1.1` 操作 |
| 蓝灯正常,WiFi 可连 | 系统正常运行 | 按当前系统后台地址访问 |
| 灯一直慢闪/快闪不停 | 固件损坏或启动失败 | 见 8.2 |
| 完全无灯、无网络、无反应 | **彻底变砖** | 见 8.3 |

### 8.2 常见故障

- **进不了 Breed**:断电 → 按住 reset 键不放 → 上电 → 约 10 秒后松开 →
  电脑自动获取 IP → 访问 `192.168.1.1`;
- **OpenWrt 启动失败**(只刷了 kernel1、没刷 rootfs0,或版本不匹配):
  回 Breed 重刷,务必 Kernel1 与 RootFS0 成对刷入、版本一致;
- **Breed 里刷固件失败/中断**:直接重来,重新上传刷写,不影响 Breed 本身;
- **WiFi 信号极差或 5G 消失**:EEPROM(无线校准数据)丢失——
  在 Breed"固件更新"里把备份的 `backup-eeprom-*.bin` 写回对应分区;
- **opkg 报架构不匹配**:只安装 `mipsel_24kc` 架构的软件包。

### 8.3 彻底变砖(Breed 也进不去)怎么办

1. **首选**:多试几次"断电 + 按住 reset 上电"——Breed 容错性很好,别急着下结论;
2. **次选**:用 USB-TTL 线接路由器串口针脚,看启动日志定位卡在哪一步;
3. **终极手段**:拆机,用 SPI 编程器(CH341A + 夹具)离线烧写闪存:
   - 用 `路由器原始系统备份\backup-flash-dump-raw-xiaomi_r3g_factory.bin`(raw 全片)
     直接烧回,即可恢复出厂数据;
   - 或只烧入 Breed 后再走标准刷机流程;
4. 以上都不行时,只能说:这就是为什么第 4 节的备份如此重要。

---

## 9. 附录

### 9.1 关键后台地址与账号速查

| 阶段 | 后台地址 | 账号 / 密码 |
| --- | --- | --- |
| 原厂系统 | `http://192.168.31.1` | 小米账号密码 / 管理密码 |
| OpenWRTInvasion 激活后(SSH/Telnet/FTP) | `192.168.31.1` | `root` / `root` |
| Breed 控制台 | `http://192.168.1.1` | 无需登录 |
| OpenWrt | `http://192.168.1.1` | `root`(首次登录自设密码) |
| Padavan | `http://192.168.31.1` | `admin` / `admin`(以固件实际为准) |

### 9.2 常用命令速查

```sh
# 查看闪存分区表
cat /proc/mtd

# 备份分区(以 eeprom 为例,分区号按 cat /proc/mtd 实际输出)
dd if=/dev/mtd2 of=/tmp/eeprom.bin

# 刷写 Bootloader(刷 Breed)——最危险
mtd -r write /tmp/breed.bin Bootloader

# OpenWrt 离线安装软件包
opkg install /tmp/xxx.ipk

# minieap 锐捷认证(认证成功后加 --save-params 保存,以后直接 minieap 即可)
minieap -u 学号 -p 密码 -n wan --module rjv3 --save-params
minieap --daemon                 # 后台运行
ps | grep minieap                # 检查认证进程是否存活

# TTL 统一(nftables)
nft add rule inet fw4 mangle_forward iifname "br-lan" counter ip ttl set 64

# 导出 OpenWrt 配置备份(生成 /tmp/backup-*.tar.gz,用 WinSCP 下载)
sysupgrade -b /tmp/backup.tar.gz
```

### 9.3 文件 MD5 校验值

```text
openwrt-24.10.6-...-initramfs-kernel.bin    cc372653d001243b142934c68ee31332
openwrt-24.10.6-...-squashfs-kernel1.bin    8a439a63031d8182fea531aef9360abe
openwrt-24.10.6-...-squashfs-rootfs0.bin    58c844d808814234d798ffe5335b73a6
openwrt-24.10.6-...-squashfs-sysupgrade.bin 60f92f1598ef9b779999e9a8f8b4a34d
OpenWRTInvasion-0.0.8.zip                   4d8aeb022ea6b6527d015e86d3b59f3c
MI-4_3.4.3.9-099.trx                        bc61f162323fa5c01da3ab56221c5618
breed-mt7621-xiaomi-r3g.bin                 e65d388129a6d1ac39abf99329f1978b
putty-64bit-0.85-installer.msi              112f138b2a80e53e94f956bdca0bcf0b
WinSCP-6.5.7-Setup.exe                      d105790e33d887217bd38ec0ed41147b
minieap_0.93.7-1_musl_mipsel_24kc.ipk       6ae5d30fbc577559b440fc63e8bb88e9
Padavan备份/Storage_MI-4.TBZ                bd5278e346edf1e85f6a8aaa96bcdff2
```

### 9.4 参考资料与下载地址

- OpenWRTInvasion(SSH 激活):https://github.com/acecilia/OpenWRTInvasion
- Breed 官方下载:https://breed.hackpascal.net/
- OpenWrt 官方固件(ramips/mt7621 → xiaomi_mi-router-4):https://downloads.openwrt.org/
- minieap:https://github.com/updateing/minieap
- 参考教程:
  - 小米路由器4千兆版刷OpenWrt(Fisher's Blog)
  - 小米路由R4A千兆版安装breed+OpenWRT教程以及救砖(CSDN)

---

*最后更新:2026-09-21*