在mapper映射文件中，以`<mapper/>`作为根节点，其下可以有的子节点分别是：
```
select, insert, update, delete, cache, cache-ref, resultMap, parameterMap, sql
```
本节就来介绍这些子节点。

### 先来看看 insert
insert, update, delete三个标签的属性和使用方法相似，这里就以insert为例介绍。
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<!-- namespace 的值对应一个DAO -->
<mapper namespace="com.wzt.batis.UserMapper">
            <!-- 1. id 对应DAO中的一个方法 -->
    <insert id="insert"
            <!-- 2. parameterType 将要传入语句的参数的完全限定类名或别名
                这个属性是可选的，因为 MyBatis 可以通过 TypeHandler 推断出具体传入语句的参数 -->
            parameterType="user"
            <!-- 3. parameterMap 可以通过额外的标签来对入参进行更复杂的操作 -->
            parameterMap="uuu"
            <!-- 4. useGeneratedKeys 这会令 MyBatis 使用 JDBC 的 getGeneratedKeys 方法来取出由数据库内部生成的主键 
                 5. keyProperty 将获得的主键赋值给指定的Java类中的属性，如此处的id 
                 6. keyColumn 从数据库表的id列取主键值，数据库设定了主键则不需要显式配置 -->
            useGeneratedKeys="true" keyProperty="id" keyColumn="id"
            <!-- 7. flushCache 任何时候只要语句被调用，都会导致本地缓存和二级缓存都会被清空，默认为true-->
            flushCache="true" 
            <!-- 8. statementType STATEMENT，PREPARED 或 CALLABLE 的一个。这会让 MyBatis 分别使用 Statement，PreparedStatement 或 CallableStatement，默认值：PREPARED。-->
            statementType="PREPARED" 
            <!-- 9. timeout 这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数。-->
            timeout="20"/>
    
    <parameterMap id="uuu" type="user"/>
</mapper>
```
代码见真章：
#### 数据库表
```sql
CREATE TABLE `demoTable` (
  `id` INT COLLATE utf8_bin NOT NULL AUTO_INCREMENT COMMENT '编号',
  `name` VARCHAR (36) COLLATE utf8_bin NOT NULL COMMENT '姓名',
  `age` INT NOT NULL COMMENT '年龄',
  `gender` CHAR (10) COLLATE utf8_bin NOT NULL COMMENT '性别',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin
```
#### DO 类
省略getter/setter方法：
```java
public class UserDo {
    private Integer id;
    private String name;
    private Integer age;
    private String gender;
}
```
#### Mapper 接口
```java
@Mapper
public interface UserMapper {
    Integer insert(UserDo user);
}
```
#### Mapper 映射文件
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.wzt.batis.UserMapper">
    <insert id="insert" parameterType="user">
        INSERT INTO demoTable (id,name,age,gender)VALUES (
          #{id},#{name},#{age},#{gender}
        )
    </insert>
</mapper>
```
#### 测试类
```java
public class TestMain {
    public static void main(String[] args) {
        SqlSession sqlSession = getSessionFactory().openSession(true);
        UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
        UserDo user = new UserDo(1,"LiLei",12,"male");
        Integer res = userMapper.insert(user);
    }

    /**
     * Mybatis 通过SqlSessionFactory获取SqlSession, 然后才能通过SqlSession与数据库进行交互
     */
    private static SqlSessionFactory getSessionFactory() {
        SqlSessionFactory sessionFactory = null;
        String resource = "mybatis-config.xml";
        try {
            sessionFactory = new SqlSessionFactoryBuilder()
                    .build(Resources.getResourceAsReader(resource));
        } catch (IOException e) {
            e.printStackTrace();
        }
        return sessionFactory;
    }
}
```
执行后数据库的 demoTable 表中多了一条数据。

`userMapper.insert(user)`成功执行返回1（失败则为0）。

如果插入时不指定id，采用数据库的自增主键，并想要得到记录的id值，可以这样修改：
```xml
<insert id="insert" parameterType="user" useGeneratedKeys="true" keyProperty="id">
    INSERT INTO demoTable (name,age,gender)VALUES (
      #{name},#{age},#{gender}
    )
</insert>
```
或者使用 `<selectKey/>` 标签：
```xml
<insert id="insert1" parameterType="UseR">
    <selectKey keyProperty="id" resultType="int" order="AFTER" >
        SELECT LAST_INSERT_ID()
    </selectKey>
    INSERT INTO demoTable (name,age,gender)VALUES (
      #{name},#{age},#{gender}
    )
</insert>
```
如此，在`userMapper.insert(user)`后便可从`user`对象中取得id了。

**注意：** 数据库的自增id赋值给了`keyProperty`指定的属性，`insert`方法的返回值仍然是**1或0**。

这里用到了 `<selectKey/>` 标签，它可以返回记录在数据库中的id，看上去和`<insert/>`标签的属性`useGeneratedKeys`和`keyProperty`功能重复了。
那么除了这个作用外，`<selectKey/>`还有什么作用呢？

我们知道，MySQL支持主键自增，但是其它的数据库（如oracle）不支持啊；此外如果主键是UUID的，自增也无能为力了。此时`<selectKey/>`就能派上用场了。

先将DO类的id改成String类型，然后mapper映射文件变成这样：
```xml
<insert id="insert" parameterType="user">
    <!-- order变成了BEFORE，代表在insert前执行 -->
    <selectKey keyProperty="id" resultType="string" order="BEFORE" >
        SELECT UUID()
    </selectKey>
    <!-- 这里也加上了id列 -->
    INSERT INTO demoTable (id, name, age, gender)VALUES (
      #{id}, #{name}, #{age}, #{gender}
    )
</insert>
```
这样就会在insert语句前先执行selectKey，将UUID赋给id，然后再插入数据库。

不过在网上看到这里有个隐患，就是通过mybatis生成的UUID可能存在重复，emmm，所以实际工程中还是用java的UUID，更加方便，还不重复！

### 重头戏 select
