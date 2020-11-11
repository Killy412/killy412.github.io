---
title: 快速搭建单群组FISCO BCOS联盟链
date: 2020-11-11 13:15:56
categories:
  - "区块链"
tags:
  - "区块链"
---

### 准备环境
#### 安装依赖
开发部署工具 build_chain.sh脚本依赖于openssl, curl，使用下面的指令安装。 若为CentOS，将下面命令中的apt替换为yum执行即可。macOS执行brew install openssl curl即可（macOS自带的openssl指令选项不同，请执行安装标准openssl）。

```bash
# ubuntu环境安装依赖
sudo apt install -y openssl curl  
# centos环境安装依赖
sudo yum install -y openssl openssl-devel 
```
<!--more-->
#### 创建操作目录
```bash
cd ~ && mkdir -p fisco && cd fisco
```
#### 下载build_chain.sh脚本
```bash
curl -#LO https://github.com/FISCO-BCOS/FISCO-BCOS/releases/download/v2.6.0/build_chain.sh && chmod u+x build_chain.sh
```

### 生成一条单群组4节点的FISCO链
在fisco目录下执行下面的指令，生成一条单群组4节点的FISCO链.请确保机器的 30300-30303，20200-20203，8545-8548端口没有被占用。

```bash
bash build_chain.sh -l 127.0.0.1:4 -p 30300,20200,8545 [参数]
```

- 参数
  * `-i` 默认`127.0.0.1`,加上参数`0.0.0.0`,使外网可以访问
  * `-p` 指定节点的起始端口,每个节点占用三个端口,分别是p2p,channel,jsonrpc使用,分割端口,必须指定三个端口.同一个IP下的不同节点所使用端口从起始端口递增.
  * `-l` 用于指定要生成的链的IP列表以及每个IP下的节点数,以逗号分隔

命令执行成功会输出All completed。如果执行出错，请检查nodes/build.log文件中的错误信息。
```bash
Checking fisco-bcos binary...
Binary check passed.
==============================================================
Generating CA key...
==============================================================
Generating keys ...
Processing IP:127.0.0.1 Total:4 Agency:agency Groups:1
==============================================================
Generating configurations...
Processing IP:127.0.0.1 Total:4 Agency:agency Groups:1
==============================================================
[INFO] Execute the download_console.sh script in directory named by IP to get FISCO-BCOS console.
e.g.  bash /home/ubuntu/fisco/nodes/127.0.0.1/download_console.sh
==============================================================
[INFO] FISCO-BCOS Path   : bin/fisco-bcos
[INFO] Start Port        : 30300 20200 8545
[INFO] Server IP         : 127.0.0.1:4
[INFO] Output Dir        : /home/ubuntu/fisco/nodes
[INFO] CA Key Path       : /home/ubuntu/fisco/nodes/cert/ca.key
==============================================================
[INFO] All completed. Files in /home/ubuntu/fisco/nodes
```


### 启动FISCO BCOS链
启动所有节点
```bash
bash nodes/127.0.0.1/start_all.sh
```

停止所有节点
```bash
bash nodes/127.0.0.1/stop_all.sh
```

启动成功会输出类似下面内容的响应。否则请使用netstat -an | grep tcp检查机器的30300~30303，20200~20203，8545~8548端口是否被占用。
```bash
try to start node0
try to start node1
try to start node2
try to start node3
 node1 start successfully
 node2 start successfully
 node0 start successfully
 node3 start successfully
```

检查进程是否启动
```bash
ps -ef | grep -v grep | grep fisco-bcos
```

正常情况会有类似下面的输出； 如果进程数不为4，则进程没有启动（一般是端口被占用导致的）
```bash
fisco       5453     1  1 17:11 pts/0    00:00:02 /home/ubuntu/fisco/nodes/127.0.0.1/node0/../fisco-bcos -c config.ini
fisco       5459     1  1 17:11 pts/0    00:00:02 /home/ubuntu/fisco/nodes/127.0.0.1/node1/../fisco-bcos -c config.ini
fisco       5464     1  1 17:11 pts/0    00:00:02 /home/ubuntu/fisco/nodes/127.0.0.1/node2/../fisco-bcos -c config.ini
fisco       5476     1  1 17:11 pts/0    00:00:02 /home/ubuntu/fisco/nodes/127.0.0.1/node3/../fisco-bcos -c config.ini
```

### 文章参考[FISCO BCOS官方文档](https://fisco-bcos-documentation.readthedocs.io/zh_CN/latest/docs/installation.html)