+++
date = '2024-07-19T07:43:47+02:00'
title = 'Pairings for the Rest of Us, part 2'
math = true
summary = "Part 2 of the series on elliptic curve pairings explores Miller’s loop and the final exponentiation in the Tate pairing. It aims to help beginners build an intuition about pairings by simplifying complex terms and illustrating the process step-by-step."
series = ['Pairings for the rest of us']
tags = ['Elliptic curves', 'Cryptography']
ShowShareButtons = true
ShowPostNavLinks = true
ShowToc = true
+++
## The Tate Pairing

Understanding elliptic curve (EC) pairings led me through many books and research papers. Each source taught me something new, but they often assumed a high level of math knowledge and introduced complex concepts. [Part 1](/post/pairings-for-the-rest-of-us-1), where we focused on finding the subgroups G1 and G2, was challenging but still manageable. However, when it came to the actual pairing calculations, I found myself struggling with terms like "function divisors" and "equivalence classes."

But don't worry. It's possible to understand what's happening under the hood without diving too deep into advanced math, of course, by sacrificing generality and precision. This article will help you build a basic intuition about the Tate pairing. We will skip some of the more complex details, like divisors, and use simple explanations and analogies instead. This approach might not be perfectly accurate, but it's how I learned the subject myself. My goal is to give you a good foundation and encourage you to explore more detailed resources later if you're interested.

In [Part 1](/post/pairings-for-the-rest-of-us-1), we explored the "Tiny JubJub" curve, defined by $y^2 = x^3 + 8x + 8 \mod 13$, and extended our base field $\mathbb{F}_{13}$ to $\mathbb{F}_{13^4}$. We learned about the Frobenius endomorphism and the Trace map, which helped us find the subgroups $\mathbb{G}_1$ and $\mathbb{G}_2$. This sets the stage for Part 2, where we will implement the Tate pairing.

However, before we dive into the details, let's quickly review some basics to make sure we're all on the same page.

### Point and Line: Distance Relationship
Every two points $(x_1, y_1)$ and $(x_2, y_2)$ on the plane can be connected by a line, and the slope of this line can be calculated with the following formula:

$$
m = \frac{y_2 - y_1}{x_2 - x_1}
$$

Now, if we have a third point $(x_t, y_t)$ and we want to check whether it lies on the same line, we substitute in $v = m \cdot (x_t - x_1) - (y_t - y_1)$ and check if $v == 0$.

Let's consider the points:

$A = (x_1, y_1) = (-1, 1)$

$B = (x_2, y_2) = (3, 2)$

$C = (x_3, y_3) = (7, 3)$

$D = (x_4, y_4) = (5, 1)$

$E = (x_5, y_5) = (2, 1)$

