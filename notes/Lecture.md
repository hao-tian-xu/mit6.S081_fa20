# <u>lecture 1: O/S overview</u>

## overview

### course goal

- understand design and implementation of O/S
- hands-on experience

### O/S purpose

- abstract Hardware
- multiplex
- isolation
- sharing
- security
- performance

### O/S organization

- user applications: vi, gcc, DB, &c
- **<u>kernel services</u>**
- h/w: CPU, RAM, disk, net, &c

### O/S kernel services

- process 
- memory allocation
- file contents
- file names, directories
- access control (security)
- many others: users, IPC, network, time, terminals

### API-kernel

- system calls
- `fd = open("out", 1)`
- look like function calls but are not

### why hard/interesting

- unforgiving
- tensions
	- efficient vs abstract/general
	- powerful vs simple
	- flexible vs secure

### class structure

- lectures
	- reading/paper before lecture
- labs
	- systems programming
	- O/S primitives, e.g. thread switching
	- O/S kernel extensions to xv6, e.g. network

- xv6 (system) + risk-v (instruction set) + qemu (cpu simulator)

## *Thinking*

介绍了课程的组织形式和主要内容（kernel），然后演示了教材第一章的内容。

# <u>Lecture 2: C and gdb</u>

## Memory in C

- Static Memory
	- Global variables, accessible throughout the whole program
	- Defined with static keyword, as well as variables defined in global scope.
- Stack Memory
  - Local variables within functions. Destroyed after function exits.
- Heap Memory
  - You control creation and destruction of these variables: malloc(), free()
  - Can lead to memory leaks, use-after-free issues.



## Pointers in C

### Pointer Syntax

- `int x = 5;`
- `int *x_addr = &x;` (same as `int* x_addr = &x;`) -> ex: 0x7ffd2766a948
- `*x_addr = 6;` -> you can use the * operator to access the underlying value
- `int x_value = *x_addr;` dereferencing -> this gives 6
- `int arr1[10];` -> Arrays are secretly pointers! More on that later
- `int *arr2[20];` -> Array of pointers, making arr2 a pointer to a pointer
- `void *myPtr;`

### Pointer Arithmetic

- Suppose we have some `int *i` with value 0x100002.
	- `i++;` -> 0x100006
	- `i += 4;` -> 0x100016
- Pointers add and subtract in increments of the base data’s length (in bytes).

## Arrays in C

- `char myString[40];` -> type of `myString` is `char*`
- `char* myArrayOfStrings[20];` -> type of `myArrayOfStrings` is `char**`
- `int counting[5] = {1, 2, 3, 4, 5};` -> type of `counting` is `int*`

### `[]`

The bracket operator (i.e accessing `arr[1]`) is just syntactic sugar for pointer arithmetic. 

- If we have int `arr[4] = {5, 6, 7, 8};` these are equivalent: 
	- `arr[2] = 50;` 
	- `*(arr + 2) = 50;` -> Remember pointer arithmetic! 
	- `2[arr] = 50;` -> Addition is commutative!

### Downsides

- We are allowed to access or modify illegal memory by accessing an array out of bounds. C provides no checking whatsoever. 
- **Use your size variables whenever possible!**

## Bitwise Operators in C

- & (and): `10001 & 10000` -> 10000 
- | (or): `10001 | 10000` -> 10001 
- ^ (xor): `10001 ^ 10000` -> 00001 
- ~ (complement): `~10000` -> 01111
- << (left shift): `1 << 4` -> 10000 (binary) -> 16 (decimal) 
- \>> (right shfit): `10101 >> 3` -> 10 (binary)

## Casting in C

- To cast in C: `(newType)variable`
- void* to char*: `(char*)myVoidPtr`
- uint64 from expression: `(uint64)(2 + 3)`, `(uint64)myVoidPtr`

## `#include` in C

