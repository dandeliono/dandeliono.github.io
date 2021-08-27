# 掌握MyBatis插件原理轻松写出自己的PageHelper分页插件-阿里云开发者社区
在 MyBatis 中插件式通过拦截器来实现的，那么既然是通过拦截器来实现的，就会有一个问题，哪些对象才允许被拦截呢？

真正执行 Sql 的是四大对象：Executor，StatementHandler，ParameterHandler，ResultSetHandler。

而 MyBatis 的插件正是基于拦截这四大对象来实现的。需要注意的是，虽然我们可以拦截这四大对象，但是并不是这四大对象中的所有方法都能被拦截，下面就是官网提供的可拦截的对象和方法汇总：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/b0ab18e1-85b3-4ce7-8359-2981bfb1b190.png?raw=true)

首先我们先来通过一个例子来看看如何使用插件。

## 1、 首先建立一个 MyPlugin 实现接口 Interceptor，然后重写其中的三个方法 (注意，这里必须要实现 Interceptor 接口，否则无法被拦截)。

```null
package com.lonelyWolf.mybatis.plugin;

import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;
import java.util.Properties;

@Intercepts({@Signature(type = Executor.class,method = "query",args = {MappedStatement.class,Object.class, RowBounds.class, ResultHandler.class})})
public class MyPlugin implements Interceptor {

    
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("成功拦截了Executor的query方法，在这里我可以做点什么");
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target,this);
    }

    @Override
    public void setProperties(Properties properties) {
        System.out.println("自定义属性:userName->" + properties.getProperty("userName"));
    }
}
```

@Intercepts 是声明当前类是一个拦截器，后面的 @Signature 是标识需要拦截的方法签名，通过以下三个参数来确定

(1)type：被拦截的类名

(2)method：被拦截的方法名

(3)args：标注方法的参数类型

## 2、 我们还需要在 mybatis-config 中配置好插件。

```null
<plugins>
    <plugin interceptor="com.lonelyWolf.mybatis.plugin.MyPlugin">
        <property name="userName" value="张三"/>
    </plugin>
</plugins>
```

这里如果配置了 property 属性，那么我们可以在 setProperties 获取到。

完成以上两步，我们就完成了一个插件的配置了，接下来我们运行一下：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/71853790-fc74-46c1-be54-caa8ba05a104.png?raw=true)

可以看到，setProperties 方法在加载配置文件阶段就会被执行了。

接下来让我们分析一下从插件的加载到初始化到运行整个过程的实现原理。

## 插件的加载

既然插件需要在配置文件中进行配置，那么肯定就需要进行解析，我们看看插件式如何被解析的。我们进入 XMLConfigBuilder 类看看  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/3a3b64d0-c360-41f4-929a-00b63cff8939.png?raw=true)

解析出来之后会将插件存入 InterceptorChain 对象的 list 属性  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/a3a772a3-63a0-4028-8c59-27931fda8e52.png?raw=true)

看到 InterceptorChain 我们是不是可以联想到，MyBatis 的插件就是通过责任链模式实现的。

既然插件类已经被加载到配置文件了，那么接下来就有一个问题了，插件类何时会被拦截我们需要拦截的对象呢？

其实插件的拦截是和对象有关的，不同的对象进行拦截的时间也会不一致，接下来我们就逐一分析一下。

## 拦截 Executor 对象

我们知道，SqlSession 对象是通过 openSession() 方法返回的，而 Executor 又是属于 SqlSession 内部对象，所以让我们跟随 openSession 方法去看一下 Executor 对象的初始化过程。  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/027dcb5a-11e2-498e-9bf9-db63da9e2b56.png?raw=true)

可以看到，当初始化完成 Executor 之后，会调用 interceptorChain 的 pluginAll 方法，pluginAll 方法本身非常简单，就是把我们存到 list 中的插件进行循环，并调用 Interceptor 对象的 plugin 方法：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/543eb986-e8e9-4955-90d9-ed4473a4cad9.png?raw=true)

再次点击进去：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/21c1e68f-5613-41c8-8da6-1c1c2e2ce234.png?raw=true)

到这里我们是不是发现很熟悉，没错，这就是我们上面示例中重写的方法，而 plugin 方法是接口中的一个默认方法。

这个方法是关键，我们进去看看：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/3a791490-5ed0-4251-8d7d-cd34427a1ea6.png?raw=true)

可以看到这个方法的逻辑也很简单，但是需要注意的是 MyBatis 插件是通过 JDK 动态代理来实现的。

而 JDK 动态代理的条件就是被代理对象必须要有接口，这一点和 Spring 中不太一样，Spring 中是如果有接口就采用 JDK 动态代理，没有接口就是用 CGLIB 动态代理。

正因为 MyBatis 的插件只使用了 JDK 动态代理，所以我们上面才强调了一定要实现 Interceptor 接口。

而代理之后汇之星 Plugin 的 invoke 方法，我们最后再来看看 invoke 方法：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/2b675017-1c9f-41af-9750-871d4c36f494.png?raw=true)

而最终执行的 intercept 方法，就是我们上面示例中重写的方法。

## 其他对象插件解析

