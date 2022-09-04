# Spring中的父子容器

## 背景

在很长的一段时间里面，关于Spring父子容器这个问题我一直没太关注，但是上次同事碰见一个奇怪的bug于是我决定重新了解一下Spring中的父子容器。

项目是一个老的SSM项目，同事在使用AOP对Controller层的方法进行拦截做token验证。这个功能在实际的开发项目中很常见对吧，估计大家都能轻易解决。但是问题就处在了AOP上面，根据AOP失效的八股文全部排查了一遍，问题还是没有解决。但是神奇的问题出现了，我尝试把切点放在Service中的方法AOP生效了。然后我看了下配置文件，发现了问题所在。

- **root-context.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/aop https://www.springframework.org/schema/aop/spring-aop.xsd">

    <context:component-scan base-package="com.buydeem.container">
        <context:exclude-filter type="regex" expression="com.buydeem.container.controller.*"/>
    </context:component-scan>

    <aop:aspectj-autoproxy/>

</beans>
```

- **mvc-context.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/aop https://www.springframework.org/schema/aop/spring-aop.xsd">

    <context:component-scan base-package="com.buydeem.container">
        <context:exclude-filter type="regex" expression="com.buydeem.container.controller.*"/>
    </context:component-scan>

</beans>
```

- **TokenAspect**

```java
@Component
@Aspect
@Slf4j
public class TokenAspect {

    @Pointcut("execution (public * com.buydeem.container.controller..*.*(..))")
    //@Pointcut("execution (public * com.buydeem.container.service..*.*(..))")
    public void point(){

    }

    @Before("point()")
    public void before(){
      log.info("before");
    }

}
```

其实问题所在就是父子容器造成的，现在我们使用的SpringBoot中基本上不会出现问题，默认情况下SpringBoot中只会有一个容器，而在传统的SSM架构中我们很可能会有两个容器。在传统的SSM架构中，我们会创建两个配置文件，一个用来创建Controller层的容器通常是子容器，而Service和Dao层的容器通常就是父容器。

## 父子容器相关接口

在IOC容器时，Spring中通常会提到两个顶级接口BeanFactory和ApplicationContext，这两个都是IOC容器接口，相比BeanFactory而言，ApplicationContext提供了更强大的功能。

### HierarchicalBeanFactory

该接口作为BeanFactory的子接口，它的定义如下：

```java
public interface HierarchicalBeanFactory extends BeanFactory {

    BeanFactory getParentBeanFactory();

    boolean containsLocalBean(String name);

}
```

从它名称可以看出，它是一个有层级的BeanFactory，它提供的两个方法其中一个就是用来获取父容器的。

### ConfigurableBeanFactory

上面说了HierarchicalBeanFactory提供了获取父容器的方法，那么父容器是怎么设置的呢？而设置父容器的方法则被定义在ConfigurableBeanFactory接口中。从名字可以看出它是一个可配置的BeanFactory，设置父容器的方法定义如下：

```java
void setParentBeanFactory(BeanFactory parentBeanFactory) throws IllegalStateException;
```

### ApplicationContext

上面讲了BeanFactory中获取和设置父容器相关接口和方法，而ApplicationContext中同样提供了一个方法用来获取父容器。

```java
ApplicationContext getParent();
```

### ConfigurableApplicationContext

与BeanFactory中设置父容器一样，ConfigurableApplicationContext提供了一个用来设置父容器的方法。

```java
void setParent(@Nullable ApplicationContext parent);
```

## 特性

通过上面介绍我们明白了什么是父子容器，那么它有哪些特性呢？使用时需要注意什么呢？

示例代码如下：

- 父容器配置

```java
@Component
public class ParentService {
}


@Configuration
public class ParentContainerConfig {

    @Bean
    public ParentService parentService(){
        return new ParentService();
    }
}
```

- 子容器配置

```java
@Component
public class ChildService {
}

@Configuration
public class ChildContainerConfig {

    @Bean
    public ChildService childService(){
        return new ChildService();
    }
}
```

### 子容器能获取到父容器中的Bean

```java
@Slf4j
public class App {
    public static void main(String[] args) {
        //父容器
        ApplicationContext parentContainer = new AnnotationConfigApplicationContext(ParentContainerConfig.class);
        //子容器
        ConfigurableApplicationContext childContainer = new AnnotationConfigApplicationContext(ChildContainerConfig.class);
        childContainer.setParent(parentContainer);
        //从子容器中获取父容器中的Bean
        ParentService parentService = childContainer.getBean(ParentService.class);
        log.info("{}",parentService);
        //getBeansOfType无法获取到父容器中的Bean
        Map<String, ParentService> map = childContainer.getBeansOfType(ParentService.class);
        map.forEach((k,v) -> log.info("{} => {}",k,v));
    }
}
```

