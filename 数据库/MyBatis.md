# MyBatis
```xml title:pom.xml
    <!-- mysql驱动 -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>5.1.32</version>
    </dependency>
    <!-- mybatis -->
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.2.8</version>
    </dependency>
    <!-- 整合log4j -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-log4j12</artifactId>
        <version>1.6.4</version>
    </dependency>
```

配置文件版本的使用方法
使用mybaits-config.xml进行主文件配置，可以将每个业务Mapper单独出一个文件然后导入

主配置文件详细配置如下：
```xml title:mybaits-config.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration
    PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-config.dtd">

<!-- MyBatis的全局配置文件 -->
<configuration >


	<!-- 别名 -->
    <typeAliases>
        <package name="pojo"/>
    </typeAliases>

    <!-- 1.配置环境，可配置多个环境（比如：develop开发、test测试） -->
    <environments default="develop">
        <environment id="develop">
            <!-- 1.1.配置事务管理方式：JDBC/MANAGED
            JDBC：将事务交给JDBC管理（推荐）
            MANAGED：自己管理事务
              -->
            <transactionManager type="JDBC"></transactionManager>
            <!-- 1.2.配置数据源，即连接池 JNDI/POOLED/UNPOOLED
                JNDI：已过时
                POOLED：使用连接池（推荐）
                UNPOOLED：不使用连接池
             -->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/yonghedb?characterEncoding=utf-8"/>
                <property name="username" value="root"/>
                <property name="password" value="root"/>
            </dataSource>
        </environment>
    </environments>
    
    <!-- 2.导入Mapper配置文件，如果mapper文件有多个，可以通过多个mapper标签导入 -->
    <mappers>
        <mapper resource="xxx.xml"/>
    </mappers>
</configuration>
```

```xml title:xxxMapper.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
        
<mapper namespace="pojo">
	<!-- 
	由于上面配置了 <typeAliases> 别名，所以在这里的 resultType 可以直接写 Student，而不用写类的全限定名 pojo.Student
	namespace 属性其实就是对 SQL 进行分类管理，实现不同业务的 SQL 隔离
	parameterType：要求输入参数的类型
	resultType：输出的类型
	 -->
	<select id="listStudent" resultType="Student">
        select * from  student
    </select>
    
	<!-- 模糊查询 -->
	<select id="findStudentByName" parameterMap="java.lang.String" resultType="Student">
	    SELECT * FROM student WHERE name LIKE '%${value}%' 
	</select>
	
	
    <insert id="addStudent" parameterType="Student">
        insert into student (id, studentID, name) values (#{id},#{studentID},#{name})
    </insert>

    <delete id="deleteStudent" parameterType="Student">
        delete from student where id = #{id}
    </delete>

    <select id="getStudent" parameterType="_int" resultType="Student">
        select * from student where id= #{id}
    </select>

    <update id="updateStudent" parameterType="Student">
        update student set name=#{name} where id=#{id}
    </update>

</mapper>
```
使用：
```java
public class TestMyBatis {

    public static void main(String[] args) throws IOException {
        // 根据 mybatis-config.xml 配置的信息得到 sqlSessionFactory
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        // 然后根据 sqlSessionFactory 得到 session
        SqlSession session = sqlSessionFactory.openSession();
        
        // 增加学生
        Student student1 = new Student();
        student1.setId(4);
        student1.setStudentID(4);
        student1.setName("新增加的学生");
        session.insert("addStudent", student1);
        
        // 删除学生
        Student student2 = new Student();
        student2.setId(1);
        session.delete("deleteStudent", student2);
		
        // 获取学生
        Student student3 = session.selectOne("getStudent", 2);
		
        // 修改学生
        student3.setName("修改的学生");
        session.update("updateStudent", student3);
		
        // 最后通过 session 的 selectList() 方法调用 sql 语句 listStudent
        List<Student> listStudent = session.selectList("listStudent");
        for (Student student : listStudent) {
            System.out.println("ID:" + student.getId() + ",NAME:" + student.getName());
        }
		
        // 提交修改
        session.commit();
        // 关闭 session
        session.close();
    }
}
```

## 复杂查询
### 一对一查询
假设现在有结构如下
```java
@Data 
public class UserDetail { 
	int id; 
	String description; 
	Date register; 
	String avatar; 
} 

@Data 
public class User { 
	int id; 
	String name; 
	int age; 
	UserDetail detail; 
}  
```

此时希望查询User时一并查出`UserDetail`，有两种方式
#### 方式一
首先定义好`sql`语句
```xml
<select id="selectUserById" resultMap="test"> 
select * 
from user 
left join user_detail 
on user.id = user_detail.id 
where user.id = #{id}
</select>
```

