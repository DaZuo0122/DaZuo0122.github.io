---
title: Building Babel - a fuzzy LLM vs the OS
categories: [Thoughts]
tags: [os]
math: true
---

*A post about reliability, memory, and the compiler we didn’t mean to write.*

When people talk about “prompt engineering,” it often sounds like a bag of tricks: write clearer instructions, add examples, constrain the format, keep history short, and pray. But if you zoom out, the pattern looks less like copywriting and more like systems engineering. We’re trying to run workloads on a machine whose “CPU” is probabilistic, whose “RAM” is fixed-size, and whose caching behavior depends on keeping the same prefix intact.


That framing is useful, because it pushes us toward familiar tools: define an ISA, build a runtime, control errors with redundancy and verification, and manage memory with explicit lifetimes. 

---

## 0 — The foundation: why the OS analogy is structurally correct

### 0.1 A minimal machine model

If you strip away the marketing, an LLM session is a constrained compute device with:

* a **finite internal working set** (context window),
* a **state-dependent transition function** (next-token generation),
* and an **acceleration cache** tied to previously seen prefixes (KV cache).

That is already enough to justify an OS framing, because OS thinking starts from exactly these questions:

* What is my *working set limit*?
* What is *resident* vs *non-resident* state?
* Where do failures originate: computation, I/O, or state corruption?
* How do I bound resource usage and isolate side effects?

So we don’t start with “LLM = OS.”
We start with “LLM session = state machine with bounded memory + caching,” and OS is the natural language we use to design such machines.

### 0.2 Formal mapping: LLM as a bounded-state transducer

A practical theoretical model looks like this:

* Let $X$ be the set of possible contexts (token sequences) with max length $N$.
* Let $Y$ be token outputs.
* The model implements a stochastic policy:
  $$
  \pi(y \mid x)
  $$
  where $x \in X$.

In each interaction, you append some new tokens to $x$, then the model emits tokens $y$, producing a new context $x' = \text{append}(x, y)$, then truncation/packing happens due to the context limit $N$.

From an OS perspective, the key point is not stochasticity. The key point is **boundedness**:

* The model is not a Turing machine with unlimited tape.
* It is a finite-memory device (finite context window) with expensive state transitions.

That’s why memory management dominates in practice.

### 0.3 “Main context ⇒ RAM” : the working-set equivalence

In OS terms, RAM is defined by three properties:

1. **bounded capacity**
2. **fast access for computation**
3. **content determines behavior** (program correctness depends on which pages are resident and what they contain)

An LLM context window has exactly those properties:

* bounded capacity: fixed token limit $N$
* fast access: everything in-context is “directly addressable” by attention
* content determines behavior: the probability distribution $\pi(\cdot\mid x)$ changes when $x$ changes

That’s enough to justify the equivalence “context behaves like RAM,” even though the representation isn’t bytes.

If you want a stronger OS-flavored claim: the LLM’s output is effectively a function of its **resident working set**. If important facts fall out of the window, the model behaves as if the memory page was evicted.

### 0.4 “Immutable prefix ⇒ kernel text” is about invariants and reentrancy

Kernel code in an OS is special for reasons that map cleanly:

* it establishes invariants (security model, syscalls, scheduling policy)
* it is expected to be stable/reentrant across workloads
* it is always “mapped” (logically present) and influences everything

Similarly, the immutable prefix (role, policies, high-level goals, tool contracts) is the part of context we want:

* stable across turns,
* consistently applied to all tasks,
* and reusable for performance.

This isn’t metaphorical; it’s a design constraint. If you mutate “kernel invariants” mid-execution, you get undefined behavior in both worlds.

### 0.5 “KV cache ⇒ TLB/cache” is justified by prefix-dependent reuse

KV cache reuse is not magic; it’s a computational caching mechanism:

* If the prefix is identical, the model can reuse previously computed key/value tensors for those tokens.
* If the prefix changes, cache reuse degrades.

That is the same structural property as a TLB/cache: stable mappings/prefixes produce high hit rates; churn kills locality.

From an OS dev perspective, this creates an optimization target: keep the “kernel prefix” stable to maximize cache locality across turns.

### 0.6 “Skills compiler” is justified by separation of concerns

OS devs separate *policy* from *mechanism*:

* policy decides what to do (scheduler policy, VM policy)
* mechanism executes it deterministically (context switch, page fault handler)

This is exactly what the compiler/runtime split is doing:

* **LLM is policy**: produce a plan from a messy spec
* **Sandbox runtime is mechanism**: execute primitives deterministically

So “NL → instructions compiler” isn’t just a nice idea; it’s literally applying a core OS design principle: isolate fuzzy/high-level decisions from low-level mechanisms, so failures can be contained and reasoned about.

### 0.7 Why binaries (success/failure) matter: observability and debugging

OS engineering relies on observability:

* syscalls return error codes
* page faults are explicit
* sched events are traceable
* invariants can be asserted

When you compile skills into basic instructions with binary status, you create **syscall-like boundaries**. That gives you:

* localized failure (which instruction failed)
* structured error types (timeout vs empty input vs permission)
* the ability to retry/repair minimally

This is the real foundation of the “ECC/control loop” later: error correction requires a detectable syndrome. Binary instruction results are your syndrome.

---


## Part 1 — Reliability: the fuzzy ALU problem, and an ECC-shaped solution

### The reliability issue: deterministic execution vs probabilistic compilation

In a classic machine, the critical property is that execution is deterministic: given the same instruction stream and machine state, you get the same result. That’s what makes debugging possible, and it’s why “bit flips” are an exceptional event handled by ECC, parity checks, and redundancy.

LLMs invert that. The core model is best understood as a conditional distribution $P(y \mid x)$: the next token depends on the prompt/context. Even if you force deterministic decoding, the *system-level behavior* remains fragile because the mapping from a messy human request to an internal strategy is not explicit and not stable. Small context changes, minor phrasing differences, or irrelevant baggage in the prompt can flip the “mode” the model enters. In practice, this looks like the ALU occasionally returning the wrong result, except the “wrongness” is semantic, not bit-level.

A direct way to improve reliability is to reduce the amount of “semantic work” the model must do while it is producing final outputs. Instead of asking the LLM to execute tasks in free-form language, we ask it to **compile** the request into a small set of **deterministic primitives**. Then we run those primitives in a runtime we control.

### The solution: compile to a tiny ISA (`read/write/grep/exec`) and execute in a strict sandbox

Assume we define a minimal instruction set:

* `read`  — load data from a declared source
* `write` — store a value into a declared state slot
* `grep`  — deterministically match/filter/extract
* `exec`  — call a deterministic tool (curl, a parser, a local command) inside a sandbox

Now, a “skill” is no longer a blob of natural language. It becomes a program: a sequence of these instructions. Web search becomes “write the query → exec curl → read the response → grep for indicators → write the extracted results.” Local doc search becomes “read the corpus → grep for pattern → write the match list.”

You can visualize the pipeline like a compiler+runtime split:

```
User request (natural language)
          |
          v
+-------------------+
|  Skills Compiler  |   (LLM: NL -> ISA plan)
+-------------------+
          |
          v
 ISA Program: [write, exec, read, grep, ...]
          |
          v
+-------------------+
| Sandbox Runtime   |   (deterministic executor + logs)
+-------------------+
          |
          v
Structured result + trace (success/failure per instruction)
```

This architecture deliberately moves uncertainty into one place: compilation. Execution becomes observable and mostly deterministic.

### Why this increases reliability

Let $U$ be the user request (the “spec”), $C$ be the compiler (LLM), $P$ the produced plan (ISA program), $R$ the runtime, and $O$ the observed output. Let $V(U,O)\in{0,1}$ be a checker that says whether the output satisfies the request (even a weak checker helps).

Because the runtime is deterministic and instrumented, the overall success probability decomposes conceptually into:

* how often the compiler emits a correct plan, and
* how often correct plans actually pass in the runtime environment.

Informally:

$$
\Pr[V=1] \approx \Pr[P\ \text{correct}] \cdot \Pr[V=1\mid P\ \text{correct}]
$$

If your runtime is strict and your primitives are deterministic, $\Pr[V=1\mid P\ \text{correct}]$ is high. That’s the central win: you turn “LLM unpredictability everywhere” into “LLM uncertainty mainly at compilation time.” Once the failure surface is concentrated, you can apply ECC-like techniques there.

### ECC for compilation: redundancy + decoding

ECC works by adding redundancy and then decoding based on constraints that detect and correct errors. You can do the same for plans:

1. generate multiple candidate plans $P_1, …, P_k$
2. statically validate them (types, allowed effects, resource access)
3. partially execute cheap prefixes if needed
4. select the plan that passes checks / yields valid outputs

Even without strong independence assumptions, plan diversity (different “compilation prompts,” constraints, or decomposition templates) tends to decorrelate failure modes. With a checker, the selection step becomes the decoder.

Here’s an OS-style view of this redundancy:

```
             +------------------+
U ----------> |  Compiler (LLM)  |
             +------------------+
                |    |    |
               P1   P2   P3    ... (redundant plans)
                \    |    /
                 \   |   /
                  v  v  v
            +-------------------+
            |  Verifier/Decoder |
            +-------------------+
                     |
                     v
                 choose P*
                     |
                     v
              +----------------+
              |  Runtime Exec  |
              +----------------+
```

### Control theory as practical “ECC glue”: localize failures, repair minimally

Once every step is an instruction with a success/failure status, you gain something that free-form prompting almost never gives you: precise failure localization.

* `exec curl` failed → network/tool failure; retry/backoff or switch endpoint
* `grep` failed → query too narrow or wrong target; rewrite only the grep or the query
* `read` failed → missing resource; fix source selection
* `write` failed → schema/invariant violation; repair the state shape

This is where a control loop becomes natural. The “plant” is your plan+runtime; the “sensor” is the instruction trace; the “controller” decides whether to retry, patch the plan, or escalate. You don’t need to force PID math onto it, but the conceptual mapping is clean: proportional fixes handle immediate errors, integral memory of repeated errors adjusts defaults, and derivative-like logic prevents oscillations (e.g., endless retries).

