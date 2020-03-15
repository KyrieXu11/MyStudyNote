

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



## @RequestParam的作用

如果访问的url是`localhost:8080/hello?id=1`，后台怎么获取id的值呢？

那就是这种方式：

```java
	@GetMapping("/hello")
    public Integer getParam(@RequestParam("id") Integer id){
        return id;
    }
```

1. 当然这个如果不想要其中的参数是必要的参数的话，可以在`@RequestParam`中多设置一个参数**@RequestParam(required = false)`**
2. 这个可以简写成 `@RequestParam Integer id`
3. 可以设置默认值：参数是`defaultValue = "test"`
4. 获取url中所有的参数

```java
    @PostMapping("/api/foos")
    @ResponseBody
    public String updateFoos(@RequestParam Map<String,String> allParams) {
        return "全部的参数是 " + allParams.entrySet();
    }
```





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

所谓的跨域就是**当一个请求url的协议、域名、端口三者之间任意一个与当前页面url不同即为跨域**，打个比方，前端启动的端口是`8081`，后端启动的端口是`8080`，那么两者之间的通信如果不通过`nginx`的话，必将发生跨域。









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

## SpringBoot整合WebSocket实现在线群聊

### 1.首先导入需要的依赖

```xml
		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-websocket</artifactId>
            <version>2.2.0.RELEASE</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.webjars/sockjs-client -->
        <dependency>
            <groupId>org.webjars</groupId>
            <artifactId>sockjs-client</artifactId>
            <version>1.1.2</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.webjars.bower/jquery -->
        <dependency>
            <groupId>org.webjars.bower</groupId>
            <artifactId>jquery</artifactId>
            <version>3.4.1</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.webjars/stomp-websocket -->
        <dependency>
            <groupId>org.webjars</groupId>
            <artifactId>stomp-websocket</artifactId>
            <version>2.3.3</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.webjars/webjars-locator-core -->
        <dependency>
            <groupId>org.webjars</groupId>
            <artifactId>webjars-locator-core</artifactId>
            <version>0.41</version>
        </dependency>
```

这些依赖包分别是：`websocket` 、`JQuery`

`sockjs-client`: 创建了一个低延迟、全双工的浏览器和web服务器之间通信通道，[点击访问Sockjs-Client相关说明][ https://www.jianshu.com/p/9e37d343267e ]。

`stomp-websocket`：[点击访问Stomp-websocket相关][ https://blog.csdn.net/m0_37542889/article/details/83750665 ]

`webjars-locator-core`：webjars的版本定位工具

### 2.配置后台的WebSocket

```java
//WebSocketConfig类
package com.example.websocket.Config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
//开启消息代理
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
//    配置消息代理
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
//        设置消息代理的前缀
        registry.enableSimpleBroker("/topic");
//        规定哪些消息对方法处理哪些消息给代理处理
//        这个是为了区分开来两个方式
//        如果是`/app`开始的那就是代理处理
        registry.setApplicationDestinationPrefixes("/app");
    }

//    建立连接点,websocket连接的建立，服务端必须有一个点给他去建立
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
//        添加连接点，启用SockJS
        registry.addEndpoint("/chat").withSockJS();
    }
}
```

1. 因为是配置类，所以需要加上`@Configuration`注解，第二个注解是开启消息代理，消息代理就是待会客户端发消息需要经过消息代理来广播， 能够在 WebSocket 上启用 STOMP 。
2. 设置消息代理的前缀为`/topic`，以便客户端指定消息代理来广播消息，所以待会的`controller`需要使用`@sendto("/topic/**")`来发送消息到`/topic`的消息代理
3. 而应用程序目的的前缀是客户端发送消息需要发送消息的目的地的前缀，因为我在`controller`中使用了`@MessageMapping("/hello")`，所以客户端发消息需要发送到`/app/hello`中
4. `registerStompEndpoints`这个方法的作用就是建立连接点，`addEndpoint`中的字符串是前端建立`sockjs`中的字符串，二者要保持一致

```java
//	Controller类
package com.example.websocket.Controller;

import com.example.websocket.Bean.Message;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;

@Controller
public class GreetingController {

//    用于处理浏览器中的消息，浏览器通过hello接口来发送消息给服务端
    @MessageMapping("/hello")
//    因为有topic由代理广播到连接上来的客户端上去
    @SendTo("/topic/greetings")
    public Message greeting(Message message){
        return message;
    }
    
    //	message中包含两个字段：name,content
}
```

Controller需要返回给消息代理一个Message对象

### 3.页面配置

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script type="text/javascript"
            src="webjars/jquery/3.4.1/dist/jquery.min.js"></script>
    <script type="text/javascript"
            src="webjars/sockjs-client/1.1.2/sockjs.min.js"></script>
    <script type="text/javascript"
            src="webjars/stomp-websocket/2.3.3/stomp.min.js"></script>

</head>
<body>
<div>
    <table>
        <tr>
            <td>请输入用户名</td>
            <td><input type="text" id="name"></td>
        </tr>
        <tr>
            <td><input type="button" id="connect" value="连接"></td>
            <td><input type="button" id="disconnect" value="断开连接"></td>
        </tr>
    </table>
</div>
<div id="chat" style="display: none;">
    <table>
        <tr>
            <td>请输入聊天内容</td>
            <td><input type="text" id="content"></td>
            <td><input type="button" id="send" value="发送"></td>
        </tr>
    </table>
    <div id="conversation">正在聊天中.....</div>
</div>
<script>
    $(function () {
        $("#connect").click(function () {
            connect();
        });
        $("#disconnect").click(function () {
            if(stompClient!=null){
                //断开连接
                stompClient.disconnect();
            }
            setConnected(false);
        });

        $("#send").click(function () {
            //第三个参数是一个JSON对象，发送的就是JSON对象
            stompClient.send("/app/hello",{},JSON.stringify({'name':$("#name").val(),'content':$("#content").val()}))
        })
    });

    var stompClient = null;

    function showGreeting(msg) {
        $("#conversation").append('<div>'+msg.name+'：'+msg.content+'<div>')
    }

    function connect() {
        if(!$("#name").val()){
            return;
        }
        //和chat建立连接
        var socket = new SockJS('/chat');
        //建立stompClient
        stompClient=Stomp.over(socket);
        //建立连接，第一个参数：设置基本配置比如优先级，第二个参数是连接成功的回调，第三个参数是连接失败的回调
        stompClient.connect({},function (success) {
            //设置连接成功之后的控件的状态
            setConnected(true);
            //订阅服务上的消息，第一个参数订阅服务器的地址，即设置监听
            //第二个参数为回调的消息
            stompClient.subscribe('/topic/greetings',function (msg) {
                //回调的消息存放在msg的body(这里的body是页面的属性就比如header之类的)中
                //将返回的消息格式化成json格式
                showGreeting(JSON.parse(msg.body));
            })
        })
    }
    
    function setConnected(flag) {
        //设置控件属性
        $("#connect").prop("disabled",flag);
        $("#disconnect").prop("disabled",!flag);
        if(flag){
            $("#chat").show();
        }else {
            $("#chat").hide();
        }
    }
</script>
</body>
</html>
```