- .h files contain declarations (specs) 
- .c files contain definitions (implementations) 
- Basically never `#include` a .c file!
- Aux
	- Include Guards help deal with nested/duplicate `#includes` (not used that much in xv6)
	- Use the `extern` keyword! Extends function’s visibility to all files in the program.



# <u>Lecture 3: OS organization</u>

## UNIX interface

### Abstracts the H/W resources

- Processes (instead of cores): fork()
	- ​	<img src="image.assets/Screen Shot 2021-11-09 at 22.52.02.png" alt="Screen Shot 2021-11-09 at 22.52.02" style="zoom:25%;" />

- Memory (instead of physical memory): exec(), brk()
- Files instead of disk blocks: open(), read(), write()
- Pipes instead of shared physical memory: pipe()

## Kernel

### OS must be defensive

- Solution: use CPU hardware support
	- User/kernel mode (privilege modes)
	- Virtual memory

- add instruction to change mode in controlled way
	- `ecall <n>`
		- enters kernel mode at a pre-agreed entry point

### Kernel is the Trusted Computing Base (TCB)



### Monolothic kernel vs. Microkernel



## Development using Qemu

- What is "to emulate"?
	- Qemu is a C program that faithfully implements a RISC-V processor
- [big idea: software = hardware]





# <u>Lecture 4. Page Tables</u>

- address spaces
- paging hardware (risc-v)
- xv6 vm code + layout

### address space

- each program
- independent --> isolation

### page tables (h/w)

- every app has its own map
- satp register: pointer to page table
	- change register value when switch app
- strategy
	- ​    <img src="image.assets/Screen Shot 2021-10-29 at 18.05.10.png" alt="Screen Shot 2021-10-29 at 18.05.10" style="zoom:25%;" />
	- store per page: 4kB
	- 27-bit index, 12-bit offset
	- PA: 44-bit index, 12-bit offset
- risc-v pagetable
	- ​    <img src="image.assets/Screen Shot 2021-10-29 at 18.04.40.png" alt="Screen Shot 2021-10-29 at 18.04.40" style="zoom: 25%;" />
	- 3-stage: three 9-bit index page table
	- page table entry PTE
		- flags
			- v, r, w, e
- translation look-aside buffer TLB
	- cache of pte entries [va, pa]
	- switch page table --> flush TLB

### kernel page layout

-   ​	<img src="image.assets/Screen Shot 2021-11-02 at 09.46.51.png" alt="Screen Shot 2021-11-02 at 09.46.51" style="zoom:25%;" />
- 0x8000000 determined by h/w (board)
	- above to dram
	- below to i/o devices
	
-   cache
  - cache with va before MMU
  - cache with pa after MMU



# <u>L5. RISC-V Calling Convention and Stack Frames</u>

- Assembly
	- c --> assembly (.S / .asm) --> binary (.o)
	- RISC-V vs. x86-64
		- Reduced ISA vs. Complex ISA (CISC)
		- fewer/simple instructions vs. 13k instructions
		- RISC: RISC-V, ARM
		- x86: backwards compatibility

- Registers
	- ​	<img src="image.assets/Screen Shot 2021-11-04 at 11.34.47.png" alt="Screen Shot 2021-11-04 at 11.34.47" style="zoom:25%;" />
	- Saver
		- Caller: not preserved across fn call
		- Callee: preserved across fn call
- Stack
	- sp --> bottom of stack
	- fp --> top of current frame
	- asm function
		- fn prologue
		- body
		- epilogue

## *Thinking*

- x86和arm/risc-v代表着不同的技术路线，前者由于向下兼容导致了数量庞大的指令集

# <u>L6. Isolation & System Call Entry/Exit</u>

### user -> kernel transition

- important for isolation and performance

### What needs to happen when a program makes a system call

- CPU resources are set up for user execution (not kernel)
	- 32 registers (including sp), pc, privilege mode, satp, stvec, sepc, sscratch, ...

### supervior

- r/w ctl regs
	- satp, stvec, sepc, sscratch, ...
- use ptes w/o PTE_U

## demo

