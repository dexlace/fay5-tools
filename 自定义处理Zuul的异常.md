# 自定义处理Zuul的异常

当Zuul转发==请求超时==时，系统返回如下响应：

```json
{
    "timestamp": "2019-08-07T07:58:21.938+0000",
    "status": 504,
    "error": "Gateway Timeout",
    "message": "com.netflix.zuul.exception.ZuulException: Hystrix Readed time out"
}
```

当处理转发请求的微服务==模块不可用时==，系统返回：

```json
{
    "timestamp": "2019-08-07T08:01:31.829+0000",
    "status": 500,
    "error": "Internal Server Error",
    "message": "GENERAL"
}
```

在gateway模块中自定义一个过滤器，继承`SendErrorFilter`

```java
@Slf4j
@Component
public class ImoodGatewayErrorFilter extends SendErrorFilter {
    @Override
    public Object run() {
        try {
            ImoodResponse imoodResponse = new ImoodResponse();
            RequestContext ctx = RequestContext.getCurrentContext();
            String serviceId = (String) ctx.get(FilterConstants.SERVICE_ID_KEY);

            /**
             * 获取具体的报错信息
             */
            ExceptionHolder exception = findZuulException(ctx.getThrowable());
            String errorCause = exception.getErrorCause();
            Throwable throwable = exception.getThrowable();
            String message = throwable.getMessage();
            message = StringUtils.isBlank(message) ? errorCause : message;

            
            imoodResponse = resolveExceptionMessage(message, serviceId, imoodResponse);

            HttpServletResponse response = ctx.getResponse();
            ImoodUtil.makeResponse(
                    response, MediaType.APPLICATION_JSON_UTF8_VALUE,
                    HttpServletResponse.SC_INTERNAL_SERVER_ERROR, imoodResponse
            );
            log.error("Zull sendError：{}", imoodResponse.getMessage());
        } catch (Exception ex) {
            log.error("Zuul sendError", ex);
            ReflectionUtils.rethrowRuntimeException(ex);
        }
        return null;
    }

    private ImoodResponse resolveExceptionMessage(String message, String serviceId, ImoodResponse imoodResponse) {
        if (StringUtils.containsIgnoreCase(message, "time out")) {
            return imoodResponse.message("请求" + serviceId + "服务超时");
        }
        if (StringUtils.containsIgnoreCase(message, "forwarding error")) {
            return imoodResponse.message(serviceId + "服务不可用");
        }
        return imoodResponse.message("Zuul请求" + serviceId + "服务异常");
    }
}

```

要让我们自定义的Zuul异常过滤器生效，还需要在配置文件中添加如下配置，让默认的异常过滤器失效：

```yml
zuul:
  SendErrorFilter:
    error:
      disable: true
```