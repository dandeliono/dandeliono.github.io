# Spring/Spring boot正确集成Quartz及解决@Autowired失效问题 - 从此寂静无声 - 博客园
# quartz

(1) 项目依赖:

```null
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.0.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.quartz-scheduler</groupId>
            <artifactId>quartz</artifactId>
        </dependency>
    </dependencies>
```

(2) 问题代码:

```null
@Component
public class UnprocessedTaskJob extends QuartzJobBean {

    private TaskMapper taskMapper;

    @Autowired
    public UnprocessedTaskJob(TaskMapper taskMapper){
        this.taskMapper = taskMapper;
    }
}

private JobDetail generateUnprocessedJobDetail(Task task) {
    JobDataMap jobDataMap = new JobDataMap();
    jobDataMap.put(UnprocessedTaskJob.TASK_ID, task.getId());
    return JobBuilder.newJob(UnprocessedTaskJob.class)
            .withIdentity(UnprocessedTaskJob.UNPROCESSED_TASK_KEY_PREFIX + task.getId(), UnprocessedTaskJob.UNPROCESSED_TASK_JOB_GROUP)
            .usingJobData(jobDataMap)
            .storeDurably()
            .build();
    }
```

(3) 提炼问题:

以上代码存在错误的原因是,`UnprocessedTaskJob`添加`@Component`注解, 表示其是`Spring IOC`容器中的`单例`类.  
然而`Quartz`在创建`Job`是通过相应的`Quartz Job Bean`的`class`反射创建相应的`Job`. 也就是说, 每次创建新的`Job`时, 都会生成相应的`Job`实例. 从而, 这与`UnprocessedTaskJob`是`单例`相冲突.  
查看代码提交记录, 原因是当时认为不添加`@Component`注解, 则无法通过`@Autowired`引入由`Spring IOC`托管的`taskMapper`实例, 即无法实现`依赖注入`.

然而令人感到奇怪的是, 当我在开发环境去除了`UnprocessedTaskJob`的`@Component`注解之后, 运行程序后发现`TaskMapper`实例依然可以注入到`Job`中, 程序正常运行...

## Spring 托管 Quartz

### 代码分析

网上搜索`Spring`托管`Quartz`的文章, 大多数都是`Spring MVC`项目, 集中于如何解决在`Job`实现类中通过`@Autowired`实现`Spring`的`依赖注入`.  
网上大多实现均依赖`SpringBeanJobFactory`去实现`Spring`与`Quartz`的集成.

```null

public class SpringBeanJobFactory extends AdaptableJobFactory
        implements ApplicationContextAware, SchedulerContextAware {
}


public class AdaptableJobFactory implements JobFactory {
}
```

通过上述代码以及注释可以发现:  
(1) `AdaptableJobFactory`实现了`JobFactory`接口, 可以藉此创建标准的`Quartz`实例 (仅限于`Quartz` 2.1.4 及以上版本);  
(2) `SpringBeanJobFactory`继承于`AdaptableJobFactory`, 从而实现对`Quartz`封装实例的属性依赖注入.  
(3) `SpringBeanJobFactory`实现了`ApplicationContextAware`以及`SchedulerContextAware`接口 (`Quartz`任务调度上下文), 因此可以在创建`Job Bean`的时候注入`ApplicationContex`以及`SchedulerContext`.

Tips:  
以上代码基于`Spring` 5.1.8 版本.  
在`Spring 4.1.0`版本, `SpringBeanJobFactory`的实现如以下代码所示:

```null
public class SpringBeanJobFactory extends AdaptableJobFactory
    implements SchedulerContextAware{

    
}
```

因此, 在早期的`Spring`项目中, 需要封装`SpringBeanJobFactory`并实现`ApplicationContextAware`接口 (惊不惊喜?).

### Spring 老版本解决方案

