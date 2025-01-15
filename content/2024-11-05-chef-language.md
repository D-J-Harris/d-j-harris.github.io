---
title: Chef - a Recipe-Oriented Language
description: "Creating a fun language for personal use, and unpicking intution for stack-based languages and parsing."
date: 2024-11-05
toc: true
taxonomies:
  tags: [rust, languages]
---

> Code for this project can be found [here](https://github.com/D-J-Harris/chef/)

<br />

Following on from my previous work on a [Rust-based Lox interpreter](../crafting-interpreters), I wanted to get creative and design a language that had a unique look, and would allow me to test my understanding of the content I had picked up from the [Crafting Interpreters](https://craftinginterpreters.com/) book.

The outcome of this is `chef` - a recipe-oriented language! This has been a lot of fun to work on, and I want to cover some of the decision choices I made that branch from Lox, and the interesting side-effects these had in the code.

![Fibonacci Function in Chef](/assets/content/chef-language/fib_recipe.png "Fibonacci Recipe in Chef")
_Function to find the nth number in the Fibonacci sequence, in `chef`_

## Design Decisions

### Structure

I wanted to design a language that looked and felt like a recipe at first sight - usability wasn't a primary concern here. The first item to look at was code structure: usually recipes front-load the required ingredients, and move from there to step-by-step instructions. It felt natural for "ingredients" to be variables in the language, and for them to feel "global" - you wouldn't want to be halfway through a recipe and find a new ingredient requirement!

```
Decision 1. Variable declaration should happen up-front.
Decision 2. Programme statements should form a numbered lists of "steps".
```

There was also the question of functions, classes and closures, and modelling these concepts in a way that remains the look and feel of a recipe. In the end I decided to keep the language lean, and retain only functions in the form of their own set of "sub-steps" that should be followed. Using action words like "bake" or "whisk" for function names could help maintain this spirit.

I wanted the language to feel almost prose-like, with minimal nesting. So function declarations are also all declared up front. This leads itself to defining the programme in sections - you have your ingredients (variables), your actions or utensils you will need (functions) and your statements (steps). This separation also helps keep the source looking clean.

```
Decision 3. Functions declared up-front, also with their own steps.
Decision 4. Programme should be split into sections.
```

### Naming and Syntax

When was the last time you read a recipe that asked you to whisk the EntityFactoryComponent? Or to bake `x` for 5 minutes? To address this, I moved from an `Ident` (identifier) token model, which represents _any_ non-keyword string of characters, to two separate `VarIdent` and `FunIdent` tokens for variables and functions, each represented by an allow-list of words. For example, variables could only be `chocolate`, `banana`, `sugar` etc. and functions only action words like `whisk`.

I also decided to do away with symbols, such as parantheses in if-statements and curly braces for blocks. This led to interesting challenges, as I will discuss later, but helped to keep the prose-like style I was going for. Spoiler: the final language only uses `(` and `)` for grouping, `"` for strings and `,` for function argument lists.

```
Decision 5. Use a pre-defined list of words for variables and functions.
Decision 6. Minimise use of symbols not used in natural language.
```

### Scoping

When writing the original Lox interpreter, variables could take one of three forms: global, local or as an "upvalue" (for variables captured outside of a closure's scope). I rid `chef` of closures, which left just two types to think about.

Having all variables declared up-front, and never inline, seems to lend itself well to a world where all variables are global. However, typically having only global variables can make a programme more difficult to reason about - every interaction becomes a side effect. Instead, the top-level of the programme (the outer scope) can also be interpreted as the first local scope. This final decision leads to functions being pure (no side effects), and also having fewer lookups of heap-allocated variables from some map-structure (not that I was focusing on performance).

In the long run, pre-defining all variables and functions in the outer scope led to the interesting property that the language does not have "call frames" in the original sense - all opcodes are contiguous, and function calls can be redirections of the instruction pointer rather than loading of function opcodes into a new call frame. Again, fewer allocations.

```
Decision 7. No global variables, and only one code block (no concept of "function code").
```

## Implementation

The link to the project code can be found at the top of the article - this will not be a deep dive on every change from the Lox implementation of a compiler / bytecode interpreter to `chef`, but I wanted to highlight some of the more interesting changes

### Calling Functions

The design of having all function declarations up-front means all opcodes are laid out before compiling the rest of the programme. What this means is that we can shift our view of what it means to enter a function:

_before_:

- 1. Add a new frame to the call-stack
- 2. Instruction pointer and opcodes now accessed through looking at the heap-allocated function code
- 3. Frame captures where on the stack the function (callee) lives, with arguments coming after it

_after_:

- 1. Jump the instruction pointer to the start of the function, capturing where you were before so you can continue where you left off once the function ends
- 2. Profit

This is pretty cool! Functions no longer need to be heap-allocated, and can be put on the stack with minimal information required:

```rust
pub struct Function {
    pub name: String,    // only needed for debugging
    pub arity: u8,       // only needed for error handling
    pub ip_start: usize, // function size can be restricted to one word!
}
```

### Garbage Collection (not!)

Unlike the core focus of my [previous article](../crafting-interpreters), GC did not play a role here at all! This was by design of the values that could be taken:

```rust
pub enum Value {
    Nil,
    Number(f64),
    Boolean(bool),
    String(String),
    Function(Function),
    NativeFunction(NativeFunction),
}
```

When the code had closures and heap-allocated function call-frames, then it was possible for the values in the programme to become self-referential and contain cycles that could not be easily cleaned up without some form of garbage collection.

## Features

### Argument lists

Argument lists in function signatures and invocations read like proper English:

- 1 argument : `whisk with x`
- 2 arguments: `whisk with x and y`
- 3 arguments: `whisk with x, y and z`
- 4 arguments: `whisk with x, y, z and i`

### Steps

Yes, the code will not compile if your step count is not monotonically increasing from `1.`

### VSCode Extension

I also delved into the world of extension development as part of this project, which was fun! The code can be found [here](https://github.com/D-J-Harris/chef-colouring)

Some neat features of this extension include the syntax declaration and colouring, as well as assistance with code indentation and step incrementing when hitting `Enter` and moving to the next step of a given function body.
