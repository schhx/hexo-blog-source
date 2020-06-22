---
title: SpringBoot入门教程--实现简单业务
categories:
  - SpringBoot入门教程
tags:
  - SpringBoot入门教程
date: 2020-04-04 15:56:18
---

本篇文章我们通过一个简单的业务来学习如何定义接口和访问数据库。<!-- more -->业务非常简单，对用户增删改查，用户信息也非常简单，只有用户名和年龄两个属性。

## 实现用户增删改查

### 配置数据库

首先我们需要创建一个数据库表来存储用户信息，建表语句如下：

```
CREATE SCHEMA IF NOT EXISTS `spring_boot_simple_tutorial` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

CREATE TABLE IF NOT EXISTS `spring_boot_simple_tutorial`.`user` (
  `id` VARCHAR(37) NOT NULL COMMENT 'UUID',
  `username` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL DEFAULT '' COMMENT '用户名',
  `age` int NOT NULL COMMENT '年龄',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='用户表';
```

在 ```application.yml``` 中配置数据库信息：

```
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/spring_boot_simple_tutorial?useUnicode=true&characterEncoding=utf8&allowMultiQueries=true&autoReconnect=true&useSSL=true
    username: root
    password: rootroot
    driver-class-name: com.mysql.cj.jdbc.Driver

```

在上篇文章中，我们在 ```pom.xml``` 注释掉了 ```spring-boot-starter-jdbc``` 的引用，现在可以把引用还原回来了。

### 定义实体对象

```
@Data
@Accessors(chain = true)
public class User {

    private String id;

    private String username;

    private Integer age;
}
```

### 定义 UserService 和实现类 UserServiceImpl

这里使用 JdbcTemplate 来访问数据库，因为业务比较简单，没必要使用较为复杂的 ORM 框架，JdbcTemplate 能够满足我们的需求。

```
public interface UserService {

    /**
     * 新增用户
     * @param username
     * @param age
     * @return
     */
    User create(String username, int age);

    /**
     * 删除用户
     * @param id
     */
    void delete(String id);

    /**
     * 修改用户
     * @param id
     * @param username
     * @param age
     * @return
     */
    User update(String id, String username, int age);

    /**
     * 查询用户
     * @param id
     * @return
     */
    User getById(String id);
}

@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public User create(String username, int age) {
        String id = UUID.randomUUID().toString();
        jdbcTemplate.update("insert into user ( id, username, age ) values ( ?, ?, ? ) ", id, username, age);
        return getById(id);
    }

    @Override
    public void delete(String id) {
        jdbcTemplate.update("delete from user where id = ?", id);
    }

    @Override
    public User update(String id, String username, int age) {
        jdbcTemplate.update("update user set username = ?, age = ? where id = ?", username, age, id);
        return getById(id);
    }

    /**
     * 获取一个对象时可能会报错，对象数为0是报 EmptyResultDataAccessException，对象数多于1个时报 IncorrectResultSizeDataAccessException
     *
     * @param id
     * @return
     */
    @Override
    public User getById(String id) {
        try {
            return jdbcTemplate.queryForObject("select * from user where id = ?", new Object[]{id}, new BeanPropertyRowMapper<>(User.class));
        } catch (EmptyResultDataAccessException e) {
            return null;
        }
    }
}
```

### 定义 UserController

对于接口的形式，有人喜欢 RESTful 风格，有人喜欢传统的方式，其实没有好坏之分，适合就好。这里我还是使用了比较传统的接口方式。

```
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private UserService userService;

    @PostMapping("/create")
    public User createUser(@RequestBody CreateUserReq req) {
        return userService.create(req.getUsername(), req.getAge());
    }

    @PostMapping("/delete")
    public void deleteUser(@RequestParam String userId) {
        userService.delete(userId);
    }

    @PostMapping("/update")
    public User updateUser(@RequestBody UpdateUserReq req) {
        return userService.update(req.getId(), req.getUsername(), req.getAge());
    }

    @GetMapping("/get")
    public User getUser(@RequestParam String userId) {
        return userService.getById(userId);
    }

}
```

## 测试

我们现在来测试下新增一个用户：

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdilv25mjcj31d20s2422.jpg)

调用成功后，可以看到数据库新增了一条记录：

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdilvm80dlj313k09igqp.jpg)

更新、查询、删除用户这里我就不演示了，大家可以自己去测试下。

---

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhu6adriej31hb0hqace.jpg)

