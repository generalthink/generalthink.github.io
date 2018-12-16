title: fastjson定制序列化以及反序列化
date: 2015-10-26 20:03:19
tags: [json,java]
categories: java
description: fastjson定制序列化以及反序列化
keywords: java fastjson 序列化
---

**先来看一段序列化的代码**
```java
	public class FastJsonTest {

	  public FastJsonTest() {
	  
	  }
	  public static void main(String[] args) {
	    Map<String, String> map = new HashMap<String, String>();
	    map.put("12", "zhangsan1");
	    map.put("121", "zhangsan2");
	    map.put("122", "zhangsan3");

	    String gString = JSON.toJSONString(map, new PropertyFilter() {
	      
	      @Override
	      public boolean apply(Object object, String name, Object value) {
	       System.out.println(object.toString()+"..."+name+"..."+value);
	       if(name.equalsIgnoreCase("121") || value.equals("zhangsan2"))
	         return false;
	       
	        return true;
	      }
	    }, SerializerFeature.BrowserCompatible);//SerializerFeature.WriteNonStringKeyAsString

	    System.out.println(gString);//{"122":"zhangsan3","12":"zhangsan1"}
	  }

	}
```

toJSONString 有很多重构的方法:
![toJSONString](/images/hexo-toJSONString.png)

SerializerFeature: 在其中可以定义序列化成json串的时候的各种特性
  1. SerializerFeature.BrowserCompatible 标示兼容游览器
  2. SerializerFeature.WriteNonStringKeyAsString 将非字符串类型的key当成字符串来处理
  
使用上面两种中的任意一种都会为key加上双引号。

SerializeFilter:  定制序列化，它是一个接口,fastjson中预定义了部分常用的filter
  1. PropertyPreFilter 根据PropertyName判断是否序列化
  2. PropertyFilter 根据PropertyName和PropertyValue来判断是否序列化
  3. NameFilter 修改Key，如果需要修改Key,process返回值则可
  4. ValueFilter 修改Value
  5. BeforeFilter 序列化时在最前添加内容
  6. AfterFilter 序列化时在最后添加内容

更多序列化信息请参考:<https://github.com/alibaba/fastjson/wiki/%E5%AE%9A%E5%88%B6%E5%BA%8F%E5%88%97%E5%8C%96>

反序列化请参考:<https://github.com/alibaba/fastjson/wiki/ParseProcess>



