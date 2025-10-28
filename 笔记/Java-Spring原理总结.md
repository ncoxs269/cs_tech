2025-10-28 11:29
Status: #idea
Tags: [[Java]] [[Spring]]

# 1 Spring 容器高层视图

Spring 启动时读取应用程序提供的 Bean 配置信息，并在 Spring 容器中生成一份相应的 Bean 配置注册表，然后根据这张注册表实例化 Bean，装配好 Bean 之间的依赖关系，为上层应用提供准备就绪的运行环境。
![[image-178.png]]
Bean 缓存池：`HashMap` 实现。

# 2 IOC 容器介绍

Spring 通过一个配置文件描述 Bean 及 Bean 之间的依赖关系，利用 Java 语言的反射功能实例化 Bean 并建立 Bean 之间的依赖关系。 Spring 的 IoC 容器在完成这些底层工作的基础上，还提供了 Bean 实例缓存、生命周期管理、 Bean 实例代理、事件发布、资源装载等高级服务。

- `BeanFactory` 是 Spring 框架的基础设施，面向 Spring 本身；
    
- `ApplicationContext` 面向使用 Spring 框架的开发者，几乎所有的应用场合我们都直接使用 `ApplicationContext` 而非底层的 `BeanFactory`。
    

## 2.1 BeanFactory

### 2.1.1 体系架构

BeanFactory 体系架构：
![[image-179.png]]
- **BeanDefinitionRegistry**： Spring 配置文件中每一个 `<bean>` 节点元素在 Spring 容器里都通过一个 `BeanDefinition` 对象表示，它描述了 Bean 的配置信息。而 `BeanDefinitionRegistry` 接口提供了向容器手工注册 `BeanDefinition` 对象的方法。
    
- **BeanFactory** 接口位于类结构树的顶端 ，它最主要的方法就是 `getBean(String beanName)`，该方法从容器中返回特定名称的 Bean，`BeanFactory` 的功能通过其他的接口得到不断扩展：
    
- **ListableBeanFactory**：该接口定义了访问容器中 Bean 基本信息的若干方法，如查看Bean 的个数、获取某一类型 Bean 的配置名、查看容器中是否包括某一 Bean 等方法；
    
- **HierarchicalBeanFactory**：父子级联 IoC 容器的接口，子容器可以通过接口方法访问父容器； 通过 `HierarchicalBeanFactory` 接口， Spring 的 IoC 容器可以建立父子层级关联的容器体系，**子容器可以访问父容器中的 Bean，但父容器不能访问子容器的 Bean**。Spring 使用父子容器实现了很多功能，比如在 Spring MVC 中，展现层 Bean 位于一个子容器中，而业务层和持久层的 Bean 位于父容器中。这样，展现层 Bean 就可以引用业务层和持久层的 Bean，而业务层和持久层的 Bean 则看不到展现层的 Bean。
    
- **ConfigurableBeanFactory**：是一个重要的接口，增强了 IoC 容器的可定制性，它定义了设置类装载器、属性编辑器、容器初始化后置处理器等方法；
    
- **AutowireCapableBeanFactory**：定义了将容器中的 Bean 按某种规则（如按名字匹配、按类型匹配等）进行自动装配的方法；
    
- **SingletonBeanRegistry**：定义了允许在运行期间向容器注册单实例 Bean 的方法；
    

### 2.1.2 例子

使用 Spring 配置文件为 `Car` 提供配置信息：beans.xml：

```xml
<?xml version="1.0" encoding="UTF-8" ?>   
<beans xmlns="Index of /schema/beans"   
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"   
    xmlns:p="http://www.springframework.org/schema/p"   
    xsi:schemaLocation="Index of /schema/beans   
    http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">   
      
    <bean id="car1" class="com.baobaotao.Car"   
        p:brand="红旗CA72"   
        p:color="黑色"   
        p:maxSpeed="200" />   
</beans>
```

通过 `BeanFactory` 装载配置文件，启动 Spring IoC 容器：

