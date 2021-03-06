## 01、课程目标

- 会使用nginx进行反向代理
- 实现商品分类查询功能
- 了解跨域解决方案
- 掌握cors解决跨域





## 02、乐优微服务：后端返回前端数据问题

我们项目是前后端分离的项目，后端在处理过程需要给前端返回数据结果，这里分两种情况，一种是正常情况（也就是没有发生异常），另一种是发生异常的情况。我们希望不管是正常情况，还是异常情况下，后端都可以给前端不同的响应状态码以及响应内容，这样有利于前端根据不同情况进行业务处理。

### 1）模拟场景

我们预设这样一个场景，假如我们模拟新增商品，只传一个id，如果id为1则抛出异常。

### 2）代码

在ly-item中编写模拟的service：

```java
package com.leyou.item.service;

import org.springframework.stereotype.Service;

@Service
public class ItemService {
    
    public Long saveItem(Long id){
        // 模拟添加操作，如果id为1抛异常
        if(id.equals(1L)){
            throw new RuntimeException("Id不能为1！");
        }
        return id;
    }

}
```

模拟controller：

```java
@RestController
public class ItemController {
    @Autowired
    private ItemService itemService;

    @PostMapping("/save")
    public Long saveItem(@RequestParam("id") Long id){
        return itemService.saveItem(id);
    }

}
```

### 3）测试

请求

![image-20200312091743947](assets/image-20200312091743947.png)

结果

![image-20200312091804543](assets/image-20200312091804543.png)



经过测试，我们发现，在正常情况下只能返回200状态码，在异常情况下只能返回500状态码。显然这不满足我们的需求！接下来分两种情况进行讨论。



## 03、乐优微服务：正常情况数据返回

我们可以通过spring提供的状态码来给前端返回相应的状态码，具体操作如下：

```java
package com.leyou.item.controller;

import com.leyou.item.service.ItemService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import javax.servlet.http.HttpServletResponse;

@RestController
public class ItemController {
    @Autowired
    private ItemService itemService;

    /**
     * ResponseEntity类
     *   1）status：修改响应状态码
     *   2）body： 修改响应正文
     * @param id
     * @return
     */
    @PostMapping("/save")
    public ResponseEntity<Long> saveItem(@RequestParam("id") Long id){
        return ResponseEntity.status(HttpStatus.CREATED).body(itemService.saveItem(id));
    }

}
```

![1603501096362](assets/1603501096362.png)



## 04、乐优微服务：异常情况数据返回

### 1）场景说明

依然使用上面第二步模拟的案例，参数传1，结果如下：

![image-20200312093002970](assets/image-20200312093002970.png)

此刻是spring给我们封装了一个异常对象，并返回给了客户端，但是这个数据一般企业都希望自己来封装。因为这里所有异常的状态码都是500，我们希望错误的状态码更细致一些，换句话来说，就是我们希望能够自己指定异常时的状态码。而RuntimeException异常又无法自定义状态码，只有自己定义一个异常类了。

### 2）自定义异常

首先我们先在公共子模块中定义一个乐优异常类，这样每个模块都可以共用这个异常类。

我们在ly-common模块的 com.leyou.common.exception.pojo包下创建如下异常类

```java
package com.leyou.common.exception.pojo;

import lombok.Getter;

/**
 * 自定义异常类，封装自定义异常信息
 */
@Getter
public class LyException extends RuntimeException{
    private Integer status;

    public LyException(Integer status,String message){
        super(message);
        this.status = status;
    }

}


```



### 3）改造ly-item-service

我们把原来的`throw new RuntimeException`改为如下：

```java
@Service
public class ItemService {
    
    public Long saveItem(Long id){
        // 模拟添加操作，如果id为1抛异常
        if(id.equals(1L)){
            throw new LyException(501, "Id不能为1！");
        }
        return id;
    }
}   
```

此刻我们就可以指定自己需要的状态码了

**==问题：==**

再次测试

![image-20200312093854966](assets/image-20200312093854966.png)

发现状态码并没有改变。

可以明白，spring并不认识我们的LyException，它依然把LyException当成了来RuntimeException处理了。

所以我们要在common模块中自定义拦截异常的处理器类。



### 4）全局异常拦截器

接下来，我们使用SpringMVC提供的统一异常拦截器，因为是统一处理，我们放到`ly-common`项目中：

```java
package com.leyou.common.exception.controller;

import com.leyou.common.exception.pojo.LyException;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

/**
 * 全局自定义异常拦截器（处理器）
 */
@ControllerAdvice  // 就会覆盖SpringMVC自带异常处理
public class LyExceptionController {

    /**
     * 定义异常处理定义
     */
    @ExceptionHandler(value = LyException.class) // 定义需要捕获什么异常
    public ResponseEntity<LyException> handlerException(LyException e) {
        return ResponseEntity.status(e.getStatus()).body(e);
    }
}


```

然后测试

![image-20200312095034400](assets/image-20200312095034400.png) 



