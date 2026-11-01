# Cairn

A design sketch for Cairn, a programming language with no source text, as of 2026-10-31. Programs are built and changed only by moves, and a move can only take a valid program to another valid program. The intended editor is an AI or a search process such as a genetic algorithm, not a person typing. The first target is evolving [bytebeat](#bytebeat-kernel-v0) music.

The name comes from a cairn: a stack of stones, built one stone at a time, that marks a path. Cairn is a stack language, and its workspace is a list of stacks.

Nothing here has been built. The bytebeat kernel is the proposed first step.

## The idea

Text lets a program sit in invalid states: half-typed expressions, broken references, code that doesn't compile. Refactoring tools work differently. They go from one valid program straight to another with a specific transform. Cairn makes that the only way to edit.

"Valid" means three things for every program the system can hold:

- it parses, which is trivial because there is no text to parse;
- it type-checks, including ownership in later versions;
- it runs without undefined behavior, in the sense Rust uses. Termination is not guaranteed.

A small set of fundamental moves must be able to reach any valid program, and moves stay small. A move that replaces the whole program would make the guarantee meaningless.

Two properties fall out of this. First, the space of programs is safe to explore: every offspring of a mutation or crossover runs. Second, a running program can be changed while it runs, with the guarantee that the change won't introduce undefined behavior.

## Why a stack language

The first version of this design used tree-shaped programs, like Hazel and Pantograph. Trees need holes: when you add an `if`, its branches don't exist yet, so something has to stand in for them. Holes work, but they add a whole layer of semantics.

Concatenative (stack) languages such as Forth, Joy, and Factor avoid holes. A program is a flat sequence of words, and each word takes values from the top of a stack and pushes results. Every prefix of a valid program is itself valid and runnable. Unfinished work is not a hole; it is values left on the stack that nothing has used yet.

```text
state              stack effect
(empty)            ( -- )            the nop program
3                  ( -- int )
3 4                ( -- int int )
3 4 +              ( -- int )
3 4 + dup          ( -- int int )
3 4 + dup *        ( -- int )
```

## Stack effects

A word's stack effect, written `( inputs -- outputs )`, says what it takes from the top of the stack and what it leaves. The top of the stack is on the right.

| Word | Effect | Meaning |
|---|---|---|
| `3` | `( -- int )` | push an int |
| `+` | `( int int -- int )` | add the top two |
| `dup` | `( a -- a a )` | copy the top value |
| `swap` | `( a b -- b a )` | swap the top two |
| `drop` | `( a -- )` | discard the top value |

A lowercase letter such as `a` stands for one value of any type. A word never touches what lies below its inputs, and `..a` names that untouched rest of the stack: any number of values, possibly none. `+` in full is `( ..a int int -- ..a int )`. For simple words the `..a` is implied. For words that run code, it states a contract. In `while ( ..a [..a -- ..a bool] -- ..a )`, the same `..a` in three places means the loop body must leave the stack in the shape it found it. Type theorists call `..a` a row variable.

Effects compose. Running A then B matches B's inputs against A's outputs, from the top:

- If A leaves more than B needs, the extra stays below. `3 4 dup` is `( -- int int int )`.
- If B needs more than A leaves, the composite gains inputs. `4 +` is `( int -- int )`.
- If types don't match, the composition is invalid. `true +` fails.

Two properties make stack languages suit this design. Any contiguous piece of a valid program is itself valid, because a type error can only occur between two neighbouring words and cutting the sequence can't create a new neighbour. And a program is valid exactly when its pieces' effects fit together.

## Fragments and composition

An early version of this design grew conditionals and loops in place and froze them once used. That was building a separate program inside the main one. Cairn instead treats every piece of code as an independent, immutable fragment with its own effect, and edits are compositions of fragments.

The workspace is a stack of fragments. Moves are words at a meta level whose values are programs:

| Move | Effect | Legal when |
|---|---|---|
| `lit w` | `( -- P )` makes a one-word fragment | always |
| `compose` | `( P Q -- PQ )` runs P then Q | P's outputs fit Q's inputs |
| `split` | `( P n -- P1 P2 )` cuts at position n | always |
| `quote` | `( P -- [P] )` makes a fragment that pushes P as a value | always |
| `unquote` | `( [P] -- P )` | always |
| `dup`, `swap`, `drop` | shuffle the workspace | always; fragments are immutable |
| `define-type` | adds a type and its generated words | always; nothing uses the type yet |

Editing an existing fragment means building a new one. To replace a piece, split around it and compose the left part, the new middle, and the right part; the composition only succeeds if the new middle has the same effect. A change deep inside nested quotations rebuilds each level on the way up and shares everything else, the way persistent data structures and Unison's hashed definitions do.

Fragments are content-addressed, so two identical fragments are the same fragment. Old versions persist after an edit, which the live-update model relies on. Two workspaces merge by union. Fragments never change, so they can't conflict; only names can.

"The program" is whichever fragment is designated as the entry point. The system holds many small valid programs and assembles them, rather than one program that evolves.

## Structured data

Types are defined by a move, and each definition generates the words that build and take apart its values.

Records bundle values:

```text
define type Point = { x: int, y: int }
  point    ( int int -- Point )
  unpoint  ( Point -- int int )

3 4 point unpoint dup * swap dup * +      // 25
```

Variants hold one of several cases. The generated `match` takes one quotation per case, and every case must end with the same effect `r`:

```text
define type Shape = Circle(int) | Rect(int int)
  circle  ( int -- Shape )
  rect    ( int int -- Shape )
  match   ( Shape [int -- r] [int int -- r] -- r )
```

Recursive types give lists and trees without a standard library. Each recursive type also generates a fold, which always terminates because it only recurses on smaller pieces:

```text
define type List = Nil | Cons(int List)
  fold-list  ( List r [int r -- r] -- r )
```

A type freezes once anything uses it. Adding a field or a case would otherwise require inventing values or branches at every existing use. To change a type, define the new version, write a conversion word, and move uses over one at a time. The conversion word doubles as the migration for live data when a running program switches types.

## Functions and macros

A function is a named sequence of words, and its effect is computed from its body. `: square dup * ;` is `( int -- int )`. Words come out generic for free: `: nip swap drop ;` is `( a b -- b )`.

Once something calls a word, its effect is its contract. The body can change in any way that keeps the effect. Changing the effect means defining a new word and moving callers over. The same rule governs types and live updates.

Quotations are anonymous functions. `curry ( x [x ..a -- ..b] -- [..a -- ..b] )` moves a value into a quotation. If the captured value can't be copied, the quotation can run only once, which is Rust's `Fn` and `FnOnce` distinction with no extra rules.

A function is a sequence of operations on data. A macro is a sequence of moves on programs. Because moves are words at the meta level, a macro is just a meta-level word, and it can only produce valid programs because it can only call kernel moves. `define-type` is a built-in macro. Extract-function, inline, add-a-field, and migrate-callers are library macros. An explorer that learns macros learns editing strategies, not just code.

## Control flow

Conditionals come from types. Define `Bool = False | True` and the generated `match` is `if`:

```text
if  ( ..a bool [..a -- ..b] [..a -- ..b] -- ..b )
```

Both branches must have the same effect, which falls out of composition type-checking. With ownership, both branches must consume the same owned values, so neither path can leak one. A branchless `select ( a a bool -- a )` computes both values and keeps one; the other is dropped, so `select` only works on droppable types.

Loops have one primitive:

```text
while  ( ..a [..a -- ..a bool] -- ..a )
```

The body runs at least once, and the loop repeats while the body leaves `true`. The stack shape `..a` is the loop state carried between iterations, and owned values can only live there, so iterations can't leak.

```text
0 5 [ dup rot + swap 1 - dup 0 > ] while drop      // sum 1..5 = 15
```

Termination comes in three grades. Folds and `times ( ..a int [..a -- ..a] -- ..a )` always terminate. `while` may not. During search, every run gets a step budget, and running out counts as an indeterminate result rather than a crash. `times` and the folds are library words built from `while` and `match`.

## Types and ownership

Types arrive in layers. Each layer is added when a project needs it.

| Layer | Examples | Adds |
|---|---|---|
| Base types | `i8`…`i64`, `u8`…`u64`, `f32`, `f64`, `bool` | numeric work |
| User types | records, variants, recursive types | data structures |
| Generics | `List a`, `Option a` | one definition for every element type |
| Quotations | `[..a -- ..b]` | higher-order code |
| Abilities | `copy`, `drop` | ownership |
| Units | `Meters`, `Seconds` | dimensional checks |
| Sizes | `Bits 8`, `Array 16 i32` | bit widths and checked array lengths |

Every primitive is total. Integers wrap on overflow, division by zero returns a fixed value, shifts mask their amount, and array access uses sized types or returns an `Option`. No program can trap.

Ownership uses abilities, as in the Move language. A type with `copy` can be passed to `dup`. A type with `drop` can be discarded. An `int` has both, a file handle has only `drop`, and an energy token in an artificial-life simulation has neither, so it can only be moved or consumed. Values live on the stack and there are no references, so every value has exactly one owner by construction. That rules out use-after-free and data races without a borrow checker.

Full dependent types, as in Lean, are out of scope. Lean is built on the Calculus of Inductive Constructions, where types can depend on values and proofs are values. Putting specifications in types would make every offspring correct, but every mutation would then need a proof, and the search would become theorem proving. Natural-number indices for sizes and bit widths give most of the practical benefit and need no proofs. Types guarantee safety; tests and fitness find correctness.

## The kernel

Cairn follows the LCF approach from theorem provers such as HOL Light. A valid program is an opaque type that only a small trusted kernel can construct. Macros, mutation operators, and AI-written moves can combine kernel moves freely but can't fake a valid program. In LCF terms, a valid program is a theorem, kernel moves are inference rules, and macros are tactics.

This differs from Lean, which follows the de Bruijn criterion: tactics build a complete proof term and an independent kernel re-checks it. Cairn checks every move as it happens, so every intermediate state is valid, not just the final one.

The full kernel is `compose`, `split`, `quote`, `unquote`, `define-type`, the effect checker, and the object-level primitives including `dip` and `while`. Everything else is library code written in Cairn. The kernel can't define itself; it is written in a host language and kept small enough to audit. A later step is to state the kernel rules in Lean and prove each move preserves validity, as Hazelnut did in Agda, so trust rests on Lean's kernel instead.

## Wiring diagrams

Sequential composition (`compose`) and side-by-side composition (`par`) are the two operations of a monoidal category, and with stack shuffles they describe any way of connecting computations. `par P Q` runs P on one part of the stack and Q on the part above it:

```text
P : ( a -- c )    Q : ( b -- d )    par P Q : ( a b -- c d )
par P Q  =  [P] dip Q        where dip ( ..a x [..a -- ..b] -- ..b x )

x ──[1 +]── x+1
y ──[2 *]── y*2
```

Under these two operations a Cairn program is a wiring diagram, and the diagram is its natural rendering. `swap` crosses wires, `dup` splits one, and `drop` ends one. Ownership is visible: a wire carrying a non-copyable value can't split, and a non-droppable one can't dead-end. The two halves of a `par` share no values, so they can run concurrently without data races.

The model resembles ladder logic and Function Block Diagram from IEC 61131-3, and FPGA design. A long-running program is a scan loop: loop state is the PLC's retained state or a circuit's registers, the step budget is a PLC watchdog, and the bounded forms (`fold`, fixed `times`) roughly match what can be synthesized to hardware. In hardware, copying a wire is nearly free, so linearity would apply to physical resources such as pins and memory ports instead.

## Changing a running program

Moves are typed and fragments have stable identities, so the runtime knows exactly what changed. Updates fall into tiers by difficulty:

1. Swapping a fragment for one with the same effect takes effect at the next safe point. Every loop iteration is one, because the stack is always shape `..a` there.
2. Changing code not on any active call takes effect at the next call, as in Erlang's hot code loading.
3. Changing a type of live data runs the conversion word on every live value. Each value has one owner, so no other references need fixing.
4. Changing a function mid-execution needs on-stack replacement. It is postponed.

"Won't break" means no undefined behavior, not correct behavior. PLC online edits show the extra layer needed for the second: staged acceptance and behavioral checks that every edit must pass. Effects such as I/O that already happened need their own rules.

## Uses

A genetic algorithm fits Cairn well. In classic genetic programming many offspring are invalid and get discarded; in Cairn every offspring runs.

- Mutation is a move: replace a fragment with another of the same effect, insert an effect-neutral fragment, or swap a primitive for one with the same effect.
- Crossover cuts a segment from each parent with the same effect and swaps them. It is always valid. In tree-based GP this needed strongly typed GP; here it is one effect comparison.
- Fragments that recur in winners become named library genes.
- Offspring that behave identically on test inputs collapse into one, as in bottom-up enumerative synthesis.
- Live updates let a program mutate while it runs.

Candidate projects, in rough order of effort:

1. Bytebeat music, evolved by ear. Small, fast feedback, and every program makes some sound. The first target.
2. Superoptimization of sorting networks and bit tricks. Measurable against AlphaDev and STOKE results.
3. Modular synth patches. The wiring diagram is a patch.
4. Controllers for a simulated plant (traffic lights, elevators, tanks) with safety rules as hard checks.
5. Bot arenas with co-evolution.
6. Small circuits, synthesized to an FPGA.
7. Artificial life in which energy is a linear type, so the type system enforces conservation and organisms can't copy resources. The most novel use of Cairn's ownership.

Genetic programming has a record of working on small programs and struggling to scale, and programs bloat without a size penalty. The fragment library is the main tool against both. Whether it works is the central research question.

## Bytebeat kernel v0

Bytebeat evaluates a formula of a counter `t` once per sample, typically 8,000 times a second, and plays the low 8 bits. The classic `t*(t>>5|t>>8)` is `t t 5 >> t 8 >> | *` in Cairn.

### Decisions

- **One type, `u32`.** Arithmetic wraps on overflow.
- **Total primitives.** Nothing can trap.
- **`t` is a word.** A program has effect `( -- n )` and can read `t` anywhere.
- **No quotations or loops.** `select` covers conditionals. Each sample runs in time proportional to program length, which makes the interpreter safe to run on an audio thread.
- **Immutable, content-addressed fragments.**

### Words

`a b -` means `a - b`. Effects are (inputs, outputs, maximum depth).

| Word | Effect | Meaning |
|---|---|---|
| `lit n` | (0,1,1) | push n |
| `t` | (0,1,1) | push the sample counter |
| `+ - *` | (2,1,2) | wrapping arithmetic |
| `/ %` | (2,1,2) | a zero divisor gives 0 |
| `& \| ^` | (2,1,2) | bitwise |
| `~` | (1,1,1) | bitwise not |
| `<< >>` | (2,1,2) | shift by `b & 31` |
| `< > =` | (2,1,2) | 1 or 0 |
| `select` | (3,1,3) | `a b c select` gives a if c ≠ 0, else b |
| `dup` | (1,2,2) | |
| `drop` | (1,0,1) | |
| `swap` | (2,2,2) | |
| `over` | (2,3,3) | |

### Effects with one type

With one type there are no mismatches, so an effect is three counts. Composing P then Q:

```text
extra = max(0, Q.in − P.out)
left  = max(0, P.out − Q.in)

in    = P.in + extra
out   = left + Q.out
depth = max(extra + P.depth, left + Q.depth)
```

Composition can't fail in v0. Maximum stack depth is known before running, so the interpreter allocates one fixed-size stack and overflow is impossible.

### API

```rust
pub struct Fragment {        // fields private: only the kernel constructs one
    words: Arc<[Word]>,
    effect: Effect,          // (inputs, outputs, depth)
    hash: Hash,
}

impl Fragment {
    pub fn word(w: Word) -> Fragment;
    pub fn compose(&self, q: &Fragment) -> Fragment;
    pub fn split(&self, at: usize) -> (Fragment, Fragment);
    pub fn effect(&self) -> Effect;
    pub fn words(&self) -> &[Word];
}

pub struct Program(Fragment);   // invariant: inputs == 0, outputs >= 1

impl Program {
    pub fn new(f: Fragment) -> Option<Program>;   // the only check that can fail
    pub fn sample(&self, t: u32) -> u8;           // top value & 0xFF
}
```

Rust's module privacy enforces the LCF guarantee. Every `Program` never underflows, stays within a known depth, has no undefined behavior, and runs each sample in bounded time. Extra values left on the stack are discarded.

### Library

Everything below uses only `word`, `compose`, and `split` and needs no trust.

- **Replace:** split around a segment and compose in one with the same effect.
- **Random fragment of a given effect:** grow random words, tracking the effect, until it reaches the target.
- **Mutations:** point (`+` → `^`), constant (`lit n` → `lit n±k`), insert a (1,1) fragment, delete a segment that leaves the stack unchanged, regrow a segment.
- **Crossover:** swap segments with the same (inputs, outputs). For example, swapping `5 >>` in `t t 5 >> t 8 >> | *` with `10 >> 42 &` from `t t 10 >> 42 & *` gives `t t 10 >> 42 & t 8 >> | *`.
- **Deduplication:** by hash for identical programs, and by the first 65,536 samples for programs that sound identical.

### Evolution and playback

Seed about 64 programs: random `( -- n )` fragments plus a few classic formulas. Score them interactively, with a listener keeping favourites from 4-second clips as Picbreeder did with images, or automatically by penalizing silence and noise and rewarding rhythm. Add a small length penalty. Select by tournament, breed by crossover and mutation, deduplicate, repeat.

The audio thread holds the current `Program` behind an atomic pointer. The stack is empty between samples, so a swap needs no migration. The new program plays from the next sample and `t` keeps counting, so the music changes without restarting.

### Roadmap

| Version | Adds |
|---|---|
| v1 | State carried between samples, `( s -- s sample )`, for filters and echoes. The first real live-update migration problem. |
| v2 | Fragment library and naming. |
| v3 | Types such as `f32` and `bool`; `compose` gains its first real type check. |
| v4 | Quotations, `while`, `define-type`, and the meta-level move language. |

## Open questions

- **Ownership propagation.** Rust-level safety with references needs ownership diffs, analogous to Pantograph's type diffs. Cairn sidesteps references for now; no prior structure-editor work handles them.
- **Move-set searchability.** Nobody has measured how the choice of core and derived moves affects search efficiency.
- **Output width.** Eight-bit is classic bytebeat; wider output sounds better but changes the character.
- **Fitness.** Interactive or automatic first.
- **Primitive set.** Whether 20 bytebeat words is the right count. Shifts and bitwise operators do most of the musical work.
- **Who defines types.** Letting the explorer define types widens the search; fixing them up front keeps it focused, as AlphaDev's fixed instruction set did.
- **Effects in live updates.** Hazel's resume guarantee holds only for pure code.

## Prior art

| Work | Relevance |
|---|---|
| [Hazelnut](https://arxiv.org/abs/1607.04180) (POPL 2017) | Typed structure editor calculus. Proves every edit state is well-typed and every well-typed term is constructible. |
| [Hazel live programming](https://arxiv.org/abs/1805.00155) (POPL 2019) | Runs programs with holes; filling a hole resumes evaluation, proven for pure code. |
| [Pantograph](https://arxiv.org/abs/2411.16571) (POPL 2025) | Typed edits with type diffs that propagate, such as adding a parameter and updating call sites. |
| [Grove](https://par.nsf.gov/biblio/10593045) (POPL 2025) | Commutative edit log over a graph with stable IDs; conflicts as explicit holes. |
| [Typed holes for LLMs](https://arxiv.org/abs/2409.00921) (OOPSLA 2024) | Expected types and type definitions improve LLM completions by up to an order of magnitude for small models. |
| [CodeStruct](https://arxiv.org/abs/2604.05407) (2026) | Agents editing by syntax-tree entity instead of text; large gains for weak models. |
| [AlphaDev](https://www.nature.com/articles/s41586-023-06004-9) (Nature 2023) | Builds assembly one legal instruction at a time; action pruning made the search tractable. |
| [Oxide](https://arxiv.org/abs/1903.00982) | Formal model of Rust's borrow checker; shows why ownership is non-local under edits. |
| [Unison](https://www.unison-lang.org/docs/the-big-idea/) | Content-addressed code stored as syntax trees, with names as metadata. |
| Joy, Factor, Forth, Cat, Kitten | Concatenative languages; Joy's quotations make code and data the same. |
| LCF, HOL Light, Lean | Small trusted kernel with untrusted tactics. Lean's inductive types and recursors match `define-type` and folds. |
| Move | Linear resource types with `copy` and `drop` abilities. |
| Strongly typed GP, PushGP | Typed genetic programming, and a stack language for GP that makes invalid instructions no-ops. |
| DreamCoder | Learns a library of abstractions from solved tasks. |
| Lava, Clash, PrefixRL | Hardware described with series and parallel combinators; RL over always-legal circuits. |

Hazelnut, Pantograph, Grove, AlphaDev, and Oxide were read in part. The typed-holes and CodeStruct papers were read in part through truncated HTML. The other entries are cited from memory and haven't been rechecked.
