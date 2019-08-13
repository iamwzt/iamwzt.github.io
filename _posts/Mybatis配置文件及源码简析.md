### typeAliases 别名

#### 配置文件
```xml
<typeAliases>
    <typeAlias type="com.wzt.batis.UserDo" alias="UserDo"/>
    <package name="com.wzt.batis"/>
</typeAliases>
```
`<typeAliases/>`有两类子节点：
- `<typeAlias/>`：为某个类指定别名，`alias`属性可选，不设置时默认为非限定类名的小写；
- `<package/>`：为某个package下所有（非匿名/接口/局部类）类指定别名，若类无`@Alias`注解，默认为非限定类名的小写；

**要求：package 标签必须在 typeAlias 后面**

#### 源码简析

来看源码：
> XMLConfigBuilder#typeAliasesElement()

```java
  private void typeAliasesElement(XNode parent) {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        // 解析 package 子标签
        if ("package".equals(child.getName())) {
          String typeAliasPackage = child.getStringAttribute("name");
          configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
        } else {
          // 解析 typeAlias 子标签
          String alias = child.getStringAttribute("alias");
          String type = child.getStringAttribute("type");
          try {
            Class<?> clazz = Resources.classForName(type);
            if (alias == null) {
              typeAliasRegistry.registerAlias(clazz);
            } else {
              typeAliasRegistry.registerAlias(alias, clazz);
            }
          } catch (ClassNotFoundException e) {
            throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
          }
        }
      }
    }
  }
```
> 注册 package 子标签所指向的包下类<br>
> TypeAliasRegistry.java

```java
  public void registerAliases(String packageName, Class<?> superType) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> typeSet = resolverUtil.getClasses();
    for (Class<?> type : typeSet) {
      // 匿名类、接口、局部类，通通退散！
      if (!type.isAnonymousClass() && !type.isInterface() && !type.isMemberClass()) {
        // 看这里
        registerAlias(type);
      }
    }
  }
```
> 如果有@Alias注解，则用其值，否则用非限定类名；<br>
> 对单个 typeAlias 的解析也会进到这里

```java
  public void registerAlias(Class<?> type) {
    String alias = type.getSimpleName();
    Alias aliasAnnotation = type.getAnnotation(Alias.class);
    if (aliasAnnotation != null) {
      alias = aliasAnnotation.value();
    }
    // 向下
    registerAlias(alias, type);
  }

  public void registerAlias(String alias, Class<?> value) {
    if (alias == null) {
      throw new TypeException("The parameter alias cannot be null");
    }
    // 看这里，所有的别名最后都是小写的
    String key = alias.toLowerCase(Locale.ENGLISH);
    if (typeAliases.containsKey(key) && typeAliases.get(key) != null && !typeAliases.get(key).equals(value)) {
      throw new TypeException("The alias '" + alias + "' is already mapped to the value '" + typeAliases.get(key).getName() + "'.");
    }
    // 所谓注册，就是放到一个Map里
    typeAliases.put(key, value);
  }
```

总结一下：
- 通过 `<typeAlias/>` 标签可以单独为某个类设置别名；
- 通过 `<package/>` 标签可以为某个包下所有类设置别名（匿名类、接口、局部类除外）；
- `<typeAlias/>` 不指定别名，则用类的非限定名作为别名；
- `<package/>` 若没在类上加 `@Alias` 注解，则用类的非限定名作为别名；
- 类的别名注册在一个Map中，别名实际都是小写的；
- 有些我们可以直接在Mapper文件中使用的别名（如string/int等），是Mybatis提前给注册好的。

### TypeHandler 类型处理器
#### 配置文件
```xml
<typeHandlers>
    <!-- handler属性直接配置我们要指定的TypeHandler -->
    <typeHandler handler=""/>
    <!-- javaType 配置java类型，例如String, 如果配上javaType, 那么指定的typeHandler就只作用于指定的类型 -->
    <typeHandler javaType="" handler=""/>
    <!-- jdbcType 配置数据库基本数据类型，例如varchar, 如果配上jdbcType, 那么指定的typeHandler就只作用于指定的类型  -->
    <typeHandler jdbcType="" handler=""/>
    <!-- 也可两者都配置 -->
    <typeHandler javaType="" jdbcType="" handler=""/>
    <!--当配置package的时候，mybatis会去配置的package扫描TypeHandler-->
    <package name="com.wzt.batis"/>
</typeHandlers>
```

