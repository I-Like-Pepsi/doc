# Mybatis-Plus实现单表CRUD的原理

## 导读

现在使用Mybatis-plus的人越来越多，基本上对于单表的增删改查是不需要写代码即可完成。使用的时候我们只需要在原来的Mapper接口文件上继承BaseMapper即可自动完成单表的CRUD。之前一直都是使用，并没有想过是如何实现的，通过一下午时间大致明白了Mybatis-plus是如何实现通用的单表CRUD了。

## 示例

在讲原理之前我们先来看简单的看一下如何Mybatis如何实现一个查询，示例代码如下：

```java
@Slf4j
public class MybatisApp {
    public static void main(String[] args) throws IOException {
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(inputStream);

        try (SqlSession session = factory.openSession()) {
            UserEntity user = (UserEntity) session.selectOne("com.buydeem.mapper.UserMapper.getById",1);
            log.info("{}",user);
        }
    }
}
```

- mybatis-config.xml配置文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "https://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <properties resource="jdbc.properties">
    </properties>
    <settings>
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>
    <typeAliases>
        <package name="com.buydeem.entity"/>
    </typeAliases>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="mapper/UserMapper.xml"/>
    </mappers>

</configuration>
```

- UserMapper.xml文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "https://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.buydeem.mapper.UserMapper">

    <select id="getById" resultType="UserEntity">
        select * from t_user where id = #{id}
    </select>
</mapper>
```

示例代码很简单，首先通过读取配置文件创建一个`SqlSessionFactory`，然后获取`SqlSession`执行查询。

## Configuration

通过前面的简单示例，貌似还看不出来什么原因。先看```SqlSession.selectOne()```实现，跟踪其源码，最后的实现如下：

