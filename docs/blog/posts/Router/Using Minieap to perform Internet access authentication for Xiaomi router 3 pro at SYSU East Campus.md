---
date:
  created: 2024-08-02
  updated: 2024-08-02
categories:
  - Router
tags:
  - Computer Network
comments: true
---

# 在小米路由器 3 Pro 上使用 Minieap 实现中山大学东校区有线上网。

## 背景知识

通常来说，中大的同学们通过叫做 `SYSU-SECURE` 或者 `SYSU-SECURE-2.4G` 的无线网络进行上网。采用这种上网方式，只需输入学生自己的 NetID 以及密码就可以完成认证。然而，对于想要使用宿舍提供的有线网络接入的同学来说，情况会不一样。由于每一位同学的座位前只提供一个网口，所以如果有多个设备需要上网，就需要将这个网卡接到路由器，让路由器进行一次网络地址转换(NAT)来规避这个问题。

不过采用有线接入仍然需要认证，同样需要 NetID 以及对应的密码。中大[网络与信息化中心(Information Center, Inc)](https://inc.sysu.edu.cn/) 为 Windows, macOS 以及 Linux 平台提供了锐捷认证客户端([个人用户有线网络接入 - 相关下载](https://inc.sysu.edu.cn/service/wired-network-access))。不过，在我的小米路由器 3 Pro上面进行上网认证仍然是一个棘手的挑战。我的小米路由器 3 Pro 上安装了 OpenWrt 23.05.4 (Linux Kernel 5.15.150)作为操作系统。[OpenWrt](https://openwrt.org/) 是一个面向嵌入式设备的 Linux 发行版，路由器是其所面向的设备之一。截至 2024 年 8 月份，Linux 的锐捷认证客户端都是只有 x86 平台的二进制，而小米路由器 3 Pro 使用了 MIPS 的芯片，不能直接使用这个二进制。

由于锐捷客户端没有开源，我没有办法对其做 MIPS 平台的移植。在考察了 GitHub 上面对于锐捷协议实现的客户端以后，我选择了 [Minieap](https://github.com/updateing/minieap) 这个客户端。

## 编译 Minieap

关于编译的详细操作方法，应当阅读项目的 [README](https://github.com/updateing/minieap#readme-ov-file)。

首先下载 [openwrt-sdk-23.05.4-ramips-mt7621_gcc-12.3.0_musl.Linux-x86_64.tar.xz](https://downloads.openwrt.org/releases/23.05.4/targets/ramips/mt7621/openwrt-sdk-23.05.4-ramips-mt7621_gcc-12.3.0_musl.Linux-x86_64.tar.xz)，这是在 x86_64 平台上交叉编译小米路由器 3 Pro上面软件的软件开发包(SDK)。


利用下面的脚本将 SDK 中的二进制加载到 shell 的环境变量中。
```bash
#!/bin/bash
export PATH=$PATH:/path/to/sdk/staging_dir/toolchain-mipsel_24kc_gcc-12.3.0_musl/bin
export STAGING_DIR=/path/to/sdk/staging_dir/toolchain-mipsel_24kc_gcc-12.3.0_musl/
```


将代码仓库克隆下来，修改位于根目录下的 `config.mk` 文件。注意到下面这几行:

```
# Example for cross-compiling
# CC := arm-brcm-linux-uclibcgnueabi-gcc
# ENABLE_ICONV := true
# CUSTOM_CFLAGS += -I/home/me/libiconv-1.14/include
# CUSTOM_LIBS += /home/me/arm/libiconv.a
# PLUGIN_MODULES += ifaddrs
# STATIC_BUILD := true
```
修改编译器相关的选项。
```
# Example for cross-compiling
CC := mipsel-openwrt-linux-musl-gcc-12.3.0
CXX := mipsel-openwrt-linux-musl-g++-12.3.0
# ENABLE_ICONV := true
# CUSTOM_CFLAGS += -I/home/me/libiconv-1.14/include
# CUSTOM_LIBS += /home/me/arm/libiconv.a
# PLUGIN_MODULES += ifaddrs
STATIC_BUILD := true
```

接着，在根目录下面执行 `make` 即可编译出可执行文件`minieap`。

```bash
$ file minieap
minieap: ELF 32-bit LSB executable, MIPS, MIPS32 rel2 version 1 (SYSV), statically linked, with debug_info, not stripped
```

## 在路由器上运行 Minieap 

我使用了由 [chenjunyu19](https://github.com/chenjunyu19) 提供的配置。在此对他表示感谢。

```
username=[NetID account name]
password=[NetID password]
nic=eno1
module=rjv3
fake-dns2=0.0.0.0
fake-serial=0
stage-timeout=0
max-retries=0
```

根据 chenjunyu 的说明，`stage-timeout` 和 `max-retries` 需要设置为 0 来防止自动退出。

## 将 Minieap 添加到开机自启动

OpenWrt 并不使用常见的 `systemd` 作为 `init` 进程，而是选用了 [BusyBox](https://www.busybox.net) 中的 `init`。在比较新的 OpenWrt 上，需要编写符合 [procd](https://openwrt.org/docs/techref/procd)规范的 `initscript` 。

有关 `procd` 以及 `initscripts` 的介绍可以看下面几个链接：

- https://openwrt.org/docs/guide-developer/procd-init-script-example
- https://openwrt.org/docs/techref/initscripts
- https://openwrt.org/docs/guide-developer/procd-init-scripts

下面是我所编写的简单 `initscript` ，仅作参考(文件名 `minieap`)。

```
#!/bin/sh /etc/rc.common
USE_PROCD=1
START=95
STOP=1
start_service() {
        procd_open_instance [minieap]
        procd_set_param command /usr/bin/minieap -c /etc/minieap.conf
        procd_close_instance
}
stop_service() {
        kill $(pgrep minieap)
}
```

将脚本放到指定位置(`/etc/init.d/` 文件夹下)后，通过以下方式来进行开机自启动设置，并启用服务。

```bash
$ service minieap enable

$ service minieap start

$ service minieap status
running
```