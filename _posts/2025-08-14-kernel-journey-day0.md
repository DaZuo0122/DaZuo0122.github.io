## Day 0: Welcome to the Kernel Journey

Welcome to this hands-on exploration of Linux kernel tracing and dynamic instrumentation! This blog series is designed to take you from a beginner to a practitioner in using powerful kernel tracing tools like ftrace, kprobes/uprobes, perf, and eBPF.

## Who Should Read This

This series is for those who want to understand what's happening inside the Linux kernel. You should have:

- Basic Linux command-line skills(Bash, GNU tools, C programming language, etc)
- Fundamental understanding of how operating systems work (processes, system calls, memory management)
- Comfort with installing software and compiling code
- Access to Linux machine(ideally Debain/Ubuntu), Labs were tested on Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-153-generic x86_64)

You don't need to be a kernel expert, but you should be comfortable with technical concepts and not afraid of diving into kernel documentation.

## What This Blog Series Is (and Isn't)

This is **NOT** a theoretical overview of kernel internals. You won't find lengthy explanations of kernel data structures or algorithms.

Instead, this is a **hands-on lab series** where you'll:
- Set up a working environment with all necessary tools
- Complete 8 days of practical exercises
- Learn by doing, with real tools on real problems
- Gain practical skills you can apply immediately to performance analysis, debugging, and system observability

## What You'll Learn

By the end of this 8-day journey, you'll be able to:
- Use ftrace to trace kernel functions and analyze call graphs
- Deploy dynamic probes (kprobes/uprobes) to instrument any kernel or user function
- Profile applications with perf and create performance flamegraphs
- Write and deploy eBPF programs for advanced tracing and monitoring
- Combine multiple tools for comprehensive system analysis
- Debug complex performance issues using kernel tracing

## Why Learn Kernel Tracing?

Modern systems are complex, and understanding what's happening below your applications is crucial for:
- Performance optimization
- Debugging mysterious system behavior
- Security analysis
- System observability and monitoring

Kernel tracing tools give you a superpower - the ability to look inside a running system without modifying or restarting it.

## Getting Started

Each day will include:
- A further reading list (usually documentation or blog posts)
- Hands-on labs with step-by-step instructions


Let's get started !