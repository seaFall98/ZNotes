[[MyBatis 加密插件（教学版）]]

下面给一套**偏生产可用**的 MyBatis 字段加密插件。

特点：

- 基于 `@EncryptedField` 标记敏感字段；
    
- 使用 **AES-GCM**，不是教学代码里的 AES-CBC 固定 IV；
    
- 每次加密使用随机 IV；
    
- 密文带版本前缀，避免重复加密；
    
- insert / update / select 参数加密；
    
- query 结果自动解密；
    
- 写入前临时加密，SQL 执行后恢复原对象，避免业务对象被污染；
    
- 支持普通对象、`List`、数组、`Map`、批量参数；
    
- Spring Boot 自动注册。
    

---

# 1. 字段注解：`@EncryptedField`

```java
package com.example.mybatis.crypto;

import java.lang.annotation.*;

/**
 * 标记需要加密存储的字段。
 *
 * 适用场景：
 * - 手机号
 * - 身份证号
 * - 邮箱
 * - 地址
 * - 银行卡号
 *
 * 注意：
 * - 字段类型建议使用 String。
 * - 密码不要用可逆加密，应该用 BCrypt / Argon2 这类哈希算法。
 */
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface EncryptedField {
}
```

---

# 2. 配置类：`MybatisCryptoProperties`

```java
package com.example.mybatis.crypto;

import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * 加密配置。
 *
 * 建议：
 * - key 不要写死在代码里；
 * - 生产环境从 KMS、Vault、环境变量、配置中心读取；
 * - AES-256 key 长度为 32 字节。
 */
@ConfigurationProperties(prefix = "app.mybatis.crypto")
public class MybatisCryptoProperties {

    /**
     * 是否启用插件。
     */
    private boolean enabled = true;

    /**
     * AES-GCM 密钥。
     *
     * 要求：
     * - AES-128：16 字节
     * - AES-192：24 字节
     * - AES-256：32 字节
     *
     * 示例：0123456789abcdef0123456789abcdef
     */
    private String key;

    /**
     * 密文版本前缀。
     *
     * 后续密钥轮换、算法升级时，可以通过版本区分。
     */
    private String versionPrefix = "ENC:v1:";

    public boolean isEnabled() {
        return enabled;
    }

    public void setEnabled(boolean enabled) {
        this.enabled = enabled;
    }

    public String getKey() {
        return key;
    }

    public void setKey(String key) {
        this.key = key;
    }


    public String getVersionPrefix() {
        return versionPrefix;
    }

    public void setVersionPrefix(String versionPrefix) {
        this.versionPrefix = versionPrefix;
    }
}
```

---

# 3. AES-GCM 加解密器：`AesGcmStringEncryptor`

