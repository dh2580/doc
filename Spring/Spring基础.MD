Spring框架基础
=========================
## 1、Spring 框架四大原则
>+ 使用 `POJO` 进行轻量级和最小侵入式开发 
>
>+ 通过依赖注入和基于接口编程实现松耦合
>
>+ 通过 `AOP` 和默认习惯进行 *声明式编程*
>
>+ 使用 `AOP` 和模板（`template`）减少模块化代码


## 2、控制反转（`IOC`）与依赖注入
>### **控制反转（`IOC`）**
>将对象的生命周期交给 `Spring` 容器来进行管理，不再由对象自己管理，比如对象的新建从 `new XXX()` 到通过声明为 `Spring` 容器中的一个 `Bean`，由 `Spring` 容器进行创建
>
>### **依赖注入**
>容器负责维护对象之间的依赖关系，而不是对象本身解决自己和其他对象之间的依赖关系


## 3、声明 `Bean` 的常用注解
>+ `@Component`（组件，没有明确的角色）
>+ `@Service` （在业务逻辑层，即**service** 层使用）
>+ `@Repository`（在数据访问层，即**dao** 层使用）
>+ `@Controller`(在展示层，即 `MVC` -> `Spring-MVC` 层使用)


## 4、注入 `Bean` 的常用注解
>+ `@Autowired`（**Spring** 提供的注解）
>+ `@Inject`（**JSR-330** 提供的注解）
>+ `@Resource`（**JSR-250** 提供的注解） 


## 5、`Bean` 的 `Scope`
>### **`SingleTon`**
>一个 `Spring` 容器中有且仅有一个 `Bean` 实例，默认配置
>
>### **`ProtoType`**
>每次调用都会新建一个 `Bean` 实例
>
>### **`Request`**
>`Web` 项目中，给每一个 `http request` 新建一个 `Bean` 实例
>
>### **`Session`**
>`Web` 项目中，给每一个 `http session` 新建一个 `Bean` 实例
>
>### **`GlobalSession`**
>`portal` 应用中，给每一个 `global http session` 新建一个 `Bean` 实例
>
>### **`@StepScope`**
>在 `Spring Batch` 中使用的一种 `Scope`

## 6、`Java` 配置示例
    @Configuration
    @ComponentScan("org.dh2580")
    public class JavaConfig{
        @Bean
        public XXX getXX(){
            ...
        }
    }


## 7、`AOP` 示例代码
    @Aspect
    @Component
    public class LogAspect{
        @Before("@annotation(org.dh2580.annotation.MethodLog)")
        public void before(JoinPoint jp){
            //todo log record
        }
    }
>在使用 `Spring AOP` 功能时，需要引入以下依赖包

    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
    </dependency>
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjweaver</artifactId>
    </dependency>


## 8、`Bean` 的初始化与销毁
>### `Java` 配置方式实现
    @Configuration
    @ComponentScan("org.dh2580")
    public class JavaConfig{
        @Bean(initMethod="init",destroyMethod="destory")
        public XXX getXX(){
            ...
        }
    }
>### 注解方式
    @Component
    public class XXX(){
        @PostConstruct
        public void init(){
            ...
        }
        
        @PreDestroy
        public void destroy(){
            ...
        }
        ...
    }


## 9、`Profile` 处理
>### 声明不同条件下的 `Bean`

    @Configuration
    @ComponentScan("org.dh2580")
    public class JavaConfig{
        @Bean
        @Profile("dev")
        public XXX getDevXXX(){
            ...
        }

        @Bean
        @Profile("prod")
        public XXX getProdXXX(){
            ...
        }
    }

>### `Profile` 设置
>
>+ 通过 `Environment` 的 `setActiveProfiles(string profiles)` 来设置
>
>+ 通过 `JVM` 的 `spring.profiles.active` 来设置
>
>+ `Web` 应用中通过 `Servlet` 的 `context parameter` 参数来设置

> 其中，`Web` 应用中，`Servlet2.5`， 以下如下所示

    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframewor.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>spring.profiles.active</param-name>
            <param-value>prod</param-value>
        </init-param>
    </servlet>

> 在 `Servlet3.0` 及其以上，如下所示
    
    public class WebInit implements WebApplicationInitializer{
        @Override
        public void onStartup(ServletContext container) throws ServletException{
            container.setInitParameter("spring.profiles.default","prod");
        }
    }


## 10、`SpringAware` 相关接口
+ `BeanNameAware`： 获取容器中该 `Bean`的名称

+ `BeanFactoryAware`： 获取当前的 `bean factory`

+ `ApplicationContextAware`： 获取当前的 `application context`

+ `ResourceLoaderAware`： 获取资源加载器，可以在当前 `Bean` 中实现资源加载


## 11、`Spring` 之多线程处理
1. 在配置类头，启用 `@EnableAsync` 标记

2. 配置类实现 `AsyncConfigurer` 接口

3. 在 `Bean` 中，对方法添加 `@Async` 注解

>配置类示例如下所示
    
    @Configuration
    @ComponentScan("org.dh2580")
    @EnableAsync
    public class JavaConfig implements AsyncConfigurer{
        @Override
        public Executor getAsyncExecutor(){
            ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
            taskExecutor.setCorePoolSize(5);
            taskExecutor.setMaxPoolSize(10);
            taskExecutor.setQueueCapacity(25);
            taskExecutor.initialize();
            return taskExecutor;
        }
    }

>`Bean` 示例如下

    @Component
    public class XXX(){
        @Async
        public void asyncMethod(){
            ...
        }
    }


## 12、`Spring` 之定时任务
1. 配置类中，启用 `@EnableScheduling` 标记

2. 对 `Bean` 中需要定时执行的方法，添加 `@Scheduled` 注解

>配置类示例如下所示
    
    @Configuration
    @ComponentScan("org.dh2580")
    @EnableScheduling
    public class JavaConfig{
        
    }

>`Bean` 示例如下

    @Component
    public class XXX(){
        //每 5 秒执行一次
        @Scheduled(fixRate = 5000)
        public void xxxMethod1(){
            ...
        }

        //每天 12 点执行一次
        @Scheduled(cron = "0 0 12 ? * *")
        public void xxxMethod2(){
            ...
        }
    }


## 13、`Spring` 之条件注解（`@Conditional`）
1. 创建一个条件类，实现 `Condition` 接口

2. 在 `Bean` 声明地方添加 `@Conditional` 注解

> 条件类示例如下

    public class XXXCondition implements Condition{
        @Override
        public boolean matches(ConditionContext context,AnnotatedTypeMetadata metadata){
            ...
            return true;
        }
    }

> `Bean` 声明如下所示

    @Configuration
    @ComponentScan("org.dh2580")
    public class JavaConfig{
        @Bean
        @Conditional(XXXCondition.class)
        public XXX getXXX(){
            ...
        }
    }


## 14、`Spring` 之单元测试
1. 添加 `spring-test` 、`junit` 依赖至项目
    
2. 在单元测试类中，添加标记注解 `@RunWith`、`@ContextConfiguration`

> 单元测试示例代码如下所示

    @RunWith(SpringJunit4ClassRunner.class)
    @ContextConfiguration(classes = {JavaConfig.class})
    @ActiveProfiles("dev")
    public class XXXTest {
        @Autowired
        private XXXBean xxx;

        @Test
        public void XXXMethodText(){
            ...
        }
    }


