+++
title = 'Compiling Brainfuck to Webassembly With Go'
date = 2021-12-31
+++

Go is a minimal language that may make writing a compiler or interpreter more difficult to conceptualize than with other languages like Haskell or C++. Today I want to show you how Go can be an effective tool for learning compiler-writing, with the addition that we get to explore a little bit of what WebAssembly has to offer by writing our own compiler targeting it.

First, it is very important to understand our input language, Brainfuck, and how it works. It is the most famous esoteric programming language, first uploaded to Aminet in 1993 in an attempt to write the smallest possible compiler. Brainfuck is essentially a very basic Turing machine operating on an infinitely large one-dimensional array of bytes known as the cells or tape. The 8 operations of Brainfuck move the cursor, modify cells, and compare cell values.

```bf
These are the 8 characters/operations of all Brainfuck programs:
+     increment the cell being pointed to
-     decrement the cell being pointed to
>     move the cursor right one cell
<     move the cursor left one cell
.     print the cell's value as an ASCII character
,     read the next character input from user into cell
[     jump to matching closing bracket if cell is zero
]     jump to matching opening bracket if cell is NOT zero
The below is a multiplication of cell 1 to cell 2 resulting in 36:
++++++++ set cell 1 to 8
[        jump to closing bracket if the current cell is 0
  >      go right to cell 2
  ++++   increment four times
  <      go back to cell 1
  -      decrement cell 1
]        jump to opening bracket if the current cell is not 0
>        go to cell 2
.        prints '$'
>,.      go to cell 3; read an input; print it out
```

Brainfuck is a great contender for a first compiler because it requires very little parsing and code generation magic to get things rolling. We will be targeting WebAssembly (WASM) to generate portable and optimized Brainfuck programs that can be run on web browsers. WebAssembly is a new binary language for modern web browsers. It is smaller and executes faster than JavaScript.

WebAssembly needs JavaScript to interact with the DOM. In WebAssembly, you import and export functions, constants, and even the memory the program uses. Although WASM is a binary format which would be complicated, but very possible to directly compile to, it just makes it easier to fix code generation problems if we generate the text equivalent of WebAssembly, called WAT, or WebAssembly Text. So, let’s get familiar with the WAT syntax since that is the code our Go program is going to emit.

```wat
(module
 (export "add" (func $add))
 (func $add (param $a i32) (param $b i32) (result i32)
  (i32.add (local.get $a) (local.get $b))
 )

 (func $mul (export "mul") (param $a i32) (param $b i32) (result i32)
  local.get $a
  local.get $b
  i32.mul
 )
)
```

This is a simple WebAssembly program that exports two functions: add and mul. Both functions accept two i32 arguments and return an i32. You may notice that I have written the functions using different syntaxes. Namely, add is using the [S-expression syntax](https://webassembly.github.io/spec/core/text/instructions.html#folded-instructions). Both are valid, but S-expression style is usually preferred.

So, what’s going on here? WebAssembly is a stack-based language like Forth, where there is an implicit stack of values which you push and pop. At the end of add and mul, it is implied that we are leaving an item on the stack as our return value. That value we left was the result of calling i32.add or i32.mul. In our mul function, we push the value of $a to the stack, then the value of $b on top of it. When we subsequently call i32.mul, it pops both params off the stack, and pushes the result; $a multiplied by $b. add follows suit.

WebAssembly actually has no idea that JavaScript, or any other language for that matter, exists. WebAssembly has only 32- and 64-bit integers and floats, functions, tables, and contiguous arrays of memory. In order to share functions or data with JavaScript, you mark things as imported or exported. An exported item is local to the WebAssembly code but accessible from JavaScript. Exported items are commonly functions declared in WebAssembly that you may want to call from JavaScript. An imported item is local to JavaScript, but readable/writable from WebAssembly. Imported items are commonly global variables or memory declared in JavaScript.

```wat
(module
 (import "js" "mem" (memory 1))
 (import "console" "putChar" (func $putChar (param i32)))
 (import "console" "getChar" (func $getChar (result i32)))
 (global $cellptr (import "js" "cellptr") (mut i32))

 (export "runBrainfuck" (func $runBrainfuck))
 (func $runBrainfuck
  ;; do nothing
 )
)
```

Here we import our memory (one 64KB page), the global cell pointer value, and IO functions from JavaScript. putChar will receive a Unicode character value and print it to the webpage. getChar will return the next 8-bit character of input received by the user. These two functions are the only windows our WebAssembly program will have to the outside world, and even they are governed by how we implement them in JavaScript.

Finally, we declare and export our main function, runBrainfuck. When we call this function from JavaScript, our generated Brainfuck program will be executed. The memory we import from JavaScript will be used as our tape. We use functions like i32.store8 and i32.load8_u to write and read 8 bytes of memory to our cells.

index.html:
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Brainfuck -> WASM Example</title>
    <style>
        #container {
            font-family: monospace;
        }
        #input {
            display: block;
        }
    </style>
