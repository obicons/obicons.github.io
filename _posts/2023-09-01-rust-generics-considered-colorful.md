---
title: 'Rust: Generics Considered Colorful'
date: 2023-09-01
permalink: /posts/2023/09/rust-generics-considered-colorful
tags:
  - types
  - rust
---

This post shows that Rust's generics are colorful. I'll demonstrate an
example to show what I mean, and what the problems are.

# Motivating Example

Consider this silly code:
```rust
trait MyTrait {
    fn foo(&self);
}

struct S1;

impl MyTrait for S1 {
    fn foo(&self) {
        println!("S1::foo()");
    }
}

fn call_foo<T>(t: &T) where T: MyTrait {
    t.foo();
}

fn main() {
    let s1 = S1{};
    call_foo(&s1);
}
```
This seems fine so far.

Now, let's suppose we have an collection of `MyTraits`, like this:
```rust
// Previous code not shown.

struct S2;

impl MyTrait for S2 {
    fn foo(&self) {
        println!("S2::foo()");
    }
}

fn main() {
    let v: Vec<&dyn MyTrait> = vec![&S1{}, &S2{}];
    for x in v {
        call_foo(x);
    }
}
```
This produces this compilation error:
```
Compiling playground v0.0.1 (/playground)
error[E0277]: the size for values of type `dyn MyTrait` cannot be known at compilation time
  --> src/main.rs:28:18
   |
28 |         call_foo(x);
   |         -------- ^ doesn't have a size known at compile-time
   |         |
   |         required by a bound introduced by this call
   |
   = help: the trait `Sized` is not implemented for `dyn MyTrait`
```
The problem is that Rust generics are monomorphized, but monomorphization is not supported for trait objects.

`call_foo` is a [colored function](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/).
The code doesn't compile because trait objects are the wrong color.

# Does this Matter In Real Life?
Yes. Here's an example: The [Rust bindings for interacting with the Z3
theorem prover](https://docs.rs/z3/latest/z3/) have a trait
`z3::ast::Ast` to represent terms, constants, and expressions.  As
you're building a theory, you may want to maintain a vector of your
constants in a `Vec<Box<dyn z3::ast::Ast>>`. Once Z3 has constructed a
model that satisfies your theory, you'll probably want to query the model for
the values of constants via the method `pub fn get_const_interp<T: Ast<'ctx>>(&self, ast: &T) -> Option<T>`. 

Well, you just shot your foot off. You can't call this method on a
trait object, so now you need to redo the work you just did. And the
new code is going to be a *whole* lot uglier. 

# Fix 1: Prefer Trait Objects
[In contrast to the orthodox Rust
opinion](https://www.lurklurk.org/effective-rust/generics.html), we
should prefer to use trait objects *unless* we explicitly need to
combine multiple trait bounds or dynamic dispatch is a performance issue. Here's what I mean:
```rust
// Previous code not shown.

fn call_foo(x: &dyn MyTrait) {
    x.foo();
}

fn main() {
    let v: Vec<&dyn MyTrait> = vec![&S1{}, &S2{}];
    for x in v {
        call_foo(x);
    }
}
```

Note that this trait object is general enough to work with many data
structures. For example, we can still use a `Box` with this implementation:
```rust
// Previous code not shown.

fn main() {
    let v2: Vec<std::boxed::Box<dyn MyTrait>> = vec![std::boxed::Box::new(S1{}), 
                                                     std::boxed::Box::new(S2{})];
    for x in &v2 {
        call_foo(x.as_ref());
    }
    
    call_foo(v2[0].as_ref());
}
```

And, of course, we can still use `call_foo` on a specific instance:
```rust
// Previous code not shown.

fn main() {
    let s = S1{};
    call_foo(&s);
}
```

# Fix 2: Always Implement Your Traits for Trait Objects
You should just always implement your traits for trait objects:
```rust
// Previous code not shown.

impl MyTrait for &dyn MyTrait {
    fn foo(&self) {
        (**self).foo();
    }
}

fn call_foo<T>(x: &T) where T: MyTrait {
    x.foo();
}

fn main() {
    let v: Vec<&dyn MyTrait> = vec![&S1{}, &S2{}];
    for x in v {
        call_foo(&x);
    }
}
```

Note that this code also works on other kinds of trait objects:
```rust
// Previous code not shown.

fn main() {
    let v2: Vec<std::boxed::Box<dyn MyTrait>> = vec![std::boxed::Box::new(S1{}), 
                                                     std::boxed::Box::new(S2{})];
    for x in &v2 {
        call_foo(&x.as_ref());
    }

    call_foo(&v2[0].as_ref());

    let v3: Vec<std::rc::Rc<dyn MyTrait>> = vec![std::rc::Rc::new(S1{}), 
                                                 std::rc::Rc::new(S2{})];
    for x in &v3 {
        call_foo(&x.as_ref());
    }

    call_foo(&v3[0].as_ref());
}

```

If you create a trait then you must be the one that implements it for
trait objects. Per the [coherence
rule](https://doc.rust-lang.org/book/ch10-02-traits.html) a trait can
only be implemented for a type by the crate that defines the trait or
defines the type.

# Fix 3: Fix Rust
There's a lot of code in the wild that share the same pain-point as
the Z3 example I mentioned. It shouldn't be difficult to use
generics. [Effective
Rust](https://www.lurklurk.org/effective-rust/generics.html) does
explain the reason for the current design rather well. But I feel like
this is an area that can be improved on.
