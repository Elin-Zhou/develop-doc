## 代理

### ProxyFactory

当需要手动给`Bean`创建代理对象时，一般情况下通过CGLib或动态代理实现，在有些情况下还需要根据对象的类型区分使用生成代理的方式，可以通过`ProxyFactory`快速生成一个代理类，并支持多个切面。

```java
public class Bean {
    public String hello() {
        return "hello";
    }

    public static void main(String[] args) {
        ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.setTarget(new Bean());
        proxyFactory.addAdvice((MethodInterceptor) invocation -> {
            if (!"hello".equals(invocation.getMethod().getName())) {
                return invocation.proceed();
            }
            return "elin, " + invocation.proceed() + "!";
        });
        Object proxy = proxyFactory.getProxy();
        System.out.println(((Bean) proxy).hello());
    }
}
```

### AopUtils

在`Spring`中，通过一些方式获取到的`Bean`可能已经被增强，即可能被代理，获取被代理的对象可能会有一些意想不到的情况发生，例如通过代理对象获取类注解时可能将会获取不到。

`AopUtils`提供一些方法来判断是否为代理对象，例如`AopUtils#isAopProxy`；也可以调用`AopUtils#getTargetClass`获取到被代理的对象类型等等。



## Bean

### BeanDefinitionBuilder

有些场景下，需要手动注册`BeanDefinition`，可以通过`BeanDefinitionBuilder`进行创建`BeanDefinition`。建议通过`BeanDefinitionBuilder#genericBeanDefinition`创建，因为`GenericBeanDefinition`涵盖了`RootBeanDefinition`和`ChildBeanDefinition`的功能。

```java
BeanDefinitionBuilder definitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(Bean.class);
//通过变量名设置属性
definitionBuilder.addPropertyValue("name", "elin");
AbstractBeanDefinition beanDefinition = definitionBuilder.getBeanDefinition();
```



### ClassPathBeanDefinitionScanner

在一些场景下，可能需要手动扫描某个路径下的类进行处理。例如`MyBatis`通过扫描某个路径下的mapper接口自动生成`mapper`实例。

`ClassPathBeanDefinitionScanner#scan(String)`将会扫描入参路径，将其中的类解析为`BeanDefinition`注册到当前容器中。

```java
BeanDefinitionRegistry registry = null;
ApplicationContext applicationContext = null;

ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(registry);
scanner.setResourceLoader(applicationContext);
//此处设置扫描包含类的过滤去，AnnotationTypeFilter将会通过注解进行过滤，举个例子，只有加了Mapper注解的类会被注册进来
scanner.addIncludeFilter(new AnnotationTypeFilter(Mapper.class));
//要扫描的包路径
scanner.scan("com.xxelin.mappers");
```



如果有必要，例如修改扫描到的`BeanDefinition`，可以重写`doScan`方法；如果不满足于`addIncludeFilter`或`addExcludeFilter`两类过滤器，可以重写`isCandidateComponent`方法手动写条件进行过滤。