再定义好`resultMap`
```xml
<resultMap id="test" type="com.test.entity.User"> 
	<id property="id" column="id"/> 
	<result property="name" column="name"/> 
	<result property="age" column="age"/> 
	<association property="detail" column="id" javaType="com.test.entity.UserDetail"> 
		<id property="id" column="id"/> 
		<result property="description" column="description"/> 
		<result property="register" column="register"/> 
		<result property="avatar" column="avatar"/> 
	</association> 
</resultMap>
```
这时`association`标签的`column`和`javaType`可以不填，Mybatis会自动推断

#### 方式二
使用嵌套select语句进行查询，我们可以在查询`user`表的时候，同时查询`user_detail`表的对应信息，分别执行两个选择语句，最后再由Mybatis将其结果合并，效果和第一种方法是一样的
```xml
<select id="selectUserById" resultMap="test"> 
	select * from user where id = #{id} 
</select> 

<resultMap id="test" type="com.test.entity.User"> 
	<id property="id" column="id"/> 
	<result property="name" column="name"/> 
	<result property="age" column="age"/> 
	<association property="detail" column="id" select="selectUserDetailById" javaType="com.test.entity.UserDetail"/> 
</resultMap> 

<select id="selectUserDetailById" resultType="com.test.entity.UserDetail"> 
	select * from user_detail where id = #{id} 
</select>
```
但是这时`association`标签的`column`和`javaType`必填，否则会报错


### 一对多查询
例如有数据结构如下
```java
@Data 
public class Book { 
	int bid; 
	String title; 
} 

@Data 
public class User { 
	int id; 
	String name; 
	int age; 
	List<Book> books; //直接得到用户所属的所有书籍信息 
}
```
首先定义sql语句
```sql
select * 
from user 
left join book on user.id = book.uid 
where user.id = #{id}
```
此时的`resultMap`里面用的是`collection`
```xml
<resultMap id="test" type="com.test.entity.User"> 
	<id column="id" property="id"/> 
	<result column="name" property="name"/> 
	<result column="age" property="age"/> 
	<collection property="books" ofType="com.test.entity.Book"> 
		<id column="bid" property="bid"/> 
		<result column="title" property="title"/> 
	</collection> 
</resultMap>
```

如果用注解，应该这样写
```java
@Results({ 
	@Result(id = true, column = "id", property = "id"), 
	@Result(column = "id", property = "detail", one = @One(select = "selectDetailById")) 
}) 

@Select("select * from user where id = #{id};") 
User selectUserById(int id); 

@Select("select * from user_detail where id = #{id}") 
UserDetail selectDetailById(int id);
```

在配置`@Result`注解时，只需要将one或是many参数进行填写即可，它们分别代表一对一关联和一对多关联，使用`@One`和`@Many`注解来指定其他查询语句进行嵌套查询，就像是使用`association`和`collection`那样

### 多对一查询
比如每个用户现在都有一个小组，但是他们目前都是在同一个小组中，此时我们查询所有用户信息的时候，需要自动携带他们的小组
```java
@Data 
public class Group { 
	int id; 
	String name; 
} 
@Data 
public class User { 
	int id; 
	String name; 
	int age; 
	Group group; 
}
```
实际上这里跟我们之前的一对一非常类似，只需要让查询出来的每一个用户都左连接分组信息即可，这样Mybatis就可以通过`association`来自动处理了
```xml
<select id="selectAllUser" resultMap="test2"> 
	select *, groups.name as gname 
	from user 
	left join `groups` 
	on user.gid = groups.id 
</select> 
<resultMap id="test2" type="com.test.entity.User"> 
	<id column="id" property="id"/> 
	<result column="name" property="name"/> 
	<result column="age" property="age"/> 
	<association property="group"> 
		<id column="gid" property="id"/> 
		<result column="gname" property="name"/> 
	</association> 
</resultMap>
```
## 使用注解开发
例如上面的`Select`和`insert`语句可以写为
```java
public interface TestMapper { 
	@Select("select * from user") 
	List<User> selectAllUser(); 
	
	@Options(useGeneratedKeys = true, keyColumn = "id", keyProperty = "id")  //插入后立刻自动赋值自增id到对象中
	@Insert("insert into user (name, age) values (#{name}, #{age})") 
	int insertUser(User user);
}
```
这里`useGeneratedKeys`设置为`true`表示我们希望获取数据库生成的键，`keyProperty`设置为User类中的需要获取自增结果的属性名，`keyColumn`为数据库中自增的字段名称，但是一般情况下不需要手动设置，但是某些数据库（像 PostgreSQL）中，当主键列不是表中的第一列的时候，必须设置。

