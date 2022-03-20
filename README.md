# Cubob
Some Rust helpers for structural output in display mode.

## Name
This project has autogenerated name, thanks to [This Word Does Not Exist](https://thisworddoesnotexist.com) project.
The word definition can be checked [here](https://l.thisworddoesnotexist.com/SHko).

## Purpose
Rust standard library provides some nice output primitives as methods of [std::fmt::Formatter](https://doc.rust-lang.org/std/fmt/struct.Formatter.html): [debug_list](https://doc.rust-lang.org/std/fmt/struct.Formatter.html#method.debug_list), [debug_struct](https://doc.rust-lang.org/std/fmt/struct.Formatter.html#method.debug_struct) and so on. 
Moreover, it also provides an amazing derive macro for [std::fmt::Debug](https://doc.rust-lang.org/std/fmt/trait.Debug.html), which perfectly performs all the routine of implementing usage of mentioned primitives. 
Unfortunately, there no display-mode analogs of those primitives, and there are no derive macro for [std::fmt::Display](https://doc.rust-lang.org/std/fmt/trait.Display.html) itself. 
The last fact is stated in the documentation right with the explanation: it is so to encourage developers to implement std::fmt::Display accordingly to the related type purpose. 
I agree with that approach, but I don't believe, however, that this explanation is enough to avoid implementing any suitable primitives at all, whereas existing primitives are too debug-oriented (for example, they always print type name, or always print None values even when it isn't actually needed in display mode).
This library/crate/repo is a little attempt to fullfill mentiond gap: it provides some tiny primitves for discussed cases (see examples).

## Examples
Lets consider the next structure:
```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}
```
With standard std::fmt::Debug implementation via derive macro it will have the next output for `{:?}` format:
```console
Point { x: 0, y: 0 }
```
And the next output with prettified (`{:#?}`) format:
```console
Point { 
    x: 0, 
    y: 0,
}
```
Continuing the example, one can describe the next structure with related outputs (accordingly):
```rust
#[derive(Debug)]
struct Line {
    a: Point,
    b: Point,
}
```
```console
Line { a: Point { x: 0, y: 0 }, b: Point { x: 1, y: 1 } }
```
```console
Line { 
    a: Point { 
        x: 0, 
        y: 0, 
    }, 
    b: Point { 
        x: 1, 
        y: 1 ,
    } 
}
```
The same output will be reproduced if one implements it for display mode (actually, this is the way it is implemented for debug mode "inside" the derive macro):
```rust
impl Display for Point {
    fn fmt(&self, f: &mut Formatter<'_>) -> FmtResult {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}

impl Display for Line {
    fn fmt(&self, f: &mut Formatter<'_>) -> FmtResult {
        f.debug_struct("Line")
            .field("a", &self.a)
            .field("b", &self.b)
            .finish()
    }
}
```
Such output in display mode can be too detailed (I personally found type information quite annoying). One can do slightly better using some tricks with std::fmt::Formatter methods:
```rust
impl Display for Point {
    fn fmt(&self, f: &mut Formatter<'_>) -> FmtResult {
        f.debug_map()
            .entry(&"x", &self.x)
            .entry(&"y", &self.y)
            .finish()
    }
}

impl Display for Line {
    fn fmt(&self, f: &mut Formatter<'_>) -> FmtResult {
        f.debug_map()
            .entry(&"a", &self.a)
            .entry(&"b", &self.b)
            .finish()
    }
}
```
Here is the result (prettified to simplify reading):
```console
{
    "a": Point {
        x: 0,
        y: 0,
    },
    "b": Point {
        x: 1,
        y: 1,
    },
}
```
Type name for `Line` has been removed, but stayed for `Potin` - because due to call of `debug_map` (*debug_* !) it uses debug mode for outputting all entries, and provided `std::fmt::Display` implementation isn't even used.
Moreover, this approach puts `"` symbols around field names as they are treated like map keys, which is fair, but seems redundant in considered case.
The next attempt will deal with both mentioned effect:
```rust
use std::format_args;

impl Display for Point {
    fn fmt(&self, f: &mut Formatter<'_>) -> FmtResult {
        f.debug_set()
            .entry(&format_args!("x: {:#}", self.x))
            .entry(&format_args!("y: {:#}", self.y))
            .finish()
    }
}

impl Display for Line {
    fn fmt(&self, f: &mut Formatter<'_>) -> FmtResult {
        f.debug_set()
            .entry(&format_args!("a: {:#}", self.a))
            .entry(&format_args!("b: {:#}", self.b))
            .finish()
    }
}
```
This approach reaches our goal for prettified format:
```console
{
    a: {
        x: 0,
        y: 0,
    },
    b: {
        x: 1,
        y: 1,
    },
}
```
But it has a problem for non-prettified one:
```console
{a: {
    x: 0,
    y: 0,
}, b: {
    x: 1,
    y: 1,
}}
```
The reason is in format strings like `"a: {:#}"`. It doesn't matter for `Point` due to scalar values of its fields, but it strikes back for `Line`, because its fields are structs themselves: they are always outputted prettified, because related format string specify that no matter what - even if the `Line` examplar itself is being outputted as non-prettified.
To avoid those problems one should inject some variation in the output code:
```rust
impl Display for Point {
    fn fmt(&self, f: &mut Formatter<'_>) -> FmtResult {
        if f.alternate() {
            f.debug_set()
                .entry(&format_args!("x: {:#}", self.x))
                .entry(&format_args!("y: {:#}", self.y))
                .finish()
        } else {
            f.debug_set()
                .entry(&format_args!("x: {}", self.x))
                .entry(&format_args!("y: {}", self.y))
                .finish()
        }
    }
}

impl Display for Line {
    fn fmt(&self, f: &mut Formatter<'_>) -> FmtResult {
        if f.alternate() {
            f.debug_set()
                .entry(&format_args!("a: {:#}", self.a))
                .entry(&format_args!("b: {:#}", self.b))
                .finish()
        } else {
            f.debug_set()
                .entry(&format_args!("a: {}", self.a))
                .entry(&format_args!("b: {}", self.b))
                .finish()
        }
    }
}
```
The result is:
```console
Non-prettified: {a: {x: 0, y: 0}, b: {x: 1, y: 1}}
Prettified: {
    a: {
        x: 0,
        y: 0,
    },
    b: {
        x: 1,
        y: 1,
    },
}
```
It works! But the code became quite messy, Its no big deal for small structs and programs, but becomes one when a program have a lot of structs with a lot of fields. So here is a Cubob solution: some abstractions which help to achieve the same goals with simplier actions:
```rust
use cubob::display_struct;

impl Display for Point {
    fn fmt(&self, f: &mut Formatter<'_>) -> FmtResult {
        display_struct(
            f,
            &[
                (&"x", &self.x),
                (&"y", &self.y),
            ],
        )
    }
}

impl Display for Line {
    fn fmt(&self, f: &mut Formatter<'_>) -> FmtResult {
        display_struct(
            f,
            &[
                (&"a", &self.x),
                (&"b", &self.y),
            ],
        )
    }
}
```
This code produces the same behaviour as the previous, but lets to keep code simplier and clearer.