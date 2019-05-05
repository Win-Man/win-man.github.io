---
title: 从Lambda到方法引用
date: 2016-09-25 11:08:13
tags:
- Java8
- Lambda
- 方法引用
categories:
- Java
- 基础知识
---

　　Java8 中引入的 Lambda 表达式已经能在某些方面很大程度上简化我们的代码了，但是如果问有没有更加优雅的码代码方式呢，答案是肯定的。Java8 中出现了一个新的功能：**方法引用**。

> 方法引用：仅仅调用特定方法的 Lambda 的一种快捷写法。

　　方法引用，让我们以更加简洁，语义化的方式去码写我们的代码。

## 如何将 Lambda 表达式转换为方法引用

　　比方说我现在有一个 `List<String>` 类型的列表，里面存了一些内容：

``` java
List<String> numStrings = new ArrayList<>();
numStrings.add("123");
numStrings.add("234");
numStrings.add("345");
numStrings.add("456");
numStrings.add("567");
```

　　下面我们从对这个 List 进行一些操作来说明一下方法引用的三种类型。

### 一、指向**静态方法**的方法引用

　　比如说我想将这个 List 中所有的字符转转换成 int 类型的，用 Lambda 应该如何完成呢。我们先确定我们需要参数化的行为就是：对字符串进行某种特定的操作，然后我们直接使用 Java 内置的 Function 函数式接口来完成任务。让我们来调用一下函数式接口：

``` java
public static List<Integer> strToInt(List<String> strs, Function<String,Integer> f){
    List<Integer> result = new ArrayList<>();
    for(String s:strs){
        result.add( f.apply(s));
    }
    return result;
}
```

　　接着将 Lambda 表达式传进去就好了：

``` java
public static void main(String[] args){
    List<String> numStrings = new ArrayList<>();
    numStrings.add("123");
    numStrings.add("234");
    numStrings.add("345");
    numStrings.add("456");
    numStrings.add("567");
    List<Integer> ints = strToInt(numStrings,(String s) ->  Integer.parseInt(s));
    System.out.println(ints);
}
```
　　这就是 Lambda 的写法，如何转换成方法引用呢。看到我们其实是调用了 `Integer.parseInt()` 这个静态方法。对于这种指向静态方法的 Lambda 表达式转换规则如下:

>Lambda: (args) -> ClassName.staticMethod(args)

>方法引用： ClassName::staticMethod

　　所以 `(String s) ->  Integer.parseInt(s)` 可以改写为 `Integer::parseInt`，方法引用会自动完成传参等工作。

### 二、指向**任意类型实例方法**的方法引用

   　改写规则：

>Lambda: (arg0,rest) -> arg0.instanceMethod(rest)

>方法引用：ClassName::instanceMethod      /*ClassName是arg0的类型*/

   如果对上面的 List 进行处理，得到一个存放原先 List 中每个字符串长度的 List 的话。Lambda 写法如：`List<Integer> ints = strToInt(numStrings,(String s) -> s.length());`;参照改写规则，我们可以改写为: `List<Integer> ints = strToInt(numStrings,String::length);`。

### 三、指向**现有对象的实例方法**的方法引用

　　　改写规则：

>Lambda:(args) -> expr.instanceMethod(args)

>方法引用:expr::instanceMethod

　　如果对上面的 List 进行处理，得到一个存放原先 List 中每个字符串反转之后的 List 的话。Lambda 可以这么写:
``` java
public class FunctionReference {
    public static List<Integer> strToInt(List<String> strs, Function<String,Integer> f){
        List<Integer> result = new ArrayList<>();
        for(String s:strs){
            result.add( f.apply(s));
        }
        return result;
    }
    public static void main(String[] args){
        List<String> numStrings = new ArrayList<>();
        numStrings.add("123");
        numStrings.add("234");
        numStrings.add("345");
        numStrings.add("456");
        numStrings.add("567");
        StrClass sc = new StrClass();
        List<Integer> ints = strToInt(numStrings,(String s) -> sc.reverseInt(s));
        System.out.println(ints);
    }
}
class StrClass{
    public int reverseInt(String s){
        int n = Integer.parseInt(s);
        int result = 0;
        while(n > 0){
            result *= 10;
            result += (n % 10);
            n /= 10;
        }
        return result;
    }
}
```

