+++
title = 'Understanding Compilers — for Humans v2'
# date = 2018-10-29T16:04:54-05:00
date = 2018-10-29
+++
Note about this post: this was copied from the [original Medium article](https://towardsdatascience.com/understanding-compilers-for-humans-version-2-157f0edb02dd) on the 11th of March, 2024. Some images and content were lost to time.

![Cover Image](cover.webp)

Understanding your compiler internally allows you to use it effectively. Walk through how programming languages and compilers work in this chronological synopsis. Lots of links, example code, and diagrams have been composed to aid in your understanding.

---

## Author’s Note

*Understanding Compilers — For Humans (Version 2)* is a successor to my second article on Medium, with over 21 thousand views. I am so glad I could make a positive impact on people’s education, and I am excited to bring **a complete rewrite based on the input I received from the original article.**

https://medium.com/@fivemoreminix/understanding-compilers-for-humans-ba970e045877?source=post_page-----157f0edb02dd--------------------------------

I chose Rust as this work’s primary language. It is verbose, efficient, modern, and seems, by design, to be really simple for making compilers. I enjoyed using it. https://www.rust-lang.org/

This article is written for the goal of keeping the reader’s attention, and to not have 20 pages of mind numbing reading. There are many links in the text that will guide you to resources that go deeper on topics that intrigue you. Most links direct you to Wikipedia.

Feel free to drop any questions or suggestions in the comment section at the bottom. Thank you for your interest, and I hope you enjoy.

---

## Introduction
### What is a Compiler

**In summary, what you may call a programming language is really just software, called a compiler, that reads a text file, processes it a lot, and generates binary.** Since a computer can only read 1s and 0s, and humans write better Rust than they do binary, compilers were made to turn that human-readable text into computer-readable machine code.

A compiler can be any program that translates one text into another. For example, here is a compiler written in Rust that turns 0s into 1s, and 1s into 0s:

```rust
// An example compiler that turns 0s into 1s, and 1s into 0s.
 
fn main() {
    let input = "1 0 1 A 1 0 1 3";
    
    // iterate over every character `c` in input
    let output: String = input.chars().map(|c|
        if c == '1' { '0' }
        else if c == '0' { '1' }
        else { c } // if not 0 or 1, leave it alone
    ).collect();
    
    println!("{}", output); // 0 1 0 A 0 1 0 3
}
```

While this compiler doesn’t read a file, doesn’t generate an AST, and doesn’t produce binary, it is still considered a compiler for the simple reason that it translates an input.

### What a Compiler Does

In short, compilers take source code and produce binary. Since it would be pretty complicated to go straight from complex, human readable code to ones and zeros, compilers have several steps of processing to do before their programs are runnable:

1. Reads the individual characters of the source code you give it.
2. Sorts the characters into words, numbers, symbols, and operators.
3. Takes the sorted characters and determines the operations they are trying to perform by matching them against patterns, and making a tree of the operations.
4. Iterates over every operation in the tree made in the last step, and generates the equivalent binary.

While I say the compiler immediately goes from a tree of operations to binary, it actually generates assembly code, which is then assembled/compiled into binary. Assembly is like a higher-level, human-readable binary. Read more about what assembly is [here](https://en.wikipedia.org/wiki/Assembly_language).

![A flowchart showing the process of a compiler: the input is sent to the lexer, which produces tokens, which are fed into the parser to produce an abstract syntax tree, which gets fed through the code generator to produce the final program.](compiler_process.webp)

### What is an Interpreter

[Interpreters](https://en.wikipedia.org/wiki/Interpreter_(computing)) are much like compilers in that they read a language and process it. Though, **interpreters skip code generation and execute the AST [just-in-time](https://en.wikipedia.org/wiki/Just-in-time_compilation).** The biggest advantage to interpreters is the time it takes to start running your program during debug. A compiler may take anywhere from a second to several minutes to compile a program before execution, while an interpreter begins executing immediately, with no compilation. The biggest downside to an interpreter is that it requires to be installed on the user’s computer before the program can be executed.

![A flowchart depicting the process of an interpreter: first the input is read into the lexer which produces tokens, and the tokens are fed into the parser to produce an abstract syntax tree, and finally that is interpreted by walking the tree.](interpreter_process.webp)

*This article refers mostly to compilers, but it should be clear the differences between them and how compilers relate.*

## Lexical Analysis

The first step is to split the input up character by character. This step is called [lexical analysis](https://en.wikipedia.org/wiki/Lexical_analysis), or tokenization. The major idea is that we group characters together to form our words, identifiers, symbols, and more. Lexical analysis mostly does not deal with anything logical like solving 2+2 — it would just say that there are three tokens: a number: 2, a plus sign, and then another number: 2.

Let’s say you were lexing a string like 12+3: it would read the characters 1, 2, +, and 3. We have the separate characters but we must group them together; one of the major tasks of the tokenizer. For example, we got 1 and 2 as individual letters, but we need to put them together and parse them as a single integer. The + would also need to be recognized as a plus sign, and not its literal character value — the character code 43.

![Tokens of the expression `foo(bar*12);`](tokenization.webp)

If you can see code and make more meaning of it that way, then the following Rust tokenizer can group digits into 32-bit integers, and plus signs as the Token value Plus.

https://play.rust-lang.org/?gist=070c3b6b985098a306c62881d7f2f82c&version=stable&mode=debug&edition=2015&source=post_page-----157f0edb02dd--------------------------------

You can click the “Run” button at the top left corner of the Rust Playground to compile and execute the code in your browser.

In a compiler for a programming language, the lexer may need to have several different types of tokens. For example: symbols, numbers, identifiers, strings, operators, etc. It is entirely dependent on the language itself to know what kind of individual tokens you would need to extract from the source code.

## Parsing

The parser is truly the heart of the syntax. The parser takes the tokens generated by the lexer, attempts to see if they’re in certain patterns, then associates those patterns with expressions like calling functions, recalling variables, or math operations. The parser is what literally defines the syntax of the language.

The difference between saying `int a = 3` and `a: int = 3` is in the parser. The parser is what makes the decision of how syntax is supposed to look. It ensures that parentheses and curly braces are balanced, that every statement ends with a semicolon, and that every function has a name. The parser knows when things aren’t in the correct order when tokens don’t fit the expected pattern.

There are several different types of parsers that you can write. One of the most common is a top-down, recursive-descent parser. Recursive-descent parsing is one of the simplest to use and understand. All of the parser examples I created are recursive-descent based.

The syntax a parser parses can be outlined using a grammar. A grammar like EBNF can describe a parser for simple math operations like 12+3.

(Image lost to time)

Remember that the grammar file is not the parser, but it is rather an outline of what the parser does. You build a parser around a grammar like this one. It is to be consumed by humans and is simpler to read and understand than looking directly at the code of the parser.

The parser for that grammar would be the expr parser, since it is the top-level item that basically everything is related to. The only valid input would have to be any number, plus or minus, any number. expr expects an additive_expr, which is where the major addition and subtraction appears. additive_expr first expects a term (a number), then plus or minus, another term.

![A diagram showing the AST produced for parsing the expression `12+3`](ast.webp)

The tree that a parser generates while parsing is called the abstract syntax tree, or AST. The AST contains all of the operations. The parser does not calculate the operations, it just collects them in their correct order.

I added onto our lexer code from before so that it matches our grammar and can generate ASTs like the diagram. I marked the beginning and end of the new parser code with the comments // BEGIN PARSER // and // END PARSER //.

https://play.rust-lang.org/?gist=205deadb23dbc814912185cec8148fcf&version=stable&mode=debug&edition=2015&source=post_page-----157f0edb02dd--------------------------------

We can actually go much further. Say we want to support inputs that are just numbers without operations, or adding multiplication and division, or even adding precedence. This is all possible with a quick change of the grammar file, and an adjustment to reflect it inside of our parser code.

(Image lost to time)

https://play.rust-lang.org/?gist=1587a5dd6109f70cafe68818a8c1a883&version=nightly&mode=debug&edition=2018&source=post_page-----157f0edb02dd--------------------------------

![A diagram showing tokenization and parsing from Wikipedia](wikipedia_parsing.gif)

Scanner (a.k.a. lexer) and parser example for C. Starting from the sequence of characters "if(net>0.0)total+=net*(1.0+tax/100.0);", the scanner composes a sequence of tokens, and categorizes each of them, e.g. as identifier, reserved word, number literal, or operator. The latter sequence is transformed by the parser into a syntax tree, which is then treated by the remaining compiler phases. The scanner and parser handles the regular and properly context-free parts of the grammar for C, respectively. Credit: Jochen Burghardt. [Original](https://commons.wikimedia.org/wiki/File:Xxx_Scanner_and_parser_example_for_C.gif).

## Generating Code

The [code generator](https://en.wikipedia.org/wiki/Code_generation_(compiler)) takes an AST and emits the equivalent in code or assembly. The code generator must iterate through every single item in the AST in a recursive descent order — much like how a parser works — and then emit the equivalent, but in code.

https://godbolt.org/z/K8416_?source=post_page-----157f0edb02dd--------------------------------

If you open the above link, you can see the assembly produced by the example code on the left. Lines 3 and 4 of the assembly code show how the compiler generated the code for the constants when it encountered them in the AST.

The Godbolt Compiler Explorer is an excellent tool and allows you to write code in a high level programming language and see its generated assembly code. You can fool around with this and see what kind of code should be made, but don’t forget to add the optimization flag to your language’s compiler to see just how smart it is. (-O for Rust)

If you are interested in how a compiler saves a local variable to memory in ASM, [this article](https://norasandler.com/2018/01/08/Write-a-Compiler-5.html) (section “Code Generation”) explains the stack in thorough detail. Most times, advanced compilers will allocate memory for variables on the heap and store them there, instead of on the stack, when the variables are not local. You can read more about storing variables in [this StackOverflow answer](https://stackoverflow.com/a/18446414).

Since assembly is an entirely different, complicated subject, I won’t talk much more about it specifically. I just want to stress the importance and work of the code generator. Furthermore, a code generator can produce more than just assembly. The Haxe compiler has a backend that can generate over six different programming languages; including C++, Java, and Python.

Backend refers to a compiler’s code generator or evaluator; therefore, the front end is the lexer and parser. There is also a middle end, which mostly has to do with optimizations and IRs explained later in this section. The back end is mostly unrelated to the front end, and only cares about the AST it receives. This means one could reuse the same backend for several different front ends or languages. This is the case with the notorious GNU Compiler Collection.

I couldn’t have a better example of a code generator than my C compiler’s backend; you can find it [here](https://github.com/fivemoreminix/ox/blob/master/src/generator.rs).

After the assembly has been produced, it would be written to a new assembly file (.s or .asm). That file would then be passed through an assembler, which is a compiler for assembly, and would generate the equivalent in binary. The binary code would then be written to a new file called an object file (.o).

Object files are machine code but they are not executable. For them to become executable, the object files would need to be linked together. The linker takes this general machine code and makes it an executable, a shared library, or a static library. More about linkers [here](https://en.wikipedia.org/wiki/Linker_(computing)#Overview).

Linkers are utility programs that vary based on operating systems. A single, third-party linker should be able to compile the object code your backend generates. There should be no need to create your own linker when making a compiler.

![A diagram depicting a library and two object files being linked into a library, DLL, and executable file.](linking.webp)

A compiler may have an intermediate representation, or IR. An IR is about representing the original instructions losslessly for optimizations or translation to another language. An IR is not the original source code; the IR is a lossless simplification for the sake of finding potential optimizations in the code. Loop unrolling and vectorization are done using the IR. More examples of IR-related optimizations can be found in [this PDF](http://www.keithschwarz.com/cs143/WWW/sum2011/lectures/140_IR_Optimization.pdf).

## Conclusion

When you understand compilers, you can work more efficiently with your programming languages. Maybe someday you would be interested in making your own programming language? I hope this helped you.

### Resources & Further Reading

* http://craftinginterpreters.com/ — guides you through making an interpreter in C and Java.
* https://norasandler.com/2017/11/29/Write-a-Compiler.html — probably the most beneficial “writing a compiler” tutorial for me.
* My C compiler and scientific calculator parser can be found [here](https://github.com/asmoaesl/ox) and [here](https://github.com/asmoaesl/rsc).
* An example of another type of parser, called a precedence climbing parser, can be found [here](https://play.rust-lang.org/?gist=d9db7cfad2bb3efb0a635cddcc513839&version=stable&mode=debug&edition=2015). Credit: Wesley Norris.
