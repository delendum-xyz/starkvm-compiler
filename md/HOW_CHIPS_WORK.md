# Chip Sugar

The semantic of chips are explained below. For the moment, lets
look at a chip and see what the parser does with it:
```
chip poly1 (m:int) (b:int) 
  connector io
   pin inp : %<int
   pin out : %>int
{
   while true do
     var x = read io.inp;
     write (io.out, m * x + b);
   done
}
```
This simple chip evaluates a polynomial of degree 1.
Here's what the parser translates it to:
```
proc poly1 (m:int) (b:int) (io: (inp: %< int, out: %>int)) () 
{
   while true do
     var x = read io.inp;
     write (io.out, m * x + b);
   done
}
```
So a chip is just an ordinary procedure used to model a coroutine.
The connector name `io` is the name of a parameter which has
a record type, namely `(inp: %< int, out %> int)` which is a
record with two fields, `inp` and `out`, which are input and output
channels respectively, on which you can read and write an `int`.

Suppose you bind up to the connector:
```
device poly_4_7 = poly1 4 7;
```
The word `device` is a synonym for `var` so you actually wrote:
```
var poly_4_7 = poly1 4 7;
```
The variable is a partial closure of type
```
(inp: %<int, out: %>out) -> unit
```
Now, you can create some channels:
```
var i1,o1 = mk_ioschannel_pair[int]();
var i2,o2 = mk_ioschannel_pair[int]();
```
Note carefully, `i1` and `o1` are pointers to the *same* channel object,
type cast so one can be used for reading and one for writing. It's
not a pair of channels, it's one channel with different access permissions.

Similarly, `i2` and `o2` are the same channel.

Now you can bind these channels to the chip by staging another
partial closure:
```
var p: 1->0 = poly_4_7 (inp=i1, out=o2);
```
The type `1->0` is the type of a procedure accepting a unit `()`.
This is the type we need to get the ball rolling:
```
spawn_fthread p;
```
The procedure `spawn_fthread` accepts a unit procedure and spawns
a *fibre* (aka *fthread*) by adding the procedure continuation
to it's own scheduler active list. In Felix, it actually cheats
and puts itself on the active list, and makes the scheduler
run the new fibre immediately. This is consistent with the abstract
semantics in which the choice of which active fibre to run is indeterminate.

So now, the chip starts running. You can prove that by adding
a debug print before the loop. But right now, it will "deadlock"
the program, because we own the channel it is trying to read from,
but we haven't fed it any data. So lets do that 
```
write (o1,42);
var result = read (i2);
println$ result;
```
We write a value, 42, on channel 1, and read the result from channel 2.
Writing lets the chip read the value and continue on to its write
operation. Then it stalls and we get control back, and can read the
result on channel 2. The chip now goes back to trying to read,
and stalls again. Our program is again "deadlocked".

Now, if you actually try this exactly as written it will actually run!
It won't deadlock at all!

The reason is simple: the mainline, after reading the result,
terminates .. and in doing so there is no longer an owner
of the channel. You must remember your mainline is a coroutine
and the run time system activated it by spawning it.

So the mainline can terminate, and leave other fibres running.
No problem!

Here is a complete test program:
```
chip poly1 (m:int) (b:int) 
  connector io
   pin inp : %<int
   pin out : %>int
{
   println$ "Poly start";
   while true do
     println$ "Poly waiting to read";
     var x = read io.inp;
     println$ "Poly read " + x.str;
     var y = m * x + b;
     println$ "Poly waiting to write " + y.str;
     write (io.out, y);
     println$ "Poly wrote " + y.str;
   done
}

device poly_4_7 = poly1 4 7;
var i1,o1 = mk_ioschannel_pair[int]();
var i2,o2 = mk_ioschannel_pair[int]();
var p: 1->0 = poly_5_7 (inp=i1, out=o2);

println$ "Spawning fibre";
spawn_fthread p;

println$ "Writing to chip";
write (o1,42);

println$ "Reading from chip";
var r = read i2;

println$ "Got result " + r.str + ", terminating";
```
which will give result like so:
```
~/starkvm-compiler>flx t
Spawning fibre
Poly start
Poly waiting to read
Writing to chip
Poly read 42
Poly waiting to write 175
Reading from chip
Got result 175, terminating
Poly wrote 175
Poly waiting to read
~/starkvm-compiler>
```

Notice in the debug trace that the mainline claims to be about to terminate
whilst the chip is still running. In fact, it must terminate before
the chip can starve to death, since it owns the channel the chip is waiting
on, and the chip cannot die until the other owner, the mainline, actually
terminates, releasing the channels it owns.

# Ciruit Sugar

One of the critical things in circuit design is to to get the correct
termination semantics, both fibres and channels must be anonymous.
There is no handle for a fibre, but channels have names. 

Because termination requires loss of reachability, it's important
for code constructing cicruits to *forget* the channels it created
as soon as possible so that they are anonymous. As long as
an active or running fibre owns a channel, another fibre
trying to do I/O on that channel will stall, because the system
can see the active owner *could* do I/O on the channel,
even if it doesn't.

