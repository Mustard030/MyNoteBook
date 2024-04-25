
# SpringBoot的各种注解
## 1. 组件注册
### @Configuration、@SpringBootConfiguration
这两个注解本质是一样的，`@SpringBootConfiguration`里面也有`@Configuration`，但是语义可以为Springboot自身的Configuration类

### @Bean、@Scope、@Lazy
Bean是作用在有`@Configuration`注解的类的**方法**上的，它允许程序员详细定义作为Bean对象的生成的方法，默认**方法名**即为Bean名称
并且可以通过配合`@Scope`限定Bean的作用域
1. ==Singleton==：这是**默认**的单例模式，意味着在整个应用程序生命周期中，无论如何调用，始终只有一个实例被创建和维护。
2. ==Prototype==：这是一个多例模式，每次请求时都会生成新的实例。
3. Request：这种模式下，每个HTTP请求都会产生一个新的Bean实例，并且这个实例只在当前请求内有效。
4. Session：在这种模式下，每个HTTP请求也会产生一个新的Bean实例，但这个实例只在当前HTTP session内有效。
5. GlobalSession：类似于标准的HTTP Session作用域，但它仅在基于Portlet的Web应用中有意义。

**Spring 只帮我们管理单例模式 Bean 的完整生命周期，对于 prototype 的 bean ，Spring 在创建好交给使用者之后则不会再管理后续的生命周期。**
@Lazy只针对单例Bean，在第一次使用才加载到容器中


### @Autowired, @Resource, @Qualifier
`@Autowired`可以标注在构造器，方法，参数，成员变量，注解上，先ByType再ByName

`@Qualifier`可以指定要注入的bean的名字，`@Autowired`和`@Qualifier`一起使用等于`@Resource(name="xxx")`

`@Resource`是先ByName再ByType
- 成员变量：此时先根据你的类型找到所有这个类型的Bean（ByType），如果只有一个这个类型的Bean则直接赋值，否则根据变量名（ByName）找到对应名字的Bean
- 任意非static方法：根据入参的信息找到一个Bean赋值
- 构造方法：有多个构造方式时，构造Bean的时候会尝试使用无参构造方法，如果没有无参构造方法则报错，除非在其中一个构造方法上加上`@Autowired`，只有一个构造方法时就用那个
- 注解：定义自己的注解的时候用，相当于你的注解带有了`@Autowired`的功能
- 方法参数：没用，spring framework会直接忽略，除了一种情况就是在JUnit的Jupiter测试支持上
其实还能装配多个实例，比如现在有几个Service实现了
```java
@Autowired 
private List<IUser> userList; 

@Autowired 
private Set<IUser> userSet; 

@Autowired 
private Map<String, IUser> userMap;

```

### @Controller、@Service、@Repository、@Component
本质都是`@Component`注解不过是语义上的不同，默认注册的Bean名称为类名首字母小写


### @Import
因为第三方类就算写上`@Configuration`也没法被主类包扫描扫到，所以使用`@Import`把第三方的包里的类导入进来

### @ComponentScan
用来代替配置文件中的 `component-scan` 配置，开启组件扫描，即自动扫描包路径下的 `@Component` 注解进行注册 bean 实例到 context 中。


## 2. 条件注解
### @Conditional({Condition})
这是 Spring 4.0 添加的新注解，用来标识一个 Spring Bean 或者 Configuration 配置文件，当满足指定的条件才开启配置。不用标注这个注解，直接使用下面的就行
要实现自定义的`Conditional`，需要自定义一个类并且实现`Condition`接口，重写`matches`方法其中的`context`是上下文，`AnnotatedTypeMetaData`是注释信息


### @ConditionOnXxx
下列注解都与 `@Conditional` 注解组合

- @ConditionalOnBean：当容器中有指定的 Bean 才开启配置。
- @ConditionalOnMissingBean：和 `@ConditionalOnBean` 注解相反，当容器中没有指定的 Bean 才开启配置。

- @ConditionalOnClass：当容器中有指定的 Class 才开启配置。
- @ConditionalOnMissingClass：和 @ConditionalOnMissingClass 注解相反，当容器中没有指定的 Class 才开启配置。

