---
title: HTTP里的幂等性
date: 2020-11-17 01:17:37
tags: ["http"]
---

机缘巧合之下，曾经看到过一个 bug，具体表现为：iPhone 上的浏览器不能正确通过 CDN 鉴权。简单来说当时的情况是，我们需要同时上传 3 个文件，我们会用用户 token 来换 3 个独立的随机数 id，这三个 id 会被 CDN 服务器认为合法，用户可以直接上传到 CDN 上而无需在我们自己服务器上中转。

但 iOS 用户很快就出现了一个奇怪的问题，用户的 3 个文件只能成功上传 1 个，剩下 2 个就无法正常上传。再进一步调试后我们发现，在上传任意一个文件后，剩下两个 id 变成了非法。再进一步地，我们发现 Safari 的请求返回值，居然和我们预期的完全不一样...

要搞懂这个 bug 背后的原因，我们首先得了解一下，什么是幂等性。

<!-- more -->

### 什么是幂等性

> HTTP/1.1 中对幂等性的定义是：一次和多次请求某一个资源对于资源本身应该具有同样的结果（网络超时等问题除外）。也就是说，其任意多次执行对资源本身所产生的影响均与一次执行的影响相同。

这里需要关注几个重点：

- 幂等关注的是以后的多次请求是否对资源产生的副作用。
- 幂等性指的是作用于结果而非资源本身。可能某个方法可能每次会得到不同的返回内容，但并不影响资源，这样的也满足幂等性，例如 get 服务器当前时间
- 网络超时等问题，不是幂等的讨论范围。

HTTP/1.1 规格上 API 的 GET 是幂等的，如果我们把 x 当作服务器的状态，f 是 GET 请求操作，那么我们有：
![img](1.jpg)
这确保了多次调用接口产生的副作用，和一次调用是一致的。我们似乎可以得到推论认为每次 GET 请求的返回都应该是一样的。

