## spring

### [字]spring生命周期，几种scope区别，aop实现有哪几种实现，接口代理和类代理会有什么区别

### spring ioc初始化流程
<details><summary>详情</summary>  

IOC 容器的初始化过程分为三步骤：Resource 定位、BeanDefinition 的载入和解析，BeanDefinition 注册

- **Resource 定位**。我们一般用外部资源来描述 Bean 对象，所以在初始化 IOC 容器的第一步就是需要定位这个外部资源。

- **BeanDefinition 的载入和解析**。装载就是 BeanDefinition 的载入。BeanDefinitionReader 读取、解析 Resource 资源，也就是将用户定义的 Bean 表示成 IOC 容器的内部数据结构：BeanDefinition。在 IOC 容器内部维护着一个 BeanDefinition Map 的数据结构，在配置文件中每一个都对应着一个BeanDefinition对象。

- **BeanDefinition 注册**。向IOC容器注册在第二步解析好的 BeanDefinition，这个过程是通过 BeanDefinitionRegistery 接口来实现的。在 IOC 容器内部其实是将第二个过程解析得到的 BeanDefinition 注入到一个 HashMap 容器中，IOC 容器就是通过这个 HashMap 来维护这些 BeanDefinition 的。在这里需要注意的一点是这个过程并没有完成依赖注入，依赖注册是发生在应用第一次调用 getBean() 向容器索要 Bean 时。当然我们可以通过设置预处理，即对某个 Bean 设置 lazyinit 属性，那么这个 Bean 的依赖注入就会在容器初始化的时候完成。
</details>

### DI依赖注入流程

过程在Ioc初始化后，依赖注入的过程是用户第一次向IoC容器索要Bean时触发

- 如果设置lazy-init=true，会在第一次getBean的时候才初始化bean， lazy-init=false，会容器启动的时候直接初始化（singleton bean）；

- 调用BeanFactory.getBean（）生成bean的；

- 生成bean过程运用装饰器模式产生的bean都是beanWrapper（bean的增强）

### Bean的生命周期

- 实例化Bean：Ioc容器通过获取BeanDefinition对象中的信息进行实例化，实例化对象被包装在BeanWrapper对象中
- 设置对象属性（DI）：通过BeanWrapper提供的设置属性的接口完成属性依赖注入；
- 注入Aware接口（BeanFactoryAware， 可以用这个方式来获取其它 Bean，ApplicationContextAware）：Spring会检测该对象是否实现了xxxAware接口，并将相关的xxxAware实例注入给bean
- BeanPostProcessor：自定义的处理（分前置处理和后置处理）
- InitializingBean和init-method：执行我们自己定义的初始化方法
- 使用
- destroy：bean的销毁

IOC：控制反转：将对象的创建权，由Spring管理. DI（依赖注入）：在Spring创建对象的过程中，把对象依赖的属性注入到类中。

### Spring如何解决循环依赖?解决循环依赖的几个缓存？
<details><summary>详情</summary>  

假设场景如下，A->B->A

- 1、实例化A，并将未注入属性的A暴露出去，即提前曝光给容器Wrap
- 2、开始为A注入属性，发现需要B，调用getBean（B）
- 3、实例化B，并注入属性，发现需要A的时候，从单例缓存中查找，没找到时继而从Wrap中查找，从而完成属性的注入
- 4、递归完毕之后回到A的实例化过程，A将B注入成功，并注入A的其他属性值，自此即完成了循环依赖的注入

Spring中循环依赖场景有：

- 构造器的循环依赖
- 属性的循环依赖

Spring 如何解决的?
- Spring 为了解决单例的循环依赖问题，使用了 三级缓存 ，递归调用时发现 Bean 还在创建中即为循环依赖
- 单例模式的 Bean 保存在如下的数据结构中：

```
//  一级缓存：用于存放完全初始化好的 bean  
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

//  二级缓存：存放原始的 bean 对象（尚未填充属性），用于解决循环依赖  
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);

// 三级级缓存：存放 bean 工厂对象，用于解决循环依赖  
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

//bean 的获取过程：先从一级获取，失败再从二级、三级里面获取
//创建中状态：是指对象已经 new 出来了但是所有的属性均为 null 等待被 init

检测循环依赖的过程如下：
A 创建过程中需要 B，于是 A 将自己放到三级缓里面 ，去实例化 B
B 实例化的时候发现需要 A，于是 B 先查一级缓存，没有，再查二级缓存，还是没有，再查三级缓存，找到了！
然后把三级缓存里面的这个 A 放到二级缓存里面，并删除三级缓存里面的 A
B 顺利初始化完毕，将自己放到一级缓存里面（此时B里面的A依然是创建中状态）
然后回来接着创建 A，此时 B 已经创建结束，直接从一级缓存里面拿到 B ，然后完成创建，并将自己放到一级缓存里面
如此一来便解决了循环依赖的问题
```
- singletonObjects：第一级缓存，里面放置的是实例化好的单例对象；earlySingletonObjects：第二级缓存，里面存放的是提前曝光的单例对象；singletonFactories：第三级缓存，里面存放的是要被实例化的对象的对象工厂
- 创建bean的时候Spring首先从一级缓存singletonObjects中获取。如果获取不到，并且对象正在创建中，就再从二级缓存earlySingletonObjects中获取，如果还是获取不到就从三级缓存singletonFactories中取（Bean调用构造函数进行实例化后，即使属性还未填充，就可以通过三级缓存向外提前暴露依赖的引用值（提前曝光），根据对象引用能定位到堆中的对象，其原理是基于Java的引用传递），取到后从三级缓存移动到了二级缓存完全初始化之后将自己放入到一级缓存中供其他使用，
- 因为加入singletonFactories三级缓存的前提是执行了构造器，所以构造器的循环依赖没法解决。
- 构造器循环依赖解决办法：在构造函数中使用@Lazy注解延迟加载。在注入依赖时，先注入代理对象，当首次使用时再创建对象说明：一种互斥的关系而非层次递进的关系，故称为三个Map而非三级缓存的缘由 完成注入；
</details>

### Spring中使用了哪些设计模式？

- 工厂模式：spring中的BeanFactory就是简单工厂模式的体现，根据传入唯一的标识来获得bean对象；
- 单例模式：提供了全局的访问点BeanFactory；
- 代理模式：AOP功能的原理就使用代理模式（1、JDK动态代理。2、CGLib字节码生成技术代理。）
- 装饰器模式：依赖注入就需要使用BeanWrapper；
- 观察者模式：spring中Observer模式常用的地方是listener的实现。如ApplicationListener。
- 策略模式：Bean的实例化的时候决定采用何种方式初始化bean实例（反射或者CGLIB动态字节码生成）



## springmvc

### [字]说下SpringMVC不同用户登录的信息怎么保证线程安全的？



## springboot

### SpringBoot 启动流程

