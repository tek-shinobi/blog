---
title: "Frontend CSS:Common Tricks:Avoid Text Wrapping at Whitespace"
date: 2017-09-07T13:59:41+03:00
draft: false 
categories: ["frontend"]
tags: ["css"]
---
Here is how to avoid wrapping of text in elements like btn, div etc (block, or inline-block elements).

add this to the btn styling:
`white-space: nowrap;`

You will find this code often. We strip off any padding around ul because it is automatically added by browsers
```css
ul{
padding: 0;
}
```

If you want the list items to be displayed horizontally, instead of vertically, just put an inline-block
```css
ul li {
display: inline-block;
}
```
