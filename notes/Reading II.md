# **xv6: a simple, Unix-like teaching operating system**

# <u>Chapter 1 Operating system interfaces</u>

## 1.1 processes and memory

```c
int fork()
int exec(char *file, char *argv[])
```

## 1.2 I/O and file descriptiors

```c
int read(int fd, char *buf, int n)
int write(int fd, char *buf, int n)
```

- the **file descriptor** interface abstracts away the differences between files, pipes, and devices, making them all look like <u>streams of bytes</u>.
	- by convention, a process reads from file descriptor 0 (standard input), writes output to file descriptor 1 (standard output), and writes error messages to file descriptor 2 (standard error)
	- The `close` system call releases a file descriptor
	- although `fork` copies the file descriptor table, each underlying file offset is shared between parent and child.

### I/O redirection

```c
char *argv[2];
    argv[0] = "cat";
    argv[1] = 0;
    if(fork() == 0) {
      close(0);
      open("input.txt", O_RDONLY);
      exec("cat", argv);
}
```

## 1.3 pipes

- A ***pipe*** is a small kernel buffer exposed to processes as <u>a pair of file descriptors, one for reading and one for writing</u>.

	```c
	int p[2];
	char *argv[2];
	argv[0] = "wc";
	argv[1] = 0;
	pipe(p);
	if(fork() == 0) {
	  close(0);
	  dup(p[0]);
	  close(p[0]);
	  close(p[1]);
	  exec("/bin/wc", argv);
	} else {
	  close(p[0]);
	  write(p[1], "hello world\n", 12);
	  close(p[1]);
	}
	```

- Pipes may seem no more powerful than temporary files: the pipeline

	```shell
	echo hello world | wc
	```

	could be implemented without pipes as

	```shell
	echo hello world >/tmp/xyz; wc </tmp/xyz
	```

	- four advantages
		1. pipes automatically clean themselves up
		2. pipes can pass arbitrarily long streams of data
		3. pipes allow for parallel execution of pipeline stages
		4. pipes’ blocking reads and writes are more efficient than the non-blocking semantics of files

## 1.4 file system

- The directories form a tree, starting at a special directory called the *root*.

- aths that don’t begin with `/` are evaluated relative to the calling process’s *current directory*, which can be changed with the `chdir` system call

	```c
	chdir("/a");
	chdir("b");
	open("c", O_RDONLY);
	open("/a/b/c", O_RDONLY);
	```

## 1.5 Real world

- Unix’s combination of “standard” **<u>file descriptors</u>**, **<u>pipes</u>**, and convenient **<u>shell syntax</u>** for operations on them was a major advance in writing **<u>general-purpose reusable programs</u>**.
	- Unix unified access to multiple types of resources (files, directories, and devices) with <u>a single set of file-name and file-descriptor interfaces</u>.

## *Thinking*

这篇阅读主要介绍了xv6支持的系统调用，并展示这些调用如何组合满足各种使用功能的。

- 有一个统一的文件描述符系统，从而实现读写，并通过对文件描述符的操作实现I/O重定向



# <u>Chapter 2 Operating system organization</u>

## *Thinking*

- 概略的介绍了 kernel 的组织方式，各个问题会在之后的文章中详细介绍



# <u>Chapter 3 Page tables</u>

- Isolation
- Indirection: allow operating systems to perform many tricks

## *Thinking*

- 详细介绍了page table的组织和运行逻辑，以及xv6中的代码实现

# <u>Chapter 4 Traps and system calls</u>

 

- the CPU to set aside ordinary execution of instructions and force a transfer of control to special code

	- a system call

	- an exception

	- a device interrupt

### trap

- a trap forces a transfer of control into the kernel; 
- the kernel saves registers and other state so that execution can be resumed; 
- the kernel executes appropriate handler code (e.g., a system call imple- mentation or device driver); 
- the kernel restores the saved state and returns from the trap; 
- and the original code resumes where it left off.

### trap handling

