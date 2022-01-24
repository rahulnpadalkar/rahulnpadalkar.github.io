---
title: "Writing Use State Hook From Scratch"
date: 2022-01-01T14:28:47+05:30
draft: false
---

> If you prefer video over text. Here’s a [video](https://youtu.be/8ULtIbhXGYU) i made a few weeks back.
> 

The `useState` hook is probably the most used hook in React. For those who don’t remember, here’s a quick refresher. 

```jsx
const [count,setCount] = useState(0)
         |      |                 |
       value  mutation       initial value
```

The `useState` hook is a function that returns, a value and a function to mutate that value. An initial value can be passed as parameter to the `useState` function. 

So if you dumb it down, **useState is basically a “stateful” function.** Stateful here means, the variable value is persisted across multiple re-runs of the function.

With that knowledge, let’s start our journey of writing `useState` hook for scratch. We will first try to build a naive solution and then build on that.

# Super Naive.

```jsx
let count = 0;
function addCount() {
	count = count + 1;
}

addCount();
console.log(count) //1
addCount();
console.log(count) //2
```

Like i said, pretty naive solution. Declaring state in global scope is pretty bad. Any other piece of code can change it and break our component “state”. Also we haven’t figured out how to set the initial state

# We can do better.

```jsx
const React = (function(){
  function useState(initVal) {
    let count = initVal;
  	function setCount() {
   		count = count + 1;
 		}
		return [count, setCount] 
  }
  return {useState}
})()

let [num, setNum] = React.useState(0);
console.log(num) //0
setNum()
console.log(num) //0 ????
```

This might seem overwhelming so let me break it down.

- An [IIFE](https://www.freecodecamp.org/news/iife-in-javascript-what/) returns an object which holds the `useState` method.
- The `useState` method accepts an initial value and returns an array with 2 items
    - `count`
    - `setCount`  → a method to mutate `count` ie. increments the count by 1.

### Dry run

On calling the `useState` hook, the count variable is set to initial value ie.`0` and that can be verified by the output of the 1st `console.log` statement.

Then to mutate the count value we call `setNum` and ....... it doesn’t work! Shoot!

The problem is, `num` isn’t reflecting the newly set value. 

# It works! (almost) 🥳.

```jsx
const React = (function(){
  let _count;
  function useState(initVal) {
    _count = _count || initVal;
    const count = () => _count
  	function setCount() {
   		_count = _count + 1;
 		}
		return [count, setCount] 
  }
  return {useState}
})()

let [num, setNum] = React.useState(0);
console.log(num()) //0
setNum()
console.log(num()) //1
console.log("🥳") // 🥳
```

### What changed?

- Renamed `count` to `_count`
- Moved `_count` into the closure of useState function. **This is important!**
- `_count` value is set to `initVal` if `_count` isn’t set.
- Created a function `count` which returns `_count`

But wait, now `count` isn’t a variable anymore. It’s a function! This isn’t how `useState` works!

# Let’s Re(a)ctify this!

```jsx
const React = (function(){
  let count;
  function useState(initVal) {
    count = count || initVal;
  	function setCount(newVal) {
   		count = newVal;
 		}
		return [count, setCount];
  }
  function render(component) {
    const a = component();
    a.render();
    return a;
  }
  return {useState, render}
})()

function wrapper() {
  const [num, setNum] = React.useState(0);
  return {
		render: () => {console.log(`Renderd with num: ${num}`)},
    click: () => {setNum(num + 1)}
	}
}

let component = React.render(wrapper); // Rendered with num: 0
component.click();                     // changes component state
component = React.render(wrapper);     // Rendered with num: 1
component.click()
component = React.render(wrapper);    // Rendered with num: 2
component.click()
component = React.render(wrapper)     // Rendered with num: 
```

### What changed?

- Added a `render` method in the object returned by IIFE.
- Added a component `wrapper` which uses the `useState` hook. The component returns an object with 2 methods →
    - `render` which logs a string to the console since we aren’t interacting with DOM here.
    - `click` a function which mutates the state. Typically this would be a event handler.

### Dry run

- The component is rendered with initial value ie.`0`.
- The `click` call updates the value of `count` internally. Since the state has been mutated, we re-render the component.
- This time during render, since `count` variable is set, the value of `num` is set to  `1` and the component is rendered with that value.

**Notice how closures help in persisting the value of `count` across multiple re-runs of the `useState` function.**

And that’s it! That’s how you write a `useState` hook from scratch.

If you found the article helpful, share it with other folks. This article is based on a talk *[Getting closure on react hooks](https://youtu.be/KJP1E-Y-xyo)* by [*swyx*](https://twitter.com/swyx). Do check it out!

 

You can follow me on [*twitter*](https://twitter.com/rahulnpadalkar). I recently tweeted about *[How dates are broken in JavaScript](https://twitter.com/rahulnpadalkar/status/1475073921137328128)*.

I also make videos on [*YouTube*](https://www.youtube.com/channel/UCrhExmTHdRNFDRlg7wz_UYA)! These are some of my most viewed ones 

- [Creating a GraphQL API in minutes with Postgraphile](https://youtu.be/AsVN_zm7lVM)
- [Vercel Server-less functions with NodeJS](https://youtu.be/af7W0Hy0LEE)
