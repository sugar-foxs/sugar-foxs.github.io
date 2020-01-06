---
layout:     post
title:      mybatis加载配置
subtitle:   
date:       2019-12-29 23:00:00
author:     sugar-foxs
catalog: 	true
tags:
    - mybatis
    - spring
---

> 这篇文章主要看下mybatis的配置文件配置了哪些东西，做了什么工作。
查看源码可以追踪到XMLConfigBuilder类，这个类主要是做加载配置工作，生成Configution。通过Configuration新建SqlSessionFactory，这个以后再细看，这篇主要看加载配置过程。
<!-- more -->

# XMLConfigBuilder
- 先看下这个类的代码，主要代码如下，

```java
public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    // 从根节点configuration开始解析
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }

  private void parseConfiguration(XNode root) {
    try {
      Properties settings = settingsAsPropertiess(root.evalNode("settings"));
      //issue #117 read properties first
      propertiesElement(root.evalNode("properties"));
      loadCustomVfs(settings);
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectionFactoryElement(root.evalNode("reflectionFactory"));
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```
- 从上面代码能够看到，在根节点下配置了10个子节点， 分别为：properties、typeAliases、plugins、objectFactory、objectWrapperFactory、settings、environments、databaseIdProvider、typeHandlers、mappers。

下面分别对10个子节点进行解析,进入具体的方法中找寻答案：

## properties
- 一起看下代码，注释写在其中：
```java
private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
      // 获取properties所有子节点property作为default，其实就是一个map
      Properties defaults = context.getChildrenAsProperties();
      // properties有resource和url属性，接着往下看
      String resource = context.getStringAttribute("resource");
      String url = context.getStringAttribute("url");
      // 这里表明resource和url只能有一个
      if (resource != null && url != null) {
        throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
      }
      // 接下来是对resource和url的不同处理
      // 获取到的k-v如果有相同k会覆盖之前property节点的值,因为是map，所以会覆盖
      if (resource != null) {
        defaults.putAll(Resources.getResourceAsProperties(resource));
      } else if (url != null) {
        defaults.putAll(Resources.getUrlAsProperties(url));
      }
      // 将这个节点下的配置加入到全局的configuration和parser中。
      Properties vars = configuration.getVariables();
      if (vars != null) {
        defaults.putAll(vars);
      }
      parser.setVariables(defaults);
      configuration.setVariables(defaults);
    }
  }
```
- 接下来看下resource和url有什么不同：
```java
public static Properties getResourceAsProperties(String resource) throws IOException {
    Properties props = new Properties();
    InputStream in = getResourceAsStream(resource);
    props.load(in);
    in.close();
    return props;
}

public static Properties getUrlAsProperties(String urlString) throws IOException {
    Properties props = new Properties();
    InputStream in = getUrlAsStream(urlString);
    props.load(in);
    in.close();
    return props;
}

// 可以看到差别只在getResourceAsStream和getUrlAsStream方法之间，他们的入参不同，返回对象相同。分别看到最底层方法。

public static InputStream getResourceAsStream(ClassLoader loader, String resource) throws IOException {
    InputStream in = classLoaderWrapper.getResourceAsStream(resource, loader);
    if (in == null) {
      throw new IOException("Could not find resource " + resource);
    }
    return in;
}
public static InputStream getUrlAsStream(String urlString) throws IOException {
    URL url = new URL(urlString);
    URLConnection conn = url.openConnection();
    return conn.getInputStream();
}
```
- resource是用来引入类路径下的资源，url是用来引入网络路径的资源，两者只能选其一。
- 共同点都是转成流获取到属性，只是获取的源头不一样而已。