public class BeanFactoryTest {   
    public static void main(String[] args) throws Throwable{  
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();  
        Resource res = resolver.getResource("classpath:com/baobaotao/beanfactory/beans.xml");   
        BeanFactory bf = new XmlBeanFactory(res);  
        System.out.println("init BeanFactory.");  
        Car car = bf.getBean("car",Car.class);  
        System.out.println("car bean is ready for use!");  
}

- `XmlBeanFactory` 通过 `Resource` 装载 Spring 配置信息并启动 IoC 容器，然后就可以通过 `BeanFactory.getBean(beanName)`方法从 IoC 容器中获取 Bean 了。通过 `BeanFactory` 启动 IoC 容器时，并不会初始化配置文件中定义的 Bean，初始化动作发生在第一个调用时。
    
- 对于单实例（ singleton）的 Bean 来说，`BeanFactory` 会缓存 Bean 实例，所以第二次使用 `getBean()` 获取 Bean 时将直接从 IoC 容器的缓存中获取 Bean 实例。Spring 在 `DefaultSingletonBeanRegistry` 类中提供了一个用于缓存单实例 Bean 的缓存器，**它是一个用 `HashMap` 实现的缓存器**，单实例的 Bean 以 beanName 为键保存在这个 `HashMap` 中。
    
- 值得一提的是，在初始化 `BeanFactory` 时，必须为其提供一种日志框架，比如使用Log4J， 即在类路径下提供 Log4J 配置文件，这样启动 Spring 容器才不会报错。
    

## 2.2 ApplicationContext

### 2.2.1 体系架构

`ApplicationContext` 由 `BeanFactory` 派生而来，提供了更多面向实际应用的功能。**在 `BeanFactory` 中，很多功能需要以编程的方式实现，而在 `ApplicationContext` 中则可以通过配置的方式实现**。
![[image-180.png]]
`ApplicationContext` 继承了 `HierarchicalBeanFactory` 和 `ListableBeanFactory` 接口，在此基础上，还通过多个其他的接口扩展了 `BeanFactory` 的功能：

- `ClassPathXmlApplicationContext`：默认从类路径加载配置文件
    
- `FileSystemXmlApplicationContext`：默认从文件系统中装载配置文件
    
- `ApplicationEventPublisher`：让容器拥有发布应用上下文事件的功能，包括容器启动事件、关闭事件等。实现了 `ApplicationListener` 事件监听接口的 Bean 可以接收到容器事件 ， 并对事件进行响应处理 。 在 `ApplicationContext` 抽象实现类 `AbstractApplicationContext` 中，我们可以发现存在一个 `ApplicationEventMulticaster`，它负责保存所有监听器，以便在容器产生上下文事件时通知这些事件监听者。
    
- `MessageSource`：为应用提供 i18n 国际化消息访问的功能；
    
- `ResourcePatternResolver` ： 所有 `ApplicationContext` 实现类都实现了类似于`PathMatchingResourcePatternResolver` 的功能，可以通过带前缀的 Ant 风格的资源文件路径装载 Spring 的配置文件。
    
- `LifeCycle`：该接口是 Spring 2.0 加入的，该接口提供了 `start()` 和 `stop()` 两个方法，主要用于控制异步处理过程。在具体使用时，该接口同时被 `ApplicationContext` 实现及具体 Bean 实现， `ApplicationContext` 会将 start/stop 的信息传递给容器中所有实现了该接口的 Bean，以达到管理和控制 JMX、任务调度等目的。
    
- `ConfigurableApplicationContext` 扩展于 `ApplicationContext`，它新增加了两个主要的方法： `refresh()` 和 `close()`，让 `ApplicationContext` 具有启动、刷新和关闭应用上下文的能力。在应用上下文关闭的情况下调用 `refresh()` 即可启动应用上下文，在已经启动的状态下，调用 `refresh()` 则清除缓存并重新装载配置信息，而调用 `close()` 则可关闭应用上下文。这些接口方法为容器的控制管理带来了便利，但作为开发者，我们并不需要过多关心这些方法。
    

### 2.2.2 WebApplicationContext

#### 2.2.2.1 体系结构

`WebApplication` 体系架构：
![[image-181.png]]
`WebApplicationContext` 是专门为 Web 应用准备的，它允许从相对于 Web 根目录的路径中装载配置文件完成初始化工作。从`WebApplicationContext` 中可以获得 `ServletContext` 的引用，整个 Web 应用上下文对象将作为属性放置到 `ServletContext` 中，以便 Web 应用环境可以访问 Spring 应用上下文。

#### 2.2.2.2 Spring 和 Web 应用的上下文融合

`WebApplicationContext` 定义了一个常量 `ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`，在上下文启动时， `WebApplicationContext` 实例即以此为键放置在 `ServletContext` 的属性列表中，因此我们可以直接通过以下语句从 Web 容器中获取 `WebApplicationContext`：

WebApplicationContext wac = (WebApplicationContext)servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
![[image-182.png]]
`WebApplicationContext` 的初始化方式：`WebApplicationContext` 需要 `ServletContext` 实例，它必须在拥有 Web 容器的前提下才能完成启动的工作。可以在 `web.xml` 中配置自启动的 `Servlet` 或定义 Web 容器监听器（ `ServletContextListener`），借助这两者中的任何一个就可以完成启动 Spring Web 应用上下文的工作。Spring 分别提供了用于启动 `WebApplicationContext` 的 Servlet 和 Web 容器监听器：

- `org.springframework.web.context.ContextLoaderServlet`；
    
- `org.springframework.web.context.ContextLoaderListener`
    

由于 `WebApplicationContext` 需要使用日志功能，比如日志框架使用 Log4J，用户可以将 Log4J 的配置文件放置到类路径 WEB-INF/classes 下，这时 Log4J 引擎即可顺利启动。如果 Log4J 配置文件放置在其他位置，用户还必须在 `web.xml` 指定 Log4J 配置文件位置。

## 2.3 BeanFactory 和 ApplicationConext 的区别

1. `BeanFactory` 是 Spring 框架的基础设施，面向 Spring 本身；`ApplicationContext` 面向使用 Spring 框架的开发者。
    
2. 在 `BeanFactory` 中，很多功能需要以编程的方式实现，而在 `ApplicationContext` 中则可以通过配置的方式实现。
    
3. `ApplicationContext` 通过多个其他的接口扩展了 `BeanFactory` 的功能：
    
    - `ApplicationEventPublisher`：让容器拥有发布应用上下文事件的功能，包括容器启动事件、关闭事件等。
        
    - `MessageSource`：为应用提供 i18n 国际化消息访问的功能；
        
