---
title: 开发文档
categories:
  - Leasing
date: 2019-09-025 23:36:15
---

### 总体架构

本项目旨在实现一个可在线处理租赁业务的原型系统，其特点在于合同的在线编辑，业务中涉及所有数据对象的高度可配置以及配置过程的零编码。为了实现这个特点，我们设计了不同于传统Web项目的开发模式。

{% qnimg devpattern.png %}

传统的开发模式通过代码实现数据对象的类定义并基于关系型数据库进行管理，在这种模式下，数据对象在运行时是无法修改的，因此难以实现数据对象的高度可配置。本开发模式下，通过模式管理工具来实现数据对象的定义，通过一个模式来描述一个数据对象的格式，并通过一个非结构化的文档来存储数据对象。

模式是对一个数据对象的定义。它可以类比为传统模式下的java代码类定义，描述了一类数据对象应该具有的数据和数据类型定义以及该数据对象与其他数据对象的关联关系。但与java代码的类定义不同，模式本身也是运行时可管理的数据，而不是运行时无法更改的代码或配置文件。由此，用户可以在运行时完全通过前端对数据对象进行定义，而无需通过编码的方式来定义一类数据对象，从而实现数据对象的高度可配置和配置过程的零编码。在本项目中，我们将数据对象以JSON对象的形式存储，并采用JSON-schema作为数据对象的模式，定义一类数据对象。

由于模式本身是可以在运行时进行管理的数据，因此它可以通过前端进行创建，修改，删除和查询。前端可以通过接入模式管理工具来实现模式的可视化定义。在本项目中，我们用JSON-schema作为数据对象的模式，并用React作为前端基础框架，因此我们采用React架构的JSON-schema-editor组件作为模式管理工具。



### schema管理

#### 创建schema

- 数据说明（schema数据说明，注意说明schema引用关系）

Json Schema定义了一套词汇和规则，用来定义Json元数据。这些元数据定义给出了Json数据需要满足的各项规范（成员、结构、类型等）。

以一个合同的schema框架为例，可能的设计如下：

``` json
{
  "customer_id": {"type": "string"},//客户编号，是schema中对客户数据的引用方式，表示该客户签订了本份合同
  "lease": {"type": "leaseType"},//待租物品，定制类型leaseType
  "ship": {
    "type": "object",
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
import schemaEditor from '@/components/JsonSchema/index.js';
/* schema editor的配置属性 */
const config = {
  /* 自定义类型的初始值；必填，否则自定义类型会报错； */
  defaultSchema: {
    tel: {
      type: 'rate',
    },
    leaseType: {
      type: 'lease',
    }
  }
};
const SchemaEditor = schemaEditor(config);
render(
  /* 使用metaSchema将leaseType和tel字段加入进可用字段列表 */
  <SchemaEditor data={schema} onChange
    metaSchema={['string', 'number', 'array', 'object', 'boolean', 'integer', 'tel', 'leaseType']} />,
	document.getElementById('root')
)
```



- 高级组件（advanced setting部分）

定制某种类型的advanced settings，然后在其与先前schema中创建的对应字段之间建立映射

如下是对Object类型的各项操作的定制：

```jsx
/* @/components/JsonSchema/components/SchemaComponents/SchemaOther.js */
class CustomizedSchemaObject extends PureComponent {
	/* 将属性值写回schema data中 */
  changeOtherValue = (value, name, data) => {
    data[name] = value;
    this.context.changeCustomValue(data);
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

//通过mapping使得定义与之前设置的schema类型字段相匹配
const mapping = data => ({
  object: <CustomizedSchemaObject data={data} />,
  //...
}[data.type]);

/* advanced settings中的内容封装为CustomItem，定制所需要关注的内容是optionForm */
const CustomItem = (props, context) => {
  const { data } = props;
  const optionForm = mapping(JSON.parse(data));

  return (
    <div>
      <div>{optionForm}</div>
      <div className="default-setting">{LocalProvider('all_setting')}</div>
      <AceEditor
        data={data}
        mode="json"
        onChange={e => handleInputEditor(e, context.changeCustomValue)}
      />
    </div>
  );
};
```



- 后台接口（schema管理接口）

创建schema的接口

