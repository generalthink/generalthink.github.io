---
title: 老大喊我用AOP记录下日志
date: 2019-08-22 10:42:32
tags: [java,AOP,Spring]
---

老大喊我记录下API的操作日志,免得前端甩锅,主要记录新增,修改,删除等操作。我想了下就决定用AOP来实现这个功能。

由于使用的是SpringBoot，所以首先应该在依赖中引入AOP包。
```xml
 <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```
一般引入了AOP之后，一般不用做其他特殊配置,也不用加上@EnableAspectJAutoProxy注解。但是它仍有两个属性需要我们注意
<!--more-->

```
# 相当于@EnableAspectJAutoProxy
spring.aop.auto=true

# 默认使用jdk来实现AOP,如果想要使用CGLIB的话，这里要改成true
spring.aop.proxy-target-class=false
```

### 实现日志记录功能

我想要记录某个API对模块进行了什么操作，操作的key,对于修改，删除来说我们记录id,对于新增来说，最开始没有id,我们记录name即可(也可以是其他属性),所以OpLog注解是这样的

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
@Documented
public @interface OpLog {

  // 模块
  String module();
  // 类型
  String opType();
  // 级别，默认一般
  String level() default OpLevel.COMMON;
  // 需要记录的key
  String key();
}
```

为了让这个功能更加易用,所以我要求其他开发者必须在Controller上加上注解@OpLog,为什么是Controller呢?因为API是在Controller上,我并不关心具体业务，我只要记录对应的操作即可。在Controller层你只需要这要做就行了

```java
@RestController
@RequestMapping("/user")
public class CpGroupController {

    // 注意这里的userParam和@RequestBody的userParam名字必需要要一致
    @OpLog(module = "用户管理", opType = "新增", key = "userParam.name")
    @PostMapping("/add")
    public BaseResponse<Boolean> add(@RequestBody UserParam userParam) {
        
        return ResponseUtil.success(userService.add(userParam));
    }
}
```

接下来就是切面代码的编写了，其中我们要记录访问的url以及必要的操作信息。


```java
@Component
@Aspect
@Slf4j
public class OpLogAspect {

    @Autowired
    private OperateLogService operateLogService;

    int paramNameIndex = 0;
    int propertyIndexFrom = 1;

    @Pointcut("execution(public * com.generalthink.springboot.web.controller..*.*(..))")
    public void webLogPointcut() {
    }

    @Around(value = "webLogPointcut() && @annotation(opLog)", argNames = "joinPoint,opLog")
    public Object around(ProceedingJoinPoint joinPoint, OpLog opLog) throws Throwable {

        Object objReturn = joinPoint.proceed();

        try {
            OperateLog operationLog = generateOperateLog(joinPoint, opLog);

            operateLogService.save(operationLog);
        } catch (Exception e) {
            log.error("operateLog record error", e);
        }

        return objReturn;
    }

    private OperateLog generateOperateLog(ProceedingJoinPoint joinPoint, OpLog opLog) {

        String requestUrl = extractRequestUrl();

        Object recordKey = getOpLogRecordKey(joinPoint, opLog);

        return OperateLog.builder()
                .module(opLog.module())
                .opType(opLog.opType())
                .level(opLog.level())
                .operateTimeUnix(CommonUtils.getNowTimeUnix())
                .recordKey(recordKey != null ? recordKey.toString() : null)
                .url(requestUrl)
                .operator(getCurrentUser())
                .build();
    }

    // 获取当前用户
    private String getCurrentUser() {
        Subject subject = SecurityUtils.getSubject();
        return (String) subject.getPrincipal();
    }

    // 获取想要记录的key
    private Object getOpLogRecordKey(ProceedingJoinPoint joinPoint, OpLog opLog) {
        String key = opLog.key();

        if(StringUtils.isEmpty(key)) {
            return null;
        }

        String[] keys = key.split("\\.");

        //入参  value
        Object[] args = joinPoint.getArgs();

        CodeSignature codeSignature = (CodeSignature) joinPoint.getSignature();

        // 获取Controller方法上的参数名称
        String[] paramNames = codeSignature.getParameterNames();

        Object paramArg = null;

        for (int i = 0; i < paramNames.length; i++) {
            String paramName = paramNames[i];
            if (paramName.equals(keys[paramNameIndex])) {
                paramArg = args[i];
                break;
            }
        }

        if(keys.length == 1) {
            return paramArg;
        }

        // 获取参数的值
        Object paramValue = null;
        if(null != paramArg) {
            try {
                paramValue = getParamValue(paramArg, keys, propertyIndexFrom);
            } catch (IllegalAccessException e) {
                log.error("parse field error", e);
            }
        }

        return paramValue;
    }

    public Object getParamValue(Object param, String[] keys, int idx)
            throws IllegalAccessException {
        Optional<Field> fieldOptional = getAllFields(new ArrayList<>(), param.getClass())
                .stream()
                .filter(field -> field.getName().equalsIgnoreCase(keys[idx]))
                .findFirst();

        if(!fieldOptional.isPresent()) {
            return null;
        }
        
        // 设置属性可访问,因为bean当中属性一般都是private
        Field field = fieldOptional.get();
        field.setAccessible(true);


        if(idx + 1 == keys.length) {
            return field.get(param);
        }

        return getParamValue(field.get(param), keys, idx + 1);
    }

    // 递归获取所有Field
    public List<Field> getAllFields(List<Field> fieldList, Class<?> type) {

        // 获取当前类的所有Field
        fieldList.addAll(Arrays.asList(type.getDeclaredFields()));
        
        // 获取父类的所有Field
        Class<?> superClass = type.getSuperclass();
        if(superClass != null && superClass != Object.class) {
            getAllFields(fieldList,superClass);
        }

        return fieldList;
    }
    
    // 获取访问URL
    private String extractRequestUrl() {
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder
                .getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();

        return request.getRequestURL().toString();
    }
}

```
主要就是记录了操作了什么模块,操作内容等等,对于修改和删除操作我们可以记录id,但是对于save操作,我们没有办法记录id,只能记录其他属性,比如说name,就可以记录保存的数据了

```
{
  name: "zhangsan",
  age: 20
}

```
对于上面的数据就会把"zhangsan"这个值保存下来,后面就可以通过查询日志表知道是谁操作的,修改和删除也是同样的.
