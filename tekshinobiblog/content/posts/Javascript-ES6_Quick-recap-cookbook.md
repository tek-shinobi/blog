---
title: "Javascript ES6:Quick Recap Cookbook"
date: 2017-09-05T11:25:18+03:00
draft: false 
categories: ["javascript"]
tags: ["javascript", "ES6"]
---
```javascript
// crreate a whole random number between [0,20) 
function wholeRandomNumber(){
    return Math.floor(Math.random() * 20)
}
console.log(wholeRandomNumber())

// random numbers within a range [min, max]
function randomRange(min, max){
    return Math.floor(Math.random()*(max-min + 1)) + min
}

console.log(randomRange(1,2))

// parseInt

// return integer from a string. It will returrn NaN if string cannot be converted to a number
function convertToIntegerr(str){
    return parseInt(str)
}
console.log("parseInt: "+ convertToIntegerr("55"))
console.log("parseInt: "+ convertToIntegerr("ABC"))

// parseInt also takes in a radix. So it can convert a binary/octal string to base 10 integer. 
// Radix is the base, like 2 for binary. Default is base 10
function convertToIntegerradix(str, base){
    return parseInt(str, base)
}
console.log("parseInt: "+ convertToIntegerradix("10001", 2))
console.log("parseInt: "+ convertToIntegerradix("a0", 16))

// Object.freeze 
// use this to make objects read-only
// note that using const makes simple variables (that contain datatypes other than object, array, dict) read-only
// objects, arrays can be mutated even after being declared const. To make these read-only, use Object.freeze
function freezeObj(){
    "use strict"
    const MATH_CONSTANTS = {
        PI: 3.14
    }
    Object.freeze(MATH_CONSTANTS)

    try {
        MATH_CONSTANTS.PI = 99
    } catch( ex ){
        console.log(ex)
    }
    return MATH_CONSTANTS.PI
}

console.log("freeze object: " + freezeObj())

// anonymous function
// below are all equivalent functions
const todaysDate = function(){
    return new Date()
}

const todaysDateArrow = () => {
    return new Date()
}
// if the function has just one line, do this
const todaysDateArrowOneLiner = () => new Date()

// Arrow funtions work very well with Higher Order Functions like map, filter and reduce
// i.e. whenever one function takes another function as an arguement, 
// its a good candidate for arrow function
// example: iterate over an array and filter all positive integers  and square them and return their sum
myArr = [1,-2, 1.5, 3.4, 5]
const sum = arr => arr.filter(num=> Number.isInteger(num) && num > 0)
                      .map(x => x**2)
                      .reduce((acc, currVal)=>acc+currVal, 0)
console.log("arrow higher order sum: " + sum(myArr))

// rest operator allows you to create a function with variable number of aguments
// rest opeartor is ...
// ...args what this does is that rest operator creates an array and calls it args
const secondSum = (...args) => args.filter(num=> Number.isInteger(num) && num > 0)
                      .map(x => x**2)
                      .reduce((acc, currVal)=>acc+currVal, 0)
console.log("arrow higher order sum using rest operator: " + secondSum(1,2,3))

// spread operator looks exactly like rest operator. three dots.
// but it expands an already existing array. So, it takes an array and spreads it out 
// into its individual parts
// you can only use spread operator in an arguement to a function or in an array literal
const myArray = [1,2,3]
console.log("arrow higher order sum using rest and spread operator: " + secondSum(...myArray))
myArrayCopy = [...myArray]

// Destructuring
// its a way to neatly assign values taken from an object, to variables  
var voxel = {x:3.6, y:7.4, z:6.4}

// we wish to get values contained inside voxel object
// old way
var a = voxel.x
var b = voxel.y
var c = voxel.z

// using destructuring. we are creating three variables a, b and c and 
// assigning them values in x, y and z properties of the object
const {x:a1, y:b1, z:c1} = voxel // a = 3.6, b = 7.4 and c = 6.4 
const {x, y, z} = voxel
console.log(`shortcut destructuring: x:${x} y:${y} z:${z}`)
// nested destructuring. We will destructure nested objects
const myNestedObj = {
    entry1: {
        first:"entry1-first",
        second:"entry1-second"
    },
    entry2: {
        first:"entry2-first",
        second:"entry2-second"
    }
}
const { entry1: { first: entry1First}} = myNestedObj
console.log(`nested destructuring: entry1First:${entry1First}`)
// use destructuring assignment to assign variables from arrays
// the difference between destructuring from an array to destructuring from an object
// is that in case of array, we cannot specify what object to pick. It just goes in 
// accordance to the order.
const [a2, b2] = [1,2,3,4]
// only first two elements will be destructured. rest are omitted.
console.log(`a1:${a2} b1:${b2} array destructuring`)
// use destruturing assignment with rest operator
const source = [1,2,3,4,5,6,7]
// I want to omit first two and capture the rest in an array
const [,,...myarr1] = source
console.log(`destruturing with rest operator: myarr1:${myarr1}`)
// use destructuring objects to pass in function arguments
const myobj = {
    min: "min-val",
    max: "max-val",
    letter: "letter-val",
    better: "better-val"
}
const myFunc = ({ min, max }) => {
    console.log(`destructured objects in function args: min:${min} max:${max}`)
}
myFunc(myobj)

// write consise object literal declarations using simple fields
// say that we have a function that creates an object and 
// the object literal has key, value pairs like so:
// long way
const createPerson_longWay = (name, age, gender) => {
    return {
        name: name,
        age: age,
        gender: gender
    }
}

// short way
const createPerson_shortWay = (name, age, gender) => ({name, age, gender})
console.log(createPerson_longWay('Ting_name long way', 20, 'ting_gender long way'))
console.log(createPerson_shortWay('Ting_name short way', 20, 'ting_gender short way'))

// write consise declarative functions
// explanation: an object can contain a function
// long way to put a function within an object
const bicycle_long_way = {
    gear: 2,
    setGear: function(newGear){
        "use strict"
        this.gear = newGear
    }
}
bicycle_long_way.setGear(3)
console.log(`bicycle long way: ${ bicycle_long_way.gear }`)
// short way to put a function within an object
const bicycle_short_way = {
    gear: 2,
    setGear(newGear){
        "use strict"
        this.gear = newGear
    }
}
bicycle_short_way.setGear(3)
console.log(`bicycle short way: ${ bicycle_short_way.gear }`)

// use class syntax to define a constructor function
// ES6 provides syntactic sugar to help create objects using class keyword
// older way to create an object
var SpaceShuttle_old = function(targetPlanet){
    this.targetPlanet = targetPlanet
}
// here the above function is essentially a constructor function that creates an 
// object via new keyword as shown below
var zeus = new SpaceShuttle_old('Jupiter')
console.log(`old way without class: ${zeus.targetPlanet}`)
// class syntax replaces the constructor function creation
class SpaceShuttle {
    constructor(targetPlanet){
        this.targetPlanet = targetPlanet
    }
}
var new_zeus = new SpaceShuttle('Jupiter new')
console.log(`new way with class: ${new_zeus.targetPlanet}`)

// use getters and setters to control access to an object
// getters and setters, although implemented as function, act like properties 
// when accessed
class Book{
    constructor(name){
        this._name = name
    }

    get name(){
        return this._name
    }

    set name(updatedName){
        this._name = updatedName
    }
}
var book = new Book('Godzilla')
console.log(`constructed book name: ${book.name}`)
book.name = 'updated Godzilla'
console.log(`updated book name: ${book.name}`)

// difference between import and require
// require was the old way of importing functions/variables from another file
// import and export are new ES6 syntax to import/export only certain parts of an external file
```

