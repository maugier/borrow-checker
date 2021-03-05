---
title: WTF is Borrow Checking ?!??1
author: Max
...

## Borrow Checking

*Borrow checking* is a core feature of the *rust* programming language,
that improves *memory safety*.

- Doesn't exist in any other mainstream languages.
- Implemented using *kinds*, *generics*, and *subtyping variance*.
- Relies heavily on *sugar* and *ergonomics* to keep the code readable.

Sometimes seen as the rust beginner's **worst enemy**

# What's wrong with C

## Types catch errors !

<!-- types let us reason about the meaning of the values, even if
     it is bits and bytes all the same in the end. -->

```c
// from foo.h
int foo(int bar);
```
. . .

```c
return foo(1234);           // looks okay, i guess ?
```

. . .

<!-- there is nothing that would prevent that code from working; but
     from the types, we know that it almost certainly does not make sense,
     so the compiler will reject it unless the programmer is more explicit
     about it. -->

```c
return foo(0.01);           // probably nonsense 
return foo("hello world");  // probably nonsense
```

. . .

This has *no runtime cost*!

## Not all errors, though

<!-- Consider the following prototype: -->

```c
char *foo(char *bar, const char *baz);
```

<!-- From the types, what do we know about the proper way to use this function ? -->

. . .

```c
char hello[] = "hello";   
char *one = foo(hello, "e");      // looks okay
char *two = foo(one  , "l");      //
```

<!-- This code typechecks. We aren't doing anything wrong. -->

## Not all errors, though

<!-- But what if we do that ? -->

```c
char *foo(char *bar, const char *baz);
```

```c
char hello[] = "hello";   
char *one = foo(hello, "e");      // looks okay
char *two = foo(one  , "l");      //
free(one);                        // should you do that? should you not?
return two;                       // is this even allowed?
```
. . .

<!-- According to the type checker, all is correct.
     but what about this function ? -->

```c
char *strtok(char *str, const char *delim);
```

<!-- It has exactly the same signature as the function above. But if you used
     it in the way it is show, it would break horribly:

     - You are freeing a pointer to the stack (undefined behavior)
     - You are returning a pointer to the frame about to go away -->

## Const is broken

<!-- It is not the only issue with this signature -->

```c
char *foo(char *bar, const char *baz)
```

<!-- What does `const` mean? it means that the function promises
     that it won't try to modify the pointee.

     Does that mean you can assume that the pointee will not change?

     No: -->
. . .

```c
char *baz = ...;
const char *bar = baz;

foo(bar, baz);
```

<!-- `const` was a lie all along. It's not enough to promise you won't
     modify *one* pointee in particular, you also have to be sure that
     no other pointer points to the same location. -->

<!-- this is called the aliasing problem. Compilers do aliasing analysis
     to work around it. -->

## The Problems

1. The same type (`char *`) can represent **completely different things**
    - Pointers to the *stack* should never be *freed*
2. C's type system does not take **time** into account
    - A given pointer can be *valid in the past*
    - Then get *deallocated*, becoming *invalid*
    - Then *reallocation* may make it valid again... but with a *different type*!
3. Const is broken
    - Read-only access is only one side of the coin. The other is *aliasing*.

. . .

<!-- There are many possible solutions we can envision: -->

Possible solutions:
 - Never (de)allocate
 - Dynamic checking (GC, reference counting, `shared_ptr`/`unique_ptr`)
 - Pure language
 - `restrict`

<!-- If you never allocate any memory, then you never need to free it. Problem 1 solved!
      Even if you allocate it, you can choose to never free it, and instead offload all
      the work to the OS. Not an option for long-running programs, though!

      Dynamic checking works very fine, but in this talk we are mostly interested about
      what *static* measures we can employ.

      A pure language doesn't solve the allocation problem, but it does solve the aliasing
      one.

      `restrict` is a semi-recent addition to C that allows the type system to carry aliasing
      information. -->

## The Solutions

The type system focuses on proving correctness at *small scale*, and enable optimizations.
Complex cases can be handled by library code.

<!-- this means, for instance, that if you have a reference-counted smart pointer, you
     could copy it without touching the reference count, *iff* you can prove that the
     copy never outlives the original. -->

1. Automatic resource management, by the *ownership* mechanism
2. Cheap pointers when correctness can be proved with *borrow checking*
3. Explicit *mutables* over *shared*, instead of explicit *const* over *non-const*.