```java
package com.example.mybatis.crypto;

import javax.crypto.Cipher;
import javax.crypto.spec.GCMParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;
import java.security.SecureRandom;
import java.util.Base64;

/**
 * AES-GCM 字符串加解密器。
 *
 * 密文格式：
 *
 * ENC:v1:Base64(IV + CIPHER_TEXT)
 *
 * 其中：
 * - IV：12 字节随机数；
 * - CIPHER_TEXT：AES-GCM 加密后的密文，内部包含认证标签。
 */
public class AesGcmStringEncryptor {

    private static final String ALGORITHM = "AES";
    private static final String TRANSFORMATION = "AES/GCM/NoPadding";

    /**
     * GCM 推荐 12 字节 IV。
     */
    private static final int IV_LENGTH_BYTES = 12;

    /**
     * 认证标签长度，128 bit 是常用选择。
     */
    private static final int TAG_LENGTH_BITS = 128;

    private final SecretKeySpec secretKeySpec;
    private final String versionPrefix;
    private final SecureRandom secureRandom = new SecureRandom();

    public AesGcmStringEncryptor(MybatisCryptoProperties properties) {
        if (properties.getKey() == null || properties.getKey().isBlank()) {
            throw new IllegalArgumentException("app.mybatis.crypto.key must not be blank");
        }

        byte[] keyBytes = properties.getKey().getBytes(StandardCharsets.UTF_8);
        int keyLength = keyBytes.length;

        if (keyLength != 16 && keyLength != 24 && keyLength != 32) {
            throw new IllegalArgumentException(
                    "AES key length must be 16, 24, or 32 bytes, actual: " + keyLength
            );
        }

        this.secretKeySpec = new SecretKeySpec(keyBytes, ALGORITHM);
        this.versionPrefix = properties.getVersionPrefix();
    }

    /**
     * 判断一个值是否已经是本插件生成的密文。
     */
    public boolean isEncrypted(String value) {
        return value != null && value.startsWith(versionPrefix);
    }

    /**
     * 加密明文。
     *
     * 注意：
     * - 空字符串是否加密取决于业务，这里选择保持原样；
     * - 已经加密过的内容直接返回，避免重复加密。
     */
    public String encrypt(String plainText) {
        if (plainText == null || plainText.isEmpty()) {
            return plainText;
        }

        if (isEncrypted(plainText)) {
            return plainText;
        }

        try {
            byte[] iv = new byte[IV_LENGTH_BYTES];
            secureRandom.nextBytes(iv);

            Cipher cipher = Cipher.getInstance(TRANSFORMATION);
            GCMParameterSpec parameterSpec = new GCMParameterSpec(TAG_LENGTH_BITS, iv);
            cipher.init(Cipher.ENCRYPT_MODE, secretKeySpec, parameterSpec);

            byte[] cipherText = cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));

            ByteBuffer byteBuffer = ByteBuffer.allocate(iv.length + cipherText.length);
            byteBuffer.put(iv);
            byteBuffer.put(cipherText);

            String payload = Base64.getEncoder().encodeToString(byteBuffer.array());
            return versionPrefix + payload;
        } catch (Exception e) {
            throw new IllegalStateException("Encrypt field failed", e);
        }
    }

    /**
     * 解密密文。
     *
     * 注意：
     * - 非本插件格式的值直接返回；
     * - 这样可以兼容历史明文数据、空值、非加密字段。
     */
    public String decrypt(String encryptedText) {
        if (encryptedText == null || encryptedText.isEmpty()) {
            return encryptedText;
        }

        if (!isEncrypted(encryptedText)) {
            return encryptedText;
        }

        try {
            String payload = encryptedText.substring(versionPrefix.length());
            byte[] allBytes = Base64.getDecoder().decode(payload);

            ByteBuffer byteBuffer = ByteBuffer.wrap(allBytes);

            byte[] iv = new byte[IV_LENGTH_BYTES];
            byteBuffer.get(iv);

            byte[] cipherText = new byte[byteBuffer.remaining()];
            byteBuffer.get(cipherText);

            Cipher cipher = Cipher.getInstance(TRANSFORMATION);
            GCMParameterSpec parameterSpec = new GCMParameterSpec(TAG_LENGTH_BITS, iv);
            cipher.init(Cipher.DECRYPT_MODE, secretKeySpec, parameterSpec);

            byte[] plainBytes = cipher.doFinal(cipherText);
            return new String(plainBytes, StandardCharsets.UTF_8);
        } catch (Exception e) {
            throw new IllegalStateException("Decrypt field failed", e);
        }
    }
}
```

---

# 4. MyBatis 插件核心：`MybatisFieldCryptoInterceptor`

