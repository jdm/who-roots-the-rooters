# Safe DOM manipulation in Servo: tools for cross-language memory management

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

We're taking this as another opportunity
in the [Servo project](https://github.com/servo/servo/)
to advance the state of the art.
We have a new approach for DOM memory management,
and we get to use some of
the [Rust](http://www.rust-lang.org/) language's exciting features,
like auto-generated trait implementations,
lifetime checking,
and custom static analysis plugins.


## Memory management for the DOM

It's essential that we never destroy a DOM object
while it's still reachable from either JavaScript or native code —
such [use-after-free bugs](http://cwe.mitre.org/data/definitions/416.html) often produce exploitable security holes.
To solve this problem, most existing browsers use
[reference counting](http://en.wikipedia.org/wiki/Reference_counting)
to track the pointers between underlying low-level DOM objects.
When JavaScript retrieves a DOM object
(through [`getElementById`](https://developer.mozilla.org/en-US/docs/Web/API/document.getElementById) for example),
the browser builds a "reflector" object in the JavaScript VM
that holds a reference to the underlying low-level object.
If the JavaScript garbage collector determines that
a reflector is no longer reachable,
it destroys the reflector
and decrements the reference count
on the underlying object.

This solves the use-after-free issue.
But to keep users happy,
we also need to keep the browser's memory footprint small.
This means destroying objects as soon as
they are no longer needed.
Unfortunately, the cross-language "reflector" scheme
introduces a major complication.

Consider a C++ `Element` object which holds a reference-counted pointer to an `Event`:

```cpp
struct Element {
    RefPtr<Event> mEvent;
};
```

Now suppose we add an event handler to the element from JavaScript:

```javascript
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

Existing browsers resolve this problem in several ways.
Some do nothing, and leak memory.
Some try to manually break possible cycles,
by nulling out `mEvent` for example.
And some implement a [cycle collection](https://developer.mozilla.org/en-US/docs/Interfacing_with_the_XPCOM_cycle_collector) algorithm
on top of reference counting.

None of these solutions are particularly satisfying,
so we're trying something new in Servo by choosing not to
reference count DOM objects at all.
Instead, we give the JavaScript garbage collector full responsibility
for managing those native-code DOM objects.
This requires a fairly complex interaction
between Servo's Rust code
and the [SpiderMonkey](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/SpiderMonkey) garbage collector,
which is written in C++.
Fortunately,
Rust provides some cool features
that let us build this
in a way that's fast, secure, and maintainable.


## Auto-generating field traversals

How will the garbage collector
find all the references between DOM objects?
In [Gecko](https://developer.mozilla.org/en-US/docs/Mozilla/Gecko)'s cycle collector this is done with
a lot of hand-written annotations, e.g.:

```cpp
NS_IMPL_CYCLE_COLLECTION(nsFrameLoader, mDocShell, mMessageManager, mChildMessageManager)
```

This macro describes which members of a C++ class
should be added to a graph of potential cycles.
Forgetting an entry can produce a memory leak.
In Servo the consequences would be even worse:
if the garbage collector can't see all references,
it might free a node that is still in use.
It's essential for both security and programmer convenience
that we get rid of this manual listing of fields.

Rust has a notion of [traits](http://doc.rust-lang.org/tutorial.html#traits),
which are similar to [type classes](http://learnyouahaskell.com/types-and-typeclasses) in Haskell
or interfaces in many OO languages.
A simple example is the [`Collection` trait](http://doc.rust-lang.org/std/collections/trait.Collection.html):

```rust
pub trait Collection {
    fn len(&self) -> uint;
}
```

Any type implementing the `Collection` trait
will provide a method named `len`
that takes a value of the type
(by reference, hence `&self`)
and returns an unsigned integer.
In other words,
the `Collection` trait describes
any type which is a collection of elements,
and the trait provides
a way to get the collection's length.

Now let's look at the [`Encodable` trait](http://doc.rust-lang.org/serialize/trait.Encodable.html),
used for serialization.
Here's a simplified version:

```rust
pub trait Encodable {
    fn encode<T: Encoder>(&self, encoder: &mut T);
}
```

Any type which can be serialized
will provide an `encode` method.
The `encode` method itself is generic;
it takes as an argument
any type `T` implementing the trait `Encoder`.
The `encode` method visits the data type's fields by
calling [`Encoder` methods](http://doc.rust-lang.org/serialize/trait.Encoder.html)
such as `emit_u32`, `emit_tuple`, etc.
The details of the particular serialization format
(e.g. JSON)
are handled by the `Encoder` implementation.

The `Encodable` trait is special,
because the compiler can implement it for us!
Although this mechanism was intended for painless serialization,
it's exactly what we need
to implement garbage collector trace hooks
without manually listing data fields.

Let's look at [Servo's implementation](https://github.com/servo/servo/blob/1c0e51015fc1a5ba0e189f114e35019af27d68ca/src/components/script/dom/document.rs#L68-L80)
of the DOM's [`Document` interface](https://developer.mozilla.org/en-US/docs/Web/API/document):

```rust
#[deriving(Encodable)]
pub struct Document {
    pub node: Node,
    pub window: JS<Window>,
    pub is_html_document: bool,
    ...
}
```

The [`deriving`](http://doc.rust-lang.org/tutorial.html#deriving-implementations-for-traits) attribute
asks the compiler to write an implementation of `encode`
that recursively calls `encode`
on `node`, `window`, etc.
The compiler will complain
if we add a field to `Document` that doesn't implement `Encodable`,
so we have compile-time assurance
that we're tracing all the fields
of our objects.

Note the difference between the `node` and `window` fields above.
In the [object hierarchy of the DOM spec](http://dom.spec.whatwg.org/#interface-document),
every `Document` is also a `Node`.
Rust doesn't have inheritance for data types,
so we implement this by storing a `Node` struct
within a `Document` struct.
As in C++,
the fields of `Node` are included in-line with the fields of `Document`,
without any pointer indirection.
And the auto-generated `encode` method will visit those fields.

A `Document` also has an associated `Window`,
but this is not a containing or "is-a" relationship.
The `Document` just has a pointer to a `Window`,
one of many pointers to that object,
which can live in native DOM data structures
or in JavaScript reflectors.
These are precisely the pointers
we need to tell the garbage collector about.
We do this with a [custom pointer type](https://github.com/servo/servo/blob/1c0e51015fc1a5ba0e189f114e35019af27d68ca/src/components/script/dom/bindings/js.rs#L107-L110) `JS<T>`,
for example `JS<Window>` above.
The implementation of [`encode` for `JS<T>`](https://github.com/servo/servo/blob/1c0e51015fc1a5ba0e189f114e35019af27d68ca/src/components/script/dom/bindings/trace.rs#L51-L56)
is not auto-generated;
this is where we actually call
the SpiderMonkey trace hooks.


## Lifetime checking for safe rooting

The Rust code in Servo
needs to pass DOM object pointers as function arguments,
store DOM object pointers in local variables,
and so forth.
We need to register these additional temporary references
as [roots](http://en.wikipedia.org/wiki/Tracing_garbage_collection#Reachability_of_an_object)
in the garbage collector's reachability analysis.
And we need to make sure we don't touch an object from Rust
when it's not rooted;
this could introduce a use-after-free vulnerability.

To make this happen,
we need to expand our repertoire of GC-managed pointer types.
We already talked about `JS<T>`,
which represents a reference
between two GC-managed DOM objects.
These are not rooted;
the garbage collector only knows about them
when `encode` reaches one
as part of the tracing process.

When we want to use a DOM object from Rust code,
we call the [`root` method](https://github.com/servo/servo/blob/1c0e51015fc1a5ba0e189f114e35019af27d68ca/src/components/script/dom/bindings/js.rs#L155-L161) on `JS<T>`.
For example:

```rust
fn load_anchor_href(&self, href: DOMString) {
    let window = self.window.root();
    window.load_url(href);
}
```

The `root` method returns a `Root<T>`,
which is stored in a stack-allocated local variable.
When the `Root<T>` is destroyed at the end of the function,
its destructor will un-root the DOM object.
This is an example of the [RAII idiom](http://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization),
which Rust inherits from C++.

Of course,
a DOM object might make its way through
many function calls
and local variables
before we're done with it.
We want to avoid the cost of
telling SpiderMonkey about each and every step.
Instead,
we have another type `JSRef<T>`,
which represents a pointer to a GC-managed object
which is already rooted elsewhere.
Unlike `Root<T>`,
`JSRef<T>` can be copied at negligible cost.

We shouldn't un-root an object
if it's still reachable through `JSRef<T>`.
So it's important that
a `JSRef<T>` can't outlive
its originating `Root<T>`.
Situations like this are common in C++ as well.
No matter how smart your smart pointer is,
you can take a bare reference to the contents
and then erroneously use that reference
past the lifetime of the smart pointer.

Rust solves this problem
with a compile-time [lifetime checker](http://doc.rust-lang.org/guide-lifetimes.html).
The type of a reference includes
the region of code over which it is valid.
In most cases,
lifetimes are [inferred](http://en.wikipedia.org/wiki/Type_inference)
and don't need to be written out in the source code.
Inferred or not,
the presence of lifetime information
allows the compiler to reject
use-after-free and other dangerous bugs.

Not only do lifetimes protect
Rust's built-in reference type,
we can use them in our own data structures
as well.
`JSRef` is actually [defined](https://github.com/servo/servo/blob/1c0e51015fc1a5ba0e189f114e35019af27d68ca/src/components/script/dom/bindings/js.rs#L443-L447) as

```rust
pub struct JSRef<'a, T> {
    ...
```

`T` is the familiar type variable,
representing the type of DOM structure we're pointing to,
e.g. `Window`.
The somewhat odd syntax `'a` is a [lifetime variable](http://doc.rust-lang.org/guide-lifetimes.html#named-lifetimes),
representing the region of code
in which that object is rooted.
Crucially,
this lets us write a [method](https://github.com/servo/servo/blob/1c0e51015fc1a5ba0e189f114e35019af27d68ca/src/components/script/dom/bindings/js.rs#L415-L419) on `Root`
with the following signature:

```rust
pub fn root_ref<'a>(&'a self) -> JSRef<'a, T> {
    ...
```

What this syntax means is:

* **`<'a>`**: "for any lifetime `'a`",
* **`(&'a self)`**: "take a reference to a `Root` which is valid over lifetime `'a`",
* **`-> JSRef<'a, T>`**: "return a `JSRef` whose lifetime parameter is set to `'a`".

The final piece of the puzzle
is that we put a [marker](http://doc.rust-lang.org/std/kinds/marker/struct.ContravariantLifetime.html) in the `JSRef` type
saying that it's only valid
for the lifetime corresponding to that parameter `'a`.
This is how we extend the lifetime system
to enforce our application-specific property
about garbage collector rooting.
If we try to compile something like this:

```rust
fn bogus_get_window<'a>(&self) -> JSRef<'a, Window> {
    let window = self.window.root();
    window.root_ref()  // return the JSRef
}
```

we get an error:

```
document.rs:199:9: 199:15 error: `window` does not live long enough
document.rs:199         window.root_ref()
                        ^~~~~~
document.rs:197:57: 200:6 note: reference must be valid for the lifetime 'a as defined on the block at 197:56...
document.rs:197     fn bogus_get_window<'a>(&self) -> JSRef<'a, Window> {
document.rs:198         let window = self.window.root();
document.rs:199         window.root_ref()
document.rs:200     }
document.rs:197:57: 200:6 note: ...but borrowed value is only valid for the block at 197:56
document.rs:197     fn bogus_get_window<'a>(&self) -> JSRef<'a, Window> {
document.rs:198         let window = self.window.root();
document.rs:199         window.root_ref()
document.rs:200     }
```

We also [implement](https://github.com/servo/servo/blob/1c0e51015fc1a5ba0e189f114e35019af27d68ca/src/components/script/dom/bindings/js.rs#L429-L441)
the [`Deref` trait](http://doc.rust-lang.org/std/ops/trait.Deref.html)
for both `Root<T>` and `JSRef<T>`.
This allows us to access
fields of the underlying type `T`
through a `Root<T>` or `JSRef<T>`.
Because `JS<T>` does *not* implement `Deref`,
we have to root an object
before using it.

The DOM methods of `Window` (for example)
are defined in [a trait](https://github.com/servo/servo/blob/1c0e51015fc1a5ba0e189f114e35019af27d68ca/src/components/compositing/windowing.rs#L61-L81)
which is implemented for `JSRef<Window>`.
This ensures that `self` is rooted
for the duration of the method call,
which would not be guaranteed
if we implemented the methods
on `Window` directly.

You can check out
the [Servo project wiki](https://github.com/servo/servo/wiki/Using-DOM-types)
for more of the details that didn't make it into this article.

## Custom static analysis

To recap,
the safety of our system depends on
two major parts:

* The auto-generated `encode` methods ensure that SpiderMonkey's garbage collector can see all of the references between DOM objects.
* The implementation of `Root<T>` and `JSRef<T>` guarantee that we can't use a DOM object from Rust without telling SpiderMonkey about our temporary reference.

But there's a hole in this scheme.
We could copy an unrooted pointer
— a `JS<T>`
— to a local variable on the stack,
and then at some later point,
root it
and use the DOM object.
In the meantime,
SpiderMonkey's garbage collector won't know about
that `JS<T>` on the stack,
so it might free the DOM object.
To really be safe,
we need to make sure that `JS<T>`
*only* appears in traceable DOM structs,
and never in local variables,
function arguments,
and so forth.

This rule doesn't correspond to
anything that already exists in Rust's type system.
Fortunately,
the Rust compiler can load
"[lint plugins](https://github.com/rust-lang/rust/pull/15024)" providing custom static analysis.
These basically take the form of new compiler warnings,
although in this case we set the default severity to "error".

We have already [implemented a plugin](https://github.com/kmcallister/servo/commit/c20b50bbbbcdc8ce3551adbc1e039a727cf89995)
which simply forbids `JS<T>` from appearing at all.
Because lint plugins are part of
the usual [warnings infrastructure](http://doc.rust-lang.org/rust.html#lint-check-attributes),
we can use the `allow` attribute in places
where it's okay to use `JS<T>`,
like DOM struct definitions
and the implementation of `JS<T>` itself.

Our plugin looks at every place where the code mentions a type.
Remarkably, this adds only a fraction of a second
to the compile time for Servo's largest subcomponent.
(Rust compile times are dominated by
[LLVM](http://llvm.org/)'s back-end optimizations
and code generation.)
The current version of the plugin
is very simple
and will miss some mistakes,
like storing a struct containing `JS<T>` on the stack.
However,
lint plugins run at a late stage of compilation
and have access to full compiler internals,
including the results of type inference.
So we can make the plugin
incrementally more sophisticated in the future.

We won't necessarily catch every possible mistake.
It's hard to achieve full [soundness](http://en.wikipedia.org/wiki/Soundness)
with ad-hoc extensions to a type system.
As the name "lint plugin" suggests,
the idea is to catch common mistakes
at a low cost to programmer productivity.
By combining this
with the lifetime checking built in to Rust's type system,
we hope to achieve a degree of security and reliability
far beyond what's feasible in C++.
And the checking is all done at compile time;
there's no penalty in the generated machine code.

It's an open question
how our garbage-collected DOM will perform
compared to a traditional reference-counted DOM.
The [Blink](http://www.chromium.org/blink) team has performed
[similar experiments](http://www.chromium.org/blink/blink-gc),
but they don't have Servo's luxury
of starting from a clean slate
and using a cutting-edge language.
We expect the biggest gains will come
when we move to allocating DOM objects
within the JavaScript reflectors themselves.
Since the reflectors need to be traced no matter what,
this will reduce the cost of managing native DOM structures
to almost nothing.

If you find this stuff interesting,
we'd love to have your help on Rust and Servo!
Both are open-source projects
with a large number of community contributors.
Here are some resources for getting started:

* The [Rust Tutorial](http://doc.rust-lang.org/tutorial.html) and the [Rust Guide](http://doc.rust-lang.org/guide.html)
* Places to talk about Rust: [Reddit](http://www.reddit.com/r/rust), [mailing list](https://mail.mozilla.org/listinfo/rust-dev), [IRC](http://client00.chat.mibbit.com/?server=irc.mozilla.org&channel=%23rust) (`#rust` on `irc.mozilla.org`)
* [Guide for new Rust contributors](https://github.com/rust-lang/rust/wiki/Note-guide-for-new-contributors)
* [Guide for contributing to Servo](https://github.com/servo/servo/blob/master/CONTRIBUTING.md)
* ["Easy" issues in Servo](https://github.com/servo/servo/labels/E-easy)
* [Servo IRC channel](http://client00.chat.mibbit.com/?server=irc.mozilla.org&channel=%23rust): `#servo` on `irc.mozilla.org`
