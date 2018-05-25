---
title: Intellij IDEA 2017集成MyBatis三剑客
date: 2018-05-25 16:44:00
categories: mybatis系列
tags: [MyBatis-Generate,Mybatis-Plus,MyBatis-PageHelper]
description: Intellij IDEA 2017集成MyBatis三剑客（MyBatis-Generate、Mybatis Plus、MyBatis-PageHelper）
---

MyBatis三剑客指的是：`MyBatis-Generate`、`Mybatis Plus`、`MyBatis-PageHelper`

## MyBatis-Generate

使用 Mybatis Generator 这个maven插件来快速生成 Dao 类, mapper 配置文件和 Model 类.

> MyBatis Generator（简称MBG）是MyBatis的代码生成器.可以自动查询数据库中的所有表,然后生成可以访问表的基础对象类型.解决了对数据库操作有最大影响的一些简单的CRUD增删改查操作,但是仍需要对联合查询和存储过程手写SQL语句和对象.

[MyBatis Generator中文文档](https://www.kancloud.cn/wizardforcel/java-opensource-doc/152983)

#### 1.在pom文件中添加插件
```

<plugin>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-maven-plugin</artifactId>
    <version>1.3.2</version>
    <configuration>
         <verbose>true</verbose>
         <overwrite>true</overwrite>
    </configuration>
</plugin>

```

#### 2.在maven项目中的resource中创建xml文件与properties资源文件

![](/assets/img/1.png)

* 名称可以随便取，这里以 generatorConfig.xml 为名
* 资源文件为 datasource.properties 文件，这个可不要，这里用是因为方便管理而已

#### 3.配置generatorConfig.xml与资源文件

`generatorConfig.xml`

```

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <!--导入属性配置-->
    <properties resource="datasource.properties"></properties>

    <!--指定特定数据库的jdbc驱动jar包的位置-->
    <classPathEntry location="${db.driverLocation}"/>

    <context id="default" targetRuntime="MyBatis3">

        <!-- optional，旨在创建class时，对注释进行控制 -->
        <commentGenerator>
            <property name="suppressDate" value="true"/>
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>

        <!--jdbc的数据库连接 -->
        <jdbcConnection
                driverClass="${db.driverClassName}"
                connectionURL="${db.url}"
                userId="${db.username}"
                password="${db.password}">
        </jdbcConnection>


        <!-- 非必需，类型处理器，在数据库类型和java类型之间的转换控制 -->
        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>


        <!-- Model模型生成器,用来生成含有主键key的类，记录类 以及查询Example类
            targetPackage     指定生成的model生成所在的包名
            targetProject     指定在该项目下所在的路径
        -->
        <!--<javaModelGenerator targetPackage="com.mmall.pojo" targetProject=".\src\main\java">-->
        <javaModelGenerator targetPackage="org.mmall.pojo" targetProject="./src/main/java">
            <!-- 是否允许子包，即targetPackage.schemaName.tableName -->
            <property name="enableSubPackages" value="false"/>
            <!-- 是否对model添加 构造函数 -->
            <property name="constructorBased" value="true"/>
            <!-- 是否对类CHAR类型的列的数据进行trim操作 -->
            <property name="trimStrings" value="true"/>
            <!-- 建立的Model对象是否 不可改变  即生成的Model对象不会有 setter方法，只有构造方法 -->
            <property name="immutable" value="false"/>
        </javaModelGenerator>

        <!--mapper映射文件生成所在的目录 为每一个数据库的表生成对应的SqlMap文件 -->
        <!--<sqlMapGenerator targetPackage="mappers" targetProject=".\src\main\resources">-->
        <sqlMapGenerator targetPackage="mappers" targetProject="./src/main/resources">
            <property name="enableSubPackages" value="false"/>
        </sqlMapGenerator>

        <!-- 客户端代码，生成易于使用的针对Model对象和XML配置文件 的代码
                type="ANNOTATEDMAPPER",生成Java Model 和基于注解的Mapper对象
                type="MIXEDMAPPER",生成基于注解的Java Model 和相应的Mapper对象
                type="XMLMAPPER",生成SQLMap XML文件和独立的Mapper接口
        -->

        <!-- targetPackage：mapper接口dao生成的位置 -->
        <!--<javaClientGenerator type="XMLMAPPER" targetPackage="com.mmall.dao" targetProject=".\src\main\java">-->
        <javaClientGenerator type="XMLMAPPER" targetPackage="org.mmall.dao" targetProject="./src/main/java">
            <!-- enableSubPackages:是否让schema作为包的后缀 -->
            <property name="enableSubPackages" value="false" />
        </javaClientGenerator>


        <table tableName="mmall_shipping" domainObjectName="Shipping" enableCountByExample="false" enableUpdateByExample="false" enableDeleteByExample="false" enableSelectByExample="false" selectByExampleQueryId="false"></table>
        <table tableName="mmall_cart" domainObjectName="Cart" enableCountByExample="false" enableUpdateByExample="false" enableDeleteByExample="false" enableSelectByExample="false" selectByExampleQueryId="false"></table>
        <table tableName="mmall_cart_item" domainObjectName="CartItem" enableCountByExample="false" enableUpdateByExample="false" enableDeleteByExample="false" enableSelectByExample="false" selectByExampleQueryId="false"></table>
        <table tableName="mmall_category" domainObjectName="Category" enableCountByExample="false" enableUpdateByExample="false" enableDeleteByExample="false" enableSelectByExample="false" selectByExampleQueryId="false"></table>
        <table tableName="mmall_order" domainObjectName="Order" enableCountByExample="false" enableUpdateByExample="false" enableDeleteByExample="false" enableSelectByExample="false" selectByExampleQueryId="false"></table>
        <table tableName="mmall_order_item" domainObjectName="OrderItem" enableCountByExample="false" enableUpdateByExample="false" enableDeleteByExample="false" enableSelectByExample="false" selectByExampleQueryId="false"></table>
        <table tableName="mmall_pay_info" domainObjectName="PayInfo" enableCountByExample="false" enableUpdateByExample="false" enableDeleteByExample="false" enableSelectByExample="false" selectByExampleQueryId="false"></table>
        <table tableName="mmall_product" domainObjectName="Product" enableCountByExample="false" enableUpdateByExample="false" enableDeleteByExample="false" enableSelectByExample="false" selectByExampleQueryId="false">
            <columnOverride column="detail" jdbcType="VARCHAR" />
            <columnOverride column="sub_images" jdbcType="VARCHAR" />
        </table>
        <table tableName="mmall_user" domainObjectName="User" enableCountByExample="false" enableUpdateByExample="false" enableDeleteByExample="false" enableSelectByExample="false" selectByExampleQueryId="false"></table>


        <!-- geelynote mybatis插件的搭建 -->
    </context>
</generatorConfiguration>

```

`datasource.properties`

```
db.driverLocation =  E:\\jre\\mysql-connector-java-5.1.6.jar
db.driverClassName = com.mysql.jdbc.Driver
db.url = jdbc:mysql://127.0.0.1:3306/mmall?characterEncoding=utf-8
db.username = root
db.password = root

```
#### 4.运行

* 方式一：

![](/assets/img/2.png)

* 方式二：

    * 在Intellij IDEA添加一个“Run运行”，这个少用 略
    

## Mybatis Plus

> Mybatis Plus（简称MP）是一个 Mybatis 的增强工具，在 Mybatis 的基础上只做增强不做改变，为简化开发、提高效率而生。

> [Mybatis Plus中文文档](http://mp.baomidou.com/#/)


#### 1.功能

* 提供Mapper接口与配置文件中对应SQL的导航
* 编辑XML文件时自动补全
* 根据Mapper接口, 使用快捷键生成xml文件及SQL标签
* ResultMap中的property支持自动补全，支持级联(属性A.属性B.属性C)
* 快捷键生成@Param注解
* XML中编辑SQL时, 括号自动补全
* XML中编辑SQL时, 支持参数自动补全(基于@Param注解识别参数)
* 自动检查Mapper XML文件中ID冲突
* 自动检查Mapper XML文件中错误的属性值
* 支持Find Usage
* 支持重构从命名
* 支持别名
* 自动生成ResultMap属性
* 快捷键: Option + Enter(Mac) | Alt + Enter(Windows)

#### 2.安装与破解

* 这是一个IDE插件，目前是收费的，这里我用的是Intellij IDEA的

![](/assets/img/3.png)

需要破解

## MyBatis-PageHelper

> 这个一个通用的分页插件,使用时 Mybatis 最低版本不能低于3.3
> 原理：通过aop再截获我们执行sql的时候把相关的数据再执行一次

[GitHub地址](https://github.com/pagehelper/Mybatis-PageHelper)

#### 1.在pom文件中添加依赖

```
<!-- mybatis pager -->
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper</artifactId>
    <version>4.1.0</version>
</dependency>

<dependency>
    <groupId>com.github.miemiedev</groupId>
    <artifactId>mybatis-paginator</artifactId>
    <version>1.2.17</version>
</dependency>

<dependency>
    <groupId>com.github.jsqlparser</groupId>
    <artifactId>jsqlparser</artifactId>
    <version>0.9.4</version>
</dependency>

```

#### 2.在spring配置文件内添加配置

```
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mapperLocations" value="classpath*:mappers/*Mapper.xml"></property>

    <!-- 分页插件 -->
    <property name="plugins">
        <array>
            <bean class="com.github.pagehelper.PageHelper">
                <property name="properties">
                    <value>
                        <!-- 数据库方言 -->
                        dialect=mysql
                    </value>
                </property>
            </bean>
        </array>
    </property>
</bean>

```

> 本文引用自：https://www.godql.com/blog/2017/06/03/Intellij_IDEA_Integration_MyBatis/