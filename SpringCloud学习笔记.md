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

在服务调用关系中，会有两个不同的角色：

**服务提供者**：一次业务中，被其它微服务调用的服务。（提供接口给其它微服务）

**服务消费者**：一次业务中，调用其它微服务的服务。（调用其它微服务提供的接口）



但是，服务提供者与服务消费者的角色并不是绝对的，而是相对于业务而言。

如果服务A调用了服务B，而服务B又调用了服务C，服务B的角色是什么？

- 对于A调用B的业务而言：A是服务消费者，B是服务提供者
- 对于B调用C的业务而言：B是服务消费者，C是服务提供者

因此，服务B既可以是服务提供者，也可以是服务消费者。











# Eureka注册中心

问题：

* 服务消费者该如何获取服务提供者的地址信息？
  * 服务提供者启动时向eureka注册自己的信息
  * eureka保存这些信息
  * 消费者根据服务名称向eureka拉取提供者信息
* 如果有多个服务提供者，消费者该如何选择？
  * 服务消费者利用负载均衡算法，从服务列表中挑选一个
* 消费者如何得知服务提供者的健康状态？
  * 服务提供者会每隔30秒向EurekaServer发送心跳请求，报告健康状态
  * eureka会更新记录服务列表信息，心跳不正常会被剔除
  * 消费者就可以拉取到最新的信息



在Eureka架构中，微服务角色有两类：

* EurekaServer：服务端，注册中心
* EurekaClient：客户端







## 实践内容

* 搭建EurekaServer
* 将user_service、order_service都注册到eureka
* 在order_service中完成服务拉取，然后通过负载均衡挑选一个服务，实现远程调用





## 实现步骤

### 准备

1. 使用刚才的项目或者拷贝出一个和刚才一样的项目
2. 在idea中，点击服务选项卡
3. 右键user_service
4. 按ctrl+D键
5. 点击环境
6. 在程序实参中填入--server.port=8083
7. 名称为UserServiceApplication2
8. 点击确定
9. 运行





### 搭建EurekaServer



#### 创建子项目

项目名称为eureka_server，为spring boot项目





#### 更改maven  pom文件

主要的依赖：

```xml
<!--eureka-server依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
```





eureka_server的pom.xml文件：

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


    <artifactId>eureka_server</artifactId>
    <version>0.0.1</version>
    <name>eureka_server</name>
    <description>eureka_server</description>


    <properties>
        <java.version>11</java.version>
    </properties>

    <dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!--eureka-server依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
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



