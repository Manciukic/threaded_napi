# Threaded NAPI
This patch set is the final project of the Kernel Programming class of Prof. Abeni at Scuola Superiore Sant'Anna .
All it does is creating a new kernel thread for each CPU (using the cpu hotplug interface) where NAPI poll is done (instead of doing it in a softirq). Doing that is slightly less efficient but more predictable since softirq can execute part of its work inside another task's context.

In order to enable threaded NAPI you need to enable CONFIG\_THREADED\_NAPI in .config .
