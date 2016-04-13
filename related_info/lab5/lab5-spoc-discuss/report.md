通过在`kern/process/proc.c`与`kern/schedule/sched.c`中添加调试输出。以`user/exit.c`作为用户进程，运行`make qemu`的输出如下：

```
...
++ setup timer interrupts
schedule: current process pid = 0, state = 2
schedule: next process pid = 1, runs = 1, state = 2
proc_run: current pid = 0, state = 2
proc_run: before switch_to: prev pid = 0, next pid = 1
do_fork: alloc_proc: state = 0
do_fork: ready to wakeup pid = 2, state = 0
wakeup_proc: pid = 2, state = 0
wakeup_proc: done, state = 2
do_wait: pid = 0
schedule: current process pid = 1, state = 1
schedule: next process pid = 2, runs = 1, state = 2
proc_run: current pid = 1, state = 1
proc_run: before switch_to: prev pid = 1, next pid = 2
kernel_execve: pid = 2, name = "exit".
I am the parent. Forking the child...
do_fork: alloc_proc: state = 0
do_fork: ready to wakeup pid = 3, state = 0
wakeup_proc: pid = 3, state = 0
wakeup_proc: done, state = 2
I am parent, fork a child pid 3
I am the parent, waiting now..
do_wait: pid = 3
schedule: current process pid = 2, state = 1
schedule: next process pid = 3, runs = 1, state = 2
proc_run: current pid = 2, state = 1
proc_run: before switch_to: prev pid = 2, next pid = 3
I am the child.
do_yield: pid = 3
schedule: current process pid = 3, state = 2
schedule: next process pid = 3, runs = 2, state = 2
do_yield: pid = 3
schedule: current process pid = 3, state = 2
schedule: next process pid = 3, runs = 3, state = 2
do_yield: pid = 3
schedule: current process pid = 3, state = 2
schedule: next process pid = 3, runs = 4, state = 2
do_yield: pid = 3
schedule: current process pid = 3, state = 2
schedule: next process pid = 3, runs = 5, state = 2
do_yield: pid = 3
schedule: current process pid = 3, state = 2
schedule: next process pid = 3, runs = 6, state = 2
do_yield: pid = 3
schedule: current process pid = 3, state = 2
schedule: next process pid = 3, runs = 7, state = 2
do_yield: pid = 3
schedule: current process pid = 3, state = 2
schedule: next process pid = 3, runs = 8, state = 2
before child exit.do_exit: pid = 3, state = 2, error code = -66436
do_exit: done, state = 3
wakeup_proc: pid = 2, state = 1
wakeup_proc: done, state = 2
schedule: current process pid = 3, state = 3
schedule: next process pid = 2, runs = 2, state = 2
proc_run: current pid = 3, state = 3
proc_run: before switch_to: prev pid = 3, next pid = 2
proc_run: after switch_to: prev pid = 2, next pid = 3
do_wait: found: pid = 3, state = 3
do_wait: pid = 3
do_wait: pid = 0
waitpid 3 ok.
exit pass.
do_exit: pid = 2, state = 2, error code = 0
do_exit: done, state = 3
wakeup_proc: pid = 1, state = 1
wakeup_proc: done, state = 2
schedule: current process pid = 2, state = 3
schedule: next process pid = 1, runs = 2, state = 2
proc_run: current pid = 2, state = 3
proc_run: before switch_to: prev pid = 2, next pid = 1
proc_run: after switch_to: prev pid = 1, next pid = 2
do_wait: found: pid = 2, state = 3
schedule: current process pid = 1, state = 2
schedule: next process pid = 1, runs = 3, state = 2
do_wait: pid = 0
all user-mode processes have quit.
init check memory pass.
do_exit: pid = 1, state = 2, error code = 0
kernel panic at kern/process/proc.c:463:
    initproc exit.
...
```

状态码定义在`kern/process/proc.h`中：

```c
enum proc_state {
    PROC_UNINIT = 0,  // uninitialized
    PROC_SLEEPING,    // sleeping
    PROC_RUNNABLE,    // runnable(maybe running)
    PROC_ZOMBIE,      // almost dead, and wait parent proc to reclaim his resource
};
```

简要分析一下用户进程 (pid = 2) 的生命周期：

```
do_fork: alloc_proc: state = 0
do_fork: ready to wakeup pid = 2, state = 0
wakeup_proc: pid = 2, state = 0
wakeup_proc: done, state = 2
```

在`do_fork`中，进程被分配出来，处于PROC_UNINIT状态，之后分配`pid`号，并初始化各项资源。接下来被唤醒，进入PROC_RUNNABLE态。

```
I am the parent, waiting now..
do_wait: pid = 3
schedule: current process pid = 2, state = 1
schedule: next process pid = 3, runs = 1, state = 2
proc_run: current pid = 2, state = 1
proc_run: before switch_to: prev pid = 2, next pid = 3
I am the child.
```

父进程 (pid = 2) fork出了子进程 (pid = 3) 后，请求 wait，进入PROC_SLEEPING态。接下来调度子进程执行，在父进程到子进程的上下文切换完成后，子进程占用 CPU，输出提示信息`I am the child`。

子进程退出后（处于PROC_ZOMBIE状态），

```
proc_run: current pid = 3, state = 3
proc_run: before switch_to: prev pid = 3, next pid = 2
proc_run: after switch_to: prev pid = 2, next pid = 3
do_wait: found: pid = 3, state = 3
do_wait: pid = 3
do_wait: pid = 0
waitpid 3 ok.
exit pass.
```

完成子进程到父进程的上下文切换，这里`after switch_to`行表明已经返回到上一次从父进程到子进程的上下文切换的那段程序。

```
do_exit: pid = 2, state = 2, error code = 0
do_exit: done, state = 3
```

接下来，父进程从PROC_RUNNABLE态进入PROC_ZOMBIE态，完成资源回收。
