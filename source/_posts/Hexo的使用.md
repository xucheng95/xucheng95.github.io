---
title: Hexo的使用
date: 2023-06-07T16:52:31+08:00
tags: Hexo
categories: Hexo
mathjax: False
---

&emsp;&emsp;Hexo是一款基于node.js的博客框架，优势是扩展性强、部署方便、界面美观，本博客网站就是基于此框架搭建而成的。在使用Hexo的过程中，也是遇到了很多问题，本文就这些问题和解决方法进行记录，会持续补充。

&emsp;&emsp;**问题1：基于github的Hexo博客搭建流程**

&emsp;&emsp;可以参考知乎高赞专栏文章[《超详细Hexo+Github博客搭建小白教程》](https://zhuanlan.zhihu.com/p/35668237)，能解决大部分遇到的问题。

&emsp;&emsp;**问题2：代码块的字体大小大于正文**
  
&emsp;&emsp;我用的是butterfly主题，在_config.butterfly.yml中设置code-font-size: 13px，调整代码块字体的大小。多试几次，调整到最终看着比较舒服的大小。

&emsp;&emsp;**问题3：公式的字体大小大于正文**

&emsp;&emsp;在主题目录下修改mathjax.pug，例如我的路径是/themes/butterfly/layout/includes/third-party/math/mathjax.pug，将其中的scale值由原来的1.2改为1.0就能保证公式和正文对齐。


&emsp;&emsp;**问题4：无法部署到github，报出错误“remote: Support for password authentication was removed on August 13, 2021. Please use a personal access token instead.”**

&emsp;&emsp;需要重新创建token。打开github用户的Settings，找到Developer Settings，依次点击Personal access tokens、Tokens(classic)、Generate new token(classic)，输入用户密码后，填写相关的token信息，生成token。生成的token千万记得复制下来保存好，以后就不在显示了。重新使用命令 ```hexo d``` 进行部署，输入github的账户名，密码就填写刚刚生成的token，以后部署就无需输入账户名和密码了。

