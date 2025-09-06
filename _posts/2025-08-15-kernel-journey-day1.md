## Environment setup
I'm using `Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-153-generic x86_64)`. Before diving into kernel tracing, we firstly need to install tools/dependencies.
```bash
sudo apt update
sudo apt install -y linux-tools-$(uname -r) linux-cloud-tools-$(uname -r) bpftrace bpfcc-tools trace-cmd kernelshark build-essential linux-headers-$(uname -r) libelf-dev clang llvm libbpf-dev
```
This includes basic packages(eg. git), kernel headers and common dev libs, tracing & eBPF userland tools, libbpf development files and bpftool.
Then we use `ls -d /sys/kernel/debug/tracing` to check `debugfs` is mounted. If meet fatal errors, we can use command `sudo mount -t debugfs none /sys/kernel/debug || true` to mount it.
We may also want to check BTF/vmlinux availability which provides us advanced eBPF features (fentry/fexit, better stack symbols).
```bash
sudo bpftool btf dump file /sys/kernel/btf/vmlinux > /dev/null 2>&1 && echo "BTF: /sys/kernel/btf/vmlinux present" || echo "No /sys/kernel/btf/vmlinux"
```
We can install it with commands(as example)
```bash
# Install prerequisites
sudo apt install -y bc flex bison libssl-dev dwarves linux-source

# Prepare kernel build directory and config that enables BTF
zcat /proc/config.gz > .config 2>/dev/null || cp /boot/config-"$(uname -r)" .config

scripts/config --enable CONFIG_DEBUG_INFO
scripts/config --enable CONFIG_DEBUG_INFO_BTF
scripts/config --enable CONFIG_DEBUG_INFO_REDUCED  # optional: reduces size

yes "" | make olddefconfig

# Build vmlinux with debug info (BTF)
# In kernel source root
make -j$(nproc) vmlinux

# Generate BTF from vmlinux using pahole
# from kernel source root
sudo pahole -J vmlinux

sudo pahole --btf=vmlinux.btf vmlinux

file vmlinux.btf
readelf -s vmlinux.btf | head

# Install the generated BTF
KVER=$(uname -r)
sudo mkdir -p /boot/boot
sudo cp vmlinux.btf /boot/vmlinux-"$KVER".btf
```
Finally, we crate a workspace and git repo for labs  
```bash
mkdir -p ~/kernel-tracing && cd ~/kernel-tracing
git init
```

## Day 1 — ftrace fundamentals
Prerequests: `/sys/kernel/debug/tracing` exists and tools installed.  
Further reading: kernel docs (Documentation/trace/ftrace.txt) and short blog posts (Brendan Gregg, Steven Rostedt).

### Lab 1 — Walking around
1. Show available tracers and events:  
```bash
cat /sys/kernel/debug/tracing/available_tracers
ls /sys/kernel/debug/tracing/events | head -n 20
```
2. View current trace (should be empty):  
```bash
sudo cat /sys/kernel/debug/tracing/trace | sed -n '1,40p'
```

