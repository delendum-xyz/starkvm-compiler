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
class Add[T] {
  virtual fun add: T * T -> T;
  virtual type U;
  fun add3 (x:T, y:T, z:T) => add(add(x,y),z);
  type Int = int;
}
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
fun f[T with Add[T]] (x:T) => add(x,x);
```
# Coroutines

## spawn_fthread
```
spawn_fthread: 1->0->0
spawn_fthread { printon$ "Hello"; }
```
Service call to spawn a new fibre on callee's scheduler.
Adds fibre to scheduler active list.

## Run 
```
run: 1->0->0
run { printon$ "Hello"; }
```
Subroutine to spawn a new fibre on a new scheduler.
Subroutine call terminates when the scheduler has no active fibres.

## Read
```
var x = read chan;
```
Synchronisation primitive.
Read value from channel. Suspends fibre until
matching write performed by moving to channel wait list.
Fibre moved to scheduler
active list on completion of data transfer.

## write
```
write (chan,value);
```
Synchronisation primitive.
Write value to channel. Suspends fibre until
matching read performed by moving to channel wait list.
Fibre moved to scheduler active list on completion of
data transfer.


## suicide
```
suicide;
```
Terminates fibre, releasing all channels.

## Operaion.
A scheduler owns a list of active fibres. It picks
one at random and runs it until the fibre terminates
or suspends itself.

Fibres suspends themselves by moving from active
to a suspended state and waiting on a channels
wait list.

I/O occurs when a write is performed on a channel
with at least one suspended reader, or a read is
performed on a channel with at least one suspended
writer. Then both reader and writer are moved the
the scheduler active list.

A scheduler terminates when its active list is empty
and the currently running fibre suspends itself.

Channels are owned by fibres. Channels also own
suspended fibres. A reader fibre is starved and 
a writer blocked if suspended on an unreachable channel, 
in which case the garbage collector reaps the channel
and all fibres waiting on it. Reaping fibres eliminates
reachability of its channels via the fibre.

Coroutines are a sequential programming construct, all
events are totally ordered. Operations between synchronisation
points are universally atomic. However, the actual ordering
of events is not deterministic, which, to cite Tony Hoare,
is a vital source of abstraction.

If the ordering is weakened by replacing indeterminacy
with concurrency the semantics change from CSC (Communicating
Sequential Coroutines) to CSP (Communicating Sequential
Processes) and data synchronisation using locks may
be required to enforce atomicity of data operations.
However, abstraction is increased and real time 
performance enhanced concurrency.

## Note: FP myth
There is a myth that lists are finite and streams infinite.
The is incorrect. A list can be scanned recursively determining
the reachable lenth which is only a lower bound on the list's length.
Functional objects are not persistent, they're immortal, just
as garbage collection is an optimisation, so too construction an optimisation
for an operation properly called location: construction is a misnomer
because the world is read only.

Dually, streams operations begin at a definite time so the length 
of a stream is bounded from the start, and has no definite end,
so have an upper bound on the number of reads that can be
performed, fixed by the number of writes. Dually the number
of writes is bounded by the number of reads.

Streams and lists, or more generally inductive and coinductive
types are bridged by swapping from the spatial world of functions
and products to the temporal world of cofunctions and sums using
the `run` subroutine and termination semantics. There is a bridge
in the opposate direction using pointers to mutable store.

We note, iterators and generators are special cases of bridges.

# Advantages of Coroutines
Coroutines provide better modularity and superior scalability
at the price of indeterminate evaluation ordering compared
to functions: they operate using continuation semantics,
which can be difficult to manage. 

Functions are also have indeterminate ordering, depending
on the evaluation strategy, however dependencies tighten
event relations to a partial order. Trees are much easier to
understand than arbitrary circuits.








