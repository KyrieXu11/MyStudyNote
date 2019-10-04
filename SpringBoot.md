

## JSON的一些注解

1. `@Jsonproperties`的作用是给属性起一个别名，如果使用了这个注解那前端传回来的参数必须使用这个别名传回来。
2. `@Jsonignore`在传值的时候自动忽略加上这个注解的属性，传回来为空。
3. `@JsonPropertyOrder`改变传值到前端的值的顺序。

## @ConditionalOnMissBean的作用

如果没有提供容器的话，Spring Boot会自动使用预设容器来实现功能。



## @Responsebody和@RestController的作用

对`json`格式的数据的响应。

1. `@Responsebody`的作用是用于接收和响应序列化数据（JSON），可以支持嵌套JSON数据结构。
2. `@RestController`它有两层含义：
   * 作为控制器注入到Spring上下文环境。
   * 请求响应为数据序列化（默认序列化方式是JSON），而不是跳转到html或模板页面。

@Responsbody是用于方法上的，而@RestController是用于类上的。



## @RequestBody

用于接收前端给后端发来的JSON字符串



## @HttpMessageConverter的作用：

1. 将服务器端返回的对象序列化成`Json`格式的数据
2. 将前端返回的`Json`数据反序列成JAVA对象

SpringMVC自动配置了`Jackson`和`Gson`的`HttpMessageConverter`，springboot进行了自动化配置，操作`Json`离不开`HttpMessageConverter`

因为`@ConditionalOnMissBean`的注解的关系，如果没有自动配置`HttpMessageConverter`的话，将会自动采用springboot自动化配置的`HttpMessageConverter`。



## SpringBoot文件上传

源码在`CommonsMultipartResolver.class`下的`MultipartResolver`，按`Ctrl+H`可以查看实现类，会发现有两个实现类：

1. `StandardServletMultipartResolver`，这个是新推出的，在使用的时候可以不用额外添加依赖 
2. `CommonsMultipartResolver`，这个兼容性比较好，可以兼容以前的代码，但是使用之前需要添加相关的依赖，`Common-fileupload`

```properties
# 单个文件最大上传大小
spring.servlet.multipart.max-file-size=1MB
# 总文件上传大小
spring.servlet.multipart.max-request-size=50MB
# 上传文件的临界值(达到这个大小的时候 ，就不能再向内存写，而是借用硬盘)
spring.servlet.multipart.file-size-threshold=1B
```

JAVA代码如下：

```java
package com.study.crud.Controller;

import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.multipart.MultipartFile;

import javax.servlet.http.HttpServletRequest;
import java.io.File;
import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.UUID;

@RestController
public class UploadController {

   SimpleDateFormat sdf = new SimpleDateFormat("yyyy/mm/dd");

//   HttpServletRequest request是为了获取请求地址
    @PostMapping("/upload")
    public String fileUpLoad(MultipartFile multipartFile, HttpServletRequest request){
        String format=sdf.format(new Date());
//        从服务器请求保存的地址
        String realPath=request.getServletContext().getRealPath("/img")+format;
//        打开保存的文件
        File fold = new File(realPath);
//        如果没有，就创建一个
        if(!fold.exists()){
            fold.mkdirs();
        }
        String oldName= multipartFile.getOriginalFilename();
        String newName= UUID.randomUUID().toString()+oldName.substring(oldName.lastIndexOf("."));
        try {
            multipartFile.transferTo(new File(fold,newName));
            String url = request.getScheme() + "//" + request.getServerName() + ":" + request.getServerPort() + "/img" + format + newName;
            return url;
        }
        catch (IOException e) {
            e.printStackTrace();
        }
        return "error";
    }
}
```

## 多文件上传

使用`MultipartFile`数组，使用循环接用单文件上传的方法。



## @ControllerAdvice的相关知识

### 处理全局异常

+ 在类中的方法上加上一个`@ExceptionHandler(xxx.class)`，xxx指定异常的类型，比如上传文件大小超过最大值的异常，表示：**只有超过文件大小的异常**才会调用此方法来处理异常，其他的异常**不会**进来。

有两种方法;

1. 仅仅输出错误信息

```java
package com.study.crud.Error;

import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.multipart.MaxUploadSizeExceededException;

import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.PrintWriter;

@ControllerAdvice
public class MyError {

    @ExceptionHandler(MaxUploadSizeExceededException.class)
    public void MaxFileSize(MaxUploadSizeExceededException e, HttpServletResponse response) throws IOException {
        response.setContentType("text/html;charset=utf-8");
        PrintWriter writer = response.getWriter();
        writer.write("上传文件大小超过限制");
        writer.flush();
        writer.close();
        e.printStackTrace();
    }
}
```

2. 通过网页的方式输出错误信息

这种方式是有返回值的，还要建立前端页面`error.html`，通过获取`${error}`来获取error，从而显示出错误信息。

```java
@ExceptionHandler(MaxUploadSizeExceededException.class)
    public ModelAndView printException(MaxUploadSizeExceededException e){
        ModelAndView mv = new ModelAndView("error");
        mv.addObject("error","上传文件大小超过限制");
        return mv;
    }
```

### 预设全局数据

