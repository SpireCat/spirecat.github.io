---
layout: post
title: 码农孙的笔记(1)——Tomcat类加载机制
date: 2018-02-10 00:00:00 +0300
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: normalnight.jpeg # Add image post (optional)
tags: [Record] # add tag
---

&emsp;&emsp;*这是新开的一个系列叫“码农孙的笔记”，为了归档&记录一些零碎知识点。（名字有点鬼畜，无视）*

&emsp;&emsp;最近在公司处理的大多都是与工程相关的事情，频频与Tomcat打交道。之前在学校中用Tomcat都是泛泛在用，没有深究过一些原理。比如今天要讲的Tomcat类加载器，只知道有一个规定的加载顺序，但不知其所以然，于是整理这篇文章对Tomcat类加载机制进行归纳。

&emsp;&emsp;对于一个健全的Java Web服务器，一定会实现自己定义的类加载器，并且有一些显著存在的问题需要类加载器解决：

* 部署在同一个服务器的多个Web应用之间所使用的Java类库可以互相隔离。很显然这是为了迎合不同应用需要依赖同意第三方类库不同版本的需求。

* 部署在同一个服务器的多个Web应用之间所使用的Java类库可以互相共享。同理，对于一些通用的第三方类库，同一个服务器上部署多套显然是不合理的。

* 部署在服务器上的多个Web应用与服务器本身所使用的Java类库可以互相隔离、共享（我们熟知，一部分Web应用服务器本身就是Java代码编写的）。

* “一定程度”的热替换功能。熟悉ClassLoader的同学对于“动态加载”和“热替换”一定不会陌生，而这里我想说的热替换主要针对Web应用的，包括但不限于“ASP、JSP等修改后”、“静态资源文件修改后”、“Class文件修改后”无需重启的能力。

&emsp;&emsp;其实解决类库的问题是十分简单的，依靠事先规划好的不同用途的文件路径便可轻松解决，剩下的如何选择的问题丢给类加载器来搞定。对于Tomcat而言，包含3组目录（“／common／\*”、“／server/\*”、“／shared／\*”）可以存放Java类库，另外Web应用程序自身的“／WEB-INF/\*”目录，共分为4组。4组目录设定分别如下：

* ／common --- 被Tomcat和所有Web应用程序所共同使用

* ／server --- 被Tomcat使用，对Web应用程序不可见we b

* ／shared --- 被Web应用程序使用，对Tomcat不可见

* ／${webapp.name}／WEB-INF --- 仅能被此Web应用使用，对其他Web应用和Tomcat均不可用。

&emsp;&emsp;为了支持这套目录结构，Tomcat采用了一套 **双亲委派模型** 的类加载器来实现。在这里我们岔开话题，简单介绍下双亲委派模型。 *（完整的JVM的类加载原理会在“深入理解JVM”系列中详细解释，待更）*

&emsp;&emsp;双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应该有自己的父类加载器，而这种父子关系一般通过组合关系来实现，而不是通过继承。某个特定的类加载器在接到加载类的请求时，首先将加载任务委托给父类加载器，依次递归，如果父类加载器可以完成类加载任务，就成功返回；只有父类加载器无法完成此加载任务时，才自己去加载。

<center><img src="/assets/img/tomcat-class-load.jpg"/></center>

&emsp;&emsp;Common、Catelina、Shared和WebApp是由Tomcat自己定义的类加载器，分别加载／common、／server、／shared和／${webapp.name}／WEB-INF中的Java类库。需要注意的是，其中WebApp类加载器和JSP类加载器通常会存在多个实例，**每一个Web应用程序对应一个WebApp类加载器，每一个JSP文件对应一个JSP类加载器。** 这种机制也就帮助Tomcat实现了热替换，即，当服务器检测到JSP文件被修改时，会替换掉目前的JSP类加载器实例，并新创建一个JSP类加载器实例。

&emsp;&emsp;然，对于Tomcat6.X的版本，已经将／common、／server、和／shared三个目录默认合并至／lib目录，这个目录里的类库相当于以前的／common目录中类库的作用。这和我们之前介绍的概念并不冲突，只是做了Tomcat目录结构的简化，你仍然可以指定tomcat／conf／catalina.properties配置文件中的server.loader和share.loader项来建立CatalinaClassLoader和SharedClassLoader实例，否则默认创建CommonClassLoader实例。

&emsp;&emsp;以上。
