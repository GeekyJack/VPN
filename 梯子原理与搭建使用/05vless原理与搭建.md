视频教程：[点击进入观看](https://bulianglin.com/g/aHR0cHM6Ly95b3V0dS5iZS83R0hoOTFBWUFtTQ)
上节我们详细介绍了vmess协议的通信流程以及搭建`vmess+ws+tls+web`的节点，达到了最大的伪装效果
但是上节我说过，vmess他没有类似trojan可以自带伪装，所以要给vmess协议做伪装的话还得单独搭建一个web服务器（nginx）来接收不是vmess流量的数据并对其进行分流实现伪装，而且vmess对于现在都套tls的情况下还是会对协议头部数据进行加密，并且还得和系统时间对应,显得有些冗余
针对这种情况，`vless`协议应运而生，他的出现就是为了解决vmess上述问题，不会进行额外加密也无需校对时间，可以看成是轻量化的vmess，这节我们就来讲讲vless协议，并使用`xray`内核来搭建vless节点。



```shell
#关闭防火墙：
ufw disable

#xray官方一键安装脚本：
bash -c "$(curl -L github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install -u root
#启动Xray：
systemctl start xray.service 
#重启Xray：
systemctl restart xray.service 
#Xray状态：
systemctl status xray.service
```

**申请证书：**

```shell
 #安装acme：
 curl https://get.acme.sh| sh
 #安装socat：
 apt install socat
 #添加软链接：
 ln -s  /root/.acme.sh/acme.sh /usr/local/bin/acme.sh
 #切换CA机构：
 acme.sh --set-default-ca --server letsencrypt
 #申请证书： 
 acme.sh  --issue -d 替换为你的域名  --standalone -k ec-256
 #安装证书： 
 acme.sh --installcert -d 替换为你的域名 --ecc  --key-file   /usr/local/etc/xray/server.key   --fullchain-file /usr/local/etc/xray/server.crt 
```

**xray配置文件：**

```json
{
    "log": {
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "port": 443,
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "72bac1c4-02de-49b4-e498-fa8767638c23", 
                        "flow": "xtls-rprx-direct"
                    }
                ],
                "decryption": "none",
                "fallbacks": [
                    {
                        "dest": 8388
                    }
                ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "xtls",
                "xtlsSettings": {
                    "alpn": [
                        "http/1.1"
                    ],
                    "certificates": [
                        {
                            "certificateFile": "/usr/local/etc/xray/server.crt", 
                            "keyFile": "/usr/local/etc/xray/server.key"
                        }
                    ]
                }
            }
        },
        {
            "port": 8388,
            "listen": "127.0.0.1",
            "protocol": "trojan",
            "settings": {
                "clients": [
                    {
                        "password": "111"
                    }
                ],
                "fallbacks": [
                    {
                        "dest": "180.76.138.44:80"
                    }
                ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "none"
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom"
        }
    ]
}
```

**视频时间线：**
00:00 vmess的问题和vless的出现
01:18 吃瓜：V2Ray和Xray为什么分家？
04:40 vless+xtls+回落通信过程
14:48 XTLS存在被主动探测的特征
19:15 vless+xtls+回落实战搭建
26:37 总结和下集预告