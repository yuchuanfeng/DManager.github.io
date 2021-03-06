---
layout: post
title: "掌柜客户端团队架构升级之路"
date: 2018-09-12 10:42:50 +0800
styles: [data-table]
comments: true
author: [薯片]
published: true
---

## 前言

![team photo][teamphoto][^1]

记得我是16年3月31号入职二维火的。在过去的两年多时间里，我一直在火掌柜客户端团队工作。一直以来都想写一篇blog记录一下掌柜团队的变化，这次正好借公众号系列文章的机会来完成这个小目标。

不知道看官大人发现没有，标题中的`团队`[^2]两字显得有点突兀。原因是在下不光想在这里聊聊掌柜客户端着两年多来的变化，也想说说团队工作方式的转变。

<!-- more -->

## 旧掌柜

既然说到升级，不得不简单聊一下升级前掌柜是什么样子的。

![旧掌柜截图][image-1]

上图就是16年掌柜首页的样子。16年的掌柜可以说是一个软件，而并不是C/S架构模型中的Client，我们所说的客户端。总结来说当时我们主要面临这些问题。

1. 很多逻辑是放在端上的，只有在不得已的情况下（类似数据要保存）才会跟服务端发生数据交换。
2. 客户端错误没有监控，我们并不知道客户端哪里出现了异常、闪退，和发生的次数。
3. 业务数据没有埋点。
4. iOS客户端整个App没有导航系统。所有页面都堆在一个Base View上，互相切换通过show/hide的方式进行的。这也导致了App生命周期混乱。没在使用的页面、组件不会销毁。
5. 通过通知在页面间传递消息。
6. 第三方组件管理不规范，不少使用的组件已经停止维护或者版本落后。
7. 有了一定的代码量，但是所有代码在一个项目一个仓库中。
8. git 分支管理随意。

## git flow工作流

由于原来git分支流程管理比较混乱，面对正在膨胀的需求，要进行多人并行开发非常吃力。我们最终引入了git flow的工作流程。

同时我们对master进行了push写保护。

具体流程介绍可以参考文档:[git-flow][1]

## 模块化

一六年四月，原来作为掌柜App中一个模块的`供应链`，在功能上做了很多扩展，并且需要独立成一个App。

当时我们的做法是，把掌柜整个App写了一份podspec为供应链做成了一个独立的基础模块，供应链新增的功能分层分业务放在不同的模块中。最后做了一个供应链壳工程通过Cocoapods把需要的模块组装到一起使之可以编译运行。

选择这个方案的时候，我们有两个目标是明确的：

1. 代码不能分裂。供应链和掌柜的业务都会有各自的发展，没有一个统一的底层库，代码分裂情况必然严重。
2. 互相模块独立。各自业务在自己仓库开发，互不影响。

虽然愿景是美好的，但是坑也是有不少的。原本我们的设想是，把给供应链提供模块的代码，和掌柜app的代码放在一个repo的两个分支中。反正在同一棵git的log树中，定期做一次merge因该是不难的。但是结果发现merge成本越来越大。

于是我们决定尽快抽出公用的底层 ，把供应链对掌柜的依赖彻底解开。幸运的是，随着供应链团队的独立，供应链客户端的兄弟对公共库的构建也出了不少力。目前掌柜和供应链共同维护着公共的底层库。

所以，供应链的独立也是掌柜声势浩大的模块化的开端。16年6月那个时间节点，我们的Android和iOS开发大概是各8个人。同时并行的项目应该在7个左右。我们认识到目前单工程单仓库的模式将会严重阻碍我们的开发效率和业务的发展。我们也明白没有壮士断腕的决心和勇气模块化是很难推行下去的。但是掌柜To B的业务特点又根本不可能给我们时间停下业务项目突击把模块化做完。

于是我们拍脑袋根据业务把模块拆开，每个人头上都分配不同的业务方向。在项目进行的过程中把相关模块拆开。一开始我们做的比较粗放型，比如会员独立的时候。我们不管依赖关系先把业务相关代码移到一个新仓库，写个podspec独立成pod让主工程依赖。这中间可能会有些底层公共代码，发现后再独立成一个底层库。这样的操作我们一直都在进行，分离业务库大概花了一个Q的时间。目前掌柜iOS客户端中三方库加私有库应该是183个。**各业务模块都可以脱离主工程独立运行。**

![火掌柜架构图][Architecture]

## 私有镜像仓库构建

开始我们cocoapods第三方组件用的是公共仓库，所以大多数三方组件代码是托管在github上面的。经常拉取的时候会花费很长时间。为了加快开发速度，我们把用到的三方组件git人肉搬运到了公司的gitlab上，同时把podspec放到了自己私有源中（spec中的source指向内部git地址）。每一个开发都有自己的责任第三方pod，定期同步更新。目前执行下来情绪稳定。

近期打算写一个脚本自动去同步github上最新版本的源码。解放一点生产力。

## 为不同屏幕尺寸的iPhone适配，添加导航控制

14年开始开发掌柜iOS客户端只对320/640px宽度的iPhone进行了适配，没有用自动布局。要一口气对整个app进行适配对掌柜客户端来说是不可能的。

我们的做法是分模块逐个击破。改几个页面私下找测试验证，验证完提交到待发布分支。等到整个App所有页面都用Autolayout适配完成，把iOS的启动图添加上iPhone6/7/X尺寸的图片。同时要求在新项目中所有新页面都用autolayout写。

![适配前后对比图][diff]

## 网络请求统一处理

在iOS更新到9以后，发现debug环境下生成的包网络请求一切正常。而打出正式包以后所有请求都发不出去。对比了两个环境的差异后，发现是开了编译优化导致ASIHTTPRequest部分功能失效。无奈暂时把生产环境的编译优化关掉。同时开始进行A2A（ASIHTTTPRequest to AFNetworking）行动.

