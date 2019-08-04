---
title: 认证、授权与访问控制
categories:
  - Leasing
date: 2019-07-26 20:45:04
---



### Auth鉴权模块

#### 1. Auth鉴权模块概述

使用 `auth-config.xml` 配置文件进行权限内容的配置，模块负责针对系统资源进行拦截与过滤、动态权限配置的功能实现。

##### Auth鉴权模块设计图

![](https://i.loli.net/2019/08/03/9QqzImbZwY2rojB.png)



##### Auth鉴权模块各个子模块功能

| 模块                   | 职责                                                         |
| ---------------------- | ------------------------------------------------------------ |
| SecurityConfig         | 负责Spring Security的相关配置,基本用户信息配置               |
| AuthorityUtils         | 负责实现基本的当前用户信息获取功能                           |
| ActionType             | 规定了用户可以对资源进行操作的行为, 分为 <br>ADD , UPDATE , DELETE , QUERY |
| AuthrorizationScope    | 规定了过滤器进行基本过滤的范围, 分为<br>USER , ALL , ORGANIZATION , NONE |
| ConfigUtils            | 负责读取配置文件内容                                         |
| AuthCache              | 负责根据读取到的配置文件内容进行进一步的解析<br>获取到配置文件所存储的用户权限信息 |
| RepositoryFilterAspect | 数据访问层过滤切片, 负责对 **资源过滤范围** 内容进行过滤     |
| ControllerIncepAspect  | 控制层拦截与过滤切片 , 负责对  **资源是否可访问拦截，<br>资源属性筛选** 功能实现 |
| ModelHandler           | 数据访问过滤切片的过滤行为接口类，负责声明行为内容<br>通过 策略模式，来针对不同的实体类型赋予不同的行为 |
| ContractModelHandler   | 过滤行为接口实现类，专门用于**Contract** 实体的过滤操作      |
| TemplateModelHandler   | 过滤行为接口实现类，专门用于**Template** 实体的过滤操作      |
| ModelHandlerHelper     | 过滤切片辅助类，负责接口的具体实现类分配                     |
| AuthModel              | 控制层注解，辅助控制层切片进行方法筛选                       |
| QueryModel             | 数据访问层注解，辅助数据访问层切片进行方法筛选               |

### 2. Auth鉴权模块功能描述

#### 2.1 认证

```
借助Spring Security与Spring适配程度高的特点，使用其默认的 Form 表单登录的功能 ， 来快速实现内置用户配置、用户权限配置。不需要专门创建和维护用户表、角色表、权限表等内容。若有此方面的业务需要，可自行之后迭代修改
```

##### 2.1.1 接口概述

 <table>
   <tr>
          <td colspan="3">提供的服务（供接口）</td>
        </tr>
        <tr>
          <th rowspan="3">SecurityConfig.configure(http)</th>
          <td>语法</td>
          <td>protected void configure(HttpSecurity http) throws Exception</td>
        </tr>
        <tr>
          <td>功能</td>
          <td>spring security配置路由权限信息，默认进行表单登录</td>
        </tr>
  <tr>
        <tr>
          <th rowspan="3">SecurityConfig.configure(auth)</th>
          <td>语法</td>
          <td>protected void configure(final AuthenticationManagerBuilder auth) throws Exception</td>
        </tr>
        <tr>
          <td>功能</td>
          <td>配置默认可以进行登录的用户, 其中用户有 A , B , C , D ; 账号密码相同</td>
        </tr>
   </tr>

同时运行前后端工程后，输入 `localhost:8000` 可以见到如下界面，提供用户登录。

- 注意: 此界面为 `Spring Security` 的默认登录页面。如要进行更改，可以通过添加 `AuthProvider` 进行自主认证(具体参见Spring Security 文档)
- 认证成功之后，`Cookie` 会自动添加 `JSESSIONID` 名称， 作为之后用户认定的凭证



#### 2.2 权限配置

采用 `auth-config.xml` 配置文件的形式，来针对用户可以进行 `Actions` 的 `Models` 权限赋予，实现动态化权限配置。

##### 2.2.1 文件结构说明

基本 `xml` 文件结构如下

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<root>
    <UserModel user="A">
        <target model="..."    											# 绑定model名称
                actions="ADD,DELETE,UPDATE,QUERY"		# 可以执行的操作,利用 ',' 隔开
                columns="content"										# 筛选的columns,利用 ',' 隔开 ; 如果使用全部属性，则用 '*' 通配符表示
                packagePath="com.cvicse.leasing.model" # model所在包名,用于后续反射的方便使用

        >
        </target>
    </UserModel>
</root>
```

当前项目的 `leasing-backend/src/main/resources/auth-config.xml` 已经内置了相关配置，具体如下

<table>
  <tr>	
    <tr>
       <th>用户名</th>
         <td>A</td>
   </tr>
  	<tr>
  		<td>
      	model
      </td>
      <td>
      	Contract , 可以执行ADD,DELETE,UPDATE,QUERY 操作，筛选 content 属性
      </td>
  	</tr>
  </tr>
  <tr>	
    <tr>
       <th>用户名</th>
         <td>B</td>
   </tr>
  	<tr>
  		<td>
      	model
      </td>
      <td>
      	Template , 可以执行ADD,DELETE,UPDATE,QUERY 操作，筛选 content 属性
      </td>
  	</tr>
  </tr>
  <tr>	
    <tr>
       <th>用户名</th>
         <td>C</td>
   </tr>
  	<tr>
  		<td>
      	model
      </td>
      <td>
      	Contract , 可以执行ADD,DELETE,UPDATE,QUERY 操作，筛选 content 属性
      </td>
  	</tr>
	<tr>
  		<td>
      	model
      </td>
      <td>
      	Template , 可以执行ADD,DELETE,UPDATE,QUERY 操作，筛选 content 属性
      </td>
  	</tr>
  </tr>
  <tr>	
    <tr>
       <th>用户名</th>
         <td>D</td>
   </tr>
  	<tr>
  		<td>
      	model
      </td>
      <td>
      	Contract , 可以执行ADD,DELETE,UPDATE,QUERY 操作，筛选所有属性
      </td>
  	</tr>
	<tr>
  		<td>
      	model
      </td>
      <td>
      	Template , 可以执行ADD,DELETE,UPDATE,QUERY 操作，筛选所有属性
      </td>
  	</tr>
  </tr>


配置文件如下

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<root>
    <UserModel user="A">
        <target model="Contract"
                actions="ADD,DELETE,UPDATE,QUERY"
                columns="content"
                packagePath="com.cvicse.leasing.model"

        >
        </target>
    </UserModel>
    <UserModel user="B">
        <target model="Template"
                actions="ADD,DELETE,UPDATE,QUERY"
                columns="content"
                packagePath="com.cvicse.leasing.model"
        >
        </target>
    </UserModel>
    <UserModel user="C">
        <target model="Contract"
                actions="ADD,DELETE,UPDATE,QUERY"
                columns="content"
                packagePath="com.cvicse.leasing.model"
        >
        </target>
        <target model="Template"
                actions="ADD,DELETE,UPDATE,QUERY"
                columns="content"
                packagePath="com.cvicse.leasing.model"
        >
        </target>
    </UserModel>
    <UserModel user="D">
        <!--        设置所有column-->
        <target model="Contract"
                actions="ADD,DELETE,UPDATE,QUERY"
                columns="*"
                packagePath="com.cvicse.leasing.model"
        >
        </target>
        <target model="Template"
                actions="ADD,DELETE,UPDATE,QUERY"
                columns="*"
                packagePath="com.cvicse.leasing.model"
        >
        </target>
    </UserModel>
</root>
```



#### 2.3 权限赋予

##### 2.3.1 接口描述

<table>
   <tr>
          <th colspan="3">ConfigUtils</th>
        </tr>
        <tr>
          <th rowspan="3">ConfigUtils.readFromXml(path)</th>
          <td>语法</td>
          <td>public static String readFromXml(String path)</td>
        </tr>
        <tr>
          <td>功能</td>
          <td>根据path路径名称，读取xml文件内容</td>
        </tr>
  <tr>
  </table>

<table>
   <tr>
          <th colspan="3">AuthCache</th>
        </tr>
        <tr>
          <th rowspan="3">AuthCache.load()</th>
          <td>语法</td>
          <td>private void load()</td>
        </tr>
        <tr>
          <td>功能</td>
          <td>类实例化之后便进行load装载xml配置文件，用于了临时存储</td>
        </tr>
  <tr>
    <tr>
         <th rowspan="3">AuthCache.hasUser(name)</th>
          <td>语法</td>
          <td>public UserModel hasUser(String username)</td>
        </tr>
        <tr>
          <td>功能</td>
          <td>判定cache中是否存在某一个用户</td>
        </tr>
  </table>

#### 2.4 资源拦截

```
采用srping AOP 中的 @Around 环绕式编写方式，添加控制层的切片内容，根据AuthCache中的缓存信息，判别是否需要对用户请求进行拦截操作
```

<table>
   <tr>
          <th colspan="3">ControllerIncepAspect</th>
        </tr>
        <tr>
          <th rowspan="3">ControllerIncepAspect.filterAuth(jointPoint,
            authModel)</th>
          <td>语法</td>
          <td>public Object filterAuth(ProceedingJoinPoint joinPoint,
                             AuthModel authModel) throws Throwable</td>
        </tr>
        <tr>
          <td>功能</td>
          <td>筛选所有 @AuthModel 标注的注解</td>
        </tr>
  <tr>
    <tr>
         <th rowspan="3">ControllerIncepAspect.checkModelAuth(authmodel)</th>
          <td>语法</td>
          <td>private boolean checkModelAuth(AuthModel authModel)</td>
        </tr>
        <tr>
          <td>功能</td>
          <td>检验cache中的用户是否拥有该model的使用权限</td>
        </tr>
  </table>

##### 切片Active方式

通过在Controller 的某些方法上添加 @AuthModel注解，即可将该方法纳入控制层注解的作用范围中

示例如下：

```java
...  
		@AuthModel(targetModel = Template.class, actionType = ActionType.QUERY)
    @GetMapping
    public List<Template> getTemplates() {
        logger.info("All Templates requested");
        return templateService.getAllTemplate();
    }
...
```

通过如上注解，告知切片：当前作用资源类型为 `Template`, 相关的行为是 `ActionType.QUERY` ，也就是 执行**查询**操作



其他功能在 **资源过滤 - 属性过滤** 中进行补充 

#### 2.5 资源过滤

```
同时在控制层切片和数据访问层切片进行资源的过滤：
	- 数据访问层切片进行第一轮过滤，实现用户可见资源范围过滤
	- 控制层切片进行第二轮过滤，实现资源的属性过滤
```

##### 2.5.1 可见资源范围过滤

<table>
   <tr>
          <th colspan="3">RepositoryFilterAspect</th>
        </tr>
        <tr>
          <th rowspan="3">RepositoryFilterAspect.filterAuth(jointPoint,
            queryModel)</th>
          <td>语法</td>
          <td>public Object filterAuth(ProceedingJoinPoint joinPoint,
                             AuthModel queryModel) throws Throwable</td>
        </tr>
        <tr>
          <td>功能</td>
          <td>筛选所有 @QueryModel 标注的注解</td>
        </tr>
  <tr>
  </table>

###### 策略模式

考虑到数据访问层涉及的实体Model类型不同，采用 **策略模式** 进行行为的动态绑定，根据实体的类型采用不同的筛选策略 `preHandle(joinPoint, queryModel)`

相关模块间关系如下：

![](https://i.loli.net/2019/08/03/agdeIcWRlm8VBP9.jpg)

其中，`ModelHandler` 作为接口，有两个接口实现类 `ContractModelHandler` 和 `TemplateModelHandler` ；而 `ModelHandlerHelper` 作为 **策略分发、实例化** 的操作，其维护了实体名称和相关 Handler 的映射表，能够快速进行策略的分发工作。

###### Example 数据层访问修改

在资源作用范围过滤的逻辑中，要求仅能够对拥有参数 `Example` 的访问接口进行筛选，即进一步修改 `Example` 中的数据访问Query 内容，从而能够与 **分页** 同时进行，避免将大容量数据导入到内存中再次进行筛选。 

##### 2.5.2 属性过滤

```
根据配置的具体属性规则，筛选指定需要保留的资源属性；使用Java反射的机制，根据Field的名称来进行筛选；而对于List类型的数据暂时使用 Json 的反序列化来进行属性筛选
```

<table>
   <tr>
          <th colspan="3">ControllerIncepAspect</th>
        </tr>
    <tr>
         <th rowspan="3">ControllerIncepAspect.responseFilter(result,model)</th>
          <td>语法</td>
          <td>private Object responseFilter(Object result, TargetModel model) throws IllegalAccessException</td>
        </tr>
        <tr>
          <td>功能</td>
          <td>根据配置的columns规则，对资源属性进行过滤</td>
        </tr>
  </table>

### 3 演示

##### 3.1 拦截

初次开启系统

![](https://i.loli.net/2019/08/03/IqXdjzg5NJsC9Bw.png)

首先登陆用户 `A` ，密码也为 `A` 

由于 `/api/templates` 指向的资源类型为 `Template` ， A 用户仅对 `Contract` 拥有访问权限; 则系统返回 `403` 状态码

![](https://i.loli.net/2019/08/03/utCXevfWLDz1IbS.jpg)

仍旧在 A 用户下，点击 `合同管理` ，正常显示，不会有 `403` 状态码产生

![](https://i.loli.net/2019/08/03/Dlvi8OzhbVjMHEc.jpg)



##### 3.2 过滤

认证：C 用户

调用接口 ： `GET /api/templates`

结果：

1. C 用户可以正常对 `Template` 资源进行增删改查
2. C 用户的资源内容中，只能够查看 `content` 字段；成功去除 `id` 字段

![](https://i.loli.net/2019/08/03/cvWkaZfz46y7Lxe.jpg)

认证： D 用户

调用接口 : `GET /api/templates`

结果：

1. D 用户使用 `*` 通配符，可以正常显示所有字段

![](https://i.loli.net/2019/08/03/6OndmoYHtIcv3EZ.jpg)