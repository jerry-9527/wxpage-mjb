### 前言
一直在找原生基础上扩展属性的框架, 主要想干净点, 自己容易扩展, 后来找到了 wxpage(腾讯视频出品), 在它基础上对 router 改了一下 [github 地址](https://github.com/clevok/wxpage)

参考[文章](https://wetest.qq.com/lab/view/294.html?from=content_csdnblog)
[mixins混入替代方案,全局状态管理替代方法](./COMPOSITION.MD);

## 目录

* [刚入门时,我的错误理解](#刚入门时,我的错误理解)


* [预加载](#预加载)
    - [preload的实现方案](#preload的实现方案)
    - [为什么要将请求保存在一个对象中而不是在onPreLoad直接更改data内容](#为什么要将请求保存在一个对象中而不是在onPreLoad直接更改data内容)
    - [预加载的研究性的方案:废弃](#预加载的研究性的方案)
    - [预加载的另一个小方案:适合长期不变的数据](#预加载的另一个小方案适合长期不变的数据)
    - [wxpage的preload原理同wepy](#wxpage的preload原理同wepy)


* [跨页面通讯方案](#跨页面通讯方案)
    - [全局状态管理](#全局状态管理)
    - [全局状态管理之小程序版Composition](./COMPOSITION.MD)
    - [事件发布订阅](#事件发布订阅)
    - [onShow配合其他数据](#onShow配合其他数据)
    - [hack模式:不推荐](#hack模式)


* [页面的设计模式](#页面的设计模式)
    - [自定义组件模式](#自定义组件模式)
        - [如何给自定义组件根设置样式](如何给自定义组件根设置样式)
    - [import模式](#import模式)
    - [混入模式](#混入模式)
    - [小程序版Composition](./COMPOSITION.MD)

* [视图层显示优化](#视图层显示优化)
    - [按钮加载成功后显示](#按钮加载成功后显示)


* [setData优化](#setData优化)


* [哪些可以使用绝对路径](#哪些可以使用绝对路径)
    - [引用自定义组件](#引用自定义组件)
    - [组件关系引用](#组件关系引用)
    - [wxss引用](#wxss引用)
    - [js引用](#js引用)


* [小程序框架设计想法](#小程序框架设计想法)
    - [搭建私有npm和github](#搭建私有npm和github)
    - [自定义组件方面](#自定义组件方面)
    - [脚手架](#脚手架)
    - [路由方面](#路由方面)
    - [wxss方面](#wxss方面)
        - [css3变量](#css3变量)
        - [可配置方案](#可配置方案)
        - [主题色方案](#主题色方案)


* [wxpage解析未完](./WXPAGE.MD)

* [wxml的一些问题](./WXPAGE.MD)

* [自定义组件的一些问题](./COMPONENT.MD)

------------

## 刚入门时, 我的错误理解
小程序加载类似于 单页面, 主包的内容在加载小程序的时候, 相关页面都直接加载(所有的Page,Component(不管有没有引用都会加载)),其他的js,除非被引入,才会加载

一开始我曾经还以为, 小程序的加载方式 是 类似 传统 多个html页面（或者hubilder App 方案

    // A.js
    const config = require('./config');
    console.log('A.js 初始化');
    Page({
        onLoad() {
            console.log('A.onLoad');
        }
    });

    // B.js
    console.log('B.js 初始化');
    Page({
        onLoad() {
            console.log('B.onLoad');
        }
    });

一开始我以为 从A页面 加载B页面的时候， B.js会整个 重新加载

    // B.js 初始化
    // B.onLoad

事实上, 每次跳到B页面, 仅仅 B页面Page对象会进行`深拷贝`, 进入 页面的生命周期, js文件不会重复引用


且主包的所有的内容在一开始就会加载, 就像 vue webpack加载方式, 而分包的内容就像 webpack配置单独的模块, 只有加载到该模块后才会加载该模块下的所有的内容

    // App 加载的时候， 所有的js 都会记载

    // A.js 初始化
    // B.js 初始化
    // 等等其他的页面

    // 然后加载首页A页面, 进入A页面的生命周期， 触发钩子 onLoad方法
    // A.onLoad

> 不知道咋形容, 其实就是单页面的加载方式, 每次仅仅切换 Page({}) 对象而已.
*还有就是, 你可以写个js, 存储一个对象, 这个对象的数据, 一个A页面改动了, B页面读取的也是改动之后的*

> 在小程序启动时，会把所有调用Page()方法的object存在一个队列里（如下图）。每次页面访问的时候，微信会重新创建一个新的对象实例（实际上就是深拷贝）。

[](https://f.wetest.qq.com/gqop/10000/20000/LabImage_77ebebbdda7eb0340c5f1939ba93c1e1.png)


------------


## 预加载

预加载 的 实现方案 就是在小程序页面加载机制上 ，A页面通过事件监听的等方式, 先调用 对应页面B 的 原型（即将要被深拷贝的页面对象） 的特殊的方法。来达到 分离业务逻辑，提前请求的目的

### preload的实现方案

其实就是给每一个Page页面注册了个事件, 在调用跳转方法`this.$router`的时候, emit了个方法,唤起对应 响应页面 并 执行对象的方法比如 `onPreLoad` , 比如, 提起请求, 将请求保存在一个 对象中, 在那个页面加载onLoad后, 再从对象中取出这个请求, 做处理


### 为什么要将请求保存在一个对象中而不是在onPreLoad直接更改data内容
为什么要将请求保存在一个对象中,onLoad后再取出,而不是在onPreLoad直接更改data内容

这是小程序一个特殊点
小程序主包所有的页面都加载, 在进入一个页面的时候, 会 `深拷贝` 对应页面的Page内的对象, 并扩展一些属性, 存入 [页面栈中](https://developers.weixin.qq.com/miniprogram/dev/framework/app-service/route.html) getCurrentPages(我猜的)


因为 请求是 需要时间的, 你在A页面调用了B页面的 `onPreLoad`, 这个方法，更改 `this.data.name`, 但是, 如果 该页面已经深拷贝了, 请求还没回来, 后来 更改 `this.data.name` 实际更改了 `js文件` 内的data, 深拷贝后的 `this`, 不是原先那个, 两个`this`没有关系, 毕竟, 拷贝后的 `page对象` 没有执行`onPreLoad`方法


### 预加载的研究性的方案

> 那么如果是在前一个页面直接更改要跳转页面的 data的属性, 是立即执行, 不耗时, 且我延后跳转到该页面，深拷贝是否就是我想要的对象了呢

一开始, 我感觉似乎是这样, 蛮有道理的, 然而实际却不太一样

    // A.js
    {
        preload () {
            event.emit('preload');
        }
    }

    // B.js
    event.on('preload', ()=> {
        item.onPreload();
        setTimeout(()=> {
            wx.navigateTo({
                url: 'B'
            })
        }, 200);
    });

    let item = {
        data: {
            demo1: 'demo1.name',
            // demo2: config.demo2 // 引入的方式也一样的
            demo2: {
                name: 'demo2.name'
            }
        },
        onPreload: function () {
            this.data.demo1 = 'change.demo1';
            this.data.demo2.name = 'change.demo2';
        }
    }
    Page.P(item);

一开始感觉, A.js 点击 , B.js 更改了item.data属性, 小程序加载B页面, 深拷贝页面对象, 此时的深拷贝对象应该是更改后的, 结果！！！ 没有变化。
再后来发现

**`只有在`2.4.0及以下`的版本上面的方法能起作用，但是不要用这个了，了解下就好, 新版本不支持`**

---

组件的加载：不管用不用，都先加载, 组件的生命周期, 且只有在页面中wxml中引用才会触发, 仅仅json中配置引用没用, 多个页面引用相同的组件, 这个组件的this是不同的,不必担心监听事件的问题

---

### 预加载的另一个小方案适合长期不变的数据
那就是 stroage

    a.js
    Page({
        data: {
            name: wx.getStorageSync('name')
        }
    });

为什么说时候长期不变的数据呢, 回归小程序加载的特性, a.data.name的值在 小程序加载的时候 被赋值了, 在切换页面中, 哪怕你更改了 storage.name的值, 初始化的data.name 依然是 加载之初读到的数据, 因此, 适合保存用户头像, 姓名, 到本地, 等下一次 加载小程序的时候, data.name 初始值 自动是 上次的了, onLoad的时候再重新拉一遍 覆盖上去。
说他适合长期不变，主要是 你要是请求太慢的话，先显示老的数据，如果你的项目对消息及时性要求高的话。还不如初始是null呢
毕竟默认初始数据 只有小程序 重新加载后才生效

### wxpage的preload原理同wepy

上面的那个方式,必须保证页面跳转前 `数据`已经加载完了, 再跳转，才能读取深拷贝改变后的data, 这样就很不好了
所以, 该框架采用, 跳转的时候, 就请求加载数据, 把请求体保存在内存（变量），等B页面加载完后，onLoad() 再读取这个变量，赋值到data
,说白了，就是提前请求。

然后，为了分离代码，框架又 监听了事件

[官方示例: 参考代码](https://github.com/tvfe/wxpage/issues/25)

    // 页面index
    Page.P('index', {
        clickToDetail() {
            this.$preload('detail')
        }
    })


    // 页面detail
    Page.P('detail', {
        onPreload() {
            this.$put('detail_preload_fetch', this.fetch())
        },
        onLoad() {
            ;(this.$take('detail_preload_fetch')||this.fetch()).then(data=>{
                console.log(data)
                // do somthing
            })
        },
        fetch() {
            return fetch('xxx')
        }
    })

`this.$preload` 是emit事件, detail监听到执行 `onPreload` 方法，`$put` 将 `this.fetch()`保存到一个名为 `channel(brideg.js)` 的对象中
然后调转到detail页面，再通过`this.$take`取出`channel`对应的`key` 这样就实现啦
如果你有疑问, 那就是 onLoad 内的this和 onPreload 内的this 不一样， 为什么可以取，因为 put,和take是框架扩展的两个方法而已，真正还是往包里（channel）拿里拿

### setData不影响对象深拷贝特性

        let a = {name: 2};
        this.setData({
            redis: a,
            [`redis2[0]`]: a
        });
        console.log(this.data.redis);
        console.log(this.data.redis2);
        a.name = 7;
        console.log(this.data.redis);
        console.log(this.data.redis2);
        this.setData({
            [`redis.name`]: 9
        })
        console.log(this.data.redis);
        console.log(this.data.redis2);

------------

## 跨页面通讯方案
我们经常会碰到 比如在用户中心页面跳转到设置中心页面, 设置后 要求 用户中心页面 数据 也能跟着一起改变
(参考)[https://segmentfault.com/a/1190000008895441#articleHeader6];

 1. 全局状态管理
 2. 事件发布订阅
 3. onShow/onHide + localStorage 或 globalData 或 单独写个js存储对象
 4. hack模式(不建议)


### 全局状态管理
其实就是类似 vuex, redux, mobx, 详情百度 小程序官方也出了[小程序的 MobX 绑定辅助库](https://developers.weixin.qq.com/miniprogram/dev/extended/utils/mobx.html)

### [全局状态管理之小程序版Composition](./COMPOSITION.MD)
这个是另一种思路, 蛮有意思的, 只能作为参考


### 事件发布订阅
原理：我比较喜欢的方案, 就是事件发布订阅, 建议大家自己写一个, 通过this来绑定, 页面销毁的时候再移除掉
[event.js](https://github.com/clevok/wxpage-mjb/blob/master/src/core/event.js)

使用场景：后面的页面 对前面的页面做修改
缺点：要非常注意重复绑定的问题, 页面加载得注意移除, 目前我有用这个

### onShow配合其他数据
原理：这种就是 onShow/onHide的时候读取本地存储, 或者 app.globalData 或者 保存在 其他js中的数据, 显示出来

使用场景：用在在一些跳转到页面后 再执行
优点：实现简单，容易理解
缺点：如果完成通信后，没有即时清除通信数据，可能会出现问题。另外因为依赖localStorage，而localStorage可能出现读写失败，从面造成通信失败
注意点：页面初始化时也会触发onShow

### hack模式
这个不建议使用, 比较容易出现， 不知道这个页面在哪里被改的情况
原理： 就是把每个页面的this维护起来, *(其实也可以通过 `getCurrentPages()`)* 可以直接访问到这个页面的方法，属性等，直接更改

优点：一针见血，功能强大，可以向要通信页面做你想做的任何事。无需要绑定，订阅，所以也就不存在重复的情况
缺点：有风险，谁知道会不会突然无效了呢，且页面业务逻辑大了，不知道在哪里做的更改影响到了

---

## 组件
关于组件我有新发现了,组件由3部分组成, `component.js`,`component.wxml`,`component.wxss` 其中 `component.js`只会初始化一次，就是在页面加载的时候 (加载app的时候就初始化,而不是进入到页面), 对了,这里有所有的组件公用一个环境变量的风险


### 公用一个环境变量的风险
我们常常用闭包来实现一些特效, 比如函数防抖

```js
function debounce(callback, time) {
    let timer = null;
    return function () {
        if (timer) clearTimeout(timer);
        timer = setTimeout(function() {
            callback();
        }, time);
    };
}

Component({
    properties: {
        value: {
            type: String,
            value: '',
            observer: debounce(function(){
                // do something
            }, 200)
        }
    }
})
```
比如实现一个输入框 箭头输入, 200不变后才返回结果，！！！！但是, 这个有问题, `就是多个组件同时存在的情况下`
一般你可以以为,一个组件的时候,会初始化一次, 两个组件的存在的时候,他会初始化二次,实际上呢！
他只会初始化一次,且是在app加载的时候初始化(这个不能忘)

这样会照成什么样的结果呢,
一个页面的两个组件的value一起变化,实际上他们是公用一个 `timer` 的！！！, 又因为节流会导致只响应最后一次刚刚的那个组件

#### 解决方案
把timeer放到 `this.data` 下, 不要被公用掉, 通过this来区分唯一

```js
observer: (function () {
    return function (newvalue, oldvalue) {
        if (this.timer) clearTimeout(this.timer);
            this.timer = setTimeout(() => {
                // do something
            }
        }, 400);
    };
})()
```

------------

## 页面的设计模式
设计页面的结构有很多种, 小程序页面类似于vue,可以参考vue开发模式,小程序也有自己特殊的组件, import模式


### 自定义组件模式
此模式 与 vue 相似, 可以自定义组件内做自己的事情, 数据改变通知外部方式, 不应该主动调用外部方法
就是页面可以拆分成多块, 把按钮,选择器之类的基本 组件拆分, 当然, 也可以把页面作为组件(该组件可以直接作为页面,也可以作为组件)

参考 (使用 Component 构造器构造页面)[https://developers.weixin.qq.com/miniprogram/dev/framework/custom-component/component.html]

对了, 自定义组件可以绝对路径引用
```js
{
    'index-join-list': '/pagesComponents/index/pages/joinList/joinList'
}
```

#### 如何给自定义组件根设置样式
也就是 shadow-root, 要么在 自定义组件设置
```css
:host {
    // 注意了, 这样样式不能写在@import里面的, 不会起作用
}
```
还有就是直接给自定义组件标签上加class
```js
    <login class="demo"></login>
```
这里需要注意一点,就是 如果你设置demo宽度 500px; 没有用, 你还需要设置 display: inline-block;才可以
当然 `:host` 没有这个问题, class优先级大于 :host
![host](./static/readme1.png)

### import模式
这个模式有点意思, 利用 小程序模板(template)[https://developers.weixin.qq.com/miniprogram/dev/reference/wxml/template.html]和mixins(需自己实现)来混合实现
这个方式有点意思, 在形式上实现了页面拆分, 数据又统一, 并不是自定义组件通过通知改变上层数据, 而是在分层中直接改变数据
也就是,所有的业务逻辑都集合在了主页面上, 但是开发体验上却又是分散在不同的层,
对了, 模板也能够点击事件, 触发的点击事情是在引用的那个主页面上

```js

// index.wxml
<template>
    <include src="./action.wxml"></include>

    // 也可以采用 include 方式
    <import src="./action.wxml"></import>
    <template is="action" data="{{}}"></template>

    <view>主页面</view>
</template>

// action.wxml
模板action.wxml里的点击事件, 实际上会触发引入改页面的的js事件

<script>

    // 引用action的页面组件
    import action from 'action.js'

    Page({
        mixins: [action]
    });
</script>

```

### 混入模式
混入其实就是mixins,实现代码共享,但也有很大的问题,至于怎么实现很简单,详情百度 mixins如何实现


### [小程序版Composition](./COMPOSITION.MD)
一开始是为了解决mixins带来的问题,如果页面不复杂,那么混入带来的负面影响很小,如果很大的话,那带来的影响就很大了,你常常看到某个方法里调用另一个方法,却找不到在哪,代码也确实提示,当然,自己维护自己的代码还好,一单别人维护起来,那就蛋疼了


------------

## 视图层显示优化
### 按钮加载成功后显示
很多时候,我们会有这种需求,那便是一个按钮本来是隐藏的,后来再出现,所以弄一个`hide`属性
>结论: 我们应该反过来, 给这个view本身一个隐藏的属性, 当完成后, 让他显示

下面的案例是想这个按钮初始化, 一开始赋予 hide 属性
```js
    <view class="{{isCheckFinshed?'hide':''}}"></view>

    Page({
        data: {
            isCheckFinshed: true'
        }
    })
```
但实际上, 一开始view 直接显示出来了!!!(部分安卓手机上,ios基本看不出来), 也就是没有附加hide这个样式

#### 研究是因为变量默认是false导致的吗

可以简单的理解, class,wxml 这里的{{}} 一开始我们就当都不会搞出来, 就当不会做逻辑判断,解析生成class或文本

 (wx:if到是会, 但是视图是默认变量是false)

2.
```js
    <view class="fd--column flex-1" wx:if="{{haveSenderAddress}}">
        1
    </view>

    <view wx:else>
        2
    </view>

    Page({
        data: {
            haveSenderAddress: true
        }
    })
```
在比如说一些场景, true显示1, false显示2,
尽管默认 haveSenderAddress 一开始是true, 有的时候也会先显示2, 尤其在卡一点的手机上

**`猜测`** data内的属性在第一次加载的时候，对于视图层，相当于null

**`实践`** 所以我们用了两个属性, 来区分, 在卡一点的安卓手机上,成功实验出来

```js
    <view class="fd--column flex-1" wx:if="{{!isEmpty}}">
        1
    </view>

    <view wx:else>
        2
    </view>
    Page({
        data: {
            isEmpty: false
        }
    })

```
基本不会出现2的情况了
结论： 猜测, 默认视图层的变量都是空

---

在查看[`生命周期`](https://developers.weixin.qq.com/miniprogram/dev/framework/app-service/page-life-cycle.html)的时候
![生命周期](https://res.wx.qq.com/wxdoc/dist/assets/img/page-lifecycle.2e646c86.png)

在视图层(`view thread`)自己先有一个初始化(`init`)过程, `inited`完成后会通知给 `appservice thread`,然后`appservice thread`开始第一次的`send initial data`

那么问题出现在视图层(`view thread`)初始化过程中, 也就是将data原本已经有了的变量更新到视图层中这一过程在卡一点的手机上会慢！！
1. 视图层做好准备, 逻辑层做好准备(等待视图层第一次初始化完成)
2. 视图层开始初始化, 将data数据初始化给视图层
3. 视图层初始化完毕, 发送通知给逻辑层, 告诉他我准备好了
4. 然后逻辑层如果onShow,onLoad有setData操作, 此时将正在处理

我猜想中间有一次 没有按照理想状态来是因为我们把变量默认当中true, 可能是第一步到第二部, 视图层初始化有的手机, 差距显示大, 视图层变量当中没有来对待

```js
     // 大约在 65271行
    // 注解, 到这一步骤, 没想到, 试图层已经出来了, 而且是默认的配置对象, 也就是说, 并不是我刚刚理解的那样
    // 第一次渲染 是在 onLoad和onShow之前的,
    Mt(Qe, n, bt.newPageTime, void 0, !1),

    R() && (__wxAppData[e] = l.data,

    __wxAppData[e].__webviewId__ = n,
    __appServiceSDK__.publishUpdateAppData());
    var _ = __appServiceSDK__._getOpenerEventChannel();
    _ && (Qe.eventChannel = _),
    debugger
    l.__callPageLifeTime__("onLoad", r), // 注解 响应 onLoad
    l.__callPageLifeTime__("onShow"), // 注解 响应 onShow

```

那么最好的处理方式是,把试图层默认变量是false去处理一些状态

-----------

## setData优化

### 针对数组
1.添加

```js
    // 一般分页可能是这样
    let list = [1,2,3];
    this.setData({
        list
    })

    // 第二页
    let list = list.concat([4,5,6]);
    this.setData({
        list
    })

    // 优化方案2
    this.setData({
        [`list${this.data.list.length}`]: list
    })

    // 或者不怕麻烦的话
    this.setData({
        'list[0]': 0,
        'list[1]': 1,
        'list[2]': 2
    })

```
也就是专门针对索引 添加, 具体应该自己写专门处理的方法去返回setData语句

2.删除

```js
    // 可以通过view下功夫
    <block wx:for="{{list}}">
        <block wx:if="{{item}}">

        </block>
    </block>

    this.setData({
        'list[3]': null
    });
```

3.改
一样针对具体的索引

>其实也就是做一件事情, 减少setData传入的对象的复杂度

-----------

## 哪些可以使用绝对路径

### 引用自定义组件
实现访问根目录
```js
    usingComponents: {
        'c-speech-recognition': '/components/CSpeechRecognition/CSpeechRecognition'
    }
```

### 组件关系引用`

```js
    relations: {
        // 下面也可以配置引用根目录
        '/custom-li': {
            type: 'child',
            linked: function(target) {}
        }
    }
```

### wxss引用
/ 实现访问根目录
```css
@import '/style';
```

### js引用

配合webpack路径别名, vscode配置 `jsconfig.json`
```js
{
    "compilerOptions": {
        "target": "es6",
        "baseUrl": "./",
        "paths": {
            // 路径别名
            "@/*": ["src/*"],
        },
        "module": "es6",
        "allowSyntheticDefaultImports": true
    },
    "include": [
        "src/**/*"
    ],
    "exclude": ["node_modules", "dist"]
}
```

`wxs不支持`

------

## 小程序框架设计想法

### 搭建私有npm和github
可以使用 verdaccio, Gogs或gitLab

### 脚手架
个人还是比较喜欢在原生小程序基础上进行扩展, 不太想采用mpvue, wepy, 主要是很多框架都号称提高性能, 实际上在一些场景, 这些框架根本就是拖累, 尤其wepy1(2待定)在list, 更改其中某一项的name你就会发现问题所在
预采用`pandora`
小程序框架想要在 `wxpage` 基础上更改

### 路由方面
想要采用路由集中化管理设置别名, 类似vue ,扩展 微信路由跳转方法, 要求统一使用自己的路由别名跳转对于的页面
(原因是因为有的时候,更改分包名,又不敢删除原来的页面, 当然最主要的原因就是为了跳转预加载等方面实现);

调用扩展的路由方面,对应唤起对应的页面的预加载的方法(扩展session等概念), 同时也可以拦截onShow方法, 可以扩展出的实现监听路由变化的方法, 不管是后退还是前进, 都可以拦截, 原先 `wxpage` 有一个很好的概念, 可以监听来自其他地方跳转到该页面的方法, 在这里再调用页面上的预加载方法, 具体看[wxpage的preload原理同wepy](#wxpage的preload原理同wepy)

需要同时提供<navigator>跳转方式,以及wx的跳转方式

在跳转方面, 跳转前将路由url编码下,用户页面的onLoad接受到的是解码后的

### 自定义组件方面
基础的自定义组件应该不依赖外部css, 可以依赖css变量, 但是主要注意设置默认值

### wxss方面

#### css3变量
```css
目前是想了很久, 最终采用了 --color, --size, --spacing这类
page {
  /* 颜色 --color开头, 至于用具体的红色蓝色命名还是success,warn来命名,我更习惯了具体的warn这里命名,只是业务太复杂 */
  --color-blue: #007bff;
  --color-red: #dc3545;
  --color-success: #28a745;
  --color-info: #17a2b8;
  --color-warning: #ffc107;

  /** 尺寸统一 x...s, xxs, xs, s, m, l, xl, xxl, xxxl, x...l, 来区分, m是标准, 可以往前往后扩展, 没有采用small,normal,之类的,主要担心尺寸会很多,还有就是方便上手,避免写的太长 **/
  --size-xs: 28rpx;
  --size-s: 28rpx;
  --size-m: 30rpx;
  --size-l: 30rpx;
  --size-xl: 30rpx;

  --spacing-s: 30rpx;
  --spacing-m: 30rpx;
  --spacing-l: 30rpx;
}

```
因为小程序 自定义组件样式隔离,app定义全局样式,也无法污染到自定义组件(有开关,但是又版本要求),除了自定义组件 import 基础样式css文件外, css3变量, 是无视,直接可用的,不需要引入, 只需要app定义好即可

全面采用css3变量, 可以很好的维系所有的自定义组件的样式(会传入), 如果之后有主题色的需求,
变量名注意抽象点

#### 可配置方案
css目前主要想分为 layout(布局), spacing(边距), colors(颜色), font(字体)
其中一些边距呀, colors, font可以参考vuetify命名, 将里面的具体的颜色,字体大小, 通过page下的变量维系起来,更改这些变量实现更改所有的尺寸

#### 主题色方案
利用css变量
```css
.page {
    --main: red
}
/* 其他主题 */
.page.dark {
    --main: green
}
```

#### 给自定义组件节点root添加样式
`写在@import里面是无效的`

最好的方式是在自定义节点加
```css
:host {
    color: green
}
```
https://developers.weixin.qq.com/miniprogram/dev/framework/custom-component/wxml-wxss.html
使用场景: 我有一个模态框, 下面有插槽, 里面添加若干个按钮并排显示,所以在插槽外部设置display:flex;
因为有shadow-root存在,btn这个设置一个flex-1 没有用,因为被封装进#shadow-root(影子根),除非直接给自定义组件上加样式
当然还有中方法就是通过 `:host`
```html
<modal>
    <view style="display:flex">
        <btn>
            #shadow-root
        </btn>

        // 次要解决方法
        <btn class="flex-1">
            #shadow-root
        </btn>
    </view>

</modal>


```
```css
// 详见红版里面的模态框
在按钮指定组件样式内写，相当于直接给根上定义样式
:host {
    flex:1
}
```

------------

## [wxpage解析未完](./WXPAGE.MD)
