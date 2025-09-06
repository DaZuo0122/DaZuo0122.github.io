## Day 2 — kprobes & uprobes
 >[!WARNING]  
 > Probing kernel functions can crash the system if you probe unsuitable symbols or use wrong fetch args.
 > Keep an extra SSH session open in case you need to reboot.
 
Further reading: LWN “An introduction to kprobes”, kernel kprobe docs.  
 
### Lab 1 — Walking around
1. Add a simple kprobe via kprobe_events:
```bash
# add probe: p/group/event kernel_symbol
sudo sh -c 'echo "p:kp_test/do_sys_open do_sys_open" > /sys/kernel/debug/tracing/kprobe_events' || true
```


2. Check current kprobe:
```bash
sudo cat /sys/kernel/debug/tracing/kprobe_events
```
You may see someting like this: 
```text
p:kp_test/do_sys_open do_sys_open
```
You can clear all existing kprobes with:
```bash
sudo sh -c 'echo > /sys/kernel/debug/tracing/kprobe_events'
```

### Lab 2 — perf probe (a user-friendly kprobe/uprobes wrapper)
1. Create a kernel probe with perf:  

```bash
# create probe that records entry to do_sys_open
sudo perf probe -a do_sys_openat2

sudo perf probe -l | grep do_sys_openat2
```
You may see someting like this: 
```text
probe:do_sys_openat2 (on do_sys_openat2)
```
perf is way more easier than directly interact with `kprobe_events`.


2. Record and show events:  

```bash
sudo perf record -e probe:do_sys_openat2 -a -- sleep 1
```
You may see someting like this:
```text
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.184 MB perf.data (4 samples) ]
```
As it only captured 4 samples, we don't need to use `head`/`tail` to control the output.  
To show the captured samples, we can use:
```bash
sudo perf script

# To open in TUI mode, you can use
sudo perf report
```
and the output is shown below:
```text
 perf 57348 [003]  3923.922547: probe:do_sys_openat2: (ffffffffb099b4a0)
sleep 57351 [003]  3923.926350: probe:do_sys_openat2: (ffffffffb099b4a0)
sleep 57351 [003]  3923.926402: probe:do_sys_openat2: (ffffffffb099b4a0)
sleep 57351 [003]  3923.927194: probe:do_sys_openat2: (ffffffffb099b4a0)
```
Let's take first line to do a short breakdown:
```text
 perf 57348 [003]  3923.922547: probe:do_sys_openat2: (ffffffffb099b4a0)
```

- "perf" — the command/comm that recorded the sample (this is the task name at sample time).
- "57348" — PID (process id / thread id) that was running when the probe fired.
- "[003]" — CPU number where the sample was taken (CPU 3).
- "3923.922547" — timestamp in seconds (relative to perf recording start) with microsecond precision.
- "probe:do_sys_openat2:" — the event name: a dynamic kprobe you created named do_sys_openat2.
- "(ffffffffb099b4a0)" — the kernel instruction pointer / address where the probe hit (hex vaddr in kernel space).

The sample records the current task at the instant the probe executed. If do_sys_openat2 ran while the scheduler had sleep (or perf) scheduled on that CPU, the task column shows that task. That doesn’t mean sleep called do_sys_openat2 — it only reflects who was running when the kernel executed that code path (kernel work runs on behalf of the current task).


3. Remove perf probe:
```bash
sudo perf probe -d do_sys_openat2
```

### Lab 3 — uprobes on user binaries 
1. Find function name for ELF symbol:
```bash
# list symbols
nm -D /lib/x86_64-linux-gnu/libc.so.6 | grep -E ' fopen|open|read|write' | head -n 20
```