- four stages
	1. hardware actions taken by the RISC-V CPU, 
	2. an assembly “vector” that prepares the way for kernel C code, 
	3. a C trap handler that decides what to do with the trap, 
	4. and the system call or device-driver service routine
- three cases
	- traps from user space, 
	- traps from kernel space, 
	- and timer interrupts

## 4.1 RISC-V trap machinery

### control registers

- Each RISC-V CPU has a set of control registers that the kernel writes to tell the CPU how to handle traps, and that the kernel can read to find out about a trap that has occured

	- `stvec`: The kernel writes the address of its trap handler here; the RISC-V jumps here to handle a trap.

	- `sepc`: When a trap occurs, RISC-V saves the program counter here
	- `scause`: The RISC-V puts a number here that describes the reason for the trap.
	- `sscratch`: The kernel places a value here that comes in handy at the very start of a trap handler
	- `sstatus`: The SIE bit in sstatus controls whether device interrupts are enabled

### steps

- the RISC-V <u>hardware</u> does the following for all trap types (other than timer interrupts):
	1. If the trap is a device interrupt, and the `sstatus` SIE bit is clear, don’t do any of the following.
	2. Disable interrupts by clearing SIE
	3. Copy the `pc` to `sepc`
	4. Save the current mode (user or supervisor) in the SPP bit in `sstatus`.
	5. Set `scause` to reflect the trap’s cause.
	6. Set the mode to supervisor.
	7. Copy `stvec` to the `pc`.
	8. Start executing at the new `pc`.

- hardware doesn't do (to provide flexibility to software)
	- doesn’t switch to the kernel page table
	- doesn’t switch to a stack in the kernel
	- doesn’t save any registers other than the `pc`

## 4.6 Page-fault exceptions

- Xv6’s response to exceptions is quite boring: 
	- if an exception happens in user space, the kernel kills the faulting process. 
	- If an exception happens in the kernel, the kernel panics.

- RISC-V has three different kinds of page fault: 
	- load page faults (when a load instruction cannot translate its virtual address), 
	- store page faults (when a store instruction cannot translate its virtual address), and 
	- instruction page faults (when the address for an instruction doesn’t translate). 
	- The value in the `scause` register indicates the type of the page fault 
	- and the `stval` register contains the address that couldn’t be translated.



# <u>Chapter 5 Interrupts and device drivers</u>



# <u>Chapter 6 Locking</u>

## Concurrency Control Techniques

- The word *concurrency* refers to situations in which multiple instruction streams are interleaved, due to 
	- multiprocessor parallelism, 
	- thread switching, or 
	- interrupts.

- lock: 
	- ensures mutual exclusion
	- critical section

## Race Conditions

- A race is often a sign of a bug, either 
	- a lost update (if the accesses are writes) or 
	- a read of an incompletely-updated data structure.
- difficult to reproduce and debug
	- e.g. adding print statements while debugging might change the timing of the execution enough to make the race disappear

## Rules to use locks

- when to use
  - any time a variable can be written by one CPU at the same time that another CPU can read or write it, a lock should be used to keep the two operations from overlapping.
  - if an invariant involves multiple memory locations, typically all of them need to be protected by a single lock to ensure the invariant is maintained.
- Locks in xv6
	- ​	<img src="image.assets/Screen Shot 2021-11-24 at 09.39.00.png" alt="Screen Shot 2021-11-24 at 09.39.00" style="zoom:33%;" />


## Deadlock

- If a code path through the kernel must hold several locks at the same time, it is important that all code paths acquire those locks in the same order. If they don’t, there is a risk of *deadlock*, e.g.
	- two code paths in xv6 need locks A and B
	- code path 1 acquires locks in the order A then B
	- code path 2 acquires locks in the order B then A
- The file-system code contains xv6’s longest lock chains. For example, creating a file requires simultaneously holding 
	- a lock on the directory, 
	- a lock on the new file’s inode, 
	- a lock on a disk block buffer, the disk driver’s `vdisk_lock`, and 
	- the calling process’s `p->lock`. 
	- To avoid deadlock, file-system code always acquires locks in the order mentioned in the previous sentence.

