# CS 537 Midterm 2: Concurrency Mega Study Guide

Based on patterns from Sample Exams 1, 2, and 3. Every concept is
explained with exam-solving strategies.

---

## 1. THREADS FUNDAMENTALS

### What threads share vs. what's private

This is tested in literally every single sample exam. Here's the
definitive breakdown:

**Shared across threads (same process):** - Code (text segment) - Heap -
Global variables - Open file descriptors - Page table (and therefore the
Page Table Base Register / PTBR) - Address space

**Private to each thread:** - Stack (each thread gets its own) -
Registers (including the instruction pointer / program counter) - Thread
ID

**How to answer exam questions:**

- "Threads share the same PTBR" = **True**. Same process = same page
  table = same PTBR.
- "Threads share the instruction pointer" = **False**. Each thread
  runs different code at different points, so each needs its own IP.
- "Threads share registers" = **False**. The physical hardware
  registers are the same, but the OS saves/restores register state on
  context switches, so each thread has its own *logical* register set.
  However, if the question asks "do two threads read/write the same
  physical hardware register" the answer is **True** since there's
  only one physical register set per core.
- "Code running in one thread cannot access the stack of another
  thread" = **False**. Stacks live in the same address space. You
  *can* access another thread's stack via pointers (it's just a bad
  idea). The stacks are separate allocations, but not protected from
  each other.
- "Which of these is different for each thread?" = **Stack** (not
  heap, not page table, not code).

### User-level vs. Kernel-level threads

**User-level threads:** - The OS knows nothing about them. The OS sees
one process. - Thread creation, exit, join, lock/unlock are all handled
in user space. - Operations are very fast (no system call overhead). -
**Big downside:** if one thread blocks (e.g., on I/O), the entire
process blocks because the OS only sees one schedulable entity. - The OS
scheduler does NOT schedule user-level threads; the user-level thread
library does. - Porting is easier because the thread library is in user
space. - "Must rely on the OS scheduler for scheduling" = **False** for
user-level threads (this is a trick, the user thread library does its
own scheduling).

**Kernel-level threads:** - The OS is fully aware of each thread. -
Context switching involves a system call (slower). - One thread blocking
doesn't block the whole process.

**Exam pattern:** "With user-level threads, the OS implements: (a)
thread creation and exit (b) creation, exit, and join (c) creation,
exit, join, and lock/unlock (d) none of the above" = **(d) none of the
above**. The OS doesn't implement any of these for user-level threads.

---

## 2. THREAD CREATION WITH pthread_create

### The API

```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine)(void *), void *arg);
```

- `thread`: pointer to a pthread_t that stores the thread ID
- `attr`: usually NULL for defaults
- `start_routine`: the function the new thread will run
- `arg`: a single void\* argument passed to that function

### The Classic Race Condition Question (Q3, Sample 1)

```c
void *worker(void *arg) {
    int *i = (int *) arg;
    *i = *i + 1;
    pthread_exit(NULL);
}

int main(int argc, char *argv[]) {
    int x = 1;
    pthread_t p1;
    pthread_create(&p1, NULL, worker, (void *) &x);
    x = x + 1;
    printf("x = %d\n", x);
    return(0);
}
```

**Analysis:** The main thread passes `&x` to the worker. Both the main
thread and the worker can modify `x`. After `pthread_create`: - The
worker does `*i = *i + 1` (increments x) - The main does `x = x + 1`
(increments x) - The main prints x

The problem: we don't know the ordering. The worker might run before,
after, or interleaved with the main thread's `x = x + 1`.

Possible values of x at the printf: - If worker runs first: x goes 1 -\>
2 (worker) -\> 3 (main), prints 3 - If main runs `x = x + 1` first, then
worker runs: x goes 1 -\> 2 (main) -\> 3 (worker), prints 3 - If main
runs `x = x + 1` and printf before worker: prints 2 - There's also the
interleaving where both read x=1, both write x=2, main prints 2

Answer: **d. I can't tell** because the output depends on scheduling.

### The Pointer/Memory Allocation Pattern (Q49-51, Sample 1)

```c
void *printer(void *arg) {
    char *p = (char *) arg;
    printf("%c", *p);
    return NULL;
}

int main(int argc, char *argv[]) {
    pthread_t p[5];
    for (int i = 0; i < 5; i++) {
        char *c = malloc(sizeof(char));  // separate allocation each time!
        *c = 'a' + i;
        pthread_create(&p[i], NULL, printer, (void *) c);
    }
    for (int i = 0; i < 5; i++)
        pthread_join(p[i], NULL);
    return 0;
}
```

Key insight: each thread gets its own malloc'd character. There's no
shared mutable state. Each thread will print exactly one unique
character from {a, b, c, d, e}.

