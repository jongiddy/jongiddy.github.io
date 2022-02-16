---
title: "Rust Method Resolution"
date: 2022-02-14T11:19:13Z
draft: false
---

Consider this code:
```rust
use std::rc::Rc;

fn main() {
	// A new `Rc` has a count of 1.
	let a = Rc::new(());
	assert!(Rc::strong_count(&a) == 1);

	// Calling `a.clone()` uses the `Rc::clone` implementation
	// and increments the count to 2.
	let b = a.clone();
	assert!(Rc::strong_count(&a) == 2);

	// Calling `(&&a).clone()` uses the `&T::clone` implementation
	// and the count is unchanged.
	let c = (&&a).clone();
	assert!(Rc::strong_count(&a) == 2);

	// What happens when we call `(&a).clone()`?
	let d = (&a).clone();
	dbg!(Rc::strong_count(&a));
}
```


The Rust reference counting type [`Rc` implements the `Clone` trait](https://doc.rust-lang.org/std/rc/struct.Rc.html#impl-Clone). Every call to `clone` increments the strong count for the `Rc` instance. This allows us to count how many times `Rc::clone` has been called.

---
![Rc::Clone](/rc-clone-doc.png)

---

The Rust shared reference type [`&T` also implements the `Clone` trait](https://doc.rust-lang.org/std/primitive.reference.html#trait-implementations-1). (`T` can be any type including `Rc`). Note that the documentation for `&T` points out that "this will not defer to T’s Clone implementation".

---
![&T::Clone](/ref-clone-doc.png)

---

The *receiver* for a method is the expression before the dot. With `a.clone()` the receiver is `a` with type `Rc<()>`.
The reference count increases, showing that we called `Rc::clone`.

```rust
	// Calling `a.clone()` uses the `Rc::clone` implementation
	// and increments the count to 2.
	let b = a.clone();
	assert!(Rc::strong_count(&a) == 2);
}
```

With `(&&a).clone()` the receiver `&&a` has type `&&Rc<()>`.
The reference count does not increase, showing that we did not call `Rc::clone`.
We called `&T::clone` instead, and that does not delegate to `Rc`'s `Clone` implementation.

```rust
	// Calling `(&&a).clone()` uses the `&T::clone` implementation
	// and the count is unchanged.
	let c = (&&a).clone();
	assert!(Rc::strong_count(&a) == 2);
```

The question is what happens when we call `(&a).clone()`?

```rust
	// What happens when we call `(&a).clone()`?
	let d = (&a).clone();
	dbg!(Rc::strong_count(&a));
```

The receiver has type `&Rc<()>`. This type does not match `Rc<T>`. It does match `&T`, where `T` is `Rc<()>`.

The intuitive answer is that `(&a).clone()` calls the `&T::clone` implementation, the count remains unchanged, and the code prints `Rc::strong_count(&a) = 2`.

But the code does in fact print `Rc::strong_count(&a) = 3`, showing that it calls `Rc::clone`.

When you look at a Rust `impl` block, such as `impl<T> Rc<T> {}`, it appears to implement methods for a single (possibly generic) receiver type (`Rc<T>`), and the different forms for methods (`self, &self, &mut self`) indicate how the receiver is borrowed in the method.

However the `self` argument to the method plays a more significant part than expected.

First, it's useful to know that the ubiquitous method parameters `&self` and `&mut self` are syntactic sugar to combine a parameter called `self` with the parameter's type:

![Shorthand Self](/self-shorthand.png)

This helps understand this [section of the docs](https://doc.rust-lang.org/reference/items/associated-items.html#methods):

---
![Self Types](/self-types.png)

---

What this means is that an `impl` block can define methods for multiple types. These types are related to the type named in the `impl` block. For example, an `impl<T> Rc<T>` block can define methods for `Rc<T>`, `&Rc<T>`, `&mut Rc<T>`, `Box<Rc<T>>`, `Rc<Rc<T>>`, `Arc<Rc<T>>`, `Pin<Rc<T>>`, and also combinations such as `&Box<Pin<Rc<T>>>`.

The `Clone` trait defines the method signature `fn clone(&self) -> Self`, shorthand for `fn clone(self: &Self) -> Self`

When `Self` is `Rc<T>`:
```rust
impl Clone for Rc<T> {
	fn clone(self: &Rc<T>) -> Rc<T> {...}
}
```

The implementing type is `Rc<T>` but the method `clone` is implemented for the associated type `&Rc<T>`.
So `Rc<T>::clone` gets called when the receiver is `&Rc<T>`, including for `&Rc<()>`.

Similarly, `impl<T> Clone for &T` implements a `clone` method matching the generic type `&&T`. So `&T::clone` gets called when the receiver is `&&T`, which includes `&&Rc<()>`.

```rust
impl Clone for &T {
	fn clone(self: &&T) -> &T {...}
}
```

There is no `clone` method for the type `Rc<T>`.

So our question above was misplaced. It is not why does `&a.clone()` call `Rc::clone`.

The real question is why does `a.clone()` call `Rc::clone` when `Rc<()>` does not have a `clone` method?

`a.clone()` works because Rust uses a [procedure that matches methods against a chain of types](https://doc.rust-lang.org/reference/expressions/method-call-expr.html) derived from the receiver.

---
![Method Resolution](/method-resolution.png)

---

For `Rc<()>` the chain starts `Rc<()>, &Rc<()>, &mut Rc<()>`, and then continues through `Deref` to `(), &(), &mut ()`. For each type in the chain, Rust looks for a method that matches.

For `a.clone()` the first candidate, `Rc<()>`, does not match any `clone` method. So we move on.

The second candidate, `&Rc<()>`, does: the `Rc::clone` implementation.  So Rust creates a reference to the receiver, and passes that to the `Rc::clone` implementation, incrementing the count to 3.

If this example has too much reference counting and not enough dogs, then this [`users.rust-lang.org` post](https://users.rust-lang.org/t/when-will-auto-deref-happen-for-receiver-in-method-call/43209) is for you!
