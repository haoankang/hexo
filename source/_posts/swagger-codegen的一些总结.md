---
title: swagger-codegen的一些总结
date: 2019-05-16 15:08:02
categories:
tags: swagger
---

## 1. swagger是什么？
  swagger是用来设计、构建、文档、合作、管理api的工具，可以生成、描述、调用和可视化Restful风格
的web服务，方便前后端分离；早先吾使用来描述api文档，生成文档接口页面展示，后来是先写文档，利用
swagger-codegen生成代码；
## 2. swagger-codegen的几种使用方式.
  官方github地址是：https://github.com/swagger-api/swagger-codegen
可以看到，主要有四个modules：
* swagger-generator : 用来生成swagger页面，利用相关注释描述.
* swagger-codegen :  利用swagger文档生成接口代码.
* swagger-codegen-cli : swagger-codegen的命令行可执行jar包.
* swagger-codegen-maven-plugin ： swagger-codegen的maven插件.
  swagger-codegen-cli的基本用法，下载jar包，在命令行中如下方式：
  ```
   java -jar swagger-codegen-cli-2.2.3.jar generate -i swagger-api.yml -l spring -o /java --model-package cn.com.controller.model --api-package cn.com.controller.api
  ```
  swagger-codegen直接引入maven依赖，可以以工具类的方式生成代码.
  swagger-codegen-maven-plugin以maven插件方式工作，这里主要介绍这种;
## 3. swagger-codegen-maven-plugin的一些坑.
1. 在pom中直接引入插件.
    ```
    <plugin>
        <groupId>io.swagger</groupId>
        <artifactId>swagger-codegen-maven-plugin</artifactId>
        <version>2.4.5</version>
        <executions>
            <execution>
                <goals>
                    <goal>generate</goal>
                </goals>
                <configuration>
                    <inputSpec>${project.basedir}/src/main/resources/swagger-api.yml</inputSpec>
                    <output>${project.basedir}</output>
                    <language>spring</language>
                    <library>spring-boot</library>
                    <modelPackage>${default.package}.model</modelPackage>
                    <apiPackage>${default.package}.api</apiPackage>
                    <invokerPackage>${default.package}</invokerPackage>
                    <generateApiTests>false</generateApiTests>
                    <generateModelTests>false</generateModelTests>
                    <generateApiDocumentation>true</generateApiDocumentation>
                    <generateModelDocumentation>true</generateModelDocumentation>
                    <generateSupportingFiles>true</generateSupportingFiles>
                    <configOptions>
                        <interfaceOnly>false</interfaceOnly>
                        <dateLibrary>joda</dateLibrary>
                        <java8>true</java8>
                        <!--<sourceFolder>src/gen/java/main</sourceFolder>-->
                    </configOptions>
                </configuration>
            </execution>
        </executions>
    </plugin>
    ```
2. 注意需要maven中配置{default.package}变量，然后执行mvn clean compile,生成相应代码;
3. 使用起来比较简单，主要是官方文档匮乏，可配置参数通过下述方式查询帮助：
    * 可以直接在idea的命令行中使用"mvn swagger-codegen:help -Ddetail=true"方式;
    * 可以下载**相应版本**的swagger-codegen-cli.jar包，"java -jar ***.jar config-help -l spring"方式，推荐这种方式，比较方便全
4. 利用好.swagger-codegen-ignore文档，例如pom.xml，因为swagger默认会自动生成覆盖pom.xml，需要在.swagger-codegen-ignore文档中添加
   过滤，包括之后不想覆盖的ControllerApi也可以在这里添加过滤；
5. interfaceOnly参数第一次生成时建议false，可以帮助生成swagger的辅助文件，例如/swagger-ui.html映射；之后建议true，只生成更新api.