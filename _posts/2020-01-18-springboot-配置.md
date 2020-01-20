---
layout: post
title:  "springboot配置"
categories: springboot
tags: springboot
author: 网络
---

* content
{:toc}











## spring中访问配置的几种方式

### 本地配置文件读取

* Environment#getProperty

环境变量包括：system properties、-D参数、application.properties(.yml)、通过@PropertySource引入的配置文件配置，它的原理其实就是从内存中取出PropertySource列表，遍历取配置，可以查看AbstractEnvironment.getProperty方法源码。

* 直接读取properties本地文件
* @Value，配合@PropertySource
* @ConfigurationProperties，配合@PropertySource

#### 示例代码

针对上面的4种方式基于springboot项目做的[示例代码](https://gitee.com/qigangzhong/springboot-demo/tree/master/config/config-basics)。

#### 配置注入原理分析

spring-beans包下面有一个AutowiredAnnotationBeanPostProcessor类，注入@Value，@Autowired属性值就是靠它来完成的。

```java
SpringApplication.run
    refresh
    AbstractApplicationContext.refresh
        finishBeanFactoryInitialization
        DefaultListableBeanFactory.preInstantiateSingletons
            AbstractBeanFactory.getBean
                doGetBean
                AbstractAutowireCapableBeanFactory.createBean
                doCreateBean
                    createBeanInstance
                applyMergedBeanDefinitionPostProcessors

AutowiredAnnotationBeanPostProcessor.postProcessMergedBeanDefinition
    findAutowiringMetadata
        buildAutowiringMetadata
            //这里通过反射的方式遍历所有的filed和method，检查是否打了@Autowired、@Value标签
            //得到AutowiredFieldElement、AutowiredMethodElement的集合，最后都缓存到LinkedList<InjectionMetadata.InjectedElement>链表里面，然后包装到InjectionMetadata对象中
            ReflectionUtils.doWithLocalFields
            ReflectionUtils.doWithLocalMethods



//回到AbstractAutowireCapableBeanFactory.populateBean
AbstractAutowireCapableBeanFactory.populateBean
    AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues

        findAutowiringMetadata
        InjectionMetadata.inject
        //在这个方法里面通过反射的方式将值注入到@Autowired，@Value中
        AutowiredAnnotationBeanPostProcessor.inject
        //解析@Value("${XXX}")中的值的方法是通过PropertySourcesPlaceholderConfigurer.resolveStringValue
        //最终还是通过PropertySourcesPropertyResolver.getPropertyAsRawString来获取配置的值的，然后反射注入到@Value字段中
```

### 本地配置文件刷新

#### 1. 使用定时器自动刷新PropertySource

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

```java
@Component
public class PropertyLoader {

    @Autowired
    private StandardEnvironment environment;

    @Scheduled(fixedRate=1000)
    public void reload() throws IOException {
        MutablePropertySources propertySources = environment.getPropertySources();
        String propertiesFilePropertySoureName = "applicationConfig: [classpath:/application.properties]";
        Properties properties = new Properties();
        InputStream inputStream = getClass().getResourceAsStream("/application.properties");
        properties.load(inputStream);
        inputStream.close();
        propertySources.replace(propertiesFilePropertySoureName, new PropertiesPropertySource(propertiesFilePropertySoureName, properties));
    }
}
```

```java
@SpringBootApplication
@EnableScheduling
@PropertySource("classpath:/application.properties")
public class MainApp  {
    public static void main(String[] args) {
        SpringApplication.run(MainApp.class, args);
    }
}
```

```java
@RestController
public class DemoController {
    @Autowired
    private StandardEnvironment environment;

    @Value("${demo.name}")
    private String name;

    @GetMapping("/getNameFromEnvironment")
    public String getNameFromEnvironment(){
        return environment.getProperty("demo.name");
    }

    /**
     * 配置文件修改后这个方式无法获取到最新的配置值
     * 因为@Value的字段值是在spring容器启动的时候注入，通过刷新PropertySource的方式无法重新注入值
     * @return
     */
    @GetMapping("/getNameFromValue")
    public String getNameFromValue(){
        return name;
    }
}
```

这种方式有一个缺点就是，无法使用@Value的方式获取配置值，因为@Value的字段值注入之后并不会被刷新，只能通过`environment.getProperty("key")`的方式。

[示例代码](https://gitee.com/qigangzhong/springboot-demo/tree/master/config/config-refresh)

#### 2. EnvironmentPostProcessor

[示例代码](https://gitee.com/qigangzhong/share.demo/tree/master/environment-post-processor-demo)

#### 3. PropertySourceLocator

[示例代码](https://gitee.com/qigangzhong/share.demo/tree/master/property-source-locator-demo)

#### 4. Archaius PolledConfigurationSource

[示例代码](https://gitee.com/qigangzhong/share.demo/tree/master/springboot-archaius-demo)

#### 

## 参考

[Spring Boot @ConfigurationProperties: Binding external configurations to POJO classes](https://www.callicoder.com/spring-boot-configuration-properties-example/)
