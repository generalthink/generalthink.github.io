title: SpringMVC源码分析
date: 2018-08-13 21:43:56
tags: [java,Spring]
categories: java
description: SpringMVC是常用的MVC框架,Spring的代码中也用到了大量的设计模式,分析源码不仅可以让你更加熟练的使用，还能让你更有信心的使用，本文就是基于这一原则进行源码分析。
keywords: java,SpringMVC源码分析,Spring
---

### 强大的DispatcherServlet
还记得在web.xml中配置的DispatcherServlet吗?其实那个就是SpringMVC框架的入口，这也是struts2和springmvc不同点之一，struts2是通过filter的，而springmvc是通过servlet的。看下servlet的结构图
![结构图](/images/dispatcherServlet_in_Spring.png)
![类图](/images/dispatcherServetl_diagram_in_spring.png)

从上面这张图很明显可以看出DispatcherServlet和Servlet以及Spring的关系。而我们今天的重点就从DispatchServlet说起。


在分析之前我用SpringBoot搭建了一个很简单的后台项目，用于分析。代码如下

```java
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;

@Data
@Builder
@AllArgsConstructor
public class User {

private Integer id;

private String name;

private Integer age;

private String address;

public User() {

}
}

/**
* @author generalthink
*/
@RestController
@RequestMapping("/user")
public class UserController {

@RequestMapping(value = "/{id}",method = RequestMethod.GET)
public User getUser(HttpServletRequest request,@PathVariable Integer id) {

//创建一个user,不走数据库只是为了分析springmvc源码
User user = User.builder()
.id(id)
.age(ThreadLocalRandom.current().nextInt(30))
.name("zzz" + id)
.address("成都市").build();

return user;
}

@RequestMapping(value = "/condition",method = RequestMethod.GET)
public User getByNameOrAge(@RequestParam String name,@RequestParam Integer age) {
User user = User.builder().name(name).age(age).address("成都市").id(2).build();
return user;
}

@PostMapping
public Integer saveUser(@RequestBody User user) {

Integer id = user.getName().hashCode() - user.getAge().hashCode();

return id > 0 ? id : -id;
}

}

```

这里为了方便调试把关注点更多集中在SpringMVC源码中,所以这里的数据都是伪造的。而且这里的关注点也集中到使用注解的Controller(org.springframework.stereotype.Controller)，而不是Controller接口(org.springframework.web.servlet.mvc.Controller),这两者的区别主要在意一个只用标注注解，一个需要实现接口，但是它们都能完成处理请求的基本功能。我们都知道访问servlet的时候默认是访问service方法的，所以我们将断点打在HttpServlet的service方法中，此时查看整个调用栈如下
![调用栈](/images/dispatcherServlet_invoke_stack.png)

从这里我们也知道了请求时如何从servlet到了DispatcherServlet的，我们先来看一下DispatcherServlet的doDiapatch的方法逻辑，这里把核心逻辑列出来了，把其他的一些非核心逻辑移除了

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        HttpServletRequest processedRequest = request;
        
        //注意这里放回的是HandlerExecutionChain对象
        HandlerExecutionChain mappedHandler = null;
        boolean multipartRequestParsed = false;
        
            ModelAndView mv = null;
            Exception dispatchException = null;

            //检查是否存在文件上传
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // 根据当前request获取handler,handler中包含了请求url,以及最终定位到的controller以及controller中的方法
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response);
                return;
            }

            // 通过handler获取对应的适配器,主要完成参数解析
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // 调用Controller中的方法
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            applyDefaultViewName(processedRequest, mv);
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        
            processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
        
    }
