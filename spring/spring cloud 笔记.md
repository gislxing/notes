# spring cloud 实战笔记

## 注册中心 Eureka server

    注册中心基本proterties配置如下：
        # 服务名称
        spring.application.name=eureka-service
        
        # 端口号
        server.port=8888
    
        # 注册中心实例地址
        eureka.instance.hostname=localhost
    
        # 不向注册中心注册自己
        eureka.client.register-with-eureka=false
    
        # 关闭服务检索功能即查询服务功能
        eureka.client.fetch-registry=false
    
        # 设置心跳检测频率
        eureka.server.renewal-percent-threshold=0.49
    
        # 关闭注册中心自我保护
        eureka.server.enable-self-preservation=false
    
        # 服务地址
      eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
    
    1、配置高可用注册中心
        1、新建spring boot项目，添加注解@EnableEurekaServer启用注册中心
        1、pom文件引入以下依赖
            <parent>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-parent</artifactId>
                <version>1.4.3.RELEASE</version>
            </parent>
    
            <dependencies>
                <dependency>
                    <groupId>org.springframework.cloud</groupId>
                    <artifactId>spring-cloud-starter-eureka-server</artifactId>
                    <version>1.2.6.RELEASE</version>
                </dependency>
            </dependencies>
        2、新建application-peer1.properties并添加以下配置
            spring.application.name=eureka-service
            server.port=8888
            #spring.profiles=peer1
    
            eureka.instance.hostname=peer1
            # 将 ip 地址注册到注册中心 
            eureka.instance.prefer-ip-address=true
            eureka.client.register-with-eureka=true
            eureka.client.fetch-registry=true
            eureka.client.serviceUrl.defaultZone=http://peer2:8889/eureka
            eureka.server.renewal-percent-threshold=0.49
        3、新建application-peer2.properties并添加以下配置
            spring.application.name=eureka-service
            server.port=8889
            #spring.profiles=peer2
    
            eureka.instance.hostname=peer2
            #eureka.instance.prefer-ip-address=true
            eureka.client.register-with-eureka=true
            eureka.client.fetch-registry=true
            eureka.client.serviceUrl.defaultZone=http://peer1:8888/eureka
            eureka.server.renewal-percent-threshold=0.49
        4、windows修改etc/hosts文件, linux修改etc/profile文件,新增以下配置
            127.0.0.1 peer1 peer2
        5、在命令行启动注册中心
            java -jar eureka-server-1.0-SNAPSHOT.jar --spring.profiles.active=peer1
            java -jar eureka-server-1.0-SNAPSHOT.jar --spring.profiles.active=peer2
    
    2、服务的注册与消费
        1、服务的发现由Eureka的客户端完成，服务消费由Ribbon完成（Ribbon是一个基于HTTP/TCP的客户端负载均衡器）
        2、创建服务提供者和消费者
            1、新建spring boot项目，并引入以下依赖
                <dependency>
                    <groupId>org.springframework.cloud</groupId>
                    <artifactId>spring-cloud-starter-eureka</artifactId>
                </dependency>
                <dependency>
                    <groupId>org.springframework.cloud</groupId>
                    <artifactId>spring-cloud-starter-ribbon</artifactId>
                </dependency>
                <dependency>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-web</artifactId>
                </dependency>
            2、启动主类添加注解@EnableDiscoveryClient以获取服务发现能力
            3、在主类中创建RestTemplate实例，并添加@LoadBalanced注解以开启负载均衡
                @Bean
                @LoadBalanced
                public RestTemplate restTemplate(){
                    return new RestTemplate();
                }
            4、创建controller并在其中使用上面创建的restTemplate调用服务提供者提供的接口
        3、服务续约
            // 定义每隔多少秒续约一次，默认30秒
            eureka.instance.lease-renewal-interval-in-seconds=30
            // 定义服务失效的时间，默认90秒
            eureka.instance.lease-expiration-duration-in-seconds=90
        4、整合健康监测spring boot actuator模块
            将客户端的健康监测交给spring-boot-starter-actuator模块的/health端点
                1、pom文件中引入(此处只需引入依赖即可开启spring boot的健康监测)
                    <dependency>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-starter-actuator</artifactId>
                    </dependency>
                2、(可选) application.properties文件中添加以下配置(此为将springcloud的健康监测交给actuator管理)
                    eureka.client.healthcheck.enabled=true
    
    3、多网卡环境下IP的选择
        1、忽略指定名称的网卡
            spring:
                cloud:
                    inetutils:
                        ignored-interfaces:
                            - eth0
                            - eth*
        2、指定使用的网络地址（使用正则）
            spring:
                cloud:
                    inetutils:
                        preferred-networks:
                            - 192.168
        3、使用本地地址
            spring:
                cloud:
                    inetutils:
                        use-only-site-local-interfaces: true
        4、手动指定IP地址
            eureka:
                instance:
                    prefer-ip-address: true
                    ip-address: 127.0.0.1
    4、自定义Ribbon配置
        1、java代码自定义配置
            1、创建Ribbon配置类并添加@Configuration注解
            2、编写负载均衡算法实现
                @Configuration
                public class RibbonConfiguration{
                    @Bean
                    public IRule ribbonRule(){
                        // 使用Ribbon提供的随机算法实现负载均衡
                        return new RandomRule();
                    }
                }
            3、为指定的服务自定义负载均衡算法
                @Configuration
                @RibbonClient(name="服务名称", configuration=RibbonConfiguration.class)
                public class TestConfiguration{}
        2、在application.yml配置文件中自定义Ribbon配置
            1、配置格式：
                <服务名称>：
                    ribbon：
                        属性: xxx
            2、支持的配置属性：
                NFLoadBalancerClassName: 配置ILoadBanlancer的实现类
                NFLoadBalancerRuleClassName: 配置IRule的实现类
                NFLoadBalancerPingClassName: 配置IPing的实现类
                NIWSServerListClassName: 配置ServerList的实现类
            3、例如配置simple-provider-user服务的负载均衡算法为随机:
                simple-provider-user:
                    ribbon:
                        NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
    5、脱离Eureka使用Ribbon，即服务不注册到注册中心或者没有注册中心
        1、引入Ribbon依赖
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-starter-ribbon</artifactId>
            </dependency>
        2、在application.yml配置文件中添加服务列表配置
            <服务名称>：
                    ribbon：
                        listOfServers: 服务地址1, 服务地址2, 服务地址3...
            例如：
                simple.provider.user:
                    ribbon:
                        listOfServers: http://localhost:8081, http://localhost:8082

