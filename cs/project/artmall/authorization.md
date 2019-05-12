---
title: artmall——authorization
date: 2018-09-25 15:02:20
tags: artmall
categories: project
---

## shiro中Authorization基础知识
授权，即权限访问控制。   
我们通过角色来规定权限，将用户进行角色的分类   
shiro提倡使用权限和明确为角色指定权限来代替将权限隐含与角色中的方法。    
### 基于资源的权限管理(Resource-Based Access Control)   
以下的例子就是将权限隐含在角色中,对于每一个权限控制，就直接判断是否属于这个角色,但是如果又有一个角色可以访问这个权限，就需要找到这个代码段，在if后面加上。如此代码就会很难扩展。   
```java
if (user.hasRole("Project Manager") || user.hasRole("Department Manager") ) {
    //show the project report button
} else {
    //don't show the button
}
```  
以下就是Resource-Based Access Control,对于访问的资源判断这个user或者这个role是否含有这个权限。

```java 
if (user.isPermitted("projectReport:view:12345")) {
    //show the project report button
} else {
    //don't show the button
}
```   

### Realm
Subject实际上是shiro的用户，Subject通过与角色或者权限关联来确定是否被允许执行程序内特定的动作(即是否有此权限)   
Realm从数据库里取出数据，告诉shiro这个角色或者权限是否存在(所以数据库里面需要权限控制的表，即五张表的关系)    

### 授权对象  
>在 Shiro 中执行授权可以有三种途径：   
> - 程序代码--你可以在你的 JAVA 代码中执行用类似于 if 和 else 的结构来执行权限检查。   
> - JDK 注解--你可以在你的 JAVA 方法上附加权限注解     
> - JSP/GSP 标签--你可以基于角色和权限控制 JSP 或 GSP 页面输出内容。   
  


## 权限控制的具体实现    
通过以上的基础知识，我们知道，要实现权限控制，我们需要对资源进行控制   
### 1. 建立五张表   
### 2. 获取角色和权限   
因为我们是使用jwt进行身份的验证，jwt里面也存储了用户的id，所以可以直接提取出jwt里面的id，去数据库里面查询用户的角色和权限，重写了`AuthorizingRealm`的`doGetAuthorizationInfo`方法   
>问题：在用户访问界面的时候要进行拦截，shiroFilter如何定义？在这里获取了role的用处只不过是赋予这个user一个role，但是没有起到权限控制的作用。   
回答：通过permissions来控制，在拦截的时候调用此方法，来判断是否有权限   

- 拦截器的自定义   
 

  
### 3. 代码实现   
当一个请求发过来，先被Shiro的FilterChain进行拦截，如果有符合的url，就进入对应的拦截器，在拦截器里面执行`subject.login(token)`就会调用Realm中的`doGetAuthenticationInfo`和数据库中的数据进行匹配，验证   
在拦截器中执行`suject.hasrole("Admin")`或者`subject.isPermitted("\admin\test")`就会调用Realm中的`doGetAuthorizationInfo`查找数据库，进行权限的匹配。    
现在的思路，因为我们不使用注释来进行权限控制(因为我们的权限控制是动态的从数据库中查找)所以只要获取到请求的url,使用subject.isPermitted(请求的资源)来和数据库中的资源进行匹配，如果匹配成功，则允许访问，匹配失败，则反之。   
- 自定义拦截链，因为是使用jwt进行登录，所以对所有请求进行拦截，进入JWTFilter()        
```java 代码体现  
    @Configuration
public class ShiroConfiguration {



    @Bean
    public ShiroFilterFactoryBean shirFilter(SecurityManager securityManager){
        System.out.println("ShiroConfiguration.shirFilter()");
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager);
        //把自定义的filter集成到shiro配置里面，添加自定义拦截器
        Map<String,Filter> filterMap = new HashMap<>();
        filterMap.put("jwt", new JWTFilter());
        shiroFilterFactoryBean.setFilters(filterMap);
        //定义url规则
        Map<String,String> filterRuleMap = new HashMap<>();

        filterRuleMap.put("/**","jwt");
        filterRuleMap.put("401","anon");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterRuleMap);
        return shiroFilterFactoryBean;
    }
```     
- 自定义拦截器     
 我们先来看一下官方默认的拦截器如何实现的，比如说`RolesAuthorizationFilter`通过重写`isAccessAllowed`，我们可以模仿官方的写法    
