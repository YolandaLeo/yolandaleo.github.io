---
layout: post
categories: ['development']
tag: design
title: "工作中的设计模式(Draft"
---

最近刚好在通过Sonar审视自己项目的代码，解决了一些比较明显的Vulnerabilities之后，发现项目中有许多subclass都出现了因为@Inject一些组件，以及初始化，注册变量等导致Sonar报出大量Duplicate code smell。虽然可以通过设置Sonar rule来绕过这些可能不必要的警告，但是我决定先看看有没有更优雅的方案。

以下是目前代码的结构描述:
<!--more-->

<img src="{{ site.baseurl }}/img/subclass_description.png" class="inline"/>

目前项目需要监听业务消息，并根据配置确定时候需要某个MessageProducer来处理这个消息，产生特定的变量计算，作为输出结果。对于每一个变量，对应于一个Messageroducer，可以预见到，但变量数量增加，这些MP也将变得难以维护，并且大量@Injection代码重复。Sonar也会给到Code Smell - Major的报告。
我对于这个问题，一下子想到的办法是抽取出AbstractProducer，来维护这些依赖注入。
