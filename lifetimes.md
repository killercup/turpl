% Ownership and Lifetimes

Ownership is the breakout feature of Rust. It allows Rust to be completely
memory-safe and efficient, while avoiding garbage collection. Before getting
into the ownership system in detail, we will consider a simple but *fundamental*
language-design problem.



# The Tagged Union Problem

The core of the lifetime and mutability system derives from a simple problem:
internal pointers to tagged unions. For instance, consider the following code:

```rust
enum Foo {
    A(u32),
    B(f64),
}

let mut x = B(2.0);
if let B(ref mut y) = x {
    *x = A(7);
    // OH NO! a u32 has been interpretted as an f64! Type-safety hole!
    // (this does not actually compile)
    println!("{}", y);

}
```

The problem here is an intersection of 3 choices:

* data in a tagged union is inline with the tag
* tagged unions are mutable
* being able to take a pointer into a tagged union

Remove *any* of these 3 and the problem goes away. Traditionally, functional
languages have avoided this problem by removing the mutable
option. This means that they can in principle keep their data inline (ghc has
a pragma for this). A garbage collected imperative language like Java could alternatively
solve this problem by just keeping all variants elsewhere, so that changing the
variant of a tagged union just overwrites a pointer, and anyone with an outstanding
pointer to the inner data is unaffected thanks to The Magic Of Garbage Collection.

Rust, by contrast, takes a subtler approach. Rust allows mutation,
allows pointers to inner data, and its enums have their data allocated inline.
However it prevents anything from being mutated while there are outstanding
pointers to it! And this is all done at compile time.

Interestingly, Rust's `std::cell` module exposes two types that offer an alternative
approach to this problem:

* The `Cell` type allows mutation of aliased data, but
instead forbids internal pointers to that data. The only way to read or write
a Cell is to copy the bits in or out.

* The `RefCell` type allows mutation of aliased data *and* internal pointers, but
manages this through *runtime* checks. It is effectively a thread-unsafe
read-write lock.

For more details see Dan Grossman's *Existential Types for Imperative Languages*:

* [paper][grossman-paper] (Advanced)
* [slides][grossman-slides] (Simple)

[grossman-paper]: http://homes.cs.washington.edu/~djg/papers/exists_imp.pdf
[grossman-slides]: https://homes.cs.washington.edu/~djg/slides/esop02_talk.pdf




# Lifetimes

Rust's static checks are managed by the *borrow checker* (borrowck), which tracks
mutability and outstanding loans. This analysis can in principle be done without
any help locally. However as soon as data starts crossing the function boundary,
we have some serious trouble. In principle, borrowck could be a massive
whole-program analysis engine to handle this problem, but this would be an
atrocious solution. It would be terribly slow, and errors would be horribly
non-local.

Instead, Rust tracks ownership through *lifetimes*. Every single reference and value
in Rust is tagged with a lifetime that indicates the scope it is valid for.
Rust has two kinds of reference:

* Shared reference: `&`
* Mutable reference: `&mut`

The main rules are as follows:

* A shared reference can be aliased
* A mutable reference cannot be aliased
* A reference cannot outlive its referrent (`&'a T -> T: 'a`)

However non-mutable variables have some special rules:

* You cannot mutate or mutably borrow a non-mut variable,

Only variables marked as mutable can be borrowed mutably, though this is little
more than a local lint against incorrect usage of a value.




# Weird Lifetimes

Given the following code:

```rust
struct Foo;

impl Foo {
    fn mutate_and_share(&mut self) -> &Self { &*self }
    fn share(&self) {}
}

fn main() {
    let mut foo = Foo;
    let loan = foo.mutate_and_share();
    foo.share();
}
```

One might expect it to compile. We call `mutate_and_share`, which mutably borrows
`foo` *temporarily*, but then returns *only* a shared reference. Therefore we
would expect `foo.share()` to succeed as `foo` shouldn't be mutably borrowed.

However when we try to compile it:

```text
<anon>:11:5: 11:8 error: cannot borrow `foo` as immutable because it is also borrowed as mutable
<anon>:11     foo.share();
              ^~~
<anon>:10:16: 10:19 note: previous borrow of `foo` occurs here; the mutable borrow prevents subsequent moves, borrows, or modification of `foo` until the borrow ends
<anon>:10     let loan = foo.mutate_and_share();
                         ^~~
<anon>:12:2: 12:2 note: previous borrow ends here
<anon>:8 fn main() {
<anon>:9     let mut foo = Foo;
<anon>:10     let loan = foo.mutate_and_share();
<anon>:11     foo.share();
<anon>:12 }
          ^
```

