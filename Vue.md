## 1.Vue项目搭建

使用`Vue-Cli`快速搭建Vue项目：

`Vue Create <YourProjectName>`然后会有两个选项，一个选项就是选择默认配置，而另一个选项就是自己手动筛选项目添加的依赖包。

这个项目首先选择了`vue-router`和`babel`，创建完成之后，删除原有的页面，新建一个登陆文件：`login.vue`，此时项目的基本框架搭好了。

## 2.使用ElementUI

首先导入`Element`，官网上面都有介绍，`npm install element`，然后在`main.js`中`import`相应的包，

```
import ElementUI from 'element-ui'
import 'element-ui/lib/theme-chalk/index.css'

Vue.use(ElementUI)
```

这样就可以使用element中的组件了

## 3.前后端接口对接

使用`axios`来进行vue前端和java后端的接口对接

使用一个`js`文件封装处理后台传来的数据

```js
axios.interceptors.response.use(success=>{
    // success.status 为http的响应码 success.data.status 为自定义的响应码
    // 业务上的错误，不是服务器错误
    if(success.status && success.status == 200 && success.data.status == 500){
        Message.error({message:success.data.msg});
        return ;
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
```

