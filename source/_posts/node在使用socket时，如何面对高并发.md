---
title: node在使用socket时，如何面对高并发
date: 2019-11-24 15:13:06
tags:
---
### 服务器如何应对聊天室高峰期（使用node搭建socket服务）
> 聊天室在直播场景中应用广泛，而直播的盛行，导致聊天室经常处于人数众多，即高并发的状态。本文主要讲解队列在聊天室高并发下的应用。

#### 一、背景
在node中使用socket.io去实现聊天室，可以同时兼容ie8等环境。加上node通过事件循环机制，以及异步的I/O操作，从而可以实现大量的聊天室消息同时并发，但不会阻塞主进程的运行，使聊天室消息可以及时被响应。
但是，当聊天室消息过多时，对服务器的带宽和网速要求提高。这时不仅需要增加服务器（使用负载均衡），提高服务器带宽，还要对发送的消息实现控制，从而控制带宽。通过队列，可以实现对聊天消息量的控制。


#### 二、队列
日常生活中，我们做很多事情都需要排队，比如上公交车的时候。而排队都是先排先进的，而队列也是这样。队列就是一组数据，先进先出。如下图所示：
![5fed150010168950017d1e38c6967757.png](evernotecid://4DB8DB53-1814-4A14-B958-C27969DAC6E3/appyinxiangcom/10748170/ENResource/p296)

#### 三、队列在高并发中的作用
当聊天室同时存在多人时，需要将消息同时发送给多个人。比如，房间同时存在一万人，如果每个人发一条消息，那就有1亿的并发量。由此可以看出，控制消息的发送，可以从房间的在线人数入手。
##### 1、流程设计
假设在线人数为count，则设计如下流程图：
![dce329e9c13e4d6853dc2a3648eb6f19.png](evernotecid://4DB8DB53-1814-4A14-B958-C27969DAC6E3/appyinxiangcom/10748170/ENResource/p299)

* 当聊天室收到消息时，需要判断房间在线人数，决定是否加入队列，队列使用redis去实现。
* 开启一个独立的node进程，不断地定时，去检查redis队列。如果有数据，则出队列并发送，并将在线人数count进行累加，保存在total里面。
* 每次发送完数据，就检查total的值，如果超过一定人数，就停止出队列，定时一段时间后，再去队列获取数据。

##### 2、具体代码实践示例
`````javascript
// 队列广播
class EmitMsgUseQueue {
  constructor({
    checkTime = DEFAULT_CHECK_TIME,
    runTime = DEFAULT_RUN_TIME,
    io, client, queue
  } = {}) {
    this.checkTime = checkTime;
    this.runTime = runTime;
    this.io = io;
    this.client = client;
    this.queue = queue;
    this.run();
  }
  run() {
    const that = this;
    setTimeout(() => {
      this.checkAndEmit().then(() => this.run.call(that));
    }, this.checkTime);
  }
  checkAndEmit() {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        this.pop().then(emitMsg => {
          if (!emitMsg) return resolve();
          this.emit(emitMsg);
          resolve(this.checkAndEmit());
          return;
        }).catch(reject);
      }, this.runTime);
    });
  }
  pop() {
    return this.client.lpop(this.queue).then(list => {
      if (!list) return null;
      // logger.info(list);
      const emitMsg = parseJson(list);
      if (!emitMsg) return null;
      return emitMsg;
    }).catch(err => logger.error(`获取广播消息队列${this.queue}时出错`, err));
  }
  emit(emitMsg) {
    const done = ({ sendData, roomId }) => {
      if (roomId) {
        logger.info(sendData);
        this.io.to(roomId).emit('message', stringify(sendData));
      }
    };
    if (Array.isArray(emitMsg)) {
      emitMsg.forEach(el => {
        done(el);
      });
    } else {
      done(emitMsg);
    }
  }
  pushMsgToQueueForAsyncEmit(obj) {
    return pushMsgToQueueForAsyncEmit({ obj, client: this.client, queue: this.queue });
  }
}
`````
上面是一个基础类，实现基本的定时任务，以及检查队列，发送队列，而下面的类则增加了对在线人数的检查。


````javascript
// 根据count来设置队列广播
class EmitMsgUseQueueByCount extends EmitMsgUseQueue {
  constructor({
    checkTime = DEFAULT_CHECK_TIME,
    runTime = DEFAULT_RUN_TIME,
    totalCount = MAX_COUNT_SECOND,
    io, client, queue,
  } = {}) {
    super({ checkTime, runTime, io, client, queue });
    this.totalCount = totalCount;
  }
  checkAndEmit() {
    let currentCount = 0;
    const doIt = () =>
      this.pop().then(emitMsg => {
        if (!emitMsg) return 'over';
        const { count } = emitMsg;
        currentCount += parseInt(count);
        this.emit(emitMsg);
        return currentCount < this.totalCount ? 'keep' : 'outdo';
      });
    return new Promise(resolve => {
      setTimeout(async () => {
        let isKeep = true;
        while (isKeep) {
          const result = await doIt();
          switch (result) {
            case 'outdo':
              isKeep = false;
              currentCount = 0;
              resolve(this.checkAndEmit());
              break;
            case 'over':
              isKeep = false;
              resolve();
              break;
          }
        }
      }, this.runTime);
    });
  }
}
```