需要更改父工程的pom.xml文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <!--
      -maven项目核心配置文件-
    Project name(项目名称)：spring_cloud_demo_eureka
    Author(作者）: mao
    Author QQ：1296193245
    GitHub：https://github.com/maomao124/
    Date(创建日期)： 2022/7/11
    Time(创建时间)： 13:08
    -->
    <groupId>mao</groupId>
    <artifactId>spring_cloud_demo</artifactId>
    <packaging>pom</packaging>
    <version>0.0.1</version>


    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.9.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>11</java.version>
    </properties>

    <modules>
        <module>user_service</module>
        <module>order_service</module>
        <module>eureka_server</module>
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



添加了一个模块eureka_server





#### 编写启动类



在eureka_server中，添加@EnableEurekaServer注解

```java
package mao.eureka_server;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication
{

    public static void main(String[] args)
    {
        SpringApplication.run(EurekaServerApplication.class, args);
    }

}
```





#### 更改配置文件

将配置文件更改为yml格式



配置内容：

```yaml
# eureka_server 配置文件

server:
  port: 10080

spring:
  application:
    name: eurekaserver

# eureka相关配置
eureka:
  client:
    # 自己本身也是，使用要配置
    service-url:
      defaultZone: http://127.0.0.1:10080/eureka/



# 设置日志级别，root表示根节点，即整体应用日志级别
logging:
  # 日志输出到文件的文件名
  file:
    name: eureka_server.log
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
```







### 注册user_service



#### 加入依赖

在user_service项目引入spring-cloud-starter-netflix-eureka-client的依赖：

```xml
 <!--eureka-client 依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```



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

        <!--eureka-client 依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
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





#### 更改配置

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


  application:
    name: userservice

eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:10080/eureka/



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







###  注册order_service

#### 加入依赖

在order_service项目引入spring-cloud-starter-netflix-eureka-client的依赖：

```xml
 <!--eureka-client 依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```



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

        <!--eureka-client 依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
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





#### 更改配置

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




  application:
    name: orderservice

eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:10080/eureka/


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







### 启动测试

将使用实例都启动，查看是否能成功启动



#### eureka_server

```sh
OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.9.RELEASE)

2022-07-11 14:43:27.578  INFO 10804 --- [           main] m.eureka_server.EurekaServerApplication  : No active profile set, falling back to default profiles: default
2022-07-11 14:43:28.099  WARN 10804 --- [           main] o.s.boot.actuate.endpoint.EndpointId     : Endpoint ID 'service-registry' contains invalid characters, please migrate to a valid format.
2022-07-11 14:43:28.180  INFO 10804 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=eab41717-91c5-31b2-ac05-97d9556dfb28
2022-07-11 14:43:28.392  INFO 10804 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 10080 (http)
2022-07-11 14:43:28.399  INFO 10804 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-11 14:43:28.399  INFO 10804 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-11 14:43:28.509  INFO 10804 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-11 14:43:28.509  INFO 10804 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 918 ms
2022-07-11 14:43:28.560  WARN 10804 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-11 14:43:28.561  INFO 10804 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-11 14:43:28.566  INFO 10804 --- [           main] c.netflix.config.DynamicPropertyFactory  : DynamicPropertyFactory is initialized with configuration sources: com.netflix.config.ConcurrentCompositeConfiguration@16a5c7e4
2022-07-11 14:43:28.838  INFO 10804 --- [           main] c.s.j.s.i.a.WebApplicationImpl           : Initiating Jersey application, version 'Jersey: 1.19.4 05/24/2017 03:20 PM'
2022-07-11 14:43:28.878  INFO 10804 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2022-07-11 14:43:28.879  INFO 10804 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2022-07-11 14:43:28.975  INFO 10804 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2022-07-11 14:43:28.975  INFO 10804 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2022-07-11 14:43:29.196  WARN 10804 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-11 14:43:29.197  INFO 10804 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-11 14:43:29.303  INFO 10804 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-11 14:43:29.828  INFO 10804 --- [           main] DiscoveryClientOptionalArgsConfiguration : Eureka HTTP Client uses Jersey
2022-07-11 14:43:29.843  WARN 10804 --- [           main] ockingLoadBalancerClientRibbonWarnLogger : You already have RibbonLoadBalancerClient on your classpath. It will be used by default. As Spring Cloud Ribbon is in maintenance mode. We recommend switching to BlockingLoadBalancerClient instead. In order to use it, set the value of `spring.cloud.loadbalancer.ribbon.enabled` to `false` or remove spring-cloud-starter-netflix-ribbon from your project.
2022-07-11 14:43:29.878  INFO 10804 --- [           main] o.s.c.n.eureka.InstanceInfoFactory       : Setting initial instance status as: STARTING
2022-07-11 14:43:29.908  INFO 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Initializing Eureka in region us-east-1
2022-07-11 14:43:30.033  INFO 10804 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2022-07-11 14:43:30.034  INFO 10804 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2022-07-11 14:43:30.034  INFO 10804 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2022-07-11 14:43:30.034  INFO 10804 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2022-07-11 14:43:30.228  INFO 10804 --- [           main] c.n.d.s.r.aws.ConfigClusterResolver      : Resolving eureka endpoints via configuration
2022-07-11 14:43:30.253  INFO 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Disable delta property : false
2022-07-11 14:43:30.253  INFO 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Single vip registry refresh property : null
2022-07-11 14:43:30.253  INFO 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Force full registry fetch : false
2022-07-11 14:43:30.253  INFO 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Application is null : false
2022-07-11 14:43:30.253  INFO 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Registered Applications size is zero : true
2022-07-11 14:43:30.253  INFO 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Application version is -1: true
2022-07-11 14:43:30.253  INFO 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Getting all instance registry info from the eureka server
2022-07-11 14:43:30.289  INFO 10804 --- [           main] c.n.d.s.t.d.RedirectingEurekaHttpClient  : Request execution error. endpoint=DefaultEndpoint{ serviceUrl='http://127.0.0.1:10080/eureka/}, exception=java.net.ConnectException: Connection refused: no further information stacktrace=com.sun.jersey.api.client.ClientHandlerException: java.net.ConnectException: Connection refused: no further information
...
...
...

2022-07-11 14:43:30.291  INFO 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Initial registry fetch from primary servers failed
2022-07-11 14:43:30.291  WARN 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Using default backup registry implementation which does not do anything.
2022-07-11 14:43:30.291  INFO 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Initial registry fetch from backup servers failed
2022-07-11 14:43:30.295  INFO 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Starting heartbeat executor: renew interval is: 30
2022-07-11 14:43:30.299  INFO 10804 --- [           main] c.n.discovery.InstanceInfoReplicator     : InstanceInfoReplicator onDemand update allowed rate per min is 4
2022-07-11 14:43:30.309  INFO 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Discovery Client initialized at timestamp 1657521810308 with initial instances count: 0
2022-07-11 14:43:30.328  INFO 10804 --- [           main] c.n.eureka.DefaultEurekaServerContext    : Initializing ...
2022-07-11 14:43:30.329  INFO 10804 --- [           main] c.n.eureka.cluster.PeerEurekaNodes       : Adding new peer nodes [http://127.0.0.1:10080/eureka/]
2022-07-11 14:43:30.347  INFO 10804 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2022-07-11 14:43:30.347  INFO 10804 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2022-07-11 14:43:30.347  INFO 10804 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2022-07-11 14:43:30.347  INFO 10804 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2022-07-11 14:43:30.461  INFO 10804 --- [           main] c.n.eureka.cluster.PeerEurekaNodes       : Replica node URL:  http://127.0.0.1:10080/eureka/
2022-07-11 14:43:30.487  INFO 10804 --- [           main] c.n.e.registry.AbstractInstanceRegistry  : Finished initializing remote region registries. All known remote regions: []
2022-07-11 14:43:30.494  INFO 10804 --- [           main] c.n.eureka.DefaultEurekaServerContext    : Initialized
2022-07-11 14:43:30.503  INFO 10804 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
2022-07-11 14:43:30.547  INFO 10804 --- [           main] o.s.c.n.e.s.EurekaServiceRegistry        : Registering application EUREKASERVER with eureka with status UP
2022-07-11 14:43:30.548  INFO 10804 --- [           main] com.netflix.discovery.DiscoveryClient    : Saw local status change event StatusChangeEvent [timestamp=1657521810548, current=UP, previous=STARTING]
2022-07-11 14:43:30.549  INFO 10804 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_EUREKASERVER/192.168.202.1:eurekaserver:10080: registering service...
2022-07-11 14:43:30.550  INFO 10804 --- [      Thread-10] o.s.c.n.e.server.EurekaServerBootstrap   : Setting the eureka configuration..
2022-07-11 14:43:30.550  INFO 10804 --- [      Thread-10] o.s.c.n.e.server.EurekaServerBootstrap   : Eureka data center value eureka.datacenter is not set, defaulting to default
2022-07-11 14:43:30.550  INFO 10804 --- [      Thread-10] o.s.c.n.e.server.EurekaServerBootstrap   : Eureka environment value eureka.environment is not set, defaulting to test
2022-07-11 14:43:30.556  INFO 10804 --- [      Thread-10] o.s.c.n.e.server.EurekaServerBootstrap   : isAws returned false
2022-07-11 14:43:30.556  INFO 10804 --- [      Thread-10] o.s.c.n.e.server.EurekaServerBootstrap   : Initialized server context
2022-07-11 14:43:30.573  INFO 10804 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 10080 (http) with context path ''
2022-07-11 14:43:30.574  INFO 10804 --- [           main] .s.c.n.e.s.EurekaAutoServiceRegistration : Updating port to 10080
2022-07-11 14:43:30.610  INFO 10804 --- [io-10080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2022-07-11 14:43:30.610  INFO 10804 --- [io-10080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2022-07-11 14:43:30.614  INFO 10804 --- [io-10080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 4 ms
2022-07-11 14:43:30.655  INFO 10804 --- [io-10080-exec-1] c.n.e.registry.AbstractInstanceRegistry  : Registered instance EUREKASERVER/192.168.202.1:eurekaserver:10080 with status UP (replication=false)
2022-07-11 14:43:30.666  INFO 10804 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_EUREKASERVER/192.168.202.1:eurekaserver:10080 - registration status: 204
2022-07-11 14:43:30.689  INFO 10804 --- [           main] m.eureka_server.EurekaServerApplication  : Started EurekaServerApplication in 3.873 seconds (JVM running for 4.407)
2022-07-11 14:43:31.174  INFO 10804 --- [io-10080-exec-2] c.n.e.registry.AbstractInstanceRegistry  : Registered instance EUREKASERVER/192.168.202.1:eurekaserver:10080 with status UP (replication=true)
2022-07-11 14:43:39.292  INFO 10804 --- [io-10080-exec-4] c.n.e.registry.AbstractInstanceRegistry  : Registered instance ORDERSERVICE/192.168.202.1:orderservice:8081 with status UP (replication=false)
2022-07-11 14:43:39.803  INFO 10804 --- [io-10080-exec-5] c.n.e.registry.AbstractInstanceRegistry  : Registered instance ORDERSERVICE/192.168.202.1:orderservice:8081 with status UP (replication=true)
2022-07-11 14:43:43.794  INFO 10804 --- [io-10080-exec-7] c.n.e.registry.AbstractInstanceRegistry  : Registered instance USERSERVICE/192.168.202.1:userservice:8082 with status UP (replication=false)
2022-07-11 14:43:44.305  INFO 10804 --- [io-10080-exec-8] c.n.e.registry.AbstractInstanceRegistry  : Registered instance USERSERVICE/192.168.202.1:userservice:8082 with status UP (replication=true)
2022-07-11 14:43:47.597  INFO 10804 --- [io-10080-exec-9] c.n.e.registry.AbstractInstanceRegistry  : Registered instance USERSERVICE/192.168.202.1:userservice:8083 with status UP (replication=false)
2022-07-11 14:43:48.107  INFO 10804 --- [io-10080-exec-1] c.n.e.registry.AbstractInstanceRegistry  : Registered instance USERSERVICE/192.168.202.1:userservice:8083 with status UP (replication=true)
```





#### user_service

```sh
OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.9.RELEASE)

2022-07-11 14:43:41.372  INFO 12908 --- [           main] mao.user_service.UserServiceApplication  : No active profile set, falling back to default profiles: default
2022-07-11 14:43:41.873  INFO 12908 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=0baa3470-9fb2-3010-aecb-a38d71a1785b
2022-07-11 14:43:42.105  INFO 12908 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8082 (http)
2022-07-11 14:43:42.111  INFO 12908 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-11 14:43:42.111  INFO 12908 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-11 14:43:42.224  INFO 12908 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-11 14:43:42.225  INFO 12908 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 841 ms
2022-07-11 14:43:42.307  INFO 12908 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2022-07-11 14:43:42.392  INFO 12908 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-07-11 14:43:42.445  WARN 12908 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-11 14:43:42.446  INFO 12908 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-11 14:43:42.448  WARN 12908 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-11 14:43:42.448  INFO 12908 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-11 14:43:42.551  INFO 12908 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-11 14:43:42.826  INFO 12908 --- [           main] DiscoveryClientOptionalArgsConfiguration : Eureka HTTP Client uses Jersey
2022-07-11 14:43:42.975  WARN 12908 --- [           main] ockingLoadBalancerClientRibbonWarnLogger : You already have RibbonLoadBalancerClient on your classpath. It will be used by default. As Spring Cloud Ribbon is in maintenance mode. We recommend switching to BlockingLoadBalancerClient instead. In order to use it, set the value of `spring.cloud.loadbalancer.ribbon.enabled` to `false` or remove spring-cloud-starter-netflix-ribbon from your project.
2022-07-11 14:43:43.070  INFO 12908 --- [           main] o.s.c.n.eureka.InstanceInfoFactory       : Setting initial instance status as: STARTING
2022-07-11 14:43:43.113  INFO 12908 --- [           main] com.netflix.discovery.DiscoveryClient    : Initializing Eureka in region us-east-1
2022-07-11 14:43:43.262  INFO 12908 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2022-07-11 14:43:43.262  INFO 12908 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2022-07-11 14:43:43.375  INFO 12908 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2022-07-11 14:43:43.375  INFO 12908 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2022-07-11 14:43:43.616  INFO 12908 --- [           main] c.n.d.s.r.aws.ConfigClusterResolver      : Resolving eureka endpoints via configuration
2022-07-11 14:43:43.643  INFO 12908 --- [           main] com.netflix.discovery.DiscoveryClient    : Disable delta property : false
2022-07-11 14:43:43.643  INFO 12908 --- [           main] com.netflix.discovery.DiscoveryClient    : Single vip registry refresh property : null
2022-07-11 14:43:43.643  INFO 12908 --- [           main] com.netflix.discovery.DiscoveryClient    : Force full registry fetch : false
2022-07-11 14:43:43.643  INFO 12908 --- [           main] com.netflix.discovery.DiscoveryClient    : Application is null : false
2022-07-11 14:43:43.643  INFO 12908 --- [           main] com.netflix.discovery.DiscoveryClient    : Registered Applications size is zero : true
2022-07-11 14:43:43.643  INFO 12908 --- [           main] com.netflix.discovery.DiscoveryClient    : Application version is -1: true
2022-07-11 14:43:43.643  INFO 12908 --- [           main] com.netflix.discovery.DiscoveryClient    : Getting all instance registry info from the eureka server
2022-07-11 14:43:43.741  INFO 12908 --- [           main] com.netflix.discovery.DiscoveryClient    : The response status is 200
2022-07-11 14:43:43.747  INFO 12908 --- [           main] com.netflix.discovery.DiscoveryClient    : Starting heartbeat executor: renew interval is: 30
2022-07-11 14:43:43.752  INFO 12908 --- [           main] c.n.discovery.InstanceInfoReplicator     : InstanceInfoReplicator onDemand update allowed rate per min is 4
2022-07-11 14:43:43.763  INFO 12908 --- [           main] com.netflix.discovery.DiscoveryClient    : Discovery Client initialized at timestamp 1657521823762 with initial instances count: 1
2022-07-11 14:43:43.764  INFO 12908 --- [           main] o.s.c.n.e.s.EurekaServiceRegistry        : Registering application USERSERVICE with eureka with status UP
2022-07-11 14:43:43.765  INFO 12908 --- [           main] com.netflix.discovery.DiscoveryClient    : Saw local status change event StatusChangeEvent [timestamp=1657521823765, current=UP, previous=STARTING]
2022-07-11 14:43:43.766  INFO 12908 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_USERSERVICE/192.168.202.1:userservice:8082: registering service...
2022-07-11 14:43:43.796  INFO 12908 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_USERSERVICE/192.168.202.1:userservice:8082 - registration status: 204
2022-07-11 14:43:43.799  INFO 12908 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8082 (http) with context path ''
2022-07-11 14:43:43.800  INFO 12908 --- [           main] .s.c.n.e.s.EurekaAutoServiceRegistration : Updating port to 8082
2022-07-11 14:43:43.910  INFO 12908 --- [           main] mao.user_service.UserServiceApplication  : Started UserServiceApplication in 3.253 seconds (JVM running for 3.846)
```



```sh
mao.user_service.UserServiceApplication --server.port=8083
OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.9.RELEASE)

2022-07-11 14:43:45.194  INFO 3548 --- [           main] mao.user_service.UserServiceApplication  : No active profile set, falling back to default profiles: default
2022-07-11 14:43:45.713  INFO 3548 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=0baa3470-9fb2-3010-aecb-a38d71a1785b
2022-07-11 14:43:45.944  INFO 3548 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8083 (http)
2022-07-11 14:43:45.951  INFO 3548 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-11 14:43:45.951  INFO 3548 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-11 14:43:46.067  INFO 3548 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-11 14:43:46.068  INFO 3548 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 861 ms
2022-07-11 14:43:46.157  INFO 3548 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2022-07-11 14:43:46.249  INFO 3548 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-07-11 14:43:46.301  WARN 3548 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-11 14:43:46.302  INFO 3548 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-11 14:43:46.304  WARN 3548 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-11 14:43:46.304  INFO 3548 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-11 14:43:46.406  INFO 3548 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-11 14:43:46.701  INFO 3548 --- [           main] DiscoveryClientOptionalArgsConfiguration : Eureka HTTP Client uses Jersey
2022-07-11 14:43:46.847  WARN 3548 --- [           main] ockingLoadBalancerClientRibbonWarnLogger : You already have RibbonLoadBalancerClient on your classpath. It will be used by default. As Spring Cloud Ribbon is in maintenance mode. We recommend switching to BlockingLoadBalancerClient instead. In order to use it, set the value of `spring.cloud.loadbalancer.ribbon.enabled` to `false` or remove spring-cloud-starter-netflix-ribbon from your project.
2022-07-11 14:43:46.923  INFO 3548 --- [           main] o.s.c.n.eureka.InstanceInfoFactory       : Setting initial instance status as: STARTING
2022-07-11 14:43:46.956  INFO 3548 --- [           main] com.netflix.discovery.DiscoveryClient    : Initializing Eureka in region us-east-1
2022-07-11 14:43:47.104  INFO 3548 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2022-07-11 14:43:47.104  INFO 3548 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2022-07-11 14:43:47.205  INFO 3548 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2022-07-11 14:43:47.205  INFO 3548 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2022-07-11 14:43:47.426  INFO 3548 --- [           main] c.n.d.s.r.aws.ConfigClusterResolver      : Resolving eureka endpoints via configuration
2022-07-11 14:43:47.451  INFO 3548 --- [           main] com.netflix.discovery.DiscoveryClient    : Disable delta property : false
2022-07-11 14:43:47.451  INFO 3548 --- [           main] com.netflix.discovery.DiscoveryClient    : Single vip registry refresh property : null
2022-07-11 14:43:47.451  INFO 3548 --- [           main] com.netflix.discovery.DiscoveryClient    : Force full registry fetch : false
2022-07-11 14:43:47.451  INFO 3548 --- [           main] com.netflix.discovery.DiscoveryClient    : Application is null : false
2022-07-11 14:43:47.452  INFO 3548 --- [           main] com.netflix.discovery.DiscoveryClient    : Registered Applications size is zero : true
2022-07-11 14:43:47.452  INFO 3548 --- [           main] com.netflix.discovery.DiscoveryClient    : Application version is -1: true
2022-07-11 14:43:47.452  INFO 3548 --- [           main] com.netflix.discovery.DiscoveryClient    : Getting all instance registry info from the eureka server
2022-07-11 14:43:47.544  INFO 3548 --- [           main] com.netflix.discovery.DiscoveryClient    : The response status is 200
2022-07-11 14:43:47.549  INFO 3548 --- [           main] com.netflix.discovery.DiscoveryClient    : Starting heartbeat executor: renew interval is: 30
2022-07-11 14:43:47.554  INFO 3548 --- [           main] c.n.discovery.InstanceInfoReplicator     : InstanceInfoReplicator onDemand update allowed rate per min is 4
2022-07-11 14:43:47.564  INFO 3548 --- [           main] com.netflix.discovery.DiscoveryClient    : Discovery Client initialized at timestamp 1657521827563 with initial instances count: 1
2022-07-11 14:43:47.566  INFO 3548 --- [           main] o.s.c.n.e.s.EurekaServiceRegistry        : Registering application USERSERVICE with eureka with status UP
2022-07-11 14:43:47.566  INFO 3548 --- [           main] com.netflix.discovery.DiscoveryClient    : Saw local status change event StatusChangeEvent [timestamp=1657521827566, current=UP, previous=STARTING]
2022-07-11 14:43:47.567  INFO 3548 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_USERSERVICE/192.168.202.1:userservice:8083: registering service...
2022-07-11 14:43:47.596  INFO 3548 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8083 (http) with context path ''
2022-07-11 14:43:47.597  INFO 3548 --- [           main] .s.c.n.e.s.EurekaAutoServiceRegistration : Updating port to 8083
2022-07-11 14:43:47.598  INFO 3548 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_USERSERVICE/192.168.202.1:userservice:8083 - registration status: 204
2022-07-11 14:43:47.701  INFO 3548 --- [           main] mao.user_service.UserServiceApplication  : Started UserServiceApplication in 3.266 seconds (JVM running for 3.806)
```





#### order_service

```sh
OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.9.RELEASE)

2022-07-11 14:43:36.915  INFO 608 --- [           main] m.order_service.OrderServiceApplication  : No active profile set, falling back to default profiles: default
2022-07-11 14:43:37.402  INFO 608 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=78719a60-c314-3629-9a9c-8de76e6dfe1f
2022-07-11 14:43:37.629  INFO 608 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8081 (http)
2022-07-11 14:43:37.636  INFO 608 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-11 14:43:37.636  INFO 608 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-11 14:43:37.751  INFO 608 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-11 14:43:37.751  INFO 608 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 823 ms
2022-07-11 14:43:37.833  INFO 608 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2022-07-11 14:43:37.923  INFO 608 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-07-11 14:43:37.987  WARN 608 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-11 14:43:37.988  INFO 608 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-11 14:43:37.990  WARN 608 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-11 14:43:37.990  INFO 608 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-11 14:43:38.091  INFO 608 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-11 14:43:38.366  INFO 608 --- [           main] DiscoveryClientOptionalArgsConfiguration : Eureka HTTP Client uses Jersey
2022-07-11 14:43:38.513  WARN 608 --- [           main] ockingLoadBalancerClientRibbonWarnLogger : You already have RibbonLoadBalancerClient on your classpath. It will be used by default. As Spring Cloud Ribbon is in maintenance mode. We recommend switching to BlockingLoadBalancerClient instead. In order to use it, set the value of `spring.cloud.loadbalancer.ribbon.enabled` to `false` or remove spring-cloud-starter-netflix-ribbon from your project.
2022-07-11 14:43:38.591  INFO 608 --- [           main] o.s.c.n.eureka.InstanceInfoFactory       : Setting initial instance status as: STARTING
2022-07-11 14:43:38.624  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Initializing Eureka in region us-east-1
2022-07-11 14:43:38.765  INFO 608 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2022-07-11 14:43:38.765  INFO 608 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2022-07-11 14:43:38.867  INFO 608 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2022-07-11 14:43:38.867  INFO 608 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2022-07-11 14:43:39.103  INFO 608 --- [           main] c.n.d.s.r.aws.ConfigClusterResolver      : Resolving eureka endpoints via configuration
2022-07-11 14:43:39.128  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Disable delta property : false
2022-07-11 14:43:39.128  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Single vip registry refresh property : null
2022-07-11 14:43:39.128  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Force full registry fetch : false
2022-07-11 14:43:39.128  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Application is null : false
2022-07-11 14:43:39.128  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Registered Applications size is zero : true
2022-07-11 14:43:39.128  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Application version is -1: true
2022-07-11 14:43:39.129  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Getting all instance registry info from the eureka server
2022-07-11 14:43:39.235  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : The response status is 200
2022-07-11 14:43:39.240  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Starting heartbeat executor: renew interval is: 30
2022-07-11 14:43:39.245  INFO 608 --- [           main] c.n.discovery.InstanceInfoReplicator     : InstanceInfoReplicator onDemand update allowed rate per min is 4
2022-07-11 14:43:39.255  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Discovery Client initialized at timestamp 1657521819254 with initial instances count: 1
2022-07-11 14:43:39.257  INFO 608 --- [           main] o.s.c.n.e.s.EurekaServiceRegistry        : Registering application ORDERSERVICE with eureka with status UP
2022-07-11 14:43:39.258  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Saw local status change event StatusChangeEvent [timestamp=1657521819258, current=UP, previous=STARTING]
2022-07-11 14:43:39.259  INFO 608 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_ORDERSERVICE/192.168.202.1:orderservice:8081: registering service...
2022-07-11 14:43:39.288  INFO 608 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8081 (http) with context path ''
2022-07-11 14:43:39.289  INFO 608 --- [           main] .s.c.n.e.s.EurekaAutoServiceRegistration : Updating port to 8081
2022-07-11 14:43:39.293  INFO 608 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_ORDERSERVICE/192.168.202.1:orderservice:8081 - registration status: 204
2022-07-11 14:43:39.401  INFO 608 --- [           main] m.order_service.OrderServiceApplication  : Started OrderServiceApplication in 3.192 seconds (JVM running for 3.718)
```



都能运行





### 在order_service完成服务拉取



服务拉取是基于服务名称获取服务列表，然后在对服务列表做负载均衡



修改order_ervice的代码，修改访问的url路径，用服务名代替ip、端口：

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
        String url = "http://userservice/user/" + userId;
        User user = restTemplate.getForObject(url, User.class);
        //放入order里
        order.setUser(user);
        //返回数据
        return order;
    }
}
```





在RestTemplateConfig中的RestTemplate添加负载均衡注解@LoadBalanced：

```java
package mao.order_service.config;

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
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
    @LoadBalanced
    public RestTemplate restTemplate()
    {
        return new RestTemplate();
    }
}
```





### 重启order_service

重启order_service



### 访问服务

http://localhost:8081/order/101



结果：

```json
{"id":101,"price":699900,"name":"Apple 苹果 iPhone 12 ","num":1,"userId":1,"user":{"id":1,"username":"柳岩","address":"湖南省衡阳市"}}
```





打印的日志：

```sh
2022-07-11 14:43:36.915  INFO 608 --- [           main] m.order_service.OrderServiceApplication  : No active profile set, falling back to default profiles: default
2022-07-11 14:43:37.402  INFO 608 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=78719a60-c314-3629-9a9c-8de76e6dfe1f
2022-07-11 14:43:37.629  INFO 608 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8081 (http)
2022-07-11 14:43:37.636  INFO 608 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-11 14:43:37.636  INFO 608 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-11 14:43:37.751  INFO 608 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-11 14:43:37.751  INFO 608 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 823 ms
2022-07-11 14:43:37.833  INFO 608 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2022-07-11 14:43:37.923  INFO 608 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-07-11 14:43:37.987  WARN 608 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-11 14:43:37.988  INFO 608 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-11 14:43:37.990  WARN 608 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-11 14:43:37.990  INFO 608 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-11 14:43:38.091  INFO 608 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-11 14:43:38.366  INFO 608 --- [           main] DiscoveryClientOptionalArgsConfiguration : Eureka HTTP Client uses Jersey
2022-07-11 14:43:38.513  WARN 608 --- [           main] ockingLoadBalancerClientRibbonWarnLogger : You already have RibbonLoadBalancerClient on your classpath. It will be used by default. As Spring Cloud Ribbon is in maintenance mode. We recommend switching to BlockingLoadBalancerClient instead. In order to use it, set the value of `spring.cloud.loadbalancer.ribbon.enabled` to `false` or remove spring-cloud-starter-netflix-ribbon from your project.
2022-07-11 14:43:38.591  INFO 608 --- [           main] o.s.c.n.eureka.InstanceInfoFactory       : Setting initial instance status as: STARTING
2022-07-11 14:43:38.624  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Initializing Eureka in region us-east-1
2022-07-11 14:43:38.765  INFO 608 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2022-07-11 14:43:38.765  INFO 608 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2022-07-11 14:43:38.867  INFO 608 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2022-07-11 14:43:38.867  INFO 608 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2022-07-11 14:43:39.103  INFO 608 --- [           main] c.n.d.s.r.aws.ConfigClusterResolver      : Resolving eureka endpoints via configuration
2022-07-11 14:43:39.128  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Disable delta property : false
2022-07-11 14:43:39.128  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Single vip registry refresh property : null
2022-07-11 14:43:39.128  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Force full registry fetch : false
2022-07-11 14:43:39.128  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Application is null : false
2022-07-11 14:43:39.128  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Registered Applications size is zero : true
2022-07-11 14:43:39.128  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Application version is -1: true
2022-07-11 14:43:39.129  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Getting all instance registry info from the eureka server
2022-07-11 14:43:39.235  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : The response status is 200
2022-07-11 14:43:39.240  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Starting heartbeat executor: renew interval is: 30
2022-07-11 14:43:39.245  INFO 608 --- [           main] c.n.discovery.InstanceInfoReplicator     : InstanceInfoReplicator onDemand update allowed rate per min is 4
2022-07-11 14:43:39.255  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Discovery Client initialized at timestamp 1657521819254 with initial instances count: 1
2022-07-11 14:43:39.257  INFO 608 --- [           main] o.s.c.n.e.s.EurekaServiceRegistry        : Registering application ORDERSERVICE with eureka with status UP
2022-07-11 14:43:39.258  INFO 608 --- [           main] com.netflix.discovery.DiscoveryClient    : Saw local status change event StatusChangeEvent [timestamp=1657521819258, current=UP, previous=STARTING]
2022-07-11 14:43:39.259  INFO 608 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_ORDERSERVICE/192.168.202.1:orderservice:8081: registering service...
2022-07-11 14:43:39.288  INFO 608 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8081 (http) with context path ''
2022-07-11 14:43:39.289  INFO 608 --- [           main] .s.c.n.e.s.EurekaAutoServiceRegistration : Updating port to 8081
2022-07-11 14:43:39.293  INFO 608 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_ORDERSERVICE/192.168.202.1:orderservice:8081 - registration status: 204
2022-07-11 14:43:39.401  INFO 608 --- [           main] m.order_service.OrderServiceApplication  : Started OrderServiceApplication in 3.192 seconds (JVM running for 3.718)
2022-07-11 14:46:42.133  INFO 608 --- [nio-8081-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2022-07-11 14:46:42.133  INFO 608 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2022-07-11 14:46:42.137  INFO 608 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 4 ms
2022-07-11 14:46:42.323 DEBUG 608 --- [nio-8081-exec-1] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:46:42.338 DEBUG 608 --- [nio-8081-exec-1] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:46:42.355 DEBUG 608 --- [nio-8081-exec-1] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:46:42.459  INFO 608 --- [nio-8081-exec-1] c.netflix.config.ChainedDynamicProperty  : Flipping property: userservice.ribbon.ActiveConnectionsLimit to use NEXT property: niws.loadbalancer.availabilityFilteringRule.activeConnectionsLimit = 2147483647
2022-07-11 14:46:42.482  INFO 608 --- [nio-8081-exec-1] c.n.u.concurrent.ShutdownEnabledTimer    : Shutdown hook installed for: NFLoadBalancer-PingTimer-userservice
2022-07-11 14:46:42.482  INFO 608 --- [nio-8081-exec-1] c.netflix.loadbalancer.BaseLoadBalancer  : Client: userservice instantiated a LoadBalancer: DynamicServerListLoadBalancer:{NFLoadBalancer:name=userservice,current list of Servers=[],Load balancer stats=Zone stats: {},Server stats: []}ServerList:null
2022-07-11 14:46:42.492  INFO 608 --- [nio-8081-exec-1] c.n.l.DynamicServerListLoadBalancer      : Using serverListUpdater PollingServerListUpdater
2022-07-11 14:46:42.512  INFO 608 --- [nio-8081-exec-1] c.netflix.config.ChainedDynamicProperty  : Flipping property: userservice.ribbon.ActiveConnectionsLimit to use NEXT property: niws.loadbalancer.availabilityFilteringRule.activeConnectionsLimit = 2147483647
2022-07-11 14:46:42.516  INFO 608 --- [nio-8081-exec-1] c.n.l.DynamicServerListLoadBalancer      : DynamicServerListLoadBalancer for client userservice initialized: DynamicServerListLoadBalancer:{NFLoadBalancer:name=userservice,current list of Servers=[192.168.202.1:8083, 192.168.202.1:8082],Load balancer stats=Zone stats: {defaultzone=[Zone:defaultzone;	Instance count:2;	Active connections count: 0;	Circuit breaker tripped count: 0;	Active connections per server: 0.0;]
},Server stats: [[Server:192.168.202.1:8082;	Zone:defaultZone;	Total Requests:0;	Successive connection failure:0;	Total blackout seconds:0;	Last connection made:Thu Jan 01 08:00:00 CST 1970;	First connection made: Thu Jan 01 08:00:00 CST 1970;	Active Connections:0;	total failure count in last (1000) msecs:0;	average resp time:0.0;	90 percentile resp time:0.0;	95 percentile resp time:0.0;	min resp time:0.0;	max resp time:0.0;	stddev resp time:0.0]
, [Server:192.168.202.1:8083;	Zone:defaultZone;	Total Requests:0;	Successive connection failure:0;	Total blackout seconds:0;	Last connection made:Thu Jan 01 08:00:00 CST 1970;	First connection made: Thu Jan 01 08:00:00 CST 1970;	Active Connections:0;	total failure count in last (1000) msecs:0;	average resp time:0.0;	90 percentile resp time:0.0;	95 percentile resp time:0.0;	min resp time:0.0;	max resp time:0.0;	stddev resp time:0.0]
]}ServerList:org.springframework.cloud.netflix.ribbon.eureka.DomainExtractingServerList@7e521c27
2022-07-11 14:46:43.495  INFO 608 --- [erListUpdater-0] c.netflix.config.ChainedDynamicProperty  : Flipping property: userservice.ribbon.ActiveConnectionsLimit to use NEXT property: niws.loadbalancer.availabilityFilteringRule.activeConnectionsLimit = 2147483647
2022-07-11 14:47:10.370 DEBUG 608 --- [nio-8081-exec-2] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:47:10.371 DEBUG 608 --- [nio-8081-exec-2] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:47:10.373 DEBUG 608 --- [nio-8081-exec-2] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:47:10.572 DEBUG 608 --- [nio-8081-exec-3] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:47:10.572 DEBUG 608 --- [nio-8081-exec-3] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:47:10.574 DEBUG 608 --- [nio-8081-exec-3] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:47:11.102 DEBUG 608 --- [nio-8081-exec-4] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:47:11.102 DEBUG 608 --- [nio-8081-exec-4] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:47:11.103 DEBUG 608 --- [nio-8081-exec-4] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:47:11.700 DEBUG 608 --- [nio-8081-exec-5] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:47:11.701 DEBUG 608 --- [nio-8081-exec-5] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:47:11.702 DEBUG 608 --- [nio-8081-exec-5] m.o.mapper.OrderMapper.findById          : <==      Total: 1
```



正常





### 测试负载均衡

将日志清空



访问20次



order_service打印的日志：

```sh
2022-07-11 14:50:13.056  WARN 608 --- [nio-8081-exec-9] c.a.druid.pool.DruidAbstractDataSource   : discard long time none received connection. , jdbcUrl : jdbc:mysql://localhost:3306/cloud_order, version : 1.2.8, lastPacketReceivedIdleMillis : 122648
2022-07-11 14:50:13.077 DEBUG 608 --- [nio-8081-exec-9] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:13.077 DEBUG 608 --- [nio-8081-exec-9] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:13.079 DEBUG 608 --- [nio-8081-exec-9] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:13.596 DEBUG 608 --- [io-8081-exec-10] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:13.596 DEBUG 608 --- [io-8081-exec-10] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:13.598 DEBUG 608 --- [io-8081-exec-10] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:14.050 DEBUG 608 --- [nio-8081-exec-1] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:14.051 DEBUG 608 --- [nio-8081-exec-1] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:14.053 DEBUG 608 --- [nio-8081-exec-1] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:14.499 DEBUG 608 --- [nio-8081-exec-3] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:14.499 DEBUG 608 --- [nio-8081-exec-3] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:14.500 DEBUG 608 --- [nio-8081-exec-3] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:14.959 DEBUG 608 --- [nio-8081-exec-2] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:14.959 DEBUG 608 --- [nio-8081-exec-2] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:14.961 DEBUG 608 --- [nio-8081-exec-2] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:15.419 DEBUG 608 --- [nio-8081-exec-4] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:15.419 DEBUG 608 --- [nio-8081-exec-4] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:15.420 DEBUG 608 --- [nio-8081-exec-4] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:15.819 DEBUG 608 --- [nio-8081-exec-5] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:15.819 DEBUG 608 --- [nio-8081-exec-5] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:15.820 DEBUG 608 --- [nio-8081-exec-5] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:16.269 DEBUG 608 --- [nio-8081-exec-6] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:16.269 DEBUG 608 --- [nio-8081-exec-6] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:16.271 DEBUG 608 --- [nio-8081-exec-6] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:16.666 DEBUG 608 --- [nio-8081-exec-7] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:16.667 DEBUG 608 --- [nio-8081-exec-7] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:16.668 DEBUG 608 --- [nio-8081-exec-7] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:17.078 DEBUG 608 --- [nio-8081-exec-8] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:17.078 DEBUG 608 --- [nio-8081-exec-8] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:17.081 DEBUG 608 --- [nio-8081-exec-8] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:17.587 DEBUG 608 --- [nio-8081-exec-9] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:17.588 DEBUG 608 --- [nio-8081-exec-9] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:17.589 DEBUG 608 --- [nio-8081-exec-9] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:18.016 DEBUG 608 --- [io-8081-exec-10] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:18.017 DEBUG 608 --- [io-8081-exec-10] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:18.018 DEBUG 608 --- [io-8081-exec-10] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:18.439 DEBUG 608 --- [nio-8081-exec-1] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:18.439 DEBUG 608 --- [nio-8081-exec-1] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:18.440 DEBUG 608 --- [nio-8081-exec-1] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:18.880 DEBUG 608 --- [nio-8081-exec-3] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:18.881 DEBUG 608 --- [nio-8081-exec-3] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:18.882 DEBUG 608 --- [nio-8081-exec-3] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:19.287 DEBUG 608 --- [nio-8081-exec-2] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:19.288 DEBUG 608 --- [nio-8081-exec-2] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:19.290 DEBUG 608 --- [nio-8081-exec-2] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:19.707 DEBUG 608 --- [nio-8081-exec-4] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:19.707 DEBUG 608 --- [nio-8081-exec-4] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:19.708 DEBUG 608 --- [nio-8081-exec-4] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:20.147 DEBUG 608 --- [nio-8081-exec-5] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:20.147 DEBUG 608 --- [nio-8081-exec-5] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:20.149 DEBUG 608 --- [nio-8081-exec-5] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:20.618 DEBUG 608 --- [nio-8081-exec-6] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:20.618 DEBUG 608 --- [nio-8081-exec-6] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:20.620 DEBUG 608 --- [nio-8081-exec-6] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:21.019 DEBUG 608 --- [nio-8081-exec-7] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:21.020 DEBUG 608 --- [nio-8081-exec-7] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:21.021 DEBUG 608 --- [nio-8081-exec-7] m.o.mapper.OrderMapper.findById          : <==      Total: 1
2022-07-11 14:50:21.430 DEBUG 608 --- [nio-8081-exec-8] m.o.mapper.OrderMapper.findById          : ==>  Preparing: select * from tb_order where id = ?
2022-07-11 14:50:21.430 DEBUG 608 --- [nio-8081-exec-8] m.o.mapper.OrderMapper.findById          : ==> Parameters: 101(Long)
2022-07-11 14:50:21.432 DEBUG 608 --- [nio-8081-exec-8] m.o.mapper.OrderMapper.findById          : <==      Total: 1
```



user_service1打印的1日志：

```sh
2022-07-11 14:50:14.063  WARN 12908 --- [nio-8082-exec-4] c.a.druid.pool.DruidAbstractDataSource   : discard long time none received connection. , jdbcUrl : jdbc:mysql://localhost:3306/cloud_user, version : 1.2.8, lastPacketReceivedIdleMillis : 182949
2022-07-11 14:50:14.085 DEBUG 12908 --- [nio-8082-exec-4] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:14.085 DEBUG 12908 --- [nio-8082-exec-4] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:14.087 DEBUG 12908 --- [nio-8082-exec-4] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:14.965 DEBUG 12908 --- [nio-8082-exec-5] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:14.965 DEBUG 12908 --- [nio-8082-exec-5] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:14.967 DEBUG 12908 --- [nio-8082-exec-5] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:15.823 DEBUG 12908 --- [nio-8082-exec-6] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:15.823 DEBUG 12908 --- [nio-8082-exec-6] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:15.824 DEBUG 12908 --- [nio-8082-exec-6] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:16.670 DEBUG 12908 --- [nio-8082-exec-7] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:16.670 DEBUG 12908 --- [nio-8082-exec-7] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:16.672 DEBUG 12908 --- [nio-8082-exec-7] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:17.591 DEBUG 12908 --- [nio-8082-exec-8] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:17.592 DEBUG 12908 --- [nio-8082-exec-8] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:17.593 DEBUG 12908 --- [nio-8082-exec-8] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:18.442 DEBUG 12908 --- [nio-8082-exec-9] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:18.442 DEBUG 12908 --- [nio-8082-exec-9] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:18.444 DEBUG 12908 --- [nio-8082-exec-9] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:19.292 DEBUG 12908 --- [io-8082-exec-10] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:19.293 DEBUG 12908 --- [io-8082-exec-10] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:19.294 DEBUG 12908 --- [io-8082-exec-10] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:20.152 DEBUG 12908 --- [nio-8082-exec-1] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:20.153 DEBUG 12908 --- [nio-8082-exec-1] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:20.154 DEBUG 12908 --- [nio-8082-exec-1] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:21.022 DEBUG 12908 --- [nio-8082-exec-2] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:21.023 DEBUG 12908 --- [nio-8082-exec-2] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:21.024 DEBUG 12908 --- [nio-8082-exec-2] m.u.mapper.UserMapper.findById           : <==      Total: 1
```



user_service2打印的1日志：

```sh
2022-07-11 14:50:13.088  WARN 3548 --- [nio-8083-exec-6] c.a.druid.pool.DruidAbstractDataSource   : discard long time none received connection. , jdbcUrl : jdbc:mysql://localhost:3306/cloud_user, version : 1.2.8, lastPacketReceivedIdleMillis : 122676
2022-07-11 14:50:13.109 DEBUG 3548 --- [nio-8083-exec-6] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:13.109 DEBUG 3548 --- [nio-8083-exec-6] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:13.111 DEBUG 3548 --- [nio-8083-exec-6] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:13.600 DEBUG 3548 --- [nio-8083-exec-7] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:13.601 DEBUG 3548 --- [nio-8083-exec-7] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:13.602 DEBUG 3548 --- [nio-8083-exec-7] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:14.502 DEBUG 3548 --- [nio-8083-exec-8] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:14.503 DEBUG 3548 --- [nio-8083-exec-8] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:14.504 DEBUG 3548 --- [nio-8083-exec-8] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:15.422 DEBUG 3548 --- [nio-8083-exec-9] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:15.422 DEBUG 3548 --- [nio-8083-exec-9] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:15.424 DEBUG 3548 --- [nio-8083-exec-9] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:16.274 DEBUG 3548 --- [io-8083-exec-10] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:16.274 DEBUG 3548 --- [io-8083-exec-10] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:16.275 DEBUG 3548 --- [io-8083-exec-10] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:17.084 DEBUG 3548 --- [nio-8083-exec-1] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:17.084 DEBUG 3548 --- [nio-8083-exec-1] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:17.085 DEBUG 3548 --- [nio-8083-exec-1] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:18.021 DEBUG 3548 --- [nio-8083-exec-2] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:18.021 DEBUG 3548 --- [nio-8083-exec-2] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:18.022 DEBUG 3548 --- [nio-8083-exec-2] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:18.884 DEBUG 3548 --- [nio-8083-exec-3] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:18.884 DEBUG 3548 --- [nio-8083-exec-3] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:18.885 DEBUG 3548 --- [nio-8083-exec-3] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:19.711 DEBUG 3548 --- [nio-8083-exec-4] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:19.711 DEBUG 3548 --- [nio-8083-exec-4] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:19.713 DEBUG 3548 --- [nio-8083-exec-4] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:20.623 DEBUG 3548 --- [nio-8083-exec-5] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:20.623 DEBUG 3548 --- [nio-8083-exec-5] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:20.624 DEBUG 3548 --- [nio-8083-exec-5] m.u.mapper.UserMapper.findById           : <==      Total: 1
2022-07-11 14:50:21.434 DEBUG 3548 --- [nio-8083-exec-6] m.u.mapper.UserMapper.findById           : ==>  Preparing: select * from tb_user where id = ?
2022-07-11 14:50:21.434 DEBUG 3548 --- [nio-8083-exec-6] m.u.mapper.UserMapper.findById           : ==> Parameters: 1(Long)
2022-07-11 14:50:21.435 DEBUG 3548 --- [nio-8083-exec-6] m.u.mapper.UserMapper.findById           : <==      Total: 1
```



user_service被访问了9次 ，user_service2被访问了11次







## 项目地址

https://github.com/maomao124/spring_cloud_demo_eureka.git







# Ribbon负载均衡

## 负载均衡流程

1. order_service发起请求到达Ribbon
2. Ribbon向eureka-server拉取userservice的服务列表
3. eureka-server返回服务列表到Ribbon
4. 轮询调用





## 负载均衡策略

Ribbon的负载均衡规则是一个叫做IRule的接口来定义的，每一个子接口都是一种规则



|  **内置负载均衡规则类**   |                         **规则描述**                         |
| :-----------------------: | :----------------------------------------------------------: |
|      RoundRobinRule       | 简单轮询服务列表来选择服务器。它是Ribbon默认的负载均衡规则。 |
| AvailabilityFilteringRule | 对以下两种服务器进行忽略：   （1）在默认情况下，这台服务器如果3次连接失败，这台服务器就会被设置为“短路”状态。短路状态将持续30秒，如果再次连接失败，短路的持续时间就会几何级地增加。  （2）并发数过高的服务器。如果一个服务器的并发连接数过高，配置了AvailabilityFilteringRule规则的客户端也会将其忽略。并发连接数的上限，可以由客户端的<clientName>.<clientConfigNameSpace>.ActiveConnectionsLimit属性进行配置。 |
| WeightedResponseTimeRule  | 为每一个服务器赋予一个权重值。服务器响应时间越长，这个服务器的权重就越小。这个规则会随机选择服务器，这个权重值会影响服务器的选择。 |
|     ZoneAvoidanceRule     | 以区域可用的服务器为基础进行服务器的选择。使用Zone对服务器进行分类，这个Zone可以理解为一个机房、一个机架等。而后再对Zone内的多个服务做轮询。 |
|     BestAvailableRule     |       忽略那些短路的服务器，并选择并发数较低的服务器。       |
|        RandomRule         |                  随机选择一个可用的服务器。                  |
|         RetryRule         |                      重试机制的选择逻辑                      |





## 更改规则

通过定义IRule实现可以修改负载均衡规则，有两种方式



### 代码方式

创建RibbonConfig类

```java
package mao.order_service.config;

import com.netflix.loadbalancer.IRule;
import com.netflix.loadbalancer.RandomRule;
import org.springframework.context.annotation.Bean;

/**
 * Project name(项目名称)：spring_cloud_demo_eureka
 * Package(包名): mao.order_service.config
 * Class(类名): RibbonConfig
 * Author(作者）: mao
 * Author QQ：1296193245
 * GitHub：https://github.com/maomao124/
 * Date(创建日期)： 2022/7/11
 * Time(创建时间)： 20:59
 * Version(版本): 1.0
 * Description(描述)： 无
 */

public class RibbonConfig
{
    /**
     * 配置Ribbon负载均衡规则
     *
     * @return RandomRule
     */
    @Bean
    public IRule randomRule()
    {
        return new RandomRule();
    }
}
```





### 配置文件方式

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




  application:
    name: orderservice

eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:10080/eureka/


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


# 配置负载均衡规则
userservice:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule


server:
  port: 8081


mybatis:
  type-aliases-package: mao.order_service
  configuration:
    map-underscore-to-camel-case: true
```





## 饥饿加载

Ribbon默认是采用懒加载，即第一次访问时才会去创建LoadBalanceClient，请求时间会很长。 而饥饿加载则会在项目启动时创建，降低第一次访问的耗时，通过下面配置开启饥饿加载：



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




  application:
    name: orderservice

eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:10080/eureka/


# 开启debug模式，输出调试信息，常用于检查系统运行状况
#debug: true

# 设置日志级别，root表示根节点，即整体应用日志级别
logging:
 # 日志输出到文件的文件名
  file:
     name: order_server.log
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


# 配置负载均衡规则
#userservice:
#  ribbon:
#    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule


ribbon:
  eager-load:
    # 开启饥饿加载
    enabled: true
    # 指定对 userservice 这个服务饥饿加载
    clients: userservice


server:
  port: 8081


mybatis:
  type-aliases-package: mao.order_service
  configuration:
    map-underscore-to-camel-case: true
```







# Nacos注册中心

## Nacos安装

### Windows安装

#### 下载

在Nacos的GitHub页面，提供有下载链接，可以下载编译好的Nacos服务端或者源代码：

GitHub主页：https://github.com/alibaba/nacos

GitHub的Release下载页：https://github.com/alibaba/nacos/releases





#### 解压

将这个包解压到任意非中文目录下：



```sh
PS H:\opensoft\nacos> ls


    目录: H:\opensoft\nacos


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         2022/7/12     19:03                bin
d-----         2022/7/12     19:03                conf
d-----         2022/7/12     19:03                target
-a----         2021/3/30     14:26          16583 LICENSE
-a----         2021/3/30     14:26           1305 NOTICE


PS H:\opensoft\nacos>
```

```sh
PS H:\opensoft\nacos\conf> ls


    目录: H:\opensoft\nacos\conf


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         2021/3/30     14:26           1224 1.4.0-ipv6_support-update.sql
-a----         2022/1/27     10:43           7142 application.properties
-a----         2022/1/27     10:43           6515 application.properties.example
-a----         2022/7/12     19:09            673 cluster.conf
-a----         2022/1/27     10:43          25710 nacos-logback.xml
-a----        2021/12/20     14:16          10660 nacos-mysql.sql
-a----        2021/12/20     14:16           8795 schema.sql


PS H:\opensoft\nacos\conf>
```



Nacos的默认端口是8848

修改配置文件中的端口：

更改application.properties的内容：

```sh
server.port=8848
```





#### 启动

进入bin目录

```sh
PS H:\opensoft\nacos> cd bin
PS H:\opensoft\nacos\bin> ls


    目录: H:\opensoft\nacos\bin


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         2021/3/30     14:26            954 shutdown.cmd
-a----         2021/3/30     14:26            951 shutdown.sh
-a----         2022/1/27     10:43           3340 startup.cmd
-a----         2022/1/27     10:43           4923 startup.sh


PS H:\opensoft\nacos\bin>
```



命令：

```sh
startup.cmd -m standalone
```



```sh
PS H:\opensoft\nacos\bin> ./startup.cmd -m standalone
"nacos is starting with standalone"

         ,--.
       ,--.'|
   ,--,:  : |                                           Nacos 1.4.3
,`--.'`|  ' :                       ,---.               Running in stand alone mode, All function modules
|   :  :  | |                      '   ,'\   .--.--.    Port: 8848
:   |   \ | :  ,--.--.     ,---.  /   /   | /  /    '   Pid: 15096
|   : '  '; | /       \   /     \.   ; ,. :|  :  /`./   Console: http://192.168.202.1:8848/nacos/index.html
'   ' ;.    ;.--.  .-. | /    / ''   | |: :|  :  ;_
|   | | \   | \__\/: . ..    ' / '   | .; : \  \    `.      https://nacos.io
'   : |  ; .' ," .--.; |'   ; :__|   :    |  `----.   \
|   | '`--'  /  /  ,.  |'   | '.'|\   \  /  /  /`--'  /
'   : |     ;  :   .'   \   :    : `----'  '--'.     /
;   |.'     |  ,     .-./\   \  /            `--'---'
'---'        `--`---'     `----'

2022-07-12 19:12:55,740 INFO Bean 'org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler@162be91c' of type [org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)

2022-07-12 19:12:55,751 INFO Bean 'methodSecurityMetadataSource' of type [org.springframework.security.access.method.DelegatingMethodSecurityMetadataSource] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)

