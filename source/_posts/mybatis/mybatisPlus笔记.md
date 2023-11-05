---
title: mybatisPlus笔记
auther: gzj
tags:
  - 学习笔记
  - mybatisPlus
categories: mybatis
description: MyBatis-Plus（简称 MP）是一个 MyBatis 的增强工具，在 MyBatis 的基础上只做增强不做改变，为简化开发、提高效率而生。
abbrlink: 13667b25
date: 2021-01-12 16:16:58
---

版本：3.4.1

## mybatis-plus的yml常用配置

```yaml
mybatis-plus:
  # Mapper.xml 文件位置 Maven 多模块项目的扫描路径需以 classpath*: 开头
  mapperLocations: classpath*:com/vanhr/**/xml/*Mapper.xml
#  #通过父类（或实现接口）的方式来限定扫描实体
#  typeAliasesSuperType: com.vanhr.user.dao.entity.baseEntity
#  #枚举类 扫描路径 如果配置了该属性，会将路径下的枚举类进行注入，让实体类字段能够简单快捷的使用枚举属性
#  typeEnumsPackage: com.vanhr.user.dao.enums
#  #通过该属性可指定 MyBatis 的执行器，MyBatis 的执行器总共有三种：
#  # ExecutorType.SIMPLE：该执行器类型不做特殊的事情，为每个语句的执行创建一个新的预处理语句（PreparedStatement）
#  # ExecutorType.REUSE：该执行器类型会复用预处理语句（PreparedStatement）
#  # ExecutorType.BATCH：该执行器类型会批量执行所有的更新语句
#  executorType: SIMPLE
#  # 指定外部化 MyBatis Properties 配置，通过该配置可以抽离配置，实现不同环境的配置部署
#  configurationProperties:
  configuration: # MyBatis 原生支持的配置
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl #日志输出
    # 是否开启自动驼峰命名规则（camel case）映射
    mapUnderscoreToCamelCase: true
    # 枚举处理类,如果配置了该属性,枚举将统一使用指定处理器进行处理
    #   org.apache.ibatis.type.EnumTypeHandler : 存储枚举的名称
    #   org.apache.ibatis.type.EnumOrdinalTypeHandler : 存储枚举的索引
    #   com.baomidou.mybatisplus.extension.handlers.MybatisEnumTypeHandler : 枚举类需要实现IEnum接口或字段标记@EnumValue注解.(3.1.2以下版本为EnumTypeHandler)
#    defaultEnumTypeHandler: com.baomidou.mybatisplus.extension.handlers.MybatisEnumTypeHandler
    # 配置JdbcTypeForNull, oracle数据库必须配置
    jdbc-type-for-null: null
  global-config: # 全局策略配置
    # 是否控制台 print mybatis-plus 的 LOGO
    banner: false
    db-config:
      # id类型
      id-type: auto
      # 表名是否使用下划线命名，默认数据库表使用下划线命名
      table-underline: true
      #是否开启大写命名，默认不开启
#      capital-mode: false
#      #逻辑已删除值,(逻辑删除下有效) 需要注入逻辑策略LogicSqlInjector 以@Bean方式注入
#      logic-not-delete-value: 0
#      #逻辑未删除值,(逻辑删除下有效)
#      logic-delete-value: 1
```

## crud操作

### 插入操作

```
@SpringBootTest
public class CRUDTests {
    @Autowired
    private UserMapper userMapper;
    @Test
    public void testInsert(){
        User user = new User();
        user.setName("Helen");
        user.setAge(18);
        user.setEmail("55317332@qq.com");
        int result = userMapper.insert(user);
        System.out.println(result); //影响的行数
        System.out.println(user); //id自动回填
    }
}
```

#### 插入时的主键策略

MP 支持多种主键策略 默认是推特的“雪花算法“ ，也可以设置其他策略。

**IdType**

