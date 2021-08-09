# Spring Boot 排除自动配置的 4 种方法
Spring Boot 提供的自动配置非常强大，某些情况下，自动配置的功能可能不符合我们的需求，需要我们自定义配置，这个时候就需要排除 / 禁用 Spring Boot 某些类的自动化配置了。

比如：数据源、邮件，这些都是提供了自动配置的，我们需要排除 Spring Boot 的自动化配置，交给我们自己来自定义，该如何做呢？

今天栈长给你介绍 4 种排除方式，总有一种能帮到你！

#### 方法 1

使用 `@SpringBootApplication` 注解的时候，使用 exclude 属性进行排除指定的类：

    @SpringBootApplication(exclude = {DataSourceAutoConfiguration.class, MailSenderAutoConfiguration.class})
    public class Application {
        
    }

自动配置类不在类路径下的时候，使用 excludeName 属性进行排除指定的类名全路径：

    @SpringBootApplication(excludeName = {"org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration", "org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration"})
    public class Application {
        
    }

#### 方法 2

单独使用 `@EnableAutoConfiguration` 注解的时候：

    @...
    @EnableAutoConfiguration
    (exclude = {DataSourceAutoConfiguration.class, MailSenderAutoConfiguration.class})
    public class Application {
        
    }

自动配置类不在类路径下的时候，使用 excludeName 属性进行排除指定的类名全路径：

    @...
    @EnableAutoConfiguration {"org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration", "org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration"})
    public class Application {
        
    }

#### 方法 3

使用 Spring Cloud 和 `@SpringCloudApplication` 注解的时候：

    @...
    @EnableAutoConfiguration
    (exclude = {DataSourceAutoConfiguration.class, MailSenderAutoConfiguration.class})
    @SpringCloudApplication
    public class Application {
        
    }

Spring Cloud 必须建立在 Spring Boot 应用之上，所以这个不用多解释了。

#### 方法 4

终极方案，不管是 Spring Boot 还是 Spring Cloud 都可以搞定，在配置文件中指定参数 `spring.autoconfigure.exclude` 进行排除：

    spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\\
        org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration

或者还可以这样写：

    spring.autoconfigure.exclude\[0\]\=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
    spring.autoconfigure.exclude\[1\]\=org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration

如果你用的是 yaml 配置文件，可以这么写：

    spring:     
      autoconfigure:
        exclude:
          - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
          - org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration
