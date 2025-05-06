---
layout: post
title: "Polymorphism with Rust, part 0: Introduction"
tags: [rust, polymorphism, inheritance]
---

This is the first post in a four part series on polymorphism with Rust. In this
series, I discuss various methods for implementing polymorphism in Rust with a
particular focus on inheritance like behaviour. I look at the advantages and
disadvantages of each method, and discuss some more complicated use cases that
can cause problems in certain situations.

In this post I discuss what polymorphism is, why we want to implement it in
Rust, and set up a motivating example that I will use for the rest of the
series.

This series assumes a basic understanding of Rust.

<div class="table-of-contents" markdown=1>
**Contents**

* Do not remove this line (it will not be displayed)
{:toc}

**This series**

<!-- TODO: Add post URLs -->
- [**Part 0: Introduction**](/)
- [Part 1: Enums](/)
- [Part 2: Generics](/)
- [Part 3: Trait objects](/)
</div>

## What is polymorphism?

First of all, let's talk about what we mean by polymorphism in computer
programming. Polymorphism, or at least how I'm going to treat it for these blog
posts, is the ability to substitute different types so long as they all have
some shared functionality. In other words, the ability to create a type
signature that accepts multiple types so long as all of those types have some
shared functionality.

For example, consider the following function in Rust:

```rust
fn do_something(x: T) {
    x.something();
}
```

We would like a type signature for `T` that accepts any type that has the
`something` method. This is a particular flavour of polymorphism that is often
associated with interfaces or inheritance in other languages. There are other
aspects to polymorphism, but this is what I will be focusing on in this series.

### OOP and inheritance

Object oriented programming (OOP) languages, such as C++, usually achieve this
type of polymorphism through inheritance. In this case, any child class that
inherits from a parent class can be used in place of that parent class. Below is
an example of this in C++.

```c++
class Pet
{
public:
    virtual string speak() const = 0;
};

class Cat : public Pet
{
public:
    string speak() const override
    {
        return "Meow!";
    }
};

class Dog : public Pet
{
public:
    string speak() const override
    {
        return "Woof!";
    }
};

void printSpeak(Pet &pet)
{
    cout << pet.speak();
}
```

In this example, we have a parent class called `Pet`, two child classes `Cat`
and `Dog`, and a function `printSpeak` that takes a reference to an instance of
`Pet`. In this case we could pass an instance of `Cat` or `Dog` to the
`printSpeak` function as they both inherit from `Pet`, as can be seen below.

```c++
void main()
{
    Cat myCat;
    Dog myDog;

    letsHear(myCat);
    letsHear(myDog);
}
```

Rust does not have inheritance so we will be exploring other methods for how we
can implement this behaviour in Rust, specifically enums, generics, and trait
objects (`dyn Trait`).

## Why is polymorphism useful?

Polymorphism is extremely useful for reducing the amount of code you need to
write. Instead of writing five nearly identical functions for five different
types that have some shared functionality, you can just write one and the
compiler will fill in the gaps.

I think it is particularly interesting to look at how we can get
inheritance-like behaviour in Rust. It is not uncommon to rewrite programmes
from OOP languages in Rust, and in trying to match the behaviour of the original
code like for like, this challenge often comes up. This was how I originally got
interested in this topic as we will discuss below.

## Motivating example

A while back I found myself writing a ray tracing engine in Rust. Ray tracing is
a method for generating computer graphics. It involves sending out "light rays"
from a camera and bouncing them off objects in the scene.

The book I was following was written in C++ so I regularly came across this
problem of converting classic OOP inheritance into Rust. The motivating example
and challenges with various implementations I discuss here are based on this
experience.

To start off our example we're going to setup two type aliases and a struct:

```rust
type Vec3 = (f32, f32, f32);

type Color = (f32, f32, f32);

struct Ray {
    pub origin: Vec3,
    pub direction: Vec3,
}
```

Our type aliases `Vec3` and `Color` represent a 3D vector and an RGB triple
respectively. Our struct `Ray` represents a light ray that we will fire into our
scene.

The book then goes on to create a parent class that all hittable objects will
inherit from. The obvious analogy to this in Rust is a trait:

```rust
trait Hittable {
    fn does_hit(&self, ray: &Ray) -> bool;
}
```

Our trait has one method `does_hit` that tests if the given ray hits our object,
returning a boolean.

We then have two structs that implement this trait:

```rust
#[derive(Clone)]
struct Sphere {
    pub center: Vec3,
    pub radius: f32,
}

impl Hittable for Sphere {
    fn does_hit(&self, ray: &Ray) -> bool {
        // ...
    }
}

#[derive(Clone)]
struct Triangle {
    pub point1: Vec3,
    pub point2: Vec3,
    pub point3: Vec3,
}

impl Hittable for Triangle {
    fn does_hit(&self, _ray: &Ray) -> bool {
        // ...
    }
}
```

I have omitted the actual implementations of the `does_hit` method as that is
beyond the scope of this article and not very important. Note that we are
deriving the `Clone` trait for both of these structs, this will be important
later.

Finally we have two constructs that interact with these hittable objects:

```rust
fn get_ray_color(ray: &Ray, object: &T) -> Color {
    // ...
}
```

A function `get_ray_color` that takes references to a ray and some hittable
object and returns the color of that ray based on whether or not it hits the
object. This will serve as an example of a polymorphic function.

```rust
struct Scene {
    pub objects: Vec<T>,
}
```

And a struct `Scene` that will hold some list of hittable objects in our scene.
This will serve as an example of a polymorphic list. Importantly we want to
store both spheres and triangles in the same scene.

Right now this code will not compile because the `T` placeholder we have for a
hittable object is not valid. The question we need to answer is: what type can
we substitute for `T` that will allow us to pass any type that implements the
`Hittable` trait?

Naively we might try the following:

```rust
fn get_ray_color(ray: &Ray, object: &Hittable) -> Color {
    // ...
}

struct Scene {
    pub objects: Vec<Hittable>,
}
```

But again, this does not compile; the Rust compiler warning us that a trait
cannot be used in place of a type.

## Advanced use cases

In addition to the above, we are also going to have a look at three more
advanced use cases. First we will look at implementing multiple traits, in
particular we will be looking at taking an input type that implements both
`Hittable` and `Clone`. Second we will look at how to access the underlying
concrete type, imagine our `get_ray_color` function needs to handle a special
case for triangles. Finally we will look at nesting a type inside of itself, in
this case we will be looking at nesting one `Scene` inside of another.

<!-- TODO: Add link -->
**Next post:** [Part 1: Enums]