```

可以看到核心逻辑其实非常简单，首先检查是不是multipart request，如果是则对当前的request进行一定的封装(提取文件等），然后获取对应的handler(保存了请求url对应的controller以及method以及一系列的Interceptor),然后在通过handler获取到对应的handlerAdapter(参数组装)，通过它来进行最终方法的调用

### 解析multipart

那么是如何解析当前请求是文件上传请求呢？这里直接进入到checkMultipart方法看看是如何解析的：

```java
//我精简了下代码，只提取了核心逻辑
protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
    if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {        
        return this.multipartResolver.resolveMultipart(request);    
    }
    return request;
}
```

从这里可以看出通过multipartResolver判断当前请求是否是文件上传请求,如果是则返回MultipartHttpServletRequest(继承自HttpServletRequest).不是则返回原本request对象。
那么问题来了multipartResolver是什么时候初始化的呢？


我们在idea中可以直接将断点定位到multipartResolver属性上，进行请求访问这个时候会发现断点直接进入到了initMultipartResolver方法中，接着跟踪整个调用栈，可以发现调用关系如下：
![初始化multipartResovler](/images/init_mulitpartResovler_in_SpringMVC.png)

图上表明了是在初始化servlet的时候对multipartResolver进行了初始化的。

```java
private void initMultipartResolver(ApplicationContext context) {

//从Spring中获取id为multipartResolver的类
    this.multipartResolver = context.getBean("multipartResolver", MultipartResolver.class);
}

```

MultipartResolver接口有CommonsMultipartResolver以及StandardServletMultipartResolver2种实现，CommonsMultipartResolver接口是依赖于commons-upload组件实现的，而 StandardServletMultipartResolver是依赖于Servlet的part(servlet3才存在)实现的.两者判断是否是文件上传请求的方法isMultipart均是通过判定请求方法是否为post以及content-type头是否包含multipart/来进行判定的。
 
### DispatchServlet初始化了哪些内容


```java
protected void initStrategies(ApplicationContext context) {
   initMultipartResolver(context);  //初始化multipartResolver
   initLocaleResolver(context);//初始化localeResolver
   initThemeResolver(context);//初始化themResolver
   initHandlerMappings(context);//初始化handerMappings
   initHandlerAdapters(context);//初始化handlerAdapters
   initHandlerExceptionResolvers(context);
   initRequestToViewNameTranslator(context);
   initViewResolvers(context);//初始化试图解析器
   initFlashMapManager(context);
}
```
这些初始化的内容都会在后面被逐一使用，这里先有一个印象。


### 根据请求获取mapperHandler


还是进入到getHander方法中看看到底做了什么？

```java
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
if (this.handlerMappings != null) {
    for (HandlerMapping hm : this.handlerMappings) {
        HandlerExecutionChain handler = hm.getHandler(request);
        if (handler != null) {
            return handler;
            }
        }
    }
    return null;
}

```

根据HandlerMapping来查看对应的handler,那么进入到initHandlerMappings方法中查看如何初始化handlerMappings

![初始化handlerMappings](/images/init_handlerMapping_in_springmvc.png)

其中获取默认的handlerMappings是去spring-webmvc的org.springframework.web.servlet中的DispatcherServlet.properties中查找，文件内容是这样的

![DispatcherServlet.properties](/images/DispatcherServlet.properties.png)
因为detechAllhanderMappings默认为true，所以会获取到所有HanderMapping的实现类，来看看它的类图结构是怎样的
![HandlerMapping类图](/images/handlerMapping_interface_structure_in_springmvc.png)

![this.handlerMappings的值](/images/this.handlerMappings_in_springmvc.png)

这几个HandlerMapping的作用如下：
SimpleUrlHandlerMapping : 允许明确指定URL模式和Handler的映射关系，内部维护了一个urlMap来确定url和handler的关系
BeanNameUrlHandlerMapping:  指定URL和bean名称的映射关系，不常用，我们的关注点也主要集中在RequestMappingHandlerMapping中

这里也基本明确了HandlerMapping的作用:帮助DispatcherServlet进行Web请求的URL到具体类的匹配,之所以称为HandlerMapping是因为在SpringMVC中并不局限于
必须使用注解的Controller我们也可以继承Controller接口，也同样可以使用第三方接口，比如Struts2中的Action
![RequestMappingHandlerMapping](/images/RequestMappingHandlerMapping.png)

接着看下getHandler的实现：

```java
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   if (this.handlerMappings != null) {
      for (HandlerMapping hm : this.handlerMappings) {
         HandlerExecutionChain handler = hm.getHandler(request);
         if (handler != null) {
            return handler;
         }
      }
   }
   return null;
}

