---
layout:     post
title:      "Blog Program"
subtitle:   "Blog项目的笔记心得"
date:       2021-07-19 12:00:00
author:     "Pengyujie"
header-img: "img/tag-bg.jpg"
tags:
    - Java
    - SpringBoot

---

>码神之路Blog项目的笔记心得



### Blog项目笔记

#### ThreadLocal

> ThreadLocal是一个线程内部的存储类，每个线程都有一个ThreadLocal，用来存储该线程的信息，比如获取该线程的用户信息，但是是用完之后需要进行删除，不然有内存泄漏风险。

比如可使用ThreadLocal保存用户登入信息（实际上是利用的一个ThreadLocalMap进行数据的存储，注意最后的回收），这样可以快速获取用户信息。

内存泄漏原因：

在ThreadLocal中是kv保存的，k是弱引用，v是强引用。当内存不足时，弱引用被回收，强引用还在，这时候由于k没了，v就会被永远存在

![ThreadLocal](/img/notes/Blog/1.jpg)

```java
package com.pengyujie.blogapi.utils;

import com.pengyujie.blogapi.dao.pojo.SysUser;

//线程变量隔离
public class UserThreadLocal {//用来保存用户信息，下次直接读取
    private UserThreadLocal(){ }

    private static final ThreadLocal<SysUser> LOCAL = new ThreadLocal<>();//创建一个ThreadLocalMap k就是LOCAL v就是你要存入的值sysUser

    public static void put(SysUser sysUser){//存入
        LOCAL.set(sysUser);
    }

    public static SysUser get(){//获取
        return LOCAL.get();
    }

    public static  void remove(){//删除
        LOCAL.remove();
    }
}
```

然后登入时，使用put存入，使用get取出，最后不需要使用之后进行remove

```java
UserThreadLocal.put(sysUser);//存入UserThreadLocal 方便下次直接取出


SysUser sysUser = UserThreadLocal.get();//取出数据

UserThreadLocal.remove();//删除数据 避免内存泄漏
```



#### 登入拦截器的实现

~~~java
package com.pengyujie.blogapi.handler;

import com.alibaba.fastjson.JSON;
import com.pengyujie.blogapi.dao.pojo.SysUser;
import com.pengyujie.blogapi.service.LoginService;
import com.pengyujie.blogapi.utils.UserThreadLocal;
import com.pengyujie.blogapi.vo.ErrorCode;
import com.pengyujie.blogapi.vo.Result;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Component
@Slf4j
public class LoginInterceptor implements HandlerInterceptor {

    @Autowired
    LoginService loginService;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //在执行controller方法(MVC中 Controller也叫Handler)之前执行
        /*
        * 1.需要进行判断接口路径是否为Controller方法
        * 2.判断token是否为空，未登入
        * 3.如果token不为空，登入验证，loginService checkToken
        * 4.如果验证成功，放行
        * */
        if (!(handler instanceof HandlerMethod)){//判断这个是否为一个HandlerMethod类型
            return true;//不是handler方法类型   放行
        }
        //否则是handler方法类型   拦截

        String token = request.getHeader("Authorization");//获取当前用户token
        //打印日志
        log.info("=================request start===========================");
        String requestURI = request.getRequestURI();
        log.info("request uri:{}",requestURI);
        log.info("request method:{}",request.getMethod());
        log.info("token:{}", token);
        log.info("=================request end===========================");

        if(StringUtils.isBlank(token)){//判断当前用户token是否为空
            Result result = Result.fail(ErrorCode.NO_LOGIN.getCode(), ErrorCode.NO_LOGIN.getMsg());
            response.setContentType("application/json;charset=utf-8");
            response.getWriter().print(JSON.toJSONString(result));//未登入，则返回未登入信息
            return false;//未登入 拦截
        }
        //token不为空
        SysUser sysUser= loginService.checkToken(token);
        if(sysUser==null){
            Result result = Result.fail(ErrorCode.NO_LOGIN.getCode(), ErrorCode.NO_LOGIN.getMsg());
            response.setContentType("application/json;charset=utf-8");
            response.getWriter().print(JSON.toJSONString(result));//未登入，则返回未登入信息
            return false;//未登入 拦截
        }
        //sysUser不为空，登入验证成功，放行
        UserThreadLocal.put(sysUser);//存入UserThreadLocal 方便下次直接取出
        return true;//放入
    }


    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        UserThreadLocal.remove();//如果用完不删除ThreadLocal中的信息，会有内存泄漏的风险
    }
}
~~~



