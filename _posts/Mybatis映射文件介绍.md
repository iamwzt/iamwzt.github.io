在mapper映射文件中，以`<mapper/>`作为根节点，其下可以有的子节点分别是：
```
select, insert, update, delete, cache, cache-ref, resultMap, parameterMap, sql
```
本节就来介绍这些子节点。先来看看insert，update 和 delete：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<!-- namespace 的值对应一个DAO -->
<mapper namespace="com.wzt.batis.UserMapper">
    <!-- 1. id 对应DAO中的一个方法 -->
    <insert id="add"
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
