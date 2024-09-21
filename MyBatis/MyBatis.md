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
select * from user left join book on user.id = book.uid where user.id = #{id}
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

## 分页组件



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