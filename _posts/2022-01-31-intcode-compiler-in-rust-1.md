---
layout: post
date: 2022-01-31
title: "Writing an Intcode Compiler in Rust (Part 1)"
---

This is the first post in a series that documents my efforts to write a compiler for the Intcode language that was created for the 2019 Advent of Code event.  For more information about the goals of this project and an index of posts, see [here](/intcode-compiler).

## The Intcode Virtual Machine

The specifications for the Intcode Virtual Machine (the IVM) [^1] are laid out across several AoC challenges.  You should definitely go through them all and find your own solutions (a complete list is in the [index post](/intcode-compiler)).

[^1]: Strictly speaking, the source material describes an *abstract* machine, an unimplemented description of a computing machine.  When we later built an Intcode interpreter, that interpreter will be a *virtual* machine, which is a working simulation in software of an abstract machine.  One could also construct a physical machine that implements the IVM.  (Any sufficiently motivated and weird-in-a-good-way readers are highly encouraged to do this and tell me about it!)  Thus, the abstract machine is a specification for concrete machines, which can be implemented either in software (as virtual machines) or in atoms (as physical machines).  For simplicity's sake and to avoid a namespace collision with AWS, I'll consistently refer to the IVM for this project.

In its final form, the IVM consists of an unboundedly long array of linear memory.  At program start, the first chunk of memory is occupied by the program code, and all subsequent values are initialized to zero.  Other than its memory, the IVM maintains two pieces of state.  The first is the program counter (`PC`), which starts at 0.  The second is the "relative base" (`RBP`), which can be used to specify addresses relative to an offset.

### Instruction Set

On each "tick", the IVM reads the block of memory at the program counter, which encodes both an operation (the opcode) and a set of addressing modes that are used to interpret the operation's parameters.  The IVM supports 9 operations:

* `ADD` (opcode 1).  This operation takes 3 parameters, and writes the result of adding the first two parameters to the location specified by the third.
* `MUL` (opcode 2).  This operation takes 3 parameters, and writes the result of multiplying the first two parameters together to the location specified by the third.
* `IN` (opcode 3).  This operation pauses until an integer input value is received from the environment, then writes that value to the location specified by its sole parameter.
* `OUT` (opcode 4).  This operation returns the value specified by its sole parameter as an output to the environment.
* `JIF` (opcode 5).  If the first parameter specifies a number other than zero, set the program counter to the value specified by the second parameter.
* `JNOT` (opcode 6).  If the first parameter is zero, set the program counter to the value specified by the second parameter.
* `LESS` (opcode 7).  If the first parameter is less than the second parameter, write `1` to the location specified by the third parameter.  Otherwise, write `0`.
* `EQ` (opcode 8).  If the first parameter equals the second parameter, write `1` to the location specified by the third parameter.  Otherwise, write `0`.
* `RBP` (opcode 9).  Add the single parameter to the current relative base, which is relevant to one of the addressing modes discussed below.
* `HALT` (opcode 99).  Halt execution.

After processing the instruction, the program counter automatically advances if a jump did not occur.  The program counter advances past the current opcode plus its parameters.  So, after an `ADD` operation, the counter advances by 4.  After `JIF`, the program counter advances by 3 if there was not a jump.  (If there was a jump, the IVM continues on to the next instruction without further adjusting the program counter.)

### Addressing modes
Each parameter can be provided in one of three addressing modes:
* Position (mode 0).  The value used by the operation is the value in the memory location specified by the parameter.  For example, if an `ADD` operation has two position-mode parameters `1` and `72`, the result of the `ADD` will be the sum of the values stored at addresses `1` and `72`.
* Immediate (mode 1).  The value used by the operation is the parameter itself.  In the previous example, the result of the `ADD` would be `73`.
* Relative (mode 2).  This is similar to position mode, except the referenced location is determined by adding the value of the parameter to the current relative base.  If the relative base is `0`, then relative mode and position mode are the same.  In the previous example, if the relative base was `10`, then the result of the add would be the values stored at positions `11` and `82`.

Immediate mode is invalid for write instructions.  For example, a write parameter of `31` means that either that the value should be written to position `31` (in position mode) or that the value should be written to `31` plus the relative base (in relative mode).

