---
title: Docker常用命令
date: 2020-11-09 22:03:47
tags:
  - "Docker"
  - "运维"
categories:
  - "Docker"
toc: true
---

## 安装 Docker 虚拟机

```bash
yum install -y yum-utils # 安装依赖软件包
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo #添加软件仓库 阿里云docker安装包
yum makecache fast  # 启用缓存
yum list docker-ce --showduplicates | sort -r  # 查看版本 选择一个
yum install docker-ce-<version> -y        # 安装
systemctl enable docker # 开机自启
systemctl start docker  # 启动
systemctl stop docker  # 停止
```
<!-- more -->

### 设置镜像加速器

```bash
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io
```

```bash
{"registry-mirrors":["https://docker.mirrors.ustc.edu.cn"]}
BashCC++C#CSSGoHaskellHTMLJavaJavaScriptJSONJSXkotlinPHPPowerShellPythonRubyRustSQLSwiftTypeScriptXML

## 新方法 编辑 /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn",
    "https://reg-mirror.qiniu.com",
    "https://registry.docker-cn.com"
  ]
}
sudo systemctl daemon-reload  # 更新修改的配置文件
```

编辑/etc/docker/daemon.json 文件，把结尾的逗号去掉

- 管理镜像

```bash
#搜索镜像
docker search 关键字
#下载镜像
docker pull [选项] [Docker Registry 地址[:端口]/]仓库名[:标签]
docker pull 镜像名字
#查看镜像
docker images -a
#重命名镜像
docker tag 旧镜像 新镜像
#删除镜像
docker rmi 镜像ID
#导出镜像
docker save -o 压缩文件路径 镜像名字
#导入镜像
docker load < 压缩文件路径
```

- 构建镜像

```bash
docker build [选项] <上下文路径>  # 构建镜像
```

- 创建容器

```bash
#查看运行的容器
docker ps 查看运行的容器
#创建普通容器
docker run -it --name 别名 镜像名字 程序名字
#创建含有端口映射的容器   -d{后台运行容器}
docker run -d --name 别名 -p 宿主机端口:容器端口 镜像名字 程序名字
#创建含有挂载目录的容器
docker run -d --name 别名 -v 宿主机目录:容器目录 --privileged 镜像名字 程序名字
#删除容器
docker rm -f 容器id
#######################操作容器
#暂停容器
docker pasue 容器
#恢复容器
docker unpause 容器
#停止容器
docker stop 容器
#启动容器
docker start -i 容器
docker rm -f xxx 删除容器
docker restart 容器id
docker update --restart=always   #容器id  更新启动方式  docker启动容器就启动
                       =no  # 不自动启动
docker logs -f -t --tail 行数 容器ID  实时查看docker日志
docker inspect [容器id]  # 查看容器信息
docker volume prune  # 清理多余的数据卷
# 拷贝docker日志
docker container cp [容器id]:[日志路径] [拷贝的目标路径]
# docker运行日志路径
docker inspect --format='{{.LogPath}}' [容器id]
# 日志copy
docker logs <options> [容器id] >& logs/myFile.log
```

- 进入容器

```bash
docker exec -it [容器id] bash
```

- 导出容器

```bash
docker export [容器ID] > [本地文件.tar]
```

- 登录账号

```bash
# 阿里云镜像仓库登录
sudo docker login --username=killy412 registry.cn-beijing.aliyuncs.com
# 注销
docker logout
```

## docker 安装 redis

- 拉取镜像

```bash
docker pull daocloud.io/library/redis:5.0.5
```

- 运行容器

```bash
docker run -p 6379:6379 --name redis -v /redis/redis.conf:/etc/redis/redis.config -v /redis/data:/data -d redis:5.0.5 redis-server --appendonly yes  --requirepass "1qaz@WSX" # 设定密码
```

## docker 安装 mysql

- 拉取镜像

```bash
docker pull mysql:8.0.15
```

- 运行容器

```
docker run -p 3306:3306 --restart=always --name mysql -e MYSQL_ROOT_PASSWORD=1qaz@WSX -v /mysql/conf/my.cnf:/etc/my.cnf -d mysql:8.0.15 --default-authentication-plugin=mysql_native_password
```

- 客户端连不上

1. `select user, host, plugin from user;` 查询加密插件
2. `ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'password';` 修改加密方式

## docker 安装 rabbitmq

- 运行容器

```bash
docker run -d --hostname rabbit-host --name rabbitmq --restart=always -e RABBITMQ_DEFAULT_USER=user -e RABBITMQ_DEFAULT_PASS=1qaz@WSX -p 15672:15672 -p 5672:5672 rabbitmq:3.7.15-management
```

- --hostname：指定容器主机名称
- --name:指定容器名称
- -p:将 mq 端口号映射到本地

1. 进入容器 docker exec -it xxx bash
2. 添加用户

```cmd
rabbitmqctl add_user root 123456
```

3. 赋予权限

```bash
rabbitmqctl set_permissions -p / root ".*" ".*" ".*"
```

4. 赋予角色

```
rabbitmqctl set_user_tags root adminstrator
```

5. 查看用户列表以及角色

```
rabbitmqctl list_users
```

## docker 安装 es

```bash
docker run -e ES_JAVA_OPTS="-Xms512m -Xmx512m" -d -p 9200:9200 -p 9300:9300 -v /etc/elsearch/elsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml --name es-master
elasticsearch:6.7.0
```

- 设置 jvm 线程数

```bash
vim /etc/sysctl.conf
# 添加这个
vm.max_map_count=262144
# 保存后执行这个命令
sysctl -p
```

- 安装插件-

```bash
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.7.0/elasticsearch-analysis-ik-6.7.0.zip
```

## <font color="red">删除 none 镜像</font>

```bash
docker images|grep none|awk '{print $3}'|xargs docker rmi
docker rmi $(docker images | grep "^<none>" | awk "{print $3}")
```

## **服务器重启,使用容器重启命令**