1. 上面的几个控件都不用说了，设置了一个全局变量`stompClient`并且初始化为`null`
2. 第一个函数：点击连接按钮的话就调用`connect`函数，如果点击断开连接的话，判断客户端对象是否为null，如果不是就调用`disconnect`函数
3. `connect`函数：如果输入框中的值是空的就不继续执行，否则**使用SockJS和后端设置的连接点进行建立连接**·，`调用Stomp`的over方法，建立`stompClient`即客户端，调用`stompClient`的connect方法，有三个参数，第一个参数是设置优先级之类的基本参数，第二个参数是连接成功的回调，第三个参数是连接失败的回调，在函数中设置监听，并且获取信息，并且展示出来
4. 发送消息的函数是发送到`/hello`接口，然后再发送到消息代理上去。

### 4.其他的知识

**@OnOpen** 表示有浏览器链接过来的时候被调用
**@OnClose** 表示浏览器发出关闭请求的时候被调用
**@OnMessage** 表示浏览器发消息的时候被调用
**@OnError** 表示有错误发生，比如网络断开了等等

**sendMessage** 用于向浏览器回发消息 

```java
package com.how2java.bitcoin;
 
import java.io.IOException;
import javax.websocket.OnClose;
import javax.websocket.OnError;
import javax.websocket.OnMessage;
import javax.websocket.OnOpen;
import javax.websocket.Session;
import javax.websocket.server.ServerEndpoint;
 
/**
 * @ServerEndpoint 注解是一个类层次的注解，它的功能主要是将目前的类定义成一个websocket服务器端,
 * 注解的值将被用于监听用户连接的终端访问URL地址,客户端可以通过这个URL来连接到WebSocket服务器端
 */
@ServerEndpoint("/ws/bitcoinServer")
public class BitCoinServer {
     
    //与某个客户端的连接会话，需要通过它来给客户端发送数据
    private Session session;
 
    @OnOpen
    public void onOpen(Session session){
        this.session = session;
        ServerManager.add(this);    
    }
     
    public void sendMessage(String message) throws IOException{
        this.session.getBasicRemote().sendText(message);
    }
 
    @OnClose
    public void onClose(){
        ServerManager.remove(this); 
    }
 
    @OnMessage
    public void onMessage(String message, Session session) {
        System.out.println("来自客户端的消息:" + message);
    }
 
    @OnError
    public void onError(Session session, Throwable error){
        System.out.println("发生错误");
        error.printStackTrace();
    }
 
}
```



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

这个就是在内存中存放用户信息，在登陆验证的时候对比。

### 4.通过对应的路径拦截

在上述配置类的基础上添加一个重写的方法：

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
//				开启配置
        http.authorizeRequests()
//                在admin路径下的任意地址，只能admin权限的才能登陆
                .antMatchers("/admin/**").hasRole("admin")
//                user路径下的所有地址，在User和admin中任选一个都可登陆
                .antMatchers("/user/**").hasAnyRole("admin","user");
    }
```

### 一些方法的作用

`.authorizeRequests()`的作用就是开启配置。

`.anyRequest().authenticated()`:任何请求都只要登陆了就可以访问。

`.PermitAll`:和登陆相关的接口不需要验证，直接通过。

### 5.使用Postman来测试接口

在`4`中的代码增加下面的代码：

```java
//		剩下的只要登陆就能访问				
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

如果有中文的话，还是在body里面写用户名和密码，再使用post请求发送

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
//				允许所有登陆用户和匿名用户访问
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

通过重写`configure`来加载userService，判断用户的状态(用户名密码和输入密码是否相等 等作用)，数据库存放的密码都是密文存储，加密方式就是通过`BCryptPasswordEncoder`。

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

UserService和上面的一样，返回的都是一个用户信息。

而MenuService就是调用Mapper中的查询方法，将对应的角色和路径查出来。

#### 5.创建过滤器

过滤器最主要的功能就是：**分析出==请求地址==，根据地址分析需要哪些角色**

