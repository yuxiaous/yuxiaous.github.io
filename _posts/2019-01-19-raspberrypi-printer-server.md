---
layout: post
title:  "Raspberry Pi 打印服务"
categories: blog
---
平时喜欢捣鼓些小玩具，例如树莓派就有2个，一个3B，一个Zero W。可是树莓派真的是吃灰派呢，刚买来时很兴奋，装好系统没玩两天就丢在一边吃灰了。那台3B现在做了路由器，负责一个透明代理的业务，应了那句老话：“买前生产力，买后路由器”，哈哈哈。还有台 Zero W 怎么办呢？ Zero W 那么小，那么可爱，但很废柴啊。只有一个 Micro USB 接口，用的时候还要配 OTG 转接线。

终于有一天，需求到来了。我在某鱼二手市场上淘了一台老古董二手激光打印机，只要200块。主要用于平时打印些资料什么的，实际用下来效果还是很满意的。但是毕竟是老古董了，只有 USB 接入方式。每次打印都要抱着电脑跑去打印机旁插上 USB 线，这画面我不想看。还有手机呢？也要将资料传到电脑里，再抱着电脑过去打印吗？绝对不要！！

我需要的是一种可以将这个老古董打印机改造成无线打印机的方法，需要同时支持 Windows、Linux 以及在手机上提交打印任务。这时我的余光瞄向了身旁的 Raspberry Pi Zero W ，把你买回来也有2年多了，这2年没少照顾你。是时候到轮到你了，皮卡派！

> 事后经过测量，这个 Raspberry Pi Zero W 在工作时的功率只有 1W 左右，非常符合我的需求，可以 7x24 小时的连续工作。

接下来就开始详细说明整个打印服务的过程。

### 安装和配置 Raspberry Pi

安装和配置 Raspberry Pi 可以参考我之前的文章进行[《如何在没有显示器的情况下安装和使用 Raspberry Pi》]({% post_url 2019-01-18-startup-raspberrypi-without-screen %})。

因为是 Zero W，所以还需要事先准备一根 OTG 线。
![otg](/assets/2019-01-19-otg1.jpg)

插在 Raspberry Pi Zero W 弱小的身躯上。
![otg](/assets/2019-01-19-otg2.jpg)

### 安装和配置打印服务

这里我们用的是苹果家的开源打印服务 CUPS(Common UNIX Printing System) 。它有几个优点：

1. 采用 IPP 以加强网络打印功能；
2. 可自动检测网络打印机；
3. Web接口设置工具；
4. 支持PPD(PostScript Printer Description)打印机文件；
5. 支持大多数打印机使用。

通过 ssh 使用 pi 用户连接上 Raspberry Pi之后，输入以下命令进行安装：

``` shell
sudo apt-get update
sudo apt-get upgrade

# Install cups
sudo apt-get install cups

# Add more printer drivers
sudo apt-get install printer-driver-splix

sudo usermod -aG lpadmin pi

sudo cupsctl --remote-any

sudo /etc/init.d/cups restart
```

> 我二手淘来的打印机是施乐的 Xerox Phaser 3117 ，这款打印机因为太过时，CUPS 里没有驱动，经过好一番功夫，终于找到解决方法，就是安装 `printer-driver-splix` 这个驱动扩展。

最后访问 `https://192.168.1.xxx:631` （你的 Raspberry Pi 的 IP 地址）进行配置。

关于如何配置我这里就不重复写了，网上的教程非常多，而我这肯定也不是最详细的。

为大家推荐几个，随便选一个看看就行：

- [How to Add a Printer to Your Raspberry Pi (or Other Linux Computer)](https://www.howtogeek.com/169679/how-to-add-a-printer-to-your-raspberry-pi-or-other-linux-computer/)
- [Raspbian: How to add a printer on your Raspberry Pi? (CUPS)](https://raspberrytips.com/install-printer-raspberry-pi/)

7x24 小时全年无休工作中的树莓派君
![img](/assets/2019-01-19-rsp.jpg)

### 安装更多打印机驱动

在寻找我这台老古董打印机驱动的时候，找到了这么一个打印机驱动网站 [http://foo2hbpl.rkkda.com/](http://foo2hbpl.rkkda.com/) 。虽然里面没有我这款打印机的驱动，但或许能用在其他老古董打印机上，所以在这里先记录一下。