**ParentService**是父容器中的Bean，但是我们在子容器中却能获取到，这说明在子容器中是可以获取到父容器中的Bean的，但是并不是所有方法都能，所以在使用时我们需要注意。这也解释了一个问题，那就是在SSM架构中为什么我们能在Controller中获取到Service，如果不是这个特性那我们的肯定是不行的。

### 父容器不能获取子容器中的Bean

子容器能获取到父容器中的Bean，但是父容器却不能获取到子容器中的Bean。

```java
@Slf4j
public class App {
    public static void main(String[] args) {
        //父容器
        ApplicationContext parentContainer = new AnnotationConfigApplicationContext(ParentContainerConfig.class);
        //子容器
        ConfigurableApplicationContext childContainer = new AnnotationConfigApplicationContext(ChildContainerConfig.class);
        childContainer.setParent(parentContainer);

        try {
            ChildService childService = parentContainer.getBean(ChildService.class);
            log.info("{}",childService);
        }catch (NoSuchBeanDefinitionException e){
            log.error(e.getMessage());
        }
    }
}
```

上面的代码运行时会抛出异常，因为父容器是无法获取到子容器中的Bean的。

## SSM中的父子容器

回到我们最初的问题，在SSM中存在这两个容器，这也是导致我们前面AOP失败的原因。那么SSM中的父子容器是如何被创建和设置的呢？

### web.xml

首先要解答这个问题我们需要先来看一下**web.xml**中的配置。

```xml
<?xml version="1.0" encoding="UTF-8"?>

<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
  <context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/root-context.xml</param-value>
  </context-param>

  <servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>/WEB-INF/mvc-context.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
  </servlet>

  <servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>

</web-app>
```

通常这个配置如上所示，我们需要关注的就两分别为

**ContextLoaderListener**和**DispatcherServlet**。

### 父容器创建

其中**ContextLoaderListener**就是Servlet中的监听器，当Servlet容器启动时就会调用```contextInitialized()```方法进行初始化，该方法的实现如下：

```java
    @Override
    public void contextInitialized(ServletContextEvent event) {
        initWebApplicationContext(event.getServletContext());
    }
```

而```initWebApplicationConte()```的实现则在**ContextLoader**这个类中，该方法的实现如下：

```java
    public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
        if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
            throw new IllegalStateException(
                    "Cannot initialize context because there is already a root application context present - " +
                            "check whether you have multiple ContextLoader* definitions in your web.xml!");
        }

        servletContext.log("Initializing Spring root WebApplicationContext");
        Log logger = LogFactory.getLog(ContextLoader.class);
        if (logger.isInfoEnabled()) {
            logger.info("Root WebApplicationContext: initialization started");
        }
        long startTime = System.currentTimeMillis();

        try {
            // Store context in local instance variable, to guarantee that
            // it is available on ServletContext shutdown.
            if (this.context == null) {
                //创建WebApplicationContext容器
                this.context = createWebApplicationContext(servletContext);
            }
            if (this.context instanceof ConfigurableWebApplicationContext) {
                ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
                if (!cwac.isActive()) {
                    // The context has not yet been refreshed -> provide services such as
                    // setting the parent context, setting the application context id, etc
                    if (cwac.getParent() == null) {
                        // The context instance was injected without an explicit parent ->
                        // determine parent for root web application context, if any.
                        ApplicationContext parent = loadParentContext(servletContext);
                        cwac.setParent(parent);
                    }
                    //配置并刷新WebApplicationContext
                    configureAndRefreshWebApplicationContext(cwac, servletContext);
                }
            }
            //将WebApplicationContext的引用保存到servletContext中（后面会用到）
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

            ClassLoader ccl = Thread.currentThread().getContextClassLoader();
            if (ccl == ContextLoader.class.getClassLoader()) {
                currentContext = this.context;
            }
            else if (ccl != null) {
                currentContextPerThread.put(ccl, this.context);
            }

            if (logger.isInfoEnabled()) {
                long elapsedTime = System.currentTimeMillis() - startTime;
                logger.info("Root WebApplicationContext initialized in " + elapsedTime + " ms");
            }

            return this.context;
        }
        catch (RuntimeException | Error ex) {
            logger.error("Context initialization failed", ex);
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
            throw ex;
        }
    }
```

