
The Rust reference counting `Rc` type [implements the `Clone` trait](https://doc.rust-lang.org/std/rc/struct.Rc.html#impl-Clone). Every call to `clone` increments the strong count for the `Rc` instance. This allows us to count how many times `Rc::clone` has been called.

The Rust shared reference type `&T` also [implements the `Clone` trait](https://doc.rust-lang.org/std/primitive.reference.html#trait-implementations-1). (`T` can be any type including `Rc`). Note that the documentation for `&T` points out that "this will not defer to T’s Clone implementation".



Consider this code:
```rust
use std::rc::Rc;

fn main() {
	// A new `Rc` has a count of 1.
	let a = Rc::new(());
	assert!(Rc::strong_count(&a) == 1);

	// Calling `a.clone()` uses the `Rc::clone` implementation and increments the count to 2.
	let b = a.clone();
	assert!(Rc::strong_count(&a) == 2);

	// Calling `(&&a).clone()` uses the `&T::clone` implementation and the count is unchanged.
	let c = (&&a).clone();
	assert!(Rc::strong_count(&a) == 2);

	// What happens when we call `(&a).clone()`?
	let d = (&a).clone();
	dbg!(Rc::strong_count(&a));
}
```

The receiver for a method is the expression before the dot. For example, for `a.clone()` the receiver is `a`.

When assigning to `b` the receiver `a` has type `Rc<()>`. The reference count increases, indicating that we called `Rc::clone`.

When assigning to `c` the receiver `&&a` has type `&&Rc<()>`. The reference count does not increase, indicating that we did not call `Rc::clone`. We called `&T::clone` instead, and, as we know, that does not use `Rc`'s Clone implementation.

The question is what happens when we call `(&a).clone()`?
The receiver has type `&Rc<()>`. This type does not match `Rc<T>`. It does match `&T`, where `T` is `Rc<()>`.

The intuitive answer is that `(&a).clone()` calls the `&T::clone` implementation, the count remains unchanged, and the code prints `Rc::strong_count(&a) = 2`.

So it might seem surprising that running the code prints `Rc::strong_count(&a) = 3`.

---

When you look at a Rust `impl` block, such as `impl<T> Rc<T>`, it appears to implement methods for a single (possibly generic) type (`Rc<T>`), and the different forms for methods (`self`, `&self`, `&mut self`) just indicate how the receiver is borrowed.

However the `self` argument to the method plays a more significant part than expected.

An `impl` block contains methods for multiple types that are ["associated" with the named "implementing type"](https://doc.rust-lang.org/reference/items/associated-items.html#methods).

The `Clone` trait provides the method signature `fn clone(&self) -> Self`, shorthand for `fn clone(self: &Self) -> Self`

When `Self` is `Rc`:
```rust
impl Clone for Rc<T> {
	fn clone(self: &Rc<T>) -> Rc<T> {...}
}
```
This implements a `clone` method that matches the type `&Rc<T>`. So `Rc<T>::clone` gets called when the receiver is `&Rc<T>`, which includes `&Rc<()>`.

Similarly, `impl<T> Clone for &T` implements a `clone` method matching the generic type `&&T` which includes `&&Rc<T>`. So ``&T::clone` gets called when the receiver is `&&T`, which includes `&&Rc<()>`.

There is no `clone` method that matches the type `Rc<T>`.

The question is not why does `&a.clone()` call `Rc::clone`. The question is why does `a.clone()` call `Rc::clone`?

---

`a.clone()` works because Rust uses a [consistent procedure to match methods against a chain of types](https://doc.rust-lang.org/reference/expressions/method-call-expr.html) derived from the receiver. The chain starts `Rc<()>`, `&Rc<()>`, `&mut Rc<()>`, and then continues through any `Deref` trait. For each type in the chain, Rust looks for a method that matches.

For `a.clone()` the first candidate, `Rc<()>`, does not match any `clone` method. The second candidate, `&Rc`, does: the `Rc::clone` implementation.  So Rust creates a reference to the receiver, and passes that to the `Rc::clone` implementation, incrementing the count to 3.

For another example of this behaviour using non-standard-library types, see https://users.rust-lang.org/t/when-will-auto-deref-happen-for-receiver-in-method-call/43209