# Ownership

## Paths

<!-- Here is a picture of a running program: -->

```
 +--------+ +--------+ +--------+
 ¦ Thread ¦ ¦ Thread ¦ ¦ Thread ¦
 +--------+ +--------+ +--------+
     v         ...        ...
   Stack
   
   [main..........................]
     [f..........................]
       [g..........] [g........]
       [h.....] [i.]  [j][k..]
           [l.]

   Time ->
```

<!-- A program is a collection of threads (but we will ignore multithreading for now.)
     Each thread is associated with a context, that evolves in time. -->

 - Each thread contains a *stack* of *scopes*, containing *bindings*.

<!-- A Binding is just a formal way to say that a *value* is bound to a *name*. We don't
     call them variables because they cannot always be changed (mutated.) -->

 - A *path* is a way to access a resource, starting from a binding, then possibly following
 pointers, struct or enum members, array indices, etc.

## Principles

 - A value of type `T` is *owned* by exactly one path, at any point in time
   - Paths starting from lower stack frames are still there, even if they are shadowed. 
   - Ownership can be transfered by moving values between local variables,
or trough *function arguments* or *return values*
 - Other paths can *borrow* values

<!-- the compiler does a LOT of static checking around borrows, to ensure they are sound! -->

 - Borrows default to *shared* but can be **mutable**: `&mut T`
 - Resources like heap are managed by owned types, for instance *Boxes*: `Box<T>`

<!-- as an example, here are three function signatures in rust: -->

```rust
fn foo(x: &[u8])     -> Box<[u8]> { ... } // Always allocate, preserve the arg
fn foo(x: Box<[u8]>) -> Box<[u8]> { ... } // May reallocate, x must not be reused
fn foo(x: &[u8])     -> &[u8]     { ... } // Does not allocate. Result tied to arg.
```
<!-- all of them are refinements of the C signature we saw above; in practice, when
     compiled, all of them would receive a pointer as an argument, and return another
     pointer. But:

    Signature 1 borrows its argument. It means it will accept any pointer, promise
    not to modify it, and return a new heap-allocated pointer. It is the responsibility
    of the caller to free the returned pointer after they are done.

    Signature 2 takes ownership of the argument, and returns an owned pointer. That means
    it could be returning the same pointer after modifying the contents, or it could
    be allocating a new one and freeing the old one. Either way, the caller can only
    pass a heap pointer that it allocated itself (or received ownership from its own caller).
    After this call, the argument can no longer be used (because it may have been deallocated!).

    Signature 3 borrows its argument, and also returns a borrow. The returned borrow points to
    some part of the argument, so the returned pointer must not be freed, and it cannot be used
    after the argument has been freed. How can the compiler assume that? We'll see in a moment.
     -->

## Move Semantics

<!-- A language like C allows you to (shallow) copy any variable. This is problematic
     if we want to enforce our rule that only one path can own a resource at the same time.

     However, in most cases, copies are fine!

     To make the language easier to use, we: -->

Distinguish *Copy* and **!Copy** types

<!-- Copy types are types that are safe to copy bit by bit and forget about it.

     Non-copy types are not safe to copy; and thus, you can only use them once; after one use,
     you lose them. (In practice, of course, the memory is still there; the compiler just won't
     let you access it.)

     Note that this is only about shallow copies. Deep copy with an explicit constructor is
     always allowed, we call that *cloning*. -->

A *Copy* is in the sense of `memcpy()`. A *Clone* is a copy
using a copy (clone) constructor.

Among *Copy* types are:
    - Native *integers*
    - *Structs* with *Copy* members
    - *Arrays* of *Copy* types
    - *Borrows* (references)

Notably **!Copy** are:
    - **Box** (heap-allocated unique ptr)
    - **Rc**, **Arc** (reference-counted smart pointers)

**!Copy** variables become dead after being used in an expression,
unless it is below a borrow.

## Move Semantics 2

<!-- an example of how ownership rules work: -->

```rust
let x: Box<VeryLargeStruct> = VeryLargeStruct::create("some", "args");

do_some_foo(&*x);         // Convert the Box<T> to &T
eprintln!("x is {}", x);  // OK, x is still ours
do_something_else(x);     // This time we give up ownership

eprintln!("x is {}", x);  // This won't compile; x isn't alive
```


