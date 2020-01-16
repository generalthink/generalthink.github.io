---
title: 你真的会用Java8 Optional吗？
date: 2020-01-16 14:32:24
tags: Java
---


Java8之前我们在写代码的时候，经常会遇到返回null的情况，如果这种情况不加以判断,你就会碰到NullPointerException(NPE)。而在Java8中，Optional类型是一种更好的表示缺少返回值的形式。

首先来看一段代码,这可能是以前大多数人的写法

```java
private void getIsoCode( User user){
  if (user != null) {
    Address address = user.getAddress();
    if (address != null) {
      Country country = address.getCountry();
      if (country != null) {
          String isocode = country.getIsocode();
          if (isocode != null) {
              isocode = isocode.toUpperCase();
          }
      }
    }
  }
}
```

<!--more-->

而当我们有了Optional之后,上面的代码就可以缩减很大一部分，答案我会在后面给出。

我见过有人的写法是这样的

```java
Optional<T> optionalValue = ... ;
optionalValue.get().someMethod();

或者

if(optionalValue.isPresent()) {
  optionalValue.get().someMethod();
}
```
它并不比下面的方式安全

```java
T value = ... ;
value.someMethod();

或者

if(value != null) {
  value.someMethod();
}

```

如果你在你的代码中出现了上面使用Optional的片段，那么你该好好优化下了。


其实高效使用Optional的关键在于，使用一个 **接受正确值或者返回另一个替代值** 的方法。

### 创建Optional

```java

// 创建一个空的Optional实例
public static<T> Optional<T> empty() {
  Optional<T> t = (Optional<T>) EMPTY;
  return t;
}

// 根据传入的值创建一个非空的实例, value不能为空，否则抛出NPE
 public static <T> Optional<T> of(T value) {
  return new Optional<>(value);
}

// 根据传入的值创建实例，value可以为空
public static <T> Optional<T> ofNullable(T value) {
  return value == null ? empty() : of(value);
}
```

上面的三个方法就是我们构造Optional的途径,可以看到实际上Optional是对value的封装。
**需要注意的是`ofNullable` 和 `of`的区别，推荐使用`ofNullable`方法**


### 其他常用方法

1. `boolean isPresent()`
**判断value是否存在**

2. `Optional<T> filter(Predicate<? super T> predicate)`   
**判断value是否满足条件**


3. `Optional<U> map(Function<? super T, ? extends U> mapper)`
**对其中的value执行一个函数，将其变成另一个值。返回的值会被Optional.ofNullable封装**

4. `Optional<U> flatMap(Function<? super T, Optional<U>> mapper)`
**也是对其中的value执行一个函数,注意和map的区别在于,执行这个函数返回的值是Optional类型，返回的值不会被封装。**

5. `orElse(T other)`
**当value存在时,返回value，不存在时返回other**

6. `orElseGet(Supplier<? extends T> other)`
**和orElse一样，只是这里的参数是通过传入的function来决定的**

7. `T orElseThrow`
**当value存在时,返回value，不存在时抛出异常**


### 代码优化

开头给出的代码就可以被优化为下面的代码

```java
 String isoCode = Optional.ofNullable(user)
    .map(User::getAddress)  //Optional<Address>
    .map(Address::getCountry)  //Optional<Country>
    .map(Country::getIsocode)  // Optional<String>
    .orElse("empty");
```

不过有一点需要注意的是orElse和orElseGet的区别在于,无论是否满足条件orElse中的方法始终会被执行,而orElseGet中的只有当value为空时才会执行。

```java
Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCountry)
    .map(Country::getIsocode)
    .orElse(getIsoCode());

```

你可能觉得没什么，但是如果你的业务中获取这个值要去数据库查询,那么每一次只要运行这个代码就都要去查询，这样就造成了不必要的性能损失了,还是一个很大的问题的。


### 技术总结

1. Optional是为了更优雅的判断null而诞生的,但是并不代表有null的地方一定就要用Optional代替
2. Optional一般用于方法返回值,不用于属性(无法被序列化)
3. Optional用于多层次null判断有奇效