---

## Part 2 — Memory: context is RAM, and we need a real allocator

### The memory issue: fixed context window behaves like physical memory

If the LLM is single-threaded, the context window is its working memory. It’s fixed size. It is expensive. It gets polluted. And when it overflows, you start doing swap-like hacks: aggressive summarization, pruning, or retrieval-augmented generation (RAG).

The OS analogy isn’t decorative here. It’s operationally accurate:

* **context window** is physical RAM (bounded)
* **immutable prefix** is kernel code mapped and hot
* **KV cache** behaves like a cache/TLB where prefix stability matters
* “adding more prompt” is “allocating memory,” and you can run out

So memory management isn’t just “write less.” It’s deciding what stays resident, what gets offloaded, and how to keep the working set small.

### The solution: arena allocator + staged lifetimes + lazy loading + offload

Rust’s arena allocator is a good mental model because it matches a core workflow truth: most intermediate data is useful only during a phase. You don’t need fine-grained frees; you need cheap allocation and bulk reclamation at phase boundaries.

We treat the context as an append-only arena with a bump pointer. We also define “stages” in a skill:

1. compile plan
2. execute primitives
3. validate
4. commit minimal results
5. free intermediates

At each stage start, we take a checkpoint. During the stage, we append logs and intermediate outputs. When the stage completes, we write a compact “commit record” (summary + hashes + pointers), then reset the arena to the checkpoint. That gives you explicit lifetimes and prevents context leaks.

Here is the memory layout you proposed, as an OS-style diagram:

```
+----------------------------------------------------------------------------------+
|                           LLM Context Window (fixed)                             |
+----------------------------------------------------------------------------------+
|  Immutable Prefix  |  Disk Index (pointers)  |  Working Set (stage arena) | Free |
|  (kernel text)     |  (skills/tools/docs)    |  (plan + outputs + trace)  |      |
+----------------------------------------------------------------------------------+
```

And here is the arena checkpoint mechanism:

```
Stage S begins:
    bump_ptr = B
    checkpoint[S] = B

During stage:
    append(plan fragments)
    append(exec outputs)
    append(grep matches)
    append(validation notes)

Stage S commits:
    write(commit_record[S])   <-- compact, structured
    bump_ptr = checkpoint[S]  <-- bulk free (arena reset)
```

### Why this works

This approach gives two strong properties that are “theoretical” in the systems sense.

**(1) Bounded working set by construction.**
If you cap the amount of intermediate data a stage is allowed to keep (or the number/size of instruction traces), then your memory usage becomes bounded not by “how long the conversation is,” but by “how big the current stage is.” That’s exactly how a well-structured kernel avoids unbounded growth: scope lifetimes, control allocations, and reclaim aggressively when a phase ends.

**(2) Reduced need for lossy compaction.**
Most prompt-based “memory management” relies on summarization, which is effectively lossy compression. An arena+stages approach keeps the majority of the “bulk data” as structured runtime artifacts (commit records, hashes, pointers), not as raw narrative. That reduces how often you must compress meaning into fewer tokens. Prune and RAG become swap mechanisms rather than the default mode.

### Lazy loading: the disk index is your page table

The “disk index” part is important. It’s the page table of your LLM OS: it stores *addresses* (skill names, tool contracts, doc IDs), not the content itself. You keep this index resident because it’s small and highly reusable. When a plan needs a resource, it pages it in via deterministic steps:

* `exec(fetch_doc, id)`
* `read(output)`
* `grep(...)`

This is consistent with your “treat prune and RAG as last option” stance. The system shouldn’t drag content into RAM unless it’s demanded by the current working set.

### Prune & RAG: swap/compaction when you must, not when you feel anxious

Eventually you will hit memory pressure. When you do, you have two OS-like fallback moves:

* **pruning**: compact resident memory by removing older low-value regions
* **RAG/swap**: offload bulk to external storage and page it back when needed

The key is policy: these are last-resort mechanisms, triggered by pressure and guided by working-set logic, not something you do on every turn.

---

## Are we building the Tower of Babel?

In mood terms, yes: we are building a translation layer from human intent to machine action, and history says that “universal translators” are where complexity goes to multiply.

But there’s a difference between *Babel as confusion* and *Babel as a compiler toolchain*. The dangerous version is when natural language directly triggers actions without strict interfaces. That system becomes un-debuggable and hard to secure because the space of meanings is unbounded and the boundary between interpretation and execution is blurry.

The OS-shaped version is constrained:

* the target language is not “whatever the model feels like,” it’s a tiny ISA
* the runtime is deterministic and instrumented
* the boundary between compilation (fuzzy) and execution (controlled) is explicit
* failures are localized to instruction traces
* memory is managed with lifetimes and checkpoints, not hope

So yes, we’re building Babel—but the grown-up, systems version. Not a tower of endless languages, but a tower with a syscall table, an allocator, and something that looks suspiciously like ECC wrapped around the only fuzzy component: compilation.