但是，虽然我们要的信息都有了，但是返回的异常信息太多太多，很多都是框架的信息，前端根本用不到这些信息，如果我们将这些信息返回，那么响应的数据多了，一定程度上响应了数据的响应速度，所以我们需要进一步优化。





### 5）自定义异常结果类

为了让异常结果更友好，我们不在LyExceptionController返回LyException，而是自定义一个响应的类，这个类只包含异常响应的结果。

提供返回异常信息的自定义类

```java
package com.leyou.common.exception.pojo;

import lombok.Getter;
import org.joda.time.DateTime;

/**
 * 异常结果类，封装异常结果信息
 */
@Getter
public class ExceptionResult {
    private Integer status; //异常状态码
    private String message; //异常消息
    private String timestamp;//异常发生时间

    public ExceptionResult(LyException e){
        this.status = e.getStatus();
        this.message = e.getMessage();
        this.timestamp = DateTime.now().toString("yyyy-MM-dd HH:mm:ss");
    }

}


```

定义好类后，我们改造全局异常处理器类：

```java
package com.leyou.common.exception.controller;

import com.leyou.common.exception.pojo.ExceptionResult;
import com.leyou.common.exception.pojo.LyException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;

/**
 * 全局异常拦截
 */
@ControllerAdvice  //该注解会把所有Controller的异常拦截
public class LyExceptionController {

    /**
     * 拦截具体异常方法
     */
    @ExceptionHandler(value = LyException.class)
    @ResponseBody
    public ResponseEntity<ExceptionResult> handlerException(LyException e){
        return ResponseEntity.status(e.getStatus()).body(new ExceptionResult(e));
    }


}

```



重启微服务，然后测试

![image-20200312095541899](assets/image-20200312095541899.png) 

到这一步，我们的全局异常问题已经解决了，要的响应码和响应内容都是我们自定义的了。



但是还有一个问题，在实际开发中，我们的异常是很多的，而且这些异常是和前端约定好的，但现在异常响应码和异常响应内容都是写死在代码中，如果某一个响应码或者响应内容需要修改，那么所有用到这个响应码和响应内容的地方都要跟着改，很不方便，会造成很多重复的工作，所以还需要进一步优化。





### 6）自定义异常枚举

因为现在的异常响应信息包含多个内容，所以只用字符串就很难满足我们的需求，所以我们需要定义一个类来装这些信息，但是异常信息其实都是提前约定好的，所以用枚举比较符合我们需求。



枚举：把一件事情的所有可能性列举出来。在计算机中，枚举也可以叫多例，单例是多例的一种情况 。

单例：一个类只能有一个实例。

多例：一个类只能有有限个数的实例。



单例的实现：

- 私有化构造函数
- 在成员变量中初始化本类对象
- 对外提供静态方法，访问这个对象

 

枚举代码：

```java
package com.leyou.common.exception.pojo;

import lombok.Getter;

/**
 * 黑马程序员
 */
@Getter
public enum ExceptionEnum {
    INVALID_FILE_TYPE(400, "无效的文件类型！"),
    INVALID_PARAM_ERROR(400, "无效的请求参数！"),
    INVALID_PHONE_NUMBER(400, "无效的手机号码"),
    INVALID_VERIFY_CODE(400, "验证码错误！"),
    INVALID_USERNAME_PASSWORD(400, "无效的用户名和密码！"),
    INVALID_SERVER_ID_SECRET(400, "无效的服务id和密钥！"),
    INVALID_NOTIFY_PARAM(400, "回调参数有误！"),
    INVALID_NOTIFY_SIGN(400, "回调签名有误！"),

    CATEGORY_NOT_FOUND(404, "商品分类不存在！"),
    BRAND_NOT_FOUND(404, "品牌不存在！"),
    SPEC_NOT_FOUND(404, "规格不存在！"),
    GOODS_NOT_FOUND(404, "商品不存在！"),
    CARTS_NOT_FOUND(404, "购物车不存在！"),
    APPLICATION_NOT_FOUND(404, "应用不存在！"),
    ORDER_NOT_FOUND(404, "订单不存在！"),
    ORDER_DETAIL_NOT_FOUND(404, "订单数据不存在！"),

    DATA_TRANSFER_ERROR(500, "数据转换异常！"),
    INSERT_OPERATION_FAIL(500, "新增操作失败！"),
    UPDATE_OPERATION_FAIL(500, "更新操作失败！"),
    DELETE_OPERATION_FAIL(500, "删除操作失败！"),
    FILE_UPLOAD_ERROR(500, "文件上传失败！"),
    DIRECTORY_WRITER_ERROR(500, "目录写入失败！"),
    FILE_WRITER_ERROR(500, "文件写入失败！"),
    SEND_MESSAGE_ERROR(500, "短信发送失败！"),
    INVALID_ORDER_STATUS(500, "订单状态不正确！"),
    STOCK_NOT_ENOUGH_ERROR(500, "库存不足！"),

    UNAUTHORIZED(401, "登录失效或未登录！");

    private int status;
    private String message;

    ExceptionEnum(int status, String message) {
        this.status = status;
        this.message = message;
    }
}
```