```java
  public class RolesAuthorizationFilter extends AuthorizationFilter {
    public RolesAuthorizationFilter() {
    }
    public boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws IOException {
        Subject subject = this.getSubject(request, response);
        String[] rolesArray = (String[])((String[])mappedValue);
        if (rolesArray != null && rolesArray.length != 0) {
            Set<String> roles = CollectionUtils.asSet(rolesArray);
            return subject.hasAllRoles(roles);
        } else {
            return true;
        }
    }
}   
```    
下面是我的具体实现 
```java   
    /**
     * 判断用户是否想要登入。
     * 拦截ajax请求，判断shi
     * 检测header里面是否包含Authorization字段即可
     */
    @Override
    protected boolean isLoginAttempt(ServletRequest request, ServletResponse response) {
        HttpServletRequest req = (HttpServletRequest) request;
        String authorization = req.getHeader("Authorization");
        return authorization != null;
    }

     /**
     * 执行登录
     */
    @Override
    protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        //获取token
        String authorization = httpServletRequest.getHeader("Authorization");
        JWTToken token = new JWTToken(authorization);
        // 提交给realm进行登入，如果错误他会抛出异常并被捕获
        getSubject(request, response).login(token);
        // 如果没有抛出异常则代表登入成功，返回true
        return true;
    }   

    /**
     * 这一段是重点，在前面的filter拦截下后，实例了JWTFilter，就会调用此方法
     * 来判断是否可以通过。mapppedValue在我调试的时候是null，源码调试放在后面说下    
     **/
     @Override
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        LOGGER.debug("拦截开始");
        if (isLoginAttempt(request, response)) {
            //执行登录，看token是否有效
            try {
                executeLogin(request, response);
                flag = 1;    //有效则使flag为1；
            } catch (Exception e) {
                return false;  //token如果不匹配，则返回false，执行onAccessDenied对访问拒绝进行处理
            }
            //获取当前subject
            Subject subject = this.getSubject(request, response);
            //获取到请求的url
            HttpServletRequest httpServletRequest = (HttpServletRequest) request;
            String perm = httpServletRequest.getRequestURI();
            //对获取到的url，使用isPermitted(url)来调用realm中的doGetAuthorizationInfo,查找数据库看是否具有此权限
            if (perm != null && perm.length()>0){
                if (subject.isPermitted(perm)) {
                    return true;
                }
            }
        }
        return false; //执行onAccessDenied对拒绝访问进行处理;
    }   


     /**
     * 对返回的false进行处理
     * @param request
     * @param response
     * @return
     * @throws Exception
     */
    @Override
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
        if (flag == 0){
            throw new Exception("token错误");
        }else {
            throw new Exception("没有此权限");
        }
    }

```   

- 权限和身份的验证  
  这个就要和数据库打交道，所以自然是配置realm，之前的博客里说了，我是使用了多realm    
  但是每一次验证会把所有的realm都验证一遍，之前设置了只要通过一个就算是通过    
  权限的验证统一使用token进行验证，所有除了JWTRealm中写了`doGetAuthorizationInfo`，其他的都返回null,关于登录的验证就请参照前面的博客，这里就只放出权限的验证     
  ```java   
   /**
     * 通过token获取到id，然后获取到角色和权限
     *因为我们是使用jwt进行身份的验证，jwt里面也存储了用户的id   
     *所以可以直接提取出jwt里面的id，去数据库里面查询用户的角色和权限   
     *重写了`AuthorizingRealm`的`doGetAuthorizationInfo`方法   
     * @param principalCollection
     * @return
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        String token = (String) principalCollection.getPrimaryPrincipal();
        Long userId = JWTUtil.getUserNo(token);
        SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();
        simpleAuthorizationInfo.setRoles(userService.getRoles(userId));
        simpleAuthorizationInfo.setStringPermissions(userService.getPermissions(userId));
        return simpleAuthorizationInfo;
    }
  ```
  
还存在的问题:   
- 只做到了抛出异常，但是不知道在哪里对异常进行捕获，然后返回规定的响应码    

## 源码调试与分析   

### 1. FilterChain加载顺序  


>Shiro对Servlet容器的FilterChain进行了代理，即ShiroFilter在继续Servlet容器的Filter链的执行之前，通过ProxiedFilterChain对Servlet容器的FilterChain进行了代理；即先走Shiro自己的Filter体系，然后才会委托给Servlet容器的FilterChain进行Servlet容器级别的Filter链执行；Shiro的ProxiedFilterChain执行流程：1、先执行Shiro自己的Filter链；2、再执行Servlet容器的Filter链（即原始的Filter）。       
login和show的界面是任何用户都可以访问的   
```java  
        Map<String,String> filterRuleMap = new HashMap<>();
        filterRuleMap.put("/*/login","anon");
        filterRuleMap.put("/show","anon");
        filterRuleMap.put("/**","jwt");
        filterRuleMap.put("401","anon");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterRuleMap);
        return shiroFilterFactoryBean;
```   

对`shiroFilterFactoryBean`加断点，查看filterRuleMap的顺序   
 ![1](http://prdj7tprx.bkt.clouddn.com/shiroFilterFactoryBean.png)

### 过滤器  
过滤器是一种代码重用技术，可以改变HTTP请求的内容，相应，及header信息。   
当容器接收到传入的请求时，它将获取列表中的第一个过滤器并调用doFilter方法，传入ServletRequest 和 ServletResponse，和一个它将使用的FilterChain对象的引用。 

参考:
http://jinnianshilongnian.iteye.com/blog/2025656    
http://www.hillfly.com/2017/179.html/comment-page-1
  