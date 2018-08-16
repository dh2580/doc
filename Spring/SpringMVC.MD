Spring MVC
====================
## 1、Spring MVC `java` 类型配置
>+ 项目路径中引入以下依赖项目

    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
    </dependency>
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>jstl</artifactId>
    </dependency>
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>javax.servlet.jsp</groupId>
        <artifactId>jsp-api</artifactId>
        <scope>provided</scope>
    </dependency>

>+ 新建项目配置类，并配置 `ViewResolver`

    @Configuration
    @EnableWebMvc
    @Componscan("org.code.dh2580")
    public class JavaConfig{
        @Bean
        public InternalResourceViewResolver viewResolver(){
            InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
            viewResolver.setPrefix("/WEB-INF/classes/views/");
            viewResolver.setSuffix(".jsp");
            viewResolver.setViewClass(JstlView.class);
            return viewResolver;
        }
    }

>+ 新建 `Web` 应用初始化类，实现 `WebApplicationInitializer` 接口，此类的功能相当于 `web.xml` 的功能

    public class WebInitializer implements WebApplicationInitializer {
        @Override
        public void onStartUp(ServletContext servletContext) throws ServletException{
            AnnotationConfigWebApplicationContext ctx = new AnnotationConfigWebApplicationContext();
            ctx.register(JavaConfig.class);
            ctx.setServletContext(servletContext);
            
            ServletRegistration.Dynamic dynamic = servletContext.addServlet("dispatcher", new DispatcherServlet(context));
            dynamic.addMapping("/");
            dynamic.setLoadOnStartup(1);
        }
    }

或者继承 `AbstractAnnotationConfigDispatcherServletInitializer` 也可以

    public class WebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
        @Override
        protected Class<?>[] getRootConfigClasses() {
            return new Class[]{JavaConfig.class};
        }

        @Override
        protected Class<?>[] getServletConfigClasses() {
            return new Class[]{JavaConfig.class};
        }

        @Override
        protected String[] getServletMappings() {
            return new String[]{"/"};
        }
    }

