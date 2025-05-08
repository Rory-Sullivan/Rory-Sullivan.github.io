---
layout: post
title: "Polymorphism with Rust, part 2: Generics"
tags: [rust, polymorphism, inheritance, generics]
---

This is the third post in a four part series on polymorphism with Rust. In this
post I discuss how generics can be used to implement polymorphism in Rust.

<div class="table-of-contents" markdown=1>
**Contents**

* Do not remove this line (it will not be displayed)
{:toc}

**This series**

<!-- TODO: Add post URLs -->
- [Part 0: Introduction]({% post_url 2025-05-06-polymorphism_with_rust0 %})
- [Part 1: Enums]({% post_url 2025-05-06-polymorphism_with_rust1 %})
- [**Part 2: Generics**](/)
- [Part 3: Trait objects](/)
</div>

## What are generics?

Generics allow us to generalize a function for multiple types, and we are able
to specify certain traits that a generic type must implement. The syntax for
this is as follows:

```rust
fn do_something<T: Trait>(x: T) {}
```

> **Aside**<br />
> There is an alternative equivalent syntax using a where clause:
>
> ```rust
> fn do_something<T>(x: T) where T: Trait {}
> ```

This is achieved by compile time monomorphization. In other words the compiler
creates a different version of the function for each concrete type it is called
with. You can read more about generics
[here](https://doc.rust-lang.org/book/ch10-00-generics.html).

## Polymorphic functions

This is perfect for our `get_ray_color` function as we can see here:

```rust
fn get_ray_color<H: Hittable>(ray: &Ray, object: &H) -> Color {
    let does_hit = object.does_hit(ray);
    // ...
}
```

When we specify `H` as `<H: Hittable>` this implies that any type that is
substituted for `H` must implement `Hittable`. Hence we can pass in either a
`Sphere` or a `Triangle` to this function.

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

Since we implemented `Hittable` for our `Object` enum (in [part 1]({% post_url
2025-05-06-polymorphism_with_rust1 %}) of this series) we could even pass that
in to our function. Thus it is not necessary to pick between the enum pattern I
described previously and generics, we can use them both at the same time.

## Polymorphic lists

What about our `Scene` struct? Well generics can be applied to structs as
follows:

```rust
struct Scene<H: Hittable> {
    pub objects: Vec<H>,
}
```

> **Aside**<br />
> `Vec<T>` is in fact already a generic struct.

Then you might think we could do the following:

```rust
let _ = Scene {
    objects: vec![
        Sphere {
            center: (0.0, 0.0, 0.0),
            radius: 0.0,
        },
        // Adding a Triangle is a compiler error
        Triangle {
            point1: (0.0, 0.0, 0.0),
            point2: (0.0, 0.0, 0.0),
            point3: (0.0, 0.0, 0.0),
        },
    ],
};
```

However this fails with a compiler error of 'mismatched types' when we try to
add the `Triangle`.

It may not be immediately obvious why this is a problem so let's dig a little
deeper. When we create an instance of `Scene<H>` the compiler assigns a concrete
value to `H`. In this case the value of `H` is inferred from the first type that
we add to our vector, so `H = Sphere`. When we try to add a second element the
compiler is expecting to find another `Sphere` but instead it finds a `Triangle`
and raises the 'mismatched types' error.

In other words our `Scene<H>` can have a concrete value of `Scene<Sphere>` (and
only contain spheres) or `Scene<Triangle>` (and only contain triangles), it
cannot be both and contain both spheres and triangles.

More generally speaking this is because `Vec` needs to know the size and type of
the values it will be storing at compile time and so it can only store values of
a single type that has a known size. This is not unique to `Vec` and applies to
most other container types in Rust.

> **Aside**<br />
> Enums are able to get around this because they are in fact a sized type. They
> simply take the size of their largest variant.

<!-- TODO: Link to part 3 -->
So generics will not work for our `Scene` struct, at least not on their own. As
we will see in part 3 this is possible with trait objects and we could provide a
trait object as the concrete type, such as `Scene<Box<dyn Trait>>`.

## Advanced use cases

### Multiple traits

What about our case from before where `get_ray_color` also requires our input
type to implement `Clone`. This functionality is built into generics so we can
simply do this:

```rust
fn get_ray_color<H: Hittable + Clone>(ray: &Ray, object: &H) -> Color {
    // ...
}
```

This generalizes to any additional traits.

### Underlying type

<!-- TODO: Add link to part 3 -->
As for accessing the underlying type... well that's a little more complicated. I
will show you what works here but I'm not going to explain it in detail as I
will be going over this in part 3 when we discuss trait objects, and it should
make more sense once we understand what `dyn Trait` is.

Imagine we need to do something special to triangles at the start of our
`get_ray_color` function. This can be achieved by using the `Any` trait from the
standard library as follows:

```rust
use std::any::Any;

fn get_ray_color<H: Hittable + 'static>(ray: &Ray, object: &H) -> Color {
    // Upcast H to dyn Any
    let my_any: &dyn Any = object;

    // Try to downcast dyn Any to Triangle
    if let Option::Some(_triangle) = my_any.downcast_ref::<Triangle>() {
        todo!(); // Do something special here
    }
    // ...
}
```

Note that for this to work we need to impose an additional constraint of
`'static` on `H`. This is a constraint of upcasting to the `Any` trait.

### Nested type

As we saw earlier creating a version of our `Scene` struct that can store
multiple different types of objects using generics is not possible so this has
limited usefulness. However it is possible to create a `Scene` that contains
other scenes so long as they all have the same generic type. This can be done as
follows:

```rust
pub fn make_scene() -> Scene<Scene<Triangle>> {
    Scene {
        objects: vec![
            Scene {
                objects: vec![Triangle {
                    point1: (0.0, 0.0, 0.0),
                    point2: (0.0, 0.0, 0.0),
                    point3: (0.0, 0.0, 0.0),
                }],
            },
            // ...
        ],
    }
}
```

## Advantages & disadvantages

I think the main advantage of generics is that they are very flexible and you
can define complex type constraints without a lot of effort. They really
encapsulate the spirit if polymorphism in the sense of "I don't care what type
this is, just so long as it does X".

Generics also don't incur any performance costs.

The main disadvantage of course is that it does not work in all situations, like
we saw with our `Scene` type.

Another minor disadvantage is that they can increase your binary size. This is
because the Rust compiler monomorphizes generics at compile time. In most cases
though this is probably not a concern.

## `impl Trait`

`impl Trait` is another feature of Rust that is very similar to generics, also
using compile time monomorphization. It can be used to represent a type that
implements a given trait much like generics can. It can only be used for
function parameters and return types however. Here is an example with our
`get_ray_color` function:

```rust
fn get_ray_color(ray: &Ray, object: &impl Hittable) -> Color {
    // ...
}
```

The concrete type is inferred by the compiler, and unlike generics you cannot
use the turbofish syntax to specify a particular type.

The main use case of `impl Trait` is to return types that cannot be named like
closures (`impl Fn(T)`) and iterators (`impl Iterator<Item = T>`).

<!-- TODO: Add link -->
**Next post:** [Part 3: Trait objects]