```java
package com.example.mybatis.crypto;

import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.SqlCommandType;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;

import java.lang.reflect.Array;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * MyBatis 字段加解密插件。
 *
 * 拦截点：
 * - Executor.update：insert / update / delete
 * - Executor.query：select
 *
 * 行为：
 * - INSERT / UPDATE / SELECT 参数：执行 SQL 前加密；
 * - SELECT 结果：执行 SQL 后解密；
 * - 写操作执行完成后，会恢复入参对象的原始明文，避免业务侧对象被污染。
 *
 * 注意：
 * 1. 随机 IV 的 AES-GCM 不能直接支持等值查询。
 *    例如 where phone = ?，如果 phone 每次加密结果都不同，数据库无法匹配。
 *
 * 2. 如果确实需要按手机号、身份证号等字段查询，建议额外维护 blind index 字段：
 *
 *    phone_encrypt     存 AES-GCM 密文
 *    phone_hash        存 HMAC-SHA256(phone)
 *
 *    查询时使用 phone_hash 做等值匹配，phone_encrypt 用于回显解密。
 */
@Intercepts({
        @Signature(
                type = Executor.class,
                method = "update",
                args = {MappedStatement.class, Object.class}
        ),
        @Signature(
                type = Executor.class,
                method = "query",
                args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
        ),
        @Signature(
                type = Executor.class,
                method = "query",
                args = {
                        MappedStatement.class,
                        Object.class,
                        RowBounds.class,
                        ResultHandler.class,
                        org.apache.ibatis.cache.CacheKey.class,
                        org.apache.ibatis.mapping.BoundSql.class
                }
        )
})
public class MybatisFieldCryptoInterceptor implements Interceptor {

    private final AesGcmStringEncryptor encryptor;

    /**
     * 字段缓存，避免每次反射扫描。
     */
    private final Map<Class<?>, List<Field>> encryptedFieldCache = new ConcurrentHashMap<>();

    public MybatisFieldCryptoInterceptor(AesGcmStringEncryptor encryptor) {
        this.encryptor = encryptor;
    }

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();

        MappedStatement mappedStatement = (MappedStatement) args[0];
        Object parameter = args.length > 1 ? args[1] : null;

        SqlCommandType commandType = mappedStatement.getSqlCommandType();

        if (commandType == SqlCommandType.INSERT || commandType == SqlCommandType.UPDATE) {
            return handleWrite(invocation, parameter);
        }

        if (commandType == SqlCommandType.SELECT) {
            return handleSelect(invocation, parameter);
        }

        return invocation.proceed();
    }

    /**
     * 处理 INSERT / UPDATE：
     * 1. SQL 执行前加密参数；
     * 2. 执行 SQL；
     * 3. finally 中恢复原始明文参数。
     */
    private Object handleWrite(Invocation invocation, Object parameter) throws Throwable {
        List<FieldSnapshot> snapshots = new ArrayList<>();

        try {
            encryptObjectGraph(parameter, snapshots, newVisitedMap());
            return invocation.proceed();
        } finally {
            restoreSnapshots(snapshots);
        }
    }

    /**
     * 处理 SELECT：
     * 1. SQL 执行前加密查询参数；
     * 2. 执行 SQL；
     * 3. 恢复查询参数；
     * 4. 解密查询结果。
     */
    private Object handleSelect(Invocation invocation, Object parameter) throws Throwable {
        List<FieldSnapshot> snapshots = new ArrayList<>();

        Object result;
        try {
            encryptObjectGraph(parameter, snapshots, newVisitedMap());
            result = invocation.proceed();
        } finally {
            restoreSnapshots(snapshots);
        }

        decryptObjectGraph(result, newVisitedMap());
        return result;
    }

    /**
     * 加密对象图。
     *
     * 支持：
     * - 普通 Java Bean
     * - Collection
     * - Array
     * - Map
     *
     * 使用 visited 防止循环引用和重复处理。
     */
    private void encryptObjectGraph(
            Object object,
            List<FieldSnapshot> snapshots,
            IdentityHashMap<Object, Boolean> visited
    ) {
        if (object == null || isSimpleValueType(object.getClass())) {
            return;
        }

        if (visited.containsKey(object)) {
            return;
        }
        visited.put(object, Boolean.TRUE);

        if (object instanceof Collection<?> collection) {
            for (Object item : collection) {
                encryptObjectGraph(item, snapshots, visited);
            }
            return;
        }

        if (object.getClass().isArray()) {
            int length = Array.getLength(object);
            for (int i = 0; i < length; i++) {
                encryptObjectGraph(Array.get(object, i), snapshots, visited);
            }
            return;
        }

        if (object instanceof Map<?, ?> map) {
            for (Object value : map.values()) {
                encryptObjectGraph(value, snapshots, visited);
            }
            return;
        }

        List<Field> fields = getEncryptedFields(object.getClass());

        for (Field field : fields) {
            try {
                Object rawValue = field.get(object);

                if (rawValue == null) {
                    continue;
                }

                if (!(rawValue instanceof String plainText)) {
                    throw new IllegalStateException(
                            "@EncryptedField only supports String field, class="
                                    + object.getClass().getName()
                                    + ", field="
                                    + field.getName()
                    );
                }

                String encryptedValue = encryptor.encrypt(plainText);

                if (!Objects.equals(plainText, encryptedValue)) {
                    snapshots.add(new FieldSnapshot(object, field, plainText));
                    field.set(object, encryptedValue);
                }
            } catch (IllegalAccessException e) {
                throw new IllegalStateException(
                        "Encrypt field access failed, class="
                                + object.getClass().getName()
                                + ", field="
                                + field.getName(),
                        e
                );
            }
        }
    }

    /**
     * 解密对象图。
     *
     * SELECT 结果一般是 List，但这里也兼容单对象、Map、数组等结构。
     */
    private void decryptObjectGraph(Object object, IdentityHashMap<Object, Boolean> visited) {
        if (object == null || isSimpleValueType(object.getClass())) {
            return;
        }

        if (visited.containsKey(object)) {
            return;
        }
        visited.put(object, Boolean.TRUE);

        if (object instanceof Collection<?> collection) {
            for (Object item : collection) {
                decryptObjectGraph(item, visited);
            }
            return;
        }

        if (object.getClass().isArray()) {
            int length = Array.getLength(object);
            for (int i = 0; i < length; i++) {
                decryptObjectGraph(Array.get(object, i), visited);
            }
            return;
        }

        if (object instanceof Map<?, ?> map) {
            for (Object value : map.values()) {
                decryptObjectGraph(value, visited);
            }
            return;
        }

        List<Field> fields = getEncryptedFields(object.getClass());

        for (Field field : fields) {
            try {
                Object rawValue = field.get(object);

                if (rawValue == null) {
                    continue;
                }

                if (!(rawValue instanceof String encryptedText)) {
                    throw new IllegalStateException(
                            "@EncryptedField only supports String field, class="
                                    + object.getClass().getName()
                                    + ", field="
                                    + field.getName()
                    );
                }

                String plainText = encryptor.decrypt(encryptedText);

                if (!Objects.equals(encryptedText, plainText)) {
                    field.set(object, plainText);
                }
            } catch (IllegalAccessException e) {
                throw new IllegalStateException(
                        "Decrypt field access failed, class="
                                + object.getClass().getName()
                                + ", field="
                                + field.getName(),
                        e
                );
            }
        }
    }

    /**
     * 获取当前类及父类中所有带 @EncryptedField 的字段。
     */
    private List<Field> getEncryptedFields(Class<?> clazz) {
        return encryptedFieldCache.computeIfAbsent(clazz, this::scanEncryptedFields);
    }

    private List<Field> scanEncryptedFields(Class<?> clazz) {
        List<Field> result = new ArrayList<>();

        Class<?> current = clazz;
        while (current != null && current != Object.class) {
            for (Field field : current.getDeclaredFields()) {
                if (!field.isAnnotationPresent(EncryptedField.class)) {
                    continue;
                }

                if (Modifier.isStatic(field.getModifiers())) {
                    continue;
                }

                field.setAccessible(true);
                result.add(field);
            }

            current = current.getSuperclass();
        }

        return Collections.unmodifiableList(result);
    }

    /**
     * 恢复 SQL 执行前的原始字段值。
     *
     * 这样业务代码里传入的对象不会在 mapper 调用后变成密文。
     */
    private void restoreSnapshots(List<FieldSnapshot> snapshots) {
        for (int i = snapshots.size() - 1; i >= 0; i--) {
            FieldSnapshot snapshot = snapshots.get(i);
            try {
                snapshot.field().set(snapshot.target(), snapshot.originalValue());
            } catch (IllegalAccessException e) {
                throw new IllegalStateException(
                        "Restore encrypted field failed, field=" + snapshot.field().getName(),
                        e
                );
            }
        }
    }

    /**
     * IdentityHashMap 使用对象地址判断重复，适合处理对象图遍历。
     */
    private IdentityHashMap<Object, Boolean> newVisitedMap() {
        return new IdentityHashMap<>();
    }

    /**
     * 简单类型无需递归处理。
     */
    private boolean isSimpleValueType(Class<?> clazz) {
        return clazz.isPrimitive()
                || CharSequence.class.isAssignableFrom(clazz)
                || Number.class.isAssignableFrom(clazz)
                || Boolean.class == clazz
                || Character.class == clazz
                || Date.class.isAssignableFrom(clazz)
                || java.time.temporal.Temporal.class.isAssignableFrom(clazz)
                || UUID.class == clazz
                || clazz.isEnum()
                || clazz.getName().startsWith("java.")
                || clazz.getName().startsWith("javax.")
                || clazz.getName().startsWith("jakarta.");
    }

    /**
     * MyBatis 插件包装目标对象。
     */
    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    /**
     * 如果需要支持 XML 配置参数，可以在这里接收 properties。
     * Spring Boot 场景一般不需要。
     */
    @Override
    public void setProperties(Properties properties) {
        // no-op
    }

    /**
     * 记录字段原始值，用于 SQL 执行后恢复。
     */
    private record FieldSnapshot(Object target, Field field, Object originalValue) {
    }
}
```

