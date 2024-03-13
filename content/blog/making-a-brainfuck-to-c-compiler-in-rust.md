+++
title = 'Making a Brainfuck to C Compiler in Rust'
date = 2018-03-03
tags = ['brainfuck', 'rust', 'c', 'compiler']
#description = "Making a Brainfuck to C compiler using Rust."
+++

Let’s make a tokenizer and code generator to understand the basics behind tiny compilers. After this tutorial, you will have a small Brainfuck compiler that generates very fast, portable C code.

---

I assume you already know how to use Rust. You don’t necessarily need to be able to write C to follow along, but that alone shouldn’t be too hard if you don’t already.

Why compile to C? Because every computer under the sun has a C compiler, and they’re pretty damn fast! I mean what do you expect with more than forty years of development.

## What You’ll Need
* The Rust compiler, and optionally, Cargo.
* Any C compiler. I prefer GCC, but Clang works just as well.
* At least a “good” understanding of your local command line.
* An understanding of [Brainfuck](https://en.wikipedia.org/wiki/Brainfuck#Language_design) and its (eight) instructions.

# Tokenization

Okay, before we can start slapping away at our keyboards like an 80s sci-fi, we need to make a new binary project with Cargo.

If you aren’t using Cargo, I will just assume you know good and well how to use the compiler by itself, so I’ll leave you to it.

Open your terminal and navigate to some folder where you’d like to keep this project folder. Then, “as I’m sure you already know, but I’ll explain regardless,” create the project by executing `cargo new brainfast`. If you didn’t notice, we’re calling this compiler “brainfast”, because it will generate very fast code. Duh.

Open main.rs in your favorite text editor. If you get an error when opening the file, you should certainly switch text editors, because no one else has that problem.

Let’s add a new block of code to the bottom of our file. This block will be the function that tokenizes our source. Make yours look like mine(TM):

```rust
fn tokenize(input: &str) -> Vec<Token> {
    let mut tokens = Vec::<Token>::new();
    let mut chars = input.chars();
    tokens
}
```

You’ll get an error about Token being undefined, and that’s quite alright. We just haven’t defined a token yet, but we will in the coming step.

You see that we pass back a vector of tokens. A token in this compiler is an enum. The enum will contain all of the different possible instructions, and when we read a character that matches that instruction, we push that enum value onto the vector.

This means our tokenizer reads a flat stream of characters, then outputs a flat stream of enums.

This function is not done, so let’s add another thing just below the line `let mut chars = input.chars();` :

```rust
while let Some(c) = chars.next() {
    /// ...
}
```

chars is an iterator over the characters of input . When we iterate through all the available characters, we can use a while let block.

Now, all we need to do is check if the character c matches an instruction from Brainfuck. We will accomplish this with a match, because that’s what its made to do. Add this block inside of the while let block:

```rust
match c {
    '+' => tokens.push(Add),
    '-' => tokens.push(Sub),
    '>' => tokens.push(Right),
    '<' => tokens.push(Left),
    ',' => tokens.push(Read),
    '.' => tokens.push(Write),
    '[' => tokens.push(BeginLoop),
    ']' => tokens.push(EndLoop),
    _ => {}
}
```

That’s our entire tokenizer! Your final function should be exactly:

```rust
fn tokenize(input: &str) -> Vec<Token> {
    let mut tokens = Vec::<Token>::new();
    let mut chars = input.chars();
    while let Some(c) = chars.next() {
        match c {
            '+' => tokens.push(Add),
            '-' => tokens.push(Sub),
            '>' => tokens.push(Right),
            '<' => tokens.push(Left),
            ',' => tokens.push(Read),
            '.' => tokens.push(Write),
            '[' => tokens.push(BeginLoop),
            ']' => tokens.push(EndLoop),
            _ => {}
        }
    }
    tokens
}
```

Now we will define a Token . At the top of the file, above main , add this enum:

```rust
#[derive(Debug, PartialEq, Copy, Clone)]
enum Token {
    Add,       // +
    Sub,       // -
    Right,     // >
    Left,      // <
    Read,      // ,
    Write,     // .
    BeginLoop, // [
    EndLoop,   // ]
}
use self::Token::*;
```

Great. Now you should have no errors.
Let’s talk about this enum we’ve created. It contains a value for every instruction so we can represent them all. It also has a few derives. PartialEq , because that way we can perform equality comparisons with it, and Copy and Clone , so we can toss them around while holding onto the value. And finally, Debug , so we can print all of the tokens.

We also make a use for it. We start with self , because it refers to this file. We then refer to the Token type, and finally use the wildcard "*" to mean “all”. We’re making it easier to mention a Token, by allowing one to write Add instead of Token::Add.

Now we’re ready to tokenize! Let’s add some more to main, before we test it out. Rewrite your main function to match the following:

```rust
fn main() {
    let tokens = tokenize("+-><,.[] > + - ABC , .. DE . F");
    println!("{:?}", tokens);
}
```

Now, we’re ready to test. In your command line, execute `cargo run` .

You should see something like:

```rust
[Add, Sub, Right, Left, Read, Write, BeginLoop, EndLoop, Right, Add, Sub, Read, Write, Write, Write]
```

If you put the same string as me when calling the tokenize function in main . You may have noticed that we did not pick up the spaces or the characters that were also in that string, and you’d be correct. We did not pick up those characters simply because we were not reading them.

Remember we matched only for the characters we were willing to read, so everything else is ignored.

# Code Generation

Here’s all we have to do next:

* Initialize the C source file to be ready to read and write to a tape (cells).
* Turn the tokens into instructions.
* Test the generated C program by compiling and running it.

Let’s start by creating a function that accepts tokens, and generates a C source file. The output of the function is simply a string of characters, because that’s all a file is.

```rust
fn generate(tokens: &[Token]) -> String {
    let mut output = String::new();
    
    for &token in tokens {
    }
}
```

We create an empty String called output , so that we can add our generated code to it. This string represents our output source file.

We iterate over the tokens with a for loop. The block is executed for every token in our array. This is equivalent to when we wrote while let Some(c) = chars.next() for our tokenizer, but a simple array like the value of tokens does not implement the Iterator trait.

Now we can do something like “if the token is this, then write whatever code to ‘output’”. And that is exactly what we will do.

Add the following code to the inside of your for loop block:

```rust
match token {
    Add => {
        // Increment the value at the selected cell
    }
    Sub => {
        // Decrement the value at the selected cell
    }
    Right => {
        // Change our selected cell to the next to the right
    }
    Left => {
        // Change our selected cell to the next to the left
    }
    Read => {
        // Read a single character into the selected cell
    }
    Write => {
        // Print the character at the selected cell
    }
    BeginLoop => {
        // Begin a loop at the current cell
        output_source.push_str("while () {\n");
    }
    EndLoop => {
        // Close a loop
        output_source.push_str("}\n");
    }
}
```

That was a lot! All we did was prepare to generate code depending on the type of token found. We already know (sort of) what a BeginLoop and EndLoop token should generate. They loop until the value at the selected cell is zero.

For more information about Brainfuck’s instructions and their C equivalents, visit this Wikipedia article: https://en.wikipedia.org/wiki/Brainfuck#Language_design

Before we can actually edit cells, we have to have made them! We need to create an array of 30,000 characters, then a pointer to choose between them. We would do that with the C code:

```c
char tape[30000] = {0};
char *ptr = tape;
```

Then, we can get the value at the selected cell (ptr) with *ptr. We can change the selected cell with the value of ptr, because it points to a position in the array.

= {0} means all of the 30,000 cells in the tape are initialized to zero.

If you don’t understand pointers, I don’t blame you. They can be confusing, especially at first glance. Read up on them here: https://en.wikipedia.org/wiki/Pointer_(computer_programming)

So, if we need specific C code added to the beginning of all of our compiled programs, then we could create a preface. This preface is just the C code that appears before our compiled code, regardless of the program being compiled. The same C code will be required every time.

Create a new file in the same folder as your main.rs source file. Name this preface.c .

Inside of the new C source file, add the following code (intentionally including two blank lines at the end):

```c
#include "stdio.h"
int main()
{
    char tape[30000] = {0};
    char *ptr = tape;
```

Great, now we can save and close this file. Return to our main Rust source file.

Change your generate function so that output is now initialized as our newly created C source code preface:

```rust
let mut output = String::from(include_str!("preface.c"));
```

Instead of output initially being an empty string, it is now the text of our file. include_str! is a macro that makes a string of any file’s contents you provide, at compile time.

So now, we just start generating code based on the tokens, as if we were writing c code for our program. This is better explained by finishing our code generator’s match block:

```rust
match token {
    Add => {
        // Increment the value at the selected cell
        output.push_str("++*ptr;\n");
    }
    Sub => {
        // Decrement the value at the selected cell
        output.push_str("--*ptr;\n");
    }
    Right => {
        // Change our selected cell to the next to the right
        output.push_str("++ptr;\n");
    }
    Left => {
        // Change our selected cell to the next to the left
        output.push_str("--ptr;\n");
    }
    Read => {
        // Read a single character into the selected cell
        output.push_str("*ptr=getchar();\n");
    }
    Write => {
        // Print the character at the selected cell
        output.push_str("putchar(*ptr);\n");
    }
    BeginLoop => {
        // Begin a loop at the current cell
        output.push_str("while (*ptr) {\n");
    }
    EndLoop => {
        // Close a loop
        output.push_str("}\n");
    }
}
```

Editor's note: this code is a great example of enabling unsafe memory access! Don't use this for real.

Great, our code generator is functionally complete. It generates the correct code based on the tokens. Now we need to finish the entire function off, by returning output and closing our C source code’s mainfunction.

Immediately after the for loop, push a closing brace to output, and finally, just before the closing brace, return output .

generate should now be:

```rust
fn generate(tokens: &[Token]) -> String {
let mut output = String::from(include_str!("preface.c"));
for &token in tokens {
        match token {
            Add => {
                // Increment the value at the selected cell
                output.push_str("++*ptr;\n");
            }
            Sub => {
                // Decrement the value at the selected cell
                output.push_str("--*ptr;\n");
            }
            Right => {
                // Change our selected cell to the next to the right
                output.push_str("++ptr;\n");
            }
            Left => {
                // Change our selected cell to the next to the left
                output.push_str("--ptr;\n");
            }
            Read => {
                // Read a single character into the selected cell
                output.push_str("*ptr=getchar();\n");
            }
            Write => {
                // Print the character at the selected cell
                output.push_str("putchar(*ptr);\n");
            }
            BeginLoop => {
                // Begin a loop at the current cell
                output.push_str("while (*ptr) {\n");
            }
            EndLoop => {
                // Close a loop
                output.push_str("}\n");
            }
        }
    }
    output.push_str("}\n");
    output
}
```

Great. Now we return to our main function, and bind the generated C code to a variable, before printing it:

```rust
fn main() {
    let tokens = tokenize("+-><,.[] > + - ABC , .. DE . F");
    println!("{:?}", tokens);
    let generated_code = generate(&tokens);
    println!("Generated code:\n{}", generated_code);
}
```

And that’s all there is to it. Try running it! cargo run, if you don’t remember. You should see all of the tokens printed first, then “Generated code:” on the next line, and all the C code that has been generated, filling the lines after that.

```c
#include "stdio.h"
int main()
{
    char tape[20000] = {0};
    char *ptr = tape;
*ptr++;
*ptr--;
ptr++;
ptr--;
*ptr=getchar();
putchar(*ptr);
while (*ptr) {
}
ptr++;
*ptr++;
*ptr--;
*ptr=getchar();
putchar(*ptr);
putchar(*ptr);
putchar(*ptr);
}
```

You will also notice, that we did not print very pretty C code. This is because we didn’t add code telling it how to indent when it was being generated. I won’t go over pretty printing in this article.

You can put all of the generated C code from our compiler into a C compiler like GCC or Clang, and build a binary with it.

To compile the fastest possible Brainfuck program using GCC, compile with the flag -O3. -O3 means “maximum optimizations.”

Modern compilers have so much depth, and are always becoming more complex. Even though those tools are so detailed, they act much the same. They take one input and turn it into another, through few or many phases. Ours just happens to be very simple.
