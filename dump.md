Goals:
* deriving trace hooks
* lifetimes for safe rooted references
* lint for unsafe type uses

---

A web browser's purpose in life
is to mediate interaction between a user and a document.
These days,
a "document" can be a full-fledged interactive application.
Users expect a browser to be fast and responsive,
so the core layout and rendering algorithms
are typically implemented in low-level native code.
At the same time,
JavaScript code in the document
can perform complex modifications
through the [Document Object Model](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model).
This means the browser's representation of a document in memory
is a cross-language data structure,
bridging the gap between
low-level native code
and the high-level, garbage-collected world of JavaScript.

This is one of many opportunities we have
in the [Servo project](https://github.com/servo/servo/)
to advance the state of the art.
We have a new approach for DOM memory management,
and we're using some exciting features of
the [Rust](http://www.rust-lang.org/) language and compiler
to support it.
We can facilitate the interaction between
Servo's Rust code,
the [SpiderMonkey](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/SpiderMonkey) garbage collector (written in C++),
and the JavaScript code from the document itself,
in a way that's fast, secure, and produces maintainable code.


# Memory management for the DOM

It's essential that we never destroy a DOM object
while it's still accessible from either JavaScript or native code -
such [use-after-free bugs](http://cwe.mitre.org/data/definitions/416.html) often result in exploitable security holes.
To solve this problem, most existing browsers use
[reference counting](http://en.wikipedia.org/wiki/Reference_counting)
to track the pointers between underlying low-level DOM objects.
When JavaScript retrieves a DOM object
(through [`getElementById`](https://developer.mozilla.org/en-US/docs/Web/API/document.getElementById) for example),
the browser builds a "reflector" object in the JavaScript VM
that holds a reference to the underlying low-level object.
If the JavaScript garbage collector determines that
a reflector is no longer accessible,
it destroys the reflector
and decrements the reference count
on the underlying object.

This solves the use-after-free issue.
But to keep users happy,
we also need to keep the browser's memory footprint small.
This means destroying objects as soon as possible
when they are no longer needed.
Unfortunately, the cross-language "reflector" scheme
introduces a major complication.

Consider a C++ `Element` object which holds a reference-counted pointer to an `Event`:

```cpp
struct Element {
    RefPtr<Event> mEvent;
}
```

Now suppose we add an event handler to the element from JavaScript:

```js
elem.addEventListener('load', function (event) {
    event.originalTarget = elem;
});
```

When the event fires,
the handler adds a property on the `Event`
which points back to the `Element`.

We now have a cross-language reference cycle,
with an `Element` pointing to an `Event` within C++,
and an `Event` reflector pointing to the `Element` reflector in JavaScript.
The C++ refcounting will never destroy a cycle,
and the JavaScript garbage collector can't trace through
the C++ pointers,
so these objects will never be freed.

[ FIXME: draw a graph? ]

Existing browsers resolve this problem in several ways:
some do nothing, and leak memory;
some try to manually break possible cycles,
by nulling out `mEvent` for example;
some implement a cycle collection algorithm
on top of reference counting.

None of these solutions are particularly satisfying,
so we're trying something new in Servo by choosing not to use
reference counting on DOM objects at all.
Instead, we give the JavaScript garbage collector full responsibility
for managing those native-code DOM objects.
Before we manipulate a DOM object from native code,
we must first [root](http://en.wikipedia.org/wiki/Tracing_garbage_collection#Reachability_of_an_object) it in the garbage collector
to ensure it won't be destroyed inadvertently.

This is where we take advantage of some cool Rust features.
We use compiler-derived traits
to implement garbage collector hooks
without error-prone boilerplate.
We use a custom static analysis pass
to check that DOM objects are rooted when they need to be.
Finally, we use lifetime checking to ensure at compile time
that pointers to the interior of a DOM object
cannot outlive the rooting.

---

# Compiler-derived traits

We have to tell the JavaScript garbage collector
about references between our DOM objects
so it can determine reachability.
Gecko's cycle collector faces a similar problem,
and contains a lot of hand-written annotations such as

```cpp
NS_IMPL_CYCLE_COLLECTION(nsFrameLoader, mDocShell, mMessageManager, mChildMessageManager)
```

This macro declares the members of a C++ class
that should be added to a graph of potential cycles.
Forgetting an entry can result in a memory leak.
In Servo the consequences would be even worse:
if the garbage collector can't see all references,
it might free a node that is still in use.
It's essential for both security and programmer convenience
that we get rid of this manual effort.

Rust has a notion of [traits](http://doc.rust-lang.org/tutorial.html#traits),
which are similar to [type classes](http://learnyouahaskell.com/types-and-typeclasses) in Haskell
or interfaces in many OO languages.
A simple example is the [`Clone` trait](http://doc.rust-lang.org/std/clone/trait.Clone.html):

```rust
pub trait Clone {
    fn clone(&self) -> Self;
}
```

Any type implementing the `Clone` trait
will provide a method named `clone`
that takes a value of the type
(by reference, hence `&self`)
and returns a new value of the same type (`Self`).
Clearly, the `Clone` trait describes objects that can be copied.

Another example is the [`Encodable` trait](http://doc.rust-lang.org/serialize/trait.Encodable.html) for serialization:

```rust
pub trait Encodable<Enc: Encoder<Err>, Err> {
    fn encode(&self, encoder: &mut Enc) -> Result<(), Err>;
}
```

This definition is more complicated than `Clone`
because the trait has type parameters `Enc` and `Err`,
and `Enc` itself is required to be a type implementing
another trait `Encoder`.

Any data type that can be serialized will provide an `encode` method,
which visits the data type's fields by 
calling [`Encoder`](http://doc.rust-lang.org/serialize/trait.Encoder.html),
methods such as `emit_u32`, `emit_tuple`, etc.
This leaves the details of a particular serialization format
(such as JSON)
up to the `Encoder`.

The `Encodable` trait is special,
because the compiler can implement it for us!
All we need to do is provide a [`deriving`](http://doc.rust-lang.org/tutorial.html#deriving-implementations-for-traits) attribute
on the definition of a type:

```rust
#[deriving(Encodable)]
pub struct Element {
    pub node: Node,
    pub local_name: Atom,
    pub namespace: Namespace,
    ...
}
```

The Rust compiler will automatically generate an `encode` method
that calls the respective `encode` methods on `node`, `local_name`, etc.
An `Element` contains a `Node` (directly, by value)
so the generated code will recursively visit the fields of `node`,
and so on.
This nicely replaces
the manual listing of data fields
that we had in C++.
The compiler will yell at us
if we add a field to `Element` that doesn't implement `Encodable`,
providing compile-time assurance
that we're tracing all the fields
of our objects.

The pointers between DOM objects are represented by a type `JS<T>`:

```rust
#[deriving(Encodable)]
pub struct Node {
    pub type_id: NodeTypeId,
    pub parent_node: Option<JS<Node>>,
    pub first_child: Option<JS<Node>>,
    ...
```

Each node [optionally](http://doc.rust-lang.org/std/option/type.Option.html) points to
a parent node, a first child, and so forth.
The [`Encodable` implementation for `JS<T>`](https://github.com/servo/servo/blob/1c0e51015fc1a5ba0e189f114e35019af27d68ca/src/components/script/dom/bindings/trace.rs#L51-L56)
is *not* automatically derived;
this is where we actually tell
the JavaScript garbage collector
about these pointers.


# Lifetimes

By yielding all control over deallocation to SpiderMonkey's garbage collector, we now need to solve the safety problem that other browsers use reference counting to avoid. We've developed a simple set of types that enforce sound rooting practices, such that it should be impossible to use pointers to GC-owned values in unsafe ways.

* `Root<T>` - a stack-allocated value that "roots" a GC-owned value for the duration of the root's lifetime (ie. prevents it being collected)
* `JSRef<T>` - a freely-cloneable smart pointer to a rooted, GC-owned value that can be dereferenced to interact with the wrapped value
* `JS<T>` - a non-stack-allocated pointer to a GC-owned value
* `Temporary<T>` - a stack-allocated, movable pointer to a GC-owned value that ensures its wrapped value is rooted for the duration of the wrapper's lifetime

These types allow us to enforce the following rules that make Servo's DOM implementation safe:

* All interactions with the underlying GC-owned value (both calling methods and accessing members) must occur through a root
  * Only `JSRef` implements any dereferencing behaviour. `Temporary` and `JS` values merely carry around a pointer and provide facilities to create stack-bounded roots.
  * All methods take `JSRef` arguments, and all methods for DOM type Foo are implemented on `JSRef<Foo>` types, rather than `Foo` itself. This ensures that argument and self pointers are always safe to use.
* Any reference to a GC-owned value on the heap must be reachable by the GC
  * `JS<T>` pointers implement the tracing trait described previously. 
* No reference to a rooted value can outlive its root
  * `JSRef<T>`'s formal type is actually `JSRef<'a, T>`. The `'a` refers to a lifetime variable, which refers to the lifetime of the originating root for this reference. For this reason, the Rust compiler enforces that a `JSRef` value cannot outlive its owning root.

The last point is the one unique to Rust. In Rust, every value has a lifetime that encompasses its allocation and deallocation. This is easiest to understand in the context of lexical scopes:
```
  struct Container<'a, 'b> {
    int_ptr: &'a int,
    str_ptr: &'b str,
  }

  {
    let some_int = 5;                               // 'some_int
    {                                               // ^
      let some_str = "foo";                         // |  'some_str
      {                                             // |  ^
        let container = Container { a: &a, b: &b }; // |  |  'container
        println!("{:?} {:?}",                       // |  |  ^
                 *container.int_ptr,                // |  |  |
                 *container.str_ptr);               // |  |  v
      }                                             // |  v
    }                                               // v
  }
```

`Container` is a structure containing safe references to live values - the Rust compiler enforces this by only allowing pointers to values with lifetimes that are at least as long as the lifetime `'container` in this case. The lifetimes `'some_int` and `'some_str`, which are equivalent to the surrounding lexical scopes, fulfill this property, and are substituted for the generic lifetimes `'a` and `'b'` for the type of the value `container`.



Custom static analysis pass
===========================

As described previously, the Rust compiler can ensure that it's impossible to misuse `JSRef` values and cause use-after-free errors. However, there are no such guarantees about `JS<T>` values, which must only be used as members of heap-allocated types. If it is used as a stack value instead, the compiler does not know that it's no longer safe to create stack-bounded roots from the pointers, as the wrapped value may already have been deallocated. Therefore, we have created a custom static analysis that executes as part of every build and causes a build error if any `JS<T>` value is encountered that is transitively reachable from a stack location.



---

To quickly review, there exists specific allocation control in languages like C++:
```
  Image* image = new Image();
  ...
  delete image;
```
while in JavaScript, the following behaves quite differently, despite appearing superficially similar:
```
  var image = new Image();
  ...
  delete image;
```
In JavaScript, the programmer lacks control over the deallocation of values. The `delete` operator seen here merely removes a reference to the value, but it will not be deallocated until the next time the browser virtual machine invokes a full garbage collection, and only then if there are no other outstanding references to the value. 


