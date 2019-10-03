# WEB开发

## 写在最开头，每次导入完依赖之后，一定要点maven那个刷新的按钮

## 1.自动配置原理

都在maven依赖中的`Autoconfig`的jar包下可以看见

xxxAutoConfiguration: 帮我们给容器中自动配置组件

xxxProperties: 配置类来封装配置文件的内容

## 2.SpringBoot对静态资源的映射规则

还是写在前面，我现在在`static`文件夹下放了`assets`文件夹，访问该文件夹下的css样式，**如果**是以下这样的方式的话

```html
<link rel="stylesheet" src="assets/css/signin.css">
```

那在重新跳转回该页面的时候就会访问失败，**应该是以下的访问方式**：

```html
<link rel="stylesheet" src="/assets/css/signin.css">
```

没错，就是多了个`/`我就找了半天的错...

[WebJars][https://www.webjars.org/]：对Bootstrap、JQuery封装成**Maven**或者**Gradle**的形式，方便项目打包成Jar。

1. 所有/webjars/\**，都去**classpath:/META-INF/resources/webjars/**找资源。例如，如果现在已经导入了JQuery的依赖，如果想访问jQuery的话可以通过`localhost:8080/webjars/jquery/(版本)/jquery.js`

```java
@ConfigurationProperties(
    prefix = "spring.resources",
    ignoreUnknownFields = false
)
public class ResourceProperties {
    //可以设置和静态资源相关的参数，缓存时间等
```

```java
//webmvcproperties.class
public void addResourceHandlers(ResourceHandlerRegistry registry) {
            if (!this.resourceProperties.isAddMappings()) {
                logger.debug("Default resource handling disabled");
            } else {
                Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
                CacheControl cacheControl = this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl();
                if (!registry.hasMappingForPattern("/webjars/**")) {
                    this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{"/webjars/**"}).addResourceLocations(new String[]{"classpath:/META-INF/resources/webjars/"}).setCachePeriod(this.getSeconds(cachePeriod)).setCacheControl(cacheControl));
                }

                String staticPathPattern = this.mvcProperties.getStaticPathPattern();
                if (!registry.hasMappingForPattern(staticPathPattern)) {
                    this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{staticPathPattern}).addResourceLocations(WebMvcAutoConfiguration.getResourceLocations(this.resourceProperties.getStaticLocations())).setCachePeriod(this.getSeconds(cachePeriod)).setCacheControl(cacheControl));
                }

            }
        }
```

2. /**：访问当前项目的任何资源（静态资源文件夹）

```
"classpath:/META-INF/resources/", "classpath:/resources/", "classpath:/static/", 
"classpath:/public/"
"/" : 当前项目的根路径
```

3. 欢迎页都是自动寻找静态文件夹下的`index.html`文件
4. 所有的"/**/favaicon.ico"都是在静态文件夹下

## 3.模板引擎(thymeleaf)

#### 1.引入thymeleaf

```xml
<!-- https://mvnrepository.com/artifact/org.thymeleaf/thymeleaf -->
<dependency>
    <groupId>org.thymeleaf</groupId>
    <artifactId>thymeleaf</artifactId>
    <version>3.0.11.RELEASE</version>
</dependency>
```

```html
<html xmlns:th="http://www.thymeleaf.org">
```

#### 2.使用Getmapping和Controller

[点击跳转到对应的文档页面][https://spring.io/guides/gs/serving-web-content/]

```xml
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>
<dependency>
	 <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

这里的`thymeleaf`不能加上版本号，不然`IDEA`启动的时候会报错

#### 3.表达式语法

[Thymeleaf官方文档][https://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf.html]：见第四条

```yaml
Simple expressions:
	Variable Expressions: ${...}   获取变量值，OGNL表达式
		1.获取属性，调用方法 
		2.使用内置对象
		3.内置得到工具对象
	Selection Variable Expressions: *{...}  	选择表达式，和${}差不多，配合<th:object>使用
	Message Expressions: #{...}   	获取国际核内容
	Link URL Expressions: @{...}
	Fragment Expressions: ~{...}
Literals
	Text literals: 'one text', 'Another one!',…
	Number literals: 0, 34, 3.0, 12.3,…
	Boolean literals: true, false
	Null literal: null
	Literal tokens: one, sometext, main,…
Text operations:
	String concatenation: +
	Literal substitutions: |The name is ${name}|
	Arithmetic operations:
	Binary operators: +, -, *, /, %
	Minus sign (unary operator): -
Boolean operations:
	Binary operators: and, or
	Boolean negation (unary operator): !, not
	Comparisons and equality:
	Comparators: >, <, >=, <= (gt, lt, ge, le)
	Equality operators: ==, != (eq, ne)
Conditional operators:
	If-then: (if) ? (then)
	If-then-else: (if) ? (then) : (else)
	Default: (value) ?: (defaultvalue)
Special tokens:
	No-Operation: _
```

## 4.SpringMVC自动配置

1. 自动配置了`viewresolver(视图解析器)`根据**返回值**得到视图对象，决定视图对象如何渲染(转发、重定向等等)
   - ContentNegotiatingViewResolver：组合所有视图解析器
   - 可以自定义一个视图解析器，然后ContentNegotiatingViewResolver会自动的组合自定义的视图解析器，在主类中写自定义的视图解析器。自己给**容器**添加一个视图解析器，自动的组合进来。添加到容器中就是**@Bean**
2. 自动注册了**转换器**：方法返回值是`string`类型的，转换器可以自动的将类型转换成相应的类型。自动注册了**格式化器**：例如日期转换器。自己添加的格式化器和转换器，只需要添加在**容器**中就可以用
3. HttpMessageConverters：SpringMVC用来转换Http请求和响应的。HttpMessageConverters要从容器中确定。所以自己添加HttpMessageConverters，就加到容器中，即加**@Bean**和**@Component**注解即可。

#### 1.如何修改SpringBoot的默认配置

模式：

1. SpringBoot在自动配置**很多**组件的时候，先看容器中有没有用户自己配置得到(@Bean @Component)，如果有的话，就直接用用户自己配置的，没有的话就自动配置，有些组件可以将用户配置的和自动配置相互结合。
2. xxxConfigurer进行扩展配置

#### 2.扩展SpringMVC

编写一个配置(@Configuration)，继承WebMvcConfigAdapter，通过重写方法来使用相应功能，但是不能标注EnableWebMvc

原理：

1. WebMvcAutoConfiguration是SpringMVC的自动配置类
2. 在做其他自动配置时会导入，@Import(**EnableWebMvcConfiguration**.class)
3. 容器中所有的WebMvcAutoConfiguration都会一起起作用，自定义的配置类也会被调用。

#### 3.全面接管SprIngMvc

只用要在配置类中添加`@EnableWebMvc`，此时所有的springboot对springmvc的自动配置全部失效

## 5.国际化

1. 在resources文件夹下创建一个包`package(名称随便)`，然后创建`xxx.properties`,xxx是网页的名称，以及创建`xxx_en_US或者xxx_zh_CN.peoperties`文件

2. 要在主设置文件中设置`spring.messages.basename=package.xxx`

# SpringBoot和MyBatis整合

## 1.Maven依赖导入

```xml
<!-- https://mvnrepository.com/artifact/org.mybatis.spring.boot/mybatis-spring-boot-starter -->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.1.0</version>
        </dependency>
```

## 2.Mybatis配置

```properties
# 取消thmeleaf的网页缓存，实时预览网页效果
spring.thymeleaf.cache=false

# MyBatis的配置
spring.datasource.url=jdbc:mysql://localhost:3306/springboot?useSSL=false
spring.datasource.username=root
spring.datasource.password=qq18717128499
spring.datasource.driver-class-name=com.mysql.jdbc.Driver

# 开启数据库和实体层的驼峰命名规则
mybatis.configuration.map-underscore-to-camel-case=true
```

中间的那一部分是配置连接的部分，最下面的是配置**变量命名规则对应**，数据库一般采用下划线分割的方法，而变量采用驼峰命名规则方法，这样才能对应起来，确认正常使用。

## 3.Mybatis使用

新建一个持久层**Mapper**

直接贴上写的代码

```java
package com.study.crud.Mapper;

import com.study.crud.Enities.Question;
import org.apache.ibatis.annotations.*;

import java.util.List;

@Mapper
public interface QuestionMappper {
//    private Integer id;
//    private String title;
//    private String description;
//    private String tag;
//    private Long gmtCreate;
//    private Long gmtModified;
//    private Integer creator;
//    private Integer viewCount;
//    private Integer likeCount;
//    private Integer commentCount;
    @Select("select * from question")
    List<Question> findAll();

    @Delete("delete from question where id=#{id}")
    boolean del(@Param(value = "id") Integer id);

    @Update("update question set description=#{description},title=#{title},gmt_create=#{gmtCreate},gmt_modified=#{gmtModified} where id=#{id}")
    boolean edit(Question question);

    @Select("select * from question where id=#{id}")
    Question findByID(@Param(value = "id") Integer id);

    @Insert("insert into question(description,title,gmt_create,gmt_modified) values(#{description},#{title},#{gmtCreate},#{gmtModified})")
    boolean add(Question question);
}
```

中间的注释部分是**实体层**的变量。

## 4.Mybatis注解进行一对多和多对多查询

### 1.实体类：

```java
//	Role类
package com.example.springsecuritydb.Enity;

import lombok.Data;

@Data
public class Role {
    private Integer id;
    private String name;
    private String nameZh;
}
```



```java
//	Menu类
package com.example.springsecuritydb.Enity;

import lombok.Data;

import java.util.List;

@Data
public class Menu {
    private Integer id;
    private String pattern;
    private List<Role> roles;
}
```

### 2.对应的Mapper

```java
//	RoleMapper
package com.example.springsecuritydb.Mapper;

import com.example.springsecuritydb.Enity.Role;
import org.apache.ibatis.annotations.*;

import java.util.List;

@Mapper
public interface RoleMapper {

    @Results({
            @Result(property = "name",column = "name"),
            @Result(property = "nameZh",column = "nameZh")
    })
    @Select("select * from role where id= #{id}")
    List<Role> getById(@Param("id") Integer id);
}
```



```java
//	MenuMapper
package com.example.springsecuritydb.Mapper;

import com.example.springsecuritydb.Enity.Menu;
import org.apache.ibatis.annotations.*;

import java.util.List;

@Mapper
public interface MenuMapper {

    @Select("select * from menu")
    @Results({
            @Result(property = "id",column = "id"),
            @Result(property = "pattern",column = "pattern"),
            @Result(property = "roles",column = "id",many = @Many(select ="com.example.springsecuritydb.Mapper.RoleMapper.getById"))
    })
    List<Menu> getAllMenus();
}
```

### 3.这里就说多对多的问题

所谓的多对多就是，一个实体类中含有另一个表的信息使用`@Many`来调用另一个`Mapper`的查询方法进行连接查询，得到结果，其中`property`是实体类中的属性，`column`是表中对应的列，也是传参所需的参数。

## 5.LomBok插件使用(这个是真的好用)

### 1.引入Lombok依赖

```xml
		<dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.8</version>
            <scope>provided</scope>
        </dependency>
```

### 2.使用以及作用

直接在实体类前面添加注解`@Data`，**作用**就是免去实体类中的`getter()`和`setter()`，让代码变得简洁多了。

# 使用SpringBoot整合其他框架发现的问题记录

## 1.设置视图和页面的映射

```java
public class MyMvcConfig extends WebMvcConfigurationSupport {
    @Override
    protected void addViewControllers(ViewControllerRegistry registry) {
        super.addViewControllers(registry);
        registry.addViewController("/main.html").setViewName("question");
        registry.addViewController("/").setViewName("login");
        registry.addViewController("/index.html").setViewName("login");
//        registry.addRedirectViewController("/edit/success","question");
        registry.addViewController("/toadd").setViewName("add");
    }
}
```

这样就可以使用`xxx.html`来访问对应的视图，**不过**使用这个的时候，会把静态资源给拦截了，无论是本地的静态资源还是使用`webjars`的静态资源，都会被拦截，那有什么解决办法呢？

## 2.解决静态资源被拦截的问题

```java
	@Override
    protected void addResourceHandlers(ResourceHandlerRegistry registry) {
        super.addResourceHandlers(registry);
        registry.addResourceHandler("/**").addResourceLocations("classpath:/static/");
        registry.addResourceHandler("/webjars/**").addResourceLocations("/webjars/");
    }
```

就是在刚才的`MyMvcConfig.java`文件中重写`addResourceHandlers`，最后的`addResourceLocations`就是静态文件资源所在路径，有了这两个，那么就可以写登陆页面了。

## 3.登陆页面遇到的一些问题

由于上面设置了`/`为`login`视图的映射，所以首页就是`login`页面，`login`页面的表单提交之后会发送一个请求到`/user/login`。

当时在这里遇见了一个很奇怪的问题，那就是：页面表单如果提交的是这个请求的话，那么静态资源文件不能在页面上显示出来，但是我可以通过浏览器地址栏来访问对应的静态文件，之后的解决办法就是：我发现我在导入静态文件的时候

```html
<link rel="stylesheet" th:href="@{webjars/bootstrap/4.3.1/css/bootstrap.css}">
```

写成了上面的样子，乍一看没有什么错误，但是仔细一看，`webjars`前面少了一个`/`，**下面的**才是**正确**的，**上面的**是**错误**的

```html
<link rel="stylesheet" th:href="@{/webjars/bootstrap/4.3.1/css/bootstrap.css}">
```

这样我的登陆页面静态资源无效的问题就解决了。

***

## 4.有待更改的地方

但是说到静态资源文件失效我**现在**还碰到了一个很**古怪的问题**，就是：前一天写的代码，关掉IDEA没有动了，第二天打开IDEA，启动项目，发现静态资源文件又失效了，**要先登陆**进`/question`页面，下一次项目重启的话就是好的，我i现在在怀疑的是，`Thymeleaf`的缓存是否应该关闭，我在**主配置文件**中写了：

```properties
# 取消thmeleaf的网页缓存，实时预览网页效果
spring.thymeleaf.cache=false
```

我在怀疑这个问题，这个地方先留着吧，有待更改。

---

## 5.页面调用删除的方法

```html
<td><a th:href="${'/del/'+question.id}">删除</a></td>
```

使用`thymeleaf`提供的`${'/del/'+question.id}`方法，将ID写入地址栏，方便后台获取数据，那后台是如何获取数据的呢？

```java
	@GetMapping("/del/{id}")
    public String deleteQuestion(@PathVariable(value = "id") Integer id,
                                 HttpSession session){
        boolean flag = questionMappper.del(id);
        if(flag){
            session.setAttribute("msg","删除成功");
        }
        return "redirect:/question?msg";
    }
```

就是使用`PathVariable`的方法从地址栏中获取参数。说到这里又引出了下一个问题。

## 6.重定向时后台怎么给前台传递参数(可能有更好的解决办法)

就是使用`session`对象通过`setAttribute`的方法传递参数，使用方法就和`Model`传递参数一样，但是，我觉得这个方法肯定不是最优解，`session`对象是在服务器启动起来，到服务器断开一直都是存在的，如果单单只是在重定向的时候传递参数，我觉得有点**杀鸡用牛刀**的感觉。`session`对象，我在登陆的时候也使用了，所以下一个问题**又双叒叕**的来了。

## 7.判断是否登陆的问题

如果**没有特别设置**的话**没有登陆**和**已经登陆**的状态都是一样的，都可以随意访问系统的任何页面，所以就必须设置**拦截器**，来拦截没有登陆的情况的请求。下面就是自己定义的拦截器：

```java

package com.study.crud.Config;

import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class LoginHandlerInterCeptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        Object user = request.getSession().getAttribute("username");
        if(user==null){
            request.setAttribute("msg","没有权限，请先登陆");
            request.getRequestDispatcher("/index.html").forward(request,response);
            return false;
        }else {
            return true;
        }
    }
}
```

在登陆的时候使用`session`对象：首先判断用户名和密码是否匹配，如果匹配的话就给`session`设置用户名，然后在拦截器中获取`session`对象，判断是否为空，如果不为空就不拦截，如果是空的就拦截。

但是只有这个是不行的，还必须在`MyMvcConfig`中注册拦截器

```java
public class MyMvcConfig extends WebMvcConfigurationSupport {
    @Override
    protected void addInterceptors(InterceptorRegistry registry) {
        super.addInterceptors(registry);
        registry.addInterceptor(new LoginHandlerInterCeptor()).addPathPatterns("/**")
                .excludePathPatterns("/index.html","/","/user/login");
    }
}
```

## 8.编辑和添加前台多参数怎么传回后台

知道很少的参数的话就在前台设置控件的`name`属性，后台使用类似`@RequestParam(name = "username") String username`这种方式来获取参数，但是**参数多了**之后呢？**SpringMVC**将前台的对应表单所有元素的`name`属性和后台的对象对比，如果后台参数(对象类型)有前台**所有**的`name`属性，就会自动传值，当然**属性名称要相匹配**。举个例子：

```java
	@GetMapping("/add")
    public String addQuestion(Question question){
        questionMappper.add(question);
        return "redirect:/question";
    }
