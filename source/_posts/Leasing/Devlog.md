---
title: 开发文档
categories:
  - Leasing
date: 2019-09-25 23:36:15
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
    "linkList": {
      "type": "array",
      "items": {
        "type": "link"
      }
    },//linkList存储一系列编号，包括客户编号、资金流编号等，作外键使用
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

- 扩展定义

通过Json Schema Definitions定制若干种数据类型

以上述合同的schema为例，根据客户所需待租物品的不同，可以设计自定义类型leaseType，指定待租物品的种类、数量、大小等属性

也可对应租赁起止日期设计类型：date，通过设置format来使其符合日期定义

```json
{
  "definitions": {
    //...
    "link": {
      "type": "link",
      "customized": "link"
    },
    "leaseType": {
      "type": "object",
      "customized": "leaseType",
      "properties": {
        "kind": { "type": { "enum": ["ship", "truck"]}},
        "amount": {"type": "number"},
        "size": { "type": { "enum": ["little", "medium", "large"]}}
      },
      "required": ["kind", "amount", "size"]
    },
    "date": {
      "type": "string",
      "customized": "date",
      "format": "date"
    }
  }
}
```

其中link属于一项关联属性，用以实现级联查询操作（如从contract的schema中获取到对应的客户信息）

link的content可以是一整个文档，也可以是某一个文档的外键，后台通过检索该外键获取到对应的文档信息

使用时可参照如下格式：

```jsx
{
  "link": {
    "type": "link",
    "discription": "...",
    "by": { "enum": { ["value", "reference"] } },
    "content": //链接或者文档
  }
}
```

一个具体的多层级联查询所用到的link如下（以存在的A、B、C三个文档为例，在A文档中写下如下linkA，在B文档中写下linkB，可通过向后端传字符串A.linkA.linkB来访问到文档C，此处的字符串格式可自定义，由后台的处理方法决定）：

```jsx
{//A.linkList.linkA
  "link": {
    "type": "link",
    "name": "linkA",
    "discription": "link to B by reference"
    "by": "reference",
    "content": "collection1/B.id" //外键为B的id
  }
}

{//B.linkList.linkB
  "link": {
    "type": "link",
    "name": "linkB",
    "discription": "link to C by value",
    "by": "value",
    "content": C 	//代表文档C的字符串
  }
}
```

- 前端界面（schema editor部分）

Schema editor直接使用我们打包好的组件，包含以下三个可选参数

- data：当前schema的源文本
- onChange：editor更改事件处理方法 注：暂时不支持onChange方法中调用setState** 
- extensions：各个definition的定义

以上述合同schema为例，definitions的定义有如下几步：

1. 选取一个类型（下例中为link）写成js文件

```jsx
// '@/components/Link/schema.js'
const linkDef = {
    "link": {
        "customized": "link",
        "type": "link"
    }
}
```

2. 为该类型添加定制组件，即advacned settings，这部分在下面一小节作出说明

```jsx
// '@/components/Link/template.js'
class CustomizedSchemaLink extends React.Component {
		// ...
}
```

3. 为该类型添加定制渲染方式，即Form表单的显示样式，这部分在documents中给予说明

```jsx
// '@/components/Link/field.js'
export default function LinkField(props) {
 		// ... 
}
```

4. 将上述三个文件中的定义合并为一个extension条目

```jsx
// '@/components/Link/index.js'
import field from './field'
import template from './template'
import schema from './schema'

export default {
    field,
    template,
    schema
}
```

5. 对所有类型都执行上述操作，最后合并所有类型成为一个综合的definitions

```jsx
import date from '@/components/Date/index'
import leaseType from '@/components/LeaseType/index'
import link from '@/components/Link/index'
const extensions = {
    date,
    leaseType,
    link
}
```

用户可以通过schema editor来构建合同的schema，这个schema在用户建立表单form时可供选择

``` jsx
import SchemaEditor from 'json-schema-editor-visual-lab'

const definitions = {
  //...
}

class App extends React.Component {
  render() {
    return (
      //...
      <SchemaEditor data = {this.state.schema}
        onChange = {schema => console.log(schema)}
        extensions = {definitions}
        />
    )
  }
}
```