- "Can print 'bdeca'?" = **True**. Any permutation of {a,b,c,d,e} is
  possible because thread scheduling order is nondeterministic.
- "Can print 'aabbc'?" = **False**. Each character is allocated
  separately and each thread prints one character exactly once. No
  character can repeat.
- "Can deadlock or hang forever?" = **False**. No locks, no shared
  resources, no blocking. Every thread just prints and returns.

### The Dangerous Pointer Pattern (Q52-54, Sample 1)

```c
void *background_malloc(void *arg) {
    int **int_ptr = (int **) arg;
    *int_ptr = calloc(1, sizeof(int));  // allocates and zeros
    **int_ptr = 24;
    return NULL;
}

int main(int argc, char *argv[]) {
    pthread_t p1;
    int *result = NULL;
    pthread_create(&p1, NULL, background_malloc, &result);
    printf("%d\n", *result);  // DANGER: result might still be NULL!
    return 0;
}
```

There's no join or synchronization between the create and the printf.
Possible scenarios:

- "Can print 0?" = **True**. If the child has run calloc (which zeros
  memory) but hasn't done `**int_ptr = 24` yet, result points to
  zeroed memory, so `*result` = 0.
- "Can crash?" = **True**. If the child hasn't run at all yet, result
  is still NULL, and `*result` dereferences NULL = segfault.
- "Can hang forever?" = **False**. There's nothing that blocks
  indefinitely. The main thread will either crash or print something
  and return.

---

## 3. RACE CONDITIONS AND ASSEMBLY

### The mov/add/mov pattern

This is the bread and butter of these exams. The critical section for
incrementing a shared variable at some memory address looks like:

```asm
mov 0x123, %eax    # load value from memory into register
add $0x1, %eax     # increment register
mov %eax, 0x123    # store back to memory
```

This is THREE instructions, not atomic. A context switch can happen
between any of them.

**"Will this work correctly without locks on multiple threads?"**

Ask yourself: is there shared mutable state being accessed
non-atomically?

- If the code only reads from memory and does computation in registers
  (no write-back), it might be safe.
- If there's a read-modify-write to a shared address, it's
  **incorrect** without synchronization.

**Sample 1, Q4:**

```asm
mov 0x123, %eax
add $0x1, %eax
call printf
```

This only reads 0x123 into eax, adds 1, then prints. It never writes
back to 0x123. Since %eax is per-thread (each thread has its own
register context), this is safe. Answer: **Correctly**.

**Sample 1, Q5:**

```asm
mov 0x123, %ebx
mov 0x345, %ecx
add %ebx, %ecx
mov %ebx, 0x123
```

This writes %ebx back to 0x123. But wait, %ebx was loaded from 0x123 and
never modified (the add stores into ecx). So it's writing the same value
back. But another thread could have changed 0x123 in the meantime, and
this thread would overwrite it with a stale value. Answer:
**Incorrectly**.

### Tracing Assembly Interleavings (Sample 3, Q83-90)

For these questions, you need to track: - Each thread's register values
(registers are per-thread!) - The shared memory contents

**Strategy:** 1. Label which thread is running each instruction 2. Track
registers separately for Thread 0 and Thread 1 3. Track the shared
memory location 4. When a thread gets interrupted, its register values
are saved and restored later 5. When a thread does `mov 2000, %ax`, it
reads the CURRENT value at address 2000

---

## 4. LOCKS

### Three Core Properties of Locks

1. **Mutual Exclusion:** Only one thread in the critical section at a
   time.
2. **Progress (aka Liveness):** If no thread holds the lock, a thread
   trying to acquire it must eventually succeed. Threads outside the
   critical section cannot prevent others from entering.
3. **Bounded Waiting (optional but nice):** A thread waiting to acquire
   the lock will get it within a bounded number of turns. Prevents
   starvation.