config配置

~~~java
package com.pengyujie.blogapi.config;

import com.pengyujie.blogapi.handler.LoginInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebMVCConfig implements WebMvcConfigurer {

    @Autowired
    LoginInterceptor loginInterceptor;

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        //跨域配置  协议 接口 域名 有任意一项不同视为跨域
        registry.addMapping("/**").allowedOrigins("http://localhost:8080/");
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
//        registry.addInterceptor(loginInterceptor)
//                .addPathPatterns("/");
//        registry.addInterceptor(loginInterceptor)//拦截除了登入注册外的所有接口
//                .addPathPatterns("/**").excludePathPatterns("/login").excludePathPatterns("/register");
        //拦截test接口，后续实际遇到需要拦截的接口时，在配置为真正的拦截接口
        registry.addInterceptor(loginInterceptor)
                .addPathPatterns("/test")
                .addPathPatterns("/comments/create/change")
                .addPathPatterns("/articles/publish");
    }
}
~~~







#### 使用线程池增加阅读数

不使用线程池的情况

ArticleServiceImpl:

```java
 @Autowired
    ThreadService threadService;

    @Override
    public Result findArticleById(Long articleId) {//文章详情
        /*
        * 1.根据id查询文章信息
        * 2.根据bodyId和category去做关联查询
        * */
        Article article =articleMapper.selectById(articleId);//根据id查询出article信息
        ArticleVo articleVo = copy(article, true, true,true,true);
        //查询完文章之后，应该对阅读数进行一个更新
        //如果这时候做更新操作，加写锁，阻塞其他的读操作，性能就会比较低（不可优化）
        //更新增加了接口的耗时，就会导致在进行点开查看文章时候，需要时间变久（可优化）
        //一旦更新出问题，需要保证也不影响文章查看操作
        //利用线程池进行，可以将更新操作，扔到线程池去执行，这样就和主线程不相关了
        threadService.updateArticleViewCount(articleMapper,article);
        return Result.success(articleVo);

    }
```

ThreadService:

```java
package com.pengyujie.blogapi.service;

import com.pengyujie.blogapi.dao.mapper.ArticleMapper;
import com.pengyujie.blogapi.dao.pojo.Article;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

@Component
public class ThreadService {

    //期望此操作在线程池进行，不影响原有的主线程
    @Async("taskExecutor")
    public void updateArticleViewCount(ArticleMapper articleMapper, Article article) {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
}
```

此时执行，发现刷新文章需要等待5s，影响了主线程。





这时候我们去配置线程池，并开启。只需要去添加一个ThreadPoolConfig和修改一下ThreadService( 添加@Async("taskExecutor") )。如下

ThreadPoolConfig

```java
package com.pengyujie.blogapi.config;


import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync//开启多线程
public class ThreadPoolConfig {

    @Bean("taskExecutor")
    public Executor asyncServiceExecutor(){
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        //设置核心线程数
        executor.setCorePoolSize(5);
        //设置最大线程数
        executor.setMaxPoolSize(20);
        //配置队列大小
        executor.setQueueCapacity(Integer.MAX_VALUE);
        //设置线程活跃时间(秒)
        executor.setKeepAliveSeconds(60);
        //设置默认线程名称
        executor.setThreadNamePrefix("更新阅读次数线程");
        //等待所有任务结束后关闭线程池
        executor.setWaitForTasksToCompleteOnShutdown(true);
        //执行初始化
        executor.initialize();
        return executor;

    }
}
```

ThreadService:

```java
package com.pengyujie.blogapi.service;


import com.pengyujie.blogapi.dao.mapper.ArticleMapper;
import com.pengyujie.blogapi.dao.pojo.Article;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

@Component
public class ThreadService {

    //期望此操作在线程池进行，不影响原有的主线程
    @Async("taskExecutor")
    public void updateArticleViewCount(ArticleMapper articleMapper, Article article) {   
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

我们需要进行与主线程不影响的操作时就可以使用这种方法只需要在ThreadService中写入我们的业务，这就是线程池的使用



- 使用线程池增加阅读数，在ThreadService中写入业务。

```java
package com.pengyujie.blogapi.service;


import com.pengyujie.blogapi.dao.mapper.ArticleMapper;
import com.pengyujie.blogapi.dao.pojo.Article;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

@Component
public class ThreadService {

    //期望此操作在线程池进行，不影响原有的主线程
    @Async("taskExecutor")
    public void updateArticleViewCount(ArticleMapper articleMapper, Article article) {
    	int ViewCounts = article.getViewCounts();//得到当前的阅读数
        Article articleUpdate = new Article();
        articleUpdate.setViewCounts(ViewCounts+1);

        LambdaUpdateWrapper<Article> updateWrapper = new LambdaUpdateWrapper<>();//创建一个LambdaUpdateWrapper对象去对数据库进行操作
        updateWrapper.eq(Article::getId,article.getId());//条件数据库中id与article的id相同
        //设置一个 为了在多线程环境下线程安全 （修改之前保证没有被修改过 再+1）
        updateWrapper.eq(Article::getViewCounts,ViewCounts);
        //update article set view_count= ViewCounts+1 where id=article的id and view_count=ViewCounts
        articleMapper.update(articleUpdate,updateWrapper);//执行更新
        
    }
}
```



#### 所有错误替换为系统异常提示

~~~java
package com.pengyujie.blogapi.handler;

import com.pengyujie.blogapi.vo.Result;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;

@ControllerAdvice
public class AllExceptionHandler {

    @ExceptionHandler(Exception.class)
    @ResponseBody
    public Result doException(Exception ex){
        ex.printStackTrace();
        return Result.fail(-999,"系统异常");
    }
}

~~~



#### 自定义注解

统一缓存处理优化

自定义一个缓存注解

```java
package com.pengyujie.blogapi.common.cache;

import java.lang.annotation.*;

@Target(ElementType.METHOD)//使用在方法中
@Retention(RetentionPolicy.RUNTIME)//在运行时有效
@Documented//将我们的注解生成在javadoc中
public @interface Cache {
    long expire() default  1*60*1000;//设置缓存过期时间
    String name() default "";//设置缓存标识
}

```



定义一个切面（切面定义了通知和切点的关系）

```java
package com.pengyujie.blogapi.common.cache;

import com.alibaba.fastjson.JSON;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.pengyujie.blogapi.vo.Result;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.codec.digest.DigestUtils;
import org.apache.commons.lang3.StringUtils;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.time.Duration;
//aop 定义一个切面，切面定义了切点和通知的关系
@Aspect
@Component
@Slf4j
public class CacheAspect {

    private static final ObjectMapper objectMapper = new ObjectMapper();

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @Pointcut("@annotation(com.pengyujie.blogapi.common.cache.Cache)")
    public void pt(){}

