## HTTP  1.0 1.1 2.0---20180903
### 问题驱动
#### 为什么说HTTP是无状态的？
HTTP 是一种不保存状态，无状态（stateless）协议。HTTP 协议自身不对请求和响应之间的通信状态进行保存。也就是说在 HTTP 这个级别，协议对于发送过的请求或响应都不做持久化处理。使用 HTTP 协议，每当有新的请求发送时，就会有对应的新响应产生。协议本身并不保留之前一切的请求或响应报文的信息。这是为了更快地处理大量事务，确保协议的可伸缩性，而特意把 HTTP 协议设计成如此简单的。

随着 Web 的不断发展，无状态而导致业务处理变得棘手的情况增多了。比如，用户登录到一家购物网站，即使他跳转到该站的其他页面后，也需要能继续保持登录状态。针对这个实例，网站为了能够掌握是谁送出的请求，需要保存用户的状态。
HTTP/1.1 虽然是无状态协议，但为了实现期望的保持状态功能，于是引入了 Cookie 技术。有了 Cookie 再用 HTTP 协议通信，就可以管理状态了。

一句话通俗解释无状态（摘自知乎）：就是第二次来你无法识别它曾经来过。

#### 什么是Cookie？
Cookie 技术通过在请求和响应报文中写入 Cookie 信息来控制客户端的状态。Cookie 会根据从服务器端发送的响应报文内的一个叫做 Set-Cookie 的首部字段信息，通知客户端保存 Cookie。当下次客户端再往该服务器发送请求时，客户端会自动在请求报文中加入 Cookie 值后发送出去。服务器端发现客户端发送过来的 Cookie 后，会去检查究竟是从哪一个客户端发来的连接请求，然后对比服务器上的记录，最后得到之前的状态信息。

现在可以看知乎上另一个比喻：

有状态：

A：你今天中午吃的啥？

B：吃的大盘鸡。

A：味道怎么样呀？

B：还不错，挺好吃的。

无状态：

A：你今天中午吃的啥？

B：吃的大盘鸡。

A：味道怎么样呀？

B：？？？啊？啥？啥味道怎么样？

所以需要cookie这种东西：

A：你今天中午吃的啥？

B：吃的大盘鸡。

A：你今天中午吃的大盘鸡味道怎么样呀？

B：还不错，挺好吃的。

#### 从0.9到2.0（知乎）
* HTTP/0.9时代：短连接

每个HTTP请求都要经历一次DNS解析、三次握手、传输和四次挥手。反复创建和断开TCP连接的开销巨大，在现在看来，这种传输方式简直是糟糕透顶。

* HTTP/1.0时代：持久连接（长连接）

概念提出人们认识到短连接的弊端，提出了持久连接的概念，在HTTP/1.0中得到了初步的支持。

持久连接，即一个TCP连接服务多次请求：

客户端在请求header中携带Connection: Keep-Alive，即是在向服务端请求持久连接。如果服务端接受持久连接，则会在响应header中同样携带Connection: Keep-Alive，这样客户端便会继续使用同一个TCP连接发送接下来的若干请求。（Keep-Alive的默认参数是[timout=5, max=100]，timeout表示一个响应结束后，再过5s没有请求就断开。max表示TCP连接内可连续发送的最大请求数。）

当服务端主动切断一个持久连接时（或服务端不支持持久连接），则会在header中携带Connection: Close，要求客户端停止使用这一连接。

![v2-755722cfb502cebbe639bc311270eb47_hd](https://user-images.githubusercontent.com/6982311/45009474-ed434280-b03a-11e8-8060-da91f4880690.png)

HTTP/1.1时代：持久连接成为默认的连接方式；提出pipelining概念

HTTP/1.1开始，即使请求header中没有携带Connection: Keep-Alive，传输也会默认以持久连接的方式进行。目前所有的浏览器都默认请求持久连接，几乎所有的HTTP服务端也都默认开启对持久连接的支持，短连接正式成为过去式。（HTTP/1.1的发布时间是1997年，最后一次对协议的补充是在1999年，我们可以夸张地说：HTTP短连接这个概念已经过时了近20年了。）同时，持久连接的弊端被提出 —— HOLB（Head of Line Blocking）即持久连接下一个连接中的请求仍然是串行的，如果某个请求出现网络阻塞等问题，会导致同一条连接上的后续请求被阻塞。所以HTTP/1.1中提出了pipelining概念，即客户端可以在一个请求发送完成后不等待响应便直接发起第二个请求，服务端在返回响应时会按请求到达的顺序依次返回，这样就极大地降低了延迟。

![v2-d254b4a421391fc3c2178ffbc0b023a1_hd](https://user-images.githubusercontent.com/6982311/45009568-6f336b80-b03b-11e8-9582-763798e44603.png)

然而pipelining并没有彻底解决HOLB（线头阻塞（Head of line blocking）简称：HOLB），为了让同一个连接中的多个响应能够和多个请求匹配上，响应仍然是按请求的顺序串行返回的。同理，pipelining还有请求的Front of queue blocking问题。所以pipelining并没有被广泛接受，几乎所有代理服务都不支持pipelining，部分浏览器不支持pipelining，支持的大部分也会将其默认关闭。

SPDY和HTTP/2：multiplexing

multiplexing即多路复用，在SPDY中提出，同时也在HTTP/2中实现。multiplexing技术能够让多个请求和响应的传输完全混杂在一起进行，通过streamId来互相区别。这彻底解决了holb问题，同时还允许给每个请求设置优先级，服务端会先响应优先级高的请求。

![v2-b99ceeeb044581939a9964b5d3ce1a88_hd](https://user-images.githubusercontent.com/6982311/45009654-fa146600-b03b-11e8-8f51-335182eb69b6.png)


现在Chrome、FireFox、Opera、IE、Safari的最新版本都支持SPDY，Nginx/Apache HTTPD/Jetty/Tomcat等服务端也都提供了对SPDY的支持。另外，谷歌已经关闭SPDY项目，正式为HTTP/2让路。可以认为SPDY是HTTP/2的前身和探路者。

#### 什么是TCP慢启动（slow start）？
  
http1.0连接无法复用直接导致每次请求都需要经历三次握手和慢启动，三次握手在高延迟下影响效果非常明显，慢启动则对大文件类请求影响较大，尽管可以通过设置 Connection:Keep-Alive 来复用短时间内的连接，但依然处理不了时间跨度比较大的请求。 

什么是TCP的慢启动（slow start）呢？最初的TCP在连接建立成功后会向网络中发送大量的数据包，这样很容易导致网络中路由器缓存空间耗尽，从而发生拥塞。因此新建立的连接不能够一开始就大量发送数据包，而只能根据网络情况逐步增加每次发送的数据量，以避免上述现象的发生。

#### http 2.0 主要还做了哪些优化？
* 多路复用 (Multiplexing)

* 二进制分帧。即应用层(HTTP)和传输层(TCP or UDP)之间增加一个二进制分帧层。

* 首部压缩（Header Compression）

* 服务端推送（Server Push）服务端推送是一种在客户端请求之前发送数据的机制。在 HTTP/2 中，服务器可以对客户端的一个请求发送多个响应。

更多http2.0学习见另一篇md。
