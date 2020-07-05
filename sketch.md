# Intro & “Justification”

This talk will describe my experiences with getting into IR-based metaprogramming, and my
conclusions about two main libraries of the existing infrastructure: IRTools and Cassette.  It does
two things:

1. Describe my learning process from zero knowledge about IR things on, and
2. my conclusions about IRTools and Cassette from a “user perspective”.

## Scope

As prerequisites, I assume basic familiarity with metaprogramming, `@generated` functions, the AST,
and IR.  Having a rough idea about how Zygote works would be ideal.  I will not talk about typed IR,
type stability issues of IR transformations, or the coming `AbstractInterpreter` interface (exciting
times to come!).

## Story

At the beginning of my master’s thesis, I was given a very loose task of analysing the dependecy
structure in Turing models, with the goal of improving some inference algorithms.  Now, Turing is a
PPL embedded in Julia by very simple means: a normal function is transformed a bit by a macro.
There’s no structure tracked, and the model is not reified in any way.

So I started thinking about what would be a good approach to extract the structure of a Turing
model.  At first, I was thinking about just basic computation graph extraction, like in trace-based
automatic differentiation, and was researching in that direction.  But there’s more interesting
structure to a model than just the computation graph of a loss function.  During my literature
search, I had loosely heard about a new AD library called Zygote, that implemented automatic
differentiation through transforming the intermediate representation of Julia.

So, after drawing many examples and thinking through a lot of possible needs, I decided that there
is need for a way to track the complete execution of a function, and that this is most easily
achieved by combining ideas I had heard from Zygote and the AD literature: a trace of the executed
IR, but with
- recursive structure preserved, and
- control flow recorded as well.


# “Compiler Passes” 101

Just a short reminder of where `@generated` functions live:

(Figure)

When the JIT compiler hits a call of a generated function during code exectution, instead of looking
up a method from the best fitting method from the method table based on the argument types, it will
call the implementation of the generated function, which should return some AST.  This AST will then
be compiled and used (and cached for future use as well).

So, you basically have a function from types to code.  Usually, this is used for things like
generating code based on `Array` dimensions, statically known sizes of `Tuple`s, names of
`NamedTuples`s, etc.

But you can also do something fancier: custom “compiler passes”, that not only generated code based
on their argument types, but _transform_ the code of any other function.  The basic structure of
such a compiler pass `@generated` function is the following:

1. The interface of the “pass function” should contain the function and it’s arguments.
2. In the function body, you get the code of the method you want based on the arguments.
3. Transform that code like you want
4. Return it to the compiler!

One crucial thing here is that `@generated` functions allow not only `Expr` ASTs as return values,
but also `CodeInfo` – the internal representation of Julia IR.


## Example: `sloppyif`

…

## Cassette and IRTools

First, you can do all this just with `Base` in principle.  However, there  are two libraries that
make IR-based metaprogramming easier for you: Cassette and IRTools.

### Cassette

Cassette has a lot of infrastructure based on the compiler-pass technique.  You can do “plain”
compiler passes with it, but are pretty much on your own when you do so: you get the raw `CodeInfo`
and return a transformed `CodeInfo` (so the only thing you don’t have to do is to set up the
interface and find the right method yourself).

The main target (and advantage) of Cassette is one specific, very general transformation, that has
already been implemented for you: _overdubbing_.  Basically, you can overload what it means to call
a function in Julia code:

```julia
f(args...)
```

becomes

```julia
Cassette.prehook(context, f, args...)
%n = Cassette.overdub(context, f, args...)
Cassette.posthook(context, %n, f, args...)
%n
```

where you can control the behaviour by overloading `overdub` and dispatching on the type of `f` and
the “context” (on the the one hand, this is a trait to allow differntiation and composition of
several transformations; but it can also be used to preserve state within your overdubbed code).

