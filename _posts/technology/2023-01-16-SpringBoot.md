---
layout:     post
title:      "SpringBoot"
subtitle:   "SpringBoot"
date:       2023-01-16 12:00:00
author:     "Pengyujie"
header-img: "img/tag-bg.jpg"
catalog: true
tags:
    - springboot
    - Java

---



## SpringBoot



### 自动配置原理

因此springboot底层实现自动配置的步骤是：

springboot应用启动；

- @SpringBootApplication起作用 @SpringBootApplication是一个联合注解；
- @EnableAutoConfiguration起作用；
- @AutoConfigurationPackage：这个组合注解主要是@Import(AutoConfigurationPackages.Registrar.class)，它通过将Registrar类导入到容器中，而Registrar类作用是扫描主配置类同级目录以及子包，并将相应的组件导入到springboot创建管理的容器中；
- @Import(AutoConfigurationImportSelector.class)：它通过将AutoConfigurationImportSelector类导入到容器中，AutoConfigurationImportSelector类作用是通过selectImports方法实现将配置类信息交给SpringFactory加载器进行一系列的容器创建过程（SPI）



### 最先执行

~~~java
最新需要在项目启动后立即执行某个方法，然后特此记录下找到的四种方式

    
    
注解@PostConstruct
使用注解@PostConstruct是最常见的一种方式，存在的问题是如果执行的方法耗时过长，会导致项目在方法执行期间无法提供服务。

@Component
public class StartInit {

//    @Autowired   可以注入bean
//    ISysUserService userService;

    @PostConstruct
    public void init() throws InterruptedException {
        Thread.sleep(10*1000);//这里如果方法执行过长会导致项目一直无法提供服务
        System.out.println(123456);
    }
}




CommandLineRunner接口
实现CommandLineRunner接口 然后在run方法里面调用需要调用的方法即可，好处是方法执行时，项目已经初始化完毕，是可以正常提供服务的。

同时该方法也可以接受参数，可以根据项目启动时: java -jar demo.jar arg1 arg2 arg3 传入的参数进行一些处理。详见： Spring boot CommandLineRunner启动任务传参

@Component
public class CommandLineRunnerImpl implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        System.out.println(Arrays.toString(args));
    }
}



实现ApplicationRunner接口
实现ApplicationRunner接口和实现CommandLineRunner接口基本是一样的。

唯一的不同是启动时传参的格式，CommandLineRunner对于参数格式没有任何限制，ApplicationRunner接口参数格式必须是：–key=value

@Component
public class ApplicationRunnerImpl implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        Set<String> optionNames = args.getOptionNames();
        for (String optionName : optionNames) {
            List<String> values = args.getOptionValues(optionName);
            System.out.println(values.toString());
        }
    }
}


实现ApplicationListener
实现接口ApplicationListener方式和实现ApplicationRunner，CommandLineRunner接口都不影响服务，都可以正常提供服务，注意监听的事件，通常是ApplicationStartedEvent 或者ApplicationReadyEvent，其他的事件可能无法注入bean。

@Component
public class ApplicationListenerImpl implements ApplicationListener<ApplicationStartedEvent> {
    @Override
    public void onApplicationEvent(ApplicationStartedEvent event) {
        System.out.println("listener");
    }
}



四种方式的执行顺序
注解方式@PostConstruct 始终最先执行

如果监听的是ApplicationStartedEvent 事件，则一定会在CommandLineRunner和ApplicationRunner 之前执行。

如果监听的是ApplicationReadyEvent 事件，则一定会在CommandLineRunner和ApplicationRunner 之后执行。

CommandLineRunner和ApplicationRunner 默认是ApplicationRunner先执行，如果双方指定了@Order 则按照@Order的大小顺序执行，大的先执行。
    
~~~



![3](/img/notes/springboot/3.png)



### 属性绑定

WxApplication启动类

~~~java
package com.example.wx;

import com.alibaba.fastjson.JSONObject;
import com.example.wx.springboot.properties.MyProperties;
import com.example.wx.springboot.MyUserService;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;

@SpringBootApplication
/**
 * 如果配置文件名称 改变可以使用@PropertySource注解
 */
//@PropertySource("application_pengyujie.properties")
public class WxApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext applicationContext = SpringApplication.run(WxApplication.class, args);

        /**
         * @Value注解属性绑定  一般不会用这种
         */
        MyUserService bean = applicationContext.getBean(MyUserService.class);
        System.out.println(JSONObject.toJSONString(bean));

        /**
         * @ConfigurationPropertiesScan("com.example.wx.springboot.properties") 注解扫描进行属性绑定
         */
        MyProperties bean1 = applicationContext.getBean(MyProperties.class);
        System.out.println(JSONObject.toJSONString(bean1));


    }

}

~~~





AppConfig配置类 

配置上二个注解