虽然方法较长，但实际上我们需要关注的就三点：

- 创建容器

- 配置并刷新容器

- 将容器设置到servletContext中

### 子容器创建

子容器的创建我们需要关注的就是web.xml中```DispatcherServlet```配置了，```DispatcherServlet```说白了就是一个Servlet，当Servlet容器在实例化Servlet后就会调用其```init()```方法就行初始化，而`DispatcherServlet`的继承如下图所示：

![DispatcherServlet](https://files.mdnice.com/user/14227/5dcf58a8-eb51-43e2-8e16-6c953dd4e771.png)

而```init()```方法的实现则是在**HttpServletBean**中，方法定义如下：

```java
    public final void init() throws ServletException {

        // Set bean properties from init parameters.
        PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
        if (!pvs.isEmpty()) {
            try {
                BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
                ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
                bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
                initBeanWrapper(bw);
                bw.setPropertyValues(pvs, true);
            }
            catch (BeansException ex) {
                if (logger.isErrorEnabled()) {
                    logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
                }
                throw ex;
            }
        }

        // Let subclasses do whatever initialization they like.
        initServletBean();
    }
```

从实现上可以看出并没有子容器相关代码，但是它留了一个方法，用来让子类扩展实现自己的初始化。而该方法的实现则是在**FrameworkServlet**中实现的，源码如下：

```java
    protected final void initServletBean() throws ServletException {
        getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
        if (logger.isInfoEnabled()) {
            logger.info("Initializing Servlet '" + getServletName() + "'");
        }
        long startTime = System.currentTimeMillis();

        try {
            this.webApplicationContext = initWebApplicationContext();
            initFrameworkServlet();
        }
        catch (ServletException | RuntimeException ex) {
            logger.error("Context initialization failed", ex);
            throw ex;
        }

        if (logger.isDebugEnabled()) {
            String value = this.enableLoggingRequestDetails ?
                    "shown which may lead to unsafe logging of potentially sensitive data" :
                    "masked to prevent unsafe logging of potentially sensitive data";
            logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
                    "': request parameters and headers will be " + value);
        }

        if (logger.isInfoEnabled()) {
            logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
        }
    }
```

而实际创建子容器的实现则是在```initWebApplicationContext()```方法中实现的，该方法会创建子容器，并将先前创建的父容器从servletContext中取出来设置为子容器的父容器。

### 验证

```java
@Component
public class HelloService {

    @Autowired
    private ApplicationContext context;

    public String sayHello(){
        return "Hello World";
    }

    public ApplicationContext getContext(){
        return context;
    }
}

@RestController
@Slf4j
public class HelloWorldController {

    @Autowired
    private HelloService helloService;
    @Autowired
    private ApplicationContext context;


    @GetMapping("hello")
    public String helloWorld(){
        //获取Service中的容器
        ApplicationContext parentContext = helloService.getContext();
        //service中的容器并不等于controller中的容器
        log.info("parentContext == context ? {}",parentContext == context);
        //controller中的容器的父容器就是service中的容器
        log.info("{}",parentContext == context.getParent());
        return helloService.sayHello();
    }
}
```

上面代码中我们分别在HelloService和HelloWorldController中分别注入ApplicationContext，执行程序最后的打印结果如下：

```
14:45:23.443 [http-nio-8080-exec-2] INFO  c.b.c.c.HelloWorldController - parentContext == context ? false
14:45:23.451 [http-nio-8080-exec-2] INFO  c.b.c.c.HelloWorldController - true
```

从上面的打印结果可以看出HelloService和HelloWorldController中的容器并不是同一个。

## 解决办法

回到我们最初的问题，我们现在知道了AOP失效的原因是因为父子容器导致的，因为我们只在父容器中开启了@AspectJ支持，在子容器中我们并没有开启。

#### 只使用一个容器

既然问题是由父子容器导致的，那我们将controller层也交给父容器管理那是不是就可以了。实际上是没有问题的，但是并不推荐这么做。

### 开启子容器@AspectJ支持

在子容器的配置文件中增加如下配置：

```xml
<aop:aspectj-autoproxy/>
```

## 总结

对于Spring中父子容器的内容就讲到这里了，后续如果还有新的发现会继续更新相关内容。文中示例代码地址：