**What is NOT a required property:** - **Fairness** (not a formal
requirement, though bounded waiting implies some fairness) -
**Priority** (locks don't respect thread priority by default) -
**Defined ordering** (basic locks don't guarantee FIFO)

### Spinlocks

The simplest lock. A thread "spins" (loops) checking a flag until it's
free.

```c
typedef struct { int flag; } lock_t;

void acquire(lock_t *lock) {
    while (xchg(&lock->flag, 1) == 1)
        ; // spin
}

void release(lock_t *lock) {
    lock->flag = 0;
}
```

Key facts for exams: - Spinlocks do **NOT** guarantee FIFO /
first-come-first-served ordering. Any thread can grab the lock when it
becomes free. - Spinlocks do **NOT** provide bounded waiting. A thread
can theoretically starve. - Spinlocks **waste CPU cycles** while
spinning. - Spinlocks are better for **short critical sections** where
the overhead of blocking/context switching would exceed the time spent
spinning. - On a **uniprocessor**, spinning is almost always wasteful
because the lock holder can't make progress while another thread spins.

### Spinlock with Yield

Instead of pure spinning, the thread calls `yield()` to give up the CPU:

```c
void acquire(lock_t *lock) {
    while (xchg(&lock->flag, 1) == 1)
        yield();  // give up CPU
}
```

**Cost analysis (Sample 1, Q20):** With yield on a uniprocessor, 10
threads trying to access a data structure. Each non-lock-holding thread
will yield, causing a context switch. Context switch = 10 microseconds.
Each waiting thread yields, gets scheduled, tries the lock, yields again
= each attempt costs \~10 microseconds. With 9 waiting threads each
potentially being scheduled once before the lock holder runs again,
that's roughly O(10 threads \* 10 microseconds).

### Blocking Locks (Mutex)

Instead of spinning, put the thread to sleep until the lock is free.

```c
void acquire(lock_t *lock) {
    while (xchg(&lock->flag, 1) == 1)
        park();  // put thread to sleep
}
```

**Cost analysis (Sample 1, Q21):** With blocking locks, waiting threads
are put to sleep. They don't consume CPU while waiting. The cost is just
the context switches to put them to sleep and wake them up. With 10
threads, that's roughly O(10 threads \* 10 microseconds) for the context
switches.

**When to use blocking locks vs. spinlocks:** - **Short critical
sections, multiprocessor:** Spinlocks (avoid context switch overhead) -
**Long critical sections:** Blocking locks (don't waste CPU spinning) -
**System low on memory / swapping a lot (Sample 1, Q23):** Blocking
locks. If the system is swapping, a spinning thread could be swapped
out, meaning it wastes time spinning AND causes page faults. Blocking is
better.

### Ticket Locks

```c
typedef struct {
    int ticket;  // next ticket to hand out
    int turn;    // current ticket being served
} lock_t;

void acquire(lock_t *lock) {
    int myturn = FAA(&lock->ticket);  // fetch-and-add, atomic
    while (lock->turn != myturn)
        ; // spin
}

void release(lock_t *lock) {
    lock->turn++;
}
```

Key facts: - **Provides bounded waiting** because threads are served in
FIFO order. - **Provides fairness** (every thread gets served in
order). - Does **NOT** provide lower latency than spinlocks. If
anything, it's slightly higher latency due to the atomic FAA. The
advantage is fairness, not speed. - "Ticket locks provide lower latency
than spin locks" = **False**.

### Two-Phase Waiting

First spin for a short while, then block if the lock isn't available.
Best of both worlds.

- Does **NOT** need to know how long the lock will be held. It just
  guesses: spin first, then give up and block.
- "Two-phase waiting does not need to know the amount of time the lock
  will be held" = **True**.

### Disabling Interrupts

```c
void acquire() { cli(); }  // disable interrupts
void release() { sti(); }  // enable interrupts
```

- Works on uniprocessors for kernel code.
- **Bad idea for user programs:** a user could hog the CPU forever.
- **Bad idea on multiprocessors:** only disables interrupts on one
  core; other cores can still access shared data.
- "Disabling interrupts is a good way to implement locks in some
  situations" = **True** (specifically: kernel code on uniprocessors
  for short critical sections).
- "Turning off interrupts can make one process keep control of CPU for
  arbitrary time" = **True**.

### Building Locks with XCHG and CompareAndSwap

**XCHG (Exchange):**

```c
int xchg(int *addr, int newval) {
    int old = *addr;
    *addr = newval;
    return old;
}
```

This is atomic in hardware. The idea: try to swap your value into the
lock. If you get back the "free" value, you got the lock.

If lock held = 1, free = 0 (the standard case): - `init: flag = 0` -
`acquire: while(xchg(&flag, 1) == 1)` (keep trying to set it to 1; if
you get back 1, someone else has it) - `release: flag = 0`

If lock held = 0, free = 1 (the REVERSED case from Sample 2, Q51-52): -
`init: flag = 1` and `release: flag = 1` -
`acquire: while(xchg(&flag, 0) == 0)` (try to set it to 0/held; if you
get back 0, someone already holds it)

**CompareAndSwap (CAS):**

```c
int CompareAndSwap(int *addr, int expected, int new) {
    int actual = *addr;
    if (actual == expected)
        *addr = new;
    return actual;
}
```

If lock held = 1, free = 0: - `init: flag = 0`, `release: flag = 0` -
`acquire: while(CAS(&flag, 0, 1) == 1)` means "if flag is 0 (free), set
to 1 (held). If I got back 1, someone else has it, keep trying."

**Wait-free add with CAS (Sample 3, Q79-82):**

```c
void add(int *val, int amt) {
    do {
        int old = *val;
    } while (CmpAndSwap(val, old, old + amt) == 0);
}
```

Q1=val (address), Q2=old (expected), Q3=old+amt (new value), Q4=0
(failure case, keep trying).

- "A context switch can occur in the middle of XCHG" = **False**. The
  whole point of atomic instructions is they complete without
  interruption at the hardware level.

### The OS and Locks

- "The OS scheduler is aware of lock()/unlock() calls" = **False**.
  The scheduler preempts threads regardless of lock state. It has no
  idea a thread holds a lock.
- "Locks prevent the OS from performing a context switch during
  critical sections" = **False**. A timer interrupt can happen
  anytime.
- "If two threads are waiting on a blocking lock, the one with higher
  priority will always acquire it first" = **False**. Lock
  implementations don't typically respect scheduling priority.

---

## 5. LOCK PLACEMENT (Concurrent Data Structures)

### The Hash Table Question (Sample 1, Q22)

```c
#define BUCKETS (101)
typedef struct __list_t {
    pthread_mutex_lock lock;  // A: per-bucket lock
    Node_t *head;
}
typedef struct __hash_t {
    list_t lists[BUCKETS];
    pthread_mutex_lock lock;  // B: global hash lock
} hash_t;

int Hash_Insert(hash_t *H, int key) {
    int bucket, ret;
    pthread_mutex_lock(&H->lock);              // C: acquire global
    bucket = key % BUCKETS;
    pthread_mutex_lock(H->lists[bucket].lock); // D: acquire per-bucket
    ret = List_Insert(&H->lists[bucket], key);
    pthread_mutex_unlock(&H->lock);            // unlock global and/or per-bucket
    return(ret);
}
```

**Best placement: A & D** (per-bucket lock defined in the list struct,
acquired in the insert function for just that bucket). This gives
maximum concurrency because threads operating on different buckets don't
contend.

**Why not B & C (global lock)?** It works for correctness but kills
concurrency since only one insert can happen at a time across the entire
hash table.

---

## 6. CONDITION VARIABLES

### The API

```c
pthread_cond_wait(cond_t *cv, mutex_t *m);  // atomically: release lock, sleep, reacquire lock on wake
pthread_cond_signal(cond_t *cv);            // wake up ONE waiting thread
pthread_cond_broadcast(cond_t *cv);         // wake up ALL waiting threads
```

### Critical Facts (tested repeatedly)

1. **After returning from cond_wait(), the thread holds the mutex.** =
   **True**. cond_wait atomically releases the lock when sleeping and
   reacquires it before returning.
2. **After returning from cond_wait(), the condition it was waiting for
is true.** = **False!** Another thread might have changed the state
   between the signal and this thread reacquiring the lock. This is why
   you MUST use a **while loop**, not an if statement.
3. **cond_signal was called previously; does cond_wait return
immediately?** = **False/C**. Signals are NOT saved. If no thread is
   currently waiting when signal is called, the signal is lost. This is
   different from semaphores!
4. **cond_broadcast wakes only one thread** = **False**. Broadcast
   wakes ALL waiting threads.
5. **It's always correct to use broadcast instead of signal** =
   **True** (with while loops). It may be slower (wakes threads
   unnecessarily) but won't cause bugs if you're rechecking the
   condition in a while loop.
6. **You can modify state associated with CVs without holding the
mutex** = **False**. Always hold the lock when checking or modifying
   the shared state.

### The While vs. If Pattern

```c
// CORRECT:
while (condition_not_met)
    cond_wait(&cv, &lock);

// WRONG:
if (condition_not_met)
    cond_wait(&cv, &lock);
```

Why while? Because of **spurious wakeups** and **stolen wakeups**.
Thread A might be woken by a signal, but by the time it reacquires the
lock, Thread B has already consumed the resource.

### Traffic Light Question (Sample 1, Q34)

"Waiting cars can only go when light is green."

- `if (light == green) cond_wait(...)` = WRONG. Waits when it should
  GO.
- `while (light == red) cond_wait(...)` = WRONG. Doesn't account for
  yellow.
- `while (light != green) cond_wait(...)` = **CORRECT**. Waits as long
  as it's not green. Uses while loop.
- `if (light == red) cond_wait(...)` = WRONG. Uses if (not while), and
  ignores yellow.

### thread_join/thread_exit Implementation

This appears in ALL THREE sample exams. Here's the correct version:

```c
int done = 0;
mutex_t m;
cond_t c;

void thread_join() {
    mutex_lock(&m);           // p1
    while (done == 0)         // p2
        cond_wait(&c, &m);   // p3
    mutex_unlock(&m);         // p4
}

void thread_exit() {
    mutex_lock(&m);           // c1
    done = 1;                 // c2
    cond_signal(&c);          // c3
    mutex_unlock(&m);         // c4
}
```

**This works in ALL cases.** The while loop handles both orderings: - If
parent calls join first: it sees done==0, waits. Child eventually sets
done=1, signals. Parent wakes, rechecks while, done==1, exits. - If
child calls exit first: sets done=1, signals (no one waiting, signal
lost). Parent calls join, sees done==1, skips the while loop, exits
immediately.

**Sample 3's variant (c2 and c3 swapped):**

```c
void thread_exit() {
    mutex_lock(&m);           // c1
    cond_signal(&c);          // c2 (signal BEFORE setting done)
    done = 1;                 // c3
    mutex_unlock(&m);         // c4
}
```

Does this still work? **Yes, in all cases.** Because: - If parent is
waiting in cond_wait: the signal wakes it, but it can't proceed until it
reacquires the lock. The child still holds the lock, so the child
finishes c3 (done=1) and c4 (unlock) first. Then parent reacquires lock,
checks while(done==0), sees done=1, proceeds. - If child runs first
entirely: sets done=1 before parent calls join. Parent sees done!=0,
skips wait.

### Schedule Tracing (Sample 2, Q64-70)

For the buggy version (using `if` instead of `while`, signal before
done=1):

```c
void thread_join() {
    mutex_lock(&m);       // p1
    if (done == 0)        // p2
        cond_wait(&c, &m);// p3
    mutex_unlock(&m);     // p4
}

void thread_exit() {
    mutex_lock(&m);       // c1
    done = 1;             // c2
    mutex_unlock(&m);     // c3
    cond_signal(&c);      // c4
}
```

**HUGE BUG HERE:** The signal is OUTSIDE the lock (c4 happens after c3
unlocks). Also uses `if` not `while`.

For schedule tracing: - Count letters to know how many lines each thread
executes - P executes p1, p2, p3, p4 in order. C executes c1, c2, c3, c4
in order. - Remember: mutex_lock blocks if lock is held. cond_wait
blocks if the wait hasn't been satisfied. - Track: who holds the lock,
what is `done`, who is sleeping

Example: **CCCPPPC** = C runs c1,c2,c3 then P runs p1,p2,p3 then C runs
c4. - C: c1 (lock), c2 (done=1), c3 (unlock) - P: p1 (lock), p2 (if
done==0? No, done=1), skips to p4... wait, only 3 P's so: p1(lock),
p2(done=1, skip if), p4(unlock? but that's only p1,p2,p4 = 3 lines) -
Actually p1 is mutex_lock (1 tick), p2 is the if check (1 tick), since
done=1 it skips p3, p4 is mutex_unlock (1 tick). That's 3 P ticks. -
Then C: c4 (cond_signal, but nobody is waiting, harmless) - Result:
Parent returned from thread_join, child completed thread_exit. **No
problem, both ran to completion.** Answer: (b).

---

## 8. THE MEMORY ALLOCATOR / cond_signal vs cond_broadcast PROBLEM

### Sample 1, Q42-43

```c
int bytesLeft = 0;  // starts at 0

void* allocate(int size) {
    lock(&m);
    while (bytesLeft < size)
        cond_wait(&c, &m);
    void *ptr = ...;
    bytesLeft -= size;
    unlock(&m);
    return ptr;
}

void free(void *ptr, int size) {
    lock(&m);
    bytesLeft += size;
    cond_signal(&c);  // only wakes ONE thread!
    unlock(&m);
}
```

**Q42:** T1: allocate(100), T2: allocate(20), T3: free(30). - T1 tries
to allocate 100, bytesLeft=0 \< 100, T1 sleeps. - T2 tries to allocate
20, bytesLeft=0 \< 20, T2 sleeps. - T3 frees 30, bytesLeft=30. Signals,
waking ONE thread. - If T1 wakes: bytesLeft=30 \< 100, goes back to
sleep. T2 still sleeping. - If T2 wakes: bytesLeft=30 \>= 20, allocates.
bytesLeft=10. T1 still sleeping. - **Could lead to T1 blocked, or both
blocked** (if T1 is woken, rechecks, goes back to sleep, and no more
signals come). Answer: **(a) Could lead to both blocked**.

This is the classic problem with `cond_signal` vs `cond_broadcast`.
Using broadcast would fix it.

---

## 9. CONCURRENCY BUGS

### Three Types

**1. Atomicity Violation:** Operations that should be atomic aren't.
Classic: check-then-act without a lock.

```c
// Thread 1:        // Thread 2:
if (ptr != NULL)    ptr = NULL;
    ptr->field++;
```

Thread 1 checks ptr, it's not NULL. Thread 2 sets ptr to NULL. Thread 1
dereferences NULL. Crash.

**2. Ordering Violation:** Operations happen in an unexpected order. One
thread assumes another has already done something.

```c
// Sample 1, Q36:
// Main thread creates child, doesn't wait
// Main might read data before child initializes it
```

The code in Q36 has the main thread create a child that sets
`data->started = 1`, but main doesn't join or wait. Main could read
`started` before the child sets it. This is an **ordering** bug.

**3. Deadlock:** Two or more threads are stuck waiting for each other.
See next section.

### How to Identify the Bug Type

**Sample 1, Q37:**

```c
struct global_data { int started; char *buffer; } data;

void *child_thread(void *arg) {
    if (!data.started) {
        data.buffer = malloc(100);
        data.started = 1;
    }
    // do something
}
// Two threads run child_thread
```

Two threads both check `data.started`. Both see it's 0 (not started).
Both enter the if block. Both malloc and set started=1. Double malloc,
memory leak. This is an **atomicity** bug since the check-then-act is
not atomic.

**Sample 1, Q38:**

```c
void *child_thread(void *arg) {
    if (!data.started) {            // check WITHOUT lock
        pthread_mutex_lock(&data.lock);
        if (!data.started) {        // double-check WITH lock
            data.buffer = malloc(100);
            data.started = 1;
        }
        pthread_mutex_unlock(&data.lock);
    }
    // do something
}
```

The first check of `data.started` is outside the lock! Thread 1 checks
started (it's 0), then gets preempted. Thread 2 checks started (0),
acquires lock, mallocs, sets started=1, unlocks. Thread 1 resumes,
acquires lock, checks again (now it's 1), skips the malloc. But the
FIRST check was still a race. Actually, in this case the double-checked
locking pattern works correctly here for this specific case since the
second check inside the lock catches it. But reading `data.started`
without the lock is technically a race. The answer is **(a) Atomicity**
because reading `started` outside the lock is a non-atomic
check-then-act.

Wait, actually looking more carefully: the outer if prevents entering
the lock section at all once started=1. The inner if handles the race.
This is actually the double-checked locking pattern and in C with
volatile it can work. But without volatile/memory barriers, the compiler
could optimize the read. The exam likely expects **(d) none** since the
double-check inside the lock makes it correct... but it depends on the
memory model. This is tricky. For the exam, focus on whether shared
state is accessed without a lock.

---

## 10. DEADLOCK

### Four Necessary Conditions (ALL must hold)

1. **Mutual Exclusion:** Resources can only be held by one thread at a
   time.
2. **Hold and Wait:** A thread holds one resource while waiting for
   another.
3. **No Preemption:** Resources can't be forcibly taken away.
4. **Circular Wait:** Thread A waits for B, B waits for A (or a longer
   chain).

"What is NOT a necessary condition for deadlock?" = **Unfairness**
(Sample 1, Q55). Also **Lock Ordering** (Sample 2, Q46) is NOT a
condition; it's a prevention strategy.

### Breaking Deadlock

Eliminate ANY ONE of the four conditions: - **Break mutual exclusion:**
Use atomic instructions instead of locks. - **Break hold-and-wait:**
Acquire all locks at once atomically. - **Break no preemption:** Use
trylock; if you can't get a lock, release what you hold. - **Break
circular wait:** Always acquire locks in a global ordering.

### Lock Ordering

If Thread 1 acquires lock A then B, and Thread 2 acquires lock B then A,
deadlock is possible (but not guaranteed, it depends on timing). To
prevent: always acquire in the same order (e.g., always A before B).

"Deadlock WILL ALWAYS occur if A acquires X,Y and B acquires Y,X" =
**False**. It CAN occur but won't necessarily. If A finishes before B
starts, no deadlock.

### Trylock and Livelock (Sample 1, Q39)

```c
// Thread 1:              // Thread 2:
top:                      top2:
lock(A);                  lock(B);
if (trylock(B) == -1) {   if (trylock(A) == -1) {
    unlock(A);                unlock(B);  // NOTE: Bug! Should be unlock(B)
    goto top;                 goto top2;
}                          }
```

Wait, looking at the actual code in the exam: Thread 2 does `unlock(A)`
in its failure branch, but Thread 2 doesn't hold A! This is a bug
(unlocking a lock you don't hold). But ignoring that typo and focusing
on the concept:

If both threads keep acquiring one lock, failing to get the second,
releasing, and retrying, they can loop forever without making progress.
This is **livelock**. Answer: **(b) Now suffers from livelock**.

### The Global Lock Pattern (Sample 1, Q40-41)

```c
void vector_add(vector_t *v_dst, vector_t *v_src) {
    Pthread_mutex_lock(&global);
    Pthread_mutex_lock(&v_dst->lock);
    Pthread_mutex_lock(&v_src->lock);
    ????????   // What goes here?
    // ... do the addition ...
    XXXXX
    YYYYY
}
```

The global lock is only needed to prevent deadlock during lock
acquisition. Once you have both v_dst and v_src locks, you can release
the global lock. So `????????` = `Pthread_mutex_unlock(&global)`.

At the end (XXXXX and YYYYY), you release v_dst and v_src locks (in
either order). So YYYYY is whichever lock you didn't unlock in XXXXX.
The global lock was already released.

### function1/function2 Deadlock Question (Sample 1, Q44)

```c
function1() {
    mutex_unlock(&lockA);  // releases A!
    // do some work
    mutex_lock(&lockA);    // reacquires A
}

function2() {
    mutex_lock(&lockA);
    mutex_lock(&lockB);
    function1();           // unlocks A inside!
    mutex_unlock(&lockB);
    mutex_unlock(&lockA);
}
```

Two threads both call function2. Thread 1 acquires A, then B, then calls
function1 which unlocks A. Now Thread 2 can acquire A! Thread 2 acquires
A, tries to acquire B (held by Thread 1). Meanwhile Thread 1 (inside
function1) finishes work and tries to reacquire A (held by Thread 2).
**Deadlock!** Answer: **(a) can deadlock**.

---

## 11. PRODUCER-CONSUMER STATE ANALYSIS (Sample 2, Q56-63)

These questions give you specific thread positions and ask if that state
is possible. Strategy:

**Track invariants:** -
`numempty + numfull + items_in_progress = buffer_size (4)` - Only one
thread can hold the mutex at a time - Threads at p4 (get_empty) or c4
(get_full) hold the mutex - Threads at p6 (fill_buffer) or c6
(use_buffer) do NOT hold the mutex (released at p5/c5) - Threads at p3
(sleeping) or c3 (sleeping) do NOT hold the mutex (released by
cond_wait)

**Check mutex consistency:** At most one thread can be at a position
where it holds the mutex (p2,p4,p5,p7,p8,p9,p10 or
c2,c4,c5,c7,c8,c9,c10). Threads at p1/c1 are trying to acquire it.
Threads at p6/c6 don't hold it. Threads at p3/c3 sleeping don't hold it.

**Check buffer consistency:** Count how many items are being filled
(threads at p6), how many are full, how many are being consumed (threads
at c6), how many are empty. Make sure it's consistent with 4 total
slots.

Example: **Q56:** Pa:p6, Pb:p6, Ca:c4, Cb:c4. Both consumers at c4 means
both hold the mutex... but only one can hold it at a time. **Not
possible.**

Example: **Q57:** Pa:p1, Pb:p1, Ca:c1, Cb:c1. All four threads trying to
acquire the mutex. Possible if no one currently holds it (everyone just
started a new iteration). **Possible.**

---

## 12. READER-WRITER LOCKS

### The Concept

- Multiple readers can hold the lock simultaneously.
- Only one writer can hold the lock, and no readers can hold it while
  a writer does.
- "Multiple readers OR a single writer" = **True**.
- "Multiple readers AND a single writer" = **False**. They're mutually
  exclusive.

### Priority

The implementation from Sample 3 gives **priority to writers**. When a
reader releases and there are waiting writers, a writer gets the lock
next (even if readers are also waiting). This can starve readers.

"You can have multiple readers or multiple writers" = **False**. Only
one writer at a time. "Readers have higher priority than writers" /
"Writers have higher priority than readers" = Depends on implementation.
The default answer for "in general" is **(d) None of the above** since
it depends on the implementation.

---

## 13. SCALING AND PERFORMANCE

### Strong Scaling

Fix the problem size, add more cores. If a task takes 16 seconds on 4
cores, with perfect strong scaling on 8 cores it takes **8 seconds**
(linear speedup).

### Amdahl's Law

Speedup is limited by the serial portion of the program. If 10% of the
code is serial, max speedup = 10x regardless of how many cores you add.

### Key True/False

- "Doubling threads always halves runtime" = **False**. Amdahl's Law,
  synchronization overhead, and contention prevent perfect scaling.
- "Concurrency always produces deterministic results" = **False**.
  That's the whole problem with concurrency.
- "Incorrect behavior cannot lead to catastrophic results" =
  **False**. Therac-25 killed people.

---

# 15. RACE DETECTION (Lecture 15 Add-on)

## What is a Data Race (Exam Definition)

Formal definition: - Two memory accesses to the same location - At least
one is a write - The accesses are not ordered by synchronization

Key exam insight: - In C/C++ → data race = undefined behavior = always a
bug

---

## Ways to Find Concurrency Bugs

1. Static analysis
   - Pros: no execution needed
   - Cons: false positives common
2. Run many times
   - Pros: simple
   - Cons: may never trigger bug
3. Exhaustive / systematic testing
   - Controlled scheduler (e.g., CHESS)
   - Deterministic + complete
   - Very slow
4. Race detection tools (ThreadSanitizer)
   - Practical balance

---

## ThreadSanitizer (IMPORTANT)

### How it works

Instruments: - Memory reads/writes - Lock operations - Synchronization
events

---

## Lockset-Based Detection

Idea: - For each memory location → track which locks protect it - If two
threads access the same location: - AND no common lock → potential race

Exam trap: - Lockset alone → false positives possible

---

## Happens-Before (VERY IMPORTANT)

### Definition

Event ordering: - If E1 happens-before E2 (E1 ≺ E2) → safe ordering - If
neither: - E1 ⊀ E2 AND E2 ⊀ E1 → simultaneous (race possible)

### Sources of happens-before edges

- Program order (same thread)
- Lock/unlock
- Signal/wait (condition variables)
- Thread creation/join

### Key rule

Race if: - Same memory location - At least one write - No happens-before
ordering - No common lock

---

## Happens-Before Example Insight

- Signal → Wait creates ordering across threads
- Happens-before is transitive

Exam takeaway: - If A → B and B → C, then A → C

---

## Race Detection = Combined Model

A race is reported if: 1. Same memory location 2. At least one write 3.
No happens-before relation 4. No common lock

---

## False Positives & False Negatives

### False Positives

- Code is actually safe but detector flags it
- Example: flag-based synchronization without locks

### False Negatives

- Bug exists but not observed
- Happens because:
  - Not all schedules explored

### Performance cost

- \~3–5× slower execution

---

## Subtle Race Pattern (VERY TESTABLE)

Example:

int shared_data = 0; int ready = 0;

void \*writer() { shared_data = 42; ready = 1; }

void \*reader() { while (!ready) {} printf("%d\n", shared_data); }

Looks correct, but: - No synchronization - Compiler/CPU reordering
possible

→ Data race

---

## Linearizability (NEW CONCEPT)

### Definition

Each operation appears to occur: - At one instant between invocation and
return

### What it means for exams

- Even concurrent operations should look sequentially consistent
- If results can’t be explained by any valid ordering → not
  linearizable

---

## Relationship to Your Study Guide

This lecture strengthens: - Race conditions - Atomicity vs ordering
bugs - Condition variables - Locks

New addition: - Formal detection frameworks (lockset + happens-before)

---

# 16. GENERATED SAMPLE QUESTIONS

## Q1. Data Race Identification

int x = 0;

Thread 1: x = x + 1;

Thread 2: printf("%d\n", x);

Answer: Data race

---

## Q2. Happens-Before

Thread 1: lock(L); x = 5; unlock(L);

Thread 2: lock(L); print(x); unlock(L);

Answer: No race

---

## Q3. Signal/Wait Ordering

Thread 1: x = 10; cond_signal(&c);

Thread 2: cond_wait(&c, &m); print(x);

Answer: Safe

---

## Q4. Lockset Trick Question

Two threads access x: - Both hold lock A - One also holds lock B

Answer: No race

---

## Q5. Happens-Before vs Simultaneous

Which indicates a possible race?

Answer: E1 ⊀ E2 AND E2 ⊀ E1

---

## Q6. False Positive Scenario

Thread 1: flag = 1;

Thread 2: if (flag) do_work();

Answer: Might be correct but flagged

---

## Q7. ThreadSanitizer Behavior

Answer: Reads, writes, and synchronization

---

## Q8. Linearizability

Put(1, 10) Get(1) → returns 0

Answer: Not linearizable

---

## Q9. Race vs Atomicity

if (x \> 0) x--;

Answer: Atomicity violation

---

## Q10. Performance Tradeoff

Answer: 3–5× slower

---

# 17. HIGH-PROBABILITY EXAM ADDITIONS

Expect questions on: - Formal definition of data race - Happens-before
reasoning - Lockset vs happens-before differences - False positives vs
false negatives - ThreadSanitizer behavior - Linearizability basics

Most likely tricky question type: “Is this a race even though it looks
ordered?”

→ If no explicit synchronization → YES, race

---

## EXAM-TAKING STRATEGIES

### For True/False

Look for absolute words: "always", "never", "cannot", "will". These are
usually **False** because there are almost always exceptions in OS.

### For Code Tracing

1. Draw a table with columns for each thread's registers, shared
   memory, and lock state.
2. Execute one instruction at a time following the given schedule.
3. Remember: registers are per-thread, memory is shared.

### For "Which bug?" Questions

1. Is shared data accessed without proper synchronization? -\>
   **Atomicity**
2. Does one thread assume another has already done something? -\>
   **Ordering**
3. Do threads wait for each other in a cycle? -\> **Deadlock**
4. Do threads keep retrying without making progress? -\> **Livelock**
