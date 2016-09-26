# 数据和安全

## 安全总览


几乎每一位使用 LeanCloud 的开发者都会问，如何能够保证自己应用数据的安全？对安全的关注说明你也是位对产品负责、对用户负责、对自己负责、做事态度认真的开发者，这也正是 [LeanCloud 所信守的价值观](http://open.leancloud.cn)。


LeanCloud 提供的服务都是基于 HTTP 协议的，我们云端对客户端发过来的每一个请求，都进行了身份鉴别（Authentication）和访问授权（Authorization）的严格检查。

### 身份鉴别（Authentication）

LeanCloud 为每一个应用准备了三个身份鉴别需要的标识：

- **appId**：这是一个应用的全局唯一标识，类似于我们个人的身份证号码。
- **appKey**：用于判断来自该应用客户端的请求是否合法的验证序号。
- **masterKey**：也是一种 appKey，只是使用它来进行请求的时候 LeanCloud 云端会忽视所有的访问权限控制（因为它拥有**超级权限**，用它来发起操作，类似于在 Linux 系统中的 root 在进行操作。），因此禁止在客户端或不信任的环境下使用，以免泄漏。

我们在使用 SDK 或者直接请求 REST API 的时候，都需要在 HTTP Header 中提供 appId 和 appKey 信息：appId 用来标识是哪一个应用发起的请求，appKey 则用来对请求的合法性进行鉴权。网络请求格式如 [REST API 文档](./rest_api.html#请求格式) 所示，所以 appId 和 appKey 对一个应用来说非常重要。

网络世界中总有一些别有用心的人存在，appId 和 appKey 在这里是明文传输的，尽管我们做了很多努力，但还是难保不会泄漏。所以，为了避免这种风险，我们又采用了一种更安全的鉴权方式：
**在 HTTP Header 中不再直接填写 appKey，而用一个「签名字符串」代替。**
签名字符串在客户端计算，由 appKey（或者 masterKey） 加上请求发起时的时间（精确到**毫秒**），再对它们进行 MD5 签名之后得到的字符串。LeanCloud 云端收到这样的请求之后，根据 appId 可以找到内部保存的 appKey，然后经过同样的散列函数计算出签名，比较后如果签名不匹配或者请求已经超时，则认为是一个非法请求而直接丢弃，如果是一个合法请求，则再进入下面的访问授权检查流程。详细的签名格式请参考[更安全的鉴权方式](./rest_api.html#更安全的鉴权方式)。

如此一来，网络传输过程中，我们可以做到不再暴露任何 appKey 的信息，但是对于原生的 iOS、Android 应用，或者 Web 站点来说，黑客还可以通过反编译等手段，获知到代码中在初始化 LeanCloud SDK 时设定的 appId 和 appKey。所以我们不能假设 appKey 不会泄漏，而应该使用下一节介绍的访问控制列表来加强数据的安全性。

### 访问授权（Authorization）
最灵活的保护你应用数据安全的方式是通过访问控制列表（Access Control List），通常简称为「ACL 机制」。ACL 背后的机制是将每个操作授权给一部分 **用户**（User）或者 **角色**（Role），只允许这些用户或角色执行这些操作。例如，一个用户必须拥有读权限（或者属于一个拥有读权限的角色）才可以获取一个对象的数据，同时，一个用户需要拥有写权限（或者属于一个拥有写权限的角色）才可以更改或者删除一个对象。

ACL 工作的前提是**用户（User）**和**角色（Role）**。**用户（类名`_User`）**是 LeanCloud 内建的账户系统，自动支持用户注册、登录、验证等操作，详细内容请参考[用户系统](./rest_api.html#用户-1)。**角色（类名`_Role`）**是一种有名称的对象，包含了用户和其他角色（也就是说角色也有层次关系），将权限授予一个角色代表该角色所包含的其他角色也会得到相应的权限，详细内容请参考[文档：角色](./rest_api.html#角色-1)。

LeanCloud 允许你对两种目标设置 ACL：
#### Class 级别的 ACL
创建 Class 的时候，控制台会弹出一个窗口，如下图：

<div style="width: 600px;"><img src="images/security/acl_template.png" class="img-responsive" /></div>

在这里选择一个默认的 ACL 权限。


另外，针对已经存在的 Class 你可以更新整个 Class 权限。进入 [控制台 > 存储 > 数据](/data.html?appid={{appid}}#/)，选择一个 Class，再点击右侧菜单中的 **其他** > **权限设置**。


![image](images/cla_permission.png)

在这里，你可以设置在一个 Class 上的不同操作权限：

* GET - 通过 objectId 获取单个对象。
* Find - 通过指定条件来遍历所有符合目标的对象。
* Update - 修改一个已经存在的对象。
* Create - 在 Class 表中插入一个新对象。
* Delete - 删除任意既有对象。
* Add fields - 给 Class 增加新的属性列

对于每一种权限，又可以开放给不同用户：

* 所有用户 － 指所有客户端都可以进行这一操作，相当于 public 权限
* 登录用户 － 只有使用 LeanCloud 内建账户系统（AVUser）并且进行了登录的客户端，才可以执行这一操作。
* 指定用户 － 只有指定用户或者角色才可以执行这一操作，这是对操作权限进行最精确控制的方式。

譬如我们有一个匿名发帖的应用，所有人都可以发表帖子，但是只有经过管理员审核后的帖子才能被展示出来。我们可以有两张表，第一张表用来存放审核前的帖子，这张表的 create 权限就可以开放给所有用户；第二张表用来存放审核后的帖子，这张表的 create 权限就只开放给管理员。

##### 对客户端隐藏部分列

对于 Class 来说，还有一种特殊需求，就是我们希望某些列的数据不在客户端展示出来。比如，匿名发帖的应用，你仍然希望发帖的时候，也记录下真实的作者，但不希望将此信息返回给客户端；账户系统中可能存有用户身份证号码，我们也希望通过 SDK 进行查询的时候，永远不要返回出去。这时候就需要能够对于普通查询永远隐藏部分列。

LeanCloud 也允许将部分列的数据设置为 「客户端不可见」。设置了之后，当客户端发起查询的时候，返回的结果将不包含相关字段。当然，在 LeanCloud 控制台，或者使用 masterKey 来做查询的话，是可以看到完整的数据的。

#### Object 级别的 ACL
在 LeanCloud 中，还可以对 Class 中的每一条记录（Object）单独设置 ACL，这时是按照读、写分离来对不同用户、角色进行授权的。在应用控制台中进行操作的界面如下图所示：

![image](images/acl.png)

这样我们就可以做到：

* 对于私有数据，read 和 write 都可以限制为对象创建者所有（其读写权限都设置成创建者自身，并且不开放「所有用户」的读、写权限）。
* 一个信息公告板的帖子，作者和属于「版主」角色的成员可拥有修改权限，普通访客则只允许 浏览（给「所有用户」开放「读」权限，给「作者」和「版主角色」开放「写」权限）。
* 应用内全局的每日广播消息，对所有用户是只读的，只有管理员可以修改它（给「所有用户」开放「读」权限，给「管理员角色」开放「写」权限）。
* 一条从一个用户发往另一个用户的消息，可以将读和写的访问许可限制到关联的这两个用户，其他人一概不可读写（对「所有用户」关闭「读、写」权限，只对两个用户开放权限）。

使用 LeanCloud SDK，你可以设置一个默认的 ACL 给客户端所有创建的对象。如果你同时开启允许匿名用户的功能，你可以保证你的数据拥有严格限制到每个单独用户的 ACL 权限。基于用户和角色的权限管理指南，详细信息请参考如下文档：
- [JavaScript 权限管理指南](./acl_guide-js.html)
- [Objective-C 权限管理指南](./acl_guide-ios.html)
- [Android 权限管理指南](./acl_guide-android.html)

### Web 应用安全设置
在通用的两层鉴权机制之外，我们单独讨论一下 Web 应用的安全问题。

对于 Web 应用来说，盗取 appId／appKey 的难度会更小一些，所以我们需要做一些特别的防御措施。关键点是，我们要能够保证其他人把你的 appId 偷过去之后，他也无法直接使用你的服务器资源。Web 端可以通过 `Web 安全域名` 来对请求来源做限制，可以简单的防御住 Web 的服务器资源盗取。

设置「Web 安全域名」后，仅可在该域名下通过 JavaScript SDK 调用服务器资源，域名配置策略与浏览器域安全策略一致，要求域名协议、域和端口号都需严格一致，不支持子域和通配符。所以如果你要配置一个域名，要写清楚协议、域和端口，缺少一个都可能导致访问被禁止。举例说明一下域名的区别：

```
// 跨域
www.a.com:8080
www.a.com

// 跨域 
www.a.com:8080
www.a.com:80

// 跨域  
a.com
www.a.com

// 跨域
xxx.a.com
www.a.com

// 不同协议，跨域         
http:
https:
```

这样就可以防止其他人通过外网其他地址盗用你的服务器资源。但是要注意，**Web 安全域名**所能达到的目的是防御恶意部署，而不是防御伪造脏数据（恶意用户通过绑定 host 方式还是有可能访问到应用的数据），所以要想对数据进行更多细粒度的控制，需要配合 ACL 来使用。

在 WebView 中使用，建议通过 WebView 去加载一个部署好的、有域名的 Web，然后缓存在本地，这样可以通过 **Web 安全域名** 来做限制。


如果在前端使用 JavaScript SDK，当你打算正式发布出去的时候，请务必配置 **Web 安全域名**，方法是进入 [控制台 > 设置 > 安全中心 > **Web 安全域名**](/app.html?appid={{appid}}#/security)。



![image](images/security/web-host.png)

## 安全中心


安全中心，是我们为每个应用提供的设置基本安全的入口，位置在 [控制台 / 设置 / 安全中心](/app.html?appid={{appid}}#/security)。除了之前讲过的**Web 安全域名**外，这里还可以设置或查看：


### 服务开关

服务开关，是用来开启或者关闭当前应用所使用的服务，从根本上防止由于 AppId 和 AppKey 泄露后而可能会引发的服务资源被盗取的问题。

![image](images/security/service-switch.png)

### 操作日志

操作日志中会显示应用创建者及所有协作者的重要操作记录，比如删除数据操作的历史、操作用户名、操作 IP 及操作时间等，这个日志的目的是为了遇到问题更好地定位故障缘由，排查可能的恶意操作，防止应用数据被错误地改动。

## 其它安全措施

### 应用安全选项


进入 [控制台 > 设置 > 应用选项](/app.html?appid={{appid}}#/permission)，选中或取消选中列表项目即可启用或者关闭对应的功能。



- 用户账号 > **用户注册时，发送验证邮件**<br/>
  是否要求你应用里的注册用户验证邮箱， 默认不启用。如果启用，每次用户注册，都会发送一封邮件到用户提供的邮箱，要求认证，具体请看数据存储指南的用户一节。
- 其他 > **禁止客户端创建 Class**<br/>
  是否禁止客户端动态创建 Class。如果启用，新的 Class 将只能通过云端管理平台来创建，而无法通过 SDK 或者 REST API 方式来动态创建了。
- 聊天、推送 > **禁止从客户端进行消息推送**<br/>
  是否禁止从客户端推送消息，如果启用，这那么通过 SDK 或者 REST API 都被禁止推送消息，只能通过我们管理平台提供的推送界面来推送消息。


如果想彻底关闭一个应用的数据存储、短信发送、消息推送和聊天（实时通信）功能，请进入 [控制台 > 设置 > 安全中心](/app.html?appid={{appid}}#/security) 来进行相应设置。


### 自动备份

LeanCloud 目前会每天备份一次应用数据，防止用户误操作删除了重要数据。如果发生误删除，请及时联系我们进行恢复。

### SSL 加密传输

首先，我们所有的 API 请求都通过 [SSL加密传输](http://zh.wikipedia.org/wiki/%E5%AE%89%E5%85%A8%E5%A5%97%E6%8E%A5%E5%B1%82)，保证传输过程中的数据安全性和可靠性。并且，我们的 iOS／Android SDK 还加入了防范中间人攻击的措施，彻底排除网络抓包和嗅探的威胁。

### 第三方加固

对于 Android 应用，除了代码混淆之外，还可以使用第三方加密工具，隐藏 classes.dex，通过动态加载的方法进一步提高应用的安全性。下面我们简单介绍一下爱加密。

[爱加密](http://www.ijiami.cn/) 是专为移动开发者提供安全服务的一个平台，可解决开发者面临的应用安全问题。加密的步骤很简单：

1. 提交应用；
2. 下载加密后的 apk 文件；
3. 下载爱加密提供的签名工具，对应用进行签名。

相关步骤还可以见下面的截图。

申请账号，提交应用，下载签名工具：

![image](images/ijiami_2.png)

加密后重新签名：

![image](images/ijiami_1.png)

这样得到的 apk 文件，普通的反编译之后得到的是，

![image](images/decompile.png)

可以看到，代码被隐藏起来了，应用被破解的难度大幅增加了。

对于任何移动应用来说。因为客户端代码运行在一台移动设备上，因此可能会有不受信任的客户强行修改代码并发起恶意的请求。选择正确的方式来保护你的应用非常重要，但是正确的方式取决于你的应用，以及应用存储的数据。

我们提供多种方式使用权限控制来获得安全性。如果你有关于任何保护你应用安全的最佳方式的问题，我们都鼓励你联系我们的客户支持。我们一直努力提供更多功能给开发者来保护你的应用，也希望大家持续地给我们反馈，感谢。