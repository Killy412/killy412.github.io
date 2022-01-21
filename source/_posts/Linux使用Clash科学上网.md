---
title: Linux使用Clash科学上网
date: 2022-01-21 22:57:59
tags: 
  - "Linux"
  - "科学上网"
categories: "Linux"
---

## 安装Clash 

1. [官方下载](https://github.com/Dreamacro/clash/releases),选择一个合适电脑架构的.

2. 解压缩,并且赋予执行权限
```shell
cd
mkdir clash && cd clash
gzip clash.gz
sudo chmod +x clash
```
3. 下载配置文件
```shell
wget -O config.yaml [订阅链接]
```

4. 配置systemd服务,并且自启动. 新建脚本 `vim /etc/systemd/system/clash.service`,内容如下
```shell
[Unit]
Description=clash daemon

[Service]
Type=simple
User=root
ExecStart=~/clash/clash -d ~/clash
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

启动服务
```shell
systemctl daemon-reload
systemctl enable clash.service
systemctl start clash.service
```

## 定时更新订阅

- 创建定时任务脚本 `vim ~/clash/update-config.sh`
```shell
#!/bin/bash

# 设置clash路径
clash_path="~/clash"

# 停止clash
systemctl stop clash.service

# 取消代理
unset https_proxy

# 如果配置文件存在，备份后下载，如果不存在，直接下载
if [ -e $clash_path/config.yaml ]; then
	mv $clash_path/config.yaml $clash_path/configbackup.yaml
	wget -O $clash_path/config.yaml "[订阅链接]"
else
	wget -O $clash_path/config.yaml "[订阅链接]"
fi

# 重启clash
systemctl restart clash.service

# 重设代理
export https_proxy="http://127.0.0.1:7890"
```

- 配置crontab任务
```shell
sudo crontab -e 
# 分钟 小时 日 月 年 comm
0 0 1,15 * * sh ~/clash/update-config.sh
systemctl restart cron.service
```

## WebUI使用

- clone dashboard项目
```shell
git clone https://github.com/Dreamacro/clash-dashboard.git
cd clash-dashboard/
git checkout -b gh-pages origin/gh-pages
```

- 更改`config.yaml`文件中 `external-ui` 属性为上一步clone的dashboard文件路径.

- `systemctl restart clash.service` 重启之后,访问`localhose:9090/ui` 就可以看到clash相关配置界面了