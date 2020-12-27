### docker 安装nginx 配置ssl

#### 1.生成ssl正式

参考https://www.jianshu.com/p/cad3377692c9

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
IP.1 = 192.168.3.238
IP.2 = 192.168.3.46
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
        --name xiy-nginx \
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
docker run --detach \
        --name xiy-nginx \
        -p 443:443\
        -p 57202:57202 \
        -v /etc/nginx/qlh/nginx1.19/data:/usr/share/nginx/html\
        -v /etc/nginx/qlh/nginx1.19/nginx.conf:/etc/nginx/nginx.conf/\
        -v /etc/nginx/qlh/nginx1.19/log:/var/log/nginx/\
        -v /etc/nginx/qlh/nginx1.19/cert:/etc/nginx/cert\
        -d nginx:1.19

```



##### 1.6运行过程中可能有异常

       ###### 1.6.1 查看是否成功启动了xiy-nginx,若docker ps

```sh
docker ps
#docker ps 查看不到的情况下则没成功启动
docker ps -all #查看所有容器包括为启动的
docker logs xiy-nginx #查看容器启动日志
docker rm xiy-nginx #删除容器命令 若该容器是启动状态则必须docker stop xiy-nginx才能执行移除命令
```