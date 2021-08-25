---
layout:     post
title:      "Mybatis-Plus"
subtitle:   "mybatisPlus"
date:       2021-08-01 12:00:00
author:     "Pengyujie"
header-img: "img/tag-bg.jpg"
catalog: true
tags:
    - mybatisPlus
    - Java

---

> Mybatis-Plus不需要写sql就可以实现一些简单的CRUD。
>
> Mybatis-Plus可以自动生成entity、service、mapper等一些代码节省程序员开发时间。



### 特性

- 无侵入：只做增强不做改变，引入它不会对现有工程产生影响，如丝般顺滑
- 损耗小：启动即会自动注入基本 CURD，性能基本无损耗，直接面向对象操作，BaseMapper
- 强大的 CRUD 操作：内置通用 Mapper、通用 Service，仅仅通过少量配置即可实现单表大部分 CRUD 操作，更有强大的条件构造器，满足各类使用需求，以后简单的CRUD操作，不用自己编写了 ！
- 支持 Lambda 形式调用：通过 Lambda 表达式，方便的编写各类查询条件，无需再担心字段写错
- 支持主键自动生成：支持多达 4 种主键策略（内含分布式唯一 ID 生成器 - Sequence），可自由配置，完美解决主键问题
- 内置代码生成器：采用代码或者 Maven 插件可快速生成 Mapper 、 Model 、 Service 、 Controller 层代码，支持模板引擎，更有超多自定义配置等您来使用（自动帮你生成代码）
- 内置分页插件：基于 MyBatis 物理分页，开发者无需关心具体操作，配置好插件之后，写分页等同于普通 List 查询
- 分页插件支持多种数据库：支持 MySQL、MariaDB、Oracle、DB2、H2、HSQL、SQLite、Postgre、SQLServer 等多种数据库
- 内置全局拦截插件：提供全表 delete 、 update 操作智能分析阻断，也可自定义拦截规则，预防误操作

## 快速入门

***官方链接：https://baomidou.com/guide/\***

### 导入Pom配置文件

```xml
 <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.3.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <dependencies>
        <!--1.数据库驱动-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <!--2.lombok-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        <!--3.mybatis-plus  版本很重要3.0.5-->
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.0.5</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

### 连接数据库配置

```properties
#数据库连接配置
spring:
  datasource:
    url: jdbc:mysql://192.168.1.10:3306/mybatis-plus?serverTimezone=GMT%2B8&useUnicode=true&characterEncoding=UTF-8
    username: root
    password: lichu
    driver-class-name: com.mysql.jdbc.Driver
