---
title: artmall:spingboot整合shiro-基本登录实现
date: 2018-09-06 21:40:08
tags: artmall
categories: project
---

我要实现的shiro是多realm且使用jwt进行数据的传输（因为是前后端分离）,一开始我按照网上的直接进行整合，没有通过测试，发现一大堆的错误，却又不知道怎么该。带我的一个小姐姐要我重新全部搭过，一步一步进行测试,所以我要实现shiro的整合一共分为4步：

1、基础的登录实现（就是本篇）

2、实现token

3、实现多realm

4、权限控制

<!-- more -->

一、基础知识   
### Authentication Sequence(认证序列)
1. 代码调用Subject.login方法，向AuthenticationToken(认证令牌)实例得构造函数传递用户得身份和证明
2. DelegationSuject通过调用securityManager.login(token)将令牌转交给程序得SecurityManager(如何实现的？后期要进行源码分析)
3. SecurityManager通过调用authenticator.authenticate(token)将其转交给Authenticator实例(通常为ModularRealmAuthenticator)
4. 如果配置了多个Realm，ModularRealmAuthenticator实例将使用起配置的AuthenticationStrategy开始多Realm身份验证的尝试。
5. 每一个配置的 Realm 都被检验看其是否支持)提交的AuthenticationToken，如果支持，则该 Realm 的 getAuthenticationInfo) 方法随着提交的牌被调用，getAuthenticationInfo
方法为特定的 Realm 有效提供一次独立的验证尝试

二、步骤

1、添加依赖
```xml
    <dependency>
        <groupId>org.apache.shiro</groupId>
        <artifactId>shiro-spring</artifactId>
        <version>1.4.0</version>
    </dependency>
```

2、在ssm结构里是用xml文件进行配置，但是springboot里基本不用xml文件，用JavaBean进行配置
```java
    //新建一个ShiroConfiguration.java
    /**
     * shiro核心通过Filter实现，通过URL规则进行过滤和权限校验
     * 配置shiro，相当于ssm结构中xml文件
     */
    @Configuration
    public class ShiroConfiguration {
    
    		//配置过滤器（核心配置）
    		@Bean
        public ShiroFilterFactoryBean shirFilter(SecurityManager securityManager){
            System.out.println("ShiroConfiguration.shirFilter()");
            ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
    				//一定要
            shiroFilterFactoryBean.setSecurityManager(securityManager);
    				Map<String,String> filterChainDefinitionMap =  new LinkedHashMap<String,String>();
            //配置不会被拦截的链接 顺序判断
            //anon:所有url都可以匿名访问
            filterChainDefinitionMap.put("/static/**","anon");
            //配置退出,shiro已内置logout过滤器
            filterChainDefinitionMap.put("/logout","logout");
            //
            filterChainDefinitionMap.put("/register","anon");
            filterChainDefinitionMap.put("/hello","anon");
            filterChainDefinitionMap.put("/info","anon");
            filterChainDefinitionMap.put("/login","anon");
    
    
            //表示需要认证，没有登陆是不能访问的
            filterChainDefinitionMap.put("/**","authc");
            //配置shiro默认登陆界面，不过在前后端分离中应该有前端路由控制
            shiroFilterFactoryBean.setLoginUrl("/unauth");
    				//未授权界面;
            shiroFilterFactoryBean.setUnauthorizedUrl("/403");
            shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
    
            return shiroFilterFactoryBean;
    
    		//定义自己的myRealm，要重写其验证方式
    		@Bean
    		public MyRealm myRealm(){
            MyRealm myRealm = new MyRealm();
    	      return myRealm;
    		}
    
    		@Bean
        public SecurityManager securityManager(){
    	      System.out.println("securityManager.log");
            DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
    				securityManager.setRealm(myRealm());
    				return securityManager;
        }
    }
```

3、Realm：检查提交的凭证是否和后台数据匹配，要实现AuthorizingRealm里的抽象类doGetAuthenticationInfo，来实现用户的验证

