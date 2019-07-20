---
title: Node.js流（一）Readable Stream
date: 2019-07-20 22:06:09
tags: Node.js
---

### 生产者消费者问题

如果想要深入的理解stream，生产者消费者问题是必须要跨过去的一个问题。说一下我对这个问题的理解。这个问题是多进程共享一段数据的问题。一个是生产者，一个是消费者，二者中间有一个缓冲区，生产者往缓冲区添加数据，消费者从缓冲区取数据。那么由此就产生了一些问题，那就是生产者不应该在缓冲区满的时候继续添加数据，而消费者同样不应该在缓冲区空的时候去取数据。那么有什么解决方法呢？

其解决方法就是生产者如果发现缓冲区满的时候就去休眠，当下次消费者消费数据时再将生产者唤醒。同样对于消费者也是，如果发现缓冲区为空时就休眠，只有当下次生产者向缓冲区添加数据的时候再将其(消费者)唤醒。

### Readable Stream

我们知道请求request也是一个可读流，那么针对readable stream的一个常见用法就是取post请求的请求体数据，看下代码

```js
const http = require('http');

const getBody = (req) => {
  // 暂定请求体为json
  let result = '';
  return new Promise((resolve) => {
    req.on('data', (chunk) => {
      result += chunk.toString();
    });
    req.on('end', () => {
      if (!result) resolve({});
      else resolve(JSON.parse(result));
    });
  });
};


const app = http.createServer((req, res) => {
  getBody(req).then((data) => {
    res.write(JSON.stringify(data));
    res.end();
  });
});

app.listen(3000, () => {
  console.log('app is starting!!!');
});
```

先不看上面的代码，我们可以看下为什么说request也是一个Readable Stream，可以直接看Node.js的源代码，在/_lib/_http_incoming.js这个文件里。

```js
function IncomingMessage(socket) {
  Stream.Readable.call(this);
  //  其余代码省略....
}
Object.setPrototypeOf(IncomingMessage.prototype, Stream.Readable.prototype);
Object.setPrototypeOf(IncomingMessage, Stream.Readable);
```

再看上面的代码，我们发现通过事件监听就可以取到请求体中的数据。为什么可以这样就能取到数据，这个时候就需要看一下源代码了。具体代码在lib/_stream_readable.js里。

先看下构造函数里面的内容，我们主要是了解其内部原理，大部分都可以不用管。

```js
function Readable(options) {
  this._readableState = new ReadableState(options, this, isDuplex);
  
  Stream.call(this);
}
```
主要也就这两行代码，其中ReadableState稍后再看，Stream是一个C++模块，可以不用管。由于我们最上面写获取请求体的代码是事件监听，那就直接从那开始看起吧。

```js
Readable.prototype.on = function(ev, fn) {
  const res = Stream.prototype.on.call(this, ev, fn);
  const state = this._readableState;

  if (ev === 'data') {
    state.readableListening = this.listenerCount('readable') > 0;

    if (state.flowing !== false)
      this.resume();
  } else if (ev === 'readable') {
    if (!state.endEmitted && !state.readableListening) {
      state.readableListening = state.needReadable = true;
      state.flowing = false;
      state.emittedReadable = false;
 
      if (state.length) {
        emitReadable(this);
      } else if (!state.reading) {
        process.nextTick(nReadingNextTick, this);
      }
    }
  }

  return res;
};
```

看代码，我们发现其专门针对data和readable两种监听类型做了判断。我们之前使用的是监听data事件，那就先从这个看吧。由于flowing默认为null，因此会执行this.resume(); 继续往下看 resume方法

```js
Readable.prototype.resume = function() {
  const state = this._readableState;
  if (!state.flowing) { // 初始为null
    state.flowing = !state.readableListening; // flowing设置为true
    resume(this, state);
  }
  state.paused = false;
  return this;
};
```
state.flowing为null, 而state.readableListening = this.listenerCount('readable') > 0，也就是我们是否注册了readable事件。所以state.readableListening = false; 从而state.flowing为true，然后执行resume。

```js
function resume(stream, state) {
  if (!state.resumeScheduled) { // 初始值为false
    state.resumeScheduled = true;
    process.nextTick(resume_, stream, state); // 下一次事件轮询执行
  }
}

function resume_(stream, state) {
  if (!state.reading) { // 初始为false
    stream.read(0); // 并不消费数据，会触发_read
  }

  state.resumeScheduled = false;
  stream.emit('resume');
  flow(stream); // 流开始流动
  if (state.flowing && !state.reading)
    stream.read(0); 
}
```
那么如果是readable事件呢？

```js
if (!state.endEmitted && !state.readableListening) {
  state.readableListening = state.needReadable = true;
  state.flowing = false;
  state.emittedReadable = false;

  if (state.length) { // 初始为0
    emitReadable(this);
  } else if (!state.reading) { // 初始为false
    process.nextTick(nReadingNextTick, this);
  }
}
```

因此会将flowing设置为false，并且会执行nReadingNextTick

```js
function nReadingNextTick(self) {
  self.read(0);
}
```

从上面能看出来，readable事件并不会自动消费流数据，这时候还是需要我们手动调用read()方法。

read和readable的调用时间是： readable比read优先控制流， 每次在有新的数据时会调用readable，除非手动调用read，否则不会消费流，如果手动调用read()，就会触发read事件。如果到达了数据的终点，则会再次触发readable，之后才会触发end事件。

上面是消费流的情况，那么资源池向缓冲区添加内容的机制又是怎样的呢？这就需要看push操作流

```js
Readable.prototype.push = function(chunk, encoding) {
  return readableAddChunk(this, chunk, encoding, false);
};

function readableAddChunk(stream, chunk, encoding, addToFront) {
  const state = stream._readableState;

  chunk = Buffer.from(chunk, encoding).toString(state.encoding);
  if (chunk === null) {
    state.reading = false;
    onEofChunk(stream, state);
  } else {
    addChunk(stream, state, chunk, true);
  }
  // 直到到达highWaterMark的值时，返回false
  return !state.ended &&
    (state.length < state.highWaterMark || state.length === 0);
}

function addChunk(stream, state, chunk, addToFront) {
  if (state.flowing && state.length === 0 && !state.sync) { // flowing模式 触发data方法
    state.awaitDrain = 0;
    stream.emit('data', chunk);
  } else { // 暂停模式 
    state.length += state.objectMode ? 1 : chunk.length;
    if (addToFront)
      state.buffer.unshift(chunk);
    else
      state.buffer.push(chunk);

    if (state.needReadable)
      emitReadable(stream);
  }
  maybeReadMore(stream, state);
}
```

这是删减后其主要的思路的部分，可以看出资源池不断的向缓冲区添加内容，达到highWaterMark就会返回false。而highWaterMark的大小是8M。并且如果是在监听readable事件，会在push的时候触发readable事件，但是不会消费数据，除非我们手动调用read(). 而如果是在监听data事件，则会一直触发data事件消费数据。