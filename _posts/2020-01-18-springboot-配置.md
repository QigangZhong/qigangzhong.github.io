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

#### 5. 类似appolo的方式，反射@Value字段并定时更新值

[Apollo 2 如何支持 @Value 注解自动更新](https://www.jianshu.com/p/502bc54f86c1)

## springboot自动配置原理

### @EnableAutoConfiguration

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
	Class<?>[] exclude() default {};
	String[] excludeName() default {};
}
```

通过@Import的方式导入了AutoConfigurationImportSelector这个核心类，那么@Import是什么时候被spring容器扫描执行的呢？

```java
SpringApplication.run
SpringApplication.refreshContext
SpringApplication.refresh
AbstractApplicationContext.refresh()
AbstractApplicationContext#invokeBeanFactoryPostProcessors
PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors
BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry
ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry
ConfigurationClassPostProcessor#processConfigBeanDefinitions
// 下面这个方法是判断配置类是full模式还是lite模式的bean
// 【full模式】：就是标记@Configuration的类，里面的@Bean都是交给容器管理的，默认是单例的
// 【lite模式】：就是标记@Component,@ComponentScan,@Import,@ImportResource的类
// 与完整的@Configuration不同，lite @Bean方法不能声明bean之间的依赖关系，【并且不会被CGLIB代理，**不是单例的**】
//如果bean标记了@Configuration,@Component,@ComponentScan,@Import,@ImportResource会被加入到`List<BeanDefinitionHolder> configCandidates`集合，都当做配置类来处理
ConfigurationClassUtils#checkConfigurationClassCandidate
//回到ConfigurationClassPostProcessor类
// 【核心方法】，这个方法里面处理了ImportSelector的实现类
ConfigurationClassParser#parse

// ************************************************************************
//处理配置类，这里标记了@Import的bean会被处理
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    this.deferredImportSelectors = new LinkedList<DeferredImportSelectorHolder>();

    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        try {
            if (bd instanceof AnnotatedBeanDefinition) {
                parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
            }
            else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
            }
            else {
                parse(bd.getBeanClassName(), holder.getBeanName());
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
        }
    }

    // 最后才处理DeferredImportSelector的实现类
    //就是在这里AutoConfigurationImportSelector.selectImports被执行
    //老版本SpringBoot这个类的名字叫EnableAutoConfigurationImportSelector
    processDeferredImportSelectors();
}
// ************************************************************************

```

#### AutoConfigurationImportSelector

这个AutoConfigurationImportSelector是实现自动配置的核心类，它实现了DeferredImportSelector，会被spring容器加载到容器里面，通过spring SPI的方式把我们自定义的自动配置类加载到容器中了。

```java
//【注意，它实现了DeferredImportSelector接口】
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {
    //1. 核心方法selectImports
    @Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(
				autoConfigurationMetadata, annotationMetadata);
		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
	}

    protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata, AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return EMPTY_ENTRY;
		}
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
        //2. 这里获取配置
		List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
		configurations = removeDuplicates(configurations);
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		checkExcludedClasses(configurations, exclusions);
		configurations.removeAll(exclusions);
		configurations = filter(configurations, autoConfigurationMetadata);
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return new AutoConfigurationEntry(configurations, exclusions);
	}

    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
			AnnotationAttributes attributes) {
        //3. 这里使用spring的SPI机制，加载了EnableAutoConfiguration类对应的配置类，见下方图片
        //所以在我们的自定义项目jar包的/META-INF/spring.factories里面配置了自己的类，就会被自动加载进来
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
				getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
		Assert.notEmpty(configurations,
				"No auto configuration classes found in META-INF/spring.factories. If you "
						+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}

    protected Class<?> getSpringFactoriesLoaderFactoryClass() {
		return EnableAutoConfiguration.class;
	}
    //...
}
```

![EnableAutoConfiguration.jpg](/images/spring/EnableAutoConfiguration.jpg)

举个例子，我们使用的redis的自动配置类

```java
@Configuration
@ConditionalOnClass(RedisOperations.class)
//RedisProperties是redis的配置类
//spring.redis开头的配置都会被加载到这个类里面来
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(name = "redisTemplate")
	public RedisTemplate<Object, Object> redisTemplate(
			RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
		RedisTemplate<Object, Object> template = new RedisTemplate<>();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

	@Bean
	@ConditionalOnMissingBean
	public StringRedisTemplate stringRedisTemplate(
			RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
		StringRedisTemplate template = new StringRedisTemplate();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

}
```

## 参考

[Spring Boot @ConfigurationProperties: Binding external configurations to POJO classes](https://www.callicoder.com/spring-boot-configuration-properties-example/)
