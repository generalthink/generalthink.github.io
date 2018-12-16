title: Graphql在项目中的使用(二)
date: 2018-10-30 20:56:08
tags: Graphql
categories: Graphql
description: GraphQLResolver的使用,使用GraphQLResolver自定义接口数据的返回
keywords: Graphql,GraphQLResolver,SpringMVC
---

在上一篇文章中我们介绍了Graphql集成SpringMVC，今天就来介绍下GraphQLResolver的使用。
假设我们想要Book这个API返回desc这个字段,desc可能并不在数据库中,或者是你想个性化定制你的返回数据,这个时候就轮到GraphQLResolver了。

### GraphQLResolver的使用

1. 首先将schema.graphqls中关于book的声明变更如下：

```
type Book {
id: Long
name: String
url: String
person: Person
desc: String
}
```
但是Book类的声明仍然没有任何变化

```java
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

 ```

 但是我们的声明中是添加了desc这个字段的,那么我们怎么处理这个字段呢？


 ```java
import com.coxautodev.graphql.tools.GraphQLResolver;
import com.generalthink.graphql.com.generalthink.graphql.bean.Book;
import org.springframework.stereotype.Component;

@Component
public class BookResolver implements GraphQLResolver<Book> {
    //获取desc字段的处理方式，getUpperFieldName
    public String getDesc(Book book) {
        if("Java".equals(book.getName())) {
            return "great book";
        } else {
            return "unrecommened book";
        }
    }

}

```
2. 在GraphqlEndopoint中声明需要处理的resolver:

```java

@Component
public class GraphqlEndpoint extends SimpleGraphQLServlet {

@Autowired
public GraphqlEndpoint(Query query, Mutation mutation, BookResolver bookResolver) {
super(SchemaParser.newParser()
.file("schema.graphqls")
.resolvers(query,mutation,bookResolver)
.scalars(Scalars.JAVA_LONG)
.build()
.makeExecutableSchema());
}
}

```

此时使用grapqhql playgound发起请求：
![grapqhl palyground](/images/graphql2-playground-post.png)

### Graphql Playground的使用

playground工具还有一个chome插件版,可以在应用商店搜索graphiql安装,同样可以提供一样的功能，上面看了知道如何发起请求，但是在我们前端中如何发起请求呢？

首先打开chrome调试工具
![打开dev Tool](/images/graphql2-playground-open-console.png)

发起graphql请求，可以在dev tools中看到具体的请求，此时可以将请求复制为cURL或者查看具体的发起的请求体
![请求拷贝为cURL](/images/graphql2-devtool-copy2curl.png)

这里我们把具体请求导出为cURL，然后导入到postman中
![导入请求到postman](/images/graphql2-curl-postman.png)

查看请求结果
![在postman中发起请求](/images/graphql2-get-request-in-postman.png)

会发现在web请求中，是将请求参数全部放到body体中发送到后端的。在这次使用过程中，发现graphql所有返回的参数都是可以由前端自己定制的,而且使用graphql playground的过程可以让我们知道请求需要哪些参数，返回可以由哪些值，使用还是极为方便的。