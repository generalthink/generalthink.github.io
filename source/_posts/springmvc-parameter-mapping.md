---
title: springmvc的执行流程以及参数映射
date: 2024-12-26 17:44:45
tags: [SpringMVC, Handler, HandlerMapping, Jackson]
---

在之前的文章[SpringMVC源码深度分析](https://juejin.cn/post/6844903911426359303)分析了SpringMVC的启动流程，现在在来回顾下总体流程。

！[总体流程](/images/)


### 核心组件

在简要讲解下几个关键的组件

1. Handler

其实我们写的那些 Controller 中的方法，一个标注了 `@RequestMapping` 注解的方法，就是一个 **Handler** 。Handler 中要做的事情，就是处理客户端发来的请求，并声明响应视图 / 响应 json 数据

2. HandlerMapping

HandlerMapping 作为处理器映射器，它的作用就是根据 uri ，去匹配查找能处理的 Handler, HandlerMapping 查找到可以处理请求的 Handler 之后，返回的是`HandlerExecutionChain`, 其中**封装了Handler以及拦截器**


3. HandlerAdapter

HandlerAdapter 作为处理器适配器，它的作用就是执行上面封装好的 HandlerExecutionChain， 执行 Handler 之后，虽然我们写的返回值可能是返回视图名称，也可能是 借助@ResponseBody 响应 json 数据，但在框架内部，最终都是封装了一个 ModelAndView 对象，返回给 HandlerAdapter 。DispatcherServlet在来解析 ModelAndView 对象判断到底是返回视图还是直接返回JSON数据。

4. ViewResolver

负责解析ModelAndView,响应请求数据,比如我们常见的返回json数据的解析器`MappingJackson2JsonView`

### 组件初始化

DispatchServlet在启动支持会初始化它需要使用到的组件,这种思想需要记住，谁使用谁初始化

```java
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}

```
这里默认会去加载`DispatcherServlet.properties`文件的内容，它在spring-webmvc这个包下面，打开这个文件我们可以看到springmvc所需要的所有核心组件

```
# Default implementation classes for DispatcherServlet's strategy interfaces.
# Used as fallback when no matching beans are found in the DispatcherServlet context.
# Not meant to be customized by application developers.

org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
	org.springframework.web.servlet.function.support.HandlerFunctionAdapter


org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager

```

所以即便我们使用springmvc啥也不配置，也有默认策略兜底， 当然我在实际使用中也很少见到有谁去配置。

由 properties 文件可知，默认的 HandlerMapping 会初始化三个实现,但是我们只需要关注`RequestMappingHandlerMapping`, 就是它

就可以实现 `@Controller + @RequestMapping` 的注解式 WebMvc 开发。 
> @RestController实际上也是@Controller+@ResponseBody


我们接下来要进入的主题就是分析参数是如何解析的,现在我们抛出下面的问题

1. @PathVariable的参数是如何解析的
2. @RequestParam如何解析的
3. Java对象作为参数如何解析的
4. @RequestBody如何解析的

### Adapter

从properties文件知道,每一个handlerMapping都会有对应的Adaptor, RequestMappingHandlerMapping对应的Adapter是`RequestMappingHandlerAdapter`, 
在初始化这个Adapter的时候会去初始化各种参数解析器以及返回值处理器

```java
	@Override
	public void afterPropertiesSet() {
		// Do this first, it may add ResponseBody advice beans
		initControllerAdviceCache();

		if (this.argumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.initBinderArgumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
			this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.returnValueHandlers == null) {
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}

```

这里我们聚焦在参数处理器上，这里的顺序其实很重要,我们看几个比较重要的resolver

其中

+ PathVariableMethodArgumentResolver 用于解析 @PathVariable参数

+ PathVariableMapMethodArgumentResolver 解析Map类型@PathVariable参数

+ RequestResponseBodyMethodProcessor 用于解析 @RequestBody参数

+ RequestHeaderMethodArgumentResolver 用于解析 @RequestHeader参数

+ RequestParamMethodArgumentResolver 用于解析@RequestParam以及简单类型参数

+ ServletModelAttributeMethodProcessor 用于解析复杂对象参数



如果我们有有以下方法

```java
@GetMapping("/users/{id}")
public String getUser(@PathVariable Long id, User user) {
    // 方法实现
}

```
+ id 参数会由 PathVariableMethodArgumentResolver 处理，因为它有 @PathVariable 注解。
+ user 参数会由 ServletModelAttributeMethodProcessor 处理，因为它是一个复杂对象类型。


虽然这两个解析器不会直接冲突，但是会出现一些混淆的情况

```java
@GetMapping("/users/{user}")
public String getUser(@PathVariable User user) {
    // 方法实现
}

```
`PathVariableMethodArgumentResolver` 会首先尝试解析，因为有 @PathVariable 注解。但是，它可能无法将路径变量直接转换为 User 对象。这时，Spring MVC 会抛出`MethodArgumentConversionNotSupportedException`异常，而不是回退到使用 `ServletModelAttributeMethodProcessor`。


如果确实需要在 @PathVariable 中使用复杂对象，可以实现自定义的 `HandlerMethodArgumentResolver`。


#### RequestParamMethodArgumentResolver

很显然如果我们的参数标记了@RequestParam注解,那么明显就该这个处理器解析，但是如果没有标明这个注解呢？

```java
	public static boolean isSimpleProperty(Class<?> type) {
		Assert.notNull(type, "'type' must not be null");
		return isSimpleValueType(type) || (type.isArray() && isSimpleValueType(type.getComponentType()));
	}

	
	public static boolean isSimpleValueType(Class<?> type) {
		return (Void.class != type && void.class != type &&
				// 是简单类型或者其包装类型
				(ClassUtils.isPrimitiveOrWrapper(type) ||
				Enum.class.isAssignableFrom(type) ||
				CharSequence.class.isAssignableFrom(type) ||
				Number.class.isAssignableFrom(type) ||
				Date.class.isAssignableFrom(type) ||
				Temporal.class.isAssignableFrom(type) ||
				URI.class == type ||
				URL.class == type ||
				Locale.class == type ||
				Class.class == type));
```
如果是上面提到的类型那么也属于这个处理器解析解析的。然后参数值就是通过`request.getParameterValues(name);`获取的。

解析的代码在父类AbstractNamedValueMethodArgumentResolver中
```java
	@Override
	@Nullable
	public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
		MethodParameter nestedParameter = parameter.nestedIfOptional();

		// 解析到参数名
		Object resolvedName = resolveEmbeddedValuesAndExpressions(namedValueInfo.name);
		
		// 获取到参数值实际上通过request.getParameterValues(name)获取
		Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);
		
		if (binderFactory != null) {
			WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
			// 使用WebDataBinder进行数据转换
			arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
		}
		return arg;
	}
```

这里的convertIfNecessarty方法有两种实现

```java
public <T> T convertIfNecessary(@Nullable Object value, @Nullable Class<T> requiredType,
			@Nullable MethodParameter methodParam) throws TypeMismatchException {

	return getTypeConverter().convertIfNecessary(value, requiredType, methodParam);
}

@Nullable
public <T> T convertIfNecessary(@Nullable String propertyName, @Nullable Object oldValue, @Nullable Object newValue, @Nullable Class<T> requiredType, @Nullable TypeDescriptor typeDescriptor) throws IllegalArgumentException {
    PropertyEditor editor = this.propertyEditorRegistry.findCustomEditor(requiredType, propertyName);
    ConversionFailedException conversionAttemptEx = null;
    ConversionService conversionService = this.propertyEditorRegistry.getConversionService();

// PropertyEditor为空，则使用ConversionService进行转换
if (editor == null && conversionService != null && newValue != null && typeDescriptor != null) {
            TypeDescriptor sourceTypeDesc = TypeDescriptor.forObject(newValue);
            if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
             
                return conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
               
            }
        }

        // 使用editor转换
        Object convertedValue = newValue;
        if (editor != null || requiredType != null && !ClassUtils.isAssignableValue(requiredType, newValue)) {
            convertedValue = this.doConvertValue(oldValue, convertedValue, requiredType, editor);
        }


		// 省略其他代码
        return convertedValue;

}
```

上面的代码我保留了核心部分,可以知道参数转换的时候先看有没有自定义的editor,如果没有则使用ConversionService。 

比如对于Date的转换我们可以注册一个Editor来实现

```java
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        binder.registerCustomEditor(Date.class, new PropertyEditorSupport() {
            public void setAsText(String text) {
                this.setValue(DateUtils.parseDate(text, "yyyy-MM-dd"));
            }
        });
    }

```

这样子当接收Date作为普通参数的时候的时候，前端只要传递的格式是yyyy-MM-dd就也可以被转换成Date类型了，不然你就会收到一个异常： ConversionFailedException：Failed to convert from type [java.lang.String] to type [java.util.Date]

当然SpringMVC还有一个默认转换的实现, 在接收参数的地方加上@DateTimeFormat注解也可以正常接收Date参数
> 具体代码可以参考 FormattingConversionService#addFormatterForFieldAnnotation

```java
  @GetMapping("/user")
    public User test(String name, Long id, @DateTimeFormat(pattern="yyyy-MM-dd") Date birth) {

        return new User(id, name, birth);
    }

```

同样的我们可以使用一个自定义的Convert,并将其注入到Spring中，这样子ConversionService就可以通过我们自定义的converter进行转换

```java
@Component
public class DateConverter implements Converter<String , Date> {
    @Override
    public Date convert(String source) {
        try {
            return DateUtils.parseDate(source, "yyyy-MM-dd");
        } catch (ParseException e) {
            throw new RuntimeException(e);
        }
    }
}

```

### ServletModelAttributeMethodProcessor

```java
@Override
public boolean supportsParameter(MethodParameter parameter) {
	return (parameter.hasParameterAnnotation(ModelAttribute.class) ||
			(this.annotationNotRequired && !BeanUtils.isSimpleProperty(parameter.getParameterType())));
}
```

可以看到这个处理器除了解析`@ModelAttribute`注解外，非简单类型的都归这个处理器处理。 annotationNotRequired 在兜底的处理中为true, 如果为false,那就只处理`@ModelAttribute`。


关于参数解析,代码比较长了我就不贴出来了，其主要逻辑如下

1. 通过反射构造出对象
2. 通过WebDataBinder进行数据转换
3. 通过反射设置property属性值

从中我们可以知道上面提到的**三种数据转换方式**在ServletModelAttributeMethodProcessor同样是试用的。

### RequestResponseBodyMethodProcessor

```java
@Override
public boolean supportsParameter(MethodParameter parameter) {
	return parameter.hasParameterAnnotation(RequestBody.class);
}

```

可以看到这里处理器只支持@RequestBoday的数据,然后参数解析是通过`HttpMessageConverter`接口来解析的，spring默认提供了很多转换接口比如常用的`MappingJackson2HttpMessageConverter`这个就是jackson转换器。

而这其中WebDataBinder只用于参数校验，比如你声明了@Validated注解,那么会进行参数验证。

如果我们想要自定义解析规则只需要实现一个自定义的JsonDeserializer就行了。

比如我们现在有这么一个需求，如果前端传递的user参数是{}我需要解析到后端是null，而不是一个User(id=null,name=null)这样的数据。

```json
{
	"emailName": "test email", 
	"user": {},
}

```
通过`@JsonComponent`方式添加JsonDeserializer，这样就不用再每个需要这样转换的地方添加注解了。

```java
@JsonComponent
public class UserConverter extends JsonDeserializer<User> implements Converter<String, User> {
    @Override
    public User deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        TreeNode treeNode = p.getCodec().readTree(p);
        return convert(treeNode.toString());
    }

    @Override
    public User convert(String source) {
        if(StringUtils.isBlank(source) || source.equals("{}")) {
            return null;
        }
        // new一个新的，避免死循环
        ObjectMapper objectMapper = new ObjectMapper();
        try {
            return objectMapper.readValue(source, User.class);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }
}

```

### 总结

现在我们知道了不同参数是如何被解析的了，如果后面再遇到有参数解析的问题就可以找到对应的类去看源码了。 springmvc的参数解析对我们再也不是迷了。