---

# 5. Spring Boot 自动配置：`MybatisCryptoAutoConfiguration`

```java
package com.example.mybatis.crypto;

import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.boot.autoconfigure.ConfigurationCustomizer;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

/**
 * MyBatis 字段加密插件自动配置。
 *
 * 如果项目里使用 mybatis-spring-boot-starter，
 * 推荐通过 ConfigurationCustomizer 注册拦截器。
 */
@Configuration
@EnableConfigurationProperties(MybatisCryptoProperties.class)
@ConditionalOnProperty(
        prefix = "app.mybatis.crypto",
        name = "enabled",
        havingValue = "true",
        matchIfMissing = true
)
public class MybatisCryptoAutoConfiguration {

    @Bean
    public AesGcmStringEncryptor aesGcmStringEncryptor(MybatisCryptoProperties properties) {
        return new AesGcmStringEncryptor(properties);
    }

    @Bean
    public MybatisFieldCryptoInterceptor mybatisFieldCryptoInterceptor(
            AesGcmStringEncryptor encryptor
    ) {
        return new MybatisFieldCryptoInterceptor(encryptor);
    }

    /**
     * 注册到 MyBatis Configuration。
     *
     * 这个写法比手动操作 SqlSessionFactory 更干净。
     */
    @Bean
    public ConfigurationCustomizer mybatisCryptoConfigurationCustomizer(
            MybatisFieldCryptoInterceptor interceptor
    ) {
        return configuration -> configuration.addInterceptor(interceptor);
    }

    /**
     * 可选兜底：
     * 某些项目没有走 ConfigurationCustomizer，可以用这个方式强行注册。
     *
     * 一般情况下二选一即可。
     * 如果你用了上面的 ConfigurationCustomizer，这段可以不要。
     */
    // @Bean
    public Object mybatisCryptoInterceptorRegister(
            List<SqlSessionFactory> sqlSessionFactories,
            MybatisFieldCryptoInterceptor interceptor
    ) {
        for (SqlSessionFactory sqlSessionFactory : sqlSessionFactories) {
            sqlSessionFactory.getConfiguration().addInterceptor(interceptor);
        }
        return new Object();
    }
}
```

