---
layout: post
title:  "Centos配置Jenkins实现Android自动打包并上传到蒲公英"
date:   2018-12-06 17:32:10
categories: Android
tags: Jenkins 
---

* content
{:toc}

本文记录Centos配置Jenkins实现Android自动打包并上传到蒲公英。




# Centos配置Jenkins 实现Android自动打包并上传到蒲公英

操作系统为macOS，Jenkins安装操作系统为Centos。

大致流程：

* 安装sun JDK，并配置环境变量。
* 安装git。
* 安装gradle，并配置环境变量。
* 安装Android SDK，并配置环境变量。
* 安装、配置Jenkins。

## 卸载openJDK（centos默认安装openJDK）

> rpm -qa \| grep jdk

该命令会列出所有的jdk内容

> yum remove 具体的jdk

## centos安装zip unzip（用于后续解压安装包）
> yum install zip unzip

## centos安装sun java

通过`wget`下载可能会存在无法打开问题，本次使用本地下载，再上传到服务器上的操作。
因为本次操作使用的系统为`macOS`，所以在终端输入

> scp 下载的jdk完整路径 root@ip地址:~

登录到服务器之后将文件移动到想要移动的位置。
下一步，解压文件
> tar zxvf jdk文件.tar.gz

## centos 安装git
> yum install git

## centos 安装gradle并配置环境
> wget https://downloads.gradle.org/distributions/gradle-5.0-all.zip

解压gradle，我这里直接放在了最顶层目录

> unzip gradle-5.0-all.zip

配置环境变量

> vim /etc/profile

编辑文件，在文件中输入以下内容

```vim
export GRADLE_HOME=gradle-5.0
export PATH=${GRADLE_HOME}/bin:${PATH}
```
vim命令模式下输入`i`进入输入模式，输入以上内容按`esc`返回命令模式，在命令模式下输入`:`进入底线命令模式，在底线命令模式下输入`w`保存文件，之后输入`q`退出vim程序。

之后执行

> source /etc/profile

让配置生效。

检验Gradle是否安装成功

> gradle -version

## centos安装Android SDK，并配置环境变量。

进入`opt`目录，创建`androidSDK`目录。

> cd opt

> mkdir androidSDK

> cd androidSDK/

> wget https://dl.google.com/android/repository/sdk-tools-linux-3859397.zip

> unzip sdk-tools-linux-3859397.zip

配置环境变量
> vim /etc/profile

```vim
export ANDROID_HOME=/opt/androidSDK/tools
export PATH=${ANDROID_HOME}/bin:${PATH}
```

之后执行

> source /etc/profile

检查当前sdk安装信息

> sdkmanager --list

安装需要的package

> sdkmanager "build-tools;28.0.1"

## 安装Jeknins

> wget -O /etc/yum.repos.d/jenkins.repo http://jenkins-ci.org/redhat/jenkins.repo

> rpm --import http://pkg.jenkins-ci.org/redhat/jenkins-ci.org.key

> yum install jenkins

启动Jeknins

> sudo service jenkins start

停止Jeknins

> sudo service jenkins stop

Jenkins的安装目录

> /var/lib/jenkins/

Jeknins配置文件的目录

> /etc/sysconfig/jenkins

## Jenkins配置

安装完Jekins之后，在浏览器输入`http://<服务器ip>:8080/ `就能打开Jenkins的欢迎页。
欢迎页需要输入一个初始密码，这个在安装之后就存在，并且会提示这个密码文件所在的位置，使用centos的`cat`命令就可以查看。

> cat 初始密码文件位置

之后复制这个密码，输入。

下一步安装基本插件跟创建账户，过于简单，不贴图。

进行了以上操作之后，就会进入到Jeknins的主页。

![Jenkins_Home](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jeknins_Home.jpg)

下面是进行繁琐的环境配置。

* 配置全局Android SDK

**系统管理-系统设置**

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_SDK.jpg)

* 配置全局JDK、Git跟Gradle

**系统管理-全局工具配置**

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_git_gradle_jdk.png)

配置以上环境之后就可以创建项目，本文以自己github上的某一个项目为例，[项目地址](https://github.com/qfxl/Architecturekotlin)。
 
![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins1.jpg)    
    
![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins2.jpg)

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins3.jpg)

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins4.jpg)

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins5.jpg)

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins6.jpg)

> clean assemble${flavor}${buildType} --stacktrace --debug

保存。

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_build1.jpg)

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_build2.jpg)
    
顺利的话，apk就会生成，如何查看？

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_dest1.jpg)

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_dest2.jpg)

## 将打包的apk自动发布到蒲公英。

[蒲公英官方文档有Jenkins集成发布说明。](https://www.pgyer.com/doc/view/jenkins_plugin)

**如何操作？**

怎么申请蒲公英，认证，获取key之类的操作不做记录。

* 安装upload to pgyer插件

**系统管理-插件管理**

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_plugin.png)

点击直接安装。

修改项目的配置信息。

![image](https://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_pg0.jpg)

![image](https://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_pg1.jpg)

![image](https://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_pg2.jpg)

经过上述操作，apk将顺利发布到蒲公英。

## Jenkins打包自动生成apk的下载二维码。

这个操作需要借助另外一个插件，`description setter`。

安装该插件之后，修改工程的配置信息。

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_qrcode.jpg)

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_qrcode2.jpg)

``` html
<a href="${appBuildURL}"><img src="${appQRCodeURL}" width="118" height="118"/></a>
```

上述两个参数，上传到蒲公英成功之后会自动写入全局变量。

最后，Jenkins默认的`标记格式器` 为 `纯文本`不支持html标签，所以需要修改一下。

**系统管理-全局安全配置**

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_html.jpg)

保存。

再次构建完成的效果图如下：

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_build_success.png)

## 修改build name

如果不想构建名称是 `#1 #2`这样的话可以选择修改名称。
借助插件`build-name-setter`。
安装完成之后，在项目配置里如下修改。

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_build_name.png)

之后再build就是如下效果：

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/Jenkins_build_rename.png)

## 可能会遇到的问题汇总

### 阿里云服务器配置完成之后无法通过8080端口打开Jenkins

阿里云ecs服务器需要添加安全组规则开放该端口，否则无法通过外网访问，具体操作如下。

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/aliyun_1.jpg)

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/aliyun_2.jpg)

![image](http://qfxl.oss-cn-shanghai.aliyuncs.com/images/aliyun_3.jpg)

### Jenkins打包异常，文件夹没有读写权限。

> chmod 777 -R 你的目录