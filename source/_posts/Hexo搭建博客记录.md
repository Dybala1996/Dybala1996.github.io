---
title: Hexo搭建博客记录
date: 2021-09-19 01:25:56
tags:
---

俗话说，好记性不如烂笔头，越发觉得记录总结的重要性，很多时候以为自己干过的事情不会忘，可一个月后忘的一干二净.....希望自己能养成记录总结的好习惯。

最近想把学习总结都系统的整理一下，之前都是放到了公司用的飞书云文档里，不得不说飞书云文档做的确实很好用，但还是想把自己的学习所获放到属于自己的空间中，遂照着[网上教程](https://www.bilibili.com/video/BV1Yb411a7ty)搭建了此博客。

这是我第一篇BLOG，使用[Typora](https://www.typora.io/)编辑器，顺便系统的学习一下Markdown语法。

---

# 需要的依赖

- git
- Node.js
- npm
- cnpm (npm的国内镜像源)
- hexo

# 具体步骤

## 安装git

太简单了，略。

但是别忘了配置一下git邮箱和用户名，否则后续往远端push代码时可能会产生错误。

```bash
git config --global user.email xxx
git config --global user.name xxx
```

## 安装Node.js

直接去[官网](https://nodejs.org)下就好了，安装完后会自动包含npm工具。

## 安装cnpm

单纯为了加快下载速度，使用国内的镜像源。

```bash
npm install -g cnpm --registry=http://registry.npm.taobao.org	
```

## 安装Hexo框架

```bash
cnpm install -g hexo-cli
```

## 创建工作目录

```bash
mkdir BLOG
cd BLOG
```

## 初始化Hexo

可能会报权限错误，切到root用户执行即可。

```bash
sudo hexo init
```

## 部署到GitHub仓库

创建一个GitHub仓库，要注意的是仓库名称必须是**YourGitHubName.github.io**

在工作根目录BLOG中，编辑_config.yml文件。

```text
----
#配置_config.yml 
-----
	# Deployment
	## Docs: https://hexo.io/docs/deployment.html
	deploy:
  		type: git
 		repo: 这里加你的GitHub仓库地址
  		branch: master
-----
```

部署：

```bash
hexo d
```



# 产生的问题

1. 部署到github时连接github超时

   解决办法：开梯子。

{% asset_image p1.png %}

2. 部署到github使用账号密码登陆时，报错。原因在于github从2021年8月13就开始使用token登陆，好像是为了提高安全性啥的。

   解决办法：不要用自己的github密码登陆，直接使用tokens作为密码来登录。[创建token的方法](https://docs.github.com/en/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token)

&nbsp;
{% asset_image p2.png %}

3. 在文章中插入图片后无法显示。原因在于引用的图片路径不对。
&nbsp;
{% asset_image p3.png %}
&nbsp;
可以打开文章网址页面，使用F12查看文章中图片引用的路径，发现路径是错的。
&nbsp;
{% asset_image p4.png %}
&nbsp;
{% asset_image p5.png %}
&nbsp;
从github中仓库可以看到，在博客中的图片经过hexo到处理实际上是上传到了如下位置
&nbsp;
{% asset_image p6.png %}
&nbsp;
​	解决办法：首先要安装一个插件，这个插件安装后当使用hexo new一篇新文章时，会在文章目录下新建一个与文章名字相同的文件夹用来存储文章中引用的图片，并且可以将引用的图片路径自动转换为推送到github后的图片所处的仓库路径地址。

```bash
npm install https://github.com/CodeFalling/hexo-asset-image --save
```

修改插件内容，这步必须做，否则后续图片到路径前会多出来一个/.io 路径，同样会导致图片加载不出来。

```text
'use strict';
var cheerio = require('cheerio');

// http://stackoverflow.com/questions/14480345/how-to-get-the-nth-occurrence-in-a-string
function getPosition(str, m, i) {
  return str.split(m, i).join(m).length;
}

var version = String(hexo.version).split('.');
hexo.extend.filter.register('after_post_render', function(data){
  var config = hexo.config;
  if(config.post_asset_folder){
    	var link = data.permalink;
	if(version.length > 0 && Number(version[0]) == 3)
	   var beginPos = getPosition(link, '/', 1) + 1;
	else
	   var beginPos = getPosition(link, '/', 3) + 1;
	// In hexo 3.1.1, the permalink of "about" page is like ".../about/index.html".
	var endPos = link.lastIndexOf('/') + 1;
    link = link.substring(beginPos, endPos);

    var toprocess = ['excerpt', 'more', 'content'];
    for(var i = 0; i < toprocess.length; i++){
      var key = toprocess[i];
 
      var $ = cheerio.load(data[key], {
        ignoreWhitespace: false,
        xmlMode: false,
        lowerCaseTags: false,
        decodeEntities: false
      });

      $('img').each(function(){
		if ($(this).attr('src')){
			// For windows style path, we replace '\' to '/'.
			var src = $(this).attr('src').replace('\\', '/');
			if(!/http[s]*.*|\/\/.*/.test(src) &&
			   !/^\s*\//.test(src)) {
			  // For "about" page, the first part of "src" can't be removed.
			  // In addition, to support multi-level local directory.
			  var linkArray = link.split('/').filter(function(elem){
				return elem != '';
			  });
			  var srcArray = src.split('/').filter(function(elem){
				return elem != '' && elem != '.';
			  });
			  if(srcArray.length > 1)
				srcArray.shift();
			  src = srcArray.join('/');
			  $(this).attr('src', config.root + link + src);
			  console.info&&console.info("update link as:-->"+config.root + link + src);
			}
		}else{
			console.info&&console.info("no src attr, skipped...");
			console.info&&console.info($(this));
		}
      });
      data[key] = $.html();
    }
  }
});

```

修改_config.yml中的内容

```text
post_asset_folder: true
```

此后，在文章中插入图片时不能采用markdown的语法，使用如下方式插入图片，虽然不能在markdown中预览，但最终是可以在你的文章页面中出现的。注意：example.jpg是在文章的图片存储目录中，也就是与文章所处统一路径的同名文件夹下，前面不需要加任何路径。

```text
{% asset_image example.jpg %}
```