</head>
<body>
    <span id="container"></span>
    <input type="text" id="input" onkeypress="keyPress(event)">
    <script src="./main.js"></script>
</body>
</html>
```

main.js:
```js
var memory = new WebAssembly.Memory({initial: 1}); // Allocate 1 64KB page of memory
const cellptr = new WebAssembly.Global({value: 'i32', mutable: true})

const container = document.getElementById('container');
const input = document.getElementById('input');
var submittedInput = "Test input.\0";
var inputCharIdx = 0;

function keyPress(event) {
    if (event.keyCode === 13) {
        submittedInput = input.value + '\0';
        input.disabled = true;
    }
}

var importObject = {
    console: {
        putChar: function(ch) {
            switch (ch) {
                case 10: ch = "<br>"; break;
                case 32: ch = "&nbsp;"; break;
                default: ch = String.fromCharCode(ch);
            }
            container.innerHTML += ch;
        },
        getChar: function() {
            const result = submittedInput.charCodeAt(inputCharIdx);
            if (inputCharIdx + 1 < submittedInput.length) {
                inputCharIdx++;
            }
            return result;
        }
    },
    js: {
        mem: memory,
        cellptr,
    }
};

WebAssembly.instantiateStreaming(fetch('brainfuck.wasm'), importObject)
    .then(({instance}) => {
        instance.exports.runBrainfuck();
        console.log(new Uint8Array(memory.buffer, 0, 8)); // Print first 8 cells of memory
    });
```

Above, I included the index.html page and main.js code that serve as the interface to our WebAssembly module. In JavaScript we declare the memory and cellptr constants and include them in importObject. We also define the functions for putChar and getChar there. At the bottom, we fetch our WebAssembly module, give it our importObject, and then call runBrainfuck().

If we revisit our earlier WebAssembly template, we can see exactly how we are importing these types from our JavaScript object:

```wat
(module
 (import "js" "mem" (memory 1))
 (import "console" "putChar" (func $putChar (param i32)))
 (import "console" "getChar" (func $getChar (result i32)))
 (global $cellptr (import "js" "cellptr") (mut i32))

 (export "runBrainfuck" (func $runBrainfuck))
 (func $runBrainfuck
  ;; do nothing
 )
)
```

I hope that by now I have made it clear how WebAssembly relates to our web page and JavaScript. Now we can take a look at a Brainfuck program and how we expect it translate to WebAssembly.

```bf
+++++++++++++ 13
[>+++++<-]    times 5
>.+.+.        ABC
```

This program sets the value at the first cell to 13, then while the first cell is not empty, it adds 5 to the second cell, and decrements the first cell. The result of this operation is clearing the first cell and leaving the second cell with the result of 13 times 5. Finally, the program prints the ASCII character of 65, adds one, prints 66, adds one, and prints 67. The result is the output “ABC”. We are going to translate this program to WebAssembly to see how these essential functions will work.

Firstly, adding 13 to the first byte of memory is simple if you do it directly:

```wat
(i32.const 0)
(i32.store8 (i32.add (i32.load8_u 0) (i32.const 13)))
```

We push the byte index of the memory we want to store to, then we push the addition of the byte at cell 0 with our constant 13. After those two values are on the stack, we call i32.store8 to update the cell with the new value. We can instead use our global variable to tell us which cell we are currently pointing to:

```wat
(global.get $cellptr)
(i32.add (i32.load8_u (global.get $cellptr) (i32.const 13)))
(i32.store8)
```

*I unfolded the expression here because it was overflowing the line.*

The above is what would be produced if your compiler knew that repeating occurrences of ‘+’ and ‘-’ should be combined into one whole addition or subtraction. In other words, your compiler would produce fewer instructions if you made it optimize the input program. A more direct translation would actually be:

```wat
(global.get $cellptr)
(i32.add (i32.load8_u (global.get $cellptr) (i32.const 1)))
(i32.store8)