@Configuration
@ConfigurationPropertiesScan("com.example.wx.springboot.properties")

~~~java
package com.example.wx.springboot.config;


import org.springframework.boot.context.properties.ConfigurationPropertiesScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConfigurationPropertiesScan("com.example.wx.springboot.properties")
public class AppConfig {

}

~~~



**第一种@ConfigurationProperties(prefix = "pengyujie")注解属性绑定**

MyProperties类 会将配置文件对应的属性自动注入到该Properties类

需要绑定的配置类

~~~java
package com.example.wx.springboot.properties;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * prefix 前缀名称
 */
@ConfigurationProperties(prefix = "pengyujie")
@Data
public class MyProperties {
    private String name;

    private String age;

    private String birthirday;
}

~~~



#### 第二种**@Value注解属性绑定**

通过@Value("${pengyujie.name}") 将配置文件中对应的文件名称的value 注入到对应的属性

MyUserService类

~~~java
package com.example.wx.springboot;

import lombok.Data;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
@Data
public class MyUserService {
    @Value("${pengyujie.name}")
    private String name;

    @Value("${pengyujie.age}")
    private String age;

    @Value("${pengyujie.birthirday}")
    private String birthirday;
}

~~~







### 配置文件优先级



优先级

properties文件>yml文件



设置的环境变量的优先级

JVM环境变量>操作系统环境变量>properties文件环境变量>yml文件环境变量





### Profiles机制



#### 不同配置文件：

![image-20220720094400031](/img/notes/springboot/1.png)

![image-20220720094400031](/img/notes/springboot/2.png)

指定生效配置文件即可





~~~java
package com.example.wx.springboot.config;


import com.example.wx.springboot.MyUserService;
import org.springframework.boot.context.properties.ConfigurationPropertiesScan;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Configuration
@ConfigurationPropertiesScan("com.example.wx.springboot.properties")
public class AppConfig {
//
//    @Bean
//    @Profile("dev")
//    public MyUserService myUserService_dev(){
//        return new MyUserService();
//    }
//
//
//    @Bean
//    @Profile("prod")
//    public MyUserService myUserService_prod(){
//        return new MyUserService();
//    }
//
//


//    @Bean
//    /**
//     * 只有当配置文件里 存在spring.profiles.active=cs环境的条件下才会进行执行
//     */
//    @Profile("cs")
//    public void myUserService_cs(){
//        System.out.println("cs");
//    }

}

~~~















# Spring



IOC和AOP的实现原理与理解

spring的学习是看的周瑜老师的视频，如果不清楚可以看下周瑜老师的高级spring底层原理源码教程

