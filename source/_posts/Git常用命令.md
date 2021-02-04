---
title: Git常用命令
date: 2021-02-04 09:43:27
tags: "Git"
categories: "Git"
---

## Git 命令

```shell

#推送本地内容
git push -u origin(本地库名字) master(分支) 第二次提交不用-u
```

### 创建版本库

```shell
git init  #初始化 git 本地仓库
git add xxx #添加文件到本地库
git commit -m "" #提交文件
#本地库和远程库关联
git remote add name(本地库名字) git@github.com:path(用户名)/xxx.git(仓库名)
git remote add 本地库名字 远程仓库路径

```

### 版本状态

```shell
git status  #查看当前 git 的状态
git diff #查看修改的地方
git log--pretty[--oneline](提交信息一行输出) [-p](查看修改详情 增删修改文件数量) [--graph]以图形查看 查看日志
git reset --hard HEAD^^ #回退 head 表示当前版本
git reflog #记录每次命令
```

### 分支

```shell
git branch            #查看分支
git branch xxx        #创建本地分支
git checkout xx       #切换本地分支
git checkout -b xxx   #创建+切换分支
git fetch origin <分支名>     # 拉取远程分支到本地
git checkout -b 本地分支x origin/远程分支x  # 拉取远程分支到本地
git tag         # 查看所有tag
git checkout -b 本地分支x tag名  # 切换到某次tag处代码
git merge  xxx     #合并某分支到当前分支
git branch -d xxx    #删除本地分支
git branch -d -r origin/branchname #删除远程分支
git branch -m|-M oldbranch newbranch #重命名分支 如果new名字已经存在 需要使用-M强制重命名
git pull origin xxx #拉取分支上的代码
git br -m [old_br] [new_br]  #分支重命名
# 将其他分支的修改移到另一分支
git stash          # 在master分支中：
git checkout dev   # 切换到dev分支
git stash apply    # 将修改同步到dev分支
```

推送本地分支到远程分支
远程已有 remote_branch 分支并且已经关联本地分支 且本地已经切换到本地分支 git push
远程已有 remote_branch 分支但未关联本地分支 且本地已经切换到本地分支 git push -u origin/remote_branch
远程没有 remote_branch 分支 本地已经切换到本地分支 git push origin localhost_branch:remote_branch

正常情况我们要 clone 一个 github 工程是这样的

git**@github**.com:jj/JForm.git

如今在 github 工程是这样的
git **clone github**:jj/JForm.git
git_ali 阿里
github github

### 删除某个文件的历史记录

```shell
# linux下
git filter-branch --index-filter "git rm -r --cached --ignore-unmatch  <file/dir>" HEAD

# windows下
git filter-branch --index-filter "git rm -r --cached --ignore-unmatch <file/dir>" HEAD
```

## 回滚撤销操作

1. 工作区

```shell
git diff  #查看工作区的修改
git checkout -- .  # 撤销工作区的修改
git checkout -- [filename]
git add .    # 提交到暂存区
```

2. 暂存区

```shell
git diff --staged   # 查看暂存区的修改
git reset .         # 撤销暂存区的修改 状态返回到工作区

git reset --hard  # 如果已暂存，但未提交本地仓库之前，想把所有文件直接抛弃(而不是从暂存区删除)
git commit -m ""    # 提交到本地仓库
git rm --cached src/test.pptx  # 删除暂存区的文件
```

3. 本地仓库

```shell
git checkout a18c6fa   # 撤销本地提交,回到指定某次log的状态
git reset --hard HEAD~1  # 重置之前的提交

git revert 711bb0b  # 撤销某次修改

git merge --abort   #撤销合并
git push origin master --force #强制提交
```

## pull request

1. fork 项目,克隆自己的仓库到本地

2. 获取最新代码

   ```shell
   # 在本地仓库添加源仓库地址
   git remote add upstream <原仓库地址>
   # 更新代码
   git pull upstream master

   ```

   fork 来的 master 主分支作为跟踪源仓库代码.

3. 创建分支,贡献自己的代码.

   ```shell
   # 创建分支
   git checkout -b <分支名>
   # 提交代码
   git commit -a -m ""
   ```

4. 合并修改

   ```shell
   # 切换master分支
   git checkout master
   # 更新远程代码
   git pull upstream master
   # 切回到分支
   git checkout <分支名>
   # 合并分支
   git rebase master
   # 提交代码到自己的远程仓库
   git push origin <分支名>
   ```

5. Pull Request

   ```
   在自己的代码仓库,找到分支,点击new pull request 添加注释提交
   或者
   切换到 branch1 分支的代码仓库，点击 Compare & pull request 按钮，添加注释后提交。
   ```

## 拉取 github 代码慢的解决方法

```shell
# 拉取代码
git clone https://github.com/facebook/react.git

# 如果拉取速度很慢，可以考虑如下2个方案：

# 1. 使用cnpm代理
git clone https://github.com.cnpmjs.org/facebook/react

# 2. 使用码云的镜像（一天会与react同步一次）
git clone https://gitee.com/mirrors/react.git
```

## 电脑上多 git 仓库配置

在用户路径下`.ssh`文件夹下创建`config`文件. 配置对应的服务器地址和私钥路径.

```
# 该文件用于配置私钥对应的服务器

# Default github user
Host github.com
HostName github.com
User git
IdentityFile C:/Users/xxx/.ssh/github_rsa

# Default github user
Host other.com
HostName other.com
User git
IdentityFile C:/Users/xxx/.ssh/other_rsa
```

- 生成密钥文件命令
  ```shell
  ssh-keygen -t rsa -C "<邮件名>"
  ```
