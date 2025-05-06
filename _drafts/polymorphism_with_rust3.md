---
layout: post
title: "Polymorphism with Rust, part 3: Trait objects"
tags: [rust, polymorphism, inheritance, trait objects, dyn trait]
---

This is the fourth post in a four part series on polymorphism with Rust. In this
post I discuss how trait objects can be used to implement polymorphism in Rust.

<div class="table-of-contents" markdown=1>
**Contents**

* Do not remove this line (it will not be displayed)
{:toc}

**This series**

<!-- TODO: Add post URLs -->
- [Part 0: Introduction]({% post_url 2025-05-06-polymorphism_with_rust0 %})
- [Part 1: Enums]({% post_url 2025-05-06-polymorphism_with_rust1 %})
- [Part 2: Generics](/)
- [**Part 3: Trait objects**](/)
</div>

## What are trait objects?

Trait objects are a special type in Rust represented by `dyn Trait`, that can
represent any type that implements a given trait. Unlike generics which do
compile time monomorphization, trait objects use dynamic dispatch at run time.
This means we do not need to know the underlying type at compile time. This
gives trait objects a lot of flexibility but also means that they are an unsized
type, because the compiler does not know the size of the underlying type at
compile time. As we will see this comes with certain limitations.

The most notable one is that we cannot directly refer to `dyn Trait` in our
function or struct definitions, some level of indirection is required. In our
examples we will focus on `&dyn Trait` and `Box<dyn Trait>`, but this also works
with other pointer types in Rust like `Rc` and `Arc`.

> **Aside**<br />
> Wondering what the difference between `&dyn Trait` and `Box<dyn Trait>` is?
> They're both just pointers to something that implements a trait right? The
> fundamental difference is that `Box<dyn Trait>` takes ownership of the
> underlying instance while `&dyn Trait` simply borrows it.

## Polymorphic functions

Updating our `get_ray_color` function to use `dyn Trait` is quite straight
forward:

```rust
fn get_ray_color(ray: &Ray, object: &dyn Hittable) -> Color {
    // ...
}
```
It will then accept either a reference to a `Sphere` or a `Triangle` much like
the generic version:

```rust
let ray = Ray {
    // ...
};
let sphere = Sphere {
    // ...
};
let triangle = Triangle {
    // ...
};

let _ = get_ray_color(&ray, &sphere);
let _ = get_ray_color(&ray, &triangle);
```

However this does not offer any benefits over the generic version, so in most
cases I would recommend sticking with that. The real benefit of `dyn Trait` we
will see in the next section.

## Polymorphic lists

Unlike generics trait objects are really the solution we need to make our scene
accept any type that implements a trait. Although the size of a trait object is
not known at compile time the size of a pointer to it is and so we can create a
vector of such pointers. In the case of our `Scene` struct we want it to take
ownership of the objects passed into it so we will use `Box<dyn Hittable>`.

```rust
struct Scene {
    pub objects: Vec<Box<dyn Hittable>>,
}
```

We can now happily add spheres, triangles, and any other type that implements
`Hittable` to our scene:

```rust
fn make_scene() -> Scene {
    Scene {
        objects: vec![
            Box::new(Sphere {
                center: (0.0, 0.0, 0.0),
                radius: 0.0,
            }),
            Box::new(Triangle {
                point1: (0.0, 0.0, 0.0),
                point2: (0.0, 0.0, 0.0),
                point3: (0.0, 0.0, 0.0),
            }),
        ],
    }
}
```

Note that typically you would not want to box elements before adding them to a
vector as this is really a double indirection, however in this case it is
necessary because of the nature of trait objects.

### `impl Trait for dyn trait`

You might be wondering about passing a trait object to our generic
`get_ray_color` function from earlier in this series.

```rust
fn get_ray_color<H: Hittable>(ray: &Ray, object: &H) -> Color {
    let does_hit = object.does_hit(ray);
    // ...
}
```

The `dyn Hittable` type does implement `Hittable`, that's kind of it's whole
thing. There is also an automatic implementation provided for `&dyn Hittable`.
So this should be straight forward right? Let's try the following:

