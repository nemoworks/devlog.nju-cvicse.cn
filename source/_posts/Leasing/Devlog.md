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

Json Schema定义了一套词汇和规则，用来定义Json元数据。这些元数据定义给出了Json数据需要满足的各项规范（成员、结构、类型等）。

以一个合同的schema框架为例，可能的设计如下：

```json
{
  "customer_id": {"type": "string"},//客户编号，是schema中对客户数据的引用方式，表示该客户签订了本份合同
  "lease": {"type": "leaseType"},//待租物品，定制类型leaseType
  "ship": {
   	"amount": {"type": "number"},
   	"size": {"type": "size"}
  },//待租物品为ship时，可定制字段ship，指定ship的数量及大小
  "startDate": {"type": "date"},
  "endDate": {"type": "date"}//租赁起止日期，类型为date
  //...可按需求添加其余字段及类型
}
```

- 扩展定义（schema扩展方法，注意说明关联属性定义）

通过Json Schema Definitions定制若干种数据类型

以上述合同的schema为例，根据客户所需待租物品的不同，可以设计自定义类型leaseType，指定待租物品的种类、数量、大小等属性

也可对应租赁起止日期设计类型：date，指定年、月、日（实际实现时可选用正则匹配字符串或设置数值大小等各种方式，此处为示范用法组合了两种方式）

之后用properties引用需要用到的各项属性

```jsx
{
  "definitions": {
    //...
    "leaseType": {
      "type": "object",
      "properties": {
        "kind": {"type": {"enum": ["ship", "truck", "van"]}}
        "amount": {"type": "number"},
        "size": {"type": {"enum": ["large", "medium", "little"]}}
      },
      "required": ["kind", "amount", "size"]
    },
    "date": {
      "type": "object",
      "properties": {
        "year": {
          "type": "string",
          "pattern": "^[0-9]{4}$"
        },
        "month": {
          "type": "number",
          "minimum": "1",
          "maximum": "12"
        },
        "day": {
          "type": "number",
          "minimun": "1",
          "maximum": "31"
        }
      },
      "required": ["year", "month", "day"]
    }
  }，
  
  "type": "object",
  
  "properties": {
    //...
    "lease": {"$ref": "#/definitions/leaseType"},
    "startDate": {"$ref": "#/definitions/date"},
    "endDate": {"$ref": "#/definitions/date"}
  }
}
```

- 前端界面（schema editor部分）

（这段表述还需要修改）

在确定template之后，可以通过设定类型字段来配置当前用户所需schema的各项内容

如下使用metaSchema将leaseType和tel字段加入进可用字段列表

```jsx
/* schemaEditor抽屉 */
    const schemaEditorDrawer = (
      <Drawer title={formatMessage({ id: 'templateEditor.schemaEditor' })} placement="right" closable={false} onClose={_ => this.setState({ schemaVisible: false })} visible={this.state.schemaVisible} width="50%">
        <SchemaEditor
          data={JSON.stringify(template.content.schema)}
          onChange={schema => this.updateSchemaHandler(template, schema)}
          metaSchema={['string', 'number', 'array', 'object', 'boolean', 'integer', 'tel', 'leaseType']}
        />
      </Drawer>
    )

```



- 高级组件（advanced setting部分）

定制某种类型的advanced settings，然后在其与先前schema中创建的对应字段之间建立映射

如下是对String类型的各项操作的定制：

```jsx
class CustomizedSchemaString extends PureComponent {
  constructor(props, context) {
    //...
  }

  componentWillReceiveProps(nextprops) {
    //...
  }

  //定义各项操作对应的方法
  changeOtherValue = (value, name, data) => {
    //...
  };

  changeEnumOtherValue = (value, data) => {
    //...
  };

  changeEnumDescOtherValue = (value, data) => {
    //...
  };

  onChangeCheckBox = (checked, data) => {
    //...
  };

  render() {//在渲染时将各项操作链接至对应的方法
    const { data } = this.props;
    return (
        <div>
      	//...
          <Col span={20}>
            <Input
              value={data.default}
              placeholder={LocalProvider('default')}
              onChange={e => this.changeOtherValue(e.target.value, 'default', data)}
              />
          </Col>
	</div>
	//...
    );
  }
}

//通过一个mapping使得定义与之前设置的schema类型字段相匹配
const mapping = data => ({
  string: <CustomizedSchemaString data={data} />,
  //...
  // lease: <CustomizedSchemaLease data={data} />,
  // party: <CustomizedSchemaParty data={data} />,
}[data.type]);
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

#### chart

#### spread sheet



### 流程管理

每个task都可以关联一个schema 将schema具体内容放入formporperty中

研究task的具体生命周期
