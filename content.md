#### 偶然想到，自己还没有做好使用nginx作为反向代理，搭配多个域名的https，心血来潮，就问了下bing怎么做。

#### docker-compose.yml

```dockerfile
version: '3.7'

services:
  nginx-proxy:
    image: nginx
    container_name: nginx-proxy
    restart: always
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./nginx-proxy/certs:/etc/nginx/certs:ro
      - ./nginx-proxy/conf.d:/etc/nginx/conf.d # mount the local config directory


  nginx1:
    image: nginx
    container_name: nginx1
    restart: always
    environment:
      - VIRTUAL_HOST=swjnxyf.cc # change this to your first domain name
    volumes:
      - ./nginx1/html:/usr/share/nginx/html # mount the local html directory



```

```shell
docker-compose up -d 
[root@localhost server]# tree
.
├── docker-compose.yml
├── nginx1
│   └── html
├── nginx2
│   └── html
└── nginx-proxy
    ├── certs
    └── conf.d

```

##### cd nginx-proxy/conf.d

#### nginx.conf

```nginx
upstream swjnxyf.cc {
  server nginx1;
}

server {
  listen 80;
  server_name swjnxyf.cc;

  location /.well-known/acme-challenge/ {
    root /usr/share/nginx/html;
    try_files $uri =404;
  }

  location / {
    return 301 https://$host$request_uri;
  }
}

server {
  listen 443 ssl;
  server_name swjnxyf.cc;

  ssl_certificate /etc/nginx/certs/swjnxyf.cc.crt;
  ssl_certificate_key /etc/nginx/certs/swjnxyf.cc.key;



  location / {
    proxy_pass http://swjnxyf.cc; # proxy to nginx2 container for other requests
  }
}

server {
  listen 443 ssl;
  server_name swjnxyf.cn;

  ssl_certificate /etc/nginx/certs/swjnxyf.cn.crt;
  ssl_certificate_key /etc/nginx/certs/swjnxyf.cn.key;


  location / {
    proxy_pass http://swjnxyf.cn; # proxy to nginx2 container for other requests
  }
}

```

#### 这样，就可以完成基本的转发请求和对应域名的https证书了。

##### 最后，需要进入certs文件使用openssl生产域名的证书

```shell
openssl genrsa -out swjnxyf.cc.key 2048
openssl req -new -key swjnxyf.cc.key -out swjnxyf.cc.csr
openssl x509 -req -days 365 -in swjnxyf.cc.csr -signkey swjnxyf.cc.key -out swjnxyf.cc.crt
```

#### then ,reload the nginx-proxy

```
docker exec -it nginx-proxy /bin/sh

# nginx -t	//that's right
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

# nginx -s reload
2023/06/17 15:00:53 [notice] 36#36: signal process started

Because I have modified the hosts file of win10, resolved the domain names I need to the IP of the virtual machine, I can experiment smoothly. Access https://swjnxyf.cc/ is OK. Prompt certificate risk, ignore it.。
```