- pte
	- trapframe and trampoline without permission u
- `ecall` does 3 things
	- change mode from user to supervisor
	- save `pc` in `sepc`
	- jump to `stvec` (i.e. set `pc` to `stvec`)



# <u>L8: Page Faults</u>

## page faults

### RISC-V page faults

- 3 of 16 exceptions are related to paging
- Exceptions cause controlled transfers to kernel

### Information at page fault

  1) The virtual address that caused the fault
	See `stval` register; page faults set it to the fault address
  2. The type of violation that caused the fault

	See `scause` register value (instruction, load, and Store page fault)

  3. The instruction and mode where the fault occurred

	User IP: `tf->epc`
	U/K mode: implicit in usertrap/kerneltrap

## change page tables on page fault

### lazy allocation

- sbrk() is old fashioned
- moderns OSes allocate memory lazily

### zero-fill on demand

- text, data, bss(global variables with value 0)
- map all pages on bss to one pa on startup with read only
- when store, page fault

### copy-on-write(cow) fork

- `fork()` --> `exec()`

### demand paging

- `exec()`
	- load page from File not eagerly but when needed 

### use virtual memory larger than physical memory

- evict a page to File
	- LRU: least recently used
	- non-dirty ones first
	- accessed bit
- use the just-free-page
- restart instruction

### memory-mapped files

- idea: allow access to files using load and store
- kernel page-in pages of a file on demand

## *Thinking*

- 利用page fault实现一些lazy/on-demand的物理内存管理，从而节约内存/提高效率
	- 惰性内存分配 lazy allocation
		- sbrk仅仅增加进程的大小，不实际分配物理内存
		- 当进程使用的时候引出pagefault，如果在范围内则分配后返回执行程序
	- 写入时复制 copy-on-write
		- fork时不实际复制物理内存，当需要写入时再复制物理内存给该进程
	- 按需分页 demand paging
		- 运行硬盘中的程序时，只初始化页表，而不初始化页表项（即不把内容复制到物理内存中），在需要时再进行复制和页表项的初始化
- 总体的逻辑是
	1. 当特定进程需要物理内存分配但并非即时使用时，先分配虚拟内存而不分配物理内存，并将该部分虚拟内存设法标记
	2. 当进程需要调用该部分虚拟内存时出现page fault
	3. 检查标记，分配物理内存并更新页表
	4. 返回该进程继续执行程序



# <u>L9: Interrupts</u>

### Interrupt

- h/w wants attention now
	- s/w save its work, 
	- process interrupt
	- resume its work

- asynchronous
- concurrency
- program devices

### Device Driver

### RISC-V interrupt-related registers

- `sie` --- supervisor interrupt enabled register
	    one bit per software interrupt, external interrupt, timer interrupt
- `sstatus` --- supervisor status register
	    one bit to enable interrupts
- `sip` --- supervisor interrupt pending register
- `scause` --- supervisor cause register
- `stvec` --- supervisor trap vector register
- `mdeleg` --- machine delegate register



# <u>L10. Locks</u>

### why talk about locking?

- apps want to use multi-core processors for parallel speed-up
- so kernel must deal with parallel system calls
- and thus parallel access to kernel data (buffer cache, processes, &c)
	- locks help with correct sharing of data
- locks can limit parallel speedup

### race condition

### the lock abstraction

- ```pseudocode
	lock l
	acquire(l)
	  x = x + 1  //-- "critical section"
	release(l)
	```

- programs have many locks

	- more parallelism

### when to lock

- conservative rule:
	- 2 processes access a shared data structure + one writes
		- lock data structure
	- too strict: lock-free programming
	- too loose: `printf("xxxx")`

### lock perspectives

1. locks help to avoid lost updates
2. locks make multi-step operations atomic
3. locks help to maintain invariant
	- assume the invariants are true at start of operation
	- operation uses locks to hide temporary violation of invariants
	- operation restores invariants before releasing locks

### Deadlock

