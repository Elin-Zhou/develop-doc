本文将描述如何通过BeanPostProcessor以及与其相关的技术来生成接口的实现类并自动注入bean中。实现的效果类似于Mybatis的mapper仅需要定义接口，而无需手动生成实现类。

阅读本文之前，请先了解学习`BeanPostProcessor`、`InstantiationAwareBeanPostProcessor`以及`BeanDefinitionRegistryPostProcessor`，可以参考此文[Spring支持的扩展接口](./Spring支持的扩展接口.md)；同时需要了解`FactoryBean`与`ClassPathBeanDefinitionScanner`的概念，参考[Spring常用工具](./Spring常用工具.md)

为了结合实际需求，请读者设想如下场景：在对接某第三方公司时，对方提供了Http接口以供调用，如果使用常规方式，需要在代码中拼接url，然后使用诸如HttpClient一类的工具获取结果响应。如果这样的代码散落在项目当中，将会非常影响观感，有些同学会做一些封装，为每个http接口封装一个工具方法以供调用，但是需要写很多接口及其实现类，其代码大同小异，本文就将对此场景进行封装，最终实现仅需要开发同学创建对应的接口（Interface），无需写实现类，在调用端直接用@Autowire注入即可调用。

本文主要是以学习为目的而并非解决实际问题，上述的场景也只是工作中很简单的一个场景，所以本文将使用两种方式来实现该需求，以记录更多有用的知识点。



## 分析需求


再简单看一下我们的需求，整理一下主要是要实现两点
1. 创建接口的实现类
2. 将实现类注入bean中

其中的第二点其实不需要刻意编写代码来实现，只要由Spring容器来创建对应的代理类，再通过@Autowire就已经可以自动注入了（额外讲一个知识点，@Autowire注入是由AutowiredAnnotationBeanPostProcessor来实现的，也是本文的主角BeanPostProcessor的一个实现类）

所以，我们只需要考虑怎么在Spring体系下载容器中创建一个代理类就好，了解Spring容器启动原理的同学都知道，Spring创建一个bean主要分为两步，先获取对应bean的BeanDefinition，然后再通过BeanDefinition来创建对应实例。

方法一将描述如何通过FactoryBean创建代理类，这种是比较常见的动态创建实现类的方式，例如Mybatis就是通过此方式。方法二是通过InstantiationAwareBeanPostProcessor来创建，根据前面讲过的InstantiationAwareBeanPostProcessor的简介可以了解到，InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法会被Spring在创建实例前调用，如果该方法返回了不为null的对象，则容器将直接使用该对象而不执行后续的实例化代码。

## 实现

### 准备工作

因为要模拟调用http接口，所以这里提供了两个http方法，分别有一个和两个参数，返回参数简单起见使用了String类型。

```
package com.xxelin.mammon.http;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/say")
public class HelloController {

    @RequestMapping("/hello")
    public String hello(@RequestParam("name") String name) {
        return "hello " + name + " !";
    }

    @RequestMapping("/something")
    public String saySomething(@RequestParam("something") String something, @RequestParam("name") String name) {
        return String.format("%s say:%s", name, something);
    }

}

```


定义一个注解，用来标记是http接口，有一个value参数，表示实际调用的接口地址
```
package com.xxelin.mammon.http;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RemoteService {
    /**
     * 接口地址
     *
     * @return
     */
    String value();
}
```

根据http接口实现一个java接口，这个接口就是最终可以直接调用的，偷个懒，复用了springmvc中的RequestMapping和RequestParam两个注解，这里已经改变了原来的语义，自定义一个新的也是可以的，所以不要奇怪为什么要用这两个注解了。
```
package com.xxelin.mammon.http;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;

@RemoteService("http://127.0.0.1:4096/say")
public interface HelloService {

    //懒得重新写新的注解了，这里复用一下RequestMapping和RequestParam两个注解，分别表示请求地址和参数名称
    @RequestMapping("/hello")
    String hello(@RequestParam("name") String name);

    @RequestMapping("/something")
    String saySomething(@RequestParam("something") String something, @RequestParam("name") String name);
}
```


### 方法一：使用FactoryBean创建代理类