假如现在我们的实体类字段名称与数据库不同，此时该如何像之前一样配置`resultMap`呢？
```java
public class User { 
	int uid; 
	String username; 
	int age; 
}
```
在原来的`resultMap`中，应该设置为
```xml
<resultMap id="test" type="User"> 
	<id property="id" column="uid"/> 
	<result column="name" property="username"/> 
</resultMap>
```
可以使用`@Results`注解来实现这种操作，它的使用方式与`resultMap`几乎没什么区别
```java
@Results({ 
	@Result(id = true, column = "id", property = "uid"), 
	@Result(column = "name", property = "username") 
}) 
@Select("select * from user") 
List<User> selectAllUser();
```
也可以单独在XML中配置一个`resultMap`然后直接通过注解的形式引用
```java
@ResultMap("test")
@Select("select * from user") 
List<User> selectAllUser();
```

还可以使用注解进行动态SQL的配置
```xml
<select id="selectUserById" resultType="User"> 
	select * from user where id = #{id} 
	<if test="id > 3"> 
		and age > 18 
	</if> 
</select>
```
Mybatis针对于所有的SQL操作都提供了对应的Provider注解，用于配置动态SQL，我们需要先创建一个类编写我们的动态SQL操作
```java
public class TestSqlBuilder { 
	public static String buildGetUserById(int id) { 
		return new SQL(){{ //SQL类中提供了常见的SELECT、FORM、WHERE等操作 
			SELECT("*"); 
			FROM("user"); 
			WHERE("id = #{id}"); 
			if (id > 3) { 
				WHERE("age > 18"); 
			} 
		}}.toString(); 
	} 
}
```
构建完成后，接着我们就可以使用`@SelectProvider`来引用这边编写好的动态SQL操作
```java
@SelectProvider(type = TestSqlBuilder.class, method = "buildGetUserById") 
User selectUserById(int id);
```
如果遇到了多个参数的情况，我们同样需要使用`@Param`来指定参数名称，包括TestSqlBuilder中编写的方法也需要添加，否则必须保证形参列表与这边接口一致。虽然这样可以实现和之前差不多的效果，但是这实在是太过复杂了，我们还需要单独编写一个类来做这种事情，实际上我们也可以直接在`@Select`中编写一个XML配置动态SQL，Mybatis同样可以正常解析：
```java
@Select(""" 
		<script> 
			select * from user where id = #{id} 
			<if test="id > 3"> and age > 18 </if> 
		</script> 
		""") 
User selectUserById(int id);
```
这里只需要包括一个script标签我们就能像之前XML那样编写动态SQL了，只不过由于IDEA不支持这种语法的识别，可能会出现一些莫名其妙的红标，但是是可以正常运行的