- if an operation needs two locks
- solution: order locks, acquire locks in order

### locks vs. modularity

??

### locks vs. performance

- need to split up data structures
- best split is a challenge
- may need to rewrite code
	- lot of work!
- strategy
	1. start with coarse-grained locks
	2. measure

### broken `acquire(struct lock *l)`

- ```c
	while(1) {
		if (l->locked = 0) {
	    l-locked = 1;
	    return;
	  }
	}
	```

- race condition

### h/w test and set support

- `amoswap`
	- atomic memory operation swap
- `__sync_lock_acquire`
- `__sync_lock_release`

### interrupt

- `pop_on`, `push_off`

### memory ordering

- locked <-- 1
		x <-- x + 1
	locked <-- 0
- shouldn't be reorganized by compiler
- `__sync_synchronize`

### wrap up

- locks good for correctness
	can be bad for performance
- locks complicate programming
	- **<u>don't share if you don't have to</u>**
	- start with coarse-grained
	- use race detector



# <u>L11. Thread Switching</u>

### thread

- thread: 
	- one serial execution
	- state: pc, regs, stack
- interleave
	- multi-core
	- switch: many more threads than cores
- shared memory?
	- xv6 kernel threads: yes
	- xv6 user processes: no
	- linux user: yes (multiple threads per process)

### thread challenges

- switching - interleave
	- scheduling
- what to save/restore
- compute-bound

### timer interrupts

- h/w timer: interrupt (per several milli seconds?)
- kernel handler
	- yields: switch -- turn a thread from running to runnable
- pre-emptive scheduling
	- vs. voluntary scheduling
- thread "state"
	- running: in the cpu
	- runnable: save cpu state (pc, regs)
	- sleeping: waiting for some io event

### xv6

- switch from user stack to kstack
- switch kernel threads
- no direct user context switching

### scheduler

- TF = trapframe = saved user thread registers
- CTX = context = saved kernel thread registers
- getting from one process to another involves multiple transitions:
	- user -> kernel
		- saves user registers in trapframe    
	- kernel thread -> scheduler thread
		- saves kernel thread registers in context
	- scheduler thread -> kernel thread
		- restores kernel thread registers from ctx
	- kernel -> user
		- restores user registers from trapframe

### code

## *Thinking*

- 多个进程的调度（scheduling）
	- 概念
		- 程序program：通常储存在硬盘的文件中的指令合集
		- 进程process：被一个或多个并发线程执行的程序的实例（the instance of a computer program that is being executed by one or many threads）
		- 线程thread (of execution)：能够被调度器（scheduler）管理的最小指令序列，通常是进程的组成部分
	- 过程
		- 进程的开始
			- fork或userinit调用allocproc, 进而调用forkret
			- forkret中冒充设置context中的ra和sp至`usertrapret`
			- 当进程第一次被scheduler调度时开始运行usertrapret
		- 进程的调度
			- usertrap判定为timer interrupt，进而yield
			- yield调用sched，进而调用swtch转换context至scheduler
			- scheduler查找下一个RUNNABLE进程调用swtch转换context至该进程的sched
	- 细节
		- atomic operation
			- context的转换和进程状态的设置对于其他cpu来说需要是atomic的，因此使用锁
		- 保存ra而不是pc
			- 因为context实在swtch中转换的，保存swtch的pc并没有意义，保存swtch的返回地址，即调用swtch的地址
		- 协程coroutine
			- scheduler和sched总是互相switch



# <u>L12</u>

- small steps
  - make it working
  - test and debug a little bit a time
  - may find didn't fully understand the problem until debugging
- debug
	- use `printf()` to find infomation and check your assumptions
	- roll back to a previously working solution and add modifications instructions by instructions and see which step causes the bug
- modify uvmcopy
  - scause 2: illegal instruction
  - scause 15: store fault
- virtual address check
  - MAXVA
  - PTE_U, PTE_V
  	- not for user or not valid (trampoline, trapframe, guard page...)
  - walk()
