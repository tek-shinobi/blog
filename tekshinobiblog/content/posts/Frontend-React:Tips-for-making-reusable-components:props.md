---
title: "Frontend React:Tips for Making Reusable Components:props"
date: 2017-09-07T09:50:40+03:00
draft: false 
categories: ["frontend"]
tags: ["javascript", "react"]
---


## `props.children`
```javascript
const Picture = (props) => {
  return (
    <div>
      <img src={props.src}/>
      {props.children}
    </div>
  )
}
```
```javascript
render () {
  return (
    <div className='container'>
      <Picture key={picture.id} src={picture.src}>
          //what is placed here is passed as props.children  
      </Picture>
    </div>
  )
}
```
Instead of invoking the component with a self-closing tag `<Picture />` if you invoke it will full opening and closing tags `<Picture> </Picture>` you can then place more code between it and display using `this.props.children` (`props.children` for stateless components) .

This decouples the <Picture> component from its content and makes it more reusable.