就是在该类中的方法上加上`@ModelAttribute(value = "xxx")`，xxx为想要的全局数据名称，然后所有的**Controller都可以访问到这个全局数据**，下面是java代码：

```java
@ControllerAdvice
public class GlobalData{
    @ModelAttribute(value = "info")
    public Map<String,Object> Mydata(){
        Map<String,Object> mp = new HashMap<>();
        mp.put("name","K");
        mp.put("sex","M");
        return mp;
    }  
}
```

```java
@Controller
public class TestController{
    @GetMapping("/hello")
    public String Hello(Model model){
        Map<String,Object>  mp = model.asMap();
        //输出mp,结果是：info:{sex=M,name=k};
        return "Hello";
    }
}
```

### 请求参数预处理

#### 问题的产生

后台从前台获取参数的时候，可能多个对象的参数名称一致，比如现在有两个实体类：

```java
// Book 类
String name;
Integer ID;
// getter、setter、tostring()省略
```

```java
// Author 类
String name;
Integer Age;
// getter、setter、tostring()省略
```

```java
@RestController
public class TestController{
    
    @PostMapping("/test")
    public void getInfo(Book book,Author author){
        sout(book);
        sout(autor);
    }
}
```

如果这个时候前台传过来的格式化参数为：

| 参数 | 值   |
| ---- | ---- |
| name | 111  |
| id   | 222  |
| name | 333  |
| Age  | 444  |

