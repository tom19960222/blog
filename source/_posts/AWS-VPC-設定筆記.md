---
title: AWS VPC 設定筆記
date: 2022-09-02 19:38:02
tags: [AWS]
---

# 緣起
AWS VPC 在管理複雜的網路設置時確實是個強大的工具，對於我這種喜歡把東西分門別類放得很漂亮，就連機器也不放過的人來說規劃實作起來真的很爽，不過也因為設定實在太細了，一段時間沒做又因為專案需求需要實作時就會漏東漏西，那就來寫一篇筆記給未來的自己當作參考吧。

~~我才不會說是因為自己多犯了好幾個蠢，多 debug 了 3 小時才來寫筆記勒~~

# 情境
本次預計實作的架構如下：
1. VPC 使用網段 10.9.0.0/16
2. subnet 一共 6 個，每個都拿一段 /24，分為 ELB 用 2 個 + EC2 用 2 個 + RDS 用 2 個，每個都要在不同 AZ (為了每個服務都可以開 HA)
3. Security Group 一共 3 個，分別給 ELB (HTTP + HTTPS)、EC2 (HTTP + SSH)、RDS (PostgreSQL) 用

# 設定步驟
好啦，規劃好了就可以開始設定了

## 1. 新增 VPC
有需要 IPv6 的話記得在 `IPv6 CIDR block` 選 `Amazon-provided IPv6 CIDR block`，不然選 `No IPv6 CIDR block` 的話就是 IPv4 only 了

我個人習慣用 `專案名稱-環境名稱 (production / staging)` 來當 VPC 的名稱

## 2. 修改 VPC 的 DNS 相關設定
選擇單個 VPC 以後在 
- Actions -> `Edit DNS hostnames` 
- Actions -> `Edit DNS resolution` 

兩項裡面分別把功能啟用

從 [AWS 的文件](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.html) 來看，`DNS resolution` 的設定是用來開關此 VPC 裡面私有域名的解析 (如 `ip-10-9-1-123.ap-northeast-1.compute.internal` 這樣的 domain)，同時也影響 AWS 會不會幫你放一台 DNS Server 在你的 VPC 裡面

另外 `DNS hostnames` 的設定是用來開關在這個 VPC 裡面拿到 public ip 的機器會不會一起拿到一個解析到該 public ip 的 domain (如 `ec2-18-123-123-123.ap-northeast-1.compute.amazonaws.com` 這樣的 domain)

主要是為了使用 Elastic Beanstalk 來部署應用程式這邊的設定才一定要開，不過為了方便個人是都會開著

## 3. 新增 subnet
記得如果要做 HA 的話，同個服務內的每個 subnet 要放在不同的 AZ，不然假設一個 AZ 炸了你的服務還是照樣炸開。

當然如果你遇上了日本沉沒還是救不了你，不過反正最頭痛的也不會是你 (茶。

個人習慣用 `專案名稱-環境名稱-放的服務 (elb / ec2 / rds)-編號 (2位數字)` 來命名

### **特別注意** 
這次新踩到的坑，如果你有用 Elastic Beanstalk 來幫你部署 ELB，或是 ELB 的 target 是用 `instance` 而不是 `ip` 的話，要注意 ELB 所使用的 subnet 的 AZ 要和 EC2 subnet 的 AZ 一樣或是完全涵蓋
(ex: ELB 用 `ap-northeast-1a` + `ap-northeast-1b` + `ap-northeast-1d`，EC2 用 `ap-northeast-1a` + `ap-northeast-1d`)

不然在 Target Group 裡你的 target 註冊上去以後狀態會是 `unused`，說明顯示 `Target is in an Availability Zone that is not enabled for the load balancer`。實際連線看看的話當然是完全連不到。
![](2021091301-AWSELB.png)

## 4. 修改 subnet 設定
如果你希望這個 subnet 的機器都自動拿到一個 public ip，可以在 Actions -> `Modify auto-assign IP settings` 裡面打開他

## 5. 新增 Route Table
## 6. 新增 Internet Gateway 並 attach 到 VPC
## 7. 新增 default gateway
在剛剛新增的 Route Table 裡面新增 default gateway (Destination 為 `0.0.0.0/0`，target 為剛剛新增的 Internet Gateway)

如果你忘了新增又剛好像我一樣直接用了 Elastic Beanstalk 部署的話，那就會產生一個很詭異的情形是部署跑很久都不會成功，想 SSH 進 EC2 機器鳥都不鳥你，那就要想想是不是自己犯蠢了

## 8. 新增 Security Group 與規則
除了服務會用到的 port (如 HTTP/HTTPS) 以外，要注意允許其他 Security Group 來的連線 (ex: RDS 的 Security Group 要允許 EC2 的 Security Group 來的連線)

如果有直接連入 debug 之類的需求，我習慣會把公司對外的固定 IP 設定成允許直接連入，不過這部分就看你如何拿捏安全性和方便了

個人習慣用 `專案名稱-環境名稱-放的服務` 來命名

## 9. 新增允許自己連到自己的 Security Group 的規則
如果不允許的話在機器上用自己的 public ip 連線到自己會被防火牆擋下來，然後你就會一頭霧水。Elastic Beanstalk 之類的工具遇到這類問題尤其難 debug，他看起來就是莫名不會動

# 恭喜搞定
好了，設定大致上就到這邊結束，恭喜你搞定了網路設定，到了會用這種規模的應用程式，搞定網路以後還有更多問題等著你呢 (?
