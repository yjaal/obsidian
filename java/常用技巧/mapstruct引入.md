```toc
```
参考： https://www.mapstruct.plus/mapstruct/1-5-5-Final.html

## 引入

```groovy
// 这里忽略掉未映射的字段
compileJava {
    // dependsOn clean // 20160129 toutzhang CI 上用 clean build -x test，本地不用每次清理
    options.with {
        incremental = true
        failOnError = true
        compilerArgs += ['-Xlint:unchecked', '-Xlint:deprecation', '-parameters',
                         '-Amapstruct.unmappedTargetPolicy=IGNORE']
    }
}

var mapStructVersion = "1.5.5.Final"
var mapStruct = "org.mapstruct:mapstruct:${mapStructVersion}"
var mapStructProcessor = "org.mapstruct:mapstruct-processor:${mapStructVersion}"


dependencies {
    implementation mapStruct
    annotationProcessor mapStructProcessor
    testAnnotationProcessor mapStructProcessor
}
```

IDEA 安装插件 `MapStruct Support`

## 使用

### 基本使用

```java
@Mapper
public interface ContactWayChannelMapper {

    ContactWayChannelMapper INSTANCE = Mappers.getMapper(ContactWayChannelMapper.class);

    @Mapping(target = "relationId", ignore = true)
    @Mapping(target = "creator", ignore = true)
    @Mapping(target = "channelId", ignore = true)
    @Mapping(target = "channelCategoryName", ignore = true)
    @Mapping(target = "channelCategoryId", ignore = true)
    @Mapping(target = "id", source = "id")
    @Mapping(target = "channelName", source = "channelName")
    ContactChannelDto channel2Dto(ContactWayChannelEntity entity);
}
```

这里其实 `id、channelName` 在两个对象中的名字都是相同的，可以无需使用注解说明。而其他字段其实在配置中我们已经进行了忽略，可以不配置。当然如果需要配置则需要手动指定映射关系。

后面在 `build` 时就会在 `build/generated/sources/annotationProcessor` 目录下面找到实现类。当然如果使用 `Ctrl+F9` 编译一定注意使用 `gradle` 编译，如果使用 `IDEA` 原生编译会将实现类生成在 `src` 目录下面。


### 在转换类中添加自定义方法

还是使用上面的案例，如果上面方法无法满足需求，也可以自定义实现方法

```java
@Mapper
public interface ContactWayChannelMapper {

    ContactWayChannelMapper INSTANCE = Mappers.getMapper(ContactWayChannelMapper.class);


    default ContactChannelDto channel2Dto(ContactWayChannelEntity entity){
    // 自定义实现
}
```


### 多来源参数

```java
@Mapper
public interface AddressMapper {

    @Mapping(target = "description", source = "person.description")
    @Mapping(target = "houseNumber", source = "address.houseNo")
    DeliveryAddressDto personAndAddressToDeliveryAddressDto(Person person, Address address);
}
```

这里是将两个参数对象映射为一个 `DTO` 对象。如果在多个来源对象中，都有定义相同名称的属性，那么在映射时，必须通过 `@Mapping` 来指定这个属性的来源参数。

当然还可以直接使用一个具体类型的参数进行转换

```java
@Mapper
public interface AddressMapper {

    @Mapping(target = "description", source = "person.description")
    @Mapping(target = "houseNumber", source = "hn")
    DeliveryAddressDto personAndAddressToDeliveryAddressDto(Person person, Integer hn);
}
```


### 嵌套对象属性映射

```java
@Mapper
public interface CustomerMapper {

    @Mapping( target = "name", source = "record.name" )
    @Mapping( target = ".", source = "record" )
    @Mapping( target = ".", source = "account" )
    Customer customerDtoToCustomer(CustomerDto customerDto);
}
```

这里其实 `.` 就表示将 `CustomerDto.record` 对象中的所有属性都映射到 `Customer.record` 中。

当发生命名冲突时，可以通过显式定义映射关系来解决这些冲突。例如上面的例子中，name 属性同时存在于 `CustomerDto.record` 和 `CustomerDto.account` 中，`@Mapping(target = "name", source = "record.name")` 就是为了解决这个冲突的。


### 映射 Map 为 Bean

```java
public class Customer {

    private Long id;
    private String name;

    //getters and setter omitted for brevity
}

@Mapper
public interface CustomerMapper {

    @Mapping(target = "name", source = "customerName")
    Customer toCustomer(Map<String, String> map);

}
```


最后，如果映射场景较为复杂，请注意检查生成的实现类。