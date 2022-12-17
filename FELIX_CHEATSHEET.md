# Application

| syntax | assoc | prec | Notes
| -------| ----- | ---- | ----
| f $ x  | R     | low  | Haskell
| f x    | L     | mid  | math
| x . f  | L     | hi   | OO
| #f     | prefix| vhi  | f ()
| p &. x |       |      | (&p) . x
| p *. x |       |      | (*p) . x
| p -> x |       |      | (*p) -> x

# Type Constructors
| syntax | assoc | prec | Notes
| -------| ----- | ---- | ----
| A -> B | R     | vlow | Function
| A + B  | assoc | low  | Sum (numbered choice)
| A * B  | assoc | med  | Product (tuple)
| A ^ B  | L     | vhi  | array
| A +* B | L     | hi   | coarray
| &X     | prix  | vhi  | RW address
| &<X     | prix  | vhi  | R address
| &>X     | prix  | vhi  | W address
| %X     | prix  | vhi  | RW channel 
| %<X     | prix  | vhi  | R channel  
| %>X     | prix  | vhi  | W channel 

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