## Locks and interrupt handlers

- To avoid this situation, if a spinlock is used by an interrupt handler, a CPU must never hold that lock with interrupts enabled
	- Xv6 is more conservative: when a CPU acquires any lock, xv6 always disables interrupts on that CPU



## Instruction and memory ordering

- Many compilers and CPUs, however, execute code out of order to achieve higher performance.
	- If an instruction takes many cycles to complete, a CPU may issue the instruction early so that it can overlap with other instructions and avoid CPU stalls



## Sleep locks

- Holding a spinlock that long would lead to waste if another process wanted to acquire it, since the acquiring process would waste CPU for a long time while spinning.
- Another drawback of spinlocks is that a process cannot yield the CPU while retaining a spinlock; we’d like to do this so that other processes can use the CPU while the process with the lock waits for the disk.
- Thus we’d like a type of lock that yields the CPU while waiting to acquire, and allows yields (and interrupts) while the lock is held.
- Because sleep-locks leave interrupts enabled, they cannot be used in interrupt handlers. Be- cause acquiresleep may yield the CPU, sleep-locks cannot be used inside spinlock critical sections

## *Thinking*

- 锁：并发控制技术之一 —— 并发过程中，保护共享数据
- 细节：
	- 死锁：固定多个锁的获取顺序
	- 锁和中断处理程序：若interrupt handler和被打断的进程调用同一个锁则永远无法返回进程，因此如果interrupt handler使用了某个锁，则cpu在持有该锁时要禁用interrupt
	- 汇编过程中的重新排序：在临界区段（critical section）前后设置屏障
	- 持有锁的进程在sleep时如何让其他进程运行：sleep lock

# <u>Chapter 7 Scheduling</u>

- Any operating system is likely to run with more processes than the computer has CPUs, 
	- so a plan is needed to time-share the CPUs among the processes
- A common approach is to provide each process with the illusion that it has its own virtual CPU by *multiplexing* the processes onto the hardware CPUs

## Multiplexing

- Conditions
	1. First, xv6’s sleep and wakeup mechanism switches when a process waits for device or pipe I/O to complete, or waits for a child to exit, or waits in the sleep system call
	2. Second, xv6 periodically forces a switch to cope with processes that compute for long periods without sleeping
- Challenges
	1. First, how to switch from one process to another?
	2. Second, how to force switches in a way that is transparent to user processes?
		- Xv6 uses the standard technique of driving context switches with timer interrupts
	3. Third, many CPUs may be switching among processes concurrently, and a locking plan is necessary to avoid races
	4. Fourth, a process’s memory and other resources must be freed when the process exits, but it cannot do all of this itself because (for example) it can’t free its own kernel stack while still using it
	5. Fifth, each core of a multi-core machine must remember which process it is executing so that system calls affect the correct process’s kernel state
	6. Finally, sleep and wakeup allow a process to give up the CPU and sleep waiting for an event, and allows another process to wake the first process up. Care is needed to avoid races that result in the loss of wakeup notifications

<img src="image.assets/Screen Shot 2021-11-26 at 11.42.30.png" alt="Screen Shot 2021-11-26 at 11.42.30" style="zoom:33%;" />



## Scheduling

- The scheduler exists in the form of a special thread per CPU, each running the scheduler function
- A process that wants to give up the CPU must acquire its own process lock `p->lock`, release any other locks it is holding, update its own state (`p->state`), and then call `sched`
	- `yield` follows this convention, as do `sleep` and `exit`
	- `sched` double-checks those conditions and then an implication of those conditions: since a lock is held, interrupts should be disabled

### Coroutines

- A kernel thread always gives up its CPU in `sched` and always switches to the same location in the scheduler, which (almost) always switches to some kernel thread that previously called `sched`
- The procedures in which this stylized switching between two threads happens are sometimes referred to as *coroutines*; 
	- in this example, `sched` and `scheduler` are co-routines of each other.