**改造LyException类**

```java
package com.leyou.common.exception.pojo;

import lombok.Getter;

/**
 * 自定义异常类，封装自定义异常信息
 */
@Getter
public class LyException extends RuntimeException{
    private Integer status;

    public LyException(Integer status,String message){
        super(message);
        this.status = status;
    }

    public LyException(ExceptionEnum exceptionEnum){
        super(exceptionEnum.getMessage());
        this.status = exceptionEnum.getStatus();
    }
}


```

 

**改造ly-item-service**

```java
package com.leyou.item.service;

import com.leyou.common.exception.pojo.ExceptionEnum;
import com.leyou.common.exception.pojo.LyException;
import org.springframework.stereotype.Service;

@Service
public class ItemService {
    
    public Long saveItem(Long id){
        // 模拟添加操作，如果id为1抛异常
        if(id.equals(1L)){
            //throw new RuntimeException("Id不能为1！");
            throw new LyException(ExceptionEnum.UNAUTHORIZED);
        }
        return id;
    }

}
```

 

## 05、解决域名解析问题

### 1）统一环境

我们现在访问页面使用的是：http://localhost:8081

有没有什么问题？

实际开发中，会有不同的环境：

- 开发环境：自己的电脑
- 测试环境：提供给测试人员使用的环境
- 预发布环境：数据是和生成环境的数据一致，运行最新的项目代码进去测试
- 生产环境：项目最终发布上线的环境

如果不同环境使用不同的ip去访问，可能会出现一些问题。为了保证所有环境的一致，我们会在各种环境下都使用域名来访问。

我们将使用以下域名：

- 主域名是：www.leyou.com，
- 管理系统域名：manage.leyou.com
- 网关域名：api.leyou.com
- ...

但是最终，我们希望这些域名指向的还是我们本机的某个端口。

那么，当我们在浏览器输入一个域名时，浏览器是如何找到对应服务的ip和端口的呢？



域名要访问，先从==浏览器缓存==中解析，然后从==本地host文件==解析，最后去==域名根服务器==解析域名。



### 2）域名解析

一个域名一定会被解析为一个或多个ip。这一般会包含两步：本地host和万网两步，如果任何一步解析成功了，就不会继续往下解析了。

- 本地域名解析

  浏览器会首先在本机的hosts文件中查找域名映射的IP地址，如果查找到就返回IP ，没找到则进行域名服务器解析，一般本地解析都会失败，因为默认这个文件是空的。

  - Windows下的hosts文件地址：C:\Windows\System32\drivers\etc
  - Linux下的hosts文件所在路径： /etc/hosts 

- 域名服务器解析

  本地解析失败，才会进行域名服务器解析，域名服务器就是网络中的一台计算机，里面记录了所有注册备案的域名和ip映射关系，一般只要域名是正确的，并且备案通过，一定能找到。



### 3）解决域名解析问题

我们不可能去购买一个域名，因此我们可以伪造本地的hosts文件，实现对域名的解析。修改本地的host为：

```
127.0.0.1 www.leyou.com
127.0.0.1 api.leyou.com
127.0.0.1 manage.leyou.com
127.0.0.1 image.leyou.com
```

这样就实现了域名的关系映射了。

解压资料中的SwitchHosts软件

![image-20200312102912295](assets/image-20200312102912295.png)

 

 将上面提供号的服务名称配置到host文件中去

![image-20200312102942279](assets/image-20200312102942279.png)

我们添加了四个映射关系：

- 127.0.0.1 api.leyou.com           ：我们的网关Zuul
- 127.0.0.1 manage.leyou.com  ：我们的后台系统地址
- 127.0.0.1 www.leyou.com        : 这个是网站主页
- 127.0.0.1 image.leyou.com      : 这个是图片地址

现在，ping一下域名试试是否畅通：

![1574933988404](assets/1574933988404.png) 

OK了！ 





## 06、Nginx的简介

虽然域名解决了，但是现在如果我们要访问，还得自己加上端口：`http://manage.leyou.com:8081`。

这就不够优雅了。我们希望的是直接域名访问：`http://manage.leyou.com`。这种情况下端口默认是80，如何才能把请求转移到9001端口呢？

这里就要用到反向代理工具：Nginx

### 1）什么是Nginx

![1526187409033](assets/1526187409033.png) 



nginx可以作为web服务器，但更多的时候，我们把它作为网关，因为它具备网关必备的功能：

- 反向代理
- 负载均衡
- 动态路由
- 请求过滤

总结NGINX有两大核心功能：第一，可以做为静态资源服务器来使用，并发和性能远高于tomcat。第二个功能就是做反向代理服务器。

### 2）Nginx作为web服务器

Web服务器分2类：

