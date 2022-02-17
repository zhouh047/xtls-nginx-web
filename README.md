# 性能
经实测 Vless+TCP+xtls-rprx-direct 比 Vmess+ws+tls+web 延迟降低一半，Chrome首页加载、Google搜索速度均有明显改善，非常建议使用。

# 配置服务器
基于Centos 7/8

## 1、关闭SELinux
```
setsebool -P httpd_can_network_connect 1 && setenforce 0
```

## 2、放行防火墙
```
firewall-cmd --permanent --add-service=https; firewall-cmd --permanent --add-service=http; firewall-cmd --reload;
```

## 3、安装内核
V2ray-Core最新版已经将XTLS移除，如果你希望用XTLS功能，请加上版本参数：
```
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh) --version 4.32.1

```
查看内核版本

```
[root@ecs-83EPf ~]# v2ray -version
V2Ray 4.32.1 (V2Fly, a community-driven edition of V2Ray.) Custom (go1.15.4 linux/amd64)
A unified platform for anti-censorship.
```

## 4、配置V2ray
```
vi /usr/local/etc/v2ray/config.json
```
写入
```
{
    "log": {
        "loglevel": "warning"
    }, 
    "inbounds": [
        {
            "listen": "0.0.0.0", 
            "port": 443, 
            "protocol": "vless", 
            "settings": {
                "clients": [
                    {
                        "id": "****", // 填写UUID，可以使用/usr/bin/v2ray/v2ctl uuid生成
                        "level": 0, 
                        "email": "a@b.com",
                        "flow":"xtls-rprx-direct"
                    }
                ], 
                "decryption": "none", 
                "fallbacks": [
                    {
                        "dest": 2233
                    }
                ]
            }, 
            "streamSettings": {
                "network": "tcp", 
                "security": "xtls", 
                "xtlsSettings": {
                    "serverName": "****", //换成自己的域名
                    "alpn": [
                        "http/1.1"
                    ], 
                    "certificates": [
                        {
                            "certificateFile": "/etc/pki/tls/certs/****.crt", // 换成你的证书，绝对路径
                            "keyFile": "/etc/pki/tls/private/****.key"  // 换成你的私钥，绝对路径
                        }
                    ]
                }
            }
        }
    ], 
    "outbounds": [
        {
            "protocol": "freedom", 
            "settings": { }
        }
    ]
}
```
写完可以验证一下
```
/usr/local/bin/v2ray -test -config=/usr/local/etc/v2ray/config.json
```

```
[root@ecs-83EPf ~]# /usr/local/bin/v2ray -test -config=/usr/local/etc/v2ray/config.json
V2Ray 4.32.1 (V2Fly, a community-driven edition of V2Ray.) Custom (go1.15.4 linux/amd64)
A unified platform for anti-censorship.
2020/12/26 13:48:57 [Info] v2ray.com/core/main/jsonem: Reading config: /usr/local/etc/v2ray/config.json
Configuration OK.
```

然后设置开机自启并启动v2ray服务：
```
systemctl enable v2ray; systemctl start v2ray
```

V2ray 常用命令

查看V2ray 状态：``` service v2ray status```

启动V2Ray:``` service v2ray start```

重启V2Ray:``` service v2ray restart```

停止V2Ray：```  service v2ray stop```

查看V2Ray版本:``` v2ray -version```

## 5、配置Nginx
安装Nginx
```
yum install -y nginx
```

编辑配置文件
```
vi /etc/nginx/conf.d/default.conf
```
写入
```
server {
    listen 80 ;
    #换成自己的域名
	  server_name xxx.com; 
	  rewrite ^(.*)$ https://${server_name}$1 permanent; 
    if ($request_method  !~ ^(POST|GET)$) { return  501; }
    autoindex off;
    server_tokens off;
 }
 
 server{
    listen 2233;
    #换成自己的域名
	  server_name xxx.com;
	
	  access_log /var/log/nginx/access.log;
    error_log   /var/log/nginx/error.log   error;
	  location /{
		  root  /usr/share/nginx/html/; 
	  }
	
	  index index.php index.html index.htm;
	
	  if ($request_method  !~ ^(POST|GET)$) { return 501; }
    autoindex off;
    server_tokens off;
}
```

再验证一下Nginx配置是否正确，输入：
```
nginx -t
```

```
[root@ecs-83EPf ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

把网页模板文件夹里的所有东西，包括index.html，blog.html，以及css，fonts，img，js几个文件夹，全部上传到
```
/usr/share/nginx/html/
```

然后设置开机自启并启动Nginx服务：
```
systemctl enable nginx; systemctl start nginx
```

Nginx 常用命令

查看Nginx 状态：```service nginx status```

启动Nginx:```  service nginx start```

重启Nginx:```service nginx restart```

停止Nginx：```  service nginx stop```




