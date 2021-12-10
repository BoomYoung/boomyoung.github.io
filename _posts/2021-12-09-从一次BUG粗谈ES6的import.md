---
title: 从一次BUG粗谈ES6的import
layout: article
tags:
  - 前端
  - react
mode: immersive
lang: zh-Hans
outhor: Boom Young
pageview: true
article_header:
  type: cover
  image:
    src: /screenshot.jpg
---

![img.png](/assets/blog/2021-12-09/01.png)

### 问题描述

报错很明显，`this.props.dispatch is not a function`，我在组件内无法使用dispatch函数。

### 问题分析

```shell
.
└── src
    ├── App.jsx
    └── router.js
    └── models
    └── services
    └── Alert
        └── index.jsx
        └── Rules
            └── index.jsx
            └── Add.js
```

src/Alert/Rules/index.jsx

```javascript
import React, { Component } from 'react';
import { connect } from 'dva';

export class Rules extends Component {
    constructor(props) {
        super(props);
        this.state = {
            data: [],
            regionOptions: [],
        };
    };

    getData(region) {
        this.props.dispatch({
            type: 'alert/getUAlertRules',
            params: {
                "region": region,
            },
            callback: (rsp) => {
                if (rsp.RetCode === 0) {
                    // message.success(获取成功', 1);
                    this.setState({
                        data: rsp.Data,
                        regionOptions: this.regionOptions(rsp.Data)
                    })
                } else {
                    message.error(rsp.Msg, 2);
                }
            }
        })
    }
  // 以下省略
	...
  
  function mapStateToProps(state) {
    return {
      
    };
}

export default connect(mapStateToProps)(Rules);
```

众所周知，dispatch是redux的一个异步请求方法，在dva中要使用该方法，通常需要使用dva/connect将State绑定到组件中，State是什么？state是整个应用的状态数据，即包含dispatch方法。

```js
import { connect } from 'dva';

function mapStateToProps(state) {
  return { todos: state.todos };
}
connect(mapStateToProps)(App);
```

connect方法需要传入mapStateToProps函数，该函数返回一个对象，用于建立 State 到 Props 的映射关系。此时在App内部`this.props`就拥有了dispatch方法。可问题就出现在这，该方法我已经实现，可为什么依然无法使用`this.props.dispacth`呢？

于是我打印了下`this.props`惊奇的发现竟然是个空对象`{}`!

这说明一个问题，connect并没有生效，在我翻阅资料，扣字找BUG都没行的时候，我放弃了。绕过了model直接去请求了后端接口，奈斯没问题！


但是，当我新写了一个子组件`src/Alert/Rules/Add.js`里也用了dispatch，并且在`src/Alert/Rules/index.jsx`里调用`Add`时，我惊了，竟然没有报错！Why？！

这至少说明connect的单词没有写错，我决定从app.js里一层一层往下找。

基本调用关系很明确，App => router => Alert index => Rules index => Add，当我查阅到Alert/index.js代码时我终于发现了问题所在。

src/Alert/index.jsx

```
import React, { PureComponent } from 'react';
import { connect } from "dva";
import { TabNav } from 'common-components';
import { Rules } from './Rules'

export class Alert extends PureComponent {
    render() {
        return (
            <div>
                <TabNav defaultMenuKey={'statement'}
                    routes={[
                        {
                            title: '告警规则管理',
                            key: 'rules',
                            content: <Rules />
                        },
                    ]}
                />
            </div>
        );
    }
}

function mapStateToProps(state) {
    return {
    };
}

export default connect(mapStateToProps)(Alert);
```

小伙伴们有没有看出端倪，对，我使用了`import { Rules } from './Rules'`的方式来导入Rules包。下面就ES6及以上包的导入导出来详细说说。

### export

export包导出有两种方式，一种是export default，一种是不带default直接导出。

```js
// export.js

// 直接导出
export function foo() {};
export var bar = 1;

// default导出
function hello() {}；
export default hello；
```

### import

import导入包也有两种方式，一种带{}，一种不带，与上面export一一对应：

```
// import.js

// 导入foo()和变量bar
import {foo, bar} from './export.js' // 正确
import foo, bar from './export.js' // 报错

// 导入default包
import hello from './export.js' // 正确
import { hello } from './export.js' // 报错
```

回到上面的代码中，src/Alert/index.jsx中import { Rules }是直接导入的Rules包，而包裹了connect方法的Rules是使用default导出的，问题找到了，修改src/Alert/index.jsx代码删掉import的{}，问题迎刃而解！


### 总结
无。。。