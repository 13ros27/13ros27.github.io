+++
title = "Designing a const `array::from_fn` in stable Rust"
date = 2024-12-04
[extra]
toc = true
+++

### The Problem
`const` functions in Rust have been steadily increasing in functionality since their introduction in [1.31](https://blog.rust-lang.org/2018/12/06/Rust-1.31-and-rust-2018.html#const-fn) however they still have major limitations, leaving many things still inexpressible. One such function is `core::array::from_fn`, which could be very useful in constants, as Guillaume Endignoux explained in his [blog post](https://gendignoux.com/blog/2024/06/17/const-array-from-fn.html) earlier this year.

The goal is at the simple end to be able to generate something like
```rust
const MULTIPLES_OF_2: [usize; 10] = [0, 2, 4, 6, 8, 10, 12, 14, 16, 18];
```
with a function rather than needing to manually type the values.
```rust
const MULTIPLES_OF_2: [usize; 10] = core::array::from_fn(|n| n * 2);
```
Unfortunately however running this (as of Rust 1.83) will error, and it seems unlikely to change anytime soon.
```rust
error[E0015]: cannot call non-const fn `std::array::from_fn::<usize, 10, {closure@src/main.rs:1:58: 1:61}>` in constants
```

### Generating it manually
While we can't just call an existing function, we can use a loop to create this ourselves, which can be useful for long or complicated arrays.
```rust
use core::mem::{transmute, MaybeUninit};

const MULTIPLES_OF_2: [usize; 10] = {
    let mut array = [const { MaybeUninit::uninit() }; 10];
    let mut i = 0;
    while i < 10 { // <-- Note 1
        array[i] = MaybeUninit::new(i * 2);
        i += 1;
    }
    // SAFETY: We initialised each `MaybeUninit` in the loop so we can `assume_init`
    unsafe { transmute(array) } // <-- Note 2
};
```
There are a couple of slight oddities here, marked with comments:
1. We have to use a `while` loop to fake a `for` loop here because the latter isn't supported in `const` (without [const_for](https://github.com/rust-lang/rust/issues/87575))
2. This `transmute` is standing in for `MaybeUninit::array_assume_init` as that is currently unstable (without [maybe_uninit_array_assume_init](https://github.com/rust-lang/rust/issues/96097))

Despite these limitations this is a functional solution, but its _very_ clunky, lets see if we can improve it.

The first thing to consider is whether we can place this wholesale in a function, translating the 10 to a const generic `N` and placing `i * 2` back into a passed function.
```rust
const fn from_const_fn<T, const N: usize>(cb: impl FnMut(usize) -> T) -> [T; N] {
    let mut array = [const { MaybeUninit::uninit() }; N];
    let mut i = 0;
    while i < N {
        array[i] = MaybeUninit::new(cb(i));
        i += 1;
    }
    // SAFETY: We initialised each `MaybeUninit` in the loop so we can `assume_init`
    unsafe { transmute(array) }
}
```
Well, initially the compiler complains about `[MaybeUninit<T>; N]` not necessarily being the same size as `[T; N]` (because `transmute` doesn't handle generics well, this will come back to bite us later). However if we comment out the `transmute` and replace it with `todo!()` we see a much worse error.
```rust
error[E0015]: cannot call non-const closure in constant functions
```
This seems to pretty straightforwardly dash our hopes without the [const_trait_impl](https://github.com/rust-lang/rust/issues/67792) feature, we can't make a function that uses a callback.

### What about a macro?
While we clearly can't do this with a function, declarative macros effectively copy-paste code into our program, so should work just fine. And indeed it does (sorry for ruining the surprise :D).
```rust
macro_rules! from_const_fn {
	($cb:expr, $n:expr) => {{
    	let mut array = [const { ::core::mem::MaybeUninit::uninit() }; $n];
    	let mut i = 0;
    	while i < $n {
        	array[i] = ::core::mem::MaybeUninit::new($cb(i));
        	i += 1;
    	}
        // SAFETY: We initialised each `MaybeUninit` in the loop
        //  so we can `assume_init`
    	unsafe { ::core::mem::transmute(array) }
	}};
}

const fn multiply(n: usize) -> usize {
	n * 2
}
const MULTIPLES_OF_2: [usize; 10] = from_const_fn!(multiply, 10);
```
Well _maybe_, we have this extra `n` variable which defines how long the array is, what if we leave the array length at 10 but change `n` to `15`.
```rust
error[E0512]: cannot transmute between types of different sizes, or dependently-sized types
```
Okay, fair enough, we're trying to transmute `[MaybeUninit<usize>; 15]` to `[usize; 10]` and they don't fit together. But what if we get smarter.
```rust
const fn multiply(n: usize) -> u8 {
    n as u8 * 2
}
const MULTIPLES_OF_2: [bool; 10] = from_const_fn!(multiply, 10);

>>> error[E0080]: it is undefined behavior to use this value
```
Ahhh, it appears we've done an oopsy. We're generating a `u8` and then secretly converting it to a `bool` but its then only valid to be 0 or 1 so errors with 2. When used in `const` this isn't terrible because the compiler will at least catch it for us, but it can't do that if we use this at runtime, and that's how you get [nasal demons](https://groups.google.com/g/comp.std.c/c/ycpVKxTZkgw/m/S2hHdTbv4d8J?hl=en&pli=1).

If we look at our macro we only have one use of `unsafe`, so it must be that `transmute` at the end. What we're trying to do with this is convert `[MaybeUninit<T>; $n]` to `[T; $n]` however we haven't put any limitations on it so there is no guarantee `T` will be the same for both sides (or even that `$n` is the same). However we can't just add a turbofish to specify this as we can't name the type `T` in our macro, as we don't have a type signature. Instead we need to wrap the transmute in a function that can then limit it for us.
```rust
/// # Safety
/// It is up to the caller to guarantee that all elements of the array are
///  in an initialized state.
const unsafe fn array_assume_init<T, const N: usize>(
    array: [MaybeUninit<T>; N],
) -> [T; N] {
    // SAFETY: Guaranteed by caller
    unsafe { transmute(array) }
}
```
This properly encodes what we are trying to do with our types, preventing misuse, but it also runs afoul of the `transmute` issue we saw earlier. While `transmute` is largely quite happy with __whatever__ you try to do with it, it does at least check that you aren't changing the size of your type. However these checks don't always play nicely with generics, to the extent that casting `MaybeUninit<T>` to `T` doesn't work with `transmute`, although in practice these are guaranteed to be the same size and alignment.

### Turning lead into differently-sized gold
If we want to implement this function; `transmute` won't cut it. Luckily Rust has a bit of a black sheep in the form of `union`, typically considered the realm of C inter-op (and also used in types like `MaybeUninit`). This allows us to input our value in as one type and pull it out as another, with no limitations on the types involved. With this we can make an even more wildly unsafe counterpart to a function already often considered a last resort of unsafe programming, `transmute_unchecked`.
```rust
use core::mem::ManuallyDrop;

/// # Safety
///  - The caller must follow all invariants of `mem::transmute`
///  - `size_of::<Src>() == size_of::<Dst>()`
const unsafe fn transmute_unchecked<Src, Dst>(src: Src) -> Dst {
    #[repr(C)]
    union Transmute<Src, Dst> {
        src: ManuallyDrop<Src>,
        dst: ManuallyDrop<Dst>,
    }

    let alchemy = Transmute {
        src: ManuallyDrop::new(src),
    };
    // SAFETY: Guaranteed by caller
    unsafe { ManuallyDrop::into_inner(alchemy.dst) }
}
```

While this allows us to pass in `Src` and `Dst` that are different sizes, we don't actually need that in this case, so if it's possible to enforce their equality at compile time we probably should. To do this we can use a trick not available when `transmute` was originally designed — post-monomorphisation errors.
```rust
/// # Safety
/// See `mem::transmute`
const unsafe fn transmute_const<Src, Dst>(src: Src) -> Dst {
    const fn check_equal<Src, Dst>() {
        assert!(size_of::<Src>() == size_of::<Dst>());
    }
    const { check_equal::<Src, Dst>() };

    // SAFETY:
    //  - We checked `size_of::<Src>() == size_of::<Dst>()` above
    //  - Everything else guaranteed by caller
    unsafe { transmute_unchecked::<Src, Dst>(src) }
}
```

### Putting it all together
With this we can change the call to `transmute` in `array_assume_init` to `transmute_const` and put it all together, resulting in a fully functional `from_const_fn` macro.
```rust
macro_rules! from_const_fn {
    ($cb:expr, $n:expr) => {{
        let mut array = [const { ::core::mem::MaybeUninit::uninit() }; $n];
        let mut i = 0;
        while i < $n {
            array[i] = ::core::mem::MaybeUninit::new($cb(i));
            i += 1;
        }
        // SAFETY: We initialised each `MaybeUninit` in the loop
        //  so we can `assume_init`
        unsafe { $crate::array_assume_init(array) }
    }};
}

const fn multiply(n: usize) -> u8 {
    n as u8 * 2
}
const MULTIPLES_OF_2: [u8; 10] = from_const_fn!(multiply, 10);
```

### Why does this not feel fully satisfying?
We did it, it works, but that extra number when calling it is very clunky and *should* be completely unnecessary. After all `array::from_fn` doesn't need you to specify the length, instead implying it from context. But macros don't get type elision, and we found early on that a function was not going to cut it. So what about both, a macro calling a function. *Well*, this would allow us to place the callback into the function directly (so that we don't have to pass it as an argument) and the return type should give us the `N` that we need, so it seems worth a try.
```rust
macro_rules! from_const_fn {
    ($cb:expr) => {{
        const fn from_const_fn<T, const N: usize>() -> [T; N] {
            let mut array = [const { ::core::mem::MaybeUninit::<T>::uninit() }; N];
            let mut i = 0;
            while i < N {
                array[i] = ::core::mem::MaybeUninit::new($cb(i));
                i += 1;
            }
            // SAFETY: We initialised each `MaybeUninit` in the loop
            //  so we can `assume_init`
            unsafe { $crate::transmute_const(array) }
        }

        from_const_fn()
    }};
}
```
We can even remove `array_assume_init` as now that its all wrapped in a function we can give `array` a specific type, constraining `transmute_const` sufficiently. However the compiler now complains that `$cb(i)` is returning a `u8` where its expecting `T`. And it has a point, we're no longer proving that the callback returns the same type as `array` expects. Luckily however, while we can't call functions passed as arguments, we can still pass one, and therefore use it to constrain `T`.
```rust
macro_rules! from_const_fn {
    ($cb:expr) => {{
        /// # Safety
        /// `$cb` must return the same type as the passed function `_`
        const unsafe fn from_const_fn<T, const N: usize>(
            _: ::core::mem::ManuallyDrop<impl FnMut(usize) -> T>, // <-- Note 1
        ) -> [T; N] {
            let mut array = [const { ::core::mem::MaybeUninit::<T>::uninit() }; N];
            let mut i = 0;
            while i < N {
                // SAFETY: `$cb(i)` returns `T` as guaranteed by caller
                array[i] = ::core::mem::MaybeUninit::new(unsafe {
                    $crate::transmute_const($cb(i)) // <-- Note 2
                });
                i += 1;
            }
            // SAFETY: We initialised each `MaybeUninit` in the loop
            //  so we can `assume_init`
            unsafe { $crate::transmute_const(array) }
        }

        // SAFETY: `$cb` is the passed function so it returns the same type.
        unsafe { from_const_fn(::core::mem::ManuallyDrop::new($cb)) }
    }};
}
```

There are a couple of slightly unusual additions here marked with comments:
1. We wrap our function input in a `ManuallyDrop` as otherwise the compiler complains about `Drop` in a `const` function, which isn't currently supported. While this in theory could cause it to leak memory, in practice I don't know a way of making a function that needs dropping without using `move` closures. And if we try and pass a `move` closure to this macro (even at runtime) the compiler will complain so this `ManuallyDrop` causes no issues.
2. Despite the return type of `$cb` now being guaranteed to match `T` the compiler still can't prove this so we still have to transmute it, it's just safe to do so now.

### Drop Safety
Thanks to an event sometimes known as the '[Leakpocalypse](https://cglab.ca/%7Eabeinges/blah/everyone-poops/)' it's not considered unsound in Rust to leak memory. But where we it's avoidable it _is_ impolite. If we drop a `MaybeUninit<T>` it will not run the destructor on `T` because it doesn't know whether the type is initialised or not and therefore whether to destruct it (this is the same for any `union`). But as the function writer we do know whether it's initialised or not as everything up to `i` is initialised, and everything after isn't. Why would not dropping this ever matter though, after all it will only leak memory if it crashed halfway through and then the program ends so it doesn't matter. However if the passed callback function panics halfway through and then we catch the unwinding program and force it to continue, we have just leaked memory.

To fix this we can use a `Guard` struct that implements `drop` and then simply `mem::forget` it when we have fully initialised the array. As of [Rust 1.83](https://blog.rust-lang.org/2024/11/28/Rust-1.83.0.html#new-const-capabilities) we can use mutable references in constants, making this possible on stable.
```rust
use core::{mem::MaybeUninit, ptr};

pub struct Guard<'a, T, const N: usize> {
    pub array: &'a mut [MaybeUninit<T>; N],
    index: usize,
}

impl<T, const N: usize> Drop for Guard<'_, T, N> {
    fn drop(&mut self) {
        // SAFETY: `array` must be initialised up to `index` so we can
        //  reinterpret a slice up to there as `[T]`
        let slice = unsafe {
            ptr::from_mut(self.array.get_unchecked_mut(..self.index)) as *mut [T]
        };
        // SAFETY:
        //  - `slice` is a pointer formed from a mutable slice so is
        //     valid, aligned, nonnull and unique
        //  - The values held in `slice` were generated safely so
        //     must uphold their invariants
        unsafe { ptr::drop_in_place(slice) }
        eprintln!("Panicked, dropped {} items", self.index);
    }
}

impl<'a, T, const N: usize> Guard<'a, T, N> {
    pub const fn new(array: &'a mut [MaybeUninit<T>; N]) -> Self {
        Self { array, index: 0 }
    }

    pub const fn get_index(&self) -> usize {
        self.index
    }

    /// # Safety
    ///  - `self.array` must be initialised up to and including the new `self.index`
    ///  - `self.array.len()` must be greater than `self.index`
    pub const unsafe fn increment(&mut self) {
        self.index += 1;
    }
}
```
Then we can simply use `Guard::get_index` instead of storing `i` separately in our function body. The reason we have to use getter and setter methods for `index` here is that changing `index` can be unsound if `array` isn't actually sufficiently initialised.

### Closure Support
The primary remaining annoyance with our `from_const_fn!` in comparison to `array::from_fn` is that when used in `const` we can't use a closure in it, despite the functions used often being very simple, because closures aren't supported in `const`. However this isn't a function, its a macro, so we can parse the closure ourselves and convert it into a function (provided it doesn't do anything closure specific).
```rust
macro_rules! convert_function {
    (|$var:ident $(: $_:ident)?| $(-> $__:ident)? $body:expr) => {
        /// # Safety
        /// `$body` must return `T`
        const unsafe fn callback<T>($var: usize) -> T {
            // By placing `$body` in a separate expression we prevent running
            //  `unsafe` code without a visible `unsafe` block
            let body = $body;
            // SAFETY: Guaranteed by caller
            unsafe { $crate::transmute_const(body) }
        }
    };
    ($cb:expr) => {
        /// # Safety
        /// `$cb` must return `T`
        const unsafe fn callback<T>(i: usize) -> T {
            // SAFETY: Guaranteed by caller
            unsafe { $crate::transmute_const($cb(i)) }
        }
    }
}
```
The closure signature `|$var:ident $(: $_:ident)?| $(-> $__:ident)? $body:expr` is a little complicated but all it does is recognise `|<variable>| <body>` while ignoring type signatures.

### And we're done!
With this conversion macro and the guard changes above, we're done, a fully functional `array::from_fn` equivalent that supports everything except closures borrowing from their environment, and works in `const`.
```rust
#[macro_export]
#[doc(hidden)]
macro_rules! convert_function {
    (|$var:ident $(: $_:ident)?| $(-> $__:ident)? $body:expr) => {
        /// # Safety
        /// `$body` must return `T`
        const unsafe fn callback<T>($var: usize) -> T {
            // By placing `$body` in a separate expression we prevent running
            //  `unsafe` code without a visible `unsafe` block
            let body = $body;
            // SAFETY: Guaranteed by caller
            unsafe { $crate::transmute_const(body) }
        }
    };
    ($cb:expr) => {
        /// # Safety
        /// `$cb` must return `T`
        const unsafe fn callback<T>(i: usize) -> T {
            // SAFETY: Guaranteed by caller
            unsafe { $crate::transmute_const($cb(i)) }
        }
    };
}

#[macro_export]
macro_rules! from_const_fn {
    // We have to accept `$cb` as a token stream otherwise it won't correctly match in `convert_function!`
    ($($cb:tt)*) => {{
        $crate::convert_function!($($cb)*);

        /// # Safety
        /// `$cb` must return the same type as the passed function `_`
        const unsafe fn from_const_fn<T, const N: usize>(
            _: ::core::mem::ManuallyDrop<impl FnMut(usize) -> T>,
        ) -> [T; N] {
            let mut array = [const { ::core::mem::MaybeUninit::<T>::uninit() }; N];
            let mut guard = $crate::Guard::new(&mut array);
            while guard.get_index() < N {
                // SAFETY: `$cb(i)` returns `T` as guaranteed by caller
                let val = unsafe { callback(guard.get_index()) };
                guard.array[guard.get_index()] = ::core::mem::MaybeUninit::new(val);
                guard.increment();
            }
            ::core::mem::forget(guard);
            // SAFETY: i == N so the whole array is initialised
            unsafe { $crate::transmute_const(array) }
        }

        // SAFETY: `$cb` is the passed function so it returns the same type.
        unsafe { from_const_fn(::core::mem::ManuallyDrop::new($($cb)*)) }
    }};
}

use core::{
    mem::{ManuallyDrop, MaybeUninit},
    ptr,
};

#[doc(hidden)]
pub struct Guard<'a, T, const N: usize> {
    pub array: &'a mut [MaybeUninit<T>; N],
    index: usize,
}

impl<T, const N: usize> Drop for Guard<'_, T, N> {
    fn drop(&mut self) {
        // SAFETY: `array` must be initialised up to `index` so we can
        //  reinterpret a slice up to there as `[T]`
        let slice = unsafe {
            ptr::from_mut(self.array.get_unchecked_mut(..self.index)) as *mut [T]
        };
        // SAFETY:
        //  - `slice` is a pointer formed from a mutable slice so is
        //     valid, aligned, nonnull and unique
        //  - The values held in `slice` were generated safely so
        //     must uphold their invariants
        unsafe { ptr::drop_in_place(slice) }
        eprintln!("Panicked, dropped {} items", self.index);
    }
}

impl<'a, T, const N: usize> Guard<'a, T, N> {
    pub const fn new(array: &'a mut [MaybeUninit<T>; N]) -> Self {
        Self { array, index: 0 }
    }

    pub const fn get_index(&self) -> usize {
        self.index
    }

    /// # Safety
    ///  - `self.array` must be initialised up to and including the new `self.index`
    ///  - `self.array.len()` must be greater than `self.index`
    pub const unsafe fn increment(&mut self) {
        self.index += 1;
    }
}

/// # Safety
/// See `mem::transmute`
#[doc(hidden)]
pub const unsafe fn transmute_const<Src, Dst>(src: Src) -> Dst {
    const fn check_equal<Src, Dst>() {
        assert!(size_of::<Src>() == size_of::<Dst>());
    }
    const { check_equal::<Src, Dst>() };

    // SAFETY:
    //  - We check they are the same size above
    //  - Everything else guaranteed by caller
    unsafe { transmute_unchecked::<Src, Dst>(src) }
}

/// # Safety
///  - The caller must follow all invariants of `mem::transmute`
///  - `size_of::<Src>() == size_of::<Dst>()`
const unsafe fn transmute_unchecked<Src, Dst>(src: Src) -> Dst {
    #[repr(C)]
    union Transmute<Src, Dst> {
        src: ManuallyDrop<Src>,
        dst: ManuallyDrop<Dst>,
    }

    let alchemy = Transmute {
        src: ManuallyDrop::new(src),
    };
    // SAFETY: Guaranteed by caller
    unsafe { ManuallyDrop::into_inner(alchemy.dst) }
}
```

All the code related to this can be found on [Github](https://github.com/13ros27/from_const_fn).
