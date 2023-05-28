---
title: 在node中使用单元测试
date: 2019-08-25 15:12:29
tags:
---
## 在node中使用单元测试
> 背景：目前聊天室开发时，都是在前端页面进行调试。在聊天室项目本身没有提供测试的环境。所以这里开始引入单元测试，方便后面的开发。

#### 1、使用的工具
主要是用[mocha](https://mochajs.org/#asynchronous-code)和node原生的断言[assert](https://nodejs.org/api/assert.html)。

安装：
`````
npm install mocha --save-dev
`````
可以局部安装并在package.json中，添加test，如
``````
"test": "mocha ./test/*.js"
```````
这样所有测试文件都会被执行。
也可以全局安装，如
``````
npm install mocha -g
``````

执行单个测试文件，直接在命令行中运行，如执行测试文件testForSign.js
``````
mocha test/testForsign.js
``````

#### 2、例子
从聊天室出发，我们有接口、socket消息、通用方法、数据库测试等，下面进行举例。
##### 接口
下面介绍对开始签到接口的测试：
`````javascript
const { ajax, createApiSign } = require('../common/common')();
const assert = require('assert');

describe('to test sign', () => { // 对测试的描述
  it('test request port', async () => { // 开始一次测试
  // 使用try捕捉所有错误，assert断言本身不通过时，也会抛出错误
    try {
      const result = await toStartSign();
      // 这里断言接口返回是否为200，如果不是，会抛出错误。这里第三个参数可以加上，表示需要的错误提示信息。
      assert.equal(result.code, 200);
    } catch (err) {
      // 这里直接使用fail抛出异常
      assert.fail('request startSign port fail, ' + err.message);
    }
  });
});

// 请求开始签到接口的函数
function toStartSign() {
  const formData = {
    roomId: '200060',
    message: '开始签到啦！！！！',
    userId: '1566315813544',
    timestamp: Date.now(),
  };
  formData.sign = createApiSign('polyvChatSign', formData);
  return ajax({
    url: 'https://apichat.polyv.net/front/startSign',
    formData,
    dataType: 'body'
  });
}
``````
在命令行中输入`mocha test/testForSign.js`，执行后的结果：
``````bash
0 passing (244ms)
  1 failing

  1) to test sign
       test request port:
     AssertionError [ERR_ASSERTION]: request startSign port fail, 找不到用户socketId，用户未登陆聊天室
      at Context.it (test/testForSign.js:10:14)
      at process._tickCallback (internal/process/next_tick.js:68:7)
``````
由于运行时未登陆聊天室，所以接口返回的状态码不是200

##### socket消息
使用模块`socket.io-client`连接聊天室，如
`````javascript
// 连接聊天室，公共方法，在common.js里面
connectSocket({ userId = '123456', nick = 'yang', roomId = 200060, type = 'user' } = {}) {
    // 连接聊天室
    const socket = io.connect('ws://chat.polyv.net', {
      'query': 'token=',
      'transports': ['websocket']
    });
    if (!(userId && nick && roomId && type)) return socket;
    // 发送login事件进行登陆
    socket.on('connect', function () {
      var loginData = {
        EVENT: 'LOGIN',
        values: [nick, 'https://livestatic.videocc.net/assets/wimages/missing_face.png', userId],
        roomId,
        type
      };
      socket.emit('message',JSON.stringify(loginData));
    });
    return socket;
}
``````
下面测试开始签到，使用事件`SIGN_IN`来举例测试：





