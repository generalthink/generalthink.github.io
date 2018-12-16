title: fastjson使用
date: 2015-10-02 12:49:50
tags: [json,java]
categories: java
description: fastjson快速入门以及基本使用
keywords: java json fastjson
---

#### 1. fastjson的主要API

  **fastjson入口类是com.alibaba.fastjson.JSON,主要的API是JSON.toJSONString,和parseObject**

  ```java	
	package com.alibaba.fastjson;

	public abstract class JSON {

	    public static final String toJSONString(Object object);

	    public static final <T> T parseObject(String text, Class<T> clazz, Feature... features);

	}
	```
  

  **序列化:**
  ```java
      String jsonString = JSON.toJSONString(obj);
  ```

  **反序列化:**

  ```java
        VO vo = JSON.parseObject("...", VO.class);
  ```

  **泛型反序列化:**

  ```java  
	import com.alibaba.fastjson.TypeReference;

	List<VO> list = JSON.parseObject("...", new TypeReference<List<VO>>() {});
  ```

####  2. JSON、JSONObject、JSONArray三者之间的关系

  ```java
    public class JSONObject extends JSON implements Map<String, Object>;

    public class JSONArray extends JSON implements List<Object>
  ```
  也就是说,JSONObject实际上就是一个Map,其处理方式和Map类似,而JSONArray实质上是一个List,其处理方式和List类似。


####  3. 实例讲解

  **涉及到的实体Bean:**

  ```java 
    public class User {
      int id,
      int name;
      Date birthday;
      List<Phone> phones;
      public User() {

      }
    	//省略get,set方法
    }

    public class Phone {
	    int phoneNum;
	    public Phone() {
			
	  }
		//省略get,set方法
	}

  ```

  **测试:**

  ```java   
	public void testObject() {
	  User user = new User();
	  user.setId(1);
	  user.setName("abc");
	  user.setBirthday(new Date());
	  List<Phone> phones = new ArrayList<Phone>();
	  Phone p1 = new Phone();
	  p1.setPhoneNum(12123123);
			
	  Phone p2 = new Phone();
	  p2.setPhoneNum(23423423);
	  phones.add(p1);
	  phones.add(p2);
			
	  user.setPhones(phones);
			
	  JSON.DEFFAULT_DATE_FORMAT = "yyyy-MM-dd hh:mm:ss";
	  System.out.println("JSON:"+JSON.toJSONString(user,SerializerFeature.WriteDateUseDateFormat));
	  //该注释的代码和上面两行的代码效果一致
	  //System.out.println(JSON.toJSONStringWithDateFormat(user, "yyyy-MM-dd hh:mm:ss"));
			

	  JSONObject jobj = new JSONObject();
	  jobj.put("user", user);
	  System.out.println("JSONObject:"+jobj.toJSONString());
			

	  JSONArray jArray = new JSONArray();
	  jArray.add(user);
	  System.out.println("JSONArray:"+jArray.toJSONString());		
			
	}
  ```
  **输出:**

  ```javascript
	JSON:
	{
		"birthday":"2015-09-30 03:06:36",
		"id":1,
		"name":"abc",
		"phones":[
			{
				"phoneNum":12123123
			},
			{
				"phoneNum":23423423
			}
		]
	}

	JSONObject:
	{
	  "user": {
	    "birthday": 1443597247350,
	    "id": 1,
	    "name": "abc",
	    "phones": [
	      {
	        "phoneNum": 12123123
	      },
	      {
	        "phoneNum": 23423423
	      }
	    ]
	  }
	}

	JSONArray:
	[
	  {
	    "birthday": 1443597247350,
	    "id": 1,
	    "name": "abc",
	    "phones": [
	      {
	        "phoneNum": 12123123
	      },
	      {
	        "phoneNum": 23423423
	      }
	    ]
	  }
	]
  ```

  **测试:**

  ```java
	public void testList() {
		  List<User> list = new ArrayList<User>();
		  User u1 = new User();
		  u1.setId(2);
		  u1.setName("abc");
		  u1.setBirthday(new Date());
		  User u2 = new User();
		  u2.setId(3);
		  u2.setName("cde");
		  list.add(u1);list.add(u2);
			
		  String result = JSON.toJSONStringWithDateFormat(list, "yyyy-MM-dd hh:mm:ss");
		  System.out.println(result);
		  //[{"birthday":"2015-09-30 03:34:38","id":2,"name":"abc"},{"id":3,"name":"cde"}]
			
		  //List<User> refList = JSON.parseObject(result, new ArrayList<User>().getClass());和下面的代码效果一致
		  List<User> refList2 = JSON.parseObject(result, new TypeReference<List<User>>(){});
		  System.out.println(refList2.size());
		  //输出:2
		  List<User> users = JSON.parseArray(result, User.class);
		  System.out.println(users.size());
		  //输出:2
			
		  JSONObject obj = new JSONObject();
		  obj.put("abc", 123);
		  Boolean b = obj.getBoolean("ccc");
		  System.out.println(b);
		}
  ```

#### 4. 控制哪些值序列化

  **可以通过扩展实现根据object或者属性名称或者属性值进行判断是否需要序列化。例如:**

  ```java
	PropertyFilter filter = new PropertyFilter() {
		public boolean apply(Object source, String name, Object value) {
		    if ("id".equals(name)) {
		        int id = ((Integer) value).intValue();
		        return id >= 100;
		    }
		    return false;
		}
	};
	JSON.toJSONString(obj, filter); // 序列化的时候传入filter
  ```

  **同样可以通过SimplePropertyPreFilter来决定哪些值可以序列化:**

  ```java
  User vo = new User();

  vo.setId(123);
  vo.setName("flym");

  SimplePropertyPreFilter filter = new SimplePropertyPreFilter(VO.class, "name");
  Assert.assertEquals("{\"name\":\"flym\"}", JSON.toJSONString(vo, filter));

  ```

  **同样可以通过注解的方式来判断哪些只需要序列化,详细请查看fastjson的官方文档**

####  5. 参考文档
  [fastjson下载以及文档参考](https://github.com/alibaba/fastjson)