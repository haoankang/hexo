---
title: '''oracle系列——wallet使用'''
date: 2019-12-21 13:00:54
categories: oracle
tags: oracle,wallet
---

**1.前言**
这几天客户项目上用到oracle wallet，调研对接和使用问题，做一下总结.

**2.oracle wallet作用简介**
oracle wallet主要用于安全方面，可以免密登陆，从而不暴露用户名和密码；一般这种不用用户名密码的，肯定是用证书了，so，这就是oracle wallet；

**3.基本使用**
这里不建议使用docker容器的oracle，因为测试的大部分oracle镜像都缺胳膊少腿的，所以还是自己搭一个oracle服务器吧；
引用一个搭建步骤：https://www.acgist.com/article/497.html，可能会报错，直接搜相关问题解决；

然后就是创建和使用wallet.（服务端创建）
>* 创建wallet:新建一个目录，执行命令.————mkstore -wrl <wallet_location> -create
>* 创建客户端连接服务端的网络连接串，每个连接串对应一个数据库用户，在$ORACLE_HOME/network/admin/tnsnames.ora文件中.
```sql
myorcl =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.100.1.111)(PORT = 1521))
    (CONNECT_DATA=
      (SERVER = DEDICATED)
      (SERVICE_NAME=acgist)
    )
  )

```
>* 配置wallet添加用户认证信息.————mkstore -wrl <wallet_location> -createCredential myorcl 
>* 在$ORACLE_HOME/network/admin/sqlnet.ora中配置<wallet_location>；
```sql
WALLET_LOCATION=(SOURCE=(METHOD=FILE)(METHOD_DATA=(DIRECTORY=/home/oracle/wallet_111)))
SQLNET.WALLET_OVERRIDE = TRUE
```
>* 然后可以在服务端先测试一下: sqlplus /@myorcl，如果能直连成功，则说明没问题；

在客户机上免密登陆.
>* 需要先安装oracle的客户端，设置相关环境变量；(客户端和服务端不一定要匹配，我这里服务端是11.2.0.1.0，客户端是12.2.0.1)
>* 把服务端的wallet包拷贝到客户端，假设路径还是/home/oracle/wallet_111
>* 在$ORACLE_HOME/network/admin/下新建tnsnames.ora和sqlnet.ora，内容和要使用的数据库保持一致；
>* 验证: sqlplus /@myorcl

**4.应用程序使用wallet——在安装oracle客户端的机器上部署**
如果客户机上已经安装了oracle客户端，则可以直接使用oci连接，从而无需改造程序；例如笔者用的spring boot，可以直接这样修改数据源配置：
```sql
spring:
  datasource:
    driver-class-name: oracle.jdbc.driver.OracleDriver
    url: jdbc:oracle:oci://@myorcl
    username:
    password:
```
这里需要特别注意的是：ojdbc的包一定要与客户端版本一致，例如笔者环境jdk1.8，oracle客户端12.2.0.1，那么要引用下面依赖：
```sql
 <!-- https://mvnrepository.com/artifact/com.oracle/ojdbc8 -->
        <dependency>
            <groupId>com.oracle</groupId>
            <artifactId>ojdbc8</artifactId>
            <version>12.2.0.1</version>
            <scope>provided</scope>
        </dependency>
```

**5.应用程序使用wallet——在未安装oracle客户端的机器上部署**
客户机未安装oracle客户端，则需要使用thin连接，这时需要按照以下步骤改造程序：
>* 在客户机上新建wallet目录并把服务端的wallet拷贝过来，假设目录：/home/oracle/wallet_111，并且在此目录下新建tnsnames.ora并配置使用的数据库.
>* 应用程序中引入以下依赖包.
```sql
 <!-- https://mvnrepository.com/artifact/com.oracle/ojdbc8 -->
        <dependency>
            <groupId>com.oracle</groupId>
            <artifactId>ojdbc8</artifactId>
            <version>12.2.0.1</version>
            <scope>provided</scope>
        </dependency>

        <!-- https://mvnrepository.com/artifact/com.oracle.ojdbc/oraclepki -->
        <dependency>
            <groupId>com.oracle.ojdbc</groupId>
            <artifactId>oraclepki</artifactId>
            <version>19.3.0.0</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/com.oracle.ojdbc/osdt_cert -->
        <dependency>
            <groupId>com.oracle.ojdbc</groupId>
            <artifactId>osdt_cert</artifactId>
            <version>19.3.0.0</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/com.oracle.ojdbc/osdt_core -->
        <dependency>
            <groupId>com.oracle.ojdbc</groupId>
            <artifactId>osdt_core</artifactId>
            <version>19.3.0.0</version>
        </dependency>
```
>* 应用程序中自定义DataSource.
```sql
@Bean
    public DataSource dataSource(){
        OracleDataSource dataSource = null;
        try {
            dataSource = new OracleDataSource();
            Properties properties = new Properties();
            String oracle_net_wallet_location = "/home/oracle/wallet_111";  //目录下应包含wallet的相关文件和tnsnames.ora
            properties.put("oracle.net.tns_admin", oracle_net_wallet_location);
            properties.put("oracle.net.wallet_location", "(source=(method=file)(method_data=(directory="+oracle_net_wallet_location+")))");
            dataSource.setConnectionProperties(properties);
            dataSource.setURL("jdbc:oracle:thin:@myorcl");
        }catch (Exception e){
            e.printStackTrace();
        }
        return dataSource;
    }
```
>* 以上就可以了；
>* 需要注意一些特殊引用，例如quartz.quartz定义数据源有三种方式:配置文件、JNDI、自定义ConnectionProvider；这里笔者选用的第三种方式集成，具体可以查看quartz官方文档，不再赘述.