- kfree
  - called in kinit->freerange, but not allocated yet

# <u>L13. Coordination -Sleep & Wakeup</u>

- lock
	- hold p->lock when switch
	- no other lock when switch
		- or it may cause deadlock

## Coordination

- situation
	- pipes
	- disk read
	- wait
- strategies
	- busy wait
		- `while pipe buf is empty {;}`
	- sleep/wakeup

### lost wakeups

# <u>L14. File System</u>

### why useful

- user-friendly names/pathnames
- share fiels between user/process
- persistence/durability

### why interesting

- abstraction is useful
- crash safety: wednesday lecture
- disk layout
- performance <-- storage devices are slow --> buffer cache / concurrency

### API example / file system calls

- `fd = open("x/y", -);`
	- pathname: human readable
- `write(fd, "abc", 3);`
	- offset is implicit
- `link("x/y", "x/z");`
	- multiple names
- `unlink("x/y");`
- `write(fd, "def", 3);`
	- fd: object independent of name
- `close(fd);`

### file system structures

- `inode`: file info, independent of the name
	- inode # / link count / open fd count
- fd must maintain an offset

### fs layers

- memory
	- names / fds
	- inode --> read / write
	- ichace --> synchronization
	- logging
	- buf cache
- device
	- disk

### storage

- ssd: 100us -- 1ms
- hdd: 10ms

### disk layout

- 0: boot
- 1: super block (size, ninodes): metadata
- 2-31: log (for transactions)
- 32-44: inodes (array of inodes, packed into blocks)
- 45: block in-use bitmap (0=free, 1=used)
- 46-: data

### on-disk inode

- type (free, file, directory, device)
- nlink
- size
- addr[12+1]
	- 12 direct block #
	- 1 indirect block #: to a block which stores 256 block #s
	- max file size: (256+12)*1024 bytes
	- extension: double-indirect block

### directories

- dir = file w. some structures
- content is array of dirents
- dirent (directory entry):
	- inum
	- 14-byte file name

#### code

- sys_open -> create -> ialloc
  - -> bread -> bget
  - -> log_write



# <u>L15. Crash Recovery, Logging</u>

### crash safety

- problem: crash can lead the on-disk fs to be incorrect state
- solution: logging

### risks

- fs operations are multi-step disk operations
- crash may leave fs invariants violated
- after reboot
  - crash again
  - no crash --> but r/w data incorrectly

### logging

- feature
  - atomic fs calls
  - fast recovery
  - high performance (not xv6)
- steps
  1. log writes
  2. commit op
  3. install
  4. clean log
- reboot
  - commit 0: do nothing
  - commit n: re-install + clean log

### code

### challenges

- eviction
  - pin / unpin
- fs op must fit in log
- concurrent fs calls
  - group commit

### summary

- logging for multi-step fs ops
- but: performance? (**<u>double write to disk</u>**)



# <u>L16. Linux ext3 crash recovery</u>

- log = journal
- ext3 = ext2 + journal

### xv6 log review

- write-ahead rule
- freeing rule

### problem

- too many writes per operation
  - write twice per block
- synchronous

### ext3

- mem
  - cache
  - transaction info
    - seq.: block #s
    - handles

### ext3 log format

- asynchronous
- batching
- concurrency

#### asynchronous syscalls

- i/o concurrency + batching
- prob
  - syscall returns: data might not on disk
    - --> `fsync(fd)`: make sure changes on disk before return

#### batching 批处理

- one "open" xaction
  - hundreds of syscalls
- write absorption
- disk scheduling
  - e.g. sequential block no

#### concurrency

- syscalls in parallel
- many older xactions 
  - one open
  - committing to log
  - writing to home locations
  - freed

### ext3 code

```pseudocode
sys_unlink()
	h = start()
	get(h, blockno)...
	modify blocks in cache
	stop(h)
```

### steps in commit