    - `ResourcePatternResolver` ： 所有 `ApplicationContext` 实现类都实现了类似于`PathMatchingResourcePatternResolver` 的功能，可以通过带前缀的 Ant 风格的资源文件路径装载 Spring 的配置文件。
        
4. `ApplicationContext` 会利用 Java 反射机制自动识别出配置文件中定义的 `BeanPostProcessor`、 `InstantiationAwareBeanPostProcessor` 和 `BeanFactoryPostProcessor`，并自动将它们注册到应用上下文中；而后者需要在代码中通过手工调用 `addBeanPostProcessor()` 方法进行注册。
    
5. `BeanFactroy` 采用的是延迟加载形式来注入 Bean 的，即只有在使用到某个 Bean 时(调用 `getBean()`)，才对该 Bean 进行加载实例化，这样，我们就不能发现一些存在的 Spring 的配置问题。而`ApplicationContext` 则相反，它是在容器启动时，一次性创建了所有的 Bean。这样，在容器启动时，我们就可以发现 Spring 中存在的配置错误。
    

# 3 Bean 生命周期

## 3.1 流程
![[image-183.png]]
流程如下：

1. 当调用者通过 `getBean(beanName)` 向容器请求某一个 Bean 时，如果容器注册了`org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor` 接口（此接口继承 `BeanPostProcessor` 接口），在实例化 Bean 之前，将调用接口的 `postProcessBeforeInstantiation()` 方法；
    
2. 根据配置情况调用 Bean 构造函数或工厂方法实例化 Bean；
    
3. 如果容器注册了 `InstantiationAwareBeanPostProcessor` 接口，在实例化 Bean 之后，调用该接口的 `postProcessAfterInstantiation()` 方法，可在这里对已经实例化的对象进行一些“梳妆打扮”；
    
4. 如果 Bean 配置了属性信息，容器在这一步着手将配置值设置到 Bean 对应的属性中，不过在设置每个属性之前将先调用 `InstantiationAwareBeanPostProcessor` 接口的 `postProcessPropertyValues()` 方法；
    
5. 调用 Bean 的属性设置方法设置属性值；
    
6. 如果 Bean 实现了 `org.springframework.beans.factory.BeanNameAware` 接口，将调用`setBeanName()` 接口方法，将配置文件中该 Bean 对应的名称设置到 Bean 中；
    
7. 如果 Bean 实现了 `org.springframework.beans.factory.BeanFactoryAware` 接口，将调用 `setBeanFactory()` 接口方法，将 `BeanFactory` 容器实例设置到 Bean 中；
    
8. 如果 `BeanFactory` 装配了 `org.springframework.beans.factory.config.BeanPostProcessor` 后处理器，将调用 `BeanPostProcessor` 的 `Object postProcessBeforeInitialization(Object bean, String beanName)` 接口方法对 Bean 进行加工操作。其中入参 `bean` 是当前正在处理的 Bean，而 `beanName` 是当前 Bean 的配置名，返回的对象为加工处理后的 Bean。
    
    用户可以使用该方法对某些 Bean 进行特殊的处理，甚至改变 Bean 的行为， `BeanPostProcessor` 在 Spring 框架中占有重要的地位，为容器提供对 Bean 进行后续加工处理的切入点， Spring 容器所提供的各种“神奇功能”（如 AOP，动态代理等）都通过 `BeanPostProcessor` 实施；
    
9. 如果 Bean 实现了 `InitializingBean` 的接口，将调用接口的 `afterPropertiesSet()` 方法；
    
10. 如果在 `<bean>` 通过 `init-method` 属性定义了初始化方法，将执行这个方法；
    
11. `BeanPostProcessor` 后处理器定义了两个方法：其一是 `postProcessBeforeInitialization()` 在第 8 步调用；其二是 `Object postProcessAfterInitialization(Object bean, String beanName)` 方法，这个方法在此时调用，容器再次获得对 Bean 进行加工处理的机会；
    
12. 如果在 `<bean>` 中指定 Bean 的作用范围为 `scope=“prototype”`，将 Bean 返回给调用者，调用者负责 Bean 后续生命的管理， Spring 不再管理这个 Bean 的生命周期。如果作用范围设置为 `scope=“singleton”`，则将 Bean 放入到 Spring IoC 容器的缓存池中，并将 Bean 引用返回给调用者， Spring 继续对这些 Bean 进行后续的生命管理；
    
13. 对于 `scope=“singleton”` 的 Bean，当容器关闭时，将触发 Spring 对 Bean 的后续生命周期的管理工作，首先如果 Bean 实现了 `DisposableBean` 接口，则将调用接口的 `afterPropertiesSet()` 方法，可以在此编写释放资源、记录日志等操作；
    
14. 对于 `scope=“singleton”` 的 Bean，如果通过 `<bean>` 的 `destroy-method` 属性指定了 Bean 的销毁方法， Spring 将执行 Bean 的这个方法，完成 Bean 资源的释放等操作。
    

## 3.2 方法分类

可以将这些方法大致划分为三类：

- **Bean 自身的方法**：如调用 Bean 构造函数实例化 Bean，调用 Setter 设置 Bean 的属性值以及通过`<bean>` 的 `init-method` 和 `destroy-method` 所指定的方法；
    
- **Bean 级生命周期接口方法**：如 `BeanNameAware`、 `BeanFactoryAware`、 `InitializingBean` 和 `DisposableBean`，这些接口方法由 Bean 类直接实现；
    
- **容器级生命周期接口方法**：在上图中带“★” 的步骤是由 `InstantiationAwareBeanPostProcessor` 和 `BeanPostProcessor` 这两个接口实现，一般称它们的实现类为“ 后处理器” 。 后处理器接口一般不由 Bean 本身实现，它们独立于 Bean，实现类以容器附加装置的形式注册到 Spring 容器中并通过接口反射为 Spring 容器预先识别。当Spring 容器创建任何 Bean 的时候，这些后处理器都会发生作用，所以这些后处理器的影响是全局性的。当然，用户可以通过合理地编写后处理器，让其仅对感兴趣 Bean 进行加工处理
    

`ApplicationContext` 和 `BeanFactory` 另一个最大的不同之处在于：**`ApplicationContext` 会利用 Java 反射机制自动识别出配置文件中定义的 `BeanPostProcessor`、 `InstantiationAwareBeanPostProcessor` 和 `BeanFactoryPostProcessor`，并自动将它们注册到应用上下文中；而后者需要在代码中通过手工调用 `addBeanPostProcessor()` 方法进行注册**。这也是为什么在应用开发时，我们普遍使用 `ApplicationContext` 而很少使用 `BeanFactory` 的原因之一。

# 4 IOC容器工作机制

## 4.1 Bean 加载过程

### 4.1.1 Spring 架构

Spring 的高明之处在于，它使用众多接口描绘出了所有装置的蓝图，构建好 Spring 的骨架，继而通过继承体系层层推演，不断丰富，最终让 Spring 成为有血有肉的完整的框架。所以查看 Spring 框架的源码时，有两条清晰可见的脉络：

1. 接口层描述了容器的重要组件及组件间的协作关系；
    
2. 继承体系逐步实现组件的各项功能。
    

接口层清晰地勾勒出 Spring 框架的高层功能，框架脉络呼之欲出。有了接口层抽象的描述后，不但 Spring 自己可以提供具体的实现，任何第三方组织也可以提供不同实现， 可以说 Spring 完善的接口层使框架的扩展性得到了很好的保证。纵向继承体系的逐步扩展，分步骤地实现框架的功能，这种实现方案保证了框架功能不会堆积在某些类的身上，造成过重的代码逻辑负载，框架的复杂度被完美地分解开了。

### 4.1.2 Spring 组件

Spring 组件按其所承担的角色可以划分为两类：

1. **物料组件**：`Resource`、`BeanDefinition`、`PropertyEditor`以及最终的 Bean 等，它们是加工流程中被加工、被消费的组件，就像流水线上被加工的物料；
    

- `BeanDefinition`：Spring 通过 `BeanDefinition` 将配置文件中的 `<bean>` 配置信息转换为容器的内部表示，并将这些 `BeanDefinition` 注册到 `BeanDefinitionRegistry` 中。Spring 容器的后续操作直接从 `BeanDefinitionRegistry` 中读取配置信息。
    

2. **加工设备组件**：`ResourceLoader`、`BeanDefinitionReader`、`BeanFactoryPostProcessor`、`InstantiationStrategy` 以及 `BeanWrapper` 等组件像是流水线上不同环节的加工设备，对物料组件进行加工处理。
    