使用schema editor构建上述contract schema，如下例：

{% qnimg schema_editor.png %}

示例代码：

[前端示例](https://github.com/nemoworks/sddm-frontend)

- 高级组件（advanced setting部分）

用户构建schema时，实际上是通过将各种组件拼合在一起的方式来创建的

高级组件使得用户可以对其中的特殊类型进行扩展

定制某种类型的advanced settings，实际上是实现一个自行定制的组件，然后通过defintions的component属性传入editor，使得该advanced settings与自己的对应特殊类型相关联

在打包好的schema editor内部，会检查当前传入的definitions是否在内置的基础类型上有扩展，若有则会采用传入的参数对类型列表进行更新，并使用对应的advanced settings

原有类型Map：

```jsx
export const mapping = data => ({
  string: <CustomizedSchemaString data={data} />,
  number: <CustomizedSchemaNumber data={data} />,
  boolean: <CustomizedSchemaBoolean data={data} />,
  integer: <CustomizedSchemaNumber data={data} />,
  array: <CustomizedSchemaArray data={data} />,
  object: <CustomizedSchemaObject data={data} />,
}[data.type]);
```

当传入definitions之后，该Map有如下更新：

```jsx
export const mapping = data => ({
  //...(原有类型)
  link: <CustomizedSchemaLink data={data} />,
  leaseType: <CustomizedSchemaLeaseType data={data} />,
  date: <CustomizedSchemaDate data={data} />,
}[data.type]);
```

由此组件实现了不同类型的定制及高级配置

Advanced settings中的props包含的data与context两个参数，其中data表示该块schema，context是上下文，可提供将当前所填内容写回整个schema的功能

如下是对link类型的高级设置的定制：

```jsx
/* @/components/JsonSchema/components/SchemaComponents/SchemaLink.js */
import React from 'react'
import { Cascader, Row, Col, Input, Button } from 'antd'
import PropTypes from 'prop-types';
import LocalProvider from '../LocalProvider/index.js';
import { connect } from 'dva'

const mapStateToProps = state => ({
  link: state['jsonSchema-link']
})

class CustomizedSchemaLink extends React.Component {

  render() {
    const { data, dispatch } = this.props;
    console.log(this.state.options)
    return (
      <div>
        <div className="default-setting">{LocalProvider('base_setting')}</div>
        <Row className="other-row" type="flex" align="middle">
          <Col span={4} className="other-label">
            {LocalProvider('link')}：
          </Col>
          <Col span={20}>
            /* link类型advanced settings组件的核心是一个级联选择框 */
            <Cascader placeholder="please select customer schema and document" style={{ width: '100%' }}
              options={this.state.options}//动态选项
              onChange={selectedOptions => {
                /* 将选中的key和val写回当前的schema中 */
                console.log(selectedOptions)
                this.changeOtherValue(selectedOptions[0], "key", data)
                this.changeOtherValue(selectedOptions[2], "val", data)
              }} loadData={this.loadData} changeOnSelect >
            </Cascader>
          </Col>
        </Row>
      </div>
    );
  }

  state = {
    options: []
  }

	/* 回写schema方法 */
  changeOtherValue = (value, name, data) => {
    data[name] = value;
    this.context.changeCustomValue(data);
  };

	componentDidMount() {
    /* 渲染第一层可选项 */
    this.props.dispatch({
      type: 'jsonSchema-link/getCollectionList',
      callback: collectionList => {
        const options = collectionList.map(item => ({ value: item, label: item, isLeaf: false }))
        this.setState({ options })
      }
    })
  }

	/* 级联选择动态加载数据 */
  loadData = selectedOptions => {
    /* 渲染第二层可选项，向后台请求符合要求的所有schema */
    if (selectedOptions.length === 1) {
      const targetOption = selectedOptions[selectedOptions.length - 1];
      targetOption.loading = true;
      /* 向后台请求collection为key的schema */
      this.props.dispatch({
        type: 'jsonSchema-link/getSchemaList',
        key: targetOption.value,
        callback: schemaList => {
          targetOption.loading = false;
          targetOption.children = schemaList.map(item => ({ value: item.id, label: item.name, isLeaf: false }))
          this.setState({
            options: [...this.state.options],
          });
        }
      })
    }
    /* 渲染第三层可选项，向后台请求符合要求的所有document */
    if (selectedOptions.length === 2) {
      const targetOption = selectedOptions[selectedOptions.length - 1];
      targetOption.loading = true;
      /* 根据选中的schemaId向后台请求相关的document */
      this.props.dispatch({
        type: 'jsonSchema-link/getDocumentListBySchemaId',
        id: targetOption.value,
        callback: documentList => {
          targetOption.loading = false;
          targetOption.children = documentList.map(item => ({ value: item.id, label: item.name }))
          this.setState({
            options: [...this.state.options],
          });
        }
      })
    }
  };
}
CustomizedSchemaLink.contextTypes = {
  changeCustomValue: PropTypes.func,
};

export default connect(mapStateToProps)(CustomizedSchemaLink)
```

下图以link为例展示了advanced settings的效果：

{% qnimg link_advancedsettings_select.png %}

选择好对应的document后的效果：

{% qnimg link_advancedsettings_selected.png %}



- 后台接口（schema管理接口）

首先对schema类进行说明：

```java
@Document(collection = "Schema")
public class Schema {
  @Id
  private String id;//作为数据库主键
  private JSONObject schemaContent;//包含了schema中的所有内容
  private Status status;//指示位，表示当前schema的状态
	//...
}

public enum  Status {
  Created, Deleted;//两个值分别对应创建和删除
}
```

创建schema，并设定status为Created（即创建）：

service层接口：

```java
public Schema createSchema(JSONObject content){
  logger.info("create schema.");
  Schema schema = new Schema(content);
  schema.setStatus(Status.Created);
  return this.schemaRepository.save(schema);
}
```

controller层接口：

```java
public Schema createSchema(@RequestBody JSONObject params){
  logger.info("create new Schema. ");
  return schemaService.createSchema(params);
}
```

版本维护实现基于JaVers，JaVers是一个轻量级，完全开源的Java库，用于追踪和记录数据中的更改。它可以配合repository使用，只需要添加一行注解`@JaversSpringDataAuditable`即可

```java
@Repository
@JaversSpringDataAuditable
public interface SchemaRepository extends MongoRepository<Schema,String> {
  List<Schema> findAllByStatus(Status status);
}
```

当数据库中有数据发生修改时，JaVers会自动追踪和记录数据库中发生的修改，存为特定数据类型Snapshot，在获取时可以通过版本号来访问到某次修改过后的数据内容

在创建schema时，JaVers会对应生成初始版本记录，记录效果如下图：

{% qnimg schema_javers.png %}

#### 删除schema

删除作为一种标记操作，而不是真的从数据库中将schema删除

若直接将schema删除，在之后就无法实现从对应生成的合同中查看该schema等相关操作

实现方式为修改schema的status属性为Deleted

接口名称及实现：

service层：

```java
public void deleteSchema(String id){
  this.schemaRepository.findById(id).ifPresent(schema -> {
    schema.setStatus(Status.Deleted);
    this.schemaRepository.save(schema);
  });
}
```

controller层：

```java
public List<Schema> deleteSchema(@PathVariable String id){
  logger.info("delete schema "+id);
  schemaService.deleteSchema(id);
  return schemaService.getAllSchemas();
}
```

#### 更新schema

接口名称及实现:

service层：

```java
public Schema updateSchema(String id,JSONObject jsonObject) {
  logger.info("update schema by id.");
  this.schemaRepository.findById(id).ifPresent(schema -> {
    schema.setSchema(jsonObject);
    this.schemaRepository.save(schema);
  });
  return this.schemaRepository.findById(id).get();
}
```

controller层：

```java
public Schema updateSchema(@PathVariable String id,@RequestBody JSONObject params){
  logger.info("update schema by Id "+ id);
  return schemaService.updateSchema(id,params);
}
```

此时的更新操作会被JaVers识别，记录到版本历史中

#### 查询schema（后台接口）

获取所有未被删除的schema：

service层：

```java
public List<Schema> getAllSchemas(){
  logger.info("get all schemas");
  return schemaRepository.findAllByStatus(Status.Created);
}
```

controller层：

```java
public List<Schema> getSchemas(){
  logger.info("get all schemas");
  return schemaService.getAllSchemas();
}
```

根据id获取对应schema：

service层：

```java
public Schema getSchemaById(String id){
  logger.info("get schema by id");
  Schema schema = schemaRepository.findById(id).get();
  if(schema.getStatus().equals(Status.Created)){
    return schemaRepository.findById(id).get();
  }else return null;
}
```

controller层：

```java
public Schema getSchemaById(@PathVariable String id){
  logger.info("get schema by id "+id);
  return schemaService.getSchemaById(id);
}
```

根据schemaId和javers记录的提交编号检索对应schema的历史版本：

service层：

```java
public Schema getSchemaWithJaversCommitId(String schemaId, String commitId) throws SchemaNotFoundException{
  Schema schema = this.getSchema(schemaId);
  JqlQuery jqlQuery= QueryBuilder.byInstance(schema).build();
  List<CdoSnapshot> snapshots = javers.findSnapshots(jqlQuery);
  for(CdoSnapshot snapshot:snapshots){
    if(snapshot.getCommitId().getMajorId()== Integer.parseInt(commitId))
      return JSON.parseObject(javers.getJsonConverter().toJson(snapshot.getState()),Schema.class);
  }
  return null;
}
```

controller层：

```java
public JSONArray getSchemaWithCommitId(@PathVariable String id, @RequestParam(value = "commitId", defaultValue = "null") String commitId) {
  if (commitId.equals("null")) {
    try {
      logger.info("Get Schema commits with schemaId "+ id);
      return this.schemaService.trackSchemaChangesWithJavers(id);
    } catch (Exception e) {
      throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Schema Not Found.", e);
    }
  } else {
    try {
      logger.info("Cet Schema commit with schemaId and commitId" + commitId);
      JSONArray jsonArray = new JSONArray();
      jsonArray.add(this.schemaService.getSchemaWithJaversCommitId(id, commitId));
      return jsonArray;
    } catch (Exception e) {
      throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Schema Not Found.", e);
    }
  }
}
```



### document管理

#### 创建document（从schema创建document）

- 数据说明（document数据说明）

以上文中第一个创建的合同schema为例，schema与对应数据组合成为document，一个合同的document如下：

```json
{
  "linkList": [
    {
      "link": "customer"
    },
    {
    	"link": "cashflow"
    },
    {
      "link": "lease"
    }
  ],
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

{% qnimg select_schema.png %}

以合同为例，用户从合同schema生成document需要以下几步

1. 通过选择模板选择对应schema
2. 使用该schema生成form，供用户填写数据
3. 获取用户填写的数据，生成对应document

一个创建好的form表单如下：

{% qnimg form_editor.png %}

- 高级组件（advanced component部分）

针对不同的待填项，可以设置其属性（包括填写类型，填写方式，对所填项的各种要求等）对应这些项在表单中的渲染方式，这些组件均由用户自定义，构成最终需要生成的表单

```jsx
import date from '@/components/Date/index'
import leaseType from '@/components/LeaseType/index'
import link from '@/components/Link/index'

const extensionsForm = {
    date: date.field,
    leaseType: leaseType.field,
    link: link.field
}

render() {
  return (
    // ...
    <Form schema={JSON.parse(this.state.schema)}
      extensions={extensionsForm}
      />
  )
}
```

以渲染link类型对应的组件LinkField为例：

```javascript
/* @/components/JsonSchemaForm/components/fields/LinkField.js */
import React from 'react'
import { connect } from 'dva'
import { Button, Checkbox, Select } from 'antd'

const mapStateToProps = state => ({
    link: state['jsonSchemaForm-link']
})

class LinkField extends React.Component {

    render() {
        const { label, schema, link } = this.props
        const selectList = link.content.map(item => <Select.Option value={item}>{item}</Select.Option>)
        debugger
        return (
            <div>
                <label>{label}</label>
          			/* LinkField组件的核心是一个下拉选择框，可选项根据向后台请求得到的document渲染 */
                <Select defaultValue={this.props.formData} style={{ width: 120 }} onChange={value => this.props.onChange(value)}>
                    {selectList}
                </Select>
            </div>
        )
    }

    state = {
        selected: this.props.formData,
    }

    componentDidMount() {
     	 	/* 根据schema中的key和val，向后台请求对应的document */
        this.props.dispatch({
            type: 'jsonSchemaForm-link/getDocument',
            key: this.props.schema.key,
            val: this.props.schema.val,
        })
    }
}

export default connect(mapStateToProps)(LinkField)
```

- 后台接口（document管理接口）

首先对document类进行说明：

```java
@org.springframework.data.mongodb.core.mapping.Document(collection = "Document")
public class Document {
  @Id
  private String id;//数据库主键
  private String schemaId;//对应的schema的编号
  private String collectionName;//属于什么类型的文档，如客户、合同等
  private JSONObject data;//用户填写的json数据
  private Status status;//同schema中的status，表示是否删除
}
```

创建document，实现方式为从某个版本、某个类型的schema创建，接口如下：

service层：

```java
public Document createNewDocument(String schemaId,JSONObject content{
  logger.info("create new document by schemaId "+schemaId);
  Document document = new Document();
  document.setSchemaId(schemaId);
  document.setData(content);
  document.setStatus(Status.Created);
  document.setCollectionName(content.getString("title"));
  documentRepository.save(document);
  return mongoTemplate.insert(document,document.getCollectionName());
}
```

controller层：

```java
public Document createDocument(@RequestBody JSONObject params, @RequestParam(value = "schemaId",defaultValue = "null") String schemaId, @RequestParam(value = "collectionName",defaultValue = "null") String collectionName){
  logger.info("create new Document by SchemaId "+schemaId+" and collectionName "+collectionName);
  return documentService.createNewDocument(schemaId,params);
}
```

版本维护同样通过JaVers实现，使用JaVers定义repository后，JaVers会自动监听数据修改：

```java
@Repository
@JaversSpringDataAuditable
public interface DocumentRepository extends MongoRepository<Document,String> {
}
```

document的版本记录效果如下：

{% qnimg document_javers.png %}

#### 删除document

document中删除操作与schema相同，即修改status属性为Deleted，以备后续需要时进行查看，接口如下：

service层：

```java
public void deleteDocument(String id,String collectionName){
  Document document1 = this.getDocumentByIdAndCollectionName(id,collectionName);
  document1.setStatus(Status.Deleted);
  mongoTemplate.save(document1,collectionName);
  documentRepository.save(document1);
}
```

controller层：

```java
public List<Document> deleteDocument(@PathVariable String id, @RequestParam(value = "collectionName",defaultValue = "null") String collectionName){
  logger.info("delete document "+id);
  this.documentService.deleteDocument(id,collectionName);
  return this.documentService.getAllDocumentsByCollectionName(collectionName);
}
```

#### 更新document

更新document时由于数据发生变化，JaVers会记录修改内容，实现版本维护，接口如下：

service层：

```java
public Document updateDocument(String id,String collectionName,JSONObject content){
  logger.info("update document by id.");
  this.documentRepository.findById(id).ifPresent(document -> {
    document.setData(content);
    this.documentRepository.save(document);
  });
  Document document = this.getDocumentByIdAndCollectionName(id,collectionName);
  document.setData(content);
  return mongoTemplate.save(document,collectionName);
}
```

controller层：

```java
public Document updateDocument(@PathVariable String id,@RequestParam(value = "collectionName",defaultValue = "null")String collectionName,@RequestBody JSONObject params){
  logger.info("update document by Id "+ id+" and collectionName "+collectionName);
  return documentService.updateDocument(id,collectionName,params);
}
```

#### 查询document（后台接口）

获取某个类型的所有未被删除的document：

service层：

```java
public List<Document> getAllDocumentsByCollectionName(String collectionName){
  logger.info("get all documents by collectionName.");
  Query query = new Query();
  query.addCriteria(Criteria.where("status").is(Status.Created));
  return mongoTemplate.find(query,Document.class,collectionName);
}
```

controller层：

```java
public List<Document> getDocuments(@RequestParam(value = "collectionName",defaultValue = "null")String collectionName){
  logger.info("get all documents by collectionName "+collectionName);
  return this.documentService.getAllDocumentsByCollectionName(collectionName);
}
```

在某个类型的document中根据id检索：

service层：

```java
public Document getDocumentByIdAndCollectionName(String id,String collectionName){
  logger.info("get document by id and collectionName"+id);
  return this.mongoTemplate.findOne(new Query(Criteria.where("_id").is(id)),Document.class,collectionName);
}
```

controller层：

```java
public Document getDocumentByIdAndCollectionName(@PathVariable String id,@RequestParam(value = "collectionName",defaultValue = "null")String collectionName){
  logger.info("get document by id "+id+" and collectionName "+collectionName);
  return documentService.getDocumentByIdAndCollectionName(id,collectionName);
}
```

根据documentId和javers记录的提交编号检索对应document的历史版本：

service层：

```java
public Document getDocumentWithJaversCommitId(String documentId, String commitId){
  Document document = this.documentRepository.findById(documentId).get();
  JqlQuery jqlQuery = QueryBuilder.byInstance(document).build();
  List<CdoSnapshot> snapshots = javers.findSnapshots(jqlQuery);
  for (CdoSnapshot snapshot : snapshots) {
    if (snapshot.getCommitId().getMajorId() == Integer.parseInt(commitId))
      return JSON.parseObject(javers.getJsonConverter().toJson(snapshot.getState()), Document.class);
  }
  return null;
}
```

controller层：

```java
public JSONArray getDocumentWithCommitId(@PathVariable String id,@RequestParam(value = "commitId", defaultValue = "null") String commitId) {
  if (commitId.equals("null")) {
    try {
      logger.info("Get Document commits with documentId "+ id);
      return this.documentService.trackDocumentChangesWithJavers(id);
    } catch (Exception e) {
      throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Document Not Found.", e);
    }
  } else {
    try {
      logger.info("Cet Document commit with commitId" + commitId);
      JSONArray jsonArray = new JSONArray();
      jsonArray.add(this.documentService.getDocumentWithJaversCommitId(id, commitId));
      return jsonArray;
    } catch (Exception e) {
      throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Document Not Found.", e);
    }
  }
}
```

关联（聚合）查询，根据document中记录的对应link访问到其他document：

```java
public Document getCompleteDocument(String id, String collectionName, JSONArray filterFactors){
  logger.info("get document by id "+id);
  List<AggregationOperation> operations = Lists.newArrayList();
  operations.add(Aggregation.match(Criteria.where("_id").is(id)));
  for(int i=0;i<filterFactors.size();i++){
    JSONObject filterFactor = filterFactors.getJSONObject(i);
    LookupOperation lookupOperation = LookupOperation.newLookup()
      .from(filterFactor.getString("collectionName"))
      .localField(filterFactor.getString("localParam"))
      .foreignField(filterFactor.getString("foreignParam"))
      .as(filterFactor.getString("localParam"));
    operations.add(lookupOperation);
  }
  Aggregation aggregation = Aggregation.newAggregation(operations);
  AggregationResults<Document> contractAggregationResults= mongoTemplate.aggregate(aggregation,collectionName, Document.class);
  List<Document> documents = contractAggregationResults.getMappedResults();
  if(documents.size()==1)
    return documents.get(0);
  else return null;
}
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

（报表）从外部Excel导入数据，分析后生成charts

#### spread sheet



### 流程管理

每个task都可以关联一个schema 将schema具体内容放入formporperty中

研究task的具体生命周期