The best way to forget channels is to construct them in a subroutine,
spawn the fibres you want running after connecting all the channels
up, and then return control from that procedure, which makes that
procedure's stack frame unreachable. So the fibre which called
that procedure no longer owns the channels, only the chips it
spawned do.

When a fibre stalls on an I/O operation, the ownership relation
is changed: now the fibre, which owns the channel, suspends
itself and puts itself on the channel's wait list: so now,
the channel owns the fibre! A beautiful circularity!

But remember, fibres are anonymous, so the only entity, other
than a channel, which can own a fibre, is the scheduler.
And since the fibre is no longer running or active but
suspended it has lost ownership of the fibre, and can no
longer reach it.

So now, the fibre is owned by the channel, and the only way
to reach that fibre is from an active fibre owned by the scheduler
which can reach that same channel.

So what a circuit does is it connects a swag of chips with channels
without exposing the channel names to the programmer:
```
device a = ..
device b = ..
circuit 
  connect a.out, b.inp
endcircuit
```
The circuit statement constructs a channel with two endpoints, plugs
the channels into the devices, and then spawns the devices to start
the circuit running .. without giving you, the programmer, any way
to hold onto the channels, because you don't know their names.

It is saving you a lot of boilerplate, as well as abstracting
away the channel names to ensure you cannot interfere in the 
termination semantics.

# Deadlock

Impossible! Well, provided you forget channels you are not going
to use, Felix circuits *cannot* deadlock.

There are two scenarios that you need to be aware of.

First, if you hang onto a channel that a fibre is trying
to do I/O on, but refuse to satisfy the I/O request,
this is called a *livelock*. It is usually a consequence
of not forgetting channels when you should have.

So what happens in the classic deadlock: fibre A writes
on channel 2, then on channel 1, fibre B reads on channel 2,
and then on channel 1.

The astute reader will guess immediately! Both fibres die
immediately because there are no active fibres left,
so the system does NOT deadlock, it terminates correctly.

This may well be unexpected, but death by starvation is
NOT a bug, it is the correct and ONLY way to terminate
other than suicide.

If you have a long pipeline with a source and a sink,
and all chips in the pipeline have an infinite loop
doing reads and writes, the process will run forever.

If you need to terminate the process, then you will
add one chip with a counter that copies input to output
a finite number of times then suicides (by simply
dropping off the end of the code).

As soon as that chip terminates the whole pipeline
automatically collapses. You do not need to do any
tests to discover if you could get more data,
or if there is a receiver for data you write,
in fact you *cannot* do such tests, there is no API
for it because it is a not a well defined concept.

# How Chips Work

Before I can explain this, you will note Felix has two familiar
conventional modular program constructions: procedures and functions.
- A function returns a control and a value and may not have side effects.
- A procedure returns control but no value, and must have side effects.

Both functions and procedures are special cases of subroutines
which are a special case of the most general construction, the routine. 

Routines accept arguments and control,
they do not return either control or a value. They may stop, halting
the program, call another routine, or in general, resume any
suspension. Note that calling a routine requires first creating
continuation suspended at its entry point, then resuming it.

When a routine (and thus a function or procedure) is invoked
it creates an object called a continuation which encapsulates
the current execution state. Various operations can suspend
the current execution, and suspended executions are represented
by continuations. Continuations are first class values and can be
passed to a routine as arguments.

A subroutine is a routine in which there is an implicit continuation,
called the current continuation, passed secretly as an argument by
the caller, it is in fact the callers continuation itself which is
passed. The subroutine, when finished, resumes that continuation
storing a return value in the caller's continuation if it's a function,
or not, if it's a procedure.


Note carefully, routines, subroutines, functions and procedures
are lexically contiguous blocks of text which are parsed into
an AST term. Do not confuse the term with the run time results
of invocation .. even though I regularly do!

Subroutines naturally stack, and as you might guess, the current
continuation passed to a subroutine is nothing more than the stack,
on top of which a return address was pushed. Note, the continuation
is not merely the return address but the callers execution context,
which is encapsulated in its stack frame and indeed the whole
of the stack up the call chain.

In Felix, the situation is complicated: functions use the machine
stack for C compatibility, but a heap allocated linked list called
a spaghetti stack for procedures. This causes all kinds of grief
but is essential for integration with conventional stack based
languages.

# Coroutines

There is another special case of the routine: the coroutine.
Coroutines are spawned which creates an execution context
called a fibre. Every fibre has its own stack, it can call
any subroutine or function, however it cannot return since
it is not passed the spawner's current continuation.

Instead, a coroutine can either suicide, or run an infinite
loop. However coroutines have several ways to suspend.
- a coroutine can spawn another fibre 
- a coroutine can read or write a channel

Spawning a new fibre suspends both the spawner and
the spawnee by adding them both to the scheduler
active list. Since the spawner was running and now
isn't, the scheduler now picks another fibre from
the active list to resume (typically the spawnee.)

