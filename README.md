# Threaded NAPI
This patch set is the final project of the Kernel Programming class of Prof. Abeni at Scuola Superiore Sant'Anna.
All it does is creating a new kernel thread for each CPU (using the CPU hot-plug interface) where NAPI poll is done (instead of doing it in a softirq). Doing that is slightly less efficient but more predictable since softirq can execute part of its work inside another task's context.

## How to use it
In order to build a kernel with support for threaded NAPI you need to enable `CONFIG_THREADED_NAPI` in `.config`.

When this config is enabled, threaded napi is enabled by default. In order to disable it, you can pass `threaded_napi=off` in the command line parameters of the kernel or you can also disable it at runtime with:
```bash
sysctl net.core.knapid_enabled=0
```

## Implementation
After having an idea of what all of that kernel code does, the implementation is pretty simple. All the logic is contained in the file `net/core/dev.c`. The following explanation does not consider enabling threaded napi at runtime as this will be discussed later.

First of all, a new set of kernel threads, one for each processor, is created inside the function `net_dev_init`, which gets called early in the boot process, with: 
```c
smpboot_register_percpu_thread(&napi_threads);
```

The kernel thread executes this simple algorithm:
```
if threaded napi is enabled and there is work to do
then 
    napi_poll_everything
else 
    sleep until someone wakes it up
```
Where `napi_poll_everything` executes the same code as the old `net_rx_action` (in fact it was just renamed and the new `net_rx_action` basically just calls `napi_poll_everything`).

Finally, every hardware interrupt from the NIC will trigger a call to `____napi_schedule` which has been modified to choose between raising a softirq or waking up the newly created dedicated napi thread:
```c
if (!knapid_enabled){
    __raise_softirq_irqoff(NET_RX_SOFTIRQ);
} else {
    wakeup_napid();
}
```

### Switching napi mode at runtime
In order to make it possible to switch napi mode at runtime, a few little changes were added:
 1. `napi_poll_everything` must be protected from concurrent access since it could be executed independently from softirq or from the kernel thread. This was done using a cpu-local flag whose read and write were protected by disabling preemption.
 2. If there is more work to do after `napi_poll_everything`, the correct napi mode should be chosen.

If the napi mode is switched once, there are no problems since the current mode will end its work as soon as possible before leaving it to continue to the other mode, while new interrupts will just call the correct mode (which may fail executing if the other mode is still running).
If the napi mode is switched more than once in a small time, there may be some problems, therefore once switched the user is forced to wait a cooldown period before switching back modes again.

There may have been more correct and more clean solutions to this problem (using a reader/writer lock? a sequential lock? another synchronization mechanism?) but I think this one was the best choice since it's simple, it works and its drawback is minimal for the user.

## Performance test
To check that the proposed solution works, two tests were carried:
 - ping test
 - iperf test

 All tests were carried on by running the modified kernel in a VM with 1 vCPU and attaching it to a virtual bridge in order to be able to communicate with the host machine. 
 The host machine is a laptop running Fedora 30 with Linux 5.1.12:
  - CPU: Intel(R) Core(TM) i7-5500U CPU @ 2.40GHz (2 physical cores; 2 threads per core).
  - RAM: 16GB.

 ### ping test

 An average ping between the guest VM (1 vCPU) and the host laptop () among 10000 samples was calculated for both modes:
  - softirq: 212us
  - knapid: 241us

You can see that there's no big difference.

### iperf test

The iperf tool was run three times, only in download, in order to check:
 - maximum throughput and cpu utilization
 - cpu utilization at 1Gbit
 - cpu utilization at 100Mbit

 Cpu utilization was retrieved using the `top` utility inside the VM. The following legend may be useful to read its output:
  - us: user
  - sy: system
  - ni: niced processes
  - id: idle
  - wa: waiting for I/O
  - hi: hardware interrupt
  - si: software interrupt
  - st: cpu "stolen" by guest VM

Maximum throughput:
  - sotfirq: 2.25 Gbits/sec
  - knapid: 2.31 Gbits/sec

Cpu utilization at maximum throughput:
 - sotfirq: %Cpu(s):  0.4 us, 56.3 sy,  0.0 ni,  2.0 id,  0.0 wa,  0.0 hi, 41.3 si,  0.0 st
 - knapid: %Cpu(s):  0.3 us, 99.7 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st

Cpu utilization at 1Gbit:
 - sotfirq: %Cpu(s):  0.9 us, 16.5 sy,  0.0 ni, 64.2 id,  0.0 wa,  0.0 hi, 18.4 si,  0.0 st
 - knapid: %Cpu(s):  0.8 us, 49.2 sy,  0.0 ni, 49.6 id,  0.0 wa,  0.0 hi,  0.4 si,  0.0 st

Cpu utilization at 100Mbit:
 - sotfirq: %Cpu(s):  0.0 us,  1.1 sy,  0.0 ni, 98.9 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
 - knapid: %Cpu(s):  0.0 us, 10.3 sy,  0.0 ni, 89.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st

 From these results, you can see that, while throughput remains fairly the same, the cpu utilization is higher for the threaded napi. This was expected due to the increased overheads in context switch, which softirq avoids by running in another process' context.
