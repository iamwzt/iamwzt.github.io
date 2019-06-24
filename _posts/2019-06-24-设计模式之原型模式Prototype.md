---
layout:     post
title:      设计模式之原型模式
subtitle:   原型模式简介
date:       2019-06-25
author:     iamwzt
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - design pattern
    - prototype
---

## 原型模式
> 简而言之，以`clone`代替`new`；<br>
> 适用于大量创建对象/对象依赖对象复杂，以致创建成本高的场景下

> 利用了所有类都有的`clone()`方法（继承自Object类），来实现对象的复制，根据类属性是否复制，可分成**深拷贝**和**浅拷贝**

### 组成
1. 原型接口（继承`Cloneable`接口，该接口是个标记接口，没有任何方法）
2. 实现原型接口的类
3. 管理原型的类

### 代码示例
#### 原型接口
```java
// 不实现Cloneable的类在运行clone()方法时，会抛出 CloneNotSupportedException 异常
public interface Product extends Cloneable {
    void use();
    void createClone();
}
```

#### 实现原型接口的类
```java
public class ProductA implements Product {
    private String name;

    public ProductA(String name) {
        this.name = name;
    }

    @Override
    public void use() {
        System.out.printf("I am product: [%s]!\n", name);
    }

    @Override
    public Product createClone() throws CloneNotSupportedException {
        return (Product) clone();
    }
}
```

#### 管理原型类clone的类
```java
public class Manager {
    private static Map<String, Product> dict = new HashMap<>();
    public static void register(String name, Product product){
        dict.put(name, product);
    }
    public static Product getClone(String name) throws CloneNotSupportedException {
        Product product = dict.get(name);
        if (product != null) {
            return product.createClone();
        }
        return null;
    }
}
```
#### 使用者
```java
public class User {
    public static void main(String[] args) throws CloneNotSupportedException {
        Product p1 = new ProductA("p1");
        Product p2 = new ProductA("p2");
        Manager.register("p1",p1);
        Manager.register("p2",p2);
        Product p11 = Manager.getClone("p1");
        Product p22 = Manager.getClone("p2");
        p11.use();
        p22.use();
    }
}
```
控制台输出：
```
I am product: [p1]!
I am product: [p2]!
```
