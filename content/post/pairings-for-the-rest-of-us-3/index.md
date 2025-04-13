+++
date = '2024-08-11T10:52:24+02:00'
title = 'Pairings for the Rest of Us, part 3'
math = true
summary = "Learn how to implement elliptic curve pairings from scratch in Rust. This hands-on guide covers finite field arithmetic, elliptic curve operations, Miller's algorithm, and real code examples using TinyJubJub and MoonMath curves."
series = ['Pairings for the rest of us']
tags = ['Elliptic curves', 'Cryptography', 'Rust']
ShowShareButtons = true
ShowPostNavLinks = true
ShowToc = true
+++
## From Scratch in Rust

## Introduction
In this article, we will build on the two previous articles where we explored the theoretical aspects of elliptic curve pairings:

- [Pairings for the Rest of Us, part 1: Finding G1 and G2](/post/pairings-for-the-rest-of-us-1)
- [Pairings for the Rest of Us, part 2: The Tate Pairing](/post/pairings-for-the-rest-of-us-2)

This time, we will take a **practical approach** by writing code in Rust to implement elliptic curve pairings **from scratch**. If you’re not familiar with the basic concepts, I recommend reading the earlier articles, as they explain the background you will need for this one.

### Why Start from Scratch?
Our goal is to **avoid using any existing libraries** and implement everything from the ground up. This approach is ideal for learning complex topics like elliptic curve pairings because it forces us to deeply understand how everything works.

When I first started, I took notes on elliptic curve pairings just for myself. As I worked on the material, I realized that many developers struggle with these ideas. I wish I had found something to explain them clearly when I was learning. This is why I decided to share my journey, hoping it can help others.

### A Learning Journey, Not a Tutorial
This article is not meant to be a "zero-to-hero" tutorial on elliptic curves. While I’ll briefly mention some basics, it’s not a guide to learning elliptic curves from the beginning—that’s covered in other resources. Instead, this article is more like a **journal of my learning process**, showing my progress with real code and what I learned along the way.

### The Real Gaps Show When You Code
**Writing code is one of the best ways to learn**. You might think you understand something, but it’s only when you try to implement it in code that you discover where your understanding is incomplete. I experienced this myself when I tested my **TinyJubJub** implementation on real-world curves and got inconsistent results. After much debugging, I realized that my implementation lacked key components, like twists, which I had read about in every resource but never fully understood until I hit the issue myself.

### What to Expect
This article will walk you through the initial version, much as I first approached it. In the next part, we’ll focus on identifying the issues, go over missing elements, and work on optimizations step by step.

