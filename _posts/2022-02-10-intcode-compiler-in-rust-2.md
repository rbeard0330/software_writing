---
layout: compiler_post
date: 2022-02-10
title: "Writing an Intcode Compiler in Rust (Part 2) - The Runtime"
---

In [our last post](/writing/2022-01-31-intcode-compiler-in-rust-1.md), we talked about the Intcode virtual machine and designed a low-level IR to represent Intcode in a human-readable form.  In this post, we'll put together an interpreter to execute compiled Intcode programs.

## The Interpreter

The core of our interpreter is a struct that represents the virtual machine:

```rust
#[derive(Debug)]
pub struct GenericIntcoder<Tape> where Tape: Index<usize, Output=Int> + IndexMut<usize> {
    tape: Tape,
    pub position: usize,
    relative_base: usize,
    inputs: Vec<Int>,
    initial_data: Vec<Int>,
}

type Intcoder = GenericIntcoder<[Int; TAPE_LENGTH]>;
const TAPE_LENGTH: usize = 50_000;
type Int = i32;
```

This struct is ultimately pretty simple, but it looks intimidating because of the generics.  In essence, the struct maintains a tape of some type, which is the "memory" for our program.  The generic constraints require that the tape's type be one that allows us to index it with `usize`s to set and retrieve `Int`s, which are an alias for `i32`s.  We declare an alias `Intcoder` for the concrete version of the generic struct where the storage is provided by a fixed-length array.  Using generics here adds some boilerplate (as you will soon see), but it makes it possible to adapt the program to use a dynamically sized `Vec` or a sparse data structure, which is required for one of the Advent of Code problems.

In addition to the tape, the struct tracks the position of the current instruction and the relative base.  It can also store inputs to be fed into the program when needed.  This will allow Intcode programs to run in a non-interactive mode by supplying the necessary inputs up front.  Finally, the struct remembers its initial state, which allows it to be reset conveniently.

Initializing a new `Intcoder` is straightforward.  The constructor expects an iterable of integers:

```rust
impl Intcoder {
    pub fn new<T>(data: T) -> Intcoder
        where
            T: IntoIterator<Item = Int>,
    {
        let mut result = GenericIntcoder {
            tape: [0; TAPE_LENGTH],
            position: 0,
            relative_base: 0,
            inputs: Vec::new(),
            initial_data: data.into_iter().collect::<Vec<_>>(),
        };
        result.reset();
        result
    }
}
```

The call to reset copies the data from `initial_data` to the tape.  Note that this `impl` block is not generic, because it needs to initialize a concrete instance of the tape.

For convenience, we will also add an alternative constructor that accepts a comma-separated string of int values:
```rust
pub fn from_int_str<S>(data: S) -> Result<Intcoder, ParseIntError>
    where
        S: AsRef<str>,
    {
        let ints = data.as_ref()
            .split(',')
            .map(|value| value.parse::<Int>())
            .collect::<Result<Vec<_>, ParseIntError>>()?;
        Ok(Self::new(ints))
    }
```

To run the Intcode program, we define a core method `tick`, which advances the program by one step.  A single tick can produce a variety of results: a value to output, a request for input, the end of the program, or nothing at all.  To represent these possibilities, `tick` has the following signature:

```rust
fn tick(&mut self) -> Option<Interrupt> {
    todo!()
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum Interrupt {
    Halt(Int),
    InputRequired,
    Output(Int),
    InvalidOpcode(Int),
}
```

We can then build on the `tick` method to create a practical way to run an Intcode program in chunks:
```rust
pub fn run(&mut self) -> Interrupt
{
    loop {
        match self.tick() {
            Some(Interrupt::Output(v)) => println!("Output: {}", v),
            Some(interrupt) => return interrupt,
            _ => {}
        }
    }
}
```
This method calls `tick` repeatedly until an interrupt is thrown.  Output interrupts are handled by simply logging the output, while other outputs are to be handled by the caller.  Note that our implementation of `tick` will handle `InputRequired` interrupts if it has a stored input to provide.  This allows a client to initialize an `Intcoder`, provide whatever inputs will be required, then run the program to completion with a single call to `run`.

All that's left is to implement `tick`.  First, we define some helper methods to handle reads and writes to the tape:

```rust
pub fn read_offset(&self, offset: usize) -> Int {
    self.read_absolute(self.position + offset)
}

pub fn read_absolute(&self, position: usize) -> Int {
    self.tape[position]
}

pub fn write_offset(&mut self, offset: usize, value: Int) {
    self.write_absolute(self.position + offset, value)
}

pub fn write_absolute(&mut self, position: usize, value: Int) {
    self.tape[position] = value;
}
```

We then use these to define `load` and `store` operations:

```rust
fn load_from(&self, offset: usize) -> Int {
    let target = self.read_offset(offset);
    self.read_absolute(target as usize)
}

fn load(&self, offset_to_destination: usize, mode: Mode) -> Int {
    match mode {
        Mode::Immediate => self.read_offset(offset_to_destination),
        Mode::Position => self.load_from(offset_to_destination),
        Mode::Relative => self.load_from(self.relative_base + offset_to_destination),
    }
}

fn store(&mut self, offset_to_destination: usize, mode: Mode, value: Int) {
    let destination = self.read_offset(offset_to_destination) as usize;
    match mode {
        Mode::Immediate => panic!("Attempted to store in immediate mode"),
        Mode::Position => self.write_absolute(destination, value),
        Mode::Relative => self.write_absolute(self.relative_base + destination, value),
    }
}
```

The `load` and `store` operation expect to receive the offset to a parameter from the relevant instruction together with an addressing mode and, for a store, the value to store.

Now we can start writing our `tick` method, which consists of a single `match` expression against the value at the current position on the tape:
```rust
match self.read_offset(0) {
    99 => {
        Some(Interrupt::Halt( self.read_absolute(0)))
    }
    inst if inst % 100 == 1 => {
        let mut modes = Modes::new(inst);
        let [v1, v2] = read_args(self, &mut modes);
        self.store(3, modes.next().unwrap(), v1 + v2);
        self.position += 4;
        None
    }
    // Other instructions omitted
}
```
The special halt instruction is easy to handle.  For other instructions, we use guard clauses to match on the last two digits of the instruction, which are the opcode.  The snippet above handles an opcode of 1, which is `ADD`.  The first step is to extract the addressing modes from the instruction.  `Modes` is a simple iterator struct.  When constructed, it discards the final two digits of its input.  Its `next` method simply yields and discards the final digit of its input, allowing us to easily cycle over the addressing modes of each parameter.

Addressing modes in hand, we read the arguments we need for the instruction, which we do with a helper function:
```rust
pub fn read_args<Tape, const N: usize>(coder: &mut GenericIntcoder<Tape>, modes: &mut Modes) -> [Int; N]
    where Tape: Index<usize, Output=Int> + IndexMut<usize>
{
    let mut result = [0; N];
    (0..N).into_iter().for_each(|i| result[i] = coder.load(i + 1, modes.next().unwrap()));
    result
}
```
The `Tape` type parameter in the function signature is just noise, but the `N` parameter is important.  Notice that the function does *not* require the caller to pass the number of arguments to read a parameter.  Instead, the compiler *infers* the number of arguments needed to match the requested return type.  So when we write:
```rust
let [v1, v2] = read_args(self, &mut modes);
```
the compiler sees that it needs a function returning a size-2 array and generates an appropriate version of the function.  But we can also write:
```rust
let [value] = read_args(self, &mut modes);
```
to process an output instruction, and the compiler detects that it should only read 1 argument.

Once the arguments are read, we do the necessary logic, store the result, and adjust the current position to the next instruction.  The other instructions are handled similarly.

## A CLI for our compiler

Now that we have some executable code, it's a good time to talk about how it will run.  For simplicity, we're going to have a single binary that will bundle both the compiler and the interpreter.  Command-line arguments will allow the user to determine whether the program compiles a source program to intcode, runs an already-compiled program, or both.

The code for the binary is fairly dull, and relies heavily on the excellent [clap](https://docs.rs/clap/latest/clap/) crate for creating CLI tools.  Here's the help-text generated by `clap`:

```
USAGE:
    icc.exe [OPTIONS] <SOURCE> <DEST>

ARGS:
    <SOURCE>    File to compile or run. File format will be inferred from extension
    <DEST>      Destination directory for compiled output. Filename will be generated from input filename

OPTIONS:
    -e, --emit <EMIT>    Code to emit [possible values: all, none, intcode, llir]
    -h, --help           Print help information
    -r, --run            Set to run the program after compilation
```

## Our First Program Run

Now that we have an interpreter, we're ready write and execute a real program!  Consider the following simple program, written in LLIR, which takes an input, multiplies it by 10, and outputs the result:
```
IN &a
IMUL #a 10
OUT &a
```
We can hand-compile this to:
```
  0(03) &a
010(02) #a 10 &a  
  1(04) &a
```
`a` ends up being at position 3, so the final code is:
```
  0(03) 3
011(02) 0 10 3  
  1(04) 3
```
which uglifies to: `3,3,1102,0,10,3,4,3,99`.  We then stick this code into a text file and run the CLI:

```
$ cargo run --bin icc -- -r ints/input_x_10.ints compiled
   Compiling compiler v0.1.0 (C:\Code\compiler)
    Finished dev [unoptimized + debuginfo] target(s) in 1.09s
     Running `target\debug\icc.exe -r ints/input_x_10.ints compiled`
Input required
-81
Output: -810
```
Success!

In the next entry, we'll work on automating the compilation from LLIR to Intcode.
