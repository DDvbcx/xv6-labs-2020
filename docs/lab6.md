## Lab7: Multithreading

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
>为什么两个线程都丢失了键，而不是一个线程？确定可能导致键丢失的具有2个线程的事件序列。在answers-thread.txt中提交您的序列和简短解释。[!TIP] 为了避免这种事件序列，请在notxv6/ph.c中的put和get中插入lock和unlock语句，以便在两个线程中丢失的键数始终为0。相关的pthread调用包括：
>pthread_mutex_t lock; // declare a lock
>pthread_mutex_init(&lock, NULL); // initialize the lock
>pthread_mutex_lock(&lock); // acquire lock
>pthread_mutex_unlock(&lock); // release lock
>当make grade说您的代码通过ph_safe测试时，您就完成了，该测试需要两个线程的键缺失数为0。在此时，ph_fast测试失败是正常的。

使用 pthread 的锁机制，在保证速度的同时解决同步问题。多个写线程同时写 table 时，存在同步问题，所以数据错误，在 put 时加锁即可解决。但是不能直接一个锁，如果对所有 table 共用一个锁，那么 put 实际上单线程没差，甚至多线程比单线程还要慢，这是因为每一次只能有一个线程在写 table，并在多线程等待 lock 还会花时间，因此比单线程还慢，pf_fast 测试无法通过。
因为每个 bucket（==5） 之间是互不影响的，因此对每个 bucket 一个独立的锁即可。即使这样，也只能在 thread_num <= 5 时加速，因为同一时间最多 5 个线程在写，当超过 5 时多余的线程会被阻塞。   

```c
// notxv6/ph.c
pthread_mutex_t lock[NBUCKET] = { PTHREAD_MUTEX_INITIALIZER }; // 每个散列桶一把锁

static 
void put(int key, int value)
{
  
  int i = key % NBUCKET;
  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    pthread_mutex_lock(&lock[i]);
    // the new is new.
    insert(key, value, &table[i], table[i]);
    pthread_mutex_unlock(&lock[i]);
  }
  
}
```

### Barrier

> 您的目标是实现期望的屏障行为。除了在ph作业中看到的lock原语外，还需要以下新的pthread原语；详情请看这里和这里。
>// 在cond上进入睡眠，释放锁mutex，在醒来时重新获取
>pthread_cond_wait(&cond, &mutex);
>// 唤醒睡在cond的所有线程
>pthread_cond_broadcast(&cond);

本 lab 用在实现多线程同步屏障，即必须等所有线程均到一个点，才能继续执行。实现很简单，在 barrier 中将 bstate.nthread ++，如果 bstate.nthread == nthread 则通过 pthread_cond_broadcast 唤醒所有线程，否则通过 pthread_cond_wait 阻塞线程。注意锁的问题即可，代码如下：

```c
// notxv6/barrier.c
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread ++;
  if(bstate.nthread == nthread){
    bstate.nthread = 0;
    bstate.round ++;
    pthread_cond_broadcast(&bstate.barrier_cond); 
  } else {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex); // 本身就会释放锁，被唤醒后再次获得锁
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```