To keep the code simple and easy to follow, I will organize it in a flat structure. I’ll point out the important parts and explain the design choices, but I won’t go over every single line of code. The complete code is available in [this GitHub repository](https://github.com/brozorec/pairings-from-scratch) for you to explore.

Keep in mind that the **final result won’t be optimized**. Right now, our main goal is to understand how everything works. Improving performance will be something to focus on later.

### What We’ll Cover
In this article, we’ll cover these main topics:

1. Finite field arithmetic
2. Polynomials for field extensions
3. Elliptic curve operations
4. Miller's algorithm for Tate Pairings

As in the earlier articles, we will mainly work with the **TinyJubJub** curve, using Rust’s native `i16` type to keep things simple. You can also find tests for the **BLS6_6** curve (or MoonMath curve) in the repository, where we will use `i64` for field elements.

## Finite Fields and Basic Arithmetic
Let's jump straight into the code and start building a [finite field](https://en.wikipedia.org/wiki/Finite_field). As promised, we’ll keep it simple and build everything step by step, without relying on external libraries.

### 1. Defining the `FiniteField` Trait
First, we need to define a `FiniteField` trait that will represent the properties and behaviors that any finite field should have. It acts as a wrapper and includes:

- An associated type `T` that's the underlying type used to store the value of a field element. The native Rust `i16` is enough for the **TinyJubJub** curve, however, when we implement **BN254**, we'll need a type way bigger such as `BigInt`.
- Functions for read-only access, such as retrieving the modulus, zero, and one elements.
- Basic operations, such as reducing values under the modulus and computing multiplicative inverses.

Here’s the definition of the trait:

```rust
pub trait FiniteField {
    type T;
    
    fn modulus() -> Self::T;
    fn zero() -> Self::T;
    fn one() -> Self::T;

    fn reduce(value: Self::T) -> Self::T {
        (Self::modulus() + value) % Self::modulus()
    }
    
    fn inverse(value: Self::T) -> Self::T;
}

```
This is our template for what a finite field looks like. It ensures any struct that implements this trait has a modulus and can reduce elements. We’ve also set up default behavior for reducing a value under the modulus. Lat's also add a default implementation for finding the multiplicative inverse.

#### Multiplicative Inverse
The default multiplicative inverse function uses the [**Extended Euclidean algorithm**](https://en.wikipedia.org/wiki/Extended_Euclidean_algorithm), which computes the greatest common divisor and the associated coefficients such that $ax+bp=1$. When $gcd(x,p)=1$, the inverse of $x$ is $a$, because $ax ≡ 1 \mod  p$.
```rust
pub trait FiniteField {
    // ... continuing from above
    
    fn inverse(value: Self::T) -> Self::T {
        let zero = Self::zero();
        
        // Extended Eucledean algorithm
        let mut r0 = Self::modulus();
        let mut r1 = value.clone();
        let mut t0 = zero.clone();
        let mut t1 = Self::one();

        while r1 != zero {
            let quotient = r0.clone() / r1.clone();

            let r2 = r0 - quotient.clone() * r1.clone();
            r0 = r1;
            r1 = r2;

            let t2 = t0 - quotient * t1.clone();
            t0 = t1;
            t1 = t2;
        }

        Self::reduce(t0)
    }
}
```
> Keep in mind that when we start dealing with extensions fields in the next section, where the associated type `T` will be a `Polynomial`, we'll need a slightly different `inverse()`.

#### Creating a Finite Field
Now that we’ve defined our `FiniteField` trait, let’s implement it for a specific finite field $\mathbb{F}_{13}$:
```rust
pub struct Ff13;

impl FiniteField for Ff13 {
    type T = i16;

    fn modulus() -> Self::T { 13 }

    fn one() -> Self::T { 1 }

    fn zero() -> Self::T { 0 }
}
```

### 2. Implementing the `FieldElement` Struct
Next, we’ll define a `FieldElement` struct that will represent individual elements from a concrete finite field. It is parameterized by `M`, which determines the field (i.e., the modulus) the element comes from. This way, we can have elements for different fields while reusing the same code structure.

```rust
pub struct FieldElement<M: FiniteField>(M::T);

impl<M: FiniteField> FieldElement<M> {
    pub fn new(value: M::T) -> Self {
        FieldElement(M::reduce(value))
    }
    
    pub fn inverse(&self) -> Self {
        Self::new(M::inverse(&self.0))
    }
    
    // more functions...
}
```

This struct encapsulates the value of an element and automatically reduces any input under the modulus when creating a new instance. 

Now, let’s create some elements in $\mathbb{F}_{13}$:

```rust
let three = FieldElement::<Ff13>::new(3);
let nine = three.inverse();
```

To simplify the code, you can also create a type alias:
```rust
type Fe13 = FieldElement<Ff13>

let three = Fe13::new(3);
```


### 3. Operator Overloading
In Rust, we can use operator overloading to define custom behavior for operators such as `+`, `-`, `*`, and `/` when applied to `FieldElement`. This allows us to use normal arithmetic symbols to manipulate field elements.

Here’s an example of how to overload the division `/` operator:

```rust
use std::ops::Div;

// Division is equivalent to multiplying the left-hand side
// by the multiplicative inverse of the right-hand side.
impl<M: FiniteField> Div for FieldElement<M> {
    type Output = Self;

    fn div(self, rhs: Self) -> Self {
        self * rhs.inverse()
    }
}
```

Now you can divide field elements just like regular numbers:
```rust
let a = Fe13::new(7);
let b = Fe13::new(3);
let c = Fe13::new(11);
// (7 / 3) % 13 = 7 * 9 % 13 = 11
assert_eq!(a / b, c);
```

The implementations of all other operators can be found [here](https://github.com/brozorec/pairings-from-scratch/blob/be402d819e9cf9330da6b2ef28cb0391dc496851/src/field_element.rs#L76).

#### Exponentiation
Raising field elements to a power is computed using the [Square-and-Multiply algorithm](https://en.wikipedia.org/wiki/Exponentiation_by_squaring), which iterates over the bits of the exponent, squaring an accumulator in each iteration and multiplying by the base when a bit is set.

```rust
pub fn pow<S: NonExtendedField>(&self, exp: S::T) -> Self {
    if exp == S::zero() {
        return Self::one();
    }

    let mut acc = Self::one();

    for bit in S::to_bits(exp).iter() {
        acc = acc.clone() * acc.clone();

        if *bit {
            acc = acc * self.clone();
        }

    }

    acc
}
```
You might notice that the exponent must implement `NonExtenedField`, but more about this new trait in the next section.

## Extension Fields and Polynomials
So far, we’ve introduced the building blocks for working with finite fields in Rust: the `FiniteField` trait defining the essential properties of a field and the`FieldElement` struct providing an abstraction for individual elements from the field. Now, we’ll move forward and explore **Extension Fields**. 

### 1. Extension Fields
As we discussed in [Pairings for the Rest of Us, part 1: Finding G1 and G2](/post/pairings-for-the-rest-of-us-1#field-extensions) , an extension field is a way to expand a base field to include additional elements, creating a larger field with more complex structures. For example, if we start with the field $\mathbb{F}_{13}$ (which contains elements $\{0, 1, ..., 12\}$), we can create a quadratic extension $\mathbb{F}_{13^2}$ that contains 169 elements. All those new elements are represented as pairs $(a, b)$, where $a$ and $b$ are elements from $\mathbb{F}_{13}$.

#### Representing Extension Fields with Polynomials
Extension fields can be elegantly represented using polynomials.  Let's revisit the example from [Part 1](/post/pairings-for-the-rest-of-us-1#field-extensions). In $\mathbb{F}_{13^2}$, we can represent each element as a degree-1 polynomial:

$(4, 7)$ becomes $4 + 7x$
$(0, 5)$ becomes $0 + 5x$

The modulus for this extension field is chosen as an irreducible polynomial over the base field. In our case, that's $x^2 + 2 = 0$ in $\mathbb{F}_{13}$.

#### Polynomial Implementation
To handle elements in extension fields, we introduce the `Polynomial` struct. This struct lets us treat polynomials as first-class citizens in our code, making it easy to perform arithmetic operations on them within those fields.

```rust
pub struct Polynomial<C>(Vec<C>);

impl<C> Polynomial<C> {

    pub fn new(coeffs: Vec<C>) -> Self {
        // Creates a new polynomial, trimming trailing zeros.
    }

    // more functions...
}
```

Here, `C` is a generic type that represents the coefficients of the polynomial. In our case, **these coefficients will be elements of the base field**.

The `Polynomial` struct provides several key methods:

- **Degree Calculation**: The degree of a polynomial is determined by the highest power with a non-zero coefficient, ignoring trailing zeros. It's designed to return the index of the last non-zero coefficient, ensuring that trailing zeros don't affect the polynomial's degree.

```rust
pub fn degree(&self) -> usize {
    self.coefficients()
        .iter().rposition(|coeff| coeff != &C::default()).unwrap_or(0)
}
```

- **Division and Modulo**: The `div_mod` function implements [polynomial long division](https://en.wikipedia.org/wiki/Polynomial_long_division), returning both the quotient and remainder. It works by iteratively subtracting multiples of the divisor from the remainder until the degree of the remainder is less than the degree of the divisor.

```rust
fn div_mod(&self, divisor: &Self) -> (Self, Self) {
    let mut quotient = Polynomial::new(vec![C::default()]);
    let mut remainder = self.clone();

    while remainder.degree() >= divisor.degree()
        && *remainder.leading_coefficient() != C::default()
    {
        let degree = remainder.degree() - divisor.degree();
        let mut coeffs = vec![C::default(); degree + 1];

        let rem_coeff = remainder.leading_coefficient().clone();
        let div_coeff = divisor.leading_coefficient().clone();
        let leading_coeff = rem_coeff / div_coeff;

        coeffs[degree] = leading_coeff;
        let term = Polynomial::new(coeffs);

        quotient = quotient + term.clone();
        remainder = remainder - term * divisor.clone();
    }

    (quotient, remainder)
}
```

Similar to `FieldElement`, we use operator overloading to define arithmetic operations for polynomials. For brevity, we skip posting code snippets, but you can check how they were implemented [here](https://github.com/brozorec/pairings-from-scratch/blob/be402d819e9cf9330da6b2ef28cb0391dc496851/src/polynomial.rs#L155). These overloaded operators allow for intuitive manipulation of polynomials. For example, you can write `poly1 + poly2` to add two polynomials or `poly1 * poly2` to multiply them.

The `Polynomial` struct will be very useful when using it as the associated type `T` in the `FiniteField` trait. We can now work with fields like $\mathbb{F}_{13^2}$ and $\mathbb{F}_{13^4}$ in more natural and intuitive way.

#### Example
Let's explore how to use the `Polynomial` struct to implement a concrete extension field and perform arithmetic operations within it.

```rust {linenos=inline hl_lines=[4,"7-9"]}
pub struct Ff13_2;

impl FiniteField for Ff13_2 {
    type T = Polynomial<Ff13>;

    // x^2 + 2 is irreducible in Ff13
    fn modulus() -> Self::T {
        Polynomial::from(vec![2, 0, 1])
    }

    #[polynomial_inverse]
    fn inverse(value: &Self::T) -> Self::T;
    
    // more functions...
}
```
In this snippet, we creat a new struct `Ff13_2` (for $\mathbb{F}_{13^2}$) and make it work like a finite field by implementing the `FiniteField` trait for it. The important part is on row 4, where we say that the `type T` associated with this field is `Polynomial<Ff13>`. This means that **elements in our new field will be polynomials, and these polynomials will have coefficients from the base field**, $\mathbb{F}_{13}$.
    
Next, the `modulus` function (row 7) returns the irreducible polynomial  $x^2 + 2$, as explained [here](/post/pairings-for-the-rest-of-us-1#field-extensions).

Recall we mentioned in the previous section, we need a different way to compute inverses for polynomials compared to regular elements. The `inverse` function (row 12) is annotated with `#[polynomial_inverse]`, indicating that the polynomial inverse algorithm is provided as an [attribute macro](https://doc.rust-lang.org/reference/procedural-macros.html#attribute-macros). Instead of copying this new method every time when we create a new extension field, we will use this macro. The code for it can be found in the "derive_lib" crate [here](https://github.com/brozorec/pairings-from-scratch/blob/main/derive/src/lib.rs).

> For computing polynomial inverses we use almost the same algorithm as for regular integers, that's the **Extended Euclidean algorithm**. The only difference is at the last step where we multiply the final result by the inverse of the `r0` leading coefficient: `t0 * r0.leading_coefficient().inverse()`.


With this setup, we can perform arithmetic operations in $\mathbb{F}_{13^2}$ using the operator overloading provided by the `Polynomial` struct. For example, we can multiply two elements in $\mathbb{F}_{13^2}$ like this:
```rust
let a: Fe13_2 = Polynomial::from(vec![7, 3]).into();
let b: Fe13_2 = Polynomial::from(vec![5, 6]).into();

// (7 + 3x) * (5 + 6x) % (x^2 + 2) = (12 + 5x)
assert_eq!(a * b, Polynomial::from(vec![12, 5]).into());
```

### 2. The `NonExtendedField` Trait
You might notice that we have a `NonExtendedField` trait in addition to the `FiniteField` trait. Remember we encountered it already when we were discussing [exponentiation](#Exponentiation).

```rust
pub trait NonExtendedField:
    FiniteField<T: Shr<usize, Output = Self::T> + BitAnd + Pow<usize, Output = Self::T>>
{
    fn to_bits(s: Self::T) -> Vec<bool>;

    fn to_uint(s: Self::T) -> Option<usize>;

    fn from_uint(s: usize) -> Option<Self::T>;
}
```

The `NonExtendedField` trait is used to differentiate between non-extended fields and extension fields. In the context of finite fields, a non-extended field is typically a field with elements that can be directly represented by integers.

This trait provides **additional functionality** specific to non-extended fields, **which may not be applicable** or efficiently implementable for extension fields. This includes methods for converting field elements to and from unsigned integers or to their bit representations.

`NonExtendedField` helps enforce **type safety** by ensuring that operations specific to non-extended fields are only available for appropriate field types, as we will see later when we deal with elliptic curves.

Our approach to designing finite fields might seem a bit odd at first glance. You might think, *"Why not just have a basic `Field` trait and then two separate traits for extended and non-extended fields?"*. We've set it up this way because **non-extended fields are actually just a special case of extended fields**. For instance, take the number 3 in the field $\mathbb{F}_{13}$ (that's our non-extended field). Now, if we look at this same element in $\mathbb{F}_{13^2}$ (an extended field), we can write it as $(3, 0)$ or $3 + 0x$ if we're using polynomial form.

The struct `Ff13` defines our non-extened field $\mathbb{F}_{13}$ so it has to implement the `NonExtendedField` as well:
```rust
impl NonExtendedField for Ff13 {
    // ...
}
```
Our design reflects the real-world relationship between these different types of fields. We can treat non-extended fields as a subset of extended fields, which they really are, mathematically speaking. It might take a bit to wrap your head around, but it actually makes our code more true to the math it represents.

## EC Points and Pairings
We’ve come this far, and now we’ve reached the heart of it — **elliptic curve pairings**. Everything up to this point has been building towards this. Using the **TinyJubJub** curve, we’ll finally see how pairings work.

We start by defining the `EllipticCurve` trait:
```rust
pub trait EllipticCurve {
    type BaseField: FiniteField;
    type ScalarField: NonExtendedField;

    fn a() -> FieldElement<Self::BaseField>;
    fn b() -> FieldElement<Self::BaseField>;

    fn embedding_degree() -> usize;

    fn order() -> <Self::ScalarField as FiniteField>::T;

    fn r() -> <Self::ScalarField as FiniteField>::T;
}
```
This trait encapsulates the essential properties of an elliptic curve:
- `a` and `b` are the coefficients of the curve equation $y^2 = x^3 + ax + b$,
- `r` returns the largest cofactor of the order of the non-extended curve,
- `embedding_degree` is the smallest positive integer $k$ such that $r$ divides $q^k - 1$ without remainder,
- `order` returns the number of points on the extended curve.

For TinyJubJub, we implement this trait as follows:
```rust
impl EllipticCurve for TinyJJ {
    type BaseField = Ff13_4;
    type ScalarField = Ff13;

    fn a() -> FieldElement<Self::BaseField> {
        Polynomial::from(vec![8]).into()
    }

    fn b() -> FieldElement<Self::BaseField> {
        Polynomial::from(vec![8]).into()
    }

    fn embedding_degree() -> usize { 4 }

    fn order() -> <Self::ScalarField as FiniteField>::T { 28800 }

    // the order of the non-extened curve is 20 whose factors are 5 * 2 * 2
    // r is the biggest factor
    fn r() -> <Self::ScalarField as FiniteField>::T { 5 }
}
```
#### Base Field vs. Scalar Field
In elliptic curve cryptography, we work with two distinct fields.
1. The base field (`Ff13_4` for TinyJubJub): all coordinates of points and coefficients of the curve equation are elements of the this field.
2. The scalar field (`Ff13` for TinyJubJub) is used for scalar multiplication on the curve points. It's typically smaller than the base field and is related to the curve's order.

### 1. The `AffinePoint` Enum
We represent points on the curve using the `AffinePoint` enum:
```rust
pub enum AffinePoint<E: EllipticCurve> {
    Infinity,
    XY(FieldElement<E::BaseField>, FieldElement<E::BaseField>),
}
```

#### Why Use an Enum?
1. It allows us to define the point at infinity explicitly.
2. It forces us to handle the infinity case in all our operations, preventing potential bugs.
2. Affine points are more intuitive to work with compared to projective points, which use three coordinates $(x, y, z)$.
---
We implement operator overloading for `AffinePoint`, similar to what we did for `FieldElement` and `Polynomial`.

#### Points Addition
[Point addition](/post/pairings-for-the-rest-of-us-2#ec-points-addition-and-doubling) on elliptic curves follows specific geometric rules. For two points $P=(x_1,y_1)$ and $Q=(x_2,y_2)$, the resulting point depends on whether the points are distinct, identical, or involve the point at infinity.
```rust
impl<E: EllipticCurve> Add for AffinePoint<E> {
    type Output = Self;

    fn add(self, other: Self) -> Self::Output {
        if self == other {
            return self.double();
        }

        if self == -other.clone() {
            return Self::Infinity;
        }

        match (self.clone(), other) {
            (AffinePoint::Infinity, p) | (p, AffinePoint::Infinity) => p,
            (AffinePoint::XY(x1, y1), AffinePoint::XY(x2, y2)) => {
                // implementation here
                // slope = (y2 - y1) / (x2 - x1)
            }
        }
    }
}
```
#### Multiplication by Scalar
For multiplying a point by a scalar, we use the Double-and-Add algorithm, which we discussed in [Pairings for the Rest of Us, part 2: The Tate Pairing](/post/pairings-for-the-rest-of-us-2#multiplication-by-scalar-double-and-add-algorithm) of this series. Check the implementation [here](https://github.com/brozorec/pairings-from-scratch/blob/be402d819e9cf9330da6b2ef28cb0391dc496851/src/elliptic_curve.rs#L145).

#### Point Doubling
An interesting observation about point doubling:
```rust
pub fn double(&self) -> Self {
    match self {
        AffinePoint::XY(x, y) => {
            // the doubled point will be infinity if and only if y = 0
            if y.is_zero() {
                return Self::Infinity;
            }

            // implementation here
            // slope = (3*x^2 + a) / 2*y
        }
        _ => AffinePoint::Infinity,
    }
}
```
We can quickly check if doubling an elliptic curve point would result in the point at infinity. For an elliptic curve point $P = (x, y)$, the doubled point $2P$ will be the point at infinity if and only if $y = 0$. This is because the formula for calculating the slope of the tangent line (which is used in point doubling) involves dividing by $y$. If $y = 0$, this division is undefined, which geometrically corresponds to a vertical tangent line that intersects the curve at infinity.

#### Trace Map
Another important topic we introduced in [Part 1](/post/pairings-for-the-rest-of-us-1#frobenius-endomorphism-and-trace-map) was the Trace map, which helped us to filter out points and identify G1 and G2, the two subgroups crucial for pairing operations. 

The implementation calculates the trace of a point by applying the Frobenius endomorphism repeatedly and summing the results. This operation is used to check whether points are in the correct subgroups before performing all pairing operations.
```rust
pub fn trace_map(&self) -> Self {
    match self {
        AffinePoint::XY(x, y) => {
            let mut point = AffinePoint::XY(x.clone(), y.clone());
            for i in 1..E::embedding_degree() {
                let power = E::ScalarField::modulus().pow(i);
                let new_x = x.pow::<E::ScalarField>(power.clone());
                let new_y = y.pow::<E::ScalarField>(power);
                point = point + AffinePoint::new_xy(new_x, new_y);
            }
            point
        }
        _ => AffinePoint::Infinity,
    }
}
```

### 2. The Tate Pairing
The Tate pairing is a bilinear map that **takes two points from different subgroups, G1 and G2, of an elliptic curve and produces an element in the extension field**. It consists of two main parts: the Miller loop and the final exponentiation.

#### The Miller Loop
The core of the Tate pairing is the Miller loop, which iteratively computes the distance relationship between the lines drawn by the **Double-and-Add algorithm** and the second input point `q`. As we discussed in [Part 2](/post/pairings-for-the-rest-of-us-2#miller%E2%80%99s-loop):

*"We will take $P$ and apply the "Double-and-Add" algorithm to multiply it by $r$. Every loop step draws new lines between an accumulator point and the point $P$, so the job is to compute the distance relationship between each of those lines and $Q$. The resulting values will be from $F_{13^4}$, and the multiplication of all the $F_{13^4}$ values will be the output of the Miller's loop."*


```rust
fn miller_loop(p: &AffinePoint<Self>, q: &AffinePoint<Self>) -> 
    FieldElement<Self::BaseField>
{
    let mut point = p.clone();
    let mut f = FieldElement::<Self::BaseField>::one();

    let bits = Self::ScalarField::to_bits(Self::r());
    for bit in bits.iter().skip(1) {
        let f_new = dist_relationship(&point, &point, &q);
        f = f.clone() * f * f_new;
        point = point.clone().double();

        if *bit {
            let f_new = dist_relationship(&point, &p, &q);
            f = f * f_new;
            point = point + p.clone()
        }
    }

    assert!(point.is_inf());
    f
}
```
 1. The code above carries out an implicit multiplication of `p` by `r`, using the Double-and-Add algorithm for multiplying a point by a scalar. The result of this multiplication is the point at infinity, as `p` is of order `r`.
2. At each step in the process, a value from the base field `f` is calculated from a distance relationship between the current line and the point `q`.
3. This value is multiplicatively accumulated, and its ﬁnal value is the output of Miller’s loop.

#### Final Exponentiation
The output from the Miller loop is not a point but a value from the extension field. However, **the output from the pairing is required to have the same order as the input points**. To ensure that the pairing result has the correct order, we must raise $f$ to some power to bring down its order to $r$. It turns out that if we raise $f$ by the power of $(q^k - 1 ) / r$, we'll get a value from the field of order $r$.
```rust
fn final_exponentiation(f: FieldElement<Self::BaseField>) ->     
    FieldElement<Self::BaseField>
{
    let k = Self::embedding_degree();
    let q = Self::ScalarField::modulus();
    let one = Self::ScalarField::one();
    let f_exp = (q.pow(k) - one) / Self::r();

    f.pow::<Self::ScalarField>(f_exp)
}
```


### Example: Pairings in Action
Now that we’ve covered some theory and walked through the code, let’s see an actual example of pairing two points on the `TinyJubJub` curve.

In our `main.rs`, we’ll define two affine points, `p` and `q`, and then compute their pairing using the Miller loop and final exponentiation. Here’s the example code:
```rust
let p = AffinePoint::<TinyJJ>::new_xy(
    Polynomial::from(vec![8]).into(),
    Polynomial::from(vec![8]).into(),
);
let q = AffinePoint::<TinyJJ>::new_xy(
    Polynomial::from(vec![7, 0, 4]).into(),
    Polynomial::from(vec![0, 10, 0, 5]).into(),
);
let result: Fe13_4 = p.pairing(&q);
```
In this example, `p` and `q` are affine points defined over the `TinyJubJub` curve, with their respective $x$ and $y$ coordinates represented as polynomials. The result of the pairing, stored in `result`, is an element of the finite field $F_{13^4}$.

Now, let's execute this with logging enabled to observe the detailed steps of the Miller loop and the pairing process. Run the following command:
```bash!
LOG_MODE=true cargo run --release
```

The output will provide a breakdown of each step in the Miller loop:

```bash
bit   | operation  | distance relationship     | accumulated distance      | accumulator point
----- | ---------- | ------------------------- | ------------------------- | ---------------
0     | double     | 2 + 3*x + 11*x^2 + 8*x^3  | 2 + 3*x + 11*x^2 + 8*x^3  | (7, 11)
1     | double     | 11 + 3*x + 1*x^2 + 8*x^3  | 2 + 6*x + 10*x^2 + 12*x^3 | (8, 5)
      | add        | 12 + 4*x^2                | 9 + 2*x + 11*x^2 + 12*x^3 | Infinity 
----- | ---------- | ------------------------- | ------------------------- | ---------------
Output from Miller's loop: 9 + 2*x + 11*x^2 + 12*x^3
Output from pairing p and q: 3 + 7*x + 7*x^2 + 6*x^3
```
The pairing result `3 + 7*x + 7*x^2 + 6*x^3` matches what we expect from Part 2 of the series, confirming that our implementation works as intended.

Now that you’ve seen how pairing works on the **TinyJubJub** curve, you can take it a step further by experimenting with the **MoonMath** (also known as **BLS6_6**) curve, which is already defined in the codebase. By swapping out **TinyJubJub** for **MoonMath**, you can observe how the pairing operations differ when applied to a more complex curve. Simply modify the curve type and rerun the program to see the Miller loop and pairing steps for **BLS6_6**. This will give you valuable insight into how pairings operate on larger curves, allowing you to compare results and deepen your understanding of elliptic curve pairings.

## Conclusion
In this article, we’ve gone through a practical, hands-on implementation of elliptic curve pairings in Rust, using the **TinyJubJub** curve. We covered everything from point addition and multiplication to running the Miller loop and getting real outputs. By following the step-by-step process, we connected theory with actual code and got a better understanding of how pairings work.

That said, this implementation is far from optimized. It works well with small curves like **TinyJubJub**, but if you try it on bigger ones like **BN254**, you’ll run into issues. This is something we’ll tackle in the next part, where we’ll focus on addressing these limitations and improving performance.

There’s plenty more to explore, so stay tuned as we continue refining and extending our code!

---
Feel free to share any feedback, corrections, or suggestions to help improve the content and make it clearer for everyone. Thanks for following along, and don’t hesitate to reach out to me on X at [@BoyanBarakov](https://x.com/BoyanBarakov)