### Lab 2 — Simple function tracer
1. Enable function tracer, trace a short workload:  
```bash
sudo sh -c 'echo function > /sys/kernel/debug/tracing/current_tracer'
sudo sh -c 'echo 1 > /sys/kernel/debug/tracing/tracing_on'
ls /tmp >/dev/null
sleep 0.5
sudo sh -c 'echo 0 > /sys/kernel/debug/tracing/tracing_on'
sudo sed -n '1,120p' /sys/kernel/debug/tracing/trace
```
You may see someting like this: 
```text
# tracer: function
#
# entries-in-buffer/entries-written: 205085/4268565   #P:4
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| / _-=> migrate-disable
#                              |||| /     delay
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
#              | |         |   |||||     |         |
          <idle>-0       [003] dN...  4753.900675: _raw_spin_lock <-raw_spin_rq_lock_nested
          <idle>-0       [003] dN...  4753.900675: update_rq_clock <-__schedule
          <idle>-0       [003] dN...  4753.900675: pick_next_task <-__schedule
          <idle>-0       [003] dN...  4753.900676: pick_next_task_fair <-pick_next_task
          <idle>-0       [003] dN...  4753.900676: put_prev_task_idle <-pick_next_task_fair
          <idle>-0       [003] dN...  4753.900676: pick_next_entity <-pick_next_task_fair
          <idle>-0       [003] dN...  4753.900676: set_next_entity <-pick_next_task_fair
          <idle>-0       [003] dN...  4753.900676: clear_buddies <-set_next_entity
          <idle>-0       [003] dN...  4753.900677: __update_load_avg_se <-update_load_avg
          <idle>-0       [003] dN...  4753.900677: __update_load_avg_cfs_rq <-update_load_avg
          <idle>-0       [003] d....  4753.900677: psi_task_switch <-__schedule
          <idle>-0       [003] d....  4753.900677: psi_flags_change <-psi_task_switch
          <idle>-0       [003] d....  4753.900678: iterate_groups <-psi_task_switch
          <idle>-0       [003] d....  4753.900678: psi_group_change <-psi_task_switch
          <idle>-0       [003] d....  4753.900678: iterate_groups <-psi_task_switch
          <idle>-0       [003] d....  4753.900679: enter_lazy_tlb <-__schedule
       rcu_sched-14      [003] d....  4753.900680: finish_task_switch.isra.0 <-__schedule
       rcu_sched-14      [003] d....  4753.900681: raw_spin_rq_unlock <-finish_task_switch.isra.0
       rcu_sched-14      [003] .....  4753.900681: __timer_delete_sync <-schedule_timeout
       rcu_sched-14      [003] .....  4753.900681: __try_to_del_timer_sync <-__timer_delete_sync
       rcu_sched-14      [003] .....  4753.900681: lock_timer_base <-__try_to_del_timer_sync
       rcu_sched-14      [003] .....  4753.900681: _raw_spin_lock_irqsave <-lock_timer_base
...
...
```
This is the output captured from Linux’s function tracer (ftrace). It records kernel function entry events and prints a timeline of which kernel functions ran, from which task/CPU, and when.	   
We take a simple line to explain how to read it.  
`-0       [003] dN...  4753.900675: _raw_spin_lock <-raw_spin_rq_lock_nested`
- Task: (PID 0), CPU 3, flags dN..., timestamp 4753.900675 s.
- Event: kernel entered `_raw_spin_lock`; its caller was `raw_spin_rq_lock_nested`("A <- B" means ftrace recorded function A being called from function B).
The sequence:  
```text
pick_next_task <-__schedule
pick_next_task_fair <-pick_next_task
put_prev_task_idle <-pick_next_task_fair
```
shows the scheduler (__schedule) calling `pick_next_task`, which calls `pick_next_task_fair`, which in turn called `put_prev_task_idle` — i.e., a call chain inside the scheduler while switching tasks.  
The many `rcu_sched` and `rcu_gp_*` entries show the RCU (Read-Copy-Update) kernel background thread doing its periodic work (cleanup, reporting grace periods, accelerating callbacks). Those are normal kernel housekeeping functions.


You may also want to ask "Why is there many internal calls and many RCU/scheduler lines?"  
Here's a short answer:
- The function tracer logs virtually all traced kernel function entries; the kernel runs many housekeeping functions continuously (scheduler, RCU, timers, spinlock operations), so trace output is very noisy even during a short interval.
- The tracer has low overhead but can still produce a huge amount of output (your header shows entries-written vs entries-in-buffer).


2. Filter tracing to a PID (trace only a single process):
```bash
# start a background workload
sleep 2m & pid=$!
sudo sh -c "echo $pid > /sys/kernel/debug/tracing/set_ftrace_pid"
sudo sh -c 'echo 1 > /sys/kernel/debug/tracing/tracing_on'
# run some commands in another shell that the PID executes (or kill after a bit)
sleep 1
sudo sh -c 'echo 0 > /sys/kernel/debug/tracing/tracing_on'
sudo cat /sys/kernel/debug/tracing/trace | head -n 60
sudo sh -c 'echo > /sys/kernel/debug/tracing/set_ftrace_pid'   # clear PID filter
```
You may see someting like this: 
```text
# tracer: function
#
# entries-in-buffer/entries-written: 7676/7676   #P:4
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| / _-=> migrate-disable
#                              |||| /     delay
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
#              | |         |   |||||     |         |
           sleep-4991    [003] d.... 14097.044677: finish_task_switch.isra.0 <-__schedule
           sleep-4991    [003] d.... 14097.044680: raw_spin_rq_unlock <-finish_task_switch.isra.0
           sleep-4991    [003] ..... 14097.044680: __cond_resched <-do_nanosleep
           sleep-4991    [003] ..... 14097.044680: rcu_all_qs <-__cond_resched
           sleep-4991    [003] ..... 14097.044681: hrtimer_active <-do_nanosleep
           sleep-4991    [003] ..... 14097.044681: hrtimer_try_to_cancel.part.0 <-do_nanosleep
           sleep-4991    [003] ..... 14097.044681: _raw_spin_lock_irqsave <-hrtimer_try_to_cancel.part.0
           sleep-4991    [003] d.... 14097.044681: __remove_hrtimer <-hrtimer_try_to_cancel.part.0
           sleep-4991    [003] d.... 14097.044682: _raw_spin_unlock_irqrestore <-hrtimer_try_to_cancel.part.0
           sleep-4991    [003] ..... 14097.044682: ktime_get <-do_nanosleep
           sleep-4991    [003] ..... 14097.044683: ns_to_timespec64 <-do_nanosleep
           sleep-4991    [003] ..... 14097.044683: nanosleep_copyout <-do_nanosleep
           sleep-4991    [003] ..... 14097.044683: put_timespec64 <-nanosleep_copyout
...
...
```
Here we crated a filter that only shows the `sleep 2m` task(in this case, pid == 4991).


### Lab 3 — Function graph tracer (call stacks, durations)
Enable function tracer, trace a short workload:  
```bash
sudo sh -c 'echo function > /sys/kernel/debug/tracing/current_tracer'
sudo sh -c 'echo 1 > /sys/kernel/debug/tracing/tracing_on'
ls /tmp >/dev/null
sleep 0.5
sudo sh -c 'echo 0 > /sys/kernel/debug/tracing/tracing_on'
sudo sed -n '1,120p' /sys/kernel/debug/tracing/trace
```
You may see someting like this: 
```text
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
   3)   0.251 us    |        } /* nsecs_to_jiffies */
   3)   0.239 us    |        nsecs_to_jiffies();
   3)   0.242 us    |        nsecs_to_jiffies();
   3)   0.241 us    |        nsecs_to_jiffies();
   3)   0.243 us    |        nsecs_to_jiffies();
   3)   0.251 us    |        nsecs_to_jiffies();
   3)   0.239 us    |        nsecs_to_jiffies();
   3)   0.238 us    |        nsecs_to_jiffies();
   3)   0.241 us    |        nsecs_to_jiffies();
   3)   0.241 us    |        nsecs_to_jiffies();
...
...
   3) + 96.976 us   |      } /* collect_percpu_times */
   3)   0.525 us    |      update_averages();
   3)   0.224 us    |      nsecs_to_jiffies();
   3)               |      queue_delayed_work_on() {
   3)               |        __queue_delayed_work() {
   3)   1.381 us    |          add_timer();
   3)   1.906 us    |        }
   3)   2.403 us    |      }
   3)   0.253 us    |      mutex_unlock();
   3) ! 103.496 us  |    } /* psi_avgs_work */
   3)               |    __cond_resched() {
   3)   0.231 us    |      rcu_all_qs();
   3)   0.672 us    |    }
   3)   0.255 us    |    _raw_spin_lock_irq();
   3)   0.320 us    |    pwq_dec_nr_in_flight();
   3) ! 106.471 us  |  } /* process_one_work */
...
...
 ------------------------------------------
   3)  kworker-4950  =>    <idle>-0
 ------------------------------------------

   3)               |      finish_task_switch.isra.0() {
   3)   0.221 us    |        raw_spin_rq_unlock();
   3)   0.661 us    |      }
   3) ! 144.631 us  |    } /* schedule_idle */
   3) @ 159340.0 us |  } /* do_idle */
   3)               |  do_idle() {
   3)   0.286 us    |    nohz_run_idle_balance();
   3)               |    tick_nohz_idle_enter() {
   3)   0.364 us    |      ktime_get();
   3)   0.920 us    |    }
   3)               |    arch_cpu_idle_enter() {
   3)   0.247 us    |      tsc_verify_tsc_adjust();
   3)   0.257 us    |      local_touch_nmi();
   3)   1.192 us    |    }
   3)   0.238 us    |    tick_check_broadcast_expired();
   3)               |    cpuidle_idle_call() {
   3)   0.352 us    |      cpuidle_get_cpu_driver();
   3)   0.309 us    |      cpuidle_not_available();
   3)               |      tick_nohz_idle_stop_tick() {
   3)   0.245 us    |        can_stop_idle_tick();
   3)               |        tick_nohz_next_event() {
   3)   0.229 us    |          rcu_needs_cpu();
   3)   0.795 us    |          get_next_timer_interrupt();
   3)   0.236 us    |          timekeeping_max_deferment();
   3)   2.386 us    |        }
   3)               |        tick_nohz_stop_tick() {
   3)   0.225 us    |          calc_load_nohz_start();
   3)   0.363 us    |          quiet_vmstat();
   3)   5.834 us    |          hrtimer_start_range_ns();
   3)   7.381 us    |        }
   3)   0.287 us    |        nohz_balance_enter_idle();
   3) + 11.491 us   |      }
```
It shows kernel function call stacks with per-call durations (how long each function ran) and timestamps for CPU 3 during the tracing window.  
Some key points about how to read it:  
- Repeated `nsecs_to_jiffies()` lines: small utility conversions called many times; each takes ~0.24 µs. Lots of repetition is normal for frequently-called helpers.
- A block like: `queue_delayed_work_on() { __queue_delayed_work() { add_timer(); } }` shows `queue_delayed_work_on called`, which calls `__queue_delayed_work` which called `add_timer`; durations beside each show how long each call took.
- Lines marked with "!" (e.g., `! 103.496 us | } /* psi_avgs_work */`) show the total time spent in that function (including callees) — useful to spot slow functions.
- The `kworker-4950 => -0` separator indicates a context switch: the CPU switched from the worker thread to the idle task. After that, the trace shows the idle path (do_idle) and its nested calls (tick/timers, cpuidle code). The "@ 159340.0 us" is a larger timestamp marker for an event gap.
We found out that much of the trace is kernel housekeeping: RCU, scheduler, kworker work items, timers, and idle-loop activities. Those are expected on any running system.  
And the `function_graph` tracer is excellent for:  
- Seeing which kernel functions dominate time (look for large "!" durations).
- Understanding call paths and where time is spent (self vs callees).