2022-07-12 19:12:56,230 INFO Tomcat initialized with port(s): 8848 (http)

2022-07-12 19:12:56,666 INFO Root WebApplicationContext: initialization completed in 3288 ms

2022-07-12 19:13:06,401 INFO Initializing ExecutorService 'applicationTaskExecutor'

2022-07-12 19:13:06,567 INFO Adding welcome page: class path resource [static/index.html]

2022-07-12 19:13:06,913 INFO Creating filter chain: Ant [pattern='/**'], []

2022-07-12 19:13:06,950 INFO Creating filter chain: any request, [org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@360e9c06, org.springframework.security.web.context.SecurityContextPersistenceFilter@659feb22, org.springframework.security.web.header.HeaderWriterFilter@12ad1b2a, org.springframework.security.web.csrf.CsrfFilter@4694f434, org.springframework.security.web.authentication.logout.LogoutFilter@101a461c, org.springframework.security.web.savedrequest.RequestCacheAwareFilter@2f4b98f6, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@58d6e55a, org.springframework.security.web.authentication.AnonymousAuthenticationFilter@5ebffb44, org.springframework.security.web.session.SessionManagementFilter@6e3eb0cd, org.springframework.security.web.access.ExceptionTranslationFilter@5f631ca0]

2022-07-12 19:13:07,041 INFO Initializing ExecutorService 'taskScheduler'

