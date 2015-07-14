title: 为你的ghost加上disqus
date: 2014-09-18 20:10:18
tags: [disqus, ghost]
---
如果你想为你的Ghost博客带来一点社交化的属性，偏偏Ghost官方并没有提供评论功能，而你自己也懒得写一套评论系统，那么使用第三方评论托管，是个不错的选择。
<!--more-->
[**Disqus**](https://disqus.com/)就是这样一家第三方评论系统, 为你的网站提供评论托管服务。

它的使用很简单，比如，如果你希望评论框附加在每篇文章的最下方，那么以默认主题casper为例：

打开themes/caper/post.hbs， 在`footer`标签下方添加以下html：

```html
<div id="disqus_thread"></div>
<script type="text/javascript">
	var disqus_shortname = 'x1a0l0ng';
	var disqus_identifier = '{{url absolute=false}}';
	(function() {
		var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
		dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
		(document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
	})();
</script>
<noscript>Please enable JavaScript to view the <a href="http://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
<a href="http://disqus.com" class="dsq-brlink">comments powered by <span class="logo-disqus">Disqus</span></a>
```

除了`disqus_shortname`和`disqus_identifier`， disqus还提供其他参数可配，具体看[这里](https://help.disqus.com/customer/portal/articles/472098-javascript-configuration-variables)

> 似乎disqus被墙了，那么还有多说，友言等第三方可供选择。不过我这一没备案的野路子，国内服务还是先别用着吧。估摸着什么时候去备个案才行，不然就算我再根正苗红，GFW也不认啊。
> 电信宽带是好的，似乎是移动宽带的问题。
