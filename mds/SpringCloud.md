## Eureka

服务注册与发现中心。

遵守AP原则, *ZK遵守的是CP原则*



### 使用

*注意：模块可能以cloud为父组件，provider为提供者，consumer为消费者,  register为服务端*

#### 1. 父组件处理步骤

1. POM

   ```xml
   <!-- 配置可访问resource文件中 $$ 中的参数可以获取 -->
   <build>
   	<finalName>microservicecloud</finalName>
     <resources>
     	<resource>
       	<directory>src/main/resources</directory>
         <filtering>true</filtering>
       </resource>
     </resources>
     <plugins>
     	<plugin>
       	<groupId>org.apache.maven.plugins</groupId>
         <artifactId>maven-resources-plugin</artifactId>
         <configuration>
         	<delimiters>
           	<delimiter>$</delimiter>
           </delimiters>
         </configuration>
       </plugin>
     </plugins>
   </build>
   ```

   

#### 2. 服务端处理：(registry)

1. POM

   ```xml
   <!-- 服务端Eureka依赖 -->
   <dependency>
    <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-eureka-server</artifactId>
   </dependency>
   ```

   

2. YAML

  ```yaml
  server:
    port: xxxx
  eureka:
    client:
      service-url:
        defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka,http://eureka7002.com:7002/eureka # 设置与eureka server交互的地址查询服务和注册服务都需要依赖这个地址, 注意逗号左右不能有空格
      register-with-eureka: false # false表示不注册自身
      fetch-registry: false # false表示自身是注册中心，职责就是维护服务实例，不需要检索服务
    instance:
      lease-renewal-interval-in-seconds: 5
      lease-expiration-duration-in-seconds: 10
      hostname: eureka7001.com # eureka服务端的实例名称
    server:
      enable-self-preservation: false # 测试时关闭自我保护机制，保证不可用服务及时踢出
  ```

  

3. ServerAPP主启动类添加注解@EnableEurekaServer



#### 3. Client端处理步骤：(provider)

1. POM

   ```xml
   <!-- 客户端Eureka依赖 -->
   <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
   </dependency>
   
   <!-- 添加监控依赖，微服务/info信息一同提供 -->
   <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
   </dependency>
   ```

   

2. YAML

   ```yaml
   eureka:
     client: 
       service-url:
         defaultZone: @eureka.client.service-url.defaultZone@ # http://xx:xx/eureka 注册到服务列表
   	instance:
   		instance-id: ${spring.cloud.client.ip-address}:${server.port} # 实例名称显示方式
       prefer-ip-address: true # 访问路径显示IP地址
   
   info: # 配置微服务'/info'超链接信息，伴随actuator服务
   	app.name: xxx-microservicecloud
   	company.name: zozo.cn
   	build.artifactId: $project.artifactId$ # 使用父组件的build插件，可以获取$$间的参数
   	build.version: $project.version$
   ```

   

3. 主启动类添加注解@EnableEurekaClient



#### 4. 服务发现 (provider)

1. 主启动类添加@EnableDiscoveryClient

2. 代码中获取服务信息

  ```java
  // Controller层
  @Autowired
  private DiscoveryClient client;
  
  @RequestMapping(value="/discovery", method=RequestMethod.GET)
  public Object discovery() {
    List<String> list = client.getServices();
    System.out.println("Service list:" + list);
  
  }
  ```



## Ribbon

客户端负载均衡工具。

### 使用

1.1 POM

- consumer端

> spring-cloud-starter-eureka
>
> spring-cloud-starter-ribbon
>
> spring-cloud-starter-config
>

1.2 YAML

- consumer端

  ```yaml
  配置Eureka发现服务
  ```

1.3 对需要访问其他服务的对象添加@LoadBalance

> 例如消费者通过RestTemplate访问提供者时，可以为new RestTemplate添加
>
> 此时，使用restTemplate填充的URL可以使用服务名称作为URL代替IP访问

1.4 自定义Ribbon配置类，使用自定义配置

Consumer主启动类添加 @RibbonClient(name="Provider服务名称", configuration=MySelfRule.class)

***注意***：自定义Ribbon配置类MySelfRule.class不可处于 `@ComponentScan`扫描的包路径下

```java
@Configuration
public class MySelfRule{
  @Bean
  public IRule myRule() {
    return new RandomRule(); # 使用随机轮训
  }
}
```



## Feign

*声明式的WebService客户端* 只需要创建一个接口，并在上面添加注解即可

