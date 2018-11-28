# docker nginx 部署多个项目

## 前提条件
- 1、本地电脑和服务器已安装 docker，下载方法自行谷歌吧
- 2、在 docker hub 上已有账号， 注册传送门: https://hub.docker.com/
- 3、需要对 docker 已有所熟悉 ，并了解Dockerfile里的一些指令
## 使用Dockerfile 制作镜像

假如本机有一个叫web的项目


在web根目录下新建Dockerfile，写入以下内容

```
FROM nginx:1.13.6-alpine
LABEL maintainer="lilywang <lilywang.cd@gmail.com>"

ARG TZ="Asia/Shanghai"

ENV TZ ${TZ}

RUN apk upgrade --update \
    && apk add bash tzdata \
    && ln -sf /usr/share/zoneinfo/${TZ} /etc/localtime \
    && echo ${TZ} > /etc/timezone \
    && rm -rf /var/cache/apk/*

COPY dist /usr/share/nginx/html 

CMD ["nginx", "-g", "daemon off;"]

```




此时web里的文件结构为：
```
.
|____Dockerfile
|____dist // 为项目打包后的文件
| |____index.html
```
接下来在bash 进入到web目录

`cd web`

`docker build -t lilywang711/web .`

看到打印信息中有如下就说明镜像已经构建成功了
```
Successfully built 4c050212ce0d
Successfully tagged lilywang711/web:latest
```
也可以输入`docker images` 查看当前的镜像列表

接下来输入命令 `docker push lilywang711/web` 就可将刚才构建好的镜像上传到docker hub里面，方便等会儿我们在服务端拉取镜像

如果是有多个项目需要部署，那就按照以上步骤重复来就行，有多少个项目就构建多少个镜像

## 服务端部署

ssh 登陆服务器，在当前用户目录下，新建 nginx 文件夹，并在里面新建nginx.conf
在 nginx.conf 中写入以下内容

```
user nginx;
worker_processes  2;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;
events {
    use epoll;
    worker_connections  2048;
}
http {
    include  /etc/nginx/mime.types;
    # include /etc/nginx/conf.d/*.conf;
    root /usr/share/nginx/html;
    index  index.html index.htm;
    server {
        listen 80;
        server_name  a.yourdomain.cn;
        location / {
        }
    }
    server {
        listen 80;
        server_name  b.yourdomain.cn;
        location / {
            proxy_pass http://your_vps_ip:81;
        }
    }
    server {
        listen 80;
        server_name  localhost;
        location / {
        }
    }
}
```
接下来

启动docker `systemctl start docker`

拉取刚才制作并上传好的两个镜像 

`docker pull lilywang711/web` 

`docker pull lilywang711/web1`

输入以下命令启动容器

```
docker run -itd --name web -p 80:80 -v /root/nginx/nginx.conf:/etc/nginx/nginx.conf lilywang711/web
// -i 交互模式运行容器, -t 为容器分配一个伪终端，-d 后台运行容器，可直接连写 -itd
// --name 是给该容器起个叫web的名字，方便辨识
// -p 是绑定端口 本机端口80:容器端口80
// -v 声明volume，意思是将容器中的/etc/nginx/nginx.conf 挂载到 宿主机里的/root/nginx/nginx.conf，以后配置nginx只需要修改/root/nginx/nginx.conf就行了
```
另外一个lilywang711/web1镜像也同理，修改下端口和名字就好了

`docker run -itd --name web1 -p 81:80 -v /root/nginx/nginx.conf:/etc/nginx/nginx.conf lilywang711/web1`

此时输入 `docker ps` 就可以看到这两个容器已经跑起来了

docker化项目并在nginx部署就已经完成了
在浏览器输入 http://a.yourdomain.cn 和 http://b.yourdomain.cn 就可以看到效果了，分别对应本地电脑中的web 和 web1 项目