2. Create uprobe via perf:
```bash
sudo perf probe -x /lib/x86_64-linux-gnu/libc.so.6 fopen
sudo perf record -e probe_libc:fopen -a -- ls -al /etc /tmp
sudo perf script
sudo perf probe -d fopen
```
You may see someting like this:
```text
ls 58601 [001]  7265.787657: probe_libc:fopen: (7f278b6af630)
ls 58601 [001]  7265.788087: probe_libc:fopen: (7f278b6af630)
ls 58601 [001]  7265.788464: probe_libc:fopen: (7f278b6af630)
ls 58601 [001]  7265.788550: probe_libc:fopen: (7f278b6af630)
ls 58601 [001]  7265.788627: probe_libc:fopen: (7f278b6af630)
ls 58601 [001]  7265.789349: probe_libc:fopen: (7f278b6af630)
ls 58601 [001]  7265.790035: probe_libc:fopen: (7f278b6af630)
ls 58601 [001]  7265.793291: probe_libc:fopen: (7f278b6af630)
ls 58601 [001]  7265.793902: probe_libc:fopen: (7f278b6af630)
```
As explained in Lab 2, we can infer that while ls (PID 58601) was running on CPU 1, it called libc's fopen at user address 0x7f278b6af630 at those timestamps — there's multiple fopen hits captured as ls(with flags -a -l) opened files.


### Lab 4 — using bpftrace for kprobe/uprobes
1. Simple bpftrace kprobe one-liner:
```bash
sudo bpftrace -e 'kprobe:do_sys_openat2 { printf("%s %d\n", comm, pid); }' -c '/usr/bin/ls /'
```
You may see someting like this:
```text
Attaching 1 probe...
ls 58679
ls 58679
bin   cdrom  etc   lib    lib64   lost+found  mnt  proc  run   snap  swap.img  tmp  var
boot  dev    home  lib32  libx32  media       opt  root  sbin  srv   sys       usr
ls 58679
ls 58679
ls 58679
ls 58679
ls 58679
```
In this example we only showed command and pid (`printf("%s %d\n", comm, pid);`), skipped other columns. 


2. Uprobe example:  
 >[!WARNING]  
 > Some may want to trigger uprobe with following command, and find out system freezing:  
 > `sudo bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc.so.6:getpid { @[comm] = count(); }' -- /usr/bin/ls /`  
 > That's because "fopen" and other unsafe methods may casue race/recursion or deadlock during probe installation or execution that stalls the process


To mess with uprobe in a safe way, we can write a trigger program(in C, or other language you like) ourselves.
```c
// trigger_uprobe.c
#include <stdio.h>
void target() { puts("hit"); }
int main(){ for(int i=0;i<10;i++) target(); }
```
To compile it, use:
```bash
gcc -g -O0 ./trigger_uprobe.c -o ./trigger_uprobe
```
Then we can run it and attach bpftrace:
```bash
sudo bpftrace -e 'uprobe:./trigger_uprobe:target { @[comm] = count(); }' -c ./trigger_uprobe
```

You may see someting like this:
```text
Attaching 1 probe...
hit
hit
hit
hit
hit
hit
hit
hit
hit
hit


@[trigger_uprobe]: 10
```
The 10 "hit" lines are the program’s own output from calling target() 10 times.
bpftrace attached an uprobe to target() and counted hits in a map keyed by command name.
The final line `@[trigger_uprobe]: 10` is bpftrace printing that map: the process named "trigger_uprobe" invoked target() 10 times (count = 10).


### Cleanup after the labs
The following bash script can be used to list and delete perf created probes:
```bash
#!/usr/bin/env bash
set -euo pipefail

# List perf-created probes
echo "perf probe --list output:"
sudo perf probe --list || true
echo

# Remove probes registered via 'perf probe'
echo "Removing perf probes (perf probe -d)..."
# Get list of perf probes, one per line (skip header lines)
mapfile -t PROBES < <(sudo perf probe --list 2>/dev/null | sed -n '1,200p' | sed 's/^[[:space:]]*//;/^$/d')
if [ "${#PROBES[@]}" -eq 0 ]; then
  echo "  No perf probes found."
else
  for p in "${PROBES[@]}"; do
    # perf probe --list prints lines like "probe:foo" or "p:bar"
    name="$p"
    echo "  Deleting $name"
    sudo perf probe -d "$name" || echo "    failed to delete $name"
  done
fi
echo "done"
```