---

# 6. `application.yml` 配置

```yaml
app:
  mybatis:
    crypto:
      enabled: true
      # 生产环境不要明文写在配置文件里。
      # 推荐用环境变量、KMS、Vault、配置中心注入。
      key: ${APP_MYBATIS_CRYPTO_KEY}
      version-prefix: "ENC:v1:"
```

环境变量示例：

```bash
APP_MYBATIS_CRYPTO_KEY=0123456789abcdef0123456789abcdef
```

注意：这里是 **32 字节**，对应 AES-256。

---

# 7. PO 对象使用方式

```java
package com.example.employee.infrastructure.po;

import com.example.mybatis.crypto.EncryptedField;
import lombok.Data;

import java.time.LocalDateTime;

@Data
public class EmployeePO {

    private Long id;

    private String employeeNumber;

    /**
     * 入库时自动加密。
     * 出库时自动解密。
     */
    @EncryptedField
    private String employeeName;

    @EncryptedField
    private String phone;

    @EncryptedField
    private String idCardNo;

    private String employeeLevel;

    private String employeeTitle;

    private LocalDateTime createTime;

    private LocalDateTime updateTime;
}
```

---

# 8. Mapper 正常写，不需要手动加解密

```java
package com.example.employee.infrastructure.mapper;

import com.example.employee.infrastructure.po.EmployeePO;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

import java.util.List;

@Mapper
public interface EmployeeMapper {

    int insert(EmployeePO employeePO);

    int update(EmployeePO employeePO);

    EmployeePO selectById(@Param("id") Long id);

    List<EmployeePO> selectByEmployeeNumber(@Param("employeeNumber") String employeeNumber);
}
```