- web应用服务器（动态服务器），如：
  - tomcat
  - resin
  - jetty
- web服务器（静态服务器），如：
  - Apache 服务器（.httpaccess）
  - Nginx
  - IIS

区分：web服务器不能解析jsp等页面，只能处理js、css、html等静态资源。
并发：web服务器的并发能力远高于web应用服务器。



### 3）Nginx作为反向代理

什么是反向代理？

- 代理：通过客户机的配置，实现让一台服务器代理客户机，客户的所有请求都交给代理服务器处理。
- 反向代理：用一台服务器，代理真实服务器，用户访问时，不再是访问真实服务器，而是代理服务器。

nginx可以当做反向代理服务器来使用：

- 我们需要提前在nginx中配置好反向代理的规则，不同的请求，交给不同的真实服务器处理
- 当请求到达nginx，nginx会根据已经定义的规则进行请求的转发，从而实现路由功能



利用反向代理，就可以解决我们前面所说的端口问题，如图 

如果是安装在本机：

![1526016663674](assets/1526016663674.png) 

 

正向代理即是`客户端代理`, 代理客户端，服务端不知道实际发起请求的客户端 。VPN

反向代理即是`服务端代理`, 代理服务端，客户端不知道实际提供服务的服务端。

![1600251871684](assets/1600251871684.png)



![1600251879982](assets/1600251879982.png)



## 07、Nginx的反向代理配置

先安装Nginx，安装非常简单，把课前资料提供的nginx直接解压即可，绿色免安装，舒服！

![img](assets/xiaoyaya.gif) 



把软件解压到本地某个目录即可，目录结构：

![1574996389482](assets/1574996389482.png) 



> ### 使用

nginx可以通过命令行来启动，操作命令：

- 启动：`start nginx.exe`
- 停止：`nginx.exe -s stop`
- 重新加载：`nginx.exe -s reload`

我们需要让nginx反向代理我们的服务器，因此需要自定义nginx配置。

![1574996542375](assets/1574996542375.png) 

为了看起来清爽，我们先在nginx主配置文件`nginx.conf`中使用include指令引用我们的配置：

```
	include vhost/*.conf;
```

如图所示：

![1574996632209](assets/1574996632209.png) 

然后在nginx.conf所在目录新建文件夹vhost：

![1574996714707](assets/1574996714707.png) 

并在vhost中创建文件leyou.conf：

![1574997008925](assets/1574997008925.png) 

填写如下配置：

```nginx
upstream leyou-manage{
	server	127.0.0.1:9001;
}
upstream leyou-gateway{
	server	127.0.0.1:10010;
}
upstream leyou-portal{
	server	127.0.0.1:9002;
}

server {
	listen       80;
	server_name  manage.leyou.com;
	
	location / {
	    proxy_pass   http://leyou-manage;
		proxy_connect_timeout 600;
		proxy_read_timeout 5000;
	}
}
server {
	listen       80;
	server_name  www.leyou.com;
	
	location / {
	    proxy_pass   http://leyou-portal;
		proxy_connect_timeout 600;
		proxy_read_timeout 5000;
	}
}
server {
	listen       80;
	server_name  api.leyou.com;
	
	location / {
	    proxy_pass   http://leyou-gateway;
		proxy_connect_timeout 600;
		proxy_read_timeout 5000;
	}
}
```

解读：

- upstream：定义一个负载均衡集群，例如leyou-manage
  - server：集群中某个节点的ip和port信息，可以配置多个，实现负载均衡，默认轮询
- server：定义一个监听服务配置
  - listen：监听的断开
  - server_name：监听的域名
  - location：匹配当前域名下的哪个路径。例如：`/`，代表的是一切路径
    - proxy_pass：监听并匹配成功后，反向代理的目的地，可以指向某个ip和port，或者指向upstream定义的负载均衡集群，nginx反向代理时会轮询中服务列表中选择。





> ### 使用域名访问测试

重载nginx，然后用域名访问后台管理系统：

![1574997922919](assets/1574997922919.png) 

现在实现了域名访问网站了，中间的流程是怎样的呢？

![1575000188871](assets/1575000188871.png) 

1. 浏览器准备发起请求，访问http://mamage.leyou.com，但需要进行域名解析

2. 优先进行本地域名解析，因为我们修改了hosts，所以解析成功，得到地址：127.0.0.1（本机）

3. 请求被发往解析得到的ip，并且默认使用80端口：http://127.0.0.1:80

   本机的nginx一直监听80端口，因此捕获这个请求

4. nginx中配置了反向代理规则，将manage.leyou.com代理到http://127.0.0.1:9001

5. 主机上的后台系统的webpack server监听的端口是9001，得到请求并处理，完成后将响应返回到nginx

6. nginx将得到的结果返回到浏览器





## 08、表关系分析：商品-分类-品牌

商城的核心自然是商品，而商品多了以后，肯定要进行分类，并且不同的商品会有不同的品牌信息，其关系如图所示：