- @ConditionalOnWebApplication：当前项目类型是 WEB 项目才开启配置。
	当前项目有以下 3 种类型。
```java
enum Type {
	/**
	 * Any web application will match.
	 */
	ANY,
	
	/**
	 * Only servlet-based web application will match.
	 */
	SERVLET,
	
	/**
	 * Only reactive-based web application will match.
	 */
	REACTIVE
}
```
- @ConditionalOnNotWebApplication：和 @ConditionalOnWebApplication 注解相反，当前项目类型不是 WEB 项目才开启配置。

- @ConditionalOnProperty：当指定的属性有指定的值时才开启配置。
- @ConditionalOnExpression：当 SpEL 表达式为 true 时才开启配置。
- @ConditionalOnJava：当运行的 Java JVM 在指定的版本范围时才开启配置。
- @ConditionalOnResource：当类路径下有指定的资源才开启配置。
- @ConditionalOnJndi：当指定的 JNDI 存在时才开启配置。
- @ConditionalOnCloudPlatform：当指定的云平台激活时才开启配置。
- @ConditionalOnSingleCandidate：当指定的 class 在容器中只有一个 Bean，或者同时有多个但为首选时才开启配置。


## 3. 属性绑定
### @ConfigurationProperties、@EnableConfigurationProperties
将容器中任意组件(Bean)的属性值和配置文件的配置项的值进行绑定
比如配置在类上:
```java
@Data
@Component  //首先放到容器中
@ConfigurationProperties("myconfig") //再使用这个注解指定前缀
public class MyConfigurationProperties {
	private Long id;
	private String name;
}
```
或使用在Bean的方法上
```java
@Configuration
public class AppConfig {
	@Bean
	@ConfigurationProperties("myconfig") //再使用这个注解指定前缀
	public XX xx(){
			return new XX();//我们自己new新的对象
	}
}
```
只要配置文件里的键值能和上面的属性名对应上就能绑定
```yml
myconfig:  
  id: 1  
  name: "sda"
```

使用上面这个方法需要先注册容器，但是使用`@EnableConfigurationProperties`则不需要
假设还是这个类
```java
@Data
//@Component  //现在不放到容器中了
@ConfigurationProperties("myconfig") //还是使用这个注解指定前缀
public class MyConfigurationProperties {
	private Long id;
	private String name;
}
```
则需要在配置类上使用
```java
@EnableConfigurationProperties(MyConfigurationProperties.class)
@Configuration
public class AppConfig {
	...
}
```
并且会默认把填写的这个组件自己放到容器中
**其实主要用法还是在第三方类上用，原因与Import注解相同**，相当于Import+ConfigurationProperties



# SpringBoot自动配置原理
在许多的Starter中都引入了`spring-boot-starter`，而这个包又引入了`spring-boot-autoconfigure`，又因为这个包里囊括了所有场景的所有配置，只要这个包生效，相当于所有场景的Springboot官方写好的整合功能就生效了
但是由于主类注解`@SpringBootApplication`默认只扫描当前类所在的包及其子包，故也扫描不到这些配置类，因此现在只要能扫描到这些配置的小包，这些小包的配置就能生效
关键就在于`@SpringBootApplication`中的`@EnableAutoConfiguration`即开启自动配置的核心，`@EnableAutoConfiguration`这个注解里有`@Import(AutoConfigurationImportSelector.class)`提供功能：批量给容器中导入组件
这里面核心方法是`getAutoConfigurationEntry()`，其中的`getCandidateConfigurations()`中会有提示，这些类是从`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`文件来的
项目启动时利用`@Import()`批量导入机制把`spring-boot-autoconfigure`包下的那个imports文件写上的那些自动配置类（XxxxAutoConfiguration）自动导入进来
但是这些配置类不是全部生效的，他们都有一些`@ConditionalOnXxx`条件来控制只有导入了他们的依赖才会生效
XxxAutoConfiguration这个类里面放了很多Bean注册到容器中，每个自动配置类都可能有`@EnableConfigurationProperties(XXProperties.class)`，用来把配置文件中配的指定前缀的属性值封装到`XXProperties`属性类中，属性类又通过`@ConfigurationProperties`与配置文件的对应前缀绑定，这样给容器中放的所有组件的一些核心参数，都来自于`XXProperties`，这就把配置文件和核心组件的底层参数值联系起来了