# MyBaits Plus
[官网](https://baomidou.com/)

**依赖**
```xml
<dependency>  
    <groupId>com.baomidou</groupId>  
    <artifactId>mybatis-plus-boot-starter</artifactId>  
</dependency>
<dependency>  
    <groupId>com.mysql</groupId>  
    <artifactId>mysql-connector-j</artifactId>  
</dependency>
```

**配置数据源**
```yml
spring: 
	datasource: 
		url: jdbc:mysql://localhost:3306/test 
		username: root 
		password: 123456 
		driver-class-name: com.mysql.cj.jdbc.Driver
```

**常见配置**
```yaml
mybatis-plus:  
  type-aliases-package: com.example.demo.entity  
  mapper-locations: classpath*:/mapper/**/*.xml  
  configuration:  
    map-underscore-to-camel-case: true # 驼峰命名规则  
    cache-enabled: false # 关闭Mybatis二级缓存  
  global-config:  
    db-config:  
      id-type: auto # 主键类型  
      # id-type: ASSIGN_ID # 默认全局使用雪花算法  
      update-strategy: not_null # 只更新非空字段
```

**实体类**
```java
@Data 
@TableName("user")   //对应的表名 
public class User { 
	@TableId(type = IdType.AUTO) //对应的主键 
	int id; 
	@TableField("name") //对应的字段 
	String name; 
	@TableField("email") 
	String email; 
	@TableField("password") 
	String password; 
    
	@TableField(fill = FieldFill.INSERT)
	LocalDateTime createTime;
    
    @TableField(fill = FieldFill.INSERT)  // 初始自动填充
	@Version  //乐观锁注解
	Integer version;
}
```
其中`@TableField`有个属性`fill`，用于自动填充。`MetaObjectHandler`的写法如下：

```java
@Slf4j
@Component  //确保能被Spring容器管理
public class MyMetaObjectHandler implements MetaObjectHandler {
    /**
     * 插入数据时的填充规则
     *
     * @param metaObject Meta 对象
     */
    @Override
    public void insertFill(MetaObject metaObject) {
        log.info("开始插入数据...");
        LocalDateTime now = LocalDateTime.now();
        // 应该从上下文中获取当前用户名
        String creator = "xxx";
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, now);
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, now);
        this.strictInsertFill(metaObject, "creator", String.class, creator);
        this.strictInsertFill(metaObject, "updater", String.class, creator);
        this.strictInsertFill(metaObject, "deleted", Integer.class, 0);
        this.strictInsertFill(metaObject, "version", Long.class, 1L);
    }
    
    /**
     * 更新数据时的填充规则
     *
     * @param metaObject Meta 对象
     */
    @Override
    public void updateFill(MetaObject metaObject) {
        log.info("开始更新数据...");
        // 应该从上下文中获取当前用户名
        String creator = "xxx";
        this.strictUpdateFill(metaObject,"updateTime", LocalDateTime.class, LocalDateTime.now());
        this.strictUpdateFill(metaObject, "updater", String.class, creator);
    }
}

```



**Mapper**

```java
@Mapper  
public interface UserMapper extends BaseMapper<User> {  
  
//    @Select("select * from user where id=#{id}")  //这里不再需要写方法了，只要继承BaseMapper填入泛型参数即可
//    User findUserById(int id);  
}
```

**条件构造器用法**
```java
QueryWrapper<User> wrapper = Wrappers  
        .<User>query()  
        .gt("id",1)  
        .orderByDesc("id");  
  
System.out.println(mapper.selectList(wrapper));
```

**更进一步地，直接把Service也自动生成了**
```java title:"service / UserService.java"
@Service 
public interface UserService extends IService<User> { //除了继承模版，我们也可以把它当成普通Service添加自己需要的方法 }
```

```java title:"service / impl / UserServiceImpl.java"
@Service //需要继承ServiceImpl才能实现那些默认的CRUD方法 
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService { }
```

**批量插入**
批量插入建议使用IService的`saveBatch`，并且配合jdbc的`rewriteBatchedStatements=true`参数，这个参数能使批量插入的一堆insert语句转化成一条insert拼多个values的形式，速度最优

**逻辑删除**
在配置文件中配置逻辑删除的字段名和值
```yaml
mybatis-plus:  
  global-config:  
    db-config:  
      logic-delete-field: is_delete # 逻辑删除字段名  
      logic-delete-value: 1 # 逻辑删除值  
      logic-not-delete-value: 0 # 逻辑未删除值
```
这里配置是全局的，可以单独设置，逻辑删除的标记字段加上注解
```java
@TableLogic(value=0, delval=1)  //value是逻辑未删除值，delval是逻辑已删除值
```
逻辑删除会导致垃圾数据越来越多，导致查询效率降低，并且sql中都要对逻辑删除字段做判断。如果不能删除，更建议将数据迁移到其他表。

**枚举类型**
在枚举类型的存储字段加上`@EnumValue`注解，本来还需要在配置文件中指定`default-enum-type-handler: com.baomidou.mybatisplus.core.handlers.MybatisEnumTypeHandler`，但是在MP3.5.2版本后无需再配置

**JSON类型**
假如数据库中有json数据，希望映射到对象中去，首先在实体类的`@TableName(value="表名", autoResultMap=true)`设置`autoResultMap`，然后在需要映射成对象的字段上加上`@TableField(typeHandler=JacksonTypeHandler.class)`或者其他类型的`typeHandler`

## 分页组件
首先配置分页插件
```java
@Configuration
@EnableTransactionManagement //开启事务  
@MapperScan(basePackages = "com.**.dao")  
public class MybatisPlusConfig {  
    /**  
     * 注册插件  
     */  
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {  
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();  
        //添加分页插件  
        PaginationInnerInterceptor pageInterceptor = new PaginationInnerInterceptor();  
        //设置请求的页面大于最大页的操作，true调回首页，false继续请求，默认是false  
        pageInterceptor.setOverflow(false);  
        //单页分页的条数限制，默认500限制  
        pageInterceptor.setMaxLimit(500L);  
        //设置数据库类型  
        pageInterceptor.setDbType(DbType.DM);  
  
        interceptor.addInnerInterceptor(pageInterceptor);  
        //拒绝全表更新和删除操作  
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());  
        return interceptor;  
    }  
  
}
```

使用：
```java
// 分页参数
Page<User> page = Page.of(pageNo, pageSize)
// 排序参数,true为升序，false为降序
page.addOrder(new OrderItem("XX", false));
// 查询
Page<User> result = userService.page(page);

```
其实执行page方法之后，MP会把结果填回到page对象中，理论上是可以不用重新赋值给result的，不过习惯还是返回

**通用分页实体转MP的分页插件**

## 乐观锁插件

乐观锁是一种并发控制机制，用于确保在记录更新时，该记录未被其他事务修改。MybatisPlus提供了`OptimisticLockerInnerInterceptor`插件。

乐观锁的实现一般包含以下几个步骤：

1. 读取记录时，获取当前版本号
2. 在更新记录时，将这个版本号一同传递
3. 执行更新操作时，设置`version=newVersion`的条件为`version=oldVersion`
4. 如果版本号不匹配，则更新失败

配置：

```java
@Configuration
@MapperScan("org.example.springbootdemo.mapper")
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor(){
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());  // 乐观锁插件
        return interceptor;
    }
}

```



建立一个字段`optimisticLockVersion`，并加上`@Version`。

如果是自定义的更新语句，则必须传入整个实体类且含有乐观锁字段才能自动更新，又或者手动处理更新
```java
@Update("UPDATE user SET name = #{name}, version = version + 1 WHERE id = #{id} AND version = #{version}")
int updateById(User user);
```

**注意：**

- version字段仅支持`int`、`Integer`、`long`、`Long`、`Date`、`Timestamp`、`LocalDateTime`类型
- 对于整数类型，newVersion是oldVersion+1
- newVersion会自动写回到实体对象中
- 支持内置的 `updateById(entity)` 和 `update(entity, wrapper)`, `saveOrUpdate(entity)`, `insertOrUpdate(entity) (version >=3.5.7)` 方法
- 自定义方法更新时如果满足内置参数的参数条件方式也会执行乐观锁逻辑，例如自定义`myUpate(entity)` 这个和 `updateById(entity)` 是等价的，会提取参数进行乐观锁填充，但更新实现需要自行处理
- 在 `update(entity, wrapper)` 方法中，`wrapper` 不能复用



## 多表联查

依赖：

```xml
<dependency>
    <groupId>com.github.yulichang</groupId>
    <artifactId>mybatis-plus-join-boot-starter</artifactId>
    <version>${mybatis-plus-join.version}</version>
</dependency>
```

Mapper不再继承BaseMapper，改为继承MPJBaseMapper

使用：

```java
MPJLambdaWrapper<Student> wrapper = new MPJLambdaWrapper<>();
// 构建查询哪些数据，并解析到VO的哪个属性中
wrapper.selectAs(Student::getSname, StudentVO::getSname)
        .selectAs(Student::getSage, StudentVO::getSage)
        .selectAs(Student::getSsex, StudentVO::getSsex);
// 连表
wrapper.leftJoin(Student1.class, Student1::getSname, Student::getSname)  //副表表实体的class，两个表的on条件
        .leftJoin(Student2.class, Student2::getSname, Student::getSname);

//后续构建where条件
```




## 配置文件加密
==从Mybatis Plus 3.3.2开始才支持这个功能==
将配置文件中的内容替换为加密后的字符串，并且在前面加上`mpw:`
```yaml
spring:
    datasource:
        url: mpw:qRhvCwF4GOqjessEB3G+a5okP+uXXr96wcucn2Pev6Bf1oEMZ1gVpPPhdDmjQqoM
        password: mpw:Hzy5iliJbwDHhjLs1L0j6w==
        username: mpw:Xb+EgsyuYRXw7U7sBJjBpA==
```
**密钥加密**
使用 AES 算法生成随机密钥，并对敏感数据进行加密。

```java
// 生成16位随机AES密钥
String randomKey = AES.generateRandomKey();

// 使用随机密钥加密数据
String encryptedData = AES.encrypt(data, randomKey);

//例：
 
import com.baomidou.mybatisplus.core.toolkit.AES;
 
public class Test {
    public static void main(String[] args){
        String data = "as123";
        String randomKey = "d1104d7c3b616f0b";  //替换成自己的
        // 使用密钥加密数据
        String encryptedData = AES.encrypt(data, randomKey);
        System.out.println(encryptedData);
    }
}
```

**如何使用**
在启动应用程序时，通过命令行参数或环境变量传递密钥。

// Jar 启动参数示例（在IDEA中设置Program arguments，或在服务器上设置为启动环境变量）
`--mpw.key=d1104d7c3b616f0b`

dockerfile
`ENTRYPOINT ["java","-jar","app.jar","--mpw.key=d1104d7c3b616f0b"]`



## 集成Druid数据源

MP默认的数据源使用Hikari框架，这里更换成Druid

```xml
<!-- 阿里数据库连接池 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>${druid.version}</version>
</dependency>
```

常用配置项：

```yaml
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://x.x.x.x:3306/xxx?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
      username: xxx
      password: xxx
      # 如果initialSize数量比较多，开启异步初始化可以加快应用启动速度
      async-init: true
      # 初始化连接数
      initial-size: 5
      # 最大连接数
      max-active: 20
      # 最小空闲连接数
      min-idle: 5
      # 获取连接等待超时时间
      max-wait: 60000
      # 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
      time-between-eviction-runs-millis: 60000
      # 增强time-between-eviction-runs-millis的周期性连接检查，minIdle内的空闲连接，每次检查强制验证连接有效性
      keep-alive: true
      # 配置一个连接在池中最小生存的时间，单位是毫秒
      min-evictable-idle-time-millis: 300000
      # 配置一个连接在池中最大生存的时间，单位是毫秒
      max-evictable-idle-time-millis: 172800000
      # 检测连接是否有效的sql，要求设置testWhileIdle=true，否则无效
      test-while-idle: true
      validation-query: SELECT 1
      validation-query-timeout: 1
      # 是否开启连接的检测功能，在获取连接时检测连接是否有效
      test-on-borrow: false
      # 归还连接时检测连接是否有效
      test-on-return: false
      # 是否缓存preparedStatement，也就是PSCache，PSCache对支持游标的数据库性能提升巨大，比如说oracle，在mysql下建议关闭。
      pool-prepared-statements: false
      
      filter:
        stat:
          enabled: true
          db-type: mysql
          log-slow-sql: true
          slow-sql-millis: 3000
          connection-stack-trace-enable: true
        wall:
          enabled: true
          db-type: mysql
          config:
            # 允许插入
            insert-allow: true
            # 禁止删除
            delete-allow: false
            # 允许更新
            update-allow: true
            # 禁止删除表
            drop-table-allow: false
      stat-view-servlet:
        # http://localhost:8080/druid/index.html
        # 启用StatViewServlet
        enabled: true
        # 访问内置监控页面的路径，内置监控页面的首页是/druid/index.html
        url-pattern: /druid/*
        # 不允许清空统计数据，重新计算
        reset-enable: false
        # 配置访问监控页面的账号密码
        login-username: admin
        login-password: admin
        # 允许访问的地址，不配置或为空则允许所有地址访问
        allow: 127.0.0.1
        # 拒绝访问的地址，优先级高于allow
        # deny:
```



## 多数据源

依赖：

```xml
<!-- 数据库连接池-->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
    <version>${dynamic-datasource.version}</version>
</dependency>
```

配置

```yaml
spring:
  autoconfigure:
    # 忽略Druid的默认配置
    exclude: com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceAutoConfigure
  datasource:
    dynamic:
      primary: master
      # 是否启用严格模式，当所有的数据源都没有可用连接时会报错，阻止系统启动
      strict: false
      datasource:
        master:
          url: jdbc:mysql://x.x.x.x:3306/xxx?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
          username: xxx
          password: xxx
          driver-class-name: com.mysql.cj.jdbc.Driver
          type: com.alibaba.druid.pool.DruidDataSource
        slave:
          url: jdbc:mysql://x.x.x.x:3306/xxx?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
          username: xxx
          password: xxx
          driver-class-name: com.mysql.cj.jdbc.Driver
          type: com.alibaba.druid.pool.DruidDataSource
    druid:
      # 如果initialSize数量比较多，开启异步初始化可以加快应用启动速度
      async-init: true
      # 初始化连接数
      initial-size: 5
      # 最大连接数
      max-active: 20
      # 最小空闲连接数
      min-idle: 5
      # 获取连接等待超时时间
      max-wait: 60000
      # 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
      time-between-eviction-runs-millis: 60000
      # 增强time-between-eviction-runs-millis的周期性连接检查，minIdle内的空闲连接，每次检查强制验证连接有效性
      keep-alive: true
      # 配置一个连接在池中最小生存的时间，单位是毫秒
      min-evictable-idle-time-millis: 300000
      # 配置一个连接在池中最大生存的时间，单位是毫秒
      max-evictable-idle-time-millis: 172800000
      # 检测连接是否有效的sql，要求设置testWhileIdle=true，否则无效
      test-while-idle: true
      validation-query: SELECT 1
      validation-query-timeout: 1
      # 是否开启连接的检测功能，在获取连接时检测连接是否有效
      test-on-borrow: false
      # 归还连接时检测连接是否有效
      test-on-return: false
      # 是否缓存preparedStatement，也就是PSCache，PSCache对支持游标的数据库性能提升巨大，比如说oracle，在mysql下建议关闭。
      pool-prepared-statements: false
      
      filter:
        stat:
          enabled: true
          db-type: mysql
          log-slow-sql: true
          slow-sql-millis: 3000
          connection-stack-trace-enable: true
        wall:
          enabled: true
          db-type: mysql
          config:
            # 允许插入
            insert-allow: true
            # 禁止删除
            delete-allow: false
            # 允许更新
            update-allow: true
            # 禁止删除表
            drop-table-allow: false
      stat-view-servlet:
        # http://localhost:8080/druid/index.html
        # 启用StatViewServlet
        enabled: true
        # 访问内置监控页面的路径，内置监控页面的首页是/druid/index.html
        url-pattern: /druid/*
        # 不允许清空统计数据，重新计算
        reset-enable: false
        # 配置访问监控页面的账号密码
        login-username: admin
        login-password: admin
        # 允许访问的地址，不配置或为空则允许所有地址访问
        allow: 127.0.0.1
        # 拒绝访问的地址，优先级高于allow
        # deny:
```

然后在对应的Service里面打注解`@DS("数据源名称")`，但是这样很容易写错，可以自行封装一个。

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@DS("master")
public @interface Master { }
```

这样会更方便些，但是baomidou也有一个这样的实现，但是他固定了名字，如果还是用master、slave可以用他的注解，名字不一样就自己封装一个


## 自定义字符转义拦截器
```java
import com.baomidou.mybatisplus.core.conditions.AbstractWrapper;
import com.baomidou.mybatisplus.extension.plugins.inner.InnerInterceptor;
import com.vcmy.infrastructure.common.annotation.LikeEscape;
import lombok.NoArgsConstructor;
import lombok.SneakyThrows;
import lombok.extern.slf4j.Slf4j;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.binding.MapperMethod;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;
import org.springframework.util.StringUtils;

import javax.annotation.PostConstruct;
import java.lang.reflect.*;
import java.util.*;

/**
 * 支持字符串 LIKE 转义的 MyBatis Plus 拦截器
 */
@Slf4j
@NoArgsConstructor
public class CharEscapeInnerInterceptor implements InnerInterceptor {

    private CharEscape charEscape = new CharEscapeImpl();

    public CharEscapeInnerInterceptor(CharEscape charEscape) {
        this.charEscape = charEscape;
    }

    @PostConstruct
    public void init() {
        log.info("CharEscapeInnerInterceptor initialized");
    }

    @Override
    @SneakyThrows
    @SuppressWarnings("unchecked")
    public void beforeQuery(Executor executor, MappedStatement ms, Object parameter,
                            RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
        if (parameter == null) return;

        if (parameter instanceof MapperMethod.ParamMap) {
            MapperMethod.ParamMap<Object> map = (MapperMethod.ParamMap<Object>) parameter;
            for (Map.Entry<String, Object> entry : map.entrySet()) {
                String paramName = entry.getKey();
                Object value = entry.getValue();

                // 方法参数注解优先级高
                LikeEscape likeEscape = getLikeEscapeAnnotation(ms, paramName);
                if (likeEscape != null && value instanceof String) {
                    String escaped = wrapWithLikeMode(charEscape.charEscape((String) value), likeEscape.mode());
                    map.put(paramName, escaped);
                    continue;
                }

                if (value instanceof AbstractWrapper) {
                    AbstractWrapper<?, ?, ?> wrapper = (AbstractWrapper<?, ?, ?>) value;
                    getObjCharEscape(wrapper.getParamNameValuePairs());
                    getObjCharEscape(wrapper.getEntity());
                }

                getObjCharEscape(value); // DTO 字段处理
            }
        } else {
            getObjCharEscape(parameter);
        }
    }

    /**
     * 自动处理对象的字段（递归）
     */
    @SuppressWarnings("unchecked")
    public Object getObjCharEscape(Object object) throws Exception {
        if (object == null || object instanceof Long) return object;

        if (object instanceof String) return object;

        if (object instanceof Map) {
            Map<Object, Object> map = (Map<Object, Object>) object;
            for (Map.Entry<Object, Object> entry : map.entrySet()) {
                map.put(entry.getKey(), getObjCharEscape(entry.getValue()));
            }
            return map;
        }

        if (object instanceof Collection) {
            for (Object item : (Collection<?>) object) {
                getObjCharEscape(item);
            }
            return object;
        }

        for (Field field : object.getClass().getDeclaredFields()) {
            boolean accessible = field.isAccessible();
            field.setAccessible(true);

            if (Modifier.isFinal(field.getModifiers()) || field.get(object) == null) {
                continue;
            }

            Object fieldValue = field.get(object);

            if (field.isAnnotationPresent(LikeEscape.class) && fieldValue instanceof String) {
                LikeEscape annotation = field.getAnnotation(LikeEscape.class);
                String escaped = wrapWithLikeMode(charEscape.charEscape((String) fieldValue), annotation.mode());
                field.set(object, escaped);
            } else {
                getObjCharEscape(fieldValue); // 递归子对象
            }

            field.setAccessible(accessible);
        }

        return object;
    }

    /**
     * 获取 Mapper 方法参数上的注解
     */
    private LikeEscape getLikeEscapeAnnotation(MappedStatement ms, String paramName) {
        try {
            String mapperClassName = ms.getId().substring(0, ms.getId().lastIndexOf('.'));
            String methodName = ms.getId().substring(ms.getId().lastIndexOf('.') + 1);
            Class<?> mapperClass = Class.forName(mapperClassName);

            for (Method method : mapperClass.getMethods()) {
                if (!method.getName().equals(methodName)) continue;

                Parameter[] parameters = method.getParameters();
                for (Parameter parameter : parameters) {
                    if (!parameter.isAnnotationPresent(LikeEscape.class)) continue;

                    if (parameter.isAnnotationPresent(Param.class)) {
                        String realName = parameter.getAnnotation(Param.class).value();
                        if (realName.equals(paramName)) {
                            return parameter.getAnnotation(LikeEscape.class);
                        }
                    }
                }
            }
        } catch (Exception e) {
            log.warn("getLikeEscapeAnnotation failed: {}", e.getMessage());
        }
        return null;
    }

    /**
     * 根据模糊模式拼接 %
     */
    private String wrapWithLikeMode(String value, LikeEscape.Mode mode) {
        switch (mode) {
            case ALL: return "%" + value + "%";
            case LEFT: return "%" + value;
            case RIGHT: return value + "%";
            default: return value;
        }
    }

    /**
     * 字符转义器接口
     */
    public interface CharEscape {
        String charEscape(String charString);
    }

    /**
     * 默认实现：LIKE 参数中的 \、% 和 _ 都会转义
     */
    private static class CharEscapeImpl implements CharEscape {
        @Override
        public String charEscape(String charString) {
            if (!StringUtils.hasLength(charString)) return charString;
            charString = charString.trim();
            return charString
                    .replace("\\", "\\\\")
                    .replace("%", "\\%")
                    .replace("_", "\\_");
        }
    }
}
```

配套的注解
```java
import java.lang.annotation.*;

@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface LikeEscape {

    /** 模糊匹配类型：前缀、后缀、全模糊 */
    Mode mode() default Mode.ALL;

    enum Mode {
        /** %xxx% */
        ALL,

        /** xxx% */
        RIGHT,

        /** %xxx */
        LEFT,

        /** 不拼接 %，仅转义 */
        NONE
    }
}
```

添加这个拦截器
```java
@Configuration  
@EnableTransactionManagement  
public class MybatisPlusConfig {  
  
    @Bean  
    public MybatisPlusInterceptor mybatisPlusInterceptor() {  
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();  
        // 根据实际使用的数据库类型选择  
        PaginationInnerInterceptor paginationInnerInterceptor = new PaginationInnerInterceptor(DbType.DM);  
        paginationInnerInterceptor.setMaxLimit(1000L);  
        interceptor.addInnerInterceptor(paginationInnerInterceptor);  
        // 乐观锁版本拦截，使用时传参需要加上版本号  
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());  
        // 字符转义拦截器  
        interceptor.addInnerInterceptor(new CharEscapeInnerInterceptor());  
        return interceptor;  
    }  
}
```

## **代码生成器**
直接从Mapper到Controller直接全部生成好

**依赖**
```xml
<dependency> 
	<groupId>com.baomidou</groupId> 
	<artifactId>mybatis-plus-boot-starter</artifactId> 
	<version>3.5.3.1</version> 
</dependency> 
<dependency> 
	<groupId>com.baomidou</groupId> 
	<artifactId>mybatis-plus-generator</artifactId> 
	<version>3.5.3.1</version> 
</dependency> 
<dependency> 
	<groupId>org.apache.velocity</groupId> 
	<artifactId>velocity-engine-core</artifactId> 
	<version>2.3</version> 
</dependency>
```

配置一下数据源，然后在测试类里用FastAutoGenerator作为生成器
```java title:test.java
@Test 
void contextLoads() { 
	FastAutoGenerator .create(
		new DataSourceConfig
		.Builder(dataSource)
		) 
		.globalConfig(builder -> { 
			builder.author("lbw"); //作者信息，一会会变成注释 
			builder.commentDate("2024-01-01"); //日期信息，一会会变成注释 
			builder.outputDir("src/main/java"); //输出目录设置为当前项目的目录 
			}
		) 
		.execute(); 
		}
```