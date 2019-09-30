---
layout: post
title:  "如何在没有显示器的情况下安装和使用 Raspberry Pi"
categories: blog
---
树莓派是很有意思的玩具，手里有2台，一台是 3B，一台是 Zero W。

刚买回 3B 的时候还费尽心思搞来键盘、鼠标、一根网线和一根 HDMI 线接在了电视上（我没有显示器鸭 囧）。捣鼓了一会那简陋的界面，并配置好 wifi 和 ssh ，就取下 HDMI 线和网线，可以放飞了。因为我平时通过 ssh 连接上去操作就够了，不需要用到图形界面。后来觉得图像界面确实没用，就又改装了 Raspbian Lite 。

直到我买了 Zero W 来玩，这家伙简直让人爱不释手。那小巧的身躯，那优美的弧线，我能盘上一整天。但是小虽小，但问题来了，上面那个 Micro HDMI 要咋整？没有 usb 有线网卡咋整？为了只装一次的系统就去买吗？用完之后闲置下来真的好吗？而且我现在就想用啊怎么办？只能靠万能的谷歌了。

搜了一圈后得出一个结论：可行！但是网上的资料非常零散。经过一阵折腾，终于在没有网线没有显示器的情况下成功为这台 Zero W 配置好了 wifi 和 ssh。

### 安装 OS

系统选择官方出品的 [Raspbian Lite](https://downloads.raspberrypi.org/raspbian_lite_latest) 。将 SD 卡接在读卡器上插入电脑，使用烧录工具 [balenaEtcher](https://www.balena.io/etcher/) 将下载的系统镜像烧录到 SD 卡上即可。不展开说了。

重新插入 SD 卡，就可以在 Finder 或 文件资源管理器 里看见一个叫 `boot` 的盘符，这个我们下面会用到。

### 配置 ssh

因为 Raspbian 默认是将 ssh 关闭的，想要开启它需要一个 flag 。而这个 flag 不是某某配置文件中的一行内容，而是一个文件。

- 在 `boot` 中新建一个名为 `ssh` 的文件

这个文件不需要填写任何内容，只需要有这么一个文件存在就可以了。Raspbian 系统初次启动后，会开启 ssh 服务，并删除这个文件。

### 配置 wifi

- 在 `boot` 中新建一个名为 `wpa_supplicant.conf` 的文件
- 在文件 `wpa_supplicant.conf` 中写入如下内容：

``` shell
country=US
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="YOUR_WIFI_SSID"
    scan_ssid=1
    psk="YOUR_WIFI_PASSWORD"
    key_mgmt=WPA-PSK
}
```

以上内容将 `YOUR_WIFI_SSID` 替换为 wifi 的名称，`YOUR_WIFI_PASSWORD` 替换为 wifi 的密码。

### 连接树莓派

- 将 SD 卡插入树莓派，通电并等待一会，直到 LED 指示灯从闪烁转为常亮。
- 进入你的路由器，找到名为 `raspberrypi` 的树莓派，得到它的 ip 地址。
- 通过 ssh 连接树莓派，用户名是 `pi`，密码是 `raspberry`。
- 最后将自己电脑的 ssh 公钥添加到用户 `pi` 的 `~/.ssh/authorized_keys` 中。
