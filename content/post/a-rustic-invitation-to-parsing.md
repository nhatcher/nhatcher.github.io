---
title: "A Rustic invitation to parsing"
date: 2023-04-20T11:08:43+02:00
Description: ""
Tags: []
Categories: []
DisableComments: false
---

(A version of this post appeared first in the [EqualTo blog](https://www.equalto.com/blog/a-rustic-invitation-to-parsing))

Here we will show you how to build a simple parser. Keep reading to learn about algorithms, lexers, parsers and a little bit of Rust. And find all the code in our [GitHub repository](https://github.com/nhatcher/sample-desk-calcualtor).

## Introduction. The theory of parsing

At my previous company we built a spreadsheet engine from scratch in the Rust programming language. We can compile it to WebAssembly and run it in the browser or compile it to machine code and run it headless on the server. The first step in writing such an engine would be to write an Excel formula parser. Since writing a full fledged Excel formula parser is a daunting task we will split across multiple articles. In this first instalment today we will show you how to build a parser and interpreter for a desk calculator.

### Parsing essentials

Parsing serves as the front end of a compiler and it is considered a solved problem today. Its primary purpose is to transform a human readable string into an object, the abstract syntax tree, that a computer can understand and process.

If you are creating a new programming language you will need to create a parser.

In general, there are two main approaches to parsing: you can either use a *parser generator* (also known as a "compiler compiler") or create a **handmade parser** from scratch. Times have changed, and while 30 years ago the popular choice was the parser generator, today most folks would create a handmade parser from scratch.

Parser generators typically follow a bottom-up, non-recursive algorithm. They usually run in linear time and linear space, and are table-driven. The classic example is [Bison](https://en.wikipedia.org/wiki/GNU_Bison). There are many others, such as the famous [ANTLR](https://www.antlr.org/), which actually uses a top-down recursive algorithm. With these tools, you only need to specify the grammar for your language, and they will generate the code for your parser.

There are some disadvantages to parser generators:

1. They introduce a dependency on the parser generator in your build system.
2. Since the generated code may not conform to your coding standards, you will likely need to treat it as an external dependency, further complicating your build process.
3. You'll probably have to make compromises on your grammar to make it compatible with the parser generator.

Handmade parsers also have their drawbacks, as they're more susceptible than parser generators to errors and performance issues. Errors are more likely to occur in your code than in your grammar, although you can mitigate risks with thorough testing. On the other hand, it is easier to handle unusual corner cases in your language with a handwritten parser, which offers greater flexibility. Moreover, error reporting and recovery are much simpler with handwritten parsers.

In general, if you are writing a "small" language, I prefer a handmade parser. However, if you are working with a complex language, have time constraints, and have multiple contributors to the project, a parser generator might be the better option. These days, Bison is my parser generator of choice. [PEG](https://en.wikipedia.org/wiki/Parsing_expression_grammar) is waiting in the wings, but is yet to take the crown.

One final thing about handmade parsers... they are a lot of fun! :)

In this next section we will build a calculator using a Pratt parser. There is a fair amount of theory involved, so if some aspects are unclear upon first reading, continue reading and return to those parts later for clarification.

## A desk calculator

> What I cannot create, I do not understand
>
> Richard Feynman

In their now almost 40 year old book [The Unix Programming Environment](https://en.wikipedia.org/wiki/The_Unix_Programming_Environment) Brian Kernighan and Rob Pike urge us to build a desk calculator. They used the standard techniques at the time, a parser generator _yacc_ (predecessor to Bison) and _lex_ (today you could use flex).

We are going to follow in their footsteps, and create a handmade parser for a desk calculator.

You can find all the code for this example in our [GitHub repository](https://github.com/nhatcher/sample-desk-calcualtor).

### The one with the REPL

First thing we need is a _REPL_ (Read-Eval-Print-Loop). Go ahead create a new Rust project with `cargo new calculator` and substitute the `main.rs` file with:

```rust
use std::io::{stdin, stdout, Write};

fn evaluate(input: &str) -> String {
    input.to_string()
}

fn main() {
    loop {
        let mut input = String::new();
        print!("Input: ");
        let _ = stdout().flush();
        stdin()
            .read_line(&mut input)
            .expect("Failed reading command");
        input = input.trim().to_string();

        if input == ".exit" {
            println!("Bye!");
            break;
        }

        let output = evaluate(&input);
        println!("Output: {}", output);
    }
}
```

That is an echo REPL, it responds to user input by printing their input back to them (hence "echo"). You can run your program with `cargo run`. If you type `.exit` you exit the program.

### The one with the grammar

We are going to split the evaluate function in two parts:

1. **Parse:** parsing the formula into an _[abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree)_ (or AST for short)
2. **Evaluate:** traversing the parse tree in order to evaluate. The second part is the job of an _interpreter_.

When writing a parser the first thing you need to do is to spell out the grammar for your language:

```
expression    => primary operator expression | primary
primary       => Number | Variable_name | '(' expression ')' | function_call
function_call => Function_name '(' expr ')'
operator      => '+' | '-' | '*' | '/'
```

These are the blueprints of the program we are going to write. There are formal, very theoretical specifications of what a grammar is, but we will be a bit loose here.

### The one with the lexer

A lexer is an algorithm that processes an input string and provides a list of indivisible atoms in the language, known as _tokens_. Some tokens consist of a single character, such as `(` or the plus operator `+`, while others are composed of multiple characters, like the function name `Sin` or a number like `2.3e-12`.

Once your grammar is defined, you should create your data structures. Rust is very convenient for this, due to the availability of _[algebraic data types](https://en.wikipedia.org/wiki/Algebraic_data_type)_ and _enum varieties_.

The list of tokens will be an _enum_:

```rust
// Tokens
enum Token {
    Number(f64),
    Name(String),
    OpenParenthesis,
    CloseParenthesis,
    Plus,
    Minus,
    Times,
    Divide,
}
```

The lexer (also known as a scanner or tokenizer), will get a string like `"2.3+4*(Sin(3+7)+5)^2"` and return one by one its components or _tokens_:

```
[
    Number(2.3),
    Plus,
    Number(4.0),
    Times,
    OpenParenthesis,
    Name("Sin"),
    OpenParenthesis,
    Number(3.0),
    Plus,
    Number(7.0),
    CloseParenthesis,
    Plus,
    Number(5.0),
    Power, 
    Number(2.0)
]
```



The algorithm that performs this task is a [Deterministic Finite Automaton](https://en.wikipedia.org/wiki/Deterministic_finite_automaton) (DFA). Essentially, if your tokens can be described by Regular Expressions (RE), they can be converted to a DFA and easily implemented in code (as per [Thompson's construction](https://en.wikipedia.org/wiki/Thompson%27s_construction)). In this article, we will not delve into the theorem itself; instead, we will focus on how we can apply it to our specific example.

Outline of the desk calculator lexer:

```rust
pub struct Lexer {
    input_chars: Vec<char>,
    position: usize,
}

impl Lexer {
    pub fn new(input_text: &str) -> Lexer {
        let input_chars: Vec<char> = input_text.chars().collect();
        Lexer {
            input_chars,
            position: 0,
        }
    }

    pub fn next_token(&mut self) -> Token {
        self.consume_whitespace();
        match self.read_next_char() {
            Some(ch) => match ch {
                '+' => Token::Plus,
                '-' => Token::Minus,
                '*' => Token::Times,
                '/' => Token::Divide,
                '^' => Token::Power,
                '(' => Token::OpenParenthesis,
                ')' => Token::CloseParenthesis,
                '0'..='9' | '.' => {
                    self.position -= 1;
                    self.read_number()
                }
                'A'..='Z' => self.read_function(ch),
                _ => Token::Illegal(format!("Unexpected character: {}", ch)),
            },
            None => Token::EoI,
        }
    }

    fn read_next_char(&mut self) -> Option<char> {
        let next_char = self.input_chars.get(self.position);
        if next_char.is_some() {
            self.position += 1;
        }
        next_char
    }

    fn peek_char(&self) -> Option<char> {
        self.input_chars.get(self.position)
    }

    fn consume_whitespace(&mut self) {
        while Some(char) = self.input_chars.get(self.position) {
            if !char.is_whitespace() {
                break;
            }
            self.position += 1;
        }
    }

    fn read_function_name(&mut self, first_letter: char) -> Token {
        todo!()
    }

    fn read_number(&mut self) -> Token {
        todo!()
    }
}

```

Let's review. First, observe the Lexer definition:

```rust
pub struct Lexer {
    input_chars: Vec<char>,
    position: usize,
}
```

If we were to make the same structure in TypeScript, Python, Kotlin, ... the definition would be obvious:

```kotlin
class Lexer(
    val input: String,
    var position: UInt,
) { /*...*/ }
```

But in Rust we have many things to consider. Do we want the data to be owned by the Lexer, or is the data shared? The first option is easier but implies that the Lexer will make a copy of the data and do memory allocations on the heap. The second is a bit more complex because it involves lifetimes, but it's faster and more memory efficient.

But the questions do not end there. You might want to use a _[smart pointer](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html)_ such as a `Box<Node>`, if you just want to store the pointer in the stack. We will use that when we discuss the parser. The _abstract syntax tree_ is an acyclic graph, but if we were to deal with a cyclic graph then you'd probably need to use an `Rc<Node>`.

Also, notice how we convert the `String` to a vector of `char`. Rust has a `String` type, but it is simply a vector of bytes encoding a UTF-8 string, which complicates accessing individual characters (due to multibyte encodings).

Most of the code in the lexer should be relatively straightforward to read, with the possible exception of the "number recognizer".

It's surprisingly difficult to parse numbers. As recently as November 2022, [Daniel Lemire](https://lemire.me/en/) demonstrated that it is possible to [parse numbers at gigabytes per second](https://lemire.me/blog/2021/01/29/number-parsing-at-a-gigabyte-per-second/). Number parsing has two steps:

1. _Recognizing_ the number by identifying the integer part, decimal part, and exponent.
2. _Finding_ the closest floating-point number to the real number we recognized.

The second step is the more difficult of the two.

The inverse problem of finding the most visually appealing way to display a given floating-point number currently relies on the [Ryū algorithm](https://github.com/ulfjack/ryu) by [Ulf Adams](https://www.youtube.com/watch?v=kw-U6smcLzk&ab_channel=PLDI2018), which is one of my favorite _modern_ algorithms.


A RE for a number might be:

```
/^[+-]?[0-9]+(.[0-9]+)?([Ee][+-][0-9]+)?$/
```

Consider this diagram (_accepting states_ are 3, 5 and 8):

![Diagram](/images/parse.png)

Our number recognizer, a _finite automata_, has 8 states. An "accepting state" is a state in which you can finish your parsing. It's pretty easy to transform the above diagram into code:

```rust
    fn read_number(&mut self) -> Token {
        let mut state = State1;
        let mut str = "".to_string();
        let mut accept = true;
        while accept {
            if let Some(&c) = self.peek_char() {
                match state {
                    State1 => {
                        if c.is_ascii_digit() {
                            state = State3;
                        } else if c == '-' || c == '+' {
                            state = State2;
                        } else {
                            return Token::Illegal(format!("Expecting digit or + or -, got {}", c));
                        }
                    }
                    State2 => {
                        if c.is_ascii_digit() {
                            state = State3;
                        } else {
                            return Token::Illegal(format!("Expecting digit got  {}", c));
                        }
                    }
                    State3 => {
                        // Accepting state
                        if c == '.' {
                            state = State4;
                        } else if c == 'E' || c == 'e' {
                            state = State6;
                        } else if !c.is_ascii_digit() {
                            accept = false;
                        }
                    }
                    State4 => {
                        if c.is_ascii_digit() {
                            state = State5;
                        } else {
                            return Token::Illegal(format!("Expecting digit got  {}", c));
                        }
                    }
                    State5 => {
                        // Accepting state
                        if c == 'e' || c == 'E' {
                            state = State6;
                        } else if !c.is_ascii_digit() {
                            accept = false;
                        }
                    }
                    State6 => {
                        if c == '+' || c == '-' || c.is_ascii_digit() {
                            state = State7;
                        } else {
                            return Token::Illegal(format!("Expecting '+'or '-' got  {}", c));
                        }
                    }
                    State7 => {
                        if c.is_ascii_digit() {
                            state = State8;
                        } else {
                            return Token::Illegal(format!("Expecting digit got  {}", c));
                        }
                    }
                    State8 => {
                        // Accepting state
                        if !c.is_ascii_digit() {
                            accept = false;
                        }
                    }
                }
                if accept {
                    str.push(c);
                    self.position += 1;

                }
            } else {
                break;
            }
        }
        Token::Number(str.parse::<f64>().unwrap())
    }
```

### The one with the parser
> To iterate is human, to recurse divine.
>
> Peter Deutsch

Once we are able to lex tokens we are ready to build the AST. We will use a top down recursive parser.
The data structure is something like:


```rust
// Abstract syntax tree nodes
enum Node {
    Operation {
        op: Operation,
        left: Box<None>,
        right: Box<Node>,
    },
    Function {
        fun: Function,
        arg: Box<Node>
    }
    Number(f64),
    Variable(String)
}

enum Function {
    Sin,
    Cos,
    Tan,
    Ln,
    Exp,
}

enum Operator {
    Plus,
    Minus,
    Times,
    Divide, 
}
```

Note that in Rust we use a smart pointer (`Box`) to link to the next node in a tree structure.

Although we are building a simple calculator we are actually going to face one of the most difficult problems in parsing: left recursion and precedence.

The correct parse tree for the formula `3+4*8` is:
```
     +
    / \
   3   *
      / \
     4   8
```
And not:
```
     *
    / \
   +   8
  / \
 3   4
```
The latter parse tree describes `(3+4)*8`, not `3+4*8`. That is to say: `*` has precedence over `+`.

Traditionally, grammar is adapted to reflect the precedence rules so as to produce the desired parse tree. But we will follow a different strategy: "top down precedence parsing" was invented by [Vaughan Pratt](https://tdop.github.io/) and popularized in his build of jslint by [Douglas Crockfort](https://crockford.com/javascript/tdop/tdop.html) and [Bob Nystrom](https://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/) of [Game programming patterns](http://gameprogrammingpatterns.com/) and [Crafting interpreters](http://craftinginterpreters.com/) fame, and must be one of the most beautiful recursive parsing algorithms ever written.

Here we are inspired by the work of [Alex Kladov](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html) in implementing a Pratt parser.

The central idea is to attach some _binding powers_ to each operator. `*` will have a higher binding power than `+`, for instance. 

The basic implementation of a top down recursive Pratt parser will therefore resemble:

```rust
pub struct Parser {
    lexer: Lexer,
}

fn infix_binding_power(op: &Operator) -> u8 {
    match op {
        Operator::Plus => 1
        Operator::Minus => 1
        Operator::Times => 3
        Operator::Divide => 3
        Operator::Power => 5,
    }
}

impl Parser {
    pub fn parse(input_text: &str) -> Result<Node, ParserError> {
        let mut lexer = Lexer::new(input_text);
        let mut parser = Parser {
            lexer,
        };
        parser.parse_expression(0)
    }

    fn parse_expression(&mut self, left_bp: u8) -> Result<Node, ParserError> {
        let mut lhs = self.parse_primary()?;
        loop {
            let op = match &self.next_token {
                Token::EoI => break,
                Token::CloseParenthesis => break,
                Token::Plus => Operator::Plus,
                Token::Minus => Operator::Minus,
                Token::Times => Operator::Times,
                Token::Divide => Operator::Divide,
                Token::Power => Operator::Power,
                unexpected => todo!()
            };

            let right_bp = binding_power(&op);
            if right_bp < left_bp {
                break;
            }

            let rhs = self.parse_expression(right_bp)?;

            lhs = Node::BinaryOp {
                op,
                left: Box::new(lhs),
                right: Box::new(rhs),
            };
        }
        Ok(lhs)
    }

    fn parse_primary(&mut self) -> Result<Node, ParserError> {
        match self.lexer.next_token() {
            Token::Number(value) => {
                Ok(Node::Number(value))
            }
            Token::Plus => {
                let primary = self.parse_primary()?;
                Ok(Node::UnaryOp {
                    op: UnaryOperator::Plus,
                    right: Box::new(primary),
                })
            }
            Token::Minus => {
                let primary = self.parse_primary()?;
                Ok(Node::UnaryOp {
                    op: UnaryOperator::Minus,
                    right: Box::new(primary),
                })
            }
            Token::Name(name) => {
                if self.lexer.next_token() != Token::OpenParenthesis {
                    return Ok(Node::Variable(name));
                }
                let argument = self.parse_expression(0)?;

                self.expect(Token::CloseParenthesis)?;

                let index = match name.as_str() {
                    "Cos" => Function::Cos,
                    "Sin" => Function::Sin,
                    "Tan" => Function::Tan,
                    "Log" => Function::Log,
                    "Exp" => Function::Exp,
                    _ => {
                        return Err(ParserError {
                            position: self.lexer.get_position(),
                            message: format!("Invalid function name: '{}'", name),
                        });
                    }
                };
                Ok(Node::Function {
                    index,
                    arg: Box::new(argument),
                })
            }
            Token::OpenParenthesis => {
                let primary = self.parse_expression(0)?;
                self.expect(Token::CloseParenthesis)?;
                Ok(primary)
            }
            Token::CloseParenthesis | Token::Times | Token::Divide => Err(ParserError {
                position: self.lexer.get_position(),
                message: format!("Unexpected token: '{value}'"),
            }),
        }
    }
}
```

You will probably remember from your school days that every recursive algorithm has an equivalent iterative algorithm. In this particular case the iterative algorithm is called _[shunting yard algorithm](https://en.wikipedia.org/wiki/Shunting_yard_algorithm)_ and was invented by Edsger Dijkstra 1961 for the Algol programming language.

The first part, the `parse_expression` part is the top down precedence parser, the `parse_primary` is just a standard top down recursive parser that mimics the grammar that we repeat here because it is that important:

```
expression    => primary operator expression | primary
primary       => number | variable_name | '(' expression ')' | function_call
function_call => function_name '(' expr ')'
operator      => '+' | '-' | '*' | '/'
```

Note that the most difficult part is to tell apart a function call from a variable name. The way to do that is to realize that a function call must have a `(` after the name. This sort of thing is what is relatively easy in a handmaid parser, but awkward in a parser generator.


### The one with the walking interpreter

Although not the focus for this post, a calculator would be incomplete without an evaluator. It is the soul of a tree walking interpreter.

Once we have the AST, evaluation is pretty straightforward:

```rust
pub fn evaluate(input: &str) -> Result<f64, ParserError> {
    let node = Parser::parse(input)?;
    evaluate_node(&node)
}

pub fn evaluate_node(node: &Node) -> Result<f64, ParserError> {
    match node {
        Node::Number(f) => Ok(*f),
        Node::Variable(s) => {
            if s == "PI" {
                Ok(std::f64::consts::PI)
            } else {
                Err(ParserError {
                    position: 0,
                    message: format!("Unknown constant: {s}"),
                })
            }
        }
        Node::Function { index, arg } => {
            let argument = evaluate_node(arg)?;
            match index {
                Function::Cos => Ok(argument.cos()),
                Function::Sin => Ok(argument.sin()),
                Function::Tan => Ok(argument.tan()),
                Function::Log => Ok(argument.ln()),
                Function::Exp => Ok(argument.exp()),
            }
        }
        Node::BinaryOp { op, left, right } => {
            let lhs = evaluate_node(left)?;
            let rhs = evaluate_node(right)?;
            match op {
                Operator::Plus => Ok(lhs + rhs),
                Operator::Minus => Ok(lhs - rhs),
                Operator::Times => Ok(lhs * rhs),
                // Missing error handling
                Operator::Divide => Ok(lhs / rhs),
                Operator::Power => lhs.powf(rhs),
            }
        }
        Node::UnaryOp { op, right } => match op {
            UnaryOperator::Plus => Ok(evaluate_node(right)?),
            UnaryOperator::Minus => Ok(-evaluate_node(right)?),
        },
    }
}
```


### The one with the exercises

1. (easy) Add more functions to the calculator.
2. (medium) Add a `Sum` function with three arguments like `Sum(1, 100, n*n)` or `Sum(n*n, {n, 1, 100})` depending on your taste.
3. (medium) Add variables `a = 7.4` or `a = 4*Sin(3)` that you can use in later expressions. You will need to maintain a symbol table
4. (difficult) Add for loops and if branches and create your first mini language!
5. (difficult) Rewrite the parser using Dijkstra's shunting yard algorithm.


## General references

If you want a simple and direct account of many of the issues discussed here, I cannot recommend this book enough:
* [Introduction to Compilers and Language Design](https://www3.nd.edu/~dthain/compilerbook/) by Douglas Thain

Thorsten Ball has written a really nice book very much in the same spirit as this post:
* [Writing an interpreter in go](https://interpreterbook.com/)

Robert Nystrom's book is freely available and is a thing of beauty:
* [Crafting interpreters](https://craftinginterpreters.com/)

Also worth mentioning:
* [Build your own Lisp](https://buildyourownlisp.com/)

A couple of more academic references:

* [Engineering a compiler](https://www.elsevier.com/books/engineering-a-compiler/cooper/978-0-12-815412-0) by Keith Cooper and Linda Torczon
* [Crafting a compiler](https://www.pearson.ch/HigherEducation/Pearson/EAN/9780136067054/Crafting-A-Compiler) by Charles N. Fischer, Ron K. Cytron and Richard J. LeBlanc  

The dragon book will always remain a classic in the subject:

* [Compilers: Principles, Techniques, and Tools](https://suif.stanford.edu/dragonbook/)

If you are up for a MOOC, I can recommend this course describing the COOL programming language by Alex Aiken
* [Compilers](https://www.edx.org/course/compilers)

Or the course by Jeffrey Ullman (one of the authors of the dragon book). This might be a bit advanced.
* [Automata theory](https://www.edx.org/course/automata-theory)

Both courses from Stanford University