    - `InstantiationStrategy`：负责实例化 Bean 操作，相当于 Java 语言中 `new` 的功能，并不会参与Bean 属性的配置工作。属性填充工作留待 `BeanWrapper` 完成
        
    - `BeanWrapper`：继承了 `PropertyAccessor` 和 `PropertyEditorRegistry` 接口，`BeanWrapperImpl` 内部封装了两类组件：（1）被封装的目标Bean（2）一套用于设置Bean属性的属性编辑器。
        
        它具有三重身份：（1）Bean 包裹器（2）属性访问器 （3）属性编辑器注册表。
        
        `PropertyAccessor`：定义了各种访问 Bean 属性的方法。`PropertyEditorRegistry`：属性编辑器的注册表。
        

### 4.1.3 作业流程

该图描述了 Spring 容器从加载配置文件到创建出一个完整 Bean 的作业流程：
![[image-184.png]]
1. `ResourceLoader` 从存储介质中加载 `Spring` 配置信息，并使用 `Resource` 表示这个配置文件的资源；
    
2. `BeanDefinitionReader` 读取 `Resource` 所指向的配置文件资源，然后解析配置文件。配置文件中每一个 `<bean>` 解析成一个 `BeanDefinition` 对象，并保存到 `BeanDefinitionRegistry` 中；
    
3. 容器扫描 `BeanDefinitionRegistry` 中的 `BeanDefinition`，使用 Java 的反射机制自动识别出 Bean 工厂后处理后器（实现 `BeanFactoryPostProcessor` 接口）的 Bean，然后调用这些 Bean 工厂后处理器对 `BeanDefinitionRegistry` 中的 `BeanDefinition` 进行加工处理。主要完成以下两项工作：
    
    - 对使用到占位符的 `<bean>` 元素标签进行解析，得到最终的配置值，这意味对一些半成品式的`BeanDefinition` 对象进行加工处理并得到成品的 `BeanDefinition` 对象；
        
    - 对 `BeanDefinitionRegistry` 中的 `BeanDefinition` 进行扫描，通过 Java 反射机制找出所有属性编辑器的 Bean（实现 `java.beans.PropertyEditor` 接口的Bean），并自动将它们注册到 Spring 容器的属性编辑器注册表中（`PropertyEditorRegistry`）；
        
4. Spring 容器从 `BeanDefinitionRegistry` 中取出加工后的 `BeanDefinition`，并调用`InstantiationStrategy` 着手进行 Bean 实例化的工作；
    
5. 在实例化 Bean 时，Spring 容器使用 `BeanWrapper` 对 Bean 进行封装，`BeanWrapper` 提供了很多以Java 反射机制操作 Bean 的方法，它将结合该 Bean 的 `BeanDefinition` 以及容器中属性编辑器，完成 Bean 属性的设置工作；
    
6. 利用容器中注册的 Bean 后处理器（实现 `BeanPostProcessor` 接口的 Bean）对已经完成属性设置工作的 Bean 进行后续加工，直接装配出一个准备就绪的 Bean。
    

## 4.2 web 环境下 Spring 容器、SpringMVC 容器启动过程

1. 首先，对于一个 web 应用，其部署在 web 容器中，web 容器提供其一个全局的上下文环境，这个上下文就是 `ServletContext`，其为后面的 Spring IoC 容器提供宿主环境；
    
2. 其次，在 `web.xml` 中会提供有 `ContextLoaderListener`（或 `ContextLoaderServlet`）。在 web 容器启动时，会触发容器初始化事件，此时 `ContextLoaderListener` 会监听到这个事件，其`contextInitialized` 方法会被调用，在这个方法中，Spring 会初始化一个启动上下文，这个上下文被称为根上下文，即 `WebApplicationContext`，这是一个接口类，确切的说，其实际的实现类是`XmlWebApplicationContext`。这个就是 Spring 的 IoC 容器，其对应的 Bean 定义的配置由 `web.xml` 中的 `context-param` 标签指定。
    
