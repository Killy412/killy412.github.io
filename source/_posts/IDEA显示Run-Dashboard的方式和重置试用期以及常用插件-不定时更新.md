---
title: IDEA显示Run Dashboard的方式和重置试用期以及常用插件(不定时更新..)
date: 2021-01-02 12:48:30
tags:
  - "工具"
  - "IDEA"
  - "插件"
categories: "工具"
---

## IDEA 显示 Run Dashboard 的方式

在.idea 文件夹下面找到 workspace.xml 文件, 添加以下标签

```xml
<component name="RunDashboard">
    <option name="configurationTypes">
      <set>
        <option value="SpringBootApplicationConfigurationType" />
      </set>
    </option>
  </component>
```

<!--more-->

## 常用插件

- Translation 翻译
- One Dark Theme 暗黑主题
- Rainbow Brackets 括号变色
- Key Promoter X 快捷键设置
- CodeGlancePro 代码缩略图,和 vscode 文档打开类似
- mybatisX # mybatis 插件
- RestfulTool 路径映射搜索工具

- ASM bytecode viewer 字节码查看工具
- Maven Helper maven 工具,可以查看当前 maven 依赖冲突情况
- Jrebel 热部署,需要破解
- Alibaba Java Coding Guilelines 阿里编码规约
- Alibaba cloud Toolkit 云部署工具
- GsonFormat json 格式化工具
- GenerateAllSetter 一键调用对象的所有 setter 方法
- GenerateSerialVersionUID 生成 serialVersionUUID
- .env files support

## 重置 idea 试用期

idea 更新的很快,然后上个版本的破解方法可能就无效了.所有对常更新的用户来说破解不那么重要,过期重置一下试用期更方便.
Windows 环境没有试过,linux 亲测过,可以使用.不过现在有一个重置插件可以使用.

- linux

```shell
rm -rf ~/.java/.userPrefs/jetbrains/pycharm
cd ~/.config/JetBrains/PyCharm2020.2
rm -rf eval
rm -rf options/other.xml
```

- windows

```shell
cd "C:%HOMEPATH%\.IntelliJIdea*\config"
rmdir "eval" /s /q
del "options\other.xml"
reg delete "HKEY_CURRENT_USER\Software\JavaSoft\Prefs\jetbrains\idea" /f
```
