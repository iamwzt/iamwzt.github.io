## 建造者模式
> 又称生成者模式，封装一个类的具体创建过程，使用者无需知道类的具体组成

### 组成
1. 一个产品类（`Product`）
2. 一个抽象的建造者（`Builder`）
3. 具体的建造者（`ConcreteBuilder`）
4. 一个指挥者（`Director`）

### 示例代码
> 以画画为例，画一幅简单的画需要选定纸张(paper)、图形(shape)、颜色(color)
#### 产品类
```java
// 省略 getter/setter 方法
public class Picture {
    private String color;
    private String shape;
    private String paper;
}
```
#### 抽象建造者
```java
public abstract class AbstractBuilder {
    protected Picture picture = new Picture();

    public abstract void setColor(String color);

    public abstract void setPaper(String line);

    public abstract void setShape(String shape);

    public Picture paint() {
        return picture;
    }
}
```
#### 具体建造者
```java
public class Builder extends AbstractBuilder {
    @Override
    public void setColor(String color) {
        picture.setColor(color);
    }

    @Override
    public void setPaper(String paper) {
        picture.setPaper(paper);
    }

    @Override
    public void setShape(String shape) {
        picture.setShape(shape);
    }
}
```

#### 指挥者
```java
public class Artist {
    private AbstractBuilder builder;

    public Artist(AbstractBuilder builder) {
        this.builder = builder;
    }

    public void setBuilder(AbstractBuilder builder) {
        this.builder = builder;
    }

    public Picture paint() {
        builder.setColor("Red");
        builder.setShape("Circle");
        builder.setPaper("A4");
        return builder.paint();
    }
}
```

### 额外的
> 在《Effective Java》中提到，当一个类有大量的构造参数时，用重叠构造器模方式，或JavaBean的方式（多线程下可能导致不一致）都不是很合适，
> 此时用建造者方式也可解决该问题

#### 示例代码
```java
public class Product {
    private final String partA;
    private final String partB;

    public static class ProductBuilder{
        private Product product;
        private String partA;
        private String partB;

        public ProductBuilder partA(String partA){
            this.partA = partA;
            return this;
        }
        public ProductBuilder partB(String partB){
            this.partB = partB;
            return this;
        }
        public Product build(){
            return new Product(this);
        }
    }

    public Product(ProductBuilder builder) {
        this.partA = builder.partA;
        this.partB = builder.partB;
    }
}
```
```java
Product product = new Product.ProductBuilder().partA("AAA").partB("BBB").build();
```
> 比较易于阅读，还模拟了具名的可选参数，就像`Python`和`Scala`一样