此时输出的结果是：book:{name='111,333`，id=222}，author:{name='111,333'，Age=444};

解决的办法就是：

1. 在上述getInfo方法的参数分别对应加上@ModelAttribute("value"),value随便指定的字符串
2. 在`@ControllerAdvice`类中定义一个方法，参数需要`WebDataBinder binder`，方法上面加上`@InitBinder("value")`，然后在方法体中写`binder.setFieldDefaultPrefix("value.");`



## 自定义错误页面

1. 如果页面报错的话，在`static`或者`Template`下建立`error`文件夹，在里面放对应的错误号的html，比如`404.html、500.html`如果出现了对应的错误号的话，会自动转向对应的页面。
2. 如果同系的错误号太多了，可以使用模糊的错误号页面：`4xx.html`、`5xx.html`
3. 优先级关系：如果有精确的错误号页面，就会优先访问精确的页面，没有精确的才访问模糊的，无论精确的在哪个文件夹下，都会先访问精确的；先访问`template`下的再访问`static`下的。



## 自定义错误数据以及自定义错误视图

继承`DefaultErrorAttributes`的原因就是，这个类已经收集好了错误信息，只需要自己再添加错误信息就可以了。下面的是自定义错误数据的，但是只会自动寻找到`error`文件夹下的错误号的内容，**不让转到自定义的错误页面**。

```java
@Component
public class MyErrorAttribute extends DefaultErrorAttributes {
    @Override
    public Map<String, Object> getErrorAttributes(WebRequest webRequest, boolean includeStackTrace) {
        Map<String, Object> map = super.getErrorAttributes(webRequest, includeStackTrace);
        map.put("myerror", "这是我自定义的异常信息！");
        return map;
    }
}
```

下面的代码就是自定义了错误页面，并且出错可以转到错误页面：参数中`Map<String, Object> model`，model是上面的map，这个model不推荐也不可以改变。

```java
@Component
public class MyErrorViewResolver extends DefaultErrorViewResolver {

    public MyErrorViewResolver(ApplicationContext applicationContext, ResourceProperties resourceProperties) {
        super(applicationContext, resourceProperties);
    }

    @Override
    public ModelAndView resolveErrorView(HttpServletRequest request, HttpStatus status, Map<String, Object> model) {
        ModelAndView mv = new ModelAndView();
        mv.setViewName("javaboy");
        mv.addAllObjects(model);
        return mv;
    }
}
```



## 通过 CORS 实现跨域

### 什么是跨域？

所谓的跨域就是**当一个请求url的协议、域名、端口三者之间任意一个与当前页面url不同即为跨域**









## Spring boot 整合 JPA

### 1.添加依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
            <version>2.1.8.RELEASE</version>
        </dependency>
```

### 2.配置properties

```properties
spring.jpa.database=mysql
spring.jpa.database-platform=mysql
# 每次启动的时候对表的操作是更新，如果有表就更新，如果没有就创建
spring.jpa.hibernate.ddl-auto=update
# 设置方言
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL57Dialect
```

### 3.编写实体类

```java
// book.java
package com.sys.selectcource.enities;

import lombok.Data;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Data
@Entity(name = "book")
public class Book {
//    设置主键，并且自增
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer bookid;    
    private String name;
}
```

注意的是：

1. 实体类前面必须加上`@Entity`注解，并且标注表的名称

2. 必须设置主键，即在主属性上方加上`@Id`注解，`@GeneratedValue(strategy = GenerationType.IDENTITY)`表示自增。

**多个主键的待定**

### 4.编写DAO

```java
package com.sys.selectcource.Dao;

import com.sys.selectcource.enities.Book;
import org.springframework.data.jpa.repository.JpaRepository;

//  第一个泛型是操作的实体，第二个是主键的类型

public interface BookDAO extends JpaRepository<Book,Integer> {
}
```

无需实现什么方法，就可以创建一个对应的表了。

#### 单元测试

```java
package com.sys.selectcource;

import com.sys.selectcource.Dao.BookDAO;
import com.sys.selectcource.enities.Book;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SelectcourceApplicationTests {

    @Autowired
    BookDAO bookDAO;

    @Test
    public void contextLoads() {
        Book book=new Book();
        book.setName("三国演义");
        bookDAO.save(book);
    }
    // 这里的运行结果是在表中添加一个元素

    @Test
    public void contextLoads1() {
        Book book=new Book();
        book.setName("西游记");
        bookDAO.saveAndFlush(book);
    }
    // 这个可以更新一条数据(但是这里没有，这里还是在后面添加了一条数据)，如果要实现的话就添加一行
    // book.setBookId()括号内填想要修改的属性的ID
}
```

### 5.关键字查询方法

在上面写的`BookDAO`中添加方法，当然方法需要满足命名规则JPA才会实现自定义方法，下面是命名的方法实例。

![图片](http://www.javaboy.org/images/sb/19-5.png)

需要注意的是如果方法对应的SQL语句有多个参数的时候，参数顺序不能颠倒。

### 6.自定义查询

还是在`BookDAO`中写方法，自定义的方法，然后在上面加上`@Query()`括号里面就是SQL语句，如果想使用原生的SQL，必须在括号加上一个参数`nativeQery=true`

### 7.自定义数据修改

```java
//	下面这个方法的参数一定不能反
@Query("insert into book(name,author) value(?1,?2)",nativeQuery=true)
//	因为是修改操作所以必须加入@Modifying
@Modifying
//	JPA必须加一个@Transactional才能运行(Update/insert和Delete的时候)
@Transactional
Integer addbook(String name,String author);

//	下面这个Param和MyBatis的Param不一样，导入的包不一样
@Query("insert into book(name,author) value(:name,:author)",nativeQuery=true)
@Modifying
@Transactional
Integer AddBook(@Param("name") String name,@Param("author") String author);
```

## 使用JPA搭建RESTful服务

### 1.添加RESTful依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-rest</artifactId>
</dependency>
```

### 2.创建实体类



```java
package com.example.jparestful.Bean;

import lombok.Data;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Data
@Entity(name = "book")
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Integer bookid;
    String author;
}
```

### 3.创建接口

```java
package com.example.jparestful.DAO;

import com.example.jparestful.Bean.Book;
import org.springframework.data.jpa.repository.JpaRepository;

public interface Bookdao extends JpaRepository<Book,Integer> {
}
```

### 4.使用postman测试

地址的url是

```
localhost:8080/books
```

后面的那个是实体类的小写后面加上s，使用**GET**请求就可以看见所有数据库中的信息了。

```json
{
  "_embedded": {
    "books": [
      {
        "author": "1",
        "_links": {
          "self": {
            "href": "http://localhost:8080/books/1"
          },
          "book": {
            "href": "http://localhost:8080/books/1"
          }
        }
      },
      {
        "author": "2",
        "_links": {
          "self": {
            "href": "http://localhost:8080/books/2"
          },
          "book": {
            "href": "http://localhost:8080/books/2"
          }
        }
      },
      {
        "author": "3",
        "_links": {
          "self": {
            "href": "http://localhost:8080/books/3"
          },
          "book": {
            "href": "http://localhost:8080/books/3"
          }
        }
      },
      {
        "author": "4",
        "_links": {
          "self": {
            "href": "http://localhost:8080/books/4"
          },
          "book": {
            "href": "http://localhost:8080/books/4"
          }
        }
      },
      {
        "author": "5",
        "_links": {
          "self": {
            "href": "http://localhost:8080/books/5"
          },
          "book": {
            "href": "http://localhost:8080/books/5"
          }
        }
      },
      {
        "author": "6",
        "_links": {
          "self": {
            "href": "http://localhost:8080/books/6"
          },
          "book": {
            "href": "http://localhost:8080/books/6"
          }
        }
      },
      {
        "author": "7",
        "_links": {
          "self": {
            "href": "http://localhost:8080/books/7"
          },
          "book": {
            "href": "http://localhost:8080/books/7"
          }
        }
      },
      {
        "author": null,
        "_links": {
          "self": {
            "href": "http://localhost:8080/books/8"
          },
          "book": {
            "href": "http://localhost:8080/books/8"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:8080/books{?page,size,sort}",
      "templated": true
    },
    "profile": {
      "href": "http://localhost:8080/profile/books"
    }
  },
  "page": {
    "size": 20,
    "totalElements": 8,
    "totalPages": 1,
    "number": 0
  }
}
```

从上面可以看到很多信息，比如分页，比如作者等等。

### 5.添加基础访问地址

```properties
spring.data.rest.base-path=/kyriexu
```

这样访问的url就应该是

```
localhost:8080/kyriexu/books
```

### 6.添加自定义REST服务

在DAO类中的方法自定义一个方法:

```java
Book findBookByBookid(@Param("id") Integer id);
```

然后可以在POSTMAN中发出一个请求，请求地址是

```
http://localhost:8080/books/search
```

返回的是接口的方法名称和用法，上述例子返回的是:

```json
{
  "_links": {
    "findBookByBookid": {
      "href": "http://localhost:8080/books/search/findBookByBookid{?id}",
      "templated": true
    },
    "self": {
      "href": "http://localhost:8080/books/search"
    }
  }
```

### 7.隐藏接口

修改上面DAO类中的代码

```java
@RestResource(path = "byid",rel = "fbi")
Book findBookByBookid(@Param("id") Integer id);
```

然后发送search，返回的是

```json
{
  "_links": {
    "fbi": {
      "href": "http://localhost:8080/books/search/byid{?id}",
      "templated": true
    },
    "self": {
      "href": "http://localhost:8080/books/search"
    }
  }
}
```

在**类上面**加上**@RepositoryRestSource**的注解之后，修改的是返回所有的数据的路径和集合

如果是`@RepositoryRestResource(path = "bks",collectionResourceRel = "bks")`，访问的url就是

```
http://localhost:8080/bks
```

返回的是

![Snipaste_2019-09-29_11-01-54.png](https://i.loli.net/2019/09/29/nESmGYsKrQD3xBW.png)

## @Resource和@Autowired的区别

### 1.纸面上说说区别

**@Resource**是按名称注入的，而**@Autowired**是按类型注入的，如果想要实现按名称注入就必须和**@Qualifier**一起使用。

### 2.举个小栗子

如果一个DAO接口有多个实现类，如果仅仅只是使用**@Autowired**，那么该自动注入哪个实现类呢？

```java
//	实体类
package com.example.demo.bean;

import lombok.Data;

@Data
public class book {
    private Integer id;
    private String name;
}
```

```java
//	dao 接口
package com.example.demo.dao;

import com.example.demo.bean.book;

public interface bookdao {
    public String getinfo(book b);
}
```

```java
//	第一个实现Dao接口的实现类
package com.example.demo.dao.impl;

import com.example.demo.bean.book;
import com.example.demo.dao.bookdao;
import org.springframework.stereotype.Component;

//	注意使用了@Componenet指定名称
@Component("bookdaoImpl")
public class bookdaoImpl implements bookdao {
    @Override
    public String getinfo(book b) {
        return b.toString();
    }
}
```

```java
//	第二个实现类
package com.example.demo.dao.impl;

import com.example.demo.bean.book;
import com.example.demo.dao.bookdao;
import org.springframework.stereotype.Component;

@Component("bookdaoImpl1")
public class bookdaoImpl1 implements bookdao {
    @Override
    public String getinfo(book b) {
        return "1";
    }
}
```

上方两个实现类都使用了**@Component**注解来指定名称，在单元测试中查看效果

```java
package com.example.demo;

import com.example.demo.bean.book;
import com.example.demo.dao.bookdao;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class DemoApplicationTests {
    @Autowired
    @Qualifier("bookdaoImpl")
    bookdao bkd;

    @Test
    public void contextLoads() {
        book b=new book();
        b.setId(1);
        b.setName("wocao");
        System.out.println(bkd.getinfo(b));
    }
}
```

这个的输出结果就是book的全部信息了，如果是`@Qualifier("bookdaoImpl1")`的话，那就是输出1了。

## SpringSecurity使用

### 1.导入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### 2.直接使用不自定义配置出现的结果

现在随便写一个接口

```java
package com.example.springsecurity.Controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.ArrayList;
import java.util.List;

@RestController
public class HelloController {

    @GetMapping("/hello")
    public List<String> hello(){
        List<String> list=new ArrayList<>();
        list.add("hello");
        return list;
    }
}
```

运行后直接在浏览器中输入`localhost:8080/hello`，出现的结果：

![springsecurity.png](https://i.loli.net/2019/10/01/HaJv9rB5yEpXhTG.png)

登陆的用户名默认是：`user`，而密码是在控制台中随机生成的，每次项目启动都会自动生成：

![springsecurity_password.png](https://i.loli.net/2019/10/01/Ni2bxqSIek4ClKc.png)

这样登陆进去就会看到想要的结果了。

### 3.自定义用户名密码

有两种方式自定义用户名和密码：

#### 1.在配置文件中配置

```properties
spring.security.user.name=xq
spring.security.user.password=wdnmd
spring.security.user.roles=admin
```

#### 2.配置类中配置

```java
package com.example.springsecurity.Config;

import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean
    PasswordEncoder passwordEncoder(){
        return NoOpPasswordEncoder.getInstance();
    }
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    	auth.inMemoryAuthentication()
            .withUser("xq")
            .password("wtf")
            .roles("admin")
            .and()
            .withUser("kyriexu")
            .password("wdnmd")
            .roles("user");
	}	
}
```

### 4.通过对应的路径拦截

在上述配置类的基础上添加一个重写的方法：

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
//                在admin路径下的任意地址，只能admin权限的才能登陆
                .antMatchers("/admin/**").hasRole("admin")
//                user路径下的所有地址，在User和admin中任选一个都可登陆
                .antMatchers("/user/**").hasAnyRole("admin","user");
    }
```

### 5.使用Postman来测试接口

在`4`中的代码增加下面的代码：

```java
				.anyRequest().authenticated()
                .and()
                .formLogin()
                .loginProcessingUrl("/dologin")
                .permitAll()
                .and()
                .csrf().disable();
```

其中需要最后一行的代码，将`csrf`对系统的攻击取消，因为postman测试接口的话，就相当于注入这个攻击，上述代码测试接口的过程是下述过程。

![get_8080_hello.png](https://i.loli.net/2019/10/02/CqbPve6w8NGkcEn.png)

然后再访问下面的请求，注意是POST请求
![dologin_username_password.png](https://i.loli.net/2019/10/02/B1Et5J2Ye6KLGUg.png)

### 6.完善表单登陆

```java
				.loginPage("/login")
                .usernameParameter("usr")
                .passwordParameter("pswd")
                .successHandler(new AuthenticationSuccessHandler() {
                    @Override
                    public void onAuthenticationSuccess(HttpServletRequest req, HttpServletResponse rep, Authentication authentication) throws IOException, ServletException {
                        rep.setContentType("application/json;charset=utf-8");
                        PrintWriter writer = rep.getWriter();
                        Map<String,Object> mp=new HashMap<>();
                        mp.put("status",200);
                        mp.put("msg",authentication.getPrincipal());
                        writer.write(new ObjectMapper().writeValueAsString(mp));
                        writer.flush();
                        writer.close();
                    }
                })
                .failureHandler(new AuthenticationFailureHandler() {
                    @Override
                    public void onAuthenticationFailure(HttpServletRequest req, HttpServletResponse rep, AuthenticationException e) throws IOException, ServletException {

                    }
                })
```

在`5`中的`loginProcessUrl`后面添加上面的代码，其中**authentication**存放着的是登陆用户信息

一行一行的解释一下：1.设置登陆的页面	2.设置登陆的参数，即/`dologin?usr=xxx&pswd=xxx` 3.设置登陆成功后跳转的页面，如果是前后端分离的话，就使用这种方法，如果不是前后端分离的话，就使用`.successForwardUrl()`来跳转到登陆成功后的页面。 4.同理`failureHandler`是处理登陆失败的跳转页面，方法和登陆成功的差不多，就是根据异常的类型来提示出错的原因。

登陆成功的返回结果

![login_success.png](https://i.loli.net/2019/10/02/vfoteAnEVxLKBrQ.png)

### 7.注销处理

在上述的基础上添加下面的代码

```java
				.logout()
                .logoutUrl("/logout")
                .logoutSuccessHandler(new LogoutSuccessHandler() {
                    @Override
                    public void onLogoutSuccess(HttpServletRequest req, HttpServletResponse rep, Authentication authentication) throws IOException, ServletException {
                        rep.setContentType("application/json;charset=utf-8");
                        PrintWriter writer = rep.getWriter();
                        Map<String,Object> mp=new HashMap<>();
                        mp.put("status",200);
                        mp.put("msg","注销成功");
                        writer.write(new ObjectMapper().writeValueAsString(mp));
                        writer.flush();
                        writer.close();
                    }
                })
```

下面是使用Postman测试接口的步骤，首先访问/admin/hello get请求


![please_login.png](https://i.loli.net/2019/10/02/nzGMrN7SmDpu3e9.png)

然后再登陆，登陆成功之后，再访问/admin/hello

![hello_admin.png](https://i.loli.net/2019/10/02/bHDFeQEk46Iqx39.png)

注销成功

![logout_success.png](https://i.loli.net/2019/10/02/hKrizayH6epx9UZ.png)

再访问就提示需要登陆了。

### 8.多个HttpSecurity

在一个配置类中定义**多个内部静态配置类**，分别继承**WebSecurityConfigurerAdapter**，重写`configure(httpsecurity)`方法，分别配置，就像单个配置的方法一样，就可以实现多个HttpSecurity的访问。

### 9.密码加密

上面所有的方法都是通过

```java
	@Bean
    PasswordEncoder passwordEncoder(){
        return NoOpPasswordEncoder.getInstance();
    }
```

这种方法来告诉服务器不需要加密访问就可以登陆。那怎么通过密码加密来访问页面的呢？

在单元测试中跑下面的代码

```
BCryptPasswordEncoder bcp = new BCryptPasswordEncoder();
System.out.println(bcp.encode("wdnmd"));
```

得到的结果:

![pswd_brc.png](https://i.loli.net/2019/10/02/enVhtYa9ruAmSCH.png)

将上面的密码复制到配置类中替换原来的密码，也可以使用`wdnmd`这个密码来访问页面。

上面的配置类的代码在下面:

```java
package com.example.springsecurity.Config;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.LockedException;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.method.configuration.EnableGlobalMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.authentication.AuthenticationFailureHandler;
import org.springframework.security.web.authentication.AuthenticationSuccessHandler;
import org.springframework.security.web.authentication.logout.LogoutSuccessHandler;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.PrintWriter;
import java.util.HashMap;
import java.util.Map;

@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("xq")
                .password("$2a$10$isoeFEy3LvYU7wzm45dkUOKHRYw445fUw1E5RjBh.Gz5sjXFMBX0u")
                .roles("admin")
                .and()
                .withUser("kyriexu")
                .password("$2a$10$isoeFEy3LvYU7wzm45dkUOKHRYw445fUw1E5RjBh.Gz5sjXFMBX0u")
                .roles("user");
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
//                在admin路径下的任意地址，只能admin权限的才能登陆
                .antMatchers("/admin/**").hasRole("admin")
//                user路径下的所有地址，在User和admin中任选一个都可登陆
                .antMatchers("/user/**").hasAnyRole("admin","user")
                .anyRequest().authenticated()
                .and()
                .formLogin()
                .loginProcessingUrl("/dologin")
                .loginPage("/login")
                .usernameParameter("usr")
                .passwordParameter("pswd")
                .successHandler(new AuthenticationSuccessHandler() {
                    @Override
                    public void onAuthenticationSuccess(HttpServletRequest req, HttpServletResponse rep, Authentication authentication) throws IOException, ServletException {
                        rep.setContentType("application/json;charset=utf-8");
                        PrintWriter writer = rep.getWriter();
                        Map<String,Object> mp=new HashMap<>();
                        mp.put("status",200);
                        mp.put("msg",authentication.getPrincipal());
                        writer.write(new ObjectMapper().writeValueAsString(mp));
                        writer.flush();
                        writer.close();
                    }
                })
                .failureHandler(new AuthenticationFailureHandler() {
                    @Override
                    public void onAuthenticationFailure(HttpServletRequest req, HttpServletResponse rep, AuthenticationException e) throws IOException, ServletException {
                        rep.setContentType("application/json;charset=utf-8");
                        PrintWriter writer = rep.getWriter();
                        Map<String,Object> mp=new HashMap<>();
                        mp.put("status",401);
                        if(e instanceof LockedException){
                            mp.put("msg","帐户被锁定，登录失败");
                        }else if(e instanceof BadCredentialsException){
                            mp.put("msg","用户名或密码错误，登录失败");
                        }else {
                            mp.put("msg","登陆失败");
                        }
                        writer.write(new ObjectMapper().writeValueAsString(mp));
                        writer.flush();
                        writer.close();
                    }
                })
                .permitAll()
                .and()
                .logout()
                .logoutUrl("/logout")
                .logoutSuccessHandler(new LogoutSuccessHandler() {
                    @Override
                    public void onLogoutSuccess(HttpServletRequest req, HttpServletResponse rep, Authentication authentication) throws IOException, ServletException {
                        rep.setContentType("application/json;charset=utf-8");
                        PrintWriter writer = rep.getWriter();
                        Map<String,Object> mp=new HashMap<>();
                        mp.put("status",200);
                        mp.put("msg","注销成功");
                        writer.write(new ObjectMapper().writeValueAsString(mp));
                        writer.flush();
                        writer.close();
                    }
                })
                .and()
                .csrf().disable();
    }
}
```

controller的代码在下面：

```java
package com.example.springsecurity.Controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.ArrayList;
import java.util.List;

@RestController
public class HelloController {

    @GetMapping("/hello")
    public List<String> hello(){
        List<String> list=new ArrayList<>();
        list.add("hello");
        return list;
    }

    @GetMapping("/admin/hello")
    public String adminHello(){
        return "hello admin";
    }

    @GetMapping("/user/hello")
    public String userHello(){
        return "hello user";
    }

    @GetMapping("/login")
    public String login(){
        return "please login";
    }
}
```

### 10.方法安全

在上面的配置类中添加一条注解就是

```java
@EnableGlobalMethodSecurity(prePostEnabled = true,securedEnabled = true)
```

前面的参数的意思是开启了两个注解：

- `@PreAuthorize()`，在方法执行之前进行校验
- `@PostAuthorize()`，在方法执行之后进行校验(这个一般很少用)

后面的参数开启了：`@Secured()`注解和`@PreAuthorize()`一样，不过注意书写的问题。示例代码在下方:

```java
package com.example.springsecurity.Service;

import org.springframework.security.access.annotation.Secured;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;

@Service
public class MethodService {

    @PreAuthorize("hasRole('admin')")
    public String admin(){
        return "hello admin";
    }

    @Secured("ROLE_user")
    public String user(){
        return "hello user";
    }

    @PreAuthorize("hasAnyAuthority('admin','user')")
    public String hello(){
        return "hello hello";
    }
}
```

然后在Controller中自动注入，用请求不同的路径，调用service的方法，防止我以后看不懂，写一个小栗子在下面：

```java
	@Autowired
    MethodService methodService;

    @GetMapping("/hello1")
    public String hello1(){
        return methodService.admin();
    }
```

如果我这以后都看不懂，我就~~自杀~~算了。

然后进行测试，登陆`admin`角色，访问`/hello1`会出现`hello admin`，而访问`user()`，则会出现**404**。

### 11.从数据库中读入用户

**这里使用`spring boot 整合 Mybatis`来说明。**

步骤：

#### 1.新建实体类

因为有两张表，`role`和`user`，所以新建两个实体类，值得注意的是，实体的属性和数据库中的表的属性名称**要么相同，要么符合驼峰命名规则**。贴上代码：

```java
package com.example.springsecuritydb.Enity;

import lombok.Data;

@Data
public class Role {
    private Integer id;
    private String name;
    private String nameZh;
}
```

因为是权限表，所以相对比较简单，下面的用户表比较的蛋疼...

```java
package com.example.springsecuritydb.Enity;

import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.ArrayList;
import java.util.Collection;
import java.util.List;

public class User implements UserDetails {
    private Integer id;
    private String username;
    private String password;
    private Boolean locked;
    private Boolean enabled;
    private List<Role> roles;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    @Override
    public String getUsername() {
        return username;
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return !locked;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return enabled;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        List<SimpleGrantedAuthority> authorities=new ArrayList<>();
        for (Role role : roles) {
            authorities.add(new SimpleGrantedAuthority("ROLE_"+role.getName()));
        }
        return authorities;
    }

    @Override
    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public void setLocked(Boolean locked) {
        this.locked = locked;
    }

    public void setEnabled(Boolean enabled) {
        this.enabled = enabled;
    }

    public List<Role> getRoles() {
        return roles;
    }

    public void setRoles(List<Role> roles) {
        this.roles = roles;
    }
}
```

要注意的是，这里**绝对不允许使用Lombok插件**，原因是：如果使用了Lombok插件，就会造成了**`getter和setter`**重复载入，编译器报错...那为什么会导致重复载入呢？

在这个实体类中实现了`UserDetails`这个接口，这个接口的方法有7个，分别是：

```java
package org.springframework.security.core.userdetails;

import java.io.Serializable;
import java.util.Collection;
import org.springframework.security.core.GrantedAuthority;

public interface UserDetails extends Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();

    String getPassword();	

    String getUsername();

    boolean isAccountNonExpired();		//返回帐户是否过期

    boolean isAccountNonLocked();		//查看帐户是否被锁定

    boolean isCredentialsNonExpired();		//用户名密码是否过期

    boolean isEnabled();				//查看帐户是否可用
}
```

有**username**和**password**的相关方法(其实就是getter方法)，此时使用**lombok**必然会造成重复载入getter的错误，相关的方法的用途都已经写在上面代码中的注释里面了。

#### 2.新建服务类

有了实体类也会有服务类，这个服务类要继承`UserDetailsService`这个接口，自动通过用户名来加载用户。

```java
package com.example.springsecuritydb.Service;