```rust
let sphere: &dyn Hittable = &Sphere {
    center: (0.0, 0.0, 0.0),
    radius: 0.0,
};

let _ = get_ray_color(&ray, sphere);
```

This fails to compile with the error 'the size for values of type `dyn Hittable`
cannot be known at compilation time'. This is because there is in fact an
implicit `Sized` constraint applied to all generic types and `dyn Hittable` is
unsized. So in order for this to work we need to relax this constraint, this can
be done with the rather unique `?Sized` syntax as follows:

```rust
fn get_ray_color<H: Hittable + ?Sized>(ray: &Ray, object: &H) -> Color {
    let does_hit = object.does_hit(ray);
    // ...
}
```

What if our function takes ownership of our trait object, something like:

```rust
fn get_ray_color_own<T: Hittable>(ray: &Ray, object: T) -> Color {
    let does_hit = object.does_hit(ray);
    // ...
}
```

As we have seen before the size of our trait object is not known at compile time
so we cannot pass in a raw trait object, this leaves us with `Box<dyn Hittable>`.
However a `Box<dyn Trait>` does not implement the trait by default. So we need
to first implement `Hittable` for `Box<dyn Hittable>`. Naively we might do the
following:

```rust
impl Hittable for Box<dyn Hittable> {
    fn does_hit(&self, ray: &Ray) -> bool {
        self.does_hit(ray)
    }
}
```

However this causes an infinite recursion because when we call `does_hit` on
`self`, which is a `Box<dyn Hittable>` it calls the method we are currently
defining. To get around this we need to explicitly call the `does_hit` method
defined on `dyn Hittable`, like so:

```rust
impl Hittable for Box<dyn Hittable> {
    fn does_hit(&self, ray: &Ray) -> bool {
        <dyn Hittable>::does_hit(self, ray)
    }
}
```

> **Aside**<br />
> There is slightly cleaner way of achieving this by doubly dereferencing
> `self`, but personally I think using the explicit version above is more
> readable, especially to someone who hasn't seen this before.
>
> ```rust
> impl Hittable for Box<dyn Hittable> {
>     fn does_hit(&self, ray: &Ray) -> bool {
>         (**self).does_hit(ray)
>     }
> }
> ```

Using the `get_ray_color_own` function we defined above we can now do the
following:

```rust
let sphere: Box<dyn Hittable> = Box::new(Sphere {
    center: (0.0, 0.0, 0.0),
    radius: 0.0,
});

let _ = get_ray_color_own(&ray, sphere);
```

