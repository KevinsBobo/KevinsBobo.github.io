---
title: 将博客搭建在Github上！
comments: true
verifid: 2016101901
layout: default
category: 偷懒必须先勤快
---

> 欢迎转载，转载请注明出处！<br><br>
> [Follow me on GitHub ^\_^](http://github.com/KevinsBobo/)

关于在GitHub上搭建博客网上已经有了很多讨论，我也是看了这些讨论之后才决定将博客从搭建在OpenShift上的Wordpress转移到GitHub的，相关原理、优劣和开通方法在此不再赘述。想要从头搭建的朋友可上网搜索相关教程。本文主要记录我在不采用Jekyll等工具实现的静态页面+动态更新md文件的方法。

在GitHub上开通个人主页之后，相当于开辟了一块儿可访问的网络空间，仓库中的所有文件都可以通过Http访问，包括md纯文本文件。所以想要解决只要上传/修改md文件就可以添加/修改博客文章的问题就有了头绪。

首先通过选择官方模板文件的方式建立自己的博客页面，然后在更新之后仓库里添加article和bolg这两个文件夹。
+ article文件夹下存放MarkDowm格式的.md文件，这些是博客文章，命名格式不限。
+ blog文件夹下存放和article文件夹下md文件同名且不带后缀的文件夹，这些文件夹下都存放相同的[index.html](http://github.com/KevinsBobo/KevinsBobo.github.io/blob/master/index.html)文件。

关于这个index.html文件：

+ 是经过微调的官方主题主页文件，其中的引用都改成了绝对引用，增加了一个id为content的DIV标签。
+ 引入了[marked.min.js](http://github.com/chjj/marked/marked.min.js)这个强大的MarkDown编译库。
+ 引入了[highlight.js](http://highlightjs.org)来高亮MarkDown中的代码。
+ 增加了如下js代码来使用Marked库来编译md文件并输出博客文章：

```js
// 获取当前index.html所在的文件夹名
var dir = location.href.substring(17,location.href.lastIndexOf('/')+1);
var file;
if(dir != '/'){
	dir = dir.substring(6,dir.lastIndexOf('/'));
	file = "/article/"+dir+".md";
}else
	file = "/README.md";

// 通过文件夹名获得md文件地址
var file = "/article/"+dir+".md";

// 通过Ajax获取md文件内容
$(function(){
	$.get(file, function(result){
		// 编译md内容为html格式
		var chtml = marked(result);
		var content = $('#content');
		// 获得编译后的第一个标签文字作为页面标题
		document.title = $(chtml)[0].innerText + ' By KevinsBobo';
		// 输出html格式文章
		content.html(chtml);
		// 高亮 markdown 文档中的代码
		$('pre > code', content).each(function() {
			hljs.highlightBlock(this);
		});
	});
});
```

需要注意的地方：

+ md第一行内容最好是一级标题格式的文章标题，这样可以精准修改网页标题
+ 每在article文件夹下写一篇md文章就需要在blog文件夹下建立同名的文件夹并将根目录下的index.html文件放进去。
+ 因为文章内容是动态获取的，在网页源码中并不存在文章内容，所以搜索引擎无法抓到文章内容，注重SEO的朋友要慎重。但是Google可以抓到我存在GitHub仓库里的md文件，也就意味着我的文章可以被搜到，这对于专注于记录的我来说足够了。
+ 没有评论功能，可能对于部分朋友来说是个坑。
+ 没时间或想要直接使用的朋友可以直接克隆[我的仓库](http://github.com/KevinsBobo/KevinsBobo.github.io/)然后删除我的文章并修改根目录下的index.html和CNAME文件信息之后上传至可以创建个人主页或项目主页的仓库就可以了。