What happened? Well, the lifetime of `loan` is derived from a *mutable* borrow.
This makes the type system believe that `foo` is mutably borrowed as long as
`loan` exists, even though it's a shared reference. To my knowledge, this is not
a bug.




# Lifetime Elision

In order to make common patterns more ergonomic, Rust allows lifetimes to be
*elided* in function, impl, and type signatures.

A *lifetime position* is anywhere you can write a lifetime in a type:

```rust
&'a T
&'a mut T
T<'a>
```

Lifetime positions can appear as either "input" or "output":

* For `fn` definitions, input refers to the types of the formal arguments
  in the `fn` definition, while output refers to
  result types. So `fn foo(s: &str) -> (&str, &str)` has elided one lifetime in
  input position and two lifetimes in output position.
  Note that the input positions of a `fn` method definition do not
  include the lifetimes that occur in the method's `impl` header
  (nor lifetimes that occur in the trait header, for a default method).

* In the future, it should be possible to elide `impl` headers in the same manner.

Elision rules are as follows:

* Each elided lifetime in input position becomes a distinct lifetime
  parameter.

* If there is exactly one input lifetime position (elided or not), that lifetime
  is assigned to *all* elided output lifetimes.

* If there are multiple input lifetime positions, but one of them is `&self` or
  `&mut self`, the lifetime of `self` is assigned to *all* elided output lifetimes.

* Otherwise, it is an error to elide an output lifetime.

Examples:

```rust
fn print(s: &str);                                      // elided
fn print<'a>(s: &'a str);                               // expanded

fn debug(lvl: uint, s: &str);                           // elided
fn debug<'a>(lvl: uint, s: &'a str);                    // expanded

fn substr(s: &str, until: uint) -> &str;                // elided
fn substr<'a>(s: &'a str, until: uint) -> &'a str;      // expanded

fn get_str() -> &str;                                   // ILLEGAL

fn frob(s: &str, t: &str) -> &str;                      // ILLEGAL

fn get_mut(&mut self) -> &mut T;                        // elided
fn get_mut<'a>(&'a mut self) -> &'a mut T;              // expanded

fn args<T:ToCStr>(&mut self, args: &[T]) -> &mut Command                  // elided
fn args<'a, 'b, T:ToCStr>(&'a mut self, args: &'b [T]) -> &'a mut Command // expanded

fn new(buf: &mut [u8]) -> BufWriter;                    // elided
fn new<'a>(buf: &'a mut [u8]) -> BufWriter<'a>          // expanded

```



# Unbounded Lifetimes

Unsafe code can often end up producing references or lifetimes out of thin air.
Such lifetimes come into the world as *unbounded*. The most common source of this
is derefencing a raw pointer, which produces a reference with an unbounded lifetime.
Such a lifetime becomes as big as context demands. This is in fact more powerful
than simply becoming `'static`, because for instance `&'static &'a T`
will fail to typecheck, but the unbound lifetime will perfectly mold into
`&'a &'a T` as needed. However for most intents and purposes, such an unbounded
lifetime can be regarded as `'static`.

Almost no reference is `'static`, so this is probably wrong. `transmute` and
`transmute_copy` are the two other primary offenders. One should endeavour to
bound an unbounded lifetime as quick as possible, especially across function
boundaries.

Given a function, any output lifetimes that don't derive from inputs are
unbounded. For instance:

```
fn get_str<'a>() -> &'a str;
```

will produce an `&str` with an unbounded lifetime. The easiest way to avoid
unbounded lifetimes is to use lifetime elision at the function boundary.
If an output lifetime is elided, then it *must* be bounded by an input lifetime.
Of course, it might be bounded by the *wrong* lifetime, but this will usually
just cause a compiler error, rather than allow memory safety to be trivially
violated.

Within a function, bounding lifetimes is more error-prone. The safest and easiest
way to bound a lifetime is to return it from a function with a bound lifetime.
However if this is unacceptable, the reference can be placed in a location with
a specific lifetime. Unfortunately it's impossible to name all lifetimes involved
in a function. To get around this, you can in principle use `copy_lifetime`, though
these are unstable due to their awkward nature and questionable utility.





# Higher-Rank Lifetimes

Generics in Rust generally allow types to be instantiated with arbitrary
associated lifetimes, but this fixes the lifetimes they work with once
instantiated. For almost all types, this is exactly the desired behaviour.
For example slice::Iter can work with arbitrary lifetimes, determined by the
slice that instantiates it. However *once* Iter is instantiated the lifetimes
it works with cannot be changed. It returns references that live for some
particular `'a`.