```java
  private <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.query(ms, wrapCollection(parameter), rowBounds, handler);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

该方法从```Configuration```中获取一个```MappedStatement```实例，然后将其交给```Executor```去执行。

首先我们得分析一下这个```Configuration```是个什么东西。在之前我们的示例代码中，它实际上就是读取```mybatis-config.xml```配置文件构建出一个配置类。从```Configuration```的源码中也有体现，这个配置类中的属性与```mybatis-config.xml```能够对应起来。

```java
  MappedStatement ms = configuration.getMappedStatement(statement
```

该代码是从```Configuration```实例中获取```MappedStatement```实例，而其内部实现如下：

```java
  public MappedStatement getMappedStatement(String id, boolean validateIncompleteStatements) {
    if (validateIncompleteStatements) {
      buildAllStatements();
    }
    return mappedStatements.get(id);
  }
```

其实就是从```Configuration```中的属性```mappedStatements```获取```MappedStatement```实例。

```java
  protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection")
      .conflictMessageProducer((savedValue, targetValue) ->
          ". please check " + savedValue.getResource() + " and " + targetValue.getResource());
```

可以看见```Configuration```中的```mappedStatements```类型是```StrictMap```,它继承了HashMap并重写了put和get方法。

```java
    public V put(String key, V value) {
      if (containsKey(key)) {
        throw new IllegalArgumentException(name + " already contains value for " + key
            + (conflictMessageProducer == null ? "" : conflictMessageProducer.apply(super.get(key), value)));
      }
      if (key.contains(".")) {
        final String shortKey = getShortName(key);
        if (super.get(shortKey) == null) {
          super.put(shortKey, value);
        } else {
          super.put(shortKey, (V) new Ambiguity(shortKey));
        }
      }
      return super.put(key, value);
    } 


    public V get(Object key) {
      V value = super.get(key);
      if (value == null) {
        throw new IllegalArgumentException(name + " does not contain value for " + key);
      }
      if (value instanceof Ambiguity) {
        throw new IllegalArgumentException(((Ambiguity) value).getSubject() + " is ambiguous in " + name
            + " (try using the full name including the namespace, or rename one of the entries)");
      }
      return value;
    }
```

## MappedStatement

通过前面我们了解到```Configuration```内部用一个Map来存储```MappedStatement```，现在的问题是它是何时被放进去的。

```java
  public void addMappedStatement(MappedStatement ms) {
    mappedStatements.put(ms.getId(), ms);
  }
```

在```Configuration```中只提供了一个方法用来添加```MappedStatement```，同时还可以知道一点`Configuration`中存储```MappedStatement```的Map它的key就是```MappedStatement```的ID属性。而调用```Configuration.addMappedStatement()```方法的也只有```MapperBuilderAssistant```了，该类通过名称可以看出来就是用来构建Mapper的工具类。而我们创建的Mapper.xml文件中的sql语句其实就是通过解析xml文件，然后通过该工具类将其转换成```MappedStatement```存储到```Configuration```实例中。关于如何解析xml相关内容本文就不多说了，有兴趣的可以自己去看源码。

现在的疑问是这个```MappedStatement```有什么用？

```java
public final class MappedStatement {

  private String resource;
  private Configuration configuration;
  private String id;
  private Integer fetchSize;
  private Integer timeout;
  private StatementType statementType;
  private ResultSetType resultSetType;
  private SqlSource sqlSource;
  private Cache cache;
  private ParameterMap parameterMap;
  private List<ResultMap> resultMaps;
  private boolean flushCacheRequired;
  private boolean useCache;
  private boolean resultOrdered;
  private SqlCommandType sqlCommandType;
  private KeyGenerator keyGenerator;
  private String[] keyProperties;
  private String[] keyColumns;
  private boolean hasNestedResultMaps;
  private String databaseId;
  private Log statementLog;
  private LanguageDriver lang;
  private String[] resultSets;
}
```

其实每个```MappedStatement```对应了我们自定义Mapper接口中的一个方法，它保存了我们定义SQL语句、参数结构、返回值结构、Mybatis对它的处理方式等等配置，其内容对应的就是我们Mapper.xml中定义的内容。

<img title="" src="http://img.bakelu.cn/blog/MappedStatement%E7%BB%93%E6%9E%84.png" alt="" data-align="center">

## 中途小结

从前面我们了解的内容我们现在知道了下面几点：

- Mybatis首先会构建一个`Configuration`实例。

- `Configuration`中通过一个```mappedStatements```属性存储```MappedStatement```实例，其类型就是一个Map，Map的key为MappedStatement的ID，value为MappedStatement实例。

- Mapper.xml文件中的定义的每个增删改查被解析成```MappedStatement```实例并被保存在`Configuration`中。

到目前我们知道了，如果要让Mybatis执行一个SQL语句，我们必须将其转化成```MappedStatement```才能执行。

## Mapper接口

而实际我们使用Mybatis的时候，我们更多的使用下面这种方式：

```java
   try (SqlSession session = factory.openSession()) {
            UserMapper mapper = session.getMapper(UserMapper.class);
            UserEntity user = mapper.getById(1);
            log.info("{}",user);
        }
```

我们通过一个Mapper接口，在接口中定义SQL执行方法，那这又是如何实现的呢？

### MapperRegistry

跟踪```SqlSession.getMapper```方法其源码是通过调用```MapperRegistry```获取到一个代理类。首先我们明白一个道理，接口是无法实例化的，但是我们上面的示例代码中我们确实创建出了一个```UserMapper```实例，这又是为什么呢？

在Java中我们可以通过JDK动态代理生成一个代理类，而我们之所以能够使用UserMapper实例就是因为Mybatis给我们创建了代理对象。关于代理可以看下面的示例：

```java
public interface Animal {
    void say();
} 


public class ProxyDemo {
    public static void main(String[] args) {
        InvocationHandler dogProxy = new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println("wang wang wang");
                return null;
            }
        };
        Animal dog = (Animal) Proxy.newProxyInstance(ProxyDemo.class.getClassLoader(), new Class[]{Animal.class}, dogProxy);
        dog.say();
        System.out.println(dog.getClass().getName());
    }
}
```

这就是一个简单的JDK动态代理生成实例的代码，Animal接口就是通过动态代理来生成一个代理的Dog实例。

```java
  // MapperRegistry.getMapper()方法实现
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

