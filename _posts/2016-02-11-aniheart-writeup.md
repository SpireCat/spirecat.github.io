---
layout: post
title:  "使用NodeJS+Webpack热替换实现CSS3动态加载"
date:   2016-2-16 13:16:30
categories: WEB前端
permalink: /archivers/aniheart
---
# 项目简介

&#160; &#160; &#160; &#160;实际上这个项目的工作是在模仿国外一位名叫Samuel Reed的前端大牛的工作(<http://strml.net>)。然而遗憾的是，模仿的过程并不只是copy代码这么简单(原作是一个github上的开源项目)。
先贴出自己的初始版本(<http://spirecat.github.io/anicat>)，当然这是纯“模仿”，相似度极高。
后来做了一个相对好玩的(<http://spirecat.github.io/aniheart>)，在之前的基础上实现的新的动作，尽管技术上的创新性并不是很高。

# 技术支持

&#160; &#160; &#160; &#160;在做这个项目之前自己并没有很扎实的前端基础，所以理清架构到着手读源码到再到最终的实现，用了整整一个假期的时间(还被高中同学调侃越来越像“死宅”2333333)。为了clone出这样的一个项目，博主准备了这些方面的知识。

1. HTML
2. CSS3
3. NODEJS(include JavaScript)
4. WEBPACK
5. BABEL/BABEL-LOADER

&#160; &#160; &#160; &#160;由于没有人讲解，所以在开始的几天内长一只在努力理清这些知识脉络。( ▼-▼ )

# 原理分析


