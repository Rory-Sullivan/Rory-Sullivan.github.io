---
layout: post
title: "Polymorphism with Rust, part 0: Introduction"
tags: [rust, polymorphism, inheritance]
---

This is the first post in a four part series on polymorphism with Rust. In this
series, I discuss various methods for implementing polymorphism in Rust. I look
at the advantages and disadvantages of each method, and discuss some more
complicated use cases that can cause problems in certain situations.

This series assumes a basic understanding of Rust.

In this post I discuss what polymorphism is, why we want to implement it in
Rust, and set up a motivating example that I will use in the rest of this
series.

<div class="table-of-contents" markdown=1>
**Contents**

* Do not remove this line (it will not be displayed)
{:toc}
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
`something` method.

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
can implement this behaviour in Rust.

## Why is polymorphism useful?

Polymorphism is extremely useful for reducing the amount of code you need to
write. Instead of writing five nearly identical functions for five different
types that have some shared functionality, you can just write one and the
compiler will fill in the gaps.

In particular, I think it is very interesting to look at the case of how we can
get inheritance-like behaviour in Rust. It is common to rewrite programmes from
object oriented languages, like C++, to Rust. This was how I originally got
interested in this topic, as we will discuss below.

## Motivating example

A while back I found myself writing a ray tracing engine in Rust. Ray tracing is
a method for generating computer graphics. It involves sending out "light" rays
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

Our type aliases `Vec3` and `Color` are both a triple of numbers representing a
3D vector and an RGB triple respectively. Our struct `Ray` represents a light
ray that we will fire into our scene.

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
deriving the `Clone` trait for both of these struct, this will be important
later.

Finally we have two constructs that interact with these hittable objects:

```rust
fn get_ray_color(ray: &Ray, object: &T) -> Color {
    // ...
}

struct Scene {
    pub objects: Vec<T>,
}
```

We have a function `get_ray_color` that takes references to a ray and some
hittable object and returns the color of that ray based on whether or not it
hits the object. We have a struct `Scene` that will hold some list of hittable
objects in our scene. Importantly we want to be able to pass both spheres and
triangles to our function and store both in our list of objects.

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

We are also going to have a look at some more advanced use cases, such as
implementing multiple traits and how to access the underlying concrete type.
These can be pain points of some of the methods we discuss.