However some types are more flexible than this. In particular, a single
instantiation of a function can process arbitrary lifetimes:

```rust
fn identity(input: &u8) -> &u8 { input }
```

What is *the* lifetime that identity works with? There is none. If you think
this is "cheating" because functions are statically instantiated, then you need
only consider the equivalent closure:

```rust
let identity = |input: &u8| input;
```

These functions are *higher ranked* over the lifetimes they work with. This means
that they're generic over what they handle *after instantiation*. For most things
this would pose a massive problem, but because lifetimes don't *exist* at runtime,
this is really just a compile-time mechanism. The Fn traits contain sugar that
allows higher-rank lifetimes to simply be expressed by simply omitting lifetimes:


```rust
fn main() {
    foo(|input| input);
}

fn foo<F>(f: F)
    // F is higher-ranked over the lifetime these references have
    where F: Fn(&u8) -> &u8
{
    f(&0);
    f(&1);
}
```

The desugaring of this is actually unstable:

```
#![feature(unboxed_closures)]

fn main() {
    foo(|input| input);
}

fn foo<F>(f: F)
    where F: for<'a> Fn<(&'a u8,), Output=&'a u8>
{
    f(&0);
    f(&1);
}
```

`for<'a>` is how we declare a higher-ranked lifetime. Unfortunately higher-ranked
lifetimes are still fairly new, and are missing a few features to make them
maximally useful outside of the Fn traits.




# Subtyping and Variance

Although Rust doesn't have any notion of inheritance, it *does* include subtyping.
In Rust, subtyping derives entirely from *lifetimes*. Since lifetimes are derived
from scopes, we can partially order them based on an *outlives* relationship. We
can even express this as a generic bound: `T: 'a` specifies that `T` *outlives* `'a`.

We can then define subtyping on lifetimes in terms of lifetimes: `'a : 'b` implies
`'a <: b` -- if `'a` outlives `'b`, then `'a` is a subtype of `'b`. This is a very
large source of confusion, because a bigger scope is a *sub type* of a smaller scope.
This does in fact make sense. The intuitive reason for this is that if you expect an
`&'a u8`, then it's totally fine for me to hand you an `&'static u8`, in the same way
that if you expect an Animal in Java, it's totally fine for me to hand you a Cat.

(Note, the subtyping relationship and typed-ness of lifetimes is a fairly arbitrary
construct that some disagree with. I just find that it simplifies this analysis.)

Variance is where things get really harsh.

Variance is a property that *type constructors* have. A type constructor in Rust
is a generic type with unbound arguments. For instance `Vec` is a type constructor
that takes a `T` and returns a `Vec<T>`. `&` and `&mut` are type constructors that
take a lifetime and a type.

A type constructor's *variance* is how the subtypes of its inputs affects the
subtypes of its outputs. There are three kinds of variance:

* F is *covariant* if `T <: U` implies `F<T> <: F<U>`
* F is *contravariant* if `T <: U` implies `F<U> <: F<T>`
* F is *invariant* otherwise (no subtyping relation can be derived)

Some important variances:

* `&` is covariant (as is *const by metaphor)
* `&mut` is invariant (as is *mut by metaphor)
* `Fn(T)` is contravariant with respect to `T`
* `Box`, `Vec`, and all other collections are covariant
* `UnsafeCell`, `Cell`, `RefCell`, `Mutex` and all "interior mutability"
  types are invariant

To understand why these variances are correct and desirable, we will consider several
examples. We have already covered why `&` should be covariant.

To see why `&mut` should be invariant, consider the following code:

```rust
fn main() {
    let mut forever_str: &'static str = "hello";
    {
        let string = String::from("world");
        overwrite(&mut forever_str, &mut &*string);
    }
    println!("{}", forever_str);
}

fn overwrite<T: Copy>(input: &mut T, new: &mut T) {
    *input = *new;
}
```

The signature of `overwrite` is clearly valid: it takes mutable references to two values
of the same type, and replaces one with the other. We have seen already that `&` is
covariant, and `'static` is a subtype of *any* `'a`, so `&'static str` is a
subtype of `&'a str`. Therefore, if `&mut` was
*also* covariant, then the lifetime of the `&'static str` would successfully be
"shrunk" down to the shorter lifetime of the string, and `replace` would be
called successfully. The string would subsequently be dropped, and `forever_str`
would point to freed memory when we print it!

Therefore `&mut` should be invariant. This is the general theme of covariance vs
invariance: if covariance would allow you to *store* a short-lived value in a
longer-lived slot, then you must be invariant.

