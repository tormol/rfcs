- Feature Name: heap_lifetimes
- Start Date: 2017-0_-__
- RFC PR: None
- Rust Issue: None

# Summary
[summary]: #summary

Heap lifetimes track the owner value instead of a memory address, and can therefore survie moves.  
They are not declared, but have the name of another parameter, struct member or tuple index.  
Examples: `(Box<str>, Option<&'0 str>)`,
`fn(all: Vec<[u8]>, slices: Hashmap<&'all[u8], &'all[u8]>)`,
```
struct Foo<T> {
    paths: Vec<(&'grid T, &'grid T)>,
    grid: Box<[T]>,
}
```


# Motivation
[motivation]: #motivation

Currently a struct field cannot borrow heap memory owned by another field.
While allowing references to arbitrary fields would require immovable types,
references to heap memory stay valid when the owner is moved.

This can be worked around by using raw pointers, but that's `unsafe`.

Another workaround if the owner is a vector or boxed slice is to store indexes and create the slice in a getter. However that doesn't work if you want to store the borrower inside a nested pre-existing struct.

This would also make it possible to [hide internal relationships](https://gist.github.com/tomaka/da8c374ce407e27d5dac) by boxing:
```
struct MyAssets<'a> {
    programs: Vec<Program<'device>>,
    buffers: Vec<Buffer<'device>>,
    textures: Vec<Texture<'device>>,
    ...
    device: Box<Device>,
}
```


# Detailed design
[design]: #detailed-design

Heap lifetimes are implicitly upcast to an outer type when required, so the field it was created from doesn't need to be exposed:

```
struct Foo(Vec<u8>)
impl Foo {
    pub fn get(&self) -> &'self [u8] {
        &self.0[..]
    }
}
```

If the borrower implement `Drop` it must come before the owner in struct and tuple definitions in case `drop()` dereferences it.
(Assuming [Stabilize drop order](https://github.com/rust-lang/rfcs/pull/1857) is merged as is.)

When the owner is moved within a function (not neccesarily between local variables), the name of the lifetime changes, but the borrower doesn't need to be moved.

Supporting inter-referencing arrays and slices will require relative names, and is not a part of this RFC.

Heap lifetimes don't actually need to point to the heap; it just has to be somewhere that is outside the owner and stops being valid when the owner is dropped.  
For example I don't think this will interfere with custom allocators:  
It is ok for a `Vec` with a stack-based allocator to give out heap lifetimes because the memory doesn't lie within the `Vec`. And the `Vec`s allocator parameter would have a stack lifetime that it cannot outlive.

## Interaction with stack-based lifetimes

Heap lifetimes can be passed to functions and types expecting stack lifetimes as long as the owner stays outside. They then act like normal stack lifetimes.  
In a way they're casted to stack lifetimes but they're also cast back when the function returns or struct member is accessed.

Function parameters and struct members can be declared to accept both stack lifetimes and heap lifetimes with local owner. Their type look like `&'a + 'sibling T`. Then users must follow the rules of both.

## How to get one

To get refereences with heap lifetimes from existing types two traits are introduced:

```
trait DerefHeap : Deref {
	fn deref_heap(&self) -> &'self <Self as Deref>::Target;
}

trait DerefMutHeap : DerefMut + DerefHeap {
	fn deref_mut_heap(&mut self) -> &'self mut <Self as Deref>::Target;
}
```

(`'self` doesn't have any special meaning; it's just happens to be the name of a parameter.)

Alternatively [trait impls could be allowwed to extend returned lifetimes](FIXME).

To create them references with heap lifetimes, both kinds of raw pointers get the methods `as_heap_ref<'owner>(self) -> &'owner T` and `as_mut_heap_ref<'owner>(self) -> &'owner mut T`.  
Heap-allocating types creates them by calling one of those, or transmuting.


## When does the lifetime end?
Heap lifetimes must still follow mutability xor aliasing, This means one cannot eg mutably slice a `Vec` multiple times, instead use slice.split().

When mutably borrowed the owner can only be moved. if the borrow is shared, the owner can also be non-mutably borrowed.
There are also restrictions on where the the owner can be moved: it cannot be returned or passed to a function without the borrower being moved along simultaneously.

If the type of borrow (`&mut` or `&`) is not visible in the type, it is assumed to be mutable. This means that in the `MyAssets` example, `device` is untouchable as long as the other fields exists.
(Mutability xor aliasing is ensured by the borrow checking of the function that created the struct.)


# How We Teach This
[how-we-teach-this]: #how-we-teach-this

"Whereas normal stack lifetimes are bound to a variable in a scope block, heap lifetimes follow a value."

TODO expand: While this is an advanced feature, `'self` will show up in return types in documentation.


# Drawbacks
[drawbacks]: #drawbacks

* If extending return lifetimes in impls is added after this RFC is stabilized, the `Deref*Heap` traits become redundant but must be kept for backwards-compatibility. I think the workarund is ok while the feature is unstable, but should be revisited before stabilisation.

* The sometimes required ordering of fields is rarely the most logical.
  For starters I had to change the `MyAssets` example to fit with the recently defined drop order.
  It might also prevent some layout optimizations, but padding shouldn't be an issue since the borrower is at least pointer-aligned.
  I doubt this is what will prevent somebody from deriving `PartialOrd`/`Ord`, because you probably don't want to compare both borrower and owner, if implementing it at all.

* The restrictions on usage of borrowed members and parameters is not shown in their type.

* Might conflict with desings for immovable types.

* More boiler-plate for crates that want to reach std-level quality.

* Complicates the lifetime system.


# Alternatives
[alternatives]: #alternatives

* Doing nothing:  
  It is possible to work around this limitation,
  and there are several crates to makes it easier:
  [owning_ref](http://kimundi.github.io/owning-ref-rs/owning_ref/index.html),
  [refstruct](https://github.com/diwic/refstruct-rs/blob/master/README.md) and
  [self-ref](http://mixthos.github.io/self-ref/self_ref/index.html).

* [back-references](https://www.reddit.com/r/rust/comments/4n5ab2/a_take_on_selfreferencing_structs/):

* --The `DerefPure` part of [`&move`, `DerefMove` and `DerefPure`](https://github.com/rust-lang/rfcs/pull/1646)--

* A `&heap` borrow type:  
  I couldn't work out the details,
  but it's more explicit and might be easier to understand and teach.


# Unresolved questions
[unresolved]: #unresolved-questions
