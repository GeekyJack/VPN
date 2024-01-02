视频教程：[点击进入观看](https://bulianglin.com/g/aHR0cHM6Ly95b3V0dS5iZS95OHM1VWl2TU5jRQ)
vmess是一个比trojan出现更早、使用更广泛的协议
作为使用者，我们的重点是如何使用`vmess`，但是不讲vmess协议的通信流程又无法对这个协议的一些特性有深刻的认识，比如vmess节点为什么在电脑系统时间不对的时候无法链接？又比如为什么加密方式可以使用自动选择，按照之前讲ss节点说的，对称加密算法必须让两边都存在相同的密钥和加密方式才能正常解密。再比如额外ID(`alterID`)到底是个啥？还有承载vmess数据的传输协议与伪装的区别，本节的话就来带大家了解上述问题，并由浅入深实操搭建一个基于`nginx`实现web伪装的vmess节点。



> 你可能听到一些人说做`WEB伪装是自欺欺人，GFW根本就不会进行探测`，我不明白说这句话的人是不是开发GFW的，能够如此肯定不会进行探测。GFW是黑盒状态，没有人能完全了解他的工作机制，既然伪装成了https，**我们要做的是尽可能表现得和正常的网站行为是一样的，而不是去猜测GFW会不会来探测你的伪装。**

SSH连接工具（FinalShell）：[http://www.hostbuf.com/t/988.html](https://bulianglin.com/g/aHR0cDovL3d3dy5ob3N0YnVmLmNvbS90Lzk4OC5odG1s)
v2ray官方安装脚本：[https://github.com/v2fly/fhs-install-v2ray](https://bulianglin.com/g/aHR0cHM6Ly9naXRodWIuY29tL3YyZmx5L2Zocy1pbnN0YWxsLXYycmF5)

**申请证书：**

```shell
#安装acme：
curl https://get.acme.sh | sh
#安装socat：
apt install socat
#添加软链接：
ln -s  /root/.acme.sh/acme.sh /usr/local/bin/acme.sh
#切换CA机构： 
acme.sh --set-default-ca --server letsencrypt
#申请证书： 
acme.sh  --issue -d 替换为你的域名 --standalone -k ec-256
#安装证书： 
acme.sh --installcert -d 替换为你的域名 --ecc  --key-file   /usr/local/etc/v2ray/server.key   --fullchain-file /usr/local/etc/v2ray/server.crt 
```

**vmess+tcp:**

```json
{
  "inbounds": [
    {
      "port": 8388, 
      "protocol": "vmess",    
      "settings": {
        "clients": [
          {
            "id": "af41686b-cb85-494a-a554-eeaa1514bca7",  
            "alterId": 0
          }
        ]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",  
      "settings": {}
    }
  ]
}
```

**vmess+tcp(ws)+tls:**

```json
{
  "inbounds": [
    {
      "port": 8388, 
      "protocol": "vmess",    
      "settings": {
        "clients": [
          {
            "id": "af41686b-cb85-494a-a554-eeaa1514bca7",  
            "alterId": 0
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "tls",
        "tlsSettings": {
          "certificates": [
            {
              "certificateFile": "/usr/local/etc/v2ray/server.crt", 
              "keyFile": "/usr/local/etc/v2ray/server.key" 
            }
          ]
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
```

**vmess+ws+tls+web:**

```json
{
  "inbounds": [
    {
      "port": 8388,
      "listen":"127.0.0.1",
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "af41686b-cb85-494a-a554-eeaa1514bca7",
            "alterId": 0
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
        "path": "/ray"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
```

**nginx设置：**

```shell
#安装nginx：
apt install nginx
#重新加载nginx配置：
systemctl reload nginx.service
```

**nginx配置（替换http{}里的内容）：**

```nginx
server {
   listen 443 ssl;
   listen [::]:443 ssl;

   server_name v.buliang0.tk;  #你的域名
   ssl_certificate       /usr/local/etc/v2ray/server.crt; 
   ssl_certificate_key   /usr/local/etc/v2ray/server.key;
   ssl_session_timeout 1d;
   ssl_session_cache shared:MozSSL:10m;
   ssl_session_tickets off;

   ssl_protocols         TLSv1.2 TLSv1.3;
   ssl_ciphers           ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
   ssl_prefer_server_ciphers off;
    
    location / {
        proxy_pass https://www.bing.com; #伪装网址
        proxy_ssl_server_name on;
        proxy_redirect off;
        sub_filter_once off;
        sub_filter "www.bing.com" $server_name;
        proxy_set_header Host "www.bing.com";
        proxy_set_header Referer $http_referer;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header Accept-Encoding "";
        proxy_set_header Accept-Language "zh-CN";
    }
    
    location /ray {
       proxy_redirect off;
       proxy_pass http://127.0.0.1:10000;
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   }
}

server {
    listen 80;
    server_name v.buliang0.tk;    #你的域名
    rewrite ^(.*)$ https://${server_name}$1 permanent;
}
```

**视频时间线：**
00:00 vmess和v2ray的关系
01:54 vmess通信过程
11:25 vmess被精准探测
13:42 搭建vmess+tcp
17:15 vmess+tcp(ws)+tls原理
21:07 传输协议承载和伪装的区别
24:53 搭建vmess+tcp(ws)+tls
28:15 vmess+ws+tls+web原理
33:25 搭建vmess+ws+tls+web
36:35 总结：vmess还有优势吗？