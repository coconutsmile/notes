### docker 安装nginx 配置ssl

#### 1.生成ssl正式

参考https://www.jianshu.com/p/cad3377692c9

# openssl为IP签发证书（支持多IP/内外网）

> **参考文档**：
> [1. OpenSSL自签发配置有多域名或ip地址的证书](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fu013066244%2Farticle%2Fdetails%2F78725842)
> [2. 如何创建一个自签名的SSL证书(X509)](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.cnblogs.com%2Fdinglin1%2Fp%2F9279831.html)
> [3. 如何创建自签名证书？](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.racent.com%2Fblog%2Farticle-how-to-create-a-self-signed-certificate)

## 背景

- 开启https必须要有ssl证书，而安全的证书来源于受信任的CA机构签发，通常需要付费，并且他们只能为域名和外网IP签发证书。
- 证书有两个基本目的：分发公有密钥和验证服务器的身份。只有当证书是由受信任的第三方所签署的情形下，服务器的身份才能得到恰当验证，因为任何攻击者都可以创建自签名证书并发起中间人攻击。
- 但自签名证书可应用于以下背景： 
  - 企业内部网。当客户只需要通过本地企业内部网络时，中间人攻击几乎是完全没有机会的。
  - 开发服务器。当你只是在开发或测试应用程序时，花费额外的金钱去购买受信任的证书是完全没有必要的。
  - 访问量很小的个人站点。如果你有一个小型个人站点，而该站点传输的是不重要的信息，那么攻击者很少会有动机去攻击这些连接。

## 依赖

利用 OpenSSL 签发证书需要 OpenSSL 软件及库，一般情况下 CentOS、Ubuntu 等系统均已内置， 可执行 `openssl` 确认，如果提示 `oepnssl: command not found`,则需手动安装，以Centos为例：

```
yum install openssl openssl-devel -y
```

## 签发证书

#### step1: 生成证书请求文件

新建openssl.cnf，内容如下：

```
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req

[req_distinguished_name]
countryName = Country Name (2 letter code)
countryName_default = CH
stateOrProvinceName = State or Province Name (full name)
stateOrProvinceName_default = GD
localityName = Locality Name (eg, city)
localityName_default = ShenZhen
organizationalUnitName  = Organizational Unit Name (eg, section)
organizationalUnitName_default  = organizationalUnitName
commonName = Internet Widgits Ltd
commonName_max  = 64

[ v3_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]

# 改成自己的域名
#DNS.1 = kb.example.com
#DNS.2 = helpdesk.example.org
#DNS.3 = systems.example.net

# 改成自己的ip
IP.1 = 172.16.24.143
IP.2 = 172.16.24.85
```

#### step2: 生成私钥

`san_domain_com` 为最终生成的文件名，一般以服务器命名，可改。

```
openssl genrsa -out san_domain_com.key 2048
```

#### step3: 创建CSR文件

创建CSR文件命令：

```
openssl req -new -out san_domain_com.csr -key san_domain_com.key -config openssl.cnf
```

执行后，系统会提示输入组织等信息，按提示输入如即可。

测试CSR文件是否生成成功，可以使用下面的命令：

```
openssl req -text -noout -in san_domain_com.csr

//执行后，会看到类似如下的信息：
Certificate Request:
    Data:
        Version: 0 (0x0)
        Subject: C=US, ST=MN, L=Minneapolis, OU=Domain Control Validated, CN=zz
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                //...
```

#### step4: 自签名并创建证书

```
openssl x509 -req -days 3650 -in san_domain_com.csr -signkey san_domain_com.key -out san_domain_com.crt -extensions v3_req -extfile openssl.cnf
```

执行后，可看到本目录下多了以下三个文件

```
san_domain_com.crt

san_domain_com.csr

san_domain_com.key
```

至此，使用openssl生成证书已完成，以下 `nodejs项目验证` 和 `将证书导入本地` 仅是验证证书是否正常可用。

亲测有效哦

需要的openssl.cnf如下

 ```nginx
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req

[req_distinguished_name]
countryName = Country Name (2 letter code)
countryName_default = CH
stateOrProvinceName = State or Province Name (full name)
stateOrProvinceName_default = GD
localityName = Locality Name (eg, city)
localityName_default = ShenZhen
organizationalUnitName  = Organizational Unit Name (eg, section)
organizationalUnitName_default  = organizationalUnitName
commonName = Internet Widgits Ltd
commonName_max  = 64

[ v3_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]

# 改成自己的域名
DNS.1 = zjport.dubai.yw.com
#DNS.2 = helpdesk.example.org
#DNS.3 = systems.example.net

# 改成自己的ip
IP.1 = 192.168.3.46
IP.2 = 192.168.8.49
 ```



##### 1.1 生成会san_domain_com.key与san_domain_com.crt文件

