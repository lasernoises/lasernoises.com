+++
title = "Lisp Formatting"
date = "2026-02-27"
+++

This is just a quick rant about formatting in Lisp that I felt the urge to write down.

I've been reading more about Lisp and I do really like the simplicity of
S-expressions.
What really annoys me though is the typical convention about formatting that everyone seems to use.

```scheme
(let loop ((n 1))
  (if (> n 10)
    '()
    (cons n
      (loop (+ n 1)))))
```

This is just a random example from the Scheme Wikipedia page.
What really bothers me is how all the closing parens are just piled up at the end.

```scheme
(let loop ((n 1))
  (if (> n 10)
    '()
    (cons
      n
      (loop (+ n 1))
    )
  )
)
```

This is how I'd want to format that.
It feels immediately much more readable to me that way.

I get that it uses more space, but I don't think that's actually a bad thing necessarily.
Imagine I formatted my Rust code like this:

```rust
fn abc(a: bool) {
  some_other_function(
    1,
    if a {
      def()}
    else {
      2})}
```

Doesn't that feel cursed?
It just looks like I'd rather be programming in Python.

Well this is how I feel whenever I read some Lisp code.
It just feels like whoever wrote it would rather be using a whitespace sensitive language.

I actually kind of think this is the reason people make jokes about all the parens in Lisp.
Aside from basic arithmetic Lisp doesn't really have more brackets than other languages, it's just
that piling them up on the end of a line really draws attention to them.

But of course this is a personal preference thing.
I'm sure you can get used to this formatting, though I think at that point you have to actually
ignore the closing parens and look at the indentation.
And I'm also sure that using a structural editor helps with this as well.

I'm also a bit of an extremist when it comes to putting closing syntax on new lines.
For example, before using a formatter I was known for doing this:

```rust
a
  .b()
  .c()
  .d()
;
```

And even this:

```rust
some_function(
  a
    .b()
    .c()
    .d()
  ,
  12,
);
```

This also has the advantage of making the diff cleaner when you insert an additional call at the
end of the chain, similar to trailing commas.