实现了**`FilterInvocationSecurityMetadataSource`**这个接口，代码注释中都写得很详细了，就不多说了，主要的是要将最后一个重写的方法设置返回值为**true**

 1.一开始注入了MenuService，MenuService的作用是用来查询数据库中url pattern和role的对应关系，查询结果是一个List集合，集合中是Menu类，Menu类有两个核心属性，一个是url pattern，即匹配规则(比如`/admin/**`)，还有一个是List<Role>,即这种规则的路径需要哪些角色才能访问。 

 2.我们可以从getAttributes(Object o)方法的参数o中提取出当前的请求url，然后将这个请求url和数据库中查询出来的所有url pattern一一对照，看符合哪一个url pattern，然后就获取到该url pattern所对应的角色，当然这个角色可能有多个，所以遍历角色，最后利用SecurityConfig.createList方法来创建一个角色集合。 

 3.第二步的操作中，涉及到一个优先级问题，比如我的地址是`/employee/basic/hello`,这个地址既能被`/employee/**`匹配，也能被`/employee/basic/**`匹配，这就要求我们从数据库查询的时候对数据进行排序，将`/employee/basic/**`类型的url pattern放在集合的前面去比较。 

 4.如果getAttributes(Object o)方法返回null的话，意味着当前这个请求不需要任何角色就能访问，甚至不需要登录。但是在我的整个业务中，并不存在这样的请求，我这里的要求是，所有未匹配到的路径，都是认证(登录)后可访问，因此我在这里返回一个`ROLE_LOGIN`的角色，这种角色在我的角色数据库中并不存在，因此我将在下一步的角色比对过程中特殊处理这种角色。 

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
//        如果匹配不上，就返回一个login对象，表示登陆之后就能访问，就是一级菜单的角色
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
//    第一个参数存放着的是登陆用户的信息（看successhandler中的参数），第二个参数就是MyFilter中的Object参数
//    第三个参数是访问路径需要的权限
    @Override
    public void decide(Authentication authentication, Object o, Collection<ConfigAttribute> collection) throws AccessDeniedException, InsufficientAuthenticationException {
        for (ConfigAttribute Attribute : collection) {
            //            这个是判断需要的角色是否等于ROLE_LOGIN
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

要注意到配置类中有一个方法：

```java
 @Override
 protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userService);
 }
```

这个方法就是加载数据库中的用户，查看`UserDetailService`返回的是一个`user`对象，然后通过此方法来加载数据库中的用户信息。还要在config类中添加下面的代码。

```java
 http.authorizeRequests()
                .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() 					{
                    @Override
                    public <O extends FilterSecurityInterceptor> O postProcess(O o) {
                       o.setAccessDecisionManager(acessDecisionManager);
                       o.setSecurityMetadataSource(myFilter);
                       return o;
                    }
```

### 14.CSRF攻击

先来看下服务端没有禁用csrf的时候，postman怎么发送数据

#### 1.配置SpringSecurity

简单的说下下面的代码的作用吧：

1. 因为要使用postman做测试，所以做一个简单的`httpbasic`认证，方便登陆。
2. 配置了2个用户(其实这里只用一个用户就可测试了)
3. 开启接口权限，认证之后就可以访问

```java
@EnableWebSecurity
@Configuration
public class CusSecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder(10);
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        PasswordEncoder passwordEncoder=passwordEncoder();
        auth.inMemoryAuthentication()
                .withUser("kyriexu").roles("admin").password(passwordEncoder.encode("123"))
                .and()
                .withUser("xu").roles("student").password(passwordEncoder.encode("123"));
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf()
                .and()
                .authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .httpBasic();
    }
}
```

#### 2.编写接口

这里的接口是基础的四个接口，`get`、`post`、`put`、`delete`

```java
@RestController
public class HelloController {

    @GetMapping("/")
    public String get(){
        return "get";
    }
    @PostMapping("/")
    public String post(){
        return "post";
    }
    @PutMapping("/")
    public String put(){
        return "put";
    }
    @DeleteMapping("/")
    public String delete(){
        return "delete";
    }
}
```

#### 3.使用postman访问

​	get请求是正常的通过的

![csrf_get.png](https://i.loli.net/2020/02/12/vkcjZr3V4CAqlSR.png)

post请求无法通过，因为开启了`csrf`防护

![csrf_post_failed.png](https://i.loli.net/2020/02/12/DrsiSjmyqpx1GZF.png)

那么要怎么才能在csrf防护开启的情况下，能够发送post请求呢？

首先得给postman装上一个拦截器

![csrf_postman_interceptor.png](https://i.loli.net/2020/02/12/7RXvrlDP1ZyScIf.png)

然后使用get方式发送一次请求之后，查看`cookies`，里面会有一个**`XSRF-TOKEN`**的cookies，把这个加在post方式的头部，就可以发送成功了。但是因为我这里装拦截器的时候有点问题，装不上，所以就只能根据别人的视频来进行记录了。

## SpingBoot整合Redis

### 1.导入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <version>2.1.8.RELEASE</version>
</dependency>
```

### 2.配置`properties`

```properties
#	端口号一般不会改
spring.redis.port=6379
#	虚拟机中的ip地址，使用ifconfig指令查看
spring.redis.host=192.168.1.106
#	如果有配置密码的话
spring.redis.password=123
#	选择redis的数据库
spring.redis.database=0
```

### 3.编写Controller

使用`StringRedisTemplate`

```java
package com.example.springsecuritydb.Controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class RedisController {

    @Autowired
    StringRedisTemplate redisTemplate;

    @GetMapping("/set")
    public String setValue(){
        String a="abc";
        ValueOperations<String,String> ops=redisTemplate.opsForValue();
        ops.set(a,"abc");
        return a;
    }
}
```

### 4.访问`/set`接口

#### 4.1访问成功

页面数据：

![success_set.png](https://i.loli.net/2019/10/05/NvXhd6ugEszkLBj.png)

Redis中的数据：

![redis_res.png](https://i.loli.net/2019/10/05/6LJD73r5xAGTO4v.png)

#### 4.2访问失败出现的原因

配置虚拟机中的redis出问题，即IDEA控制台中输出无法连接redis

![redis_conf.png](https://i.loli.net/2019/10/05/sHJNGWXy9gPchn8.png)

将配置文件修改成上面的样子..

##### 修改方法：

1. 进入redis配置文件目录：`cd /etc/redis`
2. 修改`redis.conf`：`vim redis.conf`
3. 搜索`bind`，在普通模式下键盘输入：`/bind`，使用`n`和`N`来进行下上的搜索
4. 搜索`protected`方法同上，将`protected mode`改成`no`

修改redis密码的方法：还是进入`redis.conf`，搜索`requirepass`，修改后面的参数就可以修改密码了

### 5.防止redis关闭不了的解决办法

启动时尽量不要使用`sudo su`来启动，使用`sudo -s`就可以

因为redis开启了守护进程，此时无法使用`9`号信号和`18`号信号关闭，所以解决办法就是：

`/etc/init.d/redis-server stop`关闭redis服务，这样就可以正常使用`redis-cli shutdown`来关闭redis了。

## SpingSecurity整合Oauth2协议

### 说在前面

---

**java11有个独特的特(bu)性(g)**

不知道咋回事，反正添加这个依赖试试就好：

```xml
		<dependency>
            <groupId>com.sun.xml.bind</groupId>
            <artifactId>jaxb-core</artifactId>
            <version>2.3.0.1</version>
        </dependency>
        <dependency>
            <groupId>javax.xml.bind</groupId>
            <artifactId>jaxb-api</artifactId>
            <version>2.3.1</version>
        </dependency>
        <dependency>
            <groupId>com.sun.xml.bind</groupId>
            <artifactId>jaxb-impl</artifactId>
            <version>2.3.1</version>
        </dependency>
```

如果没有添加这些依赖的话，就会报下面的错：

```java
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'springSecurityFilterChain' defined in class path resource 
    [org/springframework/security/config/annotation/web/configuration/WebSecurityConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate 
    
[javax.servlet.Filter]: Factory method 'springSecurityFilterChain' threw exception;

nested exception is java.lang.NoClassDefFoundError: javax/xml/bind/JAXBException
```

---



### 1.编写授权服务器配置

1. 首先要继承`AuthorizationServerConfigurerAdapter`，并且给这个类加上`@Configuration`和`@EnableAuthorizationServer`
2. 然后自动注入三个`Bean`：`AuthenticationManager`、`RedisConnectionFactory`、`UserDetailsService`，其中第一个和第三个是要在后面的配置安全配置类中手动装配成Bean的。
3. 重写`configure(ClientDetailsServiceConfigurer clients)`方法：结合上面上传到github上的源码来一行一行的解释
	 * 首先设置授权服务器的id，之后获取token需要获取客户端的id
	 * 然后就是设置授权模式
	 * 设置过期时间
	 * 设置静态资源ID，静态资源的获取需要这个ID
	 * scope暂时先空着
	 * secret就是获取这个token所需要的密码，如果是密文存储的话，就需要一个密码的编码器。
4. 重写`configure(AuthorizationServerEndpointsConfigurer endpoints)`方法：
	* 这个方法的作用就是让生成的token信息存放在redis中，需要上面自动装配的三个bean
5. 重写`configure(AuthorizationServerSecurityConfigurer security)`，这个方法的作用就是设置授权服务器允许给客户端授权。

### 2.设置资源服务器

都是重写configure方法，不过第一个是设置静态资源的ID，一个是设置权限要求

### 3.服务器端安全配置

### 4.测试

使用postman发送post请求，并且设置一些必要的参数：

```
http://localhost:8080/oauth/token
```


![oauth_getToken.png](https://i.loli.net/2019/10/06/HFuK3ZxM1czU5yN.png)

查询redis中是否存储了信息：

![redis_token.png](https://i.loli.net/2019/10/06/dZY5uDiLrNQtEye.png)



测试不同的接口：

```
http://localhost:8080/admin/hello?access_token=xxx
```

就会出现：`hello admin`

## SpringBoot整个JWT（JJWT版本的实现）

### 确定jwt所用的库版本

我这里使用的jwt版本是`jjwt`，网址：https://github.com/jwtk/jjwt

### 添加jwt依赖

```xml
<dependencies>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.11.0</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.11.0</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId> <!-- or jjwt-gson if Gson is preferred -->
        <version>0.11.0</version>
        <scope>runtime</scope>
    </dependency>
```

项目不止这些依赖，还需要`springboot`和`springsecurity`的依赖，这里就不添加了。

### JWT的客户端和服务端的交互原理

下面都是个人理解所画的图，可能有错

![jwt_sequence.png](https://i.loli.net/2020/02/13/PLHUyjfp5ia6Z4V.png)

### JWT过滤器原理

![jwt_filter.png](https://i.loli.net/2020/02/13/gbZmzwqi8QtuANc.png)

一个请求需要通过多个过滤器才能接触到接口，而过滤器是需要手动去写的

#### 添加第一个过滤器——`UsernamePasswordAuthenticationFilter`

![jwt_usernamepasswordauthenticationfilter.png](https://i.loli.net/2020/02/13/FfPGxhMKcu4nN21.png)

```java
/**
 * @author KyrieXu
 * @date 2020/2/12 15:34
 **/
@EqualsAndHashCode(callSuper = true)
@Data
@AllArgsConstructor
@NoArgsConstructor
public class JwtAuthFilter extends UsernamePasswordAuthenticationFilter {

    private AuthenticationManager authenticationManager;

    public static final String key="securesecuresecuresecuresecuresecuresecuresecuresecuresecuresecuresecuresecuresecuresecuresecure";


//    下面的方法成功认证之后会到这个方法来，这个方法产生jwt的token
    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult) throws IOException, ServletException {
//        subject和jwt的字段是有关系的，忘了的话可以取官网看下debugger
//        第一个是设置headers的token
//        第二个是设置body的token
//        key转换成byte数组长度必须大于256
        String token = Jwts.builder()
                .setSubject(authResult.getName())
                .claim("authorities", authResult.getAuthorities())
                .setIssuedAt(new Date())
//                设置过期时间
                .setExpiration(java.sql.Date.valueOf(LocalDate.now().plusWeeks(2)))
//                设置签名signature,这个是至关重要的
                .signWith(Keys.hmacShaKeyFor(key.getBytes()))
                .compact();
//        token放在header里面，最好要在token前面加上认证方式
        response.addHeader("Authorization","Bearer "+token);

    }

    // 这个注解是Lombok的注解，抛出异常，简化代码
    @SneakyThrows
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
//        从request的inputstream读取内容(json格式)放到UPAuthRequest类中，就是快捷赋值创建对象
        UPAuthRequest authenticationRequest = new ObjectMapper().readValue(request.getInputStream(), UPAuthRequest.class);
        Authentication authentication=new UsernamePasswordAuthenticationToken(
                authenticationRequest.getUsername(),
                authenticationRequest.getPassword()
        );
//        确定用户名是否存在以及密码是否正确
        return authenticationManager.authenticate(authentication);
    }
}
```

上面的代码说两句吧：

1. 首先执行`attemptAuthentication`这个方法，如果校验成功，那么就继续执行`successfulAuthentication`这个方法来生成jwt，如果校验失败则抛出异常。
2. 在将token返回给客户端的时候，需要在token之前加上token的类型，我这里其实加和不加都一样的，但是在实际操作中，加上类型可以灵活的判断token的类型，进行不一样的业务处理。
3. key是一个字符串，在其转换成byte数组的时候，数组长度必须大于256。
4. 还有里面有一个`UPAuthRequest`，其实就是用户名和密码的一个自定义的类。

#### 添加第二个拦截器——`OncePerRequestFilter`

![jwt_OncePerRequestFilter.png](https://i.loli.net/2020/02/13/tKR7MvfL2Bp6zEP.png)

```java
/**
 * @author KyrieXu
 * @date 2020/2/12 17:58
 **/
public class JwtTokenVerifier extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        String tokenHeader = request.getHeader("Authorization");
        if (!StringUtils.hasText(tokenHeader) || !tokenHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }
//        从header提取token
        String token = tokenHeader.replace("Bearer ", "");
        try {
//        解析token
            Jws<Claims> claimsJws = Jwts.parser()
                    .setSigningKey(Keys.hmacShaKeyFor(JwtAuthFilter.key.getBytes()))
                    .parseClaimsJws(token);
//        获取body
            Claims body = claimsJws.getBody();
//        获取用户名，对照着 https://jwt.io/ 的debugger来看
            String username = body.getSubject();
            List<Map<String, String>> authorities = (List<Map<String, String>>) body.get("authorities");
            Set<SimpleGrantedAuthority> simpleGrantedAuthorities = new HashSet<>();
            for (Map<String, String> authoritys : authorities) {
                String authority = authoritys.get("authority");
                simpleGrantedAuthorities.add(new SimpleGrantedAuthority(authority));
            }
//            将权限放入authentication
            Authentication authentication = new UsernamePasswordAuthenticationToken(
                    username,
                    null,
                    simpleGrantedAuthorities
            );
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }catch (JwtException e){
            throw new JwtException(String.format("Token %s 不可信",token));
        }
        filterChain.doFilter(request,response);
    }
}
```

### 配置Spring Security

```java
/**
 * @author KyrieXu
 * @date 2020/2/12 11:36
 **/
@EnableWebSecurity
@Configuration
public class CusSecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder(10);
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        PasswordEncoder passwordEncoder=passwordEncoder();
        auth.inMemoryAuthentication()
                .withUser("kyriexu").roles("admin").password(passwordEncoder.encode("123"))
                .and()
                .withUser("xu").roles("student").password(passwordEncoder.encode("123"));
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/","/login").permitAll()
                .antMatchers("/admin/**").hasRole("admin")
                .antMatchers("/student/**").hasAnyRole("admin","student")
                .anyRequest().authenticated()
                .and()
                .csrf().disable()
                .sessionManagement()
//                因为jwt是无状态的，所以要设置session无状态
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
//                传进去父类的authenticationmanager
                .addFilter(new JwtAuthFilter(authenticationManager()))
//                表示这个过滤器在上面那个过滤器后面
                .addFilterAfter(new JwtTokenVerifier(),JwtAuthFilter.class);
    }
}
```

还是说一下上面的代码：

1. Spring Security的常规配置不用多说了
2. 就是添加2个拦截器，注意顺序就行.
3. AuthenticationManager==不能使用==自动注入，因为Filter不是spring组件，自动注入是为空的，而且如果把Filter表示为组件之后，就会抛出异常，运行不了。

### 编写接口测试

```java
/**
 * @author KyrieXu
 * @date 2020/2/12 11:44
 **/
