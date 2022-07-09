<h1 style="color:skyblue;text-align:center">---SpringCloud学习笔记---</h1>



----

# 服务架构演变

## 单体架构

单体架构，就是将业务的所有功能集中在一个项目中开发，打成一个包部署

优点：

* 架构简单
* 部署成本低

缺点：

* 耦合度高





## 分布式架构

分布式架构，根据业务功能对系统进行拆分，每个业务模块作为独立项目开发，称为一个服务

比如一个应用拆分为订单模块、用户模块、商品功能模块、支付模块。

优点：

* 降低服务耦合
* 有利于服务升级拓展



分布式架构的要考虑的问题：

* 服务拆分粒度如何？
* 服务集群地址如何维护？
* 服务之间如何实现远程调用？
* 服务健康状态如何感知？





## 微服务

微服务是一种经过良好架构设计的分布式架构方案，微服务架构特征：

* 单一职责：微服务拆分粒度更小，每一个服务都对应唯一的业务能力，做到单一职责，避免重复业务开发
* 面向服务：微服务对外暴露业务接口
* 自治：团队独立、技术独立、数据独立、部署独立
* 隔离性强：服务调用做好隔离、容错、降级，避免出现级联问题



优点：

* 拆分粒度更小
* 服务更独立
* 耦合度更低



缺点：

* 架构非常复杂
* 运维、监控、部署难度提高







# SpringCloud

SpringCloud是目前国内使用最广泛的微服务框架