核心流程：
1. 导入`starter`，就会导入`autoconfigure`包
2. `autoconfigure`包里面有个文件`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`，里面指定所有启动要加载的自动配置类
3. `@EnableAutoConfiguration`会把上面文件里写的所有自动配置类都导入进容器，`XxxxAutoConfiguration`是有条件注解进行按需加载
4. `XxxxAutoConfiguration`给容器中导入一堆Bean，并且这些Bean都是从`XXProperties`类中提取属性值
5. `XXProperties`又通过`@ConfigurationProperties`和配置文件进行绑定


# Bean的生命周期
即Bean的创建->初始化->销毁的过程
Bean的创建和销毁方法有4种定义方法：
1. 指定Init方法和Destroy方法使用`@Bean(initMethod="init", destroyMethod="destroy")`
2. 通过让单例Bean实现`InitializingBean`或`DisposableBean`来定义初始化或者销毁逻辑
3. 使用`@PostConstruct`和`@PreDestroy`注解在类方法上定义初始化或者销毁逻辑
4. 使用一个**自定义类**实现`BeanPostProcessor`接口，并注册为**组件**：Bean的后置处理器，在Bean初始化前后进行一些处理工作，注意这个连**容器自身的Bean也会调用**，在Spring底层大量的用到了这个接口
	postProcessBeforeInitialization：在创建实例后，在任何初始化方法执行之前
	postProcessAfterInitialization：在任何初始化方法执行之后
	参数为bean实例本体和beanName字符串
	这两个方法的返回值可以直接返回Bean本体，也可以包装后再返回
```java
	@Component  //加入容器
	public class MyBeanPostProcessor implements BeanPostProcessor {  
	    @Override  
	    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {  
	        System.out.println("MyBeanPostProcessor postProcessBeforeInitialization");  
	        return bean;  
	    }  
	  
	    @Override  
	    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {  
	        System.out.println("MyBeanPostProcessor postProcessAfterInitialization");  
	        return bean;  
	    }  
}
```

# AOP切面
主要目的是在程序运行时动态地将某段代码切入到指定方法的指定位置进行运行的编程方式，用到了代理模式
这些注解的参数为：`访问权限 返回值类型 全类名.方法名(参数类型...)`，其中方法名可以用\*代替所有方法，参数类型可以用(..)代替所有参数类型
通知方法有：
- 前置通知(`@Before()`)
- 后置通知(`@After`)
- 返回通知(`@AfterReturning`)
- 异常通知(`@AfterThrowing`)
- 环绕通知(`@Around`): 需要手动执行`joinPoint.procced()`
执行顺序为：前置通知->目标方法->后置通知->（正常执行：返回通知 / 出现异常：异常通知）
可以给出一个方法标注`@Pointcut("execution(原本的切面参数参数)")`，之后同类的方法就可以直接使用这个`方法名()`作为通知方法的参数，相当于起个别名，别的类想用这个`Pointcut`的方法就要使用`全类名.方法名()`
在切面类上加`@Aspect`表明这是切面类，再加上`@Component`注册到容器，并在配置类中加入`@EnableAspectJAutoProxy`开启基于注解的aop模式

核心在于`@EnableAspectJAutoProxy`这个注解，它导入了`@Import({AspectJAutoProxyRegistrar.class})`，利用`AspectJAutoProxyRegistrar`自定义给容器中注册Bean

**AOP失效情况：**
AOP代理通常在方法调用的外部进行拦截和增强，但对于同一个类内部的方法调用，由于绕过了代理对象，AOP增强可能会失效。为了解决这个问题，可以使用AopContext.currentProxy()方法来获取当前对象的AOP代理。AopContext.currentProxy()方法是Spring提供的一个静态方法，用于获取当前线程中正在执行的方法所属的代理对象。通过这个方法，可以在同一个类的方法中调用另一个方法，并确保AOP增强仍然生效。