    在这个 IoC 容器初始化完毕后，Spring 容器以`WebApplicationContext.ROOTWEBAPPLICATIONCONTEXTATTRIBUTE` 为属性 Key，将其存储到`ServletContext` 中，便于获取；
    
3. 再次，`ContextLoaderListener` 监听器初始化完毕后，开始初始化 `web.xml` 中配置的 `Servlet`，这个 `Servlet` 可以配置多个，以最常见的 `DispatcherServlet` 为例（SpringMVC），这个 `Servlet` 实际上是一个标准的前端控制器，用以转发、匹配、处理每个 Servlet 请求。`DispatcherServlet` 上下文在初始化的时候会建立自己的 IoC 上下文容器，用以持有 SpringMVC 相关的 Bean，这个 `Servlet` 自己持有的上下文默认实现类也是 `XmlWebApplicationContext`。
    
    在建立`DispatcherServlet` 自己的 IoC 上下文时，会利用`WebApplicationContext.ROOTWEBAPPLICATIONCONTEXTATTRIBUTE` 先从 `ServletContext` 中获取之前的根上下文(即 `WebApplicationContext`) 作为自己上下文的 parent 上下文（即第 2 步中初始化的`XmlWebApplicationContext` 作为自己的父容器）。有了这个 parent 上下文之后，再初始化自己持有的上下文（这个 `DispatcherServlet` 初始化自己上下文的工作在其 `initStrategies` 方法中可以看到，大概的工作就是初始化处理器映射、视图解析等）。
    
