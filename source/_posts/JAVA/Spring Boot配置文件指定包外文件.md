# Spring Boot配置文件指定包外文件
## 通过命令行指定[#](#1625638747)

SpringApplication 会默认将命令行选项参数转换为配置信息  
例如，启动时命令参数指定：

`java -jar myproject.jar --server.port = 9000` 

从命令行指定配置项的优先级最高，不过你可以通过 setAddCommandLineProperties 来禁用

`SpringApplication.setAddCommandLineProperties(false).` 

## 外置配置文件[#](#582670658)

Spring 程序会按优先级从下面这些路径来加载 application.properties 配置文件

-   当前目录下的 / config 目录
-   当前目录
-   classpath 里的 / config 目录
-   classpath 跟目录

因此，要外置配置文件就很简单了，在 jar 所在目录新建 config 文件夹，然后放入配置文件，或者直接放在配置文件在 jar 目录

## 自定义配置文件[#](#2596559094)

如果你不想使用 application.properties 作为配置文件，怎么办？完全没问题

`java -jar myproject.jar --spring.config.location=classpath:/default.properties,classpath:/override.properties` 

或者

`java -jar -Dspring.config.location=D:\config\config.properties springbootrestdemo-0.0.1-SNAPSHOT.jar` 

当然，还能在代码里指定

\`@SpringBootApplication
@PropertySource(value={"file:config.properties"})
public class SpringbootrestdemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootrestdemoApplication.class, args);
    }

}\` 

## 按 Profile 不同环境读取不同配置[#](#1081893387)

不同环境的配置设置一个配置文件，例如：

-   dev 环境下的配置配置在 application-dev.properties 中；
-   prod 环境下的配置配置在 application-prod.properties 中。

在 application.properties 中指定使用哪一个文件

`spring.profiles.active = dev` 

当然，你也可以在运行的时候手动指定：

`java -jar myproject.jar --spring.profiles.active = prod`  
 [https://www.cnblogs.com/xiaoqi/p/6955288.html](https://www.cnblogs.com/xiaoqi/p/6955288.html)
