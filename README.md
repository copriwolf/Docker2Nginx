
# Docker Nginx 注入环境变量(Docker2Nginx)

Docker 构建可读取环境变量的 Nginx。

Building a Nginx that can read environment variables with Docker.


【[中文说明](https://0ne.store/2023/06/05/20230605-docker-with-nginx-env/)】

## TL; DR

The official Docker Image of Nginx comes with ENVSUBST support, which allows for dynamic injection of environment variables in Docker Compose.

---

## 1 Introduction

As we need to deploy services with different domain names on multiple machines, we need to dynamically inject domain names into Nginx instead of hardcoding them in the configuration file.

However, directly writing environment variables in Docker or mounting Nginx.conf does not support dynamic injection by default, so we need to build a Nginx that supports dynamic injection ourselves.


## 2 Solution

### 2.1 Idea and Thought

The official Docker Image of Nginx comes with ENVSUBST support, which allows for dynamic injection of environment variables in Docker Compose.


### 2.2 Specific Example

#### 2.2.1 File Tree

```bash
.
|-- docker-compose.yml
`-- nginx
    |-- html
    |   `-- index.html
    |-- logs
    |   `-- nginx
    |       |-- access.log
    |       `-- error.log
    `-- nginx.conf
```

#### 2.2.1 Dockerfile

```dockerfile
version: '3.5'
services:
  nginx_lb:
    container_name: nginx_lb
    image: nginx:alpine
    ports:
     - "80:80"
     - "80:80/udp"
    volumes:
     - ./nginx/nginx.conf:/etc/nginx/templates/nginx.conf.template:ro # Nginx Setting Map
     - ./nginx/logs/nginx:/var/log/nginx            # Nginx Log Dir Map
     - ./nginx/html:/usr/share/nginx/html           # Nginx Web File Dir Map
    environment:
      API_HOST: {{DOMAIN}}
      API_PORT: {{PORT, e.g: 80}}
      # Other Nginx Params
    restart: unless-stopped
```

#### 2.2.2 Nginx.conf

```nginx

##### {{{ DEMO TEMPLATE Sever Config #####
server {
        listen    ${API_PORT};    # Listening port
        server_name ${API_URL};   # Listening domain
        root     /var/www/html;   # Nginx Web File Path
        index index.html index.htm index.php;

        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }

        location = /favicon.ico {
            log_not_found off;
        }

        # Set To Proxy is also ok.
        # proxy_connect_timeout 5s;
        # proxy_timeout 20s;
        # proxy_pass 127.0.0.1:1235;

}
##### }}} #####

##### {{{ DEMO TEMPLATE Log Format #####
log_format  main_demo  '$remote_addr - $remote_user [$time_local] "$request" '
           '$status $body_bytes_sent "$http_referer" '
           '"$http_user_agent" "$http_x_forwarded_for"';
access_log  /var/log/nginx/access.log  main_demo;
##### }}} #####
```

#### 2.2.3 Run

```bash

# 1 Get Code
$ git clone https://github.com/copriwolf/Docker2Nginx.git
# 2 Change Config
# -- Change Env Params/ Ports in `docker-compose.yml`
# -- Change Html Content in `nginx/html/index.html`
# 3 Start 
$ sudo docker compose up --build -d 
# 4 Review Logs
$ sudo docker logs -f 
```

#### 2.2.4 Extra

- Because the ENVSUBST script is mounted by default to the `/etc/nginx/conf.d` directory, `http{}` cannot be used in `nginx.conf`. Instead, it can be directly written into the server layer.
