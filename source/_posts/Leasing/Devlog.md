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
  "title": "Contract_Schema_1",
  "description": "A Sample of Contract Schema",
  "type": "object",
  
  "properties": {
    "customer_id": {"type": "string"},//客户编号，是schema中对客户数据的引用方式，表示该客户签订了本份合同
    "leases": {
      "type": "array",
      "items": {
        "type": "leaseType"
      }
    },//待租物品，定制类型leaseType
    "startDate": {"type": "date"},
    "endDate": {"type": "date"}//租赁起止日期，类型为date
    //...可按需求添加其余字段及类型
  }
}
```

以一个客户的schema为例，可能的设计如下：

```json
{
  "title": "Customer_Schema_1",
  "description": "A Sample of Customer Schema",
  "type": "object",
  
  "properties": {
    "id": {"type": "string"},
    "name": {"type": "string"},
    //...
  }
}
```

二者之间的关系可由如下类图表示：

{% qnimg customer_contract.png %}

- 扩展定义（schema扩展方法，注意说明关联属性定义）

通过Json Schema Definitions定制若干种数据类型

以上述合同的schema为例，根据客户所需待租物品的不同，可以设计自定义类型leaseType，指定待租物品的种类、数量、大小等属性

也可对应租赁起止日期设计类型：date，通过设置format来使其符合日期定义

```json
{
  "definitions": {
    //...
    "leaseType": {
      "type": "object",
      "properties": {
        "kind": { "type": { "enum": ["ship", "truck"]}},
        "amount": {"type": "number"},
        "size": { "type": { "enum": ["little", "medium", "large"]}}
      },
      "required": ["kind", "amount", "size"]
    },
    "date": {
      "type": "string",
      "format": "date"
    }
  }
}
```

对应schema可以引用需要用到的各项属性，对应可修改上述contract schema为：

```json
{
  "title": "Contract_Schema_1",
  "description": "A Sample of Contract Schema",
  "type": "object",
  
  "properties": {
    //...
    "leases": {
      "type": "array",
      "items": {"$ref": "#/definitions/leaseType"}
    },
    "startDate": {"$ref": "#/definitions/date"},
    "endDate": {"$ref": "#/definitions/date"}
  }
}
```

- 前端界面（schema editor部分）

用户可以通过schema editor来构建合同的schema，这个schema会经由template封装成为用户建立表单项时的模板

``` jsx
import React from 'react';
/* 引入schema编辑器 */
import schemaEditor from '@/components/JsonSchema/index.js';
/* 引入自定义类型 */
import { definitions } from './definitions.json';
/* schema editor的配置属性 */
const config = { defaultSchema: definitions };
const SchemaEditor = schemaEditor(config);

class TemplateEditor extends React.Component {
  //...
	render() {
    return (
      //...
      <SchemaEditor />
			//...
    )
  }
}
```

以先前定义的contract为schema模板可生成对应的表单form，如下例：

{% qnimg schema_form.png %}

- 高级组件（advanced setting部分）

定制某种类型的advanced settings，然后在其与先前schema中创建的对应字段之间建立映射

如下是对Object类型的各项操作的定制：

```jsx
import React from 'react';
/* 引入schema编辑器 */
import schemaEditor from '@/components/JsonSchema/index.js';
/* 引入自定义类型 */
import { definitions } from './definitions.json';
/* schema editor的配置属性 */
const config = { defaultSchema: definitions };
const SchemaEditor = schemaEditor(config);

