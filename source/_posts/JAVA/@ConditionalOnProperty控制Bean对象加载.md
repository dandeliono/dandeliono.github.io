# @ConditionalOnProperty控制Bean对象加载
**1、简介**

 　　SpringBoot 通过 @ConditionalOnProperty 来控制 @Configuration 是否生效

**2、说明**

    @Retention(RetentionPolicy.RUNTIME)
    @Target({ ElementType.TYPE, ElementType.METHOD })
    @Documented
    @Conditional(OnPropertyCondition.class) public @interface ConditionalOnProperty {

       String\[\] value() default {}; //数组，获取对应property名称的值，与name不可同时使用 
     String prefix() default "";//property名称的前缀(最后一个属性前的一串。比如aaa.bbb.ccc，则prefix为aaa.bbb)，可有可无 
     String\[\] name() default {};//数组，property完整名称或部分名称（可与prefix组合使用，组成完整的property名称），与value不可同时使用 
     String havingValue() default "";//可与name组合使用，比较获取到的属性值与havingValue给定的值是否相同，相同才加载配置 
     boolean matchIfMissing() default false;//缺省配置。配置文件没有配置时，表示使用当前property。配置文件与havingValue匹配时，也使用当前property
     boolean relaxedNames() default true;//是否可以松散匹配，至今不知道怎么使用的 
    }

**3、单个示例**

　a、建立一个 Configuration 配置类:

```

@Data
@Configuration
@ConditionalOnProperty(value \= "test.conditional", havingValue = "abc") public class TestConfiguration { /\*\* 默认值 \*/
    private String field = "default";

}
```

　b、建立一个测试类：

    @RestController
    @RequestMapping("test") public class TestController {

        @Autowired(required \= false) private TestConfiguration test1Configuration;

        @GetMapping("testConfiguration") public String testConfiguration() { if (test1Configuration != null) { return test1Configuration.getField();
            } return "OK";
        }

    }

c、配置文件 application.yml：

 d、通过 postman 或者其它工具发送请求。结果如下：

以上表明 TestConfiguration 配置文件生效了

**4、多种示例**

接下来，改变 @ConditionalOnProperty 中的各个属性，然后通过查看返回结果来判断 TestConfiguration 是否生效。

1、不配置 @ConditionalOnProperty，直接生效。

2、只有 value 属性，没有 havingValue 属性。如果 application.yml 配置了 test.conditional 则生效，否则不生效。  

    @ConditionalOnProperty(value = "test.conditional")

3、prefix + name 相当于 value 属性 (两者不可同时使用)。如果 application.yml 配置了 test.conditional 则生效，否则不生效

    @ConditionalOnProperty(prefix = "test", name = "conditional")

4、name 属性为一个数组，当要匹配多个值时，如果 application.yml 的配置与 name 属性中的值一一匹配则生效，否则不生效

    @ConditionalOnProperty(prefix = "test", name = { "conditional", "other" })

    test:
      conditional: abc
      other: edf

5、当 matchIfMissing=true 时：

 a、如果 application.yml 配置了 test.conditional 则生效 (此时 matchIfMissing 可有可无)，否则不生效

 b、如果 application.yml 啥都没配置则生效

    @ConditionalOnProperty(prefix = "test", name = "conditional", matchIfMissing = true)

6、加上 havingValue 属性。当 havingValue 的值与 application.yml 文件中 test.conditional 的值一致时则生效，否则不生效

    @ConditionalOnProperty(prefix = "test", name = "conditional", havingValue = "abc")

7、加上 havingValue 属性。name 属性为数组时，如果 application.yml 文件中配置了相关属性且值都一致时则生效，否则不生效

    @ConditionalOnProperty(prefix = "test", name = { "conditional", "other" }, havingValue = "abc")

    test:
      conditional: abc
      other: abc

 [https://www.cnblogs.com/xuwenjin/p/12603232.html](https://www.cnblogs.com/xuwenjin/p/12603232.html)
