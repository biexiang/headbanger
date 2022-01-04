---
title: "支持B站视频和图片"
date: 2022-01-04T22:31:49+08:00
draft: false
categories: ['Video']
---

## 图片
下面图片使用 [路过图床](https://imgtu.com/)，因为担心图床突然关闭，还是需要一个稳定点的图床，可支持迁移导出。其次图片还需要增加一个shortCode支持展示Exif信息，和图片的小图展示，点击放大功能。
[![TO7vBn.jpg](https://s4.ax1x.com/2022/01/04/TO7vBn.jpg)](https://imgtu.com/i/TO7vBn)

## 视频
之前剪辑的快银和女巫的视频
{{< bilibili 33375125 >}}

shortcodes如下：
```
// layouts/shortcodes/bilibili.html

<style>
.aspect-ratio {
    position: relative;
    width: 100%;
    height: 0;
    padding-bottom: 75%;
    }
        
.aspect-ratio iframe {
    position: absolute;
    width: 100%;
    height: 100%;
    left: 0;
    top: 0;
    }
</style>
        

<div align=center class="aspect-ratio">
    <iframe src="https://player.bilibili.com/player.html?aid={{ index .Params 0 }}&&page=1&as_wide=1&high_quality=1&danmaku=0" 
    scrolling="no" 
    border="0" 
    frameborder="no" 
    framespacing="0" 
    allowfullscreen="true"> 
    </iframe>
</div>

```