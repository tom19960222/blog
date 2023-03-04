---
title: OpenVPN 懶人安裝法
date: 2022-09-03 01:37:31
tags: [VPN, Docker]
---

本範例使用 Ubuntu Server 20.04 

安裝大略分為下列幾個步驟：
1. 安裝 Ubuntu Server
2. 安裝 Docker 
3. 安裝與設定 OpenVPN

# 安裝 Ubuntu Server
網路上有一堆教學，其實一直下一步基本上也是沒啥問題啦 (?

# 安裝 Docker
執行 `curl -sSL https://get.docker.com | sudo sh` 讓腳本幫你搞定安裝程序

安裝完成以後執行 `sudo usermod -aG docker <你的使用者名稱>` ，例如 `sudo usermod -aG docker ikaros` 這樣

這會給你的帳號使用 Docker 的權限，如果沒有做這行就只有最高權限的 `root` 帳號可以使用 Docker

做完以後記得登入再重新登入，才會套用新的權限設定 (把連線視窗關掉再連一次就可以)

# 安裝與設定 OpenVPN
這次使用人家打包好的 OpenVPN，設定起來非常簡便，詳細資料可以參考 [連結](https://hub.docker.com/r/kylemanna/openvpn)

1. 先將存放資料的 Docker volume 名稱寫入環境變數，方便後面使用
執行 `echo 'export OVPN_DATA="ovpn-data-example"' >> ~/.bashrc`
接著執行 `source ~/.bashrc` 重新載入環境變數

2. 建立存放 OpenVPN 資料用的 Docker volume
執行 `docker volume create --name $OVPN_DATA`

3. 執行 OpenVPN 的初始設定腳本
執行 `docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://VPN.SERVERNAME.COM` ，記得把後面的 `VPN.SERVERNAME.COM` 換成你自己的 domain

4. 執行 OpenVPN 的初始設定腳本 趴兔
執行 `docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki`

在跑腳本過程中會出現 `Enter New CA Key Passphrase:` 的字樣，是要你設定一組憑證的密碼，務必牢記，在新增使用者時會用到

腳本詢問 `Common Name` 時可以直接按 Enter 使用預設值

在遇到 `Enter pass phrase for /etc/openvpn/pki/private/ca.key` 時就是要你輸入剛剛憑證的密碼了

這個步驟完整做完大約會長這樣：
![Console 畫面截圖](2022032601-Openvpn-install-initpki.png)

5. 啟動 OpenVPN 伺服器
執行 `docker run -v $OVPN_DATA:/etc/openvpn -d -p 1194:1194/udp --cap-add=NET_ADMIN --restart=unless-stopped --name openvpn kylemanna/openvpn`

因為加了 `--restart=unless-stopped` 參數，之後重新開機也會自動執行

6. 新增 OpenVPN 使用者
執行 `docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full CLIENTNAME nopass`，記得將 `CLIENTNAME` 換成你想要的使用者名稱

7. 取得使用者用的 OpenVPN 設定檔
執行 `docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient CLIENTNAME > CLIENTNAME.ovpn`，記得將兩個 `CLIENTNAME` 都換成剛剛新增 OpenVPN 使用者的名稱
接著執行 `cat CLIENTNAME.ovpn` 印出設定檔的內容，全部複製下來 copy 到自己電腦就可以，或是用 `scp` 之類的工具 copy 到自己電腦也可以

執行完以上步驟以後就可以用 OpenVPN Client 之類的工具連線了

產生出來的設定檔應該會像下面那樣：
```

client
nobind
dev tun
remote-cert-tls server

remote 你的網址 1194 udp

<key>
-----BEGIN PRIVATE KEY-----
長長的 key ...
-----END PRIVATE KEY-----
</key>
<cert>
-----BEGIN CERTIFICATE-----
長長的憑證 ...
-----END CERTIFICATE-----
</cert>
<ca>
-----BEGIN CERTIFICATE-----
長長的憑證 ...
-----END CERTIFICATE-----
</ca>
key-direction 1
<tls-auth>
#
# 2048 bit OpenVPN static key
#
-----BEGIN OpenVPN Static key V1-----
長長的 key ...
-----END OpenVPN Static key V1-----
</tls-auth>

redirect-gateway def1
```

如果機器前面有防火牆之類的話要記得開 UDP 1194 port，並將 UDP 1194 port 轉發到開 OpenVPN 的機器

# 一些額外指令
- 關閉 OpenVPN Server
`docker stop openvpn`
- 重啟 OpenVPN Server
`docker restart openvpn`
- 檢視 OpenVPN Server 的記錄檔
`docker logs -f openvpn`