```

返回的handler是HandlerExecutionChain,这其中包含了真实的handler以及拦击器，可以在执行前,执行后，执行完成这三个阶段处理业务逻辑。
RequestMappingHandlerMapping的getHandler的调用逻辑如下：
![调用逻辑](/images/handlerMapping_invoke_logic_in_springmvc.png)

会遍历所有Controller的url查看是否有符合条件的match(head,url,produce,consume，method都要满足要求),采用antMatcher的方式来进行url匹配，如果匹配上了则返回对应的handler，否则返回null,如果映射发现有重复的映射（url映射相同，请求方法相同，参数相同，请求头相同，consume相同，produce相同，自定义参数相同），则会抛出异常。

而SimpleUrlHandlerMapping的调用逻辑如下：
![SimpleUrlHandlerMapping调用逻辑](/images/simple_url_handlerMapping_invoke_logic.png)

其中维护了url到handler的映射，先通过url到urlMap中找对应的handler，如果没有找到则尝试pattenMatch，成功则返回对应的handler,未匹配则返回null。

会发现处理HandlerMapping这里运用了模板方法，在抽象类中定义好了业务逻辑，具体实现只需要实现自己的业务逻辑即可。同时也符合开闭原则，完全是面向接口编程，不得不让人叹服这里的涉及逻辑。

分析到这里的时候我们会发现我们之前定义的Controller明显是符合RequestMappingHandlerMapping的策略的，所以返回的HandlerExecutionChain已经包含了需要访问的方法的全路径了。


### 关于HandlerAdapter

HandlerMapping会通过HandlerExecutionChain返回一个Object类型的Handler对象，用于Web请求处理，这里的Handler并没有限制具体是什么类型，一般来说任何类型的Handler都可以在
SpringMVC中使用，只要它是用于处理Web请求的处理对象就行。

不过对于DispatcherServlet来说就存在问题了，它无法判断到底使用的是什么类型的Handler，也无法知道是调用Handler的哪个方法来处理请求，为了以同意的方式来调用各种类型的Handler,
DispatcherServlet将不同Handler的调用职责转交给了一个成为HandlerAdapter的角色。


先看一下HandlerAdpter接口的定义

```java
public interface HandlerAdapter {

boolean supports(Object handler);


@Nullable
ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;


long getLastModified(HttpServletRequest request, Object handler);
}