![1525999005260](assets/1525999005260.png)

- 一个商品分类下有很多商品
- 一个商品分类下有很多品牌
- 而一个品牌，可能属于不同的分类
- 一个品牌下也会有很多商品



因此，我们需要依次去完成：商品分类、品牌、商品规格、商品的开发。



## 09、商品分类：页面分析

### 1）页面分析

首先我们看下要实现的效果：

![1575013720541](assets/1575013720541.png) 

商品分类之间是会有层级关系的，采用树结构去展示是最直观的方式。

一起来看页面，对应的是/pages/item/Category.vue：

![1575013837176](assets/1575013837176.png) 

页面模板：

```html
<template>
  <v-card>
    <v-flex xs12 sm10>
      <v-tree url="/item/category/of/parent" isEdit @handAdd="handleAdd"></v-tree>
    </v-flex>
  </v-card>
</template>
```

- `v-card`：卡片，是vuetify中提供的组件，提供一个悬浮效果的面板，一般用来展示一组数据。

  ![1575014301065](assets/1575014301065.png) 

- `v-flex`：布局容器，用来控制响应式布局。与BootStrap的栅格系统类似，整个屏幕被分为12格。我们可以控制所占的格数来控制宽度：

  ![1526001573140](assets/1526001573140.png) 

  本例中，我们用`sm10`控制在小屏幕及以上时，显示宽度为10格

- `v-tree`：树组件。Vuetify并没有提供树组件，这个是我们自己编写的自定义组件：

  ![1575013978201](assets/1575013978201.png) 

  里面涉及一些vue的高级用法，大家暂时不要关注其源码，会用即可。



### 2）树组件的用法

也可参考课前资料中的：《自定义Vue组件的用法.md》



这里我贴出树组件的用法指南。

> 属性列表：

| 属性名称 | 说明                             | 数据类型 | 默认值 |
| :------- | :------------------------------- | :------- | :----- |
| url      | 用来加载数据的地址，即延迟加载   | String   | -      |
| isEdit   | 是否开启树的编辑功能             | boolean  | false  |
| treeData | 整颗树数据，这样就不用远程加载了 | Array    | -      |

这里推荐使用url进行延迟加载，**每当点击父节点时，就会发起请求，根据父节点id查询子节点信息**。

当有treeData属性时，就不会触发url加载

远程请求返回的结果格式：

```json
[
    { 
        "id": 74,
        "name": "手机",
        "parentId": 0,
        "isParent": true,
        "sort": 2
	},
     { 
        "id": 75,
        "name": "家用电器",
        "parentId": 0,
        "isParent": true,
        "sort": 3
	}
]
```



> 事件：

| 事件名称     | 说明                                       | 回调参数                                         |
| :----------- | :----------------------------------------- | :----------------------------------------------- |
| handleAdd    | 新增节点时触发，isEdit为true时有效         | 新增节点node对象，包含属性：name、parentId和sort |
| handleEdit   | 当某个节点被编辑后触发，isEdit为true时有效 | 被编辑节点的id和name                             |
| handleDelete | 当删除节点时触发，isEdit为true时有效       | 被删除节点的id                                   |
| handleClick  | 点击某节点时触发                           | 被点击节点的node对象,包含全部信息                |

> 完整node的信息

回调函数中返回完整的node节点会包含以下数据：

```json
{
    "id": 76, // 节点id
    "name": "手机", // 节点名称
    "parentId": 75, // 父节点id
    "isParent": false, // 是否是父节点
    "sort": 1, // 顺序
    "path": ["手机", "手机通讯", "手机"] // 所有父节点的名称数组
}
```





### 3）树组件与后台交互

给大家的页面中已经配置了url，我们刷新页面看看会发生什么：

```html
<v-tree url="/item/category/of/parent"
        :isEdit="isEdit"
        @handleAdd="handleAdd"
        @handleEdit="handleEdit"
        @handleDelete="handleDelete"
        @handleClick="handleClick"
        />
```

刷新页面，可以看到：

 ![1552992441349](assets/1552992441349.png)

页面发起了一条请求：http://api.leyou.com/api/item/category/of/parent?pid=0 



大家可能会觉得很奇怪，我们明明是使用的相对路径，讲道理发起的请求地址应该是：

http://manage.leyou.com/item/category/of/parent

但实际却是：

http://api.leyou.com/api/item/category/of/parent?pid=0 

这是因为，我们有一个全局的配置文件，对所有的请求路径进行了约定：

![1575015552439](assets/1575015552439.png) 

里面已经定义了api的前缀：

![1575015635649](assets/1575015635649.png) 

路径是http://api.leyou.com/api，因此页面发起的一切请求都会以这个路径为前缀。

而我们的Nginx又完成了反向代理，将这个地址代理到了http://127.0.0.1:10010，也就是我们的Gateway网关，最终再被Gateway转到微服务，由微服务来完成请求处理并返回结果。



## 10、商品分类：使用Axios与微服务交互