1. block new syscalls
2. wait for outstanding syscalls in the xaction
3. open a new xaction
4. write desc block w/ block#s
5. write blocks to log
6. wait for 4 and 5 to finish
7. write commit block
8. wait for 7 to finish ("commit point")
9. write to home locations
10. wait for 9 to finish
11. re-use the part of log

### summary

- make multi-step disk operations atomic in terms of crash
- concurrency and batching for better performance

## **Question**

### xv6 log review

- [ ] log header：如何保证n是最后写入的
- [ ] group commit: 和header是如何共同工作的
  - 写入硬盘的log是所有ops都end_op后才进行的？

### ext3

- [ ] what is an open xaction
  - 可能是可以接受syscall的xaction
- [ ] linux code
- [ ] 如何解决写两次的问题

# <u>L17. User-level virtual memory</u>

- OS kernel use vm in creative ways
- Paper argues user apps can use vm too
  - response to pagefault...
  - application
    - garbage collector
    - compression
    - SVM

### what primitives?

- trap
- prot1: decrease accessibility, e.g. R+W -> R -> no access
- protN: save TLB flushes
- unprot: increase accessibility
- dirty
- map2

### unix today

- `mmap`
  - several mmaps --> map2
  - map a file/anonymous memory into memory address space
  
- `mprotect`
  - --> prot1, protN, unprot

- `unmap`: remove address range
- `sigaction`: signal handler
  - --> trap
  - e.g. sigalarm


### vm implementation

- Address Space: pagetable(h/w) + virtual memory area(VMA)
  - VMA: contiguous range of addresses 
    - w/ same permission
    - backed by same object

### user-level trap implementation 

- pte markd invalid/read-only
  - CPU jumps in kernel
  - kernel saves state
  - asks the vm system what to do? -> upcall into user space?
  - run handler -> mprotect()?
  - handler returns to kernel
  - kernel resumes interrupted process

### example (not in paper)

- huge memorization table to remember f(0), f(1), ..., f(n) and just lookup the result when `f` is called
- challenge: table might be big (even > phys memory)
- solution: use vm primitives
  - allocate huge range
  - table lookup -> pagefault -> compute, allocate and store
  - if much memory in use, free some pages
    - prot1 / protN

### garbage collector

- doesn't need to free memory explicitly
- a copying garbage collector
  - from space / to space
  - start allocating memory in from space
  - when no space in from space -> forwarding
  - discard from space
- problem: Backer real time ...
- solution: use vm
  - scanned / unscanned areas

### what has changed since 1991?

- continuous development
  - 5-level page table
  - AS identifier
  - KPTI: kernel page table isolation
- always in flux



# <u>L18. OS Organization</u>

### what kernels should actually do? 

- not a single best answer

### monolithic

- all one program
- big abstraction, e.g. fs 
  - porrtability
  - hide complexity
  - resource management

### why not monolithic?

- big --> complex --> bugs --> security prob
- general purpose --> slow
- design baked-in
- extensibility (user to change the kernel on the fly)

### microkernels

- idea: tiny kernel, that supports
  - IPC: inter process communication
  - tasks
- still used in some specialized "computers" (certian iphone function chips?)

### why?

- a sense of aesthetics
- small 
  - --> secure
  - --> verifiable, e.g. seL4
  - --> fast
  - --> more flexible
- user level
  - --> modular
  - --> customize
  - --> robust (maybe)
- o/s personalities (multiple OSes working on top of the kernel)

### challenges

- minimum syscall API?
  - support exec(), fork()...
- rest of o/s 
- IPC speed?

### L4

- 7 syscalls
- 13,000 loc (lines of code)
- task, address space, thread
- syscalls
  - thr create
  - send/recv ipc
  - mappings
  - dev access
  - intr -> ipc
  - yield
- scheduling
- pager  

## *Thinking*

- microkernel微内核
  - 只提供内核必须的基础功能，其他功能在user level实现 
    - --> 明显的性能问题
    - --> 不利于商业化？类似谷歌模块化的手机？（用户希望一个尽可能傻瓜的产品）