2、使用Feign实现声明式REST调用
    1、整合Feign，引入Feign依赖
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-feign</artifactId>
        </dependency>
    2、创建接口class
        @FeignClient(name = "simple-provider-user")
        public interface UserFeignClient {
            @RequestMapping(value = "/user/{id}", method = RequestMethod.GET)
            User getUserInfo(@PathVariable("id") Long id);
        }
        说明：
            1、@FeignClient 此注解配置服务名称(name 属性)，也可使用url属性指定完整的服务地址
            例如：@FeignClient(name = "simple-provider-user", url="http://localhost:8080/")
            2、目前方法上只能使用@RequestMapping注解，不支持@GetMapping/@PostMapping等新的注解方式
            3、@PathVariable("id") 参数必须在括号中指定
            4、Feign是基于Ribbon实现的负载均衡，如果要使用Feign必须引入Ribboon依赖
    3、调用Feign接口
            @Autowired
            private UserFeignClient userFeignClient;

            @GetMapping("/user/{id}")
            public User getUserInfo(@PathVariable Long id){
                return this.userFeignClient.getUserInfo(id);
            }
    4、对压缩的支持
        在需要对请求/响应进行压缩的时候，可以使用以下配置开启Feign的压缩功能：
            feign:
                compression:
                    request:
                        enabled: true
                        # 用于支持压缩的类型列表, 默认支持text/xml, application/xml, application/json
                        mime-types: text/xml, application/xml, application/json
                        # 请求的最小值, 默认是2048
                        min-request-size: 2048
                    response:
                        enabled: true
    5、Feign日志
        为每个Feign客户端指定日志策略，每个Feign客户端都会创建一个logger，logger的名称是每个Feign接口的完整类名。
        注意：Fegin的日志打印只对debug级别的处理
        1、添加配置类
            @Configuration
            public class FeignLogConfiguration {
                @Bean
                Logger.Level feignLoggerLevel(){
                    return Logger.Level.BASIC;
                }
            }
        2、application.yml 配置文件中添加日志级别配置
            logging:
                level:
                    # 日志级别为debug，因为Feign只对debug级别的日志处理
                    com.zbs.cloud.simpleconsumermovie.feign.UserFeignClient: debug
    6、多参数请求的处理
        1、Get请求多参数, 地址多参数请求的处理(/user?id=1&username=李四)
            1、使用 @RequestParam("参数名称") 指定请求的参数
                // Feign客户端方法
                @RequestMapping(value = "/user", method = RequestMethod.GET)
                User getUserInfo(@RequestParam("id") Long id, @RequestParam("username") String username);
    
                // controller方法
                @GetMapping("/user")
                public User getUserInfo(@RequestParam("id") Long id, @RequestParam("username") String username){
                    return this.userFeignClient.getUserInfo(id, username);
                }
            2、使用Map构建请求的参数
                // Feign客户端方法
                @RequestMapping(value = "/user", method = RequestMethod.GET)
                User getUserInfo(@RequestParam Map<String, Object> map);
    
                // controller方法
                @GetMapping("/user")
                public User getUserMap(@RequestParam Long id){
                    Map<String, Object> map = new HashMap<>();
                    map.put("id", id);
                    return this.userFeignClient.getUserInfo(map);
                }
        2、POST请求包含多个参数
            // Feign 客户端方法
            @RequestMapping(value = "/user", method = RequestMethod.POST)
            User getUserInfo(@RequestBody User user);
    
            // controller 方法
            @PostMapping(value = "/user")
            public User getUser(@RequestBody User user){
                return this.userFeignClient.getUserInfo(user);
            }

