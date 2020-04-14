

## BeanPostProcessor 族

实现了`BeanPostProcessor`及其子类接口的`Bean`将会被`BeanFactory`在执行`refresh`中某些特殊阶段进行回调，可以在这些方法内部修改或添加类似`BeanDefinition`、创建`Bean`代理等操作。

很多`Spring`框架内部使用了`BeanPostProcessor`来实现一些功能，例如`@Autowired`功能是通过`AutowiredAnnotationBeanPostProcessor`来实现。

### BeanPostProcessor

```java
public interface BeanPostProcessor {
   Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
   Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```



#### postProcessBeforeInitialization

该方法将在`Bean`实例化完成，并注入完属性后被调用，但优先于`init-method`、`InitializingBean#afterPropertiesSet`和`InitializingBean#afterPropertiesSet`。

可以在该方法中返回包装对象或代理对象。

如果此方法返回`null`，那么后续的`BeanPostProcessor`将不会被调用。

#### postProcessAfterInitialization

该方法在`postProcessBeforeInitialization`、`InitializingBean#afterPropertiesSet`、`init-method`调用完成之后被调用。

可以在该方法中返回包装对象或代理对象。

如果此方法返回`null`，那么后续的`BeanPostProcessor`将不会被调用。

### InstantiationAwareBeanPostProcessor

```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {
   Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException;

   boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;

   PropertyValues postProcessPropertyValues(
         PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException;

}
```

#### postProcessBeforeInstantiation

该方法在实例化`Bean`之前调用，可以在此处返回代理对象。

如果该方法返回值不为`null`，那么其余的实例化操作将会被短路`short-circuited`，包括其余的`InstantiationAwareBeanPostProcessor`和实例属性的注入（`populate`），但是当前`InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation`方法将仍会被调用。

#### postProcessAfterInstantiation

该方法在实例化`Bean`之后，注入属性之前被调用。

如果该方法返回`true`，那么后续的属性注入将被执行；如果返回`false`，那么将不执行属性的注入，也不会执行其他`InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation`方法。

该方法适合在自动注入属性前进行自定义字段的注入。

#### postProcessPropertyValues

该方法在`postProcessAfterInstantiation`方法调用之后，注入参数之前被调用，可以在此方法中添加或修改需要注入到`Bean`中的参数。

`@Resource`、`@Autowire`等注解就是在此处实现自动注入的，扫描对应字段的注解，并将获取到的参数值传入该方法返回值`PropertyValues`中，后续Spring 将自动把参数注入到对应的字段中。

如果该方法返回null，将不会执行属性的注入，也不会执行其他`InstantiationAwareBeanPostProcessor#postProcessPropertyValues`方法。



### SmartInstantiationAwareBeanPostProcessor

```java
public interface SmartInstantiationAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessor {

   Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException;

   Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) throws BeansException;

   Object getEarlyBeanReference(Object bean, String beanName) throws BeansException;

}
```

#### predictBeanType

预测并返回`Bean`的实际类型，如果无法预测可以返回`null`，一般在`BeanDefinition`无法确定`Bean`类型时调用。

#### determineCandidateConstructors

在`Bean`实例化前被调用，用来确认实例化需要调用的`构造方法`候选列表。

如果返回不为`null`，那么不会执行其他的`SmartInstantiationAwareBeanPostProcessor#determineCandidateConstructors`，Spring将会在该返回列表中选择一个方法进行调用。如果所有的`SmartInstantiationAwareBeanPostProcessor#determineCandidateConstructors`均返回`null`，那么将使用默认的`无参构造方法`进行调用.

#### getEarlyBeanReference

出现循环依赖时将会被调用，通过此方法返回一个`Bean`的引用。

### MergedBeanDefinitionPostProcessor

```java
public interface MergedBeanDefinitionPostProcessor extends BeanPostProcessor {

   void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);

}
```



#### postProcessMergedBeanDefinition

该方法在合并完`Bean`定义后被调用。

例如`AutowiredAnnotationBeanPostProcessor`将在此处解析所有带`@Autowire`注解的参数。

## BeanFactoryPostProcessor 族

### BeanFactoryPostProcessor

```java
public interface BeanFactoryPostProcessor {

   void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```

#### postProcessBeanFactory

该方法在所有`BeanDefinition`被加载完成后，`Bean`初始化之前被调用。

可以在此处修改已加载的`BeanDefinition`，或添加自定义的`BeanDefinition`，来实现动态注册`Bean`。



