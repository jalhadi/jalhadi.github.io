---
layout: post
title: "Cell, RefCell, and Interior Mutability in Rust"
date: 2020-07-16 19:15:43 -0700
categories: rust
---

In my self study of Rust, I've run into concepts that are different than anything else I've seen in other languages, and have had a hard time wrapping my head around. The purpose and usage of `Cell` and `RefCell` is one of those concepts. I said to myself, "why not write a blog, it'll help you, and maybe help someone else!". Putting my thoughts and inquiries in writing has been helpful for me, and I hope it's just as helpful for you!

### Background

To understand what these things are, we need to understand _Interior Mutability_ and how it relates to "normal" Rust.

What's interior mutability? Interior mutability is different than the _Inherited Mutability_ that we are so used to (even if we didn't know the technical term). Variables in Rust exhibit inherited mutability when they are defined using the `mut` keyword; this is the standard way of declaring a mutable variable. The following code is what you don't want, producing the error below it.

```rust
fn main() {
    let x = 1;
    x += 1;
    println!("The value of x is {}.", x);
}
```

Error!

```
  |
2 |     let x = 1;
  |         -
  |         |
  |         first assignment to `x`
  |         help: make this binding mutable: `mut x`
3 |     x += 1;
  |     ^^^^^^ cannot assign twice to immutable variable
```

Change `let x = 1` to `let mut x = 1`, and we solve the problem! The program correctly prints

```
The value of x is 2.
```

Ok cool, that's easy. Now what do we do when we want to update a variable in a function, but don't want to pass ownership? We just use a mutable reference!

```rust
fn add_and_print(x: &mut i32) {
    *x += 1;
    println!("The value of x in add_and_print is {}.", x);
    // mutable reference to x is dropped
}

fn main() {
    let mut x = 1; // Declare a mutable variable x
    add_and_print(&mut x); // Modify x in function
    x += 1; // Modify x again
    println!("The value of x in main is {}.", x);
}
```

which prints

```
The value of x in add_and_print is 2.
The value of x in main is 3.
```

