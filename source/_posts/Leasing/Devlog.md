---
title: 开发文档
categories:
  - Leasing
date: 2019-09-025 23:36:15
---

### 总体架构

### schema管理

#### 创建schema

- 数据说明（schema数据说明，注意说明schema引用关系）

客户类图，两个类，客户类和合同类的关系

统一例子e.g.客户的json schema 包含普通类型和定制类型

```json
{
  "name": {"type": "string"},//客户姓名
  "address": {"type": "address"},//客户地址，地址类型address
  "e-mail": {"type": "string"},//客户邮箱
  "occupation": {"type": "occupationType"},//客户职业，定制类型occupationType
  "contract": {
    "lease": {"type": "leaseType"},//欲租物品，定制类型leaseType
  	"ship": {
    	"amount": {"type": "number"},
    	"size": {"type": "size"}
  	}//欲租物品为ship时，可定制字段ship，指定ship的数量及大小
  },//客户名下引用合同类，表示客户签订了这个合同
  "contract": {
		//...  
	}//一个客户可以同时拥有多个签订的合同项
}
```

体现引用关系

- 扩展定义（schema扩展方法，注意说明关联属性定义）

通过json schema definitions定制一种数据类型e.g.电话号码

```json
{
  "type": "string",
  "pattern": "^[0-9]+$",
  "minLength": 11,
  "maxLength": 11
}//定义了一个由正则表达式匹配的长度为11的字符串
```

- 前端界面（schema editor部分）

添加一层层属性，具体的类型定制 => advanced settings

```

```



- 高级组件（advanced setting部分）

定制某种类型的advanced settings e.g. popup

```

```



- 后台接口（schema管理接口）

创建schema的接口

所有的schema放在特定的collection下

版本维护，可以选择维护版本或者不维护版本

#### 删除schema

删除作为一种标记，而不是真的删除掉

接口名称

```java

```

#### 更新schema

接口名称

```

```

版本维护，也可以选择维护版本或者不维护版本

#### 查询schema（后台接口）

包括查某个版本，关联查询

条件复合

接口名称

```

```

### document管理

#### 创建document（从schema创建document）

- 数据说明（document数据说明）

与客户schema和合同schema对应的json document是什么样

- 前端界面（document editor部分）

用户怎么从schema生成document

从schema渲染出form，原生单列form

- 高级组件（advanced component部分）

定制属性

根据属性选择渲染component

- 后台接口（document管理接口，统一接口）

创建document

从某个版本的schema创建

版本维护

接口

```

```



#### 删除document

删除标记

接口

```

```



#### 更新document

版本维护

接口

```

```

接口实现

#### 查询document（后台接口）

包括查某个版本，关联查询

条件复合

接口名称及实现

```

```



### 数据渲染

#### 富文本渲染

braft editor

定义占位符，替换占位符（简单值，表格)

#### chart

#### spread sheet



### 流程管理

每个task都可以关联一个schema 将schema具体内容放入formporperty中

研究task的具体生命周期