但如果我们仔细来看 [rfc7231](https://tools.ietf.org/html/rfc7231#section-4.2.2) 对于 HTTP 幂等的定义，其只是为了确保请求重新发送的可靠性，而不是不允许后端进行任何非幂等的操作。

### 什么情况下需要幂等性

业务开发中，经常会遇到重复提交的情况，无论是由于网络问题无法收到请求结果而重新发起请求，或是前端的操作抖动而造成重复提交情况。 在交易系统，支付系统这种重复提交造成的问题有尤其明显，比如：

用户在 APP 上连续点击了多次提交订单，后台应该只产生一个订单；

向支付宝发起支付请求，由于网络问题或系统 BUG 重发，支付宝应该只扣一次钱。 很显然，声明幂等的服务认为，外部调用者会存在多次调用的情况，为了防止外部多次调用对系统数据状态的发生多次改变，将服务设计成幂等。

### 复现 & 修复

上面的例子我们可以简单写个 demo 来复现一下，先写一个接口，每次请求后我们都返回一个 index 计数，前端通过调用这个接口来获取当前的 index 数值

```js
// index.js
let http = require("http");
let fs = require("fs");

let index = 1;
let server = http.createServer(function (request, response) {
  let url = request.url;
  if (url === "/") {
    fs.readFile("./index.html", function (err, data) {
      if (!err) {
        response.writeHead(200, { "Content-Type": "text/html;charset=UTF-8" });
        response.end(data);
      } else {
        throw err;
      }
    });
  } else if (url === "/data") {
    response.writeHead(200, { "Content-Type": "text/html" });
    response.end("index: ===> " + index++);
  }
});

// 让服务器监听8080端口:
server.listen(8080);

console.log("Server is running at http://127.0.0.1:8080/");
```

```html
<!-- index.html -->
<script>
  for (let i = 0, len = 3; i < len; i++) {
    getFunc();
  }
  function getFunc() {
    let http = new XMLHttpRequest();
    http.onreadystatechange = function () {
      if (http.status == 200 && http.readyState == 4) {
        let msg = http.responseText;
        let box = document.getElementById("box");
        box.innerHTML = JSON.parse(msg).name;
      }
    };
    //发送请求
    http.open("GET", "/data");
    http.send();
  }
</script>
```

你们可以自己尝试一下，在 chrome 浏览器中，这个接口是正常显示的，被调用了三次，3 次也都返回了服务器处理的结果。
![img](2.jpg)
![img](3.jpg)
![img](4.jpg)
![img](5.jpg)
而在 safari 浏览器中，却被做了额外的「优化」处理
![img](6.jpg)
如果仔细找一下会发现，其实从 2012 年开始，几乎每年都有人在网上问 Safari cache POST 请求和 Safari cache GET requests with cache disabled 的问题。

知道了这个 bug 背后的原因，修复方法其实不难。既然倔强的 safari 不改，那就只能在调用接口的时候主动做一下兼容了，也就是上传的时候带一些随机的参数，比如时间戳、hash 等等。

### 幂等性的作用

再回到前面的幂等性上，幂等性的作用是什么？
幂等性主要保证**多次调用对资源的影响是一致的**。

在阐述作用之前，我们利用资源处理应用来说明一下：

HTTP 与数据库的 CRUD 操作对应：

- PUT ：CREATE
- GET ：READ
- POST ：UPDATE
- DELETE ：DELETE

1. 查询

```sql
SELECT * FROM users WHERE xxx;
```

不会对数据产生任何变化，天然具备幂等性。

2. 新增

```sql
INSERT INTO users (user_id, name) VALUES (1, 'zhangsan');
```

case1：带有唯一索引（如：`user_id`），重复插入会导致后续执行失败，具有幂等性；

case2：不带有唯一索引，多次插入会导致数据重复，不具有幂等性。

3. 修改

case1：直接赋值，不管执行多少次 score 都一样，具备幂等性。

```sql
UPDATE users SET score = 30 WHERE user_id = 1;
```

case2：计算赋值，每次操作 score 数据都不一样，不具备幂等性。

```sql
UPDATE users SET score = score + 30 WHERE user_id = 1; 4. 删除
```

case1：绝对值删除，重复多次结果一样，具备幂等性。

```sql
DELETE FROM users WHERE id = 1;
```

case2：相对值删除，重复多次结果不一致，不具备幂等性。

```sql
DELETE top(3) FROM users;
```

总结：通常只需要对写请求（新增&更新）作幂等性保证。

### 如何解决幂等性问题

我们在网上搜索幂等性问题的解决方案，会有各种各样的解法，但是如何判断哪种解决方案对于自己的业务场景是最优解，这种情况下，就需要我们抓问题本质。

经过以上分析，我们得到了解决幂等性问题就是**要控制对资源的写操作**。

我们从问题各个环节流程来分析解决：
![img](7.jpeg)

#### 控制重复请求

控制动作触发源头，即前端做幂等性控制实现

> 相对不太可靠，没有从根本上解决问题，仅算作辅助解决方案。

主要解决方案：

- 控制操作次数，例如：提交按钮仅可操作一次（提交动作后按钮置灰）
- 及时重定向，例如：下单/支付成功后跳转到成功提示页面，这样消除了浏览器前进或后退造成的重复提交问题。

#### 过滤重复动作

控制过滤重复动作，是指在动作流转过程中控制有效请求数量。

1. 分布式锁

利用 Redis 记录当前处理的业务标识，当检测到没有此任务在处理中，就进入处理，否则判为重复请求，可做过滤处理。

> 订单发起支付请求，支付系统会去 Redis 缓存中查询是否存在该订单号的 Key，如果不存在，则向 Redis 增加 Key 为订单号。查询订单支付已经支付，如果没有则进行支付，支付完成后删除该订单号的 Key。通过 Redis 做到了分布式锁，只有这次订单订单支付请求完成，下次请求才能进来。

分布式锁相比去重表，将放并发做到了缓存中，较为高效。思路相同，同一时间只能完成一次支付请求。

2. token 令牌

应用流程如下：

- 服务端提供了发送 token 的接口。执行业务前先去获取 token，同时服务端会把 token 保存到 redis 中；
- 然后业务端发起业务请求时，把 token 一起携带过去，一般放在请求头部；
- 服务器判断 token 是否存在 redis 中，存在即第一次请求，可继续执行业务，执行业务完成后将 token 从 redis 中删除；
- 如果判断 token 不存在 redis 中，就表示是重复操作，直接返回重复标记给 client，这样就保证了业务代码不被重复执行。

应用流程如下：
![img](8.jpeg)

3. 缓冲队列

把所有请求都快速地接下来，对接入缓冲管道。后续使用异步任务处理管道中的数据，过滤掉重复的请求数据。
优点：同步转异步，实现高吞吐。
缺点：不能及时返回处理结果，需要后续监听处理结果的异步返回数据。
![img](9.jpeg)

#### 解决重复写

实现幂等性常见的方式有：悲观锁（for update）、乐观锁、唯一约束。

1. 悲观锁（Pessimistic Lock）

简单理解就是：假设每一次拿数据，都有认为会被修改，所以给数据库的行或表上锁。
当数据库执行 select for update 时会获取被 select 中的数据行的行锁，因此其他并发执行的 select for update 如果试图选中同一行则会发生排斥（需要等待行锁被释放），因此达到锁的效果。

> select for update 获取的行锁会在当前事务结束时自动释放，因此必须在事务中使用。（注意 for update 要用在索引上，不然会锁表）

```sql
START TRANSACTION; # 开启事务
SELETE * FROM users WHERE id=1 FOR UPDATE;
UPDATE users SET name= 'xiaoming' WHERE id = 1;
COMMIT; # 提交事务
```

2. 乐观锁（Optimistic Lock）

简单理解就是：就是很乐观，每次去拿数据的时候都认为别人不会修改。更新时如果 version 变化了，更新不会成功。
不过，乐观锁存在失效的情况，就是常说的 ABA 问题，不过如果 version 版本一直是自增的就不会出现 ABA 的情况。

```sql
UPDATE users SET name='xiaoxiao', version=(version+1) WHERE id=1 AND version=version;
```

缺点：就是在操作业务前，需要先查询出当前的 version 版本
另外，还存在一种：状态机控制
例如：支付状态流转流程：待支付->支付中->已支付
具有一定要的前置要求的，严格来讲，也属于乐观锁的一种。

3. 唯一约束

常见的就是利用数据库唯一索引或者全局业务唯一标识（如：source+序列号等）。
这个机制是利用了数据库的主键唯一约束的特性，解决了在 insert 场景时幂等问题。但主键的要求不是自增的主键，这样就需要业务生成全局唯一的主键。

全局 ID 生成方案：

- UUID：结合机器的网卡、当地时间、一个随记数来生成 UUID；
- 数据库自增 ID：使用数据库的 id 自增策略，如 MySQL 的 auto_increment。
- Redis 实现：通过提供像 INCR 和 INCRBY 这样的自增原子命令，保证生成的 ID 肯定是唯一有序的。
- 雪花算法-Snowflake：由 Twitter 开源的分布式 ID 生成算法，以划分命名空间的方式将 64-bit 位分割成多个部分，每个部分代表不同的含义。

小结：按照应用上的最优收益，推荐排序为：乐观锁 > 唯一约束 > 悲观锁。

### 总结

1. 幂等性处理 虽然复杂了业务处理，也可能会降低接口的执行效率，但是为了保证系统数据的准确性，是非常有必要的；
2. 遇到问题，善于发现并挖掘本质问题，这样解决起来才能高效且精准；
3. 选择自身业务场景适合的解决方案，而不要去硬套一些现成的技术实现，无论是组合还是创新，要记住适合的才是最好的。

### 参考文献

[MDN - 幂等](https://developer.mozilla.org/zh-CN/docs/Glossary/%E5%B9%82%E7%AD%89)
[rfc7231](https://tools.ietf.org/html/rfc7231#section-4.2.2)