```java
	//在使用本类方法时，不能直接调用
	UserService proxyUserServiceImpl = (UserService) AopContext.currentProxy();
	//然后再通过proxyUserServiceImpl调用本类其他方法
	proxyUserServiceImpl.saveUser(user);
```

在Springboot2.7以前，可以通过本类注入本类来使用本类中被增强的另一个方法，但在2.7版本后本类循环依赖会直接报错，建议直接使用上述方法。AopContext.currentProxy()的实现原理为ThreadLocal。


# SpringIOC容器的创建
核心在于`refresh()`
```java
public void refresh() throws BeansException, IllegalStateException {  
    this.startupShutdownLock.lock();  
    try {  
       this.startupShutdownThread = Thread.currentThread();  
  
       StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");  
  
       // Prepare this context for refreshing.  
       //刷新前的预处理
       prepareRefresh();  
  
       // Tell the subclass to refresh the internal bean factory.  
       //获取bean工厂
       // 1.刷新(创建)bean工厂，并设置一个序列化id
       // 2.拿到刚刚创建的bean工厂
       ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
  
       // Prepare the bean factory for use in this context.  
       //bean工厂的预准备工作
       // 1.设置beanFactory的类加载器，支持表达式解析器...
       // 2.添加部分BeanPostProcessor
       // 3.添加编译时的AspectJ支持
       // 4.添加一些组件如environment，systemProperties，systemEnvironment
       prepareBeanFactory(beanFactory);  
  
       try {  
          // Allows post-processing of the bean factory in context subclasses.  
          //准备工作完成后的后置处理
          //子类通过重写这个方法来在BeanFactory创建并预准备完成后做进一步的工作
          postProcessBeanFactory(beanFactory);  
          
===============================以上是BeanFactory的创建及预准备工作========================================
  
          StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");  


          //执行Bean工厂后置处理器（BeanFactoryPostProcessor）的方法，在BeanFactory标准初始化之后执行
          //这里找到实现了BeanFactoryPostProcessor，BeanDefinitionRegistryPostProcessor两个接口的Bean
          //先执行BeanDefinitionRegistryPostProcessor的方法：
          //  获取所有的BeanDefinitionRegistryPostProcessor，先执行实现了PriorityOrdered的，再执行实现了Ordered，最后执行其他优先级或顺序接口的
          //再执行BeanFactoryPostProcessor的方法，与上相同
          invokeBeanFactoryPostProcessors(beanFactory);  

===============================以上是BeanFactory的后置处理工作========================================
=====================================下面开始创建Bean的后置处理器======================================

          // 注册Bean的后置处理器，用于拦截Bean的创建过程
          registerBeanPostProcessors(beanFactory);  
          beanPostProcess.end();  
  
          // 用于国际化配置
          initMessageSource();  
  
          //  定义事件发布器，  
          initApplicationEventMulticaster();  
  
          // 启动web服务器Tomcat
          onRefresh();  
  
          // 查找监听器Bean放到事件发布器中
          registerListeners();  
  
          // 生产所有的单例Bean放入单例池
          //1.构造对象 实例化 通过反射创建一个对象，并放入三级缓存中
          //2.填充属性
          //3.初始化实例
          //4.构造对象
          finishBeanFactoryInitialization(beanFactory);  
  
          //构造并注册生命周期管理器lifecycleProcessor并调用所有实现了生命周期接口Lifecycle的Bean中的start方法
          //最后发布一个容器刷新完成事件
          finishRefresh();  
       }  
       catch (RuntimeException | Error ex ) {  
          if (logger.isWarnEnabled()) {  
             logger.warn("Exception encountered during context initialization - " +  
                   "cancelling refresh attempt: " + ex);  
          }  
          // Destroy already created singletons to avoid dangling resources.  
          destroyBeans();  
  
          // Reset 'active' flag.  
          cancelRefresh(ex);  
  
          // Propagate exception to caller.  
          throw ex;  
       }  
       finally {  
          contextRefresh.end();  
       }    }    finally {  
       this.startupShutdownThread = null;  
       this.startupShutdownLock.unlock();  
    }}
```


![[Pasted image 20240310161039.png]]

对于循环依赖
![[Pasted image 20240310161303.png]]



