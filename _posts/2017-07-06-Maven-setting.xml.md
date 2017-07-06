---
layout: post
title: Maven之setting.xml
categories: [maven]
description: Maven之setting.xml
keywords: maven
---
### 什么是Maven
Maven是一个项目管理和综合工具，用java编写，能简化和标准化项目建设过程。处理编译，部署，文档，团队协作和其他任务的无缝连接。 Maven增加可重用性并负责建立相关的任务。
<!--more-->
### Maven资源库
> * Maven本地资源库
> * Maven私服库
> * Maven中央仓库
一般在打包编译的时候，会先从本地仓库中查找，如果本地仓库没有，就会去设定好的私服仓库去查找，私服仓库上如果有就下载下来，没有私服仓库就会去中央仓库查找

### Maven的settings.xml文件
在安装好maven之后，我们首先要做的是根据我们的需求配置maven的settings.xml文件
settings.xml对于maven来说，相当于全局性的配置，用于所有的项目。在Maven2中存在两个settings.xml，一个位于maven2的安装目录conf下面，作为全局性配置。对于团队设置，保持一致的定义是关键，所以conf下的settings.xml就作为团队共同的配置文件。而对于每个单独的成员，如果有特殊的需要自定义的设置，如用户信息之类，就可以在~/.m2/settings.xml中进行设定

### settings.xml的基本结构如下
```xml
<settings xmlns="http://maven.apache.org/POM/4.0.0"  
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
          xsi:schemaLocation="http://maven.apache.org/POM/4.0.0  
                               http://maven.apache.org/xsd/settings-1.0.0.xsd">  
  <localRepository/>  
  <interactiveMode/>  
  <usePluginRegistry/>  
  <offline/>  
  <pluginGroups/>  
  <servers/>  
  <mirrors/>  
  <proxies/>  
  <profiles/>  
  <activeProfiles/>  
</settings>
```
```bash
#表示本地仓库的位置，默认在~/.m2/repository
<localRepository>/path/to/local/repo<localRepository/>
```
```bash
#如果不想每次编译都去查找中心库，就设置为true，前提是你必须已经下载好了必须的依赖包，默认是false
<offline>false<offline/>
```
```bash
#在POM文件中定义了distributionManagement元素，然而特定的username和password不能用于pom.xml，所以通过此配置来保存server信息，这里的id必须和后面repository中定义的id一致，主要是用来部署到nexus私服
<server>
  <id>release</id>
  <username>admin</username>
  <password>admin123</password>
</server>
```
```bash
#表示镜像仓库，指定库的镜像，这里的mirrorOf表示此镜像是那个库的镜像，*表示所有库
<mirrors>
<mirror>
  <id>nexus</id>
  <mirrorOf>*</mirrorOf>
  <name>Nexus Repository</name>
  <url>http://192.168.56.22:8081/nexus/content/groups/public/</url>
</mirror>
<mirrors/>
```
```bash
#代理服务器设置，主要用于无法直接访问中心的库用户设置
<proxies>  
   <proxy>  
     <id>myproxy</id>  
     <active>true</active>  
     <protocol>http</protocol>  
     <host>proxy.somewhere.com</host>  
     <port>8080</port>  
     <username>proxyuser</username>  
     <password>somepassword</password>  
     <nonProxyHosts>*.google.com|ibiblio.org</nonProxyHosts>  
   </proxy>  
 </proxies>
```
```bash
#自定义配置，可以指定repositories，properties，pluginRepositories，activation等元素
#这里的repositories表示，开发团队自己的开发库，定义这个库，一般是为了方便distributionManagement发布
<profiles>
<profile>
      <id>nexus</id>
      <repositories>
        <repository>
            <id>public</id>
            <name>public</name>
            <url>http://192.168.56.22:8081/nexus/content/groups/public/</url>
            <releases><enabled>true</enabled></releases>
            <snapshots><enabled>false</enabled></snapshots>
        </repository>
      </repositories>
</profile>
</profiles>
```
```bash
#properties作为maven的占位符之，包括5中类型值：
env.x：返回当前的环境变量
project.x：返回pom文件中定义的project元素值
settings.x：返回settings.xml中定义的元素
java系统属性：返回经过java.lang.System.getProperties()返回的值
x：用户自定义的值
```
```bash
#激活指定的profile，通过profile id 来指定
<activeProfiles>  
    <activeProfile>env-test</activeProfile>
</activeProfiles>
```
