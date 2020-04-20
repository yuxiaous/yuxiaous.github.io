---
layout: post
title:  "Openwrt 透明网关（改进版）"
categories: blog
---

之前用一台旧的 Openwrt 无线路由器作为二级路由器，通过连接这台无线路由器来实现一个透明网关的效果，其实一直用得还不错。

今天突发奇想，琢磨着能否将 V2Ray 安装在主路由上，并配置两个 VLAN。一个 VLAN 用作家庭网络，不走代理；另一个 VLAN 则将所有流量导向 V2Ray，同样可以达到透明网关的效果。这样不仅可以减少一台用电设备，让那台旧路由器退休；还有一个好处就是可以让代理地址变为 `192.168.1.1`，方便好记。由于我家当前使用群晖 NAS 作为主软路由，CPU 为 Intel Celeron J3455，所以在路由器上跑个 V2Ray 问题不算不大。

如果不知道如何在群晖 NAS 上搭建软路由，我之后会另写一篇文章说明，这里不展开了。


### 一、配置VLAN

VLAN（Virtual Local Area Network）虚拟局域网，是一种构建区域网络交换的网络技术，作用是在逻辑上分隔局域网。虽然在网络布线上并没有组成两个网络，一旦配置了VLAN就形同在存在两个相隔的局域网。

我这里使用VLAN的目的是将上网和留学分隔开，使用两个不同的 SSID 分别接入不同的 VLAN。在留学 VLAN 中搭建透明网关，实现随时随地学英语的良好氛围。

家中的硬件部分使用了 Unifi 的设备，包括一台8口POE交换机、一台吸顶AP和一台墙面AP，所以这部分的配置使用的是Unifi的控制器程序。

以下我将留学 VLAN 的 ID 设置为 `20`。

1、配置 SSID：

创建新的无线网络，并设置其VLAN。

![img](/assets/2019-09-27-vlan-wifi.png)

2、配置 Networks：

创建新的网络，并定义VLAN。

![img](/assets/2019-09-27-vlan-networks.png)

3、配置路由器

在路由器中新建 `Interface`，并配置网段为 `192.168.20.1/24`。

![img](/assets/2019-09-27-vlan-interface1.png)

物理网卡配置为 `eth0.20`，表示在物理网卡eth0上创建ID为20的VLAN。因为都是遵循 `IEEE 802.1Q` 协议，所以可以直接识别由交换机发来的数据包。这样就能自动将相同VLAN的无线网络接入这个Interface了。

![img](/assets/2019-09-27-vlan-interface2.png)

最后新建防火墙Zone，并隔断此Zone与其他Zone的转发。为什么要隔断？后面会提到。

![img](/assets/2019-09-27-vlan-firewall1.png)
![img](/assets/2019-09-27-vlan-firewall2.png)


### 二、安装V2Ray

1、下载安装包

为了方便安装配置，我使用了Github上 [kuoruan](https://github.com/kuoruan) 专为openwrt制作的 [openwrt-v2ray](https://github.com/kuoruan/openwrt-v2ray) 和 [luci-app-v2ray](https://github.com/kuoruan/luci-app-v2ray) 。可以在 releases 中直接下载 ipk 进行安装，也可以添加到opkg仓库使用install命令安装。我选择了直接下载ipk包进行安装。

```bash
# 连接路由器
ssh root@192.168.1.1
# 下载v2ray本体
wget https://github.com/kuoruan/openwrt-v2ray/releases/download/{version}/v2ray-core_{version}_x86_64.ipk
# 下载luci界面
wget https://github.com/kuoruan/luci-app-v2ray/releases/download/{version}/luci-app-v2ray_{version}_all.ipk
```

2、安装

```bash
opkg update
opkg install v2ray-core*.ipk
opkg install luci-app-v2ray*.ipk
```

3、配置

安装成功后，在 `Services` 中会出现 `V2Ray` 这一项，点进去后就可以对V2Ray进行配置了。

我尝试着用了一下使用luci进行配置，发现比较麻烦。索性直接使用配置文件进行配置，只需在 `Config file` 写入配置文件的路径即可，我将配置文件放在了 `/etc/v2ray/config.json`，如下图。

![img](/assets/2019-09-27-v2ray.png)

接下来是编写配置文件，V2Ray配置文件的编写可以参考[官方文档](https://www.v2ray.com/chapter_02/)。

我在配置文件中同时配置了 3 种 inbounds，分别是 ：

- socks：端口1080，用于接受socks代理
- http：端口8080，用于接受http代理
- dokodemo-door：端口3456，用于接受TCP流量转发

```json
  "inbounds": [
    {
      "protocol": "socks",
      "listen": "0.0.0.0",
      "port": 1080,
      "settings": {
        "auth": "noauth"
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http","tls"]
      }
    },
    {
      "protocol": "http",
      "listen": "0.0.0.0",
      "port": 8080,
      "settings": {
        "allowTransparent": true
      },
      "sniffing": {
        "enabled": false
      }
    },
    {
      "protocol": "dokodemo-door",
      "listen": "0.0.0.0",
      "port": 3456,
      "settings": {
        "followRedirect": true,
        "network": "tcp,udp"
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      }
    }
  ]
```

outbounds 则配置了自己在vps上搭建的服务器信息。

```json
"outbounds": [
    {
      "tag": "vps",
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "{your domain}",
            "port": 443,
            "users": [
              { "id": "{udid}", "alterId": 64 }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "wsSettings": {
          "path": "/ray"
        }
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    },
    {
      "tag": "block",
      "protocol": "blackhole",
      "settings": {}
    }
  ]
```

保存并生效后先做一下测试，在浏览器的代理插件中配置代理地址 `192.168.1.1`，端口 `1080`，类型是 `socks5`。访问一下google如果可以正常访问，则说明V2Ray配置成功。

### 三、流量转发

> 在上一篇关于使用闲置路由器配置透明网关的[文章]({% post_url 2019-08-10-gateway %})中，我仿照V2Ray官网的资料使用iptables进行流量转发。不熟悉iptables的留学生们恐怕会比较为难。在我不断搜寻后，终于找到了不使用iptables的配置的方法（[传送门](https://openwrt.org/zh-cn/doc/uci/firewall?s[]=port&s[]=forwards#%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86%E8%A7%84%E5%88%99_%E5%90%8C%E4%B8%80%E4%B8%BB%E6%9C%BA)）。只需要配置一下端口转发就可以实现的功能，实在是再好不过了。

上面第一部分提到将留学VLAN的防火墙Zone设置成与其他Zone完全隔断，是因为希望这个VLAN中的流量全都经由V2Ray处理后再转发出去。

配置流量转发，可以通过编辑 `/etc/config/firewall` 进行配置，也可以通过 luci 界面进行配置，两者是完全等价的。我研究了两者的关系后，最后采用在 luci 中添加配置。

进入 `Network` -> `Firewall` -> `Port Forwards` ，新建端口转发。
![img](/assets/2019-09-27-forword1.png)

添加、保存并生效，可以看到在 Port Forwards 列表中新增了一行内容。这样就把所有留学VLAN中的流量全部转发到V2Ray的入站端口去了。

![img](/assets/2019-09-27-forward2.png)

用手机快试试，如果不出岔子的话，以后就能在家里快乐得学习英语了。
