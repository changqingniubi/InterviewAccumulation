## 缓存定义

缓存是一种保存资源副本并在下次请求时直接使用该副本的技术。


## 从一次完整的HTTP请求看缓存分类

-  客户端缓存存在于客户端本地，如果客户端命中缓存，甚至不需要发起HTTP请求，相当于直接在本地拿数据。一般在这里的缓存技术有Cookie,localstorage,sessionstorage,本地文件或本地数据,HTTP头相关设置。比如cache-control，last-modified等参数。本地缓存的一个缺点就是刷新机制不会特别友好，所以一般缓存对数据实时性要求不高的，或者和客户端有关的数据。客户端缓存稍后详细讲解。
-  如果没有部署CDN或者CDN没有命中，请求最终才会落入应用服务器，现在的http服务器都会添加一层反向代理，例如nginx，在这一层同样会添加缓存层，代表技术是squid，varnish，当然nginx作为http服务器而言支持静态文件访问和本地缓存技术，也可以使用远程缓存。

- 如果前面的缓存机制全部失效，请求才会落入真正的服务器节点。
- 数据库作为一个应用程序，又是io密集型操作，自然少不了缓存的使用，常见的数据库MySQL，PosgreSQL都有自身定制的查询缓存，并且可以做到强一致。
- 如果数据库缓存也没有命中，操作系统在磁盘io层面还有一级缓存。cache和buffer，这两个分别是写入和读取使用的缓存空间。

## 总结
- 客户端缓存。
    - 浏览器缓存
- 服务器端缓存
    - CDN缓存
    - 代理服务器缓存
- 数据库缓存。


## 缓存的优点
- 缓解服务器压力。不用每次请求都去找服务器。
- 提升性能，在本地拿数据远比在服务器拿数据更快。
- 减少带宽消耗。

## 浏览器缓存详解

在讲浏览器缓存之前先抛出几个问题。

- 当首次请求时,返回的响应报文中，我们怎么知道响应报文的内容需不需要缓存在本地？
- 万一本地的缓存过期了，不是最新的怎么办？
- 过期了，浏览器如何操作才能拿到最新的数据？



**以上的问题都是依靠服务端返回的响应报文的响应头来控制。**

## 浏览器缓存存储在哪儿
- memory cache。
   将资源缓存到内存中，等待下次访问时不需要重新下载资源，而是直接从内存获取。
- disk cache。
 将资源存在磁盘中。直接操作对象为CurlCacheManager
 
 ## 浏览器怎么处理缓存的一系列操作的？
 
强缓存：不会向服务器发送请求，直接从缓存中读取资源。当Cache-Control和Expires没过期，则命中强缓存。

协商缓存：强制缓存失效后，浏览器携带缓存标识向服务器发起请求，由服务器根据缓存标识决定是否使用缓存的过程。


浏览器根据第一次请求资源时返回的响应头来处理缓存

### 浏览器第一次请求时

