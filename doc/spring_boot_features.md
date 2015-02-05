Spring Boot特性
===============
### SpringApplication
SpringApplication类提供了一种从main()方法启动Spring应用的便捷方式。在很多情况下，你只需委托给SpringApplication.run这个静态方法：
```java
public static void main(String[] args){
    SpringApplication.run(MySpringConfiguration.class, args);
}
```
* 自定义Banner
  
通过在classpath下添加一个banner.txt或设置banner.location来指定相应的文件可以改变启动过程中打印的banner。如果这个文件有特殊的编码，你可以使用banner.encoding设置它（默认为UTF-8）。

在banner.txt中可以使用如下的变量：

| 变量        | 描述     | 
| ----------- | :--------|
|${application.version}|MANIFEST.MF中声明的应用版本号，例如1.0|
|${application.formatted-version}|MANIFEST.MF中声明的被格式化后的应用版本号（被括号包裹且以v作为前缀），用于显示，例如(v1.0)|
|${spring-boot.version}|正在使用的Spring Boot版本号，例如1.2.2.BUILD-SNAPSHOT|
|${spring-boot.formatted-version}|正在使用的Spring Boot被格式化后的版本号（被括号包裹且以v作为前缀）,  用于显示，例如(v1.2.2.BUILD-SNAPSHOT)|

**注**：如果想以编程的方式产生一个banner，可以使用SpringBootApplication.setBanner(…)方法。使用org.springframework.boot.Banner接口，实现你自己的printBanner()方法。

* 自定义SpringApplication

如果默认的SpringApplication不符合你的口味，你可以创建一个本地的实例并自定义它。例如，关闭banner你可以这样写：
```java
public static void main(String[] args){
    SpringApplication app = new SpringApplication(MySpringConfiguration.class);
    app.setShowBanner(false);
    app.run(args);
}
```
**注**：传递给SpringApplication的构造器参数是spring beans的配置源。在大多数情况下，这些将是@Configuration类的引用，但它们也可能是XML配置或要扫描包的引用。