![Screenshot from 2024-06-11 14-01-27](https://hackmd.io/_uploads/HyXXVcTSC.png)

The slope of the line formed by points $A$ and $B$ is:

$m = \frac{2 - 1}{3 + 1} = 0.25$

To check if point $C$ lies on the line, we calculate:

$v = 0.25 \cdot (7 + 1) - (3 - 1) = 0$

Since $v = 0$, point $C$ lies on the line. However, when we plug in the coordinates of points $D$ and $E$, we get $v_D = 1.5$ and $v_E = 0.75$. Points $D$ and $E$ don't lie on the line because both have $v \neq 0$. We can see that these points are at different distances from the line, with point $E$ being closer. This leads us to conclude that $v$ represents some distance relationship between a point and a line: the bigger its value, the farther the point is from the line.

It's important to note that the value of $v$ does not represent the actual distance. The actual distance between a point and a line can be calculated using a different formula ([see Wikipedia](https://en.wikipedia.org/wiki/Distance_from_a_point_to_a_line)), which computes the distance between $D$ and the line as $\frac{6}{\sqrt{17}}$. However, we are not interested in that distance. It’s important to stress that we consider $v$ as just **an indicator of the distance relationship between a point and a line, not the Euclidean distance**.

### EC Points: Addition and Doubling

Adding two elliptic curve (EC) points, $P$ and $Q$, involves drawing a line between them. This line will intersect the curve at one more point, $R'$ $(x_t, y_t)$. After finding this point using the method described in the previous section, we flip its y-coordinate to get the new point $R$ $(x_t, -y_t)$, which is the sum of $P$ and $Q$. *(left figure)*

When we want to add a point $P$ to itself, we draw a tangent line to $P$. This tangent line intersects the curve at another point, and again, we flip its y-coordinate to get the result of $P + P$. *(right figure)*

![Screenshot from 2024-06-11 15-17-12](https://hackmd.io/_uploads/rkbNwq6S0.png)

We have already described how to compute the slope of the line between two distinct points, $P$ and $Q$, and check whether a third point lies on the line defined by $P$ and $Q$. Nevertheless, we haven’t discussed the special case of a line and its slope when it is tangent to a curve.

Let’s define a curve $E: y^2 = x^3 + ax + b$ and a point $P$ from that curve. The slope of the line tangent to the curve and passing through $P$ is given by the derivative function $\frac{dy}{dx}$ of $E$, evaluated at $P$. We won’t bother here to differentiate the equation but will directly give the formula for the slope:

$$
m = \frac{3x^2 + a}{2y}
$$

Please refer to “Pairings for Beginners” (section 2.1.2) for the derivation and in-depth discussion about those formulas.

### Multiplication by Scalar: Double-and-Add Algorithm

The "Double-and-Add" algorithm is an efficient method to compute the multiplication of an elliptic curve (EC) point $P$ by a scalar $m$. Multiplying $P$ by $m$ essentially means adding $P$ to itself $m$ times:

$$P + P + \ldots + P \quad \text{(m times)}$$

However, performing this operation in $m$ steps is inefficient, especially when dealing with scalars from large finite fields. The "Double-and-Add" algorithm optimizes this process.

We will use this instructive [example](https://www.youtube.com/watch?v=u1VRbo_fhC8) to quickly understand the algorithm. Let's consider an EC point $P$ and a scalar $m = 26$:

1. Write the binary representation of $m$: $26 = (11010)_2$
2. Loop through all the bits starting from the second most significant one
3. Initialize an accumulator $A = P$. For each bit, if it is 1, double the accumulator and add $P$ to it (i.e., $A = 2A + P$); if it is 0, just double the accumulator (i.e., $A = 2A$)

Here is the detailed step-by-step process:

| bit | operation | result       |
|-----|-----------|--------------|
| 1   | Ignore    | $A = P$      |
| 1   | Double    | $A = 2A = 2P$|
|     | Add       | $A = A + P = 3P$ |
| 0   | Double    | $A = 2A = 6P$|
| 1   | Double    | $A = 2A = 12P$|
|     | Add       | $A = A + P = 13P$ |
| 0   | Double    | $A = 2A = 26P$|

Using the "Double-and-Add" algorithm, we compute $26P$ in only 5 steps instead of 26.

---

With all the topics reviewed, we are ready to jump into our subject's nitty-gritty. The Tate pairing consists of two componenets: Miller’s loop and the final exponentiation. 

### Miller’s Loop

For a high-level overview of Miller’s loop, I’ll start with the great primer from “Computing the Tate Pairing” by Michael Scott and develop it with a concrete example.

*“The notation for the Tate pairing is $e(P, Q)$, where $P$ and $Q$ are points on the elliptic curve $E(F_{p^k})$ and are of order $r$. The Tate pairing evaluates as an element in $F_{p^k}$.
…
Basically (and very loosely), Miller’s algorithm ﬁrst carries out an implicit multiplication of $P$ by $r$, using the standard double-and-add algorithm for elliptic curve point multiplication. The result of this multiplication will (of course) be the point at inﬁnity, as $P$ is of order $r$. (…) At each step in the process, an $F_{p^k}$ value is calculated from a distance relationship between the current line and the point $Q$. This value is multiplicatively accumulated, and its ﬁnal value is the output of Miller’s loop.”*

Similarly to Part 1, we use the “TinyJubJub” curve, defined over the prime finite field $F_{13}$, to illustrate the whole process. Remember, with $r = 5$ being the largest prime factor of the curve’s order, we computed the embedding degree $k = 4$ and extended $F_{13}$ to $F_{13^4}$. Thanks to that, we found two distinct subgroups, $\mathbb{G}_1$ and $\mathbb{G}_2$. An important characteristic of those subgroups' points is that when multiplied by $r$, they all go to the infinity point. $P$ and $Q$, each respectively belonging to one or the other subgroup, will be plugged into Miller’s loop. 

As stated in the primer, we will take $P$ and apply the “Double-and-Add” algorithm to multiply it by $r$. Every loop step draws new lines between an accumulator point and the point $P$, so **the job is to compute the distance relationship** between each of those lines and $Q$. The resulting values will be from $F_{13^4}$, and the multiplication of all the $F_{13^4}$ values will be the output of the Miller’s loop.

Here's the step-by-step pairing of two points, $P$ and $Q$ from "Tiny JubJub" over $F_{13^4}$.

__Parameters and inputs__

$a = 8 \quad (\text{given by} \quad y^2 = x^3 + 8x + 8 \text{, used in slope's formula})$

$r = 5 = (101)_2$

$P = (x_P, y_P) = (8, 8)$

$Q = (x_Q, y_Q) = (4x^2 + 7, 5x^3 + 10x)$


__Initializations__

$A \quad (\text{accumulator point}) = (x_A, y_A) = (8, 8)$

$f \quad (\text{accumulated distance}) = 1$

$v \quad (\text{distance relationship}) = 1$

__Formulas for the slope $m$ according the operation__

$$
m_{add} = \frac{y_A - y_P}{x_A - x_P} \quad \text{ }
m_{double} = \frac{3{x_A}{^2} + a}{2y_A}
$$

__Steps__


Loop through all the bits of the largest prime factor $r$: $5 = (101)_2$, __ignoring the 1st bit__. Remember we operate in $\mod 13$.
1. The next bit's $0$ and the operation is "Double" leading to the slope being computed by
$$
m = \frac{3{x_A}{^2} + a}{2y_A} = \frac{3 * 8^2 + 8}{2 * 8} = 6
$$
2. The distance relationship $v$ between $A$ and $Q$ is $v = 8x^3 + 11x^2 + 3x + 2$ 
$$v = m * (x_Q - x_A) - (y_Q - y_A) = 6 * (4x^2 + 7 - 8) - (5x^3 + 10x - 8)$$
3. The distance $f$ is multiplicatevely accumulated next
$$
f = f * f * v = 1 * 1 * 8x^3 + 11x^2 + 3x + 2 = 8x^3 + 11x^2 + 3x + 2
$$
4. The accumulator point $A$ gets also doubled $A = 2A = (7, 11)$
5. The next bit's $1$ so we first double similarly to the previous steps, but also do an "Add" operation
6. We end up with $f = 12x^3 + 11x^2 + 2x + 9$ which is a value from $F_{13^4}$


| bit | operation | slope $m_{add/double}$  | distance relationship $v = m * (x_Q - x_A) - (y_Q - y_A)$ | acc. distance $f$ | acc. point $A$ |
|-----|-----------|--------------|-----------------------------|--------------------------|-------------------|
|1    | Ignore    | -            | $1$                         | $1$                      | $A = P = (8, 8)$  |
|0    | Double    | $6$          | $8x^3 + 11x^2 + 3x + 2$     | $f*f*v = 8x^3 + 11x^2 + 3x + 2$ | $A = 2A = (7, 11)$ |
|1    | Double    | $4$          | $8x^3 + x^2 + 3x + 11$      | $f*f*v = 12x^3 + 10x^2 + 6x+ 2$ | $A = 2A = (8, 5)$ |
|     | Add       | $1$          | $x_Q - x_A = 4x^2 + 12$     | $f*v = 12x^3 + 11x^2 + 2x + 9$  | $A = A + P = \mathcal{O}$ |



### Final Exponentiation

When we talk about the order of an EC point (not to be confused with a curve’s order), we mean the scalar from the base field that sends the given point to infinity when multiplied together: $P * 5 = \mathcal{O}$ in our case.

By analogy, the values from the base field also have a (multiplicative) order $m$ that sends them to the identity element (that’s 1) when they get raised to the power of $m$. For example, the multiplicative order of $4$ in $F_{13}$ is 6 because $4^6 = 1 \mod 13$.

Recall that one of the requirements for two EC points to be “pairable” is to be of the same order. This applies to the output of the pairing as well: **the resulting value must be of the same order as the EC points**.

The example above demonstrates that Miller’s loop output $f = 12x^3 + 11x^2 + 2x + 9$ is a value from the extended field $\mathbb{F}_{13^4}$. The order of $f$ is $4080$, whilst the order of $P$ and $Q$ is $5$. That’s why we must raise $f$ to some power to bring down its order to $5$. This operation is called __“Final exponentiation.”__ It turns out that if we raise $f$ by the power of $(q^k - 1 ) / r$, it will eliminate all multiples of order $r$ and we'll get a value from the field of order $5$. This is then the ﬁnal result of the Tate pairing:

$$
e(P, Q) = (12x^3 + 11x^2 + 2x + 9)^{5712} = 6x^3 + 7x^2 + 7x + 3
$$

### Final Words
This series documents my learning journey and my understanding of elliptic curve pairings. It doesn't claim to be 100% accurate or exhaustive, but aims to provide a foundational intuition for those new to the subject. I welcome any feedback, corrections, or suggestions to improve the content and help all readers better understand the subject. Thank you for joining me on this journey and feel free to connect with me on X at [@BoyanBarakov](https://x.com/BoyanBarakov).