3、Hystrix 容错处理
    1、容错的基本概念，容错机制必须满足以下两点：
        1、为网络请求设置超时
        2、使用断路器模式
    2、Hystrix 通过以下几点实现延迟和容错
        1、包裹请求：
            使用HystrixCommand / HystrixObservableCommand 包裹对依赖的调用逻辑，每个命令在独立的线程中执行
        2、跳闸机制：
            当某服务的错误率超过一定值时，Hystrix 可以自动或者手动跳闸，停止请求该服务一段时间
        3、资源隔离：
            Hystrix 为每个依赖都维护一个小型的线程池，如果该线程池已满，发往该依赖的请求就被立即拒绝，而不是排队等待。
        4、监控：
            Hystrix 可以近乎实时的监控运行指标和配置的变化
        5、回退机制：
            当请求失败、超时、被决绝，或者当断路器打开时，执行回退逻辑（此逻辑可自由定制）
        6、自我修复：
            断路器打开一段时间后，会自动进入"半开"状态。
    3、整合 Hystrix
        1、引入依赖 
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-starter-hystrix</artifactId>
            </dependency>
        2、启动类上添加注解 @EnableHystrix 开启Hystrix容错机制 
        （如果引入 Hystrix 依赖，则 Feign 就会默认使用 Hystrix 包裹所有客户端的请求，所以如果使用 Feign 了则 Feign 自动开启 Hystrix，
        如果想要在其他方法中使用 Hystrix 则此处的注解必须添加）
        3、修改 controller 让其中的方法具备容错能力，即在方法上添加注解 @HystrixCommand(fallbackMethod = "getUserInfoFallback")
            其中，getUserInfoFallback 标示在该被标注的方法调用失败时调用的方法(方法失败：请求失败 / 超时 / 被拒绝 / 异常等)
            此方法与当前的被注解标注的方法具有相同的返回值和参数
    4、Hystrix 的隔离策略
        1、Hystrix 的隔离策略有 THREAD / SEMAPHORE 两种，默认是 THREAD
        2、如果发生找不到上下文的异常时，可考虑将隔离策略设置为 SEMAPHORE，例如：
            @HystrixCommand(fallbackMethod = "getUserInfoFallback", commandProperties = {
                    @HystrixProperty(name = "execution.isolation.strategy", value = "SEMAPHORE")
            })
    5、Feign 使用 Hystrix
        1、基本用法
            1、编写回退类，实现Feign客户端，例如：
                @Component
                public class UserFeignClientFallback implements UserFeignClient
                {
                    // 省略实现类
                }
            2、给Feign客户端注解@FeignClient添加属性fallback指定回退时调用的类，例如：
                @FeignClient(name = "simple-provider-user", fallback = UserFeignClientFallback.class)
        2、输出回退原因
            1、编写回退类，实现 FallbackFactory 客户端，例如：
                @Component
                public class UserFeignClientFallbackFactory implements FallbackFactory<UserFeignClient> {

                    private static final Logger LOGGER = LoggerFactory.getLogger(UserFeignClientFallbackFactory.class);
    
                    @Override
                    public UserFeignClient create(Throwable t) {
                        final Throwable th = t;
    
                        return new UserFeignClient() {
                            @Override
                            public User getUserInfo(Long id) {
                                LOGGER.info("fallback reason: " + th.getCause());
                                return getUserInfoFallback(id);
                            }
    
                            @Override
                            public User getUserInfo(Map<String, Object> map) {
                                LOGGER.info("fallback reason: ", th);
                                return getUserInfoFallback(Long.parseLong(map.get("id").toString()));
                            }
                        };
                    }
    
                    private User getUserInfoFallback(Long id){
                        User user = new User();
                        user.setId(id);
                        user.setUsername("默认用例");
                        return user;
                    }
                }
            2、给Feign客户端注解 @FeignClient 添加属性 fallbackFactory 指定回退时调用的类，例如：
                @FeignClient(name = "simple-provider-user", fallbackFactory = UserFeignClientFallbackFactory.class)
                public interface UserFeignClient {
                    // ...
                }
        3、为 Feign 禁用 Hystrix
            只要 Hystrix 的依赖被引入，那么 Feign 就会默认使用 Hystrix 包裹所有客户端的请求，
            但是，在我们不需要使用 Hystrix 的时候就需要禁用
            1、为指定的 Feign 客户端禁用 Hystrix
                1、编写配置类：
                    @Configuration
                    public class FeignClientDisableHystrixConfiguration {
                        @Bean
                        @Scope("prototype")
                        public Feign.Builder feignBuilder(){
                            return Feign.builder();
                        }
                    }
                2、为需要禁用 Hystrix 的 Feign 客户端添加属性 configuration 并负值上面写的配置类，如：
                    @FeignClient(name = "simple-provider-user", configuration = FeignClientDisableHystrixConfiguration.class)
            2、全局禁用 Hystrix, 在 application.yml 配置文件中添加以下配置：
                hystrix.metrics.enabled: false

