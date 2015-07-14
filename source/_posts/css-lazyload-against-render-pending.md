title: 在国内从googleapis拿字体得注意要lazyload啊
date: 2014-06-24 20:10:18
tags: [css, lazy load, google]
---

起因是在家里上着觉得挺快的博客，到了公司上就白屏很长一段时间，比较郁闷。只好打开调试找原因。

ghost默认用的主题是casper，人用到了从fonts.googleapis.com扒拉下来的牛逼字体。`default.hbs`写着：

```
<link rel="stylesheet" type="text/css" href="//fonts.googleapis.com/css?family=Noto+Serif:400,700,400italic|Open+Sans:700,400" />
```
<!-- more -->
果然是得感谢我们伟大的火墙，谷歌来的必须烧死。于是页面随着这个 request 的 pending 就一直白在那了。

在家shadowsocks用着太爽，到公司毛病才暴露出来。

于是将原来模板里的相关link标签删了，在js里面加上创建该标签的代码。好了，至少不会白屏10好几秒了，暂时就这么先改着吧。

```javascript
(function(){
  	window.onload = function(){
  	  var elem = document.createElement("link");
  	  elem.rel = 'stylesheet';
		elem.type = 'text/css';
		elem.href = '//fonts.googleapis.com/css?family=Noto+Serif:400,700,400italic|Open+Sans:700,400';
		document.body.appendChild(elem);
  	};
})();
```
