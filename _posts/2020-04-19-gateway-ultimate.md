---
layout: post
title:  "Openwrt 透明网关（终极篇）"
categories: blog
---

不得了了，终于，家里的网络换上了Unifi全。。。家。。。桶。。。

在看了一篇网络上的文章后，刷新了我对软路由和硬路由的观念。软路由的NAT转发全部靠CPU计算，所以要提升性能就变成了CPU性能的比拼，而换更好的CPU感觉是个无底洞，性价比并不高，终究是不想陷入这样的泥潭的。而好一点的硬路由是有硬件NAT的，转发效率相较于软路由几乎是提升了一个量级的。所以最终的方案似乎又回到了原点——将软路由换回硬路由。不过换回硬路由后透明网关就没有了，怎么办呢？这么好用的东西难道说弃就弃吗？当然不会，软路由仍然是需要的，不过它将作为旁路由而存在，透明网关的事依旧是软路由负责的。网络拓扑图如下：

![img](/assets/2020-04-19-gateway-ultimate/topology.png)

没错，旁路由是接在交换机上的，而且更惊喜的是只需要一个网口就行了。

接下来详细讲一下终极版透明网关的搭建。

## 设备

家里的网络设备在之前就已经用上了Unifi的交换机和AP，现在唯独缺的就是一台路由器了。于是我在某鱼上以不到700元的价格买了台二手USG（UniFi Security Gateway），终于凑齐了Unifi全家桶。这台USG路由器的芯片是mips，处理性能可以说是非常弱了，但它有强大的硬件NAT芯片，数据包转发性能可达每秒百万级别。而如J1900这种CPU的软路由，数据包转发性能只有几百K。所以专用设备做专业的事，仅仅做个路由器，USG实际是比软路由更胜任的。

![img](/assets/2020-04-19-gateway-ultimate/usg.png)

USG的CPU性能非常羸弱，干不了体力活，所以网络里的所有脏活累活就得交给软路由来处理了。家里的软路由同样是某鱼二手淘来的，400元买因特尔NUC DCCP847DYE，CPU是Intel  i3-3217U，4核1.80GHz。这是台非常小巧的主机，不过只有一个网口，之前做主路由时外接了一个USB网卡做WAN口。

![img](/assets/2020-04-19-gateway-ultimate/nuc.jpg)

## 搭建

接下来就是如何配置一个终极版家庭网络了，需求如下：
- 普通设备在主网内，直接通过主路由访问互联网
- 独立的SSID用于接入透明网关，连接此SSID的设备可以免费留学
- 重启或调整旁路由时不影响主网内的设备正常访问网络
- 隔离IoT设备，Iot设备只能访问互联网，不能访问主网内服务

### 搭建主网：

主网的搭建如同普通的的家用局域网，配置WAN为PPPoE拨号，配置LAN的网段并开启DHCP，配置WIFI的SSID和密码。全部配置好之后，主网就搭建好了，主网是一个非常普通的家庭局域网。

![img](/assets/2020-04-19-gateway-ultimate/main-wan.png)

![img](/assets/2020-04-19-gateway-ultimate/main-lan.png)

![img](/assets/2020-04-19-gateway-ultimate/main-wifi.png)

### 安装路由系统：

为了充分利用小主机的性能，我安装了PVE（Proxmox VE）作为其基础操作系统。PVE是一套虚拟环境，可以在其上安装多个操作系统。我暂时安装了2个系统，一个是Ubuntu用于跑Docker和其他需要在Linux环境下运行的软件，另一个就是路由系统Openwrt。

![img](/assets/2020-04-19-gateway-ultimate/pve.png)

安装Openwrt的过程有些坑，也有些骚（嘿嘿），所以这里记录以下。

