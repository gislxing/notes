#Sping Boot 笔记

1、通过访问start.spring.io可以生成初始springBoot项目

2、springBoot项目启动方式：
    1、运行main函数启动
    2、如果在pom中引入了spring编译插件<artifactId>spring-boot-maven-plugin</artifactId>
       则可以执行运行命令：mvn spring-boot:run
    3、java -jar xx.jar

3、自定义注解
    1、在application.properties文件中添加 user.name=abc
    2、在代码中使用注解@Value("${user.name}")使用该值

4、配置文件application.properties使用随机数
    // 随机字符串
    1、com.port=${random.value}
    
    // 随机long
    2、com.value=${random.long}
    
    // 随机int
    3、com.a=${random.int}
    
    // 10以内随机数
    4、com.b=${random.int(10)}
    
    // 10-20的随机数
    4、com.c=${random.int[10, 20]}

5、命令行形式启动springBoot时对配置文件application.properties操作
    // 该--server.port=8888是将服务的端口号修改为8888，此命令等价于在application.properties中添加server.port=8888
    1、java -jar xx.jar --server.port=8888

6、开启springBoot操作控制类
    添加配置属性：endpoints.shutdown.enabled=true

7、启动springBoot时指定不同的配置文件
    新建属性文件application-alpha.properties，并配置好
    启动时如果要指定根据该配置文件启动，则启动命令如下：
        java -jar xxx.jar --spring.profiles.active=alpha

