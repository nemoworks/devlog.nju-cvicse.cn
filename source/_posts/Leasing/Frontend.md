---
title: 前端设计
categories:
  - Leasing
date: 2019-07-26 20:45:04
---



### 总体结构

#### 1 文件目录结构
```
leasing-frontend
├── CNAME
├── README.md
├── config
│   ├── config.js
│   ├── defaultSettings.js
│   └── plugin.config.js
├── jest-puppeteer.config.js
├── jest.config.js
├── jsconfig.json
├── mock
│   ├── contractDB.js
│   ├── notices.js
│   ├── route.js
│   ├── sheet.js
│   ├── templateDB.js
│   └── user.js
├── package.json
├── pom.xml
├── public
│   ├── favicon.png
│   └── icons
│       ├── icon-128x128.png
│       ├── icon-192x192.png
│       └── icon-512x512.png
├── src
│   ├── assets
│   │   ├── add.png
│   │   ├── form.png
│   │   ├── logo.png
│   │   ├── logo.svg
│   │   ├── save.png
│   │   └── tree.png
│   ├── components
│   │   ├── Authorized
│   │   ├── CopyBlock
│   │   ├── GlobalHeader
│   │   ├── HeaderDropdown
│   │   ├── HeaderSearch
│   │   ├── NoticeIcon
│   │   ├── PageLoading
│   │   ├── Perish
│   │   ├── SelectLang
│   │   └── SettingDrawer
│   ├── e2e
│   │   ├── __mocks__
│   │   ├── baseLayout.e2e.js
│   │   └── topMenu.e2e.js
│   ├── global.jsx
│   ├── global.less
│   ├── layouts
│   │   ├── BasicLayout.jsx
│   │   ├── BlankLayout.jsx
│   │   ├── UserLayout.jsx
│   │   └── UserLayout.less
│   ├── locales
│   │   ├── en-US
│   │   ├── en-US.js
│   │   ├── pt-BR
│   │   ├── pt-BR.js
│   │   ├── zh-CN
│   │   ├── zh-CN.js
│   │   ├── zh-TW
│   │   └── zh-TW.js
│   ├── manifest.json
│   ├── models
│   │   ├── contract.js
│   │   ├── contractList.js
│   │   ├── global.js
│   │   ├── login.js
│   │   ├── processInstance.js
│   │   ├── processList.js
│   │   ├── setting.js
│   │   ├── sheet.js
│   │   ├── template.js
│   │   ├── templateList.js
│   │   └── user.js
│   ├── pages
│   │   ├── 404.jsx
│   │   ├── Authorized.jsx
│   │   ├── ContractEditor
│   │   ├── ContractList
│   │   ├── FinancialCalculator
│   │   ├── ProcessInstance
│   │   ├── ProcessList
│   │   ├── TemplateEditor
│   │   ├── TemplateList
│   │   ├── Welcome.jsx
│   │   ├── document.ejs
│   │   ├── index.css
│   │   └── index.js
│   ├── service-worker.js
│   ├── services
│   │   └── user.js
│   └── utils
│       ├── Authorized.js
│       ├── authority.js
│       ├── authority.test.js
│       ├── request.js
│       ├── utils.js
│       ├── utils.less
│       └── utils.test.js
├── tests
│   └── run-tests.js
├── tsconfig.json
├── usage.md
└── yarn.lock
```

#### 2 前端分层架构


### 代码说明

#### 1 文件组织约定

#### 2 部分组件说明

- [Braft Editor](https://braft.margox.cn/)

  - 这是一个基于[draft-js](https://draftjs.org/)的Web富文本编辑器，适用于React框架，其内部并不是直接使用HTML作为组件状态的，而是实现了一个EditorState类型，本质上是一个JS对象；

  - 该编辑器支持value和onChange属性，这类似于React中原生的input组件，可以当做典型的受控组件的形式来使用；

    ```javascript
    import React from 'react'
    // 引入编辑器组件
    import BraftEditor from 'braft-editor'
    // 引入编辑器样式
    import 'braft-editor/dist/index.css'
    
    export default class EditorDemo extends React.Component {
    
        state = {
            // 创建一个空的editorState作为初始值
            editorState: BraftEditor.createEditorState(null)
        }
    
        async componentDidMount () {
            // 假设此处从服务端获取html格式的编辑器内容
            const htmlContent = await fetchEditorContent()
            // 使用BraftEditor.createEditorState将html字符串转换为编辑器需要的editorStat
            this.setState({
                editorState: BraftEditor.createEditorState(htmlContent)
            })
        }
    
        submitContent = async () => {
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
                <div className="my-component">
                    <BraftEditor
                        value={editorState}
                        onChange={this.handleEditorChange}
                        onSave={this.submitContent}
                    />
                </div>
            )
    
        }
    
    }
    ```
		
  - value属性必须是一个editorState对象，可以使用`BraftEditor.createEditorState`方法来将JSON或者HTML格式的数据转换成editorState数据；
  
    ```javascript
    // 引入EditorState
    import BraftEditor from 'braft-editor'
    
    // 将raw格式的数据转换成editorState
    const rawString = `{"blocks":[{"key":"9hu83","text":"Hello World!","type":"unstyled","depth":0,"inlineStyleRanges":[{"offset":6,"length":5,"style":"BOLD"},{"offset":6,"length":5,"style":"COLOR-F32784"}],"entityRanges":[],"data":{}}],"entityMap":{}}`
    const editorState = BraftEditor.createEditorState(rawString)
    
    // 将html字符串转换成editorState
    const htmlString = `<p>Hello <b>World!</b></p>`
    const editorState2 = BraftEditor.createEditorState(htmlString)
    ```
  
  - `editorState.toRAW()`可以将editorState对象转化为JSON，用于持久化存储；`editorState.toHTML()`可以将editorState对象转化为HTML，用于展示或打印；
  
  - 实际使用时请避免在onChange中直接toHTML，以下错误写法会导致光标回跳，或导致无限onChange；
  
    ```javascript
    // 错误示范！
    <BraftEditor
      value={BraftEditor.createEditorState(this.state.value)}
      onChange={(editorState) => this.setState({ value: editorState.toHTML() })}
    >
    ```
  

- Json schema editor

### 开发案例

#### 1 修改现有组件

- 找到组件对应的源代码，高度可复用组件的js文件在`/src/component`目录下，页面组件的js文件在`/src/pages`目录下；
- 以`/src/component/Perish/CustomIcon`为例，如果要修改组件样式，则在`CustomIcon.css`中在现有样式上进行修改，或者添加新的样式，并在`CustomIcon.js`中通过styles应用；如果要修改组件逻辑，则在`CustomIcon.js`中进行改动，涉及组件参数的改动，要在所有使用CustomIcon的地方进行对应的修改；

#### 2 构建新的页面

- 在前端目录`/src/pages`中为新页面命名一个文件夹，新页面组件相关的js、css文件都存放在该文件夹下，并且我们约定同一目录下与js文件同名的css文件负责配置前者的组件样式，例如`TemplateList/index.js`中的组件样式可以在`TemplateList/index.css`中进行配置；
- 绘制好新页面的组件后，在`config/config.js`中为新页面配置路由；
- 在`/src/model`中新增新页面的同名js文件，注意使用驼峰命名法，完成新页面涉及的model层数据处理部分，我们约定namespace和页面同名；