+++
title = "A Shared Syntax for Custom Languages"
date = "2026-08-25"
+++

## Introduction

I've been thinking about a universal syntax for programming languages, DSLs and data formats /
config files.
The motivation comes from a few different angles:

### S-Expressions

I find Lisp's S-Expressions very intriguing.
They are universal in a way the syntax of other languages doesn't tend to be.

Yet I don't quite want to use them.
A lack of experience is certainly a part of it, but I find them quite hard to read.

Then there's also the formatter thing.
Since the same syntax is used for everything it's quite hard for a formatter to know which
formatting would be appropriate.

### Data Formats

I also like using formats like JSON and RON.
Especially in combination with Serde.

There is one persistent frustration however.
And that is the lack of language server support.

It's quite frustrating to have a static schema in the form of type definitions, but to still get no
support in the editor with finding the right names and seeing available fields.

There are solutions for this, like [Tombi](https://tombi-toml.github.io/tombi/), but that's limited
to JSON Schema.
While a static schema is the baseline of what I want it's not quite sufficient.

To illustrate, let me talk about a templating language for PDFs I started to build once.
The idea was to template [Laser-PDF](https://github.com/laser-pdf/laser-pdf) elements with a
structure like Mustache.
There was an external definition of available fields.

Notably the fields could be recursive, like a GraphQL schema.
The template would first be evaluated to get a tree of used fields.
That tree would then be used to actually fetch the data.

I was building this on top of RON and Serde at the time, so I couldn't even get the static list of
elements in my editor.
But to fully support this I would have needed to get the available templating fields in the editor
as well.

Those weren't static.
The program would read the schema from a file at runtime.

So this can't be done with a JSON Schema style solution.
We need to be able to execute code in the LSP implementation.

### LLMs and Non-Turing-Complete Languages

As I expanded on [here](https://lasernoises.com/blog/non-turing-complete-languages/), I think it's a
good idea to have as much of LLM generated code as possible be in very restricted languages that
aren't Turing-complete.

Unfortunately today, switching to a custom DSL or a config format often means loosing most of the
editor support you get from your main general purpose language.

### Why not KDL?

The [KDL](https://kdl.dev/) language is very interesting and seems to want to solve some of the same
problems.
However, it's clearly not meant to be used for anything that starts to resemble a programming
language too much.
It is very focused on an XML style structure of nodes with attributes and children and since the
values of attributes can't contain other nodes, representing a programming language or certain kinds
of DSLs with it would become very awkward.

## An Attempt at a Solution

I really like using Nushell.
One interesting thing about it (and most shells to a degree) is the way nested commands start
looking like S-Expressions.

```nu
ls (pwd -P)
```

In Nushell the resemblence grows when you have to add parens toplevel when you want to split a
command over multiple lines.

What if we use this as the starting point for our universal language?
Meaning top level calls (whatever that means in the particular language; in a config format they'd
be used for enum variants for example) don't need parens around them.

In order to avoid newline sensitivity and needing toplevel parens around multiline calls we'll add
semicolons to separate them.

```
foo 12;
bar "12" (nested_call 12);
```

Now since any real laguage has some form of nesting we need something like blocks.
Since we don't want to make any decisions that are specific to procedural languages where a block is
a list of statements, we'll use the same syntax for blocks and lists.

```
foo [
  bar 12;
];
```

### Maps

Now we get to something annoying.

Clojure has a dedicated syntax for maps.
And it's a little bit awkward.
Since, as a Lisp, Clojure aims to be [homoiconic](https://en.wikipedia.org/wiki/Homoiconicity), the
quoted form of a map is also a map.

This means you can't have the same function call multiple times as a map key:

```clojure
{
  (my_function) 12
  (my_function) 13
}
```

This gets rejected even though `my_function` might have some internal state that makes it produce a
unique key each time, which is possible since Clojure isn't a purely functional language.

Of course this isn't a huge deal in practice, since there are more ways to create maps, but it's
still a little awkward.

Maps are also a bit of a point of weirdness in JSON.
For the most part JSON is just a syntax and thus doesn't say anything about the uniqueness of object
keys or whether the order of keys matters.
[RFC-8259](https://datatracker.ietf.org/doc/html/rfc8259#section-4) merely makes uniqueness a
SHOULD.

I've actually ran into this with MariaDB's `EXPLAIN FORMAT=JSON`, which sometimes produces duplicate
`"table"` keys instead of making it an array.
This is valid JSON, but means you'll have a hard time processing it with standard tools like JQ.

I want my language to also have the property of just being a syntax, but I want to avoid the problem
of unclear semantics.
I also want to make it possible to build a Lisp style homoiconic language.

Because of this I decided to just represent maps as lists of calls,
[a bit like you'd do in KDL](https://github.com/kdl-org/kdl/blob/main/JSON-IN-KDL.md):

```
map [
    my_key 12;
    my_other_key 12;
]
```

The `map` here would be how you would handle it in the hypothetical homoiconic language, in other
usecases like Serde deserialization it would be clear from context that the list needs to be parsed
into a map.
If you were to implement a Rust-style language with this syntax you'd also use that pattern for
structs:

```
MyStruct [
  field_a 12;
  field_b 12;
]
```

And for struct definitions:

```
struct MyStruct [
  field_a u32;
  field_b i16;
];
```

You can already see how the call syntax can be used for more than just representing actual function
calls.
For example enum variants would also fit this pattern.

### Pipes

There's another thing I want to steal from shells.
Pipes.

I really like how you can use pipes allmost like method calls in Nushell for example:

```nu
start (ls --long ~/tmp | sort-by --reverse created | first | get name)
```

Many programming languages implement pipes in a way that that inserts the expression before the pipe
as the first argument to the function after the pipe.
For example consider this Gleam code from their
[language tour](https://tour.gleam.run/functions/pipelines/):

```gleam
"Hello, Mike!"
  |> string.drop_end(1)
  |> string.drop_start(7)
  |> io.println
```

That is equivalent to this nested call:

```gleam
io.println(string.drop_start(string.drop_end("Hello, Mike!", 1), 7))
```

This makes it tempting to support pipelines as a purely syntactic transformation while parsing.

So we would take syntax like this:

```
"Hello, Mike!" | drop_end 1 | drop_start 7 | println
```

And parse it to the same AST as:

```
println (drop_start (drop_end "Hello, Mike!" 1) 7)
```

I don't want to do it in this way, however.
The reason is that a lot of languages have their own way of handling pipelines or method calls, that
isn't simply passing equivalent to passing a first parameter.

For example in shells it means piping to `stdin`.
Or in most object-oriented or adjacent languages method calls get resolved in the scope of the type
(meaning something like method definitions inside the class defintion instead of just calling
freestanding function with the given name).

So I think it makes sense to unify all those usecases with a single pipe syntax and let the
particular language decide what they mean of whether to support them at all.

### Beginnings of an Implementation

So far I've mostly been thinking about this language, but I have started to implement something.

I've made a declarative macro in Rust that parses my syntax into an enum.
I mostly just wanted to stop myself from thinking about parser libraries and whether to just write
my own parser.

I do however think supporting usage from a macro as well as parsing from strings will stay useful
even after we have a proper parser.
Though it'll probably become more of a function that gets called in procedural macros that define
DSLs.

The goal of the syntax is to not have any special keywords.
So anything from `true` to `async` just gets parsed as an identifier on the syntax level.
It's up to the language whether those will then be handled specially or not.

The same goes for most symbols like `+`, `-` and so on.
The exceptions are the few symbols that are needed for the structure: `(`, `)`, `[`, `]` and `|`.

As a declarative macro we can't implement these rules perfectly.
So for example `some-thing` is parsed the same as `some - thing`.
With procedural macros we could look at the byte offsets, but I'm not sure I like that.

I think it's ok for the macro version to not be perfectly identical to the actual language.
It's just about having an easy way to use a custom language inline sometimes.
For example it could be useful for testing an implementation of a DSL without having string literals
all over your tests.

I've also started implementing a Serde deserializer that operates on this data-structure.
Of course a real one should be more efficient than this and skip the AST entirely.

The interesting part is that we can't fully support being self-describing using `deserialize_any`
because our syntax doesn't differentiate between maps and lists and so we need the hint for those.

Let's see if I have the motivation and time to build out this implementation enough to make it
useable.
I decided to write this post mostly because I want to at least get the idea written down in case I
don't.
