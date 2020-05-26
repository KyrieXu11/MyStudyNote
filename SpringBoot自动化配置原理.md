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

基本上都是`xxxxAutoConfiguration`类结尾，所以上面的`loadMetadata`是加载一些自动化配置的类的入口，继续返回到`AutoConfigurationImportSelector.java`

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

```java
String name = getAnnotationClass().getName();
```

上面的语句等价于

```java
EnableAutoConfiguration.class.getName()
```

所以`name`这个变量就是`EnableAutoConfiguration`

我们接着看下去：

下面的这个方法，就是从给定的`Map`中获取属性

```java
AnnotationAttributes.fromMap()
```

官方的文档是这样写的：

> * Return an {@link AnnotationAttributes} instance based on the given map.
>
> * <p>If the map is already an {@code AnnotationAttributes} instance, it
>
> * will be cast and returned immediately without creating a new instance.
>
> * Otherwise a new instance will be created by passing the supplied map
>
> * to the {@link #AnnotationAttributes(Map)} constructor.
>
> * @param map original source of annotation attribute <em>key-value</em> pairs

大概的意思就是说：基于给定`map`返回一个`AnnotationAttributes`，如果`map`已经是一个这个类的实例了，那么这个会被强制转换并且返回，否则创建新的实例。

具体的看看这个方法的源码就会很容易弄懂：

```java
@Nullable
public static AnnotationAttributes fromMap(@Nullable Map<String, Object> map) {
   if (map == null) {
      return null;
   }
   if (map instanceof AnnotationAttributes) {
      return (AnnotationAttributes) map;
   }
   return new AnnotationAttributes(map);
}
```

为什么一个`map`会是一个这个类的实例呢？

```java
public class AnnotationAttributes extends LinkedHashMap<String, Object>
```

这个类继承自` LinkedHashMap<String, Object>`

接着看括号里面的参数

```java
metadata.getAnnotationAttributes(name, true)
```

这个方法很明显返回的是一个`Map<String,Object>`类型的，这个方法实现的功能大概是：从给定的注解取出属性。

这个方法有两个参数，第一个参数是：类名称。第二个参数是：是否将取出来的属性转换成`String`类型的。

总之就是从注解中获取属性，然后返回。

### getCandidateConfigurations

方法还是位于`AutoConfigurationImportSelector.java`

```java
List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
```

文档说的是

> Return the auto-configuration class names that should be considered

就是这个方法返回自动配置的类的名称，点进这个方法

![getCandidateConfiguration.png](https://i.loli.net/2020/05/21/Np3vxgR2lZukJ48.png)

方法有两个参数，第一个参数是：注解的元数据，就是`selectImports`中的第二个参数。而第二个参数呢，就是上面获取的属性。

#### loadFactoryNames

上面有一个方法：`loadFactoryNames`，这个方法的作用就是用给定的类加载器，加载类，而继续往下看会得到一个关键的信息，这个方法的返回值是

```java
return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
```

先说一个`getOrDefault`这个方法的作用吧。这个是`Map`提供的`api`，作用就是首先在`map`中找第一个参数对应的值，如果没找到，那么返回的不是`null`，而是后面第二个参数。

继续往下看，`loadSpringFactories`，这个方法会有一个很关键的信息

```java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
   MultiValueMap<String, String> result = cache.get(classLoader);
   if (result != null) {
      return result;
   }

   try {
      Enumeration<URL> urls = (classLoader != null ?
            classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
            ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
      result = new LinkedMultiValueMap<>();
      while (urls.hasMoreElements()) {
         URL url = urls.nextElement();
         UrlResource resource = new UrlResource(url);
         Properties properties = PropertiesLoaderUtils.loadProperties(resource);
         for (Map.Entry<?, ?> entry : properties.entrySet()) {
            String factoryTypeName = ((String) entry.getKey()).trim();
            for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
               result.add(factoryTypeName, factoryImplementationName.trim());
            }
         }
      }
      cache.put(classLoader, result);
      return result;
   }
```

省去异常，上面就是核心代码了。说一下执行的流程：

1. 首先从缓存中取

缓存的代码是：

```java
private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();
```

使用了一个`ConcurrentHashMap`来做缓存，键值对的关系是`String`->` MultiValueMap<String, String>`的映射，` MultiValueMap<String, String>`我们其实也可以把其理解为一个`Map`，他本身也是一个接口，继承；；了`Map`接口。

2. 获取资源

```java
Enumeration<URL> urls = (classLoader != null ?
      classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
      ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
```

其中`FACTORIES_RESOURCE_LOCATION`就是

```java
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
```

看看这个文件，里面

![image.png](https://i.loli.net/2020/05/26/8XmpVfQYh9RniZk.png)

`spring.factories`就像是工厂一样配置了大量的接口对应的实现类，我们通过这些配置 + 反射处理就可以拿到相应的实现类。这种类似于插件式的设计方式，只要引入对应的jar包，那么对应的`spring.factories`就会被扫描到，对应的实现类也就会被实例化，如果不需要的时候，直接把jar包移除即可。

```java
Properties properties = PropertiesLoaderUtils.loadProperties(resource);
```

这条语句就是将上面的`spring.factories`给填充到内存的`properties`对象中去

```java
StringUtils.commaDelimitedListToStringArray()
```

上面这个方法就是将后面的用逗号隔开的类名转换成数组

`loadSpringFactories`这个方法返回的就是上面的`factories`文件的键值对，而正是这些键值对，才能告诉spring，这些自动化配置类的名称是什么，才能自动化配置。

# 自己写一个Starter

自己写一个`starter`才能更好的理解上面的`spring.factories`文件的作用了。

## 命名规范

自己写的starter命名应该是：`xxx-spring-boot-starter`

spring官方写的starter的命名是：`spring-boot--starter-xxx`

## 写入依赖

创建一个`maven`项目，在`pom.xml`中加入下面的依赖(**不用创建springboot项目**)

```xml
	<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
            <version>${spring.boot.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
            <version>${spring.boot.version}</version>
	</dependency>
```

上面的`spring.boot.version`就是你的`springboot`版本，需要在`<properties>`标签中指定

## 创建配置类

配置类就是将配置文件的属性注入到`java`代码中的类。

`@ConfigurationProperties`这个注解就是将配置文件的值注入到代码中，其中必须指定`prefix`也就是配置文件中的前缀

![starter-1.png](https://i.loli.net/2020/05/26/6kMjfroQVUvsAJE.png)

## 写配置文件

在`resources`文件目录下新建一个`application.yml`或者`application.properties`文件，写入下面的配置，注意前缀需要和上面代码的前缀一致

```properties
xu.firstName=kyrie
xu.lastName=xu
xu.age=13
```

## 写对外提供的一些类

![starter-2.png](https://i.loli.net/2020/05/26/Gc1QbTsMdz8vEuS.png)

这里为了简便起见，我就写了一个`hello`方法，三个和配置文件一样的属性，`hello`方法其实就是`toString`方法，注意，**这个类不需要加任何的注解**

## 写自动配置类

![starter-3.png](https://i.loli.net/2020/05/26/B65kc7wbULamlVx.png)

这个类有几点好说的：

1. `@EnableConfigurationProperties`是开启配置文件属性注入，需要指定哪些类是配置文件注入的
2. 要标示为配置类，所以得加上`@Configuration`
3. `@ConditionalOnClass`是只有当后面指定的`xxx.class`在项目中的`classpath`才会生效，不然不生效
4. `@ConditionalOnMissingBean`是表示，只有这个`bean`没有自定义的时候，就用预先配置好的`bean`，有用户在配置的同名`bean`的时候使用用户配置的`bean`

现在还没看见`spring.factories`文件？别着急，下一步就是的了。

## 编写spring.factories文件

在`resources`文件目录下新建一个`META-INF`的文件夹，在该文件夹下新建一个`spring.factories`文件，里面指定自动配置类的类名是什么，配置如下：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.kyriexu.HelloServiceAutoConfiguration
```

## 打包或者上传到maven私服

![image.png](https://i.loli.net/2020/05/26/dWvotiPnJ4jGmg6.png)

这里为了简便起见，我直接安装到本地的maven仓库，不影响结果

## 测试

导入测试相关的依赖包

编写测试的代码

![starter-4.png](https://i.loli.net/2020/05/26/mCOdF4bATGqtQs2.png)

运行效果：

![image.png](https://i.loli.net/2020/05/26/IPXbf9DlkhC67TM.png)

当然我们也可以在项目的配置文件中重新配置，如下：

![starter-5.png](https://i.loli.net/2020/05/26/V6AetWTfa3zIZH1.png)

运行的结果也会变化，我这就不写了。