|     值      |                             描述                             |
| :---------: | :----------------------------------------------------------: |
|    AUTO     |                         数据库ID自增                         |
|    NONE     | 无状态,该类型为未设置主键类型(注解里等于跟随全局,全局里约等于 INPUT)，默认根据雪花算法生成 |
|    INPUT    |                    insert前自行set主键值                     |
|  ASSIGN_ID  | 分配ID(主键类型为Number(Long和Integer)或String)(since 3.3.0),使用接口`IdentifierGenerator`的方法`nextId`(默认实现类为`DefaultIdentifierGenerator`雪花算法) |
| ASSIGN_UUID | 分配UUID,主键类型为String(since 3.3.0),使用接口`IdentifierGenerator`的方法`nextUUID`(默认default方法) |

**相关资料：分布式系统唯一ID生成方案汇总：**https://www.cnblogs.com/haoxinyue/p/5208136.html

要想影响所有实体的配置，可以设置全局主键配置

```
#全局设置主键生成策略
mybatis-plus.global-config.db-config.id-type=auto
```

### 更新操作

#### 根据Id更新操作

**注意：**update时生成的sql自动是动态sql：UPDATE user SET age=? WHERE id=? 

```
@Test
public void testUpdateById(){
    User user = new User();
    user.setId(1L);
    user.setAge(28);
    int result = userMapper.updateById(user);
    System.out.println(result);
}
```

#### 自动填充

项目中经常会遇到一些数据，每次都使用相同的方式填充，例如记录的创建时间，更新时间等。

我们可以使用MyBatis Plus的自动填充功能，完成这些字段的赋值工作：

首先在实体上添加注解`@TableField(fill = FieldFill.INSERT)`，然后实现元对象处理器接口，不要在类上忘记添加 @Component 注解。

```java
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {//插入操作
        this.setFieldValByName("createTime", new Date(), metaObject);
        this.setFieldValByName("updateTime", new Date(), metaObject);
    }
    @Override
    public void updateFill(MetaObject metaObject) {//更新操作
        this.setFieldValByName("updateTime", new Date(), metaObject);
    }
}
```

FieldFill类的一些属性：

```
/**
* 默认不处理
*/
DEFAULT,
/**
* 插入时填充字段
*/
INSERT,
/**
* 更新时填充字段
*/
UPDATE,
/**
* 插入和更新时填充字段
*/
INSERT_UPDATE
```

#### 乐观锁

**OptimisticLockerInnerInterceptor**

> 当要更新一条记录的时候，希望这条记录没有被别人更新
> 乐观锁实现方式：
>
> > - 取出记录时，获取当前version
> > - 更新时，带上这个version
> > - 执行更新时， set version = newVersion where version = oldVersion
> > - 如果version不对，就更新失败

首先数据库表要加version字段，然后字段上加上`@Version`注解，然后元对象处理器接口添加version的insert默认值。

```java
@Version
@TableField(fill = FieldFill.INSERT)//也可以设置数据库default值
private Integer version;//version要有初始值，可以使用自动填充
```

说明:

- **支持的数据类型只有:int,Integer,long,Long,Date,Timestamp,LocalDateTime**
- 整数类型下 `newVersion = oldVersion + 1`
- `newVersion` 会回写到 `entity` 中
- 仅支持 `updateById(id)` 与 `update(entity, wrapper)` 方法
- **在 `update(entity, wrapper)` 方法下, `wrapper` 不能复用!!!**

**3.4版本后的配置写法：（如果对多个插件进行配置(本例是乐观锁和分页)，要写在一个方法内）**

```
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor()); // 乐观锁插件
    // DbType：数据库类型(根据类型获取应使用的分页方言)
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL)); // 分页插件
    return interceptor;
}
```

### 查询操作

#### **通过多个id批量查询**

完成了动态sql的foreach的功能

```java
@Test
public void testSelectBatchIds(){
    System.out.println(userMapper.selectBatchIds(Arrays.asList(1, 2, 3)));
}
```

#### **简单的条件查询**

通过Wrapper封装查询条件

```java
@Test
public void testSelectByMap(){
    //通过lambda
    System.out.println(userMapper.selectList(Wrappers.lambdaQuery(User.class)
                .eq(User::getName, "Jone").eq(User::getAge, 18)));
    //不用lambda
    System.out.println(userMapper.selectList(Wrappers.query(new User())
                .eq("name", "Jone").eq("age", 18)));
}
```