### Lab 4 — Playing with trace-cmd
Record with trace-cmd:
```bash
sudo trace-cmd record -p function_graph -o ~/kernel-trace/kernel-trace.dat -- sleep 1
sudo trace-cmd report -i ~/kernel-trace/kernel-trace.dat | sed -n '1,200p'

# You may also want to visualize on desktop-capable environment
kernelshark ~/kernel-trace/kernel-trace.dat
```
We will skip the visualization here, as we are in server environment.
You may see someting like this:
```text
cpus=4
           sleep-5204  [003] 15822.668936: funcgraph_entry:        2.263 us   |  mutex_unlock();
           sleep-5204  [003] 15822.668939: funcgraph_entry:                   |  __f_unlock_pos() {
           sleep-5204  [003] 15822.668939: funcgraph_entry:        0.258 us   |    mutex_unlock();
           sleep-5204  [003] 15822.668939: funcgraph_exit:         0.845 us   |  }
           sleep-5204  [003] 15822.668940: funcgraph_entry:                   |  exit_to_user_mode_prepare() {
           sleep-5204  [003] 15822.668940: funcgraph_entry:        0.300 us   |    fpregs_assert_state_consistent();
           sleep-5204  [003] 15822.668941: funcgraph_exit:         0.906 us   |  }
           sleep-5204  [003] 15822.668945: funcgraph_entry:        0.520 us   |  down_read_trylock();
           sleep-5204  [003] 15822.668946: funcgraph_entry:                   |  __cond_resched() {
           sleep-5204  [003] 15822.668946: funcgraph_entry:        0.251 us   |    rcu_all_qs();
           sleep-5204  [003] 15822.668946: funcgraph_exit:         0.737 us   |  }
           sleep-5204  [003] 15822.668947: funcgraph_entry:                   |  find_vma() {
           sleep-5204  [003] 15822.668947: funcgraph_entry:        0.323 us   |    vmacache_find();
           sleep-5204  [003] 15822.668947: funcgraph_exit:         0.828 us   |  }
           sleep-5204  [003] 15822.668948: funcgraph_entry:                   |  handle_mm_fault() {
           sleep-5204  [003] 15822.668948: funcgraph_entry:        0.333 us   |    mem_cgroup_from_task();
           sleep-5204  [003] 15822.668949: funcgraph_entry:                   |    __count_memcg_events() {
           sleep-5204  [003] 15822.668949: funcgraph_entry:        0.515 us   |      cgroup_rstat_updated();
           sleep-5204  [003] 15822.668950: funcgraph_exit:         1.107 us   |    }
           sleep-5204  [003] 15822.668950: funcgraph_entry:        0.244 us   |    rcu_read_unlock_strict();
           sleep-5204  [003] 15822.668951: funcgraph_entry:                   |    __handle_mm_fault() {
           sleep-5204  [003] 15822.668951: funcgraph_entry:                   |      handle_pte_fault() {
           sleep-5204  [003] 15822.668952: funcgraph_entry:                   |        do_fault() {
           sleep-5204  [003] 15822.668952: funcgraph_entry:      + 15.346 us  |          do_read_fault();
           sleep-5204  [003] 15822.668968: funcgraph_exit:       + 15.938 us  |        }
           sleep-5204  [003] 15822.668968: funcgraph_exit:       + 16.668 us  |      }
           sleep-5204  [003] 15822.668968: funcgraph_exit:       + 17.621 us  |    }
           sleep-5204  [003] 15822.668969: funcgraph_exit:       + 20.900 us  |  }
           sleep-5204  [003] 15822.668969: funcgraph_entry:        0.272 us   |  up_read();
           sleep-5204  [003] 15822.668970: funcgraph_entry:                   |  exit_to_user_mode_prepare() {
           sleep-5204  [003] 15822.668970: funcgraph_entry:        0.277 us   |    fpregs_assert_state_consistent();
           sleep-5204  [003] 15822.668970: funcgraph_exit:         0.843 us   |  }
           sleep-5204  [003] 15822.668974: funcgraph_entry:                   |  x64_sys_call() {
           sleep-5204  [003] 15822.668975: funcgraph_entry:                   |    __x64_sys_execve() {
           sleep-5204  [003] 15822.668975: funcgraph_entry:                   |      getname() {
           sleep-5204  [003] 15822.668975: funcgraph_entry:                   |        getname_flags.part.0() {
           sleep-5204  [003] 15822.668976: funcgraph_entry:        1.209 us   |          kmem_cache_alloc();
           sleep-5204  [003] 15822.668977: funcgraph_entry:        1.232 us   |          __check_object_size();
           sleep-5204  [003] 15822.668979: funcgraph_exit:         3.477 us   |        }
           sleep-5204  [003] 15822.668979: funcgraph_exit:         4.012 us   |      }
...
...
```
It shows kernel function call entry/exit pairs with per-call durations and which task/CPU executed them.  
Let's take samples to see how to read these info:  
- Example:
`sleep-5204  [003] 15822.668936: funcgraph_entry:        2.263 us   |  mutex_unlock();`
	- Process sleep (PID 5204) on CPU 3; at timestamp 15822.668936; ftrace recorded entering mutex_unlock(), and the call itself took 2.263 µs (self time shown on entry lines by trace-cmd).