axios官网：https://github.com/axios/axios



### 1）引入Axios

package.json引入依赖：

![1575266598731](assets/1575266598731.png) 

在js文件中引入：

![1575266690896](assets/1575266690896.png) 



### 2）get请求

```js
//第一种：可以把参数直接写到url的?后面
axios.get('/user?ID=12345')
  .then(function (response) {
    // 成功执行
    console.log(response);
  })
  .catch(function (error) {
    // 失败执行
    console.log(error);
  })
  .finally(function () {
    // 总是执行
  });




// 第二种：get请求传参也可以使用下面这种方式
axios.get('/user', {
    params: {
      ID: 12345
    }
  })
  .then(function (response) {
    //成功执行
    console.log(response);
  })
  .catch(function (error) {
    //失败执行
    console.log(error);
  })
  .finally(function () {
    //总是执行
  });  
```





### 3）post请求

```js
//post请求
axios.post('/user', {
    firstName: 'Fred',
    lastName: 'Flintstone'
  })
  .then(function (response) {
    //成功执行
    console.log(response);
  })
  .catch(function (error) {
    //失败执行
    console.log(error);
  });
```





### 4）设置baseURL地址

baseURL就是一个基本url地址，所有请求都会先拼接上baseURL。

![1575267208993](assets/1575267208993.png) 



### 5）把Axios加入Vue的全局属性中

以后我们会有很多个vue页面，每个页面如果都引入axios，那很麻烦，我们想要的是一次引入多次使用，怎么实现？



我们可以在Vue中加入一个我们自定义的属性，然后调用即可，怎么在一个js对象中加入一个自定义属性呢？靠的是js的prototype属性。

![1575267492933](assets/1575267492933.png) 



我们在http.js中直接添加一个http属性，以后直接调用即可：

![1575267582669](assets/1575267582669.png) 

然后在vue页面中就可以直接调用了：

![1575267683266](assets/1575267683266.png) 







## 11、商品分类：后台代码准备工作

### 1）编写分类实体类

我们在ly-pojo-item模块中添加Category实体类

```java
package com.leyou.item.pojo;

import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import lombok.Data;

import java.util.Date;

/**
 * 分类
 */
@Data
@TableName("tb_category")
public class Category {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String name;
    private Long parentId;
    private Boolean isParent;
    private Integer sort;
    private Date createTime;
    private Date updateTime;
}

```



### 2）编写分类mapper

```java
package com.leyou.item.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.leyou.item.pojo.Category;

/**
 * 分类Mapper
 */
public interface CategoryMapper extends BaseMapper<Category>{
}

```

### 3）编写分类Service

```java
package com.leyou.item.service;

import com.leyou.item.mapper.CategoryMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * 分类Service
 */
@Service
@Transactional
public class CategoryService {
    @Autowired
    private CategoryMapper categoryMapper;

    
    
}

```



### 4）编写分类Controller

```java
package com.leyou.item.controller;

import com.leyou.item.service.CategoryService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RestController;

/**
 * 分类控制器
 */
@RestController
public class CategoryController {
    @Autowired
    private CategoryService categoryService;
}

```



## 12、商品分类：根据父id查询分类列表

### 1）提供处理器

```java
package com.leyou.item.controller;

import com.leyou.item.pojo.Category;
import com.leyou.item.service.CategoryService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

/**
 * 分类控制器
 */
@RestController
public class CategoryController {
    @Autowired
    private CategoryService categoryService;

    /**
     * 根据父id查询分类
     */
    @GetMapping("/category/of/parent")
    public ResponseEntity<List<Category>> findCategoriesById(@RequestParam("pid") Long pid){
        List<Category> categories = categoryService.findCategoriesById(pid);
        //return ResponseEntity.status(HttpStatus.OK).body(categories);
        return ResponseEntity.ok(categories);
    }
}



```

### 2）提供service

```java
package com.leyou.item.service;

import com.baomidou.mybatisplus.core.conditions.query.QueryWrapper;
import com.baomidou.mybatisplus.core.toolkit.CollectionUtils;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.leyou.common.exception.pojo.ExceptionEnum;
import com.leyou.common.exception.pojo.LyException;
import com.leyou.item.mapper.CategoryMapper;
import com.leyou.item.pojo.Category;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

/**
 * 分类Service
 */
@Service
@Transactional
public class CategoryService {
    @Autowired
    private CategoryMapper categoryMapper;


    public List<Category> findCategoriesById(Long pid) {
        //1.构建查询条件
        /**
         * QueryWrapper: 用于封装所有查询条件
         *     QueryWrapper对象构建：
         *         1）QueryWrapper query = Wrappers.query() 无参   自定义查询方式。例如 同时分页+条件判断等
         *         2）QueryWrapper query = Wrappers.query(T) 有参  简单查询。 例如 根据用户名查询
         */
        Category category = new Category();
        category.setParentId(pid);//设置条件
        
        QueryWrapper wrapper = Wrappers.query(category);

        //2.执行查询
        List<Category> categories = categoryMapper.selectList(wrapper);
        
        //3.处理结果
        if(CollectionUtils.isEmpty(categories)){
            throw new LyException(ExceptionEnum.CATEGORY_NOT_FOUND);
        }

        //4.返回结果
        return categories;
    }
}



```

