

## BeanPostProcessor 族

### BeanPostProcessor

```java
public interface BeanPostProcessor {
   Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
   Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```



#### postProcessBeforeInitialization

该方法将在Bean实例化完成，并注入完属性后被调用，但优先于InitializingBean#afterPropertiesSet和init-method。

可以在该方法中返回包装对象或代理对象。

如果此方法返回`null`，那么后续的BeanPostProcessor将不会被调用。

#### postProcessAfterInitialization

该方法在postProcessBeforeInitialization、InitializingBean#afterPropertiesSet、init-method调用完成之后被调用。

可以在该方法中返回包装对象或代理对象。

如果此方法返回`null`，那么后续的BeanPostProcessor将不会被调用。

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

该方法在实例化Bean之后，注入属性之前被调用。

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

出现循环依赖时将会被调用，通过此方法返回一个Bean的引用。

### MergedBeanDefinitionPostProcessor

```java
public interface MergedBeanDefinitionPostProcessor extends BeanPostProcessor {

   void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);

}
```



#### postProcessMergedBeanDefinition

该方法在合并完Bean定义后被调用。

例如`AutowiredAnnotationBeanPostProcessor`将在此处解析所有带`@Autowire`注解的参数。

### 

## BeanFactoryPostProcessor 族

### BeanFactoryPostProcessor

### BeanDefinitionRegistryPostProcessor



## Aware 族

### EnvironmentAware

### EmbeddedValueResolverAware

### ResourceLoaderAware

### ApplicationEventPublisherAware

### MessageSourceAware

### ApplicationContextAware



### BeanNameAware

### BeanClassLoaderAware

### BeanFactoryAware





## Other 

### InitializingBean

### ApplicationListener

### SmartLifecycle

### ApplicationRunner

### WebMvcConfigurerAdapter

### ImportBeanDefinitionRegistrar