基于老版本`Spring`给出解决`Spring`集成`Quartz`解决方案.  
解决方案由[第三十九章：基于 SpringBoot & Quartz 完成定时任务分布式单节点持久化](https://www.jianshu.com/p/d52d62fb2ac6)提供 (大神的系列文章质量很棒).

```null
@Configuration
public class QuartzConfiguration
{
    
    public static class AutowiringSpringBeanJobFactory extends SpringBeanJobFactory implements
            ApplicationContextAware {

        private transient AutowireCapableBeanFactory beanFactory;

        @Override
        public void setApplicationContext(final ApplicationContext context) {
            beanFactory = context.getAutowireCapableBeanFactory();
        }

        
        @Override
        protected Object createJobInstance(final TriggerFiredBundle bundle) throws Exception {
            final Object job = super.createJobInstance(bundle);
            
            beanFactory.autowireBean(job);
            return job;
        }
    }

    
    @Bean
    public JobFactory jobFactory(ApplicationContext applicationContext)
    {
        
        AutowiringSpringBeanJobFactory jobFactory = new AutowiringSpringBeanJobFactory();
        jobFactory.setApplicationContext(applicationContext);
        return jobFactory;
    }

    
    @Bean(destroyMethod = "destroy",autowire = Autowire.NO)
    public SchedulerFactoryBean schedulerFactoryBean(JobFactory jobFactory, DataSource dataSource) throws Exception
    {
        SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
        
        schedulerFactoryBean.setJobFactory(jobFactory);
        
        schedulerFactoryBean.setOverwriteExistingJobs(true);
        
        schedulerFactoryBean.setStartupDelay(2);
        
        schedulerFactoryBean.setAutoStartup(true);
        
        schedulerFactoryBean.setDataSource(dataSource);
        
        schedulerFactoryBean.setApplicationContextSchedulerContextKey("applicationContext");
        
        schedulerFactoryBean.setConfigLocation(new ClassPathResource("/quartz.properties"));
        return schedulerFactoryBean;
    }
}
```

通过以上代码, 就实现了由`SpringBeanJobFactory`的`createJobInstance`创建`Job`实例, 并将生成的`Job`实例交付由`AutowireCapableBeanFactory`来托管.  
`schedulerFactoryBean`则设置诸如`JobFactory`(实际上是`AutowiringSpringBeanJobFactory`, 内部封装了`applicationContext`) 以及`DataSource`(数据源, 如果不设置, 则`Quartz`默认使用`RamJobStore`).

> `RamJobStore`优点是运行速度快, 缺点则是调度任务无法持久化保存.

因此, 我们可以在定时任务内部使用`Spring IOC`的`@Autowired`等注解进行`依赖注入`.

### Spring 新版本解决方案

(1) 解释

如果你使用`Spring boot`, 并且版本好大于`2.0`, 则推荐使用`spring-boot-starter-quartz`.

```null
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-quartz</artifactId>
    </dependency>
```

> Auto-configuration support is now include for the Quartz Scheduler. We’ve also added a new spring-boot-starter-quartz starter POM.  
> You can use in-memory JobStores, or a full JDBC-based store. All JobDetail, Calendar and Trigger beans from your Spring application context will be automatically registered with the Scheduler.  
> For more details read the new “Quartz Scheduler” section of the reference documentation.

以上是`spring-boot-starter-quartz`的介绍, 基于介绍可知, 如果你没有关闭`Quartz`的自动配置, 则`SpringBoot`会帮助你完成`Scheduler`的自动化配置, 诸如`JobDetail`/`Calendar`/`Trigger`等`Bean`会被自动注册至`Shceduler`中. 你可以在`QuartzJobBean`中自由的使用`@Autowired`等`依赖注入`注解.

其实, 不引入`spring-boot-starter-quartz`, 而仅仅导入`org.quartz-scheduler`,`Quartz`的自动化配置依然会起效 (这就是第一节问题分析中, 去除`@Bean`注解, 程序依然正常运行原因, 悲剧中万幸).

(2) 代码分析

```null

@Configuration
@ConditionalOnClass({ Scheduler.class, SchedulerFactoryBean.class, PlatformTransactionManager.class })
@EnableConfigurationProperties(QuartzProperties.class)
@AutoConfigureAfter({ DataSourceAutoConfiguration.class, HibernateJpaAutoConfiguration.class })
public class QuartzAutoConfiguration{

    

    @Bean
    @ConditionalOnMissingBean
    public SchedulerFactoryBean quartzScheduler() {
        
        
        SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
        SpringBeanJobFactory jobFactory = new SpringBeanJobFactory();
        
        jobFactory.setApplicationContext(this.applicationContext);
        
        
        schedulerFactoryBean.setJobFactory(jobFactory);

        
        if (this.properties.getSchedulerName() != null) {
            schedulerFactoryBean.setSchedulerName(this.properties.getSchedulerName());
        }
        schedulerFactoryBean.setAutoStartup(this.properties.isAutoStartup());schedulerFactoryBean.setStartupDelay((int) this.properties.getStartupDelay().getSeconds());
        schedulerFactoryBean.setWaitForJobsToCompleteOnShutdown(this.properties.isWaitForJobsToCompleteOnShutdown());
        schedulerFactoryBean.setOverwriteExistingJobs(this.properties.isOverwriteExistingJobs());
        if (!this.properties.getProperties().isEmpty()) {
            schedulerFactoryBean.setQuartzProperties(asProperties(this.properties.getProperties()));
        }
        if (this.jobDetails != null && this.jobDetails.length > 0) {
            schedulerFactoryBean.setJobDetails(this.jobDetails);
        }
        if (this.calendars != null && !this.calendars.isEmpty()) {
            schedulerFactoryBean.setCalendars(this.calendars);
        }
        if (this.triggers != null && this.triggers.length > 0) {
            schedulerFactoryBean.setTriggers(this.triggers);
        }
        customize(schedulerFactoryBean);
        return schedulerFactoryBean;
    }

    @Configuration
    @ConditionalOnSingleCandidate(DataSource.class)
    protected static class JdbcStoreTypeConfiguration {

        
    }
}
```

下面对`SpringBeanJobFactory`进行分析, 它是生成`Job`实例, 以及进行`依赖注入`操作的关键类.

```null

public class SpringBeanJobFactory extends AdaptableJobFactory
        implements ApplicationContextAware, SchedulerContextAware {

    @Nullable
    private String[] ignoredUnknownProperties;

    @Nullable
    private ApplicationContext applicationContext;

    @Nullable
    private SchedulerContext schedulerContext;

    
    public void setIgnoredUnknownProperties(String... ignoredUnknownProperties) {
        this.ignoredUnknownProperties = ignoredUnknownProperties;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    @Override
    public void setSchedulerContext(SchedulerContext schedulerContext) {
        this.schedulerContext = schedulerContext;
    }

    
    @Override
    protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
        
        
        
        Object job = (this.applicationContext != null ?
                        this.applicationContext.getAutowireCapableBeanFactory().createBean(
                            bundle.getJobDetail().getJobClass(), AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR, false) :
                        super.createJobInstance(bundle));

        if (isEligibleForPropertyPopulation(job)) {
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(job);
            MutablePropertyValues pvs = new MutablePropertyValues();
            if (this.schedulerContext != null) {
                pvs.addPropertyValues(this.schedulerContext);
            }
            pvs.addPropertyValues(bundle.getJobDetail().getJobDataMap());
            pvs.addPropertyValues(bundle.getTrigger().getJobDataMap());
            if (this.ignoredUnknownProperties != null) {
                for (String propName : this.ignoredUnknownProperties) {
                    if (pvs.contains(propName) && !bw.isWritableProperty(propName)) {
                        pvs.removePropertyValue(propName);
                    }
                }
                bw.setPropertyValues(pvs);
            }
            else {
                bw.setPropertyValues(pvs, true);
            }
        }

        return job;
    }

    
}


public class AdaptableJobFactory implements JobFactory {
    
    protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
        
        Class<?> jobClass = bundle.getJobDetail().getJobClass();
        return ReflectionUtils.accessibleConstructor(jobClass).newInstance();
    }
}
```

此处需要解释下`AutowireCapableBeanFactory`的作用.  
项目中, 有部分实现并未与`Spring`深度集成, 因此其实例并未被`Spring`容器管理.  
然而, 出于需要, 这些并未被`Spring`管理的`Bean`需要引入`Spring`容器中的`Bean`.  
此时, 就需要通过实现`AutowireCapableBeanFactory`, 从而让`Spring`实现依赖注入等功能.

 [https://www.cnblogs.com/jason1990/p/11110196.html](https://www.cnblogs.com/jason1990/p/11110196.html)
