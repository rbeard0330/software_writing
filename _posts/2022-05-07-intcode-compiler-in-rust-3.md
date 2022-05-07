---
layout: compiler_post
date: 2022-05-07
title: "Writing an Intcode Compiler in Rust (Part 3) - Compiling LLIR"
---

In [our last post](/writing/2022-02-10-intcode-compiler-in-rust-2.md), we wrote an interpreter to execute compiled Intcode programs.  Now we'll write code to parse our low-level IR (LLIR) and emit Intcode for it.

## Parsing

Strictly speaking, we don't *need* to have the ability to parse LLIR source files to accomplish our goal, but it will make it easier to test the backend of the compiler in isolation, and it's not that hard (or at least it doesn't have to be).  We could spend an essentially unlimited time thinking about parsers, but I want to get on to the more interesting (to me) part of the project, so we'll lean on the [Pest](https://pest.rs/) parser generator library, which creates a parser based on a PEG grammar.  The grammar for LLIR is below:

```
WHITESPACE = _{" " | "\n" | "\t" | "\r"}
COMMENT = _{"//" ~ (!line_end ~ ANY)* ~ &line_end}

word_end = _{!(ASCII_ALPHANUMERIC | "_") | EOI}
line_end = _{"\n" | EOI}
literal = @{"-"? ~ ASCII_DIGIT+ }
identifier = @{ !(reserved ~ word_end) ~ ASCII_ALPHA ~ (ASCII_ALPHANUMERIC | "_")*}

operator_0 = {"HALT"}
operator_1 = {"IN" | "OUT" | "RBP" | "JUMP" }
operator_2 = {"JIF" | "JNOT"  | "COPY" | "IADD" | "IMUL"}
operator_3 = {"ADD" | "MUL" | "LESS" | "EQ"}
reserved = _{operator_0 | operator_1 | operator_2 | operator_3}

immediate_operand = @{literal }
position_operand = @{"&" ~ identifier }
relative_operand = @{"@" ~ identifier }
label_operand = @{"$" ~ identifier }
immediate_anchor = ${literal? ~ "#" ~ identifier}
position_anchor = ${literal? ~ "&#" ~ identifier}
operand = {position_operand | relative_operand | immediate_anchor | position_anchor | immediate_operand | label_operand }

label = {"LBL" ~ identifier}
var = { "VAR" ~ identifier}
no_arg_instruction = {operator_0 | label | var}
unary_instruction = {operator_1 ~ operand}
binary_instruction = {operator_2 ~ operand ~ operand}
ternary_instruction = {operator_3 ~ operand ~ operand ~ operand}
instruction = {no_arg_instruction | ternary_instruction | binary_instruction | unary_instruction}

program = {SOI ~ instruction* ~ EOI}
```

More details about Pest's syntax are available [here](https://pest.rs/book/grammars/syntax.html), but the basic idea is that the parser tries to satisfy the `program` expression at the very end by building up some sequence of `instruction` expressions that run from the start of the input (`SOI`) to the end (`EOI`).  The `~` operator indicates that subexpressions need to happen in sequence, and the `|` operator indicates a choice of alternatives.  When run, the parser returns a tree of expressions that matched the input, and we simply create the appropriate data structures correspond to the different expressions.  By default, whitespace characters can come between the different subexpressions in a rule.  The `@` operator marks rules, such as variable names, that can't be interrupted by whitespace.

Note also that I've made one addition to the syntax of LLIR since I last worked on this.  I added a `VAR` statement, which simply creates a named memory location.  This is a preview of the discussion about variables, but I'm slipping it in here to make it easier to write the test program we'll compile later.

### Representing LLIR
Our grammar will let us parse LLIR, but what are we parsing it into?  Basically, this:
```rust
#[derive(Debug, Clone, PartialEq)]
struct Program(Vec<Instruction>);

#[derive(Debug, PartialEq, Clone)]
enum Instruction {
    Halt,
    Copy {
        dest: Operand,
        source: Operand,
    }
    // ...and many others
}

#[derive(Debug, Clone, PartialEq, Eq)]
enum Operand {
    Immediate(Int),
    Relative(Identifier),
    Position(Identifier),
    ImmediateAnchor { ident: Identifier, initial: Int },
    PositionAnchor { ident: Identifier, initial: Int },
}

#[derive(Clone, Debug, PartialEq, Eq, Hash)]
struct Identifier(String);
```
LLIR is a *linear* IR, which means it's just a sequential list of instructions. This makes the `Program` struct very natural to write as a `Vec<Instruction>`.  The `Instruction` enum is also pretty straightforward.  The most noteworthy bit of complexity here is the `Operand` enum.  Recall that one of the goals of the LLIR abstraction is to hide the complexity of indexing from the rest of the compiler. The `Operand` enum is the key to doing that. Instead of worry about exactly where a value is, it can be specified as an `Operand` with an `Identifier` (which is just a newtype of `String`).

## Emitting Intcode
Once we've parsed an LLIR program, we need to convert it to the list of integers that comprises the actual Intcode program:
```rust
impl Program {
    fn emit_intcode(&self) -> Vec<Int> {
        todo!()
    }
}
```
The obvious approach, which we'll use, is to just iterate over the `Instructions` and convert them on a one-to-one basis to their Intcode representations.  By design, this is easy to do, as all of our LLIR instructions have a direct equivalent in Intcode.  The hard part is correctly wiring up all the references.  In particular, consider the following code snippet:
```
IN   &a
JMP  $add
HALT
LBL  add
ADD  #a 10 $b
OUT  #b
```

The first integer in the output is the opcode for a position-mode input, which is `3`.  The second integer is the index where the `a` anchor will be in the final program.  Unfortunately, we don't know where that will be until we emit the rest of the program (or otherwise figure out where all the anchors are).  To solve this problem, we'll need two passes to emit a complete program.  In the first pass, we'll analyze the layout of the program and use it to determine the positions of all anchors.  Then, in the second pass, we'll use those precomputed positions to emit the Intcode program.

To make this work, each instruction will have a `layout` property that will return the following data type:
```rust
#[derive(Default)]
struct Layout<'a> {
    pub size: usize,
    pub anchors: [Option<&'a str>; 3],
    pub anchor_adjust: isize,
}
```
The first two fields are straightforward.  The `size` field is the number of integers needed to represent the instruction.  The `anchors` field indicates which of the operand slots for that instruction are anchors and, if so, what the identifier for that anchor is.  Using this information, we can calculate quickly where each anchor will lie relative to the start of the instruction, as well as the relative index to the start of the *next* instruction.  Since we know the first instruction will start at position `0`, that allows us to compute the *absolute* position of each anchor when we iterate through the layouts in order.

The `anchor_adjust` field is a kludge to help us handle `LBL` instructions.  Unlike our other LLIR instructions, `LBL`s don't actually appear in the output.  For "real" instructions, we need reserve one slot for the opcode, then an anchor that appears in the first operand slot would get the index after that.  So, for example, if an `ADD` starts at index 408 and has an anchor for its second operand, we would map that anchor to index 410: the opcode is at 408, and the first, non-anchor operand is at 409.  For virtual instructions, this doesn't quite work, since there's no opcode.  If a `LBL` instruction appears at index 408, its corresponding anchor needs to point to the start of the next instruction, which is also at index 408.  To deal with this, we set the `anchor_adjust` for `LBL` instructions to -1, and then we adjust the final computed position of the anchor by that amount.  

Now we can use the layouts to build up a hash table mapping identifier strings to indices in the final program:

```rust
impl Program {
    fn emit_intcode(&self) -> Vec<Int> {
        let mut current_index = 0;
        let mut anchor_positions = HashMap::new();
        self.0.iter().for_each(|inst| {
            let Layout {
                size,
                anchors,
                anchor_adjust,
            } = inst.layout();
            for (offset, anchor) in anchors.iter().enumerate() {
                if let Some(ident) = anchor {
                    anchor_positions.insert(
                        ident.to_string(),
                        current_index as Int + 1 + offset as Int + anchor_adjust as Int,
                    );
                }
            }
            current_index += size;
        });
        todo!();
    }
}
```

The final step is to convert each instruction into an array of integers. I decided to make this logic a method on `Instruction`:

```rust
impl Instruction {
    // ...
    fn into_ints(self, anchors: &HashMap<String, Int>) -> [Option<Int>; 4] {
        use Instruction::*;

        match self {
            Halt => [Some(99), None, None, None],
            Copy { dest, source } => {
                create_int_array(1, [source, Operand::Immediate(0), dest], anchors)
            }
            Add {
                dest,
                addends: [addend_1, addend_2],
            } => create_int_array(1, [addend_1, addend_2, dest], anchors),
            // etc.
        }
    }
}
```
The `Instruction` extracts the operands and passes them to a helper function that builds the actual array:

``` rust
fn create_int_array<const N: usize>(
    opcode: i64,
    operands: [Operand; N],
    anchors: &HashMap<String, i64>,
) -> [Option<i64>; 4] {
    let mut result = [None; 4];
    result[0] = Some(add_addressing_modes(opcode, &operands));
    for (i, operand) in operands.iter().enumerate() {
        result[i + 1] = Some(operand.as_int(anchors));
    }
    result
}

fn add_addressing_modes<const N: usize>(opcode: i64, operands: &[Operand; N]) -> i64 {
    let mut result = opcode;
    let mut base = 100;
    for operand in operands.iter() {
        result += operand.addressing_mode_code() * base;
        base *= 10;
    }
    result
}
```

With this done, it's very straightforward to fill in the todo!() above by building up a `Vec` of integers each instruction in our program:

```rust
let mut result = vec![];
for inst in self.0.clone() {
    result.extend(inst.into_ints(&anchor_positions).iter().flatten())
}
return result
```

The main point of interest here is the call to `flatten`.  Because `Option`s are iterators with 1 or 0 values, the array of `Option<Int>`s we get from the `into_ints` method is technically a nested iterator. We can simply flatten it to discard the `None` values and get an iterator of `Int`s to add to the result.

The discerning reader will also note that we clone the `Program` at the start of the loop.  Strictly speaking, this is wasteful and unnecessary, and the code should be refactored to pass around references to `Instructions` rather than owned copies.

## Running a program written in LLIR

Now, we can finally actually compile something!  Here's a factorial program in LLIR:

```
IN   &factor
COPY &result 1
LBL loop_start
IMUL &result &factor
// Decrement factor and see if we've hit the base case
IADD &factor -1
LESS &loop_test 1 &factor
JIF &loop_start &loop_test
// Output result and halt
OUT &result
HALT
// Data section
VAR result
VAR factor
VAR loop_test
```

And here's what happens when we run it:

```
$  cargo run --bin icc -- -r llirs/factorial.llir compiled
    Finished dev [unoptimized + debuginfo] target(s) in 0.06s
     Running `target\debug\icc.exe -r llirs/factorial.llir compiled`
[src/bin/icc.rs:59] &compiled = "3,25,1101,1,0,24,2,25,24,24,101,-1,25,25,107,1,25,26,1005,26,6,4,24,99,0,0,0,99"
Input required
5
Output: 120
```

## Up next

The current draft of the schedule says the next task is to introduce C--, our toy source language.  I'm going to push that back a bit, because I'm in the mood for some more theoretical stuff instead.  Next time, we'll talk about variables and procedures, which turned out to be some of the most interesting and practical things in this whole project.  Stay tuned!