Nested example:
```text
	|  __f_unlock_pos() {
	|    mutex_unlock();
	|  }
```
	- `__f_unlock_pos` called and inside it `mutex_unlock` was invoked.
- MM/fault sequence:  
	- `handle_mm_faul`t / `__handle_mm_fault` / `handle_pte_fault` / `do_fault` / `do_read_fault`
	- That chain shows the kernel resolving a page fault: user thread hit a fault, kernel walked VMAs (find_vma), handled page-table fault, and called do_read_fault which took ~15 µs. The successive funcgraph_exit lines with "+ 15.938 us" etc. show cumulative time spent in the nested calls.	
- Context switch marker:
	- Later outputs you saw (in earlier examples) like "kworker-4950 => -0" indicate the CPU switched tasks — useful to see when the traced thread stopped and idle or another task ran.
- Small helper calls repeated many times:  
	- Functions like `nsecs_to_jiffies()`, `rcu_all_qs()`, `vmacache_find(`) are short helpers called frequently; each is microsecond- or sub-microsecond-scale and produce many lines.

Let's take a close look on what the timings mean and where to look for problems:  
- Per-line times are typically:  
	- Self-time: time spent in that function body excluding callees (shown on funcgraph_entry lines).
	- Cumulative/subtree time: shown on funcgraph_exit or lines with "+" / aggregate markers — indicates total time inside that function including its callees.
