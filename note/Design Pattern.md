## 简单工厂
> 又称“静态工厂方法模式”

组成：
1. 一个工厂类
2. 一个工厂类的静态生产方法
	1. 条件判断（根据不同条件`new`相应的对象，如名称），违背了开闭原则，有新的产品需要修改代码
	2. 改进，根据反射的方式来`new`对象
3. 多个产品类

### 代码示例
#### 产品接口
```java
public interface Shape {
    void print();
}
```
#### 具体产品类
```java
public class Circle implements Shape {
    @Override
    public void print() {
        System.out.println("I am a circle.");
    }
}
```
```java
public class Triangle implements Shape {
    @Override
    public void print() {
        System.out.println("I am a triangle.");
    }
}
```
#### 简单工厂类
```java
public class SimpleFactory {
    // 通过名称来生产
    public static Shape createOneShape(String name) {
        switch (name) {
            case "triangle":
                return new Triangle();
            case "circle":
                return new Circle();
            default:
                return null;
        }
    }
    // 根据反射来生产
    public static Shape createOneShape(Class<? extends Shape> clz) throws Exception {
        return clz.newInstance();
    }
}
```
