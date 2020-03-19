---
title: 道友,JAXB了解下
date: 2020-03-18 16:23:48
tags: [XML,JAXB,java]
---


### 什么是JAXB

JAXB(Java Architecture for XML Binding简称JAXB)允许Java开发人员将Java类映射为XML表示方式。JAXB提供两种主要特性：将一个Java对象序列化为XML，以及反向操作，将XML解析成Java对象。换句话说，JAXB允许以XML格式存储和读取数据，而不需要程序的类结构实现特定的读取XML和保存XML的代码。


### 注解

| 注解  | 作用域  | 描述  |
|---|---|---|
| @XmlRootElement  | Class,Enum  | 定义XML根元素  |
| @XmlAccessorType | Package,Class  | 定义JAXB引擎用于绑定的Java类的字段和属性。它具有四个值：PUBLIC_MEMBER，FIELD，PROPERTY和NONE。  |
| @XmlAccessorOrde  | Package,Class  | 定义子项顺序  |
| @XmlType  | Class,Enum  | 类型映射(java到xml)。它定义了其子类型的名称和顺序。  |
| @XmlElement  | Field  | 将字段或属性映射到XML元素  |
| @XmlAttribute  | Field  | 将字段或属性映射到XML属性  |
| @XmlTransient  | Field  | 防止将字段或属性映射到XML  |
| @XmlValue  | Field  | 将字段或属性映射到XML标签上的文本值  |
| @XmlList  | Field,Parameter  | 将集合映射到以空格分隔的值列表。  |
| @XmlElementWrapper  | Field  | 将Java集合映射到XML包装的集合  |
| @XmlJavaTypeAdapter  | PACKAGE,FIELD,METHOD,TYPE,PARAMETER | 复杂对象转换器 |

<!--more-->

#### @XmlRootElement

将类或枚举类型映射到XML根元素。当使用@XmlRootElement时，其值在XML文档中表示为XML根元素。

```java
@XmlRootElement(name = "employee")
public class Employee implements Serializable {

}
```

输出的xml如下:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<employee/>
```

#### @XmlAccessorType

使用Java类中的哪些字段或属性来生成xml。它有四个选项

1. FIELD : 除使用了@XmlTransient注解的字段外，类中的每个非静态，非transient字段都会被映射。
2. NONE : 除非使用某些JAXB注释专门对其进行注释，否则所有字段或属性均不会被映射。
3. PROPERTY : Java类中除了使用@XmlTransient的属性，其他每个getter/setter对都将被映射,即如果没有getter/setter方法则不会被映射
4. PUBLIC_MEMBER : 除使用了@XmlTransient注解的字段外,其他每个公共的getter/setter对以及每一个public字段都会被映射

默认值是PUBLIC_MEMBER, PROPERTY和PUBLIC_MEMBER一样，都需要具有public的getter/setter方法

```java
@XmlRootElement(name = "employee")
@XmlAccessorType(XmlAccessType.PROPERTY)
public class Employee implements Serializable {

  private Integer id;

  private String firstName;

  private String lastName;

  public Employee(){}

  public Employee(Integer id, String firstName, String lastName) {
      this.id = id;
      this.firstName = firstName;
      this.lastName = lastName;
  }

  public Integer getTId() {
      return id;
  }

  public void setTId(Integer id) {
      this.id = id;
  }

  public String getFirstName() {
      return firstName;
  }

  public void setFirstName(String firstName) {
      this.firstName = firstName;
  }
}

```
生成的xml如下:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<employee>
  <firstName>zhang</firstName>
  <TId>1</TId>
</employee>
```
注意这里的TId实际上是和getter/setter方法关联的,我们一般使用的时候使用最多的是FIELD

#### @XmlAccessorOrder

控制类中字段和属性的顺序。有ALPHABETICAL(字母顺)和UNDEFINED(类中字段顺序)两个选择

```java
@NoArgsConstructor
@AllArgsConstructor
@XmlRootElement(name = "employee")
@XmlAccessorType(XmlAccessType.FIELD)
@XmlAccessorOrder(XmlAccessOrder.ALPHABETICAL)
public class Employee implements Serializable {

  private Integer id;
  private String firstName;
  private String lastName;
  private Department department;
}

@AllArgsConstructor
@NoArgsConstructor
@XmlRootElement(name = "department")
@XmlAccessorType(XmlAccessType.FIELD)
public class Department implements Serializable {

  private Integer id;
  private String name;
}

```

生成的xml如下:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<employee>
  <department>
      <id>100</id>
      <name>软件开发</name>
  </department>
  <firstName>zhang</firstName>
  <id>1</id>
  <lastName>san</lastName>
</employee>
```

#### @XmlType

将java类或者枚举映射成schema类型,可以指明类型name, namespace 以及子元素顺序。不过一般只使用propOrder

```java

@NoArgsConstructor
@AllArgsConstructor
@XmlRootElement(name = "employee")
@XmlAccessorType(XmlAccessType.FIELD)
@XmlType(propOrder={"department", "firstName", "id" , "lastName" })
public class Employee implements Serializable {

  private Integer id;
  private String firstName;
  private String lastName;
  private Department department;
}

```

输出结果和上面@XmlAccessorOrder的例子一样。在使用@XmlType的propOrder 属性时，必须列出JavaBean对象中的所有属性，否则会报错

#### @XmlElement

将java bean属性映射到对应的xml元素

```java
@NoArgsConstructor
@AllArgsConstructor
@XmlRootElement(name = "employee")
public class Employee implements Serializable {
  @XmlElement(name="employeeId")
  private Integer id;
  @XmlElement
  private String firstName;
  private String lastName;
  private Department department;
}

