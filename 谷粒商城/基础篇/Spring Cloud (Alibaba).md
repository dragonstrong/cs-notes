# 1.组件概述
Spring Cloud Alibaba： [Github中文文档](https://github.com/alibaba/spring-cloud-alibaba/blob/2022.x/README-zh.md)

| 功能 | Spring Cloud([链接](https://spring.io/projects/spring-cloud)) | Spring Cloud Alibaba([链接](https://sca.aliyun.com/zh-cn/)) |
| --- | --- | --- |
| 注册中心 | Eureka ([链接](https://spring.io/projects/spring-cloud-config)) | Nacos |
| 配置中心 | Spring Cloud Config([链接](https://spring.io/projects/spring-cloud-netflix)) | Nacos |
| 网关 | Spring Cloud Gateway([链接](https://spring.io/projects/spring-cloud-config)) |  |
| 负载均衡 | Ribbon([链接](https://cloud.spring.io/spring-cloud-netflix/multi/multi_spring-cloud-ribbon.html)) |  |
| 声明式HTTP客户端 | Open Feign([链接](https://spring.io/projects/spring-cloud-openfeign)) |  |
| 断路保护，熔断降级 | Hystrix(Netflix里) | Sentinel ([链接](https://sca.aliyun.com/zh-cn/docs/2022.0.0.0/user-guide/sentinel/overview)) |
| 调用链监控 | Sleuth([链接](https://spring.io/projects/spring-cloud-sleuth)) |  |
| 分布式事务 |  | Seata([链接](https://sca.aliyun.com/zh-cn/docs/2022.0.0.0/user-guide/seata/overview)) |

组件功能

- 注册中心：微服务上线后都应该将自己注册到注册中心，若B服务要调用A服务，B服务首先从注册中心获取A服务的所有实例，然后随机挑选一个调用即可。
- 配置中心：假设某服务分布在多台机器上，想批量修改其配置，逐一修改较为麻烦，若服务能从配置中心动态获取配置，则较为简单，这就是配置中心产生的缘由。

![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710817790797-186acdc5-a601-429b-9032-3670ac7e30f8.png#averageHue=%23c5c1bf&clientId=u24f04165-24b9-4&from=paste&height=610&id=u3b5c20bf&originHeight=610&originWidth=1155&originalType=binary&ratio=1&rotation=0&showTitle=false&size=80109&status=done&style=none&taskId=ue5c040b0-96a8-4e01-83e8-6813a2b29e4&title=&width=1155)

# 2.使用
配置参考：尽量参考官网，不同版本配置有些不同
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710822330987-dd647961-ec54-4b2e-9a44-f3161a0e638e.png#averageHue=%23fefefe&clientId=u68a36cbf-7154-4&from=paste&height=494&id=ud54b3036&originHeight=494&originWidth=1137&originalType=binary&ratio=1&rotation=0&showTitle=false&size=37737&status=done&style=none&taskId=uae9b3c8b-1bd6-4d89-bab5-090f0a56b49&title=&width=1137)

## 2.1 导入spring-cloud alibaba依赖
spring boot、spring cloud和sring cloud alibaba版本必须对应：
[Springboot Spring Cloud Alibaba版本对应表](https://sca.aliyun.com/zh-cn/docs/2022.0.0.0/best-practice/spring-boot-to-spring-cloud#spring-boot-%E4%B8%8E-spring-cloud-alibaba-%E7%89%88%E6%9C%AC%E5%AF%B9%E5%BA%94%E5%85%B3%E7%B3%BB)
```xml
 <!--spring cloud alibaba-->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>2021.0.5.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```
## 2.2 注册中心Nacos Discovery
配置参考：[链接](https://github.com/alibaba/spring-cloud-alibaba/blob/2022.x/spring-cloud-alibaba-examples/nacos-example/readme.md#spring-cloud-alibaba-nacos-discovery)
（1） pom.xml导入依赖 Nacos Discovery Starter
作用：将微服务注册到注册中心，并发现微服务
```xml
<!--nacos注册中心-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```
（2）启动Nacos Server服务器
下载：[链接](https://github.com/alibaba/nacos/releases)
运行前需先设置JAVA_HOME和PATH环境变量，否则闪退。
启动：nacos/bin ，windows下cmd：
```bash
startup.cmd -m standalone  # 非集群模式，start startup.cmd会报tomcat错误
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710829746061-2313e55c-37e8-452f-892f-b7d8fbf2e558.png#averageHue=%23131212&clientId=ua312ed4c-008e-4&from=paste&height=846&id=u8c6422b1&originHeight=846&originWidth=1685&originalType=binary&ratio=1&rotation=0&showTitle=false&size=112195&status=done&style=none&taskId=u423925ec-facd-4437-aed4-9410269d824&title=&width=1685)
报错：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1713201622083-cec5ab2d-68bd-468b-9ce5-219b5373f5f3.png#averageHue=%231d1414&clientId=u2c11b25b-7cc6-4&from=paste&height=168&id=u764ab4ea&originHeight=168&originWidth=1053&originalType=binary&ratio=1&rotation=0&showTitle=false&size=18678&status=done&style=none&taskId=u9736dedd-a4ed-4f51-bc92-052fdff4b3e&title=&width=1053)
解决：重装jdk，64位的系统尽量装64位的jdk
启动nacos后[http://localhost:8848/nacos](http://localhost:8848/nacos)无法访问：检查下面两个文件夹是否只读，取消权限限制后重新启动nacos即可访问
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1713202457520-172f1ded-52c5-4dac-8429-9aff1e6b1838.png#averageHue=%23fcf8f7&clientId=uae6c9550-501e-4&from=paste&height=248&id=u24b44d18&originHeight=248&originWidth=687&originalType=binary&ratio=1&rotation=0&showTitle=false&size=18963&status=done&style=none&taskId=uad1ee8f0-bec3-4bb4-9939-b108d38e82d&title=&width=687)
（3）application.yml中配置Nacos Server地址及服务名
```yaml
spring:
  # 注册中心地址（nacos服务器）
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
  #服务名
  application:
    name: gulimall-ware
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710835904154-18f94f01-46c4-4410-b192-09e479ecce3c.png#averageHue=%23525d44&clientId=u73ace7c7-14b4-4&from=paste&height=409&id=uba9bfca7&originHeight=409&originWidth=845&originalType=binary&ratio=1&rotation=0&showTitle=false&size=70097&status=done&style=none&taskId=u7dabe966-84d1-4326-b851-75e3f9e5050&title=&width=845)

（4）注解开启服务注册发现
```java
@EnableDiscoveryClient
```

![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710835471760-ac511e7a-5b70-4b19-819b-b91b3c10171e.png#averageHue=%23716941&clientId=ubc073464-a470-4&from=paste&height=354&id=u659312c4&originHeight=354&originWidth=1144&originalType=binary&ratio=1&rotation=0&showTitle=false&size=64037&status=done&style=none&taskId=uad2b412d-2761-4fb9-a2d6-049efd8b800&title=&width=1144)
(5) 测试
启动gulimall-ware服务和nacos服务器，浏览器访问[http://localhost:8848/nacos](http://localhost:8848/nacos)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710835998381-9ab5a961-d78d-4c35-b88c-bcd2bc32339b.png#averageHue=%23312e2c&clientId=u73ace7c7-14b4-4&from=paste&height=207&id=u81938ef8&originHeight=207&originWidth=1814&originalType=binary&ratio=1&rotation=0&showTitle=false&size=90281&status=done&style=none&taskId=u00dfc143-5dc1-4ef5-be89-ca496733c1f&title=&width=1814)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710836109311-add924df-a616-465d-9edf-387b1b20c416.png#averageHue=%23e1c496&clientId=u73ace7c7-14b4-4&from=paste&height=476&id=ud5a4dc0d&originHeight=476&originWidth=1205&originalType=binary&ratio=1&rotation=0&showTitle=false&size=58263&status=done&style=none&taskId=ufeb1a10a-119e-4edc-b1c0-88f984cb915&title=&width=1205)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710836134254-ad544ddb-14b8-4d27-ba2b-a6c2b2bd96bf.png#averageHue=%23c6c6c5&clientId=u73ace7c7-14b4-4&from=paste&height=753&id=u7c6cb30b&originHeight=753&originWidth=1163&originalType=binary&ratio=1&rotation=0&showTitle=false&size=47042&status=done&style=none&taskId=u103c6baf-7354-4857-8a39-e3d9f022588&title=&width=1163)
## 2.3 Open Feign服务间调用
### 2.3.1 简介
 Feign 是一个声明式的 HTTP 客户端，让远程调用更加简单。它提供了 HTTP 请求的模板，通过编写简单的接口和插入注解，就可以定义好 HTTP 请求的参数、格式、地 址等信息。 Feign 整合了 Ribbon（负载均衡）和 Hystrix(服务熔断)，可以让我们不再需要显式地使用这 两个组件。 SpringCloudFeign 在 NetflixFeign 的基础上扩展了对 SpringMVC 注解的支持，在其实现下，我 们只需创建一个接口并用注解的方式来配置它，即可完成对服务提供方的接口绑定。简化了 SpringCloudRibbon 自行封装服务调用客户端的开发量  
### 2.3.2使用
前提：调用方和被调用方都能注册到注册中心

1. 引入依赖

调用方pom.xml中引入openfeign和loadbalancer依赖（针对新版openfeign，旧版内置了robbin，无需引入loadbalancer，新版弃用后不引会报错）
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>

```

示例：gulimall-member调用gulimall-coupon服务，查出某会员的优惠券信息
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710839188586-1ef74927-5237-43a1-8b34-3088bd2184e8.png#averageHue=%23fafaf9&clientId=ub242efab-197c-4&from=paste&height=485&id=u1b116060&originHeight=485&originWidth=1212&originalType=binary&ratio=1&rotation=0&showTitle=false&size=57973&status=done&style=none&taskId=u12257eea-00a0-4d22-9bc3-501bb92dd45&title=&width=1212)

2. 被调用服务gulimall-coupon中定义被调用的接口

返回优惠券信息
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710839480336-967dc577-557f-404a-883c-57465d5372d5.png#averageHue=%23647449&clientId=ub242efab-197c-4&from=paste&height=350&id=uc209c6c3&originHeight=350&originWidth=1022&originalType=binary&ratio=1&rotation=0&showTitle=false&size=68201&status=done&style=none&taskId=ub1232abd-d348-4958-a7f6-c1deddc6186&title=&width=1022)

3. 调用方gulimall-member中编写一个OpenFeign接口

放在feign包下, 接口上加上@FeignClient注解，里面配置被远程调用的服务名
```java
// 声明要调用远程服务名
@FeignClient("gulimall-coupon")
public interface CouponFeignClient {
    // 声明要调用的远程接口url
    @GetMapping("/coupon/coupon/member/list")
    public R memberCoupons();
}

```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710839850819-866484c6-f91c-4dde-b290-c49612d7a786.png#averageHue=%23566c43&clientId=ub242efab-197c-4&from=paste&height=516&id=uaf835c50&originHeight=516&originWidth=955&originalType=binary&ratio=1&rotation=0&showTitle=false&size=114380&status=done&style=none&taskId=uafb8ef2d-244d-4165-bbd5-d077147c873&title=&width=955)

4. 声明远程调用哪个接口

配置调用接口的url，方法有参数照着copy，例如：
```java
// 调用gulimall
@GetMapping("/coupon/coupon/member/list")
public R memberCoupons();
```

5. 调用方主程序加上@EnableFeignClients注解

basePackages配置openfeign接口的包路径
```java
// basePackages设置openfeign接口包路径
@EnableFeignClients(basePackages = "com.atguigu.member.feign")
public class GulimallMemberApplication {
    public static void main(String[] args) {
        SpringApplication.run(GulimallMemberApplication.class, args);
    }
}

```

6. 测试

开启nacos 注册中心服务器、调用服务和被调服务
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710840346407-037fdc20-70f1-47e7-8078-4b7cd30773bd.png#averageHue=%23f0f0ef&clientId=u905252dd-420b-4&from=paste&height=163&id=u23816b32&originHeight=163&originWidth=1275&originalType=binary&ratio=1&rotation=0&showTitle=false&size=26059&status=done&style=none&taskId=u8fb06349-1ecd-49bf-820a-2f8ba4962b4&title=&width=1275)

### 2.3.3 Feign源码分析

![image-20240902225455358](assets/image-20240902225455358.png)



![image-20240902225629263](assets/image-20240902225629263.png)

调用流程

```java
/**
     * Feign的调用流程
     * 1. 构造请求数据，将对象转为json
     * RequestTemplate template = this.buildTemplateFromArgs.create(argv);
     * 2. 发送请求执行(执行成功会解码)
     * executeAndDecode(template, options);
     * 3. 执行请求时有重试机制（默认最大重试次数为5）
     * while(true) {
     *             try {
     *                 return this.executeAndDecode(template, options);
     *             } catch (RetryableException var9) {
     *                     retryer.continueOrPropagate(e);
     *                     throw ex; // 重试器有异常会抛出去
     *                     continue;  // 没异常继续重试
     *             }
     *         }
     **/
```

## 2.4 配置中心Nacos Config

### 2.4.1使用
配置参考：[链接](https://github.com/alibaba/spring-cloud-alibaba/blob/2022.x/spring-cloud-alibaba-examples/nacos-example/readme.md#spring-cloud-alibaba-nacos-config)

1. 引入Nacos Config依赖

高版本还得导bootstrap依赖
```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```

2. resources路径下创建bootstrap.properties文件

启动时它会优先application.yml加载
name: 一般取模块名
server-addr: 填nacos配置中心地址
```yaml
spring.application.name=gulimall-member
spring.cloud.nacos.config.server-addr=127.0.0.1:8848
# 若配置文件非properties则需要显示注明文件后缀,例如yml：
spring.cloud.nacos.config.file-extension=yaml
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710849275006-de689b30-6076-4fe8-bc01-4306772e56d9.png#averageHue=%235f724c&clientId=u905252dd-420b-4&from=paste&height=486&id=ue13b2aa8&originHeight=486&originWidth=1135&originalType=binary&ratio=1&rotation=0&showTitle=false&size=55278&status=done&style=none&taskId=u7eba6312-968a-414b-aa86-9cb12dae693&title=&width=1135)
3.新建application.properties配置变量
```yaml
member.user.name=zhangsan
member.user.age=30
```
4.定义获取配置的接口并设置实时刷新
使用**@RefreshScope**注解设置实时刷新
```java
@RefreshScope
@RestController
@RequestMapping("member/member")
public class MemberController {

    @Value("${member.user.name}")
    private String name;
    @Value("${member.user.age}")
    private String age;
    @RequestMapping("/properties")
    public R getProperties(){
        return R.ok().put("name",name).put("age",age);
    }
}
```
5.nacos配置管理中新增远程配置文件
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710850290078-4c227cd7-8fb3-4a1e-8e0c-2976bef7d846.png#averageHue=%23fafafa&clientId=u905252dd-420b-4&from=paste&height=391&id=ua393a053&originHeight=391&originWidth=1159&originalType=binary&ratio=1&rotation=0&showTitle=false&size=36332&status=done&style=none&taskId=u7c3459ea-b3a5-45f5-b2bb-d0674a8893f&title=&width=1159)

![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710850349621-6fc5ee75-3cb3-4ae3-9709-fc973a98dc0f.png#averageHue=%23a9a8a8&clientId=u905252dd-420b-4&from=paste&height=963&id=u9fa3e75c&originHeight=963&originWidth=1100&originalType=binary&ratio=1&rotation=0&showTitle=false&size=53219&status=done&style=none&taskId=uea352f77-302d-4508-9c85-73ca7c0502d&title=&width=1100)
dataID: 和bootstrap.properties中spring.application.name保持一致
GROUP: 默认是DEFAULT_GROUP (服务启动时可以看到所属GROUP)
配置格式：和本地配置文件属保持一致，本实验为.Properties
配置内容：将application.properties中的copy过去即可（后面修改动态刷新）
6.启动服务
可以看出加载的远程配置文件和GROUP
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710850049701-faa4a05d-c8b5-4979-af89-e0ea41b144d4.png#averageHue=%232f2d2c&clientId=u905252dd-420b-4&from=paste&height=476&id=ucb6e56f8&originHeight=476&originWidth=2172&originalType=binary&ratio=1&rotation=0&showTitle=false&size=193427&status=done&style=none&taskId=u5e864616-f24f-41bb-8891-f0daaef0217&title=&width=2172)
7.测试
获取配置：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710850638828-add60135-0b88-4356-aa3a-e80886cf3ec3.png#averageHue=%23d6b68a&clientId=u905252dd-420b-4&from=paste&height=105&id=u5f021223&originHeight=105&originWidth=537&originalType=binary&ratio=1&rotation=0&showTitle=false&size=6577&status=done&style=none&taskId=u9231e149-f63c-4de6-8e76-2dbdf9a5279&title=&width=537)
修改远程配置文件：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710850686229-d25236c4-919c-4eb5-8212-c68197bf6098.png#averageHue=%239f9f9f&clientId=u905252dd-420b-4&from=paste&height=950&id=u4c4d9866&originHeight=950&originWidth=938&originalType=binary&ratio=1&rotation=0&showTitle=false&size=42410&status=done&style=none&taskId=u0958040f-d7b1-45b2-ae9b-5c55b3431b4&title=&width=938)
再次获取配置：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710850725313-141c9be6-a310-44db-90bb-81dc1953dc89.png#averageHue=%23f9f9f8&clientId=u905252dd-420b-4&from=paste&height=97&id=u09e68159&originHeight=97&originWidth=540&originalType=binary&ratio=1&rotation=0&showTitle=false&size=6282&status=done&style=none&taskId=u1f793d5a-5ad7-4c2c-9f87-3f189e5b42b&title=&width=540)
### 2.4.2 配置中心细节
#### 2.4.2.1规则
（1）配置中心和本地都配置了相同的项，优先使用配置中心的
（2）配置中心没有配置，则使用本地配置文件
（3）配置集：所有配置的集合，一个配置文件就是一个配置集
（4）配置集ID：类似配置文件名，在nacos中为Data ID
（5）**任何配置都能放在在配置中心**
#### 2.4.2.2 命名空间
 **做环境隔离**，默认public。

- 应用场景1： 开发、测试、生产可以创建不同的命名空间（dev、test、prod）, 在bootstrap.properties中需指定命名空间（**写命名空间的唯一id而非名字**,下面为dev命名空间的配置示例）
```yaml
spring.cloud.nacos.config.namespace=ce09900a-9ad1-4ced-b99e-f2a0e3e28f62
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710852408963-33e2c68a-78fe-4af2-b551-4eee91e4706c.png#averageHue=%23fbf9f8&clientId=uff7a1554-5033-4&from=paste&height=428&id=ufed7b9b2&originHeight=428&originWidth=1040&originalType=binary&ratio=1&rotation=0&showTitle=false&size=48482&status=done&style=none&taskId=u627fa29c-4a56-4e97-bdd3-8cd8f0d5fe1&title=&width=1040)

- 应用场景2：

为每个微服务创建唯一命名空间以示区分。
#### 2.4.2.3 配置分组
默认所有配置集都属于DEFAULT_GROUP，bootstrap.properties中可以用group设置：
```yaml
spring.cloud.nacos.config.group=111
```
建议：**每个微服务创建自己的命名空间，并使用配置分组区分环境（dev、test、prod）**
示例：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710853061587-24b09f0a-6f8f-4034-97ad-faeb44d05050.png#averageHue=%23faf9f8&clientId=uff7a1554-5033-4&from=paste&height=309&id=u874fa64f&originHeight=309&originWidth=1222&originalType=binary&ratio=1&rotation=0&showTitle=false&size=34625&status=done&style=none&taskId=u711cbc45-c506-4038-bedd-02c2e2c1afc&title=&width=1222)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710853087107-d03d3074-4cf7-4d41-8b8d-b6d7d02a8d2b.png#averageHue=%23fbf9f8&clientId=uff7a1554-5033-4&from=paste&height=448&id=u8d6039af&originHeight=448&originWidth=1075&originalType=binary&ratio=1&rotation=0&showTitle=false&size=50139&status=done&style=none&taskId=ue77f151c-a890-4567-91e9-841cce7a9f9&title=&width=1075)

![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710853033099-c661561a-6d99-4e25-a82e-147cde8e08f1.png#averageHue=%235c764a&clientId=uff7a1554-5033-4&from=paste&height=410&id=u1af61218&originHeight=410&originWidth=1053&originalType=binary&ratio=1&rotation=0&showTitle=false&size=51319&status=done&style=none&taskId=u90c0fb0a-07f7-43f1-96bb-288c31cd88b&title=&width=1053)
#### 2.4.2.4 单个服务拆分多个配置集
配置文件根据功能分类：比如数据源相关配置 datasource.yml、框架相关配置mybatis.yml、其他配置other.yml
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710855198523-a53ada2f-6016-454d-b9b9-64aafbbbcf66.png#averageHue=%23596f48&clientId=u91547651-2402-4&from=paste&height=467&id=u141a45d4&originHeight=467&originWidth=1062&originalType=binary&ratio=1&rotation=0&showTitle=false&size=96803&status=done&style=none&taskId=ub94bacdf-4bf3-4e93-b0c3-d1fc57e754f&title=&width=1062)
## 2.5 Spring Cloud Gateway
### 2.5.1 网关概述
网关作为流量的入口，常用功能包括路由转发、权限校验、限流控制等。 springcloud gateway 作为 SpringCloud 官方推出的第二代网关框架，取代了 Zuul 网关  。官方文档：[链接](https://spring.io/projects/spring-cloud-gateway)   [链接](https://docs.spring.io/spring-cloud-gateway/reference/index.html)

- 场景1：一个服务分布在多台机器上，某一个实例下线后能实时感知并将请求路由到其他正常实例上。
- 鉴权、监控等统一做到网关处

![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710855475284-1a50e562-7a81-4845-b279-75d695f887fd.png#averageHue=%23c4c1bf&clientId=uedab57ac-db0b-4&from=paste&height=610&id=ub57e75e0&originHeight=610&originWidth=1146&originalType=binary&ratio=1&rotation=0&showTitle=false&size=79984&status=done&style=none&taskId=u9bb2d0c9-0b97-4155-8ea9-b23de72a76e&title=&width=1146)



### 2.5.2 使用
#### 2.5.2.1 断言(predicate)
满足条件才路由到指定url。断言规则及使用示例：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710856355486-0427ff42-e9d3-47bc-864a-7c0aeb006c44.png#averageHue=%23fdfbf9&clientId=uedab57ac-db0b-4&from=paste&height=783&id=XiPi9&originHeight=783&originWidth=1268&originalType=binary&ratio=1&rotation=0&showTitle=false&size=131788&status=done&style=none&taskId=ue45ac79a-ab8b-4094-bb36-b2c369b921e&title=&width=1268)
示例：以下规则表示当请求时刻在某一阈值之后才路由到指定uri
```yaml
spring:
  cloud:
    gateway:
      routes: #路由规则：数组，用-分开，可以配多条规则
      - id: after_route  #唯一标识
        uri: https://example.org   #路由到哪个地方
        predicates: #断言，数组，只有当所有条件为真才路由到指定uri
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
```

#### 2.5.2.2 过滤（Filter）
官方文档示例写得很清楚
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710857034949-8b263175-07e8-42e3-ad36-3d9ded281822.png#averageHue=%23d8dca1&clientId=uedab57ac-db0b-4&from=paste&height=1069&id=u2384bf9b&originHeight=1069&originWidth=1443&originalType=binary&ratio=1&rotation=0&showTitle=false&size=143891&status=done&style=none&taskId=u769e1885-9e1a-4dbd-836e-4d83fe61008&title=&width=1443)
示例：为所有请求添加请求头，并路由到指定uri
```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: https://example.org
        filters:
        - AddRequestHeader=X-Request-red, blue  # 添加请求头
```
#### 2.5.2.3 网关测试1-Query Route Predicate Factory
依赖说明：
```java

<!--nacos注册中心-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<!--nacos配置中心-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
<!--高版本使用配置中心需额外引入-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
<!--网关一般都要引：负载均衡(路由到正确的模块或实例)-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>

<!--版本控制-->
<!--spring cloud alibaba-->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2021.0.5.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

新建网关模块
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710857262978-f63b91b5-5897-41ab-bc92-66cfb1da9d4d.png#averageHue=%233d4144&clientId=uedab57ac-db0b-4&from=paste&height=714&id=u34b9a2de&originHeight=714&originWidth=757&originalType=binary&ratio=1&rotation=0&showTitle=false&size=48439&status=done&style=none&taskId=ub6107749-fd30-4041-8a4d-5c4c7629bcc&title=&width=757)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710857331519-4a10257d-03cf-40ce-9e5a-1cd363aab041.png#averageHue=%233c3f42&clientId=uedab57ac-db0b-4&from=paste&height=738&id=uc78d0d92&originHeight=738&originWidth=777&originalType=binary&ratio=1&rotation=0&showTitle=false&size=46210&status=done&style=none&taskId=u4704bd9a-a0d8-4937-999a-73d58dbb5b2&title=&width=777)
网关也需要注册到注册中心且通过注册中心获取其他服务的健康实例并路由到正确的uri，引入gulimall-common模块：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710857588131-a5e4f1f8-9356-4779-b4b3-9b5139a284b4.png#averageHue=%235d5b41&clientId=uedab57ac-db0b-4&from=paste&height=468&id=uc60e973d&originHeight=468&originWidth=1135&originalType=binary&ratio=1&rotation=0&showTitle=false&size=98341&status=done&style=none&taskId=uf8ef700b-b0ee-48f5-aad1-14eeb6df331&title=&width=1135)
开启网关服务注册发现：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710857909635-b67635d8-68bf-471e-8a79-71795d74e97c.png#averageHue=%23565b42&clientId=uedab57ac-db0b-4&from=paste&height=325&id=u6d9080d9&originHeight=325&originWidth=1125&originalType=binary&ratio=1&rotation=0&showTitle=false&size=48472&status=done&style=none&taskId=u139953cf-2d04-4d94-8f95-818fddcba7b&title=&width=1125)
配置：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710859351508-25a2c9e1-ce7a-4d9a-9ce0-185e19aa0484.png#averageHue=%235b8052&clientId=uedab57ac-db0b-4&from=paste&height=347&id=u9a40725f&originHeight=347&originWidth=917&originalType=binary&ratio=1&rotation=0&showTitle=false&size=44989&status=done&style=none&taskId=u50cbea26-2b81-44c7-b97f-c8dad1aa8b4&title=&width=917)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710858844368-d9877661-ae39-4c5e-b049-896f714bb750.png#averageHue=%23565f46&clientId=uedab57ac-db0b-4&from=paste&height=550&id=u700e7f7c&originHeight=550&originWidth=1046&originalType=binary&ratio=1&rotation=0&showTitle=false&size=62935&status=done&style=none&taskId=u7b0bf4d7-5ae2-47f7-a0dd-7cdfc71f3e7&title=&width=1046)
**排除数据源有关的配置**（没用数据库，但gulimall-common中有）：
```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710859024798-243bfb72-4027-4572-af6e-3477f27446fa.png#averageHue=%23636246&clientId=uedab57ac-db0b-4&from=paste&height=301&id=ued1e802c&originHeight=301&originWidth=1204&originalType=binary&ratio=1&rotation=0&showTitle=false&size=50091&status=done&style=none&taskId=u0add13a8-9b7d-42d6-962d-009ca7a1dd2&title=&width=1204)

示例：url为baidu转到www.baodu.com，url为qq转到qq.com(对应断言规则The Query Route Predicate Factory)
```java
http://localhost:8888/?url=baidu  #
```
配置：
```java
spring:
  cloud:
    gateway:
      routes:
        - id: test-route1
          uri: https://www.baidu.com/
          predicates:
            - Query=url,baidu
        - id: test-route2
          uri: https://www.qq.com/
          predicates:
            - Query=url,qq
```

![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710860724853-74b7729d-ff54-4e67-b057-cc860981f2d8.png#averageHue=%23698154&clientId=uedab57ac-db0b-4&from=paste&height=478&id=ue3935374&originHeight=478&originWidth=925&originalType=binary&ratio=1&rotation=0&showTitle=false&size=66323&status=done&style=none&taskId=u43e57936-f5b5-494d-884c-d4b22306a08&title=&width=925)
效果：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710860807321-f004afaa-f466-4f35-90ea-0eba8eb15884.png#averageHue=%23fdfbfa&clientId=uedab57ac-db0b-4&from=paste&height=567&id=ufedec88f&originHeight=567&originWidth=995&originalType=binary&ratio=1&rotation=0&showTitle=false&size=42216&status=done&style=none&taskId=ue126846b-6f89-41e9-9a74-8787de200d7&title=&width=995)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1710860828339-a5753a39-3bad-4346-a228-ea3ad879ee3e.png#averageHue=%23f3e8d7&clientId=uedab57ac-db0b-4&from=paste&height=478&id=u60c8760c&originHeight=478&originWidth=828&originalType=binary&ratio=1&rotation=0&showTitle=false&size=72490&status=done&style=none&taskId=u76eeee16-5a65-4272-9dfc-989e414ca95&title=&width=828)
#### 2.5.2.4 网关测试2-路径重写
对应RewritePath GatewayFilter
需求：将/api/**请求（走网关）路由到renren-fast模块，例如获取验证码：
```typescript
http://localhost:8888/api/captcha.jpg?uuid=12134we （向网关发）
转为
http://localhost:8080/renren-fast/captcha.jpg?uuid=1213dn （renren-fast）

```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1711078106435-8357f88b-afdd-42f6-9c84-370d7cc43e7f.png#averageHue=%23141414&clientId=u2e192951-289d-4&from=paste&height=847&id=uc20fb684&originHeight=847&originWidth=844&originalType=binary&ratio=1&rotation=0&showTitle=false&size=35287&status=done&style=none&taskId=ud6dfc5ca-6707-4873-b814-12f006b7d7e&title=&width=844)

![image.png](https://cdn.nlark.com/yuque/0/2024/png/34372585/1711077977622-62562100-3a5c-484c-81f5-15e762e80ec2.png#averageHue=%23151515&clientId=u2e192951-289d-4&from=paste&height=793&id=ub3e76f73&originHeight=793&originWidth=882&originalType=binary&ratio=1&rotation=0&showTitle=false&size=35711&status=done&style=none&taskId=u24aa5815-90d0-4260-9071-75a2fafe7d6&title=&width=882)
实现：网关模块配上路由规则
注意：使用uri: lb://renren-fast，必须引入spring-cloud-starter-loadbalancer依赖（负载均衡），否则会报503错误。
```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    gateway:
      routes:
        # 路径重写
        - id: rewrite-path
          uri: lb://renren-fast  #路由到renren-fast，必须引入spring-cloud-starter-loadbalancer依赖，否则会报503错误
          predicates:
          - Path=/api/**
          filters:
          - RewritePath=/api/?(?<segment>.*), /renren-fast/$\{segment} # /api/**  -> /renren-fast/**
```