## typeAliases
- 下面看下typeAliases节点，主要用途是定义别名
```java
private void typeAliasesElement(XNode parent) {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {

      //子节点如果是package，即指定包名，该包下的类都会有别名，别名默认是转成小写
      //也可以使用@Alias注解进行加别名，这个优先级比在配置文件里写的要高
      if ("package".equals(child.getName())) {
        String typeAliasPackage = child.getStringAttribute("name");
        configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
      } else {
        // 这里是typeAlias子节点，一个一个进行设置别名
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

## plugins
```
private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        // 解析每个Interceptor节点，生成Interceptor对象，并设置属性，加入到拦截器链中
        String interceptor = child.getStringAttribute("interceptor");
        Properties properties = child.getChildrenAsProperties();
        Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
        interceptorInstance.setProperties(properties);
        configuration.addInterceptor(interceptorInstance);
      }
    }
}
```
Interceptor有三个方法：intercept,plugin,setProperties。setProperties是用来设置属性给拦截器使用，主要方法是intercept和plugin。拦截器示例可见{% post_link 2019-12-29-mybatis拦截器分页 %}
- @Intercepts({ @Signature(type = StatementHandler.class, method = "prepare", args = { Connection.class, Integer.class }) })
  - 此注解是表名拦截那个类的哪个方法和方法的入参类型。其实就是拦截的规则，对哪些方法进行拦截。
  - 也可以写同一个类的不同方法，即对同一个类的不同方法进行拦截。

- interceptor ，该方法是拦截后的操作。

- plugin ,该方法一般这么写，Plugin.wrap(o, this); 作用是为目标类生成代理类。Plugin实现了InvocationHandler，熟悉动态代理的就知道了。

```java
  public static Object wrap(Object target, Interceptor interceptor) {
  Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
  Class<?> type = target.getClass();
  Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
  // 目标类的接口和Intercepts注解中signature的类有相同的，即符合注解中规则的，为其生成代理类
  if (interfaces.length > 0) {
    return Proxy.newProxyInstance(
        type.getClassLoader(),
        interfaces,
        new Plugin(target, interceptor, signatureMap));
  }
  return target;
}
```

下面看下代理类的执行方法：

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    Set<Method> methods = signatureMap.get(method.getDeclaringClass());
    if (methods != null && methods.contains(method)) {
      // 这里便是执行我们自己写的拦截器的intercept方法
      return interceptor.intercept(new Invocation(target, method, args));
    }
    return method.invoke(target, args);
  } catch (Exception e) {
    throw ExceptionUtil.unwrapThrowable(e);
  }
}
```

## objectFactory
```java
private void objectFactoryElement(XNode context) throws Exception {
  if (context != null) {
    // type是对象工厂类的类名,默认是DefaultObjectFactory
    String type = context.getStringAttribute("type");
    Properties properties = context.getChildrenAsProperties();
    ObjectFactory factory = (ObjectFactory) resolveClass(type).newInstance();
    // 设置属性
    factory.setProperties(properties);
    configuration.setObjectFactory(factory);
  }
}
```
- 知道了这个标签是生成对象工厂类，但是对象工厂类到底是做什么用的？
- 我们看先默认的对象工厂类DefaultObjectFactory
```java
public class DefaultObjectFactory implements ObjectFactory, Serializable {

  private static final long serialVersionUID = -8855120656740914948L;

  @Override
  public <T> T create(Class<T> type) {
    return create(type, null, null);
  }

  @SuppressWarnings("unchecked")
  @Override
  public <T> T create(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    Class<?> classToCreate = resolveInterface(type);
    // we know types are assignable
    return (T) instantiateClass(classToCreate, constructorArgTypes, constructorArgs);
  }

  @Override
  public void setProperties(Properties properties) {
    // no props for default
  }

  <T> T instantiateClass(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    try {
      Constructor<T> constructor;
      if (constructorArgTypes == null || constructorArgs == null) {
        constructor = type.getDeclaredConstructor();
        if (!constructor.isAccessible()) {
          constructor.setAccessible(true);
        }
        return constructor.newInstance();
      }
      constructor = type.getDeclaredConstructor(constructorArgTypes.toArray(new Class[constructorArgTypes.size()]));
      if (!constructor.isAccessible()) {
        constructor.setAccessible(true);
      }
      return constructor.newInstance(constructorArgs.toArray(new Object[constructorArgs.size()]));
    } catch (Exception e) {
      StringBuilder argTypes = new StringBuilder();
      if (constructorArgTypes != null && !constructorArgTypes.isEmpty()) {
        for (Class<?> argType : constructorArgTypes) {
          argTypes.append(argType.getSimpleName());
          argTypes.append(",");
        }
        argTypes.deleteCharAt(argTypes.length() - 1); // remove trailing ,
      }
      StringBuilder argValues = new StringBuilder();
      if (constructorArgs != null && !constructorArgs.isEmpty()) {
        for (Object argValue : constructorArgs) {
          argValues.append(String.valueOf(argValue));
          argValues.append(",");
        }
        argValues.deleteCharAt(argValues.length() - 1); // remove trailing ,
      }
      throw new ReflectionException("Error instantiating " + type + " with invalid types (" + argTypes + ") or values (" + argValues + "). Cause: " + e, e);
    }
  }

  protected Class<?> resolveInterface(Class<?> type) {
    Class<?> classToCreate;
    if (type == List.class || type == Collection.class || type == Iterable.class) {
      classToCreate = ArrayList.class;
    } else if (type == Map.class) {
      classToCreate = HashMap.class;
    } else if (type == SortedSet.class) { // issue #510 Collections Support
      classToCreate = TreeSet.class;
    } else if (type == Set.class) {
      classToCreate = HashSet.class;
    } else {
      classToCreate = type;
    }
    return classToCreate;
  }

  @Override
  public <T> boolean isCollection(Class<T> type) {
    return Collection.class.isAssignableFrom(type);
  }

}
```

## objectWrapperFactory

