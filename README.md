
## License

MIT is weird outside the US and doesn't cover patents, which means if someone
contributes something and changes their mind, then folks can have the code, but
don't have the right to use it.

Apache-2.0 covers patents, as in: contributors grant the rights to use all the patents
the code might be covered by, forever, which is great.

rustc is MIT+Apache-2.0 (along with a lot of crates), seems good.

EU funding for unstarted projects requires EUPL. More flexible if project already started.

MPL (Mozilla Public License) also worth looking into.

Most likely MIT+Apache-2.0 is fine.

CLA (Contributor License Agreement): would let us relicense easily, might be good
early on with small team, but probably not something we want in the long term.

## Funding

Crowd funding, grants, corporate sponsorship, different ways to arrive to sustainability.

## Overflow (int)

We should do it safely, three modes:

  * Trap
  * Saturate
  * Overflow
  
## Overflow (buffer)

Have a native slice type, do some checking.

Bound check instruction different from integer comparison.

## Performance

Main challenge for this is figuring out good data structures.

Focus for this is compilation speed rather than speed of the generated code, because of the usecases:

  * JavaScript (fast startup)
  * GPU - same problem
  
LLVM has a "sea of pointers", lots of indirection / bad cache locality, we can just have an array of
fixed-size instructions (larger instructions like metadata can be stored elsewhere and we can point to it).

This makes optimization passes take one big array and return another one, copying is very fast compared
to allocation, have to test how ergonomic this will be in practice.

## Vectorization

Have "Vector IR", but do not attempt automatic vectorization, which a) is a complex research area
and b) has limited results. Easier (much, much easier) to do `vector ➡ scalar` than the reverse.
See https://polly.llvm.org/

## SSA

Single static assignment is good, not sure about Phi nodes, must research various passes and see
when it is really convenient (rather than arrows pointing in the opposite direction).

We probably want SSA *always*, even when not performing any optimizations, because it is useful for codegen.

## API

API stability is a very important thing, this means the text IR but also the C public API, which
should never break inside of a major release.

The Rust public API can change more often (it'll also be safer).

API should include:

  * Builder API, to build IR without using text
  * Serialiation/Deserialization API
  * Driver API, to actually optimize/codegen

## Register allocation

Different heuristics, allow selecting between them through IR annotations, there's cases
when you want high register usage, and cases where you want to leave some available (for
example for other threads on (allocated) 256-register architectures (CUDA).

> The idea is that if thread A uses all 256 registers and thread B needs 32, then 32
registers need to be saved to stack and restored before resuming thread A. Also, you
might think you need 256 registers, but some of them can be moved out of a loop
(see LICM, Loop Invariant Code Motion).

## Optimizations

Passes to realistically target first:

  * Legalization (e.g. MIPS only has 64-bit integer arithmetic, ARM only has 64-bit & 32-bit, what if we need 16-bit wide types? Masking etc.)
  * Strength reduction (e.g. SHUFFLE => BSWAP)
  * `mem2reg`: Promoting memory to register
  * `argpromotion`: Promote 'by reference' arguments to scalars
  
But also:

  * `dce` (Dead code elimination)
  * `always-inline`
  * `memcpyopt`

## Debug

We need great tooling at all steps, examples of nice things:

  * Syntax highlighting for IR
  * Debugging what passes
  * Diffs before/after DCE (dead code elimination)
  * Graphs (graphviz), for SSA tree, dependency analysis

## DWARF & friends

rust's msvc target is fresh, LLVM got Windows calling conventions & debug info only recently (2017-2018?),
we should plan for it, apparently it's hard to get right. (But it is very important)

## Error handling

Have actual (nice) errors, crashes should only be caused by compiler bugs, not API misuse.

## Plugins / modularity

All of it needs to be modular so that third-party code doesn't *have* to live in a fork
or in the mainline repository.

## Type system

Makes sense to look at LLVM's, keeping in mind that:

  * `poison` is bad
  * `undef` is slightly less bad
  * don't pander to C-like languages, do the safe thing when possible

### Types

  * `void`, `function`,
  * integers: uN, iN, same instructions for add/sub/mul/div, N in {8, 16, 32, 64}
  * floating point types
  * pointers: two types:
    * one safe (can only take the address of something we know exists, no arithmetic),
    * one unsafe (everything's allowed)
    * also have to think about address spaces, they have numbers, that mean different things on target arch
    * larger discussion about IR target independence
  * `vector` type (SIMD, with width)
  * `label` type (because of jump tables)
  * `metadata` type: more structured than just tuples, maybe at least a hash
  * `struct`
  * `opaque` type (not opaque struct), that you can have pointers to, but not values of
  * `bool` type (not i1) - it's special-cased anyway
  * `arrays` (fixed size)
  * `slices` (address + length)
  
### Constants

  * `undef` is a value not a type, propagates until we try to optimize or codegen it, then error out
  * no `zeroinitializer`, make compiler devs be explicit
  * no poison values, not interested in speculative execution
  * const operations are important, would be interested in generating runtime instructions for these so you can step through them and figure out why the const result is wrong
