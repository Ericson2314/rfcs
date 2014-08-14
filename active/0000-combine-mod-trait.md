- Start Date: (fill me in with today's date, 2014-08-07)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Traits/type classes and ML both have their advantages, but it is possible to have the best of both
worlds. This is a proposal to extend rust modules to be as powerful as their ML cousins, and as
convenient as traits, subsuming rust mods and traits.

# Motivation

## Multiple Instances, Sorting

One big reason is the ability to have multiple instances of one trait. The classic example is
ordering:

```rust
fn sort<T : Ord>(&mut Some_Container<T>)
fn sort_by<T>(&mut Some_Container<T>, compare: |&T, &T| -> Ordering)
```

Under this proposal, sort can be rewritten as :
```rust
fn sort<T, inst : Ord<T>>(&mut Some_Container<T>)
```

Admittedly, `sort_by` is still needed for a non-static choice of the comparator, but future work on
first-class modules could rectify that.

Relatedly, consider collections that rely on Ord, such as a Priority Queue, Tree Map, or Tree Set,
and suppose they were to be extended to support multiple comparisons analogous to sort_by. Storing a
comparator closure would not be safe, because binary operations--e.g. union, intersection--rely on
both maps/sets having the same ordering. A safer solution is this:

```rust
pub struct TreeMap<K, inst : Ord<K>, V> { .. }
    root: Option<Box<TreeNode<K, V>>>,
    length: uint
}
```

union can look like:
```rust
fn union<'a, T, inst : Ord<T>>(self : &'a TreeSet<T, inst>, other : &'a TreeSet<T, inst>) -> UnionItems<'a, inst, T>
```

Now the type system prevents unions of Treesets with a different ordering.


## std lib modularity and platform compatibility.

In RFC pull #____ there is discussion on how to make the std library more easily portable to other
systems. The basic problems are

 1. Preventing developers wishing to port it from needing to hunt done many system-specific
    components scattered around the standard crates.

 2. Avoiding the need to forgo the use of one part of the standard library because another part
    depends on a future that cannot be implemented for the given platform.

There is some conflict between these goals. To reuse as much code (especially trait hierarchies)
between wildly different platforms, it is necessary to break up the standard library into very many
crates, like the current libstd facade, but much more (consider and embedded system with network
access but no filesystem.)

On the other hand to make the porters' life easier it would be nice to make one `libplatform` that
contains all system specific components in one crate.

TODO...

Example with threads, impl mods and "fake sigs". Ocaml MirageOS.


# Detailed design

First, let me link www.mpi-sws.org/~dreyer/papers/mtc/main-long.pdf where this idea basically comes
from. It also seems like Ocaml Labs is currently working on adding this to OCaml under the name
"Modular Implicits".

This proposal is very dramatic. On one hand it is a big change to try out before 1.0, on the other,
it affects the ideoms of the language and library design such that it may be hard to port existing
stuff after 1.0. I am going to assume that it wouldn't be done too soon, if it all, or that if it is
this (yay!) this detailed design would probably get changed anyways. Therefore In the discussion
below, I am going to pretend higher-kinded types already exists. The allowance of HKTs simplifies
the presentation.

## Modules++

With HKT, Rust will have (roughly, I am speculating at the of the day) following kind language:

```grammer
Kind ::= Type
       |  Lifetime
       |  Kind -> Type // e.g. struct Foo<'a, T>
```

With this proposal, this becomes:

```grammer
Kind ::= Type
       |  Lifetime
       |  Kind -> Kind
       |  Sig
```

(Granted, perhaps this definition would be constrained to disallow `* -> Lifetime`.)

A "sig" is the "type of a module", or "header file". It is generalization of the traits of today. We
already sort of have them if you think of `pub` affecting the type of the module. (As always, syntax
and names are open to bikeshedding.) Sigs are nominal, like traits, struts and enums, in that when
you make them you give them a name. So the definition of a sig looks like:

```grammer
Sig-Def    ::= sig ID < (ID : Kind)* > { Sig-Field* }

Sig-Field* ::= type ID : Kind;  // abstract type, think associated types from Haskell if you are not
                                   familiar with ML.
             | Sig-Def          // always concrete, no such thing as "abstract sig"
             | Enum/Struct-Def  // Same as today
             | Type-Alias       // Same as today + HKT
             | ID : Type        // declaration of function, global var, etc
```

Module-Def should be straitforward considering the definition of Sig-Def:

```grammer
Mod-Def ::= mod ID < (ID : Kind)* > : Sig { Mod-Field* }

Mod-Field ::= Sig-Def         // If in sig, no need to include here, if not in sig, will be private
            | Enum/Struct-def // If in sig, no need to include here, or can have name of abstract type if kind matches,
                                 or can be private
            | Type-Alias      // Same as above
            | Term-Def        // definition (not decleration) of functions, global vars, etc
```

For convenience, if a module is specified without a sig, `pub` within is used to derive sig
with same name (modulo capitalization, or something. I don't have a proposal regarding name spaces.)

If an explicit sig is mentioned, field kinds, parameter kinds, and privacy must match.

Note: Since Sigs are proper kind, pull #192 is subsumed.

The unifying idea behind sigs is we can think of them as a generalization of structs in that they
can contain more sorts of things, but all such things must be "routed" at compile time.

## Getting back traits

So how to get back everything we know and love about traits and impls? Traits today can be modeled
as a sig with a single paramter of kind `Type`:

```rust
trait ID < Ty1, Ty2 ... > : Tr1 + Tr2 ...  { Stuff }
```

becomes

```rust
sig ID < Self : Type > {
    inst1 : Tr1,
    inst2 : Tr2,
    ...


    Ty1 : Type,
    Ty2 : Type,
    ...


    Stuff
}
```

The other side of traits is instance resolution. Multiple impls are nice and all, but as experience
shows the vast majority of the time one instance per type is adequate. The big advantage of traits
over ML modules is that routing everything by hand is tedious, just as having to explicitly
instantiating polymorphic functions every time would be too.

We get back automatic module resolution by allowing the user to put forth a module as the
"canonical" instance of its sig. Then everything needing a module of that sig will receive one
automatically. If the rules for defining a module do not change much from today. The
"canonicalization" of that module is like declaring today's `impl _ for _`, in that all the same
coherence rules apply.

Modules are resolved by searching through cannonicalized modules and their parameters. Notice in my
conversion from the old syntax to the new, I put the type parameters in the body, as oppose to
keeping them as parameters. TODO.... in vs out.

To sum up, Assume `Tr1` is a sub-trait of `Tr2` today's:

```rust
impl<Ty2, Ty3, Ty4> Tr1<Ty2, Ty5> for Ty1<Ty2, Ty6> { Stuff }
```
becomes

```rust
mod <inst : Tr2, Ty1 : Type, Ty2 : Type, Ty3 : Type > new-ID : Tr<Ty0<Ty1, Ty5>> {
  mod sigs_old_super_type : Tr2 = inst;

  type sigs_old_type-param_1 : Type = Ty2;
  type sigs_old_type-param_2 : Type = Ty6;

  Stuff
}
```

## `use` in the new system

The rules are simple, if one `use`s sig, The definitions of the current module will be brought into
scope, with a module of that sig as a new first `<...>` parameter. Since `<...>` parameters can have
any kind in all places they appear, this is not a problem.

```rust
sig Foo<T> {
   fn bar<T : Type>(T, T) -> T;
}

use Foo;

// bar : fn<inst : Foo, T : Type>(T, T);
```

But notice that is probably not what we wanted. Instead:

```rust
use<T : Type> Foo<T>;

// bar : fn<T : Type, inst : Foo<T>>(T, T);
```

There we go.

Analogously, if one `use`s a module, the module's parameters get tacked on to it's contents the same way.

Trait-less `impl` blocks simply become module with the type as a single parameter.

What to do about UFCS? I suppose the best thing given the above is to just go all the way and say
`f(a,...)  == a.f(...)` for all functions.

# Drawbacks

 - It is huge change to make at this stage of Rust's development.

 - Traits will now look odder to people accustomed to OOP --- in any of its definitions.

 - The restrictions on "Trait objects" appear more arbitrary.

# Alternatives

 - Just have scoped instances, named instances in a global namespace, and modules as is.

   I chose this route because that would leave an awkward feature overlap. But these alternatives
   affect the language much less dramatically.

 - Do nothing, as always.

# Unresolved questions

If I am correct, ML Modules I think are structural, whereas traits / type classes are nominal. I
went with nominal, but just to make this proposal a tad less catastrophic.

Existential types. We can generalize trait objects into first-class modules and module-dependent
tuples, but I am not sure how that interacts with Rust's commitment to monomorphization (I fear the
worst). Therefore I propose punting and keeping today's restriction with the new system:

 - Module must have 1 type parameter
 - Types "upcasted" must not have module parameter.
 - Method return types must not contain parameters in any way.
