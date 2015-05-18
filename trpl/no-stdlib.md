% No stdlib

By default, `std` is linked to every Rust crate. In some contexts,
this is undesirable, and can be avoided with the `#![no_std]`
attribute attached to the crate.

```ignore
// a minimal library
#![crate_type="lib"]
#![feature(no_std)]
#![no_std]
# // fn main() {} tricked you, rustdoc!
```

Obviously there's more to life than just libraries: one can use
`#[no_std]` with an executable, controlling the entry point is
possible in two ways: the `#[start]` attribute, or overriding the
default shim for the C `main` function with your own.

The function marked `#[start]` is passed the command line parameters
in the same format as C:

```
#![feature(lang_items, start, no_std, libc)]
#![no_std]

// Pull in the system libc library for what crt0.o likely requires
extern crate libc;

// Entry point for this program
#[start]
fn start(_argc: isize, _argv: *const *const u8) -> isize {
    0
}

// These functions and traits are used by the compiler, but not
// for a bare-bones hello world. These are normally
// provided by libstd.
#[lang = "stack_exhausted"] extern fn stack_exhausted() {}
#[lang = "eh_personality"] extern fn eh_personality() {}
#[lang = "panic_fmt"] fn panic_fmt() -> ! { loop {} }
# // fn main() {} tricked you, rustdoc!
```

To override the compiler-inserted `main` shim, one has to disable it
with `#![no_main]` and then create the appropriate symbol with the
correct ABI and the correct name, which requires overriding the
compiler's name mangling too:

```ignore
#![feature(no_std)]
#![no_std]
#![no_main]
#![feature(lang_items, start)]

extern crate libc;

#[no_mangle] // ensure that this symbol is called `main` in the output
pub extern fn main(argc: i32, argv: *const *const u8) -> i32 {
    0
}

#[lang = "stack_exhausted"] extern fn stack_exhausted() {}
#[lang = "eh_personality"] extern fn eh_personality() {}
#[lang = "panic_fmt"] fn panic_fmt() -> ! { loop {} }
# // fn main() {} tricked you, rustdoc!
```


The compiler currently makes a few assumptions about symbols which are available
in the executable to call. Normally these functions are provided by the standard
library, but without it you must define your own.

The first of these three functions, `stack_exhausted`, is invoked whenever stack
overflow is detected.  This function has a number of restrictions about how it
can be called and what it must do, but if the stack limit register is not being
maintained then a thread always has an "infinite stack" and this function
shouldn't get triggered.

The second of these three functions, `eh_personality`, is used by the
failure mechanisms of the compiler. This is often mapped to GCC's
personality function (see the
[libstd implementation](../std/rt/unwind/index.html) for more
information), but crates which do not trigger a panic can be assured
that this function is never called. The final function, `panic_fmt`, is
also used by the failure mechanisms of the compiler.

## Using libcore

> **Note**: the core library's structure is unstable, and it is recommended to
> use the standard library instead wherever possible.

With the above techniques, we've got a bare-metal executable running some Rust
code. There is a good deal of functionality provided by the standard library,
however, that is necessary to be productive in Rust. If the standard library is
not sufficient, then [libcore](../core/index.html) is designed to be used
instead.

The core library has very few dependencies and is much more portable than the
standard library itself. Additionally, the core library has most of the
necessary functionality for writing idiomatic and effective Rust code.

As an example, here is a program that will calculate the dot product of two
vectors provided from C, using idiomatic Rust practices.

```ignore
#![feature(lang_items, start, no_std, core, libc)]
#![no_std]

# extern crate libc;
extern crate core;

use core::prelude::*;

use core::mem;

#[no_mangle]
pub extern fn dot_product(a: *const u32, a_len: u32,
                          b: *const u32, b_len: u32) -> u32 {
    use core::raw::Slice;

    // Convert the provided arrays into Rust slices.
    // The core::raw module guarantees that the Slice
    // structure has the same memory layout as a &[T]
    // slice.
    //
    // This is an unsafe operation because the compiler
    // cannot tell the pointers are valid.
    let (a_slice, b_slice): (&[u32], &[u32]) = unsafe {
        mem::transmute((
            Slice { data: a, len: a_len as usize },
            Slice { data: b, len: b_len as usize },
        ))
    };

    // Iterate over the slices, collecting the result
    let mut ret = 0;
    for (i, j) in a_slice.iter().zip(b_slice.iter()) {
        ret += (*i) * (*j);
    }
    return ret;
}

#[lang = "panic_fmt"]
extern fn panic_fmt(args: &core::fmt::Arguments,
                    file: &str,
                    line: u32) -> ! {
    loop {}
}

#[lang = "stack_exhausted"] extern fn stack_exhausted() {}
#[lang = "eh_personality"] extern fn eh_personality() {}
# #[start] fn start(argc: isize, argv: *const *const u8) -> isize { 0 }
# fn main() {}
```

Note that there is one extra lang item here which differs from the examples
above, `panic_fmt`. This must be defined by consumers of libcore because the
core library declares panics, but it does not define it. The `panic_fmt`
lang item is this crate's definition of panic, and it must be guaranteed to
never return.

As can be seen in this example, the core library is intended to provide the
power of Rust in all circumstances, regardless of platform requirements. Further
libraries, such as liballoc, add functionality to libcore which make other
platform-specific assumptions, but continue to be more portable than the
standard library itself.

