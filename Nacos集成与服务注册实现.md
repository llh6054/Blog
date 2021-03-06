### SpringCloud整合Nacos以及服务注册分析

+ [安装Nacos](https://nacos.io/zh-cn/docs/quick-start.html)

+ 完整依赖

  ```xml
  <dependencyManagement>
      <dependencies>
          <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-dependencies</artifactId>
              <version>Greenwich.SR5</version>
              <type>pom</type>
              <scope>import</scope>
          </dependency>
          <dependency>
              <groupId>com.alibaba.cloud</groupId>
              <artifactId>spring-cloud-alibaba-dependencies</artifactId>
              <version>2.1.0.RELEASE</version>
              <type>pom</type>
              <scope>import</scope>
          </dependency>
      </dependencies>
  </dependencyManagement>
  
  <dependencies>
      <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-web</artifactId>
      </dependency>
  
      <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-actuator</artifactId>
      </dependency>
  
      <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-test</artifactId>
          <scope>test</scope>
      </dependency>
  
      <dependency>
          <groupId>com.alibaba.cloud</groupId>
          <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
      </dependency>
  
  </dependencies>
  ```
  
+ 测试例子

  ```java
  @SpringBootApplication(scanBasePackages={"com.netflix.client.config"})
  @EnableDiscoveryClient
  public class NacosProviderDemoApplication {
  
      public static void main(String[] args) {
          SpringApplication.run(NacosProviderDemoApplication.class, args);
      }
  
      @RestController
      public class EchoController {
          @GetMapping(value = "/echo/{string}")
          public String echo(@PathVariable String string) {
              return "Hello Nacos Discovery " + string;
          }
      }
  }
  ```

+ http://127.0.0.1:8848/nacos/index.html查看服务

+ 服务注册实现相关

  + 从依赖关系出发

    ![image-20200325120155012](https://raw.githubusercontent.com/llh6054/imagecloud/master/img/20200325145420.png)

    spring-cloud-ablibaba-nacos-discovery是为了适配Spring-Cloud衍生出来的包，里面负责对Spring-Cloud进行适配，nacos-client为Nacos对外暴露的统一接口，nacos-common为一些通用的工具类，真的各种实现在nacos-api中，实际这几个包中除去封装的HttpClient、一些工厂类、Util和一些缓存、日志、安全等类并没有多少东西

  + [SpringBoot Starter启动逻辑]([https://tanzu.vmware.com/content/springone-platform-2017/its-a-kind-of-magic-under-the-covers-of-spring-boot-brian-clozel-st%C3%A9phane-nicoll](https://tanzu.vmware.com/content/springone-platform-2017/its-a-kind-of-magic-under-the-covers-of-spring-boot-brian-clozel-stéphane-nicoll))

    配置类：NacosDiscoveryAutoConfiguration

  + 几个重要的适配类

    NacosDiscoveryAutoConfiguration会配置需要使用得三个重要的适配类

    1. NacosServiceRegistry  负责Nacos的服务注册
    2. NacosRegistration  负责Nacos服务注册的配置，本质就是一个POJO
    3. NacosAutoServiceRegistration  真正的SpringCloud适配类实现，在这里调起NacosServiceRegistry  进行register

+ 具体分析

  入口：AbstractAutoServiceRegistration实现了EventListener接口，SpringBoot的启动主流程中

  org.springframework.boot.SpringApplication#run(java.lang.String...)发布一个事件

  ```java
  ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
  			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
  			configureIgnoreBeanInfo(environment);
  			Banner printedBanner = printBanner(environment);
  			context = createApplicationContext();
  			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
  					new Class[] { ConfigurableApplicationContext.class }, context);
  			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
  			// 在这里发布一个ApplicationEvent事件
  			refreshContext(context);
  			afterRefresh(context, applicationArguments);
  			stopWatch.stop();
  			if (this.logStartupInfo) {
  				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
  			}
  			listeners.started(context);
  			callRunners(context, applicationArguments);
  ```

  AbstractAutoServiceRegistration接收到后执行onApplicationEvent->bindEvent->start->register(由子类实现即NacosAutoServiceRegistration),然后调用AbstractAutoServiceRegistration.register

  ```java
  protected void register() {
  		this.serviceRegistry.register(getRegistration());
  	}
  ```

  serviceRegistry为NacosServiceRegistry实例，最后调用NacosServiceRegistry#register(Registration registration)进行服务的注册

  ```java
  String serviceId = registration.getServiceId();
  
  Instance instance = getNacosInstanceFromRegistration(registration);
  
  try {
      // 真正注册的地方
      namingService.registerInstance(serviceId, instance);
      log.info("nacos registry, {} {}:{} register finished", serviceId,
               instance.getIp(), instance.getPort());
  }
  catch (Exception e) {
      log.error("nacos registry, {} register failed...{},", serviceId,
                registration.toString(), e);
  }
  ```

  最后转到NamingProxy进行注册，本质为一个Http请求，至此一个节点就注册到了Nacos的服务列表中。

+ NamingProxy中的其他方法，Nacos相关功能的主要方法都定义在NamingProxy类

  +  创建NamingProxy时执行initRefreshSrvIfNeed，initRefreshSrvIfNeed中有一个定时任务负责服务的拉取

  ```java
  private void initRefreshSrvIfNeed() {
          if (StringUtils.isEmpty(endpoint)) {
              return;
          }
  
          ScheduledExecutorService executorService = new ScheduledThreadPoolExecutor(1, new ThreadFactory() {
              @Override
              public Thread newThread(Runnable r) {
                  Thread t = new Thread(r);
                  t.setName("com.alibaba.nacos.client.naming.serverlist.updater");
                  t.setDaemon(true);
                  return t;
              }
          });
  		// 定时更新 时间为30秒
          executorService.scheduleWithFixedDelay(new Runnable() {
              @Override
              public void run() {
                  refreshSrvIfNeed();
              }
          }, 0, vipSrvRefInterMillis, TimeUnit.MILLISECONDS);
  
          refreshSrvIfNeed();
      }
  ```

  + registerService 服务注册 deregisterService 服务下线还有其他的实例更新、健康状态拉取、发送心跳、获取服务列表、签名等等。

+ 写在最后

  定义一个模板方法，留下几个抽象方法让子类实现将具体的操作延时到子类是Spring的常用手段，包括Spring-Cloud-Stream对消息的处理，服务注册等都是采用这样一种模式，重要的在于理解SpringBoot的启动逻辑以及如何去调起一个外部应用。



