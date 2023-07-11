---
title: Iterating Javascript object the hard way
date: 2019-04-16
categories:
---

As a developer, we use loops in our daily life. Almost every language out there as a variation of loop. But Javascript has about half a dozen of them. `for`, `for..of`, `for..in`, `forEach`, `while`, `do..while`. Each one of them has a different use case and implementation. But they all are here for the same reason. **Iteration**.

Have you ever came across the error while trying to for of loop over objects properties

```
TypeError: x is not iterable
```

Chances are you did, and without thinking about the internals trying to switch to the normal for loop or use `Object.keys()`, `Object.values()` and `Object.entries()`

Well what is Iterable ? A literal meaning will be anything that we can loop over ? Right ?

That is not exactly how we describe Iterable. An Iterable in Javascript is something that implements the [Iterator Protocol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#The_iterator_protocol)

Objects in Javascript are not iterable. Although some built-in objects are. But the good news is that you can make a custom object iterable.

```javascript
let person = {
	name: "Raheel",
	age: 30,
	location: "Karachi"
};

person[Symbol.iterator] = () => {
	let values = Object.values(person);
	return {
		next() {
			let done = values.length === 0;
			let value = !done && values.shift();

			return {
				done,
				value
			};
		}
	};
};

for (let x of person) {
	console.log(x); // Gives Raheel, 30, Karachi
}
```

In order to make an object iterable, You must implement the Iterator Protocol. When you call the for..of loop on and iterable object, It calls the Symbol.iterator once and only once and receive instance of iterator. **(Not the object on which iterator is implemented)**

On every subsequent iteration, the Javascript engine will call the next method in the iterator object.

The semantics of the return object is `{ done: Boolean, value: Any }`. As long as the done value is false, the engine will keep on calling the next method and there is no limit for that. So it is the responsibility of the programmer to provide a mechanism for ending the loop such as there is a check on arrays length in the next method.

Obviously you should not use it in your day to day life when you can simply use easier methods but It is good to know what is happening under the hood.

By the way, you can always use [Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map) if you want an iterable object because Maps are iterable by default.