# Borrowing

## Lifetimes 

Borrows allow multiple paths to access the same value at the same time. The compiler
keeps track of all active borrows at any given time.

Simple approximation: a *lifetime* is a time interval during which a given scope exists.

 - The lifetime of *bindings* is the scope they are defined in
 - Borrows carry the *lifetype of their pointee* in their type: `&'a T`

## Lifetimes, an example.

```rust
fn foo<'a>(x: &'a [u8], idx: &usize) -> &'a u8 {
    return &x[*idx];
}

fn bar(x: &[u8]) -> &u8 {  // desugar to: bar<'a>(x: &'a [u8]) -> &'a u8
    let idx: usize = 3;
    return foo(x, &idx);
}

fn baz(x: &[u8]) -> &u8 {
    let buf: [u8; 5] = [0,1,2,3,4];
    return bar(&buf);            // NOPE
}
```

<!-- The `foo` function receives two pointers (borrows), and returns one. It is a
     stupid wrapper for indexing into an array.

     The `'a` is called a *lifetime annotation*, it's a variable that designates
     the scope for which the pointee of a borrow is valid. In our case, this scope
     is outside of the function, so we can't know anything about it. But the annotation
     expresses that the returned pointer's validity is tied to the validity of the *first*
     argument, but not of the second. -->

<!-- The `bar` function is an exemple of lifetime elision; if there is only one borrowed argument,
     then all returned borrows are assumed to be of the same lifetime as this single argument.  

     The signature claims that our returned borrow will be valid as long as x is valid
     This function passes the borrow checker, because:

     - The `foo` function's return value is valid as long as its first argument is valid
     - x is the first argument we pass to `foo`
     - therefore the return value of `foo(x,&idx)` is valid as long as x
     - therefore our own return value is valid as long as x
     -->


<!-- The `baz` function fails the borrow checker, because it claims the returned borrow is
     valid as long as x, but:

     - buf exists in the local scope
     - therefore &buf is only valid in the local scope
     - per the signature of `bar`, `bar(&buf)` is only valid as long as `&buf` is valid
     - therefore we cannot return `bar(&buf)` ! -->

## Lifetime Subtyping

```rust
fn test(x: &[u8], choice: bool) {
    let buf: [u8; 5] = [1,2,3,4,5];
    let y: &[u8] = &buf;

    let z: &[u8];
    if choice {
        z = x
    } else {
        z = y
    }
    do_something_with(z);
}
```

<!-- How can such a code compile ?

     We said that the lifetime of a pointer's pointee is part of its type. Here, we are trying to fit
     either x or y into the binding z, so they must have the same type. But y's pointee is on the local stack,
     while x was received as an argument, so they cannot have the same lifetime !

     However, the code can still be correct, as long as we assume the worst case for z (that is, that it
     is equal to y, not x; because if y is still valid, then x must also be!)

     This is implemented with *subtyping*: A borrow will automatically convert into a borrow of a smaller
     lifetime (a no-op in runtime, ofc).

     In this case, z has the same lifetime as y, and the assignment `z = x` automatically upcasts x. -->

## Some subtyping rules

<!-- In classical OO languages, we have a subtype operator: -->

*A <: B*: A is a *subtype* of B

In rust:

<!-- we have an *outlives* operator to denote that a lifetime is longer than another. -->

*'a : 'b* : 'a *outlives* 'b
*&'a T <: &'b T*

<!-- careful about the direction: a borrow type is a *subtype* of another if its lifetime *outlives* the other.

     This means *upcasting* means *shortening the lifetime* !

     Like upcasting never fails, shortening the lifetime is safe (or is it? we'll see..) -->

. . .

<!-- Like OO languages can have a generic Object type that is the supertype of all others, we
     have a special lifetime in our lifetime hierarchy: -->

for all 'a, `'static : 'a`

*'static* is the *bottom* of the lifetime lattice.

```
Time --->
[...'static................]
 [..main..................]
  [...f.....]   [..g.....]
    [..h.]
```

<!-- Careful! Because 'static outlives all other lifetimes, it means static borrows (pointers to static variables)
are a *subtype* of every borrow! So they are always acceptable, can be upcast into any other lifetime.

    That means they are at the *bottom* of the hierarchy, where Object was at the *top*.  -->

## Lifetimes and Mutability

<!-- This is the hardest part of the talk :) -->

