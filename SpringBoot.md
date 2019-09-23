

### JSON的一些注解

1. `@Jsonproperties`的作用是给属性起一个别名，如果使用了这个注解那前端传回来的参数必须使用这个别名传回来。
2. `@Jsonignore`在传值的时候自动忽略加上这个注解的属性，传回来为空。
3. `@JsonPropertyOrder`改变传值到前端的值的顺序。





### @Responsebody和@RestController的作用

对`json`格式的数据的响应。

1. `@Responsebody`的作用是用于接收和响应序列化数据（JSON），可以支持嵌套JSON数据结构。
2. `@RestController`它有两层含义：
   * 作为控制器注入到Spring上下文环境。
   * 请求响应为数据序列化（默认序列化方式是JSON），而不是跳转到html或模板页面。



### @HttpMessageConverter的作用：

1. 将服务器端返回的对象序列化成`Json`格式的数据
2. 将前端返回的`Json`数据反序列成JAVA对象

SpringMVC自动配置了`Jackson`和`Gson`的`HttpMessageConverter`，springboot进行了自动化配置，操作`Json`离不开`HttpMessageConverter`

因为`@ConditionalOnMissBean`的注解的关系，如果没有自动配置`HttpMessageConverter`的话，将会自动采用springboot自动化配置的`HttpMessageConverter`。



### SpringBoot文件上传

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

### 多文件上传

使用`MultipartFile`数组，使用循环接用单文件上传的方法。



### @ControllerAdvice的相关知识

##### 处理全局异常

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

##### 预设全局数据

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

##### 请求参数预处理

###### 问题的产生

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