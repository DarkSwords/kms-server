# KMS Server

+ 支持Windows与Office全系列激活

+ 快速部署，仅需一句命令即可使用

+ 内置各版本KMS密钥，无需自行查询

+ 网页端、命令行下快速检索密钥与说明

+ API接口，检查其他KMS服务器工作状态

## 使用方法

以 `kms.343.re` 为例，在成功部署以后：

+ 在需要KMS服务的地方，填入 `kms.343.re` 即可激活；

+ 可通过[网页](https://kms.343.re/)或命令行curl获取激活密钥：

```
# 输出操作说明
shell> curl kms.343.re

# 输出Windows的KMS密钥
shell> curl kms.343.re/win

# 输出Windows Server的KMS密钥
shell> curl kms.343.re/win-server

# 输出Office激活命令
shell> curl kms.343.re/office
```

+ 可用于测试其他KMS服务器是否正常：

```
# 端口号可不填，默认为1688
shell> curl kms.343.re/check/kms.dnomd343.top:1688
KMS Server: kms.dnomd343.top (1688) -> available
```

## 快速部署

> 如果您对Linux及容器较为熟悉，可以直接参考以下命令部署；若接触较少，建议阅读后文的详细部署流程

开启完整服务，`1688` 端口用于KMS激活，`1689` 端口用于HTTP访问（反向代理）：

```bash
docker run -d --restart=always --network host dnomd343/kms-server
# 或使用bridge网络
docker run -d --restart=always -p 1688-1689:1688-1689 dnomd343/kms-server
```
```
version: '3'
services:
  kms-server:
    image: dockerhub16/kms-server
    ports:
      - "1688-1689:1688-1689"  #1689 web访问端口
    restart: always
```

仅开启KMS服务：

```bash
docker run -d --restart=always --network host dnomd343/kms-server --disable-http
# 或使用环境变量指定
docker run -d --restart=always --network host --env DISABLE_HTTP=TRUE dnomd343/kms-server
```

开启DEBUG模式：

```bash
docker run -d --restart=always --network host dnomd343/kms-server --debug
# 或使用环境变量指定
docker run -d --restart=always --network host --env DEBUG=TRUE dnomd343/kms-server
```

使用自定义端口：

```bash
docker run -d --restart=always --network host dnomd343/kms-server --kms-port 11688 --http-port 23333
# 或使用环境变量指定
docker run -d --restart=always --network host --env KMS_PORT=11688 --env HTTP_PORT=23333 dnomd343/kms-server
```

## 部署流程

### 1. 防火墙检查

> 服务器 `1688/tcp` 端口接受外网KMS激活请求，务必检查是否被防火墙拦截。

如果开启了 `ufw` 防火墙服务，使用以下命令放行：

```
shell> ufw allow 1688/tcp
shell> ufw status
```

如果开启了 `firewalld` 防火墙服务，使用以下命令放行：

```
shell> firewall-cmd --zone=public --add-port=1688/tcp --permanent
shell> firewall-cmd --reload
shell> firewall-cmd --list-ports
```

部分云服务商可能会在网页端控制台提供防火墙服务，请在对应页面开放 `1688/tcp` 端口。

### 2. Docker检查

```
shell> docker --version
···Docker版本信息···
```

若上述命令未输出版本信息，使用以下命令安装：

```
shell> sudo curl https://get.docker.com | bash
··· Docker安装信息 ···
```

### 3. 启动KMS服务

> 本项目基于Docker构建，在[Docker Hub](https://hub.docker.com/r/dnomd343/kms-server)或[Github Package](https://github.com/dnomd343/kms-server/pkgs/container/kms-server)可以查看已构建的各版本镜像。

`kms-server` 同时发布在多个镜像源上（国内网络可使用阿里云仓库加速）：

+ `Docker Hub` ：`dnomd343/kms-server`

+ `Github Package` ：`ghcr.io/dnomd343/kms-server`

+ `阿里云镜像` ：`registry.cn-shenzhen.aliyuncs.com/dnomd343/kms-server`

> 下述命令中，容器路径可替换为上述其他源

使用以下命令启动KMS服务：

```
shell> docker run -d --restart=always --network host dnomd343/kms-server
```

> 容器使用 `1688/tcp` 与 `1689/tcp` 端口，前者用于KMS激活，后者为HTTP服务

> 容器启动时，添加 `--kms-port xxx` 与 `--http-port xxx` 选项可修改以上端口

若仅需KMS激活功能，无需执行后续步骤，本步完成后即可正常使用；同时，建议添加 `--disable-http` 选项关闭内部HTTP服务，以减少多余的性能占用。

### 5. 配置反向代理

> 这一步用于配置反向代理，对外暴露kms-server的HTTP服务

将用于KMS服务的域名DNS解析到当前服务器，配置反向代理到本机 `1689/tcp` 端口，以Nginx为例：

```
# 进入nginx配置目录
shell> cd /etc/nginx/conf.d

# 添加反向代理配置
shell> vim kms.conf
```

配置文件如下，按备注修改域名与证书：

```
server {
    listen 80;
    listen [::]:80;
    server_name kms.343.re;  # 改为自己的KMS域名

    location / {
        if ($http_user_agent !~* (curl|wget)) {  # 来自非命令行的请求，重定向到https
            return 301 https://$server_name$request_uri;
        }
        proxy_set_header X-Real-IP $remote_addr;  # 反向代理转发真实IP
        proxy_set_header Host $http_host;  # 反向代理转发当前域名
        proxy_pass http://127.0.0.1:1689;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name kms.343.re;  # 改为自己的KMS域名
    ssl_certificate /etc/ssl/certs/343.re/fullchain.pem;  # 改为自己的TLS证书文件
    ssl_certificate_key /etc/ssl/certs/343.re/privkey.pem;  # 改为自己的TLS私钥文件

    gzip on;  # 可选，开启gzip压缩，提高加载速度
    gzip_buffers 32 4K;
    gzip_comp_level 6;
    gzip_min_length 100;
    gzip_types application/javascript text/css text/xml;
    gzip_disable "MSIE [1-6]\.";
    gzip_vary on;

    location / {
        proxy_set_header X-Real-IP $remote_addr;  # 反向代理转发真实IP
        proxy_set_header Host $http_host;  # 反向代理转发当前域名
        proxy_pass http://127.0.0.1:1689;
    }
}
```

重启Nginx服务

```
shell> nginx -s reload
```

访问KMS服务域名，页面正常显示即反向代理成功。

### 6. 检查服务是否正常

检查部署的KMS服务器是否正常，例如检测新搭建的服务器 `kms.dnomd343.top` ，执行以下命令：

```
shell> curl kms.343.re/check/kms.dnomd343.top
KMS Server: kms.dnomd343.top (1688) -> available
```

输出 `available` 即工作正常，若失败请检查防火墙是否屏蔽1688端口。

## 开发相关

### 运行参数

+ `--debug` ：进入DEBUG模式，输出调试日志

+ `--kms-port` ：指定KMS激活端口，默认值为 `1688`

+ `--http-port` ：指定HTTP服务端口，默认值为 `1689`

+ `--disable-http` ：禁用HTTP服务，默认值为 `false`

+ `-v` 或 `--version` ：显示版本信息并退出

+ `-h` 或 `--help` ：显示帮助信息并退出

### 环境变量

+ `DEBUG` ：值为 `true` 时，进入DEBUG模式

+ `KMS_PORT` ：指定KMS激活端口，默认值为 `1688`

+ `HTTP_PORT` ：指定HTTP服务端口，默认值为 `1689`

+ `DISABLE_HTTP` ：值为 `true` 时，禁用HTTP服务

### JSON接口

`kms-server` 预留了以下JSON接口，用于输出内置的KMS密钥。

+ `/json`：输出全部KMS密钥；

+ `/win/json`：输出各版本Windows的KMS密钥；

+ `/win-server/json`：输出各版本Windows Server的KMS密钥；

### KMS测试

`kms-server` 内置了检测其他KMS服务器是否可用的功能，接口位于 `/check` 下，使用以下参数：

+ `host`：目标服务器IPv4、IPv6地址或域名

+ `port`：目标服务端口，可选，默认值为 `1688`

```
shell> curl -sL "https://kms.343.re/check?host=8.210.148.24" | jq .
{
  "success": true,
  "available": true,
  "host": "8.210.148.24",
  "port": 1688,
  "message": "kms server available"
}

shell> curl -sL "https://kms.343.re/check?host=kms.dnomd343.top&port=8861" | jq .
{
  "success": true,
  "available": false,
  "host": "kms.dnomd343.top",
  "port": 8861,
  "message": "kms server connect failed"
}
```

### 容器构建

**本地构建**

```
shell> git clone https://github.com/dnomd343/kms-server.git
shell> cd ./kms-server/
shell> docker build -t kms-server .
```

**交叉构建**

```
shell> docker buildx build -t dnomd343/kms-server --platform="linux/amd64,linux/arm64,linux/386,linux/arm/v7" https://github.com/dnomd343/kms-server.git --push
```

## 许可证

MIT ©2021 [@dnomd343](https://github.com/dnomd343)
