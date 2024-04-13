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