- To find hotspots, scan for large cumulative times (tens or hundreds of µs in a short sleep trace). In the sample:  
	- do_read_fault and handle_mm_fault subtree shows ~15–20 µs for a page-fault handling path — normal for a fault that allocates pages or handles I/O.
	- Any "!"/"+" long durations indicate functions worth investigating for latency.


### Lab 5 — Event tracing (tracepoints)
List some tracepoint events and enable one:
```bash
sudo ls /sys/kernel/debug/tracing/events/sched | sed -n '1,80p'
sudo sh -c 'echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable'
sudo sh -c 'echo 1 > /sys/kernel/debug/tracing/tracing_on'
# generate scheduling activity: run a CPU workload
for i in 1 2 3; do dd if=/dev/zero of=/dev/null bs=1M count=50 & done
sleep 1
sudo sh -c 'echo 0 > /sys/kernel/debug/tracing/tracing_on'
sudo sed -n '1,120p' /sys/kernel/debug/tracing/trace
sudo sh -c 'echo 0 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable'
```
You may see someting like this:
```text
# tracer: nop
#
# entries-in-buffer/entries-written: 10842/10842   #P:4
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| / _-=> migrate-disable
#                              |||| /     delay
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
#              | |         |   |||||     |         |
           <...>-5231    [002] d.... 17001.433152: sched_switch: prev_comm=sh prev_pid=5231 prev_prio=120 prev_state=Z ==> next_comm=swapper/2 next_pid=0 next_prio=120
          <idle>-0       [003] d.... 17001.433276: sched_switch: prev_comm=swapper/3 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=sudo next_pid=5230 next_prio=120
          <idle>-0       [000] d.... 17001.433550: sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=sudo next_pid=5229 next_prio=120
            sudo-5230    [003] d.... 17001.433786: sched_switch: prev_comm=sudo prev_pid=5230 prev_prio=120 prev_state=Z ==> next_comm=swapper/3 next_pid=0 next_prio=120
          <idle>-0       [001] d.... 17001.433980: sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=systemd-logind next_pid=936 next_prio=120
  systemd-logind-936     [001] d.... 17001.434175: sched_switch: prev_comm=systemd-logind prev_pid=936 prev_prio=120 prev_state=S ==> next_comm=systemd-journal next_pid=544 next_prio=119
 systemd-journal-544     [001] d.... 17001.434250: sched_switch: prev_comm=systemd-journal prev_pid=544 prev_prio=119 prev_state=R ==> next_comm=in:imuxsock next_pid=943 next_prio=120
     in:imuxsock-943     [001] d.... 17001.434372: sched_switch: prev_comm=in:imuxsock prev_pid=943 prev_prio=120 prev_state=S ==> next_comm=systemd-journal next_pid=544 next_prio=119
          <idle>-0       [002] d.... 17001.434431: sched_switch: prev_comm=swapper/2 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=rs:main Q:Reg next_pid=945 next_prio=120
   rs:main Q:Reg-945     [002] d.... 17001.434534: sched_switch: prev_comm=rs:main Q:Reg prev_pid=945 prev_prio=120 prev_state=S ==> next_comm=swapper/2 next_pid=0 next_prio=120
 systemd-journal-544     [001] d.... 17001.434635: sched_switch: prev_comm=systemd-journal prev_pid=544 prev_prio=119 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
            sudo-5229    [000] d.... 17001.435416: sched_switch: prev_comm=sudo prev_pid=5229 prev_prio=120 prev_state=R+ ==> next_comm=kworker/0:0 next_pid=2379 next_prio=120
     kworker/0:0-2379    [000] d.... 17001.435426: sched_switch: prev_comm=kworker/0:0 prev_pid=2379 prev_prio=120 prev_state=I ==> next_comm=sudo next_pid=5229 next_prio=120
            sudo-5229    [000] d.... 17001.435579: sched_switch: prev_comm=sudo prev_pid=5229 prev_prio=120 prev_state=Z ==> next_comm=swapper/0 next_pid=0 next_prio=120
          <idle>-0       [002] d.... 17001.435659: sched_switch: prev_comm=swapper/2 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=systemd next_pid=1 next_prio=120
          <idle>-0       [001] d.... 17001.435665: sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=bash next_pid=4933 next_prio=120
         systemd-1       [002] d.... 17001.436062: sched_switch: prev_comm=systemd prev_pid=1 prev_prio=120 prev_state=S ==> next_comm=swapper/2 next_pid=0 next_prio=120
            bash-4933    [001] d.... 17001.436224: sched_switch: prev_comm=bash prev_pid=4933 prev_prio=120 prev_state=S ==> next_comm=kworker/u256:3 next_pid=5205 next_prio=120
  kworker/u256:3-5205    [001] d.... 17001.436301: sched_switch: prev_comm=kworker/u256:3 prev_pid=5205 prev_prio=120 prev_state=I ==> next_comm=swapper/1 next_pid=0 next_prio=120
          <idle>-0       [002] d.... 17001.436489: sched_switch: prev_comm=swapper/2 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=sshd next_pid=4932 next_prio=120
          <idle>-0       [003] d.... 17001.436524: sched_switch: prev_comm=swapper/3 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=rcu_sched next_pid=14 next_prio=120
       rcu_sched-14      [003] d.... 17001.436536: sched_switch: prev_comm=rcu_sched prev_pid=14 prev_prio=120 prev_state=I ==> next_comm=swapper/3 next_pid=0 next_prio=120
            sshd-4932    [002] d.... 17001.436896: sched_switch: prev_comm=sshd prev_pid=4932 prev_prio=120 prev_state=S ==> next_comm=swapper/2 next_pid=0 next_prio=120
          <idle>-0       [003] d.... 17001.440574: sched_switch: prev_comm=swapper/3 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=rcu_sched next_pid=14 next_prio=120
       rcu_sched-14      [003] d.... 17001.440590: sched_switch: prev_comm=rcu_sched prev_pid=14 prev_prio=120 prev_state=I ==> next_comm=swapper/3 next_pid=0 next_prio=120
          <idle>-0       [003] d.... 17001.448707: sched_switch: prev_comm=swapper/3 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=rcu_sched next_pid=14 next_prio=120
       rcu_sched-14      [003] d.... 17001.448722: sched_switch: prev_comm=rcu_sched prev_pid=14 prev_prio=120 prev_state=I ==> next_comm=swapper/3 next_pid=0 next_prio=120
...
...
```
Here `tracer: nop` where "nop" means no special function tracer; only events are recorded.  
Let's take a look at a typical line:
```text
<...>-5231    [002] d.... 17001.433152: sched_switch: prev_comm=sh prev_pid=5231 prev_prio=120 prev_state=Z ==> next_comm=swapper/2 next_pid=0 next_prio=120
```
where:  
- `prev_comm` / `prev_pid` — the task that was running before the context switch (name and pid).
- `prev_prio` — kernel scheduling priority of the prev task. Higher numeric priority here is the kernel static priority representation (not nice-to-have niceness); typical kernel threads show 120 for default.
- `prev_state` — the previous task’s state at the time it stopped running:
	- R — running
	- S — sleeping (interruptible)
	- D — uninterruptible sleep (usually IO)
	- Z — zombie
	- I — idle or inactive (letters come from task state bits; Z in your log indicates the process was a zombie when it was descheduled)
