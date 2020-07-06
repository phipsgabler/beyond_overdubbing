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

- It is specialized for my purpose: a graph is constructed (out of `TapeConstant`, `TapeCall`, etc.)
  directly in the transformed code.
- `trackedcall` is basically overdub, and might recur based on the context argument (`%4`).
- Branch arguments and jumps are recorded as well – to do this, rewriting some branching structure
  and a new block are required!
  
```julia
julia> track(x -> f(true) + x, 1)
⟨var"#7#8"()⟩(⟨1⟩, ()...) → 2::Int64
  @1: [Arg:§1:%1] var"#7#8"()::var"#7#8"
  @2: [Arg:§1:%2] 1::Int64
  @3: [§1:%3] ⟨f⟩(⟨true⟩, ()...) → 1::Int64
    @1: [Arg:§1:%1] @3#1 → f::typeof(f)
    @2: [Arg:§1:%2] @3#2 → true::Bool
    @3: [§1:&2] return ⟨1⟩
  @4: [§1:%4] ⟨+⟩(@3, @2, ()...) → 2::Int64
    @1: [Arg:§1:%1] @4#1 → +::typeof(+)
    @2: [Arg:§1:%2] @4#2 → 1::Int64
    @3: [Arg:§1:%3] @4#3 → 1::Int64
    @4: [§1:%4] ⟨add_int⟩(@2, @3) → 2::Int64
    @5: [§1:&1] return @4 → 2::Int64
  @5: [§1:&1] return @4 → 2::Int64
```

Using such traces, we can extract dynamic dependency graphs from Turing models:

```julia
julia> @model function test0(x)
    λ ~ Gamma(2.0, inv(3.0))
    m ~ Normal(0, sqrt(1 / λ))
    x ~ Normal(m, sqrt(1 / λ))
end

julia> trackdependencies(test0(1.4))
⟨2⟩ = 1.4
⟨4:λ⟩ ~ Gamma(2.0, 0.3333333333333333) → 1.0351608689025245
⟨5⟩ = /(1, ⟨4:λ⟩) → 0.9660334253749353
⟨6⟩ = sqrt(⟨5⟩) → 0.982869994137035
⟨8:m⟩ ~ Normal(0, ⟨6⟩) → -2.0155543806491205
⟨9⟩ = /(1, ⟨4:λ⟩) → 0.9660334253749353
⟨10⟩ = sqrt(⟨9⟩) → 0.982869994137035
⟨12:x⟩ ⩪ Normal(⟨8:m⟩, ⟨10⟩) ← ⟨2⟩
```

However: I may have some bad decisions, still.  It’s pretty slow even after first compilation time.
Also, I know too little about the type stability problems occuring in transformed IR to judge what
works badly in that respect.
  
  
# Conclusions

## Cassette and IRTools Revisited

I ended up with something that works a bit like Cassette, in principle:

- Every function call is overloaded, and the `trackcall` call can decide what to do based on a
  context.
- Variables are, in a sense, “tagged”, based on their IR names.

This sort of design has also happened to others before
(cf. [Jaynes.jl](https://github.com/femtomc/Jaynes.jl)).

However: you can’t easily write such an implementation in Cassette itself using just overdubbing: 

- Branches, blocks and block arguments are not reified in the contextual dispatch mechanism
- With a compiler pass you end up bookkeeping `CodeInfo` information yourself.

IRTools works much better in this case, when you completely rebuilt the IR structure yourself with
some changes to the control flow.  But even then: there are some quirks where blocks and branches
aren’t treated as first-class citizens (like iteration over a block).

(You can theoretically use IRTools within a Cassette compiler pass, but I don’t see the benefit of
that, since you don’t need Cassette anymore at that point – you lose its advantages.)

## IRTracker Reflections

Now that I have learned about and understood most of these techniques, I have come to the
conclusion that for this specific case, instead of tracking through a function whose structure is
half known, it would be more easy and appropriate to change the way Turing records models.

Lyndon White on Slack really nailed it:

> Using Cassette on code you wrote is a bit like shooting youself with a experimental mind control
> weapon, to force your hands to move like you knew how to fly a helicopter.  Even if it works, you
> still had to learn to fly the helicopter in order to program the mind-control weapon to force
> yourself to act like you knew how to fly a helicopter.

On the other hand, IRTracker turned out to be pretty interesting in itself, and useful for more than
just the described purpose.  For example, I have sometimes used it for debugging (what you really
get is equivalent to a `println` after every atomic instruction… :))

## My Wishlist

What would be cool to have for an ideal IR-based metaprogramming system:

- The capabilities of the IRTools representation, but with more features for blocks and branches
- A system that works like overdubbing, but not only for functions: blocks and branches, and SSA
  instructions themselves, should be reified and accessible as well.
  
The first goal would be achievable the existing IRTools by extension, and maybe rethinking some of
its data structures.

The second goal could in principle be factored out from IRTracker.  I imagine something like the
following:

```julia
1: (%1, %2, %3)
  %4 = Main.rand()
  %5 = %4 < %3
  br 2 (%5) unless %5
  return %2
2:
  %6 = %2 + 1
  %7 = Main.geom(%6, %3)
  return %7
```

being transformed into something like

```julia
1: (%1, %4, %2, %3)
  %5 = quoted(%4, Arg, :(%1), %1)
  %6 = quoted(%4, Arg, :(%2),%2)
  %7 = quoted(%4, Arg, :(%3), %3)
  %8 = quoted(%4, Const, Main.rand)
  %9 = overdub(%4, Call, :(%4), %8)
  %10 = quoted(%4, Const, <)
  %11 = overdub(%4, Call, :(%5), %10, %9, %7) 
  %12 = quoted(%4, CondBranch, 2, %11)
  %13 = quoted(%4, Return, %6)
  br 2 unless %11
  br 3 (%13)
2:
  %14 = quoted(%4, Const, +)
  %15 = quoted(%4, Const, 1)
  %16 = overdub(%4, Call, :(%6), %14, %6, %5)
  %17 = quoted(%4, Const, Main.geom)
  %18 = overdub(%4, Call, :(%7), %17, %16, %7)
  %19 = quoted(%4, Return, %18)
  br 3 (%19)
3 (%20):
  %21 = overdub(%4, Return, %20)
  return %20
```

(where `%4` is the context argument, as above).







