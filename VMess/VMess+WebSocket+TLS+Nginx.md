# VMess+WebSocket+TLS+Nginx
[WebSocket+TLS+Web](https://toutyrater.github.io/advanced/wss_and_web.html)

## 服务器 V2Ray 配置
```json
{
  "inbounds": [
    {
      "port": 10000,
      "listen":"127.0.0.1",//只监听 127.0.0.1，避免除本机外的机器探测到开放了 10000 端口
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "b831381d-6324-4d53-ad4f-8cda48b30811",
            "alterId": 64
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


## Nginx 配置
```conf
server {
  listen  443 ssl;
  ssl on;
  ssl_certificate       /etc/v2ray/v2ray.crt;
  ssl_certificate_key   /etc/v2ray/v2ray.key;
  ssl_protocols         TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers           HIGH:!aNULL:!MD5;
  server_name           mydomain.me;
        location /ray { # 与 V2Ray 配置中的 path 保持一致
        proxy_redirect off;
        proxy_pass http://127.0.0.1:10000;#假设WebSocket监听在环回地址的10000端口上
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;

        # Show realip in v2ray access.log
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
}
```
```c++
// 由于 WebSocket 使用 HTTP/1.1，因此 Nginx 访问后端的 V2Ray 服务器时需要使用 HTTP/1.1 协议。添加如下设置：
proxy_http_version 1.1;

// 立 WebSocket 连接时，客户端会请求将连接升级为 WebSocket，因此还需要使 Nginx 与后端服务器建立 WebSocket 连接。在此需要设置 Nginx 访问 V2Ray 服务器时的请求头，添加如下设置：
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";

// 同时为了防止长时间没有数据发送导致 Nginx 切断连接，这里还可以设置超时时间：
proxy_read_timeout 600s;

// 由于是 Nginx 访问后端服务器，因此后端服务器并不知道客户端的真实 IP 地址。如果想要让后端服务器知道客户端的 IP 地址，同样需要设置请求头。添加如下设置：
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

// 当通过不正确的方式访问 WebSocket 的路径时，后端服务器会响应 400 Bad Request。这对 GFW 来讲是一种可能可以被探测到的特征，因此可以设置 Nginx 捕获后端服务器的 400 Bad Request 响应，并跳转到其它位置。这里以跳转到主页为例，添加如下设置：
proxy_intercept_errors on;
error_page 400 = https://name.yourdomain/;
```


## 客户端配置
```json
{
  "inbounds": [
    {
      "port": 1080,
      "listen": "127.0.0.1",
      "protocol": "socks",
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "settings": {
        "auth": "noauth",
        "udp": false
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "mydomain.me",
            "port": 443,
            "users": [
              {
                "id": "b831381d-6324-4d53-ad4f-8cda48b30811",
                "alterId": 64
              }
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
    }
  ]
}
```


## 其他说明
+ V2Ray 暂时不支持 TLS1.3，如果开启并强制 TLS1.3 会导致 V2Ray 无法连接

+ 较低版本的nginx的location需要写为 /ray/ 才能正常工作

+ 如果在设置完成之后不能成功使用，可能是由于 SElinux 机制(如果你是 CentOS 7 的用户请特别留意 SElinux 这一机制)阻止了 Nginx 转发向内网的数据。如果是这样的话，在 V2Ray 的日志里不会有访问信息，在 Nginx 的日志里会出现大量的 "Permission Denied" 字段，要解决这一问题需要在终端下键入以下命令：
```setsebool -P httpd_can_network_connect 1```

+ 请保持服务器和客户端的 wsSettings 严格一致，对于 V2Ray，/ray 和 /ray/ 是不一样的


+ 开启了 TLS 之后 path 参数是被加密的，GFW 看不到；

+ 主动探测一个 path 产生 Bad request 不能证明是 V2Ray；

+ 不安全的因素在于人，自己的问题就不要甩锅，哪怕我把示例中的 path 改成一个 UUID，依然有不少人原封不动地 COPY；

+ 使用 Header 分流并不比 path 安全， 不要迷信。