import com.example.springsecuritydb.Enity.User;
import com.example.springsecuritydb.Mapper.UserMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

@Service
public class UserService implements UserDetailsService {
    @Autowired
    UserMapper userMapper;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user=userMapper.getUserByUsername(username);
        if(user==null){
            throw new UsernameNotFoundException("用户不存在");
        }
        user.setRoles(userMapper.getRoleByID(user.getId()));
        return user;
    }
}
```

#### 3.新建持久层

```java
package com.example.springsecuritydb.Mapper;

import com.example.springsecuritydb.Enity.Role;
import com.example.springsecuritydb.Enity.User;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;

import java.util.List;


@Mapper
public interface UserMapper {
    @Select("select * from user where username = #{username}")
    User getUserByUsername(@Param(value = "username") String username);

    @Select("select * from role where id in (select rid from user_role where uid=#{id})")
    List<Role> getRoleByID(@Param(value = "id") Integer id);
}
```

这个不用多说，就是查询。

#### 4.配置SecurityConfig

```java
package com.example.springsecuritydb.Config;

import com.example.springsecuritydb.Service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean
    PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }

    @Autowired
    UserService userService;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userService);
    }
}
```

通过重写`configure`来加载，userService，判断用户的状态(用户名密码和输入密码是否相等 等作用)，数据库存放的密码都是密文存储，加密方式就是通过`BCryptPasswordEncoder`。

当然上面的还没写完，因为还没配置相关权限访问页面的状态。简单的说说，就是重写`configure(HttpSecure)`方法来配置相关的权限访问页面，具体的通过上面的单个`HttpSecure`来配置，这里就不配置了。

### 12.角色权限继承

在配置类中添加下面的代码

```java
@Bean
RoleHierarchy roleHierarchy() {
    RoleHierarchyImpl roleHierarchy = new RoleHierarchyImpl();
    String hierarchy = "ROLE_dba > ROLE_admin \n ROLE_admin > ROLE_user";
    roleHierarchy.setHierarchy(hierarchy);
    return roleHierarchy;
}
```

### 13.动态配置权限

[这里是源代码的地址，按Ctrl+鼠标左键点我即可访问](https://github.com/KyrieXu11/Some_Code)

所谓的动态权限配置就是，对应的角色路径都保存在数据库中，因为默认是写死在代码中的，参考上面的`HttpSecurity`.

#### 1.建立对应的表

![database_tables.png](https://i.loli.net/2019/10/04/AuTYW81FpyI24lf.png)

##### 1.1 Menu表

id	pattern

1	/db/**
2	/admin/**
3	/user/**

##### 1.2 menu_role

id	mid	rid

1		1	1
2		2	2
3		3	3

#### 2.创建对应的实体类

`Role`和`user`表和上面的一样，`user`实体类中需要将`ROLE_+`给去掉，因为数据库中已经加了这个前缀，说说`Menu`表吧，menu不用继承UserDetails接口，但是要添加一个属性

```java
    private List<Role> roles;