##### 1.2 创建系统挂载目录及将san_domain_com.key与san_domain_com.crt文件放入/etc/nginx/qlh/nginx1.19/cert下（因为容器一旦删除文件就会消失，内部数据无法持久化，需要把容器内部日志，配置文件等映射到外部系统）

```sh
mkdir -p /etc/nginx/qlh/nginx1.19/{data,cert,log}
```



##### 1.3创建nginx配置文件 vim /etc/nginx/qlh/nginx1.19/nginx.conf

```nginx
user root;
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent"  "$http_x_forwarded_for" "$http_x_client_address"';

    access_log   /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;
	client_max_body_size 10M;



	server 
	{
        listen 57202;
        server_name  localhost;
		charset utf-8;
		rewrite ^(.*)$   https://$host$1 permanent;
		#return 301 https://$server_name$request_uri;
    }

	
	server {
	 listen 443; 
	 server_name localhost;
	 ssl on;
	 root html;
	 index index.html index.htm;
	 #这里使用docker容器中的目录
	 ssl_certificate /etc/nginx/cert/san_domain_com.crt;
     ssl_certificate_key /etc/nginx/cert/san_domain_com.key;
	 ssl_session_timeout 5m;
	 ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
	 ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	 ssl_prefer_server_ciphers on;
	 location /file/ {
			root /usr/local/tomcat/xiy-webapp/;
			#access_log /usr/local/nginx1.19.4/logs/file.log;
			#error_log /usr/local/nginx1.19.4/logs/file_error.log;
            autoindex on;
            autoindex_exact_size off;
			autoindex_localtime on;
		}
		location /xiy {
            proxy_pass http://192.168.3.238:7201/xiy;
			#access_log /usr/local/nginx1.19.4/logs/xiy.log main;
			#error_log /usr/local/nginx1.19.4/logs/xiy_error.log;
            proxy_set_header Host $host:$server_port;
            proxy_set_header   Remote_Addr        $remote_addr;
            proxy_set_header   X-Real_Ip          $http_x_client_address;
            proxy_set_header   X-Forwarded-For    $proxy_add_x_forwarded_for;
        }
	 location /remote_addr {
			# 返回状态码200 以及一些信息。
			return 200 "Your remote_addr is $remote_addr\nYour realip_remote_addr is $realip_remote_addr\nYour realip_remote_port is $realip_remote_port\nYour X-Forwarded-For $http_x_forwarded_for";
		}
	}
}
```

##### 1.4获取1.19 nginx镜像

```
docker pull nginx:1.19
```

##### 1.5 运行容器



```sh
docker run --detach \
        --name xiy-nginx \
        -p 443:443\
        -p 57202:57202 \
        -v /etc/nginx/qlh/nginx1.19/data:/usr/share/nginx/html:rw\
        -v /etc/nginx/qlh/nginx1.19/nginx.conf:/etc/nginx/nginx.conf/:rw\
        -v /etc/nginx/qlh/nginx1.19/log:/var/log/nginx/:rw\
        -v /etc/nginx/qlh/nginx1.19/cert:/etc/nginx/cert/:rw\
        -d nginx:1.19

```

````sh
说明
docker run --detach \
        --name qlh-nginx \
        #映射端口 系统端口:容器端口
        -p 443:443\
        -p 57202:57202 \
        #静态文件目录映射从系统映射到容器
        #-v 系统路径:容器路径
        -v /etc/nginx/qlh/nginx1.19/data:/usr/share/nginx/html:rw\
        -v /etc/nginx/qlh/nginx1.19/nginx.conf:/etc/nginx/nginx.conf/:rw\
        -v /etc/nginx/qlh/nginx1.19/log:/var/log/nginx/:rw\
        -v /etc/nginx/qlh/nginx1.19/cert:/etc/nginx/cert/:rw\
        -d nginx:1.19
````

```
docker run --name qlh-nginx \
        -p 443:443\
        -p 57202:57202 \
        -v /etc/nginx/qlh/nginx1.19/data:/usr/share/nginx/html\
        -v /etc/nginx/qlh/nginx1.19/nginx.conf:/etc/nginx/nginx.conf/\
        -v /etc/nginx/qlh/nginx1.19/log:/var/log/nginx/\
        -v /etc/nginx/qlh/nginx1.19/cert:/etc/nginx/cert\
        -d nginx:qlh-https

```



##### 1.6运行过程中可能有异常

       ###### 1.6.1 查看是否成功启动了qlh-nginx,若docker ps

```sh
docker ps
#docker ps 查看不到的情况下则没成功启动
docker ps -all #查看所有容器包括未启动的
docker logs qlh-nginx #查看容器启动日志
docker rm qlh-nginx #删除容器命令 若该容器是启动状态则必须docker stop xiy-nginx才能执行移除命令
```