```

输出xml如下:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<employee>
  <employeeId>1</employeeId>
  <firstName>zhang</firstName>
</employee>
```

还可以指定对应的java类型

#### @XmlAttribute

将JavaBean属性映射到XML属性。

```java
@NoArgsConstructor
@AllArgsConstructor
@XmlRootElement(name = "employee")
@XmlAccessorType(XmlAccessType.FIELD)
public class Employee implements Serializable {

  @XmlAttribute
  private Integer id;
  private String firstName;
  private String lastName;
  private Department department;
}

```

结果如下:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<employee id="1">
  <firstName>zhang</firstName>
  <lastName>san</lastName>
  <department>
      <id>100</id>
      <name>软件开发</name>
  </department>
</employee>
```

#### @XmlTransient

防止将JavaBean属性/类型映射到XML。当放置在一个类上时，它指示类不应映射到XML。

```java
@XmlRootElement(name = "employee")
@XmlAccessorType(XmlAccessType.FIELD)
public class Employee implements Serializable {

  @XmlTransient
  private Integer id;
  private String firstName;
  private String lastName;
  private Department department;
}

```
xml如下:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<employee>
  <firstName>zhang</firstName>
  <lastName>san</lastName>
  <department>
      <id>100</id>
      <name>软件开发</name>
  </department>
</employee>

```
#### @XmlValue

用于一个类到XML Schema的映射

#### @XmlList

用于将属性映射到列表类型，单个元素多个值以空格拼接

```
@XmlRootElement(name = "employee")
@XmlAccessorType(XmlAccessType.FIELD)
public class Employee implements Serializable {
  List<String> hobbies;
}

<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<employee>
  <hobbies>Swimming</hobbies>
  <hobbies>basketball</hobbies>
</employee>

```

使用了@XmlList注解之后

```
@XmlRootElement(name = "employee")
@XmlAccessorType(XmlAccessType.FIELD)
public class Employee implements Serializable {
  @XmlList
  List<String> hobbies;
}

<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<employee>
    <hobbies>Swimming basketball</hobbies>
</employee>
```

#### @XmlElementWrapper

根据集合来产生包装元素。它必须与collection属性一起使用。

```java
@XmlRootElement(name = "employee")
@XmlAccessorType(XmlAccessType.FIELD)
public class Employee implements Serializable {

  @XmlElementWrapper
  @XmlElement(name="hobby")
  List<String> hobbies;
}

```

生成的xml如下:

```java
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<employee>
  <hobbies>
    <hobby>Swimming</hobby>
    <hobby>basketball</hobby>
  </hobbies>
</employee>
```

当然了，它还可以有更具有组合性质的做法，比如

```java
@XmlRootElement(name = "employee")
@XmlAccessorType(XmlAccessType.FIELD)
public class Employee implements Serializable {

  @XmlElementWrapper
  @XmlElements({
     @XmlElement(name="work", type=Work.class),
     @XmlElement(name="family", type = Family.class)
  })
  List<Contact> contact;
}

@XmlRootElement
@XmlAccessorType(XmlAccessType.FIELD)
public class Contact {
}

@XmlRootElement
@XmlAccessorType(XmlAccessType.FIELD)
public class Family extends Contact{
  String phone;
  String name;
}

@XmlRootElement
@XmlAccessorType(XmlAccessType.FIELD)
public class Work extends Contact{
  String phone;
  String name;
}

```

最后输出的xml格式如下

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<employee>
  <contact>
    <work>
      <phone>028-123445</phone>
      <name>xxwork</name>
    </work>
    <family>
      <phone>18312345678</phone>
      <name>xx family</name>
    </family>
  </contact>
</employee>
```
#### @XmlJavaTypeAdapter

Xml和Java属性的转换器,用于复杂对象的转换

```java
@XmlRootElement(name = "employee")
@XmlAccessorType(XmlAccessType.FIELD)
public class Employee implements Serializable {
    
    @XmlJavaTypeAdapter(Country2String.class)
    Country country;
}

class Country2String extends XmlAdapter<String, Country> {

    @Override
    public Country unmarshal(String code) throws Exception {
        return Country.findCountry(code);
    }

    @Override
    public String marshal(Country country) throws Exception {
        return country.getCode();
    }
}

@Data
@AllArgsConstructor
@NoArgsConstructor
public class Country {

  private String code;

  private String name;

  public static final List<Country> INSTANCES = new ArrayList<>();

  static {
      INSTANCES.add(new Country("zh", "china"));
  }
  
  public static Country findCountry(String codeParam) {
    return INSTANCES.stream()
            .filter(country -> codeParam.equalsIgnoreCase(country.getCode()))
            .findFirst()
            .get();
  }
    
}
```

### JAXB Example

最后附上JAXB转换xml的例子

```java
private static void jaxbObjectToXML(Employee employee) {
  try {
    JAXBContext jaxbContext = JAXBContext.newInstance(Employee.class);
    Marshaller jaxbMarshaller = jaxbContext.createMarshaller();

    jaxbMarshaller
            .setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, Boolean.TRUE);

    StringWriter sw = new StringWriter();

    //Write XML to StringWriter
    jaxbMarshaller.marshal(employee, sw);

    //Verify XML Content
    String xmlContent = sw.toString();

    System.out.println( xmlContent );

  } catch (JAXBException e) {
    e.printStackTrace();
  }
}

```