　　根据改写规则，可以将 `(String s) -> sc.reverseInt(s)` 改写为 `sc::reverseInt`。

　　总结来说的话，就是看自己的 Lambda 表达式符合哪种改写规则的话，那就可以将 Lambda 表达式改写为方法引用的方式，但是不是每一个 Lambda 表达式都可以转变成方法引用的写法的。个人觉得将 Lambda 转换成方法引用的写法并不是很符合我的习惯（可能是看到::就想起了C++中的命名空间吧）。

## 构造函数的方法引用

　　对于一个类的构造函数，我们也可以用类名加上关键字 new 来创建这个构造函数的方法引用。我们先定义一个类：

``` java
class ParamterClass{
    public int a;
    public int b;
    public int c;
    public ParamterClass(){
        this.a = 0;
        this.b = 0;
        this.c = 0;
    }
    public ParamterClass(int a){
        this.a = a;
        this.b = 0;
        this.c = 0;
    }
    public ParamterClass(int a,int b){
        this.a = a;
        this.b = b;
        this.c = 0;
    }
}
```

　　这个类有三个构造方法，分别是一个无参构造函数，一个参数的构造函数，两个参数的构造函数。然后我们看一下该如何根据需要调用不同的构造函数呢：

``` java
public class ConstructReference {

    public static List<ParamterClass> noParamterCreator(Supplier<ParamterClass> s){
        List<ParamterClass> result = new ArrayList<>();
        for(int i = 0;i < 10;i++){
            result.add(s.get());
        }
        return result;
    }

    public static List<ParamterClass> oneParamterCreator(Function<Integer,ParamterClass> f){
        List<ParamterClass> result = new ArrayList<>();
        for(int i = 0;i < 10;i++){
            result.add(f.apply(i));
        }
        return result;
    }

    public static List<ParamterClass> twoParamterCreator(BiFunction<Integer,Integer,ParamterClass> b){
        List<ParamterClass> result = new ArrayList<>();
        for(int i = 0;i < 10;i++){
            result.add(b.apply(i,i+1));
        }
        return result;
    }

    public static void main(String[] args){
        List<ParamterClass> list1 = noParamterCreator(ParamterClass::new);
        List<ParamterClass> list2 = oneParamterCreator(ParamterClass::new);
        List<ParamterClass> list3 = twoParamterCreator(ParamterClass::new);
        for(ParamterClass p :list1){
            System.out.println(String.format("List1:a->%d b->%d c->%d",p.a,p.b,p.c));
        }
        for(ParamterClass p :list2){
            System.out.println(String.format("List2:a->%d b->%d c->%d",p.a,p.b,p.c));
        }
        for(ParamterClass p :list3){
            System.out.println(String.format("List3:a->%d b->%d c->%d",p.a,p.b,p.c));
        }
    }
}
```

　　用 Supplier 函数式接口调用无参构造函数，Function 函数式接口调用一个参数的构造函数，BiFunction 函数式接口调用两个参数的构造函数。其实方法引用的写法都是一样的 `ParamterClass::new`,实际上调用不同的构造函数式通过不同的函数式接口来实现的。

## 总结

　　方法引用的出现，是为了更加简洁的运用 Lambda 表达式，让我们的代码更加清晰明了。做一个优雅写代码的码农。
> 博客地址：https://win-man.github.io/  
> 公众号：欢迎关注  

![](https://user-gold-cdn.xitu.io/2018/8/16/165435ce71d2b88b?w=258&h=258&f=jpeg&s=26568)
