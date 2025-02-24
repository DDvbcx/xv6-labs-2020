## Lab4: traps

### Uthread: switching between threads
>您的工作是提出一个创建线程和保存/恢复寄存器以在线程之间切换的计划，并实现该计划。完成后，make grade应该表明您的解决方案通过了uthread测试。     

在用户态进行线程切换，首先，需要给 thread 结构一个字段用来保存相关寄存器，我们使用 context 字段；
```c
// user/uthread.c
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct context context;       // 用户进程上下文
};
```

在 thread_create 时，将传入的 func 赋值给 context 的 ra，这样当切换到该 thread 时就会返回到 func 处从而执行。同时，保存 thread 的栈起始地址，注意栈是从高到低生长的，因此要用栈的最高地址初始化。这里就有个问题了，thread 中不是有个 stack 吗，为什么还需要 sp 来保存它？stack 只是一个被分配了的连续空间，OS 并没有把它认为是线程的栈，没有任何意义，OS 只会通过 sp 寄存器来确定线程的栈，因此这里操作的最终目的就是将 stack 的地址给 sp 寄存器，让 OS 知道这片空间是栈。

 ```c
// user/uthread.c
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->context.ra = (uint64)func;
  t->context.sp = (uint64)(t->stack) + STACK_SIZE - 1;  // sp初始指向栈底
}
 ```

在 thread_schedule 中只需要简单调用一下 thread_switch 即可，调用方式和 swtch 一样：

```c
// user/uthread.c
void 
thread_schedule(void)
{
  // ...
  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)&t->context, (uint64)&next_thread->context);
  }
  // ...
}
```

thread_switch 的实现在 uthread_switch.S 中，需要自己实现，但其实现和 swtch.S 完全一致，就是保存原寄存器，读取目标寄存器。

```assembly
thread_switch:
	/* YOUR CODE HERE */
	# 仿照switch.S

	# 当前线程
	sd ra, 0(a0)
	sd sp, 8(a0)
	sd s0, 16(a0)
	sd s1, 24(a0)
	sd s2, 32(a0)
	sd s3, 40(a0)
	sd s4, 48(a0)
	sd s5, 56(a0)
	sd s6, 64(a0)
	sd s7, 72(a0)
	sd s8, 80(a0)
	sd s9, 88(a0)
	sd s10, 96(a0)
	sd s11, 104(a0)

    # 目标线程
	ld ra, 0(a1)
	ld sp, 8(a1)
	ld s0, 16(a1)
	ld s1, 24(a1)
	ld s2, 32(a1)
	ld s3, 40(a1)
	ld s4, 48(a1)
	ld s5, 56(a1)
	ld s6, 64(a1)
	ld s7, 72(a1)
	ld s8, 80(a1)
	ld s9, 88(a1)
	ld s10, 96(a1)
	ld s11, 104(a1)
	ret    /* return to ra */
```

### Using threads