#### 第一步：创建FactoryBean
```
package com.xxelin.mammon.http;

import com.alibaba.fastjson.JSON;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.util.EntityUtils;
import org.springframework.beans.factory.FactoryBean;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;

import java.lang.reflect.Parameter;
import java.lang.reflect.Proxy;
import java.nio.charset.Charset;

public class RemoteServiceFactoryBean<T> implements FactoryBean<T> {

    private Class<T> objectType;

    public RemoteServiceFactoryBean() {
    }

    public RemoteServiceFactoryBean(Class<T> objectType) {
        this.objectType = objectType;
    }

    @Override
    public T getObject() throws Exception {
        //spring在初始化bean的时候发现该bean实现了FactoryBean接口，就会调用其getObject()方法来获取实例

        //此处使用动态代理来实现，也可以使用CGLIB，为了方便就用动态代理了
        return (T) Proxy.newProxyInstance(this.getClass().getClassLoader(), new Class[]{objectType},
                (proxy, method, args) -> {

                    //下面的代码写的比较丑陋，因为不是这次的重点，不要吹毛求疵了
                    Class<?> returnType = method.getReturnType();
                    RemoteService remoteService = objectType.getAnnotation(RemoteService.class);
                    if (remoteService == null) {
                        throw new IllegalStateException();
                    }
                    RequestMapping path = method.getAnnotation(RequestMapping.class);
                    if (path == null) {
                        throw new IllegalStateException();
                    }
                    StringBuilder sb = new StringBuilder(remoteService.value()).append(path.value()[0]).append("?");
                    Parameter[] parameters = method.getParameters();
                    int i = 0;
                    for (Parameter parameter : parameters) {
                        RequestParam param = parameter.getAnnotation(RequestParam.class);
                        if (param == null) {
                            throw new IllegalStateException();
                        }
                        //只是做个演示，这里不考虑参数的类型转化问题了，都使用String
                        sb.append(param.value()).append("=").append(args[i]).append("&");
                        i++;
                    }

                    CloseableHttpClient httpClient = HttpClientBuilder.create().build();
                    CloseableHttpResponse response = httpClient.execute(new HttpGet(sb.toString()));
                    String data = EntityUtils.toString(response.getEntity(), Charset.defaultCharset());
                    if (String.class.isAssignableFrom(returnType)) {
                        return data;
                    }
                    return JSON.parseObject(data, returnType);
                });
    }

    @Override
    public Class<T> getObjectType() {
        return this.objectType;
    }
}
```

#### 第二步：注册BeanDefinition
```
package com.xxelin.mammon.http;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.annotation.AnnotatedBeanDefinition;
import org.springframework.beans.factory.config.BeanDefinitionHolder;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor;
import org.springframework.beans.factory.support.GenericBeanDefinition;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.annotation.ClassPathBeanDefinitionScanner;
import org.springframework.core.type.filter.AnnotationTypeFilter;
import org.springframework.stereotype.Component;

import java.util.Set;

@Component
public class RemoteServiceRegistryProcessor implements BeanDefinitionRegistryPostProcessor, ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        //Spring会主动调用所有实现了BeanDefinitionRegistryPostProcessor接口的bean的postProcessBeanDefinitionRegistry方法

        //使用了ClassPathBeanDefinitionScanner来扫描指定包路径下的所有类，并将其注册为BeanDefinition
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(registry) {
            @Override
            protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
                return beanDefinition.getMetadata().isInterface() && beanDefinition.getMetadata().isIndependent();
            }

            @Override
            protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
                Set<BeanDefinitionHolder> beanDefinitionHolders = super.doScan(basePackages);

                GenericBeanDefinition definition;
                for (BeanDefinitionHolder definitionHolder : beanDefinitionHolders) {
                    definition = (GenericBeanDefinition) definitionHolder.getBeanDefinition();
                    //此处额外做了两件事，第一是修改当前BeanDefinition的class为RemoteServiceFactoryBean
                    //因为默认扫描生成的RemoteServiceFactoryBean的class是被扫描到的类，但是次类（接口）的实现类是由RemoteServiceFactoryBean创建的
                    //所以需要修改其类型，则spring在根据该BeanDefinition创建bean的时候会先创建RemoteServiceFactoryBean对象，在通过FactoryBean创建目标对象
                    //第二件事就是把实际需要的对象的类名作为构造方法的参数传入，可以看一下RemoteServiceFactoryBean的构造方法需要一个参数，spring会自动将类名转化为Class对戏
                    definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName()); // issue
                    definition.setBeanClass(RemoteServiceFactoryBean.class);
                }

                return beanDefinitionHolders;
            }
        };
        scanner.setResourceLoader(this.applicationContext);
        //此处设置扫描包含类的过滤去，AnnotationTypeFilter将会通过注解进行过滤，只有加了RemoteService注解的类会被注册进来
        scanner.addIncludeFilter(new AnnotationTypeFilter(RemoteService.class));
        //要扫描的包路径
        scanner.scan("com.xxelin.mammon.http");
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        //do nothing
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```