XML 正常写：

```xml
<mapper namespace="com.example.employee.infrastructure.mapper.EmployeeMapper">

    <insert id="insert" parameterType="com.example.employee.infrastructure.po.EmployeePO">
        INSERT INTO employee (
            employee_number,
            employee_name,
            phone,
            id_card_no,
            employee_level,
            employee_title,
            create_time,
            update_time
        )
        VALUES (
            #{employeeNumber},
            #{employeeName},
            #{phone},
            #{idCardNo},
            #{employeeLevel},
            #{employeeTitle},
            NOW(),
            NOW()
        )
    </insert>

    <update id="update" parameterType="com.example.employee.infrastructure.po.EmployeePO">
        UPDATE employee
        SET
            employee_name = #{employeeName},
            phone = #{phone},
            id_card_no = #{idCardNo},
            employee_level = #{employeeLevel},
            employee_title = #{employeeTitle},
            update_time = NOW()
        WHERE id = #{id}
    </update>

    <select id="selectById" resultType="com.example.employee.infrastructure.po.EmployeePO">
        SELECT
            id,
            employee_number AS employeeNumber,
            employee_name AS employeeName,
            phone,
            id_card_no AS idCardNo,
            employee_level AS employeeLevel,
            employee_title AS employeeTitle,
            create_time AS createTime,
            update_time AS updateTime
        FROM employee
        WHERE id = #{id}
    </select>

    <select id="selectByEmployeeNumber" resultType="com.example.employee.infrastructure.po.EmployeePO">
        SELECT
            id,
            employee_number AS employeeNumber,
            employee_name AS employeeName,
            phone,
            id_card_no AS idCardNo,
            employee_level AS employeeLevel,
            employee_title AS employeeTitle,
            create_time AS createTime,
            update_time AS updateTime
        FROM employee
        WHERE employee_number = #{employeeNumber}
    </select>

</mapper>
```

业务代码正常调用：

