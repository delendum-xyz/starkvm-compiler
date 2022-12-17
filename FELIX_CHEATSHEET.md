# Application

| syntax | assoc | prec | Notes
| -------| ----- | ---- | ----
| f $ x  | R     | low  | Haskell
| f x    | L     | mid  | math
| x . f  | L     | hi   | OO
| #f     | prefix| vhi  | f ()
| p &. x |       |      | (&p) . x
| p *. x |       |      | (*p) . x
| p -> x |       |      | (*p) . x

# Type Constructors
| syntax | assoc | prec | Notes
| -------| ----- | ---- | ----
| 0      |       |      | aka void
| 1      |       |      | aka unit
| 2      |       |      | aka bool
| A -> B | R     | vlow | Function
| A + B  | assoc | low  | Sum (numbered choice)
| A * B  | assoc | med  | Product (tuple)
| A ^ B  | L     | vhi  | array
| A *+ B | L     | hi   | coarray
| &X     | prix  | vhi  | RW address
| &<X     | prix  | vhi  | R address
| &>X     | prix  | vhi  | W address
| %X     | prix  | vhi  | RW channel 
| %<X     | prix  | vhi  | R channel  
| %>X     | prix  | vhi  | W channel 

# Primitives
```
int8 int16 int32 int64 int128 int256
uint8 uint16 uint32 uint64 uint128 uint256
short int long vlong
float double ldouble
fcomplex dcomplex lcomplex
string
```

## Record

| Example   | Type
| ------ | -----
| (a=1, b =2) |  (a:int, b:int) 

## Tuple 
| Example   | Type
| ------ | -----
| 1,"hi" |  int * string 

## Array
| Example   | Type
| ------ | -----
| 1,2,3 | int ^ 3

# Expressions
## Binary operators
| ops     | Notes
| ------ | -----
| + - * / % | math
| and or not | logic
| < <= > >= == != | comparison

## Unary operators
| ops     | Notes
| ------ | -----
| &v &<v &>v | pointer
| *p | fetch

## Conditional
```
if .. then .. elif .. else .. endif
```
## let form
```
let x = expr in expr
```
## lambda
```
fun (x:int) => x
let fun (x:int) => x in ..
proc (x:int) { println$ x; }
{ ...; return 1; } // function
{ ... } // procedure
```
# statements
| ops     | Notes
| ------ | -----
| p <- x; | store
| v = x; | &v <- x;
| f x;   | call
| f;     | f ();

# variables
```
var x = 1; // mutable, addressable, eager
val x = 1; // single assignment, not addressable, maybe lazy
```

# loops
```
for (var i = 1; i < 20; ++i; ) perform
  println$ "hello"
;
for i in 1,2,3,4 do
  println$ "hello";
done

while x < 20 do
  println$ hello;
done
```
  
# function
```
fun swapT,U] (x:T,y:U) => y,x;
fun swapT,U] (x:T,y:U) { return y,x; }
```

# procedure
```
proc hello() { println$ "Hello"; }
```

# generator
```
gen rand: 1->int = "rand()";
gen ints() {
  while true do
    var x = 1;
    yield x;
    ++x;
  done
}
```
  
# Nominal Types
## struct
```
struct Pair[T,U] {
  a:T;
  b:U;
}
var x = Pair(1, "hello");
var y = x.a;  // value projection
var p = &x.b; // pointer projection
var z = *p;
```

## variant
```
variant Maybe[T] = 
  | None
  | Some T
;
```

## type class
```
class Add[T] (
  virtual fun add: T * T -> T;
  virtual type U;
  fun add3 (x:T, y:T, z:T) => add(add(x,y),z);
  type Int = int;

```
Virtuals are overridable but may have defaults.
Non virtuals cannot be overriden.
```
instance Add[int] {
  fun add(x:int, y:int) = "$1+$2";
  typedef U = int;
}
```
Instances only need to override used entities.
```
open[int] Add[int]; // open class specialisation
fun f[T where Add[T]] (x:T) => add(x,x);
```