`MapperRegistry.getMapper()`方法的主要逻辑就是获取接口的代理工厂类，然后通过代理工厂类创建代理类的实例对象。

```java
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethodInvoker> methodCache = new ConcurrentHashMap<>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethodInvoker> getMethodCache() {
    return methodCache;
  }

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) { 
    //调用JDK动态代理创建代理类
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}
```

从源码可以看出代理工厂就是通过JDK动态代理来创建接口实例的，同时代理工厂类为了提高性能还使用Map来缓存创建的实例对象。而MapperProxy其实就是```InvocationHandler```的接口实现，这点可以从源码看出。

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
 //省略内部代码   
}
```

我们重点看一下`invoke`方法内部实现：

```java
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else {
        return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }
```

第一个判断是用来处理Object对象的，这个里面什么都不用管直接使用其原来的逻辑。第二个分支则是通过缓存获取其实现。

```java
  private MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
    try {
      return MapUtil.computeIfAbsent(methodCache, method, m -> {
        if (m.isDefault()) {
          try {
            if (privateLookupInMethod == null) {
              return new DefaultMethodInvoker(getMethodHandleJava8(method));
            } else {
              return new DefaultMethodInvoker(getMethodHandleJava9(method));
            }
          } catch (IllegalAccessException | InstantiationException | InvocationTargetException
              | NoSuchMethodException e) {
            throw new RuntimeException(e);
          }
        } else {
          return new PlainMethodInvoker(new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
        }
      });
    } catch (RuntimeException re) {
      Throwable cause = re.getCause();
      throw cause == null ? re : cause;
    }
  }
```

上面方法返回的```MapperMethodInvoker```有两种，其中```DefaultMethodInvoker```是用来处理Java8及以上JDK中接口默认方法的，这个我们不用管，我们要关心的是```PlainMethodInvoker```。该方法内部保存了一个```MapperMethod```属性，最后invoke方法实际上执行的逻辑是：

```java
    public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
      return mapperMethod.execute(sqlSession, args);
    }
```

而方法该方法最后执行的逻辑还是回到了最初```sqlSession.selectOne()```方法。

通过该部分内容我们知道了下面几点：

- Mybatis会为我们的Mapper接口生成代理实例。

- 代理实例调用调用的方法最后还是会调用sqlSession.selectOne()方法来实现。

## Mybatis-plus简单实用

```java
@Slf4j
public class App2 {
    public static void main(String[] args) throws IOException {
        String resource = "mybatis-plus-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory  = new MybatisSqlSessionFactoryBuilder().build(inputStream);

        try (SqlSession sqlSession = sqlSessionFactory.openSession()){
            User2Mapper mapper = sqlSession.getMapper(User2Mapper.class);
            UserEntity user = mapper.selectById(1);
            log.info("{}",user);
        }
    }
}
```

使用Mybatis-plus与Mybatis类似，但是可以发现其创建```SqlSessionFactory```使用的是```MybatisSqlSessionFactoryBuilder```，难道就这点不一样么？其实不是，虽然都能用来创建```SqlSessionFactory```，但是其内存创建```MybatisConfiguration```会有差别。

## MybatisConfiguration

该类是Mybatis-plus通过继承Configuration的一个子类，该类代码与Configuration基本相似，而增强Mybatis就是从此处开始的。在前面我们介绍了```MapperRegistry```，而在```MybatisConfiguration```中同样存在一个```MybatisMapperRegistry```，它的作用与```MapperRegistry```一样，但是其有一个方法需要特别关注。

```java
    public <T> void addMapper(Class<T> type) {
        if (type.isInterface()) {
            if (hasMapper(type)) {
                // TODO 如果之前注入 直接返回
                return;
                // TODO 这里就不抛异常了
//                throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
            }
            boolean loadCompleted = false;
            try {
                // TODO 这里也换成 MybatisMapperProxyFactory 而不是 MapperProxyFactory
                knownMappers.put(type, new MybatisMapperProxyFactory<>(type));
                // It's important that the type is added before the parser is run
                // otherwise the binding may automatically be attempted by the
                // mapper parser. If the type is already known, it won't try.
                // TODO 这里也换成 MybatisMapperAnnotationBuilder 而不是 MapperAnnotationBuilder
                MybatisMapperAnnotationBuilder parser = new MybatisMapperAnnotationBuilder(config, type);
                parser.parse();
                loadCompleted = true;
            } finally {
                if (!loadCompleted) {
                    knownMappers.remove(type);
                }
            }
        }
    }