### BeanDefinitionRegistryPostProcessor

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
   void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}
```



#### postProcessBeanDefinitionRegistry

该方法在所有`BeanDefinition`被加载完成后，`BeanFactoryPostProcessor#postProcessBeanFactory`之前被调用。

可以在此处修改已加载的`BeanDefinition`，或添加自定义的`BeanDefinition`，来实现动态注册`Bean`；也可以在此方法中注册其他`BeanDefinitionRegistryPostProcessor`，但如果当前Bean也是被其他`BeanDefinitionRegistryPostProcessor#postProcessBeanFactory`加载的，那么当前`Bean`加载的`BeanDefinitionRegistryPostProcessor`将没有机会被执行。



## Aware 族

实现了`Aware`接口的`Bean`将会在特定的场景中被`Spring`回调，并传入对应的参数供`Bean`使用。

### BeanNameAware

#### setBeanName

该方法在`Bean`实例化完成并注入属性后，`BeanPostProcessor#postProcessBeforeInitialization`方法前被调用。

`Bean`的名称将会从入参中被传入，该name可能会被添加例如`#`之类的前缀标识符，如果想要获得原始的name，可以调用`BeanFactoryUtils#originalBeanName(String)`。

### BeanClassLoaderAware

#### setBeanClassLoader

该方法在`Bean`实例化完成并注入属性后，`BeanPostProcessor#postProcessBeforeInitialization`方法前被调用。

`Bean`的`classLoader`将会被传入。

### BeanFactoryAware

#### setBeanFactory

该方法在`Bean`实例化完成并注入属性后，`BeanPostProcessor#postProcessBeforeInitialization`方法前被调用。

持有当前`Bean`的`BeanFactory`将会被传入。





### EnvironmentAware

#### setEnvironment

该方法将被`ApplicationContextAwareProcessor#postProcessBeforeInitialization`调用，即在`Bean`实例化完成，并注入完属性后被调用，但优先于`InitializingBean#afterPropertiesSet`和`init-method`。

将从入参传入`Environment`

### EmbeddedValueResolverAware

#### setEmbeddedValueResolver

该方法将被`ApplicationContextAwareProcessor#postProcessBeforeInitialization`调用，即在`Bean`实例化完成，并注入完属性后被调用，但优先于`InitializingBean#afterPropertiesSet`和`init-method`。

将从入参传入`StringValueResolver`

### ResourceLoaderAware

#### setResourceLoader

该方法将被`ApplicationContextAwareProcessor#postProcessBeforeInitialization`调用，即在`Bean`实例化完成，并注入完属性后被调用，但优先于`InitializingBean#afterPropertiesSet`和`init-method`。

将从入参传入`ResourceLoader`

### ApplicationEventPublisherAware

#### setApplicationEventPublisher

该方法将被`ApplicationContextAwareProcessor#postProcessBeforeInitialization`调用，即在`Bean`实例化完成，并注入完属性后被调用，但优先于`InitializingBean#afterPropertiesSet`和`init-method`。

将从入参传入`ApplicationEventPublisher`

### MessageSourceAware

#### setMessageSource

该方法将被`ApplicationContextAwareProcessor#postProcessBeforeInitialization`调用，即在`Bean`实例化完成，并注入完属性后被调用，但优先于`InitializingBean#afterPropertiesSet`和`init-method`。

将从入参传入`MessageSource`

### ApplicationContextAware

#### setApplicationContext

该方法将被`ApplicationContextAwareProcessor#postProcessBeforeInitialization`调用，即在`Bean`实例化完成，并注入完属性后被调用，但优先于`InitializingBean#afterPropertiesSet`和`init-method`。

将从入参传入`ApplicationContext`





## Other 

### FactoryBean

```java
public interface FactoryBean<T> {
   T getObject() throws Exception;
   Class<?> getObjectType();
   boolean isSingleton();
}
```



`FactoryBean`是一个定义工厂`Bean`的接口，如果一个`Bean`实现了这个接口，那么`Spring`在初始化的时候会创建该类型的实例，在通过该实例创建其对应的`Bean`。

`FactoryBean`有三个方法需要子类来实现

`isSingleton`方法用来返回当前`FactoryBean`创建的`Bean`是否为单例，如果返回`true`表示是单例(`singleton`)模式的`Bean`，如果返回`false`，表示是原型模式(`prototype`)的`Bean`。

`getObjectType`方法用来返回该`FactoryBean`将会创建的`Bean`的类型。

