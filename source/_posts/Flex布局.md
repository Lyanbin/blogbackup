---
title: Flex布局
date: 2017-03-05 19:42:24
tags: [css, flex]
---


父级元素设置`display`为`flex`或者`inline-flex`时，表示该元素将以弹性盒子方式布局。同时，该元素上的`float`、`clear`、`vertical-align`将全部失效。

```css
.box {
    display: flex; /*inline-flex*/
}
```

#### 基本概念：
* `flex`容器: 设置`display`为`flex`的父级元素为`flex`容器
* `flex`项目: 容器中所有的子元素，为容器的`flex`项目，即`flex item`
* 主轴(`main asix`): `flex`项目的填充方向，默认为水平方向；主轴的起始位置(`flex`容器边框的交叉点)为`main start`，结束位置为`main end`
* 交叉轴(`cross asix`): 与主轴垂直交叉的轴，叫交叉轴 ；交叉轴的起始位置为`cross start`，结束位置为`cross end`
* 主轴空间: `flex`项目占据的主轴空间为`main size`；占据的交叉轴空间为`cross size`


#### flex容器接收的属性：
```css
.box {
    display: flex; /*inline-flex*/
    flex-direction: row; /*主轴的方向，默认row(从左到右)，可选row-reverse(从右到左)、column(从上到下)、column-reverse(从下到上)*/
    flex-wrap: nowrap; /*是否换行，默认nowrap(不换行)，可选wrap(换行，第一行在上方)、wrap-reverse(换行，第一行在下方)*/
    flex-flow: row nowrap; /*flex-direction和flex-wrap的连写*/
    justiy-content: flex-start; /*容器内的项目在主轴上的对齐方式，默认flex-start(左对齐)，可选flex-end(右对齐)、center(居中)、space-between(两端对齐，左侧的去最左右侧去最右，项目间隔相等)、space-around(左右间隔平分)*/
    align-items: stretch; /*容器内的项目在交叉轴上的对齐方式，默认stretch(拉伸，如果没设置高度，则垂直填充)，可选默认flex-start(上对齐)、flex-end(下对齐)、center(居中)、baseline(第一行文字下对齐对齐，即文字大小不同时，以文字下基线对齐)*/
    align-content: stretch; /*多轴线对齐方式(多行)，如果只有一根轴线(单行)，则无效，可选flex-start、flex-end、center、space-between、space-around*/
}
```

#### flex项目接收的属性：
```css
.box > .item {
    order: 0; /*排序，越小越靠前，默认为0*/
    flex-grow: 0; /*分配剩余空间的比例，默认为0，有剩余空间也不分配*/
    flex-shrink: 1; /*缩小比例，如果容器空间不足，则按比例缩小，默认为1，如果为0则不缩小*/
    flex-basis: auto; /*分配空间前，项目占据的主轴大小，默认为auto，即为项目本身的大小*/
    flex: 0 1 auto; /*flex-grow, flex-shrink, flex-basis的连写*/
    align-self: auto; /*允许单个项目与其他不一样的交叉轴对齐方式，默认auto为继承父元素的align-items*/
}
```

