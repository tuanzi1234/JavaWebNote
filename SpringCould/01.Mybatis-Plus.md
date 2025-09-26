### 一、常见注解
1. **mybatis-plus原理**
   通过扫描实体类，基于反射获取实体类信息作为数据库表信息
   类名驼峰转下划线作为表名
   名为id的字段作为主键
   变量名驼峰转下划线作为字段名
2. **mybatis-plus常用注解**
   * **`@TableName`: 指定表名**
     `@TableName("表名")`
   * **`@TableId`: 指定主键**
     `@TableId(value = "id", type = IdType.AUTO)`
     `IdType.AUTO`: 主键自增
     `IdType.INPUT`: 手动输入
     `IdType.ASSIGN_ID`: 雪花算法生成ID(默认)
   * **`@TableField`: 指定表中普通字段**
     `@TableField(value = "字段名")`
     使用场景:
     (1)成员变量与数据库字段名不一致
     (2)成员变量名以is开头，且为布尔类型
     (3)成员变量名与数据库关键字冲突，此时字段名还需要加反引号
     (4)成员变量不是数据库字段，此时需要在括号里加上`exist = false`
### 二、常用配置
   `MybatisPlus`继承了`Mybatis`的原生配置和一些特有配置
1. **`MybatisPlus`配置文件**
   ```yml
   mybatis-plus:
     type-aliases-package: com.ruoyi.system.domain # 别名扫描包
     mapper-locations: classpath*:mapper/**/*.xml # Mapper映射文件路径
     config-location: 
       map-underscore-to-camel-case: true # 开启驼峰命名
       cache-enabled: false # 禁用二级缓存
     global-config:
       db-config:
         id-type: auto # id生成方式为自增
         update-strategy: not_null # 更新策略，只更新非空字段
   ```
   
### 三、核心功能
1. **条件构造器**
   例:查询`id`, `username`, `info`, `balance`字段, 并且`username`名字带`o`, 余额为1000。
   ```java
   QueryWrapper<User> queryWrapper = new QueryWrapper<User>()
         .select("id", "username", "info", "balance")
         .like("username", "o")
         .eq("balance", 1000);
   List<User> list = userMapper.selectList(queryWrapper);
   list.forEach(System.out::println);
   ```
   上述例子还可以使用lambda表达式：
   ```java
   QueryWrapper<User> queryWrapper = new QueryWrapper<User>()
        .select(User::getId, User::getUsername, User::getInfo, User::getBalance) 
        .like(User::getUsername, "o")
        .eq(User::getBalance, 1000);
   List<User> list = userMapper.selectList(queryWrapper);
   list.forEach(System.out::println);
   ```
2. **自定义SQL**
   例: 将id为1，2，3的用户的余额减200
   ```java
   void selectListByCustomSql() { 
    // 构建条件
    List<Long> ids = Arrays.asList(1L, 2L, 3L);
    int amount = 200;
    // 定义条件
    QueryWrapper<User> wrapper = new QueryWrapper<User>().in("id", ids);
    userMapper.update(wrapper, amount);
   }
   ```
   ```java
   // Mapper层
   void update(@Param(Constants.WRAPPER) QueryWrapper<User> wrapper, @Param("amount") int amount);
   ```
   ```xml
   <update id="update"> 
   UPDATE user SET amount = amount - #{amount} ${ew.customSqlSegment}
   </update>
   ```
3. **IService**
   `IService`是`Mybatis-Plus`提供的一个服务接口
   使用步骤:
   (1)创建一个接口继承`IService`:`public interface UserService extends IService<User> {...}`
   (2)创建一个实现类:`public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {...}`
4. **批量插入数据**
   (1)普通for循环(性能极差，不推荐)
   ```java
   for (int i = 0; i < 1000; i++) {
    userService.save(buildUser(i));
   }
   ```
   (2)批量插入(性能相比方法一有提升)
   ```java
   List<User> list = new ArrayList<>(100);
   for (int i = 0; i < 1000; i++) {
    list.add(buildUser(i));
    if (i % 100 == 0) {
      userService.saveBatch(list);
      list.clear();
    }
   }
   ```
   (3)开启`rewriteBatchedStatements=true`再批量插入
   在yaml的sql配置中添加`rewriteBatchedStatements=true`
   ```yml
    datasource:
        url: jdbc:mysql://127.0.0.1:3306/mp?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true
    ```
5. **逻辑删除**
   `Mybatis-plus`中提供了逻辑删除功能:
   ```yml
   mybatis-plus:
     global-config:
       db-config:
         logic-delete-field: flag # 全局逻辑删除的实体字段名，类型可以是boolean, integer
         logic-delete-value: 1 # 逻辑已删除值(默认为1)
         logic-not-delete-value: 0 # 逻辑未删除值(默认为0)
   ```
   **逻辑删除的问题**:
   (1)数据库中的垃圾数据会越来越多，影响查询效率
   (2)SQL中全都需要对逻辑删除字段做判断，影响效率
### 四、拓展功能
1. **枚举处理器**
   **在application.yml中添加**:
   ```yml
   mybatis-plus:
     configuration:
       default-enum-type-handler: com.baomidou.mybatisplus.core.handlers.MybatisEnumTypeHandler
   ```
2. **Json处理器**
   在需要使用Json的实体类字段上添加`@TableField(typeHandler = JacksonTypeHandler.class)`注解
   在实体类上添加`@TableName(value = "user", autoResultMap = true)`
### 五、插件功能
1. **分页插件**
   **使用**:
   (1)添加配置类
   ```java
   @Configuration
   public class MybatisPlusConfig { 
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() { 
      // 初始化核心插件
      MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
      // 添加分页插件
      PaginationInnerInterceptor paginationInnerInterceptor = new PaginationInnerInterceptor();
      paginationInnerInterceptor.setMaxLimit(500L); // 设置最大分页数
      interceptor.addInnerInterceptor(paginationInnerInterceptor);
      return interceptor;
    }
   }
   ```
   (2)分页条件
   ```java
   void TestPageQuery() { 
    int pageNum = 1; // 当前页
    int pageSize = 10; // 每页数量
    // 1.构建条件
    // 分页条件
    Page<User> page = new Page<>(pageNum, pageSize);
    // 排序条件
    page.addOrder(new OrderItem("id", true));
    // 2.查询
    Page<User> userPage = userService.page(page);

    // 3.解析
    long total = userPage.getTotal();
    system.out.println("总记录数：" + total);
    long pages = userPage.getPages();
    system.out.println("总页数：" + pages);
    List<User> records = userPage.getRecords();
    records.forEach(System.out::println);
   }
   ```
