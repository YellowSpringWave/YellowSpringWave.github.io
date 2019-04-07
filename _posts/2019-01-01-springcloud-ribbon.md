---
title: springcloud--ribbon配置使用
tags: springcloud
      ribbon
article_header:
  type: cover
  image:
    src:
key: ribbon
sharing: true
show_author_profile: false
comment: true
aside:
  toc: true
articles:
  type: normal
  show_excerpt: true
  excerpt_type: html
---

**简**：微服务 **ribbon** 的配置和使用，ribbon 是对消费者而言的，主要是两种，一种是 消费端``ribbon+eureka`` , 另一种是 ``ribbon不依赖eureka``

<!--more-->

作者 : 同德里小黄黄

# 一. 消费端 ribbon+eureka

## 目录结构

+ 注册中心：eureka服务
+ 提供方：eureka客户端（user-service）
+ 消费方：eureka客户端（api-gateway）

**``业务说明：拿登录操作来演示服务间的通信流程``**

## 接入ribbon步骤

+ pom增加ribbon依赖，发现`消费端具有eurek依赖的时候，里面包含了ribbon，所以这一步可以省略`{:.warning}
+ RestTemplate添加@LoadBalanced
+ 触发服务观察

## api-gateway（消费端代码片段）

+ UserController

```java
  //----------------------------登录流程-------------------------------------------
  
  
  @RequestMapping(value="/accounts/signin",method={RequestMethod.POST,RequestMethod.GET})
  public String loginSubmit(HttpServletRequest req){
    String username = req.getParameter("username");
    String password = req.getParameter("password");
    if (username == null || password == null) {
      req.setAttribute("target", req.getParameter("target"));
      return "/user/accounts/signin";
    }
    User  user =  accountService.auth(username, password);
    if (user == null) {
      return "redirect:/accounts/signin?" + "username=" + username + "&" + ResultMsg.errorMsg("用户名或密码错误").asUrlParams();
    }else {
      UserContext.setUser(user);
      return  StringUtils.isNotBlank(req.getParameter("target")) ? "redirect:" + req.getParameter("target") : "redirect:/index";
    }
  }
```

+ AccountService

```java
  /**
   * 校验用户名密码并返回用户对象
   * @param username
   * @param password
   * @return
   */
  public User auth(String username, String password) {
    if (StringUtils.isBlank(username) || StringUtils.isBlank(password)) {
       return null;
    }
    User user = new User();
    user.setEmail(username);
    user.setPasswd(password);
    try {
      user = userDao.authUser(user);
    } catch (Exception e) {
      return null;
    }
    return user;
  }
```

+ UserDao

```java
  @Autowired
  private GenericRest rest;
  @Value("${user.service.name}")
  private String userServiceName; 
  //调用远端的鉴权服务
  public User authUser(User user) {
    //userServiceName 是在注册中心注册的服务的名称，上面已经@Value{}获取了，这里没展示
    //通过服务名称代替之前的ip+端口，实现调用
    String url = "http://" + userServiceName + "/user/auth";
    ResponseEntity<RestResponse<User>> responseEntity =  rest.post(url, user, 
               new ParameterizedTypeReference<RestResponse<User>>() {});
    RestResponse<User> response = responseEntity.getBody();
    if (response.getCode() == 0) {
      return response.getResult();
   }{
      throw new IllegalStateException("Can not add user");
   }
  }
```

+ RestTemplate配置（**直连** 和 **服务名称调用**）

```java
package com.mooc.house.api.config;

import java.nio.charset.Charset;
import java.util.Arrays;

import org.apache.http.client.HttpClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.http.converter.StringHttpMessageConverter;
import org.springframework.web.client.RestTemplate;

import com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter4;

@Configuration
public class RestAutoConfig {

	public static class RestTemplateConfig {

		@Bean
		@LoadBalanced //spring 对restTemplate bean进行定制，加入loadbalance拦截器进行ip:port的替换
		RestTemplate lbRestTemplate(HttpClient httpclient) {
			RestTemplate template = new RestTemplate(new HttpComponentsClientHttpRequestFactory(httpclient));
			template.getMessageConverters().add(0,new StringHttpMessageConverter(Charset.forName("utf-8")));
			template.getMessageConverters().add(1,new FastJsonHttpMessageConvert5());
			return template;
		}
		
		@Bean
		RestTemplate directRestTemplate(HttpClient httpclient) {
			RestTemplate template = new RestTemplate(new HttpComponentsClientHttpRequestFactory(httpclient));
		    template.getMessageConverters().add(0,new StringHttpMessageConverter(Charset.forName("utf-8")));
		    template.getMessageConverters().add(1,new FastJsonHttpMessageConvert5());
		    return template;
		}
		
		 public static class FastJsonHttpMessageConvert5 extends FastJsonHttpMessageConverter4{
	          
	          static final Charset DEFAULT_CHARSET = Charset.forName("UTF-8");
	          
	          public FastJsonHttpMessageConvert5(){
	            setDefaultCharset(DEFAULT_CHARSET);
	            setSupportedMediaTypes(Arrays.asList(MediaType.APPLICATION_JSON,new MediaType("application","*+json")));
	          }

	        }
	}
}

```

+ 封装好的服务调用类

