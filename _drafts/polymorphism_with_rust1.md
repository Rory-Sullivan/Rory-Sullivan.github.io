---
layout: post
title: "Polymorphism with Rust, part 1: Enums"
tags: [rust, polymorphism, inheritance, enums]
---

This is the second post in a four part series on polymorphism with Rust. In this
post I discuss how enums can be used to implement polymorphism in Rust.

<div class="table-of-contents" markdown=1>
**Contents**

* Do not remove this line (it will not be displayed)
{:toc}

**This series**

<!-- TODO: Add post URLs -->
- [Part 0: Introduction](/)
- [**Part 1: Enums**](/)
- [Part 2: Generics](/)
- [Part 3: Trait objects](/)
</div>

## What are enums?

Enums (or more formally enumerations) are a way of representing a set of
possible states. In most programming languages this is achieved by constraining
an integer to a certain set of values, and assigning a meaning to each value.
Rust however super-charges enums by adding the ability to store data along with
state.

A great example of this is the `Option` enum. It has two possible states, `Some`
and `None`, and the `Some` state has the ability to store a value. In the
example below the `Option<i32>` returned by the function can store an `i32`
value.

```rust
fn get_even(i: i32) -> Option<i32> {
    match i % 2 == 0 {
        true => Some(i),
        false => None,
    }
}
```

You can read more about enums in Rust [here](https://doc.rust-lang.org/book/ch06-00-enums.html).

## Polymorphism with enums

This gives us a very straight forward way of grouping together types that have
shared functionality. Let's create an `Object` enum that represents all of our
`Hittable` objects.

```rust
enum Object {
    Sphere(Sphere),
    Triangle(Triangle),
}
```

In this case our enum has just two possible states, `Sphere` and `Triangle`,
each one storing an instance of the associated type. We can instantiate these
instances as follows;

```rust
// Create an instance of a Sphere
let some_sphere = Sphere {
    center: (0.0, 0.0, 0.0),
    radius: 1.0,
};
// Store that sphere in an Object enum
let object_sphere = Object::Sphere(some_sphere);

// Similarly for a triangle
let some_triangle = Triangle {
    point1: (0.0, 0.0, 0.0),
    point2: (1.0, 0.0, 0.0),
    point3: (0.0, 1.0, 0.0),
};
let object_triangle = Object::Triangle(some_triangle);
```

## Polymorphic functions

Let's apply this new type to our `get_ray_color` function.

```rust
fn get_ray_color(ray: &Ray, object: &Object) -> Color {
    let does_hit = match object {
        Object::Sphere(sphere) => sphere.does_hit(ray),
        Object::Triangle(triangle) => triangle.does_hit(ray),
    };
    // ...
}
```

It now takes in a reference to our `Object` enum which can represent either a
`Sphere` or a `Triangle`, then matches on the enum and calls `does_hit` on the
underlying type. We can now pass either a sphere or triangle into the function
as we originally wanted.

### `impl Trait for Enum`

Matching on `object` to determine which type it represents and then calling
`does_hit` is a little messy and this could get annoying if we were doing it in
several different functions. This is easily resolved by simply implementing
`Hittable` on `Object`.

```rust
impl Hittable for Object {
    fn does_hit(&self, ray: &Ray) -> bool {
        match self {
            Object::Sphere(sphere) => sphere.does_hit(ray),
            Object::Triangle(triangle) => triangle.does_hit(ray),
        }
    }
}
```

We then only have to do this in one place and our function can be simplified to:

```rust
fn get_ray_color(ray: &Ray, object: &Object) -> Color {
    let does_hit = object.does_hit(ray);
    // ...
}
```

Implementing `Hittable` for `Object` also has the advantage of allowing us to
pass `Object` to generics that accept a `Hittable`. More on this when we talk
about generics in the next post.

## Polymorphic lists

Creating our scene type with our `Object` enum is very straight forward:

```rust
struct Scene {
    pub objects: Vec<Object>,
}
```

This allows us to create a scene that contains both spheres and triangles:

```rust
fn make_scene() -> Scene {
    Scene {
        objects: vec![
            Object::Sphere(Sphere {
                center: (0.0, 0.0, 0.0),
                radius: 0.0,
            }),
            Object::Triangle(Triangle {
                point1: (0.0, 0.0, 0.0),
                point2: (0.0, 0.0, 0.0),
                point3: (0.0, 0.0, 0.0),
            }),
        ],
    }
}
```

## Advanced use cases

One of the major benefits of using enums for this purpose is that it works for
almost all use cases, as we will see here.

### Multiple traits

Imagine `get_ray_color` needs to clone `object`. All we need to do is implement
`Clone` for `Object`. Since both `Sphere` and `Triangle` implement `Clone` this
is as simple as deriving it on `Object`.

```rust
#[derive(Clone)]
enum Object {
    // ...
}
```

Now our enum implements `Hittable` and `Clone`.

In general any time all of our underlying types implement a trait, that trait
can be implemented on the enum without much difficulty.

### Underlying type

What if we have a special case that we need to handle for one of our underlying
types? That's no problem either. We can simply match on our enum to get back the
original type (as we saw in the original version of our `get_ray_color`
function). Rust even has special `if let` syntax for just this case.

```rust
fn get_ray_color(ray: &Ray, object: &Object) -> Color {
    if let Object::Triangle(triangle) = object {
        todo!() // Handle special case here
    }
    // ...
}
```

### Nested type

For our final use case, we will consider how we would nest a `Scene` instance
inside of another `Scene`. First we should implement `Hittable` for `Scene`:

```rust
impl Hittable for Scene {
    fn does_hit(&self, ray: &Ray) -> bool {
        // ...
    }
}
```

Then we can add a scene case to our enum:

```rust
pub enum Object {
    // ...
    Scene(Scene),
}

impl Hittable for Object {
    fn does_hit(&self, ray: &Ray) -> bool {
        match self {
            // ...
            Object::Scene(scene) => scene.does_hit(ray),
        }
    }
}
```

And we're done! We can add a nested scene to any other scene:

```rust
pub fn make_scene() -> Scene {
    let nested_scene = Scene {
        objects: vec![
            // ...
        ],
    };

    Scene {
        objects: vec![
            // ...
            Object::Scene(nested_scene),
        ],
    }
}
```

## Advantages & disadvantages

Personally I think enums are a great way of implementing polymorphism in Rust
and feel like the most idiomatic way of achieving this behaviour.

- Its easy to implement, even for complex use cases
- There are no performance penalties
- They do not affect your binary size

It does have it's limitations however. If you have a large number of types
implementing your trait it can be annoying to have to update the enum and it's
trait implementation every time you create a new one. Additionally, if you are
supplying a trait as part of a library and expecting users to implement it on
their own types, it is in fact impossible to keep your enum up to date. The
next two solutions we look at deal with these issues.

> **Aside**<br />
> There are some crates that can help with the tediousness of implementing this
> enum pattern. [enumtrait](https://docs.rs/enumtrait/latest/enumtrait/) and
> [enum_dispatch](https://docs.rs/enum_dispatch/latest/enum_dispatch/) are two
> examples.

<!-- TODO: Add link -->
**Next post:** [Part 2: Generics]