```java
EmployeePO employee = new EmployeePO();
employee.setEmployeeNumber("100001");
employee.setEmployeeName("张三");
employee.setPhone("13800000000");
employee.setIdCardNo("110101199001011234");
employee.setEmployeeLevel("T5");
employee.setEmployeeTitle("高级工程师");

employeeMapper.insert(employee);

// mapper 调用结束后，employee 对象里的字段仍然是明文。
// 插件内部会在 SQL 执行后恢复原值。
System.out.println(employee.getEmployeeName()); // 张三
```

---

# 9. 必须注意：随机 IV 加密不能直接做等值查询

这份插件使用的是更安全的随机 IV：

```text
同一个手机号：13800000000

第一次加密：ENC:v1:AAA...
第二次加密：ENC:v1:BBB...
第三次加密：ENC:v1:CCC...
```

所以这种 SQL 是无法正常工作的：

```sql
WHERE phone = #{phone}
```

因为数据库里的密文和本次查询参数加密后的密文大概率不同。

---

# 10. 生产环境推荐做法：加密字段 + 查询哈希字段

如果字段需要查询，例如手机号、身份证号，推荐表结构这样设计：

```sql
CREATE TABLE employee (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    employee_number VARCHAR(64) NOT NULL,

    employee_name VARCHAR(512),
    phone VARCHAR(512),
    phone_hash CHAR(64),

    id_card_no VARCHAR(512),
    id_card_no_hash CHAR(64),

    employee_level VARCHAR(32),
    employee_title VARCHAR(64),

    create_time DATETIME,
    update_time DATETIME,

    INDEX idx_phone_hash (phone_hash),
    INDEX idx_id_card_no_hash (id_card_no_hash)
);
```

存储时：

```text
phone       = AES-GCM(phone)
phone_hash  = HMAC-SHA256(phone)
```

查询时：

```sql
WHERE phone_hash = #{phoneHash}
```

也就是说：

|字段|作用|
|---|---|
|`phone`|可逆加密，用于回显|
|`phone_hash`|不可逆哈希，用于等值查询|
|`id_card_no`|可逆加密，用于回显|
|`id_card_no_hash`|不可逆哈希，用于等值查询|

这部分通常不建议完全塞进 MyBatis 插件里，最好在领域对象转换成 PO 的时候显式生成，避免查询语义隐藏太深。

---

# 11. 最小使用步骤

## 第一步：引入插件代码

放到类似包路径：

```text
com.example.mybatis.crypto
```

包括：

```text
EncryptedField.java
MybatisCryptoProperties.java
AesGcmStringEncryptor.java
MybatisFieldCryptoInterceptor.java
MybatisCryptoAutoConfiguration.java
```

---

## 第二步：配置密钥

```yaml
app:
  mybatis:
    crypto:
      enabled: true
      key: ${APP_MYBATIS_CRYPTO_KEY}
      version-prefix: "ENC:v1:"
```

---

## 第三步：给 PO 字段加注解

```java
@EncryptedField
private String phone;
```

---

## 第四步：Mapper 正常写

```xml
#{phone}
```

不需要手动加密。

---

## 第五步：验证数据库

插入前业务对象：

```text
phone = 13800000000
```

数据库里：

```text
phone = ENC:v1:Base64密文
```

查询后业务对象：

```text
phone = 13800000000
```

---

# 12. 这份代码的生产边界

这份插件适合处理：

- 入库加密；
    
- 出库解密；
    
- 对业务层透明；
    
- 数据库里避免明文存储敏感信息。
    

但以下能力建议单独设计：

|能力|建议方案|
|---|---|
|按加密字段等值查询|额外维护 HMAC 哈希列|
|模糊查询|不建议对加密字段做模糊查询|
|密钥轮换|密文版本号 + 批量迁移任务|
|多租户密钥|根据租户选择 key，需要改造 encryptor|
|历史明文数据兼容|当前代码已支持：非 `ENC:v1:` 前缀会原样返回|
|字段脱敏展示|建议放在接口 DTO 层，不要放 MyBatis 插件里|
|密码存储|不用本插件，使用 BCrypt / Argon2 哈希|