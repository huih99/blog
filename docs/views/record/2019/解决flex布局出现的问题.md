---
title:  解决flex布局出现的问题
date: 2019-09-19
tags:
  - css
categories:
  - 笔记
---

## 最近在项目中简直被flex布局搞晕了，各种意料之外的布局问题

### 一、 flex-basis到底起什么作用
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


**1.flex-basis是指元素在flex容器中在主轴上起始的尺寸，也就是还未经过flex-grow和flex-shrink处理的尺寸，对于flex-direction:row来说flex-basis就是宽度，而对于flex-direction:column来说flex-basis就是高度。**


**2.flex-basis是指定的起始尺寸，如果flex-basis尺寸小于元素内容尺寸，则还是以元素内容尺寸(content)为准，以flex-direction：row为例，如果flex-basis小于元素内容宽度(content)，则元素的实际宽度为内容宽度**

**3.元素同时指定了flex-basis和width属性的情况，flex-basis属性优先级高于width属性，当width属性小于content宽度时，width属性将作为实际应用的content尺寸，当width属性大于content尺寸时，以实际content尺寸为准， 然后同样遵循第二点规则**

**4.如果元素给了flex-basis属性和overflow不为visible的情况，则元素尺寸等于flex-basis尺寸**

### 二、 flex属性具体什么意思

**1.flex属性是一个复合属性，是flex-grow，flex-shrink,flex-basis的复合。flex属性默认值为：flex-grow:0;flex-shrink:1;flex-basis:auto;**

**2.flex:1，其实是指定的flex-grow：1；flex-shrink:1;flex-basis:0%; flex:2则是flex-grow:2;flex-shrink:1;flex-basis:0%; 这里需要注意的是使用flex:1和flex-grow:1的区别,主要区别在于flex:1是把flex-basis设置为0%，这样才能每个元素均分父元素尺寸**

**3.当元素使用flex:1想做到均分尺寸样式时，如果元素经过flex:1处理后的尺寸小于元素内容宽/高，则实际尺寸依然以元素内容尺寸为准(未设置overflow属性的情况)**