    @Around("pt()")
    public Object around(ProceedingJoinPoint pjp){
        try {
            Signature signature = pjp.getSignature();
            //类名
            String className = pjp.getTarget().getClass().getSimpleName();
            //调用的方法名
            String methodName = signature.getName();


            Class[] parameterTypes = new Class[pjp.getArgs().length];
            Object[] args = pjp.getArgs();
            //参数
            String params = "";
            for(int i=0; i<args.length; i++) {
                if(args[i] != null) {
                    params += JSON.toJSONString(args[i]);
                    parameterTypes[i] = args[i].getClass();
                }else {
                    parameterTypes[i] = null;
                }
            }
            if (StringUtils.isNotEmpty(params)) {
                //加密 以防出现key过长以及字符转义获取不到的情况
                params = DigestUtils.md5Hex(params);
            }
            Method method = pjp.getSignature().getDeclaringType().getMethod(methodName, parameterTypes);
            //获取Cache注解
            Cache annotation = method.getAnnotation(Cache.class);
            //缓存过期时间
            long expire = annotation.expire();
            //缓存名称
            String name = annotation.name();
            //先从redis获取
            String redisKey = name + "::" + className+"::"+methodName+"::"+params;
            String redisValue = redisTemplate.opsForValue().get(redisKey);
            if (StringUtils.isNotEmpty(redisValue)){
                log.info("走了缓存~~~,{},{}",className,methodName);
                Result result = JSON.parseObject(redisValue, Result.class);
                return result;
            }
            Object proceed = pjp.proceed();
            redisTemplate.opsForValue().set(redisKey,JSON.toJSONString(proceed), Duration.ofMillis(expire));
            log.info("存入缓存~~~ {},{}",className,methodName);
            return proceed;
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        return Result.fail(-999,"系统错误");
    }

}

```



在需要注解的controller进行注解

```java
@PostMapping("hot")//最热文章
    @Cache(expire = 5 * 60 * 1000,name = "hot_article")//自定义注解
    public Result hotArticle(){
        int limit= 6;
        return articleService.hotArticle(limit);
    }
```





#### SpringSecurity

步骤：

1、pom导入依赖

2、SpringSecurity的配置

~~~java
package com.pengyujie.blogadmin.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.builders.WebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;

@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    public BCryptPasswordEncoder bCryptPasswordEncoder(){//密码策略 加密策略 MD5 不安全 彩虹表 MD5 加盐
        return new BCryptPasswordEncoder();
    }

    public static void main(String[] args) {
        //加密策略 MD5 不安全 彩虹表 MD5 加盐
        String blog= new BCryptPasswordEncoder().encode("blog");//可以通过输入密码 查看加密之后的密码是啥样的
        System.out.println(blog);
    }

    @Override
    public void configure(WebSecurity web) throws Exception {
        super.configure(web);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests() //开启登录认证
//                .antMatchers("/user/findAll").hasRole("admin") //访问接口需要admin的角色
                .antMatchers("/css/**").permitAll()
                .antMatchers("/img/**").permitAll()
                .antMatchers("/js/**").permitAll()
                .antMatchers("/plugins/**").permitAll()
                .antMatchers("/admin/**").access("@authService.auth(request,authentication)") //自定义service 来去实现实时的权限认证
                .antMatchers("/pages/**").authenticated()
                .and().formLogin()
                .loginPage("/login.html") //自定义的登录页面
                .loginProcessingUrl("/login") //登录处理接口
                .usernameParameter("username") //定义登录时的用户名的key 默认为username
                .passwordParameter("password") //定义登录时的密码key，默认是password
                .defaultSuccessUrl("/pages/main.html")
                .failureUrl("/login.html")
                .permitAll() //通过 不拦截，更加前面配的路径决定，这是指和登录表单相关的接口 都通过
                .and().logout() //退出登录配置
                .logoutUrl("/logout") //退出登录接口
                .logoutSuccessUrl("/login.html")
                .permitAll() //退出登录的接口放行
                .and()
                .httpBasic()
                .and()
                .csrf().disable() //csrf关闭 如果自定义登录 需要关闭
                .headers().frameOptions().sameOrigin();
    }
}

~~~



3、权限认证操作

~~~java
package com.pengyujie.blogadmin.service;


import com.pengyujie.blogadmin.pojo.Admin;
import com.pengyujie.blogadmin.pojo.Permission;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Service;

import javax.servlet.http.HttpServletRequest;
import java.util.List;

@Service
public class AuthService {//权限认证

    @Autowired
    AdminService adminService;

    public boolean auth(HttpServletRequest request, Authentication authentication){//返回true 就是通过 返回false就是不通过
        //权限认证
        //请求路径
        String requestURL = request.getRequestURI();
        Object principal = authentication.getPrincipal();
        if(principal==null||"anonymousUser".equals(principal)){
            //未登入
            return false;
        }
        UserDetails userDetails = (UserDetails) principal;
        String username = userDetails.getUsername();
        Admin admin = adminService.findAdminByUsername(username);

        if(admin==null){
            return false;
        }
        if(1==admin.getId()){
            //超级管理员
            return true;
        }
        Long id = admin.getId();
        List<Permission> permissionList = adminService.findPermissionByAdminId(id);
        requestURL = StringUtils.split(requestURL,'?')[0];//取？前面的url 因为后面是请求参数
        for (Permission permission : permissionList) {//遍历权限路径
            if(requestURL.equals(permission.getPath())){//请求路径在权限路径范围返回true 否则返回false
                return true;
            }
        }
        return false;
        return true;
    }
}
~~~



~~~java
package com.pengyujie.blogadmin.service.impl;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.pengyujie.blogadmin.mapper.AdminMapper;
import com.pengyujie.blogadmin.pojo.Admin;
import com.pengyujie.blogadmin.pojo.Permission;
import com.pengyujie.blogadmin.service.AdminService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class AdminServiceImpl implements AdminService {

    @Autowired
    AdminMapper adminMapper;

    public Admin findAdminByUsername(String username){
        LambdaQueryWrapper<Admin> queryWrapper = new LambdaQueryWrapper<>();
        queryWrapper.eq(Admin::getUsername,username);
        queryWrapper.last("limit 1");
        Admin admin = adminMapper.selectOne(queryWrapper);//查询出username相同的那一个admin
        return admin;
    }

    @Override
    public List<Permission> findPermissionByAdminId(Long aminId) {
       return adminMapper.findPermissionByAdminId(aminId);
    }

}
~~~



~~~java
package com.pengyujie.blogadmin.service.impl;

import com.pengyujie.blogadmin.pojo.Admin;
import com.pengyujie.blogadmin.service.AdminService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Component;

import java.util.ArrayList;

@Component
public class SecurityUserByServiceImpl implements UserDetailsService {

    @Autowired
    AdminService adminService;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        //登入的时候，会把username传递到这
        //通过username查询admin表，如果admin存在，将密码告诉spring security
        //如果不存在 返回null 认证失败

        Admin admin = adminService.findAdminByUsername(username);//查出对应的admin
        if (admin==null){
            //登入失败
            return null;
        }
        UserDetails userDetails = new User(username,admin.getPassword(),new ArrayList<>());
        return userDetails;
    }
}

~~~



至此springsecurity的一个简单的登入demo完成。





#### 枚举

> 使用枚举来表示报错信息

定义一个枚举

~~~java
package com.pengyujie.blogapi.vo;

public enum ErrorCode {
    PARAMS_ERROR(10001,"参数有误"),
    ACCOUNT_PWD_NOT_EXIST(10002,"用户名或密码不存在"),
    TOKEN_ERROR(1003,"token为空"),
    NO_PERMISSION(70001,"无访问权限"),
    SESSION_TIME_OUT(90001,"会话超时"),
    NO_LOGIN(90002,"未登录"),
    ACCOUNT_EXIST(1004,"账号已经被注册")
    ;


    private int code;
    private String msg;

    ErrorCode(int code,String msg){
        this.code=code;
        this.msg=msg;
    }

    public int getCode(){
        return code;
    }
    public String getMsg(){
        return msg;
    }
}
~~~



使用枚举

~~~java
@Override
    public Result findUserByToken(String token) {
        //token是否合法校验
        //是否为空，解析是否成功 redis是否存在
        //失败则返回失败
        //成功则返回，对应的结果 LoginUserVo
        if(StringUtils.isBlank(token)){
            return Result.fail(ErrorCode.TOKEN_ERROR.getCode(), ErrorCode.TOKEN_ERROR.getMsg());
        }
        SysUser sysUser = loginService.checkToken(token);//返回用户信息

        LoginUserVo loginUserVo= new LoginUserVo();//创建一个LoginUserVo对象 将查询出来的信息放入该对象
        loginUserVo.setId(String.valueOf(sysUser.getId()));
        loginUserVo.setAccount(sysUser.getAccount());
        loginUserVo.setNickname(sysUser.getNickname());
        loginUserVo.setAvatar(sysUser.getAvatar());

        return Result.success(loginUserVo);//最后返回的应该是一个LoginUserVo
    }
~~~



#### JWT实现登入

JWT：也就是json web token 

被用于登入验证，jwt分为三部分：header 、payload 、signature

header包含使用的签名算法和令牌类型，base64url加密形成**第一部分**。payload则是自定义字段，可以进行存储权限、过期时间等，base64url加密形成**第二部分**。signature是对前二者的防止篡改签名。前二者使用base64url编码之后利用 `.` 进行连接，然后根据header的签名算法进行签名形成**第三部分**。

将这三部分用`.`组合起来就是完整的JWT。

1、编写JWTUtils 

~~~java
package com.pengyujie.blogapi.utils;

import io.jsonwebtoken.Jwt;
import io.jsonwebtoken.JwtBuilder;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;

import java.util.Date;
import java.util.HashMap;
import java.util.Map;

public class JWTUtils {
    private static final String jwtToken = "TokenRandom";//TokenRandom.createToken(32);
    private static final long time = 1000*60*60*24;//表示一天 1000毫秒 *60 *60 *24

    public static String createToken(Long userId){
        Map<String,Object> claims = new HashMap<>();
        claims.put("userId",userId);
        JwtBuilder jwtBuilder = Jwts.builder()
                //header

                //pay_load
                .setClaims(claims)
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis()+time))//有效时间到第二天的当前时间
                //signature
                .signWith(SignatureAlgorithm.HS256,jwtToken)
                ;
        String token = jwtBuilder.compact();
        return token;//这个就是完整的jwt

    }

    public static Map<String,Object> checkToken(String token){
        try{
            Jwt parse = Jwts.parser().setSigningKey(jwtToken).parse(token);
            return (Map<String, Object>) parse.getBody();
        }catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }
}
~~~



2、编写登入Controller 

~~~java
package com.pengyujie.blogapi.controller;


import com.pengyujie.blogapi.service.LoginService;
import com.pengyujie.blogapi.vo.Result;
import com.pengyujie.blogapi.vo.params.LoginParam;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/login")
public class LoginController {

    @Autowired
    LoginService loginService;

    @PostMapping
    public Result login(@RequestBody LoginParam loginParam){
        //登入  验证用户  访问用户表
        return loginService.login(loginParam);
    }
}
~~~



3、编写登入Service

~~~java
package com.pengyujie.blogapi.service.impl;

import com.alibaba.fastjson.JSON;
import com.pengyujie.blogapi.dao.pojo.SysUser;
import com.pengyujie.blogapi.service.LoginService;
import com.pengyujie.blogapi.service.SysUserService;
import com.pengyujie.blogapi.utils.JWTUtils;
import com.pengyujie.blogapi.vo.ErrorCode;
import com.pengyujie.blogapi.vo.Result;
import com.pengyujie.blogapi.vo.params.LoginParam;
import org.apache.commons.codec.digest.DigestUtils;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.Map;
import java.util.concurrent.TimeUnit;

@Service
public class LoginServiceImpl implements LoginService {

    @Autowired
    SysUserService sysUserService;
    @Autowired
    RedisTemplate<String,String> redisTemplate;

    private static final String slat="pengyujie@";

    @Override
    public Result login(LoginParam loginParam) {
        //检查参数是否合法
        //根据用户名和密码去查询 数据库中是否存在
        //不存在登入失败
        //存在使用jwt生成token返回给前端
        //token放入redis redis token：user信息 设置过期时间
        String account = loginParam.getAccount();
        String password = loginParam.getPassword();
        if(StringUtils.isBlank(account) ||StringUtils.isBlank(password)){
            return  Result.fail(ErrorCode.PARAMS_ERROR.getCode(),ErrorCode.PARAMS_ERROR.getMsg());
        }

        password = DigestUtils.md5Hex(password+slat);//密码加slat
        SysUser sysUser = sysUserService.findUser(account,password);
        if(sysUser==null){
            return Result.fail(ErrorCode.ACCOUNT_PWD_NOT_EXIST.getCode(), ErrorCode.ACCOUNT_PWD_NOT_EXIST.getMsg());
        }
        String token = JWTUtils.createToken(sysUser.getId());//创建一个token
        //将token存入redis 设置有效时间
        redisTemplate.opsForValue().set("TOKEN_"+token, JSON.toJSONString(sysUser),1, TimeUnit.DAYS);

        return Result.success(token);
    }
}
~~~





### 其他

#### URL下载网络资源

网络上一切都是流

~~~java
package com.example.demo.kuangstudy;

import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.net.URLConnection;

public class WangYiYun {
    public static void main(String[] args) throws IOException {
        //下载地址
        URL url =new URL("https://images2017.cnblogs.com/blog/1222878/201711/1222878-20171111160819825-27801316.png");//url地址
        //连接到这个资源
        URLConnection urlConnection = url.openConnection();
        InputStream inputStream = urlConnection.getInputStream();
        FileOutputStream fos =new FileOutputStream("WangYiYun.jpg");
        byte[] buffer= new byte[1024];
        int len;
        while((len=inputStream.read(buffer))!=-1){
            fos.write(buffer,0,len);//写出这个数据
        }
        fos.close();
        inputStream.close();
        System.out.println("success");
    }
}
~~~



#### JavaDoc文档

javadoc 工具软件识别以下标签：

| **标签**      |                        **描述**                        |                           **示例**                           |
| :------------ | :----------------------------------------------------: | :----------------------------------------------------------: |
| @author       |                    标识一个类的作者                    |                     @author description                      |
| @deprecated   |                 指名一个过期的类或成员                 |                   @deprecated description                    |
| {@docRoot}    |                指明当前文档根目录的路径                |                        Directory Path                        |
| @exception    |                  标志一个类抛出的异常                  |            @exception exception-name explanation             |
| {@inheritDoc} |                  从直接父类继承的注释                  |      Inherits a comment from the immediate surperclass.      |
| {@link}       |               插入一个到另一个主题的链接               |                      {@link name text}                       |
| {@linkplain}  |  插入一个到另一个主题的链接，但是该链接显示纯文本字体  |          Inserts an in-line link to another topic.           |
| @param        |                   说明一个方法的参数                   |              @param parameter-name explanation               |
| @return       |                     说明返回值类型                     |                     @return explanation                      |
| @see          |               指定一个到另一个主题的链接               |                         @see anchor                          |
| @serial       |                   说明一个序列化属性                   |                     @serial description                      |
| @serialData   | 说明通过writeObject( ) 和 writeExternal( )方法写的数据 |                   @serialData description                    |
| @serialField  |             说明一个ObjectStreamField组件              |              @serialField name type description              |
| @since        |               标记当引入一个特定的变化时               |                        @since release                        |
| @throws       |                 和 @exception标签一样.                 | The @throws tag has the same meaning as the @exception tag.  |
| {@value}      |         显示常量的值，该常量必须是static属性。         | Displays the value of a constant, which must be a static field. |
| @version      |                      指定类的版本                      |                        @version info                         |

~~~java
package com.example.demo.kuangstudy;

/**
 * @author pengyujie
 * @version 1.0
 * @since 1.8
 */
public class JavaDoc {
    String name;

    /**
     *
     * @author pengyujie
     * @param name
     * @return
     * @throws Exception
     */
    public String test(String name) throws Exception{
        return name;
    }
}
~~~

在cmd中输入

~~~cmd
C:\Users\pengyujie\IdeaProjects\MD_five\src\main\java\com\example\demo\kuangstudy\JavaDoc>javadoc -encoding UTF-8 -charset UTF-8 C:\Users\pengyujie\IdeaProjects\MD_five\src\main\java\com\example\demo\kuangstudy\JavaDoc.java
~~~

JavaDoc文档生成完毕

<img src="/img/notes/Blog/2.png">



#### 自定义异常

自定义一个异常（需要继承Exception）

~~~java
package com.example.demo.kuangstudy;

public class MyException extends Exception{
    private int age;//传入异常的参数

    public MyException(int age) {//编写一个构造器 给参数赋值
        this.age = age;
    }

    @Override
    public String toString() {//异常信息显示
        return "MyException{" +
                "年龄异常！年龄必须在1-120之间：age=" + age +
                '}';
    }
}
~~~

编写一个测试

~~~java
package com.example.demo.kuangstudy;

public class Test {
    public void setAge(int age) throws MyException {
        if(age>120||age<1){
            throw new MyException(age);//抛出异常
        }
    }


    public static void main(String[] args) throws MyException {
        Test test =new Test();
        test.setAge(130);
    }

}
~~~

运行效果：

<img src="/img/notes/Blog/3.png">

















### 错误笔记

- 今天做码神之路的blog项目，然后springboot一直报错

> Invalid bound statement (not found): com.pengyujie.blogapi.dao.mapper.TagMapper.findHotsTagIds

搞了几个小时，检查关联代码完全没有问题。发现是创建xml文件时候，前面的包都合并成了一个包使用 . 连接。

解决：

将每个文件夹分开（如下），再次执行 成功。很简单的错误，却搞了几个小时。

<img src="/img/notes/Blog/4.png">





- 报错：Cause: com.mysql.cj.jdbc.exceptions.MysqlDataTruncation: Data truncation: Out of range value for column 'article_id' at row 1

<img src="/img/notes/Blog/5.png">

超出界限   原因一般是二种：一种是值为-1，一种值过大。

我的是第二种，给的值过大，在数据库中将article_id的int类型转换为bigint



- 还有一种因为精度丢失的错误，将int转换为String类型解决成功。



- 不小心将springboot项目变为了普通项目，复原

src变为springboot项目

<img src="/img/notes/Blog/6.png">



pom变为springboot的pom

<img src="/img/notes/Blog/7.jpg">



- 空指针异常

造成原因

1、空数据equals   

~~~java
        String s =null;
        System.out.println(s.equals("nishi1"));
        //System.out.println("nishi".equals(s));//将常量放在前面，就不会出现空指针异常
~~~

2、toString（）方法

如果数据为空，使用toString（）方法就会造成空指针，可以使用String.valueOf





- 注解@Transactional失效

1、由于调用本类方法，比如 一个类的a方法，调用这个类的b方法，属于本类调用，不会生效， **只有当事务方法被当前类以外的代码调用时，才会由Spring生成的代理对象来管理。**
2、注解修饰在非public的方法上。**虽然事务无效，但不会有任何报错，这是我们很容犯错的一点。**



- 最近在项目中用到了JSONObject 等相关代码,在引入mavenjar包的时候,老是编译不出来,报Cannot resolve net.sf.json-lib:json-lib:2.4 的错误.

解决办法:
通常在引用这个包的时候需要指定 jdk的版本:

~~~java
		<dependency>
            <groupId>net.sf.json-lib</groupId>
            <artifactId>json-lib</artifactId>
            <version>2.4</version>
            <classifier>jdk15</classifier>
        </dependency>
~~~





- JSONObject

今天调用接口，使用map、list太过繁琐，最后使用JSONObject。

~~~java
		log.info("================start================");
        //rules
        JSONObject jsonObject_account1 =new JSONObject();//创建一个JSONObject类型来存储对象account1
        jsonObject_account1.put("account_in","123");
        jsonObject_account1.put("allocate_scale",3);//Number类型
        JSONObject jsonObject_account2 =new JSONObject();//创建一个JSONObject类型来存储对象account2
        jsonObject_account2.put("account_in","321");
        jsonObject_account2.put("allocate_scale",5);

        JSONArray jsonArray =new JSONArray();//创建一个jsonArray
        jsonArray.add(jsonObject_account1);
        jsonArray.add(jsonObject_account2);

        //rule_list_json
        JSONObject jsonObject_json_list1 =new JSONObject();//创建一个JSONObject类型来存储对象
        jsonObject_json_list1.put("rule_no","0001");//格式为"000X" String类型
        jsonObject_json_list1.put("rules",jsonArray);//rules字段 jsonArray类型
        JSONObject jsonObject_json_list2 =new JSONObject();
        jsonObject_json_list2.put("rule_no","0002");
        jsonObject_json_list2.put("rules",jsonArray);//rules字段 jsonArray类型

        JSONArray jsonArray2 =new JSONArray();//创建一个jsonArray
        jsonArray2.add(jsonObject_json_list1);//把上方的jsonObject_json_list1存入jsonArray2
        jsonArray2.add(jsonObject_json_list2);//把上方的jsonObject_json_list2存入jsonArray2

        generateRequest.setRule_list_json(jsonArray2.toJSONString());//jsonArray2转为String类型
        log.info("setRule_list_json："+jsonArray2.toJSONString());
        log.info("=================end=================");
~~~















