---
layout:     post
title:      spring cloud oauth2 
subtitle:   spring cloud oauth2 中组件详解
date:       2019-06-03
author:     chen
catalog: 	 true
tags:
    - spring
    - oauth2
---

## oauth中的角色
* client：调用资源服务器API的应用
* Oauth 2.0 Provider：包括Authorization Server和Resource Server
1.Authorization Server：认证服务器，进行认证和授权
2.Resource Server：资源服务器，保护受保护的资源
* user：资源的拥有者

## Spring Cloud Security Oauth2认证流程
为了便于理解，现在假设有一个名叫“脸盆网”的社交网站，用户在首次登陆时会要求导入用户在facebook的好友列表，以便于快速建立社交关系。具体的授权流程如下：

1. 用户登陆脸盆网，脸盆网试图访问facebook上的好友列表
2. 脸盆网发现该资源是facebook的受保护资源，于是返回302将用户重定向至facebook登陆页面
3. 用户完成认证后，facebook提示用户是否将好友列表资源授权给脸盆网使用（如果本来就是已登陆facebook状态则直接显示是否授权的页面）
4. 用户确认后，脸盆网通过授权码模式获取了facebook颁发的access_token
5. 脸盆网携带该token访问facebook的获取用户接口https://api.facebook.com/user,facebook验证token无误后返回了与该token绑定的用户信息
6. 脸盆网的spring security安全框架根据返回的用户信息构造出了principal对象并保存在session中
7. 脸盆网再次携带该token访问好友列表，facebook根据该token对应的用户返回该用户的好友列表信息
8. 该用户后续在脸盆网发起的访问facebook上的资源，只要在token有效期及权限范围内均可以正常获取（比如想访问一下保存在facebook里的相册）
脸盆网就是第三方应用（client），而facebook既充当了认证服务器，又充当了资源服务器。
说明:
* 标准的OAuth2授权过程中，5、6、8这几步都不是必须的，从上面贴的RFC6749规范来看，只要有1、2、3、4、7这几步，就完成了被保护资源访问的整个过程。事实上，RFC6749协议规范本身也并不关心用户身份的部分，它只关心token如何颁发，如何续签，如何用token访问被保护资源.spring security是一套完整的安全框架，它必须关心用户身份！在实际的使用场景中，OAuth2一般不仅仅用来进行被保护资源的访问，还会被用来做单点登陆（SSO）。
* Spring Security在调用user接口成功后，会构造一个OAuth2Authentication对象，这个对象是我们通常使用的UsernamePasswordAuthenticationToken对象的一个超集，里面封装了一个标准的UsernamePasswordAuthenticationToken，同时在detail中还携带了OAuth2认证中需要用到的一些关键信息（比如tokenValue，tokenType等），这时候就完成了SSO的登陆认证过程。后续用户如果再想访问被保护资源，spring security只需要从principal中取出这个用户的token，再去访问资源服务器就行了，而不需要每次进行用户授权。
* 浏览器与client之间仍然是通过传统的cookie-session机制来保持会话，而非通过token。实际上在SSO的过程中，使用到token访问的只有client与resource server之间获取user信息那一次，token的信息是保存在client的session中的，而不是在用户本地。

## Oauth 2.0 Provider
### Authorization Server
1. AuthorizationEndpoint:进行授权的服务，Default URL: /oauth/authorize
2. TokenEndpoint：获取token的服务，Default URL: /oauth/token
### Resource Server
OAuth2AuthenticationProcessingFilter：给带有访问令牌的请求加载认证

## Authorization Server详情
一般情况下，创建两个配置类，一个继承AuthorizationServerConfigurerAdapter，一个继承WebSecurityConfigurerAdapter，再去复写里面的方法。

### 主要的两种注解
1. @EnableAuthorizationServer：声明一个认证服务器，当用此注解后，应用启动后将自动生成几个Endpoint：（注：其实实现一个认证服务器就是这么简单，加一个注解就搞定，当然真正用到生产环境还是要进行一些配置和复写工作的。）
/oauth/authorize：验证
/oauth/token：获取token
/oauth/confirm_access：用户授权
/oauth/error：认证失败
/oauth/check_token：资源服务器用来校验token
/oauth/token_key：如果jwt模式则可以用此来从认证服务器获取公钥
以上这些endpoint都在源码里的endpoint包里面。

2. @Beans 
需要实现AuthorizationServerConfigurer
AuthorizationServerConfigurer包含三种配置:
ClientDetailsServiceConfigurer：client客户端的信息配置，client信息包括：clientId、secret、scope、authorizedGrantTypes、authorities
（1）scope：表示权限范围，可选项，用户授权页面时进行选择
（2）authorizedGrantTypes：有四种授权方式 
Authorization Code：用验证获取code，再用code去获取token（用的最多的方式，也是最安全的方式）
Implicit: 隐式授权模式
Client Credentials (用來取得 App Access Token)
Resource Owner Password Credentials
（3）authorities：授予client的权限
这里的具体实现有多种，in-memory、JdbcClientDetailsService、jwt等。
AuthorizationServerSecurityConfigurer：声明安全约束，哪些允许访问，哪些不允许访问
AuthorizationServerEndpointsConfigurer：声明授权和token的端点以及token的服务的一些配置信息，比如采用什么存储方式、token的有效期等。 
client的信息的读取：在ClientDetailsServiceConfigurer类里面进行配置，可以有in-memory、jdbc等多种读取方式。
jdbc需要调用JdbcClientDetailsService类，此类需要传入相应的DataSource。