### 3）启动并测试

我们不经过网关，直接访问：

![1575024871135](assets/1575024871135.png) 

然后试试网关是否畅通：

如果网关不通，记得修改下ly-gateway中的application.yml中的routes

![1575024967459](assets/1575024967459.png) 

一切OK！

然后刷新页面查看：

![1526017362418](assets/1526017362418.png)

发现报错了！

浏览器直接访问没事，但是这里却报错，什么原因？



## 13、跨域问题：什么是跨域及解决方案

### 1）跨域简介

跨域是指跨域名的访问，以下情况都属于跨域：

| 跨域原因说明       | 示例                                   |
| ------------------ | -------------------------------------- |
| 域名不同           | `www.jd.com` 与 `www.taobao.com`       |
| 域名相同，端口不同 | `www.jd.com:8080` 与 `www.jd.com:8081` |
| 二级域名不同       | `item.jd.com` 与 `miaosha.jd.com`      |

如果**域名和端口都相同，但是请求路径不同**，不属于跨域，如：

`www.jd.com/item` 

`www.jd.com/goods`



而我们刚才是从`manage.leyou.com`去访问`api.leyou.com`，这属于二级域名不同，跨域了。





### 2）为什么有跨域

跨域不一定会有跨域问题。

因为跨域问题是浏览器对于ajax请求的一种安全限制：**一个页面发起的ajax请求，只能是于当前页同域名的路径**，这能有效的阻止跨站攻击。

因此：**跨域问题 是针对ajax的一种限制**。

但是这却给我们的开发带来了不便，而且在实际生产环境中，肯定会有很多台服务器之间交互，地址和端口都可能不同，怎么办？





### 3）解决跨域问题方案

目前比较常用的跨域解决方案有3种：

- Jsonp

  最早的解决方案，利用script标签可以跨域的原理实现。

  限制：

  - 需要服务的支持
  - 只能发起GET请求

- nginx反向代理

  思路是：利用nginx反向代理把跨域为不跨域，支持各种请求方式

  缺点：需要在nginx进行额外配置，语义不清晰

- CORS

  规范化的跨域请求解决方案，安全可靠。

  优势：

  - 在服务端进行控制是否允许跨域，可自定义规则
  - 支持各种请求方式

  缺点：

  - 会产生额外的请求

我们这里会采用cors的跨域方案。





## 14、跨域问题：CORS解决跨域原理

### 1）什么是CORS

CORS是一个W3C标准，全称是"跨域资源共享"（Cross-origin resource sharing）。

它允许浏览器向跨源服务器，发出[`XMLHttpRequest`](http://www.ruanyifeng.com/blog/2012/09/xmlhttprequest_level_2.html)请求，从而克服了AJAX只能[同源](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)使用的限制。

CORS需要浏览器和服务器同时支持。目前，所有浏览器都支持该功能，IE浏览器不能低于IE10。

- 浏览器端：

  目前，所有浏览器都支持该功能（IE10以下不行）。整个CORS通信过程，都是浏览器自动完成，不需要用户参与。

- 服务端：

  CORS通信与AJAX没有任何差别，因此你不需要改变以前的业务逻辑。只不过，浏览器会在请求中携带一些头信息，我们需要以此判断是否允许其跨域，然后在响应头中加入一些信息即可。这一般通过过滤器完成即可。



### 2）实现原理

浏览器会将ajax请求分为两类，其处理方案略有差异：简单请求、特殊请求。

#### 简单请求

只要同时满足以下两大条件，就属于简单请求。：

（1) 请求方法是以下三种方法之一：

- HEAD
- GET
- POST

（2）HTTP的头信息不超出以下几种字段：

- Accept
- Accept-Language
- Content-Language
- Last-Event-ID
- Content-Type：只限于三个值`application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`



当浏览器发现发现的ajax请求是简单请求时，会在请求头中携带一个字段：`Origin`.

 ![1526019242125](assets/1526019242125.png)

Origin中会指出当前请求属于哪个域（协议+域名+端口）。服务会根据这个值决定是否允许其跨域。

如果服务器允许跨域，需要在返回的响应头中携带下面信息：

```http
Access-Control-Allow-Origin: http://manage.leyou.com
Access-Control-Allow-Credentials: true
```

- Access-Control-Allow-Origin：可接受的域，是一个具体域名或者*，代表任意
- Access-Control-Allow-Credentials：是否允许携带cookie，默认情况下，cors不会携带cookie，除非这个值是true

注意：

如果跨域请求要想操作cookie，需要满足3个条件：

