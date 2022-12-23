# Basic Block

- A basic block is a sequence of atomic operations executed sequentially in order of writing; and, 
- which has a unique label; and,
- which terminates in a switch

# Switch
A switch is a generic instruction which jumps to a basic block selected from a label array 
indexed by a value computed by the basic block previously.

- A nullary switch has no labels and is equivalent to a halt instuction.
- A unary switch with one label is equivalent to an unconditional goto
- A binary switch has two labels, equivalent to a conditional goto
- A switch of order n>1 selects a label from a read only array
- Using a program counter value on top of the miden stack

```
push.vector_offset # static array in memory
add # program counter
load # label index
```

# program unit
A program unit is a *set* of basic blocks with one singled out as the root block.
Labels of the blocks must be distinct.
Each label represented by a unique natural number.

# Execution chaining
The machine operates by starting with the root block label index on the stack.
It then executes an if/then/elif/else/end chain as follows:

- The program counter, the label of the block to execute is on the stack
- It is duplicated
- The label is compared to the current block's label.
- if they are equal, the extra label is dropped
- the atomic instructions are executed
- the next label to execute is pushed onto the stack by the terminating switch
- If the label does not match, the next block in the chain is tried.
- It is an error if one block is not executed per iteration
- Except that the emulation can be terminated by loading a special exit value onto the stack


```
dup
ifeq.<label>
  drop
  exec.atomic
  exec.atomic
  exec.atomic
  switch<n> # as above
else
  dup
  ifeq.<label> 
    drop
    exec.atomic
    exec.atomic
    switch<n> # as above
```

The chain is joined at the end, that is, control merges.
Epilogue code then calculates if the computation should exit
and terminates the initial while loop.

Note the program counter remains at the top of the stack during all
control operations.

The loop starts with a while.true instruction. Therefore it will always
be executed at least once.

The overall structure is:
```
push.<root>
push.true
while.true
  dup          # retain PC
  if ...       # begin the comparison chain
  end
  push.<run>   # 0 to terminate, 1 to continue
end

```

# Procedures

We support recursive dynamic procedure calls by adding an stacks in memory
at fixed locations. On stack is needed per coroutine.

A single SP register, a fixed memory location, holds pointer to
the top of the stack. The stack grows towards lower addresses.

To call a procedure, the caller must first push the return address
onto the memory stack. Any arguments are pushed onto the miden stack.
The caller loads the starting address of the procedure onto the
miden stack and the miden if.eq block then terminates WITHOUT a switch.

The procedure may push values onto the memory stack.

When the procedure wishes to return the SP must be reset to
point at the return address. Any return values are loaded onto
the miden stack.

The block then loads the return address onto the miden stack and
decrements the SP; then the if.eq label block terminates
without a switch.

In this extension the CALL and RETURN operations may also terminate
a basic block (as well as a switch).

# Coroutines

Coroutines operate by exchanging control in a peer to peer maner
using a BRANCH-AND-LINK instruction. The model of this instruction
is expressed in Felix as follows:
```
  proc branch-and-link (target:&LABEL, save:&LABEL)
  {
     save <- next;
     goto *target;
     next:>
  }
``` 
The current continuation of the called is the next label. It is stored at 
the address of the save argument of the instruction. Then, the program counter
stored at the adress specified by the target argument of the instruction 
is loaded into the program counter, and the callee routine is resumed.

To implement this in our model, the callee will be identified by a pointer
to its top of stack. At the top, its continuation label must be stored.
By loading this onto the miden stack and terminating the current block,
the callee coroutine will be automatically resumed. However the callee
needs to obtain a pointer to our current stack (SP) on the top of which
must be our own current continuation. Therefore we pass our SP to the
coroutine as its argument .. and when we are resumed by another coroutine,
we expect to obtain its SP as an argument.

The exact miden protocol yet to be specified.
Availability of coroutines in addition to subroutines is a major
control flow enhancement. Subroutines are a special case of coroutines.

Stack swapping coroutines are well known to all programmers: an application
and the operating system are coroutines. Iterators are a special case
of coroutines.



