- Feature Name: extern-const
- Start Date: 2015-02-02
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Symbols can be used to represent any address-sized value, not just addresses to statically allocated
memory. This RFC proposes allowing Rust to take advantage of the full expressiveness of
symbols. This is necessary to accurately  represent the semantics of weak symbols.

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

Contrived, sure? But note there is no way to write this in a way to write this in Rust (or
C). Declaring a static (global variable in C) statically allocates memory which is then loaded at
runtime, whereas here ECX is loaded with an immediate value.

My solution is to allow `const`s to be "exported" as symbols and symbols to be imported as `const`s.
Naturally in addition to the normal restrictions on what types can be constants, the type must be
equal or smaller in size than the target's word size. The key difference is while a symbol
associated with a `static` *points* to the `static`, a symbol associated with a `const` *is* the
`const`. `extern { const foo: &'static T; }` means nearly the same thing as
`extern { static foo: T; }`

The most useful application for this is to properly support externally defined weak symbols. Because
weak symbols may be undefined (in which case they default to 0), it is improper to associate them
with a static---there is no memory region to speak of if the symbol is undefined! The solution a
constant with a type such as `Option<&'static T>` or `*mut T>`. This represents the lack of safety
surrounding the weak symbol without introducing any extra indirection---an impossible task today.

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar
with the language to understand, and for somebody familiar with the compiler to implement.
This should get into specifics and corner-cases, and include examples of how the feature is used.

# Drawbacks

 - Makes the language bigger.
 - Purely superfluous unless interfacing with foreign code.

# Alternatives

 - Some one off trick to make weak symbols work properly.

# Unresolved questions

What parts of the design are still TBD?