`Box` and `Vec` are interesting cases because they're covariant, but you can
definitely store values in them! This is fine because *you can only store values
in them through a mutable reference*! The mutable reference makes the whole type
invariant, and therefore prevents you from getting in trouble.

Being covariant allows them to be covariant when shared immutably (so you can pass
a `&Box<&'static str>` where a `&Box<&'a str>` is expected). It also allows you to
forever weaken the type by moving it into a weaker slot. That is, you can do:

```rust
fn get_box<'a>(&'a u8) -> Box<&'a str> {
    Box::new("hello")
}
```

which is fine because unlike the mutable borrow case, there's no one else who
"remembers" the old lifetime in the box.

The variance of the cell types similarly follows. `&` is like an `&mut` for a
cell, because you can still store values in them through an `&`. Therefore cells
must be invariant to avoid lifetime smuggling.

`Fn` is the most confusing case, largely because contravariance is easily the
most confusing kind of variance, and basically never comes up. To understand it,
consider a function `len` that takes a function `F`.

```rust
fn len<F>(func: F) -> usize
    where F: Fn(&'static str) -> usize
{
    func("hello")
}
```

We require that F is a Fn that can take an `&'static str` and returns a usize. Now
say we have a function that can take an `&'a str` (for *some* `'a`). Such a function actually
accepts *more* inputs, since `&'static str` is a subtype of `&'a str`. Therefore
`len` should happily accept such a function!

So a `Fn(&'a str)` is a subtype of a `Fn(&'static str)` because
`&'static str` is a subtype of `&'a str`. Exactly contravariance.

The variance of `*const` and `*mut` is basically arbitrary as they're not at all
type or memory safe, so their variance is determined in analogy to & and &mut
respectively.

This is all well and good for the types the standard library provides, but
how is variance determined for type that *you* define? A struct informally
speaking inherits the variance of its fields. If a struct `Foo`
has a generic argument `A` that is used in a field `a`, then Foo's variance
over `A` is exactly `a`'s variance. However this is complicated if `A` is used
in multiple fields.

* If all uses of A are covariant, then Foo is covariant over A
* If all uses of A are contravariant, then Foo is contravariant over A
* Otherwise, Foo is invariant over A

```rust
struct Foo<'a, 'b, A, B, C, D, E, F, G, H> {
    a: &'a A,     // covariant over 'a and A
    b: &'b mut B, // invariant over 'b and B
    c: *const C,  // covariant over C
    d: *mut D,    // invariant over D
    e: Vec<E>,    // covariant over E
    f: Cell<F>,   // invariant over F
    g: G          // covariant over G
    h1: H         // would also be covariant over H except...
    h2: Cell<H>   // invariant over H, because invariance wins
}
```



# PhantomData

When working with unsafe code, we can often end up in a situation where
types or lifetimes are logically associated with a struct, but not actually
part of a field. This most commonly occurs with lifetimes. For instance, the `Iter`
for `&'a [T]` is (approximately) defined as follows:

```
pub struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T,
}
```

However because `'a` is unused within the struct's body, it's *unbound*.
Because of the troubles this has historically caused, unbound lifetimes and
types are *illegal* in struct definitions. Therefore we must somehow refer
to these types in the body. Correctly doing this is necessary to have
correct variance and drop checking.

We do this using *PhantomData*, which is a special marker type. PhantomData
consumes no space, but simulates a field of the given type for the purpose of
variance. This was deemed to be less error-prone than explicitly telling the
type-system the kind of variance that you want.

Iter logically contains `&'a T`, so this is exactly what we tell
the PhantomData to simulate:

```
pub struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T,
    _marker: marker::PhantomData<&'a T>,
}
```




# Dropck

When a type is going out of scope, Rust will try to Drop it. Drop executes
arbitrary code, and in fact allows us to "smuggle" arbitrary code execution
into many places. As such additional soundness checks (dropck) are necessary to
ensure that a type T can be safely instantiated and dropped. It turns out that we
*really* don't need to care about dropck in practice, as it often "just works".

However the one exception is with PhantomData. Given a struct like Vec:

```
struct Vec<T> {
    data: *const T, // *const for covariance!
    len: usize,
    cap: usize,
}
```

dropck will generously determine that Vec<T> does not contain any values of
type T. This will unfortunately allow people to construct unsound Drop
implementations that access data that has already been dropped. In order to
tell dropck that we *do* own values of type T and may call destructors of that
type, we must add extra PhantomData:

```
struct Vec<T> {
    data: *const T, // *const for covariance!
    len: usize,
    cap: usize,
    _marker: marker::PhantomData<T>,
}
```

