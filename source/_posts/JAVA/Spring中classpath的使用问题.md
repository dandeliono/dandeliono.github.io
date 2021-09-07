# Spring中classpath的使用问题 
在 Spring 的配置文件中，经常使用**classpath：xxx.xxx**来读取文件。对于 maven 项目，误区是读取的文件必须在**resources**下，其实并不是。

## [](#实验 "实验")实验

对于 maven 管理的项目，我们分别从三个例子分析，读取的文件位置可以在什么地方。

配置文件中加入以下配置

```xml
<bean id="txt" class="org.springframework.beans.factory.config.PropertiesFactoryBean">
	<property name="location" value="classpath:/txt/readme.txt"/>
</bean>
```

测试代码

```java
@Resource(name="txt")
protected Properties Properties;
```

### [](#第一种情况（resources下） "第一种情况（resources 下）")第一种情况（resources 下）

在 resources 下，新建文件夹 txt，在 txt 中放入 readme.txt，执行测试代码可以正常输出 txt 中的 readme.txt。

### [](#第二种情况（jar包中） "第二种情况（jar 包中）")第二种情况（jar 包中）

新建项目，文件夹 txt 中放入 readme.txt，打包生成. jar 文件，在测试项目中引入生成的 jar 包，执行测试代码获取到 jar 包中 readme.txt。

### [](#第三种情况（resouces和jar中都存在） "第三种情况（resouces 和 jar 中都存在）")第三种情况（resouces 和 jar 中都存在）

当 resources 和 jar 包中都有这个文件的时候，默认只读取 resources 下的资源文件。

## [](#后记-12月4日补充 "后记 (12 月 4 日补充)")后记 (12 月 4 日补充)

对于不同项目来说，其 classpath 不同，上述结论不适用于所有项目。

### [](#Resource "Resource")Resource

上述实验用到的**PropertiesFactoryBean**返回**java.util.Properties**，依赖注入时用到的**setlocation(Resource location)**，所以我们需要对**Resource**进行简单的研究。

### [](#Resource接口 "Resource 接口")Resource 接口

Spring 的 Resource 接口抽象了对低级资源文件的访问，接口定义如下

```java
public interface Resource extends InputStreamSource {

    boolean exists();

    boolean isOpen();

    URL getURL() throws IOException;

    File getFile() throws IOException;

    Resource createRelative(String relativePath) throws IOException;

    String getFilename();

    String getDescription();

}
```

这个接口继承于 InputStreamSource 接口，接口定义如下

```java
public interface InputStreamSource {
    InputStream getInputStream() throws IOException;
}
```

Resource 接口中的一些重要的方法如下：

-   **getInputStream()**：定位和打开资源文件，从资源文件中返回**InputStream**
-   **exists()**：返回**boolean**，表示资源是否存在
-   **isOpen()**：返回**boolean**，如果是**true**，资源就只能被访问一次，且需要被关闭。对于其一般的实现类，返回的是**false**，除了**InputStreamResource**
-   **getDescription()**：返回对这个资源文件的描述

其他的方法可以获得一个真正的**URL**对象或者**File**对象，代表资源文件。  
在 Spring 框架中，当一个资源文件被需要时，会大量地使用**Resource**接口作为参数类型。当然在我们自己的代码中，也可以使用它的实现类获取文件资源，是**URL**更有用的代替。

### [](#内置的Resource实现类 "内置的 Resource 实现类")内置的 Resource 实现类

-   **UrlResource**：包含了**java.net.URL**，可以用 URL 访问对象，如**Files**，**Http**，**Ftp**，一些前缀被使用，用来表名 URL 的类型，如**file:**，**http&#x3A;**，**ftp:**
-   **ClassPathResource**：这个类代表了一个应该从**classpath**获取的资源文件，使用**classloader**，或者**class**来载入资源
-   FileSystemResource
-   ServletContextResource
-   InputStreamResource
-   ByteArrayResource

前两个实现类都可以直接用构造器显式创建，不过大部分情况下，是用 string 表示路径来调用 api 方法（bean 依赖注入），隐式创建，如果前缀是**classpath**，则创建**ClassPathResource**。

### [](#ClassPathResource "ClassPathResource")ClassPathResource

现在可以知道上述实验是生成了**ClassPathResource**类，再看源码（代码中所用到的 path 为去掉 classpath: 前缀的路径）

```java
public ClassPathResource(String path){
	this(path, (ClassLoader)null);
}
```

```java
public ClassPathResource(String path, ClassLoader classLoader) {
	Assert.notNull(path, "Path must not be null");
	String pathToUse = StringUtils.cleanPath(path);
	if (pathToUse.startsWith("/")) {
	  pathToUse = pathToUse.substring(1);
	}
	this.path = pathToUse;
	this.classLoader = (classLoader != null ? classLoader : ClassUtils.getDefaultClassLoader());
}
```

```java
public InputStream getInputStream() throws IOException {
    InputStream is;
    if (this.clazz != null) {
      is = this.clazz.getResourceAsStream(this.path);
    }
    else{
      if (this.classLoader != null) {
        is = this.classLoader.getResourceAsStream(this.path);
      } else {
        is = ClassLoader.getSystemResourceAsStream(this.path);
      }
    }
    if (is == null) {
      throw new FileNotFoundException(getDescription() + " cannot be opened because it does not exist");
    }
    return is;
}
```

最终调用**java.lang.ClassLoader**类中的**getResource**方法，在 api 中关于查找顺序，有以下的描述

> This method will first search the parent class loader for the resource; if the parent is null the path of the class loader built-in to the virtual machine is searched. That failing, this method will invoke findResource(String) to find the resource.

此方法首先搜索资源的父类加载器；如果父类加载器为 null，则搜索的路径就是虚拟机的内置类加载器的路径。如果搜索失败，则此方法将调用 findResource(String) 来查找资源。