```

该方法与Mybatis源码最大的不同点就在此处了。此处不是使用的原有```MapperAnnotationBuilder```而是使用的自己的```MybatisMapperAnnotationBuilder```，而这个类的parse方法就是增强的关键点所在。

```java
            try {
                // https://github.com/baomidou/mybatis-plus/issues/3038
                if (GlobalConfigUtils.isSupperMapperChildren(configuration, type)) {
                    parserInjector();
                }
            } catch (IncompleteElementException e) {
                configuration.addIncompleteMethod(new InjectorResolver(this));
            }
```

parse方法中有一个判断，判断该接口是不是BaseMapper的子类，如果是就进行CRUD SQL注入。

```java
    public void inspectInject(MapperBuilderAssistant builderAssistant, Class<?> mapperClass) {
        Class<?> modelClass = ReflectionKit.getSuperClassGenericType(mapperClass, Mapper.class, 0);
        if (modelClass != null) {
            String className = mapperClass.toString();
            Set<String> mapperRegistryCache = GlobalConfigUtils.getMapperRegistryCache(builderAssistant.getConfiguration());
            if (!mapperRegistryCache.contains(className)) {
                TableInfo tableInfo = TableInfoHelper.initTableInfo(builderAssistant, modelClass);
                List<AbstractMethod> methodList = this.getMethodList(mapperClass, tableInfo);
                if (CollectionUtils.isNotEmpty(methodList)) {
                    // 循环注入自定义方法
                    methodList.forEach(m -> m.inject(builderAssistant, mapperClass, modelClass, tableInfo));
                } else {
                    logger.debug(mapperClass.toString() + ", No effective injection method was found.");
                }
                mapperRegistryCache.add(className);
            }
        }
    }
```

该方法就是增强SQL的核心代码，该通过model实体类获取到数据库表的信息也就是TableInfo，然后通过获取到BaseMapper接口中的增强方法，然后通过MapperBuilderAssistant将其解析成MappedStatement并添加到Configuration中的```mappedStatement```中，最后完成了CURD方法增强。

注意这里面用到了一个模板方法的设计模式，对于AbstractMethod其实现了大多数逻辑，而且这些逻辑都是相同的，唯一不同的就是就是注入自定义的MappedStatement，该类也只有这个方法需要子类去继承实现。

## 总结

其实Mybatis-plus的单表CRUD就是通过自动添加MappedStatement去实现的。如果我们自己去写单表的CURD也就是将自己的SQL执行方法写到XML文件中，然后Mybatis再将我们写的XML文件解析成MappedStatement保存到Configuration已给后续使用。而Mybatis-plus通过Java的反射获取到表信息，然后动态的生成MappedStatement保存到Configuration中。

最后看了Mybatis-plus的增强实现感觉Mybatis很多东西都不能很好的扩展，Mybatis-plus有很多代码都是复制的Mybatis源代码然后增加自己的逻辑来实现的。这个主要问题还是因为Mybatis的扩展性的确不如Spring。

最后说一句，Spring的设计确实比Mybatis好很多。