(global.get $cellptr)
(i32.add (i32.load8_u (global.get $cellptr) (i32.const 1)))
(i32.store8)

(global.get $cellptr)
(i32.add (i32.load8_u (global.get $cellptr) (i32.const 1)))
(i32.store8)

... ten more times
```

But for the sake of readability, we will assume that our program is optimized.

Now we need to look at looping our program. In the Brainfuck code, we looped while cell 0 was not empty. While it was not empty, we would increment our cell pointer so that it points at the next cell in memory. Then add 5 to it, go back to the first cell, subtract one, before checking if we should loop again. I implement that behavior here:

```wat
(global.get $cellptr)
(i32.store8 (i32.add (i32.load8_u (global.get $cellptr) (i32.const 13))))

(block $block0
 (br_if $block0 (i32.eq (i32.load8_u (global.get $cellptr)) (i32.const 0)))
 (loop $loop0
  (global.set $cellptr (i32.add (global.get $cellptr) (i32.const 1)))

  (global.get $cellptr)
  (i32.store8 (i32.add (i32.load8_u (global.get $cellptr) (i32.const 5))))

  (global.set $cellptr (i32.sub (global.get $cellptr) (i32.const 1)))

  (global.get $cellptr)
  (i32.store8 (i32.sub (i32.load8_u (global.get $cellptr)) (i32.const 1)))

  (br_if $loop0 (i32.ne (i32.load8_u (global.get $cellptr)) (i32.const 0)))
 )
)
```

This is actually every instruction we need to implement our compiler. The block has its own label like other functions or globals. When you use a br or br_if statement with a block’s label, it will actually exit that block entirely. This behavior is the opposite for the loop, which will repeat only upon a br or br_if statement. I wrap the loop with a block so that if the current cell is empty, it will skip the block. Otherwise, the loop runs once, the opposite condition is checked to determine if the loop should run again, and so on and so forth.

The other new instructions should be self-explanatory, but if not then please read the [WebAssembly reference](https://webassembly.github.io/spec/core/exec/index.html). It can be technical, but it is very handy!

```wat
(global.get $cellptr)
(i32.store8 (i32.add (i32.load8_u (global.get $cellptr) (i32.const 13))))

(block $block0
 (br_if $block0 (i32.eq (i32.load8_u (global.get $cellptr)) (i32.const 0)))
 (loop $loop0
  (global.set $cellptr (i32.add (global.get $cellptr) (i32.const 1)))

  (global.get $cellptr)
  (i32.store8 (i32.add (i32.load8_u (global.get $cellptr) (i32.const 5))))

  (global.set $cellptr (i32.sub (global.get $cellptr) (i32.const 1)))

  (global.get $cellptr)
  (i32.store8 (i32.sub (i32.load8_u (global.get $cellptr)) (i32.const 1)))

  (br_if $loop0 (i32.ne (i32.load8_u (global.get $cellptr)) (i32.const 0)))
 )
)

(global.set $cellptr (i32.add (global.get $cellptr) (i32.const 1)))      ;;   >

