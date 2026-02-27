+++
title = "Is it time for more non-Turing-complete languages?"
date = "2026-03-26"
+++

I've been thinking a lot about our AI future and how we can manage the growing complexity associated
with that.
It's a topic that is being discussed a lot, but what I haven't seen being talked about a lot is the
role of non-Turing-complete languages could play.

A different term for this would be DSLs (domain-specific langauge).
And DSLs are of course a bit of a classic topic people have very different opinions about.
I think it's a bit clearer to talk about non-Turing-complete langauges though, since nothing says
that a DSL can't be turing complete.
And I think it's usually bad when a DSL becomes turing complete (deliberately or not).

I think non-Turing-complete langauges have a lot of potential because there are so many thing that
can't go wrong when a language is restricted by design.
And the poperties that can't be violated can be designed into the language.
Which tends to be much harder, if not impossible to design into a programming framework or library
for the same problem space.

## An example

A concrete thing I've been thinking about HTML templating.

[Mustache](https://mustache.github.io/) is a good example of non-Turing-complete HTML templating,
but I would suggest going a bit further.

Firstly I'd prefer the templating to be structure aware and while we're at it we can ditch the ugly
SGML syntax.

```
div {
  "Some text"
  for my_var {
    span {
      "some other text with a {{ variable }}"
    }
  }
}
```

This has the property that we can statically prevent it from generating invalid html.

This is nothing special by itself, but there is one very interesting property that a Mustache-style
language has.
You can easily extract a tree of the variables and fields it uses.
And that is quite close to a GraphQL query.

So I think we could build a query langauge that integrates with GraphQL (it doesn't have to be
GraphQL in principle, just some tree structured query system).
If you then combine that with a GraphQL server generated from a DB schema, you can basically build
simple CRUD apps without any need for a Turing-complete language.

(Maybe something like this already exists. I haven't really researched this.)

I think a sensible next step would be to actually restrict the HTML tags and attributes to forbid
any JavaScript directly in the template.

## Making it work in practice

I think there are some important things to make non-Turing-complete languages not a bad idea in
practice.

### LSP integration

This might be the hardest bit.

I often find config files written in something like TOML to be a bit annoying in practice for a
relatively dumb reason.
And that is that my editor can't tell me what kind of keys I can use where.
But there isn't really a good reason it should be this way.
In principle writing a language server for such a language should be a whole lot easier than for a
full general purpose programming language.

And it seems there's projects like [Tombi](https://github.com/tombi-toml/tombi) that solve this
problem.
Though I think it's a bit unfortunate having to determine the relevant schema based on the filename.
Maybe the XML idea of directly referencing the schema in the file itself isn't the worst idea (just
please keep that ugly XML syntax away from me).

It seems the [KDL](https://kdl.dev/) language is going in an interesting direction there.

### Escape hatches

The other thing is that you need a way to get to code written in a general purpose language for
things that the non-Turing-complete langauge can't do.

I think for the HTML templating + GraphQL example this would be done at multiple layers.
Firstly for the frontend you could use web components for things that can't be done in plain HTML.
I think that's a better idea than allowing JavaScript in attributes.

Then of course if you have a generated GraphQL server you need a way to call custom code there.
And maybe you even need a way to run custom code for HTML generation, but I'm not so sure how common
that would be.

I think the important thing for all of these is that the escape hatch should be very explicit.
And that the code it calls shouldn't be inline with the non-Turing-complete langauge, but in a
separate place where you can keep an eye on its complexity.
