# 记一次spring boot 功能模块化 freemarker只能识别一个resources目录下前端展示模板问题 - 一枚码农的个人空间 - OSCHINA - 中文开源技术交流社区
今天开发个人开源框架，完善了系统模块后，集成工作流功能模块，在新的 resources 下面新建 freemarker 前端页面 (ftl 页面)，然后正常开发到展示流程表的时候，后台报错情景重新：**Could not resolve view with name '/actList' in servlet with name 'dispatcherServlet'**

![](https://static.oschina.net/uploads/space/2018/0116/183901_IMPm_3312115.png)

当时我就想是不是映射地址出错了，检查多次后发现也没错，怎么 spring mvc 就映射不到呢，而且是相同的的地址路径 (模块不一样)，和其他模块 resources 一致的路径，然后我系统模块是完全没问题的，我感觉应该是 freemarker 配置的问题，然后各种尝试，各种检查，各种查阅资料，发现，的确是 freemarker 的问题，我少配置了一项 freemarker 配置：**FreeMarkerConfigurationFactory**类下的属性：**preferFileSystemAccess，&lt;--- 对就是它，**我们来看源码发现默认是**true**

    public class FreeMarkerConfigurationFactory {

    	protected final Log logger = LogFactory.getLog(getClass());

    	private Resource configLocation;

    	private Properties freemarkerSettings;

    	private Map<String, Object> freemarkerVariables;

    	private String defaultEncoding;

    	private final List<TemplateLoader> templateLoaders = new ArrayList<TemplateLoader>();

    	private List<TemplateLoader> preTemplateLoaders;

    	private List<TemplateLoader> postTemplateLoaders;

    	private String[] templateLoaderPaths;

    	private ResourceLoader resourceLoader = new DefaultResourceLoader();

    	private boolean preferFileSystemAccess = true;

那么他是干嘛用的呢，我们看官方介绍是介绍：

Set whether to prefer file system access for template loading. File system access enables hot detection of template changes.  
If this is enabled, FreeMarkerConfigurationFactory will try to resolve the specified "templateLoaderPath" as file system resource (which will work for expanded class path resources and ServletContext resources too).

Default is "true". Turn this off to always load via SpringTemplateLoader (i.e. as stream, without hot detection of template changes), which might be necessary if some of your templates reside in an expanded classes directory while others reside in jar files.

我们随便翻一下：第一句的意思是是否以更喜欢方式来访问模板

如果启用就以 templateLoaderPath 设置的路径来加载，默认为 true，设置为 false 的话就以 SpringTemplateLoader 方式访问，那么就是流的方式访问，这可能是必要的, 如果你的一些模板驻留在一个扩展的类而其他人驻留在 jar 文件的目录。

意思就是 如果你的文件在 jar 包文件目录下，就有必要设置为 false，我们是多模块的，最终只运行一个模块，其他模块都是以 jar 包来加载的，我们要加载 jar 包里面的文件目录，必须要把这个属性设置为 false 才行，debug 源码看看，先上配置文件：

    @Bean
      public FreeMarkerConfigurer freemarkerConfig() throws IOException, TemplateException {
        FreeMarkerConfigurationFactory factory = new FreeMarkerConfigurationFactory();
        factory.setTemplateLoaderPath("classpath:/ftl/");
        factory.setDefaultEncoding("UTF-8");
        factory.setPreferFileSystemAccess(true);
        FreeMarkerConfigurer result = new FreeMarkerConfigurer();
        freemarker.template.Configuration configuration = factory.createConfiguration();
        configuration.setClassicCompatible(true);
        result.setConfiguration(configuration);
        Properties settings = new Properties();
        settings.put("template_update_delay", "0");
        settings.put("default_encoding", "UTF-8");
        settings.put("number_format", "0.######");
        settings.put("classic_compatible", true);
        settings.put("template_exception_handler", "ignore");
        result.setFreemarkerSettings(settings);
        return result;
      }

**FreeMarkerConfigurationFactory**

![](https://static.oschina.net/uploads/space/2018/0116/190228_Oq2F_3312115.png)

返回![](https://static.oschina.net/uploads/space/2018/0116/190505_StVf_3312115.png)

最终返回：

![](https://static.oschina.net/uploads/space/2018/0116/191335_abkt_3312115.png)

看到没，是单个模块的绝对路径 ftl 文件夹。

看图我们可以明确知道为 true 的话，加载的是设置的 setTemplateLoaderPath 的路径 也就是我其中一个模块下的 ftl 路径，所以**其他模块下 ftl 路径是加载不出来的**，也就**是 spring mvc 映射不到的原因。** 

我们试试 将**isPreferFileSystemAccess**设置为**false：** 

![](https://static.oschina.net/uploads/space/2018/0116/190741_z84S_3312115.png)

最终我们跟到这里：

![](https://static.oschina.net/uploads/space/2018/0116/191129_wHTf_3312115.png)

发现路径是 / ftl / 而不是之前的绝对路径，然后 set 进 spring 对 freemarker 的配置对象中。

好的，我们通过分析源码我们可以知道，多模块下，我们**设置为 false，spring 才能以相对路径来访问我们的模板文件夹**，问题解决。

```java
#spring boot 配置项
spring.freemarker.prefer-file-system-access=false
```

 [https://my.oschina.net/u/3312115/blog/1608042](https://my.oschina.net/u/3312115/blog/1608042)