### Lock

- xv6 often acquires `p->lock` in one thread and releases it in other
	- for example acquiring in `yield` and releasing in `scheduler`

## Sleep and wakeup

- Xv6 uses one called sleep and wakeup, which allow one process to sleep waiting for an event and another process to wake it up once the event has happened. 
- Sleep and wakeup are often called ***sequence coordination*** or ***conditional synchronization* mechanisms**.

### semaphore

- A semaphore maintains a count and provides two operations. 
	- The “V” operation (for the producer) increments the count. 
	- The “P” operation (for the consumer) waits until the count is non-zero, and then decrements it and returns.
- Avoid- ing busy waiting requires a way for the consumer to yield the CPU and resume only after V incre- ments the count.
	- `Sleep(chan)` sleeps on the arbitrary value `chan`, called the *wait channel*. `Sleep` puts the calling process to sleep, releasing the CPU for other work. 
	- `Wakeup(chan)` wakes all processes sleeping on `chan` (if any), causing their `sleep` calls to return.

#### lock

- The lock will force a concurrent `V` to wait until `P` has finished putting itself to sleep, so that the `wakeup` will find the sleeping consumer and wake it up. 
- Once the consumer is awake again `sleep` reacquires the lock before returning

## code

### sleep and wakeup

- The basic idea is to have sleep mark the current process as SLEEPING and then call sched to re- lease the CPU; wakeup looks for a process sleeping on the given wait channel and marks it as RUNNABLE
	- Callers of sleep and wakeup can use any mutually convenient number as the chan- nel. 
		- Xv6 often uses the address of a kernel data structure involved in the waiting.

- It is sometimes the case that multiple processes are sleeping on the same channel; for example, more than one process reading from a pipe.
	- For this reason `sleep` is always called inside a loop that checks the condition.

### wait, exit, and kill

- Note that `wait` often holds two locks; that it acquires its own lock before trying to acquire any child’s lock; and that thus all of xv6 must obey the same locking order (parent, then child) in order to avoid deadlock.
- kill (kernel/proc.c:611) lets one process re- quest that another terminate
	- It would be too complex for kill to directly destroy the victim process, since the victim might be executing on another CPU, perhaps in the middle of a sensitive sequence of updates to kernel data structures
	- <u>Thus `kill` does very little: it just sets the victim’s `p->killed` and, if it is sleeping, wakes it up</u>

## *Thinking*

- 调度 scheduling
	- 关键问题是如何让多个进程同时执行，并且每个进程假设拥有整个CPU
	- 细节
		- 锁：需要修改进程的状态标签和保存当前状态，这些需要是atomic的，而它们又在不同的函数中，因此涉及到在不同的函数中获得和释放锁
		- 协程 coroutine：`sched` 和 `scheduler` 总是在二者之间转换
- 协调 coordination （sleep和wakeup）
	- 关键在于状态的调整和检查，且涉及到较复杂的锁的使用
		- 如sleep被一个while循环包裹，每次检查状态，应对多个wait和kill的情况
		- 如sleep获得线程锁，然后释放一个参数传入的锁，应对lost wakeup
	- 应用：pipe，wait/exit/kill

# <u>Chapter 8 File system</u>

<img src="image.assets/Screen Shot 2021-11-29 at 17.35.56.png" alt="Screen Shot 2021-11-29 at 17.35.56" style="zoom:33%;" />

1. The disk layer reads and writes blocks on an virtio hard drive. 
2. The buffer cache layer caches disk blocks and synchronizes access to them, making sure that only one kernel process at a time can modify the data stored in any particular block. 
3. The logging layer allows higher layers to wrap updates to several blocks in a *transaction*, and ensures that the blocks are updated atomically in the face of crashes (i.e., all of them are updated or none). 
4. The inode layer provides individual files, each represented as an *inode* with a unique i-number and some blocks holding the file’s data. 
5. The directory layer implements each directory as a special kind of inode whose content is a sequence of directory entries, each of which contains a file’s name and i-number. 
6. The pathname layer provides hierarchical path names like `/usr/rtm/xv6/fs.c`, and resolves them with recursive lookup. 
7. The file descriptor layer abstracts many Unix resources (e.g., pipes, devices, files, etc.) using the file system interface, simplifying the lives of application programmers.