```

用来存放角色信息，就是查询到角色路径，每个角色路径都对应一个角色，roles就是对应的角色信息。

#### 3.创建持久层

```java
//	这个是MenuMapper
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

注意里面有一个注解`@Many`，就是调用`select`后面那个路径的方法，参数在`Column`给出了，就是一个简单的连接查询。

---

那么如果是多个参数呢？还有待思考，先占个坑。

所以还是使用XML配置的Mybatis来写比较的不用考虑这么多，关于Mybatis的XML写法，在另一个文件会写。

---



#### 4.创建服务层

UserService和上面的一样，而MenuService就是调用Mapper中的查询方法，将对应的角色和路径查出来。

#### 5.创建过滤器

过滤器最主要的功能就是：**分析出==请求地址==，根据地址分析需要哪些角色**

实现了**`FilterInvocationSecurityMetadataSource`**这个接口，代码注释中都写得很详细了，就不多说了，主要的是要将最后一个重写的方法设置返回值为**true**

```java
package com.example.springsecuritydb.Filter;

import com.example.springsecuritydb.Enity.Menu;
import com.example.springsecuritydb.Enity.Role;
import com.example.springsecuritydb.Service.MenuService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.access.ConfigAttribute;
import org.springframework.security.access.SecurityConfig;
import org.springframework.security.web.FilterInvocation;
import org.springframework.security.web.access.intercept.FilterInvocationSecurityMetadataSource;
import org.springframework.stereotype.Component;
import org.springframework.util.AntPathMatcher;

import java.util.Collection;
import java.util.List;

@Component
public class MyFilter implements FilterInvocationSecurityMetadataSource {
//    路径匹配的对象
    AntPathMatcher antPathMatcher=new AntPathMatcher();

    @Autowired
    MenuService menuService;

//    过滤器的最主要功能是分析出请求地址，根据地址分析需要哪些角色
    @Override
    public Collection<ConfigAttribute> getAttributes(Object o) throws IllegalArgumentException {
//        每次请求都会调用这个方法
//        获取请求地址，Object对象实际是FilterInvocation对象，集合返回的是所需要的角色
        String requestUrl = ((FilterInvocation) o).getRequestUrl();

//        可以考虑将Menu存到Redis里面去，因为Menu表基本不变
        List<Menu> allMenus = menuService.getAllMenus();
        for (Menu menu : allMenus) {
//            将pattern和请求地址比较，如果匹配的话就返回对应pattern的角色
            if(antPathMatcher.match(menu.getPattern(),requestUrl)){
                List<Role> roles=menu.getRoles();
                String[] rolestr=new String[roles.size()];
                for (int i = 0; i < roles.size(); i++) {
                    rolestr[i]=roles.get(i).getName();
                }
                return SecurityConfig.createList(rolestr);
            }
        }
//        如果匹配不上，就返回一个login对象，表示登陆之后就能访问
//        不是这个返回这个就可以访问了，还要看后续操作处理，实际上这个也就是一个标记
        return SecurityConfig.createList("ROLE_LOGIN");
    }

    @Override
    public Collection<ConfigAttribute> getAllConfigAttributes() {
        return null;
    }

//    是否支持这种方式，默认返回false，改成true
    @Override
    public boolean supports(Class<?> aClass) {
        return true;
    }
}
```