```java
private void objectWrapperFactoryElement(XNode context) throws Exception {
  if (context != null) {
    // type类的类型，实例化这个类
    String type = context.getStringAttribute("type");
    ObjectWrapperFactory factory = (ObjectWrapperFactory) resolveClass(type).newInstance();
    configuration.setObjectWrapperFactory(factory);
  }
}
```
- 用于包装Object实例。默认是DefaultObjectWrapperFactory类。

## reflectionFactory
```java
private void reflectionFactoryElement(XNode context) throws Exception {
  if (context != null) {
      String type = context.getStringAttribute("type");
      ReflectorFactory factory = (ReflectorFactory) resolveClass(type).newInstance();
      configuration.setReflectorFactory(factory);
  }
}
```
ReflectorFactory的默认实现：DefaultReflectorFactory。
```java
// 使用ConcurrentMap存储了以Class对象为key，Reflector为value.
private final ConcurrentMap<Class<?>, Reflector> reflectorMap = new ConcurrentHashMap<Class<?>, Reflector>();
```
- 也就是说ReflectorFactory是Reflector的工厂，需要哪个就从工厂里面获取。
- 下面看下Reflector的实现：
```java


```

## settings
全局设置
```java
private void settingsElement(Properties props) throws Exception {
  // 设置autoMappingBehavior，默认PARTIAL。有3种，NONE：表示取消自动映射；PARTIAL：只会自动映射没有定义嵌套结果集映射的结果集；FULL：会自动映射任意复杂的结果集（无论是否嵌套）。
  configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
  // 设置自动mapping检测到未知列时的行为。默认NONE：设么也不做；WARNING:打warn日志；FAILING:抛出SqlSessionException；
  configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
  // 设置是否缓存。默认为true.
  configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
  // 设置proxyFactory代理工厂。默认是JavassistProxyFactory
  configuration.setProxyFactory((ProxyFactory) createInstance(props.getProperty("proxyFactory")));
  // 设置是否延迟加载。默认flase.
  configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), false));
  // 设置aggressiveLazyLoading，默认为true，即不按需加载。lazyLoadingEnabled属性启用时只要加载对象，就会加载该对象的所有属性；设置为false则会按需加载，即使用到某关联属性时，实时执行嵌套查询加载该属性。
  configuration.setAggressiveLazyLoading(booleanValueOf(props.getProperty("aggressiveLazyLoading"), true));
  // 设置multipleResultSetsEnabled，默认为true.允许/不允许一条SQL返回多个结果集.
  configuration.setMultipleResultSetsEnabled(booleanValueOf(props.getProperty("multipleResultSetsEnabled"), true));
  // 设置useColumnLabel，默认为true。列标签代替列名，即sql里的字段别名。设置成false，别名映射会不生效。
  configuration.setUseColumnLabel(booleanValueOf(props.getProperty("useColumnLabel"), true));
  // 设置useGeneratedKeys，默认false.在执行添加记录之后可以获取到数据库自动生成的主键ID。在settings元素中设置useGeneratedKeys是一个全局参数，但是只会对接口映射器产生影响，对xml映射器不起效。所以一般我们会在xml映射器中增加useGeneratedKeys。
  configuration.setUseGeneratedKeys(booleanValueOf(props.getProperty("useGeneratedKeys"), false));
  // 设置defaultExecutorType.配置默认的执行器。SIMPLE 就是普通的执行器；REUSE 执行器会重用预处理语句（prepared statements）； BATCH 执行器将重用语句并执行批量更新。
  configuration.setDefaultExecutorType(ExecutorType.valueOf(props.getProperty("defaultExecutorType", "SIMPLE")));
  // 设置defaultStatementTimeout，默认为null。
  configuration.setDefaultStatementTimeout(integerValueOf(props.getProperty("defaultStatementTimeout"), null));
  // 设置defaultFetchSize。查询每次返回的数据条数，可以在select参数中使用fetchSize覆盖，mysql不支持这个参数。
  configuration.setDefaultFetchSize(integerValueOf(props.getProperty("defaultFetchSize"), null));
  // 设置是否开启下划线转驼峰命名规则，默认false。
  configuration.setMapUnderscoreToCamelCase(booleanValueOf(props.getProperty("mapUnderscoreToCamelCase"), false));
  // 设置是否允许在嵌套语句中使用分页，默认false。
  configuration.setSafeRowBoundsEnabled(booleanValueOf(props.getProperty("safeRowBoundsEnabled"), false));
  // MyBatis 利用本地缓存机制（Local Cache）防止循环引用（circular references）和加速重复嵌套查询。 默认值为 SESSION，这种情况下会缓存一个会话中执行的所有查询。 若设置值为 STATEMENT，本地会话仅用在语句执行上，对相同 SqlSession 的不同调用将不会共享数据。
  configuration.setLocalCacheScope(LocalCacheScope.valueOf(props.getProperty("localCacheScope", "SESSION")));
  // 当没有为参数提供特定的 JDBC 类型时,不设置的都默认为OTHER。
  configuration.setJdbcTypeForNull(JdbcType.valueOf(props.getProperty("jdbcTypeForNull", "OTHER")));
  // 设置懒加载触发方法：equals,clone,hascode,toString
  configuration.setLazyLoadTriggerMethods(stringSetValueOf(props.getProperty("lazyLoadTriggerMethods"), "equals,clone,hashCode,toString"));
  // 设置resultHandler检查。
  configuration.setSafeResultHandlerEnabled(booleanValueOf(props.getProperty("safeResultHandlerEnabled"), true));
  // 指定动态SQL生成的默认语言。默认XMLLanguageDriver。
  configuration.setDefaultScriptingLanguage(resolveClass(props.getProperty("defaultScriptingLanguage")));
  // 指定当结果集中值为null的时候是否调用映射对象的setter（map 对象时为 put）方法，这对于有 Map.keySet() 依赖或 null 值初始化的时候是有用的。注意基本类型（int、boolean等）是不能设置成 null 的。
  configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
  // 指定MyBatis增加到日志名称的前缀。
  configuration.setLogPrefix(props.getProperty("logPrefix"));
  // 指定MyBatis所用日志的具体实现，未指定时将自动查找。
  configuration.setLogImpl(resolveClass(props.getProperty("logImpl")));
  // 设置configurationFactory。
  configuration.setConfigurationFactory(resolveClass(props.getProperty("configurationFactory")));
}
```

