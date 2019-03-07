# Java8新特性

[TOC]



## 背景

* 顺应时代发展
* 之前版本抽象级别还不够
  - 面对大数据集合，欠缺高效的并发手段

## 函数式编程

### 概述

#### 方法和Lambda也作为一等公民

* 编程语言的整个目的就在于操作值，要是按照历史上编程语言的传统，这些值因此被称为一等值（或一等公民）

#### 行为参数化

##### 把函数作为值传递代码

##### 传递函数变为传递Lambda

### Lambda表达式

#### 函数式接口

##### 概述

- 只定义一个抽象方法的接口
- Lambda表达式允许你直接以内联的形式为函数式接口的抽象方法提供实现，并把整个表达式作为函数式接口的实例
- 函数式接口的抽象方法的签名基本上就是Lambda表达式的签名

##### 使用

- Predicate<T>
- Consumer<T>
- Function<T,R>

#### 类型

##### 类型检查：从上下文推断出Lambda需要的目标类型

- 同样的Lambda表达式，可能会有不同的函数式接口

##### 类型推断：从上下文推断出Lambda的参数类型

##### 限制

- 可以使用局部变量，但局部变量必须是final，或事实上是final的(赋值后不会被改变)

#### 方法引用

- 指向静态方法的方法引用
  - Integer的parseInt方法，写作Integer::parseInt
- 指向任意类型实例的方法引用
  - String的length方法，写作String::length，引用对象的一个方法，而这个对象就是该方法的参数
- 指向现有对象的实例的方法引用

#### 重构面向对象设计模式

##### 策略模式

``` java
public class TestValidator {

    public static void main(String[] args) {

        //use lambda

        numericValidator = new Validator((String s) -> s.matches("\\d+"));

        System.out.println(numericValidator.validate("aaaa"));

        lowerCaseValidator = new Validator((String s) -> s.matches("[a-z]+"));

        System.out.println(lowerCaseValidator.validate("bbbbb"));

    }

}

```

##### 模版方法模式

``` java
public abstract class OnlineBanking {

    public void processCustomer(int id, Consumer<Customer> makeCustomerHappy) {

        Customer customer = Database.getCustomerWithId(id);

        makeCustomerHappy.accept(customer);

    }

}
```

##### 观察者模式

``` java
// 无须显式地实例化观察者对象

interface Observer{

    void notify(String tweet);

}

interface Subject{

    void registerObserver(Observer observer);

    void notifyObservers(String tweet);

}

public class Feed implements Subject{

    private final List<Observer> observers = new ArrayList<>();

    

    @Override

    public void registerObserver(Observer observer) {

        this.observers.add(observer);

    }

    @Override

    public void notifyObservers(String tweet) {

        observers.forEach(o -> o.notify(tweet));

    }

}

public class TestFeed {

    public static void main(String[] args) {

        Feed feed = new Feed();

        feed.registerObserver((String tweet) -> {

            if (tweet != null && tweet.contains("money")) {

                System.out.println("Breaking news in NY! " + tweet);

            }

        });

//        feed.registerObserver(new Guardian());

        feed.notifyObservers("The queen said her favourite book is HarryPotter");

    }

}

 
```

##### 责任链模式

``` java
public class AssemblyLine {

    public static void main(String[] args) {

        UnaryOperator<String> headerProcessing = (String text) -> "From Raoul, Mario and Alan: " + text;

        UnaryOperator<String> spellCheckerProcessing = (String text) -> text..replaceAll("labda", "lambda");

        Function<String, String> pipeline = headerProcessing.andThen(spellCheckerProcessing);

        String result = pipeline.apply("Aren't labdas really sexy?!!");

    }

}

 
```

##### 工厂模式

``` java
public class ProductFactory{

    final static Map<String, Supplier<Product>> map = new HashMap<>();

    static {

        map.put("loan", Loan::new);

        map.put("stock", Stock::new);

        map.put("bond", Bond::new);

    }

    public static Product createProduct(String name) {

        Supplier<Product> product = map.get(name);

        if (product != null) {

            return product.get();

        }

        throw new IllegalAccessException("No such product " + name);

    }

}
```

### Java函数式编程注意项

* 函数式编程中不应该抛出任何异常，某些情况下需要处理的场景，可以使用Optional<T>包装
* 作为函数式程序，函数或方法调用的库函数如果有副作用，必须设法隐藏它们的非函数行为，否则就不能调用这些方法，需要确保它们对数据结构的任何修改对于调用者都是不可见的
  - 首次复制？？
  - 需捕获任何可能抛出的异常

## 流

### 解决的问题

- 集合处理时套路和晦涩
- 难以利用多核

### 概述

- 允许声明式地处理问题
- 遍历数据集的高级迭代器
  - 流水线操作
- 透明的并行处理，无须编写多线程代码
- 从支持数据处理操作的源生成的元素序列
  - 元素序列：像集合一样，流提供一个接口，可以访问特定元素类型的一组有序值
    - 集合是数据结构，主要目的是已特定的时间/空间复杂度存储和访问元素
      - 其元素都得先算出来才能添加到集合中
    - 流的目的在于表达计算
      - 概念上是固定的数据结构，其元素是按需计算的，只有在消费者要求的时候才会计算值
  - 源
    - 流会使用一个提供数据的源，如集合、数组或输入/输出资源
      - 从有序集合生成流会保留原有顺序
      - 由列表生成都的流，其元素顺序与列表一致
  - 数据操作处理
    - 类似于数据库的额操作，以及函数式编程语言中常用的操作
      - filter
      - map
      - reduce
      - find
      - match
      - sort
    - 可以顺序执行、也可并行执行
- 只能遍历一次，遍历完后，这个流已经被消费掉了

### 特点

- 流水线
  - 很多流操作本身会返回一个流，这样操作可以连接起来
  - 流水线的操作可以看作对数据源进行数据库式查询
- 内部迭代
  - 与使用迭代器显式迭代的集合不同，流的迭代操作是在背后进行的
  - 可以自动选择一种适合你硬件的数据表示和并行实现

### 使用

- 数据源：来执行一个查询
- 中间操作链：形成一条流的流水线
  - filter
  - map
  - limit
  - sorted
  - distinct
- 终端操作：执行流水线，并能生成结果
  - forEach
  - count
  - collect

## 默认方法

### 解决的问题

- 改变已发布的接口而不破坏已有的实现

### 概述

- 接口可以包含实现类没有提供实现方法的签名

## 更多的描述性数据避免null

- Optional<T>

## 模式匹配