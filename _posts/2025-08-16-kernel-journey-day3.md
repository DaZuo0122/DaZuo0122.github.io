## Day 3 — perf and perf+ftrace integration
Further reading: perf documentation, Brendan Gregg’s perf guides.

### Lab 1 — perf basics and sampling
Record a simple CPU profile:
```bash
sudo perf record -F 99 -a -g -- sleep 5
sudo perf report --stdio | sed -n '1,120p'
```
You may see someting like this:
```text
# To display the perf.data header info, please use --header/--header-only options.
#
#
# Total Lost Samples: 0
#
# Samples: 1K of event 'cpu-clock:pppH'
# Event count (approx.): 19282828090
#
# Children      Self  Command          Shared Object            Symbol
# ........  ........  ...............  .......................  ....................................................
#
    99.37%     0.00%  swapper          [kernel.kallsyms]        [k] secondary_startup_64_no_verify
            |
            ---secondary_startup_64_no_verify
               |
               |--74.65%--start_secondary
               |          cpu_startup_entry
               |          do_idle
               |          cpuidle_idle_call
               |          default_idle_call
               |          arch_cpu_idle
               |          native_safe_halt
               |
                --24.72%--x86_64_start_kernel
                          x86_64_start_reservations
                          start_kernel
                          arch_call_rest_init
                          rest_init
                          cpu_startup_entry
                          do_idle
                          cpuidle_idle_call
                          default_idle_call
                          arch_cpu_idle
                          native_safe_halt

    99.37%     0.00%  swapper          [kernel.kallsyms]        [k] cpu_startup_entry
            |
            ---cpu_startup_entry
               do_idle
               cpuidle_idle_call
               default_idle_call
               arch_cpu_idle
               native_safe_halt

    99.37%     0.00%  swapper          [kernel.kallsyms]        [k] do_idle
            |
            ---do_idle
               cpuidle_idle_call
               default_idle_call
               arch_cpu_idle
               native_safe_halt

...
...
```
This is system-wide perf recording that recorded almost entirely idle time: ~99.37% of samples map to the kernel idle path (swapper), with native_safe_halt as the dominant symbol.
Command breakdown:  
```bash
sudo perf record -F 99 -a -g -- sleep 5
```
- -F 99: sample at ~99 Hz (about 99 samples/sec).
- -a: system-wide (all CPUs).
- -g: capture callchains (kernel/user stack backtraces).


Some key info from the result:  
- 99.37% (Children): 99.37% of all samples fall somewhere under this entry’s callgraph.
- 0.00% (Self): almost none of the time was spent directly in that one instruction — it’s inclusive of deeper idle frames.
- The indented tree shows callchains that led to idle: e.g., secondary_startup_64_no_verify → start_secondary → cpu_startup_entry → do_idle → cpuidle_idle_call → default_idle_call → arch_cpu_idle → native_safe_halt.