官网：[Spring Cloud](https://spring.io/projects/spring-cloud)

SpringCloud集成了各种微服务功能组件，并基于SpringBoot实现了这些组件的自动装配，从而提供了良好的开 箱即用体验



服务注册发现：

* Eureka
* Nacos
* Consul



统一配置管理：

* SpringCloudConfig
* Nacos



服务远程调用：

* OpenFeign
* Dubbo



统一网关路由：

* SpringCloudGateway
* Zuul



服务链路监控：

* Zipkin
* Sleuth



流控、降级、保护：

* Hystix
* Sentinel



SpringCloud与SpringBoot的版本兼容关系如下：

|                        Release Train                         |             Boot Version              |
| :----------------------------------------------------------: | :-----------------------------------: |
| [2021.0.x](https://github.com/spring-cloud/spring-cloud-release/wiki/Spring-Cloud-2021.0-Release-Notes) aka Jubilee |                 2.6.x                 |
| [2020.0.x](https://github.com/spring-cloud/spring-cloud-release/wiki/Spring-Cloud-2020.0-Release-Notes) aka Ilford | 2.4.x, 2.5.x (Starting with 2020.0.3) |
| [Hoxton](https://github.com/spring-cloud/spring-cloud-release/wiki/Spring-Cloud-Hoxton-Release-Notes) |   2.2.x, 2.3.x (Starting with SR5)    |
| [Greenwich](https://github.com/spring-projects/spring-cloud/wiki/Spring-Cloud-Greenwich-Release-Notes) |                 2.1.x                 |
| [Finchley](https://github.com/spring-projects/spring-cloud/wiki/Spring-Cloud-Finchley-Release-Notes) |                 2.0.x                 |
| [Edgware](https://github.com/spring-projects/spring-cloud/wiki/Spring-Cloud-Edgware-Release-Notes) |                 1.5.x                 |
| [Dalston](https://github.com/spring-projects/spring-cloud/wiki/Spring-Cloud-Dalston-Release-Notes) |                 1.5.x                 |



Spring Cloud Dalston、Edgware、Finchley和Greenwich都已达到生命终止状态，不再受支持。



学习的版本是 Hoxton.SR10，因此对应的SpringBoot版本是2.3.x版本







# 服务拆分及远程调用

服务拆分注意事项：

* 单一职责：不同微服务，不要重复开发相同业务
* 数据独立：不要访问其它微服务的数据库
* 面向服务：将自己的业务暴露为接口，供其它微服务调用





## 项目准备



spring_cloud_demo：父工程，管理依赖

- order_service：订单微服务，负责订单相关业务
- user_service：用户微服务，负责用户相关业务



要求：

- 订单微服务和用户微服务都必须有各自的数据库，相互独立
- 订单服务和用户服务都对外暴露Restful的接口
- 订单服务如果需要查询用户信息，只能调用用户服务的Restful接口，不能查询用户数据库



### 导入sql文件

创建数据库cloud_order

导入以下数据：

```sql

SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for tb_order
-- ----------------------------
DROP TABLE IF EXISTS `tb_order`;
CREATE TABLE `tb_order`  (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '订单id',
  `user_id` bigint(20) NOT NULL COMMENT '用户id',
  `name` varchar(100) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '商品名称',
  `price` bigint(20) NOT NULL COMMENT '商品价格',
  `num` int(10) NULL DEFAULT 0 COMMENT '商品数量',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `username`(`name`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 109 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Compact;

-- ----------------------------
-- Records of tb_order
-- ----------------------------
INSERT INTO `tb_order` VALUES (101, 1, 'Apple 苹果 iPhone 12 ', 699900, 1);
INSERT INTO `tb_order` VALUES (102, 2, '雅迪 yadea 新国标电动车', 209900, 1);
INSERT INTO `tb_order` VALUES (103, 3, '骆驼（CAMEL）休闲运动鞋女', 43900, 1);
INSERT INTO `tb_order` VALUES (104, 4, '小米10 双模5G 骁龙865', 359900, 1);
INSERT INTO `tb_order` VALUES (105, 5, 'OPPO Reno3 Pro 双模5G 视频双防抖', 299900, 1);
INSERT INTO `tb_order` VALUES (106, 6, '美的（Midea) 新能效 冷静星II ', 544900, 1);
INSERT INTO `tb_order` VALUES (107, 2, '西昊/SIHOO 人体工学电脑椅子', 79900, 1);
INSERT INTO `tb_order` VALUES (108, 3, '梵班（FAMDBANN）休闲男鞋', 31900, 1);

SET FOREIGN_KEY_CHECKS = 1;
```



创建数据库cloud_user

导入以下数据：

```sql

SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for tb_user
-- ----------------------------
DROP TABLE IF EXISTS `tb_user`;
CREATE TABLE `tb_user`  (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `username` varchar(100) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '收件人',
  `address` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '地址',
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `username`(`username`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 109 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Compact;

-- ----------------------------
-- Records of tb_user
-- ----------------------------
INSERT INTO `tb_user` VALUES (1, '柳岩', '湖南省衡阳市');
INSERT INTO `tb_user` VALUES (2, '文二狗', '陕西省西安市');
INSERT INTO `tb_user` VALUES (3, '华沉鱼', '湖北省十堰市');
INSERT INTO `tb_user` VALUES (4, '张必沉', '天津市');
INSERT INTO `tb_user` VALUES (5, '郑爽爽', '辽宁省沈阳市大东区');
INSERT INTO `tb_user` VALUES (6, '范兵兵', '山东省青岛市');

SET FOREIGN_KEY_CHECKS = 1;
```



因为没有多个数据库，所以只能用一个数据库的多个库代替



```sh
mysql> show databases
    -> ;
+--------------------+
| Database           |
+--------------------+
| cloud_order        |
| cloud_user         |
| hotel              |
| information_schema |
| mysql              |
| performance_schema |
| sakila             |
| shop               |
| student            |
| student1           |
| student_test       |
| sys                |
| test               |
| tx                 |
| world              |
+--------------------+
15 rows in set (0.00 sec)

mysql>
```



```sh
mysql> use cloud_order
Database changed
mysql> show tables;
+-----------------------+
| Tables_in_cloud_order |
+-----------------------+
| tb_order              |
+-----------------------+
1 row in set (0.00 sec)

mysql> select * from tb_order;
+-----+---------+----------------------------------+--------+------+
| id  | user_id | name                             | price  | num  |
+-----+---------+----------------------------------+--------+------+
| 101 |       1 | Apple 苹果 iPhone 12             | 699900 |    1 |
| 102 |       2 | 雅迪 yadea 新国标电动车          | 209900 |    1 |
| 103 |       3 | 骆驼（CAMEL）休闲运动鞋女        |  43900 |    1 |
| 104 |       4 | 小米10 双模5G 骁龙865            | 359900 |    1 |
| 105 |       5 | OPPO Reno3 Pro 双模5G 视频双防抖 | 299900 |    1 |
| 106 |       6 | 美的（Midea) 新能效 冷静星II     | 544900 |    1 |
| 107 |       2 | 西昊/SIHOO 人体工学电脑椅子      |  79900 |    1 |
| 108 |       3 | 梵班（FAMDBANN）休闲男鞋         |  31900 |    1 |
+-----+---------+----------------------------------+--------+------+
8 rows in set (0.00 sec)

mysql>
```



```sh
mysql> use cloud_user
Database changed
mysql> show tables;
+----------------------+
| Tables_in_cloud_user |
+----------------------+
| tb_user              |
+----------------------+
1 row in set (0.00 sec)

mysql> select * from tb_user;
+----+----------+--------------------+
| id | username | address            |
+----+----------+--------------------+
|  1 | 柳岩     | 湖南省衡阳市       |
|  2 | 文二狗   | 陕西省西安市       |
|  3 | 华沉鱼   | 湖北省十堰市       |
|  4 | 张必沉   | 天津市             |
|  5 | 郑爽爽   | 辽宁省沈阳市大东区 |
|  6 | 范兵兵   | 山东省青岛市       |
+----+----------+--------------------+
6 rows in set (0.00 sec)

mysql>
```





### 创建项目



创建项目 ，名字为spring_cloud_demo



pom.xml文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>


    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.9.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <groupId>mao</groupId>
    <artifactId>spring_cloud_demo</artifactId>
    <version>0.0.1</version>
    <packaging>pom</packaging>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>11</java.version>
    </properties>

    <modules>
        <module>user_service</module>
        <module>order_service</module>
    </modules>


    <dependencyManagement>
        <dependencies>
            <!--spring-cloud项目依赖-->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Hoxton.SR10</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!--spring-boot druid连接池依赖-->
            <dependency>
                <groupId>com.alibaba</groupId>
                <artifactId>druid-spring-boot-starter</artifactId>
                <version>1.2.8</version>
            </dependency>

            <!--mysql依赖 spring-boot-->
            <dependency>
                <groupId>mysql</groupId>
                <artifactId>mysql-connector-java</artifactId>
                <version>8.0.27</version>
                <scope>runtime</scope>
            </dependency>

            <!--spring-boot mybatis依赖-->
            <dependency>
                <groupId>org.mybatis.spring.boot</groupId>
                <artifactId>mybatis-spring-boot-starter</artifactId>
                <version>2.2.2</version>
            </dependency>
        </dependencies>
    </dependencyManagement>


    <dependencies>
        <!--spring-boot lombok-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.20</version>
            <optional>true</optional>
        </dependency>
    </dependencies>


</project>
```





### 创建子模块

创建一个子模块，模块名为user_service

pom.xml文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>


    <parent>
        <groupId>mao</groupId>
        <artifactId>spring_cloud_demo</artifactId>
        <version>0.0.1</version>
        <relativePath>../pom.xml</relativePath> <!-- lookup parent from repository -->
    </parent>


    <artifactId>user_service</artifactId>
    <version>0.0.1</version>
    <name>user_service</name>
    <description>user_service</description>

    <properties>
        <java.version>11</java.version>
    </properties>

    <dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!--mysql依赖 spring-boot-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

        <!--spring-boot druid连接池依赖-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
        </dependency>

        <!--spring-boot mybatis依赖-->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```





创建一个新模块，模块名字为order_service

pom.xml文件如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>mao</groupId>
        <artifactId>spring_cloud_demo</artifactId>
        <version>0.0.1</version>
        <relativePath>../pom.xml</relativePath> <!-- lookup parent from repository -->
    </parent>

    <artifactId>order_service</artifactId>
    <version>0.0.1</version>
    <name>order_service</name>
    <description>order_service</description>

    <properties>
        <java.version>11</java.version>
    </properties>

    <dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!--mysql依赖 spring-boot-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

        <!--spring-boot druid连接池依赖-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
        </dependency>

        <!--spring-boot mybatis依赖-->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```





### 配置文件

将配置文件的格式改为yml格式



order_service的配置文件：

```yaml
# order 业务 配置文件

spring:


  # 配置数据源
  datasource:

    druid:
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://localhost:3306/cloud_order
      username: root
      password: 20010713




# 开启debug模式，输出调试信息，常用于检查系统运行状况
#debug: true

# 设置日志级别，root表示根节点，即整体应用日志级别
logging:
 # 日志输出到文件的文件名
  file:
     name: orer_server.log
  # 设置日志组
  group:
  # 自定义组名，设置当前组中所包含的包
    mao_pro: mao
  level:
    root: info
    # 为对应组设置日志级别
    mao_pro: debug
    # 日志输出格式
# pattern:
  # console: "%d %clr(%p) --- [%16t] %clr(%-40.40c){cyan} : %m %n"



server:
  port: 8081


mybatis:
  type-aliases-package: mao.order_service
  configuration:
    map-underscore-to-camel-case: true

```





user_service的配置文件：

```yaml
# user 业务 配置文件


spring:


  # 配置数据源
  datasource:

    druid:
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://localhost:3306/cloud_user
      username: root
      password: 20010713




# 开启debug模式，输出调试信息，常用于检查系统运行状况
#debug: true

# 设置日志级别，root表示根节点，即整体应用日志级别
logging:
  # 日志输出到文件的文件名
  file:
    name: user_server.log
  # 设置日志组
  group:
    # 自定义组名，设置当前组中所包含的包
    mao_pro: mao
  level:
    root: info
    # 为对应组设置日志级别
    mao_pro: debug
    # 日志输出格式
  # pattern:
  # console: "%d %clr(%p) --- [%16t] %clr(%-40.40c){cyan} : %m %n"



server:
  port: 8082


mybatis:
  type-aliases-package: mao.user_service
  configuration:
    map-underscore-to-camel-case: true
```







### 在user_service中创建类





#### 实体类User

位于mao.user_service.entity

```java
package mao.user_service.entity;

/**
 * Project name(项目名称)：spring_cloud_demo
 * Package(包名): mao.user_service.entity
 * Class(类名): User
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/9
 * Time(创建时间)： 13:47
 * Version(版本): 1.0
 * Description(描述)： 实体类user
 */


public class User
{
    private Long id;
    private String username;
    private String address;

    /**
     * Instantiates a new User.
     */
    public User()
    {

    }

    /**
     * Instantiates a new User.
     *
     * @param id       the id
     * @param username the username
     * @param address  the address
     */
    public User(Long id, String username, String address)
    {
        this.id = id;
        this.username = username;
        this.address = address;
    }

    /**
     * Gets id.
     *
     * @return the id
     */
    public Long getId()
    {
        return id;
    }

    /**
     * Sets id.
     *
     * @param id the id
     */
    public void setId(Long id)
    {
        this.id = id;
    }

    /**
     * Gets username.
     *
     * @return the username
     */
    public String getUsername()
    {
        return username;
    }

    /**
     * Sets username.
     *
     * @param username the username
     */
    public void setUsername(String username)
    {
        this.username = username;
    }

    /**
     * Gets address.
     *
     * @return the address
     */
    public String getAddress()
    {
        return address;
    }

    /**
     * Sets address.
     *
     * @param address the address
     */
    public void setAddress(String address)
    {
        this.address = address;
    }

    @Override
    @SuppressWarnings("all")
    public String toString()
    {
        final StringBuilder stringbuilder = new StringBuilder();
        stringbuilder.append("id：").append(id).append('\n');
        stringbuilder.append("username：").append(username).append('\n');
        stringbuilder.append("address：").append(address).append('\n');
        return stringbuilder.toString();
    }
}
```





#### UserMapper

```sh
package mao.user_service.mapper;

import mao.user_service.entity.User;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;

/**
 * Project name(项目名称)：spring_cloud_demo
 * Package(包名): mao.user_service.mapper
 * Interface(接口名): UserMapper
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/9
 * Time(创建时间)： 13:45
 * Version(版本): 1.0
 * Description(描述)： mybatis mapper接口
 */

@Mapper
public interface UserMapper
{
    @Select("select * from tb_user where id = #{id}")
    User findById(@Param("id") Long id);
}
```





#### UserService

```java
package mao.user_service.service;

import mao.user_service.entity.User;
import mao.user_service.mapper.UserMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * Project name(项目名称)：spring_cloud_demo
 * Package(包名): mao.user_service.service
 * Class(类名): UserService
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/9
 * Time(创建时间)： 13:49
 * Version(版本): 1.0
 * Description(描述)： service层，不想写接口再实现接口
 */

@Service
public class UserService
{
    @Autowired
    private UserMapper userMapper;

    /**
     * 从数据库里查询用户信息
     *
     * @param id 用户的id
     * @return User
     */
    public User queryById(Long id)
    {
        return userMapper.findById(id);
    }
}
```





#### UserController

```java
package mao.user_service.controller;

import lombok.extern.slf4j.Slf4j;

import mao.user_service.entity.User;
import mao.user_service.service.UserService;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.Resource;

/**
 * Project name(项目名称)：spring_cloud_demo
 * Package(包名): mao.user_service.controller
 * Class(类名): UserController
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/9
 * Time(创建时间)： 13:51
 * Version(版本): 1.0
 * Description(描述)： UserController
 */

@Slf4j
@RestController
@RequestMapping("/user")
public class UserController
{
    @Resource
    private UserService userService;

    /**
     * 获取用户信息
     *
     * @param id 用户的id
     * @return User
     */
    @GetMapping("/{id}")
    public User queryById(@PathVariable("id") Long id)
    {
        return userService.queryById(id);
    }
}
```





#### 启动类

```java
package mao.user_service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class UserServiceApplication
{

    public static void main(String[] args)
    {
        SpringApplication.run(UserServiceApplication.class, args);
    }

}
```





### 在order_service中创建类

#### 实体类User

```java
package mao.order_service.entity;

/**
 * Project name(项目名称)：spring_cloud_demo
 * Package(包名): mao.user_service.entity
 * Class(类名): User
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/9
 * Time(创建时间)： 13:47
 * Version(版本): 1.0
 * Description(描述)： 实体类user
 */


public class User
{
    private Long id;
    private String username;
    private String address;

    /**
     * Instantiates a new User.
     */
    public User()
    {

    }

    /**
     * Instantiates a new User.
     *
     * @param id       the id
     * @param username the username
     * @param address  the address
     */
    public User(Long id, String username, String address)
    {
        this.id = id;
        this.username = username;
        this.address = address;
    }

    /**
     * Gets id.
     *
     * @return the id
     */
    public Long getId()
    {
        return id;
    }

    /**
     * Sets id.
     *
     * @param id the id
     */
    public void setId(Long id)
    {
        this.id = id;
    }

    /**
     * Gets username.
     *
     * @return the username
     */
    public String getUsername()
    {
        return username;
    }

    /**
     * Sets username.
     *
     * @param username the username
     */
    public void setUsername(String username)
    {
        this.username = username;
    }

    /**
     * Gets address.
     *
     * @return the address
     */
    public String getAddress()
    {
        return address;
    }

    /**
     * Sets address.
     *
     * @param address the address
     */
    public void setAddress(String address)
    {
        this.address = address;
    }

    @Override
    @SuppressWarnings("all")
    public String toString()
    {
        final StringBuilder stringbuilder = new StringBuilder();
        stringbuilder.append("id：").append(id).append('\n');
        stringbuilder.append("username：").append(username).append('\n');
        stringbuilder.append("address：").append(address).append('\n');
        return stringbuilder.toString();
    }
}
```







#### 实体类order

```java
package mao.order_service.entity;

/**
 * Project name(项目名称)：spring_cloud_demo
 * Package(包名): mao.order_service.entity
 * Class(类名): Order
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/9
 * Time(创建时间)： 13:55
 * Version(版本): 1.0
 * Description(描述)： 实体类order
 */


public class Order
{
    private Long id;
    private Long price;
    private String name;
    private Integer num;
    private Long userId;
    private User user;

    /**
     * Instantiates a new Order.
     */
    public Order()
    {

    }

    /**
     * Instantiates a new Order.
     *
     * @param id     the id
     * @param price  the price
     * @param name   the name
     * @param num    the num
     * @param userId the user id
     * @param user   the user
     */
    public Order(Long id, Long price, String name, Integer num, Long userId, User user)
    {
        this.id = id;
        this.price = price;
        this.name = name;
        this.num = num;
        this.userId = userId;
        this.user = user;
    }

    /**
     * Gets id.
     *
     * @return the id
     */
    public Long getId()
    {
        return id;
    }

    /**
     * Sets id.
     *
     * @param id the id
     */
    public void setId(Long id)
    {
        this.id = id;
    }

    /**
     * Gets price.
     *
     * @return the price
     */
    public Long getPrice()
    {
        return price;
    }

    /**
     * Sets price.
     *
     * @param price the price
     */
    public void setPrice(Long price)
    {
        this.price = price;
    }

    /**
     * Gets name.
     *
     * @return the name
     */
    public String getName()
    {
        return name;
    }

    /**
     * Sets name.
     *
     * @param name the name
     */
    public void setName(String name)
    {
        this.name = name;
    }

    /**
     * Gets num.
     *
     * @return the num
     */
    public Integer getNum()
    {
        return num;
    }

    /**
     * Sets num.
     *
     * @param num the num
     */
    public void setNum(Integer num)
    {
        this.num = num;
    }

    /**
     * Gets user id.
     *
     * @return the user id
     */
    public Long getUserId()
    {
        return userId;
    }

    /**
     * Sets user id.
     *
     * @param userId the user id
     */
    public void setUserId(Long userId)
    {
        this.userId = userId;
    }

    /**
     * Gets user.
     *
     * @return the user
     */
    public User getUser()
    {
        return user;
    }

    /**
     * Sets user.
     *
     * @param user the user
     */
    public void setUser(User user)
    {
        this.user = user;
    }

    @Override
    @SuppressWarnings("all")
    public String toString()
    {
        final StringBuilder stringbuilder = new StringBuilder();
        stringbuilder.append("id：").append(id).append('\n');
        stringbuilder.append("price：").append(price).append('\n');
        stringbuilder.append("name：").append(name).append('\n');
        stringbuilder.append("num：").append(num).append('\n');
        stringbuilder.append("userId：").append(userId).append('\n');
        stringbuilder.append("user：").append(user).append('\n');
        return stringbuilder.toString();
    }
}
```





#### OrderMapper

```java
package mao.order_service.mapper;

import mao.order_service.entity.Order;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Select;

/**
 * Project name(项目名称)：spring_cloud_demo
 * Package(包名): mao.order_service.mapper
 * Interface(接口名): OrderMapper
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/9
 * Time(创建时间)： 13:54
 * Version(版本): 1.0
 * Description(描述)： Mapper接口
 */

@Mapper
public interface OrderMapper
{
    @Select("select * from tb_order where id = #{id}")
    Order findById(Long id);
}
```





#### OrderService

```java
package mao.order_service.service;

import mao.order_service.entity.Order;
import mao.order_service.mapper.OrderMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * Project name(项目名称)：spring_cloud_demo
 * Package(包名): mao.order_service.service
 * Class(类名): OrderService
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/9
 * Time(创建时间)： 13:57
 * Version(版本): 1.0
 * Description(描述)： OrderService
 */

@Service
public class OrderService
{
    @Autowired
    private OrderMapper orderMapper;

    /**
     * 获取订单数据
     *
     * @param orderId 订单的id
     * @return Order
     */
    public Order queryOrderById(Long orderId)
    {
        // 根据orderId获取订单数据
        Order order = orderMapper.findById(orderId);
        //返回数据
        return order;
    }
}

```





#### OrderController

```java
package mao.order_service.controller;

import mao.order_service.entity.Order;
import mao.order_service.service.OrderService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * Project name(项目名称)：spring_cloud_demo
 * Package(包名): mao.order_service.controller
 * Class(类名): OrderController
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/9
 * Time(创建时间)： 13:59
 * Version(版本): 1.0
 * Description(描述)： OrderController
 */

@RestController
@RequestMapping("order")
public class OrderController
{
    @Autowired
    private OrderService orderService;


    /**
     * 获取订单数据
     * @param orderId 订单的id
     * @return Order
     */
    @GetMapping("{orderId}")
    public Order queryOrderByUserId(@PathVariable("orderId") Long orderId)
    {
        return orderService.queryOrderById(orderId);
    }
}
```





#### 启动类

```java
package mao.order_service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OrderServiceApplication
{

    public static void main(String[] args)
    {
        SpringApplication.run(OrderServiceApplication.class, args);
    }

}
```







### 测试类

user_service：

```java
package mao.user_service.service;

import mao.user_service.entity.User;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

import javax.annotation.Resource;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Project name(项目名称)：spring_cloud_demo
 * Package(包名): mao.user_service.service
 * Class(测试类名): UserServiceTest
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/9
 * Time(创建时间)： 14:17
 * Version(版本): 1.0
 * Description(描述)： 测试类
 */

@SpringBootTest
class UserServiceTest
{

    @Resource
    private UserService userService;

    @Test
    void queryById()
    {
        User user = userService.queryById(1L);
        System.out.println(user);
    }
}
```



order_service：

```java
package mao.order_service.service;

import mao.order_service.entity.Order;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

import javax.annotation.Resource;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Project name(项目名称)：spring_cloud_demo
 * Package(包名): mao.order_service.service
 * Class(测试类名): OrderServiceTest
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/9
 * Time(创建时间)： 14:20
 * Version(版本): 1.0
 * Description(描述)： 测试类
 */

@SpringBootTest
class OrderServiceTest
{

    @Resource
    private OrderService orderService;

    @Test
    void queryOrderById()
    {
        Order order = orderService.queryOrderById(101L);
        System.out.println(order);
    }
}
```





### 测试

user_service：

```sh
2022-07-09 14:22:05.829 DEBUG 13836 --- [           main] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-09 14:22:05.869 DEBUG 13836 --- [           main] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-09 14:22:05.907 DEBUG 13836 --- [           main] m.u.mapper.UserMapper.findById           : <==      Total: 1
id：1
username：柳岩
address：湖南省衡阳市

```



order_service：

```sh
2022-07-09 14:23:06.999 DEBUG 15136 --- [           main] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-09 14:23:07.041 DEBUG 15136 --- [           main] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-09 14:23:07.098 DEBUG 15136 --- [           main] m.o.mapper.OrderMapper.findById          : <==      Total: 1
id：101
price：699900
name：Apple 苹果 iPhone 12 
num：1
userId：1
user：null

```



功能正常



### 启动

user_service：

```sh
C:\Users\mao\.jdks\openjdk-16.0.2\bin\java.exe -XX:TieredStopAtLevel=1 -noverify -Dspring.output.ansi.enabled=always "-javaagent:C:\Program Files\JetBrains\IntelliJ IDEA 2021.2.2\lib\idea_rt.jar=49632:C:\Program Files\JetBrains\IntelliJ IDEA 2021.2.2\bin" -Dcom.sun.management.jmxremote -Dspring.jmx.enabled=true -Dspring.liveBeansView.mbeanDomain -Dspring.application.admin.enabled=true -Dfile.encoding=UTF-8 -classpath H:\程序\大三暑假\spring_cloud_demo\user_service\target\classes;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-web\2.3.9.RELEASE\spring-boot-starter-web-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter\2.3.9.RELEASE\spring-boot-starter-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot\2.3.9.RELEASE\spring-boot-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-logging\2.3.9.RELEASE\spring-boot-starter-logging-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\ch\qos\logback\logback-classic\1.2.3\logback-classic-1.2.3.jar;C:\Users\mao\.m2\repository\ch\qos\logback\logback-core\1.2.3\logback-core-1.2.3.jar;C:\Users\mao\.m2\repository\org\apache\logging\log4j\log4j-to-slf4j\2.13.3\log4j-to-slf4j-2.13.3.jar;C:\Users\mao\.m2\repository\org\apache\logging\log4j\log4j-api\2.13.3\log4j-api-2.13.3.jar;C:\Users\mao\.m2\repository\org\slf4j\jul-to-slf4j\1.7.30\jul-to-slf4j-1.7.30.jar;C:\Users\mao\.m2\repository\jakarta\annotation\jakarta.annotation-api\1.3.5\jakarta.annotation-api-1.3.5.jar;C:\Users\mao\.m2\repository\org\yaml\snakeyaml\1.26\snakeyaml-1.26.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-json\2.3.9.RELEASE\spring-boot-starter-json-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\core\jackson-databind\2.11.4\jackson-databind-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\core\jackson-annotations\2.11.4\jackson-annotations-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\core\jackson-core\2.11.4\jackson-core-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\datatype\jackson-datatype-jdk8\2.11.4\jackson-datatype-jdk8-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\datatype\jackson-datatype-jsr310\2.11.4\jackson-datatype-jsr310-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\module\jackson-module-parameter-names\2.11.4\jackson-module-parameter-names-2.11.4.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-tomcat\2.3.9.RELEASE\spring-boot-starter-tomcat-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\apache\tomcat\embed\tomcat-embed-core\9.0.43\tomcat-embed-core-9.0.43.jar;C:\Users\mao\.m2\repository\org\glassfish\jakarta.el\3.0.3\jakarta.el-3.0.3.jar;C:\Users\mao\.m2\repository\org\apache\tomcat\embed\tomcat-embed-websocket\9.0.43\tomcat-embed-websocket-9.0.43.jar;C:\Users\mao\.m2\repository\org\springframework\spring-web\5.2.13.RELEASE\spring-web-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-beans\5.2.13.RELEASE\spring-beans-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-webmvc\5.2.13.RELEASE\spring-webmvc-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-aop\5.2.13.RELEASE\spring-aop-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-context\5.2.13.RELEASE\spring-context-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-expression\5.2.13.RELEASE\spring-expression-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\mysql\mysql-connector-java\8.0.27\mysql-connector-java-8.0.27.jar;C:\Users\mao\.m2\repository\com\google\protobuf\protobuf-java\3.13.0\protobuf-java-3.13.0.jar;C:\Users\mao\.m2\repository\com\alibaba\druid-spring-boot-starter\1.2.8\druid-spring-boot-starter-1.2.8.jar;C:\Users\mao\.m2\repository\com\alibaba\druid\1.2.8\druid-1.2.8.jar;C:\Users\mao\.m2\repository\javax\annotation\javax.annotation-api\1.3.2\javax.annotation-api-1.3.2.jar;C:\Users\mao\.m2\repository\org\slf4j\slf4j-api\1.7.30\slf4j-api-1.7.30.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-autoconfigure\2.3.9.RELEASE\spring-boot-autoconfigure-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\mybatis\spring\boot\mybatis-spring-boot-starter\2.2.2\mybatis-spring-boot-starter-2.2.2.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-jdbc\2.3.9.RELEASE\spring-boot-starter-jdbc-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\com\zaxxer\HikariCP\3.4.5\HikariCP-3.4.5.jar;C:\Users\mao\.m2\repository\org\springframework\spring-jdbc\5.2.13.RELEASE\spring-jdbc-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-tx\5.2.13.RELEASE\spring-tx-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\mybatis\spring\boot\mybatis-spring-boot-autoconfigure\2.2.2\mybatis-spring-boot-autoconfigure-2.2.2.jar;C:\Users\mao\.m2\repository\org\mybatis\mybatis\3.5.9\mybatis-3.5.9.jar;C:\Users\mao\.m2\repository\org\mybatis\mybatis-spring\2.0.7\mybatis-spring-2.0.7.jar;C:\Users\mao\.m2\repository\org\springframework\spring-core\5.2.13.RELEASE\spring-core-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-jcl\5.2.13.RELEASE\spring-jcl-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\projectlombok\lombok\1.18.18\lombok-1.18.18.jar mao.user_service.UserServiceApplication
OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.9.RELEASE)

2022-07-09 14:16:06.113  INFO 2280 --- [           main] mao.user_service.UserServiceApplication  : Starting UserServiceApplication on mao with PID 2280 (H:\程序\大三暑假\spring_cloud_demo\user_service\target\classes started by mao in H:\程序\大三暑假\spring_cloud_demo)
2022-07-09 14:16:06.115 DEBUG 2280 --- [           main] mao.user_service.UserServiceApplication  : Running with Spring Boot v2.3.9.RELEASE, Spring v5.2.13.RELEASE
2022-07-09 14:16:06.116  INFO 2280 --- [           main] mao.user_service.UserServiceApplication  : No active profile set, falling back to default profiles: default
2022-07-09 14:16:06.825  INFO 2280 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8082 (http)
2022-07-09 14:16:06.835  INFO 2280 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-09 14:16:06.835  INFO 2280 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-09 14:16:06.899  INFO 2280 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-09 14:16:06.899  INFO 2280 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 741 ms
2022-07-09 14:16:06.986  INFO 2280 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2022-07-09 14:16:07.083  INFO 2280 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-07-09 14:16:07.207  INFO 2280 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-09 14:16:07.380  INFO 2280 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8082 (http) with context path ''
2022-07-09 14:16:07.389  INFO 2280 --- [           main] mao.user_service.UserServiceApplication  : Started UserServiceApplication in 1.554 seconds (JVM running for 2.034)
```





order_service：

```sh
C:\Users\mao\.jdks\openjdk-16.0.2\bin\java.exe -XX:TieredStopAtLevel=1 -noverify -Dspring.output.ansi.enabled=always "-javaagent:C:\Program Files\JetBrains\IntelliJ IDEA 2021.2.2\lib\idea_rt.jar=49703:C:\Program Files\JetBrains\IntelliJ IDEA 2021.2.2\bin" -Dcom.sun.management.jmxremote -Dspring.jmx.enabled=true -Dspring.liveBeansView.mbeanDomain -Dspring.application.admin.enabled=true -Dfile.encoding=UTF-8 -classpath H:\程序\大三暑假\spring_cloud_demo\order_service\target\classes;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-web\2.3.9.RELEASE\spring-boot-starter-web-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter\2.3.9.RELEASE\spring-boot-starter-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot\2.3.9.RELEASE\spring-boot-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-logging\2.3.9.RELEASE\spring-boot-starter-logging-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\ch\qos\logback\logback-classic\1.2.3\logback-classic-1.2.3.jar;C:\Users\mao\.m2\repository\ch\qos\logback\logback-core\1.2.3\logback-core-1.2.3.jar;C:\Users\mao\.m2\repository\org\apache\logging\log4j\log4j-to-slf4j\2.13.3\log4j-to-slf4j-2.13.3.jar;C:\Users\mao\.m2\repository\org\apache\logging\log4j\log4j-api\2.13.3\log4j-api-2.13.3.jar;C:\Users\mao\.m2\repository\org\slf4j\jul-to-slf4j\1.7.30\jul-to-slf4j-1.7.30.jar;C:\Users\mao\.m2\repository\jakarta\annotation\jakarta.annotation-api\1.3.5\jakarta.annotation-api-1.3.5.jar;C:\Users\mao\.m2\repository\org\yaml\snakeyaml\1.26\snakeyaml-1.26.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-json\2.3.9.RELEASE\spring-boot-starter-json-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\core\jackson-databind\2.11.4\jackson-databind-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\core\jackson-annotations\2.11.4\jackson-annotations-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\core\jackson-core\2.11.4\jackson-core-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\datatype\jackson-datatype-jdk8\2.11.4\jackson-datatype-jdk8-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\datatype\jackson-datatype-jsr310\2.11.4\jackson-datatype-jsr310-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\module\jackson-module-parameter-names\2.11.4\jackson-module-parameter-names-2.11.4.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-tomcat\2.3.9.RELEASE\spring-boot-starter-tomcat-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\apache\tomcat\embed\tomcat-embed-core\9.0.43\tomcat-embed-core-9.0.43.jar;C:\Users\mao\.m2\repository\org\glassfish\jakarta.el\3.0.3\jakarta.el-3.0.3.jar;C:\Users\mao\.m2\repository\org\apache\tomcat\embed\tomcat-embed-websocket\9.0.43\tomcat-embed-websocket-9.0.43.jar;C:\Users\mao\.m2\repository\org\springframework\spring-web\5.2.13.RELEASE\spring-web-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-beans\5.2.13.RELEASE\spring-beans-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-webmvc\5.2.13.RELEASE\spring-webmvc-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-aop\5.2.13.RELEASE\spring-aop-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-context\5.2.13.RELEASE\spring-context-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-expression\5.2.13.RELEASE\spring-expression-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\mysql\mysql-connector-java\8.0.27\mysql-connector-java-8.0.27.jar;C:\Users\mao\.m2\repository\com\google\protobuf\protobuf-java\3.13.0\protobuf-java-3.13.0.jar;C:\Users\mao\.m2\repository\com\alibaba\druid-spring-boot-starter\1.2.8\druid-spring-boot-starter-1.2.8.jar;C:\Users\mao\.m2\repository\com\alibaba\druid\1.2.8\druid-1.2.8.jar;C:\Users\mao\.m2\repository\javax\annotation\javax.annotation-api\1.3.2\javax.annotation-api-1.3.2.jar;C:\Users\mao\.m2\repository\org\slf4j\slf4j-api\1.7.30\slf4j-api-1.7.30.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-autoconfigure\2.3.9.RELEASE\spring-boot-autoconfigure-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\mybatis\spring\boot\mybatis-spring-boot-starter\2.2.2\mybatis-spring-boot-starter-2.2.2.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-jdbc\2.3.9.RELEASE\spring-boot-starter-jdbc-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\com\zaxxer\HikariCP\3.4.5\HikariCP-3.4.5.jar;C:\Users\mao\.m2\repository\org\springframework\spring-jdbc\5.2.13.RELEASE\spring-jdbc-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-tx\5.2.13.RELEASE\spring-tx-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\mybatis\spring\boot\mybatis-spring-boot-autoconfigure\2.2.2\mybatis-spring-boot-autoconfigure-2.2.2.jar;C:\Users\mao\.m2\repository\org\mybatis\mybatis\3.5.9\mybatis-3.5.9.jar;C:\Users\mao\.m2\repository\org\mybatis\mybatis-spring\2.0.7\mybatis-spring-2.0.7.jar;C:\Users\mao\.m2\repository\org\springframework\spring-core\5.2.13.RELEASE\spring-core-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-jcl\5.2.13.RELEASE\spring-jcl-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\projectlombok\lombok\1.18.18\lombok-1.18.18.jar mao.order_service.OrderServiceApplication
OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.9.RELEASE)

2022-07-09 14:16:55.692  INFO 4796 --- [           main] m.order_service.OrderServiceApplication  : Starting OrderServiceApplication on mao with PID 4796 (H:\程序\大三暑假\spring_cloud_demo\order_service\target\classes started by mao in H:\程序\大三暑假\spring_cloud_demo)
2022-07-09 14:16:55.695 DEBUG 4796 --- [           main] m.order_service.OrderServiceApplication  : Running with Spring Boot v2.3.9.RELEASE, Spring v5.2.13.RELEASE
2022-07-09 14:16:55.695  INFO 4796 --- [           main] m.order_service.OrderServiceApplication  : No active profile set, falling back to default profiles: default
2022-07-09 14:16:56.381  INFO 4796 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8081 (http)
2022-07-09 14:16:56.393  INFO 4796 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-09 14:16:56.395  INFO 4796 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-09 14:16:56.456  INFO 4796 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-09 14:16:56.456  INFO 4796 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 716 ms
2022-07-09 14:16:56.542  INFO 4796 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2022-07-09 14:16:56.641  INFO 4796 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-07-09 14:16:56.764  INFO 4796 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-09 14:16:56.935  INFO 4796 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8081 (http) with context path ''
2022-07-09 14:16:56.945  INFO 4796 --- [           main] m.order_service.OrderServiceApplication  : Started OrderServiceApplication in 1.529 seconds (JVM running for 2.007)
```



成功启动





### 访问测试

user_service：

http://localhost:8082/user/1



响应结果：

```json
{"id":1,"username":"柳岩","address":"湖南省衡阳市"}
```



order_service：

http://localhost:8081/order/101



响应结果：

```json
{"id":101,"price":699900,"name":"Apple 苹果 iPhone 12 ","num":1,"userId":1,"user":null}
```





访问正常







### 项目地址

https://github.com/maomao124/spring_cloud_demo.git







## 远程调用

需求：根据订单id查询订单的同时，把订单所属的用户信息一起返回。就是把order模块返回的json数据的user字段填充数据，用户查询订单的同时订单模块查询用户信息，用户模块返回数据后再拼接再返回。



可以使用RestTemplate来实现





### 实现步骤

在order_service子项目中创建一个配置类RestTemplateConfig：

```java
package mao.order_service.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

/**
 * Project name(项目名称)：spring_cloud_demo_implement_remote_invocation_of_microservices
 * Package(包名): mao.order_service.config
 * Class(类名): RestTemplateConfig
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/9
 * Time(创建时间)： 20:08
 * Version(版本): 1.0
 * Description(描述)： RestTemplate的配置类，位于子项目order_service
 */

@Configuration
public class RestTemplateConfig
{

    /**
     * 注入RestTemplate到spring容器
     *
     * @return restTemplate
     */
    @Bean
    public RestTemplate restTemplate()
    {
        return new RestTemplate();
    }
}
```





修改order_service服务中的mao.order_service.service包中的OrderService类中的queryOrderById方法：



```java
package mao.order_service.service;

import mao.order_service.entity.Order;
import mao.order_service.entity.User;
import mao.order_service.mapper.OrderMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

/**
 * Project name(项目名称)：spring_cloud_demo
 * Package(包名): mao.order_service.service
 * Class(类名): OrderService
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/9
 * Time(创建时间)： 13:57
 * Version(版本): 1.0
 * Description(描述)： OrderService
 */

@Service
public class OrderService
{
    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private RestTemplate restTemplate;

    /**
     * 获取订单数据
     *
     * @param orderId 订单的id
     * @return Order
     */
    public Order queryOrderById(Long orderId)
    {
        // 根据orderId获取订单数据
        Order order = orderMapper.findById(orderId);
        //获得用户的id
        Long userId = order.getUserId();
        //发起远程调用
        //url
        String url = "http://localhost:8082/user/" + userId;
        User user = restTemplate.getForObject(url, User.class);
        //放入order里
        order.setUser(user);
        //返回数据
        return order;
    }
}
```



启动：

order_service：

```sh
C:\Users\mao\.jdks\openjdk-16.0.2\bin\java.exe -XX:TieredStopAtLevel=1 -noverify -Dspring.output.ansi.enabled=always "-javaagent:C:\Program Files\JetBrains\IntelliJ IDEA 2021.2.2\lib\idea_rt.jar=60546:C:\Program Files\JetBrains\IntelliJ IDEA 2021.2.2\bin" -Dcom.sun.management.jmxremote -Dspring.jmx.enabled=true -Dspring.liveBeansView.mbeanDomain -Dspring.application.admin.enabled=true -Dfile.encoding=UTF-8 -classpath H:\程序\大三暑假\spring_cloud_demo实现微服务远程调用\order_service\target\classes;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-web\2.3.9.RELEASE\spring-boot-starter-web-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter\2.3.9.RELEASE\spring-boot-starter-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot\2.3.9.RELEASE\spring-boot-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-logging\2.3.9.RELEASE\spring-boot-starter-logging-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\ch\qos\logback\logback-classic\1.2.3\logback-classic-1.2.3.jar;C:\Users\mao\.m2\repository\ch\qos\logback\logback-core\1.2.3\logback-core-1.2.3.jar;C:\Users\mao\.m2\repository\org\apache\logging\log4j\log4j-to-slf4j\2.13.3\log4j-to-slf4j-2.13.3.jar;C:\Users\mao\.m2\repository\org\apache\logging\log4j\log4j-api\2.13.3\log4j-api-2.13.3.jar;C:\Users\mao\.m2\repository\org\slf4j\jul-to-slf4j\1.7.30\jul-to-slf4j-1.7.30.jar;C:\Users\mao\.m2\repository\jakarta\annotation\jakarta.annotation-api\1.3.5\jakarta.annotation-api-1.3.5.jar;C:\Users\mao\.m2\repository\org\yaml\snakeyaml\1.26\snakeyaml-1.26.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-json\2.3.9.RELEASE\spring-boot-starter-json-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\core\jackson-databind\2.11.4\jackson-databind-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\core\jackson-annotations\2.11.4\jackson-annotations-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\core\jackson-core\2.11.4\jackson-core-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\datatype\jackson-datatype-jdk8\2.11.4\jackson-datatype-jdk8-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\datatype\jackson-datatype-jsr310\2.11.4\jackson-datatype-jsr310-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\module\jackson-module-parameter-names\2.11.4\jackson-module-parameter-names-2.11.4.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-tomcat\2.3.9.RELEASE\spring-boot-starter-tomcat-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\apache\tomcat\embed\tomcat-embed-core\9.0.43\tomcat-embed-core-9.0.43.jar;C:\Users\mao\.m2\repository\org\glassfish\jakarta.el\3.0.3\jakarta.el-3.0.3.jar;C:\Users\mao\.m2\repository\org\apache\tomcat\embed\tomcat-embed-websocket\9.0.43\tomcat-embed-websocket-9.0.43.jar;C:\Users\mao\.m2\repository\org\springframework\spring-web\5.2.13.RELEASE\spring-web-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-beans\5.2.13.RELEASE\spring-beans-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-webmvc\5.2.13.RELEASE\spring-webmvc-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-aop\5.2.13.RELEASE\spring-aop-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-context\5.2.13.RELEASE\spring-context-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-expression\5.2.13.RELEASE\spring-expression-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\mysql\mysql-connector-java\8.0.27\mysql-connector-java-8.0.27.jar;C:\Users\mao\.m2\repository\com\google\protobuf\protobuf-java\3.13.0\protobuf-java-3.13.0.jar;C:\Users\mao\.m2\repository\com\alibaba\druid-spring-boot-starter\1.2.8\druid-spring-boot-starter-1.2.8.jar;C:\Users\mao\.m2\repository\com\alibaba\druid\1.2.8\druid-1.2.8.jar;C:\Users\mao\.m2\repository\javax\annotation\javax.annotation-api\1.3.2\javax.annotation-api-1.3.2.jar;C:\Users\mao\.m2\repository\org\slf4j\slf4j-api\1.7.30\slf4j-api-1.7.30.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-autoconfigure\2.3.9.RELEASE\spring-boot-autoconfigure-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\mybatis\spring\boot\mybatis-spring-boot-starter\2.2.2\mybatis-spring-boot-starter-2.2.2.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-jdbc\2.3.9.RELEASE\spring-boot-starter-jdbc-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\com\zaxxer\HikariCP\3.4.5\HikariCP-3.4.5.jar;C:\Users\mao\.m2\repository\org\springframework\spring-jdbc\5.2.13.RELEASE\spring-jdbc-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-tx\5.2.13.RELEASE\spring-tx-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\mybatis\spring\boot\mybatis-spring-boot-autoconfigure\2.2.2\mybatis-spring-boot-autoconfigure-2.2.2.jar;C:\Users\mao\.m2\repository\org\mybatis\mybatis\3.5.9\mybatis-3.5.9.jar;C:\Users\mao\.m2\repository\org\mybatis\mybatis-spring\2.0.7\mybatis-spring-2.0.7.jar;C:\Users\mao\.m2\repository\org\springframework\spring-core\5.2.13.RELEASE\spring-core-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-jcl\5.2.13.RELEASE\spring-jcl-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\projectlombok\lombok\1.18.20\lombok-1.18.20.jar mao.order_service.OrderServiceApplication
OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.9.RELEASE)

2022-07-09 20:20:17.754  INFO 8220 --- [           main] m.order_service.OrderServiceApplication  : Starting OrderServiceApplication on mao with PID 8220 (H:\程序\大三暑假\spring_cloud_demo实现微服务远程调用\order_service\target\classes started by mao in H:\程序\大三暑假\spring_cloud_demo实现微服务远程调用)
2022-07-09 20:20:17.756 DEBUG 8220 --- [           main] m.order_service.OrderServiceApplication  : Running with Spring Boot v2.3.9.RELEASE, Spring v5.2.13.RELEASE
2022-07-09 20:20:17.756  INFO 8220 --- [           main] m.order_service.OrderServiceApplication  : No active profile set, falling back to default profiles: default
2022-07-09 20:20:18.453  INFO 8220 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8081 (http)
2022-07-09 20:20:18.462  INFO 8220 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-09 20:20:18.462  INFO 8220 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-09 20:20:18.524  INFO 8220 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-09 20:20:18.524  INFO 8220 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 729 ms
2022-07-09 20:20:18.610  INFO 8220 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2022-07-09 20:20:18.705  INFO 8220 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-07-09 20:20:18.834  INFO 8220 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-09 20:20:19.001  INFO 8220 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8081 (http) with context path ''
2022-07-09 20:20:19.010  INFO 8220 --- [           main] m.order_service.OrderServiceApplication  : Started OrderServiceApplication in 1.533 seconds (JVM running for 2.012)
```



user_service：

```sh
C:\Users\mao\.jdks\openjdk-16.0.2\bin\java.exe -XX:TieredStopAtLevel=1 -noverify -Dspring.output.ansi.enabled=always "-javaagent:C:\Program Files\JetBrains\IntelliJ IDEA 2021.2.2\lib\idea_rt.jar=60552:C:\Program Files\JetBrains\IntelliJ IDEA 2021.2.2\bin" -Dcom.sun.management.jmxremote -Dspring.jmx.enabled=true -Dspring.liveBeansView.mbeanDomain -Dspring.application.admin.enabled=true -Dfile.encoding=UTF-8 -classpath H:\程序\大三暑假\spring_cloud_demo实现微服务远程调用\user_service\target\classes;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-web\2.3.9.RELEASE\spring-boot-starter-web-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter\2.3.9.RELEASE\spring-boot-starter-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot\2.3.9.RELEASE\spring-boot-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-logging\2.3.9.RELEASE\spring-boot-starter-logging-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\ch\qos\logback\logback-classic\1.2.3\logback-classic-1.2.3.jar;C:\Users\mao\.m2\repository\ch\qos\logback\logback-core\1.2.3\logback-core-1.2.3.jar;C:\Users\mao\.m2\repository\org\apache\logging\log4j\log4j-to-slf4j\2.13.3\log4j-to-slf4j-2.13.3.jar;C:\Users\mao\.m2\repository\org\apache\logging\log4j\log4j-api\2.13.3\log4j-api-2.13.3.jar;C:\Users\mao\.m2\repository\org\slf4j\jul-to-slf4j\1.7.30\jul-to-slf4j-1.7.30.jar;C:\Users\mao\.m2\repository\jakarta\annotation\jakarta.annotation-api\1.3.5\jakarta.annotation-api-1.3.5.jar;C:\Users\mao\.m2\repository\org\yaml\snakeyaml\1.26\snakeyaml-1.26.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-json\2.3.9.RELEASE\spring-boot-starter-json-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\core\jackson-databind\2.11.4\jackson-databind-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\core\jackson-annotations\2.11.4\jackson-annotations-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\core\jackson-core\2.11.4\jackson-core-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\datatype\jackson-datatype-jdk8\2.11.4\jackson-datatype-jdk8-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\datatype\jackson-datatype-jsr310\2.11.4\jackson-datatype-jsr310-2.11.4.jar;C:\Users\mao\.m2\repository\com\fasterxml\jackson\module\jackson-module-parameter-names\2.11.4\jackson-module-parameter-names-2.11.4.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-tomcat\2.3.9.RELEASE\spring-boot-starter-tomcat-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\apache\tomcat\embed\tomcat-embed-core\9.0.43\tomcat-embed-core-9.0.43.jar;C:\Users\mao\.m2\repository\org\glassfish\jakarta.el\3.0.3\jakarta.el-3.0.3.jar;C:\Users\mao\.m2\repository\org\apache\tomcat\embed\tomcat-embed-websocket\9.0.43\tomcat-embed-websocket-9.0.43.jar;C:\Users\mao\.m2\repository\org\springframework\spring-web\5.2.13.RELEASE\spring-web-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-beans\5.2.13.RELEASE\spring-beans-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-webmvc\5.2.13.RELEASE\spring-webmvc-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-aop\5.2.13.RELEASE\spring-aop-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-context\5.2.13.RELEASE\spring-context-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-expression\5.2.13.RELEASE\spring-expression-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\mysql\mysql-connector-java\8.0.27\mysql-connector-java-8.0.27.jar;C:\Users\mao\.m2\repository\com\google\protobuf\protobuf-java\3.13.0\protobuf-java-3.13.0.jar;C:\Users\mao\.m2\repository\com\alibaba\druid-spring-boot-starter\1.2.8\druid-spring-boot-starter-1.2.8.jar;C:\Users\mao\.m2\repository\com\alibaba\druid\1.2.8\druid-1.2.8.jar;C:\Users\mao\.m2\repository\javax\annotation\javax.annotation-api\1.3.2\javax.annotation-api-1.3.2.jar;C:\Users\mao\.m2\repository\org\slf4j\slf4j-api\1.7.30\slf4j-api-1.7.30.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-autoconfigure\2.3.9.RELEASE\spring-boot-autoconfigure-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\org\mybatis\spring\boot\mybatis-spring-boot-starter\2.2.2\mybatis-spring-boot-starter-2.2.2.jar;C:\Users\mao\.m2\repository\org\springframework\boot\spring-boot-starter-jdbc\2.3.9.RELEASE\spring-boot-starter-jdbc-2.3.9.RELEASE.jar;C:\Users\mao\.m2\repository\com\zaxxer\HikariCP\3.4.5\HikariCP-3.4.5.jar;C:\Users\mao\.m2\repository\org\springframework\spring-jdbc\5.2.13.RELEASE\spring-jdbc-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-tx\5.2.13.RELEASE\spring-tx-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\mybatis\spring\boot\mybatis-spring-boot-autoconfigure\2.2.2\mybatis-spring-boot-autoconfigure-2.2.2.jar;C:\Users\mao\.m2\repository\org\mybatis\mybatis\3.5.9\mybatis-3.5.9.jar;C:\Users\mao\.m2\repository\org\mybatis\mybatis-spring\2.0.7\mybatis-spring-2.0.7.jar;C:\Users\mao\.m2\repository\org\springframework\spring-core\5.2.13.RELEASE\spring-core-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\springframework\spring-jcl\5.2.13.RELEASE\spring-jcl-5.2.13.RELEASE.jar;C:\Users\mao\.m2\repository\org\projectlombok\lombok\1.18.20\lombok-1.18.20.jar mao.user_service.UserServiceApplication
OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.9.RELEASE)

2022-07-09 20:20:20.956  INFO 2676 --- [           main] mao.user_service.UserServiceApplication  : Starting UserServiceApplication on mao with PID 2676 (H:\程序\大三暑假\spring_cloud_demo实现微服务远程调用\user_service\target\classes started by mao in H:\程序\大三暑假\spring_cloud_demo实现微服务远程调用)
2022-07-09 20:20:20.959 DEBUG 2676 --- [           main] mao.user_service.UserServiceApplication  : Running with Spring Boot v2.3.9.RELEASE, Spring v5.2.13.RELEASE
2022-07-09 20:20:20.959  INFO 2676 --- [           main] mao.user_service.UserServiceApplication  : No active profile set, falling back to default profiles: default
2022-07-09 20:20:21.660  INFO 2676 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8082 (http)
2022-07-09 20:20:21.667  INFO 2676 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-09 20:20:21.667  INFO 2676 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-09 20:20:21.727  INFO 2676 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-09 20:20:21.727  INFO 2676 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 727 ms
2022-07-09 20:20:21.815  INFO 2676 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2022-07-09 20:20:21.910  INFO 2676 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-07-09 20:20:22.033  INFO 2676 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-09 20:20:22.198  INFO 2676 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8082 (http) with context path ''
2022-07-09 20:20:22.206  INFO 2676 --- [           main] mao.user_service.UserServiceApplication  : Started UserServiceApplication in 1.52 seconds (JVM running for 1.97)
```



正常



访问测试：

http://localhost:8081/order/101



结果：

```json
{"id":101,"price":699900,"name":"Apple 苹果 iPhone 12 ","num":1,"userId":1,"user":{"id":1,"username":"柳岩","address":"湖南省衡阳市"}}
```



http://localhost:8081/order/102



结果：

```json
{"id":102,"price":209900,"name":"雅迪 yadea 新国标电动车","num":1,"userId":2,"user":{"id":2,"username":"文二狗","address":"陕西省西安市"}}
```





控制台打印内容：

order_service：

```sh
OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.9.RELEASE)

2022-07-09 20:20:17.754  INFO 8220 --- [           main] m.order_service.OrderServiceApplication  : Starting OrderServiceApplication on mao with PID 8220 (H:\程序\大三暑假\spring_cloud_demo实现微服务远程调用\order_service\target\classes started by mao in H:\程序\大三暑假\spring_cloud_demo实现微服务远程调用)
2022-07-09 20:20:17.756 DEBUG 8220 --- [           main] m.order_service.OrderServiceApplication  : Running with Spring Boot v2.3.9.RELEASE, Spring v5.2.13.RELEASE
2022-07-09 20:20:17.756  INFO 8220 --- [           main] m.order_service.OrderServiceApplication  : No active profile set, falling back to default profiles: default
2022-07-09 20:20:18.453  INFO 8220 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8081 (http)
2022-07-09 20:20:18.462  INFO 8220 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-09 20:20:18.462  INFO 8220 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-09 20:20:18.524  INFO 8220 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-09 20:20:18.524  INFO 8220 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 729 ms
2022-07-09 20:20:18.610  INFO 8220 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2022-07-09 20:20:18.705  INFO 8220 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-07-09 20:20:18.834  INFO 8220 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-09 20:20:19.001  INFO 8220 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8081 (http) with context path ''
2022-07-09 20:20:19.010  INFO 8220 --- [           main] m.order_service.OrderServiceApplication  : Started OrderServiceApplication in 1.533 seconds (JVM running for 2.012)
2022-07-09 20:21:46.954  INFO 8220 --- [nio-8081-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2022-07-09 20:21:46.954  INFO 8220 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2022-07-09 20:21:46.958  INFO 8220 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 4 ms
2022-07-09 20:21:47.185 DEBUG 8220 --- [nio-8081-exec-1] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-09 20:21:47.205 DEBUG 8220 --- [nio-8081-exec-1] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-09 20:21:47.226 DEBUG 8220 --- [nio-8081-exec-1] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-09 20:22:04.822 DEBUG 8220 --- [nio-8081-exec-2] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-09 20:22:04.822 DEBUG 8220 --- [nio-8081-exec-2] m.o.mapper.OrderMapper.findById          : ==> Parameters: 102(Long)
2022-07-09 20:22:04.823 DEBUG 8220 --- [nio-8081-exec-2] m.o.mapper.OrderMapper.findById          : <==      Total: 1
```



user_service：

```sh
OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.9.RELEASE)

2022-07-09 20:20:20.956  INFO 2676 --- [           main] mao.user_service.UserServiceApplication  : Starting UserServiceApplication on mao with PID 2676 (H:\程序\大三暑假\spring_cloud_demo实现微服务远程调用\user_service\target\classes started by mao in H:\程序\大三暑假\spring_cloud_demo实现微服务远程调用)
2022-07-09 20:20:20.959 DEBUG 2676 --- [           main] mao.user_service.UserServiceApplication  : Running with Spring Boot v2.3.9.RELEASE, Spring v5.2.13.RELEASE
2022-07-09 20:20:20.959  INFO 2676 --- [           main] mao.user_service.UserServiceApplication  : No active profile set, falling back to default profiles: default
2022-07-09 20:20:21.660  INFO 2676 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8082 (http)
2022-07-09 20:20:21.667  INFO 2676 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-09 20:20:21.667  INFO 2676 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-09 20:20:21.727  INFO 2676 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-09 20:20:21.727  INFO 2676 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 727 ms
2022-07-09 20:20:21.815  INFO 2676 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2022-07-09 20:20:21.910  INFO 2676 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-07-09 20:20:22.033  INFO 2676 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-09 20:20:22.198  INFO 2676 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8082 (http) with context path ''
2022-07-09 20:20:22.206  INFO 2676 --- [           main] mao.user_service.UserServiceApplication  : Started UserServiceApplication in 1.52 seconds (JVM running for 1.97)
2022-07-09 20:21:47.303  INFO 2676 --- [nio-8082-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2022-07-09 20:21:47.303  INFO 2676 --- [nio-8082-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2022-07-09 20:21:47.306  INFO 2676 --- [nio-8082-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 3 ms
2022-07-09 20:21:47.542 DEBUG 2676 --- [nio-8082-exec-1] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-09 20:21:47.561 DEBUG 2676 --- [nio-8082-exec-1] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-09 20:21:47.584 DEBUG 2676 --- [nio-8082-exec-1] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-09 20:22:04.825 DEBUG 2676 --- [nio-8082-exec-2] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-09 20:22:04.826 DEBUG 2676 --- [nio-8082-exec-2] m.u.mapper.UserMapper.findById           : ==> Parameters: 2(Long)
2022-07-09 20:22:04.827 DEBUG 2676 --- [nio-8082-exec-2] m.u.mapper.UserMapper.findById           : <==      Total: 1
```





### 项目地址

https://github.com/maomao124/spring_cloud_demo_implement_remote_invocation_of_microservices.git/







## 提供者与消费者

