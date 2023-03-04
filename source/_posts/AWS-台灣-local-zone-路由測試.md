---
title: AWS 台灣 local zone 路由測試
date: 2022-10-15 16:49:16
tags:
---

其實這篇的內容早在 10/6 就已經測完了，只是一直懶得動筆紀錄一下 Orz

先前就一直有聽說 AWS 有要開台灣區的意願，終於在 10/6 看到 gslin 大大貼的 [AWS 的台北區 (Local Zone) 開了](https://blog.gslin.org/archives/2022/10/06/10903/aws-%e7%9a%84%e5%8f%b0%e5%8c%97%e5%8d%80-local-zone-%e9%96%8b%e4%ba%86/)，那當然要趕快來玩玩看啦

經過測試確實和 gslin 的另一篇 [目前 AWS 台北區只能開 *.2xlarge 的機器](https://blog.gslin.org/archives/2022/10/06/10905/%e7%9b%ae%e5%89%8d-aws-%e5%8f%b0%e5%8c%97%e5%8d%80%e5%8f%aa%e8%83%bd%e9%96%8b-2xlarge-%e7%9a%84%e6%a9%9f%e5%99%a8/) 一樣，目前只能開 2xlarge 的機器還不能開 spot instance，所以就開了一台 `m5.2xlarge` 來玩玩看，價格是 $0.62 USD/hour

因為公司有網路電話相關的業務放在 Cloud 上，SIP protocol 對延遲和掉包又比較敏感，當初也是因為這點才棄 AWS 改用 GCP 台灣區。這次就來簡單測試一下島內和國際路由吧！

本次開出來的機器 IP 是 `15.220.83.12`，從 [bgp.he.net](https://bgp.he.net/ip/15.220.83.12) 來看，IP 段應該都在 `15.220.80.0/20` 內。

# 測試結果

分成島內家用/商用固網、商業服務、國際和自己在玩的自有 IP 相關的測試

## 島內家用/商用固網

### 中華非固定制 500M/250M

![到中華非固定制](到中華非固定制.png)
![從中華非固定制](從中華非固定制.png)

### 中華固定制 500M/250M

![到中華固定制](到中華固定制.png)
![從中華固定制](從中華固定制.png)

固定制和非固定制的表現差不多，可以很明顯的看出這次 local zone 的機房是[中華板橋 IDC](https://goo.gl/maps/ExigwJCmFcdv87Qq9)

### 遠傳固定制 off-net 100M/100M

![到遠傳 off-net](到遠傳off-net.png)
![從遠傳 off-net](從遠傳off-net.png)

### 板橋大大寬頻 亞太出口 360M/120M

![到板橋大大寬頻](到板橋大大寬頻.png)
![從板橋大大寬頻](從板橋大大寬頻.png)

個人猜測 AWS 和中華、遠傳都有 PNI，和亞太則是透過 `EBIX` peer。不過 `EBIX` 也是亞太自家的，某種意義上也是和亞太直接接了條線了啦 (?)

## 商業服務

### Cloudflare DNS

毫不意外的直連

~~只有中華才會死不跟你在台灣  peer 啦~~

![到cloudflare-dns](到cloudflare-dns.png)

### 遠傳 SIP gateway

比較意外的是竟然是走中華到遠傳，不過延遲看起來沒啥問題，真的有需要或是有差的話不知道可不可以請他們調整路由?

![到遠傳SIP-gateway](到遠傳SIP-gateway.png)

## 國際

### Vultr 日本

當時忘了多測一些服務了，只記得測了 Vultr 日本區，就加減看看吧

![到Vultr日本](到Vultr日本.png)
![從Vultr日本](從Vultr日本.png)

看起來到日本是走 `PCCW`，算是優質的 T1 ISP，~~總比 174 好了~~

## 自有 IP 相關

順便幫忙宣傳一下，如果對擁有一段自己的 IP 有興趣，或是想要跟 AWS、Cloudflare、HE 接 direct peering 的，可以支持一下[海豹 (@seadog007)](https://seadog007.me/) 的 [學生聯合交換中心 (STUIX)](https://stuix.io/) 計畫，群裡也有 LIR 大佬們可以幫忙申請 ASN 和 IP，如有需要歡迎聯繫。

### STUIX VPS

![到STUIX-VPS](到STUIX-VPS.png)
![從STUIX-VPS](從STUIX-VPS.png)

多虧 AWS 有接入 STUIX，基本上就是實體線路直接對接的延遲

### 自家 IP (台北)

![到家裡-public-ip](到家裡-public-ip.png)

我自己沒有跟 AWS direct peer，不過感謝海豹幫我 transit 了。

~~雖然我自己的內網路由很明顯噴到國外再噴回來了，有空再來修~~

### 自家 IP (日本)

![到Vultr日本public-ip](到Vultr日本public-ip.png)
![從Vultr日本public-ip](從Vultr日本public-ip.png)

去程一樣是走 STUIX 到我的自有 IP，~~回程則是走 Vultr -> JPIX 到 AWS~~

發現回程因為我自己網路炸了所以根本是走 Vultr 的 IP 出去的，~~一樣有空再修吧，看起來又可以再來重蓋網路了~~

# 總結

目前整體來說除了只能開 `2xlarge` 的機器以外沒啥好挑剔的，坐等台灣區從 local zone 晉升到 region 的那天 ヽ（。∀°）ノ