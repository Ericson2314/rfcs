- Feature Name: extern-const
- Start Date: 2015-02-02
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Symbols can be used to represent any address-sized value, not just addresses to statically allocated
memory. Rust almost always interprets symbols as pointers to valid memory, with the exception of an
incongruous interpretation of statics that are given special linkage (currently behind a feature
gate). This RFC proposes a new concept of "extern const"s, which allow interpreting symbols
themselves as arbitrary types. Special linkage will only be allowed for "extern consts". In this
way, statics will gain a consistent meaning, and Rust will become more expressive.

# Motivation

Consider the following x86 assembly (NASM syntax):

1. `foo.asm`
   ```assembly
   global NUM_CORES
   NUM_CORES EQU 4
   ```

2. `bar.asm`
   ```assembly
   extern NUM_CORES
   global my_proc
   
   my_proc:
       mov ECX, NUM_CORES - 1
   loop:
       j
       ; Do Something
       dec ECX
       test ECX, ECX
       jnz loop
       ret
   ```

There is no way to write `foo` in C or Rust, because there is no way to define `NUM_CORES`. Defining
a static in Rust, or global variable in C) statically allocates memory, initializes it to 4, and
defines NUM_CORE to be the address of that memory. `foo.asm` doesn't allocate any memory at all, and
simply defines the symbol `NUM_CORES` to be 4.

There are ways to write `bar` in C or Rust, but they are unsatisfactory. In either language, one
simply make a dummy global variable `NUM_CORES`---be careful not to actually use it---and take it's
*address* to initial the loop counter. It is unfortunate that this is unsafe, because there is
nothing unsafe about treating an symbol as a number.


On a different note, consider weak linking in Rust (currently behind a feature gate):
```rust
extern {
    #[linkage = "extern_weak"]
    static weak: *const libc::c_uint;
    static strong: *const libc::c_uint;
}
```
One might assume each of these statics refers to an externally defined global variable in
memory with the type of `*const libc::c_uint`. But one would be wrong.

To facilitate the fact that a weak symbol may be uninitialized and default to 0, Rust actually
*defines* `weak` as static with the type, that is statically initialized with real weak symbols
value. In this way `weak` is always defined, but may be 0. http://is.gd/I4Ikxc (playpen link)
illustrates this.

This treatment of extern weak statics violates the normal relationship between symbols and statics,
and violates the principle that declaring an externally defined global "thing" should not allocate
more memory.


My solution is to allow consts to be "exported" as symbols and symbols to be imported as consts.
Naturally in addition to the normal restrictions on what types can be constants, the type must be
equal or smaller in size than the target's word size. The key difference is while a symbol
associated with a static *points* to the static, a symbol associated with a const *is* the
`const`. `extern { const foo: &'static T; }` means nearly the same thing as
`extern { static foo: T;}`

In the `NUM_CORES` example, one simply defines and/or declares `NUM_CORES` extern consts with a type
like `usize`. `usize` cannot be derefenced so no unsafety is introduced. In the weak symbol example,
a extern consts with type `*mut libc::uint` correctly describes the fact that a weak symbol is not
safe to dereference.


# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar
with the language to understand, and for somebody familiar with the compiler to implement.
This should get into specifics and corner-cases, and include examples of how the feature is used.

# Drawbacks

 - Makes the language bigger.
 - Purely superfluous unless interfacing with foreign code.

# Alternatives

 - Get rid of extern statics as a `&'static T` extern const works just as well.

 - Some one off trick to make weak symbols work properly.

# Unresolved questions

 - None that I can think of.
