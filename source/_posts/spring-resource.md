---
title: spring-resource
date: 2020-09-24 21:56:07
tags: spring
categories: 编程
---
Spring不但支持自己定义的\@Autowired注解，还支持几个由**JSR-250**规范定义的注解，它们分别是\@Resource、\@PostConstruct以及\@PreDestroy。这里只说\@Autowired和\@Resource注解的区别。

1. \@Autowired与\@Resource都可以用来装配bean. 都可以写在字段上,或写在setter方法上。

2. @Autowired默认按类型装配（这个注解是属于spring的），默认情况下必须要求依赖对象必须存在，如果要允许null值，可以设置它的required属性为false，如：\@Autowired\(required=false\) ，如果我们想使用名称装配可以结合\@Qualifier注解进行使用，如下：

```java
@Autowired()@Qualifier("baseDao")privateBaseDao baseDao;
```

3. \@Resource（这个注解属于J2EE的），默认按照名称进行装配，名称可以通过name属性进行指定，如果没有指定name属性，当注解写在字段上时，默认取字段名进行安装名称查找，如果注解写在setter方法上默认取属性名进行装配。当找不到与名称匹配的bean时才按照类型进行装配。但是需要注意的是，如果name属性一旦指定，就只会按照名称进行装配。

```java
@Resource(name="baseDao")privateBaseDao baseDao;
```

**\@Resource装配顺序**  
1. 如果同时指定了name和type，则从Spring上下文中找到唯一匹配的bean进行装配，找不到则抛出异常  
2. 如果指定了name，则从上下文中查找名称（id）匹配的bean进行装配，找不到则抛出异常  
3. 如果指定了type，则从上下文中找到类型匹配的唯一bean进行装配，找不到或者找到多个，都会抛出异常  
4. 如果既没有指定name，又没有指定type，则自动按照byName方式进行装配；如果没有匹配，则回退为一个原始类型进行匹配，如果匹配则自动装配；

**\@Autowired装配顺序**

1. 如果没有指定\@Qualifier，默认安装类型注入，找不到则抛出异常。  
2. 如果指定了\@Qualifier，则按照名称注入，找不到则抛出异常。

一般\@Autowired和\@Qualifier一起用，\@Resource单独用。当然没有冲突的话\@Autowired也可以单独用。值得注意的是Spring官方并不建议直接在类的field上使用\@Autowired注解。参考[《Why field injection is evil》](http://olivergierke.de/2013/11/why-field-injection-is-evil/)