# Simple Block

A simple block is a sequence of labelled instructions,
including a label at the top. Together with a nominated
entry point it forms a program.

The simple block has the instructions of the basic block
together with some new ones:

- callcc target
- goto target
- cgoto target,..., target

The operations have an implicit argument, the current continuation, `cc`,
which is the address of the next instruction. Therefore these instructions
are not terminal as written.


# callcc target

This is equivalent to `call target,cc`. A label for the current
continuation is inserted when translating the program to
basic block form.

# goto target

This is equivalent to `switch target`, however any subsequent code
up to the next label is unreachable. The translator should warn or error
if unreachable code is detected, and the code is dropped because it's useless.

# cgoto target,..,target

The is the same as a switch `switch cc, target,...,target`. A label for
the current continuation is inserted when translating to basic block form.

# Translation to basic block form

The translation is a simple one pass operation.
Code is scanned until 
- a label
- a basic block terminator, or,
- one of the new instructions 
is encountered.

A basic block is emitted and collection of the next one commenced.

At the end of the scan we have a program in basic block form.
This form can then be translated directly into MVM code.

