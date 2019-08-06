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

- Braft Editor

  Braft Editor是基于draft-js开发的编辑器，而draft-js内部并不是直接使用HTML作为组件状态的，它自己实现了一个EditorState类型，本质上是一个JS对象；`editorState.toRAW()`可以将EditorState类型转化为一个JSON对象，`editorState.toHTML()`可以将EditorState类型转化为HTML；

  编辑器的value属性必须是一个editorState对象

  实际使用时请避免在onChange中直接toHTML，配合节流函数或者在合适的时机使用更恰当

### 开发案例

#### 1 修改现有组件

- 首先要找到组件对应的源代码，高度可复用组件的js文件在`/src/component`目录下，页面组件的js文件在`/src/pages`目录下，

#### 2 构建新的页面

- 在前端目录`/src/pages`中为新页面命名一个文件夹，新页面组件相关的js、css文件都存放在该文件夹下，并且我们约定同一目录下与js文件同名的css文件负责配置前者的组件样式，例如`TemplateList/index.js`中的组件样式可以在`TemplateList/index.css`中进行配置；
- 绘制好新页面的组件后，在`config/config.js`中为新页面配置路由；
- 在`/src/model`中新增新页面的同名js文件，注意使用驼峰命名法，完成新页面涉及的model层数据处理部分；