接下来我们再看看 StatementHandler，StatementHandler 是在 Executor 中的 doQuery 方法创建的，其实这个原理就是一样的了，找到初始化 StatementHandler 对象的方法：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/03644c7c-84ca-4673-a135-56d387936a0c.png?raw=true)

进去之后里面执行的也是 pluginAll 方法：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/5aa0844c-c333-4d45-9b62-cabda22ee4ac.png?raw=true)

其他两个对象就不在举例了，其实搜一下全局就很明显了：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/ef8085fa-b60b-4381-b266-bcaf0edaa8c5.png?raw=true)

四个对象初始化的时候都会调用 pluginAll 来进行判定是否有被代理。

下面就是实现了插件之后的执行时序图：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/fd4d584c-319c-4724-8e0e-bdd6e571eeb8.png?raw=true)

一个对象是否可以被多个代理对象进行代理？也就是说同一个对象的同一个方法是否可以被多个拦截器进行拦截？

答案是肯定的，因为被代理对象是被加入到 list，所以我们配置在最前面的拦截器最先被代理，但是执行的时候却是最外层的先执行。

具体点：

假如依次定义了三个插件：插件 A，插件 B 和 插件 C。

那么 List 中就会按顺序存储：插件 A，插件 B 和 插件 C。

而解析的时候是遍历 list，所以解析的时候也是按照：插件 A，插件 B 和 插件 C 的顺序。

但是执行的时候就要反过来了，执行的时候是按照：插件 C，插件 B 和插件 A 的顺序进行执行。

上面我们了解了在 MyBatis 中的插件是如何定义以及 MyBatis 中是如何处理插件的，接下来我们就以经典分页插件 PageHelper 为例来进一步加深理解。

首先我们看看 PageHelper 的用法：

```null
package com.lonelyWolf.mybatis;

import com.alibaba.fastjson.JSONObject;
import com.github.pagehelper.Page;
import com.github.pagehelper.PageHelper;
import com.github.pagehelper.PageInfo;
import com.lonelyWolf.mybatis.mapper.UserMapper;
import com.lonelyWolf.mybatis.model.LwUser;
import org.apache.ibatis.executor.result.DefaultResultHandler;
import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.IOException;
import java.io.InputStream;
import java.util.List;

public class MyBatisByPageHelp {
    public static void main(String[] args) throws IOException {
        String resource = "mybatis-config.xml";
        
        InputStream inputStream = Resources.getResourceAsStream(resource);
        
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        
        SqlSession session = sqlSessionFactory.openSession();

        PageHelper.startPage(0,10);
        UserMapper userMapper = session.getMapper(UserMapper.class);
        List<LwUser> userList = userMapper.listAllUser();
        PageInfo<LwUser> pageList = new PageInfo<>(userList);
        System.out.println(null == pageList ? "": JSONObject.toJSONString(pageList));

    }
}
```

输出如下结果：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/93636bce-5e65-49a1-a20e-7ece52eb1087.png?raw=true)

可以看到对象已经被分页，那么这是如何做到的呢？

我们上面提到，要实现插件必须要实现 MyBatis 提供的 Interceptor 接口，所以我们去找一下，发现 PageHeler 实现了 Interceptor：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/8c28f13c-aa74-48bc-8e13-873b489d456d.png?raw=true)

经过上面的介绍这个类应该一眼就能看懂，我们关键要看看 SqlUtil 的 intercept 方法做了什么：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/ad43e7f5-f8a2-493e-ae4c-459d210ddd79.png?raw=true)

这个方法的逻辑比较多，因为要考虑到不同的数据库方言的问题，所以会有很多判断，我们主要是关注 PageHelper 在哪里改写了 sql 语句，上图中的红框就是改写了 sql 语句的地方：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/77c7511b-a0f8-44c8-9f79-4f15b3549e4b.png?raw=true)

这里面会获取到一个 Page 对象，然后在爱写 sql 的时候也会将一些分页参数设置到 Page 对象，我们看看 Page 对象是从哪里获取的：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/5b64a6bd-fb78-4bbe-b451-f4c654f3f1bd.png?raw=true)

我们看到对象是从 LOCAL_PAGE 对象中获取的，这个又是什么呢？  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/f2d94c7b-f477-4fb5-967f-8bafa7f5ec97.png?raw=true)

这是一个本地线程池变量，那么这里面的 Page 又是什么时候存进去的呢？这就要回到我们的示例上了，分页的开始必须要调用：

```null
PageHelper.startPage(0,10);
```

![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/316aaf74-a83f-42cb-83ba-74e893dfe844.png?raw=true)

这里就会构建一个 Page 对象，并设置到 ThreadLocal 内。

## 为什么 PageHelper 只对 startPage 后的第一条 select 语句有效

这个其实也很简单哈，但是可能会有人有这个以为，我们还是要回到上面的 intercept 方法：  
![](https://github.com/dandeliono/dandeliono.github.io/blob/master/source/_posts/resources/2021-8-27%2018-05-14/8fdf998b-0dc0-47f5-8afb-e595f755f15f.png?raw=true)

在 finally 内把 ThreadLocal 中的分页数据给清除掉了，所以只要执行一次查询语句就会清除分页信息，故而后面的 select 语句自然就无效了。 
 [https://developer.aliyun.com/article/780497](https://developer.aliyun.com/article/780497)