**注意：**map中的key对应的是数据库中的列名。例如数据库user_id，实体类是userId，这时map的key需要填写user_id

#### 分页查询

MyBatis Plus自带分页插件，只要简单的配置即可实现分页功能

**（1）创建配置类**

```java
/**
 * 新的分页插件,一缓和二缓遵循mybatis的规则,需要设置 
 * MybatisConfiguration#useDeprecatedExecutor = false 
 * 避免缓存出现问题(该属性会在旧插件移除后一同移除)
 */ 
@Bean
 public MybatisPlusInterceptor mybatisPlusInterceptor() {
     PaginationInnerInterceptor paginationInnerInterceptor = new PaginationInnerInterceptor(DbType.MYSQL);
        paginationInnerInterceptor.setMaxLimit(500L);//单页分页条数限制
        interceptor.addInnerInterceptor(paginationInnerInterceptor); // 分页插件
     return interceptor;
 }

//不知道为什么加这个，但官网示例加了，我也加
@Bean
public ConfigurationCustomizer configurationCustomizer() {
    return configuration -> configuration.setUseDeprecatedExecutor(false);
}
```

**（2）测试selectPage分页**

**测试：**最终通过page对象获取相关数据

```java
@Test
public void testSelectPage() {
    Page<User> page = new Page<>(1,5);
    userMapper.selectPage(page, null);
    page.getRecords().forEach(System.out::println);
    System.out.println(page.getCurrent());
    System.out.println(page.getPages());
    System.out.println(page.getSize());
    System.out.println(page.getTotal());
    System.out.println(page.hasNext());
    System.out.println(page.hasPrevious());
}
```

控制台sql语句打印：SELECT id,name,age,email,create_time,update_time FROM user LIMIT 0,5 

**（3）测试selectMapsPage分页：结果集是Map**

```java
@Test
public void testSelectMapsPage() {
    Page<Map<String, Object>> page = new Page<>(1, 5);
    IPage<Map<String, Object>> mapIPage = userMapper.selectMapsPage(page, null);
    //注意：此行必须使用 mapIPage 获取记录列表，否则会有数据类型转换错误
    mapIPage.getRecords().forEach(System.out::println);
    System.out.println(page.getCurrent());
    System.out.println(page.getPages());
    System.out.println(page.getSize());
    System.out.println(page.getTotal());
    System.out.println(page.hasNext());
    System.out.println(page.hasPrevious());
}
```

### 删除操作

#### 根据id删除记录

```java
@Test
public void testDeleteById(){
    int result = userMapper.deleteById(8L);
    System.out.println(result);
}
```

#### 批量删除

```java
@Test
public void testDeleteBatchIds() {
    int result = userMapper.deleteBatchIds(Arrays.asList(8, 9, 10));
    System.out.println(result);
}
```

#### 逻辑删除

> 说明:
>
> 只对自动注入的sql起效:
>
> - 插入: 不作限制
>
> - 查找: 追加where条件过滤掉已删除数据,且使用 wrapper.entity 生成的where条件会忽略该字段
>
>   >  使用 QueryWrapper 进行查询时的sql，我们发现前面的`deleted=0`条件会让后面我们自己加的deleted条件失效
>   >
>   > SELECT * FROM test.user WHERE deleted=0 AND (deleted = ?)
>
> - 更新: 追加where条件防止更新到已删除数据,且使用 wrapper.entity 生成的where条件会忽略该字段
>
> - 删除: 转变为 更新
>
> 例如:
>
> - 删除: `update user set deleted=1 where id = 1 and deleted=0`
> - 查找: `select id,name,deleted from user where deleted=0`
>
> 字段类型支持说明:
>
> - 支持所有数据类型(推荐使用 `Integer`,`Boolean`,`LocalDateTime`)
> - 如果数据库字段使用`datetime`,逻辑未删除值和已删除值支持配置为字符串`null`,另一个值支持配置为函数来获取值如`now()`
>
> 附录:
>
> - 如果要恢复删除的数据，可以使用mybatis手写sql来实现，绕过mybatis-plus
> - 逻辑删除是为了方便数据恢复和保护数据本身价值等等的一种方案，但实际就是删除。
> - 如果你需要频繁查出来看就不应使用逻辑删除，而是以一个状态去表示。

