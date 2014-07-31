Goals:
* deriving trace hooks
* lifetimes for safe rooted references
* lint for unsafe type uses

---


Traditionally, web browsers have implemented performance-sensitive parts of the DOM in languages with good performance characteristics like C++. For example, DOM manipulation (appending/removing nodes) must be fast in order for a browser to remain competitive, so every modern browser has a `Node` type written in a low-level language. However, these APIs must remain accessible to web content written in JavaScript, so browsers must expose "reflectors" - JavaScript objects that reflect the properties and methods defined the by the specification which just call through to the underlying implementation as transparently as possible.

---


There are two particularly interesting challenges when it comes to implementing a cross-language, garbage-collected system in a non-garbage-collected language: safety and reclamation. The first refers to the difficulty of using and storing pointers to memory which could conceivably be deleted at any time, while the second describes the problem of when to free the memory in use. To solve the first, most browsers choose to use reference-counting internally, thus changing the ownership semantics of the JS reflectors - now the reflector is merely one more strong reference to the low-level value. This complicates the reclamation problem, however. Consider the following:
```
//C++
struct Element {
  RefPtr<Event> mEvent;
}

//JavaScript
elem.addEventListener('load', function(event) {
  event.originalTarget = elem;
});
```
Here we see a cross-language cycle, where the C++ code contains a strong reference to an Event object. This event is dispatched, and its handler sets an "expando property" on the event object's reflector which acts as a strong reference to the owner of the event handler. If this owner is the same element that is storing the event, we no longer have a clear way to decide when to release the memory for these two objects, since the full ownership graph is not available to us. Modern browsers resolve this problem in several ways - some do nothing (and leak memory), some try to manually break possible cycles (by nulling out mEvent, for example), and some implement a cycle collection algorithm and break them automatically. The fundamental difficulty here stems from not having the full, cross-language ownership graph available, and this is where Servo is attempting to improve the state of the art. 

Servo's solution is easy to describe - we're giving the SpiderMonkey garbage collector full control over all DOM values. To accomplish this in a safe and sound manner, we've used three exciting features of the Rust language and compiler: compiler-derived traits, lifetimes, and a custom static analysis pass.

---

Compiler-derived traits
=======================

For the garbage collector to have full control, we must be confident that it actually can see the full ownership graph when it attempts to collect garbage. Gecko's C++ cycle collector faces a similar problem, since there is a lot of handwritten code that looks like this:
```
NS_IMPL_CYCLE_COLLECTION(nsFrameLoader, mDocShell, mMessageManager, mChildMessageManager)
```
This is a macro that declares the members of a C++ class that should be added to the graph of potential cycles. The effect of forgetting an entry in this list in Gecko is a potential memory leak. In Servo, if we miss declaring an edge from one GC-owned value to another, we can end up with an exploitable use-after-free error. Therefore, we need a way to make forgetting such a link a compile-time error, and we get that in the form of derived traits.

In Rust, a trait is effectively an interface (or a [type class](http://en.wikipedia.org/wiki/Type_class), if that means more to you). The language also provides the ability to annotate arbitrary types with a `deriving` annotation, which indicates to the compiler that every contained member of the type must also implement the trait. It's easy to imagine this being useful for serialization purposes, and we make use of it to ensure that every member of a DOM type in Servo can be understood by the SpiderMonkey garbage collector. The trick is that SpiderMonkey allows each reflector to specify a "trace hook" - a function pointer that will be called each time the reflector is encountered in the object graph while performing a garbage collection. This function ends up calling the method of the trait our types derive (`deriving(Encodable)` yields `a.encode(some_encoder)`), which then recurses on each member of the type. If we add a member that doesn't implement the same trait, the compiler yells at us. Transitively, this yields the property that any garbage-collected value that is reachable via the Rust type system is visible by SpiderMonkey's garbage collector, and with a minimum of boilerplate, no less!


Lifetimes
=========

By yielding all control over deallocation to SpiderMonkey's garbage collector, we now need to solve the safety problem that other browsers use reference counting to avoid. We've developed a simple set of types that enforce sound rooting practices, such that it should be impossible to use pointers to GC-owned values in unsafe ways.

* Root<T> - a stack-allocated value that "roots" a GC-owned value for the duration of the root's lifetime (ie. prevents it being collected)
* JSRef<T> - a freely-cloneable smart pointer to a rooted, GC-owned value that can be dereferenced to interact with the wrapped value
* JS<T> - a pointer to a GC-owned value
* Temporary<T> - a stack-allocated, movable pointer to a GC-owned value that ensures its wrapped value is rooted for the duration of the wrapper's lifetime

These types allow us to enforce the following rules that make Servo's DOM implementation safe:

* All interactions with the underlying GC-owned value (both calling methods and accessing members) must occur through a root
** Only `JSRef` implements any dereferencing behaviour. `Temporary` and `JS` values merely carry around a pointer and provide facilities to create stack-bounded roots.
** All methods take `JSRef` arguments, and all methods for DOM type Foo are implemented on `JSRef<Foo>` types, rather than `Foo` itself.
* Any reference to a GC-owned value on the heap must be reachable by the GC
** `JS<T>` pointers implement the tracing trait described previously. 
* No reference to a rooted value can outlive its root
** `JSRef<T>`'s formal type is actually `JSRef<'a, T>`. The `'a` refers to a lifetime variable, which refers to the lifetime of the originating root for this reference. For this reason, the Rust compiler enforces that a `JSRef` value cannot outlive its owning root.


Custom static analysis pass
===========================

As described previously, the Rust compiler can ensure that it's impossible to misuse `JSRef` values and cause use-after-free errors. However, there are no such guarantees about `JS<T>` values, which must only be used as members of heap-allocated types. If it is used as a stack value instead, the compiler does not know that it's no longer safe to create stack-bounded roots from the pointers, as the wrapped value may already have been deallocated. Therefore, we have created a custom static analysis that executes as part of every build and causes a build error if any `JS<T>` value is encountered that is transitively reachable from a stack location.



---




The [Document Object Model](http://dom.spec.whatwg.org/) defined by the HTML specification is garbage-collected.

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


---

Browsers that implement the DOM in a non-garbage-collected language face a fundamental challenge of bridging this divide

