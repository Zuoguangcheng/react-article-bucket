#### 前言
本部分主要说明一些前端计算机网络相关基础知识，后续会不断补充。文中部分内容可能来源于其他文章，所以你也可以参考文末的参考资料。

#### 1.HSTS与前端http/https共存
上次遇到一个问题:页面为https的，同时iframe嵌套了别人的一个https的页面，但是这个别人的https页面引入了http的资源，所以浏览器会出现页面加载了不安全脚本的警告(别人的页面是给内网用户用的，所以他们不关心这个问题)。这在页面给外网用户使用的情况下就存在很大的用户体验问题。所以尝试了解了这个概念。

##### 1.1 什么是HSTS?
下面是对网上内容的做的一个总结:

1.HSTS（HTTP Strict Transport Security）的作用是`强制客户端`（如浏览器）使用HTTPS与服务器创建连接。服务器开启HSTS的方法是，当客户端通过HTTPS发出请求时，在服务器返回的超文本传输协议响应头中包含Strict-Transport-Security字段。非加密传输时设置的HSTS字段无效。 比如，https://xxx 的响应头含有如下的头:
```text
Strict-Transport-Security: max-age=31536000; includeSubDomains
```
这意味着两点:
- 强制升级为https
  在接下来的一年（即31536000秒）中，浏览器只要向xxx或其子域名发送HTTP请求时，必须采用HTTPS来发起连接。比如，用户点击超链接或在地址栏输入http://xxx/ ，浏览器应当自动将http 转写成https，然后直接向https://xxx/发送请求 (**注意**:即使使用window.open打开http的网站也会被转化为https)。
- 无法忽略证书无效的警告
  在接下来的一年中，如果xxx服务器发送的TLS证书无效，用户不能忽略浏览器警告继续访问网站

2.HSTS可以用来抵御`SSL剥离攻击(顾名思义就是剥离http到https的转化)`。SSL剥离的实施方法是阻止浏览器与服务器创建HTTPS连接。它的**前提**是用户很少直接在地址栏输入https://， 用户总是通过点击链接或3xx重定向，从HTTP页面进入HTTPS页面。所以攻击者可以在用户访问HTTP页面时替换所有https:// 开头的链接为 http:// ，达到阻止HTTPS的目的。HSTS可以很大程度上解决SSL剥离攻击，因为只要浏览器曾经与服务器创建过一次安全连接，之后浏览器会强制使用HTTPS，即使链接被换成了HTTP。另外，如果中间人使用自己的自签名证书来进行攻击，浏览器会给出警告，但是许多用户会忽略警告。HSTS解决了这一问题，一旦服务器发送了HSTS字段，用户将不再允许忽略警告

3.用户**首次访问**某网站是不受HSTS保护的。这是因为首次访问时，浏览器还`未收到`HSTS，所以仍有可能通过明文HTTP来访问。解决这个不足目前有两种方案，一是`浏览器预置HSTS域名列表`，Google Chrome、Firefox、Internet Explorer和Spartan实现了这一方案。二是将HSTS信息加入到`域名系统记录`中。但这需要保证DNS的安全性，也就是需要部署域名系统安全扩展。截至2014年这一方案没有大规模部署