    初始化完毕后，Spring 以与 `Servlet` 的名字相关(此处不是简单的以 `Servlet` 名为 Key，而是通过一些转换)的属性为属性 Key，也将其存到 `ServletContext` 中，以便后续使用。这样每个 `Servlet` 就持有自己的上下文，即拥有自己独立的 Bean 空间，同时各个 `Servlet` 共享相同的 Bean，即根上下文定义的那些 Bean。
    

# 5 Spring 循环依赖

Spring 中的循环依赖一直是 Spring 中一个很重要的话题，一方面是因为源码中为了解决循环依赖做了很多处理，另外一方面是因为面试的时候，如果问到 Spring 中比较高阶的问题，那么循环依赖必定逃不掉。

当面试官问：“请讲一讲Spring中的循环依赖。”的时候，我们到底该怎么回答？主要分下面几点：

1. 什么是循环依赖？
    
2. 什么情况下循环依赖可以被处理？
    
3. Spring 是如何解决的循环依赖？
    

## 5.1 什么是循环依赖

从字面上来理解就是 A 依赖 B 的同时 B 也依赖了 A。
体现到代码层次就是这个样子：

@Component  
public class A {  
    // A中注入了B  
    @Autowired  
    private B b;  
}  
​  
@Component  
public class B {  
    // B中也注入了A  
    @Autowired  
    private A a;  
}

当然，这是最常见的一种循环依赖，比较特殊的还有：

// 自己依赖自己  
@Component  
public class A {  
    // A中注入了A  
    @Autowired  
    private A a;  
}

## 5.2 什么情况下循环依赖可以被处理

在回答这个问题之前首先要明确一点，Spring 解决循环依赖是有前置条件的

1. 出现循环依赖的 Bean 必须要是单例
    
2. 依赖注入的方式不能全是构造器注入的方式（很多博客上说，只能解决 `setter` 方法的循环依赖，这是错误的）
    

其中第一点应该很好理解，第二点：不能全是构造器注入是什么意思呢？我们还是用代码说话：

@Component  
public class A {  
//  @Autowired  
//  private B b;  
    public A(B b) {  
​  
    }  
}  
​  
​  
@Component  
public class B {  
​  
//  @Autowired  
//  private A a;  
​  
    public B(A a){  
​  
    }  
}

在上面的例子中，A 中注入 B 的方式是通过构造器，B 中注入 A 的方式也是通过构造器，这个时候循环依赖是无法被解决，如果你的项目中有两个这样相互依赖的 Bean，在启动时就会报出以下错误：

Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'a': Requested bean is currently in creation: Is there an unresolvable circular reference?

为了测试循环依赖的解决情况跟注入方式的关系，我们做如下四种情况的测试：

|依赖情况|依赖注入方式|循环依赖是否被解决|
|---|---|---|
|AB 相互依赖（循环依赖）|均采用 setter 方法注入|是|
|AB 相互依赖（循环依赖）|均采用构造器注入|否|
|AB 相互依赖（循环依赖）|A 中注入 B 的方式为 setter 方法，B 中注入 A 的方式为构造器|是|
|AB 相互依赖（循环依赖）|B 中注入 A 的方式为 setter 方法，A 中注入 B 的方式为构造器|否|

**注意 Spring 在创建 Bean 的时候默认是按照自然排序来进行创建的**。

## 5.3 Spring 是如何解决的循环依赖

关于循环依赖的解决方式应该要分两种情况来讨论

1. 简单的循环依赖（没有 AOP）
    
2. 结合了 AOP 的循环依赖
    

### 5.3.1 简单的循环依赖（没有 AOP）

我们先来分析一个最简单的例子，就是上面提到的那个 demo：

@Component  
public class A {  
    // A中注入了B  
    @Autowired  
    private B b;  
}  
​  
@Component  
public class B {  
    // B中也注入了A  
    @Autowired  
    private A a;  
}

通过上文我们已经知道了这种情况下的循环依赖是能够被解决的，那么具体的流程是什么呢？我们一步步分析。

首先，我们要知道 Spring 在创建 Bean 的时候默认是按照自然排序来进行创建的，所以第一步 Spring 会去创建 A。与此同时，我们应该知道，Spring 在创建 Bean 的过程中分为三步：

1. 实例化，对应方法：`AbstractAutowireCapableBeanFactory`中的`createBeanInstance`方法。
    
2. 属性注入，对应方法：`AbstractAutowireCapableBeanFactory`的`populateBean`方法。
    
3. 初始化，对应方法：`AbstractAutowireCapableBeanFactory`的`initializeBean`。
    

这三个方法分别做了以下事情：

1. 实例化，简单理解就是 `new` 了一个对象
    
2. 属性注入，为实例化中 new 出来的对象填充属性
    
3. 初始化，执行 aware 接口中的方法，初始化方法，完成 AOP 代理
    

基于上面的知识，我们开始解读整个循环依赖处理的过程，整个流程应该是以 A 的创建为起点，前文也说了，第一步就是创建 A 嘛！
![[image-185.png]]
创建 A 的过程实际上就是调用 `getBean` 方法，这个方法有两层含义

1. 创建一个新的 Bean。
    
2. 从缓存中获取到已经被创建的对象。
    

我们现在分析的是第一层含义，因为这个时候缓存中还没有 A 嘛！

#### 5.3.1.1 调用 getSingleton(beanName)——三级缓存

首先调用 `getSingleton(a)` 方法，这个方法又会调用 `getSingleton(beanName, true)`，在上图中我省略了这一步：

public Object getSingleton(String beanName) {  
    return getSingleton(beanName, true);  
}

`getSingleton(beanName, true)` 这个方法实际上就是到缓存中尝试去获取 Bean，**整个缓存分为三级**：

1. `singletonObjects`，一级缓存，存储的是所有创建好了的单例 Bean
    
2. `earlySingletonObjects`，二级缓存，完成实例化，但是还未进行属性注入及初始化的对象
    
3. `singletonFactories`，三级缓存，提前暴露的一个单例工厂，二级缓存中存储的就是从这个工厂中获取到的对象
    

因为 A 是第一次被创建，所以不管哪个缓存中必然都是没有的，因此会进入 `getSingleton` 的另外一个重载方法 `getSingleton(beanName, singletonFactory)`。

#### 5.3.1.2 调用 getSingleton(beanName, singletonFactory)

这个方法就是用来创建 Bean 的，其源码如下：

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {  
    Assert.notNull(beanName, "Bean name must not be null");  
    synchronized (this.singletonObjects) {  
        Object singletonObject = this.singletonObjects.get(beanName);  
        if (singletonObject == null) {  
​  
            // ....  
            // 省略异常处理及日志  
            // ....  
​  
            // 在单例对象创建前先做一个标记  
            // 将beanName放入到singletonsCurrentlyInCreation这个集合中  
            // 标志着这个单例Bean正在创建  
            // 如果同一个单例Bean多次被创建，这里会抛出异常  
            beforeSingletonCreation(beanName);  
            boolean newSingleton = false;  
            boolean recordSuppressedExceptions = (this.suppressedExceptions == null);  
            if (recordSuppressedExceptions) {  
                this.suppressedExceptions = new LinkedHashSet<>();  
            }  
            try {  
                // 上游传入的lambda在这里会被执行，调用createBean方法创建一个Bean后返回  
                singletonObject = singletonFactory.getObject();  
                newSingleton = true;  
            }  
            // ...  
            // 省略catch异常处理  
            // ...  
            finally {  
                if (recordSuppressedExceptions) {  
                    this.suppressedExceptions = null;  
                }  
                // 创建完成后将对应的beanName从singletonsCurrentlyInCreation移除  
                afterSingletonCreation(beanName);  
            }  
            if (newSingleton) {  
                // 添加到一级缓存singletonObjects中  
                addSingleton(beanName, singletonObject);  
            }  
        }  
        return singletonObject;  
    }  
}
```

上面的代码我们主要抓住一点，通过 `createBean` 方法返回的 Bean 最终被放到了一级缓存，也就是单例池中。那么到这里我们可以得出一个结论：**一级缓存中存储的是已经完全创建好了的单例 Bean**。

#### 5.3.1.3 调用 addSingletonFactory 方法

在 `createBean` 方法的执行流程中，会调用 `addSingletonFactory` 方法，如下图所示：
![[image-186.png]]
在完成 Bean 的实例化后，属性注入之前 Spring 将 Bean 包装成一个工厂添加进了三级缓存中，对应源码如下：

```java
// 这里传入的参数也是一个lambda表达式，() -> getEarlyBeanReference(beanName, mbd, bean)  
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {  
    Assert.notNull(singletonFactory, "Singleton factory must not be null");  
    synchronized (this.singletonObjects) {  
        if (!this.singletonObjects.containsKey(beanName)) {  
            // 添加到三级缓存中  
            this.singletonFactories.put(beanName, singletonFactory);  
            this.earlySingletonObjects.remove(beanName);  
            this.registeredSingletons.add(beanName);  
        }  
    }  
}
```

这里只是添加了一个工厂，通过这个工厂（`ObjectFactory`）的 `getObject` 方法可以得到一个对象，而**这个对象实际上就是通过 `getEarlyBeanReference` 这个方法创建的**。那么，什么时候会去调用这个工厂的 `getObject` 方法呢？这个时候就要到创建 B 的流程了。

当 A 完成了实例化并添加进了三级缓存后，就要开始为 A 进行属性注入了，在注入时发现 A 依赖了 B，那么这个时候 Spring 又会去 `getBean(b)`，然后反射调用 setter 方法完成属性注入：
![[image-187.png]]
因为 B 需要注入 A，所以在创建 B 的时候，又会去调用 `getBean(a)`，这个时候就又回到之前的流程了，但是不同的是，之前的 `getBean` 是为了创建 Bean，而**此时再调用 `getBean` 不是为了创建了，而是要从缓存中获取**，因为之前 A 在实例化后已经将其放入了三级缓存 `singletonFactories` 中，所以此时 `getBean(a)` 的流程就是这样子了：
![[image-188.png]]
从这里我们可以看出，注入到 B 中的 A 是通过 `getEarlyBeanReference` 方法提前暴露出去的一个对象，还不是一个完整的 Bean，那么 `getEarlyBeanReference` 到底干了啥了，我们看下它的源码：
```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
            }
        }
    }
    return exposedObject;
}
```
它实际上就是调用了后置处理器的 `getEarlyBeanReference`，而真正实现了这个方法的后置处理器只有一个，就是通过 `@EnableAspectJAutoProxy` 注解导入的 `AnnotationAwareAspectJAutoProxyCreator`。**也就是说如果在不考虑 AOP 的情况下，上面的代码等价于：**

protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {  
    Object exposedObject = bean;  
    return exposedObject;  
}

**也就是说这个工厂啥都没干，直接将实例化阶段创建的对象返回了！所以说在不考虑 AOP 的情况下三级缓存有用嘛？讲道理，真的没什么用**，我直接将这个对象放到二级缓存中不是一点问题都没有吗？如果你说它提高了效率，那你告诉我提高的效率在哪? 那么三级缓存到底有什么作用呢？不要急，我们先把整个流程走完，在下文结合 AOP 分析循环依赖的时候你就能体会到三级缓存的作用！

到这里不知道小伙伴们会不会有疑问，B 中提前注入了一个没有经过初始化的 A 类型对象不会有问题吗？答：不会。这个时候我们需要将整个创建 A 这个 Bean 的流程走完，如下图：
![[image-189.png]]
从上图中我们可以看到，虽然在创建 B 时会提前给 B 注入了一个还未初始化的 A 对象，但是在创建 A 的流程中一直使用的是注入到 B 中的 A 对象的引用，之后会根据这个引用对 A 进行初始化，所以这是没有问题的。

### 5.3.2 结合了 AOP 的循环依赖

#### 5.3.2.1 getEarlyBeanReference

之前我们已经说过了，在普通的循环依赖的情况下，三级缓存没有任何作用。三级缓存实际上跟 Spring 中的 AOP 相关，我们再来看一看 `getEarlyBeanReference` 的代码：

protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {  
    Object exposedObject = bean;  
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {  
        for (BeanPostProcessor bp : getBeanPostProcessors()) {  
            if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {  
                SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;  
                exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);  
            }  
        }  
    }  
    return exposedObject;  
}

如果在开启 AOP 的情况下，那么就是调用到 `AnnotationAwareAspectJAutoProxyCreator` 的 `getEarlyBeanReference` 方法，对应的源码如下：

public Object getEarlyBeanReference(Object bean, String beanName) {  
    Object cacheKey = getCacheKey(bean.getClass(), beanName);  
    this.earlyProxyReferences.put(cacheKey, bean);  
    // 如果需要代理，返回一个代理对象，不需要代理，直接返回当前传入的这个bean对象  
    return wrapIfNecessary(bean, beanName, cacheKey);  
}

回到上面的例子，我们对 A 进行了 AOP 代理的话，那么此时 `getEarlyBeanReference` 将返回一个代理后的对象，而不是实例化阶段创建的对象，这样就意味着 B 中注入的 A 将是一个代理对象而不是 A 的实例化阶段创建后的对象。
![[image-190.png]]
看到这个图你可能会产生下面这些疑问：

#### 5.3.2.2 在给 B 注入的时候为什么要注入一个代理对象？

答：当我们对 A 进行了 AOP 代理时，说明我们希望从容器中获取到的就是 A 代理后的对象而不是 A 本身，因此把 A 当作依赖进行注入时也要注入它的代理对象。

#### 5.3.2.3 明明初始化的时候是 A 对象，那么 Spring 是在哪里将代理对象放入到容器中的呢？
![[image-191.png]]
在完成初始化后，Spring 又调用了一次 `getSingleton` 方法，这一次传入的参数又不一样了，`false` 可以理解为禁用三级缓存，前面图中已经提到过了，在为 B 中注入 A 时已经将三级缓存中的工厂取出，并从工厂中获取到了一个对象放入到了二级缓存中，所以这里的这个 `getSingleton` 方法就是从二级缓存中获取到这个代理后的 A 对象。`exposedObject == bean` 可以认为是必定成立的，除非你非要在初始化阶段的后置处理器中替换掉正常流程中的 Bean，例如增加一个后置处理器：

@Component  
public class MyPostProcessor implements BeanPostProcessor {  
    @Override  
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {  
        if (beanName.equals("a")) {  
            return new A();  
        }  
        return bean;  
    }  
}

不过，请不要做这种骚操作，徒增烦恼！

#### 5.3.2.4 初始化的时候是对 A 对象本身进行初始化，而容器中以及注入到 B 中的都是代理对象，这样不会有问题吗？

答：不会，这是因为不管是 cglib 代理还是 jdk 动态代理生成的代理类，内部都持有一个目标类的引用，当调用代理对象的方法时，实际会去调用目标对象的方法，A 完成初始化相当于代理对象自身也完成了初始化。

#### 5.3.2.5 三级缓存为什么要使用工厂而不是直接使用引用？换而言之，为什么需要这个三级缓存，直接通过二级缓存暴露一个引用不行吗？

答：**这个工厂的目的在于延迟对实例化阶段生成的对象的代理，只有真正发生循环依赖的时候，才去提前生成代理对象**，否则只会创建一个工厂并将其放入到三级缓存中，但是不会去通过这个工厂去真正创建对象。

我们思考一种简单的情况，就以单独创建 A 为例，假设 AB 之间现在没有依赖关系，但是 A 被代理了，这个时候当 A 完成实例化后还是会进入下面这段代码：

// A是单例的，mbd.isSingleton()条件满足  
// allowCircularReferences：这个变量代表是否允许循环依赖，默认是开启的，条件也满足  
// isSingletonCurrentlyInCreation：正在在创建A，也满足  
// 所以earlySingletonExposure=true  
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&  
                                  isSingletonCurrentlyInCreation(beanName));  
// 还是会进入到这段代码中  
if (earlySingletonExposure) {  
    // 还是会通过三级缓存提前暴露一个工厂对象  
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));  
}

看到了吧，即使没有循环依赖，也会将其添加到三级缓存中，而且是不得不添加到三级缓存中，因为到目前为止 Spring 也不能确定这个 Bean 有没有跟别的 Bean 出现循环依赖。

假设我们在这里直接使用二级缓存的话，那么意味着所有的 Bean 在这一步都要完成 AOP 代理。这样做有必要吗？

不仅没有必要，而且违背了 Spring 在结合 AOP 跟 Bean 的生命周期的设计！Spring 结合 AOP 跟 Bean 的生命周期本身就是通过 `AnnotationAwareAspectJAutoProxyCreator` 这个后置处理器来完成的，在这个后置处理的 `postProcessAfterInitialization` 方法中对初始化后的 Bean 完成 AOP 代理。**如果出现了循环依赖，那没有办法，只有给 Bean 先创建代理，但是没有出现循环依赖的情况下，设计之初就是让 Bean 在生命周期的最后一步完成代理而不是在实例化后就立马完成代理**。

## 5.4 三级缓存真的提高了效率了吗

现在我们已经知道了三级缓存的真正作用，但是这个答案可能还无法说服你，所以我们再最后总结分析一波，三级缓存真的提高了效率了吗？分为两点讨论：

1. 没有进行 AOP 的 Bean 间的循环依赖
    
    从上文分析可以看出，这种情况下三级缓存根本没用！所以不会存在什么提高了效率的说法。
    
2. 进行了 AOP 的 Bean 间的循环依赖
    
    就以我们上的 A、B 为例，其中 A 被 AOP 代理，我们先分析下使用了三级缓存的情况下，A、B 的创建流程：
![[image-192.png]]
假设不使用三级缓存，直接在二级缓存中：
![[image-193.png]]
1. 上面两个流程的唯一区别在于为 A 对象创建代理的时机不同，在使用了三级缓存的情况下为 A 创建代理的时机是在 B 中需要注入 A 的时候，而不使用三级缓存的话在 A 实例化后就需要马上为 A 创建代理然后放入到二级缓存中去。对于整个 A、B 的创建过程而言，消耗的时间是一样的。
    

综上，**不管是哪种情况，三级缓存提高了效率这种说法都是错误的**！
## 5.5 总结

1. 面试官：”Spring是如何解决的循环依赖？“
    
    答：Spring 通过三级缓存解决了循环依赖，其中一级缓存为单例池（`singletonObjects`）,二级缓存为早期曝光对象`earlySingletonObjects`，三级缓存为早期曝光对象工厂（`singletonFactories`）。
    
    当 A、B 两个类发生循环引用时，在 A 完成实例化后，就使用实例化后的对象去创建一个对象工厂，并添加到三级缓存中，如果 A 被 AOP 代理，那么通过这个工厂获取到的就是 A 代理后的对象，如果 A 没有被AOP 代理，那么这个工厂获取到的就是 A 实例化的对象。
    
    当 A 进行属性注入时，会去创建 B，同时 B 又依赖了 A，所以创建 B 的同时又会去调用 `getBean(a)` 来获取需要的依赖，此时的 `getBean(a)` 会从缓存中获取。
    
    - 首先，先获取到三级缓存中的工厂；
        
    - 然后，调用对象工工厂的 `getObject` 方法来获取到对应的对象，得到这个对象后将其注入到 B 中。
        
    - 紧接着 B 会走完它的生命周期流程，包括初始化、后置处理器等。
        
    
    当 B 创建完后，会将 B 再注入到 A 中，此时 A 再完成它的整个生命周期。至此，循环依赖结束！
    
2. 面试官：”为什么要使用三级缓存呢？二级缓存能解决循环依赖吗？“
    
    答：如果要使用二级缓存解决循环依赖，意味着所有 Bean 在实例化后就要完成 AOP 代理，这样违背了Spring 设计的原则，Spring 在设计之初就是通过 `AnnotationAwareAspectJAutoProxyCreator` 这个后置处理器来在 Bean 生命周期的最后一步来完成 AOP 代理，而不是在实例化后就立马进行 AOP 代理。
    

## 5.6 问题

为什么在下表中的第三种情况的循环依赖能被解决，而第四种情况不能被解决呢？

|依赖情况|依赖注入方式|循环依赖是否被解决|
|---|---|---|
|AB 相互依赖（循环依赖）|均采用 setter 方法注入|是|
|AB 相互依赖（循环依赖）|均采用构造器注入|否|
|AB 相互依赖（循环依赖）|A 中注入 B 的方式为 setter 方法，B 中注入 A 的方式为构造器|是|
|AB 相互依赖（循环依赖）|B 中注入 A 的方式为 setter 方法，A 中注入 B 的方式为构造器|否|

因为第三种情况中，A 可以先创建，在属性注入阶段再创建 B 并为其注入自己；而第四种情况中，A 创建的时候就需要先创建 B，这会导致 B 在属性注入阶段无法获取到 A 的对象（因为此时 A 还未创建），从而导致出错。

---
# 6 引用