1、上传ubuntu桌面版镜像文件（为什么是桌面版，后面会提到）。ubuntu镜像文件从ubuntu官网[下载](https://ubuntu.com/download/desktop)。

![img](/assets/2020-04-19-gateway-ultimate/install-openwrt-1.png)

2、右键点击pve节点，创建虚拟机，用于安装Openwrt。

![img](/assets/2020-04-19-gateway-ultimate/install-openwrt-2.png)

3、为虚拟机起一个名字openwrt19，起这个名字是因为当前Openwrt的最新版本是19.XX.X。

![img](/assets/2020-04-19-gateway-ultimate/install-openwrt-3.png)

4、安装系统选择之前上传的ubuntu桌面版（喂！不是要装openwrt吗？请继续向下看）

![img](/assets/2020-04-19-gateway-ultimate/install-openwrt-4.png)

5、硬盘大小配置1GB就可以了

![img](/assets/2020-04-19-gateway-ultimate/install-openwrt-5.png)

6、CPU选大一些，因为要运行ubuntu桌面版。

![img](/assets/2020-04-19-gateway-ultimate/install-openwrt-6.png)

7、内存尽量填大一点，因为要运行ubuntu桌面版（我真不是在逗你，真的是要装openwrt啊）

![img](/assets/2020-04-19-gateway-ultimate/install-openwrt-7.png)

8、最后确认一下内容，没问题就点Finish。

![img](/assets/2020-04-19-gateway-ultimate/install-openwrt-8.png)

9、启动后通过Console可以连接到虚拟机，等待运行ubuntu的安装引导程序停在下面的界面。选择“Try Ubuntu”，切记。

![img](/assets/2020-04-19-gateway-ultimate/install-openwrt-9.png)

10、进入桌面后开启Terminal终端。这里对为什么使用Ubuntu桌面版做一下说明。Try Ubuntu时，Ubuntu安装程序会将系统装载进内存中运行，此时硬盘是不使用的。接下来使用Terminal下载Openwrt镜像，并使用dd命令安装至硬盘中。

![img](/assets/2020-04-19-gateway-ultimate/install-openwrt-10.png)

11、安装Openwrt系统，以下我安装的是19.07.2这个版本的Openwrt。

> 小知识：Openwrt的镜像文件有2种文件系统，一种是ext4，一种是squashfs。ext4是普通的linux文件系统；squashfs是一种只读文件系统，这种文件系统会在启动后会copy一份内容在磁盘中，然后运行copy后的内容。squashfs文件系统虽然会更消耗存储空间，但可以方便的恢复初始状态，而ext4修改后是无法恢复初始状态的，各有利弊吧。

```bash
# 下载Openwrt镜像文件，
wget https://downloads.openwrt.org/releases/19.07.2/targets/x86/64/openwrt-19.07.2-x86-64-combined-squashfs.img.gz
# 解压.gz文件得到.img文件
gzip -d openwrt-19.07.2-x86-64-combined-squashfs.img.gz
# 将.img文件安装至硬盘sda
sudo dd if=openwrt-19.07.2-x86-64-combined-squashfs.img of=/dev/sda
# 检查安装情况
sudo fdisk -l
```

![img](/assets/2020-04-19-gateway-ultimate/install-openwrt-11.png)

12、安装完成后，关闭虚拟机。去除启动ISO文件，并将CPU和内存调节到适合Openwrt运行的数值。

![img](/assets/2020-04-19-gateway-ultimate/install-openwrt-12.png)

至此，Openwrt就以及安装完成了。利用Try Ubuntu来安装Openwrt也是我不久前才学会的骚操作，非常方便。

### 配置透明网关：

配置原理如下：
- 配置虚拟局域网PROXY，网段为192.168.40.X/24，VLAN为40。
- 该虚拟局域网包含主路由（192.168.40.1）和旁路由（192.168.40.2），并通过SSID为SWEET的无线信号接入。
- 主路由关闭DHCP，由旁路由负责分配该网段中的IP地址。
- 设置旁路由的网关为主路由。

用网设备通过无线网SWEET接入PROXY网络后，由于是旁路由负责DHCP，所以在获取到IP地址的同时也将网关指向了旁路由。旁路由的网关则指向主路由，主路由负责连接互联网。所以网络请求从用网设备发出后，先发往旁路由，旁路将网络请求“包装”后再发往主路由，最后主路由将网络请求发送至互联网。

![img](/assets/2020-04-19-gateway-ultimate/proxy.png)

1、创建透明网关的专属网络PROXY，启动VLAN并设置ID为40。网关设置为192.168.40.1/24，即这个虚拟局域网的网关是192.168.40.1，子网掩码是255.255.255.0。需重点注意的配置：**关闭DHCP**。

![img](/assets/2020-04-19-gateway-ultimate/config-proxy-1.png)

2、创建无线网络，SSID为SWEET，开启VLAN并设置ID为40，与PROXY网络的VLAN ID相同。如此配置后，通过这个SSID接入的设备就会自动接入PROXY网络。

![img](/assets/2020-04-19-gateway-ultimate/config-proxy-2.png)

3、设置PVE中openwrt的网卡VLAN Tag为40。双击Network Device即可打开对话框进行设置。如此设置后，该虚拟机将自动接入VLAN40这个虚拟局域网。

![img](/assets/2020-04-19-gateway-ultimate/config-proxy-3.png)

4、启动openwrt虚拟机，通过Console连接至虚拟机，修改openwrt的lan配置。

```bash
# 修改网络配置文件
vim /etc/config/network

# 修改接口lan的静态地址为192.168.40.2，与此网段的真正网关192.168.40.1相区别，避免ip冲突。
# 添加 gateway、broadcast、list dns等字段，参考如下
config interface 'lan'
    ...
    option ipaddr '192.168.40.2'
    option gateway '192.168.40.1'
    option broadcast '192.168.40.255'
    list dns '192.168.40.1'
    ...

# 保存配置文件
:wq

# 重启系统
reboot
```

![img](/assets/2020-04-19-gateway-ultimate/config-proxy-4.png)

5、访问192.168.40.2已经可以看到Openwrt的Luci界面了。

![img](/assets/2020-04-19-gateway-ultimate/config-proxy-5.png)

透明网关已经配置完成。通过无线信号SWEET接入虚拟网络PROXY，查看网络信息，可以看到获取到的IP地址是192.168.40.X，网关地址是192.168.40.2，说明已经配置成功了。此时可以正常访问网站，但还不能留学，因为还没“包装”网络请求。

### 安装V2Ray：

在openwrt中，我使用的是Github上[kuoruan](https://github.com/kuoruan)所开发的[openwrt-v2ray](https://github.com/kuoruan/openwrt-v2ray)和[luci-app-v2ray](https://github.com/kuoruan/luci-app-v2ray)。使用Console连接至Openwrt虚拟机，安装以上2个软件。

```bash
# 更新软件库并升级所有软件
opkg update
opkg list-upgradable | cut -f 1 -d ' ' | xargs opkg upgrade

# 添加kuoruan的公钥
wget -O kuoruan-public.key http://openwrt.kuoruan.net/packages/public.key
opkg-key add kuoruan-public.key
# 添加kuoruan的软件库
echo "src/gz kuoruan_packages http://openwrt.kuoruan.net/packages/releases/$(. /etc/openwrt_release ; echo $DISTRIB_ARCH)" >> /etc/opkg/customfeeds.conf
echo "src/gz kuoruan_universal http://openwrt.kuoruan.net/packages/releases/all" >> /etc/opkg/customfeeds.conf

# 更新软件库
opkg update
# 解决安装和运行kuoruan版V2Ray会出现的问题
opkg remove dnsmasq && opkg install dnsmasq-full
opkg install luci-compat
# 安装openwrt-v2ray和luci-app-v2ray
opkg install v2ray-core
opkg install luci-app-v2ray
```

### 配置V2Ray：

1、在Openwrt的Luci中打开Services>V2Ray开始配置。可以参考V2Ray的官方[配置文档](https://www.v2ray.com/chapter_02/)进行配置。

![img](/assets/2020-04-19-gateway-ultimate/config-v2ray-1.png)

2、配置Inbound

![img](/assets/2020-04-19-gateway-ultimate/config-v2ray-2.png)

3、配置Outbound

![img](/assets/2020-04-19-gateway-ultimate/config-v2ray-3.png)

4、在“Global Settings”中勾选添加到Inbounds和Outbounds。

![img](/assets/2020-04-19-gateway-ultimate/config-v2ray-4.png)

5、开启透明代理，将转发端口设置为dokodemo_door的端口。

![img](/assets/2020-04-19-gateway-ultimate/config-v2ray-5.png)

6、最后保存并启动V2Ray，如果状态显示为Running，说明已经正常工作了。

![img](/assets/2020-04-19-gateway-ultimate/config-v2ray-6.png)

## 总结：

此套终极版透明网关方案的优点在于：
- 主网采用全套Unifi设备，商用设备高效稳定。
- 普通上网不依赖旁路由，即使旁路由出了问题也不会影响普通设备。
- 透明网关拥有独立的VLAN，数据与主网是隔离的。
- 通过PVE充分利用旁路由设备的剩余性能。
