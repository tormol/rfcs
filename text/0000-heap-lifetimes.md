- Feature Name: Heap lifetimes
- Start Date: 2016-06-30
- RFC PR: None
- Rust Issue: None

# Summary
[summary]: #summary

Heap lifetimes have the name of another parameter, struct member or tuple index, and are not declared.  
They look like `(Box<str>, Option<&'0 str>)`,
`fn(all: Box<[u8]>, slices: Hashmap<&'all[u8], &'all[u8]>)` or
```
struct Foo<T> {
    grid: Box<[T]>,
    paths: Vec<(&'grid T, &'grid T)>,
}
```

# Motivation
[motivation]: #motivation

Stack-based lifetimes are sufficient in most cases because they can track the owner of the heap memory.
But they prevent a struct from having references to the memory of a `Box` or `Rc` it owns.  
While pointers into non-heap siblings break if self is moved, pointers to heap memory owned by a sibling stay the same.

In most cases one can store indexes and create the slice in a getter. In other words, multiple times instead of once.  
Another workaround is to transmute the references to `'static`, and hide that detail behind a getter:
```
struct ConfigFile {
    all: Box<str>,
    map: HashMap<&'static str, &'static str>,
}
impl Configfile {
    pub fn get_map<'a>(&'a self) -> &'a HashMap<&'a str, &'a str> {
        &self.map
    }
}
```

This would also make it possible to [hide internal relationships](https://gist.github.com/tomaka/da8c374ce407e27d5dac) by boxing:
```
struct MyAssets<'a> {
    device: Box<Device>,
    programs: Vec<Program<'device>>,
    buffers: Vec<Buffer<'device>>,
    textures: Vec<Texture<'device>>,
    ...
}
```

# Detailed design
[design]: #detailed-design

They cannot be nested (`&'foo.bar`) but they can be upcast to an outer type, so `a_box.deref()` gets the lifetime `'a_box`.  
They can also be upcast to a stack-based lifetime.

This re-introduces &'self in a limited scope: It can only be used in return types;
This RFC doesn't try to implement immovable types.

Heap lifetimes can be used as generic lifetimes;
A function that borrows a parameter doesn't care about the owner, only the caller knows.

## Construction
Raw pointers get the unsafe methods `as_heap_ref(&self) -> &'self T`
and `as_heap_mut(&mut self) -> &'self T`.

They can also be created by transmuting. This allows one to name the owner,
which might be useful: `unsafe{ mem::transmute::<_,&'ptr>(ptr.offset(n)) }`.
Here the lifetime of the offset pointer is much shorter than `ptr`'s.

## When does the lifetime end?
It ends when the owner goes out of scope, is mutably borrowed, or the borrower 'loses control' of the owner.

Control is lost if the owner is consumed by another function without the borrower being moved along simultaneously.

`&mut` ends the lifetime because it allows calling `mem::replace()`, which might drop and free the borrowed memory. There's also the posibility of aliasing.

# Drawbacks
[drawbacks]: #drawbacks

Complicates the lifetime system for something that would rarely be used.
([brson says they have never needed it in servo]().)

# Alternatives
[alternatives]: #alternatives

* A `&heap` borrow type:  
  While explicit, that is also more complex, and I could'nt work out the details.

* [back-references](https://www.reddit.com/r/rust/comments/4n5ab2/a_take_on_selfreferencing_structs/):

* Are there others?

* Doing nothing:  
  There are several ways to work around this limitation,
  and there are several crates to makes it easier:
  [owning_ref](http://kimundi.github.io/owning-ref-rs/owning_ref/index.html),
  [refstruct](https://github.com/diwic/refstruct-rs/blob/master/README.md) and
  [self-ref](http://mixthos.github.io/self-ref/self_ref/index.html).

# Unresolved questions
[unresolved]: #unresolved-questions

## `Drop`-implementing borrowers

With stack-based lifetimes, references must last longer than self,
but the order the members of a struct is dropped in is currently unspecified.  

It probably has to become slightly moree specified, but I don't know where.

## Should mutable collections like `Vec` and `HashMap` returning heap lifetimes?

Since `&mut` is banned, they should just work, but it might be possible to relax the rules a little. Barring aliasing, functions that return references with heap lifetime is probably OK.

For vectors this isn't important since a `Box<[T]>` expresses the intent better,
but other data structures might not have a constant-size equivalent.