4、使用Zuul构建微服务网关
    Zuul 使用 Ribbon 来定位注册到 Eureka server 上的服务，同时还使用了 Hystrix 进行容错，
    所有经过 Zuul 的请求都会在 Hystrix 命令中执行。
    默认情况下，Zuul 会代理所有的注册到注册中心的服务，Zuul 的路由规则如下：
        http://Zuul_host:Zuul_port/微服务的artifactId/该服务映射路径
    1、编写简单 Zuul 服务网关
        1、新建 sping boot 项目
        2、引入 Zuul 依赖
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-starter-zuul</artifactId>
            </dependency>
        3、启动主类上添加注解 @EnableZuulProxy 开启网关
        4、编辑 application.yml 配置文件将该服务网关注册到注册中心
            spring:
                application:
                    name: simple-gateway-zuul
            server:
                port: 8000
            eureka:
                client:
                    serviceUrl:
                        defaultZone: http://localhost:8888/eureka
                instance:
                    prefer-ip-address: true
        5、启动服务后，访问地址 http://localhost:8000/simple-consumer-movie/user/2 即可访问到 simple-consumer-movie 服务
        6、访问地址 http://localhost:8000/routes 可查看到当前代理的所有路由列表
    2、Zuul 配置详解
        1、自定义指定微服务的访问路径
            在 application.yml 中新增以下配置如下：
                zuul:
                    routes:
                        simple-consumer-movie: /movie/**
            这样就把 simple-consumer-movie 微服务的访问路径映射到 /movie/**
        2、忽略指定微服务
            在配置文件 application.yml 添加配置以忽略指定的微服务(多个微服务以逗号分隔)：
                zuul:
                    ignored-services: simple-provider-user, simple-xxx-user
        3、忽略所有的微服务，只代理指定的微服务：
            在 application.yml 中添加以下配置如下：
                1、忽略所有的微服务：
                    zuul:
                        ignored-services: '*'
                2、为需要代理的微服务映射路径：
                    zuul:
                        routes:
                            simple-consumer-movie: /movie/**
        4、同时指定 serviceId 和 对应路径：
            此配置和 1 的配置功能相同
            zuul:
                routes:
                    user-routes: consumer-movie     # 给路由的名称，随意写
                        service-id: simple-consumer-movie
                        path: /movie/**     # service-id 对应的路径
        5、同时配置 path 和 url：
            以下配置将 http://ip:port/** 映射到 /movie/**
            zuul:
                routes:
                    user-routes: consumer-movie     # 给路由的名称，随意写
                        url: http://ip:port/
                        path: /movie/**     # url 对应的映射路径
            * 注意，此配置使 HystrixCommand 和 Ribbon 都不起作用
        6、同时配置 path 和 url，并且 Hystrix 和 Ribbon 也能起作用：
            在 application.yml 中的配置如下：
                zuul:
                    routes:
                        user-routes: consumer-movie     # 给路由的名称，随意写
                            service-id: simple-consumer-movie
                            path: /movie/**     # service-id 对应的路径
                ribbon:
                    eureka:
                        enabled: false
                simple-consumer-movie:      # 此名称即上面 service-id 指定的名称
                    ribbon:
                        listOfServers: http:/localhost:8080/, http:/localhost:8081/
        7、使用正则表达式指定 zuul 的路由匹配规则
            @Bean
            public PatternServiceRouteMapper serviceRouteMapper(){
                return new PatternServiceRouteMapper("需要匹配服务名称正则表达式"， "映射的路径正则表达式");
            }
        8、路由前缀
            1、访问 zuul 的 /api/simple-consumer-movie/1 路径，则请求会被映射到 simple-consumer-movie/api/1
                zuul:
                    prefix: /api
                    strip-prefix: false
                    routes:
                        simple-consumer-movie: /movie/**
            2、访问 zuul 的 /movie/1 路径，则请求会被映射到 simple-consumer-movie/movie/1
                zuul:
                    routes:
                        simple-consumer-movie:
                            path: /movie/**
                            strip-prefix: false
        9、忽略某些路径
            有时候需要让 zuul 代理服务，但是需要忽略该服务中的某些路径时，配置如下(忽略路径中包含/admin/的路径)：
                zuul:
                    routes:
                        simple-consumer-movie: /movie/**
                    ignored-patterns: /**/admin/**
    3、Zuul 的安全和 Header
        可通过以下配置为服务增加敏感 Header (下面的为默认配置)：
            zuul:
                routes:
                    simple-consumer-movie:
                        sensitive-headers: Cookie, Set-Cookie
        也可使用全局配置：
            zuul:
                sensitive-headers: Cookie, Set-Cookie
    4、使用 zuul 上传文件
        1、小于 1M 的文件上传无需任何处理
        2、大于 1M 的文件上传则需要为上传路径加上前缀 /zuul/, 也可使用 zuul.servlet-path 自定义前缀，
            例如，微服务 simple-file-upload 的文件上传为 /file/upload, 那么则可使用 /zuul/simple-file-upload/file/upload
            作为大文件的上传路径
        3、如果 zuul 使用了 Ribbon 的负载均衡，那么在上传大文件的时候根据需要设置 Ribbon 超时时间
    5、zuul 过滤器
        1、过滤器类型与请求生命周期 (除了默认的过滤器外，还可以自定义过滤器类型)
            zuul 中定义了以下4中典型过滤器类型：
            1、PRE:
                请求被路由之前调用 (可使用该过滤器实现身份验证、选择请求的微服务、记录调试信息等)
            2、ROUTING:
                将请求路由到微服务时执行 (用于构建发送给微服务的请求)
            3、POST：
                请求被路由到微服务之后执行 (可用来为响应添加 HTTP Header、将响应从微服务发送到客户端等)
            4、ERROR:
                路由的整个过程中发生错误时执行该过滤器
        2、编写自定义 zuul 过滤器
            1、需要继承抽象类 ZuulFilter 并实现其中的抽想方法：
                public class ProRequestLogFilter extends ZuulFilter {
                    public static final Logger LOGGER = LoggerFactory.getLogger(ProRequestLogFilter.class);
                    /**
                    * 返回过滤器的类型: pre / route / post / error / static
                    * @return
                    */
                    @Override
                    public String filterType() {
                        return "pre";
                    }

                    /**
                    * 该 int 值指定过滤器的执行顺序，不同的过滤器允许返回相同的值
                    * @return
                    */
                    @Override
                    public int filterOrder() {
                        return 1;
                    }
    
                    /**
                    * true: 执行该过滤器; false: 不执行该过滤器
                    * @return
                    */
                    @Override
                    public boolean shouldFilter() {
                        return true;
                    }
    
                    /**
                    * 过滤器的核心方法，具体的实现逻辑
                    * 此处实现的是在请求之前打印请求的日志信息
                    * @return
                    */
                    @Override
                    public Object run() {
                        RequestContext ctx = RequestContext.getCurrentContext();
                        HttpServletRequest request = ctx.getRequest();
                        LOGGER.info(String.format("发送 %s 请求到 %s", request.getMethod(), request.getRequestURL().toString()));
                        return null;
                    }
                }
            2、启动类中增加 Bean
                @Bean
                public ProRequestLogFilter proRequestLogFilter(){
                    return new ProRequestLogFilter();
                }
        3、禁用 zuul 过滤器
            如果想要禁用过滤器则只需在 application.yml 文件中新增如下格式的配置即可：
                zuul.<过滤器类名>.<过滤器类型>.disable = true
            例如，禁用上面编写的自定义过滤器：
                zuul.ProRequestLogFilter.pre.disable = true
    6、zuul 容错与回退
        zuul 默认集成 Hystrix
        为 zuul 添加容错的回退处理和比较简单，只需要实现 ZuulFallbackProvider 接口：
            @Component
            public class MovieFallbackProvider implements ZuulFallbackProvider {
    
                @Override
                public String getRoute() {
                    // 此回退类为那个微服务使用
                    return "simple-consumer-movie";
                }
    
                /**
                * 回退的响应
                * @return
                */
                @Override
                public ClientHttpResponse fallbackResponse() {
                    return new ClientHttpResponse() {
                        @Override
                        public HttpStatus getStatusCode() throws IOException {
                            // 响应码
                            return HttpStatus.OK;
                        }
    
                        @Override
                        public int getRawStatusCode() throws IOException {
                            // 数字类型的响应吗
                            return this.getStatusCode().value();
                        }
    
                        @Override
                        public String getStatusText() throws IOException {
                            // 返回状态的文本信息
                            return this.getStatusCode().getReasonPhrase();
                        }
    
                        @Override
                        public void close() {
    
                        }
    
                        @Override
                        public InputStream getBody() throws IOException {
                            // 响应体
                            return new ByteArrayInputStream("微服务不可用，请稍后重试".getBytes());
                        }
    
                        @Override
                        public HttpHeaders getHeaders() {
                            // 响应 Header
                            HttpHeaders headers = new HttpHeaders();
                            MediaType mediaType = new MediaType("application", "json", Charset.forName("utf-8"));
                            headers.setContentType(mediaType);
                            return headers;
                        }
                    };
                }
            }
    7、zuul 高可用
        1、zuul 客户端也注册到注册中心
            此情况只需要部署多个 zuul 服务并注册注册中心即可
        2、zuul 客户端没有注册到注册中心
            此时需要使用 Nginx 等负载均衡器来实现对 zuul 的负载均衡