(call $putChar (i32.load8_u (global.get $cellptr)))                      ;;   .

(global.get $cellptr)
(i32.store8 (i32.add (i32.load8_u (global.get $cellptr) (i32.const 1)))) ;;   +

(call $putChar (i32.load8_u (global.get $cellptr)))                      ;;   .

(global.get $cellptr)
(i32.store8 (i32.add (i32.load8_u (global.get $cellptr) (i32.const 1)))) ;;   +

(call $putChar (i32.load8_u (global.get $cellptr)))
```

In the final part of the program, we call putChar three times while modifying the value of the first cell. Calling functions externally or internally works the exact same as using the standard instructions: you push the values that are params to the functions, and those values will be popped and what is left on the stack is the return value, if any.

Now we can take this example and compile it to WASM and try it in our browser. You may remember that earlier I mentioned WASM is the binary format, and WAT is the text format. There are many great tools you can download from The WebAssembly Binary Toolkit, but namely wat2wasm is going to be especially useful for converting our WAT files into WASM. If you want to follow along you should download the toolkit and add it to your PATH:

https://github.com/WebAssembly/wabt/releases

Now gather up the index.html and main.js files from earlier, along with this program (call it brainfuck.wat), and throw them all in one directory. Open the directory in your terminal and first execute `wat2wasm brainfuck.wat -o brainfuck.wasm` . The output file name “brainfuck.wasm” is important because that’s the exact file JavaScript will be looking for. Now, you should start a webserver on the project directory. An easy way is with Python: `python -m http.server` .

Now assuming all went well you can open a browser and go to localhost:8000 and see the output of the Brainfuck program! So, let’s write our compiler in Go so we don’t have to hand-translate Brainfuck programs anymore. Tip: use Ctrl + F5 when refreshing your webpage, because it will force refresh on the cached assets.

Begin with the standard setup:

main.go:
```go
package main

func main() {
}
```

And we will start by parsing command-line arguments and preparing the input file by reading it to a string. Our compiler will have a single flag: -o which sets the output filename. The program will always expect the input file to follow the arguments. Therefore, our usage is `brainfuck2wasm [options] input_file` or when developing: `go run main.go [options] input_file` .

The following is the program that does this:

```go
package main

import (
	"flag"
	"fmt"
	"io/ioutil"
	"os"
	"path"
	"strings"
)

var outFile = flag.String("out", "", "output file")

func main() {
	flag.Parse()

	if flag.NArg() != 1 {
		fmt.Fprintln(os.Stderr, "error: expected an input file")
		os.Exit(1)
	}
	fmt.Println(flag.Arg(0))
	bytes, err := ioutil.ReadFile(flag.Arg(0))
	if err != nil {
		fmt.Fprintln(os.Stderr, "error: failed to read file")
		os.Exit(1)
	}

	if *outFile == "" {
		// The outFile is the input file with a .wat extension instead
		*outFile = flag.Arg(0)
		ext := path.Ext(*outFile)
		*outFile = strings.Replace(*outFile, ext, ".wat", 1)
	}

	f, err := os.Create(*outFile)
	if err != nil {
		fmt.Fprintln(os.Stderr, "error: unable to open outfile")
		os.Exit(1)
	}
	defer f.Close()

	// generate program and write it to f here
}
```

Now we need to write our compiled program to our output file f. To start, we will need a template for our output program. For that we’ll use a constant:

```go
const watOutputFormat = `(module
 (import "js" "mem" (memory 1))
 (import "console" "putChar" (func $putChar (param i32)))
 (import "console" "getChar" (func $getChar (result i32)))
 (global $cellptr (import "js" "cellptr") (mut i32))

 (export "runBrainfuck" (func $runBrainfuck))
 (func $runBrainfuck
%s )
)
`
```

And a function to generate the instructions:

```go
	// ... the rest of main()
	fmt.Fprintf(f, watOutputFormat, generateInstructions(string(bytes)))
}

