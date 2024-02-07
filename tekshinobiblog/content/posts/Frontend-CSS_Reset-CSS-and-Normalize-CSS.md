---
title: "Frontend CSS:Reset CSS and Normalize CSS"
date: 2015-07-02T11:49:24+03:00
draft: false 
categories: ["frontend"]
tags: ["css"]
---

Often there is some guesswork involved when styling elements due to mystery spaces and padding appearing from heaven. These are due to default styling applied by browsers and can cause some confusion as they vary between different browser vendors. To remedy this, many leading developers in the frontend community have come up with two solutions:

## Reset CSS stylesheet:

This CSS stylesheet aims to remove all default styling applied. After this stylesheet is applied, the all elements in browsers across the vendors return to unstyled state. Then we can start from there and add our styles exactly the way we like.

Check [ here ](https://meyerweb.com/eric/tools/css/reset/) for an example.

## Normalize CSS stylesheet: 

This CSS stylesheet aims to make all default styling consistent across browsers rather than removing them. After this stylesheet is applied, the all elements in browsers across the vendors return to same styled state. Then we can start from there and add our styles exactly the way we like.

Check [ here ](http://necolas.github.io/normalize.css/) for an example.

In reality, we start with some CSS  framework like Bootstrap or materialize. These often have either Reset or Normalize stylesheets applied in some way. These are however useful if we are building our own CSS style framework from scratch.