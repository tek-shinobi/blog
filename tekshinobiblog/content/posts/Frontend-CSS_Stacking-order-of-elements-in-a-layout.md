---
title: "Frontend CSS:Stacking Order of Elements in a Layout"
date: 2015-08-03T11:40:26+03:00
draft: false 
categories: ["frontend"]
tags: ["css"]
---

Lets take a look at this code:

HTML:
```html
<html>
 <head></head>
 <body>
  <p>body content</p>
  <div class="block1">block1</div>
  <div class="block2">block1</div>
  <div class="float">
   <p>float</p>
   <span> class="inline">inline</span>
  </div>
  <div class="position">position</div>
 </body>
</html>
```
CSS:
```css
.block1{
 background: pink;
}
.block2{
 background: red;
 margin: -60px 0 0 20px;
}
.float{
 background: lightblue;
 float: left;
 margin: -150px 0 0 100px;
}
.inline{
 background: blue;
 color: white;
 padding: 10px 40px 10px 10px;
}
.position{
 background: orange;
 position: relative;
 left: 180px;
 top: -165px;
 height: 110px;
}
```
The natural staking order of elements in this piece of code is (in ascending order in the stack):

1. The root element (`<html>`) .. this is at the bottom of stack..level 0
2. The viewport (`<body>`)
3. Block level elements in the normal page flow
    1. block1 is displayed on top of 1 and 2.
    2. block2 stack order will be on top of block1 but in natural flow, it will be laid out below block1. If you use position: relative on block2 and then put a negative margin on it, you will see that it will be displayed on top of block1
4. Floated elements (i.e. non-positioned elements). The div with class of float will be displayed on top of 1, 2 and 3
5. Inline descendant elements in normal flow (span with class of inline will be on top of 1,2,3,4
6. Positioned elements. div with class of position is at the top.

If we want to change this stacking order, we will do so by using z-index property on the element. Note that this property only works on positioned elements (meaning those with position: relative applied to them.. using position: absolute will move them outside of the natural page flow, which is something we don’t want).

So, we can display block1 on top of stack through this:
```css
.block1{
 background: pink;
 position: relative;
 z-index: 100;
}
```
Altering stacking order through z-index is most useful for uses like displaying tool-tips and drop-down menus that always need to be displayed on top. Use this property sparingly for other scenarios.