func generateInstructions(code string) (out string) {
	for _, r := range code {
		switch r {
		case '>':
			// out += "generated instructions"
		case '<':
		case '+':
		case '-':
		case '.':
		case ',':
		case '[':
		case ']':
		}
	}
	return
}
```

Now it is possible to map each Brainfuck character to equivalent WAT instructions. As we’ve already talked about how we write equivalent code to Brainfuck programs, we can fill in the easy instructions:

```go
func generateInstructions(code string) (out string) {
	for _, r := range code {
		switch r {
		case '>':
			out += "(i32.add (global.get $cellptr) (i32.const 1))\n(global.set $cellptr)\n"
		case '<':
			out += "(i32.sub (global.get $cellptr) (i32.const 1))\n(global.set $cellptr)\n"
		case '+':
			out += "(global.get $cellptr)\n(i32.store8 (i32.add (i32.load8_u (global.get $cellptr)) (i32.const 1)))\n"
		case '-':
			out += "(global.get $cellptr)\n(i32.store8 (i32.sub (i32.load8_u (global.get $cellptr)) (i32.const 1)))\n"
		case '.':
			out += "(i32.load8_u (global.get $cellptr))\n(call $putChar)\n"
		case ',':
			out += "(i32.store8 (global.get $cellptr) (call $getChar))\n"
		case '[':
		case ']':
		}
	}
	return
}
```

Now this is a functional compiler for Brainfuck programs that do not contain loops. Loops are a little trickier because we use generated labels for them, so we have to keep track of their label names. There are two specific things we need to keep track of for loops:

* The next label number, because every label is unique. $label$0, $label$1, etc.
* The label number (an index) for the current loop depth. Because sequential loops increment this index, the depth is not always going to be equal to it. It’s an easy problem to solve using a map[int]int where the key is the loop depth of the current nested loop, and the value is the label index.

The following implements loops using the above points:

```go
func generateInstructions(code string) (out string) {
	labelIdx := 0
	loopDepth := 0
	loopDepthLabelIdxs := make(map[int]int)
	for _, r := range code {
		switch r {
		case '>':
			out += "(i32.add (global.get $cellptr) (i32.const 1))\n(global.set $cellptr)\n"
		case '<':
			out += "(i32.sub (global.get $cellptr) (i32.const 1))\n(global.set $cellptr)\n"
		case '+':
			out += "(global.get $cellptr)\n(i32.store8 (i32.add (i32.load8_u (global.get $cellptr)) (i32.const 1)))\n"
		case '-':
			out += "(global.get $cellptr)\n(i32.store8 (i32.sub (i32.load8_u (global.get $cellptr)) (i32.const 1)))\n"
		case '.':
			out += "(i32.load8_u (global.get $cellptr))\n(call $putChar)\n"
		case ',':
			out += "(i32.store8 (global.get $cellptr) (call $getChar))\n"
		case '[':
			out += fmt.Sprintf("(block $label$%d\n", labelIdx)
			out += fmt.Sprintf("(br_if $label$%d (i32.eq (i32.load8_u (global.get $cellptr)) (i32.const 0)))\n", labelIdx)

			labelIdx++ // Block and Loop sections use different labels
			out += fmt.Sprintf("(loop $label$%d\n", labelIdx)
			loopDepthLabelIdxs[loopDepth] = labelIdx // loopDepthLabelIdxs keep records of the labels for Loop sections

			labelIdx++
			loopDepth++
		case ']':
			loopDepth--
			out += fmt.Sprintf("(br_if $label$%d (i32.ne (i32.load8_u (global.get $cellptr)) (i32.const 0)))\n)\n)\n", loopDepthLabelIdxs[loopDepth])
		}
	}
	return
}
```

When a loop is started, those labels are created distinctly for the block and loop. We add a key, value pair to loopDepthLabelIdxs (for lack of a better name) of loopDepth, labelIdx. For any given depth of a loop we are generating, we can know what the correct label index is to use. Then we just increment the label index and loop depth before finishing.

When a loop is being closed (or rather determining whether to loop again), we decrement the loop depth, and generate the “loop again, if” instruction. We don’t need to delete keys from the map because they *should* always be overwritten if a loop has been opened correctly, in a valid program. I will leave it up to reader challenge to make the compiler report errors on invalid programs.

Now what we have is a fully functional Brainfuck compiler, that generates WebAssembly Text programs. From here, you’re basically done with the article, so I’ll present the challenges:

* Implement compile errors so users know their programs are invalid, and why they’re invalid.
* Read some of the WebAssembly spec and make it so that when a user reaches the end of their memory, it expands. Brainfuck is only Turing-complete on an infinite size of memory, so it’s cool to do our best.
* Optimize the generated program. If you’re not sure where to start, consider that reading and writing memory is probably the slowest. Ideally, multiple writes to the same cell in memory should be grouped into one. Consider keeping a variable when a cell is read several times. There are countless opportunities for optimization here.

As a bonus to your completed compiler, I will throw in indentation. Essentially you just pre-define how much a line should be indented, and multiply that by nested depth:

```go
func generateInstructions(code string) (out string) {
	labelIdx := 0
	loopDepth := 0
	loopDepthLabelIdxs := make(map[int]int)
	indentLevel := 0
	for _, r := range code {
		indentS := indent(2 + indentLevel)
		switch r {
		case '>':
			out += fmt.Sprintf("%s(i32.add (global.get $cellptr) (i32.const 1))\n%[1]s(global.set $cellptr)\n", indentS)
		case '<':
			out += fmt.Sprintf("%s(i32.sub (global.get $cellptr) (i32.const 1))\n%[1]s(global.set $cellptr)\n", indentS)
		case '+':
			out += fmt.Sprintf("%s(global.get $cellptr)\n%[1]s(i32.store8 (i32.add (i32.load8_u (global.get $cellptr)) (i32.const 1)))\n", indentS)
		case '-':
			out += fmt.Sprintf("%s(global.get $cellptr)\n%[1]s(i32.store8 (i32.sub (i32.load8_u (global.get $cellptr)) (i32.const 1)))\n", indentS)
		case '.':
			out += fmt.Sprintf("%s(i32.load8_u (global.get $cellptr))\n%[1]s(call $putChar)\n", indentS)
		case ',':
			out += fmt.Sprintf("%s(i32.store8 (global.get $cellptr) (call $getChar))\n", indentS)
		case '[':
			out += fmt.Sprintf("%s(block $label$%[2]d\n", indentS, labelIdx)
			blockIndentS := indent(3 + indentLevel)
			out += fmt.Sprintf("%s(br_if $label$%[2]d (i32.eq (i32.load8_u (global.get $cellptr)) (i32.const 0)))\n", blockIndentS, labelIdx)

			labelIdx++ // Block and Loop sections use different labels
			out += fmt.Sprintf("%s(loop $label$%d\n", blockIndentS, labelIdx)
			loopDepthLabelIdxs[loopDepth] = labelIdx // loopDepthLabelIdxs keep records of the labels for Loop sections

			labelIdx++
			loopDepth++
			indentLevel += 2
		case ']':
			loopDepth--
			loopIndentS := indent(2 + indentLevel)      // Parent block indent plus the loop's indent
			blockIndentS := indent(2 + indentLevel - 1) // Just parent block indent
			out += fmt.Sprintf("%s(br_if $label$%[4]d (i32.ne (i32.load8_u (global.get $cellptr)) (i32.const 0)))\n%[2]s)\n%[3]s)\n", loopIndentS, blockIndentS, indent(indentLevel), loopDepthLabelIdxs[loopDepth])
			indentLevel -= 2 // We exited the Loop and Block sections
		}
	}
	return
}

func indent(level int) string {
	return strings.Repeat(" ", level)
}
```

Thank you for taking the time to learn about my latest project. All of the code is on GitHub along with a short readme of how to use the project if you need some help or a reminder. I encourage you to fork and attempt the challenges!

https://github.com/fivemoreminix/brainfuck2wasm
