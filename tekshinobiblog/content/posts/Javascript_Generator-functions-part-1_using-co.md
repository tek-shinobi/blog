---
title: "Javascript:Generator Functions Part 1:using Co"
date: 2018-09-07T09:32:30+03:00
draft: false 
categories: ["javascript"]
tags: ["backend"]
---
Generator functions are a concept lifted likely from Python where they are used heavily in applications like web-crawlers. There are no prerequisites to this article but if you had an idea about Python generators, its the exact same concept implemented in Javascript.

## Definition

1. Generators are functions which can be exited and later re-entered. Their context (variable bindings) will be saved across re-entrances.
2. Generators are cooperative, which means that it chooses when it will allow an interruption so that it can cooperate with the rest of the program.

## Syntax

Generators are declared using the special `*` keyword right after the word function
```javascript
function* foo(){
 let x = yield "value for x'
 let y = yield "value for y'
 let z = yield "value for z'

 return (x+y+z)
}
```
Notice another special keyword `yield` which is specific to generator functions.

## `yield/*yield`
1. yield is where the magic happens inside a function. It is used to pause the function and also give out a value.
1. yield can also take in a value when we restart the function.
1. *yield is use to delegate to another outside generator function. This means that it will pause, delegate to another generator function, complete it, and then come back to the original generator function.

## co

co is Generator based control flow goodness for nodejs and the browser, using promises, letting you write non-blocking code in a nice-ish way. [ Here is its npm link ](https://www.npmjs.com/package/co)

lets first do the prerequisites
```shell
npm install node-fetch
npm install co
```
now the sample code
```javascript
const fetch = require('node-fetch')
const co = require('co')

co(function *(){
    const url = 'https://jsonplaceholder.typicode.com/posts/1'
    const response = yield fetch(url)
    const post = yield response.json()
    const title = post.title
    console.log('Title:', title)
})
```
The stuff sent as parameter to `co()` is the generator function.