If you want more information on implementing a trait for `Box<dyn Trait>` I
would recommend this [article by QuineDot](https://quinedot.github.io/rust-learning/dyn-trait-box-impl.html),
as there are a few more considerations not discussed here.

## Advanced use cases

### Multiple traits

What about our advanced case of requiring a trait object to be clone-able and
hittable. Imagine the following:

```rust
pub fn get_ray_color<H: Hittable + Clone + ?Sized>(ray: &Ray, object: &H) -> Color {
    let does_hit = object.does_hit(ray);
    let _clone = (*object).clone();
    // ...
}
```

Can we just pass in a reference to our trait object?

```rust
let sphere: &dyn Hittable = &Sphere {
    // ...
};

let _ = get_ray_color(&ray, sphere);
```

Unsurprisingly this does not work, the compiler telling us that `dyn Hittable`
does not meet the required `Clone` constraint on our function. Recall that
`Sphere` does implement clone though so can we simply supply multiple
constraints to our trait object like we can with generics?

```rust
let sphere: &(dyn Hittable + Clone) = &Sphere {
    // ...
};

let _ = get_ray_color(&ray, sphere);
```

Unfortunately the compiler says no, stating 'only auto traits can be used as
additional traits in a trait object'. As this message suggests there are certain
traits, called auto-traits, that this will work with such as `Send` and `Sync`.

There is another option, what if we define a new trait that combines `Hittable`
and `Clone` as supertraits?

```rust
trait HittableClone: Hittable + Clone {}

let sphere: &dyn HittableClone = &Sphere {
    // ...
};

let _ = get_ray_color(&ray, sphere);
```

Once again the compiler says no, stating that 'the trait `HittableClone` is not
dyn compatible'. Dyn compatibility is probably a whole topic on it's own, but in
this case the error arises because `Clone` requires the `Sized` trait. Since
`dyn Trait` is unsized it can never be clone-able.

So is this just not possible then? Well not quite. As we have seen before
`Box<dyn Trait>` is sized so this means that so long as the underlying type is
clone-able we can clone `Box<dyn Trait>`. We start by creating a new trait that
defines this behaviour, our `Hittable` trait will take this as a super trait.

```rust
trait DynHittableClone {
    fn dyn_clone<'a>(&self) -> Box<dyn Hittable + 'a>
    where
        Self: 'a;
}

trait Hittable: DynHittableClone {
    fn does_hit(&self, ray: &Ray) -> bool;
}
```

Not the lifetime `'a`, this removes the need for adding an explicit `'static`
constraint to `Hittable`.

Next we will create a blanket implementation for any type that is hittable and
clone-able, this covers our `Sphere` and `Triangle` types.

```rust
impl<T: Hittable + Clone> DynHittableClone for T {
    fn dyn_clone<'a>(&self) -> Box<dyn Hittable + 'a>
    where
        Self: 'a,
    {
        Box::new(self.clone())
    }
}
```

Finally we implement `Clone` for `Box<dyn Hittable>`, note that we have to be
very specific about which version of `dyn_clone` we call to avoid an infinite
recursion.

```rust
impl Clone for Box<dyn Hittable> {
    fn clone(&self) -> Self {
        <dyn Hittable as DynHittableClone>::dyn_clone(self)
    }
}
```

Now we can at last pass a trait object into our function that needs to clone it:

```rust
let sphere: Box<dyn Hittable> = Box::new(Sphere {
    // ...
});

let _ = get_ray_color(&ray, &sphere);
```

If this seems painful it sure is, fortunately there is a crate that can simplify
this a lot, [dyn-clone](https://crates.io/crates/dyn_clone).

### Underlying type

What about getting back the underlying type of a trait object? Well as we saw in
the generics section this is possible via the builtin `Any` trait. The `Any`
trait is designed to allow for dynamic typing in Rust, it specifically does this
via the `dyn Any` which will allow you to fallibly downcast it to any type.

So if we want to apply special handling to for a specific underlying type of
`dyn Hittable` we first need to upcast it to `dyn Any` then try to downcast it
to the underlying type we are looking for. As of Rust 1.86 trait upcasting
allows us to upcast any trait to a super trait, which should make this quite
straight forward. For this to work we need to explicitly list `Any` as a super
trait of `Hittable`:

```rust
use std::any::Any;

trait Hittable: Any {
    // ...
}
```

Then the following will work:


```rust
let some_hittable: &dyn Hittable = &Sphere {
    //...
};

let any_hittable = some_hittable as &dyn Any; // Upcast
match any_hittable.downcast_ref::<Triangle>() { // Downcast
    Some(triangle) => todo!(), // Do something special here
    None => (),
}
```

For even more information on `dyn Trait` I would recommend 'A tour of dyn Trait'
in this [book by QuineDot](https://quinedot.github.io/rust-learning/dyn-trait.html).

### Nested type

## Advantages & disadvantages

The main advantage of `dyn Trait` is that it can solve problems that the other
methods we have seen here just can't. Such as creating a list of elements that
all implement a trait without knowing the complete list of types that implement
that trait, like we saw with our `Scene` struct.

Unfortunately it has several downsides, most notably in my opinion is that it
restrictions that can be difficult to work with as we saw with implementing
`Clone` on `Box<dyn Trait>`. I hope this will improve in the future as Rust
continues to add more features.

The dynamic dispatch of functions implemented by `dyn Trait` also has some
performance overhead as the function needs to be looked up from a table at run
time. It also blocks the compiler from doing certain optimizations like inlining
a function. These minor but not nothing.

Its also worth noting that use of `Box<dyn Trait>` requires a heap allocation
that could potentially be avoided.

## Summary

To summarise this series there are several ways of doing polymorphism in Rust.
Personally I prefer using enums and generics, but trait objects are good to know
about for when you need them.

Hopefully you learnt something new!