原理：authenticationToken存放的是用户输入信息(通常为用户名和密码，这里的密码是未加密的)，要拿这个信息和数据库里的信息进行比对，shiro通过AuthenticatingRealm使用CredentialsMatcher里面的SimpleCredentialsMatcher进行凭证的比对。
```java
    //返回的信息
    public SimpleAuthenticationInfo(Object principal, Object hashedCredentials, ByteSource credentialsSalt, String realmName) {
            this.principals = new SimplePrincipalCollection(principal, realmName);
            this.credentials = hashedCredentials;
            this.credentialsSalt = credentialsSalt;
        }

    //用户输入的信息和用户在数据库存储的信息进行比对
    public boolean doCredentialsMatch(AuthenticationToken token, AuthenticationInfo info) {
            Object tokenCredentials = this.getCredentials(token);
            Object accountCredentials = this.getCredentials(info);
            return this.equals(tokenCredentials, accountCredentials);
        }

    public class MyRealm extends AuthorizingRealm {
    		//权限的验证，因为我们现在暂时只考虑登录，所以还没有写
        @Override
        protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
            return null;
        }
    
        @Autowired
        StudentService studentService;
        @Override
    		
        protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
            System.out.println("MyShiroRealm.doGetAuthenticationInfo()");
            UsernamePasswordToken token = (UsernamePasswordToken)authenticationToken;
    				//根据用户输入的studentid查询数据库里student的信息
            Student student = studentService.selectStudentByStuId(token.getUsername());
    				//获取数据库里存储的此student的salt
            ByteSource salt = ByteSource.Util.bytes(student.getSalt());
            if (student!=null){ 
    						//返回的信息包括了数据库了关于此student的登录凭证
    						return new SimpleAuthenticationInfo(student.getStudentId(),student.getHashedPwd(),salt,getName());
            }
            return null;
        }
```

4、密码加密加盐

shiroConfiguration.java

刚我们说验证使用的是SimpleCredentialsMatcher里面的doCredentialsMatch()进行比对,但是此时，我们数据库里的密码是进行了md5加密的，但是用户输入的密码是没有加盐加密的，所以肯定是不匹配的，HashedCredentialsMatcher extends SimpleCredentialsMatcher，配置使用HashedCredentialsMatcher，实现md5加密。
```java
    		@Bean
        public HashedCredentialsMatcher hashedCredentialsMatcher(){
            HashedCredentialsMatcher hashedCredentialsMatcher = new HashedCredentialsMatcher();
    
            hashedCredentialsMatcher.setHashAlgorithmName("MD5");//散列算法:这里使用MD5算法;
            hashedCredentialsMatcher.setHashIterations(1024);//散列的次数，比如散列两次，相当于 md5(md5(""));
    
            return hashedCredentialsMatcher;
        }
    		@Bean
    	  public MyRealm myRealm(){
            MyRealm myRealm = new MyRealm();
            myRealm.setCredentialsMatcher(hashedCredentialsMatcher());
           return myRealm;
    	  }
```

注意：录入信息的时候，也用使用一样的加密方式，最好不要用自己写的MD5工具类，不然生成的不一样。
```java
    newStudent.setHashedPwd(new SimpleHash("MD5",student.getHashedPwd(),ByteSource.Util.bytes(newStudent.getSalt()),1024).toString());
```

