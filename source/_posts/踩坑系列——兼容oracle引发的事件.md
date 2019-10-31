---
title: 踩坑系列——兼容oracle引发的事件
date: 2019-10-31 10:34:11
categories: 踩坑系列
tags: spring boot,mybatis,mybatis-generator
---

前两天一个项目要做数据库兼容，本身是mybatis+mysql的，要兼容oracle；这期间为了使项目更好维护，折腾了一两天，
也遇到一些坑，学到一些，这里记录一下；

##1. mybatis-generator插件选择的纠结.
   之前使用的是idea插件库中的MyBatisCodeHelper，挺好用的一个插件，mysql时用起来异常顺手，但是本人使用生成oracle时
有点问题，再加上不好做定制，而且收费的，因此打算找个更好用的工具；这里说明一下，基本所有的mybatis-generator工具底层
都是使用官方的mybatis-generator的，[文档地址这里](https://mybatis.org/generator/index.html)
<br>
   一般有三种方式：一种是集成到idea等开发工具种的plugin，例如上面的MyBatisCodeHelper，最大优点是方便，
简单使用时推荐；第二种是做成单独的应用，可以有界面也可以没界面，典型的是[mybatis-generator-gui](https://github.com/zouzg/mybatis-generator-gui)，
优点就是界面不错，也可保存配置，用起来也算方便，但是定制比较麻烦，我甚至没找到配置文件233；第三种就是
mybatis-generator-maven-plugin了，集成到maven中的，如果维护多个项目，且数据库设计不统一，那这种挺
合适的，毕竟每个项目唯一配置，定制化也比较方便；接下来主要是关于第三种方式的一些踩坑和进阶；

##2. mybatis-generator-maven-plugin的基本使用方式.
1. pom中引入plugin；
```
 <plugin>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-maven-plugin</artifactId>
    <version>1.3.7</version>
    <configuration>
        <!-- 配置文件 -->
        <!--<configurationFile>src/main/resources/mybatisGenerate/mysql/generatorConfig.xml</configurationFile>-->
        <configurationFile>src/main/resources/mybatisGenerate/oracle/generatorConfig.xml</configurationFile>
        <!-- 允许移动和修改 -->
        <verbose>true</verbose>
        <overwrite>true</overwrite>
    </configuration>
    <executions>
        <execution>
            <id>Generate MyBatis Artifacts</id>
            <phase>deploy</phase>
            <goals>
                <goal>generate</goal>
            </goals>
        </execution>
    </executions>
    <dependencies>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.47</version>
        </dependency>
        <dependency>
            <groupId>com.oracle</groupId>
            <artifactId>ojdbc8</artifactId>
            <version>12.1.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-core</artifactId>
            <version>1.3.7</version>
        </dependency>
    </dependencies>
</plugin>
```
注意mysql和oracle的驱动包要与自己使用的版本对应；

2. 创建对应<configurationFile>配置文件，例如：
```
<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <properties resource="mybatisGenerate/oracle/generator.properties"/>
    <context id="scheduler-admin" defaultModelType="flat" targetRuntime="MyBatis3">

        <plugin type="org.mybatis.generator.plugins.EqualsHashCodePlugin"/>
        <plugin type="org.mybatis.generator.plugins.SerializablePlugin"/>
        <plugin type="org.mybatis.generator.plugins.CaseInsensitiveLikePlugin"/>
        <!--覆盖生成XML文件-->
        <plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin"/>

        <commentGenerator>
            <property name="suppressDate" value="true"/>
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>
        <jdbcConnection driverClass="${jdbc.driverClassName}"
                        connectionURL="${jdbc.jdbcUrl}"
                        userId="${jdbc.username}"
                        password="${jdbc.password}">
        </jdbcConnection>
        <javaTypeResolver type="com.mybatis.generator.MyJavaTypeResolver">
            <property name="forceBigDecimals" value="false" />
        </javaTypeResolver>
        <javaModelGenerator targetPackage="xxx.domain" targetProject="./src/main/java">
            <property name="enableSubPackages" value="true"/>
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>
        <sqlMapGenerator targetPackage="mapperxml.oracle" targetProject="./src/main/resources/">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>
        <javaClientGenerator type="XMLMAPPER" targetPackage="xxx.mapper"
                             targetProject="./src/main/java">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>

        <table tableName="ml_scheduler_group" domainObjectName="MlSchedulerGroup">
            <property name="ignoreQualifiersAtRuntime" value="true"/>
            <generatedKey column="ID" sqlStatement="JDBC"/>
        </table>
    </context>
</generatorConfiguration>
```

3. 创建数据库配置文件generator.properties，内容如下：
```
jdbc.driverClassName=oracle.jdbc.driver.OracleDriver
jdbc.jdbcUrl=jdbc:oracle:thin:@10.100.1.210:2521:xe
jdbc.username=smartlearning_pretest
jdbc.password=123456
```

4. 使用mvn命令或者idea集成mvn插件生成对应的文件；

##3. 一些细节、坑、进阶.
1. 数据库驱动的版本最好对应上使用的版本，不然会导致无法识别主键；这里说最好，也就是有替代方法：<table>
标签中指定schema或者catalog等来定位数据库，例如mysql可以指定schema="yourDataBase"，还有种方法
可以参考https://mybatis.org/generator/usage/mysql.html；但是建议直接使用对应版本的驱动jar，
免得后续会有一些奇奇怪怪的问题；

2. mybatis-generator的核心就是generatorConfig.xml配置文件，这里的配置详解可以参考[官网地址](https://mybatis.org/generator/index.html)，
这里就说明一点：主键一般是自动识别的，<generatedKey>可以配置主键生成方式，我这里因为mysql主键是自增的，
oracle主键是sequence+trigger自动生成的，因此这里这样配置；如果不配做，在生成的insert等sql中，
会需要写入主键导致问题，这里需要看自己采用哪种主键生成方式了；

3. 定制化. 这里定制化主要是数据类型的全局定制javaType和jdbcType，当然你可以在每个<table>中使用<columnOverride>
来定制，但是因为每个table都需要写，如果数据表较多，会很麻烦和臃肿；步骤如下：
* 首先自定义MyJavaTypeResolver,最好在一个单独的工具module中，因为要install本地作为mybatis-generator-maven-plugin的依赖包。
新建一个工程，pom核心配置如下：
```
    <artifactId>xxx-common</artifactId>
    <version>1.0-SNASHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-core</artifactId>
            <version>1.3.7</version>
        </dependency>
    </dependencies>
```

* 自定义MyJavaTypeResolver如下：
```
public class MyJavaTypeResolver extends JavaTypeResolverDefaultImpl {

    public MyJavaTypeResolver(){
        super();
        super.typeMap.put(Types.NUMERIC, new JavaTypeResolverDefaultImpl.JdbcTypeInformation("INTEGER", new FullyQualifiedJavaType(Integer.class.getName())));
    }

//    protected FullyQualifiedJavaType calculateBigDecimalReplacement(IntrospectedColumn column, FullyQualifiedJavaType defaultType) {
//        FullyQualifiedJavaType answer;
//        if (column.getScale() <= 0 && column.getLength() <= 18 && !this.forceBigDecimals) {
//            if (column.getLength() > 11) {
//                answer = new FullyQualifiedJavaType(Long.class.getName());
//            } else {
//                answer = new FullyQualifiedJavaType(Integer.class.getName());
//            }
//        } else {
//            answer = defaultType;
//        }
//
//        return answer;
//    }
}
```
因为oracle数据表使用的数值类型都是Number，mysql中使用的是integer，因此这里这样配置，目的是把所有
number类型映射成integer，从而统一；这里顺便介绍下spring boot+mybatis+mysql/oracle方案，mapper和
model统一的，sql所在的xml是分开的，spring boot中可以直接用配置文件指定xml位置，自定义sql直接放在mapper
目录下通用，所以也需要注意兼容问题；

* 将xxx-common项目mvn install到本地即可，mybatis-generator-maven-plugin中引入相关依赖包，
在generatorConfig.xml中配置：
```
<javaTypeResolver type="com.mybatis.generator.MyJavaTypeResolver">
    <property name="forceBigDecimals" value="false" />
</javaTypeResolver>
```

* 这样就可以根据需要修改了，而且mvn插件还支持debug功能！

##4. 其他的一些.
1. pagehelper分页本身是支持mysql和oracle的，配置即可；

2. 自定义sql时，注意返回的类型如果是object，默认映射的是原始类型，例如oracle的number返回BigDecimal，
这个需要注意判断；

3. mysql和oracle的语法还是有些区别的；例如都是mybatis-generator自动生成的，像select * from
T where xx in(list),list.size=1，但是list里是空对象，这种情况下，mysql不会报错，而oracle会
报错，而且报错是类型错误；

4. 数据库兼容这种，因为数据库不一样，很多报错内容也都没法让我们直观了了解错误，因此需要远程调试；
-Xdebug -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005