8、集成mybatis（Mysql）
    1、引入依赖文件
        <dependency>
    		<groupId>org.mybatis.spring.boot</groupId>
    		<artifactId>mybatis-spring-boot-starter</artifactId>
    		<version>1.3.1</version>
    	</dependency>
        <dependency>
    		<groupId>mysql</groupId>
    		<artifactId>mysql-connector-java</artifactId>
    		<scope>runtime</scope>
    	</dependency>
    2、application.yml中添加以下配置
        spring:
            datasource:
                url: jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=utf-8
                username: root
                password: 1qaz2wsx
                driver-class-name: com.mysql.jdbc.Driver
                tomcat:
                    max-active: 20
                    max-idle: 10
                    min-idle: 5
                    initial-size: 10
            application:
                name: simple-provider-user
        server:
            port: 8080
        mybatis:
            type-aliases-package: com.zbs.cloud.entity
            mapper-locations: classpath:mapper/*.xml
    3、启动主类上增加dao扫描配置
        @MapperScan("com.zbs.cloud.dao")

9、定制 /info 端点的显示信息(引入actuator后才有效)
    info:   # 定制/info 端点的显示信息, 以下信息为自定义，其中@java.version@为pom文件中配置的节点
        app:
            name: @project.artifactId@
            encoding: @project.build.sourceEncoding@
            java:
            version: @java.version@

10、统一错误页面跳转
    Spring Boot自动配置的默认错误处理器会查找名为 error 的视图，如果找不到就用默认的白标错误视图
    因此，自定义自己的错误页面最简单的方法就是创建一个自定义视图，让解析出的视图名为 error，
    但是如果是使用了模板，则只需要创建 error.html（Thymeleaf模板）/error.ftl（freemarker模板）/error.jsp（jsp模板）等文件并放在 templates 根目录
    Spring Boot会为错误视图提供如下错误属性
        timestamp：错误发生的时间。
        status：HTTP状态码。
        error：错误原因。
        exception：异常的类名。
        message：异常消息（如果这个错误是由异常引起的）。
        errors：BindingResult异常里的各种错误（如果这个错误是由异常引起的）。
        trace：异常跟踪信息（如果这个错误是由异常引起的）。
        path：错误发生时请求的URL路径。
    发生错误后跳转到错误页面(以下的错误页面在/resources/templates/error)
        @Controller
        public class GlobalErrorController implements ErrorController {
            @GetMapping(value = "/error")
            public String defaultBadRequest(Map<String, Object> map) {
                // 此处加入自己的错误处理，或者给错误页面传递参数
                return "/error/404";
            }
            @Override
            public String getErrorPath() {
                // 错误页面路径
                return "/error";
            }
            // 获取当前请求的错误状态码，可根据此错误码跳转到不同的错误页面
            public static int getResponseStatus() {
                RequestContext ctx = RequestContext.getCurrentContext();
                HttpServletResponse response = ctx.getResponse();
                if (response == null) {
                    return HttpStatus.NOT_FOUND.value();
                }
                return response.getStatus();
            }
        }

11、为 Actuator 添加自定义度量信息
    1、将 CounterService 和 GaugeService 注入到我们需要的 bean 中，更新/增加需要的度量值
    2、实现 PublicMetrics 接口，提供自己需要的度量信息

12、创建自定义跟踪仓库存储位置（/trace 端点）
    实现 Spring Boot 的 TraceRepository 接口即可

13、保护所有 Actuator 端点
    1、在 配置属性为 actuator 端点添加访问前缀，例如：
        management.context-path = /mgmt
    2、添加访问权限
        .antMatchers("/mgmt/**").access("hasRole('ADMIN')")

14、统一异常处理
    @ControllerAdvice 注解给所有的 controller 加上额外的处理逻辑
    @ControllerAdvice
    public class GlobalExceptionHandler {
        @ExceptionHandler(value = Exception.class)
        public ModelAndView defaultErrorHandler(HttpServletRequest req, Exception e) throws Exception {
            // 或者此处自定义自己的处理逻辑，返回值类型自定义
            ModelAndView modelAndView = new ModelAndView();
            modelAndView.addObject("notFound", "404");
            modelAndView.setViewName("要跳转到的错误页面路径");
            return modelAndView;
        }
    }

15、日志处理
    在 resources 根目录下创建 logback 配置文件：logback-spring.xml，大致配置如下所示：
    <?xml version="1.0" encoding="UTF-8"?>
    <configuration scan="true" scanPeriod="60 seconds" debug="false">
        <!--输出到控制台 ConsoleAppender-->
        <appender name="consoleLog" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <charset>UTF-8</charset>
                <pattern>%d [%thread] %-5level %logger{36} %line - %msg%n</pattern>
            </encoder>
        </appender>
        <appender name="fileInfoLog" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <File>/var/log/exchange/gatewayZuul.log</File>
            <!--滚动策略，按照时间滚动 TimeBasedRollingPolicy-->
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <!--文件路径,定义了日志的切分方式——把每一天的日志归档到一个文件中,以防止日志填满整个磁盘空间-->
                <FileNamePattern>/var/log/exchange/gatewayZuul-%d{yyyy-MM-dd}.%i.log.zip</FileNamePattern>
                <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                    <!-- 超过最大值，会重新建一个压缩文件-->
                    <maxFileSize>64 MB</maxFileSize>
                </timeBasedFileNamingAndTriggeringPolicy>
                <!--只保留最近365天的日志-->
                <maxHistory>365</maxHistory>
                <!--用来指定日志文件的上限大小，那么到了这个值，就会删除旧的日志-->
                <totalSizeCap>10GB</totalSizeCap>
            </rollingPolicy>
            <!--日志输出编码格式化-->
            <encoder>
                <charset>UTF-8</charset>
                <pattern>%d [%thread] %-5level %logger{36} %line - %msg%n</pattern>
            </encoder>
        </appender>
        <!--level：指定包日志级别，如果不指定则继承root的日志级别-->
        <logger name="com.bihang.exchange" level="info" />
        <!--指定最基础的日志输出级别-->
        <root level="info">
            <!--appender将会添加到这个loger-->
            <appender-ref ref="fileInfoLog"/>
            <appender-ref ref="consoleLog"/>
        </root>
    </configuration>

-------------------------------------------------------------------------------------------

Spring常用注解
@RestController 			构造型（stereotype）注解,告诉Spring以字符串的形式渲染结果，并直接返回给调用者
@EnableAutoConfiguration 	这个注解告诉Spring Boot根据添加的jar依赖猜测你想如何配置Spring
@ComponentScan 				注解自动收集所有的Spring组件，包括 @Configuration 类
@Import 					注解可以用来导入其他配置类
@ImportResource 			注解加载XML配置文件,有两种常用的引入方式：classpath和file，
							例如：@ImportResource(locations={"classpath:test.xml"})  /  @ImportResource(locations={"file:d:\\test.xml"})
@SpringBootApplication 		注解等价于以默认属性使用 @Configuration ， @EnableAutoConfiguration 和 @ComponentScan
@ControllerAdvice			全局异常处理
@Service					service类
@Controller					控制层组件
@Repository					数据访问层，即DAO层
@Component					泛指组件，当组件不好归类是使用此注解
@Entity						实体类
@Transactional				添加事物
@RequestMapping 			注解提供路由信息
@Id							主键
@GeneratedValue				自增
@Resource					自动注入
@Configuration				配置类
@WebAppConfiguration		
@Async						异步执行方法
@ConfigurationProperties    属性注解（该Bean的属性通过setter方法从配置属性值注入）

-------------------------------------------------------------------------------------------


Junit基本注解介绍
//在所有测试方法前执行一次，一般在其中写上整体初始化的代码
@BeforeClass

//在所有测试方法后执行一次，一般在其中写上销毁和释放资源的代码
@AfterClass

//在每个测试方法前执行，一般用来初始化方法（比如我们在测试别的方法时，类中与其他测试方法共享的值已经被改
变，为了保证测试结果的有效性，我们会在@Before注解的方法中重置数据）
@Before

//在每个测试方法后执行，在方法执行完成后要做的事情
@After

// 测试方法执行超过1000毫秒后算超时，测试将失败
@Test(timeout = 1000)

// 测试方法期望得到的异常类，如果方法执行没有抛出指定的异常，则测试失败
@Test(expected = Exception.class)

// 执行测试时将忽略掉此方法，如果用于修饰类，则忽略整个类
@Ignore(“not ready yet”)

@Test
@RunWith
在JUnit中有很多个Runner，他们负责调用你的测试代码，每一个Runner都有各自的特殊功能，你要根据需要选择不同
的Runner来运行你的测试代码。
如果我们只是简单的做普通Java测试，不涉及Spring Web项目，你可以省略@RunWith注解，这样系统会自动使用默
认Runner来运行你的代码。

统一异常处理如下：
1、新建一个类
2、在class注解上@ControllerAdvice,
3、在公共方法上注解上@ExceptionHandler(value = Exception.class)

数据库的接入，大体步骤：
1、在路径src/main/resources下创建application.properties文件
2、在application.properties中加入datasouce的配置
3、在pom.xml加入mysql的依赖。
4、获取DataSouce的Connection进行测试。

默认静态资源处理:
静态资源访问优先级(静态资源默认都是在src/main/resources目录下)：META-INFO/resources > resources > static > public
如果想要自己完全控制WebMVC，就需要在@Configuration注解的配置类上增加
@EnableWebMvc（@SpringBootApplication 注解的程序入口类已经包含@Configuration），增加该注解以后
WebMvcAutoConfiguration中配置就不会生效，你需要自己来配置需要的每一项
绝对路径资源：registry.addResourceHandler("/api_files/**").addResourceLocations("file:D:/data/api_files");
@Configuration
public class MyWebConfiguration extends WebMvcConfigurerAdapter
{

	@Override
	public void addResourceHandlers(ResourceHandlerRegistry registry)
	{
		//新增jsp资源访问路径
		registry.addResourceHandler("/jsp/**").addResourceLocations("classpath:/jsp/");
		super.addResourceHandlers(registry);
	}

}

普通类调用Spring bean对象：
实现接口：ApplicationContextAware，然后加上@Component 注解即可
/**
 * 普通类调用Spring bean对象
 */
