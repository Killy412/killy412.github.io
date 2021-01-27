---
title: Linux常用命令
date: 2021-01-24 00:32:11
tags: "Linux"
categories: "Linux"
---

## 常用命令

```bash
# 查看某个进程的子进程
pstree -p [pid]
# 查看进程中子线程的资源占用情况
top -Hp [pid]
top -n 1 -H -p [pid]
# 查看磁盘情况
df -hl
# 查找程序
ps -ef | grep -v grep | grep fisco-bcos
# 查看端口占用情况
netstat -ano | grep tcp |grep 8545
# 查看端口占用进程
lsof -i:<端口>
# 添加执行权限
chmod a+x <文件名>
```

<!--more-->

## scp 拷贝文件/文件夹命令

```shell
# 远程仓库拷贝文件到本地  xx.xx.x.xx为服务器ip
scp root@xx.xx.x.xx:/opt/soft/demo.tar /opt/soft/
# 远程仓库拷贝目录到本地
scp -r root@xx.xx.x.xx:/opt/soft/test /opt/soft/
# 上传文件到服务器指定目录
scp /opt/soft/demo.tar root@xx.xx.x.xx:/opt/soft/scptest
# 上传本地目录到服务器指定目录
scp -r /opt/soft/test root@xx.xx.x.xx:/opt/soft/scptest
```

## grep 命令

```shell
# 搜索文本 grep命令相关
grep [-acinv] [--color=auto] '搜寻字符串' filename
选项与参数：
-a:将 binary 文件以 text 文件的方式搜寻数据
-c:计算找到 '搜寻字符串' 的次数
-I:忽略二进制文件
-i:忽略大小写的不同，所以大小写视为相同
-n:顺便输出行号
-v:反向选择，亦即显示出没有 '搜寻字符串' 内容的那一行！
-r:递归文件夹下的所有子文件
-L, --files-without-match  只打印没有匹配上的<文件>的名称
-l, --files-with-matches  只打印有匹配的<文件>的名称
--color=auto/always:可以将找到的关键词部分加上颜色的显示喔！
```

## screen 命令

```shell
# 新建一个名叫 test 的session，并马上进入
screen -S test
# 创建一个名叫 test 的session，但暂不进入，可用于系统启动脚本里
screen -dmS test
# 列出当前所有session
screen -ls
# 接入(attach)一个名为test会话
screen -r test
# 远程脱离(detach) test 会话
screen -d test
# 脱离(detach)当前会话，将目前的 screen session (可能含有多个 windows) 丢到后台执行
CTRL+a然后d
# 查看 screen 版本号
screen -v
# 如果由于某种原因其中一个会话死掉了（例如人为杀掉该会话），这时screen -list会显示该会话为dead状态。使用screen -wipe命令清除该会话：
screen -wipe
```

## 后台运行程序命令: nohup

```shell
nohup /root/runoob.sh > runoob.log 2>&1 &
```

**2>&1** 解释:

将标准错误 2 重定向到标准输出 &1 ，标准输出 &1 再被重定向输入到 runoob.log 文件中。

- 0 – stdin (standard input，标准输入)
- 1 – stdout (standard output，标准输出)
- 2 – stderr (standard error，标准错误输出)

## terminal 配置代理

在 .bashrc 或 .zshrc 中设置如下内容

```
alias setproxy="export ALL_PROXY=socks5://127.0.0.1:1080"
alias unsetproxy="unset ALL_PROXY"
aliaa ip="curl -i http://ip.cn"
```

在使用是手动调用这些命令进行设置。

或者使终端总是使用代理：

```
export http_proxy="socks5://127.0.0.1:1086"
export https_proxy="socks5://127.0.0.1:1086"
```

重启 terminal 生效。可以通过 curl -i http://ip.cn查看IP改变来测试是否生效