5、统一管理微服务配置 spring cloud config
    集中管理配置、不同服务不同配置、运行期间可动态调整、配置修改后可自动更新
    spring cloud config 分为 config server 和 config client 两部分
    默认使用GIT存储配置内容(也可使用svn、本地磁盘等存储)
    在其他微服务启动时，会请求 config server 以获取所需要的配置属性，然后缓存到本地
    1、编写 config server
        1、新建 GIT 仓库，用于存放配置文件
        2、新建4个文件及内容，并提交到 config-v1.0 分支
            simple-service-foo.properties
                profile=default-1.0
            simple-service-foo-dev.properties
                profile=dev-1.0
            simple-service-foo-test.properties
                profile=test-1.0
            simple-service-foo-production.properties
                profile=production-1.0
        3、修改上面4个文件内容，并提交到 config-v2.0 分支
            将 1.0 修改为 2.0
        4、新建 spring boot web 服务 config-server
        5、POM 文件中添加以下依赖：
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-config-server</artifactId>
            </dependency>
        6、启动类添加 @EnableConfigServer
        7、配置文件 application.yml 中添加以下配置如下：
            server:
                port: 8083
            spring:
                application:
                    name: config-server
                cloud:
                    config:
                        server:
                            git:
                                uri: http://zhangbinshan@10.190.169.100:9999/r/spring-cloud-config.git
                                username: zhangbinshan
                                password: zhangbinshan123
        8、config server 将 GIT 上的各分支的配置文件的映射端点规则如下：
            1、/{application}/{profile}[/{branch}]
            2、/{application}-{profile}.yml
            3、/{branch}/{application}-{profile}.yml
            4、/{application}-{profile}.properties
            5、/{branch}/{application}-{profile}.properties
            配置属性说明：
                {application}: 微服务的名称，此处应该理解为文件名称最后一个 - 之前的文件名称
                {profile}: 配置文件名称最后一个 - 之后的名称
                {branch}: GIT 的分支，如果不写则默认为 master 分支
        9、启动 config-server 服务，访问 http://localhost:8083/config-v1.0/simple-service-foo-dev.properties
            则可以访问到 config-v1.0 分支中 simple-service-foo-dev.properties 配置文件的属性
    2、本地配置文件
        application.yml 添加以下配置属性
            spring: 
                profiles:
                    active: native # 使用本地配置文件
                cloud:
                    config:
                        server:
                            native:
                                search-locations: file:///${user.dir}/configurations/   # 本地配置文件存储位置
            在Windows中，如果文件URL为绝对驱动器前缀，例如 file:///${user.home}/config-repo，则需要额外的“/”。
    3、编写 config client
        
        