### Lab 2 — Profile a workload and generate a flamegraph
1. Install FlameGraph scripts(in case you don't have them):
```bash
 git clone https://github.com/brendangregg/FlameGraph.git
```


2. Record with call-graph sampling for a target workload  
Remember the tiny C program we wrote in Day 2(Lab 4)? Let's make it a little bit heavier.
```c
// tiny_workload.c
#include <stdio.h>
void target() { puts("hit"); }
int main(){ for(int i=0;i<10000000;i++) target(); }
```
Run and record in one command:
```bash
sudo perf record -F 99 -g -- tiny_workload
```
We can take a look at the result before generating flamegraph:
```bash
sudo perf report --stdio | sed -n '1,120p' | head -n 50
```
it will be something like this: 
```text
# To display the perf.data header info, please use --header/--header-only options.
#
#
# Total Lost Samples: 0
#
# Samples: 3K of event 'cpu-clock:pppH'
# Event count (approx.): 34242423900
#
# Children      Self  Command        Shared Object      Symbol
# ........  ........  .............  .................  ............................................
#
    94.40%     0.00%  tiny_workload  [unknown]          [k] 0000000000000000
            |
            ---0
               |
               |--93.86%--write
               |          |
               |          |--79.26%--entry_SYSCALL_64_after_hwframe
               |          |          do_syscall_64
               |          |          |
               |          |          |--63.63%--x64_sys_call
               |          |          |          |
               |          |          |           --63.24%--__x64_sys_write
               |          |          |                     |
               |          |          |                      --63.01%--ksys_write
               |          |          |                                |
               |          |          |                                |--61.18%--vfs_write
               |          |          |                                |          |
               |          |          |                                |          |--59.00%--new_sync_write
               |          |          |                                |          |          |
               |          |          |                                |          |           --58.17%--tty_write
...
...
```
3. Convert perf.data to folded stack format and generate flamegraph:
```bash
sudo perf script -i perf.data > out.perf
./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
./FlameGraph/flamegraph.pl out.folded > flamegraph.svg
```
You can open flamegraph.svg in a browser or transfer it to your host.


### Lab 3 — perf + ftrace integration (measure function_graph with perf)
1. Use perf to trace ftrace events
```bash
echo function_graph | sudo tee /sys/kernel/tracing/current_tracer
echo 1 | sudo tee /sys/kernel/tracing/tracing_on
sudo perf record -a -g -- sleep 2
echo 0 | sudo tee /sys/kernel/tracing/tracing_on
sudo perf report
sudo perf script | sed -n '1,200p' | head -n 50
```


2. Use trace-cmd to record function_graph then analyze with perf script:
```bash
sudo trace-cmd record -p function_graph -o ./fg.dat -- sleep 2
sudo trace-cmd report -i ./fg.dat | sed -n '1,200p' | head -n 50
```


3. \[Optional] Compare overhead and data between perf sampling (perf record -g) and function_graph traces (trace-cmd)
 >[!TIP]  
 > You may need to reboot for a more accurate result.  
 > We will run C program tiny_workload as workload  
 
Baseline setup:
```bash
/usr/bin/time -v ./tiny_workload
sudo perf stat -a -e cycles,instructions,context-switches,cpu-migrations,page-faults -o baseline.stat ./tiny_workload
```

perf sampling (-g):
```bash
/usr/bin/time -v sudo perf record -a -g -o perf.data -- ./tiny_workload
```

function_graph via trace-cmd:
```bash
/usr/bin/time -v sudo trace-cmd record -p function_graph -o fg.dat -- ./tiny_workload
```


After getting results, you can use visualization tools(Flame graphs/kernelshark(requires GUI)) for analysis.


### Lab 4 — perf probes and event counting
1. Create a perf probe and record:
```bash
sudo perf probe -a do_sys_openat2
sudo perf record -e probe:do_sys_openat2 -a -- sleep 2
sudo perf script | sed -n '1,120p'
```
You may see someting like this:
```text
			perf  1358 [003]   627.875141: probe:do_sys_openat2: (ffffffffa239b4a0)
           sleep  1361 [003]   627.881189: probe:do_sys_openat2: (ffffffffa239b4a0)
           sleep  1361 [003]   627.881218: probe:do_sys_openat2: (ffffffffa239b4a0)
           sleep  1361 [003]   627.881503: probe:do_sys_openat2: (ffffffffa239b4a0)
         systemd     1 [002]   628.178048: probe:do_sys_openat2: (ffffffffa239b4a0)
 systemd-timesyn   788 [003]   628.405157: probe:do_sys_openat2: (ffffffffa239b4a0)
 systemd-timesyn   788 [003]   628.405264: probe:do_sys_openat2: (ffffffffa239b4a0)
```


2. Use perf stat to collect system wide CPU statistic:
```bash
sudo perf stat -e cycles,instructions,cache-references,cache-misses,bus-cycles -a sleep 5
```
 >[!TIP]  
 > If you ran this command inside VM, you may see someting like the following.  
 > That's because virtualization/hypervisor blocks or doesn't virtualize PMU
 > You can enable PMU passthrough / virtualization in the VM config.  

```text
 Performance counter stats for 'system wide':

   <not supported>      cycles
   <not supported>      instructions
   <not supported>      cache-references
   <not supported>      cache-misses
   <not supported>      bus-cycles

       5.006459663 seconds time elapsed
```


### Lab 5 — tracing user-space stacks and kernel stacks
Record both user and kernel stacks:
```bash
sudo perf record -F 99 -g --call-graph dwarf -- ./tiny_workload
sudo perf script | sed -n '1,200p'
```
You may see someting like this:
```text
tiny_workload  1476  1477.337737:   10101010 cpu-clock:pppH:
            7f5a59249887 write+0x17 (/usr/lib/x86_64-linux-gnu/libc.so.6)
            7f5a591bfeec _IO_file_write+0x2c (/usr/lib/x86_64-linux-gnu/libc.so.6)
            7f5a591c19e0 _IO_do_write+0xb0 (/usr/lib/x86_64-linux-gnu/libc.so.6)
            7f5a591c1ec2 _IO_file_overflow+0x102 (/usr/lib/x86_64-linux-gnu/libc.so.6)
            7f5a591b5fa9 puts+0x159 (inlined)
            561973fb115f target+0x16 (/home/YourUsername/kernel-trace/day3/tiny_workload)
            561973fb1181 main+0x1e (/home/YourUsername/kernel-trace/day3/tiny_workload)
            7f5a5915ed8f [unknown] (/usr/lib/x86_64-linux-gnu/libc.so.6)
            7f5a5915ee3f __libc_start_main+0x7f (/usr/lib/x86_64-linux-gnu/libc.so.6)
            561973fb1084 _start+0x24 (/home/YourUsername/kernel-trace/day3/tiny_workload)
```
This provides a clear look on callchain (from top to bottom):  
- 7f5a59249887 "write+0x17" (/usr/lib/.../libc.so.6) => Immediate frame: libc's write syscall wrapper (offset +0x17).
- /\*skipped\*/
- 7f5a591b5fa9 "puts+0x159" (inlined) => puts was inlined: perf shows it as an inlined frame (no separate call site function frame).
- 561973fb115f "target+0x16" (/home/.../tiny_workload) => the target() function at offset 0x16.
- 561973fb1181 "main+0x1e" => (the main function)
- 7f5a5915ed8f "\[unknown]" (libc) — occasional unresolved inline or small trampoline
- 7f5a5915ee3f "__libc_start_main+0x7f" (libc startup)
- 561973fb1084 "_start+0x24" (program entry)

"puts("hit")" in target() function triggers glibc I/O path: puts => buffered I/O internals => write. At runtime the top-most active code is libc’s low-level write when the sample fired, so the sample attributes to write and the libc stack above it.  
The presence of "__libc_start_main/_start" shows a normal user-space stack up from program entry.  


### Cleanup and safety
1. Use bash script in Day2 to list and delete perf created probes  

2. Delete perf.data and temporary files:
```bash
rm perf.data out.perf out.folded 2>/dev/null || true
```