class TemplateEditor extends React.Component {
  //...
  advancedTemplate = data => {
    switch(data.type) {
      case "leaseType": return <SchemaLease data={data} />;
      case "object": return <Input placeholder="object"></Input>;
      default: return <Button>null</Button>;
    }
  
	render() {
    return (
      //...
      <SchemaEditor advancedTemplate={this.advancedTemplate}/>
			//...
    )
  }
}
```

- 后台接口（schema管理接口）

创建schema的接口（后期会修改，template改回schema）

在实际运用时，若要创建一个合同项，先通过创建模板template封装schema，再选择对应的template创建所需要的合同，对应接口为：

```java
public Template createTemplate(TemplateRequest templateRequest){
  logger.info("TemplateRequest saved");
  logger.info(templateRequest.getContent().toJSONString());
  Template template =new Template(templateRequest.getContent());
  Template template1 = templateRepository.save(template);
  System.out.println("================"+template1.getId());
  return template1;
}
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

实现方式为在schema中添加一个是否删除字段delete，值为true时视为删除（TODO：添加对字段的操作）

接口名称及实现

```java
public void deleteTemplate(String id) throws TemplateNotFoundException{
  if(!this.templateRepository.findById(id).isPresent())
    throw new TemplateNotFoundException("TemplateRequest Not Found in templateRepository.");
  logger.info("TemplateRequest deleted");
  this.templateRepository.deleteById(id);
}
```

#### 更新schema

接口名称及实现

```java
public Template updateTemplate(TemplateRequest templateRequest, String id) throws TemplateNotFoundException{
  this.templateRepository.findById(id).ifPresent(template -> {
    template.content = templateRequest.getContent();
    this.templateRepository.save(template);
  });
  if(this.templateRepository.findById(id).isPresent()){
    return this.templateRepository.findById(id).get();
  }else {
    throw new TemplateNotFoundException("TemplateRequest Not Found in templateRepository.");
  }
}
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

条件复合：（TODO）

关联查询（利用属性关联性进行查找，例:从合同中查询该合同对应的CashFlow信息）:

```java
List<AggregationOperation> operations = Lists.newArrayList();
operations.add(Aggregation.match(Criteria.where("_id").is(id)));  //根据id查询到具体的contract
LookupOperation lookupOperation = LookupOperation.newLookup().from("CashFlow")
  .localField("content.cashFlowId")
  .foreignField("name")
  .as("content.cashFlowId");  //用lookup，根据Contract表中的cashFlowId映射到CashFlow这张表中的name，从而获取整个cashFlow的内容
operations.add(lookupOperation);
```



### document管理

#### 创建document（从schema创建document）

- 数据说明（document数据说明）

以上文中第一个创建的合同schema为例，schema与对应数据组合成为document，一个合同的document如下：

```json
{
  "leases": [
    {
      "kind": "ship",
      "amount": 1,
      "size": "medium"
    }
  ],
  "startDate": "2019-02-02",
  "endDate": "2019-06-09",
  //...
}
```

- 前端界面（document editor部分）

（TODO：加入操作时的截图）

以合同为例，用户从合同schema生成document需要以下几步

1. 通过选择template选择对应schema
2. 使用该schema生成form，供用户填写数据
3. 获取用户填写的数据，生成对应document

- 高级组件（advanced component部分）

针对不同的待填项，可以设置其属性（包括填写类型，填写方式等）

```java
 /* 自定义type */
    switch(schema["type"]) {
      case "party": return <PartyField {...formProps} />;
      case "lease": return <LeaseField {...formProps} />;
      default: return DefaultTemplate(formProps);
    }
```

- 后台接口（document管理接口，统一接口，后续所有接口均以合同的document为例）

创建document，实现方式为从某个版本的schema创建，接口如下：

```java
public Contract createContractByTemplateId(ContractRequest contractRequest,String templateId,String commitId){
  logger.info("create contract from templateId"+templateId);
  Contract contract = new Contract(contractRequest.getContent());
  contract.setBasicElements(contractRequest.getBasicElements());
  contract.setTemplateId(templateId);
  contract.setCommitId(commitId);
  contract.setProcessInstanceId(contractRequest.getProcessInstanceId());
  contractRepository.save(contract);
  logger.info("new Contract's id is "+contractRepository.findByTemplateIdAndCommitId(templateId,commitId).getId());
  return contractRepository.findByTemplateIdAndCommitId(templateId,commitId);
}
```

版本维护同样通过JaVers实现，使用JaVers定义repository后，JaVers会自动监听数据修改：

```java
@JaversSpringDataAuditable
public interface ContractRepository extends MongoRepository<Contract, String> {
	//...
}
```

#### 删除document

document中也会有一个是否删除字段delete，删除操作实质上为修改该字段为true，以备后续需要时进行查看

接口如下（TODO：添加对字段的操作）：

```java
public void deleteContract(String id) throws ContractNotFoundException {
  if (!this.contractRepository.findById(id).isPresent())
    throw new ContractNotFoundException("ContractRequest Not Found in contractRepository.");
  logger.info("contract deleted");
  this.contractRepository.deleteById(id);
}
```

#### 更新document

更新document时由于数据发生变化，JaVers会记录修改内容，实现版本维护

接口如下：

```java
public Contract updateContract(ContractRequest contractRequest, String id) throws ContractNotFoundException {
  this.contractRepository.findById(id).ifPresent(contract -> {
    contract.content = contractRequest.getContent();
    contract.setBasicElements(contractRequest.getBasicElements());
    contract.setProcessInstanceId(contractRequest.getProcessInstanceId());
    this.contractRepository.save(contract);
  });
  if (!this.contractRepository.findById(id).isPresent()) {
    throw new ContractNotFoundException("ContractRequest Not Found in contractRepository.");
  }
  return this.contractRepository.findById(id).get();
}
```

#### 查询document（后台接口）

查询某个版本：

```java
public Contract getContractWithJaversCommitId(String contractId, String commitId) throws ContractNotFoundException {
  Contract contract = this.getContract(contractId);
  JqlQuery jqlQuery = QueryBuilder.byInstance(contract).build();
  List<CdoSnapshot> snapshots = javers.findSnapshots(jqlQuery);
  for (CdoSnapshot snapshot : snapshots) {
    if (snapshot.getCommitId().getMajorId() == Integer.parseInt(commitId))
      return JSON.parseObject(javers.getJsonConverter().toJson(snapshot.getState()), Contract.class);
  }
  return null;
}
```

条件复合：

```java

```

关联查询：

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

不过EditorState对象无法用于展示也无法用于持久化存储，所以想把编辑器内容存进数据库就需要进行如下的数据转换：

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

在Braft Editor中，文本以block的形式存于Editor State中，所以先用json path对Editor State的所有block的文本内容进行查找，通过正则表达式匹配出符合定义的占位符，再根据不同的占位符类型进行替换。

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

用replace方法直接将占位符替换成对应的数值；如果数值不合法，会对错误进行标识。

```javascript
const formDataParser = (formData, field) => {
  let data = formData[field]
  return data && typeof (data) !== 'object' ? formData[field] : `「${field} error」`
}
```

- 替换表格占位符

通过正则表达式找到包含表格占位符的block，再按二维数组的大小插入空表格，最后将数值按顺序填入表格中，最后将占位符block删去。

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