```

### 编写实体类

```java
package com.wsk.pojo;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
@Data
@TableName("a_user")
public class User {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String name;
    private Integer age;
    private String remark;
    private String createtime;
    private String updatetime;
}
```

### 编写实体类对应的mapper接口

```java
package com.wsk.mapper;
import com.baomidou.mybatisplus.mapper.BaseMapper;
import com.wsk.pojo.User;
import org.springframework.stereotype.Repository;
//在对应的接口上面继承一个基本的接口 BaseMapper
@Mapper//代表持久层
public interface UserMapper extends BaseMapper<User> {
    //所有CRUD操作都编写完成了，不用像以前一样配置一大堆文件
}
```

### 在主启动类添加@MapperScan注解

```java
package com.wsk;
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
//扫描mapper包下的所有接口
@MapperScan("com.peng.mybatisplus.mapper")
@SpringBootApplication
public class MybatisPlusApplication {
    public static void main(String[] args) {
        SpringApplication.run(MybatisPlusApplication.class, args);
    }
}
```

### 进行Test测试

```java
@SpringBootTest(classes = App.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@RunWith(SpringRunner.class)
public class Test {
    @Autowired
    private UserMapper userMapper;

    @org.junit.Test
    public void contextLoads() {
        //参数是一个wrapper ，条件构造器，这里我们先不用 null
        User user = userMapper.selectById(1);
        System.out.println(user);
    }
}
```



- SQL MyBatis-Plus都写好了
- 方法MyBatis-Plus都写好了

## 配置日志

> 我们所有的sql是不可见的，我们希望知道他们是怎么执行的，所以要配置日志知道

```properties
#配置日志  log-impl:日志实现
mybatis-plus:
  configuration:
    #mybatis-plus配置控制台打印完整带参数SQL语句
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

<img src="/img/notes/mybatisPlus/1.png">



## CRUD扩展

### Insert

```java
@Test//测试插入
public void insertTest(){
    User user = new User();
    user.setName("wsk");
    user.setAge(18);
    user.setEmail("2803708553@qq.com");
    Integer result = userMapper.insert(user); //会帮我们自动生成id
    System.out.println(result); //受影响的行数
    System.out.println(user); //通过日志发现id会自动回填
}
```



> **数据库插入的id的默认值为:全局的唯—id**

### 主键生成策略

> **源码解释**

```java
public enum IdType {
    AUTO, //数据库id自增
    INPUT, //手动输入
    ID_WORKER, //默认的全局唯一id
    UUID, //全局唯一id  uuid
    NONE;//未设置主键
    **
}
```



### Update

```java
@Test//测试更新
public void updateTest(){
    User user = new User();
    user.setId(2L);//怎么改id？？
    //通过条件自动拼接动态Sql
    user.setName("root");
    user.setAge(12);
    user.setEmail("root@qq.com");
    int i = userMapper.updateById(user);//updateById，但是参数是个user
    System.out.println(i);
}
```

### 自动填充

创建时间、更改时间！ 这些操作一般都是自动化完成，我们不希望手动更新

阿里巴巴开发手册︰几乎所有的表都要配置 create_time、update_time！而且需要自动化

> 方式一：数据库级别（工作中不允许修改数据库级别）

1、在表中增加字段：create_time,update_time
<img src="/img/notes/mybatisPlus/2.png">

2、再次测试插入或更新方法，我们需要在实体类中同步！

```java
private Date createTime;//驼峰命名private Date updateTime;
```

3、查看结果
<img src="/img/notes/mybatisPlus/3.png">

> 方式二：代码级别

1、删除数据库的默认值，更新操作！
<img src="/img/notes/mybatisPlus/4.png">

2、实体类字段属性上需要增加注解

```java
//字段  字段添加填充内容
@TableField(fill = FieldFill.INSERT)//value = ("create_time"),
private Date createTime;
@TableField(fill = FieldFill.INSERT_UPDATE)
private Date updateTime;
```

3、编写处理器来处理这个注解即可！

```java
@Slf4j//日志
@Component//丢到springboot里   一定不要忘记把处理器加到Ioc容器中!
public class MyMetaObjectHandler extends MetaObjectHandler {//extends??
    @Override//插入时的填充策略
    public void insertFill(MetaObject metaObject) {
        log.info("==start insert ······==");
        //setFieldValByName(java.lang.String fieldName, java.lang.Object fieldVal, org.apache.ibatis.reflection.MetaObject metaObject)
        this.setFieldValByName("createTIme",new Date(),metaObject);
        this.setFieldValByName("updateTime",new Date(),metaObject);
    }
    @Override//更新时的填充策略
    public void updateFill(MetaObject metaObject) {
        log.info("==start update ······==");
        this.setFieldValByName("updateTime",new Date(),metaObject);
    }
}
```

4、测试插入/更新，观察时间

```java
@Test//测试插入
public void insertTest(){
    User user = new User();
    user.setName("live");
    user.setAge(22);
    user.setEmail("1314@qq.com");
    Integer result = userMapper.insert(user); //会帮我们自动生成id
    System.out.println(result); //受影响的行数
    System.out.println(user); //通过日志发现id会自动回填
}
@Test//测试更新
public void updateTest(){
    User user = new User();
    user.setId(1359495921613004803L);
    user.setName("test3");
    user.setAge(18); //通过条件自动拼接动态Sql
    user.setEmail("test3@qq.com");
    int i = userMapper.updateById(user);//updateById，但是参数是个user
    System.out.println(i);
}
```



### 乐观锁&悲观锁

在面试过程中经常被问到乐观锁/悲观锁，这个其实很简单

> 乐观锁：顾名思义十分乐观,他总是认为不会出现问题,无论干什么都不上锁!如果出现了问题,再次更新值测试
>
> 悲观锁：顾名思义十分悲观,他总是认为出现问题,无论干什么都会上锁!再去操作!

我们这里主要讲解 乐观锁机制!

乐观锁实现方式:

- 取出记录时,获取当前version
- 更新时,带上这个version
- 执行更新时,set version = newVersion where version = oldVersion
- 如果version不对,就更新失败

```sql
乐观锁：先查询，获得版本号
-- A
update user set name = "wsk",version = version+1 
where id = 1 and version = 1
-- B  （B线程抢先完成，此时version=2，会导致A线程修改失败！）
update user set name = "wsk",version = version+1 
where id = 1 and version = 1
```

> 测试一下Mybatis-Plus乐观锁插件

1、给数据库中增加version字段
<img src="/img/notes/mybatisPlus/5.png">

2、实体类加对应的字段

```java
@Version//乐观锁version注解
private Integer version;
```

3、注册组件

```java
//扫描mapper文件夹
@MapperScan("com.wsk.mapper")//交给mybatis做的，可以让这个配置类做扫描
@EnableTransactionManagement//自动管理事务
@Configuration//配置类
public class MyBatisPlusConfig {
    //注册乐观锁插件
    @Bean
    public OptimisticLockerInterceptor optimisticLockerInterceptor(){
        return new OptimisticLockerInterceptor();
    }
}
```

4、测试一下

- 成功

```java
@Test//测试乐观锁成功
public void testOptimisticLocker1(){
    //1、查询用户信息
    User user = userMapper.selectById(1L);
    //2、修改用户信息
    user.setAge(18);
    user.setEmail("2803708553@qq.com");
    //3、执行更新操作
    userMapper.updateById(user);
}
```

<img src="/img/notes/mybatisPlus/6.png">

<img src="/img/notes/mybatisPlus/7.png">

- 失败

```java
@Test//测试乐观锁失败  多线程下
public void testOptimisticLocker2(){
    //线程1
    User user1 = userMapper.selectById(1L);
    user1.setAge(1);
    user1.setEmail("2803708553@qq.com");
    //模拟另外一个线程执行了插队操作
    User user2 = userMapper.selectById(1L);
    user2.setAge(2);
    user2.setEmail("2803708553@qq.com");
    userMapper.updateById(user2);
    //自旋锁来多次尝试提交！
    userMapper.updateById(user1);//如果没有乐观锁就会覆盖插队线程的值
}
```

<img src="/img/notes/mybatisPlus/8.png">

<img src="/img/notes/mybatisPlus/9.png">

### Select

- 通过id查询单个用户

```java
@Test//通过id查询单个用户
public void testSelectById(){
    User user = userMapper.selectById(1L);
    System.out.println(user);
}
```

- 通过id查询多个用户

```java
@Test//通过id查询多个用户
public void testSelectBatchIds(){
    List<User> users = userMapper.selectBatchIds(Arrays.asList(1L, 2L, 3L));
    users.forEach(System.out::println);
    //System.out.println(users);
}
```

- 条件查询 通过map封装

```java
@Test//通过条件查询之一  map
public void testMap(){
    HashMap<String, Object> map = new HashMap<>();
    //自定义要查询的
    map.put("name","www");
    map.put("age",18);
    List<User> users = userMapper.selectByMap(map);
    users.forEach(System.out::println);
}
```

- xml查询 

```xml
  int selectCount(@Param(Constants.WRAPPER) QueryWrapper<User> queryWrapper);
      
  Page<User> selectByPage(@Param("page") Page page, @Param(Constants.WRAPPER) QueryWrapper<User> queryWrapper);
      
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.lichu.marketing.activity.mapper.ActivityCenterMapper">
    <!--  条件分页查询  -->
    <select id="selectCount" resultType="java.lang.Integer">
        select
    	 count(1)
        from a_user
        <!--  简略条件语句  -->
        ${ew.customSqlSegment}
    </select>
    
        <!--  条件分页查询  -->
    <select id="selectByPage" resultType="com.lichu.marketing.activity.entity.ActivityCenter">
        select
    	 *
        from a_user
        <!--  简略条件语句  -->
        ${ew.customSqlSegment}
    </select>
</mapper>



```



分页在网站的使用十分之多！

1、原始的limit分页

2、pageHelper第三方插件

3、MybatisPlus其实也内置了分页插件！

> 如何使用：

1、配置拦截器组件

```java
    /**
     * 拦截器配置
     */
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL)); // 分页插件
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());//针对 update 和 delete 语句 作用: 阻止恶意的全表更新删除
        return interceptor;
    }
```

2、直接使用page对象即可

```java
@Test//测试分页查询
public void testPage(){
    //参数一current：当前页   参数二size：页面大小
    //使用了分页插件之后，所有的分页操作都变得简单了
    Page<User> page = new Page<>(1,10);
    userMapper.selectPage(page,null);
    page.getRecords().forEach(System.out::println);
    System.out.println("总页数==>"+page.getTotal());
}![img](https://kuangstudy.oss-cn-beijing.aliyuncs.com/bbs/2021/03/01/kuangstudyf4b59842-0c11-4458-b7f3-f795f7c0c59d.png)
```

### Delete

> 基本的删除任务：

```java
@Test
public void testDeleteById(){
    userMapper.deleteById(1359507762519068681L);
}
@Test
public void testDeleteBatchIds(){
  userMapper.deleteBatchIds(Arrays.asList(1359507762519068675L,1359507762519068676L));
}
@Test
public void testD(){
    HashMap<String, Object> map = new HashMap<>();
    map.put("age","18");
    map.put("name","lol");
    userMapper.deleteByMap(map);
}
```

我们在工作中会遇到一些问题：逻辑删除！

### 逻辑删除

> 物理删除：从数据库中直接删除
>
> 逻辑删除：在数据库中没有被删除，而是通过一个变量来使他失效！ deleted=0 ==> deleted=1

**管理员可以查看被删除的记录！防止数据的丢失，类似于回收站！**

***测试一下：\***

> 1、在数据表中增加一个deleted字段

<img src="/img/notes/mybatisPlus/10.png">

> 2、实体类中添加对应属性

```java
@TableLogic//逻辑删除注解
private Integer deleted;
```

> 3、配置！

```java
//逻辑删除组件
@Bean
public ISqlInjector sqlInjector(){
    return new LogicSqlInjector();
}

#配置逻辑删除  没删除的为0 删除的为1
mybatis-plus.global-config.db-config.logic-delete-value=1
mybatis-plus.global-config.db-config.logic-not-delete-value=0
```

> 4、测试一下删除

<img src="/img/notes/mybatisPlus/11.png">

发现： 记录还在，deleted变为1

再次测试查询被删除的用户，发现查询为空



**以上所有的CRUD及其扩展操作，我们都必须精通掌握！会大大提高工作写项目的效率！**



## 条件构造器

**十分重要：Wrapper** 记住查看输出的SQL进行分析

> 1、测试一

```java
@Test
public void testWrapper1() {
    //参数是一个wrapper ，条件构造器，和刚才的map对比学习！
    //查询name不为空，email不为空，age大于18的用户
    QueryWrapper<User> wrapper = new QueryWrapper<>();
    wrapper
        .isNotNull("name")
        .isNotNull("email")
        .ge("age",18);
    List<User> userList = userMapper.selectList(wrapper);
    userList.forEach(System.out::println);
}
```

> 测试二

```java
@Test
public void testWrapper2() {
    //查询name=wsk的用户
    QueryWrapper<User> wrapper = new QueryWrapper<>();
    wrapper.eq("name","wsk");
    //查询一个数据selectOne，若查询出多个会报错
    //Expected one result (or null) to be returned by selectOne(), but found: *
    //若出现多个结果使用list或map
    User user = userMapper.selectOne(wrapper);//查询一个数据，若出现多个结果使用list或map
    System.out.println(user);
}
```

> 测试三

```java
@Test
public void testWrapper3() {
    //查询age在10-20之间的用户
    QueryWrapper<User> wrapper = new QueryWrapper<>();
    wrapper.between("age", 10, 20);//区间
    Integer count = userMapper.selectCount(wrapper);//输出查询的数量selectCount
    System.out.println(count);
}
```

> 测试四

```java
@Test
public void testWrapper4() {
    //模糊查询
    QueryWrapper<User> wrapper = new QueryWrapper<>();
    wrapper
        .notLike("name","s")
        .likeRight("email","t");//qq%  左和右？
    List<Map<String, Object>> maps = userMapper.selectMaps(wrapper);
    maps.forEach(System.out::println);
}
```

> 测试五

```java
@Test
public void testWrapper5() {
    //模糊查询
    // SELECT id,name,age,email,version,deleted,create_time,update_time 
    //FROM user 
    //WHERE deleted=0 AND id IN 
    //(select id from user where id<5)
    QueryWrapper<User> wrapper = new QueryWrapper<>();
    //id 在子查询中查出来
    wrapper.inSql("id","select id from user where id<5");
    List<Object> objects = userMapper.selectObjs(wrapper);
    objects.forEach(System.out::println);
}
```

> 测试六

```java
@Test
public void testWrapper6() {
    QueryWrapper<User> wrapper = new QueryWrapper<>();
    //通过id进行降序排序
    wrapper.orderByDesc("id");
    List<User> userList = userMapper.selectList(wrapper);
    userList.forEach(System.out::println);
}
```



## 代码自动生成器

`AutoGenerator` 是 MyBatis-Plus 的代码生成器，通过 `AutoGenerator` 可以快速生成 Entity、Mapper、Mapper XML、Service、Controller 等各个模块的代码，极大的提升了开发效率。

**[官网笔记](https://mp.baomidou.com/guide/generator.html)**

首先 编写一个主类然后启动。

~~~java
package com.peng.mybatisplus;

import com.baomidou.mybatisplus.core.exceptions.MybatisPlusException;
import com.baomidou.mybatisplus.core.toolkit.StringPool;
import com.baomidou.mybatisplus.core.toolkit.StringUtils;
import com.baomidou.mybatisplus.generator.AutoGenerator;
import com.baomidou.mybatisplus.generator.InjectionConfig;
import com.baomidou.mybatisplus.generator.config.*;
import com.baomidou.mybatisplus.generator.config.po.TableInfo;
import com.baomidou.mybatisplus.generator.config.rules.NamingStrategy;
import com.baomidou.mybatisplus.generator.engine.FreemarkerTemplateEngine;

import java.util.ArrayList;
import java.util.List;
import java.util.Scanner;

public class CodeGenerator {
    /**
     * <p>
     * 读取控制台内容
     * </p>
     */
    public static String scanner(String tip) {
        Scanner scanner = new Scanner(System.in);
        StringBuilder help = new StringBuilder();
        help.append("请输入" + tip + "：");
        System.out.println(help.toString());
        if (scanner.hasNext()) {
            String ipt = scanner.next();
            if (StringUtils.isNotBlank(ipt)) {
                return ipt;
            }
        }
        throw new MybatisPlusException("请输入正确的" + tip + "！");
    }

    public static void main(String[] args) {
        // 代码生成器
        AutoGenerator mpg = new AutoGenerator();

        // 全局配置
        GlobalConfig gc = new GlobalConfig();
        String projectPath = System.getProperty("user.dir");
        gc.setOutputDir(projectPath + "/src/main/java");
        gc.setAuthor("pengyujie");
        gc.setOpen(false);
        gc.setSwagger2(true); //实体属性 Swagger2 注解
        mpg.setGlobalConfig(gc);

        // 数据源配置
        DataSourceConfig dsc = new DataSourceConfig();
        dsc.setUrl("jdbc:mysql://localhost:3306/mybatis-plus?useUnicode=true&useSSL=false&characterEncoding=utf8");//这里的路径要和自己要自动创建的数据库的一样
        // dsc.setSchemaName("public");
        dsc.setDriverName("com.mysql.jdbc.Driver");
        dsc.setUsername("root");
        dsc.setPassword("123456");//自己的密码
        mpg.setDataSource(dsc);

        // 包配置
        PackageConfig pc = new PackageConfig();
        pc.setModuleName(scanner("模块名"));
        pc.setParent("com.mybatisPlus.auto");
        pc.setEntity("entity");
        pc.setMapper("mapper");
        pc.setService("service");
        mpg.setPackageInfo(pc);

        // 自定义配置
        InjectionConfig cfg = new InjectionConfig() {
            @Override
            public void initMap() {
                // to do nothing
            }
        };

        // 如果模板引擎是 freemarker
        String templatePath = "/templates/mapper.xml.ftl";
        // 如果模板引擎是 velocity
         //String templatePath = "/templates/mapper.xml.vm";

        // 自定义输出配置
        List<FileOutConfig> focList = new ArrayList<>();
        // 自定义配置会被优先输出
        focList.add(new FileOutConfig(templatePath) {
            @Override
            public String outputFile(TableInfo tableInfo) {
                // 自定义输出文件名 ， 如果你 Entity 设置了前后缀、此处注意 xml 的名称会跟着发生变化！！
                return projectPath + "/src/main/resources/mapper/" + pc.getModuleName()
                        + "/" + tableInfo.getEntityName() + "Mapper" + StringPool.DOT_XML;
            }
        });
        /*
        cfg.setFileCreate(new IFileCreate() {
            @Override
            public boolean isCreate(ConfigBuilder configBuilder, FileType fileType, String filePath) {
                // 判断自定义文件夹是否需要创建
                checkDir("调用默认方法创建的目录，自定义目录用");
                if (fileType == FileType.MAPPER) {
                    // 已经生成 mapper 文件判断存在，不想重新生成返回 false
                    return !new File(filePath).exists();
                }
                // 允许生成模板文件
                return true;
            }
        });
        */
        cfg.setFileOutConfigList(focList);
        mpg.setCfg(cfg);

        // 配置模板
        TemplateConfig templateConfig = new TemplateConfig();

        // 配置自定义输出模板
        //指定自定义模板路径，注意不要带上.ftl/.vm, 会根据使用的模板引擎自动识别
        // templateConfig.setEntity("templates/entity2.java");
        // templateConfig.setService();
        // templateConfig.setController();

        templateConfig.setXml(null);
        mpg.setTemplate(templateConfig);

        // 策略配置
        StrategyConfig strategy = new StrategyConfig();
        strategy.setNaming(NamingStrategy.underline_to_camel);
        strategy.setColumnNaming(NamingStrategy.underline_to_camel);
        //strategy.setSuperEntityClass("你自己的父类实体,没有就不用设置!");
        strategy.setEntityLombokModel(true);
        strategy.setRestControllerStyle(true);
        // 公共父类
        //strategy.setSuperControllerClass("你自己的父类控制器,没有就不用设置!");
        // 写于父类中的公共字段
        strategy.setSuperEntityColumns("id");
        strategy.setInclude(scanner("表名，多个英文逗号分割").split(","));
        strategy.setControllerMappingHyphenStyle(true);
        strategy.setTablePrefix(pc.getModuleName() + "_");
        mpg.setStrategy(strategy);
        mpg.setTemplateEngine(new FreemarkerTemplateEngine());
        mpg.execute();
    }
}
~~~



<img src="/img/notes/mybatisPlus/12.png">

填写要生成的对应 模块名 和数据库中已经建好的表名。



<img src="/img/notes/mybatisPlus/13.png">

自动生成完成，对于数据库数量大的时候，自动生成节省了我们大量的写代码时间