![编译器选项][config]

在迁移到AFNetworking之前我们对网络请求的调用和解析也做了规范。我们把Request和Resoonse分别抽象成了各自的数据结构。业务层使用只需要根据接口名和参数等信息构造对应的request实例，通过统一的接口交由HTTPClient负责完成基础参数组装、对应host拼接、接口签名等底层逻辑。请求完成后判断业务是否正确，最后返回给调用方一个标准的resopne实例。

正好遇到旧接口迁移，在同掌柜服务端同学的共同努力下，客户端把网络请求全部迁移到了新的网络底层框架上来。

统一网络接口后，我们做了一些测试工具，来提升开发、测试效率。比如在内部测试包中可以自由切换网络环境。我们还在对每一个请求进行了格式化操作，在console中可以直接输出一个保存了app现场的curl命令（包括app操作过程中的url、参数、header等信息）。遇到问题可以直接把命令发给服务端开发的同学方便调试。


## 指哪打哪：全页面路由配置

之前提到，掌柜iOS App没有一个统一的Navigation Controller，造成了一系列的问题。不解决是做不下去的。问题还是同一个：`掌柜是一个没有办法停下业务做架构改造的团队`。这次的改造我们把添加Navigation Controller和页面URL配置合在一起做。也是跟着项目走，做到哪些页面做相应更改。

达成的效果是目前掌柜的模块间跳转可以通过约定的URL进行。这也为后面的动态方案提供了基础。

## 持续集成

对于一个成熟的开发团队来说持续集成是必不可少的。

于是我们团队着手构建并维护了一套供全公司客户端开发使用的持续集成平台。

之前有两篇关于这方面的blog产出这里就不多说了。

[青木：《火掌柜 iOS 团队 GitLab CI 集成实践》](https://dmanager.github.io/ci/2018/06/23/ji-gitlabcide-ci-shi-jian/)
[石胆、蛋黄派：《二维火掌柜Android持续化集成探索》](https://dmanager.github.io/android/ci/2018/04/09/android-ci/)

## 动态化

掌柜动态化需要话分两头来讲。一头是首页动态化，一头是内页动态化。

起初做页面动态化是17年4月份的首页改造中进行的。由于原来仅仅是首页视觉改造，加入动态化方案后，历时3个月项目一再延期。但是经过这次改造后，首页页面的架构发生了质的变化。端只是知道有哪些种类的控件，至于控件中的内容和控件铺排的顺序一概都是由服务端接口上吐出的。这样的架构为掌柜业务的灵活发展提供了基础，也是后来首页banner位的基础架构。

对于掌柜这样一个供B端商户使用的类CMS客户端，我们有大量的页面是表单形式的。所以对于客户端开发来说不少工作是在用不同的顺序组合类似的表单控件。于是我们开始思考内页动态化方案。

我们迈出的第一步是，把所有能重复使用的交互组件通过对应的数据model来描述。在业务开发的过程中只要把不同的数据放到datasource中，页面就铺排完成了。这样的模式进行了一段时间后，我们客户端开发的嘴角弧度似乎往上扬了几度。

然后我们想要达到可以通过让产品后在配置台拖拽的方式就能生成一个页面，并且可以设置不同控件交互的动作。目前功能已经基本完成，在一些项目中有应用。

有视频为证：


<div>
<video width="100%" controls style="margin-left: auto;margin-right: auto;display: block">
    <source src="/images/restapp-2-years/IMG_3142.mp4" type="video/mp4">

</video>
</div>

之前有过一次分享：[动态化Keynote](https://dmanager.github.io/HKReporter-keynote)

`虾米`会出一系列关于动态化技术细节相关的分享。

## 能让机器干的事情绝不让人干

团队内部我们围绕着gitlab来进行任务的推进和跟踪。之前写了一点分享：[安利gitlab的一些功能 ](http://git.2dfire-inc.com/snippets/4)。

我们希望每天早上回顾一下存在的问题。于是有了每天在上钉钉群里面对gitlab上存在issues的通报。

![issue repoter][issue repoter]

青木发起了一个悬赏活动。于是有了每天悬赏任务的提醒。

![notification1][notification1]

每天早上我们会对所有模块做一次集成测试,没有通过的有通知：

![CI note][CI note]

## 团队是最重要的 

所有的这些事情离不开团队所有成员的努力。以上提到的不少事情都是我们团队成员在工作中发现问题，主动解决的。

我一直希望我们的团队能撑起业务同时又能够关注最新技术的发展。不管是二维火产品对社会效率提升的贡献，还是我们团队内部的技术沉淀，都能让我们享受到技术创新带来的便利和乐趣。

### 团队blog

在github上建了个团队blog。不定时有更新。欢迎通过RSS订阅：[blog feed][2]

[1]: https://www.git-tower.com/learn/git/ebook/cn/command-line/advanced-topics/git-flow
[2]: https://dmanager.github.io/atom.xml
[image-1]: /images/restapp-2-years/restapp_homepage_16.jpg
[teamphoto]: /images/restapp-2-years/team-photo.jpg
[issue repoter]: /images/restapp-2-years/notification2.png
[notification1]: /images/restapp-2-years/notification1.png
[CI note]: /images/restapp-2-years/CI-note.png
[Architecture]: /images/restapp-2-years/Architecture.png
[config]: /images/restapp-2-years/config.png
[diff]: /images/restapp-2-years/diff.png

[^1]: 摄于2018年8月28日，照片中诸位是当时掌柜客户端团队的全体成员。对于之后加入的、分到其他团队的、和已经离职的各位这里同样表示感谢。
[^2]: 文中记述的大多一iOS项目为主，Android团队的经历也类似。