## environments
设置多个环境配置,默认环境使用default指定。
```java
private void environmentsElement(XNode context) throws Exception {
  if (context != null) {
    if (environment == null) {
      environment = context.getStringAttribute("default");
    }
    for (XNode child : context.getChildren()) {
      String id = child.getStringAttribute("id");
      if (isSpecifiedEnvironment(id)) {
        TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
        DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
        DataSource dataSource = dsFactory.getDataSource();
        Environment.Builder environmentBuilder = new Environment.Builder(id)
            .transactionFactory(txFactory)
            .dataSource(dataSource);
        configuration.setEnvironment(environmentBuilder.build());
      }
    }
  }
}
```

## databaseIdProvider
设置根据不同的数据库厂商执行不同的语句。
```java
private void databaseIdProviderElement(XNode context) throws Exception {
  DatabaseIdProvider databaseIdProvider = null;
  if (context != null) {
    String type = context.getStringAttribute("type");
    // awful patch to keep backward compatibility
    if ("VENDOR".equals(type)) {
        type = "DB_VENDOR";
    }
    Properties properties = context.getChildrenAsProperties();
    databaseIdProvider = (DatabaseIdProvider) resolveClass(type).newInstance();
    databaseIdProvider.setProperties(properties);
  }
  Environment environment = configuration.getEnvironment();
  if (environment != null && databaseIdProvider != null) {
    String databaseId = databaseIdProvider.getDatabaseId(environment.getDataSource());
    configuration.setDatabaseId(databaseId);
  }
}
```

## typeHandlers
配置一些自定义的类型处理器，查询数据库完成后，调用TypeHandler的方法读取数据转换成java对象
```java
private void typeHandlerElement(XNode parent) throws Exception {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      if ("package".equals(child.getName())) {
        String typeHandlerPackage = child.getStringAttribute("name");
        typeHandlerRegistry.register(typeHandlerPackage);
      } else {
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

## mappers
配置xml映射，即配置在哪找到sql语句。
```java
private void mapperElement(XNode parent) throws Exception {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      // 包下的所有文件
      if ("package".equals(child.getName())) {
        String mapperPackage = child.getStringAttribute("name");
        configuration.addMappers(mapperPackage);
      } else {
        // 一个一个文件配置
        String resource = child.getStringAttribute("resource");
        String url = child.getStringAttribute("url");
        String mapperClass = child.getStringAttribute("class");
        if (resource != null && url == null && mapperClass == null) {
          ErrorContext.instance().resource(resource);
          InputStream inputStream = Resources.getResourceAsStream(resource);
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
          mapperParser.parse();
        } else if (resource == null && url != null && mapperClass == null) {
          ErrorContext.instance().resource(url);
          InputStream inputStream = Resources.getUrlAsStream(url);
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
          mapperParser.parse();
        } else if (resource == null && url == null && mapperClass != null) {
          Class<?> mapperInterface = Resources.classForName(mapperClass);
          configuration.addMapper(mapperInterface);
        } else {
          throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
        }
      }
    }
  }
}
```
- 一般都用包来配置

# 总结
- properties节点首先读取子节点property属性，然后从由resource或者url标记的地方获取属性，有相同key的属性会覆盖之前读取的值。