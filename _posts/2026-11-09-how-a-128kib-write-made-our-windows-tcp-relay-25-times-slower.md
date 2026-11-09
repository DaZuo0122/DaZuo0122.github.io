---
title: "How a 128 KiB Write Made Our Windows TCP Relay 25 Times Slower"
date: 2026-11-09 21:05:00 +0100
categories: [Programming, Profiling]
tags: [tcp, windows, etw, tokio, rio, networking, Rust]
description: "An ETW investigation of TCP window updates, silly-window avoidance, and a relay that spent its time waiting to read."
render_with_liquid: false
---

> [Previous post](https://dazuo0122.github.io/posts/ace-profiling-attorney-the-case-of-the-missing-gbits/)

The fastest way to make our Windows TCP relay slow was to give it larger writes. With 32 KiB writes from a local iperf3 client, the default async path transferred around 12 Gb/s. With 128 KiB writes, it transferred around half a gigabit. The sender, receiver, relay binary, and loopback interface were the same. The relay process was mostly idle in the slower run.

We initially treated this as an I/O-backend problem. An experimental Windows Registered I/O (RIO) relay reached a few gigabits with the 128 KiB workload, so RIO seemed to have found a faster way through Windows. But at 32 KiB, RIO transferred only about 1 Gb/s while the async path was an order of magnitude faster. The ranking reversed when the client changed one argument.

The decisive observation came later, from a TCP event trace. In the slow async case, the iperf sender repeatedly entered a silly-window-avoidance path and stopped sending for around 15 ms at a time. The proxy's read task could not consume bytes that had not arrived. A separate RIO trace showed its own receive-window and zero-window stalls. These are distinct problems with a shared theme: a relay's performance depends on the timing of two TCP connections and the cadence with which the middle process drains one and feeds the other.

This post is a worked investigation of that behavior. It uses rounded local measurements. Hostnames, exact hardware identifiers, process IDs, socket addresses, and raw ETL identifiers are left out; the Windows API names, event names, buffer sizes, and relationships between measurements are retained because they are necessary to understand the mechanism.

## Two connections, one deceptively simple copy loop

The relay accepts a client socket, connects a backend socket, and runs two independent copy directions. In the Windows async path, each direction starts with an 8 KiB buffer, grows to at most 32 KiB after repeated full reads, and shrinks after repeated small reads or an idle period. The essential data path is still conventional:

```rust
loop {
    let n = reader.read(&mut buffer).await?;
    if n == 0 {
        writer.shutdown().await?;
        break;
    }
    writer.write_all(&buffer[..n]).await?;
    adjust_buffer_from_recent_reads(n);
}
```

For an upload, the iperf client writes into the first TCP connection. The relay reads from that connection and writes into a second TCP connection to the server:

```text
iperf client             relay                          iperf server
    send()  ───── TCP A ───── read() / write_all() ───── TCP B ───── recv()
                 ↑ receiver window A                    ↑ receiver window B
```

The diagram matters because “the TCP window” is ambiguous here. TCP A's sender learns how much space the relay advertises. TCP B's sender is the relay and learns how much space the server advertises. Either connection can block progress. It also matters that iperf's `-l` controls application write length, while the relay's read buffer controls how those bytes are consumed. A 128 KiB write need not appear as one 128 KiB relay read, and a 32 KiB relay read need not become one TCP segment.

We kept the test single-stream and local so that the effects were easy to repeat. A benchmark process started a fresh relay per case, selected dynamic ports, and ran short forward and reverse transfers. Tests were interleaved rather than always running one backend first. The throughput table that started the investigation was approximately:

| Client write size | Direct connection | Tokio/Mio relay | RIO relay |
| --- | ---: | ---: | ---: |
| 32 KiB | 9–10 Gb/s | 12–13 Gb/s | 0.8–1.1 Gb/s |
| 128 KiB | 15–16 Gb/s | 0.5–0.6 Gb/s | 2.8–4.0 Gb/s |

Those are short loopback receiver-throughput measurements from repeated local cases, not network-interface capacity. Direct and proxied paths have different connection counts, so the direct number is a control for workload behavior rather than a denominator for a clean “proxy overhead” percentage. The striking relationship survives that caveat: the same async relay falls roughly 20–25-fold when the writer grows from 32 to 128 KiB, while the RIO relay moves in the other direction. Reverse transfers showed a similar reversal.

We ran 16, 64, and 96 KiB inputs as well. Tokio did not show a neat monotonic curve: a 64 KiB case was around 8–9 Gb/s and a 96 KiB case around 7 Gb/s before the 128 KiB cliff. RIO also varied non-monotonically. This immediately argued against a generic “larger application write is more expensive to copy” explanation. A `memcpy` cost would be accompanied by CPU use, and a bigger write would not normally reduce throughput by an order of magnitude while leaving the process idle.

### Why an AFD theory initially looked reasonable

On Windows, Tokio's socket readiness is implemented through Mio. In the version used by this experiment, Mio's Windows AFD path asks the kernel to poll socket readiness with `NtDeviceIoControlFile(IOCTL_AFD_POLL)` and receives completion through an IOCP-backed selector. RIO takes a different route: registered buffers and request queues produce completions on a RIO completion queue. A delay in AFD readiness emulation or its handoff to a Tokio task could therefore make `read().await` look slow even if data was already in the socket.

This was a concrete theory, not a straw man. The earlier short-connection stage probe had found occasional 10–20 ms tails before the backend read its request. If a similar wakeup delay occurred on every large transfer, it could explain low throughput with idle CPU. The blocking relay was especially valuable because it removed the Mio selector while retaining a receiver that copied and forwarded bytes. Its 128 KiB slowdown did not eliminate every possible AFD contribution, but it showed that AFD alone could not account for the entire pattern. The ETW thread and TCP events would decide whether the missing bytes were ready to read at all.

## The controls that kept us out of the wrong subsystem

We first tried a fixed 32 KiB Tokio copy buffer. It still measured around 0.5 Gb/s with 128 KiB writes. Our adaptive buffer policy had improved some direction-switch workloads, but it did not create this cliff by itself. `TCP_NODELAY` on the proxy sockets did not eliminate the 32/128 KiB reversal either.

Then we ran a small independent relay built on blocking `std::net::TcpStream` and two copying threads. Its 32 KiB case approached 9 Gb/s; its 128 KiB case fell to around 1–2 Gb/s. This version has no Tokio task or Mio readiness registration in the copy path. It did not match the async path's worst number, but it reproduced the direction of the regression. A problem shared by blocking sockets and Tokio sockets deserved attention below the runtime.

An additional Linux test used the same style of iperf workload on another development machine. Its Tokio relay remained around the same throughput at 32 and 128 KiB writes, and a Linux `splice` experiment approached that machine's direct bulk-transfer baseline. Those absolute numbers cannot be compared with Windows because the machines differ. The observation we used was that the dramatic input-size cliff was not an inherent property of our Rust loop or of iperf's 128 KiB writes on every OS.

At this stage, there were still two plausible Windows explanations. Tokio/Mio might deliver receive readiness late even though bytes were sitting in the relay's socket. Or the sender might stop transmitting because the relay's receive side was advertising too little usable space. We measured the copy loop, then changed one socket option.

## Reads account for almost the entire slow run

We wrapped the forward direction's `read().await` and `write_all().await` with optional counters. This is a wall-clock timer around async operations, so it includes waiting and task scheduling as well as any kernel work. The 128 KiB run reproduced the slow throughput even with the counters enabled:

| Client write size | Forwarded in 3 s | Accumulated `read().await` | Accumulated `write_all().await` |
| --- | ---: | ---: | ---: |
| 32 KiB | About 4.6 GB | About 1.6 s | About 1.4 s |
| 128 KiB | About 0.2 GB | About 2.9 s | About 0.1 s |

The 128 KiB connection was waiting to read. It was not filling the backend socket and then spending most of its life blocked on `write_all`. Its maximum single read wait was tens of milliseconds, the same scale as the visible throughput gap.

We inspected `SO_RCVBUF` on the accepted client socket. Untouched, `getsockopt` returned 65,536 bytes. Calling `setsockopt(SO_RCVBUF, 65536)` on that same socket still returned 65,536 from `getsockopt`, yet the throughput moved from roughly 0.5 Gb/s to several Gb/s. The code changed no sender setting, no backend socket, no runtime, and no copy-buffer algorithm. A 1 MiB explicit receive setting reached roughly 10–18 Gb/s in several short untraced runs. Explicit 2 MiB reached around 20 Gb/s once. The values were not uniformly ordered across every run, but “untouched” versus “explicitly set” was a repeatable change.

This is the point where a familiar socket debugging shortcut fails: `getsockopt(SO_RCVBUF)` is not the peer's current advertised TCP receive window. Windows [documents that explicitly setting the receive buffer changes receive-window autotuning behavior](https://learn.microsoft.com/en-us/windows/win32/winsock/sio-set-compatibility-mode). The nominal buffer value printed by the API could remain unchanged while the wire-visible behavior changed. We still needed to see that behavior rather than infer it from throughput.

One other intervention was informative but less surgical. Setting iperf's own window to 1 MiB changed both its send and receive socket buffers and moved several benchmark numbers dramatically, including direct throughput. It established that buffering and flow control could dominate the result, but did not identify which side of the relay needed attention. The accepted-socket experiment did.

## From WPR to a sender timeline

We captured CPU and Network ETW providers around one benchmark case at a time. An administrator started a limited local WPR service; ordinary benchmark processes asked it only to start or stop a fixed recording profile. We exported CPU sampled tables and short TCP/IP event slices with Windows Performance Toolkit. The first capture contained millions of events with no loss; we did not need to inspect NT internals or install a kernel debugger.

Parsing the network events correctly required one detail. The `TcpDataTransferSend` record contains `Cwnd`, `SndWnd`, `BytesSent`, `RcvWnd`, and other fields. `SndWnd` on the iperf sender's TCP A connection is the peer-advertised receive window relevant to that sender. The `RcvWnd` field on the same record describes the sender's own receive side; substituting it would have yielded a plausible but wrong story. We grouped events by connection endpoints and looked only at iperf→relay when discussing the relay's accepted socket.

Here is one 100 ms slice from each configuration. The bytes are the sum of TCP send-event `BytesSent` on TCP A, not the full benchmark's throughput calculation. Window values are medians of sender-visible `SndWnd` in those events.

| Async configuration | TCP A bytes in 100 ms | `SndWnd` median | Sender behavior in the slice |
| --- | ---: | ---: | --- |
| 128 KiB writes, untouched receive socket | ~7.5 MB | 261,888 B | Repeated ~13–15 ms gaps |
| 128 KiB writes, explicit 64 KiB | ~49 MB | 65,536 B | Mostly short 64 KiB send cycles; a few long gaps |
| 128 KiB writes, explicit 256 KiB | ~105 MB | 262,144 B | Roughly 256 KiB sends; no gap above 1 ms |
| 128 KiB writes, explicit 1 MiB | ~147 MB | 1,048,576 B | Roughly 1 MiB sends at sub-millisecond intervals |
| 32 KiB writes, untouched receive socket | ~99 MB | 261,888 B | Writes commonly merged into ~230 KiB sends |

This table makes the nominal-buffer trap concrete. The untouched socket and explicit 64 KiB socket both reported 65,536 via `getsockopt`, but their sender-visible windows were approximately 256 KiB and 64 KiB, respectively, and their send cadence differed. A larger advertised window alone is not sufficient to predict throughput: the untouched 256 KiB-ish case was much slower than the explicit 64 KiB case. The interaction between available window increments, queued data, and send policy matters.

The trace did not produce a single continuous 15 ms stack frame. It produced a sequence of TCP and scheduler events. Using time relative to the first send to avoid publishing raw trace identifiers, one representative slow interval looked like this:

```text
  +0 µs       TCP A sends ~128 KiB
  +0 µs       TcpSwsAvoidanceBegin; type-7 timer started for 5,000 µs
  +26 µs      sender thread switches out: Waiting / UserRequest
  +315 µs     proxy data thread waits for work
  +15,055 µs  next TCP A data send
  +15,207 µs  sender and proxy threads run again;
              sender's ready-to-running wait is ~1 µs
```

The exact scheduling relationship at the end of the gap is not the point; the sender was asleep for nearly the whole interval and was scheduled promptly when it became runnable. The proxy was also waiting for data. This event sequence explains why a CPU profile of the proxy alone was unhelpful: the slow run consumed less proxy CPU than the fast one because it did less work. In the ETL CPU tables, the proxy used roughly a quarter second in an eight-second view of the slow recording and several seconds in the much faster 1 MiB-buffer recording. The latter did more copying, so more CPU use was a consequence of progress.

The SWS event also records the sender's state at the decision point. In this representative cycle its `BytesToSend` field was about 65 KiB, `SendAvailable` was 128 KiB, and `MaxSndWnd` was about 256 KiB. These are more revealing than the copy buffer's 32 KiB allocation: the sender had application data and a recently observed window, but decided to defer a particular send. TCP's usable window is the right edge of the peer's advertised window minus bytes already in flight. A receiver can read 32 KiB and advance that edge without necessarily creating a send opportunity that the sender's SWS policy considers worthwhile. Once this repeats, the proxy may be ready to read while the sender waits to aggregate a better transmission opportunity. The event fields place that decision at TCP A, not at the backend-facing TCP B write.

The sender event name is not a full specification of Windows' internal algorithm. The [TCP specification's discussion of silly-window avoidance](https://www.rfc-editor.org/rfc/rfc9293.html) explains the general reason to hold a small increment of usable send window and eventually override the hold with a timer. In our trace, the SWS event and timer precede the long gap, the thread is waiting during it, and normal sending resumes after it. That is enough to locate this stall in TCP send progress. It does not tell us exactly which Windows-private threshold fired on every cycle or why a nominal 5 ms timer yielded an approximately 15 ms observed gap in this environment.

### Why the 256-byte theory failed

The untouched case's median sender-visible window was 261,888 bytes, exactly 256 bytes below an explicit 256 KiB setting. With 128 KiB application writes, that looked suspiciously like an alignment threshold. We tested it instead of publishing the arithmetic as an explanation. Explicit 255, 256, and 257 KiB settings all improved throughput by several times compared with the untouched default. In an interleaved repeat, the default remained around 0.5 Gb/s while explicit 256 KiB was around 2–3 Gb/s. One traced explicit-256 run was faster still. The tiny numeric boundary was not the robust variable; the explicit socket setting and the resulting TCP behavior were.

There is a second temptation in the ETL: counting SWS starts as if they were completed sleeps. The explicit-64 run started many type-7 timers in 100 ms but usually kept making progress without a 15 ms pause. The fast 32 KiB default-input case started some too. Direct iperf traffic had occasional long gaps. A timer can be canceled or overtaken by ACK/window progress. We therefore measured the time between actual send events and examined the corresponding thread state rather than declaring every `TcpSwsAvoidanceBegin` a stall.

Tracing itself perturbed the absolute throughput. A direct run that had previously reached about 15–16 Gb/s measured around 8 Gb/s while WPR recorded a high-volume network trace. The 100 ms slices are best used to compare event order, batch size, and the presence of repeated pauses; the untraced, interleaved runs establish the broad throughput pattern. This is why a large ETL did not replace the simpler benchmark matrix.

## RIO's 32 KiB problem has a different shape

The RIO path was an architectural experiment: Tokio remains the runtime, while registered sockets use `RIOReceive`, `RIOSend`, request queues, and a completion queue for the data path. Each direction has two registered buffer slots. One outstanding receive is posted at a time. A completed receive either starts a send or waits in the other slot until the active send finishes. The completion queue is drained in small batches; when it is empty, `RIONotify` signals an event, a registered Windows wait callback fires, and a Tokio `Notify` wakes the relay task.

There are several places this pipeline can be starved. The receive request might not be posted because both slots are owned by sends. The kernel might not complete it because no data arrives. The completion might be present but its event notification, callback, or task wake could be late. A TCP trace sees the flow-control consequence of all of them, but cannot assign time to each stage by itself.

The RIO sender-visible window in our 100 ms slice was often only tens of kilobytes, unlike the async default's roughly 256 KiB. With 128 KiB application writes, TCP A sent about 86 MB in that slice and had one long pause. With 32 KiB application writes, it sent about 5 MB and had several pauses around 10–15 ms. One of the latter intervals included `TcpZeroWindowProbingBegin`: the sender had no advertised room. The receiver then issued a burst of `TcpReceiveRequest` events and received roughly 32 KiB at a time; after that burst, the send path still experienced a long quiet interval. Some RIO gaps have SWS events, while this particular zero-window gap did not have an SWS timer at its start. Calling them all the same Windows TCP stall would erase the useful difference.

The user-space RIO counters give the next piece of the picture:

| Client write size | Average bytes per client receive completion | Times the task entered event wait in 3 s | Accumulated event wait |
| --- | ---: | ---: | ---: |
| 32 KiB | ~32.6 KiB | ~350 | ~2.8 s |
| 128 KiB | ~32.7 KiB | ~180 | ~2.4 s |

The completion size barely changes. The 128 KiB workload produces a denser stream of completions and fewer returns to the expensive-looking wait state. We do not yet know how much of an individual wait is time for the client to send another byte and how much is completion notification latency. The zero-window trace makes it plausible that receive posting and slot recycling are important, but plausibility is not a reason to rewrite the notification path blindly.

## The next experiment has four timestamps

The async path has a clear next engineering candidate: a bounded Windows receive-buffer policy, configured per connection and measured against more than bulk throughput. We would run short HTTP and HTTPS requests, half-closed and interrupted streams, upload/download reversals, several connection counts, CPU, and memory. Explicit `SO_RCVBUF` changes autotuning behavior, so adopting the 1 MiB diagnostic value everywhere would be a policy change, not a free optimization. The existing adaptive copy buffer would remain a separate variable.

The RIO path needs four operation timestamps before another structural edit: when a receive is posted, when its completion becomes visible, when the registered-wait callback runs, and when the Tokio task drains it. We would also record when the next receive is posted and align those timestamps with TCP window updates. If the delay is between a completion and callback, notification deserves work. If it is between send completion and a newly available receive slot, the two-slot pipeline deserves work. If the task is sleeping because TCP A has no incoming data, changing the runtime's wakeup strategy cannot fill the pipe.

The investigation began with a 25-fold throughput drop that looked like an indictment of an async runtime. The useful result is narrower and stronger. For the severe Tokio 128 KiB case, the receiver's socket policy changes the sender-visible window and send cadence; repeated SWS/timer-shaped gaps consume the transfer. For the RIO 32 KiB case, a narrow or zero receive window and sparse completion batches are visible, while the exact completion-to-wakeup contribution remains to be measured. Those are two concrete mechanisms to optimize and test. “Switch to a faster I/O API” was never an adequate description of either one.
