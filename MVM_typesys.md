# Typing system for MVM
- Base types
  - p : field element
  - w : machine word (4 elements)
  - u32 : unsigned integers, ring, wrapping ops
  - i32 : signed integers, local approximation of integers (un)checked ops

- Type constructors
  - product  A * B
  - sum      A + B
  - procedure A -> B 

- Special cases
  - void = 0 = sum of nothing
  - unit = 1 = product of nothing
  - bool = 2 = 1 + 1

The system can be enriched to support arrays and repeated sums.

# Interpretation

A product A * B is two values on the MVM stack, with B pushed first, then A.

A procedure type D -> C means 
- a value of type D is pulled off the stack, then
- a value of type C pushed after the operation

A unit has no representation. Pushes and pops of units are no-ops.
For example a procedure of type A -> 1 pops A from the stack,
it puts nothing back. Note this impacts projection numbering.

A void type has no values. 
Any product containing a void is a type error.
A sum may contains voids, which can be eliminated, however
note this impacts injection numbering.

A sum type requires a component to have a discriminant and argument.
The discriminant lives on top of the stack followed by the argument.
The type of the argument depends on the discriminant, which encodes
the index of the injection use to construct the sum.

In the case an argument is unit, it is not represented, and if
all the arguments are units, this is a special case commonly
known as an enumeration; in this case, the representation
of all components consists only of the discriminant, in other
words a single value on the machine stack. The bool type
is an example.

# cdrop

This is a nasty operation because it required *dependent typing*.
It has two types: X * 2 -> X and X * 2 -> unit depending on the top
pf stack boolean.


# Generics

Some operations are generic. For example stack manipulation
operations happily work on 2,u32,u65, and p.

Indeed, the type w (a machine word of 256 bits), operates on any 4 tuple
of these types.

Therefore, we need polymorphism as well.


# Objectives
Given a sequence of operations, with their types, the composition
of these operations constructs a type for their composite.
For example, given the typed operations:
```
  add # p * p -> p
  mul # p * p -> p
``
the composite type is deduced:
```
  add mul # p * p * p -> p
```
This allows type checking, for example:
```
  add and # TYPE ERROR
```
because `add` produces a value of type p but the second operand
of `and` requires a bool. A conversion operation or a static type
cast is required to fix this.

There are rules for sane programming. For example in the code:
```
if.true
  instrs1
else
  instrs0
end
```
the type of `instrs0` and `instrs1` must be the same.
If they're not, the program bugs out with an error diagnostic.

Sometimes, we want different tyoes in conditional branches.
The correct solution requires a static type case to a sum.

Note than this is problematic. A union type may be more 
appropriate. Unions are dependently typed sums. Since
we probably won't implement dependent typing, since that
required the programmer to write proofs, we may need 
to introduce a weaking operation 
```
A | B
```
which means either an A or B, static analysis cannot tell.
We already have an atomic operation needing this type:
```
cdrop : 2 * X -> X | 1
```
This says if the top of stack bool is false, the next on stack X
is left at  the top of the stack, otherwise a unit replaces it,
which means it is removed.

A sum type, on the other hand, is a discriminated union,
so the top of stack must be the discriminant, and the selected
value follows on the stack.

# Memory types
Stack values can be stored into and retrieved from memory,
so memory extents require types too. Although the same
notions apply to memory, the representation is different.

In principle memory consists of addressable machine words.
Therefore a triple of say field elements on the machine
stack will be represented by 3 machine words so that the
components are directly addressale.

However by adding a padding word on the machine stack,
we can store the whole word into a compact product
consisting of only one machine word.

Now the components are not directly addressable.
However we can still define a suitable projections,
by simply throwing the unwanted components off the
machine stack.

The calculus and type checking is non-trivial, because
the 256 bit machine word cannot be fully populated.
The best we can do is 4 field elements, because even
the library u64 type is represented as two field elements.
To put this another way, two u32 values cannot be packed
into 64 bits: we need 128 bits.

The problem is that the MVM prime field is not quite
big enough. However, products of *smaller* rings may
fit so it may be worth defining the rings N^n, the
subrange of integerss 0 .. n-1. Projections over
these types can be done in two operations: first division,
and them modulus.

In particular, binary multiplication in a ring size n can be
represented in a ring size n * n. Similarly a triple multiply
requires a ring size n ^ 3, and a multiply and add a ring
size n^2 + n. The signicance is that the modulo operation
can be deferred until after the triple operation.

This means, given statically determinate ring sizes,
the compiler can schedule several operations in the representatio
ring before applying the modulus operation, in other words,
correctness can be maintained by calculating
```
(((x + y) mod n) + z) mod n => ((x + y) + z) mod n
```
provided the representation ring has size n+2 or greater
(since at worse you can get two overflow bits).








