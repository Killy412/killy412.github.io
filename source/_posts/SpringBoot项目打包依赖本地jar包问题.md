---
title: SpringBoot项目打包依赖本地jar包问题
date: 2020-11-16 15:16:44
tags:
  - "SpringBoot"
categories:
  - "开发日志"
---

### 前言

最近公司项目业务中遇到一些关于 Word 转换 pdf 相关的功能.功能的实现用到了网上的第三方 jar 包,没有中央仓库地址,只能下载下来本地引用依赖. 因此遇到了一些引用本地 jar 包和打包的问题.

<!--more-->

### 本地引用

首先在模块根目录下(pom 文件同级目录)创建一个`lib`文件夹,把下载的 jar 包放进去. 然后在 pom 文件中加入本地 jar 包依赖.

```xml
<dependency>
    <groupId>bbb</groupId>
    <artifactId>aaa</artifactId>
    <version>1.0</version>
    <scope>system</scope>
    <!--本地jar包路径-->
    <systemPath>${basedir}/lib/xxx.jar</systemPath>
</dependency>
```

这里 groupId/artifactId/version 标签的值好象是没有要求的.

这样本地开发就可以用到 jar 包的相关东西了.但是打包时不会把本地的 jar 包打进去.

### 打包时把本地 jar 包一块打入 jar 包中

在父模块中加入配置

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <!--主要配置-->
                <includeSystemScope>true</includeSystemScope>
            </configuration>
        </plugin>
    </plugins>
</build>
```

这样再打包时就会把本地 jar 包打进去了.