`getObject`方法返回创建的`Bean`，当`Spring`需要通过`FactoryBean`创建一个新的`Bean`的时候，将会调用`FactoryBean#getObject`来获取`Bean`并放入`IOC`容器中。

#### AbstractFactoryBean

`AbstractFactoryBean`继承自`FactoryBean`，提供了`getObject`方法的默认实现并添加了两个抽象方法需要子类实现。一般情况下，`AbstractFactoryBean`需要实现四个方法

`isSingleton`与`FactoryBean`中的`isSingleton`方法相同。

`getObjectType`与`FactoryBean`中的`getObjectType`方法相同。

`createInstance`与`FactoryBean`中的`createInstance`方法相同，即`AbstractFactoryBean`中不需要实现`createInstance`方法。

`destroyInstance`方法将在`Bean`被销毁前调用，用来手动处理`Bean`销毁前需要处理的工作，例如关闭资源等。



### InitializingBean

```java
public interface InitializingBean {
   void afterPropertiesSet() throws Exception;
}
```

实现了`InitializingBean`的`Bean`将在实例化完成后被回调，具体时机是`Bean`属性注入完成、`BeanNameAware`、`BeanClassLoaderAware`、`BeanFactoryAware`三类`Aware`回调完成、`BeanPostProcessor#postProcessBeforeInitialization`调用完成之后，`BeanPostProcessor#postProcessAfterInitialization`之前。

此时，`Bean`已经基本初始化完成，此时可以在此方法内部做诸如参数校验、最终初始化等操作。



### DisposableBean

```java
public interface DisposableBean {
   void destroy() throws Exception;
}
```

实现了`DisposableBean`的`Bean`将在销毁前被回调。

可以在该方法内部做最后的清理工作，例如关闭资源等。

### ApplicationRunner

```java
public interface ApplicationRunner {
   void run(ApplicationArguments args) throws Exception;
}
```

`Spring`将在刷新完上下文之后，依次调用所有实现了`ApplicationRunner`接口`Bean`的'run'方法。

### CommandLineRunner

```java
public interface CommandLineRunner {
   void run(String... args) throws Exception;
}
```

`Spring`将在刷新完上下文之后，依次调用所有实现了`CommandLineRunner`接口`Bean`的`run`方法，与`ApplicationRunner`的差别是`CommandLineRunner`会将启动命令行中的参数传入`run`方法。

### ApplicationListener

```java
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
   void onApplicationEvent(E event);
}
```

实现`ApplicationListener`的`Bean`将在对应`ApplicationEvent`事件发生时被回调，对应的事件为`ApplicationListener`定义的类型参数，可以通过`@EventListener`实现类似效果。

可以通过`ApplicationEventPublisher#publishEvent`发布消息

### Lifecycle

```java
public interface Lifecycle {

   void start();

   void stop();

   boolean isRunning();

}
```

实现了`Lifecycle`接口的`Bean`，容器会在生命周期中的不同阶段回调对应的方法，可以在这些方法中执行特定的操作。

`start`会在容器刷新的最后阶段被调用，具体时机是创建完所有的`Bean`之后，`ContextRefreshedEvent`消息广播、调用`ApplicationRunner#run`方法之前。

`stop`会在容器关闭前被调用，具体时机是`ContextClosedEvent`消息广播之后，`DisposableBean#destory`方法调用之前。

`isRunning`用来返回当前`Bean`是否是运行中，如果返回`true`，那么不会执行`start`方法，会执行`stop`方法；如果返回`false`，那么会执行`start`方法，不会执行`stop`方法。

### ImportBeanDefinitionRegistrar

```java
public interface ImportBeanDefinitionRegistrar {

   public void registerBeanDefinitions(
         AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry);

}
```

该接口的作用类似与`BeanDefinitionRegistryPostProcessor`，可以在此处修改已加载的`BeanDefinition`，或添加自定义的`BeanDefinition`，来实现动态注册`Bean`；

不同于其他的组件只需要注入`IOC`容器即可被自动发现并被调用，`BeanDefinitionRegistryPostProcessor`子类需要通过`@Import`注解引入方可使用。

`registerBeanDefinitions`方法将会被`ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry`调用，`ConfigurationClassPostProcessor`是通过实现了`BeanDefinitionRegistryPostProcessor`接口而获得注册`BeanDefinition`的能力，所以被调用时机与`BeanDefinitionRegistryPostProcessor`一致。



