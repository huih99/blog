---
title:  深入理解flex布局中子项的flex-basis与width属性
date: 2019-09-19
tags:
  - css
categories:
  - 笔记
---

最近在项目中简直被flex布局搞晕了，各种意料之外的布局问题，最主要的就是flex容器中的元素宽度与预想的宽度差异问题,在flex容器中，以`flex-direction:row`为前提，除了width属性外，还多了一个flex-basis属性，这两者在flex容器中的元素上都会对宽度表现造成影响。

关于`flex-basis`，先看MDN上的解释：

> flex-basis: 指定了 flex 元素在主轴方向上的初始大小。如果不使用  box-sizing 改变盒模型的话，那么这个属性就决定了 flex 元素的内容盒（content-box）的尺寸。 当一个元素同时被设置了 flex-basis (除值为 auto 外) 和 width (或者在 flex-direction: column 情况下设置了height) , flex-basis 具有更高的优先级.

`flex-basis`一直以来在我的印象中就是比`width`属性优先级更高的一个属性，我认为只要设置了flex-basis属性，那么`width`属性就会被覆盖，然而实际表现确并不是如此

在张鑫旭老师的[文章](https://www.zhangxinxu.com/wordpress/2019/12/css-flex-basis/)中有讲到，实际上在Flex布局中，一个flex子项的最终尺寸是基础尺寸、弹性增长或收缩、最大最小尺寸限制共同作用的结果。

* 基础尺寸由CSS flex-basis属性，width等属性以及box-sizing盒模型共同决定；
* 弹性增长指的是flex-grow属性，弹性收缩指的是flex-shrink属性；
* 最大最小尺寸限制指的是min-width/max-width等CSS属性，以及min-content最小内容尺寸。

其中值得注意的就是**min-content**最小内容尺寸：

`width:min-content`表示的是其内部元素中收缩过后宽度最大的那个元素作为最终宽度，这里的收缩是指一些可收缩的元素缩到最小尺寸，例如文字，在默认样式下（未显式设置宽度属性），中文可随意换行，那么最小宽度就是一个字符的宽度，而对于一串英文，则会缩小到最长的单词的宽度，对于图片来说则是其固有宽度。

有一些属性会影响到最小内容尺寸，例如文字元素，如果使用了`word-break: break-all`或者`white-space: nowrap`等影响文字折行策略的属性；对于[可替换元素](https://developer.mozilla.org/zh-CN/docs/Web/CSS/Replaced_element),如果显式的设置了width，则width就作为其最小内容宽度。对于其他类元素，当width属性小于未设置width属性时的min-content宽度时，width属性将作为实际应用的min-content尺寸，当width属性大于未设置width属性时的min-content宽度时，以该min-content尺寸为准.

### 一、 flex-basis到底起什么作用

flex-basis是指元素在flex容器中在主轴上起始的尺寸，也就是还未经过flex-grow和flex-shrink处理的尺寸，对于`flex-direction:row`来说flex-basis就是宽度，而对于flex-direction:column来说flex-basis就是高度。

```html
<!DOCTYPE html>
    <html>
        <head>
            <title>
                flex布局
            </title>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <style>
                .parent {
                    display:flex;
                    width: 500px;
                }
                .son {
                    width: 100px;
                    flex-basis: 200px;
                }
            </style>
        </head>
        <body>
            <div class="parent">
                <div class="son" />
                <div class="son" />
            </div>
        </body>
    </html>
```
在flex布局下，当flex-basis属性和width属性同时设置时，flex-basis属性优先级高于width。注意：

**当flex-basis为auto时，宽度为该元素的实际计算宽度**

**当flex-basis不为auto时**

1. flex-basis是指定的起始尺寸，如果flex-basis尺寸小于元素**min-content**尺寸，则以元素**min-content**尺寸为准，以flex-direction：row为例，如果flex-basis小于元素最小内容宽度(min-content)，则元素的实际宽度为最小内容宽度。如果flex-basis大于最小内容宽度，则元素的宽度为flex-basis指定的宽度

2. 如果元素给了flex-basis属性和overflow不为visible的情况，则元素尺寸等于flex-basis尺寸

### 二、 flex属性具体什么意思

1. flex属性是一个复合属性，是flex-grow，flex-shrink,flex-basis的复合。flex属性默认值为：flex-grow:0;flex-shrink:1;flex-basis:auto;

2. flex:1，其实是指定的flex-grow：1；flex-shrink:1;flex-basis:0%; flex:2则是flex-grow:2;flex-shrink:1;flex-basis:0%; 这里需要注意的是使用flex:1和flex-grow:1的区别，主要区别在于flex:1是把flex-basis属性设置为0%，这样才能每个元素均分父元素尺寸

3. 当元素使用flex:1想做到均分尺寸样式时，如果元素经过flex:1处理后的尺寸小于元素最小内容宽/高，则实际尺寸依然以元素最小内容尺寸为准(未设置overflow属性的情况)