# luci-app-dlna-player

> 让 OpenWrt 设备（尤其是 x86 小主机）的**声卡开箱即用** —— 自动探测声卡、配置 MPD 播放、并支持手机 DLNA 投送。

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)

## ✨ 特性

- 🔍 **声卡自动探测** —— 无需手动找 `hw:X,Y`，自动识别模拟输出设备
- 🔊 **自动解除静音** —— ALSA 新装默认静音，脚本自动处理
- 🎵 **MPD 自动配置** —— 一键生成可用的 `mpd.conf`
- 📡 **DLNA 投送** —— 手机（网易云 / QQ音乐 / BubbleUPnP）直接投送
- 💾 **持久化音量** —— 绕过 tmpfs，重启后音量不丢
- 🔄 **完整开机自启** —— 服务顺序自动编排

## 📋 适用场景

- x86 小主机 / 软路由装了 OpenWrt，想用它的 **3.5mm 音频口**放歌
- 想把 OpenWrt 变成一台 **DLNA 音箱**
- 树莓派等带音频输出的设备

## 📦 安装

### 方式一：加入源码编译

```bash
cd openwrt/package/custom
git clone https://github.com/<你的用户名>/luci-app-dlna-player.git
cd ../..
make menuconfig   # LuCI → Applications → luci-app-dlna-player
make -j$(nproc)
```

### 方式二：直接装 ipk

```bash
opkg install luci-app-dlna-player_1.0.0-r1_all.ipk
```

依赖会自动安装：`mpd-full` `mpc` `upmpdcli` `alsa-utils`

## 🎯 使用方法

### 1. 放音乐到音乐目录

```bash
mkdir -p /srv/music
cp your-music.mp3 /srv/music/
```

### 2. 检查声卡是否被正确识别

```bash
logread | grep dlna-player
# 应看到：auto-detected sound card: hw:3,0
```

### 3. 播放（命令行）

```bash
mpc update       # 扫描音乐库
mpc add /        # 添加全部
mpc play         # 播放
mpc volume 40    # 设置音量
```

### 4. 手机 DLNA 投送

```
手机音乐 App → 投屏/DLNA → 选择「小主机音箱」🎵
```

## ⚙️ 配置项

配置文件：`/etc/config/dlna-player`

```uci
config audio 'main'
	option enabled '1'              # 是否启用
	option device ''                # 声卡设备，留空=自动探测（如 hw:3,0）
	option music_dir '/srv/music'   # 音乐目录
	option volume '40'              # 默认音量 0-100
	option dlna_name ''             # DLNA 显示名，留空=用主机名
	option unmute '1'               # 自动解除 ALSA 静音
	option persist_volume '1'       # 持久化音量
```

修改后应用：

```bash
uci set dlna-player.main.volume='60'
uci commit dlna-player
/etc/init.d/dlna-player start
/etc/init.d/upmpdcli restart
```

## 🏗️ 工作原理

```
启动流程：
  S50 dlna-player   → 探测声卡 + 修权限 + 解静音 + 生成 mpd.conf + 建目录
  S93 mpd           → 启动 MPD（以 root 运行）
  S94 dlna-player-volume  → 设置默认音量（有持久状态则跳过）
  S95 upmpdcli      → 启动 DLNA 服务

播放链路：
  手机 App ──DLNA──▶ upmpdcli ──MPD协议──▶ MPD ──ALSA──▶ 声卡 ──▶ 🎵
```

## ⚠️ 踩过的坑（为什么要这么写）

OpenWrt 上做音频输出有一堆隐藏问题，这个包都处理了：

| 坑 | 现象 | 解决 |
|---|---|---|
| **`/root` 权限** | MPD 起不来 | 音乐放 `/srv/music` |
| **ALSA 默认静音** | 播放无声 | 自动 `amixer ... unmute` |
| **`/dev` 是 tmpfs** | 重启后权限丢失 | 每次开机 `chmod 666` |
| **`/var` 是 tmpfs** | 重启后目录/状态丢失 | 状态文件放 `/etc` |
| **PulseAudio 抢权限** | 权限被重置 | MPD 以 root 运行 |
| **busybox 无 `seq`** | 脚本报错 | 用 `while` 循环 |
| **启动顺序** | 声卡没好就启 MPD | 用 `START=50/93/94/95` 编排 |

## 🔧 兼容性

| OpenWrt 版本 | 状态 |
|---|---|
| 24.10 | ✅ 已测试 |
| 23.05 | ⚠️ 理论可用，未测试 |
| 25.12 | ⚠️ 需要适配 `apk` |

## 📄 License

Apache-2.0