### 如何管理token
AuthorizationServerTokenServices接口:声明必要的关于token的操作
（1）当token创建后，保存起来，以便之后的接受访问令牌的资源可以引用它。
（2）访问令牌用来加载认证
接口的实现也有多种，DefaultTokenServices是其默认实现，他使用了默认的InMemoryTokenStore，不会持久化token。
### token存储方式
* InMemoryTokenStore：存放内存中，不会持久化
* JdbcTokenStore：存放数据库中
* Jwt: json web token,详见 http://blog.leapoahead.com/2015/09/06/understanding-jwt/

### 授权类型：
可以通过AuthorizationServerEndpointsConfigurer来进行配置，默认情况下，支持除了密码外的所有授权类型。相关授权类型的一些类：
（1）authenticationManager：直接注入一个AuthenticationManager，自动开启密码授权类型
（2）userDetailsService：如果注入UserDetailsService，那么将会启动刷新token授权类型，会判断用户是否还是存活的
（3）authorizationCodeServices：AuthorizationCodeServices的实例，auth code 授权类型的服务
（4）implicitGrantService：imlpicit grant
（5）tokenGranter：

### endpoint的URL的配置
（1）AuthorizationServerEndpointsConfigurer的pathMapping()方法，有两个参数，第一个是默认的URL路径，第二个是自定义的路径
（2）WebSecurityConfigurer的实例，可以配置哪些路径不需要保护，哪些需要保护。默认全都保护。
#### 自定义UI:
（1）有时候，我们可能需要自定义的登录页面和认证页面。登陆页面的话，只需要创建一个login为前缀名的网页即可，在代码里，设置为允许访问，这样，系统会自动执行你的登陆页。此登陆页的action要注意一下，必须是跳转到认证的地址。

（2）另外一个是授权页，让你勾选选项的页面。此页面可以参考源码里的实现，自己生成一个controller的类，再创建一个对应的web页面即可实现自定义的功能。

### 授权获取token流程
（1）端口号换成你自己的认证服务器的端口号，client_id也换成你自己的，response_type类型为code。
 localhost:8080/uaa/oauth/authorize?client_id=client&response_type=code&redirect_uri=http://www.baidu.com 

（2）这时候你将获得一个code值：

（3）使用此code值来获取最终的token：

curl -X POST -H "Cant-Type: application/x-www-form-urlencoded" -d 'grant_type=authorization_code&code=G0C20Z&redirect_uri=http://www.baidu.com' "http://client:secret@localhost:8080/uaa/oauth/token"

返回值：

{"access_token":"b251b453-cc08-4520-9dd0-9aedf58e6ca3","token_type":"bearer","expires_in":2591324,"scope":"app"}

（4）用此token值来调用资源服务器内容（如果资源服务器和认证服务器在同一个应用中，那么资源服务器会自己解析token值，如果不在，那么你要自己去做处理）

curl -H "Authorization: Bearer b251b453-cc08-4520-9dd0-9aedf58e6ca3" "localhost:8081/service2（此处换上你自己的url）"


## Resource Server：保护资源，需要令牌才能访问
在配置类上加上注解@EnableResourceServer即启动。使用ResourceServerConfigurer进行配置：
（1）tokenServices：ResourceServerTokenServices的实例，声明了token的服务
（2）resourceId：资源Id，由auth Server验证。
（3）其它一些扩展点，比如可以从请求中提取token的tokenExtractor
（4）一些自定义的资源保护配置，通过HttpSecurity来设置
 

使用token的方式也有两种：
（1）Bearer Token（https传输方式保证传输过程的安全）:主流
（2）Mac（http+sign）

### 如何访问资源服务器中的API？

如果资源服务器和授权服务器在同一个应用程序中，并且您使用DefaultTokenServices，那么您不必太考虑这一点，因为它实现所有必要的接口，因此它是自动一致的。如果您的资源服务器是一个单独的应用程序，那么您必须确保您匹配授权服务器的功能，并提供知道如何正确解码令牌的ResourceServerTokenServices。与授权服务器一样，您可以经常使用DefaultTokenServices，并且选项大多通过TokenStore（后端存储或本地编码）表示。
（1）在校验request中的token时，使用RemoteTokenServices去调用AuthServer中的/auth/check_token。
（2）共享数据库，使用Jdbc存储和校验token，避免再去访问AuthServer。
（3）使用JWT签名的方式，资源服务器自己直接进行校验，不借助任何中间媒介。

## oauth client
在客户端获取到token之后，想去调用下游服务API时，为了能将token进行传递，可以使用RestTemplate.然后使用restTemplate进行调用Api。
```
@Bean
public OAuth2RestOperations restOperations(
  OAuth2ProtectedResourceDetails resource, OAuth2ClientContext context) {
    return new OAuth2RestTemplate(resource, context);
}
```
注：
scopes和authorities的区别：
scopes是client权限，至少授予一个scope的权限，否则报错。
authorities是用户权限。

## 配置zuul
https://www.jianshu.com/p/24577532555d?utm_source=oschina-app



