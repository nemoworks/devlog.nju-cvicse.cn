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



### 开发案例

#### 1 修改现有组件

#### 2 构建新的页面