# <u>L19. Virtual Machines, Dune</u>

- VMM: virtual machine monitor

- pure software emulation
  - slow
- use hadware to run virtual machine instructions directly
  - difficult

### tricks

- guest kernel in user mode
- <-> trap and emulate <->
- vmm in kernel mode (loadable kernel module)

### memory

- guest pgtbl
  - gva -> gpa (guest pa)
- vmm map
  - gpa -> hpa (host pa)
- vmm "shadow" pg tble
  - gva -> hpa

### devices

- emulation: slow
- virtual device: fast, majority (a buffer)
- pass-thru nic: modern high-performance way

### hardware vms (vt-x)

- a complete set of registers for guest mode (non-root)
- vmcs: vm control structure
  - `vmlaunch`, `vmresume`, `vmcall`
- ept: extended page table
  - gpa -> hpa

## dune

- user vt-x features to support more process features
- own pagetable
- "supervisor" mode vs. "user" mode
  - run untrusted plug-ins in <u>sandbox</u> (in user mode)
  - make garbage collector faster



# <u>L20. Kernels and HLL</u>

### why c is good: complete control

- control of memory allocation and freeing
- almost no implicit, hidden code
- direct access to memory
- few dependencies

### downsides

- buffer overruns
- use-after-free bugs
- threads sharing dynamic memory

### HLLs provide memory-safety

### HLL benefits

- type safety
- automatic memory management with garbage collector
- concurrency
- abstraction

### HLL downsides

- poor performance
  - bounds, cast, nil-pointer checks
  - garbage collection
- incompativility with kernel programming
  - no direct memory access
  - no hand-written assembly
  - limited concurrency or parallelism

### goal: measure HLL trade-offs

- impact on safety
- impact on programmability
- performance cost

### biscuit overview

- bare hardware
- kernel
  - go runtime
  - biscuit
- user processes

### featuers

- multicore, threads
- journaled fs (7k LOC)
- virtual memory (2k LOC)
- TCP/IP stack (5k LOC)
- drivers: AHCI and intel 10Gb NIC (3k LOC)

### design puzzles

- runtime on bare-metal
- goroutines run different applications
- device interrupts in runtime ciritical sections
- hardest puzzel: heap exhaustion

### heap exhaustion

- panic (xv6)
- wait for memory in allocator? --> may deadlock!
- check/handle allocation failure, like c kernels? -->
  - difficult to get right
  - can't -- go implicitly allocates
  - doesn't expose failed allocations
- biscuit solution: reserve memory
  - to execute system calls, first reserve memory

## evaluation

### benefits

- go features
  - GC'ed allocation, slices, defer, multi-valued return, strings, closures, maps
- simpler concurrency
  - ref count to decide when to free --> GC
- prevents kernel exploits

### performance (HLL tax)

- 15% slower
- 15% more memory usage
- much more safety



# <u>L21. Networking</u>

### packet headers and protocals

- tcp (transmission control protocol)
- router
  - ethernet: LAN (local area network)
    - browser
    - http
    - ...
- ethernet packet
  - start flag
  - header
    - target addr
      - first 24 bits: manufacturer number
      - low 24 bits: 6 sequential number
    - sender addr
    - type
  - payload
  - end flag
- arp
  - ethernet header / ip header / udp header
- an ip packet header
- a udp packet header (or tcp)

### stack of network software

- layering
- scheduling
  - nic (network interface card)
  - intr
  - buffer
  - ip thread
  - ...
  - application



# <u>L22. Meltdown</u>

### the core of the meltdown attack

### cpu detail: speculation  + retirement

- shadow registers

### caches

- core
- L1: va|data|perms
- TLB
- L2: pa|data
- RAM

### flush + reload - x

1. `clflush x`
2. `f()`
3. `a = rdtsc`
4. `junk = *x`
5. `b = rdtsc`
6. `b-a`

### meltdown