@RestController
public class HelloController {

    @GetMapping("/")
    public String get() {
        return "get";
    }

    @PostMapping("/")
    public String post() {
        return "post";
    }

    @PutMapping("/")
    public String put() {
        return "put";
    }

    @DeleteMapping("/")
    public String delete() {
        return "delete";
    }

    @GetMapping("/student/123")
    public Authentication StuGet(){
        return SecurityContextHolder.getContext().getAuthentication();
    }

    @GetMapping("/admin/123")
    public Authentication AdminGet(){
        return SecurityContextHolder.getContext().getAuthentication();
    }
}
```

发送登陆请求：

![jwt_login.png](https://i.loli.net/2020/02/13/opAWYPCfcjZE4mB.png)

测试接口，成功！

![jwt_api_test.png](https://i.loli.net/2020/02/13/X3rzRGLQUDFZ8Oa.png)





## 关于项目的一些总结(持续更新)

### 1.要创建一个和前台传数据的类

```java
package com.example.project.model;

public class RespBean {
    private Integer status;
    private String msg;
    private Object obj;

    private RespBean() {
    }

    public static RespBean build() {
        return new RespBean();
    }

    public static RespBean ok(String msg, Object obj) {
        return new RespBean(200, msg, obj);
    }

    public static RespBean ok(String msg) {
        return new RespBean(200, msg, null);
    }

