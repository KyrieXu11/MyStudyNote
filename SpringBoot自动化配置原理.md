# SpringBoot自动化配置原理

# 写这篇博客的来由

网上很多都是互相抄来抄去的，并且排版做的也不太行，看的头皮发麻，还不如自己过一遍，废话不多说了，开始吧。

# 程序的入口

自动化配置的入口，是从`@SpringBootApplication`的注解开始的。如果不想下载源码的话，就新建一个`springboot`的项目，写一个主类

![image.png](https://i.loli.net/2020/05/21/lvg4JMbxwCU5DcY.png)

点进`@SpringBootApplication`之后，我们可以看到下面的代码，注解很多，但是我们需要看的就是`@EnableAutoConfiguration`

![2.png](https://i.loli.net/2020/05/21/UDp2oxMBQ9YvGEX.png)

再次点进`@EnableAutoConfiguration`，这里说一下这个注解有什么用，文档是这样写的：

> ```
> Enable auto-configuration of the Spring Application Context, attempting to guess and
> configure beans that you are likely to need. Auto-configuration classes are usually
> applied based on your classpath and what beans you have defined. For example, if you
> have {@code tomcat-embedded.jar} on your classpath you are likely to want a
> {@link TomcatServletWebServerFactory} (unless you have defined your own
> {@link ServletWebServerFactory} bean).
> 
> When using {@link SpringBootApplication @SpringBootApplication}, the auto-configuration of the context is automatically enabled and adding this annotation has therefore no additional effect.
> ```

上面的大概的意思就是，当你开启了自动配置的时候，spring就会猜你想要使用的哪个类。自动配置的类是基于你的`classpath`和你定义了哪些`bean`来自动化配置的。如果用了`@SpringBootApplication`注解再用这个注解没有一点卵用。ㄟ( ▔, ▔ )ㄏ

继续上面的说下去，点进`@EnableAutoConfiguration`就会看见下面的代码

![4.png](https://i.loli.net/2020/05/21/1ZTsdzA5nMbLPwI.png)

这个`@Import`注解有几个作用：

+ 声明一个bean
+ 导入`configuration`类
+ 导入`ImportSelector`的实现类
+ 导入`ImportBeanDefinitionRegistrar`的实现类

这里就是导入`ImportSelector`的实现类，所以我们点进`AutoConfigurationImportSelector`。

![autoconfigurationselector.png](https://i.loli.net/2020/05/21/YpnVlAzRShkt9Fs.png)

我们可以看到，核心的方法就是`override`的`selectImports`，所以我们重点来看看这个`selectImports`。

所在的包是：`org.springframework.boot.autoconfigure`

## loadMetadata

```java
AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
```

上面的就是加载元数据，所以我们要看看怎么加载的，点进`loadMetadata`，是在`AutoConfigurationMetadataLoader.java`中的

```java
static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader) {
		return loadMetadata(classLoader, PATH);
}
```

我们首先看看`PATH`是什么

```java
protected static final String PATH = "META-INF/spring-autoconfigure-metadata.properties";
```

`PATH`是在`META-INF`下的一个文件

![metadata_properties.png](https://i.loli.net/2020/05/21/byx8mrU2OXIC6eW.png)

我们打开这个文件看一下

![autoconfiguration_metadata_properties.png](https://i.loli.net/2020/05/21/2IGicRvFATLnb1E.png)

基本上都是`xxxxAutoConfiguration`类结尾，所以上面的`loadMetadata`是加载一些自动化配置的类，继续返回到`AutoConfigurationImportSelector.java`

## getAutoConfigurationEntry

```java
AutoConfigurationEntry autoConfigurationEntry=getAutoConfigurationEntry(autoConfigurationMetadata,
      annotationMetadata);
```

接着上面的继续看，上面方法的参数有`autoConfigurationMetadata`这个就是上面加载的元数据，点进`getAutoConfigurationEntry`，看到下面这个方法，这个方法还是在``AutoConfigurationImportSelector.java`中

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
			AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return EMPTY_ENTRY;
		}
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
		List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
		configurations = removeDuplicates(configurations);
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		checkExcludedClasses(configurations, exclusions);
		configurations.removeAll(exclusions);
		configurations = filter(configurations, autoConfigurationMetadata);
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return new AutoConfigurationEntry(configurations, exclusions);
	}
```

一行一行的来看，第一个方法就是

### getAttributes

```java
AnnotationAttributes attributes = getAttributes(annotationMetadata);
```

这个就是从元数据中获取属性，点进这个方法看

```java
protected AnnotationAttributes getAttributes(AnnotationMetadata metadata) {
		String name = getAnnotationClass().getName();
		AnnotationAttributes attributes = AnnotationAttributes.fromMap(metadata.getAnnotationAttributes(name, true));
		Assert.notNull(attributes, () -> "No auto-configuration attributes found. Is " + metadata.getClassName()
				+ " annotated with " + ClassUtils.getShortName(name) + "?");
		return attributes;
	}
```

1. 获取注解类的名称
2. 从注解类的属性中获取属性

### getCandidateConfigurations

```java
List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
```

文档说的是

> Return the auto-configuration class names that should be considered

就是这个方法返回自动配置的类的名称，点进这个方法

![getCandidateConfiguration.png](https://i.loli.net/2020/05/21/Np3vxgR2lZukJ48.png)