>+ 编写 `Controller` 

    @Controller
    public class SimpleController{
        @RequestMap("/index)
        public String index(){
            return "index";
        }
    }

## 2、Spring MVC 实现返回 `json` 和 `XML` 类型数据
### a、 返回 `json` 类型数据
>+ 引入 `jackson-databind` 依赖

    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>

>+ 编写 `Controller`

    @Controller
    public class SimpleController(){
        @RequestMapping(value = "/getPerson", produces = {"application/json;charset=UTF-8"})
        @ResponseBody
        public SimpleObject getSimpleObject(){
            return new SimpleObject(...);
        }
    }

### b、返回 `json` 和 `xml` 格式数据
>+ 引入 `jackson-dataformat-xml` 依赖

    <dependency>
        <groupId>com.fasterxml.jackson.dataformat</groupId>
        <artifactId>jackson-dataformat-xml</artifactId>
    </dependency>

>+ 编写 `Controller`

    @RestController
    public class SimpleController(){
        @RequestMapping(value = "/getPerson", produces = {"application/json;charset=UTF-8"})
        public SimpleObject getSimpleObjectAsJson(){
            return new SimpleObject(...);
        }

         @RequestMapping(value = "/getPerson", produces = {"application/xml;charset=UTF-8"})
        public SimpleObject getSimpleObjectAsXml(){
            return new SimpleObject(...);
        }
    }

## 3、 `WebMvcConfigurerAdapter` 之静态资源映射

    @Configuration
    @EnableWebMvc
    @Componscan("org.code.dh2580")
    public class JavaConfig extends WebMvcConfigurerAdapter{
        @Bean
        public InternalResourceViewResolver viewResolver(){
            InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
            viewResolver.setPrefix("/WEB-INF/classes/views/");
            viewResolver.setSuffix(".jsp");
            viewResolver.setViewClass(JstlView.class);
            return viewResolver;
        }

        //添加静态资源映射配置
        @Override
        public void addResourceHandlers(ResourceHandlerRegistry registry){
            registry.addResourceHandler("/assets/**")
                    .addResourceLocations("classpath:/assets/");
        }
    }

## 4、 `WebMvcConfigurerAdapter` 之全局拦截器

    @Configuration
    @EnableWebMvc
    @Componscan("org.code.dh2580")
    public class JavaConfig extends WebMvcConfigurerAdapter{
        @Bean
        public InternalResourceViewResolver viewResolver(){
            InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
            viewResolver.setPrefix("/WEB-INF/classes/views/");
            viewResolver.setSuffix(".jsp");
            viewResolver.setViewClass(JstlView.class);
            return viewResolver;
        }

        //添加全局拦截器
        @Override
        public void addInterceptors(InterceptorRegistry registry){
            registry.addInterceptor(new HandlerInterceptorAdapter(){
                @Override
                public boolean preHandle(HttpServletRequest req,HttpServletResponse resp,Object handler) throws Exception{
                    //todo ...

                    return true;    //返回 true 表示继续走下去，返回 false 表示直接返回
                }
                
                @Override
                public void postHandle(HttpServletRequest req,HttpServletResponse resp,Object handler,ModelAndView modelAndView){
                    //todo ...
                }
            });
        }
    }


## 4、 `WebMvcConfigurerAdapter` 之添加 `Controller`

    @Configuration
    @EnableWebMvc
    @Componscan("org.code.dh2580")
    public class JavaConfig extends WebMvcConfigurerAdapter{
        @Bean
        public InternalResourceViewResolver viewResolver(){
            InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
            viewResolver.setPrefix("/WEB-INF/classes/views/");
            viewResolver.setSuffix(".jsp");
            viewResolver.setViewClass(JstlView.class);
            return viewResolver;
        }

        //添加controller
        @Override
        public void addViewControllers(ViewControllerRegistry registry) {
            registry.addViewController("/index")    //设置映射路径
                    .setViewName("/index");         /设置返回视图
        }
    }

## 5、 Spring MVC 之全局处理配置
>+ 示例 `ControllerAdvice`

    @ControllerAdvice
    public class GlobalHandleConfigAdvice {
        @ExceptionHandler(value = Exception.class)  //表示具体的异常类型
        public ModelAndView gloablException(Exception ex,WebRequest req) {
            //表示跳转至error jsp页面
            ModelAndView modelAndView = new ModelAndView("error); 
            //视图附带属性，以便在error jsp页面中使用 ${errorMsg}
            modelAndView.addObject("errorMsg",ex.getMessage());

            return modelAndView;
        }

        @ModelAttribute
        public void addAttributes(Model model){
            model.addAttribute("msg","额外属性");
        }

        @InitBinder
        public void initBinder(WebDataBinder webDataBinder) {
            webDataBinder.setDisallowFields("id","schema);
        }
    }

>+ 示例 `Controller`

    @Controller
    public class SimpleController {
        @RequestMapping("/advice")
        public String doSomeThing(@ModelAttribute("msg") String msg) throws Exception{
            throw new Exception("Error Occured From " + msg);
        }
    }

## 6、 Spring MVC 之异步方式处理（仅限定 `Servlet 3.0` 以上）
>+ 开启异步方法支持

    public class WebInitializer implements WebApplicationInitializer {
        @Override
        public void onStartUp(ServletContext servletContext) throws ServletException{
            AnnotationConfigWebApplicationContext ctx = new AnnotationConfigWebApplicationContext();
            ctx.register(JavaConfig.class);
            ctx.setServletContext(servletContext);
            
            ServletRegistration.Dynamic dynamic = servletContext.addServlet("dispatcher", new DispatcherServlet(context));
            dynamic.addMapping("/");
            dynamic.setLoadOnStartup(1);
            //开启异步方法支持
            dynamic.setAsyncSupported(true);
        }
    }

>+ 编写返回异步结果的 `Controller` 方法

    @RestController
    public class AsyncController {
        @RequestMapping("/defer")
        public DeferredResult<String> deferredMethod(){
            //注意此处返回结果要使用 Deferred<T> 包装类进行包装
            DeferredResult<String> res = new DeferredResult<String>();
            res.setResult(System.currentTimeMillis() + "");
            return res;
        }
    }