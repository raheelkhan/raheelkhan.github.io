---
title: Understanding ownership in Rust
date: 2019-01-11
categories:
---

For 2019, I decided to learn system programming. While C is still the most popular language when it comes to writing system software but learning C can be a bit daunting managing all the low level details manually such as `malloc` and `free`

After a bit research, I decided to give a try to [Rust](https://www.rust-lang.org/) . Rust is a strongly typed **Ahead of time** compiled language that guarantees memory safety by its model of ownership.

To understand ownership we need to understand the behavior of other languages how they manage memory. For example if you want to store something in memory while working with C, you need to ask the CPU to provided your program a chunk in memory by calling `malloc` and it is your responsibility to free that memory once the variable is no longer needed.

Languages like PHP, Python, Javascript etc take this burden out of you and comes with something called “Garbage Collector”. What they do is they run Garbage Collector whose job is to go into memory and find references that are no longer valid and clean them up. This process is hidden from us as a developer and there is no way to to know when the next Garbage Collector will run.

Most of the time this is fine as now we have more advanced hardware and scaling horizontally is pretty cheap. But when we talk about system software such as Webservers , Kernels, Drivers or Embedded programming etc, We have to be careful of not wasting the resource or not deleting any data which is meant to be use down the road.

Rust comes with a third approach what they call **Ownership**. This system depends on 3 rules

1. Each value in Rust has a variable that’s called its owner.
2. There can only be one owner at a time.
3. When the owner goes out of scope, the value will be dropped.

When we assign a value to a variable the compiler asks the system a piece of memory. The memory (RAM) has two portions within it called **Stack** and **Heap**.

The analogy to understand Stack is like a pile of plates. When we have to put a plate on a pile we put it on top of it. When taking out, we take it out from the top. It works in a model which is known as LIFO (last in first out). When this pile reaches its upper bound i.e there is not enough space in a cabinet to put more plates, we refer to this term as stack overflow.

To Understand Heap the Rust Book has given a great example of a restaurant and reception. Its like a group of 5 friends went to a restaurant and ask the receptionist for a table of 5. She will point them to a certain table to be seated. If another friend later want to join them, the receptionist will have to arrange a different table of 6 persons.

If we analyse a little, we can clearly see that the stack operation is constant no matter how many plates we have. In Computer Science this is known as O(1). Whereas in case of Heap it is more cumbersome.

This is exactly the case while storing values in our program. Variables that are on Stack are faster to access where as variables on Heap are slower.

Now we have an understanding how Heap and Stack stores data , let see how Rust decides wether to store something in Heap or Stack.

Like any other strongly typed language Rust also has type. We will take String type for this discussion.

```rust
let x = String::from("Hello, World!");
let y = x;
println!("The value of x is {}", x);
```

If you try to compile this program, you will receive an error saying that x is out of scope or have been moved. Remember rules of ownership as stated above that at a given point there can only be 1 owner of value. Basically when we assign Hello, World! to x it means that x the owner of Hello, World!. When we assign y to x , Rust transfers the ownership of Hello, World! to y.

**You might be wondering that what happened to x then ?**

Every type in Rust comes with a predefined method called `drop`. As soon the variable goes out of scope i.e it is no longer in use or its value has been transferred to some other variable, Rust compiler calls this drop method to delete it. Notice this all happens at compile time NO GARBAGE COLLECTOR!

### Clone

Imagine that you still want to use x. With above explanation it looks silly isn’t it ? Rust guys are smart ! Every type as also a method **clone**.

```rust
let x = String::from("Hello, World!");
let y = x.clone();
println!("The value of x is {}", x);
println!("The value of y is {}", y);
```

### Borrowing

When Rust says that there can be only one owner of a value, it is pretty strict in that. Let see how true is that in functions calls.

```rust
fn main() {
    let x = String::from("Hello, World!");
    say_hello(x);
    println!("The value of x is {}, x");
}

fn say_hello(arg: String) {
    println!(arg);
}
```

Try compiling this program and you will see the same error complaining that x has been moved to arg. To solve this problem we have to pass x as reference so that the ownership of Hello, World! is not moved.

```rust
fn main() {
    let x = String::from("Hello, World!");
    say_hello(&x);
    println!("The value of x is {}, x");
}

fn say_hello(arg: &String) {
    println!(arg);
}
```

### Literals & Copy

So far we have been working on String type. It is worth mentioning that during Compilation Rust will put variables whose values are unknown in Heap.

**What about Stack then ?**

There is something called **Literals** in Rust. If you are coming from scripting language background, this might looks natural to you.

```rust
let x = "Hello, World!";
let num: u8 = 5;
```

Notice that here we are not using `String:from` method but directly assigning the Hello, World! to x. Assigning via literal makes sense when the data you are assigning is not supposed to be dynamic such as you might want to save some constants, or a CDN URL etc.

```rust
let num: u8 = 5;
let num2 = num;
println!("The value of num is {}, and the value of num2 is {}, num, num2);
```

The above code will be compiled successfully and you will think that this is contradictory with what we have studied above.

There is one more detail in it. Rust a a special Trait called “Copy”. Any value such as above literals or integers type whose value is known at compile time, Rust annotates them with Copy and their values will be put in Stack.

We know that the access on Stack is always O(1). This means there is no reason we should not allow num variable to keep the ownership of 5 and we can leave that.

There are few more details in this topic such as Dereferencing and Slices, Mutable References etc which I might write about in future.

By the way, Rust looks like a promising language to work on.

> [Rust tops the chart of most loved programming language on Stackoverlow Insights](Rust tops the chart of most loved programming language on Stackoverlow Insights)
