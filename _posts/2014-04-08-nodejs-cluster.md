---
layout: post
title: Node.js - 利用进程通信实现Cluster共享内存
---

Node.js的标准API没有提供进程共享内存，然而通过IPC接口的send方法和对message事件的监听，就可以实现一个多进程之间的协同机制，通过通信来操作共享内存。

IPC的基本用法：

{% highlight js %}
// worker进程 发送消息
process.send('读取共享内存');

// master进程 接收消息 > 处理 > 发送回信
cluster.on('online', function (worker) {
     // 有worker进程建立，即开始监听message事件
     worker.on('message', function(data) {
          // 处理来自worker的请求
          // 回传结果
          worker.send('result')
     });
});
{% endhighlight %}

在Node.js中，通过send和on('message', callback)实现的IPC通信有几个特点。首先，master和worker之间可以互相通信，而各个worker之间不能直接通信，但是worker之间可以通过master转发实现间接通信。另外，通过send方法传递的数据，会先被JSON.stringify处理后再传递，接收后会再用JSON.parse解析。所以Buffer对象传递后会变成数组，而function则无法直接传递。反过来说，就是可以直接传递除了buffer和function之外的所有数据类型（已经很强大了，而且buffer和function也可以用变通的方法实现传递）。

基于以上特点，我们可以设计一个通过IPC来共享内存的方案：

1、worker进程作为共享内存的使用者，并不直接操作共享内存，而是通过send方法通知master进程进行写入(set)或者读取(get)操作。

2、master进程初始化一个Object对象作为共享内存，并根据worker发来的message，对Object的键值进行读写。

3、由于要使用跨进程通信，所以worker发起的set和get都是异步操作，master根据请求进行实际读写操作，然后将结果返回给worker（即把结果数据send给worker）。

## 数据格式

为了实现进程间异步的读写功能，需要对通信数据的格式做一点规范。

首先是worker的请求数据：

{% highlight js %}
requestMessage = {
    isSharedMemoryMessage: true,  // 表示这是一次共享内存的操作通信
    method: 'set', // or 'get' 操作的方法
    id: cluster.worker.id,  // 发起操作的进程（在一些特殊场景下，用于保证master可以回信）
    uuid: uuid,  // 此次操作的（用于注册/调用回调函数）
    key: key,  // 要操作的键
    value: value  // 键对应的值（写入）
}
{% endhighlight %}

master在接到数据后，会根据method执行相应操作，然后根据requestMessage.id将结果数据发给对应的worker，数据格式如下：

{% highlight js %}
responseMessage = {
    isSharedMemoryMessage: true,  // 标记这是一次共享内存通信
    uuid: requestMessage.uuid,  // 此次操作的唯一标示
    value: value  // 返回值。get操作为key对应的值，set操作为成功或失败
}
{% endhighlight %}

规范数据格式的意义在于，master在接收到请求后，能够将处理结果发送给对应的worker，而worker在接到回传的结果后，能够调用此次通信对应的callback，从而实现协同。

规范数据格式后，接下来要做的就是设计两套代码，分别用于master进程和worker进程，监听通信并处理通信数据，实现共享内存的功能。

## User类

User类的实例在worker进程中工作，负责发送操作共享内存的请求，并监听master的回信。

{% highlight js %}
var User = function() {
    var self = this;
    self.__uuid__ = 0;

    // 缓存回调函数
    self.__getCallbacks__ = {};

    // 接收每次操作请求的回信
    process.on('message', function(data) {
       
        if (!data.isSharedMemoryMessage) return;
        // 通过uuid找到相应的回调函数
        var cb = self.__getCallbacks__[data.uuid];
        if (cb && typeof cb === 'function') {
            cb(data.value)
        }
        // 卸载回调函数
        self.__getCallbacks__[data.uuid] = undefined;
    });
};

// 处理操作
User.prototype.handle = function(method, key, value, callback) {

    var self = this;
    var uuid = self.__uuid__++;

    process.send({
        isSharedMemoryMessage: true,
        method: method,
        id: cluster.worker.id,
        uuid: uuid,
        key: key,
        value: value
    });

    // 注册回调函数
    self.__getCallbacks__[uuid] = callback;

};

User.prototype.set = function(key, value, callback) {
    this.handle('set', key, value, callback);
};

User.prototype.get = function(key, callback) {
    this.handle('get', key, null, callback);
};
{% endhighlight %}

## Manager类

Manager类的实例在master进程中工作，用于初始化一个Object作为共享内存，并根据User实例的请求，在共享内存中增加键值对，或者读取键值，然后将结果发送回去。

{% highlight js %}
var Manager = function() {

    var self = this;
   
    // 初始化共享内存
    self.__sharedMemory__ = {};
       
    // 监听并处理来自worker的请求
    cluster.on('online', function(worker) {
        worker.on('message', function(data) {
            // isSharedMemoryMessage是操作共享内存的通信标记
            if (!data.isSharedMemoryMessage) return;
            self.handle(data);
        });
    });
};

Manager.prototype.handle = function(data) {
    var self = this;
    var value = this[data.method](data);

    var msg = {
        // 标记这是一次共享内存通信
        isSharedMemoryMessage: true,             
        // 此次操作的唯一标示
        uuid: data.uuid,
        // 返回值
        value: value
    };

    cluster.workers[data.id].send(msg);
};

// set操作返回ok表示成功
Manager.prototype.set = function(data) {
    this.__sharedMemory__[data.key] = data.value;
    return 'OK';
};

// get操作返回key对应的值
Manager.prototype.get = function(data) {
    return this.__sharedMemory__[data.key];
};
{% endhighlight %}

## 使用方法

{% highlight js %}
if (cluster.isMaster) {

    // 初始化Manager的实例
    var sharedMemoryManager = new Manager();

    // fork第一个worker
    cluster.fork();

    // 1秒后fork第二个worker
    setTimeout(function() {
        cluster.fork();
    }, 1000);
     
} else {

    // 初始化User类的实例
    var sharedMemoryUser = new User();

    if (cluster.worker.id == 1) {
        // 第一个worker向共享内存写入一组数据，用a标记
        sharedMemoryUser.set('a', [0, 1, 2, 3]);
    }

    if (cluster.worker.id == 2) {
        // 第二个worker从共享内存读取a的值
        sharedMemoryUser.get('a', function(data) {
            console.log(data);  // => [0, 1, 2, 3]
        });
    }
  
}
{% endhighlight %}

以上就是一个通过IPC通信实现的多进程共享内存功能，需要注意的是，这种方法是直接在master进程的内存里缓存数据，必须注意内存的使用情况，这里可以考虑加入一些简单的淘汰策略，优化内存的使用。另外，如果单次读写的数据比较大，IPC通信的耗时也会相应增加。

完整代码：<https://github.com/x6doooo/sharedmemory>