**数据库中添加 deleted字段**

```
ALTER TABLE `user` ADD COLUMN `deleted` boolean
```

![img](https://raw.githubusercontent.com/zzugzj/blogImg/master/img/bc4cbff4-c2b8-45d5-ae8d-53439dd2330c.png)

**实体类添加deleted 字段**

并加上 @TableLogic 注解 和 @TableField(fill = FieldFill.INSERT) 注解

```
@TableLogic
@TableField(fill = FieldFill.INSERT)//也可以设置数据库default值
private Integer deleted;
```

**元对象处理器接口添加deleted的insert默认值**

```
@Override
public void insertFill(MetaObject metaObject) {
    this.setFieldValByName("deleted", 0, metaObject);
}
```

**application.properties 加入配置**

此为默认值，如果你的默认值和mp默认的一样,该配置可无

```
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: flag  # 全局逻辑删除的实体字段名(since 3.3.0,配置后可以不用往flag字段上加@TableLogic注解)
      logic-delete-value: 1 # 逻辑已删除值(默认为 1)
      logic-not-delete-value: 0 # 逻辑未删除值(默认为 0)
```

**在 MybatisPlusConfig 中注册 Bean**

```
@Bean
public ISqlInjector sqlInjector() {
    return new LogicSqlInjector();
}
```

## 性能分析

>  该功能依赖 `p6spy` 组件，完美的输出打印 SQL 及执行时长 `3.1.0` 以上版本

- p6spy 依赖引入

Maven：

```xml
<dependency>
  <groupId>p6spy</groupId>
  <artifactId>p6spy</artifactId>
  <version>最新版本</version>
</dependency>
```

- application.yml 配置：

```yaml
spring:
  datasource:
    driver-class-name: com.p6spy.engine.spy.P6SpyDriver
    url: jdbc:p6spy:mysql://localhost:3306/数据库名
    ...
```

- spy.properties 配置：

```properties
#3.2.1以上使用
modulelist=com.baomidou.mybatisplus.extension.p6spy.MybatisPlusLogFactory,com.p6spy.engine.outage.P6OutageFactory
#3.2.1以下使用或者不配置
#modulelist=com.p6spy.engine.logging.P6LogFactory,com.p6spy.engine.outage.P6OutageFactory
# 自定义日志打印
logMessageFormat=com.baomidou.mybatisplus.extension.p6spy.P6SpyLogger
#日志输出到控制台
appender=com.baomidou.mybatisplus.extension.p6spy.StdoutLogger
# 使用日志系统记录 sql
#appender=com.p6spy.engine.spy.appender.Slf4JLogger
# 设置 p6spy driver 代理
deregisterdrivers=true
# 取消JDBC URL前缀
useprefix=true
# 配置记录 Log 例外,可去掉的结果集有error,info,batch,debug,statement,commit,rollback,result,resultset.
excludecategories=info,debug,result,commit,resultset
# 日期格式
dateformat=yyyy-MM-dd HH:mm:ss
# 实际驱动可多个
#driverlist=org.h2.Driver
# 是否开启慢SQL记录
outagedetection=true
# 慢SQL记录标准 2 秒
outagedetectioninterval=2
```

> 注意！
>
> - driver-class-name 为 p6spy 提供的驱动类
> - url 前缀为 jdbc:p6spy 跟着冒号为对应数据库连接地址
> - 打印出sql为null,在excludecategories增加commit
> - 批量操作不打印sql,去除excludecategories中的batch
> - 批量操作打印重复的问题请使用MybatisPlusLogFactory (3.2.1新增）
> - 该插件有性能损耗，不建议生产环境使用。