#### 源码简析
```java
  private void typeHandlerElement(XNode parent) {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        // 解析 package 标签
        if ("package".equals(child.getName())) {
          String typeHandlerPackage = child.getStringAttribute("name");
          typeHandlerRegistry.register(typeHandlerPackage);
        } else {
          // 解析其它 typeHandler
          // 参数可以有 javaType/jdbcType，也可两者都有
          String javaTypeName = child.getStringAttribute("javaType");
          String jdbcTypeName = child.getStringAttribute("jdbcType");
          String handlerTypeName = child.getStringAttribute("handler");
          Class<?> javaTypeClass = resolveClass(javaTypeName);
          JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
          Class<?> typeHandlerClass = resolveClass(handlerTypeName);
          if (javaTypeClass != null) {
            if (jdbcType == null) {
              typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
            } else {
              typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
            }
          } else {
            typeHandlerRegistry.register(typeHandlerClass);
          }
        }
      }
    }
  }
```
> TypeHandlerRegistry.java<br>
> 在这个类中，Mybatis已经注册了很多默认的 handler

来看几个注册方法：
```java
  // 配置文件中没有配置 javaType 参数的注册方法： 
  public <T> void register(TypeHandler<T> typeHandler) {
    boolean mappedTypeFound = false;
    //  在自定义typeHandler的时候，可以加上注解MappedTypes 去指定关联的javaType
    //  因此，此处需要扫描MappedTypes注解
    MappedTypes mappedTypes = typeHandler.getClass().getAnnotation(MappedTypes.class);
    if (mappedTypes != null) {
      for (Class<?> handledType : mappedTypes.value()) {
        register(handledType, typeHandler);
        mappedTypeFound = true;
      }
    }
    // @since 3.1.0 - try to auto-discover the mapped type
    if (!mappedTypeFound && typeHandler instanceof TypeReference) {
      try {
        TypeReference<T> typeReference = (TypeReference<T>) typeHandler;
        register(typeReference.getRawType(), typeHandler);
        mappedTypeFound = true;
      } catch (Throwable t) {
        // maybe users define the TypeReference with a different type and are not assignable, so just ignore it
      }
    }
    if (!mappedTypeFound) {
      register((Class<T>) null, typeHandler);
    }
  }
  
  // 配置了 javaType 的注册方法
  public <T> void register(Class<T> javaType, TypeHandler<? extends T> typeHandler) {
    register((Type) javaType, typeHandler);
  }  

  // 三个参数齐全的注册方法
  private void register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler) {
    if (javaType != null) {
      Map<JdbcType, TypeHandler<?>> map = typeHandlerMap.get(javaType);
      if (map == null || map == NULL_TYPE_HANDLER_MAP) {
        map = new HashMap<>();
        typeHandlerMap.put(javaType, map);
      }
      map.put(jdbcType, handler);
    }
    allTypeHandlersMap.put(handler.getClass(), handler);
  }
```

所谓的注册和 typeAlias 一样，就是添加到Map中。

#### 实现一个自定义的 TypeHandler
Mybatis 实现了很多默认的 handler，都是继承自 `BaseTypeHandler`，我们要实现自己的自定义Handler，也需要继承这个类。
```java
// 两个注解分别对应配置文件中的 jdbcType 和 javaType 参数
@MappedJdbcTypes(JdbcType.VARCHAR)
@MappedTypes(String.class)
public class MyTypeHandler extends BaseTypeHandler<String> {
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, parameter);
    }

    @Override
    public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return rs.getString(columnName);
    }

    @Override
    public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return rs.getString(columnIndex);
    }

    @Override
    public String getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return cs.getString(columnIndex);
    }
}
```



