---
title: SpringBoot入门教程--异常处理
categories:
  - SpringBoot入门教程
tags:
  - SpringBoot入门教程
date: 2020-04-05 14:47:40
---

在上篇文章中我们规范了返回格式，当时我们只处理了正常的情况，但是程序总是会可能出现异常的情况。<!-- more -->那么在异常情况下怎么处理返回值呢？

SpringBoot 默认的异常处理一般是不能满足我们的需求的，比如我们在请求创建用户的接口时少传一个参数：

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdivj40f3ij31kh0u0wjx.jpg)

可以看出返回值是不符合我们定义的格式的，所以我们要实现自定义的异常处理。

## 定义异常

首先我们需要定义业务异常基类：

```
public abstract class BaseException extends RuntimeException {

    public BaseException(String message) {
        super(message);
    }

    public BaseException(String message, Throwable cause) {
        super(message, cause);
    }

    public abstract int getCode();
}
```

然后再定义一个通用的业务异常：

```
public class CommonBizException extends BaseException {
    public CommonBizException(String message) {
        super(message);
    }

    public CommonBizException(String message, Throwable cause) {
        super(message, cause);
    }

    @Override
    public int getCode() {
        return 10001;
    }
}
```

## 定义 ErrorResp

```
public class ErrorResp extends BaseResp {

    private ErrorResp(BaseException e) {
        super(e.getCode(), e.getMessage(), e.getCause() == null ? e.getMessage() : e.getCause().getMessage());
    }

    public static ErrorResp of(BaseException e) {
        return new ErrorResp(e);
    }
}
```

## 异常处理

在 SpringBoot 中通过注解 ```@RestControllerAdvice``` 和 ```@ExceptionHandler``` 来实现全局异常处理是比较好的方式。

```
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BaseException.class)
    public ErrorResp handleException(BaseException e) {
        return ErrorResp.of(e);
    }

    @ExceptionHandler(Exception.class)
    public ErrorResp handleException(Exception e) {
        return ErrorResp.of(new CommonBizException("未知异常", e));
    }
}
```

注意上述方式并不能处理所有的异常，所以我们还需要自定义一个 ErrorController。

```
@RestController
public class MyErrorController implements ErrorController {

    @Autowired
    private ErrorAttributes errorAttributes;

    @Override
    public String getErrorPath() {
        return "/error";
    }

    @RequestMapping("/error")
    public ErrorResp error(HttpServletRequest request) {
        RequestAttributes requestAttributes = new ServletRequestAttributes(request);
        Throwable throwable = errorAttributes.getError((WebRequest) requestAttributes);
        return ErrorResp.of(new CommonBizException("未知异常", throwable));
    }
}
```


## 测试

再次使用创建用户的接口测试，这时我们发现返回值已经按照我们定义的格式返回了。

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdjsdmua9cj31qk0t8q85.jpg)

当然目前我们异常处理还比较简陋，对于不认识的异常统一返回 ```未知异常```，实际项目中应该对异常进行区分，给出个性化的提示。



---

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhu6adriej31hb0hqace.jpg)