    public static RespBean error(String msg, Object obj) {
        return new RespBean(500, msg, obj);
    }

    public static RespBean error(String msg) {
        return new RespBean(500, msg, null);
    }

    private RespBean(Integer status, String msg, Object obj) {
        this.status = status;
        this.msg = msg;
        this.obj = obj;
    }

    public Integer getStatus() {

        return status;
    }

    public RespBean setStatus(Integer status) {
        this.status = status;
        return this;
    }

    public String getMsg() {
        return msg;
    }

    public RespBean setMsg(String msg) {
        this.msg = msg;
        return this;
    }

    public Object getObj() {
        return obj;
    }

    public RespBean setObj(Object obj) {
        this.obj = obj;
        return this;
    }
}
```

就是存在里面的信息可以发送到前端去，前端通过axios来获取json字符串，其中`msg`是返回给前端的信息，`obj`是登陆成功之后发送给前端的用户对象，还有状态码，传给前端给出相应的判断。

### 2.怎么传数据呢？

```java
				.successHandler(new AuthenticationSuccessHandler() {
                    @Override
                    public void onAuthenticationSuccess(HttpServletRequest req, HttpServletResponse rep, Authentication authentication) throws IOException, ServletException {
                        rep.setContentType("application/json;charset=utf-8");
                        PrintWriter writer = rep.getWriter();
                        Hr hr = (Hr) authentication.getPrincipal();
                        hr.setPassword(null);
                        RespBean respBean = RespBean.ok("登陆成功", hr);
                        String s=new ObjectMapper().writeValueAsString(respBean);
                        writer.write(s);
                        writer.flush();
                        writer.close();
                    }
                })
