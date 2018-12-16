title: Graphql在项目中的使用
date: 2018-09-27 21:59:04
tags: Graphql
categories: Graphql
description: Graphql是一种API的查询语言,由Facebook开源，旨在替代Restful,最近公司刚好有使用就研究了一下，顺便总结出来
keywords: SpringMVC集成Graphql,Graphql中集成MongoDB
---

### 什么是GraphQL

GraphQL 是一种用于 API 的查询语言。 GraphQL 对你的 API 中的数据提供了一套易于理解的完整描述，使得客户端能够准确地获得它需要的数据，而且没有任何冗余，也让 API 更容易地随着时间推移而演进，还能用于构建强大的开发者工具。GraphQL 查询不仅能够获得资源的属性，还能沿着资源间引用进一步查询。典型的 REST API 请求多个资源时得载入多个 URL，而 GraphQL 可以通过一次请求就获取你应用所需的所有数据。GraphQL API 基于类型和字段的方式进行组织，而非入口端点。你可以通过一个单一入口端点得到你所有的数据能力。

Facebook 的移动应用从 2012 年就开始使用 GraphQL。GraphQL 规范于 2015 年开源，现已经在多种环境下可用，并被各种体量的团队所使用。

### 最好的入门文档

最好的文档当然是[官方文档](https://www.howtographql.com/graphql-java/1-getting-started/)

### SpringMVC集成GraphQL


#### 引入Java包

这里为了尽量的展示整个集成流程，所以会使用SpringBoot来进行整体框架搭建。但是会将更多的流程聚焦在GraphQL中。所以第一步是引入GraphQL的jar包，关于SpringBoot的搭建,可以借助<http://start.spring.io/>来快速搭建。这里不做过多叙述。

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>3.0.1</version>
            <scope>provided</scope>
        </dependency>

        <!--集成GraphQL-->
        <dependency>
            <groupId>com.graphql-java</groupId>
            <artifactId>graphql-java</artifactId>
            <version>9.0</version>
        </dependency>
        <dependency>
            <groupId>com.graphql-java</groupId>
            <artifactId>graphql-java-tools</artifactId>
            <version>5.2.3</version>
        </dependency>
        <dependency>
            <groupId>com.graphql-java</groupId>
            <artifactId>graphql-java-servlet</artifactId>
            <version>5.2.0</version>
        </dependency>
        
    </dependencies>
```

### 开始开发

这里可以考虑实现一个具体的场景,就以二手书网站卖书举例好了。在GraphQL中查询和修改(增加,删除,更新)都是分别对应Query和Mutation.GraphQL支持的数据类型只有Int(Java中的Integer),Float,String,Enum以及Type(相当于Java中的Class),所以对于其他不支持的类型需要使用scalars自定义。

#### 定义接口描述(schema.graphqls)

```

type Book {
    id: Long
    name: String
    url: String
    person: Person
}

# 注意这里得关键字是 input
input PersonParam {
    name: String
    age: Int
}

type Person {
    name: String
    age: Int
}

type Query {
   books(name: String, size: Int): [Book]
}

type Mutation {
    newBook(name: String, url: String, person: PersonParam): Long
}

schema {
    query: Query
    mutation: Mutation
  }

scalar Long

```

在GraphQL中如果参数是一个对象而不是标量值(Float,Int,String,Enum)的话，那么数据类型需要申明为input.上面可以看到PersonParam和Person的字段都是一样的，那么两个可以使用为同一个吗？
我告诉你不行，因为PersonParam是输入参数,而Person是输出参数，两者在GraphQL中不能混为一谈,如果你这样做了，程序会报错。


#### 定义接口实现

```java


@Component
public class Query implements GraphQLQueryResolver {

    @Autowired
    private BookRespository bookRespository;

    //方法名称,参数以及返回类型要和schema.graphqls文件中的定义保持一致.DataFetchingEnvironment是GraphQL提供的上下文,其中可以获取到request,response
    public List<Book> books(String name,Integer size,DataFetchingEnvironment env) {
        //AuthContext context = env.getContext()   用于权限过滤

        return bookRespository.getBooks(name,size);
    }

}
@Component
public class Mutation implements GraphQLMutationResolver {

    @Autowired
    private BookRespository bookRespository;

    //接口需要和schema.graphqls中保持一致
    public Long newBook(String name, String url, PersonParam personParam) {
        Person person = new Person(personParam.getName(),personParam.getAge());

        return bookRespository.saveBook(name,url,person);
    }

}

```

这里的接口实际上也是请求的最终落地点，可以想成SpringMVC中的Controller，最终的执行逻辑就会从这里开始。

#### 集成MongoDB

##### application.properties中设置mongodb参数

```
spring.data.mongodb.uri=mongodb://localhost:27017/gln-local-test
```

##### 添加对MongoDB的访问

```java
@Component
public class BookRespository {

    @Autowired
    private MongoTemplate mongoTemplate;

    public List<Book> getBooks(String name, Integer size) {

        if(null == size) {
            size = 10;
        }
        Query query = new Query();
        query.addCriteria(Criteria.where("name").alike(Example.of(name))).limit(size);

        return mongoTemplate.findAll(Book.class, "book");
    }

   public Long saveBook(String name, String url, Person person) {

        long id = new Random().nextLong();

        Book book = Book.builder().id(id).name(name).url(url).person(person).build();


        mongoTemplate.save(book,"book");

        return id;
   }

}
```

#### 添加Scalar

刚才提到了GraphQL,它只提供了Int,Float,String,Enum以及自定义Type(class)类型,对于其他的需要自定义,实际上也就是定义如何序列化以及反序列化,这种自定义类型被称为scalar.刚才接口中定义了Long,所以我们需要写一个Long的scalar.实际上就是自定义序列化以及反序列化的规则

```java
public class Scalars {

    public final static GraphQLScalarType JAVA_LONG = new GraphQLScalarType("Long", "Java Long scalar",
            new Coercing() {
                @Override
                public Long serialize(Object dataFetcherResult) throws CoercingSerializeException {
                    return Long.parseLong(dataFetcherResult.toString());
                }

                @Override
                public Object parseValue(Object input) throws CoercingParseValueException {
                    return serialize(input);
                }

                @Override
                public Long parseLiteral(Object input) throws CoercingParseLiteralException {
                    if(input instanceof StringValue) {
                        return Long.parseLong(((StringValue) input).getValue());
                    } else if(input instanceof IntValue) {
                        return ((IntValue) input).getValue().longValue();
                    }

                    return null;
                }
            });
}

```

#### 添加SpringMVC支持

之前的配置只是对接口定义的诠释，但是要谈到和SpringMVC对接其实还没有，那么接下来的配置就是和SpringMVC进行对接了

```java
@Component
public class GraphqlEndpoint extends SimpleGraphQLServlet {

    @Autowired
    public GraphqlEndpoint(Query query, Mutation mutation) {
        super(SchemaParser.newParser()
                .file("schema.graphqls")
                .resolvers(query,mutation)      //引如resovler
                .scalars(Scalars.JAVA_LONG)  //引入刚才定义的scalar
                .build()
                .makeExecutableSchema());
    }
}


//统一的入口,graphQL通过这个统一入口将请求进行分发给各个resolver
@Controller
public class GraphqlController {


    @Autowired
    private GraphQLServlet graphQLServlet;

    @RequestMapping("/graphql")
    public void graphql(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {

        graphQLServlet.service(request,response);


    }


}

```

#### 添加对应的实体类

```java

@Data
@AllArgsConstructor
@NoArgsConstructor
public class Person {

    private String name;

    private String age;
}

@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class Book {
    private Long id;
    private String name;
    private String url;
    private Person person;
}

@Data
@AllArgsConstructor
@NoArgsConstructor
public class PersonParam {

    private String name;

    private String age;
}
```

#### 开始真正的表演了

接下来就要开始发起请求了,那么我们可以使用GraphQL Playground(或者google graphiql插件)或者post man来进行测试,这里我两个都使用

![postman get测试](/images/graphql-postman-get.png)


![graphql background测试](/images/graphql-test-background.png)



query后面那一串其实是
query {
  books(name:"MongoDB",size:10) {
    name
    url
    person {
      name
      age
    }
  }
}
的url转义。所以如果使用get方法发起请求，将查询参数放到url后面，需要将参数转义之后发起请求。如果使用post方法则值需要将请求体放到body中即可


#### 关于权限的验证

这里给出一个自定义的实现,在GraphqlEndpoint复写下面的方法,然后在Query或者Mutation中就可以通过DataFetchingEnvironment获取到自定义的context,通过其中的request来进行权限验证了。

```java
    protected GraphQLContext createContext(Optional<HttpServletRequest> request, Optional<HttpServletResponse> response) {
            
        //从request获取到cookie token等能证明用于身份的数据,解析出user

        return AuthContext(user,request,response);

    }


    public class AuthContext extends GraphQLContext {

    private final AuthUser authUser;

    public AuthContext(AuthUser user, Optional<HttpServletRequest> request, Optional<HttpServletResponse> response) {
        super(request, response);
        this.authUser = user;
    }

    public AuthUser getAuthUser() {
        return authUser;
    }
}
```


### 参考文档

http://graphql.github.io/learn/schema/

https://www.howtographql.com/graphql-java/1-getting-started/





