---
layout:     post
title:      Mybatis动态SQL简介
date:       2019-08-15
author:     iamwzt
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - MyBatis
---
使用传统的JDBC方法时，拼接SQL语句是个头疼的事，一不留神就少个空格多个逗号出错了。
Mybatis的动态SQL解决了这一痛点，通过几个标签便可灵活地组成SQL语句了。
和动态SQL有关的标签有：
- if
- choose (when, otherwise)
- trim (where, set)
- foreach

下面逐一介绍。

### 条件判断 if
还是用前面的例子来演示。现在有个场景，要根据age来查找男性user记录，但age有可能不传。
```xml
<select id="findUserByAge" resultType="user">
    SELECT * FROM tb_user WHERE 
    <if test="age != null">
        `age` = #{age}
    </if>
    AND `gender` = "male"
</select>
```
在这里用到了 `<if/>` 标签，如果age不为null，才会进行拼接。

但是，这里有个问题，如果age为null，岂不是多出个**AND**？

当然，可以把固定的条件放到前面，把 AND / OR 这类关键字放到后面可选的条件中。不过Mybatis还提供了更灵活的方式。

### 动态的where
根据上面提出的问题，可以将SQL语句改成这样：
```xml
<select id="findUserByAge" resultType="user">
    SELECT * FROM tb_user
    <where>
        <if test="age != null">
            `age` = #{age}
        </if>
        AND `gender` = "male"
    </where>
</select>
```
`<where/>`标签只有里面有条件语句时，才会拼接 WHERE，并且若第一个条件是 AND 或 OR 开头，会自动将其去除。

`<where/>`标签能做的，`<trim/>` 标签也可以，甚至可以做得更灵活。下面是上面 where 的 trim 实现方式：
```xml
<trim suffix="WHERE" prefixOverrides="AND |OR ">
    <!-- 省略条件语句 -->
</trim>
```
与`<where/>`相似的动态SQL还有`<set/>`

### 动态的set
和`<where/>`类似的，`<set/>`处理的UPDATE语句中的拼接问题：
```xml
<update id="updateUser" parameterType="user">
    UPDATE tb_user
    <set>
        <if test="name != null">name = #{name},</if>
        <if test="age != null">age = #{age},</if>
        <if test="gender != null">gender = #{gender},</if>
    </set>
    <where> id = #{id} </where>
</update>
```
和`<where/>`不同的是，`<set/>`会把最后的逗号给去掉。与之等效的`<trim/>`是：
```xml
<trim suffix="SET" prefixOverrides=",">
    <!-- 省略条件语句 -->
</trim>
```

### 循环语句 foreach
`<foreach/>`标签用来遍历一个集合，通常是用来构建IN条件语句的：
```xml
<select id="selectUserIn" resultType="user">
    SELECT * FROM tb_user
    WHERE id IN
    <foreach collection="list" open="(" close=")" separator="," index="index" item="item">
        #{item}
    </foreach>
</select>
```
```java
List<UserDo> selectUserIn(List<String> ids);
```
其中 collection 属性可选的值是：
- List 为 list；
- 数组为 array；
- 可用@Param 注解自定义变量名。

若为数组或List，则index为其索引；若为Map，则为key。

### 多重选择 choose
类似java的switch，动态SQL `<choose/>` 可以实现多重选择：
```xml
<select id="findUserByAgeANDGender" resultType="user">
    SELECT * FROM tb_user
    <where>
        <choose>
            <when test="age != null"> AND `age` = #{age} </when>
            <when test="gender != null"> AND `gender` = #{gender} </when>
            <otherwise> `age` > 0 AND `gender` in ('male', "female") </otherwise>
        </choose>
    </where>
</select>
```

--END--