```

上面的是后台的，下面的是前台的表单元素：

```html
<div class="mt-3 mb-5">
    <div class="container">
        <form action="/add">
            <div class="form-group row">
                <label for="Title">Title</label>
                <input type="text" class="form-control mr-sm-2 mb-2 mb-sm-0" id="Title" name="title">
            </div>
            <div class="form-group row">
                <label for="Description">Description</label>
                <input type="text" class="form-control mr-sm-2 mb-2 mb-sm-0" id="Description" name="description">
            </div>
            <div class="form-group row">
                <label for="gmtModified">gmtModified</label>
                <input type="text" class="form-control mr-sm-2 mb-2 mb-sm-0" id="gmtModified" name="gmtModified">
            </div>
            <div class="form-group row">
                <label for="gmtCreate">gmtCreate</label>
                <input type="text" class="form-control mr-sm-2 mb-2 mb-sm-0" id="gmtCreate" name="gmtCreate"><br>
            </div>
            <div class="form-group row">
                <input type="submit" class="btn btn-primary" value="确认添加">
            </div>
        </form>
    </div>
</div>
```

看name属性，如果后台的对象参数有全部name属性的话就可以一次性传递，不需要使用很复杂的方法。



# 前后端分离的学习

## 1.@Responsebody和@RestController的作用

对`json`格式的数据的响应。