[周瑜老师的思维导图笔记](https://www.processon.com/view/5fb7681f7d9c0857dda6f740)  





@PostConstruct 注解是在创建Bean的时候，spring会去识别对应的注解如果是该注解，就会先执行该方法再去创建Bean



### 我的理解：

#### IOC：控制反转 

也就是将对象的创建(Bean)的权限交由spring去管理。

实现：通过读取配置文件的扫描路径或者是注解，将需要创建的Bean的类，先生成一个beandefinition对象存放于一个Map中，然后再循环去生成Bean对象，首先会通过判断该类的构造方法去生成对应Bean的空对象（这里如果有构造方法注解@Autowired，默认用该构造方法生成对象，如果没有注解且只有一个构造方法就使用该构造方法，如果有多个则默认使用无参构造方法去生成），在生成Bean空对象之后会对对象进行依赖注入（如果产生循环依赖的问题可以通过三级缓存解决），注入完成之后就生成了bean,然后再放入一个Map也就是单例池。如果需要使用对应的对象那么就可以通过注解注入，或者getBean去获取。



#### AOP：面向切面 

也就是在不改原有逻辑的前提下，把原来的方法逻辑当做一个切面可以在前或后加入自己的业务逻辑处理比如日志之类的。

实现：原理实际是动态代理也就是在生成对应Bean的时候，存入Map单例池中的不是普通的Bean对应，而是这个Bean对应的代理对象，我们调用代理对象的方法的时候实际还会去执行其他方法然后再去执行我们需要执行的方法。







三级缓存解决循环依赖问题

实际是三个Map

一级缓存：singletonObjects  经历了完整Bean周期的代理对象

二级缓存：earlySingletonObjects  存入的是没有经历完整bean周期的代理对象

三级缓存：singletonFactories   存入的是没有经历完整bean周期的普通对象



二级缓存和三级缓存是为了效率  三级缓存存的是ObjectFactory ，二级缓存存的是ObjectFactory生成的还属性未完全注入的Bean

之所以存在三级缓存，是因为有的Bean并不一定会被使用，如果只有二级缓存，那么所有的Bean不管使用与否都需要用ObjectFactory 生成对应的bean再存入二级缓存中，那就会造成很多Bean其实最终并没有使用，但是还是通过ObjectFactory执行去创建出来了。

之所以有二级缓存，是因为有的Bean需要被多次使用，这时候每次都通过三级缓存获取ObjectFactory去执行获取显然效率太低，于是出现了一个二级缓存来存储这个Bean，下次需要注入的时候直接从二级缓存获取即可。



如果**没有循环依赖**问题

**举例：Aservice 中需要注入Bservice属性   Bservice 中需要注入其他属性** 

1.首先创建一个空Aservice 之后，将其放入三级缓存和一个创建中的Map，

2.然后对这个空Aservice的属性进行注入，查看一级缓存中是否有对应的Bservice对象，有则直接注入属性，没有则查看注入的属性是否为创建中（是否在创建中的Map中），不在创建中，不存在循环依赖问题，直接创建对应的Bservice（过程如下）。

1. *先创建一个空Bservice 之后，将其放入三级缓存和一个创建中的Map，*
2. *然后对这个空Bservice的属性进行注入，查看一级缓存中是否有对应的其他对象，有则直接注入属性，没有则查看注入的属性是否为创建中（是否在创建中的Map中），不在创建中，不存在循环依赖问题，直接创建对应的Bean进行注入。*
3. *还有其他属性也同样进行注入，AOP生成对应的代理对象*
4. *放入单例池中*

3.将Bservice属性注入，还有其他属性也同样进行注入，AOP生成对应的代理对象

4.放入单例池中





如果**有循环依赖**问题

**举例：Aservice 中需要注入Bservice属性  Bservice 中需要注入Aservice属性** 

1.首先创建一个空Aservice 之后，将其放入三级缓存和一个创建中的Map，

2.然后对这个空Aservice 的属性进行注入，查看一级缓存中是否有对应的Bservice对象，有则直接注入属性，没有则查看注入的属性是否为创建中（是否在创建中的Map中），不在创建中，不存在循环依赖问题，直接创建对应的Bservice（过程如下）。

1. *先创建一个空Bservice 之后，将其放入三级缓存和一个创建中的Map，*
2. *然后对这个空Bservice的属性进行注入，查看一级缓存中是否有对应的Aservice对象，有则直接注入属性，没有则查看注入的属性是否为创建中（是否在创建中的Map中），在创建中，说明存在循环依赖问题，首先查询二级缓存中是否有对应对象，有则直接使用，没有则查看三级缓存是否存在（一般单例在三级缓存都能查到），有直接使用三级缓存中的对象（三级缓存存放的是一个lambda表达式 会AOP生成对应的代理对象）存入二级缓存中（存入之后将三级存储的对象移除调），将lambda生成的代理对象注入到对应的属性*属性注入
3. 还有其他属性也同样进行注入，AOP生成对应的代理对象
4. *放入单例池中*

3.将Bservice属性注入，还有其他属性也同样进行注入，AOP生成对应的代理对象

4.放入单例池中







# Bean的生命周期



实例化
程序启动后，Spring把注解或者配置文件定义好的Bean对象转换成一个BeanDefination对象，然后完成整个BeanDefination的解析和加载的过程。Spring获取到这些完整的对象之后，会对整个BeanDefination进行实例化操作，实例化是通过反射的方式创建对象。

属性设置
实例化后的对象被封装在BeanWrapper对象中，并且此时对象仍然是一个原生的状态，并没有进行依赖注入。Spring根据BeanDefinition中的信息进行依赖注入, populateBean方法来完成属性的注入。

初始化
1、调用Aware接口相关的方法：invokeAwareMethod(完成beanName, beanClassLoader, beanFactory对象的属性设置)
2、调用beanPostProcessor中的前置处理方法（applyBeanPostProcessorsBeforeInitialization）
3、调用InitMethod方法：invokeInitMethod(),判断是否实现了initializingBean接口，如果有，调用afterPropertiesSet方法，没有就不调用
4、调用BeanPostProcessor后置处理方法（applyBeanPostProcessorsAfterInitialization），Spring 的Aop就是在此处实现的

销毁
判断是否实现了DisposableBean接口，调用destoryMethod方法



![image-20220720094400031](/img/notes/springboot/4.png)



小结
main方法启动后，Spring读@Bean注解，将注解修饰的对象加载到IOC容器，IOC容器将其转化为BeanDefination对象，并进行实例化，实例化后封装为BeanWrapper对象。然后Spring调用populateBean方法对BeanDefination对象进行属性填充，再调用initializeBean方法完成一些Aware操作，然后调用beanPostProcessor中的前置处理方法，如果实现了initializingBean，则调用afterPropertiesSet方法，然后调用BeanPostProcessor后置处理方法，Aop就是在此处实现的。处理完成后，bean就处于一个就绪状态，等待被调用。销毁时判断是否实现了DisposableBean接口，调用destoryMethod方法。





