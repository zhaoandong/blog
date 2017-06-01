## 背景
公司大了以后，会有很多的子域名，这个时候就需要不同的系统分享同一套登录体系。最近做开发遇到了一个关于Safari下用户登录的问题，a.com和b.com使用同一套登录体系，具体表现为a.com下登录的用户，但是b.com下却获取不到用户信息。

## 问题排查
经过排查，发现b.com上没有用户的cookie，进一步排查是由于苹果的安全策略导致在登录时统一写域失败。这个策略具体来说就是：如果用户没有访问过b.com时，是不可以在b.com上种上cookie的。

![image](http://ofyfg9y7t.bkt.clouddn.com/WechatIMG2.jpeg)

## 业界方案
既然知道了问题，而且这个问题很多公司都会遇到，那就得看看现在业界有没有成熟的解决方案。这时候我的第一反应是看看BAT是怎么做的。逛了一圈BAT的网站，发现阿里的方案不错。那我们就着重来剖析下阿里的技术方案。

我们发现即使清空了Safari的历史记录，然后在taobao.com上登录，接着我们在浏览器上输入 tmall.com，发现我们还是已登录的。按照苹果的限制，tmall.com是拿不到登录态的，所以我们来研究下taobao的做法。

### 我们在浏览器输入 tmall.com 发生了什么
![image](http://ofyfg9y7t.bkt.clouddn.com/WechatIMG6.jpeg)

通过抓包工具获取的数据包，我们可以看到，在 tmall.com中会有一次 top-tmm.taobao.com的jsonp请求，因为 taobao.com是登录态，所以这个jsonp请求里是可以带上用户的cookie信息的，而且 top-tmm.taobao.com的接口里是可以用jsonp的形式获取到cookie，然后回传给 tmall.com的，所以我们有了一个初步的方案。

### 初步方案
#### 步骤
1、a.com登录成功，可以成功在a.com上种下cookie  
2、b.com中会首先发一个jsonp的请求到 a.com上，a.com接口会根据jsonp 中带的cookie生成一个ticket，并返回到 b.com   
3、b.com用获取的ticket 提交给 b.com的后端服务。b.com的后端服务根据ticket 然后去换为相应的用户标识，并种在 b.com下，这样 b.com也是登录态了。 

#### 问题
这个方案是可行的，但是有个问题，如果b.com的页面必须是登录才能进，由前端发起jsonp 请求后，发现用户在 a.com也没登录，这时候再跳到登录页，会有一个白屏时间，用户体验不是很好。

### 进一步方案

我找了个 tmall.com下需要登录才能进去的页面 mybrand.tmall.com验证，发现并没有预想中的闪屏，下面我们接着抓包。

![image](http://ofyfg9y7t.bkt.clouddn.com/WechatIMG7.jpeg)

我们来分析下数据包，在我们输入 mybrand.tmall.com的时候，做了个302跳转到 taobao.com下。接着带了参数回 pass.tmall.com 中，从字面上看，参数都是cookie的值，最后跳到 mybrand.tmall.com。

基于此我们改进了我们的初步方案。


#### 步骤
1、在 a.com上登录成功，并种下cookie  
2、b.com校验用户登录态，如果未登录，则302跳转到 a.com/getTicket  
3、在 a.com/getTicket 中是可以获取到用户的cookie的，在这个方法中，后端服务需要将用户的 cookie 换成一个ticket，然后用 querystring 方式将ticket 带到 b.com/setCookie  
4、在b.com/setCookie 中，用参数中带来的ticket 去换取相应的用户标识，并种在 b.com，接着跳回 b.com
5、此时，b.com就是登录状态了

#### 流程图

![image](http://ofyfg9y7t.bkt.clouddn.com/charts.png)
