Spring Boot Admin 用于管理和监控一个或者多个Spring Boot应用， Spring Boot Admin 分为Server端和Client端，Client通过http向Server端注册， 也可以结合Spring Cloud 的服务注册组件Eureka 进行注册，本项目使用Eureka进行注册发现。

首先看项目结构分为：
* eureka-server 发现注册服务
* spring-boot-admin-server SpringBootAdmin服务管理端
* spring-boot-admin-client SpringBootAdmin客户端
![c1.png](https://upload-images.jianshu.io/upload_images/2151905-f8dfc69b1e0ca109.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Eureka Server
 eureka-server项目的搭建过程请看 [spring cloud 搭建集群Eureka Server](https://www.jianshu.com/p/3a8d637f3e07) ，这里就简单说下

pom.xml 依赖

``` xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
```
启动类中添加注释 `@EnableEurekaServer`
``` java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```
application.yml
``` yml
spring:
  application:
    name: eureka-server
eureka:
  instance:
    hostname: localhost
    prefer-ip-address: true
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      serviceZone: http://localhost:8761/eureka/
server:
  port: 8761
```
成功启动后访问：http://localhost:8761

## spring-boot-admin-server
admin-server中的认证模块一起说看完整的pom.xml
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.spring.boot.admin.server</groupId>
    <artifactId>spring-boot-admin-server</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>spring-boot-admin-server</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.3.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-boot-admin.version>2.0.1</spring-boot-admin.version>
        <spring-cloud.version>Finchley.RELEASE</spring-cloud.version>
    </properties>

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
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-starter-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.jolokia</groupId>
            <artifactId>jolokia-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>de.codecentric</groupId>
                <artifactId>spring-boot-admin-dependencies</artifactId>
                <version>${spring-boot-admin.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
``` 
spring-boot-admin-server的启动类：
``` java
@Configuration
@EnableAutoConfiguration
@SpringBootApplication
@EnableAdminServer
@EnableEurekaClient
public class SpringBootAdminServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootAdminServerApplication.class, args);
    }
   
}
```
spring-boot-admin-server也做为Eureka Client端；

增加一个安全配置类：
``` java
@Configuration
public class SecuritySecureConfig extends WebSecurityConfigurerAdapter {

    private final String adminContextPath;

    public SecuritySecureConfig(AdminServerProperties adminServerProperties) {
        this.adminContextPath = adminServerProperties.getContextPath();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // @formatter:off
        SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter("redirectTo");

        http.authorizeRequests()
                .antMatchers(adminContextPath + "/assets/**").permitAll()
                .antMatchers(adminContextPath + "/login").permitAll()
                .anyRequest().authenticated()
                .and()
                .formLogin().loginPage(adminContextPath + "/login").successHandler(successHandler).and()
                .logout().logoutUrl(adminContextPath + "/logout").and()
                .httpBasic().and()
                .csrf().disable();
        // @formatter:on
    }

}
```
application.yml
``` yml
server:
  port: 8766
spring:
  application:
    name: admin-server
  security:
    user:
      name: "admin"
      password: "admin"


eureka:
  instance:
      leaseRenewalIntervalInSeconds: 10
      health-check-url-path: /actuator/health
      prefer-ip-address: true
      metadata-map:
            user.name: ${spring.security.user.name}
            user.password: ${spring.security.user.password}
  client:
    registryFetchIntervalSeconds: 5
    serviceUrl:
      defaultZone: ${EUREKA_SERVICE_URL:http://localhost:8761}/eureka/

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: ALWAYS
```
这其中 security.user.name和security.user.password是访问admin-server的权限

``` yml
security:
    user:
      name: "admin"
      password: "admin"
```
注意eureka的实例下要添加 eureka.instance.metadata-map.user.name和eureka.instance.metadata-map.user.name 否则在安全模式下无法注册
``` yml
eureka:
  instance:
      metadata-map:
            user.name: ${spring.security.user.name}
            user.password: ${spring.security.user.password}
```

现在启动访问 spring-boot-admin-server http://localhost:8766

![c2.png](https://upload-images.jianshu.io/upload_images/2151905-a2b96132e82b2e8c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![c3.png](https://upload-images.jianshu.io/upload_images/2151905-30382bf526fdccea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/850)
下面这张图中间部分是因为我用火狐浏览器截图整个网页的原因
![c4.png](https://upload-images.jianshu.io/upload_images/2151905-506a0fcf2f95068b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/850)

## spring-boot-admin-client
完成的pom.xml
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.spring.boot.admin.client</groupId>
    <artifactId>spring-boot-admin-client</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>spring-boot-admin-client</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.3.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Finchley.RELEASE</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>
        <dependency>
            <groupId>org.jolokia</groupId>
            <artifactId>jolokia-core</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```
启动类不用修改
``` java
@SpringBootApplication
public class SpringBootAdminClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootAdminClientApplication.class, args);
    }
}
```
application.yml
``` yml
spring:
  application:
    name: admin-client
  security:
    user:
      name: "client"
      password: "client"

management:
  endpoints:
    web:
      exposure:
        include: "*"
logging:
  file: /var/log/sample-boot-application.log
  pattern:
    file: clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(%5p) %clr(${PID}){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n%wEx
eureka:
  instance:
    leaseRenewalIntervalInSeconds: 10
    health-check-url-path: /actuator/health
    prefer-ip-address: true
    metadata-map:
      user.name:  ${spring.security.user.name}
      user.password:  ${spring.security.user.password}
  client:
    registryFetchIntervalSeconds: 5
    service-url:
      serviceZone: http://localhost:8761/eureka/
server:
  port: 8764
```
 `security.user.name、 security.user.password` 是SpringBootAdmin客户端的认证，同理 Eureka实例必须加上
``` yml
eureka:
  instance:
    metadata-map:
      user.name:  ${spring.security.user.name}
      user.password:  ${spring.security.user.password}
```
现在启动admin-client，在admin-server中发现多了一个
![c7.png](https://upload-images.jianshu.io/upload_images/2151905-d5c766a3601a4178.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/850)

如果启动多个admin-client实例，会看到在admin-server中admin-client下面有两个实例
![c8.png](https://upload-images.jianshu.io/upload_images/2151905-fbb18eb0797bde0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/850)

### 配置邮件通知
当客户端触发UP或OFFLINE或其它状态时，会发邮件通知

首先在 spring-boot-admin-server的pom.xml中添加依赖
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```
然后在application.yml中添加 配置JavaMailSender
``` yml
###################
  # 邮件通知配置
  ##################
spring:
   mail:
      host: smtphm.qiye.163.com
      username: # 用户名
      password: # 密码
   boot:
     admin:
        notify:
          mail:
          from: # 发件人
          to: # 收件人
          enabled: true
```
配置完成，依次启动eureka-server、spring-boot-admin-server、spring-boot-admin-client
当admin-client成功注册时，停掉admin-client，就会收到一个邮件通知
![d1.png](https://upload-images.jianshu.io/upload_images/2151905-5c7c4137e79865d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


