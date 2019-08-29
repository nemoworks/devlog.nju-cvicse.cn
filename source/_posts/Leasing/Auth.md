---

title: 鉴权框架
categories:
  - Leasing
date: 2019-08-29 20:45:04

---
## 鉴权框架说明文档

#### 0. 概述

- 使用spring AOP 和注解添加的方式，提供 `web` 层的请求拦截、资源数据的行过滤和列过滤
- 拦截提供动态资源的配置，支持对于资源类型的指定、资源操作行为的指定
- 数据的行过滤支持自定义过滤策略
- 数据的列过滤支持深度过滤，通过 `json` 体的解析进行框架式的资源字段筛选

#### 1. Auth鉴权模块概述

##### 模块设计图

![](https://i.loli.net/2019/08/29/qN3xWA21erpDkni.jpg)

### 2. Auth鉴权模块使用说明

#### 2.1 认证

​	认证采用传统的 `spring security` 的认证方式，通过实现 `AuthenticationProvider` 的认证方法 `public Authentication authenticate(Authentication authentication)` 来实现自定义方式的登录。具体登录接口位于 `/api/auth/login` 下。

​	如上的认证方式后，服务端将会把 `token` 加入到响应头的 `cookie` 中，此后再次发送 `http` 请求只需要继续携带相同的 `token` 内容，服务端可以进行解析来判定请求者身份信息，从而帮助框架进行下一步的拦截、过滤行为。

#### 2.2 控制层拦截与列过滤

##### 2.2.1 实现AuthModelRealm来定制控制层拦截方式

使用方式参照 `shiro`的依赖注入，可以进行灵活的拦截器逻辑编写。

通过实现 `AuthModelRealm` 接口，具体需要实现的方法如下

```java
public interface AuthModelRealm {

    boolean IsPermitted(ActionType[] actionTypes, Class<? extends BaseEntity> model);

    void onPermissionFailed() throws RuntimeException;

    Object postHandle(Object result);
}
```

- `IsPermitted` 方法接收 `ActionType` 列表和实体类的 `Class` 对象，判定控制层拦截器是否需要拦截
- `onPermissionFailed` 方法用于 `IsPermitted` 返回为 `true` 后的回调函数，这里可以设置异常的抛出、日志编写
- `postHandle` 方法用于 `http` 请求将要返回时的数据过滤，可以进行 `json path` 的列筛选

如果你想直接使用一个现成的实体类，`NormalAuthModelRealm` 将会是一个很好的选择。它已经实现了从 `token` 中获取用户信息并判定是否需要进行请求拦截、异常处理和数据实体的列的深度过滤。

##### 2.2.2 使用AuthModel注解来辅助 AOP 编程

框架中定义了注解 `AuthModel`,用于指示某一个方法所执行的操作类型，以及对应的行为对象类型。

实例如下:

```java
    @GetMapping("/pet")
    @AuthModel(targetModel = Pet.class, actionType = {ActionType.QUERY})
    public List<Pet> getPets() {
        logger.info("Into resource [Pet]");
        return petRepository.findAll();
    }
```

在如上注解下，我们指明了资源类型为 `Pet`, 并且执行的行为是 `ActionType.QUERY`, 即查找 pet 资源。

`targetModel` 所能够接收的实体必须继承自 `BaseEntity` ; 行为类型 `actionType` 可以是 `ActionType` 枚举中的一个或多个。 

#### 2.3 数据访问层资源的行过滤

在用户请求的拦截过后，还需要根据用户或其用户组来进行资源列表的过滤。例如想要获取宠物列表信息，并且要求获取只和用户相关联的宠物列表，那么就需要进行资源的行过滤。

##### 2.3.1 实现ModelHandler来进行定制化资源行过滤

和 `AuthModelRealm` 相类似，使用者可以实现 `ModelHandler` 来进行自定义的行过滤方案。接口定义如下：

```java
public interface ModelHandler {
    Object[] preHandle(ProceedingJoinPoint joinPoint);

    Object postHandle(Object result);
}
```

- `preHandle` 在数据访问之前回调执行，可以进行数据访问条件的修改
- `postHandle`在数据访问结束后回调执行，允许实现者按需进行定制化的行过滤

类似的，有一个 `NormalAuthModelRealm` 已经实现了该接口，提供最简单的用户、组织层面的资源行过滤；如果有其他的业务需要，可以自行进行实现。

##### 2.3.2 使用注解 QueryModel 辅助 AOP 编程

针对数据访问的过滤，框架定义了 `QueryModel` ，指明数据操作行为和操作对象的类型。

- 操作行为默认是 `ActionType.QUERY` , 当前在业务不继续扩展的情况下仅支持查询操作前后的数据行过滤。如果需要进行其他操作类型的定制化方案，可以自行实现
- 操作对象的实体必须继承自 `BaseEntity` 

#### 2.4 实现AuthManager 进行权限信息的缓存和读取

行过滤可以支持如下的 `AuthorizationScope` 类型：

- ALL. 表明可以接收到查询到的所有资源
- USER. 表明可以接收到由当前用户创建的资源
- ORGANIZATION. 表明可以接收到当前用户所属的组织，以及该组织的所有下属组织创建的资源

该过滤策略当前进行全局配置，用 `query-scope.xml` 进行配置。

举例如下：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<root>
    <scope>USER</scope>
</root>
```

该配置文件指明了全局的行过滤策略为 `USER`, 即可实现对于资源的过滤，仅考虑其创建的用户。

`AuthManager` 支持对于配置文件的读取和缓存，能够给出当前请求用户的基本信息和组织部门信息，接口定义如下

```Java
public interface AuthManager {
    AuthorizationScope fetchAuthScope();

    BaseEntity fetchPrinciple();
}
```

- `fetchAuthScope` 进行配置文件的读取，读取后进行缓存
- `fetchPrinciple`支持当前请求用户的基本信息和组织部门信息的获取