在实际运用时，若要创建一个合同项，先通过创建模板template封装schema，再选择对应的template创建所需要的合同，对应接口为：

```java
Template createTemplate(TemplateRequest templateRequest);
```

所有的schema放在特定的collection下（后端TODO）

版本维护实现基于JaVers，JaVers是一个轻量级，完全开源的Java库，用于追踪和记录数据中的更改。它可以配合repository使用，只需要添加一行注解`@JaversSpringDataAuditable`即可

```java
@JaversSpringDataAuditable
public interface TemplateRepository extends MongoRepository<Template, String> {
	//...
}
```

当数据库中有数据发生修改时，JaVers会自动追踪和记录数据库中发生的修改，存为特定数据类型Snapshot，在获取时可以通过版本号来访问到某次修改过后的数据内容

在创建schema时，JaVers会对应生成初始版本记录

#### 删除schema

删除作为一种标记，而不是真的从数据库中将schema删除

若直接将schema删除，在之后就无法从对应生成的合同中查看该schema

实现方式为在schema中添加一个是否删除字段delete，值为true时视为删除（TODO）

接口名称

```java
 public void deleteTemplate(String id) throws TemplateNotFoundException;
```

#### 更新schema

接口名称

```java
public Template updateTemplate(TemplateRequest templateRequest, String id) throws TemplateNotFoundException;
```

此时的更新操作会被JaVers识别，记录到版本历史中

#### 查询schema（后台接口）

根据模版号和提交编号查询对应schema：

```java
public Template getTemplateWithJaversCommitId(String templateId, String commitId) throws TemplateNotFoundException{
        Template template = this.getTemplate(templateId);
        JqlQuery jqlQuery= QueryBuilder.byInstance(template).build();
        List<CdoSnapshot> snapshots = javers.findSnapshots(jqlQuery);
        for(CdoSnapshot snapshot:snapshots){
            if(snapshot.getCommitId().getMajorId()== Integer.parseInt(commitId))
                return JSON.parseObject(javers.getJsonConverter().toJson(snapshot.getState()),Template.class);
        }
        return null;
    }
```

条件复合（同时选取多个属性对于当前schema进行查找操作）：

关联查询（利用schema间的关联进行查找，例:从合同中查询该合同对应的客户信息）:

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