### 方法二：使用InstantiationAwareBeanPostProcessor创建代理类


#### 第一步：注册BeanDefinition

这一步跟第一种方式差不多，但是有细小的差别——不需要重写doScan方法，详细原因请看代码中的注释
```
package com.xxelin.mammon.http;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.annotation.AnnotatedBeanDefinition;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.annotation.ClassPathBeanDefinitionScanner;
import org.springframework.core.type.filter.AnnotationTypeFilter;
import org.springframework.stereotype.Component;

@Component
public class RemoteServiceRegistryProcessor implements BeanDefinitionRegistryPostProcessor, ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        //Spring会主动调用所有实现了BeanDefinitionRegistryPostProcessor接口的bean的postProcessBeanDefinitionRegistry方法

        //使用了ClassPathBeanDefinitionScanner来扫描指定包路径下的所有类，并将其注册为BeanDefinition
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(registry) {
            @Override
            protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
                return beanDefinition.getMetadata().isInterface() && beanDefinition.getMetadata().isIndependent();
            }
            //此处有别于第一种使用FactoryBean实现的方式，不需要覆盖doScan方法
            //因为第一种方式需要动过FactoryBean来创建目标对象，所以需要先创建FactoryBean，但此处直接创建目标对象，所以不需要修改BeanDefinition
        };
        scanner.setResourceLoader(this.applicationContext);
        //此处设置扫描包含类的过滤去，AnnotationTypeFilter将会通过注解进行过滤，只有加了RemoteService注解的类会被注册进来
        scanner.addIncludeFilter(new AnnotationTypeFilter(RemoteService.class));
        //要扫描的包路径
        scanner.scan("com.xxelin.mammon.http");
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        //do nothing
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```


#### 第二步：实现InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation
简单期间，直接在第一部使用的RemoteServiceRegistryProcessor类中，实现InstantiationAwareBeanPostProcessor接口并覆盖postProcessBeforeInstantiation方法
```
@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
    if (beanClass.isAnnotationPresent(RemoteService.class)) {

        return Proxy.newProxyInstance(this.getClass().getClassLoader(), new Class[]{beanClass},
                (proxy, method, args) -> {

                    //下面的代码写的比较丑陋，因为不是这次的重点，不要吹毛求疵了
                    Class<?> returnType = method.getReturnType();
                    RemoteService remoteService = beanClass.getAnnotation(RemoteService.class);
                    if (remoteService == null) {
                        throw new IllegalStateException();
                    }
                    RequestMapping path = method.getAnnotation(RequestMapping.class);
                    if (path == null) {
                        throw new IllegalStateException();
                    }
                    StringBuilder sb = new StringBuilder(remoteService.value()).append(path.value()[0]).append("?");
                    Parameter[] parameters = method.getParameters();
                    int i = 0;
                    for (Parameter parameter : parameters) {
                        RequestParam param = parameter.getAnnotation(RequestParam.class);
                        if (param == null) {
                            throw new IllegalStateException();
                        }
                        //只是做个演示，这里不考虑参数的类型转化问题了，都使用String
                        sb.append(param.value()).append("=").append(args[i]).append("&");
                        i++;
                    }

                    CloseableHttpClient httpClient = HttpClientBuilder.create().build();
                    CloseableHttpResponse response = httpClient.execute(new HttpGet(sb.toString()));
                    String data = EntityUtils.toString(response.getEntity(), Charset.defaultCharset());
                    if (String.class.isAssignableFrom(returnType)) {
                        return data;
                    }
                    return JSON.parseObject(data, returnType);
                });
    }
    return null;
}
```

可以发现，这个方法的实现与第一种方案中RemoteServiceFactoryBean类的getObject()方法基本一致，因为这两个方法都会被Spring调用，来获得Spring需要的bean。

## 使用

通过@Autowire或者@Resource把HelloService类注入需要使用的地方，即可直接调用HelloService的方法进行使用。

## 总结

其实第一种方案中，可以用ImportBeanDefinitionRegistrar接口来代替BeanDefinitionRegistryPostProcessor，两种方式大同小异，再次不再赘述，感兴趣的读者可以自行实现。

本文中的代码不多，难度也不大，主要是需要了解Spring的一些机制比如BeanDefinition是什么，如何注册，在什么时机返回代理类等等，了解了这些内容以后，实现本文的功能也就比较容易了。