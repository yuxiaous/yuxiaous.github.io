---
layout: post
title:  "如何在没有显示器的情况下安装和使用 Raspberry Pi"
categories: blog
---



在boot中添加2个文件
ssh
wpa_supplicant.conf

```
country=US
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="your_real_wifi_ssid"
    scan_ssid=1
    psk="your_real_password"
    key_mgmt=WPA-PSK
}
```

添加ssh公钥 authorized_keys