```


主要关注supports和handle方法。先看下DispatcherServlet中handlerAdapters的初始化过程，和handlerMappings的初始化过程是类似的
![初始化HandlerAdapters](/images/init_handlerAdapters_in_springmvc.png)

接着在看一下HandlerAdapter的类关系
![HandlerAdapter类图](/images/handlerAdpater_class_diagram.png)

同样的，仍然通过合适的策略寻找对应的Adapter，我们主要关注的是RequestMappingHandlerAdapter(其他的用得很少)，所以这里就主要讲解它。查看它support的实现代码：
![supports方法](/images/handler_adapter_supports_method.png)

在上面关于handler的说明中说了其实Object handler实际上是HandlerMethod，所以这里对应的HandlerAdapter就是RequestMappingHandlerAdapter。

找到对应的适配器之后，这个时候就可以调用真正的逻辑了。在这之前使用者可以通过拦截器做一些事儿，比如记录日志，打印执行时间等，所以如果想要在执行的方法之前添加一条语句，我们只需要配置自己的拦击器即可。
![执行拦截器方法](/images/handlerInterceptors_invoke.png)

接下来我们重点分析handle方法，看看它到底会做什么？，先看一下handle方法的执行流程，同样的adapter同样使用了模板方法，先在父类里面定义流程，子类只需要实现逻辑即可，所以这里首先会调用AbstracthandlerMethodAdapter的invokeHadlerMethod方法，其中对HandlerMethod进行了封装。
![invokeHandle](/images/invoke_handle_method.png)
![invokeAndHandle](/images/invoke_and_handle.png)

我们进入到第一步，看看invokeForRequest方法中主要做了什么
![invokeForRequest](/images/invoke_for_request.png)

发现这个方法的调用逻辑实际上很简单，就是解析参数，然后调用方法。我们来看一下如何进行参数解析的呢？
![参数解析](/images/arguementResolver_resolve.png)

可以看到几乎所有的核心逻辑都集中到了argumentResovlers中去，那么支持的arguementResolver有哪些？又是在哪里初始化的呢？

首先需要定位到这个属性是从哪里过来的，RequestMappingHandlerAdapter实现了InitializingBean，所以在初始化的时候会执行afterPropertiesSet方法，在这其中对arguementResolvers以及returnValueHandlers进行了初始化。
不同的resovler支持的参数解析不一样，比如说有支持HttpServletRequest注入的，有支持HttpServletREsponse注入的还有支持body体注入的等等。
![arguementResovler初始化](/images/getDefaultArgumentResolver.png)

![returnValueHandlers初始化](/images/init_return_value_handlers.png)

经过参数解析之后就得到了反射需要的数据了,class,method以及参数，最后通过java的反射api调用即可。
![反射调用](/images/invoke_true_method.png)

至此，springmvc的整个调用流程基本就清晰了。


但是到了这里问题仍然没有结束，因为我们还不知道参数具体是如何解析的。比如get方式提交的数据？post方式提交的数据？如何转换成对象的？这写问题都还存在，那我们继续研究。
这里我使用postman工具来发起请求，首先访问 Get http://localhost:8080/user/condition?name=zhangsan&age=25,定位到resolveArgument方法
![如何获取具体的arguementResolver](/images/arguementResovler_resolve.png)

接着又执行revolver.resolveArgument方法，同样的这里还是使用的模板方法，在抽象类AbstractNamedValueMethodArgumentResolver中定义流程，各个子类只需要实现自己的逻辑即可。RequestParamMethodArgumentResolver的参数就是通过request.getParameter来获取到的。获取到了参数之后就执行反射调用，这个时候就执行了我们写的UserController的对应方法，获取到了User对象，接下来就是处理返回值了，通过returnValueHandlers进行处理
![处理返回值](/images/handler_return_value.png)

handler会根据返回的类型对数据进行处理，比如说这里就通过response向请求方输出数据，输出数据也是通过messageConverter来实现的
![处理数据输出](/images/deal_result.png)

最后获取ModalAndView对象，但是这里由于没有modalAndView所以返回的null.最后在DispatcherServlet的processDispatchResult方法的调用逻辑如下
![最后的处理](/images/process_dispatcher_result.png)

那么对于这样的请求又时如何解析的呢？

```java
@PostMapping
public Integer saveUser(@RequestBody User user) {

Integer id = user.getName().hashCode() - user.getAge().hashCode();

return id > 0 ? id : -id;
}

```

同样我们聚焦在解析参数的时候，在上一个get请求的示例中我说了会先访问AbstractNamedValueMethodArgumentResolver，但是在处理@RequestBody的参数中它使用的是RequestResponseBodyMethodProcessor,它复写了resolveArgument方法。所以不会去执行父类的逻辑。
![参数解析](/images/resolve_arguement_with_post_method.png)

这里最后会定位到jakson的objectMapper中， 在spring boot中，默认使用Jackson来实现java对象到json格式的序列化与反序列化。当然是可以配置messageConvert的，只需要实现Spring的HttpMessageConverter即可。

源码分析到这里就结束了，当然其中还存在一些没有讲的地方，比如View的渲染呀，一般视图是多种多样的，又html,xml,jsp等等，所以springmvc也提供了接口供用户选择自己需要的模板，只需要实现ViewResolver接口即可。还有关于Theme,MessageResource，Exception的处理等等，如果铺开来讲篇幅实在是太长了，我更相信掌握了核心流程看其他的处理就会很简单了，所以这里也就不对其他枝节内容做分析了。


### 一图胜千言
![SpringMVC流程图](/images/springmvc_process.png)