#### 6.创建非表中角色处理的类

如果登陆用户是匿名登陆的一个实例的话，就抛出异常，否则就返回不继续执行，要将另外两个重写方法设置返回值为true，表示支持这种方式。

```java
package com.example.springsecuritydb.Config;

import org.springframework.security.access.AccessDecisionManager;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.access.ConfigAttribute;
import org.springframework.security.authentication.AnonymousAuthenticationToken;
import org.springframework.security.authentication.InsufficientAuthenticationException;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.stereotype.Component;

import java.util.Collection;

//  已经得到了需要哪些角色，将这些角色和已有角色对比
@Component
public class MyAcessDecisionManager implements AccessDecisionManager {
//    第一个参数存放着的是登陆用户的信息，第二个参数就是MyFilter中的Object参数
//    第三个参数是访问路径需要的权限
    @Override
    public void decide(Authentication authentication, Object o, Collection<ConfigAttribute> collection) throws AccessDeniedException, InsufficientAuthenticationException {
        for (ConfigAttribute Attribute : collection) {
            if("ROLE_LOGIN".equals(Attribute.getAttribute())){
                if(authentication instanceof AnonymousAuthenticationToken){
                    throw new AccessDeniedException("非法请求");
                }else {
                    return;
                }
            }
            Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
            for (GrantedAuthority authority : authorities) {
                if(authority.getAuthority().equals(Attribute.getAttribute())){
                    return;
                }
            }
        }
        throw new AccessDeniedException("非法请求");
    }

    @Override
    public boolean supports(ConfigAttribute configAttribute) {
        return true;
    }

    @Override
    public boolean supports(Class<?> aClass) {
        return true;
    }
}
```

#### 7.创建配置类

这个不多说了，就基本上和静态的差不多，代码上传到GitHub上了。