- ==> separates prev and next context.
- `next_comm` / `next_pid` / `next_prio` — the task chosen to run next on that CPU and its scheduling priority.
- Please note that swapper/N (or ) is the idle task for CPU N; next_pid=0 is the idle task.
Here's a short interpretion of the line:
```text
sudo-5229 [000] d.... 17001.435416: sched_switch: prev_comm=sudo prev_pid=5229 prev_prio=120 prev_state=R+ ==> next_comm=kworker/0:0 next_pid=2379 next_prio=120
```
sudo was running (R) and then yielded to a kernel worker. The '+' appended to prev_state often denotes the task was preempted while in a special state (kernel-specific; exact meaning can vary by kernel version and flags).


You may also want to ask "Why are there many short-lived switches":  
- Enabling sched_switch records every context switch. On a normal desktop/VM there are many kernel threads, interrupts and short-lived user tasks; thus thousands of entries quickly accumulate.
- The for-loop workload (three dd processes) produced CPU and I/O activity and therefore many scheduling events, but system services, kworkers, rcu, idle tasks, etc., also generate switches.

### Cleanup after the labs
```bash
sudo sh -c 'echo nop > /sys/kernel/debug/tracing/current_tracer'
sudo sh -c 'echo > /sys/kernel/debug/tracing/set_ftrace_pid'
sudo sh -c 'echo 0 > /sys/kernel/debug/tracing/tracing_on'
sudo sh -c 'echo 0 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable'
```