项目中使用[Braft Editor](https://braft.margox.cn/)来实现合同文档的在线编辑。Braft Editor是基于[draft-js](https://draftjs.org/)开发的富文本编辑器，支持value和onChange属性，内部使用EditorState对象作为数据格式，可以用典型的受控组件的形式来使用：

```javascript
import React from 'react'
// 引入编辑器组件
import BraftEditor from 'braft-editor'
// 引入编辑器样式
import 'braft-editor/dist/index.css'

export default class EditorDemo extends React.Component {

    state = {
        // 创建一个空的editorState作为初始值
      	// createEditorState的参数可以是HTML，也可以是JSON
        editorState: BraftEditor.createEditorState(null)
    }

    submitContent = () => {
        // 在编辑器获得焦点时按下ctrl+s会执行此方法
        // 编辑器内容提交到服务端之前，可直接调用editorState.toHTML()来获取HTML格式的内容
        const htmlContent = this.state.editorState.toHTML()
        const result = await saveEditorContent(htmlContent)
    }

    handleEditorChange = (editorState) => {
        this.setState({ editorState })
    }

    render () {
        const { editorState } = this.state
        return (
                <BraftEditor
                    value={editorState}
                    onChange={this.handleEditorChange}
                    onSave={this.submitContent}
                />
        )
    }
}
```

EditorState对象无法用于展示也无法用于持久化存储，需要进行如下的数据转换。

```javascript
const rawString = editorState.toRAW()
// editorState.toRAW()方法接收一个布尔值参数，用于决定是否返回RAW JSON对象，默认是false
const rawJSON = editorState.toRAW(true)
// 将editorState数据转换成html字符串
const htmlString = editorState.toHTML()
```

占位符定义

合同中使用占位符来将合同内容和schema表单联系起来，当表单项填入了具体数值，合同中的占位符会被对应的数字替换。

- 简单占位符

json schema editor中原生的string、number、boolean、integer类型对应的占位符格式定义为`${/* label */}`。具体例子如下，

```Javascript
// JSON schema
{
  "sample": {
    "type": "string"
  }
}
// 占位符
${sample}
```

- 表格占位符

由于array类型需要转换为表格，所以它有特殊的占位符格式，定义为`$[/* label */]`，此外，表格占位符需独占一行。具体例子如下，

```javascript
// JSON schema
{
  "sample": {
    "type": "array",
    "item": {
      "type": "string"
  	}
  }
}
// 占位符
$[sample]
```

占位符替换

先用json path对Editor State的所有block的文本内容进行查找，通过正则表达式匹配出符合定义的占位符，再根据不同的占位符类型进行替换。

```javascript
export function getEditorState({ editorContent, formData }) {

  /* 深拷贝editorContent */
  let copyContent = JSON.parse(JSON.stringify(editorContent))

  /* 占位符正则表达式 */
  const fieldRE = /(?:\$\{)(\w+)(?:\})/g
  const arrayRE = /(?:\$\[)(\w+)(?:\])/

  /* 查找并替换copyContent中的目标字段，替换规则见formDataParser */
  jp.apply(copyContent, '$..blocks[?(@.text)].text', text => text.replace(fieldRE, (_, field) => formDataParser(formData, field)))

  let preState = BraftEditor.createEditorState(copyContent)
  
  jp.apply(copyContent, '$..blocks[?(@.text)]', block => {
    const match = block.text.match(arrayRE)

    if (match && formData[match[1]]) {
      let data = formData[match[1]]
      preState = replacePlaceholder(preState, block.key, data.map(row => Object.keys(row).map(col => row[col])))
    }

    return block
  })

  return preState
}
```

- 替换简单占位符

直接将占位符替换成对应的数值；如果数值不合法，会对错误进行标识。

```javascript
const formDataParser = (formData, field) => {
  let data = formData[field]
  return data && typeof (data) !== 'object' ? formData[field] : `「${field} error」`
}
```

- 替换表格占位符

通过正则表达式找到包含表格占位符的block，再按参数插入对应大小的空表格，最后将数值按顺序填入表格中。

```javascript
// 将数值填入block
function fillBlock(editorState, raw) { 
  let newContentState;
  try {
    const text = String(raw)
    if (text === "undefined" || text === "null") {
      throw "error"
    }
    newContentState = Modifier.replaceText(editorState.getCurrentContent(), editorState.getSelection(), text)
  }
  catch (e) {
    newContentState = editorState.getCurrentContent()
  }
  return ContentUtils.createEditorState(newContentState)
}

// 替换表格占位符
function replacePlaceholder(editorState, blockKey, data) {
  let blockState;
	// 表格行列数
  const row = data.length
  const col = data[0].length
	// 找到包含表格占位符的block
  try { blockState = editorState.getCurrentContent().getBlockForKey(blockKey); }
  catch (e) {
    console.log('no such block')
    return null;
  }
  const block = blockState;
  // 把光标定位到该block
  let newEditorState = ContentUtils.selectBlock(editorState, block);
  // 在该block之后插入空表格
  newEditorState = TableUtils.insertTable(newEditorState, col, row);
  // 插入表格后光标定位到表格第一格
  let selectionBlock = ContentUtils.getSelectionBlock(newEditorState)
  while (selectionBlock.getType() !== 'table-cell') {
    console.log('next')
    newEditorState = ContentUtils.selectNextBlock(newEditorState, selectionBlock)
    selectionBlock = ContentUtils.getSelectionBlock(newEditorState)
  }
	// 填入数值
  for (let i = 0; i < row; i += 1) {
    for (let j = 0; j < col; j += 1) {
      newEditorState = fillBlock(newEditorState, data[i][j])
      newEditorState = ContentUtils.selectNextBlock(newEditorState, selectionBlock)
      selectionBlock = ContentUtils.getSelectionBlock(newEditorState)
    }
  }
  // 删除占位符block
  newEditorState = ContentUtils.removeBlock(newEditorState, block)
  return newEditorState;
}
```

#### chart

#### spread sheet



### 流程管理

每个task都可以关联一个schema 将schema具体内容放入formporperty中

研究task的具体生命周期
