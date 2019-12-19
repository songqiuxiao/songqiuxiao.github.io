---
title: hexo-localimg
date: 2018-12-18 23:57:07
tags:
---
我就是不用云端图片hin┻━┻ ヘ╰( •̀ε•́ ╰)

啊啊啊啊啊！我在hexo搭了几天博客了！最大的感慨就是！hexo到底怎么上传本地图片啊！！！今天终于找到了答案~

在 hexo 中使用本地图片是件非常让人纠结的事情，在markdown里的图片地址似乎永远无法和最后生成的网页保持一致。

难不成只能把图片放在服务器上？？？急乁[ᓀ˵▾˵ᓂ]ㄏ
<!-- more -->

总结了下，hexo 下插入图片现在大概有几个方案

❤ 放在根目录
❤ 可以是把图片放在 source/img 下，然后在 markdown 里写![img](/source/img/img.png)。显然这样在本地的编辑器里完全不能正确识别图片的位置。

❤ asset-image
❤ 在 hexo 2.x 时出现的插件，后来被吸纳进 hexo 3 core。比较尴尬的是，这种方法直接放弃了 markdown 原来的语法，使用类似的语法。markdown 本来有插入图片的语法不好好支持，专门用一个新的语法来插入本地图片，让我这种强迫症不太能接受。

❤ 我今天发现的新的好办法
CodeFalling/hexo-asset-image

❤ 使用方法：
❤ 首先_config.yml 中有 post_asset_folder:true 。
❤ 在 hexo 目录，执行：npm install https://github.com/CodeFalling/hexo-asset-image --save
❤ 假设在img1这样的目录结构（目录名和文章名一致），只要使用![img](/source/img/img.png)就可以插入图片。
❤ 生成的结构为img2同时，生成的 img 的 src=”/2017/01/11/my-blog/song.jpg”而不是愚蠢的 src=”my-blog/song.jpg” 值得一提的是，这个插件对于 CodeFalling/hexo-renderer-org 同样有效。

话说(￣.￣)+ 不知道大家有没遇到这个坑，就是一开始还好用，后面就开始报错了。。。。，如下：img3Cannot read property 'replace' of undefined

我调试了很久，最后发现是由于我前面写img标签的时候，html自动转码了，然后格式错误= =【摔！害我找那么久

还有什么问题大家一起分享嗷嗷~联系方式就藏在博客哪篇文章里(≧▽≦)/