```pseudocode
char buf[8192]

// the Flush of Flush+Reload
clflush buf[0]
clflush buf[4096]

<some expensive instruction like divide>

r1 = <a kernel virtual address>
r2 = *r1
r2 = r2 & 1      // speculated
r2 = r2 * 4096   // speculated
r3 = buf[r2]     // speculated

<handle the page fault from "r2 = *r1">

// the Reload of Flush+Reload
a = rdtsc
r0 = buf[0]
b = rdtsc
r1 = buf[4096]
c = rdtsc
if b-a < c-b:
	low bit was probably a 0
```

### kaiser fixe

- no kernel va in user pgtbl



# <u>L23. Multi-Core scalability and RCU</u>

- rcu: read-heavy shared data
- --> multiple reader but one reader
- --> read/write lock: `r_lock(l), r_unlock(l), w_lock(l), w_unlock(l)`
- --> slow /defficient

## rcu: read-copy update

### idea 1

- modify one element in a linked list
- --> create a new element, first update its next pointer, then update its parent's next pointer (committing write: atomic)
  - not good for doubly linked list (cannot modify two addresses atomically)

### idea 2

- compiler rearrangement
- --> barrier

### idea 3

- when to free garbage
- --> delaying free
  - reader: cannot context switch
  - writer: delay free until every context switch



# <u>L24. Q&A</u>

### what next

- 6.033: how to read papers
- 6.828: graduate
- 6.823, 6.111: more hardware
- 6.172, 6.035: compiler
- 6.829: networking
- 6.830: database
- 6.824: distributed
- 6.858: security
- 6.826: principles of computer systems
- researches
  - SOSP
  - OSDI
  - lwn.net: practice (linux kernel)
- hack on a project

### lab net



### lab mmap

- file system api: good for stream update

  - ```c
    fd = open("f", ...);
    read(fd, buf, len);		// offset -> len
    buf[0] = 1;
    lseek(fd, 0)					// offset -> 0
    write(fd, buf, len);
    close(fd);
    ```

- mmap: good for arbitrarylocate update



## *Thinking*

- 最后几个课程主要是主题介绍（以及论文阅读）
  - linux的硬盘恢复
    - write when clean (not write through as in xv6)
    - concurrent write in an open transaction
  - 用户层的虚拟内存使用（pagefault）
  - 微内核 microkernel
  - 虚拟机
    - tricks
      - 一个在内核中安装的虚拟机监控器VMM
      - guest kernel在user mode中
      - 涉及到privilege instruction才将权限交给vmm
    - 硬件，如vt-x
      - 提供一套完整的register给VM，这样privilege instruction也不需要进入vmm
      - 前提：guest操作系统知道自己运行在虚拟机上
    - 设备
      - 设置虚拟设备，将读写缓存在虚拟设备中，（定期？）通过interrupt对物理设备进行读写
      - --> 可以避免大量interrupt
    - 其他应用
      - 应用程序使用VMM将程序分为两个部分，如将不受信任的插件运行在“user” mode中
  - 高级语言编写kernel
    - 15%的降速和增加的内存消耗
    - 安全性极高
    - ++ 更容易debug？
  - 网络
    - 基本机制的介绍，通过lab来熟悉
  - meltdown熔断
    - 这是一个将硬件的trick（6.004）和软件的trick（6.s081）结合的极好的案例
    - 使用硬件的分支预测技术获得没有权限的信息，并在处理器取消这些操作前将其保存在用户地址空间（user address space）中
    - 问题：intel的分支预测技术在获取L1 cache的内容时不需首先查看权限，而是在分支确定后再取消操作结果
  - RCU： read-copy update
    - 大量读操作时可以使读操作不获取锁 --> 使用技巧将数据的更新变成atomic的
    - 扩展
      - 对于大量写操作时可以讲数据分区，各区有单独的锁
      - ref count可以在每个cpu核中设置，写操作不需要锁，读操作需要获得所有核对应的锁





















