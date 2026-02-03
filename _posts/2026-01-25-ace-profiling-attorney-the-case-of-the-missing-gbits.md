---
title: Ace Profiling Attorney - The Case of the Missing Gbits
categories: [Programming, Profiling]
tags: [Rust, kernel, networking]
---

> **Disclaimer:** This is not a language-war post. No “X vs Y”.  
> This is a profiling detective story about my Rust TCP forwarder [`oi`](https://github.com/DaZuo0122/oxidinetd).

---
## 0) Prologue — The Courthouse Lobby

> **Me:** I wrote a Rust TCP port forwarder. It works. It forwards.  
> 
> **Inner Prosecutor (Phoenix voice):** *Hold it!* “Works” is not a metric. How fast?  
> 
> **Me:** Not fast enough under load.  
> 
> **Inner Prosecutor:** *Objection!* “Not fast enough” is an emotion. Bring evidence.  
> 
> **Me:** Fine. I’ll bring **perf**, **strace**, and a **flamegraph**.  
> 
> **Inner Prosecutor:** Good. This court accepts only facts.

## 1) The Crime Scene — Setup & Reproduction

**Me:** Single machine, Debian 13. No WAN noise, no tunnel bottlenecks.  

**Inner Prosecutor:** *Hold it!* If it’s “single machine”, how do you avoid loopback cheating?  

**Me:** Network namespaces + veth. Local, repeatable, closer to real networking.

### Environment

- Debian 13
- Kernel: `6.12.48+deb13-amd64`
- Async Runtime: `smol`
- Test topology: `ns_client → oi (root ns) → ns_server` via veth

### Reproduction commands

**Exhibit A: Start backend server in `ns_server`**

```bash
sudo ip netns exec ns_server iperf3 -s -p 9001
````

**Exhibit B: Run client in `ns_client` through forwarder**

```bash
sudo ip netns exec ns_client iperf3 -c 10.0.1.1 -p 9000 -t 30 -P 8
```

**Inner Prosecutor:** *Hold it!* Why `-P 8`?  

**Me:** Because a forwarder can look fine in `-P 1` and fall apart when syscall pressure scales.  

**Inner Prosecutor:** …Acceptable.  

---

## 2) The Suspects — What Could Be Limiting Throughput?

**Me:** Four suspects.

1. **CPU bound** (pure compute wall)
2. **Kernel TCP stack bound** (send/recv path, skb, softirq, netfilter/conntrack)
3. **Syscall-rate wall** (too many `sendto/recvfrom` per byte)
4. **Runtime scheduling / contention** (wake storms, locks, futex)

**Inner Prosecutor:** *Objection!* That’s too broad. Narrow it down.  

**Me:** That’s what the tools are for.

---

## 3) Evidence #1 — `perf stat` (The Macro View)

**Me:** First I ask: are we burning CPU, thrashing schedulers, or stalling on memory?

**Command:**

```bash
sudo perf stat -p $(pidof oi) -e \
  cycles,instructions,cache-misses,branches,branch-misses,context-switches,cpu-migrations \
  -- sleep 33
```

**What I’m looking for:**

* Huge `context-switches` → runtime thrash / lock contention
* Huge `cpu-migrations` → unstable scheduling
* Very low IPC + huge cache misses → memory stalls
* Otherwise: likely syscall/kernel path

Output: 

```text
 Performance counter stats for process id '209785':

   113,810,599,893      cpu_atom/cycles/                                                        (0.11%)
   164,681,878,450      cpu_core/cycles/                                                        (99.89%)
   102,575,167,734      cpu_atom/instructions/           #    0.90  insn per cycle              (0.11%)
   237,094,207,911      cpu_core/instructions/           #    1.44  insn per cycle              (99.89%)
        33,093,338      cpu_atom/cache-misses/                                                  (0.11%)
         5,381,441      cpu_core/cache-misses/                                                  (99.89%)
    20,012,975,873      cpu_atom/branches/                                                      (0.11%)
    46,120,077,111      cpu_core/branches/                                                      (99.89%)
       211,767,555      cpu_atom/branch-misses/          #    1.06% of all branches             (0.11%)
       245,969,685      cpu_core/branch-misses/          #    0.53% of all branches             (99.89%)
             1,686      context-switches
               150      cpu-migrations

      33.004363800 seconds time elapsed
```


**Low context switching**:
	
   - context-switches: 1,686 over ~33s → ~51 switches/sec
	
   - cpu-migrations: 150 over ~33s → ~4.5/s → very stable CPU placement

**CPU is working hard**:
	
   - 237,094,207,911 cpu_core instructions
	
   - IPC: 1.44 (instructions per cycle) → not lock-bound or stalling badly
	
**Clean cache, branch metrics**:

   - cache-misses: ~5.3M (tiny compared to the instruction count)
	
   - branch-misses: 0.53%


	
**Inner Prosecutor:** *Hold it!* You didn’t show the numbers.  

**Me:** Patience. The next exhibit makes the culprit confess.  

---

## 4) Evidence #2 — `strace -c` (The Confession: Syscall Composition)

**Me:** Next: “What syscalls are we paying for?”

**Command:**

```bash
sudo timeout 30s strace -c -f -p $(pidof oi)
```

**What I expect if this is a forwarding wall:**

* `sendto` and `recvfrom` dominate calls
* call counts in the millions

Output (simplified):

```text
sendto   2,190,751 calls   4.146799s  (57.6%)
recvfrom 2,190,763 calls   3.052340s  (42.4%)
total syscall time:        7.200789s
```


(A) **100% syscall/copy dominated:**
	
- Almost all traced time is inside:

	- `sendto()` (TCP send)

	- `recvfrom()` (TCP recv)

(B) **syscall rate is massive**

- Total send+recv calls:

	- ~4,381,500 syscalls in ~32s
	- → ~137k `sendto` per sec + ~137k `recvfrom` per sec
	- → ~274k syscalls/sec total


**Inner Prosecutor:** *Objection!* Syscalls alone don’t prove the bottleneck.  

**Me:** True. So I brought a witness.

---

## 5) Evidence #3 — FlameGraph (The Witness)

**Me:** The flamegraph doesn’t lie. It testifies where cycles go.

**Commands:**

```bash
sudo perf record -F 199 --call-graph dwarf,16384 -p $(pidof oi) -- sleep 30
sudo perf script | stackcollapse-perf.pl | flamegraph.pl > oi.svg
```

**What the flamegraph showed (described, not embedded):**

* The widest towers were kernel TCP send/recv paths:

  * `__x64_sys_sendto` → `tcp_sendmsg_locked` → `tcp_write_xmit` → ...
  * `__x64_sys_recvfrom` → `tcp_recvmsg` → ...
* My userspace frames existed, but were comparatively thin.
* The call chain still pointed into my forwarding implementation.


**Inner Prosecutor:** *Hold it!* So you’re saying… the kernel is doing the heavy lifting?  

**Me:** Exactly. Which means my job is to **stop annoying the kernel** with too many tiny operations.  

---

## 6) The Real Culprit — A “Perfectly Reasonable” Copy Loop

**Me:** Here’s the original relay code. Looks clean, right?

```rust
let client_to_server = io::copy(client_stream.clone(), server_stream.clone());
let server_to_client = io::copy(server_stream, client_stream);

futures_lite::future::try_zip(client_to_server, server_to_client).await?;
```

**Inner Prosecutor:** *Objection!* This is idiomatic and correct.  

**Me:** Yes. That’s why it’s dangerous.  

**Key detail:** `futures_lite::io::copy` uses a small internal buffer (~8KiB in practice).
Small buffer → more iterations → more syscalls → more overhead.

If a forwarder is syscall-rate bound, this becomes a ceiling.

---

## 7) The First Breakthrough — Replace `io::copy` with `pump()`

**Me:** I wrote a manual pump loop:

* allocate a buffer once
* `read()` into it
* `write_all()` out
* on EOF: `shutdown(Write)` to propagate half-close

```rust
async fn pump(mut r: TcpStream, mut w: TcpStream, buf_sz: usize) -> io::Result<u64> {
    let mut buf = vec![0u8; buf_sz];
    let mut total = 0u64;

    loop {
        let n = r.read(&mut buf).await?;
        if n == 0 {
            let _ = w.shutdown(std::net::Shutdown::Write);
            break;
        }
        w.write_all(&buf[..n]).await?;
        total += n as u64;
    }
    Ok(total)
}
```

Run both directions:

```rust
let c2s = pump(client_stream.clone(), server_stream.clone(), BUF);
let s2c = pump(server_stream, client_stream, BUF);

try_zip(c2s, s2c).await?;
```

**Inner Prosecutor:** *Hold it!* That’s just a loop. How does that win?  

**Me:** Not the loop. The **bytes per syscall**.  

---

## 8) Exhibit C — The Numbers (8KiB → 16KiB → 64KiB)

### Baseline: ~8KiB (generic copy helper)

Throughput:

```text
17.8 Gbit/s
```


**Inner Prosecutor:** *Objection!* That’s your “crime scene” number?  

**Me:** Yes. Now watch what happens when the kernel stops getting spammed.  


### Pump + 16KiB buffer

Throughput:

```text
28.6 Gbit/s
```

`strace -c` showed `sendto/recvfrom` call count dropped:

```text
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 57.80   14.590016      442121        33           epoll_wait
 28.84    7.279883           4   1771146           sendto
 13.33    3.363882           1   1771212        48 recvfrom
  0.02    0.003843          61        62        44 futex
  0.01    0.001947          12       159           epoll_ctl
...
------ ----------- ----------- --------- --------- ----------------
100.00   25.242897           7   3542787       143 total
```


**Inner Prosecutor:** *Hold it!* That’s already big. But you claim there’s more?  

**Me:** Oh, there’s more.  


### Pump + 64KiB buffer

Throughput:

```text
54.1 Gbit/s (best observed)
```

`perf stat` output:

```text
Performance counter stats for process id '893123':

   120,859,810,675      cpu_atom/cycles/                                                        (0.15%)
   134,735,934,329      cpu_core/cycles/                                                        (99.85%)
    79,946,979,880      cpu_atom/instructions/           #    0.66  insn per cycle              (0.15%)
   127,036,644,759      cpu_core/instructions/           #    0.94  insn per cycle              (99.85%)
        24,713,474      cpu_atom/cache-misses/                                                  (0.15%)
         9,604,449      cpu_core/cache-misses/                                                  (99.85%)
    15,584,074,530      cpu_atom/branches/                                                      (0.15%)
    24,796,180,117      cpu_core/branches/                                                      (99.85%)
       175,778,825      cpu_atom/branch-misses/          #    1.13% of all branches             (0.15%)
       135,067,353      cpu_core/branch-misses/          #    0.54% of all branches             (99.85%)
             1,519      context-switches
                50      cpu-migrations

      33.006529572 seconds time elapsed
```

`strace -c` output:

```text
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 54.56   18.079500      463576        39           epoll_wait
 27.91    9.249443           7   1294854         2 sendto
 17.49    5.796927           4   1294919        51 recvfrom
...
------ ----------- ----------- --------- --------- ----------------
100.00   33.135377          12   2590253       158 total
```


**Inner Prosecutor:** *OBJECTION!* `epoll_wait` is eating the time. That’s the bottleneck!  

**Me:** Nice try. That’s a classic trap.  

---

## 9) Cross-Examination — The `epoll_wait` Trap

**Me:** `strace -c` measures time spent *inside syscalls*, including time spent **blocked**.

In async runtimes:

* One thread can sit in `epoll_wait(timeout=...)`
* Other threads do `sendto/recvfrom`
* `strace` charges the blocking time to `epoll_wait`

So `epoll_wait` dominating **does not** mean “epoll is slow”.
It often means “one thread is waiting while others work”.

**What matters here:**

* `sendto` / `recvfrom` call counts
* and how they change with buffer size

---

## 10) Final Explanation — Why 64KiB Causes a “Nonlinear” Jump

**Inner Prosecutor:** *Hold it!* You only reduced syscall calls by ~some percent. How do you nearly triple throughput?  

**Me:** Because syscall walls are **nonlinear**.  

A forwarder’s throughput is approximately:

> **Throughput ≈ bytes_per_syscall_pair × syscall_pairs_per_second**

If you’re syscall-rate limited, increasing `bytes_per_syscall_pair` pushes you past a threshold where:

* socket buffers stay fuller
* the TCP window is better utilized
* each stream spends less time in per-chunk bookkeeping
* concurrency (`-P 8`) stops fighting overhead and starts helping

Once you cross that threshold, throughput can jump until the next ceiling (kernel TCP, memory bandwidth, iperf itself).

That’s why a “small” change can create a big effect.

---

## 11) Sentencing — Trade-offs

**Inner Prosecutor:** *Objection!* Bigger buffers waste memory!  

**Me:** Sustained.  

A forwarder allocates **two buffers per connection** (one per direction).

So for 64KiB:

* ~128KiB per connection (just for relay buffers)
* plus runtime + socket buffers

That’s fine for “few heavy streams”, but it matters if you handle thousands of concurrent connections.

In practice, the right move is:

* choose a good default (64KiB is common)
* make it configurable
* consider buffer pooling if connection churn is heavy

---

## Epilogue — Case Closed (for now)

**Inner Prosecutor:** So the culprit was…  

**Me:** A perfectly reasonable helper with a default buffer size I didn’t question.  

**Inner Prosecutor:** And the lesson?  

**Me:** Don’t guess. Ask sharp questions. Use the tools. Let the system testify.  


> **Verdict:** Guilty of “too many syscalls per byte.”  
> 
> **Sentence:** 64KiB buffers and a better relay loop.  

--- 

## Ending

This was a good reminder that performance work is not guessing — it’s a dialogue with the system:

1. Describe the situation
2. Ask sharp questions
3. Use tools to confirm
4. Explain the results using low-level knowledge
5. Make one change
6. Re-measure

And the funniest part: the “clean” one-liner `io::copy` was correct, but its defaults were hiding a performance policy I didn’t want.

> **Inner Prosecutor:** “Case closed?”
>
> **Me:** “For now. Next case: buffer pooling, socket buffer tuning, and maybe a Linux-only `splice(2)` fast path — carefully, behind a safe wrapper.”

---