你也可以使用application.properties文件来配置SpringApplication。具体参考[Externalized 配置](#Externalized 配置)。查看配置选项的完整列表，可参考[SpringApplication Javadoc](http://docs.spring.io/spring-boot/docs/1.2.2.BUILD-SNAPSHOT/api/org/springframework/boot/SpringApplication.html).
    
* 流畅的构建API

如果你需要创建一个分层的ApplicationContext（多个具有父子关系的上下文），或你只是喜欢使用流畅的构建API，你可以使用SpringApplicationBuilder。SpringApplicationBuilder允许你以链式方式调用多个方法，包括可以创建层次结构的parent和child方法。
```java
new SpringApplicationBuilder()
    .showBanner(false)
    .sources(Parent.class)
    .child(Application.class)
    .run(args);
```
**注**：创建ApplicationContext层次时有些限制，比如，Web组件(components)必须包含在子上下文(child context)中，且相同的Environment即用于父上下文也用于子上下文中。具体参考[SpringApplicationBuilder javadoc](http://docs.spring.io/spring-boot/docs/1.2.2.BUILD-SNAPSHOT/api/org/springframework/boot/builder/SpringApplicationBuilder.html)

* Application事件和监听器

除了常见的Spring框架事件，比如[ContextRefreshedEvent](http://docs.spring.io/spring/docs/4.1.4.RELEASE/javadoc-api/org/springframework/context/event/ContextRefreshedEvent.html)，一个SpringApplication也发送一些额外的应用事件。一些事件实际上是在ApplicationContext被创建前触发的。

你可以使用多种方式注册事件监听器，最普通的是使用SpringApplication.addListeners(…)方法。在你的应用运行时，应用事件会以下面的次序发送：

1. 在运行开始，但除了监听器注册和初始化以外的任何处理之前，会发送一个ApplicationStartedEvent。
2. 在Environment将被用于已知的上下文，但在上下文被创建前，会发送一个ApplicationEnvironmentPreparedEvent。
3. 在refresh开始前，但在bean定义已被加载后，会发送一个ApplicationPreparedEvent。
4. 启动过程中如果出现异常，会发送一个ApplicationFailedEvent。

**注**：你通常不需要使用应用程序事件，但知道它们的存在会很方便（在某些场合可能会使用到）。在Spring内部，Spring Boot使用事件处理各种各样的任务。

* Web环境

一个SpringApplication将尝试为你创建正确类型的ApplicationContext。在默认情况下，使用AnnotationConfigApplicationContext或AnnotationConfigEmbeddedWebApplicationContext取决于你正在开发的是否是web应用。

用于确定一个web环境的算法相当简单（基于是否存在某些类）。如果需要覆盖默认行为，你可以使用setWebEnvironment(boolean webEnvironment)。通过调用setApplicationContextClass(…)，你可以完全控制ApplicationContext的类型。

**注**：当JUnit测试里使用SpringApplication时，调用setWebEnvironment(false)是可取的。

* 命令行启动器

如果你想获取原始的命令行参数，或一旦SpringApplication启动，你需要运行一些特定的代码，你可以实现CommandLineRunner接口。在所有实现该接口的Spring beans上将调用run(String… args)方法。
```java
import org.springframework.boot.*
import org.springframework.stereotype.*

@Component
public class MyBean implements CommandLineRunner {
    public void run(String... args) {
        // Do something...
    }
}
```
如果一些CommandLineRunner beans被定义必须以特定的次序调用，你可以额外实现org.springframework.core.Ordered接口或使用org.springframework.core.annotation.Order注解。

* Application退出

每个SpringApplication在退出时为了确保ApplicationContext被优雅的关闭，将会注册一个JVM的shutdown钩子。所有标准的Spring生命周期回调（比如，DisposableBean接口或@PreDestroy注解）都能使用。

此外，如果beans想在应用结束时返回一个特定的退出码（exit code），可以实现org.springframework.boot.ExitCodeGenerator接口。

### 外化配置

Spring Boot允许外化（externalize）你的配置，这样你能够在不同的环境下使用相同的代码。你可以使用properties文件，YAML文件，环境变量和命令行参数来外化配置。使用@Value注解，可以直接将属性值注入到你的beans中，并通过Spring的Environment抽象或绑定到结构化对象来访问。

Spring Boot使用一个非常特别的PropertySource次序来允许对值进行合理的覆盖，需要以下面的次序考虑属性：

1. 命令行参数
2. 来自于java:comp/env的JNDI属性
3. Java系统属性（System.getProperties()）
4. 操作系统环境变量
5. 只有在random.*里包含的属性会产生一个RandomValuePropertySource
6. 在打包的jar外的应用程序配置文件（application.properties，包含YAML和profile变量）
7. 在打包的jar内的应用程序配置文件（application.properties，包含YAML和profile变量）
8. 在@Configuration类上的@PropertySource注解
9. 默认属性（使用SpringApplication.setDefaultProperties指定）

下面是一个具体的示例（假设你开发一个使用name属性的@Component）：
```java
import org.springframework.stereotype.*
import org.springframework.beans.factory.annotation.*

@Component
public class MyBean {
    @Value("${name}")
    private String name;
    // ...
}
```
你可以将一个application.properties文件捆绑到jar内，用来提供一个合理的默认name属性值。当运行在生产环境时，可以在jar外提供一个application.properties文件来覆盖name属性。对于一次性的测试，你可以使用特定的命令行开关启动（比如，java -jar app.jar --name="Spring"）。

RandomValuePropertySource在注入随机值（比如，密钥或测试用例）时很有用。它能产生整数，longs或字符串，比如：
```java
my.secret=${random.value}
my.number=${random.int}
my.bignumber=${random.long}
my.number.less.than.ten=${random.int(10)}
my.number.in.range=${random.int[1024,65536]}
```
random.int*语法是OPEN value (,max) CLOSE，此处OPEN，CLOSE可以是任何字符，并且value，max是整数。如果提供max，那么value是最小的值，max是最大的值（不包含在内）。

* 访问命令行属性

默认情况下，SpringApplication将任何可选的命令行参数（以'--'开头，比如，--server.port=9000）转化为property，并将其添加到Spring Environment中。如上所述，命令行属性总是优先于其他属性源。

如果你不想将命令行属性添加到Environment里，你可以使用SpringApplication.setAddCommandLineProperties(false)来禁止它们。

* Application属性文件

SpringApplication将从以下位置加载application.properties文件，并把它们添加到Spring Environment中：

1. 当前目录下的一个/config子目录
2. 当前目录
3. 一个classpath下的/config包
4. classpath根路径（root）

这个列表是按优先级排序的（列表中位置高的将覆盖位置低的）。

**注**：你可以使用YAML（'.yml'）文件替代'.properties'。

如果不喜欢将application.properties作为配置文件名，你可以通过指定spring.config.name环境属性来切换其他的名称。你也可以使用spring.config.location环境属性来引用一个明确的路径（目录位置或文件路径列表以逗号分割）。
```shell
$ java -jar myproject.jar --spring.config.name=myproject
//or
$ java -jar myproject.jar --spring.config.location=classpath:/default.properties,classpath:/override.properties
```
如果spring.config.location包含目录（相对于文件），那它们应该以/结尾（在加载前，spring.config.name产生的名称将被追加到后面）。不管spring.config.location是什么值，默认的搜索路径classpath:,classpath:/config,file:,file:config/总会被使用。以这种方式，你可以在application.properties中为应用设置默认值，然后在运行的时候使用不同的文件覆盖它，同时保留默认配置。

**注**：如果你使用环境变量而不是系统配置，大多数操作系统不允许以句号分割（period-separated）的key名称，但你可以使用下划线（underscores）代替（比如，使用SPRING_CONFIG_NAME代替spring.config.name）。如果你的应用运行在一个容器中，那么JNDI属性（java:comp/env）或servlet上下文初始化参数可以用来取代环境变量或系统属性，当然也可以使用环境变量或系统属性。

* 特定的Profile属性

除了application.properties文件，特定配置属性也能通过命令惯例application-{profile}.properties来定义。特定Profile属性从跟标准application.properties相同的路径加载，并且特定profile文件会覆盖默认的配置。

* 属性占位符

当application.properties里的值被使用时，它们会被存在的Environment过滤，所以你能够引用先前定义的值（比如，系统属性）。
```java
app.name=MyApp
app.description=${app.name} is a Spring Boot application
```
**注**：你也能使用相应的技巧为存在的Spring Boot属性创建'短'变量，具体参考[Section 63.3, “Use ‘short’ command line arguments”](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#howto-use-short-command-line-arguments)。

* 使用YAML代替Properties

[YAML](http://yaml.org/)是JSON的一个超集，也是一种方便的定义层次配置数据的格式。无论你何时将[SnakeYAML ](http://code.google.com/p/snakeyaml/)库放到classpath下，SpringApplication类都会自动支持YAML作为properties的替换。

**注**：如果你使用'starter POMs'，spring-boot-starter会自动提供SnakeYAML。

* 加载YAML

Spring框架提供两个便利的类用于加载YAML文档，YamlPropertiesFactoryBean会将YAML作为Properties来加载，YamlMapFactoryBean会将YAML作为Map来加载。

示例：
```json
environments:
    dev:
        url: http://dev.bar.com
        name: Developer Setup
    prod:
        url: http://foo.bar.com
        name: My Cool App
```
上面的YAML文档会被转化到下面的属性中：
```java
environments.dev.url=http://dev.bar.com
environments.dev.name=Developer Setup
environments.prod.url=http://foo.bar.com
environments.prod.name=My Cool App
```
YAML列表被表示成使用[index]间接引用作为属性keys的形式，例如下面的YAML：
```json
my:
   servers:
       - dev.bar.com
       - foo.bar.com
```
将会转化到下面的属性中:
```java
my.servers[0]=dev.bar.com
my.servers[1]=foo.bar.com
```
使用Spring DataBinder工具绑定那样的属性（这是@ConfigurationProperties做的事），你需要确定目标bean中有个java.util.List或Set类型的属性，并且需要提供一个setter或使用可变的值初始化它，比如，下面的代码将绑定上面的属性：
```java
@ConfigurationProperties(prefix="my")
public class Config {
    private List<String> servers = new ArrayList<String>();
    public List<String> getServers() {
        return this.servers;
    }
}
```
* 在Spring环境中使用YAML暴露属性

YamlPropertySourceLoader类能够用于将YAML作为一个PropertySource导出到Sprig Environment。这允许你使用熟悉的@Value注解和占位符语法访问YAML属性。

* Multi-profile YAML文档

你可以在单个文件中定义多个特定配置（profile-specific）的YAML文档，并通过一个spring.profiles key标示应用的文档。例如：
```json
server:
    address: 192.168.1.100
---
spring:
    profiles: development
server:
    address: 127.0.0.1
---
spring:
    profiles: production
server:
    address: 192.168.1.120
```
在上面的例子中，如果development配置被激活，那server.address属性将是127.0.0.1。如果development和production配置（profiles）没有启用，则该属性的值将是192.168.1.100。

* YAML缺点

YAML文件不能通过@PropertySource注解加载。所以，在这种情况下，如果需要使用@PropertySource注解的方式加载值，那就要使用properties文件。

* 类型安全的配置属性

使用@Value("${property}")注解注入配置属性有时可能比较笨重，特别是需要使用多个properties或你的数据本身有层次结构。为了控制和校验你的应用配置，Spring Boot提供一个允许强类型beans的替代方法来使用properties。

示例：
```java
@Component
@ConfigurationProperties(prefix="connection")
public class ConnectionSettings {
    private String username;
    private InetAddress remoteAddress;
    // ... getters and setters
}
```
当@EnableConfigurationProperties注解应用到你的@Configuration时，任何被@ConfigurationProperties注解的beans将自动被Environment属性配置。这种风格的配置特别适合与SpringApplication的外部YAML配置进行配合使用。
```json
# application.yml
connection:
    username: admin
    remoteAddress: 192.168.1.1
# additional configuration as required
```
为了使用@ConfigurationProperties beans，你可以使用与其他任何bean相同的方式注入它们。
```java
@Service
public class MyService {
    @Autowired
    private ConnectionSettings connection;
     //...
    @PostConstruct
    public void openConnection() {
        Server server = new Server();
        this.connection.configure(server);
    }
}
```
你可以通过在@EnableConfigurationProperties注解中直接简单的列出属性类来快捷的注册@ConfigurationProperties bean的定义。
```java
@Configuration
@EnableConfigurationProperties(ConnectionSettings.class)
public class MyConfiguration {
}
```
**注**：使用@ConfigurationProperties能够产生可被IDEs使用的元数据文件。具体参考[Appendix B, Configuration meta-data](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#configuration-metadata)。

* 第3方配置

正如使用@ConfigurationProperties注解一个类，你也可以在@Bean方法上使用它。当你需要绑定属性到不受你控制的第三方组件时，这种方式非常有用。

为了从Environment属性配置一个bean，将@ConfigurationProperties添加到它的bean注册过程：
```java
@ConfigurationProperties(prefix = "foo")
@Bean
public FooComponent fooComponent() {
    ...
}
```
和上面ConnectionSettings的示例方式相同，任何以foo为前缀的属性定义都会被映射到FooComponent上。

* 松散的绑定（Relaxed binding）

Spring Boot使用一些宽松的规则用于绑定Environment属性到@ConfigurationProperties beans，所以Environment属性名和bean属性名不需要精确匹配。常见的示例中有用的包括虚线分割（比如，context--path绑定到contextPath）和将环境属性转为大写字母（比如，PORT绑定port）。

示例：
```java
@Component
@ConfigurationProperties(prefix="person")
public class ConnectionSettings {
    private String firstName;
}
```
下面的属性名都能用于上面的@ConfigurationProperties类：

| 属性        | 说明   |
| --------    | :----- |
|person.firstName|标准驼峰规则|
|person.first-name|虚线表示，推荐用于.properties和.yml文件中|
|PERSON_FIRST_NAME|大写形式，使用系统环境变量时推荐|

Spring会尝试强制外部的应用属性在绑定到@ConfigurationProperties beans时类型是正确的。如果需要自定义类型转换，你可以提供一个ConversionService bean（bean id为conversionService）或自定义属性编辑器（通过一个CustomEditorConfigurer bean）。

* @ConfigurationProperties校验 

Spring Boot将尝试校验外部的配置，默认使用JSR-303（如果在classpath路径中）。你可以轻松的为你的@ConfigurationProperties类添加JSR-303 javax.validation约束注解：
```java
@Component
@ConfigurationProperties(prefix="connection")
public class ConnectionSettings {
    @NotNull
    private InetAddress remoteAddress;
    // ... getters and setters
}
```
你也可以通过创建一个叫做configurationPropertiesValidator的bean来添加自定义的Spring Validator。

**注**：spring-boot-actuator模块包含一个暴露所有@ConfigurationProperties beans的端点。简单地将你的web浏览器指向/configprops或使用等效的JMX端点。具体参考[Production ready features](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#production-ready-endpoints)。

### Profiles
Spring Profiles提供了一种隔离应用程序配置的方式，并让这些配置只能在特定的环境下生效。任何@Component或@Configuration都能被@Profile标记，从而限制加载它的时机。
```java
@Configuration
@Profile("production")
public class ProductionConfiguration {
    // ...
}
```
以正常的Spring方式，你可以使用一个spring.profiles.active的Environment属性来指定哪个配置生效。你可以使用平常的任何方式来指定该属性，例如，可以将它包含到你的application.properties中：
```java
spring.profiles.active=dev,hsqldb
```
或使用命令行开关：
```shell
--spring.profiles.active=dev,hsqldb
```
* 添加激活的配置(profiles)

spring.profiles.active属性和其他属性一样都遵循相同的排列规则，最高的PropertySource获胜。也就是说，你可以在application.properties中指定生效的配置，然后使用命令行开关替换它们。

有时，将特定的配置属性添加到生效的配置中而不是替换它们是有用的。spring.profiles.include属性可以用来无条件的添加生效的配置。SpringApplication的入口点也提供了一个用于设置额外配置的Java API（比如，在那些通过spring.profiles.active属性生效的配置之上）：参考setAdditionalProfiles()方法。

示例：当一个应用使用下面的属性，并用`--spring.profiles.active=prod`开关运行，那proddb和prodmq配置也会生效：
```java
---
my.property: fromyamlfile
---
spring.profiles: prod
spring.profiles.include: proddb,prodmq
```
**注**：spring.profiles属性可以定义到一个YAML文档中，用于决定什么时候该文档被包含进配置中。具体参考[Section 63.6, “Change configuration depending on the environment”](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#howto-change-configuration-depending-on-the-environment)

* 以编程方式设置profiles

在应用运行前，你可以通过调用SpringApplication.setAdditionalProfiles(…)方法，以编程的方式设置生效的配置。使用Spring的ConfigurableEnvironment接口激动配置也是可行的。

* Profile特定配置文件

application.properties（或application.yml）和通过@ConfigurationProperties引用的文件这两种配置特定变种都被当作文件来加载的，具体参考[Section 23.3, “Profile specific properties”](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-external-config-profile-specific-properties)。

### 日志
Spring Boot内部日志系统使用的是[Commons Logging](http://commons.apache.org/logging)，但开放底层的日志实现。默认为会[Java Util Logging](http://docs.oracle.com/javase/7/docs/api/java/util/logging/package-summary.html), [Log4J](http://logging.apache.org/log4j/), [Log4J2](http://logging.apache.org/log4j/2.x/)和[Logback](http://logback.qos.ch/)提供配置。每种情况下都会预先配置使用控制台输出，也可以使用可选的文件输出。

默认情况下，如果你使用'Starter POMs'，那么就会使用Logback记录日志。为了确保那些使用Java Util Logging, Commons Logging, Log4J或SLF4J的依赖库能够正常工作，正确的Logback路由也被包含进来。

**注**：如果上面的列表看起来令人困惑，不要担心，Java有很多可用的日志框架。通常，你不需要改变日志依赖，Spring Boot默认的就能很好的工作。

* 日志格式

Spring Boot默认的日志输出格式如下：
```java
2014-03-05 10:57:51.112  INFO 45469 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/7.0.52
2014-03-05 10:57:51.253  INFO 45469 --- [ost-startStop-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2014-03-05 10:57:51.253  INFO 45469 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1358 ms
2014-03-05 10:57:51.698  INFO 45469 --- [ost-startStop-1] o.s.b.c.e.ServletRegistrationBean        : Mapping servlet: 'dispatcherServlet' to [/]
2014-03-05 10:57:51.702  INFO 45469 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
```
输出的节点（items）如下：

1. 日期和时间 - 精确到毫秒，且易于排序。
2. 日志级别 - ERROR, WARN, INFO, DEBUG 或 TRACE。
3. Process ID。
4. 一个用于区分实际日志信息开头的---分隔符。
5. 线程名 - 包括在方括号中（控制台输出可能会被截断）。
6. 日志名 - 通常是源class的类名（缩写）。
7. 日志信息。

* 控制台输出

默认的日志配置会在写日志消息时将它们回显到控制台。默认，ERROR, WARN和INFO级别的消息会被记录。可以在启动应用时，通过`--debug`标识开启控制台的DEBUG级别日志记录。
```shell
$ java -jar myapp.jar --debug
```
如果你的终端支持ANSI，为了增加可读性将会使用彩色的日志输出。你可以设置`spring.output.ansi.enabled`为一个[支持的值](http://docs.spring.io/spring-boot/docs/1.2.2.BUILD-SNAPSHOT/api/org/springframework/boot/ansi/AnsiOutput.Enabled.html)来覆盖自动检测。

* 文件输出

默认情况下，Spring Boot只会将日志记录到控制台而不会写进日志文件。如果除了输出到控制台你还想写入到日志文件，那你需要设置`logging.file`或`logging.path`属性（例如在你的application.properties中）。

下表显示如何组合使用`logging.*`：

|logging.file|logging.path| 示例 | 描述  |
| --------   | :-----  | :-----  | :-----|
|  (none)    | (none)  |         | 只记录到控制台 |
|Specific file|(none)|my.log|写到特定的日志文件里，名称可以是一个精确的位置或相对于当前目录|
|(none)|Specific folder|/var/log|写到特定文件夹下的spring.log里，名称可以是一个精确的位置或相对于当前目录|

日志文件每达到10M就会被轮换（分割），和控制台一样，默认记录ERROR, WARN和INFO级别的信息。

* 日志级别

所有支持的日志系统在Spring的Environment（例如在application.properties里）都有通过'logging.level.*=LEVEL'（'LEVEL'是TRACE, DEBUG, INFO, WARN, ERROR, FATAL, OFF中的一个）设置的日志级别。

示例：application.properties
```java
logging.level.org.springframework.web: DEBUG
logging.level.org.hibernate: ERROR
```
* 自定义日志配置

通过将适当的库添加到classpath，可以激活各种日志系统。然后在classpath的根目录(root)或通过Spring Environment的`logging.config`属性指定的位置提供一个合适的配置文件来达到进一步的定制（注意由于日志是在ApplicationContext被创建之前初始化的，所以不可能在Spring的@Configuration文件中，通过@PropertySources控制日志。系统属性和平常的Spring Boot外部配置文件能正常工作）。

根据你的日志系统，下面的文件会被加载：

| 日志系统        | 定制   |
| --------   | :-----:  | 
|Logback|logback.xml|
|Log4j|log4j.properties或log4j.xml|
|Log4j2|log4j2.xml|
|JDK (Java Util Logging)|logging.properties|

为了帮助定制一些其他的属性，从Spring的Envrionment转换到系统属性：

| Spring Environment| System Property| 评价 |
| --------   | :-----:  | :----:  |
|logging.file|LOG_FILE|如果定义，在默认的日志配置中使用|
|logging.path|LOG_PATH|如果定义，在默认的日志配置中使用|
|PID|PID|当前的处理进程(process)ID（如果能够被发现且还没有作为操作系统环境变量被定义）|

所有支持的日志系统在解析它们的配置文件时都能查询系统属性。具体可以参考spring-boot.jar中的默认配置。

**注**：在运行可执行的jar时，Java Util Logging有类加载问题，我们建议你尽可能避免使用它。

### 开发Web应用
Spring Boot非常适合开发web应用程序。你可以使用内嵌的Tomcat，Jetty或Undertow轻轻松松地创建一个HTTP服务器。大多数的web应用都使用spring-boot-starter-web模块进行快速搭建和运行。

* Spring Web MVC框架

Spring Web MVC框架（通常简称为"Spring MVC"）是一个富"模型，视图，控制器"的web框架。
Spring MVC允许你创建特定的@Controller或@RestController beans来处理传入的HTTP请求。
使用@RequestMapping注解可以将控制器中的方法映射到相应的HTTP请求。

示例：
```java
@RestController
@RequestMapping(value="/users")
public class MyRestController {

    @RequestMapping(value="/{user}", method=RequestMethod.GET)
    public User getUser(@PathVariable Long user) {
        // ...
    }

    @RequestMapping(value="/{user}/customers", method=RequestMethod.GET)
    List<Customer> getUserCustomers(@PathVariable Long user) {
        // ...
    }

    @RequestMapping(value="/{user}", method=RequestMethod.DELETE)
    public User deleteUser(@PathVariable Long user) {
        // ...
    }
}
```
* Spring MVC自动配置

Spring Boot为Spring MVC提供适用于多数应用的自动配置功能。在Spring默认基础上，自动配置添加了以下特性：

1. 引入ContentNegotiatingViewResolver和BeanNameViewResolver beans。
2. 对静态资源的支持，包括对WebJars的支持。
3. 自动注册Converter，GenericConverter，Formatter beans。
4. 对HttpMessageConverters的支持。
5. 自动注册MessageCodeResolver。
6. 对静态index.html的支持。
7. 对自定义Favicon的支持。

如果想全面控制Spring MVC，你可以添加自己的@Configuration，并使用@EnableWebMvc对其注解。如果想保留Spring Boot MVC的特性，并只是添加其他的[MVC配置](http://docs.spring.io/spring/docs/4.1.4.RELEASE/spring-framework-reference/htmlsingle#mvc)(拦截器，formatters，视图控制器等)，你可以添加自己的WebMvcConfigurerAdapter类型的@Bean（不使用@EnableWebMvc注解）。

* HttpMessageConverters

Spring MVC使用HttpMessageConverter接口转换HTTP请求和响应。合理的缺省值被包含的恰到好处（out of the box），例如对象可以自动转换为JSON（使用Jackson库）或XML（如果Jackson XML扩展可用则使用它，否则使用JAXB）。字符串默认使用UTF-8编码。

如果需要添加或自定义转换器，你可以使用Spring Boot的HttpMessageConverters类：
```java
import org.springframework.boot.autoconfigure.web.HttpMessageConverters;
import org.springframework.context.annotation.*;
import org.springframework.http.converter.*;

@Configuration
public class MyConfiguration {

    @Bean
    public HttpMessageConverters customConverters() {
        HttpMessageConverter<?> additional = ...
        HttpMessageConverter<?> another = ...
        return new HttpMessageConverters(additional, another);
    }
}
```
任何在上下文中出现的HttpMessageConverter bean将会添加到converters列表，你可以通过这种方式覆盖默认的转换器（converters）。

* MessageCodesResolver

Spring MVC有一个策略，用于从绑定的errors产生用来渲染错误信息的错误码：MessageCodesResolver。如果设置`spring.mvc.message-codes-resolver.format`属性为`PREFIX_ERROR_CODE`或`POSTFIX_ERROR_CODE`（具体查看`DefaultMessageCodesResolver.Format`枚举值），Spring Boot会为你创建一个MessageCodesResolver。

* 静态内容

默认情况下，Spring Boot从classpath下一个叫/static（/public，/resources或/META-INF/resources）的文件夹或从ServletContext根目录提供静态内容。这使用了Spring MVC的ResourceHttpRequestHandler，所以你可以通过添加自己的WebMvcConfigurerAdapter并覆写addResourceHandlers方法来改变这个行为（加载静态文件）。

在一个单独的web应用中，容器默认的servlet是开启的，如果Spring决定不处理某些请求，默认的servlet作为一个回退（降级）将从ServletContext根目录加载内容。大多数时候，这不会发生（除非你修改默认的MVC配置），因为Spring总能够通过DispatcherServlet处理请求。

此外，上述标准的静态资源位置有个例外情况是[Webjars内容](http://www.webjars.org/)。任何在/webjars/**路径下的资源都将从jar文件中提供，只要它们以Webjars的格式打包。

**注**：如果你的应用将被打包成jar，那就不要使用src/main/webapp文件夹。尽管该文件夹是一个共同的标准，但它仅在打包成war的情况下起作用，并且如果产生一个jar，多数构建工具都会静悄悄的忽略它。

* 模板引擎

正如REST web服务，你也可以使用Spring MVC提供动态HTML内容。Spring MVC支持各种各样的模板技术，包括Velocity, FreeMarker和JSPs。很多其他的模板引擎也提供它们自己的Spring MVC集成。

Spring Boot为以下的模板引擎提供自动配置支持：

1. [FreeMarker](http://freemarker.org/docs/)
2. [Groovy](http://beta.groovy-lang.org/docs/groovy-2.3.0/html/documentation/markup-template-engine.html)
3. [Thymeleaf](http://www.thymeleaf.org/)
4. [Velocity](http://velocity.apache.org/)

**注**：如果可能的话，应该忽略JSPs，因为在内嵌的servlet容器使用它们时存在一些[已知的限制](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-jsp-limitations)。

当你使用这些引擎的任何一种，并采用默认的配置，你的模板将会从src/main/resources/templates目录下自动加载。

**注**：IntelliJ IDEA根据你运行应用的方式会对classpath进行不同的整理。在IDE里通过main方法运行你的应用跟从Maven或Gradle或打包好的jar中运行相比会导致不同的顺序。这可能导致Spring Boot不能从classpath下成功地找到模板。如果遇到这个问题，你可以在IDE里重新对classpath进行排序，将模块的类和资源放到第一位。或者，你可以配置模块的前缀为classpath*:/templates/，这样会查找classpath下的所有模板目录。

* 错误处理

Spring Boot默认提供一个/error映射用来以合适的方式处理所有的错误，并且它在servlet容器中注册了一个全局的
错误页面。对于机器客户端（相对于浏览器而言，浏览器偏重于人的行为），它会产生一个具有详细错误，HTTP状态，异常信息的JSON响应。对于浏览器客户端，它会产生一个白色标签样式（whitelabel）的错误视图，该视图将以HTML格式显示同样的数据（可以添加一个解析为erro的View来自定义它）。为了完全替换默认的行为，你可以实现ErrorController，并注册一个该类型的bean定义，或简单地添加一个ErrorAttributes类型的bean以使用现存的机制，只是替换显示的内容。

如果在某些条件下需要比较多的错误页面，内嵌的servlet容器提供了一个统一的Java DSL（领域特定语言）来自定义错误处理。
示例：
```java
@Bean
public EmbeddedServletContainerCustomizer containerCustomizer(){
    return new MyCustomizer();
}

// ...
private static class MyCustomizer implements EmbeddedServletContainerCustomizer {
    @Override
    public void customize(ConfigurableEmbeddedServletContainer container) {
        container.addErrorPages(new ErrorPage(HttpStatus.BAD_REQUEST, "/400"));
    }
}
```
你也可以使用常规的Spring MVC特性来处理错误，比如[@ExceptionHandler方法](http://docs.spring.io/spring/docs/4.1.4.RELEASE/spring-framework-reference/htmlsingle/#mvc-exceptionhandlers)和[@ControllerAdvice](http://docs.spring.io/spring/docs/4.1.4.RELEASE/spring-framework-reference/htmlsingle/#mvc-ann-controller-advice)。ErrorController将会捡起任何没有处理的异常。

N.B. 如果你为一个路径注册一个ErrorPage，最终被一个过滤器（Filter）处理（对于一些非Spring web框架，像Jersey和Wicket这很常见），然后过滤器需要显式注册为一个ERROR分发器（dispatcher）。
```java
@Bean
public FilterRegistrationBean myFilter() {
    FilterRegistrationBean registration = new FilterRegistrationBean();
    registration.setFilter(new MyFilter());
    ...
    registration.setDispatcherTypes(EnumSet.allOf(DispatcherType.class));
    return registration;
}
```
**注**：默认的FilterRegistrationBean没有包含ERROR分发器类型。

* Spring HATEOAS

如果你正在开发一个使用超媒体的RESTful API，Spring Boot将为Spring HATEOAS提供自动配置，这在多数应用中都工作良好。自动配置替换了对使用@EnableHypermediaSupport的需求，并注册一定数量的beans来简化构建基于超媒体的应用，这些beans包括一个LinkDiscoverer和配置好的用于将响应正确编排为想要的表示的ObjectMapper。ObjectMapper可以根据spring.jackson.*属性或一个存在的Jackson2ObjectMapperBuilder bean进行自定义。

通过使用@EnableHypermediaSupport，你可以控制Spring HATEOAS的配置。注意这会禁用上述的对ObjectMapper的自定义。

* JAX-RS和Jersey

如果喜欢JAX-RS为REST端点提供的编程模型，你可以使用可用的实现替代Spring MVC。如果在你的应用上下文中将Jersey 1.x和Apache Celtix的Servlet或Filter注册为一个@Bean，那它们工作的相当好。Jersey 2.x有一些原生的Spring支持，所以我们会在Spring Boot为它提供自动配置支持，连同一个启动器（starter）。

想要开始使用Jersey 2.x只需要加入spring-boot-starter-jersey依赖，然后你需要一个ResourceConfig类型的@Bean，用于注册所有的端点（endpoints）。
```java
@Component
public class JerseyConfig extends ResourceConfig {
    public JerseyConfig() {
        register(Endpoint.class);
    }
}
```
所有注册的端点都应该被@Components和HTTP资源annotations（比如@GET）注解。
```java
@Component
@Path("/hello")
public class Endpoint {
    @GET
    public String message() {
        return "Hello";
    }
}
```
由于Endpoint是一个Spring组件（@Component），所以它的生命周期受Spring管理，并且你可以使用@Autowired添加依赖及使用@Value注入外部配置。Jersey servlet将被注册，并默认映射到/*。你可以将@ApplicationPath添加到ResourceConfig来改变该映射。

默认情况下，Jersey将在一个ServletRegistrationBean类型的@Bean中被设置成名称为jerseyServletRegistration的Servlet。通过创建自己的相同名称的bean，你可以禁止或覆盖这个bean。你也可以通过设置`spring.jersey.type=filter`来使用一个Filter代替Servlet（在这种情况下，被覆盖或替换的@Bean是jerseyFilterRegistration）。该servlet有@Order属性，你可以通过`spring.jersey.filter.order`进行设置。不管是Servlet还是Filter注册都可以使用spring.jersey.init.*定义一个属性集合作为初始化参数传递过去。

这里有一个[Jersey示例](http://github.com/spring-projects/spring-boot/tree/master/spring-boot-samples/spring-boot-sample-jersey)，你可以查看如何设置相关事项。

* 内嵌servlet容器支持
  1. Servlets和Filters
  2. EmbeddedWebApplicationContext
  3. 自定义内嵌servlet容器
  4. JSP的限制

### 安全

### 使用SQL数据库
* 配置DataSource
* 使用JdbcTemplate
* JPA和Spring Data
  1. 实体类
  2. Spring Data JPA仓库
  3. 创建和删除JPA数据库
  
### 使用NoSQL技术
* Redis
  1. 连接Redis
* MongoDB
  1. 连接MongoDB数据库
  2. MongoDBTemplate
  3. Spring Data MongoDB仓库
* Gemfire
* Solr
  1. 连接Solr
  2. Spring Data Solr仓库 
* Elasticsearch
  1. 连接Elasticsearch
  2. Spring Data Elasticseach仓库
  
### 消息
* JMS
  1. HornetQ支持
  2. ActiveQ支持
  3. 使用JNDI ConnectionFactory
  4. 发送消息
  5. 接收消息

### 发送邮件

### 使用JTA处理分布式事务
* 使用一个Atomikos事务管理器
* 使用一个Bitronix事务管理器
* 使用一个J2EE管理的事务管理器
* 混合XA和non-XA的JMS连接
* 支持可替代的内嵌事务管理器

### Spring集成

### 基于JMX的监控和管理

### 测试
* 测试作用域依赖
* 测试Spring应用
* 测试Spring Boot应用
  1. 使用Spock测试Spring Boot应用
* 测试工具
  1. ConfigFileApplicationContextInitializer
  2. EnvironmentTestUtils
  3. OutputCapture
  4. TestRestTemplate

### 开发自动配置和使用条件
* 理解auto-configured beans
* 定位auto-configuration候选者
* Condition注解
  1. Class条件
  2. Bean条件
  3. Property条件
  4. Resource条件
  5. Web Application条件
  6. SpEL表达式条件

### WebSockets
