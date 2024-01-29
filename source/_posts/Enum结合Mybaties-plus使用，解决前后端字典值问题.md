---
title: Enum结合Mybaties-plus使用，解决前后端字典值问题
date: 2024-01-29 15:49:24
tags: enum, java
categories: SpringBoot
---

**现状**：对于字典值，大部分情况下，前端需要展示中文名称，后端数据库中存储的是 code 或者 value 值，会造成两个问题：

1. 在后端代码中需要定义大量的 Contant 来解决 magic number 问题，较为难以维护
2. 在传递给前端时，对于列表接口需要自定义 Json 序列化处理成中文名称（label），但是对于详情接口又不需要处理

**目标**：

1. 在代码中能够更好的维护字典值，保证代码的可读性
2. 能够传递给前端 label 方便展示
3. 从数据库中存储和查询的时候还是 value 值，不需要额外处理

**解决方案**:

1. 定义枚举值对应数据库中的字典，例如

```java
import com.baomidou.mybatisplus.annotation.EnumValue;
import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonValue;

public enum FileSourceType {
    UPLOADED("1", "fileSourceType", "上传"),
    LINK_COMMON("2", "fileSourceType", "引用文件");

    @EnumValue
    //用于mybatis-plus将数据库中的值存入
    private String value;
    private String type;
    @JsonValue
    //用于传递给前端时json处理
    private String label;

    FileSourceType(String value, String type, String label) {
        this.value = value;
        this.type = type;
        this.label = label;
    }
  //...getter setter....

//用于接受前端传递过来的value值，自动成枚举。
    @JsonCreator
    public static FileSourceType getEnum(String value) {
        for (FileSourceType type : FileSourceType.values()) {
            if (type.getValue().equals(value)) {
                return type;
            }
        }
        return null;
    }

}
```

2. 配置 Java Bean

```java


public class FileDomain extends BaseEntity {
   //....

//直接使用枚举作为成员
  private FileSourceType sourceType;
  //...

}
```

3. 全局配置

- Json 序列化配置

```java
@Bean
public Jackson2ObjectMapperBuilderCustomizer jacksonObjectMapperCustomization() {
    return builder -> builder
            .featuresToEnable(SerializationFeature.WRITE_ENUMS_USING_TO_STRING);
}
```

- Mybatis-plus 配置

```yml
mybatis-plus:
  configuration:
    default-enum-type-handler: com.baomidou.mybatisplus.core.handlers.MybatisEnumTypeHandler
```

**效果**：

1. 返回的数据会自动根据枚举中的@JsonValue 注解或者 toString 方法进行处理。
2. 数据库中存储的值没有变化，还是 value 值。
3. 前端传递过来的 object 中包含枚举的 value，可以自动转换成 enum。
4. 在代码中设置值或者校验值更容易了，比如：

```java
//可读性较好
commonFile.setSourceType(FileSourceType.LINK_COMMON);
```

**缺点**：
对于前端需要使用下拉框回显的时候，需要额外创建一个 VO 来处理，此时需要 copyBean 将 enmu 获取到 value 返回。

refer: [Mybatis-plus 官方介绍](https://baomidou.com/pages/8390a4/#%E6%AD%A5%E9%AA%A41-%E5%A3%B0%E6%98%8E%E9%80%9A%E7%94%A8%E6%9E%9A%E4%B8%BE%E5%B1%9E%E6%80%A7)
