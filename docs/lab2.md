## Lab2: system calls   

### System call tracing  
>在本作业中，您将添加一个系统调用跟踪功能，该功能可能会在以后调试实验时对您有所帮助。您将创建一个新的trace系统调用来控制跟踪。它应该有一个参数，这个参数是一个整数“掩码”（mask），它的比特位指定要跟踪的系统调用。例如，要跟踪fork系统调用，程序调用trace(1 << SYS_fork)，其中SYS_fork是kernel/syscall.h中的系统调用编号。如果在掩码中设置了系统调用的编号，则必须修改xv6内核，以便在每个系统调用即将返回时打印出一行。该行应该包含进程id、系统调用的名称和返回值；您不需要打印系统调用参数。trace系统调用应启用对调用它的进程及其随后派生的任何子进程的跟踪，但不应影响其他进程。

该实验需要实现一个系统调用 trace, 用来监控进程调用的系统调用，通过 mask 来决定哪些系统调用需要监视。与进程相关联的代码在 `kernel/proc.c` 和 `kernel/sysproc.c` 中。    

1. 每个进程都需要有个 mask，以此来决定 trace 监控的系统调用，所以我们需要给进程的结构(struct proc)加上一个新的字段。  
```c
// kernel/proc.h
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)

  int trace_mask;              // mask 
};
```

2. 在创建新进程时，需要初始化 trace_mask 为 0, 跟进程操作有关的在 `kernel/proc.c` 中，创建进程会调用 allocproc 函数来进行，因此需要在其中对 trace_mask 进行初始化。   

```c
// kernel/proc.c
static struct proc*
allocproc(void)
{
  struct proc *p;
  // ...
  p->trace_mask = 0;  // 初始化为 0
  return p;
}
```

3. 子进程也需要一并监控，因此在 fork 时要将 trace_mask 传递下去，更改 fork 的代码：


```c
// kernel/proc.c
int
fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();
  // ...
  np->trace_mask = p->trace_mask;
  // ...
}
```

4. 如何打印，每当 mask 指明的系统调用被调用时，进行一次打印，方便系统追踪。而系统调用的入口函数是 syscall, 

```c
// kernel/syscall.c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;  // 系统调用编号
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // Use num to lookup the system call function for num, call it,
    // and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();
    // 是否匹配
    if((p->trace_mask >> num) & 1){
      printf("%d: syscall %s -> %d\n",p->pid, syscall_names[num], p->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```
题目完成。   

### Sysinfo 
>在这个作业中，您将添加一个系统调用sysinfo，它收集有关正在运行的系统的信息。系统调用采用一个参数：一个指向struct sysinfo的指针（参见kernel/sysinfo.h）。内核应该填写这个结构的字段：freemem字段应该设置为空闲内存的字节数，nproc字段应该设置为state字段不为UNUSED的进程数。我们提供了一个测试程序sysinfotest；如果输出“sysinfotest: OK”则通过。   


