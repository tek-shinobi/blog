---
title: "Javascript:Generator Functions Part 2"
date: 2018-09-07T09:42:46+03:00
draft: false 
categories: ["javascript"]
tags: ["backend"]
---

We will be building on prev post titled `Javascript:Generator Functions Part 1:using Co`

Here we will focus on how to implement something analogous to co. i.e. actually implement an iterable.

## How to run the function*

Steps involved in running through a generator function. If its a bit confusing, it will be clear when we look at the implementation.

1. we first invoke the function and store it in a variable.
1. The invocation of the function returns an iterable object back to us.
1. We can call ‘next‘ on this object to take us to the first yield point in the function.
1. this call will give us back an object with the properties of value and done
1. we can continue iterating through this until we are done or stop at any point

Example
```javascript
function* foo(){
    let x = yield 'Please give mea value for x'
    console.log(x)//8

    let y = yield 'Please give mea value for y'
    console.log(y)//17

    let z = yield 'Please give mea value for z'
    console.log(z)//25

    return (x+y+z)
}

let generatingFoo = foo()
//return an iterable object stored in generating foo

generatingFoo.next()
//{value: 'Please guve me a value for x', done: false}

generatingFoo.next(8)//stores the result you pass in into x
//{value: 'Please guve me a value for y', done: false}

generatingFoo.next(17)//stores the result you pass in into y
//{value: 'Please guve me a value for z', done: false}

generatingFoo.next(25)//stores the result you pass in into z
//{value: 50, done: true}
```

So, with the basic understanding of a generator out of the way,  here is the same code implemented in `part-1` but using a function called iter that shows the implementation of the iterable
```javascript
function iter(generator){
 const iterator = generator()
 function iterate(iteration){
  if(iteration.done){
   return iteration.value
  } 
  const promise = iteration.value
  return promise.then(x => iterate(iterator.next(x)))
 }
 return iterate(iterator.next())
}

iter(function *(){
    const url = 'https://jsonplaceholder.typicode.com/posts/1'
    const response = yield fetch(url)
    const post = yield response.json()
    const title = post.title
    console.log('Title:', title)
})
.catch(error=> console.error(error.stack))
.then(x => console.log('iter resulted in ', x))
```