![1](
https://raw.githubusercontent.com/changqingniubi/InterviewAccumulation/master/2.webstorage/image/1.png)

- 浏览器发起请求。
- 服务端返回响应报文。

  - Cache-Control。决定是否缓存/缓存过期时间。HTTP1.1。
  - Expires。HTTP1.0规定了缓存过期的一个绝对时间。
  - ETag。根据实体内容生成一段hash字符串，标识资源的状态，由服务端产生。只要资源有变化，Etag就会重新生成。
  - Last-Modified。服务器端文件的最后修改时间。


### 浏览器后续请求时
![1](
https://raw.githubusercontent.com/changqingniubi/InterviewAccumulation/master/2.webstorage/image/2.png)
- 浏览器在请求某一资源时，根据本地缓存资源的header信息判断是否命中强缓存。
    - 看Cache-Control是否过期。
    - 看Expires是否过期。为Cache-Control的补充。当不支持Cache-Control的时候会用到。

- 若没过期，命中强缓存，直接从缓存拿数据。本次请求不会与服务器进行通信。
- 若过期，浏览器发送请求到服务器。此时，请求会携带第一次请求返回的有关缓存的header字段信息（Last-Modified/If-Modified-Since和Etag/If-None-Match）
- 服务器分析传过来的ETag/If-None-Match看资源是否更新。
    
    - 如果资源并未更新，返回状态码304。浏览器从缓存中取数据。
    - 如果资源更新了，返回状态码200。服务器给出最新的资源返回给浏览器。

- 如果ETag为否，根据Last-Modified/If-Modified-Since的值与服务器中这个资源的最后修改时间对比。
   - 如果时间没有变化，返回304。浏览器从缓存中取数据。
   -  如果If-Modified-Since的时间小于服务器中这个资源的最后修改时间。说明文件有更新，于是返回200和新的资源文件。

### 总结
当浏览器再次访问一个已经访问过的资源时，

- 看资源是否过期，没过期则命中强缓存；
- 若没命中，进入协商缓存环节。
- 看本地缓存资源是否是最新的，是则返回304，让浏览器去缓存拿数据。
- 若本地的缓存资源不是最新的，返回200，同时返回最新的资源。

### 有Last-Modified为什么要有Etag

协商缓存中有两种方法来判断本地缓存是否是最新的。

- Last-Modified/If-Modified-Since。依据是看文件的更新时间
- ETag/If-None-Match。判断依据是根据内容是根据内容。


HTTP1.1的Etag的出现是为了解决Last-Modified/If-Modified-Since的缺点。

- 某些服务器不能精确的得到文件的最后修改时间。
- 某些文件修改非常频繁（秒为单位）。若采用第一种方法，那么缓存就没什么意义了，反正也要重新获取资源。
- 一些文件也许会周期性的更改，但是他的内容并不改变。（仅仅改变的修改时间）


### localstorage/sessionstorage

使用HTML5可以在本地存储用户的浏览数据。早些时候,本地存储使用的是 cookie。WebStorage的目的是为了克服由cookie带来的一些限制（Cookie只有4k）。WebStorage又分为两种： sessionStorage 和localStorage ，即这两个是Storage的一个实例。

#### 相同点
- localStorage和sessionStorage一样用来存储客户端临时信息的对象。
- 两者都只能存储字符串类型的对象。
- 存放数据大小为一般为5MB
- localStorage/sessionStorage不会参与与服务器的通信


它主要被用于储存一些不经常改动的，不敏感的数据，比如全国省市区县信息。还可以存储一些不太重要的跟用户相关的数据，比如说用户的头像地址、主题颜色等，这些信息可以先存储在用户本地一份，便于快速呈现，等真正从服务器端读取成功后再更改头像地址，主题颜色。

```js
//localStorage 和 sessionStorage 有着统一的API接口，这为二者的操作提供了极大的便利。
//下面以 localStorage 为例来介绍一下 API 接口使用方法，同样这些接口也适用于 sessionStorage。

添加：
localStorage.setItem('name', 'lilei');
localStorage.name = 'lilei';
localStorage.setItem('user', JSON.stringify(id:1, name:'lilei'));
//当我们要存储对象是，应先转换成我们可识别的字符串格式（比如JSON格式）再进行存储。
--------------------------------------------------
获取：
var name = localStorage.getItem('name');
var name = localStorage.name;
var name = localStorage['name'];
var user = JSON.parse(localStorage.getItem('user'));
-----------------------------------------
删除removeItem()：
var name = localStorage.getItem('name'); // 'lilei'
localStorage.removeItem('name');
name = localStorage.getItem('name'); // null
--------------------------------------------
清除所有clear()：
localStorage.clear();
var len = localStorage.length; // 0
-------------------------------------------
获取键值key(index):
localStorage.setItem('name','lilei');
var key = localStorage.key(0); // 'name'
localStorage.setItem('age', 10);
key = localStorage.key(0); // 'age'
key = localStorage.key(1); // 'name'
------------------------------------

```
#### 区别

##### localStorage

- **生命周期为永久。**意味着除非用户显示在浏览器提供的UI上清除localStorage信息，否则这些信息将永远存在。
- 相同浏览器的不同页面间可以共享相同的 localStorage。**前提是页面属于相同域名和端口**。

##### sessionStorage
- **生命周期为当前窗口或标签页。**一旦窗口或标签页被永久关闭了，那么所有通过sessionStorage存储的数据也就被清空了。
- 不同页面或标签页间无法共享sessionStorage的信息。

这里需要注意的是，页面及标 签页仅指顶级窗口，如果一个标签页包含多个iframe标签且他们属于同源页面，那么他们之间是可以共享sessionStorage的。