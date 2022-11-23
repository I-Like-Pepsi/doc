# 如何构建一个SpringBootStarter

在使用SpringBoot的时候我们经常会用到各种Starter，例如我们在SpringBoot中使用Mybatis，我们只需要引入以下依赖：

```xml
       <dependency>
           <groupId>org.mybatis.spring.boot</groupId>
           <artifactId>mybatis-spring-boot-starter</artifactId>
           <version>1.3.2</version>
       </dependency>
```

然后我们只需要简单的引入少许配置即可使用Mybatis了。

## 如何构建

平时我们使用的都是官方提供的Starter，那么我们如何构建一个自己的Starter呢？我们使用简单的例子来说明。例如我们提供一个Starter，它的核心功能就是获取一个用户，这个用户我们可以自己配置。

### 引入必要的依赖

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <!--添加了该模块以来的模块，再配置yml时候会有本模块配置属性的提示信息-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
```

上面是我们项目的依赖，实际开发时可能还需要依赖别的东西，这里为了演示只依赖了必要的依赖。

### 核心代码

- **UserProperties**用户属性配置类。

```java
@ConfigurationProperties(prefix = "user")
@Data
public class UserProperties {
    private String userName = "mac";
    private Integer age = 18;
}
```

- **UserService**和**DefaultUserService**接口定义以及默认实现

```java
public interface UserService {
    /**
     * 通过用户名获取用户
     * @return 用户
     */
    User getUser();
}


public class DefaultUserService implements UserService{

    @Autowired
    private UserProperties properties;

    @Override
    public User getUser() {
        return new User("DefaultUserService:"+properties.getUserName(),properties.getAge());
    }
}
```

- **UserServiceAutoConfig**默认自动配置。

```java
@Configuration
@EnableConfigurationProperties(UserProperties.class)
public class UserServiceAutoConfig {

    @Bean
    @ConditionalOnMissingBean
    public UserService userService(){
        return new DefaultUserService();
    }
}
```

其中**ConditionalOnMissingBean**注解的意思是当这个Bean不存在的时候才会注入这个默认实现。

#### spring.factories

上面所有的准备工作都完成了，接下来就来到最关键的一步spring.factories。它是Spring提供的一种SPI机制，实现机制上也是仿造JAVA的SPI来实现的。我们在**resources\META-INF**目录下创建一个spring.factories文件，文件内的内容如下：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.buydeem.user.UserServiceAutoConfig
```

后面的值就是我们前面定义的**s UserServiceAutoConf**全类名。

## 验证使用

通过上面简单的几步我们就完成了一个简单的SpringStarter。下面我们重新构建一个web项目来测试是否可以使用。在新的项目中我们依赖自己构建的Starter配置如下：

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>com.buydeem</groupId>
            <artifactId>user-spring-boot-starter</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>
```

接下来我们配置用户的名称和年龄这两个属性在application.properties中。

```properties
user.user-name=小明
user.age=18
```

我们通过一个简单的接口打印出我们获得的User内容。

```java
@RestController
@RequestMapping("user")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping
    public User getUser(){
        return userService.getUser();
    }
}
```

请求接口打印结果如下：

```json
{
    "userName": "DefaultUserService:小明",
    "age": 18
}
```

从上面结果我们可以看出我们构建的Starter已经能够正常使用了。但是实际我们在开发的过程中，很可能会遇到自动配置不太符合我们自己实际的要求，这时候该怎么办呢？很简单，还记得之前我们在```UserServiceAutoConfig```中添加的```ConditionalOnMissingBean```注解吗，我们可以通过覆盖掉默认的```UserService```实现注入我们自己实现的。

```java
@Configuration
public class BeanConfig {

    @Bean
    public UserService userService(){
        return new WebUserService();
    }
}
```

再次运行代码接口返回结果如下：

```json
{
    "userName": "WebUserService:小明",
    "age": 18
}
```

## 总结

简单总结一下上面的步骤就是那么几步：

- 构建Starter模块并引入必要的依赖

- Starter功能代码实现和自动配置类

- 配置spring.factories文件

示例代码：```https://github.com/I-Like-Pepsi/demo-spring-boot-starter```