2022-07-12 19:13:07,067 INFO Exposing 2 endpoint(s) beneath base path '/actuator'

2022-07-12 19:13:07,169 INFO Tomcat started on port(s): 8848 (http) with context path '/nacos'

2022-07-12 19:13:07,176 INFO Nacos started successfully in stand alone mode. use embedded storage

```





### 访问

在浏览器输入地址：http://localhost:8848/nacos

默认的账号和密码都是nacos





### Linux安装

#### 安装JDK

Nacos依赖于JDK运行



```sh
yum search jdk
```



```sh
[root@889e0484bdd2 yum.repos.d]# yum search jdk
Failed to set locale, defaulting to C.UTF-8
Last metadata expiration check: 0:06:30 ago on Mon Jul  4 06:25:24 2022.
================================================================= Name & Summary Matched: jdk ==================================================================
copy-jdk-configs.noarch : JDKs configuration files copier
java-1.8.0-openjdk.x86_64 : OpenJDK 8 Runtime Environment
java-1.8.0-openjdk-accessibility.x86_64 : OpenJDK 8 accessibility connector
java-1.8.0-openjdk-demo.x86_64 : OpenJDK 8 Demos
java-1.8.0-openjdk-devel.x86_64 : OpenJDK 8 Development Environment
java-1.8.0-openjdk-headless.x86_64 : OpenJDK 8 Headless Runtime Environment
java-1.8.0-openjdk-headless-slowdebug.x86_64 : OpenJDK 8 Runtime Environment unoptimised with full debugging on
java-1.8.0-openjdk-javadoc.noarch : OpenJDK 8 API documentation
java-1.8.0-openjdk-javadoc-zip.noarch : OpenJDK 8 API documentation compressed in a single archive
java-1.8.0-openjdk-slowdebug.x86_64 : OpenJDK 8 Runtime Environment unoptimised with full debugging on
java-1.8.0-openjdk-src.x86_64 : OpenJDK 8 Source Bundle
java-11-openjdk.x86_64 : OpenJDK 11 Runtime Environment
java-11-openjdk-demo.x86_64 : OpenJDK 11 Demos
java-11-openjdk-devel.x86_64 : OpenJDK 11 Development Environment
java-11-openjdk-headless.x86_64 : OpenJDK 11 Headless Runtime Environment
java-11-openjdk-javadoc.x86_64 : OpenJDK 11 API documentation
java-11-openjdk-javadoc-zip.x86_64 : OpenJDK 11 API documentation compressed in a single archive
java-11-openjdk-jmods.x86_64 : JMods for OpenJDK 11
java-11-openjdk-src.x86_64 : OpenJDK 11 Source Bundle
java-11-openjdk-static-libs.x86_64 : OpenJDK 11 libraries for static linking
java-17-openjdk.x86_64 : OpenJDK 17 Runtime Environment
java-17-openjdk-demo.x86_64 : OpenJDK 17 Demos
java-17-openjdk-devel.x86_64 : OpenJDK 17 Development Environment
java-17-openjdk-headless.x86_64 : OpenJDK 17 Headless Runtime Environment
java-17-openjdk-javadoc.x86_64 : OpenJDK 17 API documentation
java-17-openjdk-javadoc-zip.x86_64 : OpenJDK 17 API documentation compressed in a single archive
java-17-openjdk-jmods.x86_64 : JMods for OpenJDK 17
java-17-openjdk-src.x86_64 : OpenJDK 17 Source Bundle
java-17-openjdk-static-libs.x86_64 : OpenJDK 17 libraries for static linking
===================================================================== Summary Matched: jdk =====================================================================
icedtea-web.x86_64 : Additional Java components for OpenJDK - Java browser plug-in and Web Start implementation
jmc.x86_64 : JDK Mission Control is a profiling and diagnostics tool
jmc-core.noarch : Core API for JDK Mission Control
[root@889e0484bdd2 yum.repos.d]#
```



安装jdk17：

```sh
yum -y install java-17-openjdk.x86_64
```



验证是否安装成功：

```sh
java -version
```



```sh
[root@889e0484bdd2 /]# java -version
openjdk version "17.0.1" 2021-10-19 LTS
OpenJDK Runtime Environment 21.9 (build 17.0.1+12-LTS)
OpenJDK 64-Bit Server VM 21.9 (build 17.0.1+12-LTS, mixed mode, sharing)
[root@889e0484bdd2 /]#
```



安装成功



#### 安装nacos

将下载到的压缩包上传到linux上

解压安装包：

```sh
tar -xvf nacos-server-1.4.3.tar.gz
```

删除安装包：

```sh
rm -rf nacos-server-1.4.3.tar.gz
```



#### 启动

```sh
sh startup.sh -m standalone
```







### Docker 安装

#### 操作步骤

- Clone 项目

  ```powershell
  git clone https://github.com/nacos-group/nacos-docker.git
  cd nacos-docker
  ```

- 单机模式 Derby

  ```powershell
  docker-compose -f example/standalone-derby.yaml up
  ```

- 单机模式 MySQL

如果希望使用MySQL5.7

```powershell
docker-compose -f example/standalone-mysql-5.7.yaml up
```

如果希望使用MySQL8

```powershell
docker-compose -f example/standalone-mysql-8.yaml up
```

- 集群模式

  ```powershell
  docker-compose -f example/cluster-hostname.yaml up 
  ```

- 服务注册

  ```powershell
  curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080'
  ```

- 服务发现

  ```powershell
  curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=nacos.naming.serviceName'
  ```

- 发布配置

  ```powershell
  curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test&content=helloWorld"
  ```

- 获取配置

  ```powershell
    curl -X GET "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test"
  ```

- Nacos 控制台

  link：http://127.0.0.1:8848/nacos/



#### 公共属性配置

|                属性名称                 |                        描述                        |                             选项                             |
| :-------------------------------------: | :------------------------------------------------: | :----------------------------------------------------------: |
|                  MODE                   |              系统启动方式: 集群/单机               |              cluster/standalone默认 **cluster**              |
|              NACOS_SERVERS              |                      集群地址                      |             p1:port1空格ip2:port2 空格ip3:port3              |
|            PREFER_HOST_MODE             |                 支持IP还是域名模式                 |                   hostname/ip 默认 **ip**                    |
|            NACOS_SERVER_PORT            |                   Nacos 运行端口                   |                        默认 **8848**                         |
|             NACOS_SERVER_IP             |               多网卡模式下可以指定IP               |                                                              |
|       SPRING_DATASOURCE_PLATFORM        |             单机模式下支持MYSQL数据库              |                      mysql / 空 默认:空                      |
|           MYSQL_SERVICE_HOST            |                  数据库 连接地址                   |                                                              |
|           MYSQL_SERVICE_PORT            |                     数据库端口                     |                       默认 : **3306**                        |
|          MYSQL_SERVICE_DB_NAME          |                     数据库库名                     |                                                              |
|           MYSQL_SERVICE_USER            |                    数据库用户名                    |                                                              |
|         MYSQL_SERVICE_PASSWORD          |                   数据库用户密码                   |                                                              |
|         MYSQL_SERVICE_DB_PARAM          |                   数据库连接参数                   | default : **characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false** |
|           MYSQL_DATABASE_NUM            |                     数据库编号                     |                         默认 :**1**                          |
|                 JVM_XMS                 |                        -Xms                        |                           默认 :1g                           |
|                 JVM_XMX                 |                        -Xmx                        |                           默认 :1g                           |
|                 JVM_XMN                 |                        -Xmn                        |                          默认 :512m                          |
|                 JVM_MS                  |                 -XX:MetaspaceSize                  |                          默认 :128m                          |
|                 JVM_MMS                 |                -XX:MaxMetaspaceSize                |                          默认 :320m                          |
|               NACOS_DEBUG               |                 是否开启远程DEBUG                  |                         y/n 默认 :n                          |
|        TOMCAT_ACCESSLOG_ENABLED         |          server.tomcat.accesslog.enabled           |                         默认 :false                          |
|         NACOS_AUTH_SYSTEM_TYPE          |        权限系统类型选择,目前只支持nacos类型        |                         默认 :nacos                          |
|            NACOS_AUTH_ENABLE            |                  是否开启权限系统                  |                         默认 :false                          |
|     NACOS_AUTH_TOKEN_EXPIRE_SECONDS     |                   token 失效时间                   |                         默认 :18000                          |
|            NACOS_AUTH_TOKEN             |                       token                        | 默认 :SecretKey012345678901234567890123456789012345678901234567890123456789 |
|         NACOS_AUTH_CACHE_ENABLE         | 权限缓存开关 ,开启后权限缓存的更新默认有15秒的延迟 |                         默认 : false                         |
|               MEMBER_LIST               |           通过环境变量的方式设置集群地址           | 例子:192.168.16.101:8847?raft_port=8807,192.168.16.101?raft_port=8808,192.168.16.101:8849?raft_port=8809 |
|            EMBEDDED_STORAGE             |             是否开启集群嵌入式存储模式             |                    `embedded` 默认 : none                    |
|         NACOS_AUTH_CACHE_ENABLE         |          nacos.core.auth.caching.enabled           |                       default : false                        |
| NACOS_AUTH_USER_AGENT_AUTH_WHITE_ENABLE |     nacos.core.auth.enable.userAgentAuthWhite      |                       default : false                        |
|         NACOS_AUTH_IDENTITY_KEY         |        nacos.core.auth.server.identity.key         |                   default : serverIdentity                   |
|        NACOS_AUTH_IDENTITY_VALUE        |       nacos.core.auth.server.identity.value        |                      default : security                      |
|       NACOS_SECURITY_IGNORE_URLS        |             nacos.security.ignore.urls             | default : `/,/error,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/v1/auth/**,/v1/console/health/**,/actuator/**,/v1/console/server/**` |







### Kubernetes 安装

- **Clone 项目**

```shell
git clone https://github.com/nacos-group/nacos-k8s.git
```



> 如果你使用简单方式快速启动,请注意这是没有使用持久化卷的,可能存在数据丢失风险:

```shell
cd nacos-k8s
chmod +x quick-startup.sh
./quick-startup.sh
```

- **测试**

  - **服务注册**

  ```bash
  curl -X POST 'http://cluster-ip:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080'
  ```

  - **服务发现**

  ```bash
  curl -X GET 'http://cluster-ip:8848/nacos/v1/ns/instance/list?serviceName=nacos.naming.serviceName'
  ```

  - **发布配置**

  ```bash
  curl -X POST "http://cluster-ip:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test&content=helloWorld"
  ```

  - **获取配置**

  ```bash
  curl -X GET "http://cluster-ip:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test"
  ```





#### 部署 NFS

- 创建角色

```shell
kubectl create -f deploy/nfs/rbac.yaml
```

> 如果的K8S命名空间不是**default**，请在部署RBAC之前执行以下脚本:

```shell
# Set the subject of the RBAC objects to the current namespace where the provisioner is being deployed
$ NS=$(kubectl config get-contexts|grep -e "^\*" |awk '{print $5}')
$ NAMESPACE=${NS:-default}
$ sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/nfs/rbac.yaml
```

- 创建 `ServiceAccount` 和部署 `NFS-Client Provisioner`

```shell
kubectl create -f deploy/nfs/deployment.yaml
```

- 创建 NFS StorageClass

```shell
kubectl create -f deploy/nfs/class.yaml
```

- 验证NFS部署成功

```shell
kubectl get pod -l app=nfs-client-provisioner
```





#### 部署数据库

- 部署主库

```shell
cd nacos-k8s

kubectl create -f deploy/mysql/mysql-master-nfs.yaml
```

- 部署从库

```shell
cd nacos-k8s 

kubectl create -f deploy/mysql/mysql-slave-nfs.yaml
```

- 验证数据库是否正常工作

```shell
# master
kubectl get pod 
NAME                         READY   STATUS    RESTARTS   AGE
mysql-master-gf2vd                        1/1     Running   0          111m

# slave
kubectl get pod 
mysql-slave-kf9cb                         1/1     Running   0          110m
```





#### 部署Nacos

- 修改 **deploy/nacos/nacos-pvc-nfs.yaml**

```yaml
data:
  mysql.master.db.name: "主库名称"
  mysql.master.port: "主库端口"
  mysql.slave.port: "从库端口"
  mysql.master.user: "主库用户名"
  mysql.master.password: "主库密码"
```

- 创建 Nacos

```shell
kubectl create -f nacos-k8s/deploy/nacos/nacos-pvc-nfs.yaml
```

- 验证Nacos节点启动成功

```shell
kubectl get pod -l app=nacos


NAME      READY   STATUS    RESTARTS   AGE
nacos-0   1/1     Running   0          19h
nacos-1   1/1     Running   0          19h
nacos-2   1/1     Running   0          19h
```

#### 扩容测试

- 在扩容前，使用 [`kubectl exec`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#exec)获取在pod中的Nacos集群配置文件信息

```powershell
for i in 0 1; do echo nacos-$i; kubectl exec nacos-$i cat conf/cluster.conf; done
```

StatefulSet控制器根据其序数索引为每个Pod提供唯一的主机名。 主机名采用 - 的形式。 因为nacos StatefulSet的副本字段设置为2，所以当前集群文件中只有两个Nacos节点地址



- 使用kubectl scale 对Nacos动态扩容

```bash
kubectl scale sts nacos --replicas=3
```



- 在扩容后，使用 [`kubectl exec`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#exec)获取在pod中的Nacos集群配置文件信息

```bash
for i in 0 1 2; do echo nacos-$i; kubectl exec nacos-$i cat conf/cluster.conf; done
```



- 使用 [`kubectl exec`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#exec)执行Nacos API 在每台节点上获取当前**Leader**是否一致

```bash
for i in 0 1 2; do echo nacos-$i; kubectl exec nacos-$i curl -X GET "http://localhost:8848/nacos/v1/ns/
```





- nacos-pvc-nfs.yaml or nacos-quick-start.yaml

|          名称           | 必要 |                             描述                             |
| :---------------------: | :--: | :----------------------------------------------------------: |
| `mysql.master.db.name`  |  Y   |                           主库名称                           |
|   `mysql.master.port`   |  N   |                           主库端口                           |
|   `mysql.slave.port`    |  N   |                           从库端口                           |
|   `mysql.master.user`   |  Y   |                          主库用户名                          |
| `mysql.master.password` |  Y   |                           主库密码                           |
|    `NACOS_REPLICAS`     |  N   | 确定执行Nacos启动节点数量,如果不适用动态扩容插件,就必须配置这个属性，否则使用扩容插件后不会生效 |
|   `NACOS_SERVER_PORT`   |  N   |                          Nacos 端口                          |
|   `PREFER_HOST_MODE`    |  Y   |                   启动Nacos集群按域名解析                    |

- **nfs** deployment.yaml

|     名称     | 必要 |      描述      |
| :----------: | :--: | :------------: |
| `NFS_SERVER` |  Y   | NFS 服务端地址 |
|  `NFS_PATH`  |  Y   |  NFS 共享目录  |
|   `server`   |  Y   | NFS 服务端地址 |
|    `path`    |  Y   |  NFS 共享目录  |

- mysql

|             名称             | 必要 |                    描述                    |
| :--------------------------: | :--: | :----------------------------------------: |
|    `MYSQL_ROOT_PASSWORD`     |  N   |                 ROOT 密码                  |
|       `MYSQL_DATABASE`       |  Y   |                 数据库名称                 |
|         `MYSQL_USER`         |  Y   |                数据库用户名                |
|       `MYSQL_PASSWORD`       |  Y   |                 数据库密码                 |
|   `MYSQL_REPLICATION_USER`   |  Y   |               数据库复制用户               |
| `MYSQL_REPLICATION_PASSWORD` |  Y   |             数据库复制用户密码             |
|         `Nfs:server`         |  N   | NFS 服务端地址，如果使用本地部署不需要配置 |
|          `Nfs:path`          |  N   |  NFS 共享目录，如果使用本地部署不需要配置  |







## 服务注册到Nacos

### 准备

创建新的项目或者使用原有的项目



启动nacos服务

```sh
./startup.cmd -m standalone
```



```sh
PS H:\opensoft\nacos\bin> ./startup.cmd -m standalone
"nacos is starting with standalone"

         ,--.
       ,--.'|
   ,--,:  : |                                           Nacos 1.4.3
,`--.'`|  ' :                       ,---.               Running in stand alone mode, All function modules
|   :  :  | |                      '   ,'\   .--.--.    Port: 8848
:   |   \ | :  ,--.--.     ,---.  /   /   | /  /    '   Pid: 13628
|   : '  '; | /       \   /     \.   ; ,. :|  :  /`./   Console: http://192.168.202.1:8848/nacos/index.html
'   ' ;.    ;.--.  .-. | /    / ''   | |: :|  :  ;_
|   | | \   | \__\/: . ..    ' / '   | .; : \  \    `.      https://nacos.io
'   : |  ; .' ," .--.; |'   ; :__|   :    |  `----.   \
|   | '`--'  /  /  ,.  |'   | '.'|\   \  /  /  /`--'  /
'   : |     ;  :   .'   \   :    : `----'  '--'.     /
;   |.'     |  ,     .-./\   \  /            `--'---'
'---'        `--`---'     `----'

2022-07-12 20:40:29,577 INFO Bean 'org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler@f74e835' of type [org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)

2022-07-12 20:40:29,586 INFO Bean 'methodSecurityMetadataSource' of type [org.springframework.security.access.method.DelegatingMethodSecurityMetadataSource] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)

2022-07-12 20:40:30,119 INFO Tomcat initialized with port(s): 8848 (http)

2022-07-12 20:40:30,540 INFO Root WebApplicationContext: initialization completed in 3128 ms

2022-07-12 20:40:35,718 INFO Initializing ExecutorService 'applicationTaskExecutor'

2022-07-12 20:40:35,859 INFO Adding welcome page: class path resource [static/index.html]

2022-07-12 20:40:36,259 INFO Creating filter chain: Ant [pattern='/**'], []

2022-07-12 20:40:36,308 INFO Creating filter chain: any request, [org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@48e8c32a, org.springframework.security.web.context.SecurityContextPersistenceFilter@326e0b8e, org.springframework.security.web.header.HeaderWriterFilter@7fb53256, org.springframework.security.web.csrf.CsrfFilter@21a02556, org.springframework.security.web.authentication.logout.LogoutFilter@77bc2e16, org.springframework.security.web.savedrequest.RequestCacheAwareFilter@41184371, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@167381c7, org.springframework.security.web.authentication.AnonymousAuthenticationFilter@20a7953c, org.springframework.security.web.session.SessionManagementFilter@6993c8df, org.springframework.security.web.access.ExceptionTranslationFilter@3d90eeb3]

2022-07-12 20:40:36,405 INFO Initializing ExecutorService 'taskScheduler'

2022-07-12 20:40:36,427 INFO Exposing 2 endpoint(s) beneath base path '/actuator'

2022-07-12 20:40:36,530 INFO Tomcat started on port(s): 8848 (http) with context path '/nacos'

2022-07-12 20:40:36,536 INFO Nacos started successfully in stand alone mode. use embedded storage

```





### 更改pom文件

#### spring_cloud_demo

在父工程中添加spring-cloud-alilbaba的管理依赖：

```xml
 <!--spring-cloud-alilbaba管理依赖-->
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>2.2.6.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
```



完整依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <!--
      -maven项目核心配置文件-
    Project name(项目名称)：spring_cloud_demo_nacos
    Author(作者）: mao
    Author QQ：1296193245
    GitHub：https://github.com/maomao124/
    Date(创建日期)： 2022/7/12
    Time(创建时间)： 20:10
    -->
    <groupId>mao</groupId>
    <artifactId>spring_cloud_demo</artifactId>
    <packaging>pom</packaging>
    <version>0.0.1</version>


    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.9.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>11</java.version>
    </properties>

    <modules>
        <module>user_service</module>
        <module>order_service</module>
        <module>eureka_server</module>
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

            <!--spring-cloud-alilbaba管理依赖-->
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>2.2.6.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
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





#### user_service

注释掉原有的eureka依赖



添加nacos的客户端依赖：

```xml
<!-- nacos 客户端依赖 -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
```



完整依赖：

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

        <!--eureka-client 依赖-->
        <!--<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>-->

        <!-- nacos 客户端依赖 -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
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





#### order_service

注释掉原有的eureka依赖



添加nacos的客户端依赖：

```xml
<!-- nacos 客户端依赖 -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
```



完整依赖：

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

        <!--eureka-client 依赖-->
        <!--<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>-->

        <!-- nacos 客户端依赖 -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
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





### 更改配置

注释eureka地址，添加nacos地址：





#### user_service

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


  application:
    name: userservice

#eureka:
#  client:
#    service-url:
#      defaultZone: http://127.0.0.1:10080/eureka/


  cloud:
    nacos:
      discovery:
        # nacos 服务端地址
        server-addr:  localhost:8848



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





#### order_service

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




  application:
    name: orderservice

#eureka:
#  client:
#    service-url:
#      defaultZone: http://127.0.0.1:10080/eureka/
      
      
  cloud:
    nacos:
      discovery:
        # nacos 服务端地址
        server-addr: localhost:8848


# 开启debug模式，输出调试信息，常用于检查系统运行状况
#debug: true

# 设置日志级别，root表示根节点，即整体应用日志级别
logging:
 # 日志输出到文件的文件名
  file:
     name: order_server.log
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


# 配置负载均衡规则
#userservice:
#  ribbon:
#    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule


ribbon:
  eager-load:
    # 开启饥饿加载
    enabled: true
    # 指定对 userservice 这个服务饥饿加载
    clients: userservice


server:
  port: 8081


mybatis:
  type-aliases-package: mao.order_service
  configuration:
    map-underscore-to-camel-case: true
```





### 启动

#### user_service1

```sh

OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.9.RELEASE)

2022-07-12 20:41:45.664  INFO 4088 --- [           main] mao.user_service.UserServiceApplication  : No active profile set, falling back to default profiles: default
2022-07-12 20:41:46.107  INFO 4088 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=446433d6-76f3-3404-bc4a-db6d53ede6c8
2022-07-12 20:41:46.310  INFO 4088 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8082 (http)
2022-07-12 20:41:46.317  INFO 4088 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-12 20:41:46.317  INFO 4088 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-12 20:41:46.427  INFO 4088 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-12 20:41:46.427  INFO 4088 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 749 ms
2022-07-12 20:41:46.509  INFO 4088 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2022-07-12 20:41:46.596  INFO 4088 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-07-12 20:41:46.644  WARN 4088 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-12 20:41:46.644  INFO 4088 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-12 20:41:46.647  WARN 4088 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-12 20:41:46.647  INFO 4088 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-12 20:41:46.743  INFO 4088 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-12 20:41:46.954  INFO 4088 --- [           main] o.s.s.c.ThreadPoolTaskScheduler          : Initializing ExecutorService 'Nacos-Watch-Task-Scheduler'
2022-07-12 20:41:47.504  INFO 4088 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8082 (http) with context path ''
2022-07-12 20:41:47.764  INFO 4088 --- [           main] c.a.c.n.registry.NacosServiceRegistry    : nacos registry, DEFAULT_GROUP userservice 192.168.202.1:8082 register finished
2022-07-12 20:41:47.890  INFO 4088 --- [           main] mao.user_service.UserServiceApplication  : Started UserServiceApplication in 3.075 seconds (JVM running for 3.642)

```





#### user_service2

```sh

OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.9.RELEASE)

2022-07-12 20:42:11.550  INFO 18648 --- [           main] mao.user_service.UserServiceApplication  : No active profile set, falling back to default profiles: default
2022-07-12 20:42:11.982  INFO 18648 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=446433d6-76f3-3404-bc4a-db6d53ede6c8
2022-07-12 20:42:12.168  INFO 18648 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8083 (http)
2022-07-12 20:42:12.174  INFO 18648 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-12 20:42:12.175  INFO 18648 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-12 20:42:12.286  INFO 18648 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-12 20:42:12.287  INFO 18648 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 726 ms
2022-07-12 20:42:12.368  INFO 18648 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2022-07-12 20:42:12.452  INFO 18648 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-07-12 20:42:12.500  WARN 18648 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-12 20:42:12.500  INFO 18648 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-12 20:42:12.502  WARN 18648 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-12 20:42:12.503  INFO 18648 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-12 20:42:12.595  INFO 18648 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-12 20:42:12.789  INFO 18648 --- [           main] o.s.s.c.ThreadPoolTaskScheduler          : Initializing ExecutorService 'Nacos-Watch-Task-Scheduler'
2022-07-12 20:42:13.197  INFO 18648 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8083 (http) with context path ''
2022-07-12 20:42:13.206  INFO 18648 --- [           main] c.a.c.n.registry.NacosServiceRegistry    : nacos registry, DEFAULT_GROUP userservice 192.168.202.1:8083 register finished
2022-07-12 20:42:13.338  INFO 18648 --- [           main] mao.user_service.UserServiceApplication  : Started UserServiceApplication in 2.608 seconds (JVM running for 3.141)

```



#### order_service

```sh

OpenJDK 64-Bit Server VM warning: Options -Xverify:none and -noverify were deprecated in JDK 13 and will likely be removed in a future release.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.9.RELEASE)

2022-07-12 20:42:19.847  INFO 12516 --- [           main] m.order_service.OrderServiceApplication  : No active profile set, falling back to default profiles: default
2022-07-12 20:42:20.275  INFO 12516 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=dded3331-cedf-3595-87ef-faedbec03f47
2022-07-12 20:42:20.459  INFO 12516 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8081 (http)
2022-07-12 20:42:20.466  INFO 12516 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-07-12 20:42:20.467  INFO 12516 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.43]
2022-07-12 20:42:20.575  INFO 12516 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-07-12 20:42:20.575  INFO 12516 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 716 ms
2022-07-12 20:42:20.655  INFO 12516 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2022-07-12 20:42:20.739  INFO 12516 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-07-12 20:42:20.805  WARN 12516 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-12 20:42:20.805  INFO 12516 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-12 20:42:20.807  WARN 12516 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2022-07-12 20:42:20.807  INFO 12516 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2022-07-12 20:42:20.897  INFO 12516 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-07-12 20:42:21.090  INFO 12516 --- [           main] o.s.s.c.ThreadPoolTaskScheduler          : Initializing ExecutorService 'Nacos-Watch-Task-Scheduler'
2022-07-12 20:42:21.473  INFO 12516 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8081 (http) with context path ''
2022-07-12 20:42:21.484  INFO 12516 --- [           main] c.a.c.n.registry.NacosServiceRegistry    : nacos registry, DEFAULT_GROUP orderservice 192.168.202.1:8081 register finished
2022-07-12 20:42:21.606  INFO 12516 --- [           main] m.order_service.OrderServiceApplication  : Started OrderServiceApplication in 2.544 seconds (JVM running for 3.054)
2022-07-12 20:42:21.790  INFO 12516 --- [           main] c.netflix.config.ChainedDynamicProperty  : Flipping property: userservice.ribbon.ActiveConnectionsLimit to use NEXT property: niws.loadbalancer.availabilityFilteringRule.activeConnectionsLimit = 2147483647
2022-07-12 20:42:21.810  INFO 12516 --- [           main] c.netflix.loadbalancer.BaseLoadBalancer  : Client: userservice instantiated a LoadBalancer: DynamicServerListLoadBalancer:{NFLoadBalancer:name=userservice,current list of Servers=[],Load balancer stats=Zone stats: {},Server stats: []}ServerList:null
2022-07-12 20:42:21.820  INFO 12516 --- [           main] c.n.l.DynamicServerListLoadBalancer      : Using serverListUpdater PollingServerListUpdater
2022-07-12 20:42:21.853  INFO 12516 --- [           main] c.netflix.config.ChainedDynamicProperty  : Flipping property: userservice.ribbon.ActiveConnectionsLimit to use NEXT property: niws.loadbalancer.availabilityFilteringRule.activeConnectionsLimit = 2147483647
2022-07-12 20:42:21.857  INFO 12516 --- [           main] c.n.l.DynamicServerListLoadBalancer      : DynamicServerListLoadBalancer for client userservice initialized: DynamicServerListLoadBalancer:{NFLoadBalancer:name=userservice,current list of Servers=[192.168.202.1:8082, 192.168.202.1:8083],Load balancer stats=Zone stats: {unknown=[Zone:unknown;	Instance count:2;	Active connections count: 0;	Circuit breaker tripped count: 0;	Active connections per server: 0.0;]
},Server stats: [[Server:192.168.202.1:8082;	Zone:UNKNOWN;	Total Requests:0;	Successive connection failure:0;	Total blackout seconds:0;	Last connection made:Thu Jan 01 08:00:00 CST 1970;	First connection made: Thu Jan 01 08:00:00 CST 1970;	Active Connections:0;	total failure count in last (1000) msecs:0;	average resp time:0.0;	90 percentile resp time:0.0;	95 percentile resp time:0.0;	min resp time:0.0;	max resp time:0.0;	stddev resp time:0.0]
, [Server:192.168.202.1:8083;	Zone:UNKNOWN;	Total Requests:0;	Successive connection failure:0;	Total blackout seconds:0;	Last connection made:Thu Jan 01 08:00:00 CST 1970;	First connection made: Thu Jan 01 08:00:00 CST 1970;	Active Connections:0;	total failure count in last (1000) msecs:0;	average resp time:0.0;	90 percentile resp time:0.0;	95 percentile resp time:0.0;	min resp time:0.0;	max resp time:0.0;	stddev resp time:0.0]
]}ServerList:com.alibaba.cloud.nacos.ribbon.NacosServerList@5fa0141f
2022-07-12 20:42:22.833  INFO 12516 --- [erListUpdater-0] c.netflix.config.ChainedDynamicProperty  : Flipping property: userservice.ribbon.ActiveConnectionsLimit to use NEXT property: niws.loadbalancer.availabilityFilteringRule.activeConnectionsLimit = 2147483647

```





### 访问

### 访问服务

http://localhost:8081/order/101



结果：

```json
{"id":101,"price":699900,"name":"Apple 苹果 iPhone 12 ","num":1,"userId":1,"user":{"id":1,"username":"柳岩","address":"湖南省衡阳市"}}
```





![image-20220712205138593](C:\Users\mao\Desktop\img\image-20220712205138593.png)





## 服务分级存储模型



一个**服务**可以有多个**实例**，例如我们的user_service，可以有:

- 127.0.0.1:8081
- 127.0.0.1:8082
- 127.0.0.1:8083

假如这些实例分布于全国各地的不同机房，例如：

- 127.0.0.1:8081，在上海机房
- 127.0.0.1:8082，在上海机房
- 127.0.0.1:8083，在杭州机房

Nacos就将同一机房内的实例 划分为一个**集群**。

也就是说，user_service是服务，一个服务可以包含多个集群，如杭州、上海，每个集群下可以有多个实例，形成分级模型



微服务互相访问时，应该尽可能访问同集群实例，因为本地访问速度更快。当本集群内不可用时，才访问其它集群。

杭州机房内的order_service应该优先访问同机房的user_service。





## 服务集群属性