```java
package com.mooc.house.api.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

/**
 * 既支持直连又支持服务发现的调用
 *
 */
@Service
public class GenericRest {
	
	@Autowired
	private RestTemplate lbRestTemplate;
	
	@Autowired
	private RestTemplate directRestTemplate;
	
	private static final String directFlag = "direct://";
	
	public <T> ResponseEntity<T> post(String url,Object reqBody,ParameterizedTypeReference<T> responseType){
		RestTemplate template = getRestTemplate(url);
		url = url.replace(directFlag, "");
		return template.exchange(url, HttpMethod.POST,new HttpEntity(reqBody),responseType);
	}
	
	public <T> ResponseEntity<T> get(String url,ParameterizedTypeReference<T> responseType){
		RestTemplate template = getRestTemplate(url);
		url = url.replace(directFlag, "");
		return template.exchange(url, HttpMethod.GET,HttpEntity.EMPTY,responseType);
	}

	private RestTemplate getRestTemplate(String url) {
		if (url.contains(directFlag)) {
			return directRestTemplate;
		}else {
			return lbRestTemplate;
		}
	}

}

```



## user-service（远端的user服务，提供者代码片段）

+ controller

```java
 //------------------------登录/鉴权-------------------------- 
  @RequestMapping("auth")
  public RestResponse<User> auth(@RequestBody User user){
    User finalUser = userService.auth(user.getEmail(),user.getPasswd());
    return RestResponse.success(finalUser);
  }
```

+ UserService（直接惊动自己的mapper完成数据库查询）

```java
  /**
   * 校验用户名密码、生成token并返回用户对象
   * @param email
   * @param passwd
   * @return
   */
  public User auth(String email, String passwd) {
    if (StringUtils.isBlank(email) || StringUtils.isBlank(passwd)) {
      throw new UserException(Type.USER_AUTH_FAIL,"User Auth Fail");
    }
    User user = new User();
    user.setEmail(email);
    user.setPasswd(HashUtils.encryPassword(passwd));
    user.setEnable(1);
    List<User> list =  getUserByQuery(user);
    if (!list.isEmpty()) {
       User retUser = list.get(0);
       //校验完毕，通过JWT生成token
       onLogin(retUser);
       return retUser;
    }
    throw new UserException(Type.USER_AUTH_FAIL,"User Auth Fail");
  }
 public List<User> getUserByQuery(User user) {
    List<User> users = userMapper.select(user);
    users.forEach(u -> {
      u.setAvatar(imgPrefix + u.getAvatar());
    });
    return users;
  }
  private void onLogin(User user) {
    String token =  JwtHelper.genToken(ImmutableMap.of("email", user.getEmail(), "name", user.getName(),"ts",Instant.now().getEpochSecond()+""));
    renewToken(token,user.getEmail());
    user.setToken(token);
  }
```

``上述只是为了简单体现远程调用，所以代码进行了简写，其中还有拦截器判断cookie，部分依赖没有列举出来``{:.warning}

## 总结上述流程：

+ 消费网关API接受浏览器的请求
+ 请求到消费的service,dao
+ 消费dao通过restTemplate远程调用,    http://服务名+/user/auth
+ **在服务端user-service另外添加一个application-other.properties,我们可以通过不同的配置，启动两个提供服务，进入项目根目录，启动命令是`` mvn springboot:run -Dspring.profile.active=other  ``**
+ 然后每次浏览器访问发现，1次是A端口，1次是B端口，轮询着来

***********************

# 二 . ribbon脱离eureka

## 说明：

+ ribbon脱离eureka，也就是消费端pom不引入eureka相关的依赖。所以就需要引入ribbon的依赖
+ 配置文件需要添加``listOfServers``等配置

## 配置：

+ application.properties

```properties
#服务列表
user.ribbon.listOfServers=127.0.0.1:8083,127.0.0.1:8082
```

+ ribbon配置类代码

```java
package com.mooc.house.api.config;

import com.netflix.loadbalancer.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import com.netflix.client.config.IClientConfig;

public class NewRuleConfig {
	
	@Autowired
	private IClientConfig ribbonClientConfig;
	
	@Bean
	public IPing ribbonPing(IClientConfig config){
        //Iping ribbon每隔10秒向服务端列表发送一次ping请求，看是否成功或者失败
        //默认noOpPing不发送任何的ping命令
        //我们这里配置成Ping为health ，因为提供端加入了spring Acutor监控
		return new PingUrl(false,"/health");
	}
	
	@Bean
	public IRule ribbonRule(IClientConfig config){
         //return new RandomRule();
		//下面这个负载均衡策略更智能，loadBalance拦截之后，会对当前服务调用 成功失败结果进行记录
		//记录下一次，优先调用之前成功调用的
		return new AvailabilityFilteringRule();
        //WeightedResponseTimeRule 这个策略是根据响应时间分配权重
	}

}


```

+ 启动类代码

```java
@SpringBootApplication
//@EnableDiscoveryClient
//调用名称"user"服务的时候，使用NewRuleConfig配置
@RibbonClient(name="user",configuration=NewRuleConfig.class)
public class ApiGatewayApplication {

	public static void main(String[] args) {
		SpringApplication.run(ApiGatewayApplication.class, args);
	}
	
	@Autowired
	private DiscoveryClient discoveryClient;
	
	@RequestMapping("index1")
	@ResponseBody
	public List<ServiceInstance> getReister(){
	  return discoveryClient.getInstances("user");
	}
}	
```

# 三. 祭出一张 eureka.ribbon.restTemplate三者关系图

``图片引自 慕课网``

![ribbon+eureka+restTemplate](/assets/huangImg/mook.png)

# ending