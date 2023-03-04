---
title: 初探 Wireguard
date: 2022-09-01 15:41:08
tags: [Docker, VPN]
---

# 緣起
因為最近整理 E-Hentai 上的 favorite 時發現有部分的收藏吃了版權砲，索性開始把有收藏過的本子和喜歡的畫師都抓一份下來，反正收本子也花不了多少硬碟空間。

看漫畫的部分我習慣使用 iOS 上的 [ComicGlass](https://apps.apple.com/tw/app/comicglass-comicreader/id363992049)，除了將檔案放在本機，同時也支援從 SMB 協定開檔案或是使用自家的 [MediaServer](http://comicglass.net/download/toolsdownload/)。雖然之前使用 MediaServer 的體驗很好，但它得跑在 Windows / Mac 上，要開一台 VM 跑 Windows 實在太浪費資源了，於是便考慮架個 VPN 連回家裡的 NAS 用 SMB 開漫畫來看了。

要架 VPN 原先第一想到的是 [OpenVPN](https://openvpn.net/)，但是在 RouterOS 上胡搞瞎搞一番以後連線總是有問題 (主要都是憑證問題)，在 RouterOS 上同時要使用帳號密碼 + 憑證的驗證方式實在麻煩，就在隨意翻翻 [Awesome-Selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted) 時想起了還有 Wireguard 這個看了很久但一直沒玩過的酷玩具，就在這次來試試看吧。

# 環境
本次使用的環境是架在 Proxmox VE 上的 Alpine Linux 3.14，使用的是 `alpine-virt-3.14.2-x86_64` 這個版本

目標是讓 VPN 可以使用一個自己的網段 `192.168.6.0/24` 來與原先的內網 `192.168.2.0/24` 通訊，同時可以連上 Internet

# 實作
恭喜你看完了前面的廢話，Wireguard 的實作實在是簡易到讓人懷疑他是不是 VPN 的程度，就讓我們快速地開始吧。

```shell
# 根據 Alpine Linux Wiki 進行安裝 
# https://wiki.alpinelinux.org/wiki/Configure_a_Wireguard_interface_(wg)

# 安裝 Wireguard 管理工具包
apk add wireguard-tools

# 載入 Wireguard 模組
modprobe wireguard

# 產生 private key 與 public key
# 這邊產生的 key 會做為後面連線認證之用
wg genkey | tee privatekey | wg pubkey > publickey

```

安裝完了 Wireguard 與產生 key，接下來就要寫 Wireguard 的設定檔 `/etc/wireguard/wg0.conf` (`wg0` 可以替換成自己想要的介面名稱)

這邊簡單分成兩個版本，一個是讓 VPN Client 存取內網時保持 VPN 網段的 IP 的，這個狀況要自己設定好 Routing 的部分。另一個則是做 NAT，讓 VPN Client 存取內網或 Internet 時都會使用 VPN Server 的 IP
```ini
# 做 NAT 的版本，如果你不熟網路設定就用這個
[Interface]

# Address 是 Wireguard 介面用的 ip
Address = 192.168.6.1/24 

# Wireguard server 所監聽的 port
ListenPort = 45340

# 在這邊貼上剛剛產生的 private key
PrivateKey = SG1nXk2+kAAKnMkL5aX3NSFPaGjf9SQI/wWwFj9l9U4= 

PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE;iptables -A FORWARD -o %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE;iptables -D FORWARD -o %i -j ACCEPT

DNS = 8.8.8.8
```

```ini
# 不做 NAT 的版本，要自己設定和其他網段的 static routing
[Interface]

# Address 是 Wireguard 介面用的 ip
Address = 192.168.6.1/24 

# Wireguard server 所監聽的 port
ListenPort = 45340

# 在這邊貼上剛剛產生的 private key
PrivateKey = SG1nXk2+kAAKnMkL5aX3NSFPaGjf9SQI/wWwFj9l9U4= 

PostUp = iptables -A FORWARD -i %i -j ACCEPT;iptables -A FORWARD -o %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -j ACCEPT;iptables -D FORWARD -o %i -j ACCEPT

DNS = 8.8.8.8
```

**如果採用不做 NAT 的版本的話記得要在你的 Router 上設定 static routing，把  `192.168.6.0/24` 送到你的 Wireguard server IP**

寫好設定檔以後，執行以下指令讓 Wireguard 跑起來
```shell
# 如果在上一步驟你不是用 wg0 做為介面名稱，這邊記得要改
wg-quick up wg0
```

如果要把 Wireguard 關掉則反過來是
```shell
wg-quick down wg0
```

到這邊 server 端的基本設定就完成了，接下來是有關 Client 的設定，在這邊示範的是 Wireguard iOS App

首先在 server 上的設定檔加上以下設定
```ini
[Peer]

# 允許這個 Client 使用的網段
AllowedIPs = 192.168.6.0/24

# 在 iOS App 上產生出來的 public key
PublicKey = QNZxoNgtUF/rFW+SKm+Qs+w+3ywhM+YcUh5GUkhnOWM=

# 每幾秒要檢查一次連線是不是還活著，一般來說 10 ~ 15 秒就夠了，預設是 30 秒
PersistentKeepalive = 10
```
接著在 iOS App 端設定連線

![](2021110401-WireGuard-iOS.png)

在 `Addresses` 裡面填入想要使用的內網 IP，注意要在 server 設定的 `AllowedIPs` 網段裡面
下面的 `Public key` 填入 server 的 public key
`Endpoint` 填入 server 的 ip 和 Wireguard 的 port
`Allowed IPs` 在這邊是要填入要經由 Wireguard VPN 連線的網段，如果你要全部流量都經由 VPN 出去的話就填 `0.0.0.0/0`

到這邊就設定完成，可以試著連線看看了。
連線成功的話 Client 端會顯示有流量在跑

![](2021110402-WireGuard-iOS.png)

server 端則可以用 `wg` 指令來觀察 Client 有沒有在動

![](2021110403-WireGuard-serer.png)

到這邊基本就會動了，後續再來看看穩定性和效能表現如何


# Reference
- [Alpine Linux Wiki](https://wiki.alpinelinux.org/wiki/Configure_a_Wireguard_interface_(wg))
- [wgredlong/WireGuard](https://github.com/wgredlong/WireGuard/blob/master/2.%E7%94%A8%20wg-quick%20%E8%B0%83%E7%94%A8%20wg0.conf%20%E7%AE%A1%E7%90%86%20WireGuard.md)
- [awesome-selfhosted](https://github.com/awesome-selfhosted)