Raw pointers that own an allocation is such a pervasive pattern that the
standard library made a utility for itself called `Unique<T>` which:

* wraps a `*const T`,
* includes a PhantomData<T>,
* auto-derives Send/Sync as if T was contained
* marks the pointer as NonZero for the null-pointer optimization




# Splitting Lifetimes

The mutual exclusion property of mutable references can be very limiting when
working with a composite structure. Borrowck understands some basic stuff, but
will fall over pretty easily. Borrowck understands structs sufficiently to
understand that it's possible to borrow disjoint fields of a struct simultaneously.
So this works today:

```rust
struct Foo {
    a: i32,
    b: i32,
    c: i32,
}

let mut x = Foo {a: 0, b: 0, c: 0};
let a = &mut x.a;
let b = &mut x.b;
let c = &x.c;
*b += 1;
let c2 = &x.c;
*a += 10;
println!("{} {} {} {}", a, b, c, c2);
```

However borrowck doesn't understand arrays or slices in any way, so this doesn't
work:

```rust
let x = [1, 2, 3];
let a = &mut x[0];
let b = &mut x[1];
println!("{} {}", a, b);
```

```text
<anon>:3:18: 3:22 error: cannot borrow immutable indexed content `x[..]` as mutable
<anon>:3     let a = &mut x[0];
                          ^~~~
<anon>:4:18: 4:22 error: cannot borrow immutable indexed content `x[..]` as mutable
<anon>:4     let b = &mut x[1];
                          ^~~~
error: aborting due to 2 previous errors
```

While it was plausible that borrowck could understand this simple case, it's
pretty clearly hopeless for borrowck to understand disjointness in general
container types like a tree, especially if distinct keys actually *do* map
to the same value.

In order to "teach" borrowck that what we're doing is ok, we need to drop down
to unsafe code. For instance, mutable slices expose a `split_at_mut` function that
consumes the slice and returns *two* mutable slices. One for everything to the
left of the index, and one for everything to the right. Intuitively we know this
is safe because the slices don't alias. However the implementation requires some
unsafety:

```rust
fn split_at_mut(&mut self, mid: usize) -> (&mut [T], &mut [T]) {
    unsafe {
        let self2: &mut [T] = mem::transmute_copy(&self);

        (ops::IndexMut::index_mut(self, ops::RangeTo { end: mid } ),
         ops::IndexMut::index_mut(self2, ops::RangeFrom { start: mid } ))
    }
}
```

This is pretty plainly dangerous. We use transmute to duplicate the slice with an
*unbounded* lifetime, so that it can be treated as disjoint from the other until
we unify them when we return.

However more subtle is how iterators that yield mutable references work.
The iterator trait is defined as follows:

```rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

Given this definition, Self::Item has *no* connection to `self`. This means
that we can call `next` several times in a row, and hold onto all the results
*concurrently*. This is perfectly fine for by-value iterators, which have exactly
these semantics. It's also actually fine for shared references, as they admit
arbitrarily many references to the same thing (although the
iterator needs to be a separate object from the thing being shared). But mutable
references make this a mess. At first glance, they might seem completely
incompatible with this API, as it would produce multiple mutable references to
the same object!

However it actually *does* work, exactly because iterators are one-shot objects.
Everything an IterMut yields will be yielded *at most* once, so we don't *actually*
ever yield multiple mutable references to the same piece of data.

In general all mutable iterators require *some* unsafe code *somewhere*, though.
Whether it's raw pointers, or safely composing on top of *another* IterMut.

For instance, VecDeque's IterMut:

```rust
pub struct IterMut<'a, T:'a> {
    // The whole backing array. Some of these indices are initialized!
    ring: &'a mut [T],
    tail: usize,
    head: usize,
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<&'a mut T> {
        if self.tail == self.head {
            return None;
        }
        let tail = self.tail;
        self.tail = wrap_index(self.tail.wrapping_add(1), self.ring.len());

        unsafe {
            // might as well do unchecked indexing since wrap_index has us
            // in-bounds, and many of the "middle" indices are uninitialized
            // anyway.
            let elem = self.ring.get_unchecked_mut(tail);

            // round-trip through a raw pointer to unbound the lifetime from
            // ourselves
            Some(&mut *(elem as *mut _))
        }
    }
}
```

A very subtle but interesting detail in this design is that it *relies on
privacy to be sound*. Borrowck works on some very simple rules. One of those rules
is that if we have a live &mut Foo and Foo contains an &mut Bar, then that &mut
Bar is *also* live. Since IterMut is always live when `next` can be called, if
`ring` were public then we could mutate `ring` while outstanding mutable borrows
to it exist!