<img src="image.assets/Screen Shot 2021-12-07 at 14.17.22.png" alt="Screen Shot 2021-12-07 at 14.17.22" style="zoom:25%;" />

### Buffer cache layer

- two jobs
	- synchronize access to disk blocks to ensure that only one copy of a block is in memory and that only one kernel thread at a time uses that copy
	- cache popular blocks so that they don’t need to be re-read from the slow disk

### Logging layer

- An xv6 system call does not directly write the on-disk file system data structures. Instead, it places a description of all the disk writes it wishes to make in a *log* on the disk. 
  - Once the system call has logged all of its writes, 
  - it writes a special *commit* record to the disk indicating that the log contains a complete operation. 
  - At that point the system call copies the writes to the on-disk file system data structures. 
  - After those writes have completed, the system call erases the log on disk.
- Log design
  - The log resides at a known fixed location, specified in the superblock. 
  - It consists of a header block followed by a sequence of updated block copies (“logged blocks”). 
    - The header block contains an array of sector numbers, one for each of the logged blocks, and the count of log blocks. 
      - The count in the header block on disk is either zero, indicating that there is no transaction in the log, or non- zero, indicating that the log contains a complete committed transaction with the indicated number of logged blocks.
  - Each system call’s code indicates the start and end of the sequence of writes that must be atomic with respect to crashes.
    - To allow concurrent execution of file-system operations by different pro- cesses, the logging system can accumulate the writes of multiple system calls into one <u>transaction</u>. Thus a single commit may involve the writes of multiple complete system calls. 
    - To avoid splitting a system call across transactions, the logging system only commits when no file-system system calls are underway.
- Code
  - Absorption
    - It is common that, for example, the disk block containing inodes of several files is written several times within a transaction.

### Block allocator

- xv6’s block allocator maintains a free bitmap on disk, with one bit per block.
  - the exclusive use implied by `bread` and `brelse` avoids the need for explicit locking.
  - `balloc` and `bfree` must be called inside a <u>transaction</u>

### Inode layer

- The term *inode* can have one of two related meanings. 
  - It might refer to the <u>on-disk</u> data structure containing a file’s size and list of data block numbers. 
    - packed into a contiguous area of disk called the inode blocks
  - Or “inode” might refer to an <u>in-memory</u> inode, which contains a copy of the on-disk inode as well as extra information needed within the kernel.
    - The kernel stores an inode in memory only if there are C pointers referring to that inode
    - Pointers to an inode can come from file descriptors, current working directories, and transient kernel code such as exec.
- Separating acquisition of inode pointers from locking helps avoid deadlock in some situations, for example during directory lookup. Multiple processes can hold a C pointer to an inode returned by `iget`, but only one process can lock the inode at a time.
- dinode
  - ​	<img src="image.assets/Screen Shot 2021-12-02 at 18.55.10.png" alt="Screen Shot 2021-12-02 at 18.55.10" style="zoom: 20%;" />
  - 12 direct blocks and 1 indirect block


### directory layer

- A directory is implemented internally much like a file. Its inode has type `T_DIR` and its data is a sequence of directory entries.

### file descriptor layer

- most resources in Unix are represented as files, including devices such as the console, pipes, and of course, real files. 
  - The file descriptor layer is the layer that achieves this uniformity.



# <u>Virtual Memory Primitives for User Programs (1991)</u>

- What virtual-memory primitives should the operating system provide to user processes?

### Introduction

- "Traditional" purpose
  - Increase the size of the address space visible to user programs
- Other purposes for o/s
  - share pages between processes
  - make instruction-spaces read-only (and thus guarantted re-entrant)
  - make portions of memory zeroed-on-demand or copy-on-write
  - and so on





