##### 1.2 HTTPS与HTTP共存的两种类型?
主要包括主动和被动两种混合内容。
- [主动混合内容](https://developers.google.com/web/fundamentals/security/prevent-mixed-content/fixing-mixed-content?hl=zh-cn)
  是最危险的混合内容，浏览器会自动和完全阻止掉这部分内容。对于那些能够**修改当前页面DOM**的内容都被称为主动混合内容，比如script,link,iframe,object标签，css选择器中使用的url(如background),或者常见的XMLHTTPRequest。这些内容能够读取用户的cookie并获取认证。chrome中针对**主动混合**内容的处理如下:

  ![](./images/content-mix.png)

  如果页面出现了上面的主动混合内容，chrome将会直接抛出如下的错误信息:

  ![](./images/passive.png)

  我们注意到:img srcset里的资源也是默认会被阻止的，即下面的img会被block：
  ```html
  <img srcset="http://fedren.com/test-1x.png 1x, http://fedren.com/test-2x.png 2x" alt>
  ```
  但是使用src的不会被block:
  ```html
  <img src="http://fedren.com/images/sell/icon-home.png" alt>
  ```

- [被动混合内容](https://www.w3.org/TR/upgrade-insecure-requests/#recommendations)
  除了主动混合内容以外就是被动混合内容。浏览器对于这部分内容的处理策略是允许加载，但是会弹出一个警告。比如:images/audio/video/favicon等，他们虽然在页面中，但是无法修改当前页面的DOM。chrome中针对被动混合内容的处理如下:
  ```c
    // "Optionally-blockable" mixed content
    case WebURLRequest::kRequestContextAudio:
    case WebURLRequest::kRequestContextFavicon:
    case WebURLRequest::kRequestContextImage:
    case WebURLRequest::kRequestContextVideo:
      return WebMixedContentContextType::kOptionallyBlockable;
  ```
如果页面中出现了这部分的混合内容，将会出现如下的警告:

![](./images/active.png)

主动混合内容能够拦截http的请求，然后使用它们自己的内容来替换本来的内容。[这里](https://blog.cloudflare.com/fixing-the-mixed-content-problem-with-automatic-https-rewrites/)也提供了多个使用主动混合内容对网站攻击的例子。注意：**a标签**不会导致mix content问题,因为它们使浏览器导航到新页面。 这意味着它们通常不需要修正,但是如果a标签用于懒加载的情况是特例:

```html
<a class="gallery" href="http://googlesamples.github.io/web-fundamentals/samples/discovery-and-distribution/avoid-mixed-content/puppy.jpg">
  <img src="https://googlesamples.github.io/web-fundamentals/samples/discovery-and-distribution/avoid-mixed-content/puppy-thumb.jpg">
</a>
```
上面的例子，img标签默认加载的是缩略图，而当页面真正出现在视口中的时候会使用a标签的href值替换img的src，这样就会存在mix content的问题。这个例子告诉我们:页面onload的时候可能并没有mix content问题，但是随着网页中各种操作点击将会产生动态加载资源的情况，这也是会引起mix content的!

对于被动混合内容，如果设置**strick mode**,如下:
```html
<meta http-equiv="Content-Security-Policy" content="block-all-mixed-content">
```
那么也会被block掉。
```c
case WebMixedContentContextType::kOptionallyBlockable:
    allowed = !strict_mode;
    // 如果页面是strict-mode,是不能允许加载mix content的
    if (allowed) {
      content_settings_client->PassiveInsecureContentFound(url);
      client->DidDisplayInsecureContent();
    }
    break;
```
上面代码，如果strick_mode是true(就是CSP设置为block-all-mixed-content)，allowed就是false，被动混合内容就会被阻止。而对于主动混合内容，如果用户设置允许加载(一般都会**浏览器询问**):

![](./images/allow.png)

那么也是可以加载的:
```c
 // Strictly block subresources that are mixed with respect to their
  // subframes, unless all insecure content is allowed. This is to avoid the
  // following situation: https://a.com embeds https://b.com, which loads a
  // script over insecure HTTP. The user opts to allow the insecure content,
  // thinking that they are allowing an insecure script to run on
  // https://a.com and not realizing that they are in fact allowing an
  // insecure script on https://b.com.
  bool should_ask_embedder =
      !strict_mode && settings &&
      (!settings->GetStrictlyBlockBlockableMixedContent() ||
       settings->GetAllowRunningOfInsecureContent());
  allowed = should_ask_embedder &&
  // 该行代码表示当前client是否允许加载blockable的资源
            content_settings_client->AllowRunningInsecureContent(
                settings && settings->GetAllowRunningOfInsecureContent(),
                security_origin, url);
  break;
```
代码倒数第4行会去判断当前的client即当前页面的设置`是否允许`加载blockable的资源。另外源码注释还提到了一种特殊的情况，就是 https:\/\/a.com 的页面包含了 https:\/\/b.com 的页面， https:\/\/b.com 允许加载blockable(**主动混合**)的资源，https:\/\/a.com在非strick mode的时候页面是允许加载的，但是如果a.com是strick mode(**block-all-mixed-content**)，那么将不允许加载。

并且如果页面设置了strick mode，用户设置的允许blockable资源加载的设置将会失效：
```c
// If we're in strict mode, we'll automagically fail everything, and
  // intentionally skip the client checks in order to prevent degrading the
  // site's security UI.
  bool strict_mode =
      mixed_frame->GetSecurityContext()->GetInsecureRequestPolicy() &
          kBlockAllMixedContent ||
      settings->GetStrictMixedContentChecking();
```
这种方式就是1.3章节我最后采用的一种方式，即必须让用户`手动点击`"允许加载不安全脚本"!

##### 1.3 我是如何解决HTTP与HTTPS共存问题的?
下面讲解下我是如何解决https/http共存的问题的(react-router单页应用),(我遇到的问题就是**主动混合内容**，因为采用的是iframe嵌套别人的网页，该网页可以修改当前页面的DOM。别人https网页嵌入的http资源是image,所以是**被动混合内容**，不会直接被浏览器拦截掉),方案如下:

- 方案一
1.用户点击某一个按钮需要iframe打开别人的含有http链接的https页面时候，我使用window.open打开,代码如下:
```js
const url = `http://${window.location.host}/#/createCrowd?accountId=${oriId}&fansMust=${defaultSubscribe}`;
window.open(url, "_self", "", true);
```
此时url被设置为当前我们域名下的http版本的URL(本域名支持https/http两种请求协议),其中createCrowd这个路由会通过iframe嵌套别人的https协议的网页，但是因为createCrowd是http协议打开的，所以它本身可以打开https的iframe，同时该iframe也可以加载http的资源。而用户在iframe的页面中操作完成的时候跳转到我们的一个页面(本身的逻辑就是这样的，因为是修改别人的代码，没有想过修改这种模式)，比如/createNewPeopleGroup，此时如果用户在这个页面中点击了完成按钮，那么回到我们页面的https版本:
```js
back2https = () => {
    const url = `https://${window.location.host}/#/usersManager`;
    // 回到最初的页面URL的https版本
    //usersManager(https)=>createCrowd(http)=>iframe(https)=>/createNewPeopleGroup(https)=>usersManager(https)
    setTimeout(() => {
      window.parent.location.href = url;
    }, 50);
};
```
这种逻辑貌似很完美，但是可能由于上面的HSTS，当你使用window.open打开自己网站的http版本的时候却被chrome浏览器强制定向到https版本，所以这个方案就是无效的。于是有了方案2:

- 方案二
只需要在页面的html模板中添加了下面的meta标签即可([网上](https://stackoverflow.com/questions/34909224/http-to-https-mixed-content-issue)有说这个方案不能通过meta添加，我在本地尝试的时候是可以的(可能是由于本地的https证书问题)，但是`发送到服务端后`确实不可以)。
```html
<meta http-equiv="Content-Security-Policy" content="upgrade-insecure-requests" />
<!-- 该指令指示浏览器在进行网络请求之前升级不安全的网址-->
<!-- upgrade-insecure-requests 指令级联到 <iframe> 文档中，从而确保整个页面受到保护-->
```
通过在单页应用的html模板中添加这个http头，整个网站的不安全资源全部转化为https了，当然，这个方案需要保证所有警告的资源的https版本是存在的才行。但是这种方式由于服务端目前没有尝试，所以我只能说以后如果有尝试了再更新。

- 方案三
  此时就是1.2章节说到的特例。https:\/\/a.com 的页面包含了 https:\/\/b.com 的页面， https:\/\/b.com 允许加载blockable(**主动混合**)的资源，https:\/\/a.com在非strick mode的时候页面是允许加载的，但是如果a.com是strick mode(**block-all-mixed-content**)，那么将不允许加载。但是这种方案有一个问题：就是用户必须手动点击"允许加载不安全脚本"，这也是我最后采用的解决方案。

#### 2.CRL证书吊销列表
证书具有一个指定的寿命，但CA可通过称为**证书吊销**的过程来缩短这一寿命。CA发布一个证书吊销列表 (CRL,即Certificate Revocation List)，列出被认为不能再使用的证书的序列号。

#### 3.OCSP在线证书状态协议
OCSP(Online Certificate Status Protocol，在线证书状态协议)是维护`服务器`和其它`网络资源安全性`的两种普遍模式之一。OCSP克服了证书注销列表（CRL）的主要缺陷：必须**经常在客户端下载**以确保列表的更新。当用户试图访问一个服务器时，**在线证书状态协议**发送一个对于证书状态信息的请求。服务器回复一个“有效”、“过期”或“未知”的响应。协议规定了服务器和客户端应用程序的通讯语法。在线证书状态协议给了用户的到期的证书一个宽限期，这样他们就可以在更新以前的一段时间内继续访问服务器。Chrome默认关闭了ocsp功能，firefox 和 IE 都默认开启。

#### 4.HTTP三次握手协议
- 第一次握手
  主机A发送位码为syn＝1,随机产生seq number=1234567的数据包到服务器，主机B由**SYN=1**知道，A要求建立联机；

- 第二次握手
  主机B收到请求后要确认联机信息，向A发送ack number=(主机A的seq+1),syn=1,ack=1,随机产生seq=7654321的包

- 第三次握手
  主机A收到后检查ack number是否正确，即第一次发送的seq number+1,以及位码ack是否为1，若正确，主机A会再发送ack number=(主机B的seq+1),ack=1，主机B收到后确认seq值与ack=1则**连接建立成功**。

完成三次握手，主机A与主机B开始传送数据。为什么建立连接是三次握手，而关闭连接却是四次挥手呢？
  这是因为**服务端在**LISTEN状态下，收到建立连接请求的SYN报文后，把ACK和SYN放在一个报文里发送给客户端。而关闭连接时，当收到对方的FIN报文时，仅仅表示对方不再发送数据了但是还能接收数据，己方也未必全部数据都发送给对方了，所以己方可以立即close，也可以发送一些数据给对方后，再发送FIN报文给对方来表示同意现在关闭连接，因此，己方ACK和FIN`一般都会分开`发送。

那么TCP的为什么需要三次握手？最主要是防止已过期的连接再次传到被连接的主机。如果采用两次的话，会出现下面这种情况。

- 多余连接
  
  比如是A机要连到B机，结果发送的连接信息由于某种原因没有到达B机；于是，A机又发了一次，结果这次B收到了，于是就发信息回来，两机就连接。传完东西后，断开。结果这时候，原先没有到达的连接信息突然又传到了B机，于是B机发信息给A，然后B机就以为和A连上了，这个时候B机就在等待A传东西过去。

- 死锁会发生
  
  三次握手改成仅需要两次握手，死锁是可能发生。考虑计算机A和B之间的通信，假定B给A发送一个连接请求分组，A收到了这个分组，并发送了确认应答分组。按照两次握手的协定，A认为连接已经成功地建立了，可以开始发送数据分组。可是，B在A的应答分组在传输中被丢失的情况下，将不知道A是否已准备好，不知道A建议什么样的序列号，B甚至怀疑A是否收到自己的连接请求分组。在这种情况下，B认为连接还未建立成功，将忽略A发来的任何数据分组，只等待连接确认应答分组。而A在发出的分组超时后，重复发送同样的分组。这样就形成了死锁。

#### 5.公钥与私钥非对称加密与SSH
**对称加密**:不管是自己还是别人解密已经加密后的数据都是使用相同的密码作为秘钥，这在很多情况下是很危险的。比如我的QQ密码，微信密码都是123，那么就会存在密码泄露的问题，因为我需要告诉对方我的加密密码他才能解密，这就是对称加密算法

**非对称加密**:利用自己的密码,通过**对称加密**(加密和解密都是同样的密码)生成公钥和私钥，并把公钥发送给第三方，第三方利用该公钥进行加密，然后当前方使用自己的私钥解密。但是，使用私钥解密的过程中是需要自己的私钥以及自己的密码的，更加形象的说应该是需要钥匙(私钥)+密码，因为你要打开的锁实际上是`密码锁`。

![](./images/ssh-theory.png)

比如屌丝需要将某些重要资源发送给高富帅，此时他将资源放在一个两边都能打开的桶中(羽毛球桶)，然后使用高富帅发送的公钥(锁)对数据进行加密，那么数据传送到高富帅的时候，其就可以通过自己的私钥对数据进行解密。当然，在桶的另一面，屌丝也可以使用自己的公钥对数据进行加密，这样能够保证这一端只能通过自己的私钥打开，其他人无法获取到数据，这是一个形象的比喻，实际的过程并不需要桶的两边都加密，只是说如果需要向谁发送数据就用谁的公钥进行加密而已。**同时私钥是由公钥决定的，但却不能根据公钥计算出私钥，这也是对称加密一个重要的前提**。

最后需要注意:SSH只能保证数据传递过程的安全，如果在传递之前传送方机器密码已经被木马记录了，那么数据即使已经加密，也是不安全的传输。当然，加密和解密也可以配合相应的数据签名，从而可以验证数据传输的完整性。下面是签名和验证的基本过程:

**签名和验证**:发送方用特殊的hash算法，由明文中产生固定长度的摘要，然后利用自己的**私钥**对形成的摘要进行加密，这个过程就叫签名。接受方利用发送方的公钥解密被加密的摘要得到结果A，然后对明文也进行hash操作(所以双方要协商摘要算法)产生摘要B。最后,把A和B作比较。此方式既可以保证发送方的身份不可抵赖，又可以保证数据在传输过程中不会被篡改。

#### 4.https与http页面访问速度影响
结论:HTTPS也会**降低**用户访问速度，**增加**网站服务器的计算资源消耗。主要体现在以下两个方面:

- 协议交互所增加的网络RTT(round trip time)
下面是http网络请求图:

![](./images/http.jpg)

 用户只需要完成TCP三次握手建立TCP连接就能够直接发送`HTTP请求`获取应用层数据，此外在整个访问过程中也没有需要消耗计算资源的地方。接下来看 HTTPS 的访问过程，相比 HTTP 要复杂很多，在部分场景下，使用 HTTPS 访问有可能增加 7 个 RTT

下面是https的网络请求图:

![](./images/https.jpg)

下面是对该图的一个全面解释:

<pre>
1:三次握手建立TCP连接。耗时一个RTT。

2:使用**HTTP**发起GET请求，服务端返回302跳转到https://www.baidu.com。需要一个RTT 以及302跳转延时。

   (a)大部分情况下用户不会手动输入 https://www.baidu.com 来访问 HTTPS，服务端只能返回 302 强制浏览器跳转到 https。

   (b)浏览器处理302 跳转也需要耗时。

3:三次握手重新建立TCP连接,耗时一个RTT。

  (a)302跳转到 HTTPS服务器之后，由于端口和服务器不同，需要重新完成三次握手，建立 TCP 连接。

4:TLS完全握手阶段一,耗时至少一个RTT。

 (a)这个阶段主要是完成`加密套件`的协商和`证书`的身份认证。

 (b)服务端和浏览器会协商出相同的`密钥交换`算法、`对称加密`算法、`内容一致性校验`算法、`证书签名`算法、椭圆曲线（非ECC 算法不需要）等。

 (c)浏览器获取到证书后需要`校验证书`的有效性，比如是否过期，是否撤销。

5:解析CA站点的DNS,耗时一个 RTT

  (a)浏览器获取到证书后，有可能需要发起OCSP或者CRL请求，查询证书状态。

  (b)浏览器首先获取证书里的CA域名。

  (c)如果没有命中缓存，浏览器需要解析CA 域名的 DNS。

6:三次握手建立CA站点的TCP连接,耗时一个RTT。

 (a)DNS 解析到IP后，需要完成三次握手建立 TCP 连接。

7:发起OCSP请求，获取响应,耗时一个 RTT。

8:完全握手阶段二，耗时一个RTT及计算时间。

 (a)完全握手阶段二主要是密钥协商

9:完全握手结束后，浏览器和服务器之间进行应用层（也就是HTTP）数据传输。

当然不是每个请求都需要增加 7 个 RTT 才能完成 HTTPS 首次请求交互。大概只有不到 0.01% 的请求才有可能需要经历上述步骤，它们需要满足
如下条件：

1:必须是首次请求。即建立TCP连接后发起的第一个请求，该连接上的后续请求都不需要再发生上述行为。

2:必须要发生完全握手，而正常情况下80%的请求能实现简化握手。

3:浏览器需要开启OCSP(Online Certificate Status Protocol)或者CRL功能。Chrome默认关闭了ocsp功能，firefox 和 IE 都默认开启。

4:浏览器没有命中OCSP缓存。Ocsp一般的更新周期是7天，firefox的查询周期也是7天，也就说是7 天中才会发生一次ocsp的查询。

5:浏览器没有命中CA站点的DNS缓存。只有没命中DN缓存的情况下才会解析CA的DNS。
</pre>

- 加解密相关的计算耗时
<pre>
(1)浏览器计算耗时

  a)RSA证书签名校验，浏览器需要解密签名，计算证书哈希值。如果有多个证书链，浏览器需要校验多个证书。

  b)RSA 密钥交换时，需要使用证书公钥加密premaster。耗时比较小，但如果手机性能比较差，可能也需要 1ms 的时间。

  c)ECC密钥交换时，需要计算椭圆曲线的公私钥。

  d)ECC密钥交换时，需要使用证书`公钥`解密获取服务端发过来的ECC公钥。

  e)ECC密钥交换时，需要根据服务端公钥计算master key。

  f)`应用层`数据对称加解密。

  g)应用层数据一致性校验。

2:服务端计算耗时

 a)RSA密钥交换时需要使用证书私钥解密`premaster`。这个过程非常消耗性能。
 
 b)ECC密钥交换时，需要计算椭圆曲线的公私钥。

 c)ECC密钥交换时，需要使用证书私钥加密ECC公钥。

 d)ECC密钥交换时，需要根据浏览器公钥计算共享的master key。

 e)应用层数据对称加解密。

 f)应用层数据一致性校验。
</pre>

#### 5.为什么前端的请求被设置为Options请求
这当前端调用接口存在**跨域**的情形下会出现，这种情况下建议使用**代理服务器**，而不是前端直接发送XHR或者fetch请求到另一个服务端去。此时在真实发送数据之前会通过Options方法进行[一次探测](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)，过程如下:

- 客户端发送请求方法以及请求头

  下面是Option的数据发送:
  ```text
  OPTIONS /resources/post-here/ HTTP/1.1
  Host: bar.other
  User-Agent: Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10.5; en-US; rv:1.9.1b3pre) Gecko/20081130 Minefield/3.1b3pre
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-us,en;q=0.5
  Accept-Encoding: gzip,deflate
  Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
  Connection: keep-alive
  Origin: http://foo.example
  Access-Control-Request-Method: POST
  Access-Control-Request-Headers: X-PINGOTHER, Content-Type
  ```
(1)Options方法是HTTP/1.1方法，用于从服务端进一步获取消息，是一个安全的方法，也意味着无法修改服务端的资源本身

(2)**Access-Control-Request-Method**表示当真实请求被发送的时候采用的是POST方法

(3)**Access-Control-Request-Headers**表示当真实的请求被发送的时候将会发送X-PINGOTHER, Content-Type两个自定义的请求头。

通过以上信息，服务器可以决定是否接受这个真实的请求。

- 服务端发送可以接受的方法以及请求头
  服务端返回的数据如下:
  ```text
  HTTP/1.1 200 OK
  Date: Mon, 01 Dec 2008 01:15:39 GMT
  Server: Apache/2.0.61 (Unix)
  Access-Control-Allow-Origin: http://foo.example
  Access-Control-Allow-Methods: POST, GET, OPTIONS
  <!-- 1.服务端接受POST, GET, OPTIONS用于客户端发送真实的请求 -->
  Access-Control-Allow-Headers: X-PINGOTHER, Content-Type
  <!-- 2.表示服务端可以接受这两个自定义的HTTP头 -->
  Access-Control-Max-Age: 86400
  <!-- 3.表示通过Options获取到的信息可以缓存多久而不用再次发送OPTIONS请求 -->
  Vary: Accept-Encoding, Origin
  Content-Encoding: gzip
  Content-Length: 0
  Keep-Alive: timeout=2, max=100
  Connection: Keep-Alive
  Content-Type: text/plain
  ```
- 发送真实的请求
  ```text
  POST /resources/post-here/ HTTP/1.1
  Host: bar.other
  User-Agent: Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10.5; en-US; rv:1.9.1b3pre) Gecko/20081130 Minefield/3.1b3pre
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-us,en;q=0.5
  Accept-Encoding: gzip,deflate
  Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
  Connection: keep-alive
  X-PINGOTHER: pingpong
  Content-Type: text/xml; charset=UTF-8
  <!-- 1.自定义的头部X-PINGOTHER和Content-Type -->
  Referer: http://foo.example/examples/preflightInvocation.html
  Content-Length: 55
  Origin: http://foo.example
  Pragma: no-cache
  Cache-Control: no-cache
  ```

还有一种可能是跨域情况，比如http:\/\/foo.example发送请求到http:\/\/bar.other，需要发送HTTP Cookie，此时可能需要写出如下的代码:
```js
var invocation = new XMLHttpRequest();
var url = 'http://bar.other/resources/credentialed-content/';
function callOtherDomain(){
  if(invocation) {
    invocation.open('GET', url, true);
    invocation.withCredentials = true;
    // 设置withCredentials
    // 默认情况下跨站点的XMLHttpRequest或者Fetch不会发送Cookie
    invocation.onreadystatechange = handler;
    invocation.send(); 
  }
}
```
而[Preflighted requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)满足的条件，你可以查看这个链接。而对于那些不满足特定条件的请求并不会通过一次的多余的OPTIONS请求来完成。



参考资料:

[百度百科HSTS](https://baike.baidu.com/item/HSTS/8665782?fr=aladdin)

[Fixing the mixed content problem with Automatic HTTPS Rewrites](https://blog.cloudflare.com/fixing-the-mixed-content-problem-with-automatic-https-rewrites/)

[Upgrade Insecure Requests Sample](https://googlechrome.github.io/samples/csp-upgrade-insecure-requests/index.html)

[HTTP与HTTPS对访问速度（性能）的影响](https://www.cnblogs.com/mylanguage/p/5635524.html)

[百度百科ocsp](https://baike.baidu.com/item/ocsp/283332?fr=aladdin)

[那些年我准备的前端面试题集合](http://blog.csdn.net/liangklfang/article/details/50436536)

[非对称加密算法](https://baike.baidu.com/item/%E9%9D%9E%E5%AF%B9%E7%A7%B0%E5%8A%A0%E5%AF%86%E7%AE%97%E6%B3%95/1208652?fr=aladdin)

[非对称加密算法](http://www.360doc.com/content/16/0505/16/16915_556521577.shtml)

[非对称加密与SSH](https://www.imooc.com/video/5456)

[非对称加密里公钥、私钥的说明](http://www.cnitpm.com/pm/6014.html)

[数字签名](https://baike.baidu.com/item/%E6%95%B0%E5%AD%97%E7%AD%BE%E5%90%8D/212550?fr=aladdin)

[ 电商网站HTTPS实践之路（二）——系统改造篇](http://blog.csdn.net/zhuyiquan/article/details/69569253?locationNum=10&fps=1)

[防止混合内容](https://developers.google.com/web/fundamentals/security/prevent-mixed-content/fixing-mixed-content?hl=zh-cn)

[内容安全政策](https://developers.google.com/web/fundamentals/security/csp/?hl=zh-cn)

[Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
