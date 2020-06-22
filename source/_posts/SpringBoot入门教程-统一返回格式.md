---
title: SpringBoot入门教程--统一返回格式
categories:
  - SpringBoot入门教程
tags:
  - SpringBoot入门教程
date: 2020-04-05 09:52:24
---

在上篇文章中我们实现了简单的业务逻辑，但是我们返回给前端的数据并没有一个统一的格式，这样不利于前端处理数据。<!-- more -->一般情况下，返回值应该至少包括以下三个属性：

- ```code``` 表示请求处理的状态。
- ```message``` 表示处理结果的描述，一般情况下是用来返回错误信息。
- ```data``` 处理成功时返回的数据。

## 定义返回格式

首先我们先定义一个 BaseResp，用来统一返回值的格式：

```
@Data
@AllArgsConstructor
public abstract class BaseResp {

    private int code;

    private String message;

    private Object data;
}
```

对于处理正常的情况，我们定义一个具体的实现类 SuccessResp ：

```
public class SuccessResp extends BaseResp {

    private SuccessResp(Object data) {
        super(10000, "success", data);
    }

    public static SuccessResp of(Object data) {
        return new SuccessResp(data);
    }
}
```

对于异常情况的处理，我们放到下篇文章来讲解。

## 修改返回值

### 手动修改

我们可以手动修改每个接口的返回值，比如创建用户接口可以修改为如下：

```
    @PostMapping("/create")
    public SuccessResp createUser(@RequestBody CreateUserReq req) {
        User user = userService.create(req.getUsername(), req.getAge());
        return SuccessResp.of(user);
    }
```


### 全局修改

手动修改的方式比较繁琐，而且不太能体现返回的业务数据，那能不能不修改 Controller 而实现自动转换呢？下面我们就通过自定义 ResponseBodyAdvice 的方式实现全局默认修改：

```
@ControllerAdvice(basePackages = {"org.schhx.tutorial.controller"})
public class MyResponseBodyAdvice implements ResponseBodyAdvice {
    @Override
    public boolean supports(MethodParameter returnType, Class converterType) {
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        Object result = SuccessResp.of(body);
        if (body instanceof String) {
            result = JSON.toJSONString(result);
            response.getHeaders().add("Content-type", "application/json");
        }
        return result;
    }
}
```

## 测试

现在我们就来测试下创建用户这个接口吧。

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdiuvvfaaaj31is0u0dkd.jpg)

可以看出现在返回值已经自动转换成了统一的格式。




---

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhu6adriej31hb0hqace.jpg)