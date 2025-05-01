---
layout: page
title: "Polymorphism with Rust, part 1: Enums"
tags: [rust, polymorphism, inheritance, enums]
---


## Enums

Enums (or more formally enumerations) are a way of representing a set of
possible states. In most programming languages this is achieved by constraining
an integer to certain set of values and assigning a meaning to each of those
values. Rust however super charges enums by adding the ability to store data
along with state.

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

### Polymorphism with enums

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
// Create and instance of a Sphere
let some_sphere = Sphere {
    center: (0.0, 0.0, 0.0),
    radius: 1.0,
};
// Store that sphere in an instance of Object
let object_sphere = Object::Sphere(some_sphere);

// Similarly for a triangle
let some_triangle = Triangle {
    point1: (0.0, 0.0, 0.0),
    point2: (1.0, 0.0, 0.0),
    point3: (0.0, 1.0, 0.0),
};
let object_triangle = Object::Triangle(some_triangle);
```

### `get_ray_color` with enums

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
`Sphere` or a `Triangle`, hence we can pass either into our function as we
originally wanted.

#### `impl Trait for Enum`

However matching on `object` to determine which type it represents and then
calling `does_hit` is a little messy and this could get annoying if we were
doing it in several different functions. This is easily resolved by simply
implementing `Hittable` on `Object`.

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

We then only have to do this in one place and our function can be simplified to;

```rust
fn get_ray_color(ray: &Ray, object: &Object) -> Color {
    let does_hit = object.does_hit(ray);
    // ...
}
```

Implementing `Hittable` for `Object` also has the advantage of allowing us to
pass `Object` to generics that accept `Hittable`. More on this when we talk
about generics in the next section.

### `Scene` with enums

Creating our scene type with our `Object` enum is very straight forward;

```rust
struct Scene {
    pub objects: Vec<Object>,
}

// Create a scene with a sphere and triangle
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

This allows us to create a scene that contains both spheres and triangles.

### Advanced use cases

One of the major benefits of using enums for this purpose is that it works for
almost all use cases. For example imagine we wanted to implement `Clone` for
`Object`. Since both `Sphere` and `Triangle` implement `Clone` this is as simple
as deriving it on `Object`.

```rust
#[derive(Clone)]
enum Object {
    // ...
}
```

Now our enum implements `Hittable` and `Clone`. In general any time all of our
underlying types implement a trait, that trait can be implemented on the enum
without much difficulty.

What if we have a special case that we need to handle for one of our underlying
types? That's no problem either as we can simply match on our enum and handle
all cases individually (as we saw in the original version of our `get_ray_color`
function).

### Advantages & disadvantages

Personally I think enums are a great way of implementing polymorphism in Rust
and feel like the most idiomatic way of achieving this behaviour.

- It works for almost all cases
- It is easy to retrieve the underlying type by matching on the enum
- There are no performance penalties for using them
- They do not affect your binary size

It does have it's limitations however. If you have a large number of types
implementing your trait it can be annoying to have to update the enum and it's
trait implementation every time you create a new one. If you are supplying a
trait as part of a library and expecting users to implement it on their own
types then it is in fact impossible to keep your enum up to date. The next two
solutions we look at deal with these issues.

> **Aside**<br />
> There are some crates that can help with the tediousness of implementing this
> enum pattern. [enumtrait](https://docs.rs/enumtrait/latest/enumtrait/) and
> [enum_dispatch](https://docs.rs/enum_dispatch/latest/enum_dispatch/) are two
> examples.
