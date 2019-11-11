# Jackson 笔记

## 封装`Jackson`帮助类

```java
import com.bh.bitOctopus.bean.ResponseData;
import com.bh.bitOctopus.bean.TestBean;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.JavaType;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.springframework.util.StringUtils;

/**
 * json 帮助类
 *
 * @author zbs
 * @date 2019/11/4
 */
@Slf4j
public class JsonUtil {

    private static ObjectMapper objectMapper = new ObjectMapper();

    static {
        //忽略 在json字符串中存在，但是在java对象中不存在对应属性的情况。防止错误
        objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }

    /**
     * java 对象转换为 JSON string
     *
     * @param obj
     * @param <T>
     * @return
     */
    public static <T> String toJSONString(T obj) {
        if (obj == null) {
            return null;
        }

        try {
            return obj instanceof String ? obj.toString() : objectMapper.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            log.error("将java对象转换为字符串错误", e);
            return null;
        }
    }

    /**
     * java 对象转换为 JSON 字符串并格式化输出
     *
     * @param obj
     * @param <T>
     * @return
     */
    public static <T> String toJSONStringPretty(T obj) {
        if (obj == null) {
            return null;
        }

        try {
            return obj instanceof String ? (String) obj :
                    objectMapper.writerWithDefaultPrettyPrinter().writeValueAsString(obj);
        } catch (Exception e) {
            log.error("将java对象转换为字符串错误", e);
            return null;
        }
    }

    /**
     * 字符串转换为 java 对象
     *
     * @param str
     * @param clazz
     * @param <T>
     * @return
     */
    public static <T> T toJavaObject(String str, Class<T> clazz) {
        if (StringUtils.isEmpty(str) || clazz == null) {
            return null;
        }

        try {
            return clazz.equals(String.class) ? (T) str : objectMapper.readValue(str, clazz);
        } catch (Exception e) {
            log.warn("字符串转换为 java 对象错误", e);
            return null;
        }
    }

    /**
     * 字符串转换为 java 对象(支持泛型)
     *
     * @param str
     * @param typeReference
     * @param <T>
     * @return
     * @deprecated 创建 TypeReference 类型 : new TypeReference<ResponseData<TestBean>>(){};
     */
    public static <T> T toJavaObject(String str, TypeReference<T> typeReference) {
        if (StringUtils.isEmpty(str) || typeReference == null) {
            return null;
        }

        try {
            return objectMapper.readValue(str, typeReference);
        } catch (Exception e) {
            log.warn("字符串转换为 java 对象错误", e);
            return null;
        }
    }

    /**
     * 字符串转换为 java 集合
     *
     * @param str
     * @param collectionClass
     * @param elementClasses
     * @param <T>
     * @return
     * @deprecated 例如：将List<Student>类型的字符串转换为集合对象，则: toJavaList(str, List.class, Student.class);
     */
    public static <T> T toJavaList(String str, Class<?> collectionClass, Class<?>... elementClasses) {
        JavaType javaType = objectMapper.getTypeFactory().constructParametricType(collectionClass, elementClasses);

        try {
            return objectMapper.readValue(str, javaType);
        } catch (Exception e) {
            log.warn("字符串转换为 java 集合错误", e);
            return null;
        }
    }

}
```

## 常用注解

### @JsonIgnore忽略某个属性

将字符串转换为java对象或者将java对象转换json字符串，@JsonIgnore标注的字段都被忽略

### @JsonProperty

当Json的属性值和Java的属性值不一样时，会映射失败，用这个注解指定映射关系，在属性上用这个注解，则序列化和反序列化都会用这个值。如果序列化和反序列化的属性不一致，可以在get方法或者set方法上用这个注解，set方法影响反序列化，get方法影响序列化。

```java
public class Student {

    /** 名字 */
    private String name;
    /** 年龄 */
    private Integer age;
    /** 头像 */
    private String profileImageUrl;

    @JsonProperty("getImage")
    public String getProfileImageUrl() {
        return profileImageUrl;
    }

    @JsonProperty("setImage")
    public void setProfileImageUrl(String profileImageUrl) {
        this.profileImageUrl = profileImageUrl;
    }
}
```

### @JsonFormat

日期格式化注解

 注解@JsonFormat主要是后台到前台的时间格式的转换

```java
// 在查询出来的时间的数据库字段对应的实体类的属性上添加@JsonFormat
@Data
public class TestClass {
  
    // 设置时区为上海时区，时间格式自己据需求定。
  	// pattern:是你需要转换的时间日期的格式
   	// timezone：是时间设置为东八区，避免时间在转换中有误差
    @JsonFormat(pattern="yyyy-MM-dd",timezone = "GMT+8")
    private Date testTime;
  
}
```

### @DateTimeFormat

对应的接收前台数据的对象的属性上加@DateTimeFormat

注解@DataFormAT主要是前后到后台的时间格式的转换

```java
@DateTimeFormat(pattern = "yyyy-MM-dd")
@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss",timezone="GMT+8")
private Date symstarttime;
```

### @JsonIgnoreProperties(ignoreUnknown = true)

如果json字符串中出现java对象中没有的属性，则在将json转换为java对象时会报错：Unrecognized field, not marked as ignorable

1、在目标对象的类级别上添加注解：@JsonIgnoreProperties(ignoreUnknown = true)

2、配置

```java
static {
  //忽略 在json字符串中存在，但是在java对象中不存在对应属性的情况。防止错误
  objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
}
```

