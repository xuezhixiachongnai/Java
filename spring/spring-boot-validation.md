# SpringBootValidation

## 概述

这是springboot提供的数据验证机制，用于确保用户输入的正确性和应用程序数据的完整性

## 使用已提供的注解

如：

```java
@Data
public class UserCreateRequestVO {

    @NotBlank(message = "请选择用户", groups = UpdateUserGroup.class)
    private String userId;

    @NotBlank(message = "请输入用户名", groups = CreateUserGroup.class)
    @Size(max = 128, message = "用户名长度最大为128个字符")
    private String userName;

    @Email(message = "请填写正确的邮箱地址")
    private String email;

    @Min(value = 18, message = "用户年龄必须大于18岁")
    @Max(value = 60, message = "用户年龄必须小于60岁")
    private Integer age;

    @NotNull(message = "请输入性别")
    @EnumValid(enumClass = SexEnum.class, message = "输入性别不合法")
    private Integer sex;

    @NotEmpty(message = "请输入你的兴趣爱好")
    @Size(max = 5, message = "兴趣爱好最多可以输入5个")
    private List<String> hobbies;

    @DecimalMin(value = "50", inclusive = false, message = "体重必须大于50KG")
    private BigDecimal weight;
}
```

如果类中使用了其他自定义类作为字段，那么这个嵌套类中的校验要生效，就必须使用`@Valid`注解这个字段

在一些service、controller中的方法参数做校验，那么就必须使用`@Validated`注解标注这个类，校验规则才能生效。

如自定义类`UserCreateRequestVO`作为方法参数，需要在其上使用`@Validated`参数注解，才能使该参数内部的校验生效

## 自定义验证注解

实现这个功能，首先需要顶一个注解作为标识标注需要校验的字段

```java
@Constraint(validatedBy = {EnumValidator.class}) // 该注解用于定义校验注解，validatedBy属性是指定的校验器
@Target({ ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE,
        ElementType.CONSTRUCTOR, ElementType.PARAMETER })
@Retention(RetentionPolicy.RUNTIME)
public @interface EnumValid {

    /**
     * 不合法时 抛出异常信息
     */
    String message() default "值不合法";

    /**
     * 校验的枚举类
     * @return
     */
    Class enumClass() default Enum.class;

    /**
     * 对应枚举类中需要比对的字段
     * @return
     */
    String field() default "code";

    Class<?>[] groups() default { };

    Class<? extends Payload>[] payload() default { };
}
```

然后需要通过实现`ConstraintValidator`接口来实现自定义校验器，该类定义了校验规则

```java
// A是自定义注解，T是校验字段类型
public interface ConstraintValidator<A extends Annotation, T> {
    
    // 在验证器初始化时被调用，可以用来获取注解约束中的配置信息
    void initialize(A constraintAnnontation);
    
    // 执行实际的验证逻辑，true表示验证通过，false表示验证失败
    // value是框架自动注入的需要校验字段的值
    // ConstraintValidatorContext 是用于在自定义校验器中构建、控制、输出错误提示信息的上下文对象
    boolean isValid(T value, ConstraintValidatorContext context);
}
```

## 分组校验

可以根据场景以及业务的差异性有选择的执行特定组的验证规则

```java
// 定义两个接口用于标识不同的业务场景
public interface CreateUserGroup extends Default {  
}

public interface UpdateUserGroup extends Default {  
}

```

在指定校验规则时可以通过`@NotBlank(message = "请选择用户", groups = UpdateUserGroup.class)`给`group`属性赋值来分组

使用时，也需要为`value`赋予指定的组`@Validated(value = CreateUserGroup.class)`，相同组即可实现验证