```

上面的是登陆成功之后传给前台的数据，

1. 首先要设置返回给前台的页面的数据类型是JSON字符串
2. getWriter()：发送请求内容至前端
3. 因为`authentication`中存放的是登陆成功用户的信息，所以通过`getPrincipal`方法获取这个用户的对象，但是返回的是Object对象，所以需要强制转换成对应的用户对象
4. 将返回给前端的密码设置为空，这个步骤保护数据安全
5. 设置登陆成功的返回码和消息以及用户对象。
6. `writeValueAsString()`把java对象转化成json字符串并打印出来，括号中的参数就是java对象
7. `write()`就是将返回给前端的json字符串写到页面中，此时前端可以获取发送给前端的数据(`msg`、`obj`、`status`)
8. 关闭回复
9. ==不仅仅是上面的方式==，前端还可以通过访问后端写好的接口(`Controller`中写的接口)来获取返回的`json`数据

登陆失败就是status的状态码不和成功的状态码相同以及没有发送obj对象，==值得注意的是==，这个status是400或者500，但是请求的状态码是200，是正常的，如果不是正常的可能服务器有些问题，以及出现其他的异常，这个在前端会处理异常。

### 3.前端相关的配置

#### 1.用户输入用户名和密码的时候的判断

用户名和密码都输入完成之后，需要点击登陆，此时有个校验方法来判断用户名和密码是否都输入成功,即是否都不为空

```js
submit(){
            this.$refs.loginForm.validate((valid)=>{
                // 用户名和密码都已经输入的话
                // 只是简单的校验用户名或者密码是否输入
                if(valid){
                    // rep是因为调用了axios的方法，得到了后台传过来的JSON字符串
                    // msg是信息，obj是用户对象
                    // 后面的是后来的一些方法，如果进入了这个分支，就说明输入正常
                    // 但是无法知道用户名和密码输入是否正确
            // 因为在api.js中设置了axios，在那里根据后端返回的状态码判断用户名和密码输入是否正确
                    this.postKeyValueRequest('/dologin',this.loginForm).then(rep=>{
                        if(rep){
                            window.sessionStorage.setItem("user",JSON.stringify(rep.obj));
                            // 在Main.js中装进来的
                            // replace无法返回到登陆页
                            this.$router.replace('/home')
                        }
                    })
                }else{
                    this.$message.error("请输入用户名和密码")
                    return false;
                }
            });
        }
```

#### 2.配置页面跳转

新建`Home.vue`,配置router.js ,即加载对应页面:

```js
import Home from './views/Home.vue'

	{
      path: '/home',
      name: 'Home',
      component: Home
    }
```



登陆成功之后，使用`this.$router.replace('/home')`跳转到`home`页面去了，使用replace方法的原因是为了不让用户在登陆成功转到home页面后返回登陆页面。

#### 3.解决开发环境下跨域的问题(前后端端口不一致)

因为一个端口只能一个进程占用，所以要解决跨域的问题。解决办法就是：在项目目录下新建一个文件`vue.config.js`，添加下面的代码：

```js
let proxyObj={}
proxyObj['/'] = {
    ws: false,
    target: 'http://localhost:8080/',
    changeOrigin: true,
    pathRewrite:{
        '^/':''
    }
}

module.exports={
    devServer:{
        port: 8081,
        proxy: proxyObj,
		autoOpenBrowser: true
    }
}
```

这样就是前端的端口号是8081，后台的端口号为8080，项目运行之后自动打开浏览器。

#### 4.封装请求和后台向前台发送数据请求处理

就是将访问后台的接口封装成一个`js`文件，如果需要做请求的时候就调用这个文件中的方法：

```js
// axios为vue推荐的ajax
import axios from 'axios'
import { Message } from 'element-ui';

// 请求失败和成功的分别处理
axios.interceptors.response.use(success=>{
    // success.status 为http的响应码 success.data.status 为自定义的响应码
    // 业务上的错误，不是服务器错误
    if(success.status && success.status == 200 && success.data.status == 500){
        Message.error({message:success.data.msg});
        return ;
    }
    if(success.data.msg){
        //	显示的是后端发送回来的RespBean数据
        Message.success({message:success.data.msg})
    }
    return success.data;
},error=>{
    if(error.response.status == 504 || error.response.status == 404 ){
        Message.error({message:'服务器出问题了'});
    }else if(error.response.status == 403){
        Message.error({message:'权限不足'});
    }else if(error.response.status == 401){
        Message.error({message:'未登陆'});
    }else{
        if(error.response.data.message){
            Message.error({message:error.response.data.message});
        }else{
            Message.error({message:'未知问题'});
        }
    }
    return ;
})

// 下面的所有方法都是将请求封装
let base=''
export const postKeyValueRequest=(url,param)=>{
    return axios({
        // 请求方式是post
        method: 'post',
        // 不是单引号，是esc下面的那个符号
        // 传进来的url就是base+url
        url: `${base}${url}`,
        data: param,
        // 如果没有下面的这些代码的话，`data:param`将以json的形式传到服务器，而不是key-value
        // 后台登陆security只能支持key-value的形式传参
        transformRequest: [function(data){
            let ret='';
            for (let i in data) {
                // 这里的ret就是username=xxx&password=xxx&
                ret+=encodeURIComponent(i)+'='+encodeURIComponent(data[i])+'&'
            }
            return ret;
        }],
        headers:{
            'Content-Type':'application/x-www-form-urlencoded'
        }
    })
}

export const getRequest=(url,param)=>{
    return axios({
        method: 'get',
        url: `${base}${url}`,
        data: param
    })
}

export const putRequest=(url,param)=>{
    return axios({
        method: 'put',
        url: `${base}${url}`,
        data: param
    })
}

export const deleteRequest=(url,param)=>{
    return axios({
        method: 'delete',
        url: `${base}${url}`,
        data: param
    })
}

