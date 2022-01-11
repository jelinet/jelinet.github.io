---
layout: post
title:  "Spring-boot运行调试"
date:   2022-01-05 18:54 +0800
categories: spring-boot

---





## springboot核心原理

1. SpringBoot核心通过Maven继承依赖关系快速整合第三方框架

   ```xml
   <!-- 当你添加了相应的starter模块，就相当于添加了相应的所有必须的依赖包，包括spring-boot-starter（这是Spring Boot的核心启动器，包含了自动配置、日志和YAML）；spring-boot-starter-test（支持常规的测试依赖，包括JUnit、Hamcrest、Mockito以及spring-test模块）；spring-boot-starter-web （支持全栈式Web开发，包括Tomcat和spring-webmvc）等相关依赖。
   -->
   <parent>
   	<groupId>org.springframework.boot</groupId>
   	<artifactId>spring-boot-starter-parent</artifactId>
   	<version>2.0.0.RELEASE</version>	
   </parent>
   <dependencies>
   	<!-- SpringBoot 整合SpringMVC -->
   	<dependency>
   		<groupId>org.springframework.boot</groupId>
   		<artifactId>spring-boot-starter-web</artifactId>
   	</dependency>
   </dependencies>
   ```
   
2. 基于SpringMVC无配置文件（纯Java）完全注解化实现SpringBoot框架，Main函数启动。

   ```java
   @SpringBootApplication 
   public class Application {
   	//方式一
   	public static void main(String[] args) {
       SpringApplication.run(Application.class, args);
     }
     //方式二
     public static void main(String[] args) {
       SpringApplication app = new SpringApplication(MySpringConfiguration.class);
       app.run(args);
     }
	  //方式三
     public static void main(String[] args) {
       new SpringApplicationBuilder()
            .sources(Parent.class)
            .child(Application.class)
            .run(args);
     }
   }
   ```

### SpringApplication

springboot驱动spring应用上下文的引导类，run()方法启动Spring应用，实质上是为Spring应用创建并初始化Spring上下文。

> - Create an appropriate ApplicationContext instance (depending on your classpath)->创建ApplicationContext实例 (基于classpath)
> - Register a CommandLinePropertySource to expose command line arguments as Spring properties->注册一个 CommandLinePropertySource to 命令行参数暴露为Spring 属性
> - Refresh the application context, loading all singleton beans
> - Trigger(触发) any CommandLineRunner beans

SpringApplications 可以使用**@Configuration** 注解加载, 也可以通过以下加载:

- AnnotatedBeanDefinitionReader加载的类名
- XmlBeanDefinitionReader加载的 XML ，或者 GroovyBeanDefinitionReader 加载的 groovy 脚本
- ClassPathBeanDefinitionScanner 要扫描的包的名称

```java
	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```