As a concrete example (Examples are from the [Cassette docs](https://jrevels.github.io/Cassette.jl/stable/overdub.html).)

```julia
julia> @code_lowered 1/2
CodeInfo(
59 1 ─ %1 = (Base.float)(x)
   │   %2 = (Base.float)(y)
   │   %3 = %1 / %2
   └──      return %3
)
```

```julia
julia> @code_lowered Cassette.overdub(Ctx(), /, 1, 2)
CodeInfo(
59 1 ─       #self# = (Core.getfield)(##overdub_arguments#361, 1)
   │         x = (Core.getfield)(##overdub_arguments#361, 2)
   │         y = (Core.getfield)(##overdub_arguments#361, 3)
   │         (Cassette.prehook)(##overdub_context#360, Base.float, x)
   │   %5  = (Cassette.overdub)(##overdub_context#360, Base.float, x)
   │         (Cassette.posthook)(##overdub_context#360, %5, Base.float, x)
   │   %7  = %5
   │         (Cassette.prehook)(##overdub_context#360, Base.float, y)
   │   %9  = (Cassette.overdub)(##overdub_context#360, Base.float, y)
   │         (Cassette.posthook)(##overdub_context#360, %9, Base.float, y)
   │   %11 = %9
   │         (Cassette.prehook)(##overdub_context#360, Base.:/, %7, %11)
   │   %13 = (Cassette.overdub)(##overdub_context#360, Base.:/, %7, %11)
   │         (Cassette.posthook)(##overdub_context#360, %13, Base.:/, %7, %11)
   │   %15 = %13
   └──       return %15
)

```

This kind of transformation is useful and sufficient in many situations, like extraction of
computation graphs, direct forward-mode AD, logging, [concolic
execution](https://github.com/vchuravy/ConcolicFuzzer.jl), etc.

## IRTools

IRTools, on the other hand, focuses on a custom representation of IR that is different from
`CodeInfo`, and allows for easier manipulation, especially for things like
- renaming variables and inserting statements
- changing block arguments
- efficient sequential mutation of the IR

There is also a simpler interface called `@dynamo`, which works similar to a `@generated` function
(and is one internally), but allows you to generated output in the IRTools representation.

```julia
IRTools.@dynamo function sloppyifs(f, args...)
    ir = IRTools.IR(f, args...)
    ir === nothing && return

    for block in IRTools.blocks(ir)
        bblock = IRTools.BasicBlock(block)
        for b in eachindex(IRTools.branches(bblock))
            branch = bblock.branches[b]
            if IRTools.isconditional(branch)
                converted = push!(block, IRTools.xcall(:convert, Bool, branch.condition))
                bblock.branches[b] = IRTools.Branch(branch; condition=converted)
            end
        end
    end

    return ir
end
```

```julia
julia> f(x) = x ? 1 : 0
f (generic function with 1 method)

julia> @code_ir f(0.0)
1: (%1, %2)
  br 2 unless %2
  return 1
2:
  return 0
```

```julia
julia> @code_ir sloppyifs(f, 0.0)
1: (%1, %2)
  %3 = Base.getfield(%2, 1)
  %4 = Base.getfield(%2, 2)
  %5 = Base.convert(Bool, %4)
  br 2 unless %5
  return 1
2:
  return 0
```

# Implementation of IRTracker

Based on my requirements and thinking, I decided to use IRTools, since my transformation will
involve transformation that go beyond the capacities of `overdub`.  It was a long process, but the
representation I ended up was this:

```julia
julia> f(x) = x ? 1 : 0

julia> @code_ir f(0.0)
1: (%1, %2)
  br 2 unless %2
  return 1
2:
  return 0
```

```julia
julia> @code_tracked f(0.0)
1: (%3, %4, %1, %2)
  %5 = IRTracker.saveir!(%4, $(QuoteNode(1: (%1, %2)
  br 2 unless %2
  return 1
2:
  return 0)))
  %6 = IRTracker.TapeConstant(%1)
  %7 = IRTracker.trackedargument(%4, %6, $(QuoteNode(nothing)), $(QuoteNode(1)), $(QuoteNode(§1:%1)))
  %8 = IRTracker.record!(%4, %7)
  %9 = IRTracker.TapeConstant(%2)
  %10 = IRTracker.trackedargument(%4, %9, $(QuoteNode(nothing)), $(QuoteNode(2)), $(QuoteNode(§1:%2)))
  %11 = IRTracker.record!(%4, %10)
  %12 = Base.tuple()
  %13 = IRTracker.trackedvariable(%4, $(QuoteNode(%2)), %2)
  %14 = IRTracker.trackedjump(%4, 2, %12, %13, $(QuoteNode(§1:&1)))
  %15 = IRTracker.trackedreturn(%4, $(QuoteNode(⟨1⟩)), $(QuoteNode(§1:&2)))
  br 2 (%14) unless %2
  br 3 (1, %15)
2: (%16)
  %17 = IRTracker.record!(%4, %16)
  %18 = IRTracker.trackedreturn(%4, $(QuoteNode(⟨0⟩)), $(QuoteNode(§2:&1)))
  br 3 (0, %18)
3: (%19, %20)
  %21 = IRTracker.record!(%4, %20)
  return %19
```

- It is specialized for my purpose: a graph is constructed (`TapeConstant`, `TapeCall`, )