Addressing modes and opcodes are encoded together.  The number stored in memory is equal to the opcode + 100 * the mode for the first parameter, if used + 1000 * the mode for the second parameter, if used + 10,000 times the mode for the third parameter, if used.  Thus, a `JNOT` instruction where the first parameter is immediate and the second relative would be encoded as `6 + 100 * 1 + 1000 * 2 = 2106`.  If the opcode does not use a parameter, a zero is provided.

### The IVM as a compiler target

As a toy target for a toy compiler from a toy language, the IVM has a few advantages.  Most obviously, it's very, very simple.  It is not trivial to even *count* the number of `x86` instructions, but 1500 is a minimum.[^2]  Moreover, the most complex instructions are often critical to performance (which is why they were created).  We don't even have to worry about subtraction!  This means that instruction selection, which is a difficult problem for real compiler authors, will be trivial for us.  Indeed, the harder problem for us will be filling in the gaps between the operations supported by our source language and the instructions we can actually run on the IVM (especially division).

[^2]: See [here](https://fgiesen.wordpress.com/2016/08/25/how-many-x86-instructions-are-there/) for a discussion, which includes the following horrifying side note:

    > So how many x86 instructions are there if we count distinct iforms as distinct? Turns out, an even 6000. Is that all of them? No. There are some undocumented instructions that XED doesn’t include (in addition to the several formerly undocumented instructions that Intel at some point just decided to make official). If you look at the Intel manuals, you’ll find the curious “UD2”, the defined “Undefined instruction” which is architecturally guaranteed to produce an “invalid opcode” exception. As the name suggests, it’s not the first of its kind. Its older colleague “UD1” half-exists, but not officially so. Since the semantics of UD1 are exactly the same as if it was never defined to begin with. Does a non-instruction that is non-defined and unofficially guaranteed to non-execute exactly as if it had never been in the instruction set to begin with count as an x86 instruction? For that matter, does UD2 itself, the defined undefined instruction, count as an instruction?

Second, the IVM is essentially a register-based architecture, but it doesn't have any general-purpose registers.  Instead, we are limited to the `PC` and `RBP` registers, which are for program flow and memory management.  Data is loaded and stored directly from memory.  Again, this simplifies the normally hard problem of register allocation, thought it may make it harder for us to implement complex behaviors.  In essence, registers make a CPU *stateful*.  Given a particular instruction and a particular state of memory, a CPU with registers can do different things depending on the values in those registers.  To take a simple example, consider a standard function call.  After the function completes, the CPU needs to return to the call site, which means it needs to "remember" where the call site is.  In a register machine, the return address could be tucked in a register, but that won't be an option for us.  We'll need to manipulate memory in a way that saves the return address, can be found by the CPU when needed, and doesn't clobber data that will be needed later.  Much more on this in due time.

### Writing Intcode

In Advent of Code puzzles, Intcode programs are presented like this:
```
3,8,1001,8,10,8,105,1,0...
```
Not very illuminating.  We'll adopt a couple conventions to make this easier to read:
* First, we'll split the operations into lines.  Strictly speaking, not all Intcode programs can be split this way, since the length of a line is determined by the opcode, which can be changed during the life of a program.  We won't be modifying opcodes at runtime, so that won't be a problem.
* Recall that addressing modes are read right-to-left (with the first parameter's mode being closest to the opcode).  For clarity, leading zeroes that are used as addressing modes will be included.  Instructions with fewer than 3 parameters will be padded with spaces.
* The opcode will be parenthesized to separate it from the addressing modes.
* Explanatory comments will begin with `//` and continue to the end of a line.
* Where appropriate for reads and writes in position mode, the integer index may be replaced with a labeled reference (like `&A`), and the initial value in the target location will be either replaced or tagged with the label (like `171#A` or `#A` if the initial value isn't used).
* Relative reads and writes will usually be prefixed with a `@`, but more details on that to come.

With these rewrites, the Intcode sample above looks like this:
```
// Store input at position A
  0(03) &A 
// Add 10 to the value in position A and store it in position A (i.e., 
// increment position A by 10) 
010(01) &A 10 &A
// Because 1 is greater than 0, unconditionally jump to the position indicated 
// by the second parameter (which is in position mode).  
 01(05) 1 #A
```
## Our First IR

Now, on to our first bit of actual compiler design, which will be designing a low-level representation of an IVM program.  This format will be an intermediate representation (IR) of the program as it goes through the process of compilation.  Specifically, it will be the last step before our compiler emits the string of integers that will make up our Intcode program.  As such, it needs to be close enough to Intcode to make code generation trivial, but abstract enough to simplify analysis for the preceding step. Finally, we need to make sure that any abstractions we introduce don't hide information that prior stages of the compiler might need.

### Additional instructions?

As an example of the last consideration, we noted earlier that the IVM doesn't have a `SUB` instruction.  This isn't a big deal, and it's easy enough to simulate one:

```
SUB [arg1], [arg2], [dest]
```
is directly translatable to:
```
MUL [arg2], -1, &temp
ADD [arg1], #temp, [dest]
```

We could add the `SUB` instruction to our new IR, along with division, remainders, and additional Boolean operations.  We'll definitely need all of those things, and we'll be defining little subroutines like the one defined above that we can insert wherever the source code calls for those operations.  If we wait to add those subroutines until the final code generation step, the resulting IRs would be more compact and easier for a human to read.  The drawback, though, is that representing `SUB` as an atomic instruction in the IR *obscures* its true nature from the rest of the compiler, which could result in suboptimal decisions.  Consider the following simple program:
```
int x = 5 - 2 * y;
```
We *could* translate this code as follows:
```
MUL 2, @y, @temp
SUB 5, @temp, @x
```
But this would be a mistake! The above translation would give us the Intcode below:
```
020(02) 2 @y &temp1
000(02) -1, #temp1, &temp2
200(01) 5, #temp2, @x   
```
A better translation is:
```
0202 -2 @y &temp1
2001 5, #temp1, @x
```
In effect, we can rewrite the source code as shown below to avoid the extra multiplication.  Or, equivalently, we fold the two constant multiplications into a single operation.
```
int x = 5 + (-2 * y);
```
However, the compiler can only make this optimization if it understands that there are two multiplications rather than one.  Thus, including the `SUB` instruction in the low-level IR would lose the opportunity to do this.  Now, we could look for other ways to get to the same result.  When we convert the AST of the source code, we could look for opportunities to convert subtractions into additions and multiplications.  But that approach would require either a bunch of hand-written optimizations for each added instruction, or else a general concept of instruction cost.

We do need to keep in mind, though, that tailoring the earlier stages of the compiler to the peculiarities of Intcode makes them less portable.  A compiler that replaces division with a bunch of crazy loops (which we're going to need to do) is a bad compiler for hardware that natively supports division. On this point, we will just plead expedience and encourage you to use LLVM or GCC for your non-Intcode compilation needs.

### Code layout
So if we aren't adding new instructions, what abstractions will we introduce in this IR?  The physical organization of the program itself.  Consider again the sample Intcode snippet from the Advent of Code problem discussed above.  If we just format the raw Intcode, we get this:
```
  0(03) 8
010(01) 8 10 8  
 01(05) 1 0
```
While this is easier to read than a comma-separated list of numbers, the intent of the program is still very obscure.  It's not obviously different from the following program:

```
  0(03) 9
010(01) 9 10 9  
 01(05) 1 0
```
But where this program just loops infinitely, storing meaningless values to a location outside the code snippet, the first program jumps somewhere else.  The key to decoding the program is understanding that those enigmatic `8`s are actually pointers to a specific location in the code snippet.  These pointers are a perfect candidate for abstraction, because the rest of the compiler generally won't care about them at all.  Moreover, their actual values are volatile.  The Intcode snippet above only works the way we expect if it's the first piece of a program.  If you inserted it anywhere elsewhere, it would have totally different effects!  If we *don't* abstract this detail away, the rest of the compiler pipeline will have a huge bookkeeping burden every time it wants to relocate a piece of code.

Our abstraction will look a lot like the conventions we used above to make raw Intcode more palatable, but we'll apply them universally.  Concretely:
* Immediate-mode parameters will be written as integer literals.
* All position-mode parameters will be represented by an ampersand and a text label, like `&temp1`.
* Each such label will have a single corresponding anchor in the IR, which will be written as `#temp1`.  The anchor will be treated as an immediate-mode parameter where it's used unless it is further prefixed with `&` (like `&#temp1`).  In this case, we can't provide a useful label for the target of the reference, since the reference will change at runtime.  We will aim not to generate much code like this, and will try wherever possible to use anchors only in immediate mode.
* Relative-mode parameters will be written as labels prefixed with the @-symbol, like `@x`.  As we'll discuss in more detail later, the referents of these parameters will generally *not* appear in the source code.  Instead, we'll be using them for values that are stored in memory at runtime.

With these conventions, the code sample now looks like this:
```
  0(03) &a
010(01) &a 10 &a  
 01(05) 1 &#a
```
Our conventions fully specify the addressing modes used in the instruction, so we can omit the leading integers.  We can also replace the opcodes with descriptive names for the instructions, giving us:
```
IN  &a
ADD &a 10 &a  
JIF 1 &#a
```
Not terrible!

### Operand endianness
By design, our IR looks a lot like assembly language.  There's one big semantic difference though.  Assembly instructions typically list their destinations *first*.  `mov EAX 10` loads the value 10 into the register `EAX`.  By contrast, an IVM instruction takes its destination as the *last* operand.  `ADD &x 10 &y` stores the sum of `x` and `10` in the location indicated by `&y`.  This may be a mistake, but I think we'll ultimately avoid confusion if we reorder the operands in our IR **to match the assembly convention**.  This will make it easier to translate real assembly into IR if that's ever useful, and it reads a bit more like a source-level programming language, where the destination of an assignment comes first.

When we write raw Intcode, we'll continue to use the IVM convention.  But from this point forward, IR will use the assembly convention:

**🠉 IVM convention (`ADD &x 10 &y` stores to `&x`) 🠉**
___
**🠋 Assembly convention (`ADD &x 10 &y` stores to `&y`) 🠋**

Assembly jump instructions only take a destination argument and not a data argument.  The branching decision is based on "condition codes," which are special flags that are set by whatever the most recent arithmetic operation was.  In this case, we'll stick with the Intcode convention.  So `JIF &restart 0` will jump to position `0` if `&restart` is nonzero.

### Additional instructions??
As we discussed, we want to avoid introducing complex instructions that obscure significant implementation details from the rest of the compiler.  However, there are a few areas where simple abstractions can be added without hiding anything important:

* A copy instruction `COPY`, which can be implemented as `ADD &dest 0 &source`
* An unconditional jump instruction `JUMP`, which can be implemented as `JIF 1 destination`.
* To facilitate jump instructions, we also need a new way to specify an address.  Our existing `&` and `#` syntax allows us to specify data locations, but we want our jumps to land on opcodes, not data.[^3]  We can easily fix this by adding a `LBL` instruction, which will have one argument giving the location a name.  When we emit Intcode for the IR, the `LBL` instruction will be omitted, and references to the label name will refer to the position of the opcode for the line immediately following the label.  Label names will be alphanumeric strings (with underscores also allowed), and no prefix is needed when the label is referred to.  To help distinguish between different kinds of references, references to a label will be prefixed with `$` rather than `&`.

[^3]: The Intcode spec does not require any particular alignment of opcodes and operands, and it would be perfectly legal to jump to a location that looks like an operand based on a simple parsing of the program into lines.  But that would also be very confusing, so we won't be doing it.

* For slightly improved clarity, we can add in-place addition and multiplication.  `IADD/IMUL [dest] [value]` will reduce to `ADD/MUL [dest] [value] [dest]` (or `ADD/MUL [dest] [dest] [value]`).  This will make our IR slightly more compact, which is good.  More importantly though, this representation will hopefully make it slightly easier for the compiler to notice and optimize accumulator-type operations.

See a slightly elaborated version of the prior code sample for examples of our new instructions.
```
JUMP $continue
HALT // Not executed
LBL  continue
IN   &a // The JUMP instruction lands on the operand of this line of code. 
COPY @saved_input &a // Save a copy of the input in a relative-mode location.
IADD &a 10 // Add 10 to the input in place.
JUMP  &#a // We can't use a label here, because the destination is computed 
          // at runtime.
```

## Up next
Now that we have an IR, we can write some nontrivial programs without going mad, then compile and run them.  To do that, though, we'll need an Intcode runtime and, you know, a compiler.  Those programs will be the subject of the next two posts.