三、踩过的坑
```java
    java.lang.IllegalStateException: ApplicationEventMulticaster not initialized - call 'refresh' before multicasting events via the context
    
    ***************************
    APPLICATION FAILED TO START
    ***************************
    
    Description:
    
    Parameter 0 of method shiroFilter in com.artmuseum.artmuseumweb.shiro.config.ShiroConfiguration required a bean of type 'com.artmuseum.artmuseummanager.service.StudentService' that could not be found.
    
    
    Action:
    
    Consider defining a bean of type 'com.artmuseum.artmuseummanager.service.StudentService' in your configuration.

    org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'shirFilter' defined in class path resource [com/artmall/shiro/config/ShiroConfiguration.class]: Unsatisfied dependency expressed through method 'shirFilter' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'securityManager' defined in class path resource [com/artmall/shiro/config/ShiroConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.apache.shiro.mgt.SecurityManager]: Factory method 'securityManager' threw exception; nested exception is java.lang.ClassCastException: java.util.ArrayList cannot be cast to org.apache.shiro.realm.Realm
    	
    Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'securityManager' defined in class path resource [com/artmall/shiro/config/ShiroConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.apache.shiro.mgt.SecurityManager]: Factory method 'securityManager' threw exception; nested exception is java.lang.ClassCastException: java.util.ArrayList cannot be cast to org.apache.shiro.realm.Realm
    
    Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.apache.shiro.mgt.SecurityManager]: Factory method 'securityManager' threw exception; nested exception is java.lang.ClassCastException: java.util.ArrayList cannot be cast to org.apache.shiro.realm.Realm
    	at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:185) ~[spring-beans-5.0.8.RELEASE.jar:5.0.8.RELEASE]
    	at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:582) ~[spring-beans-5.0.8.RELEASE.jar:5.0.8.RELEASE]
    	... 37 common frames omitted
    Caused by: java.lang.ClassCastException: java.util.ArrayList cannot be cast to org.apache.shiro.realm.Realm
    	at com.artmall.shiro.config.ShiroConfiguration.securityManager(ShiroConfiguration.java:108) ~[classes/:na]
    	at com.artmall.shiro.config.ShiroConfiguration$$EnhancerBySpringCGLIB$$926e0583.CGLIB$securityManager$0(<generated>) ~[classes/:na]
    	at com.artmall.shiro.config.ShiroConfiguration$$EnhancerBySpringCGLIB$$926e0583$$FastClassBySpringCGLIB$$e22b6d11.invoke(<generated>) ~[classes/:na]
    	at org.springframework.cglib.proxy.MethodProxy.invokeSuper(MethodProxy.java:228) ~[spring-core-5.0.8.RELEASE.jar:5.0.8.RELEASE]
    	at org.springframework.context.annotation.ConfigurationClassEnhancer$BeanMethodInterceptor.intercept(ConfigurationClassEnhancer.java:361) ~[spring-context-5.0.8.RELEASE.jar:5.0.8.RELEASE]
    	at com.artmall.shiro.config.ShiroConfiguration$$EnhancerBySpringCGLIB$$926e0583.securityManager(<generated>) ~[classes/:na]
    	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_161]
    	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_161]
    	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_161]
    	at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_161]
    	at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:154) ~[spring-beans-5.0.8.RELEASE.jar:5.0.8.RELEASE]
    	... 38 common frames omitted
```

- 添加解密规则后  
       
```java
//配置文件里增加使用hash解密规则
    @Bean(name = "hashedCredentialsMatcher")
        public HashedCredentialsMatcher hashedCredentialsMatcher(){
            System.out.println("解密规则");
            HashedCredentialsMatcher hashedCredentialsMatcher = new HashedCredentialsMatcher();
            //采用MD5方式加密
            hashedCredentialsMatcher().setHashAlgorithmName("MD5");
            return hashedCredentialsMatcher();
        }
   

//但是依旧报错

    org.apache.shiro.UnavailableSecurityManagerException: No SecurityManager accessible to the calling code, either bound to the org.apache.shiro.util.ThreadContext or as a vm static singleton. This is an invalid application configuration.


//估计是加密方法有问题，就改了加密次数
    @Bean
        public HashedCredentialsMatcher hashedCredentialsMatcher(){
    
            HashedCredentialsMatcher hashedCredentialsMatcher = new HashedCredentialsMatcher();
            //采用MD5方式加密
            hashedCredentialsMatcher().setHashAlgorithmName("MD5");
            hashedCredentialsMatcher.setHashIterations(1);
    //        System.out.println("解密规则");
            return hashedCredentialsMatcher();
        }

 //依旧报错
    Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.apache.shiro.authc.credential.HashedCredentialsMatcher]: Factory method 'hashedCredentialsMatcher' threw exception; nested exception is java.lang.StackOverflowError
    	at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:185) ~[spring-beans-5.0.8.RELEASE.jar:5.0.8.RELEASE]
    	at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:582) ~[spring-beans-5.0.8.RELEASE.jar:5.0.8.RELEASE]
    	... 65 common frames omitted
    Caused by: java.lang.StackOverflowError: null
```

最后发现是，在密码录入数据库的时候加密方式和验证方式不是同一种，自然不能匹配，把验证方式改为一致后，就成功了！