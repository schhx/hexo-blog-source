---
title: SpringBoot入门教程--参数校验
categories:
  - SpringBoot入门教程
tags:
  - SpringBoot入门教程
date: 2020-04-06 09:20:15
---

在前面的文章中我们实现了用户的增删改查，对前端提供了对应的接口，但是这些接口是非常不安全的，因为我们没有对输入参数进行校验。所以这篇文章我们就来实现参数校验<!-- more -->

## 参数校验

### 新增用户校验

新增用户时我们要求用户名不能为空，年龄在 18 ~ 60 岁之间，首先我们需要对 CreateUserReq 加上以下注解。

```
@Data
@Accessors(chain = true)
public class CreateUserReq {

    @NotBlank(message = "用户名不能为空")
    private String username;

    @NotNull(message = "年龄不能为空")
    @Min(value = 18, message = "年龄不能小于18岁")
    @Max(value = 60, message = "年龄不能大于60岁")
    private Integer age;
}
```

另外我们还需要在接口上加上 ```@Valid``` 注解。

```
    @PostMapping("/create")
    public User createUser(@RequestBody @Valid CreateUserReq req) {
        return userService.create(req.getUsername(), req.getAge());
    }
```

现在我们请求测试下：

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdjt2ii5zqj31ld0u0jy7.jpg)

这时我们发现虽然报错了，但是返回值不太友好，在异常处理那篇文章中说过，对于不同的异常需要分类处理，现在我们就对参数校验抛出的异常进行特殊处理一下。

**新增 RequestException**

我们新增一个 RequestException 来表示请求不符合接口要求而导致的异常。

```
public class RequestException extends BaseException {
    public RequestException(String message) {
        super(message);
    }

    public RequestException(String message, Throwable cause) {
        super(message, cause);
    }

    @Override
    public int getCode() {
        return 10002;
    }
}
```

**在 GlobalExceptionHandler 增加对应的处理**

由于请求不符合接口要求而导致的异常有 ```BindException```、```MethodArgumentNotValidException```、```ConstraintViolationException``` 等等，下面我会对这些异常统一处理下。

```
    @ExceptionHandler(BindException.class)
    public ErrorResp handleBindException(BindException e) {
        List<FieldError> fieldErrors = e.getBindingResult().getFieldErrors();
        String errorMsg = errorFieldToString(fieldErrors);
        return ErrorResp.of(new RequestException(errorMsg));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ErrorResp handleMethodArgumentNotValidException(MethodArgumentNotValidException e) {
        List<FieldError> fieldErrors = e.getBindingResult().getFieldErrors();
        String errorMsg = errorFieldToString(fieldErrors);
        return ErrorResp.of(new RequestException(errorMsg));
    }

    private String errorFieldToString(List<FieldError> fieldErrors) {
        StringJoiner stringJoiner = new StringJoiner(";");
        for (FieldError fieldError : fieldErrors) {
            stringJoiner.add(fieldError.getDefaultMessage());
        }
        return stringJoiner.toString();
    }

    @ExceptionHandler(ConstraintViolationException.class)
    public ErrorResp handleConstraintViolationException(ConstraintViolationException e) {
        Set<ConstraintViolation<?>> set = e.getConstraintViolations();
        StringJoiner stringJoiner = new StringJoiner(";");
        for (ConstraintViolation constraintViolation : set) {
            stringJoiner.add(constraintViolation.getMessage());
        }
        String errorMsg = stringJoiner.toString();
        return ErrorResp.of(new RequestException(errorMsg));
    }

    @ExceptionHandler({
            NoHandlerFoundException.class, MissingPathVariableException.class,
            HttpMessageNotReadableException.class, HttpRequestMethodNotSupportedException.class,
            MissingServletRequestParameterException.class
    })
    public ErrorResp handleRequestException(Exception e) {
        if (e instanceof NoHandlerFoundException) {
            return ErrorResp.of(new RequestException("请求路由不存在", e));
        } else if (e instanceof MissingPathVariableException) {
            return ErrorResp.of(new RequestException("缺少路径参数", e));
        } else if (e instanceof HttpMessageNotReadableException) {
            return ErrorResp.of(new RequestException("请输入正确的参数", e));
        } else if (e instanceof HttpRequestMethodNotSupportedException) {
            return ErrorResp.of(new RequestException("HTTP请求方法错误", e));
        } else if(e instanceof MissingServletRequestParameterException){
            return ErrorResp.of(new RequestException("缺少必填参数", e));
        }
        return ErrorResp.of(new CommonBizException("未知异常", e));
    }
```

现在再次请求下可以发现返回值已经比较合理了。

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdjtvkvxufj31li0t6gpo.jpg)

更新接口处理的方法类似，大家可以自己测试下。


### 删除接口校验

现在我们测试下删除用户接口，如果不传参数 userId 返回值如下所示，这是因为我们刚才已经对 ```MissingServletRequestParameterException``` 做了处理。

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdju09wfgyj31lo0o8424.jpg)

其实对于前端来讲这个提示已经比较明显了，当然我们还可以通过方法级别的校验给出更明确的提示。

开启方法级别校验，首先需要在 Controller 上加上注解 ```@Validated```，然后把接口改造成如下形式：

```
    @PostMapping("/delete")
    public void deleteUser(@NotBlank(message = "userId不能为空") String userId) {
        userService.delete(userId);
    }
```

再次测试：

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdjude9oguj31lk0newhv.jpg)

同样查询方法也可以这样改造。

参数校验的内容其实还有很多，比如分组校验、自定义参数校验等，本文只是简单介绍下参数校验，告诉大家一定要注意参数校验，前端传入的参数都是不可信的。





---

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhu6adriej31hb0hqace.jpg)