@Component
public class TestUtil implements ApplicationContextAware
{

	private static ApplicationContext applicationContext;

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException
	{
		if (TestUtil.applicationContext == null)
		{
			TestUtil.applicationContext = applicationContext;
		}
	}

	public static ApplicationContext getApplicationContext()
	{
		return applicationContext;
	}

	// 通过name获取Bean.
	public static Object getBean(String name)
	{
		return getApplicationContext().getBean(name);
	}

	// 通过class获取Bean.
	public static <T> T getBean(Class<T> clazz)
	{
		return getApplicationContext().getBean(clazz);
	}

	// 通过name,以及Clazz返回指定的Bean
	public static <T> T getBean(String name, Class<T> clazz)
	{
		return getApplicationContext().getBean(name, clazz);
	}

}

thymeleaf模板引擎:
application.properties常用配置如

THYMELEAF (ThymeleafAutoConfiguration)

spring.thymeleaf.prefix=classpath:/templates/

spring.thymeleaf.suffix=.html

spring.thymeleaf.mode=HTML5

spring.thymeleaf.encoding=UTF-8

 ;charset=<encoding> is added

spring.thymeleaf.content-type=text/html

关闭模板缓存，开发中需要关闭

spring.thymeleaf.cache=false

在spring boot中添加自己的Servlet有两种方法，代码注册Servlet和注解自动注册（Filter和Listener也是如此）:
一、代码注册通过ServletRegistrationBean、FilterRegistrationBean 和ServletListenerRegistrationBean 获得控制;也可以通过实现ServletContextInitializer 接口直接注册。
二、在SpringBootApplication上使用@ServletComponentScan注解后，Servlet、Filter、Listener 可以直接通过@WebServlet、@WebFilter、@WebListener 注解自动注册，无需其他代码。

项目服务启动任务：
通过实现接口CommandLineRunner来实现，可以通过@Order(value=1)注解（或者实现Order接口）来规定所有CommandLineRunner实例的运行顺序

新增自动扫描包或者类，在App.java（项目启动类）类中加入如下注解：
//可以使用：basePackageClasses={},basePackages={}
@ComponentScan(basePackages={"cn.kfit","org.kfit"})	--如果添加了此注解那么spring boot的默认扫描路径就会失效，此时如果需要应将默认扫描路径也添加上





