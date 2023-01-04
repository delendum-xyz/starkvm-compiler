# Proposal to Upgrade Miden Assembler v1

- MVNM shall be the name of the exitsing Miden Assembler
- BBL shall be the name of the Basic Block Language
- SAL shall be the name of the Simple Assembler Language

# BBL
BBL is a language which adds a powerful set of control structures to MVM.
It is a pure extension and will not impact any existing MVM programs.
BBL can only be used inside an MVM mainline code stream.
BBL extends the available control structures by partitioning code
into a set of basic blocks and adding new control flow instructions.

A basic block begins with a label followed by a colon, it
should be the first symbol in a line.

Thereafter any balanced MVM code may be wriiten,
where balanced means any embedded `if`, `while`,
or `repeat` and appropriately terminated.

Code prior to the first label is called the program prologue,
code at the end is called the epilogue. [Issue: how do
we find the epilogue syntactically]

After the end of the embedded MVM code, a terminate
must be written.

These are the terminators:
- switch, an indexed goto
- callcc, a subroutine call: reserved for v2
- return, return from subroutine: reserved for v2
- cocall, a coroutine call: reserved for v2

## switch
A switch instruction consists of the identifier `switch` optionally followed by 
a sequence of dot prefixed arguments.
- switch, without arguments causes the epilogue to be evaluated
- switch.label, with one argument, is an unconditional goto
- switch.F.T with two arguments is a conditional goto
  - the stack is popped and the value tested
  - if it is 0, control transfers to label F
  - if it is 1, control transfers to label T
  - otherwise fail
 - switch.L0.L1...Ln-1 is a multiway branch
   - the stack is popped 
   - if the value is not in the range 0..n-1 inclusive fail
   - otherwise jump to the label in position n,
   - where n is the value popped off the stack

## Properties of switch
Basic blocks terminated by switch require no external memory 
and do not impact the state of the MVM stack.
The program counter `PC` is always present on the MVM when required
The program counter is never present when user MVM code is elaborated


# callcc/return
These instructions add subroutine calling capability.
- callcc has two arguments, the target label and the continuation label
- An auxilliary stack called the BBL stack in memory is required
- The stack shall grow downwards
- SP is a memory location containing the current stack pointer
- C like notation will be used to describe the semantics
- Instruction `callcc.target.cc`
   - pushes cc onto the BBL stack 
   - `*--SP = cc`
   - jumps to `target` as if a `switch.target` were elaborated
- Instruction `return`
  - pops the BBL stack
  - `ret = *SP++`
  - jumps to `ret`, as if a `switch.ret` were elaborated


# cocall
The `cocall` instruction requires further research.
It is used to exchange control between coroutines.
Coroutiunes are peers which cooerate by control exchange.
It should be an aim to remove MVM `syscall` and replace it with `cocall`.
Cocall exchanged control with another coroutine:
- EPILOGUE
- retrieve the peers saved program counter `PCpeer` representing its continuation address
- retrieve the peers saved stack pointer `SPpeer`
- set the BBL stack pointer to the peers `SP=SPpeer`
- push the peers retrieved program counter `PCpeer` onto the MVM stack
- goto the continuation address `PCpeer` as if a `switch.PCpeer` had been elaborated

The above rules specify how to resume a coroutine, but they do not specify how to presevere the state required,
in order that we ourselves may be resumed.
These rules, the prologue rules, must ensure the system is in the state required at the start of the epiloge, 
since we will be resumed by a peer executing that prolog.
- PROLOGUE
- save our program counter and stack pointer

- JOIN
- To make this work we will first decide to save the PC on the BLL stack.
- Then we save the BLL stack somehow.
- When we resume, we retrieve the BLL stack pointer somehow
- Then we can just pop the PC off the BLL stack

So now, we are left with the problem, how to our BLL stack pointer so we
can retrieve it, and, conversely, how to retrieve the peers BLL stack pointer.

- The answer to this question is TEAMWORK.
- We shall not save our SP at all, we shall require our peer to do so.
- Nor shall we retrive our SP either, we shall require our peer to give it to us.
- Howewer in exchange, we shall be handed the peers SP on resumption and must save it, and,
- we must hand it back when we resume our peer.

- The agreed transfer method must be established
- the top element of the MVM stack is the obvious solution

The `SP` resister should be protected by midenVM so it acts 
as a register even if implemented using a memory location.
In particular the user shouldn't be able to access it directly,
but should be required to use BLL specific operations to manipulate it.
This protection is required to enfore the transparency, or non-interfernce guarantee.

Therefore we now have an issue: the currently executing coroutine
has to save the peer SP somewhere. This cannot be a magical place
because any coroutine can resume any coroutine that has resumed it.
So, to save and retrieve the SP, each coroutine has to allocate
some memory at a location private to it.

There is also another issue: how to being the execution of a
coroutine.

Note also, every coroutine requires its own stack.
We will probably need an instruction
```
spawn.name.999 # spawn a coroutine, allocating stack size 999
```

# SAL

SAL is an extension of BBL which introduces new control instructions which are permitted to *drop through*.
A SAL program then resembles a traditional assembly language by allowing labels anywhere.

## break
SAL break is equivalent to `switch`, that is, it terminates the von Neumann
emulation and executes the program epilog MVM code. Note these instructions
can only be used at the top level.

## goto.label
- unconditional goto
- is equivalent to `switch.label`

## cgoto.label
- conditional goto
- is equivalent to `switch.cc.label`
- where the translator must synthesise a new label,
- replace the `cc` symbol in the instruction template with that label, and,
- issue that label as the label of the next instruction
- except that, as an optimisation of the next instruction is already labelled
- that already used label should be used instead to aid diagnostics
- if there is no next instruction, what do we do?

## subcall.label
- subroutine call instruction
- equivalent to `callcc.label.cc`
- where cc represents synthesised label as for `cgoto` above
- the correct name would be `call` but that is already an MVM instruction

# HLL
HLL is a low level language built on top of SAL.
It provides standard stuctured programming constructions
such as condition chains, various loops, etc.

This is necessary to overcome the retriction that MVM does not
support BBL terminators in the same way, so that MVM code is
regarded as atomic and so you cannot say, write a conditional
goto instruction inside MVM code since the MVM parser does not
recognise the extension.

The current proposal does not introduce features like data types,
it is intended to purely deal with control structures.


# BBL Encoding
## startup
```
push.<entry>" # Entry Point
push.1  # Make Loop start
while.true # Begin loop
```
# Case 1
```
dup       # PC+
eq.1      # first case
drop      # PC
<emit_body 1>
```
## Case j
```
else\n" +
  dup     # PC
  eq.<j>  # case j
  if.true
    drop  # PC
    <emit_body j>
```
# Last case
```
else 
  drop    # PC
  <emit_body n>
```
# Endoff
```
end.n     # join(n x end) 
dup       # PC
neq.0     # loop continues unless PC = 0
end       # while loop
drop      # PC
 
```
## switch
```
push.0 # HALT/BREAK
```
##switch.target
```
push.target
```
#switch.Fcase.Tcase
```
if.true push.Fcase else push.Tcase end
```

# Performance
This implementation is linear in the number of labels.
The if then else chain used can be replaced by a binary chop
yielding logarithmic performance.

## Invariant
The PC is always on the MVM stack on entry to one of these
instructions except the startup.