- 服务的响应头中需要携带Access-Control-Allow-Credentials并且为true。
- 浏览器发起ajax需要指定withCredentials 为true
- 响应头中的Access-Control-Allow-Origin一定不能为*，必须是指定的域名

#### 特殊请求

不符合简单请求的条件，会被浏览器判定为特殊请求,，例如请求方式为PUT。

如果是特殊请求产生的跨域会先发起一次OPTIONS预检请求，如果预检请求通过了，才会发送真正的请求。

特殊请求会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求（preflight）。

浏览器先询问服务器，当前网页所在的域名是否在服务器的许可名单之中，以及可以使用哪些HTTP动词和头信息字段。只有得到肯定答复，浏览器才会发出正式的`XMLHttpRequest`请求，否则就报错。

一个“预检”请求的样板：

```http
OPTIONS /cors HTTP/1.1
Origin: http://manage.leyou.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: X-Custom-Header
Host: api.leyou.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```

与简单请求相比，除了Origin以外，多了两个头：

- Access-Control-Request-Method：接下来会用到的请求方式，比如PUT
- Access-Control-Request-Headers：会额外用到的头信息

> 预检请求的响应

服务的收到预检请求，如果许可跨域，会发出响应：

```http
HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 01:15:39 GMT
Server: Apache/2.0.61 (Unix)
Access-Control-Allow-Origin: http://manage.leyou.com
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: X-Custom-Header
Access-Control-Max-Age: 1728000
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Content-Length: 0
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Content-Type: text/plain
```

除了`Access-Control-Allow-Origin`和`Access-Control-Allow-Credentials`以外，这里又额外多出3个头：

- Access-Control-Allow-Methods：允许访问的方式
- Access-Control-Allow-Headers：允许携带的头
- Access-Control-Max-Age：本次许可的有效时长，单位是秒，**过期之前的ajax请求就无需再次进行预检了**



如果浏览器得到上述响应，则认定为可以跨域，后续就跟简单请求的处理是一样的了。







## 15、跨域问题：解决方式一

第一种解决方案最简单，只需要在controller类上添加注解@CrossOrigin 即可！这个注解其实是CORS的实现。

![1575254948453](assets/1575254948453.png) 

源码解释：

![1575255168139](assets/1575255168139.png) 



Controller代码：

```java
package com.leyou.item.controller;

import com.leyou.item.pojo.Category;
import com.leyou.item.service.CategoryService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

/**
 * 分类Controller
 */
@RestController
@CrossOrigin // 解决跨域
public class CategoryController {
    @Autowired
    private CategoryService categoryService;

    /**
     * 根据父id查询子分类
     * 注意： 为了方便后续该方法被Feign接口调用，加上@RequestParam注解
     *  @RequestParam: /url?id=1
     *  @PathVariable: /url/1
     */
    @GetMapping("/category/of/parent")
    public ResponseEntity<List<Category>> findCategoryByPid(@RequestParam("pid") Long pid){
        List<Category> categories = categoryService.findCategoryByPid(pid);
        return ResponseEntity.ok(categories);
    }
}

```



这种方式需要在每个控制器上加上注解，不推荐这种方式。





## 16、跨域问题：解决方式二

SpringCloudGateway是Spring官方出品，他已经把各方面想到了，关于这个跨域，官方文档有详细解释：

https://cloud.spring.io/spring-cloud-gateway/reference/html/#cors-configuration

截图如下：

![1575361390402](assets/1575361390402.png)

我们只需要在application.yml文件中配置一下就可以解决跨域问题：

```yml
server:
  port: 10010
spring:
  application:
    name: api-gateway
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "http://manage.leyou.com"
            allowedHeaders:
              - "*"
            allowCredentials: true
            maxAge: 360000
            allowedMethods:
              - GET
              - POST
              - DELETE
              - PUT
              - OPTIONS
              - HEAD
      default-filters:
        - name: Hystrix
          args:
            name: fallbackcmd
            fallbackUri: forward:/fallback
      routes:
        - id: item-service   # 路由id,可以随意写
          # 代理的服务地址；lb表示负载均衡(从eureka中获取具体服务)
          uri: lb://item-service
          # 路由断言，可以配置映射路径
          predicates:
            - Path=/api/item/**
          filters:
            # 表示过滤1个路径，2表示两个路径，以此类推
            - StripPrefix=2
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            #设置API网关中路由转发请求的HystrixCommand执行超时时间
            timeoutInMilliseconds: 5000
```



## 17、课程总结



1）解决乐优商城后台返回数据给前端问题： 数据+状态码

​    1.1 正常情况下： ResponseEntity

​    1.2 异常情况下：自定义异常类 + 自定义异常拦截+ 自定义异常结果 + 自定义异常枚举



2）模拟线上环境

​    2.1 域名解析

   2.2 反向代理



3）分类业务

​     3.1 分析分类，品牌表结构

​     3.2 编写后台代码： 熟悉MyBatis-Plus用法

​     3.3 解决跨域问题






