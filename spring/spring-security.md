# SpringSecurity

## 概述

`springsecurity`基于`spring`框架，用于保护java应用程序。它提供了身份验证、授权和防护机制等功能

- 身份验证：验证用户身份
- 授权：确定用户是否有权限访问特定资源
- 安全上下文：存储已认证用户详细信息，应用程序中可以访问

## 使用场景

- 有session模式，通常是前端不分离的项目，使用cookie + session 模式存储以及校验用户权限
- 无session模式，通常是前后端分离项目，使用Jwt形式的Token校验权限

## 原理

`springsecurity`采用责任链的设计模式，有一条很长的过滤器链。通过不同的过滤器处理相应的业务流程

其中重要的过滤器有

- `UsernamePasswordAuthenticationFilter`：认证操作全靠这个过滤器
- `ExceptionTranslationFilter`：异常转换过滤器位于整个springSecurityFilterChain的后方，用来转换整个链路中出现的异常
- `FilterSecurityInterceptor`：获取所配置资源访问的授权信息，根据SecurityContextHolder中存储的用户信息来决定其是否有权限

核心接口

- **Authentication**：身份认证令牌，封装了用户相关信息
- **AuthenticationManager**：认证管理对象，用于对 Authentication 对象进行认证
- **UserDetailsService**：用户信息查询服务，定义了 loadUserByUsername 方法查询用户信息
- **UserDetails**：封装了用户的详细信息，UserDetailsService 查询的用户信息都会封装到这个对象中

## 配置

1. springsecurity5.7之前，通过继承`WebSecurityConfigurerAdapter`来实现配置

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
    }
}

```



2. 通过`@EnableWebSecurity`启用`springsecurity`的web功能

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
}
```

## 用户认证

### 密码加密

数据库存储的密码不应为明文，应该对用户出入的密码进行加密，springsecurity提供了`PasswordEncoder`接口来实现加密

```java
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // BCryptPasswordEncoder 是最常用的实现类
    }
```

可以使用`PasswordEncoder`对密码进行`encode`加密，使用`matches`对明文和密文进行匹配

### 基于数据库的用户认证

1. `AuthenticationManager`是springsecurity的认证过程总入口，可以理解为一个认证调度中心，所有的登录认证请求都会经过它来处理

   ```java
   public interface AuthenticationManager {
       // 认证方法
       Authentication authenticate(Authentication authentication) throws AuthenticationException;
   }
   ```

   主要的认证流程是

   ```sh
   接收一个 Authentication 请求对象 → 找到合适的认证器 → 调用认证逻辑 → 返回一个认证成功的 Authentication 对象（带上用户权限信息等
   ```

2. `UserDetails`是描述用户核心信息的接口

3. `UserDetailsService`是专门用来加载`UserDetails`的核心接口

   ```java
   public interface UserDetailsService {
       // 通过 username 获取用户
       UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
   }
   ```

   框架通过该接口获取数据库或内存中的用户信息，来和当前用户传入的`username`和`password`做比较，来实现认证

   ```java
   // 根据用户名和密码创建认证令牌
   Authentication authToken = new UsernamePasswordAuthenticationToken(username, password);
   // 调用认证处理对象的 authenticate 方法进行认证
   Authentication authenticate = authenticationManager.authenticate(authToken);
   
   
   // 认证令牌中的 isAuthenticated() 返回 false 代表未认证
   // 返回的认证对象 isAuthenticated() 返回 true 代表已认证
   ```

   当 `isAuthenticated()` 返回 `true`时springsecurity中的 `UsernamePasswordAuthenticationFilter`认证环节就会跳过。因此，如果时基于`session`的认证模式下，认证信息会存储在`HttpSession`中；如果是基于`token`的话就需要设置一个`TokenFilter`来设置`SecurityContext`中的已认证的`Authentication`信息

   

### 基于内存的用户认证

springsecurity默认是基于内存存储用户信息的，默认用户名是`user`，默认密码是一串随机字符串。可以在配置文件中之定义初始信息

```properties
spring.security.user.name=admin
spring.security.user.password=123456
```

项目启动后会加载到`InMemoryUserDetailsManager`内存管理器，相当于一个`Map`，然后再由`loadUserByName`方法获取

## 权限校验

1. 使用`@EnableGlobalMethodSecurity(prePostEnabled = true)`开启相关配置

2. 使用`@PreAuthorize`注解进行权限控制

   ```java
   @RestController
   public class TestController {
   
       @GetMapping("test")
       public String test() {
           return "Hello SpringSecurity!";
       }
   
       @DeleteMapping("delete")
       // delete是权限标识
       @PreAuthorize("hasAuthority('delete')")
       public String delete() {
           return "删除成功！";
       }
   }
   ```

3. 在使用`loadUserByUsername`方法查询`UserDetails`时封装权限信息

4. 自定义权限校验方法

   `GrantedAuthority` 是 Spring Security 中用来表示**用户权限**（Authority）的接口

   `SimpleGrantedAuthority`是Spring Security 提供的一个简单实现，通常用它来包装角色字符串

   使用场景

   ```sh
   在自定义 UserDetails 或加载用户权限时，将数据库或配置里的角色/权限转换为 GrantedAuthority 对象。
   授权时，AccessDecisionManager 会根据 GrantedAuthority 来决定是否允许访问某个资源。
   ```

   ```java
   // 定义用于校验权限的类
   @Component("ex")
   public class MyExpressionRoot {
       public boolean hasAuthority(String authority){
           //获取身份令牌
           Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
           //获取权限
           Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
           //循环判断
           for (GrantedAuthority grantedAuthority : authorities) {
               String role = grantedAuthority.getAuthority();
               if (role.equals(authority)) {
                   return true;
               }
           }
   
           return false;
       }
   }
   
   // 在@PreAuthorize注解中使用自定义校验方法
   @DeleteMapping("delete")
   //ex是校验实力在Spring容器中的唯一标识
   @PreAuthorize("@ex.hasAuthority('delete')")
   public String delete() {
       return "删除成功！";
   }
   ```

   