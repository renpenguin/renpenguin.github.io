---
title: The development of Chirp, a statically typed interpreted language for beginners that want to maintain their sanity
color: "#1e5085"
project: chirp
---
Programming is more accessible than ever, thanks to the extensive development of easy-to-learn, beginner-friendly languages. Unfortunately, many of these languages make trade-offs for the sake of "approachability" which come to cause intense headaches in larger projects, but wriggle their way into production all the same. This is my attempt to design and implement a headache-free alternative using the [Odin programming language](https://odin-lang.org/).

Two such beginner-friendly languages are Lua ([which I've previously discussed]({% post_url 2024-07-30-c-not-lua %})) and Python. Their easy-to-read near-pseudocode syntax makes them a great starting point for novice programmers (Python *carried* my early programming journey). However, their dynamic typing makes it easy to get erroneous values without reporting any sort of coherent error *or even reacting at all until much later in the code*. In a recent project for the [Picotron](https://www.lexaloffle.com/picotron.php), which uses Lua, I made a few typos and referenced variables that *did not exist*. The Lua compiler did **nothing**. It took a full hour to discover this as the culprit for my broken code. I would argue that dynamically typed languages make programming *harder* even than low-level language such as C.  
<small>This happened to me after my original Lua slander post, so I might post a revised version at some point soon</small>

Many modern languages are statically typed, which means that variables have a fixed type upon declaration which can't be changed to any other type without being re-declared. This way a variable is always guaranteed to have the same type that it was initialised with. Python and Lua avoid static typing for the sake of short-term simplicity, but at the cost of complete and utter mind-shattering agony should anything go wrong. In 2024, people learning to program deserve basic quality of life features like this, and shouldn't be treated like they won't understand what a type is.

I intend to design and implement a beginner-friendly programming language that doesn't sacrifice basic quality of life for short term ease-of-use. I believe that languages with a focus on this short term friendliness ultimately make it more difficult to write any meaningful project, as they are unable to offer proper errors with so little concrete information about stored variables. Thus my aim is also to provide helpful errors for developers. I'd also like to implement a more elegant version of object oriented programming and classes - Python's solution has no enforcement of attributes, which can make typos absolutely catastrophic without proper error messages, and Lua's implementation is just a cookbook for a twelve course catastrophe.

This allows me to lay out my criteria for the language:
- It must have static, implicit-where-possible typing. The user should not notice that it's there unless they try to do something wrong, but the interpreter should also never type a variable incorrectly.
- It should have clear, readable errors. If someone using the language doesn't even try to read the error, the errors have to be made more readable.
- There should be no reliance on indentation, something that trips a lot of novice programmers. We will use `{}` brackets for scope.
- It should be multi-paradigm, usable either as a procedural or object-oriented language, with OOP implemented in a clean, readable way.
- Should maintain some amount of similarity to C-like languages, so that people who start with Chirp can then move onto more common languages without too much of a jump. 
- Semi-colons should not be included. We do not need them, we will not use them.

As an extension, I'd also like to attempt the following:
- A clean way to build C libraries that can be used by the language. Python can do this, and this is why it has become so popular: it can be used to interface with much more powerful C code, through libraries like Pygame and Numpy.
- A complete graphics library, as an equivalent to Python's Turtle library.
- To compete with Lua's ease-of-implementation as a scripting language in level editors, implement Chirp as a scripting language for existing software, like a level editor or simulator.

---
## Designing the language
In the analysis stage I identified my gripes with the Lua programming language, so with the design of my own language I'm going to start by immediately fixing those issues. Arrays will start on 0. Scope will be defined using braces `{}` instead of `then` and `end`, and variables will be explicitly declared (to create a new variable you *must* use the `var` or `let` keywords) and implicitly statically typed. 

I've also decided that I want f-strings (a feature of Python introduced in 3.6 that massively simplifies string formatting) to be a core feature of the language, so I've shown below how this will look. In Chirp, lines of code will separated by a new-line character or optionally, for one liners, semi-colons. Interestingly, this first iteration of the language design is very reminiscent of Go and Swift, which means I'm able to use either language's syntax highlighting, for the most part (f-strings, for example, are not correctly highlighted):
```swift
import math, random

func main() {
    print("hi!")
    
    let dice_roll = random:range(1, 6)
    print(f"Dice roll: {dice_roll}")

    var num = 0.0
    forever {
        num += math:constants:pi / (3.0 * 4.0)
        print(f"Repeated { num } times")
        print(f"sin({ num }) = { math:sin(num) }")
    }
}

main()
```

### Memory management
Lua uses a garbage collector for memory management. A garbage collector is a subprocess of an interpreter or virtual machine that automatically deallocates any memory it believes to be no longer used. While this means that a user never has to deal with memory, such an automated system is a very dangerous game to play as certain edge cases can result in memory leaks which a user of the language will have no way of fixing, OR memory that is *still* in use being deleted and causing memory failures. The pains of garbage collection plague nearly every high-level language on Earth, so for Chirp I want to choose a more robust option. There are a few strategies I can choose from: 
- Manual Memory Management
- Simple free-on-scope-exit memory management
- Ownership and Lifetimes
- Automatic Reference Counting

**Manual memory management** is used by low level languages such as C, where the developer is expected to allocate and free memory on their own. I want my language to be easy to learn and use, so this is not an option at all.  
We could automatically free all memory when the scope it was originally created in ends, but if we have a function that returns an array, the array will be freed before it can be returned, so this will not work.  
**Ownership and Lifetimes** [are quite unique to Rust](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html), where every object in the language is owned by only one scope. Other scopes may reference objects owned by other scopes, but Rust code will only compile if those references are guaranteed to not outlive the original scope. This could be a viable option, but it often requires a lot of additional work on the user's behalf which may not be very beginner friendly.  
**Reference counting** keeps track of the number of stored pointers to a given object on the heap. When the count reaches 0, the object is freed. However, if two objects point to each other their reference counts will never reach 0, so they will never be freed. Automatic Reference counting [is used in Swift](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/), and it resolves this issue by allowing users to declare some references as "weak", which don't prevent an object from being freed and must be unwrapped to be used, in case the original object *has* been deallocated.  
For the moment, I've decided to use reference counting as it is the most concise method of memory handling with minimal and handleable tradeoff.

---
## Software development
### Tokenisation/Lexical analysis
To parse high-level code into executable instructions, we first need to convert the file with our code into tokens: symbols like `,` and `{`, keywords, string or number literals, etc. Any possible byte of the code should be encodable as a Token. Taking the Chirp example from earlier, we can pretty clearly see what different tokens need to be implemented:
- Built-in **keywords**, such as `import`, `func`, `var` and `forever`. `f` for F-strings is also implemented as a built in keyword
- **Identifiers**, effectively user defined keywords. This includes declared variable names like `num` but also external function and library names - `math`, `random:range` and `print` also count as identifiers.
- All **brackets** will be encoded as a single structure, shown in [Figure 4](#figure4). This will make them easier to resolve during Syntax Analysis.
- **Commas** will be used to separate function arguments, imported libraries and array elements.
- **Operators**, such as `+`, `-` and `&&`. Operators can be either arithmetic operators or **assignment operators**, such as `+=` or `/=`. The former will be used in expressions whereas the latter will be used in variable assignment operations.
- **New lines**, to indicate the end of a phrase.
- **Literals**, as in constant values defined in code. This includes numbers (`3.0`, `17`, `-3`), strings (`"Hello World!"`), and boolean values (`true` and `false`).

Each type of token is defined as a separate structure, and a union of all of them, the Token, is then created. The token analysis procedure will output an array of `Tokens`. Implementing something like this would be trivially easy *and* memory safe with Rust's expansive enum system, and i found that luckily, i could achieve something similar, albeit less safe, using Odin's unions (which for the most part are equivalent to C unions).
```go
Token :: union {
	Operator,
	Keyword,
	Bracket,
	Literal,
	CommaType,
	NewLineType,
}
```
Keyword, in turn, is a union which holds both identifiers and built in keywords:
```go
Keyword :: union {
	BuiltInKeyword,
	Identifier,
}
```
<span id="figure4">Figure 4:</span> The design for the bracket token data structure is shown below:
```go
Bracket :: struct {
	type: enum {
		Round,
		Square,
		Curly,
	},
	state: enum {
		Opening,
		Closing,
	},
}
```

With the Token structure implemented, I then implemented a lexer, which splits a passed string of high level code into individual tokens and output them as a `TokenStream` (a dynamic array of `Tokens`), and a "formatter", which could parse the token stream back into language code, so I could make sure that no important information was being lost during the tokenisation process. This gave me my expected output! Below is shown the manually truncated output of the program at this point. Bear in mind that the "formatted code" is reconstructed using the token list.

<div class="wrap-code"></div>

```python
[.../Project]# odin run src
------------- Parsed tokens:
["import", "math", CommaType{}, "random", NewLineType{count = 2}, "func", "main", Bracket{type = "Round", state = "Opening"}, Bracket{type = "Round", state = "Closing"}, Bracket{type = "Curly", state = "Opening"}, NewLineType{count = 1}, "print", Bracket{type = "Round", state = "Opening"}, "hi!", Bracket{type = "Round", state = "Closing"}, NewLineType{count = 1}, ...]
------------- Formatted code:
import math, random

func main() {
    print("hi!")
# -- snip --
```

Once I had the core tokeniser working I expanded on it by adding support for boolean and float literals, improving the formatter to be able to output well-formatted code even when the original was a complete unindented mess and even allowing newline and apostrophe escape characters to be used within string literals!

### Instruction Parsing/Syntax Analysis
Syntax analysis was by far the most time-consuming step of the early development process due to the sheer complexity of **expressions**. I created a function which would parse my tokens into a dynamic array of `Statements`, known as a `Block`. A Statement is a union representing one line of code including any attached lines inside `{}` brackets, such as an import/loop/if statement, variable declaration/assignment or an `Expression` representing a function/procedure call. Each statement can have `Expressions` which are a series of operations and function calls operating on variables and literals, with a single output value. For example `print(3 + math:sin(math:constants:pi / 3.0))` would be one expression which returns nothing, as `print` outputs nothing.

#### Manual memory management in Odin
At this point I also began to look into memory management, as up until this point I'd not really kept up with what memory I was and wasn't freeing. I didn't have access to a debugger, but thanks to Odin's [`TrackingAllocator`](https://odin-lang.org/docs/overview/#tracking-allocator) I was able to add a snippet to my code which would dump all the addresses that weren't yet freed at the end of the program, along with the function in which they were allocated. An example of that output is shown here:
```
=== 22 allocations not freed: ===
- 3 bytes @ odin-latest/share/core/unicode/utf8/utf8.odin(162:11)
- 5 bytes @ .../Project/src/parser/name.odin(40:26)
- 2 bytes @ odin-latest/share/core/unicode/utf8/utf8.odin(162:11)
- 112 bytes @ .../Project/src/parser/expression.odin(129:14)
- 3 bytes @ .../Project/src/parser/name.odin(40:26)
- 112 bytes @ .../Project/src/parser/expression.odin(130:14)
- 896 bytes @ .../Project/src/parser/expression.odin(102:39)
- 192 bytes @ .../Project/src/parser/statement.odin(164:3)
- 6 bytes @ .../Project/src/parser/name.odin(40:26)
- 5 bytes @ .../Project/src/parser/name.odin(40:26)
- 1536 bytes @ .../Project/src/parser/parser.odin(81:3)
```

Using this, I was able to craft procedures which would fully deallocate every part of a parsed `Block`. 

### Scope evaluation/Semantic analysis
The next step is to make sure that variables and functions referenced in the scope actually exist! To do this I created the `Scope` struct which can store `Modules` (imported libraries such as `math`, `random` and `time`), `Functions` (either interpreted or externally loaded functions like `print`), `Variables` (for variables and constants) and optionally, a reference to a "parent scope". When searching for an item, if it is not found in the current scope, the parent scope will be searched, and its parent scope will be search ad infinitum.

I then wrote the `build_scope` function, which searches for function definitions and import statements in a block and adds the function or imported modules to a `Scope`, which it then returns. At this stage I would also run a very simple evaluation of the scope, going through each instruction to make sure any referenced variables and functions were available there. If an instruction is a `VariableDefinition` statement, I add a temporary dummy variable with the same name to the block's scope, in case future statements try to access it.

### Execution
Finally I implemented basic code execution! I created a simplified example file to run so that I wouldn't have to worry about as many facets of the language:
```go
var counter = 0

forever {
	counter += 1
	print("We have now counted", counter - 1, "times")
}
```
For this simplified code block I would only need to implement variable storage, retrieval and assignment, and evaluation of expressions (i.e. the `print` function call and its arguments). At this point even interpreted function calling isn't implemented as the `print` function is built into the language and so doesn't need to worry about scope or type matching.

A bare implementation capable of operating on this simple piece of code ran without any issues! From here I properly implemented calling of imported library functions and all the possible operations. For any given operation I first evaluated either side of the operation, then attempted to match their types - if one side was an int and the other was a float, I would convert the int to a float before performing any operations.

### Fixing function scopes
I discovered that a function uses the scope it's called in as its parent scope instead of the scope it's defined in. The below code how this can be a problem.
```go
func foo() {
	print(x)
}

func bar() {
	var x = 2
	foo()
}

var x = 3

foo()
bar()
```
When `foo` is called inside `bar`, `bar` is used as the parent scope of `foo` and so when it tries to read `x` it ends up reading the value 2, from the version of `x` defined inside `bar`. Thus the output is:
```
3
2
```
To fix this, I changed the `InterpretedFunction` type to store a pointer to its parent scope on definition, which will be used as the basis for the function scope whenever it is called. After a bit of wrangling of values, i finally got it to spit out the desired result!
```
3
3
```
I even expanded the example code to make sure everything would work as intended. This example can be found [here](https://github.com/renpenguin/ChirpLang/blob/master/examples/func_scope.ch), and you can use it to test scopes in your own language if you like :)

---
## To be continued...
Hey, I hope you enjoyed reading through my development process! This was originally meant to be my programming project for school, but I ended up moving on to a different idea. There's a lot still missing, like full type checking, one-sided operators like `not`/`-` (as in negative) and even if statements XD. I'll try to implement all that eventually. In the meantime, thanks for reading and have a good rest of your day!