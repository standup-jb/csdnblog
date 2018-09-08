## SpringMVC HandlerInterceptor使用介绍
### 背景
因为公司新开发了一个统一弹屏项目，项目有一个后台管理界面，有管理页面就必然有登录，注册，授权等一系列的操作。如何实现对用户登录的判断和拦截。一般情况下我们可以使用Filter(过滤器)和Interceptor(拦截器)来实现。

### Filter和Interceptor比较
Spring的Interceptor与Servlet的Filter有相似之处，比如二者都是面向AOP编程。都能够实现权限检查，日志记录等。但是他们的实现和原理还是有本质的区别。

|Filter|Interceptor|Summary|
|---|---|---|
|Filter接口定义在Javax.servlet包中|接口HandlerInterceptor定义在org.springframework.web.servlet包中||
|Filter定义在web.xml中|||
|Filter在Servlet前后起作用,Filters通常将请求和相应当做黑河，Filter通常不考虑servlet的实现|拦截器能够深入到方法前后,异常抛出前后等，因此拦截器的使用具有更大的弹性。允许用户(hook into)请求什么的周期，在请求过程中获取信息，Interceptor通常和请求更加耦合|在Spring架构程序中，要优先使用拦截器。几乎所有Filter能做的事情，Interceptor都可以实现|
|Filter是Servlet规范规定的|而拦截器即可以用于web程序，也可以用于Application，Swing程序|使用范围不同|
|Filter是Servlet规范中定义的，是Servlet容器支持的|拦截器在Spring容器内，由Spring进行管理|规范不同|
|Filter不能够使用Spring容器资源|拦截器是Spring的组件，归Spring管理，配置在Spring文件中，因此可以使用Spring里的任何资源，对象等|Spring中使用Interceptor更加容易|
|Filter是被Server(tomcat etc)调用|Interceptor是被Spring调用|因此Filter总是优先于Intreceptor执行|
因为Filter是作用在Servlet前。Interceptor执行在Action前。所以正确的处理流程是

```
Filter前处理 --> Interceptor前处理 --> action --> Interceptor后处理 --> Filter后处理

```
如果单纯的登录Session校验的话，将校验的代码放在Interceptor和Filter里面比较的话，可以放在Filter里面会更好，因为这样请求都不会到Spring里面就已经被Servlet拒绝了。这样可以更加好的保证Spring的安全。

* 因为我们的项目需要根据权限去拦截，所以需要根据用户信息去查询数据库表。这就不可避免的要使用Spring的组件了，这种情况下就只能使用Interceptor处理了。


### Interceptor实现
下面将介绍如何使用Interceptor实现拦截功能
#### 一 创建Class实现HandlerInterceptor接口

```
public class AuthInterceptor implements HandlerInterceptor {

    //支持注入Spring相关的bean组件
    @Autowired
    IUserPrivilegeService privilegeService;
    @Override
    public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o) throws Exception {
      	//在执行Controller函数前先触发此方法
    }

    @Override
    public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView) throws Exception {
		//在执行完Controller函数后会触发此方法
    }

    @Override
    public void afterCompletion(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) throws Exception {

    }
```

#### 二 在方法的preHandle，postHandle,afterCompletion实现拦截逻辑

```

 @Override
    public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o) throws Exception {
       
       //实现具体处理逻辑    
    }
    
```

#### 三 在Spring配置文件里面加入Interceptor的配置

```

<mvc:interceptors>
        <bean class="com.XXX.XXX.auth.filter.AuthInterceptor"/>
</mvc:interceptors>

```

### 处理细节指导
我们在preHandle方法里面可以通过httpServletRequest获取到请求的全部参数,而且我们可以通过Autowired的方式将我们需要使用到的Spring的bean注入进来。preHandle方法是有一个boolean返回值的。

* preHandle 返回true,程序正常往下执行。
* preHandle 返回false,请求中断不会网线执行。

#### 小Demo
实现一个简单的通过用户名去判断这个User是否存在，如果不存在就重定向到登录页面

```
public class LoginInterceptor implements HandlerInterceptor{
	
	@Autowired
	UserService userService;
	
	@Override
	public boolean preHandle(HttpServletRequest request,HttpServletResponse responseObject o){
		String name = request.getParameter("name");
		User user = userService.queryByName(name);
		if(user==null){
			response.sendRedirect("toLogin");
			return false;
		}
		return true;
	}
}

```

#### 放在最后
希望每一篇博客对看完这篇博客的用户是有用的，有帮助的。是没有浪费你的时间的。如果觉得博客写得还不错可以关注我，也可以评论我。实现喜欢也可以给我打钱。我的目标是自己快速的成长，