When the scheduler runs out of jobs on the active list
it returns. The *run* subroutine is used to create
a scheduler and initialise the active list with a single
startup coroutine. Felix startup code is in fact a coroutine
scheduler so your mainline is already a coroutine.

Reading and writing channels works as follows: a channel
is just a list of fibres all waiting to read or write.
If you do a write on a channel and it has no readers,
you get added (suspended!) to the list of waiting writers, 
and the scheduler has to pick another fibre to run.

If, on the other hand, there is a waiting reader, data transfers
and both reader and writer are resumed by being put on the schedulers
active list.  Reading works dually.

Now the critical thing to note is this: if you try to read
on a channel, and there is no writer waiting, and there is
no active fibre that owns the channel, your read can never be
satisfied, so you can never resume, so you're dead!

The channel isn't reachable, and now it owns you,
so you're not reachable either, so the garbage collector
just deletes you and the channel as well.

This is called death by starvation, the other way to die
is if you're blocked from writing.

It's VITAL to understand these semantics, because the designers
of go-lang did not and got termination completely wrong.

It's VITAL to understand, channels are not pipes carrying
data, they're objects used to exchange control: the data transfer
is a convenient side effect of the design; a channel is simply
a list of suspended fibres.

It's VITAL to understand the system is entirely synchronous
and is entirely a sequential programming construction:
all events in the system are totally ordered, there is no
asynchronicity at all.

It's VITAL to understand you cannot open or close channels
and you absolutely cannot read an end marker. Data transfer
along channels is the correct model for streams.

In fact, the system which I call Communication Sequential
Coroutines (CSC) is precisely Tony Hoare's Communicating
Sequential Processes (CSP) with one modification: concurrent
evaluation is replaced by indeterminate but total ordering.

The point of exchange of control with coroutines is entirely under
program control, so unless you perform a read or write or spawn,
you cannot lose control, therefore, all operations between
these synchronisation points are atomic. Unlike threads,
where no operations are atomic unless explicitly synchronised.

So now you understand chips! It's just a convenient abstraction
of the coroutine semantics with absolutely superior modularity.
That modularity comes precisely because the connections between
chips are entirely independent of the chip semantics itself.
From the outside a chip is a black box, from inside the chip,
however, the environment is also a black box.

# Pipelines

The simplest circuit you can make consists of a source
with an output channel, a sink with an input channel,
an a sequence of transducers with one input and one output
channel. If these transducers are just functions that read
their argument and write the function result, a pipeline
is *precisely* a monad.

There is special syntax for constructing pipelines.
[TODO: show syntax]

# Purity and referential transparency

Coroutines obey continuation semantcs, so they're
intrinsically imperative with explicit management
of control flow. However, despite the imperative nature
chips can have the same essential property as functions:
a chip is *pure* if it depends only on data it reads
from channels and it's only effect is by writing to channels.

Chips don't have to be pure but if they are, circuits built
from them have a property called *referential transparency*
which ensures you can plug them into any circuit, and their
behaviour will not depend on the circuit.

Not all chips are pure. In fact, it is typically, if not
essentially the case that in otherwise pure circuits at
least one chip, usually a sink, must necessarily be impure,
for the same reason there's no such thing as a purely
functional program: a totally pure system with no side effects
is useless because it's unobservable.

# How to get results out of circuits

This puzzled me for two decades! Suppose you have
a chip that reads a number, add it to an accumulator
and write the result out, then loops around.
In other words, it converts a sequence into a series.

Now suppose you have a source loaded from a list of numbers
and you want to know the total. You can plug that source chip
into the series creator and then plug its output into a printer
and your result is the last value printed.

But suppose you just want the result internally, which a function
would give you easily? How do you do that? 

Remember THERE IS NO END MARKER. A list has an end marker so
you COULD add an endmarker and change the channel type to an option
type. But that's cheating!

In fact this kind of crap is precisely what we're trying to avoid!

The answer is simple: the sink is passed a pointer to a variable
in the routine that wants the result and writes every partial
result there. The last result written is the one you want.

Ok so how do you know when it's the last result?
Well you call `run`. It's that simple. It doesn't return
until your coroutines are all dead, so you cannot even 
look at the variable until it contains the final answer.

That may seem really dirty to a functional programmer but
the functional programmer totally missed the point: 
this is precisely how functions already work! The pointer
to the place to write is the stack pointer! The only difference
is that with our system, we have to make that location explicitly
available via a pointer passed as an argument to the sink,
and you will immediately note that the extra explicit work is
a consequence of a much more powerful and general abstraction.

If you want more flexible ways of plumbing a system than the
rigid constraints of functional programming, you cannot complain
if you need to do more work to configure your system: Felix has
functions so you can always use them as well!

# Can circuits replace fuctions?

Have you looked at an actual CPU? Of course circuits can
replace functions. But it's the wrong question. 

Well should they?

That's the wrong question too. Here's the right question:

How do you use circuits and functions together?

The answer is: programming is an art. It's not engineering.
There is no formula for design, it's a creative and experimental
process.