which is expected. Cool, we can update mutable references in functions and still update them where owned. In the example above, the reference `&mut x` has inherited mutability as `mut` is being passed down directly from the current scope (as we'll see, in contrast to interior mutability).

Ok, now here's a contrived example, but represents the use case of `Cell` and `RefCell`. Let's say we have this struct.

```rust
struct XStruct<'a> {
    x: &'a mut i32,
}
```

Ok, so it's a struct that holds a mutable reference to an i32, so ownership of `x` doesn't change, and the following compiles.

```rust
#[derive(Debug)]
struct XStruct<'a> {
    x: &'a mut i32,
}

fn main() {
    let mut x = 1;
    let x_struct = XStruct { x: &mut x };

    println!("The value of x_struct is {:?}.", x_struct);
    println!("The value of x is {:?}.", x);
}
```

and we get...

```
The value of x_struct is XStruct { x: 1 }.
The value of x is 1.
```

Nice, still compiles and prints the expected result! Now let's try an mutate `x` in `main`!

```rust
#[derive(Debug)]
struct XStruct<'a> {
    x: &'a mut i32,
}

fn main() {
    let mut x = 1; // Declare a mutable variable x
    let x_struct = XStruct { x: &mut x }; // Pass a mutable reference to x

    x += 1; // Modify x...

    println!("The value of x_struct is {:?}.", x_struct);
    println!("The value of x is {:?}.", x);
}
```

and the compiler says...

```
   |
8  |     let x_struct = XStruct { x: &mut x };
   |                                 ------ borrow of `x` occurs here
9  |
10 |     x += 1;
   |     ^^^^^^ use of borrowed `x`
11 |
12 |     println!("The value of x_struct is {:?}.", x_struct);
   |                                                -------- borrow later used here
```

Ughhhhhhh, we can't do this. We've made a mutable borrow when defining `x_struct`, and it hasn't gone out of scope yet. The compiler WILL NOT let us do this. To understand why we can't do this, let's review Rust's borrow rules:

1. You can have one mutable reference.
   OR (exclusive)
2. You can have multiple immutable references.

So, our issue in the above example is that we have two references to `x`, and we want those references to be mutable. (Yeah I know, `x` in `main` isn't borrowed, `x` owns it's value, but I'd still think it be able to mutate it's own data).

You may be asking, why do we have these rules in the first place? The reason is to prevent data races, which can result in nondeterministic errors, which are just awful to deal with (I won't go deeper into data races as it's a whole other topic in computer science and programming).

Ok, we have these rules, BUT I WANT MULTIPLE MUTABLE REFERENCES. That's where interior mutability comes to the rescue!

The gist of interior mutability is that we can wrap something that's _mutable_ in a structure that is _immutable_. We can then create multiple immutable references to that thing (which follows rule #2 above), get a mutable reference when we're ready, and then mutate the value! (We'll discuss the caveats later. No free lunch, right?)

### `RefCell<T>`, YAY!

So, think of `RefCell` as the tortilla of the interior mutability burrito... (ok I'll stop). Here's an example of our failed borrow example using `RefCell`.

```rust
use std::cell::RefCell;

#[derive(Debug)]
struct XStruct<'a> {
    x: &'a RefCell<i32>,
}

fn add_and_print(x_struct: &XStruct) {
    *x_struct.x.borrow_mut() += 1; // Mutably borrow and dereference the underlying data

    println!("The value of x_struct in add_and_print is {:?}.", x_struct);
}

fn main() {
    let ref_cell_x = RefCell::new(1); // ref_cell_x is immutable
    let x_struct = XStruct { x: &ref_cell_x }; // Pass an immutable reference to ref_cell_x

    add_and_print(&x_struct); // Pass an immutable reference to x_struct

    *ref_cell_x.borrow_mut() += 1; // Modify ref_cell_x

    println!("Final value of x_struct is {:?}.", x_struct);
    println!("Final value of x is {:?}.", ref_cell_x);
}
```

IT COMPILES!!!! The output we get is...

```
The value of x_struct in add_and_print is XStruct { x: RefCell { value: 2 } }.
Final value of x_struct is XStruct { x: RefCell { value: 3 } }.
Final value of x is RefCell { value: 3 }.
```

...which is exactly what we expected! The value in `ref_cell_x` first starts as 1, we call `add_and_print` which increases the value inside to 2, then we add 1 again to the value in `ref_cell_x` and get a final value of 3, and both `ref_cell_x` and `x_struct` both have the same value. We did this all without creating a mutable reference to `ref_cell_x`!

### But what about `Cell<T>`?

`Cell<T>` is just a different way to accomplish interior mutability. With `RefCell<T>`, you get a reference to the underlying data (note the `Ref` part). With `Cell<T>`, you indirectly interact with the wrapped value by using `set` and `get` (and some other fancy methods). Here's the same example from above rewritten using `Cell<T>`.

```rust
use std::cell::Cell;

#[derive(Debug)]
struct XStruct<'a> {
    x: &'a Cell<i32>,
}

fn add_and_print(x_struct: &XStruct) {
    let x = x_struct.x.get();
    x_struct.x.set(x + 1);

    println!("The value of x_struct in add_and_print is {:?}.", x_struct);
}

fn main() {
    let cell_x = Cell::new(1);
    let x_struct = XStruct { x: &cell_x };

    add_and_print(&x_struct);

    let x = cell_x.get();
    cell_x.set(x + 1);

    println!("Final value of x_struct is {:?}.", x_struct);
    println!("Final value of x is {:?}.", cell_x);
}
```

and the output is...

```
The value of x_struct in add_and_print is XStruct { x: Cell { value: 2 } }.
Final value of x_struct is XStruct { x: Cell { value: 3 } }.
Final value of x is Cell { value: 3 }.
```

See? Same result. As you can tell, the only difference is that we first had to `get` the value out of the cell, modify it, and then `set` it again.

### When should I use `RefCell<T>` vs. `Cell<T>`?

There are two main differences between `RefCell` and `Cell`. First, types wrapped in `Cell` must implement the `Copy` trait while those in `RefCell` don't need to. This makes sense. When calling `get` on a `Cell`, you are getting a copy of the wrapped data, whereas the methods associated with `RefCell` are `borrow` and `borrow_mut`, which return references to the underlying data. Copying data back and forth does come at a performance cost, especially if the wrapped data is big, therefore making `RefCell` an attractive candidate.

The second difference is that `RefCell`'s references are checked at runtime, which comes at a performance cost of verifying reference counts and possibly being completely wrong about your borrowing logic and causing your program to panic (not good); you can't cause panics using `set` and `get` with `Cell` as you're not modifying the data through reference. Here's an example of what can go wrong with `RefCell`.

```rust
use std::cell::{RefCell};

fn main() {
    let cell = RefCell::new(1);

    let mut cell_ref_1 = cell.borrow_mut(); // Mutably borrow the underlying data
    *cell_ref_1 += 1;
    println!("RefCell value: {:?}", cell_ref_1);

    let mut cell_ref_2 = cell.borrow_mut(); // Mutably borrow the data again (cell_ref_1 is still in scope though...)
    *cell_ref_2 += 1;
    println!("RefCell value: {:?}", cell_ref_2);
}
```

This program compiles, but panics halfway through running.

```
thread 'main' panicked at 'already borrowed: BorrowMutError'
```

What? Isn't the compiler supposed to prevent these types of reference errors? Yes, for "normal" mutable references. The reference rules stated earlier still apply, they're just enforced at runtime when using `RefCell`.

And this is the double edged sword of `RefCell`s. There are times where we need more flexibility than just a singular mutable reference. `RefCell` gives us this flexibility, but at the cost of not checking our references statically.

I hope this explanation of interior mutability, `Cell`, and `RefCell` was helpful to you! Feel free to contact me if you any comments, suggestions, or just need to correct me!

### Acknowledgments

I was inspired to write this post after reading [_Learning Rust With Entirely Too Many Linked Lists_](https://rust-unofficial.github.io/too-many-lists/index.html), specifically the section ["A Bad Safe Deque"](https://rust-unofficial.github.io/too-many-lists/fourth.html) where the author explores building a doubly-linked list in Rust. I recommend that section, and the whole book in general, to get a better understanding of what a real world use case for these concepts might look like.
