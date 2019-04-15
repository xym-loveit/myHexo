---
title: IntelliJ IDEA个人常用配置
date: 2018-06-22 16:12:09
categories: [常用工具]
tags: [常用工具]
description: 换电脑或者重装Idea后，个人常用配置
---

## 基本设置



### 代码提示

![](http://op7wplti1.bkt.clouddn.com/xxvi-a-settings-introduce-1.jpg)

- IntelliJ IDEA 的代码提示和补充功能有一个特性：区分大小写。如上图标注 1 所示，默认就是 `First letter` 区分大小写的。
- 区分大小写的情况是这样的：比如我们在 Java 代码文件中输入 `stringBuffer` IntelliJ IDEA 是不会帮我们提示或是代码补充的，但是如果我们输入 `StringBuffer` 就可以进行代码提示和补充。
- 如果想不区分大小写的话，改为 `None` 选项即可。



###  代码检查等级

![](http://op7wplti1.bkt.clouddn.com/xxvi-a-settings-introduce-2.gif)

- 如上图 Gif 所示，该功能用来快速设置代码检查等级。我个人一般在编辑大文件的时候会使用该功能。IntelliJ IDEA 对于编辑大文件并没有太大优势，很卡，原因就是它有各种检查，这样是非常耗内存和 CPU 的，所以为了能加快大文件的读写，我一般会暂时性设置为 `None`。

> - `Inspections` 为最高等级检查，可以检查单词拼写，语法错误，变量使用，方法之间调用等。
> - `Syntax` 可以检查单词拼写，简单语法错误。
> - `None` 不设置检查。

### 省电模式

![](http://op7wplti1.bkt.clouddn.com/xxvi-a-settings-introduce-6.jpg)

> - 如上图标注 1 所示，IntelliJ IDEA 有一种叫做 `省电模式` 的状态，开启这种模式之后 IntelliJ IDEA 会关掉代码检查和代码提示等功能。所以一般我也会认为这是一种 `阅读模式`，如果你在开发过程中遇到突然代码文件不能进行检查和提示可以来看看这里是否有开启该功能。

 

### 代码分割

![](https://dancon.gitbooks.io/intellij-idea/content/images/xxvi-a-settings-introduce-9.gif)

- 如上图 Gif 所示，IntelliJ IDEA 支持对代码进行垂直或是水平分组。一般在对大文件进行修改的时候，有些修改内容在文件上面，有些内容在文件下面，如果来回操作可能效率会很低，用此方法就可以好很多。当然了，前提是自己的显示器分辨率要足够高。



### 代码提示快捷键

![](http://op7wplti1.bkt.clouddn.com/xxvi-a-settings-introduce-13.gif)

> - 如上图 Gif 所示，默认 `Ctrl + 空格` 快捷键是基础代码提示、补充快捷键，但是由于我们中文系统基本这个快捷键都被输入法占用了，所以我们发现不管怎么按都是没有提示代码效果的，原因就是在此。我个人建议修改此快捷键为 `Ctrl + 逗号`。

 ### 显示内存

![](http://op7wplti1.bkt.clouddn.com/xxvi-a-settings-introduce-14.gif)

> - 如上图 Gif 所示，IntelliJ IDEA 14 版本默认是不显示内存使用情况的，对于大内存的机器来讲不显示也无所谓，但是如果是内存小的机器最好还是显示下。如上图演示，点击后可以进行部分内存的回收。

 ### 多文件编辑

![](http://op7wplti1.bkt.clouddn.com/xxvi-a-settings-introduce-15.jpg)

> - 如上图标注 1 所示，在打开很多文件的时候，IntelliJ IDEA 默认是把所有打开的文件名 Tab 单行显示的。但是我个人现在的习惯是使用多行，多行效率比单行高，因为单行会隐藏超过界面部分 Tab，这样找文件不方便。

 ### 单行注释

![](http://op7wplti1.bkt.clouddn.com/xxvi-a-settings-introduce-16.gif)

> - 如上图 Gif 所示，默认 IntelliJ IDEA 对于 Java 代码的单行注释是把注释的斜杠放在行数的最开头，我个人觉得这样的单行注释非常丑，整个代码风格很难看，所以一般会设置为单行注释的两个斜杠跟随在代码的头部。

### 锁定模式（Pinned Mode ）

![](http://op7wplti1.bkt.clouddn.com/xxvi-a-settings-introduce-24.gif)

如上图 Gif 所示，当我们设置了组件窗口的 `Pinned Mode` 属性之后，在切换到其他组件窗口的时候，已设置该属性的窗口不会自动隐藏 

### 显示行数、方法分割

![](http://op7wplti1.bkt.clouddn.com/xxvi-a-settings-introduce-29.jpg)

- 如上图红圈所示，默认 IntelliJ IDEA 是没有勾选 `Show line numbers` 显示行数的，但是我建议一般这个要勾选上。
- 如上图红圈所示，默认 IntelliJ IDEA 是没有勾选 `Show method separators` 显示方法线的，这种线有助于我们区分开方法，所以也是建议勾选上的。

### 多模块依赖

![](http://op7wplti1.bkt.clouddn.com/xxvi-a-settings-introduce-41.gif)

- 如上图 Gif 所示，这是一个 Maven 多模块项目，在开发多模块的时候，经常会改到其他模块的代码，而模块与模块之间是相互依赖，如果不进行 install 就没办法使用到最新的依赖。
- 所以，为了减少自己手动 install 的过程，可以把 install 过程放在项目启动之前，就像 Gif 所示那样。

### 序列化接口提示自动生成serialVersionUID

序列化的时候如果不指定`serialVersionUID`，那么实际上每次都要根据类的定义去计算一个UID，这个计算的结果很可能会受编译器的影响，容易导致UID的不一致，出现序列化/反序列化失败。
不知为何 `IntelliJ` 默认没有增加这个 `Inspection` ，那我们加一下就好了。
找到下面的路径：`File > Settings > Editor > Inspections`。首先把 `Profile` 设置成`Default IDE`，这样配置才能在所有项目中应用，否则就只在当前项目中应用。然后在`Java > Serialization issues`中，找到`Serializable class without 'serialVersionUID'`，并把校验勾上。
增加了 `Inspections` 告警之后，就可以条件激活时，触发 `Intention` 的提示，这样就可以使用 `alt + enter` 直接自动生成UID了。
话说回来，有一个叫`GenerateSerialVersionUID`的插件也是专门用来做这件事的，不过相比较之下还是直接改下配置更

 ##  常用插件

1. #### .ignore

2. #### Free MyBatis plugin

3. #### IDEA Restart

4. #### Lombok Plugin

5. #### Maven Helper

6. #### Translation

7. #### String Manipulation

8. #### JRebel for IntelliJ

9. #### FindBugs-IDEA

10. #### Alibaba Java Coding Guidelines

11. #### Gsonformat

12. #### Mongo Explorer

13. #### ASM

14. #### Generate Fluent Interface

15. ### VisualVM Launcher

16. ### MyBatisCodeHelperPro

17. ### IDE Features Trainer



##  IntelliJ  IDEA JVM参数调优

> <http://xxfox.perfma.com/jvm/generate> 
个人电脑配置：win7+64位+8G内存+4核CPU:
```

-Xmx2688m
-Xmx2688m
-XX:ReservedCodeCacheSize=340m
-XX:+UseConcMarkSweepGC
-XX:SoftRefLRUPolicyMSPerMB=50
-ea
-Dsun.io.useCanonCaches=false
-Djava.net.preferIPv4Stack=true
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow

```

## IntelliJ  IDEA  License  Server 

> http://jtb.whoniverse.eu/

## IntelliJ IDEA 官方权威快捷键

> https://www.jetbrains.com/idea/docs/IntelliJIDEA_ReferenceCard.pdf 

> 站在巨人的肩膀上 你会走的更远
> 本教程部分内容引起以下网址
> https://dancon.gitbooks.io/intellij-idea/content/this-tutorial-the-end.html 感谢原作者的辛勤付出