export const postRequest=(url,param)=>{
    return axios({
        method: 'post',
        url: `${base}${url}`,
        data: param
    })
}
```

后台是怎么向前台发送数据的呢：比如登陆：

```js
this.postKeyValueRequest('/dologin',this.loginForm).then(rep=>{
                        if(rep){
                            window.sessionStorage.setItem("user",JSON.stringify(rep.obj));
                            // 在Main.js中装进来的
                            // replace无法返回到登陆页
                            this.$router.replace('/home')
                        }
```

这个rep就是后台返回过来的一个`response`，里面就是装着登陆用户的信息，**简单来说**，如果想通过后端接口获取数据的话，就使用封装好的方法，比如`getRequest(url).then(data=>{})`这种形式，这样data中存放的就是后台的数据。

#### 5.模块化开发

就是新建组件说白了就是vue文件，然后在想用的页面中调用这个页面，比如我现在在项目根目录下面创建了一个目录`components`，在该目录下新建一个`PosMana.vue`文件，在`src/sys/sysbasic.vue`中的`<script>`中导入(import PosMana from ../../compenents/PosMana)，然后再将`PosMana`写到`vue`的`Component`属性中，然后就可以在上面的html页面通过`<PosMana ></PosMana >`来调用这个页面了。

#### 6.动态加载菜单页面

**注意这个只是动态加载，没有牵扯到权限的问题**

1.在后端写好了`/sys/config/menu`，这个接口

```java
package com.example.project.controller.Config;

import com.example.project.model.Menu;
import com.example.project.service.MenuService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/sys/config")
public class SystemConfigController {

    @Autowired
    MenuService menuService;

//    不能使用前端发送过来的ID
//    因为前端发送的ID就算经过验证也无法保证正确，所以使用存在内存中的ID
    @GetMapping("/menu")
    public List<Menu> getMenusByHrId(){
        return menuService.getMenusByHrId();
    }

}
```

2.在前端使用了`vuex`的技术，`vuex`就相当于一个`sessionstorage`，在服务器运行期间都可以存放值，这样就可以不用频繁的添加删除数据加重服务器的负担。

3.使用vuex先在`/store`下面新建一个js文件，这里就存放着vuex的存储空间`store`，这一步代码中的注释写的很详细

```js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

export default new Vuex.Store({
    // state是定义变量的地方
    state:{
        // 从服务器加载的路由地址都存放在这
        routes: []
    },
    // 相当于setter方法
    mutations:{
        // 只需要传data参数就可以，就是vue定义数据的地方
        initRoutes(state,data){
            state.routes=data;
        }
    },
    actions:{

    }
})
```

4.有了存储的地方，就要获取数据了，在`utils`包下新建一个`menu.js`工具包，里面有一个函数就是：首先判断vuex中是否有数据，如果有的话，就直接返回不继续执行，如果没有的话，就通过上面的后端接口来异步获取数据，获取的数据都是`json字符串`，所以如果想要转换成对应的页面的话，就得定义另一个函数：

### 4.限制在登陆之前访问项目其他网址

在java配置类中写入下面这段代码：

```java
				.exceptionHandling()
                .authenticationEntryPoint(new AuthenticationEntryPoint() {
                    @Override
                    public void commence(HttpServletRequest req, HttpServletResponse rep, AuthenticationException e) throws IOException, ServletException {
                        rep.setContentType("application/json;charset=utf-8");
                        PrintWriter writer = rep.getWriter();
                        RespBean respBean = RespBean.error("访问失败，请登陆！");
                        if(e instanceof InsufficientAuthenticationException){
                            respBean.setMsg("请求失败，请联系管理员");
                        }
                        writer.write(new ObjectMapper().writeValueAsString(respBean));
                        writer.flush();
                        writer.close();
                    }
                })
```

5.使用Postman测试接口的错误

如果在地址栏中输入了两个斜杠的话，就会报下面这个错

```json
{
    "timestamp": "2019-10-22T13:03:01.007+0000",
    "status": 500,
    "error": "Internal Server Error",
    "message": "The request was rejected because the URL was not normalized.",
    "path": "//system/basic/pos/"
}
```

### 5.后端发送JSON字符串给前端出现的问题

在`Bean`中没有Get方法的话，将无法反序列化成Json字符串

### 6.在RestController中数组的接收数据问题

如果方法的参数是一个数组，那么可以不用在参数前面加任何注解

```java
@DeleteMapping("/")
public RespBean A(Integer[] array){
    return RespBean.ok("删除成功！");
}
```

就像上面这样就可以使用`/array1=xx&array2=xx`来调用这个接口

### 7.关于put请求的返回值和servletrequest

如果请求的方式是`put`，那么如果在方法体内需要使用`servletRequest`的相关方法的时候，不能返回一个`RespBean`，只能单纯返回一个值....我也不知道为什么....

### 8.Maven项目中如何读取Properties文件以及resources文件夹下的文件

#### 1. 通过ResourceUtils来读取

不过这样做的话得在文件前面加上一个`classpath:`前缀，代码如下

```java
			file = ResourceUtils.getFile("classpath:application.properties");
            InputStream in = new FileInputStream(file);
            properties=new Properties();
            properties.load(in);
            System.out.println(properties.toString());
            in.close();
```

#### 2.通过类加载器来获取文件的输入流

这样做的方法比较的快捷简便，这里的`in`也是上面的输入流，通过类加载器封装好的方法直接读取输入流

```java
 		in = this.getClass().getClassLoader().getResourceAsStream("application.properties");
        properties=new Properties();
        assert in != null;
        properties.load(in);
        System.out.println(properties.toString());
        in.close();
```
下面的是获取`resouces`文件夹下的文件的方法，得到`file`之后，就可以使用`FileReader`，或者`FileInputStream`，来进行`IO`
```java
URL url = Objects.requireNonNull(this.getClass().getClassLoader().getResource("Hello.txt"));
File file=new File(url.getFile());
```

还有方式

```java
InputStream stream = this.getClass().getClassLoader().getResourceAsStream("Hello.txt");
byte[] bytes = new byte[1024];
BufferedInputStream inputStream = new BufferedInputStream(Objects.requireNonNull(stream));
int len;
while ((len = inputStream.read(bytes)) != -1) {
	System.out.println(new String(bytes, 0, len));
}
```



### 9.XML文件格式的解析方式

我这里使用的是Dom4j的方式来读取xml文件

1. 如果是java对象转换成xml格式的文件，有下面几个步骤

   1. 获取一个Document的对象：`Document doc= DocumentHelper.createDocument();`
   2. 创建一个根节点：`Element root = doc.addElement("xml");`
   3. 给这个根节点添加子节点，并且添加每个子节点的属性，添加子节点的方法就是`Element element = root.addElement(名称)`，而给每个子节点之间设置的文字是：`element.setText(文本内容)`
   4. 将上面的`Document`对象转换成`XML`格式的文本，使用`XMLWriter`，用法就和`Response`的`writer`的用法是一致相同的
   5. 值得一提的是，如果需要的话，可以设置成不转义，不转义就是`<`还是正常显示的

2. XML文本转换成java对象

   还是使用Dom4j的形式，步骤有下面几步：

   1. 将文件转换成输入流
   2. 使用`SAXReader`创建的对象，调用`read`方法，将输入流转换成`Document`对象
   3. 获取根节点下面的元素，`List<Element> elements = doc.getRootElement().elements();`即通过这种方式来获取所有的标签，然后就可以依次遍历，获取想要的信息了

### 10.`@ControllerAdvice`或者`@RestControllerAdvice`注解不起作用的原因

可能是在异常被抛出的时候，就当场被Catch掉了，所以没有进`@ControllerAdvice`处理

### 11.在项目中添加 Swagger2的支持

1. #### 导入Maven依赖

   ```xml
   		<dependency>
               <groupId>io.springfox</groupId>
               <artifactId>springfox-swagger2</artifactId>
               <version>2.9.2</version>
           </dependency>
           <dependency>
               <groupId>io.springfox</groupId>
               <artifactId>springfox-swagger-ui</artifactId>
               <version>2.9.2</version>
           </dependency>
   ```

2. #### 配置swagger2

   ```java
   @Configuration
   @EnableSwagger2
   public class Swagger2Config {
   
       @Bean
       public Docket CreateApiDescription(){
           return new Docket(DocumentationType.SWAGGER_2)
                   .pathMapping("/")
                   .select()
                   .apis(RequestHandlerSelectors.basePackage("com.kyriexu"))
                   .paths(PathSelectors.any())
                   .build().apiInfo(new ApiInfoBuilder().title("Api文档").description("Api文档").build());
       }
   }
   ```

   说白了就是创建一个类型为Docket的bean，然后还要开启swagger2，将这个类设置成一个配置类，让spring能够扫描到

3. #### 配置接口的作用和实体类的属性作用

   下面是配置接口的作用的类，由于我这里是没有接收参数的，所以单独的说一下怎么设置接收参数的作用：

   接收参数的设置是使用` @ApiImplicitParam`，如果有多个参数，则需要使用`@ApiImplicitParams`将前面的参数集合起来

   ```java
   @ApiImplicitParams({
               @ApiImplicitParam(name = "username", value = "用户名", defaultValue = "李四"),
               @ApiImplicitParam(name = "address", value = "用户地址", defaultValue = "深圳", required = true)
       })
   ```

   形式就像上面描述的一样

   ```java
   package com.kyriexu.Controller;
   
   import com.kyriexu.model.RespBean;
   import io.swagger.annotations.Api;
   import io.swagger.annotations.ApiOperation;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RestController;
   
   @RestController
   @Api("测试权限使用的接口")
   public class HelloController {
   
       @GetMapping("/hello")
       @ApiOperation("所有用户都可以访问的接口，只需要登陆")
       public RespBean hello(){
           return RespBean.ok("hello");
       }
   
       @ApiOperation("只有管理员才能访问的接口")
       @GetMapping("/admin/hello")
       public RespBean admin(){
           return RespBean.ok("管理员，你好");
       }
   
       @ApiOperation("管理员和老师能访问的接口")
       @GetMapping("/teacher/hello")
       public RespBean teacher(){
           return RespBean.ok("老师，你好");
       }
   
       @ApiOperation("管理员老师学生能访问的接口")
       @GetMapping("/student/hello")
       public RespBean student(){
           return RespBean.ok("学生，你好");
       }
   }
   ```

   而实体类的作用配置如下：

   简单的说一下，首先必须要在类上添加一个注解`@ApiModel`，让swagger知道这是一个实体类，然后再给各个属性设置`@ApiModelProperty`

   ```java
   package com.kyriexu.model;
   
   import io.swagger.annotations.ApiModel;
   import io.swagger.annotations.ApiModelProperty;
   import lombok.Data;
   import lombok.ToString;
   
   @Data
   @ToString
   @ApiModel
   public class RespBean {
       @ApiModelProperty("状态码")
       private int status;
   
       @ApiModelProperty("后端传给前端的信息")
       private String msg;
   
       @ApiModelProperty("后端传给前端的对象")
       private Object object;
   
       public RespBean(int status, String msg) {
           this.status = status;
           this.msg = msg;
       }
   
       public RespBean() {
       }
   
       public RespBean(int status, String msg, Object object) {
           this.status = status;
           this.msg = msg;
           this.object = object;
       }
   
       public static RespBean error(String msg){
           return new RespBean(500,msg);
       }
   
       public static RespBean error(String msg,Object o){
           return new RespBean(500,msg,o);
       }
   
       public static RespBean ok(String msg){
           return new RespBean(200,msg);
       }
   
       public static RespBean ok(String msg,Object o){
           return new RespBean(200,msg,o);
       }
   }
   ```


### 12.在springboot整合springsecurity的时候获取登陆用户的信息

收到spring源码的影响，知道保存bean的信息的类是`BeanDefinitionHolder`，这里也是一样的通过下面的这个代码获取信息：

```java
        User user = (User) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
```

首先先获取上下文，然后获取认证的信息，再获取认证的对象，强制转换成为`User`对象，进而获取到登陆用户的信息，然后还有两种办法就是：

```java
	@ApiOperation("测试接口1：测试能否正确获取已登陆对象的信息；测试通过")
    @GetMapping("/test1")
    public Integer getAuthentication(){
        User user = (User) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        return user.getId();
    }

    @ApiOperation("测试接口2：测试能否正确获取已登陆对象的信息；测试通过")
    @GetMapping("/test2")
    public Integer getAuthentication1(Authentication authentication){
        User user = (User) authentication.getPrincipal();
        return user.getId();
    }

    @ApiOperation("测试接口3：测试能否正确获取已登陆对象的信息；测试通过")
    @GetMapping("/test3")
    public Integer getAuthentication2(@AuthenticationPrincipal User user){
        return user.getId();
    }
```

最后一种办法（应该是全部的方式）都要让实体类User实现`UserDetails`接口才能正常运作

### 13.在项目中开启Mybatis打印sql语句的配置

```yaml
logging:
  level:
    com.kyriexu.mapper:
      debug
```

```properties
logging.level.com.kyriexu.mapper=debug
```

### 14.前端传多个数组给后端的方式

```java
	@PostMapping("/testMultiParam")
    public String recv(String[] ids,Integer status){
        logger.info(String.valueOf(status));
        StringBuilder sb = new StringBuilder();
        for (String id : ids) {
            sb.append(id);
            logger.info(id);
        }
        return sb.toString();
    }
```

上面这种方式的传参的方法是：

![传参方法.png](https://i.loli.net/2020/01/25/OjP3SoKJGcTrV12.png)

前端的ajax请求应该写成：

```javascript
data=ids=xxx&ids=xxx&status=xxx
$.ajax({
    type: 'delete',
	contentType: "application/x-www-form-urlencoded",
    data: data,
    url: '/order/deleteCartItem',
    success: function (rep) {
        // do something
 }
})
```

### 15.SpringMVC的NativeWebRequest是什么

 抽象了各种request，如果要获得实际真正的request，则调用getNativeRequest()；或者getNativeRequest(XXX.class)； 

```java
HttpServletRequest req = webRequest.getNativeRequest(HttpServletRequest.class);
HttpServletResponse resp = webRequest.getNativeResponse(HttpServletResponse.class);
```

### 16.springboot项目在使用@PropertiesSource以及@ConfigurationProperties出现的问题

如果出现：

![springboot 配置文件.png](https://i.loli.net/2020/01/25/OFghHSl3GczDP1r.png)

则在项目中添加依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
```
