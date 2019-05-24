## Spring的AOP理解  
AOP一般成为面向切面，作为面向对象的一种补充，用于将那些与业务无关，但对多个对象产生影响的公共行为和逻辑，抽取并封装成一个可重用的模块。这个模块被命名为"切面"，减少系统中的重复代码，降低模块之间的耦合度，同时提高了系统的可维护性。  

AOP实现的关键在于代理模式，AOP代理主要分为静态代理和动态代理。静态代理的代表为AspectJ；动态代理则以Spring AOP为代表的。  
1. AspectJ是静态代理的增强，所谓的静态代理，就是AOP框架会在编译阶段生成AOP代理类，因此也被称为编译时增强，它会在编译阶段将AspectJ植入到Java字节码中，运行的时候就是增强之后的AOP对象。
2. Spring AOP使用的是动态代理，所谓的动态代理就是说AOP框架不会去修改字节码，而是每次运行时在内存中临时为方法生成一个AOP对象，这个AOP对象包含了目标对象的全部方法，并且在特定的切点做了增强处理，并回调原对象的方法。

## Spring AOP中动态代理
Spring AOP中主要有两种方式，JDK动态代理和CGLIB动态代理。  
1. JDK动态代理只提供接口的代理，不支持类的代理。核心invocationHandler接口和Proxy类，InvocationHandler 通过invoke()方法反射来调用目标类中的代码，动态地将横切逻辑和业务编织在一起；接着，Proxy利用 InvocationHandler动态创建一个符合某一接口的的实例,  生成目标类的代理对象。

2. 如果代理类没有实现 InvocationHandler 接口，那么Spring AOP会选择使用CGLIB来动态代理目标类。CGLIB（Code Generation Library），是一个代码生成的类库，可以在运行时动态的生成指定类的一个子类对象，并覆盖其中特定方法并添加增强代码，从而实现AOP。CGLIB是通过继承的方式做的动态代理，因此如果某个类被标记为final，那么它是无法使用CGLIB做动态代理的。


## Spring IoC
1. IoC就是控制反转，是指创建对象的控制权的转移，以前创建对象的主动权和时机是由自己把控，而现在这种权利转移到Spring容器中，并且由容器根据配置文件去创建和管理各个实例之间的依赖，对象与对象之间的松散耦合，也有利于功能的复用。DI依赖注入和控制反转是同一概念的不同角度的描述，即应用程序在运行时依赖IoC容器来动态注入对象需要的外部资源。
2. 最直观的表达就是，IOC让对象的创建不用去New，可以由Spring自动生成，使用java的反射机制，根据配置文件在运行时动态的去创建对象以及管理对象，并调用对象的方法的。

## Spring Bean的生命周期  
- Bean的建立  
      由BeanFactory读取Bean定义文件，并生成各个实例。

- Setter注入  
      执行Bean的属性依赖注入。

- BeanNameAware的setBeanName()  
      如果Bean类实现了org.springframework.beans.factory.BeanNameAware接口，则执行其setBeanName()方法。

- BeanFactoryAware的setBeanFactory()  
      如果Bean类实现了org.springframework.beans.factory.BeanFactoryAware接口，则执行其setBeanFactory()方法。

- BeanPostProcessors的processBeforeInitialization()
      容器中如果有实现org.springframework.beans.factory.BeanPostProcessors接口的实例，则任何Bean在初始化之前都会执行这个实例的processBeforeInitialization()方法。

- InitializingBean的afterPropertiesSet()
      如果Bean类实现了org.springframework.beans.factory.InitializingBean接口，则执行其afterPropertiesSet()方法。

- Bean定义文件中定义init-method
      在Bean定义文件中使用“init-method”属性设定方法名称, 这时会执行initMethod()方法，注意，这个方法是不带参数的。

- BeanPostProcessors的processAfterInitialization()
      容器中如果有实现org.springframework.beans.factory.BeanPostProcessors接口的实例，则任何Bean在初始化之前都会执行这个实例的processAfterInitialization()方法。

- DisposableBean的destroy()
      在容器关闭时，如果Bean类实现了org.springframework.beans.factory.DisposableBean接口，则执行它的destroy()方法。

- Bean定义文件中定义destroy-method
      在容器关闭时，可以在Bean定义文件中使用“destory-method”定义的方法