例如某些字段如邮箱，需要在存入时自动加密，取出时自动解密

定义一个AESTypeHandler
```java title:AESTypeHandler.java
import lombok.SneakyThrows;  
import lombok.extern.slf4j.Slf4j;  
import org.apache.ibatis.type.JdbcType;  
import org.apache.ibatis.type.MappedJdbcTypes;  
import org.apache.ibatis.type.TypeHandler;  
  
import javax.crypto.spec.SecretKeySpec;  
import java.security.Key;  
import java.sql.CallableStatement;  
import java.sql.PreparedStatement;  
import java.sql.ResultSet;  
import java.sql.SQLException;  
import java.util.Arrays;  
import java.util.HexFormat;  
  
@Slf4j  
@MappedJdbcTypes(JdbcType.VARCHAR)  
public class AESTypeHandler implements TypeHandler<String> {  //假如要加密的数据不是varchar，在此处泛型改为其他类型即可
    private static Key DEFAULT_KEY;  
  
    static {  
        try {  
            String hexKey = "1651B346A864685E54163C67F4";    //假如你有一个AES密钥的十六进制表示字符串  
            byte[] keyBytes = HexFormat.of().parseHex(hexKey);  
            
            //创建一个AES密钥对象  
            DEFAULT_KEY = new SecretKeySpec(keyBytes, "AES");  
            log.info("AES Key create from string: " + Arrays.toString(DEFAULT_KEY.getEncoded()));  
        } catch (Exception e) {  
            throw new RuntimeException(e);  
        }    
    }  
  
    //AES密钥  
    @SneakyThrows  
    public AESTypeHandler() {  
        this.DEFAULT_KEY = AESHelper.generateKey();  
    }  
  
    //实现加密和解密方法  
    @SneakyThrows  
    private String encrypt(String plainText, Key key) {  
        //使用AES或其他加密算法进行加密操作，这里用你自己的就行  
        return AESHelper.encrypt(plainText, key);  
    }  
  
    @SneakyThrows  
    private String decrypt(String cipherText, Key key) {  
        //使用AES或其他加密算法进行加密操作，这里用你自己的就行  
        return AESHelper.decrypt(cipherText, key);  
    }  
  
    @SneakyThrows  
    @Override    
    public void setParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {  
        /**  
         * 进行数据插入时，要做的事，MyBatis新增修改时自动调用  
         *  
         * @param parameter:原始数据  
         */  
        if (parameter != null) {  
            //在这里执行加密逻辑，并将密文写入预编译语句  
            String encrypted = encrypt(parameter, AESTypeHandler.DEFAULT_KEY);  
            ps.setString(i, encrypted);  
        } else {  
            ps.setNull(i, jdbcType.TYPE_CODE);  
        }    }  
  
    @SneakyThrows  
    @Override    
    public String getResult(ResultSet rs, String columnName) throws SQLException {  
        /**  
         * 按照列名提取数据时解密  
         */  
        log.info("decrypt:{}", columnName);  
        String encrypted = rs.getString(columnName);  
        return encrypted == null ? null : decrypt(encrypted, AESTypeHandler.DEFAULT_KEY);   //执行解密逻辑  
    }  
  
    @SneakyThrows  
    @Override    
    public String getResult(ResultSet rs, int columnIndex) throws SQLException {  
        /**  
         * 按照列索引提取数据时解密  
         */  
        String encrypted = rs.getString(columnIndex);  
        return encrypted == null ? null : decrypt(encrypted, AESTypeHandler.DEFAULT_KEY);   //执行解密逻辑  
    }  
  
    @SneakyThrows  
    @Override    
    public String getResult(CallableStatement cs, int columnIndex) throws SQLException {  
        /**  
         * 对CallableStatement按照列索引提取数据时解密  
         */  
        String encrypted = cs.getString(columnIndex);  
        return encrypted == null ? null : decrypt(encrypted, AESTypeHandler.DEFAULT_KEY);   //执行解密逻辑  
    }  
}
```

若是MyBatis-Plus
在实体类中添加以下两个注解
```java title: Customer.java
@Data  
@AllArgsConstructor  
@TableName(value = "ccc", autoResultMap = true) //必须加上autoResultMap = true  
public class Customer {  
    @TableId(type = IdType.AUTO)  
    int customerId;  
    int storeId;  
    String firstName;  
    String LastName;  
    @TableField(typeHandler = AESTypeHandler.class) //并且指明typeHandler为AESTypeHandler.class  
    String email;  
    int addressId;  
    boolean active;  
}
```

若是MyBatis或MyBatisPlus中没有定义的方法则需在`customer.xml`中定义resultMap
```xml
<resultMap id="cust" type="com.example.entity.Customer" autoMapping="true">
	<result typeHandeler="com.example.typehandler.AESTypeHandler" column="email" property="email"></result>
</resultMap>

<select ... resultMap="cust"> ... </select>
<insert ... parameterMap="cust"> ... </insert>
```