## Lab3: page tables

### Print a page table 
> 定义一个名为vmprint()的函数。它应当接收一个pagetable_t作为参数，并以下面描述的格式打印该页表。在exec.c中的return argc之前插入if(p->pid==1) vmprint(p->pagetable)，以打印第一个进程的页表。如果你通过了pte printout测试的make grade，你将获得此作业的满分。 

和虚拟地址有关的内核函数位于 `kernel/vm.c` 中，因此 vmprint 也放在里面。
```c
// kernel/vm.c
void 
vmprint(pagetable_t pagetable)
{
  printf("page table %p\n", pagetable);
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if(pte & PTE_V){
      printf("..%d: pte %p pa %p\n", i, pte, PTE2PA(pte));
      pagetable_t second = (pagetable_t)PTE2PA(pte);
      for(int j = 0; j < 512; j++){
        pte = second[j];
        if(pte & PTE_V){
          printf(".. ..%d: pte %p pa %p\n", j, pte, PTE2PA(pte));
          pagetable_t third = (pagetable_t)PTE2PA(pte);
          for(int k = 0; k < 512; k++){
            pte = third[k];
            if(pte & PTE_V){
              printf(".. .. ..%d: pte %p pa %p\n", k, pte, PTE2PA(pte));
            }
          }
        }
      }
    }
  }
}
```

这里有三个 for 循环，因为是三级页表。

接下来，在 exec.c 中改动，当且仅当 pid 为 1 才调用 vmprint。
```c
// kernel/exec.c
int
exec(char *path, char **argv)
{
  // ...
  // 调用 vmprint
  if(p->pid == 1){
    vmprint(p->pagetable);
  }
  
  return argc; // this ends up in a0, the first argument to main(argc, argv)
  // ...
}
```

### Speed up system calls
> Some operating systems (e.g., Linux) speed up certain system calls by sharing data in a read-only region between userspace and the kernel. This eliminates the need for kernel crossings when performing these system calls. To help you learn how to insert mappings into a page table, your first task is to implement this optimization for the `getpid()` system call in xv6.

这个实验其实就是跳过系统调用加速读取内核。解决该题目的关键是创建一个可读 PTE 指向一块内存，该 PTE 是用户态和内核态共享的，那么用户态就可以直接读取这块内核数据，而无需经过复杂的系统调用。
porc 结构是内核态数据，用户态无法直接读取，因此需要通过系统调用 getpid 来读到 pid。这里需要我们创建一个共享 PTE，将虚拟地址 USYSCALL 映射到 pid 的物理地址，这样用户态直接读取 USYSCALL 就可以获取到 pid 了。 ugetpid 位于 `user/ulib.c`， 意味用户态的 getpid，代码如下：

```c
int
ugetpid(void)
{
  struct usyscall *u = (struct usyscall *)USYSCALL;
  return u->pid;
}
```
主要的代码实现位于 `kernel/proc.c` 中。首先，我们 proc 新增一个字段，存放 usyscall 的地址：

 ```c
// kernel/proc.h
struct proc {
  // ...  
  struct usyscall *usyscall;
}
 ```

在 allocproc 时，为其分配一块物理内存。
```c
// kernel/proc.c
static struct proc*
allocproc(void)
{
  // ...
  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // alloc usyscall
  if((p->usyscall = (struct usyscall *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  // ...
}
```

页表的初始化位于函数 proc_pagetable 中，通过 mappages 在页表中注册新的 PTE，参考 TRAPFRAME 的方式，将 USYSCALL 映射到 p->usyscall 中。注意映射之前要先赋值 p->usyscall->pid = p->pid。

```c
// kernel/proc.c
pagetable_t
proc_pagetable(struct proc *p)
{
  pagetable_t pagetable;
  // ...
  // map the trapframe page just below the trampoline page, for
  // trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  // map one read-only page at USYSCALL, store the PID
  p->usyscall->pid = p->pid;
  if(mappages(pagetable, USYSCALL, PGSIZE,(uint64)p->usyscall, PTE_R | PTE_U) < 0){
    uvmunmap(pagetable, USYSCALL, 1, 0);
    uvmfree(pagetable, 0);
  }

  return pagetable;
}
```

新增完映射后，我们需要在进程 free 时对其解映射，位于函数 proc_freepagetable 中，可以参考 TRAPFRAME 的解映射。

```c
// kernel/proc.c
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmunmap(pagetable, USYSCALL, 1, 0);
  uvmfree(pagetable, sz);
}
```

至此，页表相关的操作就完成了。但是在 freepagetable 只解映射了页表，并没有释放给 usyscall 分配的物理内存，因此需要在 freeproc 处将其释放，可以参考 trapframe 的释放方式。

```c
// kernel/proc.c
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  // ...
  if(p->usyscall)
    kfree((void*)p->usyscall);
  p->usyscall = 0;
}
```