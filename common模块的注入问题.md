# common模块的注入问题

在项目`imood-security-frame`中，对于适用于其他模块令牌不正确返回401和用户无权限返回403异常，定义了两个handler如下：

```java
public class ImoodAuthExceptionEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException authException) throws IOException {
        ImoodResponse imoodResponse = new ImoodResponse();

//        response.setContentType(MediaType.APPLICATION_JSON_UTF8_VALUE);
//        // 翻译401认证异常：token无效
//        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
//        response.getOutputStream().write(JSONObject.toJSONString(imoodResponse.message("token无效")).getBytes());
        ImoodUtil.makeResponse(response, MediaType.APPLICATION_JSON_UTF8_VALUE,
                HttpServletResponse.SC_UNAUTHORIZED, imoodResponse.message("token无效"));
    }
}
```

```java
public class ImoodAccessDeniedHandler implements AccessDeniedHandler{


        @Override
        public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException {
            ImoodResponse imoodResponse = new ImoodResponse();
            ImoodUtil.makeResponse(
                    response, MediaType.APPLICATION_JSON_UTF8_VALUE,
                    HttpServletResponse.SC_FORBIDDEN, imoodResponse.message("没有权限访问该资源"));
        }

}
```

我们的做法是使用一个config类

```java
public class ImoodAuthExceptionConfig {
    @Bean
    @ConditionalOnMissingBean(name = "accessDeniedHandler")
    public ImoodAccessDeniedHandler accessDeniedHandler() {
        return new ImoodAccessDeniedHandler();
    }

    @Bean
    @ConditionalOnMissingBean(name = "authenticationEntryPoint")
    public ImoodAuthExceptionEntryPoint authenticationEntryPoint() {
        return new ImoodAuthExceptionEntryPoint();
    }
}

```

并使用一个自定义注解，在自定义注解中使用@Import引入config类

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(ImoodAuthExceptionConfig.class)
public @interface EnableAuthExceptionHandler {
}

```

如此，在其他模块的入口类中使用以上的`@EnableAuthExceptionHandler`注解即可注入以上的两个认证异常处理器

可以去参考一下`@Import`的使用和相应源码

另外，请详阅同目录下的`深入学习Spring Boot自动装配 _ MrBird`一文