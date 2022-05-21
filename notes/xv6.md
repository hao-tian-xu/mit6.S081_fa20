##  **XV6**

### **INITIALIZATION**

- 1 `entry.S`
  - `_entry`
    1. set up a stack so that xv6 can run C code
    2. loads the stack pointer register `sp` with the address `stack0+4096`, the top of the stack, because the stack on RISC-V grows down
    3. calls into C code at `start()`
  - ...
- 2 `strat.c`
  - `start`
    1. performs some configuration that is only allowed in machine mode
    	1. ...
    	2. calls `timerinit()` to set up to receive timer interrupts
    	2. keeps each CPU's hartid in its `tp` register, for `cpuid()`
    2. then switches to supervisor mode
    	1. ...
    	2. it sets the return address to `main` by writing `main`’s address into the register `mepc`
    	3. ...
    	4. "returns" to supervisor mode by calling `mret` (provided by RISC-V), and causes the `pc` to change to `main`
  - `timerinit`
  	1. program the `CLINT` hardware (core-local interruptor) to generate an interrupt after a certain delay
  	2. set up a `scratch` area, analogous to the trapframe, to help the timer interrupt handler save registers and the address of the `CLINT` registers
  	3. sets `mtvec` to timervec and enables timer interrupts
- 3 `main.c`
  1. `consoleinit()`: to initialize the UART hardware
  1. ...
  1. `kinit()`: physical page allocator
  2. `kvminit()`: create kernel page table
  3. `kvminithart()`: turn on paging
  4. `procinit()`: process table
  5. ...
  5. `plicinit()`: set up interrupt controller
  5. `plicinithart()`: ask PLIC for device interrupts
  5. ...
  6. `userinit()`: calls `user/initcode.S`
  7. ...
  7. `scheduler()`
- <u>SHELL</u>
  - `user/initcode.S`
    - ...
    - use `ecall` (provided by RISC-V) to somehow run syscall `exec` to run a user program `init`
    - ...
    - `user/init.c`
      - `main`
        - open a file descriptor for the console
        - ...
        - starts a shell on the console

### **PROCESS BASICS**

- <u>Functions</u> in `proc.c`
	- <u>Allocation</u>
		- `allocproc`
			- Look in the process table for an UNUSED proc, return the proc pointer
				- ...
				- set up new context
					- set `p->context.ra` to `forkret()` for `scheduler()` to `swtch()` to
				- ...
			- `forkret`
				- ...
				- calls `usertrapret()` to return to user space
		- `proc_pagetable`
			- allocates a new page table with no user mappings
		- <u>Initialization</u>
			- `procinit`
				- initialize the proc table at boot time
				- `KSTACK`
			- `userinit`
				- <u>set up first user process -- its proc structure and hanging data</u>
				- calls `allocproc()`
				- ...
				- ... somehow calls `initcode.S` ...
				- ...
	- <u>Free</u>
		- `freeproc`
			- free a proc structure and the data hanging from it
			- `proc_freepagetable`
				- free a process's page table, and free the physical memory it refers to
				- `uvmunmap()` ...
				- `uvmfree()`
			- ...
	- <u>Operation</u>
		- `growproc`
			- implementation of syscall `sbrk` (space increments after program break address)
			- grow or shrink user memory by n bytes
			- `uvmalloc()` or `uvmdealloc()`
		- `fork`
			- create a new process, copying the parent 
			- `allocproc()`
			- `uvmcopy()`
			- ...
		- ...
	- <u>Coordination</u>
		- `wait`
			- `acquire(&p->lock)` to avoid lost wakeups
			- infinite loop
				- scans the process table to find a `ZOMBIE` child
				- if found
					- calls `freeproc()` to free the `ZOMBIE` process
					- `release(&p->lock)`
					- returns the child’s process ID
				- if no children, releases the lock and return -1
				- if have children but none `ZOMBIE`, `sleep()` on `p`
		- `exit`
			- ... records the exit status, frees some resources ...
			- acquires any neccessary locks to call `reparent()`, which gives any children to the `init` process
			- `wakeup1()`: wake up the parent in case it is in `wait()` (the parent's `p->lock` is still held)
			- marks the process as a `ZOMBIE`, to be freed by `wait()`
				- <u>*cannot free the proc itself when the proc is still executing (especially pagetable and stack)*</u>
			- releases the parent's `p->lock`
			- `sched()` to permanently yield
		- `kill`
			- <u>kill the process with the given pid</u>
			- set `p->killed`
			- wakeup the process
			- <u>`usertrap()` will kill the process when it tries to return to user space</u>
	- ...
- <u>Data Structures</u> in `proc.h`
	- `struct proc`
		- per-process state
		- ...
		- `struct context`: `swtch()` here to run process
		- ...
	- ...
- <u>Locks</u> in `proc.h`
	- proc's `p->lock`
		- serializes changes to process’s state `p->state`
- <u>Locks</u> in `proc.c`
	- `pid_lock`
		- serializes increments of `nextpid`
- <u>USER PORGRAM</u>

	- <u>Functions</u> in `exec.c`
		- `exec`
			- new virtual memory preparation
				1. allocates a new page table with no user mappings with `proc_pagetable()`
				2. allocates memory for each ELF segment with `uvmalloc()`
				3. loads each segment into memory with `loadseg()`
			- replacement...
			- ...
		- `loadseg`
			- load a program segment into pagetable at virtual address va
				1. uses `walkaddr()` to find the physical address of the allocated memory
				2. at which to write each page of the ELF segment, 
				3. and `readi()` to read from the file...

	- <u>Data Structures</u> in `elf.h`
		- `struct elfhdr`
			- ...
			- `struct proghdr`
				- ...

### **MEMORY**

- **PHYSICAL MEMORY** 
	- <u>ALLOCATION</u>
		- <u>For user processes, kernel stacks, page-table pages, and pipe buffers</u>
		- <u>Functions</u> in `kalloc.c`
			- `kinit`
				- calls `freerange()` to initializes the free list to hold every page between the end of the kernel and `PHYSTOP`
				- `freerange`
					- add memory to free list via per-page calls to `kfree()`
			- `kalloc`
			  - allocate one 4096-byte page of physical memory, return a pointer
			- ❋❋`kfree`❋❋
			  - fill with junk and prepends the page to the free list
		- <u>Data Structure</u> in `kalloc.c`
			- `struct kmem`
				- containing a freelist as a linked list `struct run* freelist`
			- `extern char end[]`
				- first address after kernel, defined by kernel.ld.
- **PAGE TABLE**
	- <u>Macros</u> in `riscv.h`
	  - `PGROUNDUP`, `PGROUNDDOWN`
	    - adjust va to align with page size
	  - `PA2PTE`
	    - shift left to leave space for flags
	  - `PTE2PA`
	    - shift right to remove flags
	- <u>Functions</u> in `vm.c`
	  - <u>Kernel</u>
	  	- `kvminit` 
	  		- `kalloc()`: allocates a page of physical memory to hold the root page-table page
	  		- `kvmmap()`: direct mapping - same `va` and `pa`
	
	  	- `kvmmap`: add a mapping to the kernel page table
	  		- kernel’s instructions and data
	  		- physical memory up to `PHYSTOP`
	  		- memory ranges which are actually devices

	  	- `kvminithart`
	  		- writes the physical address of the root page-table page into the register `satp`
	  	- ...
	  - <u>Allocation</u>
	  	- ❋❋`mappages`❋❋
	  		- <u>create PTEs for virtual addresses starting at va that refer to physical addresses starting at pa</u>
	  	- `uvmalloc`
	  		- allocates physical memory with `kalloc()`, and adds PTEs to the user page table with `mappages()`
	  	- ...
	  - <u>Free</u>
	  	- ❋❋`uvmunmap`❋❋
	  		- uses `walk()` to find PTEs and set values to 0, and optionally `kfree()` to free the physical memory they refer to
	  	- `uvmdealloc`
	  		- calls `uvmunmap()`
	  	- `uvmfree`
	  		- free user memory pages `uvmunmap()`, then free page-table pages `freewalk()`
	  		- `freewalk`
	  			- recursively free page-table pages. all leaf mappings must already have been removed.
	  	- `uvmcopy`
	  		- fork: given a parent process's page table, copy its memory into a child's page table.
	  	- ...
	  - <u>Walk</u>
	  	- `walk`
	  		- <u>finds the PTE for a virtual address</u>
	  	- `walkaddr`
	  		- <u>finds the physical address for a virtual address</u>
	  		- calls `walk()`
	  		- calls `PTE2PA()`
	  	- ...
	  - <u>Others</u>
	  	- `copyout`
	  		- copy from kernel to user. copy len bytes from src to virtual address dstva in a given page table.
	  		- `walkaddr()` to find pa from page table
	  		- `memmove()`
	  	-  `copyin`
	  	  - reason for being in `vm.c`: need to explicitly translate addresses
	  	- `copyinstr`
	  	  - copy a null-terminated string from user to kernel
	- <u>Data Structures</u> in `riscv.h`
	  - `uint64* pageable_t`
	  	- a pointer to a RISC-V root page-table page
	  - `uint64 pte_t`
	  	- PTE: an uint64 number
	- <u>Macros</u> in `riscv.h`
	  - `PGROUNDDOWN`
	  	- set last 12 bits to 0 (ground down to page size) 
	  - `PA2PTE`, `PTE2PA`
	  	- pa to pte, pte to pa
	  - ...
	- <u>Kernel pgtbl Layout</u>
	  - <u>Macros</u> in `memlayout.h`
	    - `KSTACK`
	      - map kernel stacks beneath the trampoline, each surrounded by invalid guard pages.
	    - ...
	  - <u>Hardware Memory-mapped Adresses</u> in `memlayout.h`
	    - `UART0`
	    - ...

### **TRAP**

- <u>Hardware Instructions</u> 
    - `ecall` (Environment Call)
        - change mode from user to supervisor
        - save `pc` in `sepc`
        - jump to `stvec` (i.e. set `pc` to `stvec`) refering to `TRAMPOLINE`
    - `sret` (Supervisor Return)
      - change mode from supervisor to user
      - jump to `stvec` (i.e. set `pc` to `stvec`) refering to trapped user code
- <u>Functions</u> in `trampoline.S`
    - `uservec` 
      - user vector
        1. swap `a0` and `sscratch`
          1. `sscratch` refering to `TRAPFRAME` when going to user space (see `usertrapret`)
        2. save the user registers in `TRAPFRAME`
        3. retrieve pointers to kernel stack, usertrap, and kernel page table, and CPU's hartid
        4. set `satp` to referring to kernel page table
          1. no crashing since `TRAMPOLINE`s' va are all the same in all page tables 
        5. call `usertrap()`
    - `userret` 
      - user return
        1. switch to the user page table
        2. copies the trapframe’s saved user `a0` to `sscratch` in preparation for a later swap with `TRAPFRAME`
        3. restores saved user registers from the trapframe
        4. swap `a0` and `sscratch` to restore the user `a0` and save `TRAPFRAME` to `sscratch` for the next trap
        5. return to user space
- <u>Functions</u> in `trap.c`
    - `usertrap`
      - changes `stvec` to point to `kernelvec` (since in the kernel now)
      - saves the `sepc` (the saved user program counter)
      - handlers
           - if syscall (`r_scause() == 8`)
               1. ...
               2. `intr_on()`
               3. interrupts are always turned off by the RISC-V trap hardware (`ecall`?)
               4. an interrupt will change sstatus &c registers, so don't enable until done with those registers
               5. `syscall()`
           - device interrupt (`devintr() != 0`)
           - else
               - pagefault (`r_scause() == 13 or 15`)
               - ...
      - call `usertrapret()`
    - `usertrapret`
      - sets up the RISC-V control registers to <u>prepare for a future trap from user space</u>
        1. call `intr_off()`
          1. we are still in kernel but we are going to change to user trap vector
        2. changing `stvec` to refer to `uservec`
        4. preparing the trapframe fields that `uservec` relies on (see `uservec` 3.)
        	1. ...
        	2. `tp`: hartid for `cpuid()`, the user process might modify it
        5. set `sepc` to the previously saved user program counter
        6. call `userret` with `a0` to `TRAPFRAME` and `a1` to user page table
          7. since we can only change satp in `trampoline.S` because it's the only code with same VAs in kernel space and every user space
    - `kerneltrap`
      1. handle traps
      	  - device interrupts: `devintr()`
      	  - exceptions: `panic()`
      	  - timer interrupt: `yield()`...
      2. due to `yeild()`, restore `sepc` and `sstatus`
      3. return to `kernelvec`
    - `devintr`
    	- ...
    	- calls `plic_claim()` to check which device interrupted
    		- if UART, calls `uartintr()`
    		- ...
    	- ...
- <u>Data Structures</u> in proc.h
  - `struct trapframe`
    - per-process data for the trap handling code in `trampoline.S`
- **TRAP FROM KERNEL SPACE**
  - <u>Functions</u> in `kernelvec.S`
    - `kernelvec`
      1. saves the registers on the stack of the interrupted kernel thread
      2. jumps to `kerneltrap()` to handle traps
      3. restore saved registers
      4. call `sret`, which copies `sepc` to `pc` and resumes the interrupted kernel code
    - `timervec`
    	1. saves a few registers in the scratch area prepared by `start()`
    	2. tells the `CLINT` when to generate the next timer interrupt
    	3. asks the RISC-V to raise a software interrupt
    	4. restores registers, and returns
- **SYSCALL**
  - <u>Functions</u> in `syscall.c`
    - `syscall`
      - ...
    - <u>Arguments</u>
      - `argint`, `argaddr`
        - retrieve the nth argument as int or pointer using `argraw()`
        - `argraw`
        	- retrieve the appropriate saved user register from the current process' trap frame (see `uservec`)
      - `argfd`
        - retrieve the nth word-sized argument as a file descriptor
      - `argstr`
        - retrieve the nth argument as string
        	- call `argaddr()` to find address, 
        	- then call `fetchstr()` to find corresponding string
      - `fetchstr`
        - fetch the nul-terminated string at addr from the current process, returns length of string
        - call `copyinstr()`
      - `fetchaddr`
        - fetch the uint64 at addr from the current process
        - call `copyin()`
  - <u>Functionality</u>
  	- Syscall from User Space
  		1. `ecall` (hw)
  		2. `uservec` (asm)
  		3. `usertrap()`
  		4. `usertrapret()`
  		5. `userret` (asm)
  		6. `sret` (hw)
- **INTERRUPT**
	- <u>Functions</u> in `plic.c`
	  - <u>the riscv Platform Level Interrupt Controller (PLIC)</u>
	  - `plic_claim`
	    - ask the PLIC what interrupt we should serve
	  - `plicinit`
	  	- enable UART interrupt request (IRQ)
	  	- enable VIRTIO interrupt request (IRQ)
	  - `plicinithart`
	  	- set UART's enable bit for this hart's S-mode
	- **CONSOLE**
	  - <u>Hardware</u>
	    - <u>When the user types a character, the UART hardware asks the RISC-V to raise an interrupt, which activates xv6’s trap handler, which calls `devintr()`</u>
	    - <u>Each time the UART finishes sending a byte, it generates an interrupt</u>
	  - <u>UART Control Registers</u> in `uart.c`
	    - <u>The UART hardware appears to software as a set of memory-mapped control registers</u>
	    - `RHR`: receive holding register (for input bytes)
	    - `THR`: transmit holding register (for output bytes)
	    - ...
	  - <u>Macro Functions</u> in `uart.c`
	  	- `ReadReg`
	  	- `WriteReg`
	  	- ...
	  - <u>Functions</u> in `uart.c`
	    - `uartinit`
	      - ...
	      - enable transmit and receive interrupts
	    - `uartintr`
	      - repeat until get -1
	      	- calls `uartgetc()` to get one character from the UART hardware
	      	- calls `consoleintr()`
	      - calls `uartstart()` to send buffered characters
	    - `uartputc`
	    	- <u>add a character to the output buffer and tell the UART to start sending if it isn't already</u>
	    	- calls `uartstart()`
	    - `uartstart`
	    	- ...
	    	- calls `WriteReg(THR, c)`
	  - <u>Functions</u> in `console.c`
	    - `consoleinit`
	    	- ...
	    	- calls `uartinit()`
	    	- connect read and write system calls to consoleread and consolewrite.
	    - `consoleread`
	    	- <u>a read system call on a file descriptor connected to the console eventually arrives here</u>
	    	- ...
	    	- woken by `consoleintr()` and observe a full line in `cons.buf`
	    	- copy it to user space, and return to user space
	    - `consolewrite`
	    	- <u>a write system call on a file descriptor connected to the console eventually arrives here</u>
	    	- repeat until...
	    		- calls `uartputc()`
	    	- ...
	    - `consoleintr`
	    	- to accumulate input characters in `cons.buf` until a whole line arrives
	    		- it treats backspace and a few other characters specially
	    	- when a newline arrives, wakes up a waiting `consoleread()`
	  - <u>Data Structure</u> in `console.c`
	    - `struct cons`
	    	- ...
	  - <u>Functionality</u>
	    - Read from UART
	      - save inputs from UART hardware
	        1. `usertrap()` user trap
	        2. `devintr()` device interrupt
	        3. `uartintr()` UART interrupt
	        	1. `uartgetc()` get a character from H/W reg
	        	2. `consoleintr()` accumulate characters and wakeup `consoleread()`
	      - read inputs from UART hardware
	        1. `sys_read()`
	        2. `fileread()`
	        3. `consoleread()` woken by `consoleintr()`
	    - Write to UART
	    	1. `sys_write()`
	    	2. `filewrite()`
	    	3. `consolewrite()`
	    	4. `uartputc()`
	    		1. `uartstart()`
	    	5. each time the UART finishes sending a byte, it generates an interrupt to arrive `uartintr()`
	    		1. `uartstart()`
	- **TIMER**
	  - ...
	  	- `timervec()`
	  	- `clockintr()`
	  	- ...

### **CONCURRENCY**

- **LOCK**
  - <u>Hardware Instructions</u>
    - `amoswap r, a`
    	- `amoswap` reads the value at the memory address `a`, writes the contents of register `r` to that address, and puts the value it read into `r`
    	- performs this sequence atomically, using special hardware to prevent any other CPU from using the memory address between the read and the write
  - <u>C Library Calls</u>
    - `__sync_lock_test_and_set`
      - boils down to `amoswap`
      - return old value of `lk->locked`
    - `__sync_lock_release`
      - boils down to `amoswap` to set `lk->locked` to `0`
    - `__sync_synchronize`
    	- a memory barrier: it tells the compiler and CPU to not reorder loads or stores across the barrier
  - <u>Data Structures</u> in `spinlock.h`
    - `struct spinlock`
    	- mutual exclusion lock
    	- `uint locked`: is the lock held?
  - <u>Functions</u> in `spinlock.c`
    - `acquire`
      1. calls `push_off()` to disable interrupts to avoid deadlock
      2. calls `__sync_lock_test_and_set()` in a loop
      3. calls `__sync_synchronize()` to avoid reordering of critical section
      4. records, for debugging, the CPU that acquired the lock
    - `release`
      1. cleans the `lk->cpu` field
      2. calls `__sync_synchronize()` to avoid reordering of critical section
      3. calls `__sync_lock_release()`
      4. calls `pop_off()` to optionally enable interrupts
    - `push_off`
      - calls `intr_off()`
      - record numbers of `push_off - pop_off` 
    - `pop_off`
      - record numbers of `push_off - pop_off` 
      - if `0`, calls `intr_on()`
  - <u>Deadlock Situations</u>
  	- didn't disable interrupt in `acquire()` --> deadlock in interrupt handler
  	- hold any spinlock before `yield()` --> dealock if any other process tryies to acquire the lock
  - **SLEEP LOCK**
    - <u>Data Structures</u> in `sleeplock.h`
    - <u>Functions</u> in `sleeplock.c`
- **SCHEDULING**
	- <u>Data Structures</u> in `proc.h`
		- `struct context`
			- saved registers for kernel context switches
			- `ra`, `sp`, and callee-saved regs
		- `struct cpu`
			- per-CPU state
			- `struct proc*`
			- `struct context`: `swtch()` here to enter `scheduler()`
			- `int noff`: Depth of `push_off()` nesting
			- `int intena`: were interrupts enabled before `push_off()`?
	- <u>Functions</u> in `swtch.S`
		- `swtch`
			- takes two arguments: `struct context *old` and `struct context *new`
			- saves the current registers in `old`, loads registers from `new`
				- saves only callee-saved registers, caller-saved registers are saved on the stack by the calling C code
				- does not save the program counter `pc`, and instead saves the `ra` register, which holds the return address from which `swtch` was called
				- saves `sp`
			- `ret`
				- returns to the instructions pointed to by the restored `ra` register, that is, the instruction from which the new thread previously called `swtch`
				- in addition, returns on the new thread’s stack pointed to by the restored `sp` register
	- <u>Functions</u> in `proc.c`
		- `yield`
			- <u>give up the CPU for one scheduling round</u>
			- acquire its own process lock `p->lock`
			- sets current process' state to `RUNNABLE`
			- calls `sched()`
		- `sched`
			- <u>switch to scheduler</u>
			- double checks if the process holds `p->lock` but not any other locks, and its state is updated
			- calls `swtch(&myproc()->context, &mycpu()->context)`
		- `scheduler`
			- scheduler never returns. It loops, doing:
				- loops over the process table looking for a runnable process, one that has `p->state == RUNNABLE`
				- sets the per-CPU current process variable `c->proc`, marks the process as `RUNNING`
				- calls `swtch(&mycpu()->context, &p->context)` to start running that process.
				- eventually that process transfers control via `swtch` back to the scheduler.
		- <u>Current Process and CPU</u>
			- `cpuid`
				- retrieve from `tp` register
				- must be called with interrupts disabled, to prevent race with process being moved to a different CPU.
			- `mycpu`
				- call `cpuid()`...
			- `myproc`
				- call `mycpu()`...
	- <u>Functionality</u>
		- Timer Interrupt
			- ...
			- `usertrap()`
			- -> `yield()`
			- -> `sched()`
			- -> `swtch()`: swtich context to `scheduler()`
			- <-> `scheduler()`: find next `RUNNABLE` process
			- -> `swtch()`: switch context to `sched()`
			- <-> `sched()`: return to `usertrap()` to resume
			- ...
		- Process Init
			- `userinit()` or `fork()`
			- -> `allocproc()`
			- -> fake `p->context.ra` and `p->context.sp` to `forkret()`
			- <- `userinit()` or `fork()`: set process state to RUNNABLE
			- `scheduler()`: loops to the process
			- -> `swtch()`: switch (fake) context to `forkret()`
			- <-> `forkret()`
			- -> `usertrapret()`: return to user space to begin executing user code
	- **COORDINATION**
		- <u>Functions</u> in `proc.c`
			- `sleep`
				- to sleep
					- acquires `myproc()->lock`
					- releases lock from caller
					- changes state to `SLEEPING`
					- calls `sched()` to yield
				- awakened
					- releases `myproc()->lock`
					- acquires lock from caller
					- returns to caller
			- `wakeup`
				- traverse the process table
					- calls `acquire(&p->lock)`
					- if `SLEEPING` and same channel, sets state to `RUNNALBE`
					- calls `release(&p->lock)`
			- `wakeup1`
				- wake up one process `p` if it is sleeping in `wait()`; used by `exit()`
		- <u>Functionality</u>
			- Pipe
			- Wait, Exit, and Kill

### **FILE SYSTEM**

- **LAYERS**
  - **BUFFER CACHE**
    - <u>Functions</u> in `bio.c` (buffer io)
      - `binit`
        - initializes the list with the `NBUF` buffers in the static array `buf`, all other access to the buffer cache refer to the linked list via `bcache.head`, not the `buf` array
      - `bread`
        - calls `bget()` to get a buffer for the given sector
        - if not valid, calls `virtio_disk_rw()` to read from disk and set valid
        - returns the buffer
        - `bget`
        	- `acquire(&bcache.lock)`
        	- scans the buffer list <u>forward</u>
        		- looking for a buffer with the given device and sector numbers
        		- if found
        			- `release(&bcache.lock)`
        			- `acquiresleep(&b->lock)`
        			- returns the buffer
        	- else, scans the buffer list <u>backward</u>
        		- looking for a buffer that is not in use
        		- ...
      - `bwrite`
        - <u>called by `commit()`</u>
        - write buffer `b`'s contents to disk. must be locked
        - calls `virtio_disk_rw()`
      - `brelse` (b-release)
        - releases the sleep-lock and moves the buffer to the front of the linked list
        - ...
    - <u>Data Structures</u> in `bio.c`
      - `struct {} bcache`
        - `struct buf buf[NBUF]`: all buffers in xv6
    - <u>Data Structures</u> in `buf.h`
      - `struct buf`
      	- state fields
      		- `int valid`: contains a copy of the block
      		- `int disk`: buffer content has been handed to the disk
      	- ...
    - <u>Locks</u> in `bio.c`
      - `bcache.lock`
      	- protects allocation of block buffer cache entries
      	- (protects information about which blocks are cached)
    - <u>Locks</u> in `buf.h`
      - `struct buf`'s `b->lock` (`struct sleeplock`)
      	- serializes operations on each block buffer
      	- (protects reads and writes of the block’s buffered content)
  - **LOGGING**
    - <u>Functions</u> in `log.c`
      - `begin_op`
        - waits until the logging system is not currently committing, and until there is enough unreserved log space to hold the writes from this call
      - `log_write`
        - acts as a proxy for `bwrite()`:
          - records the block’s sector number in memory, 
          - reserving it a slot in the log on disk, and 
          - pins the buffer in the block cache to prevent the block cache from evicting it.
        - supports *absorption*
      - `end_op`
        - decrements the count of outstanding system calls
        - if the count is now zero, it commits the current transaction by calling `commit()`
        - `commit`
          - `write_log()` -> `write_head()` -> `install_trans()` -> set `log.lh.n` to `0` -> `write_head()`
          - `write_log`
            - copies each block modified in the transaction from the buffer cache to its slot in the log on disk
          - `write_head`
            - writes the header block to disk: this is the **<u>commit point</u>**
          - `install_trans`
            - reads each block from the log and writes it to the proper place in the file system
      - `initlog`
        - called from `fsinit()` during boot before the first user process runs
        - ...
        - `recover_from_log`
          - reads the log header, and mimics the actions of `end_op` if the header indicates that the log contains a committed transaction
    - <u>Data Strcutures</u> in `log.c`
      - `struct logheader`
        - `int n`: the count of log blocks
        - `int block[LOGSIZE]`: an array of block numbers, one for each of the logged blocks
      - `struct log {} log`
        - ...
        - `int outstanding`: the number of system calls that have reserved log space
          - the total reserved space is `log.outstanding` times `MAXOPBLOCKS`
        - ...
        - `struct logheader lh`
  - **INODE / DIRECTORY / PATHNAME**
    - <u>Functions</u> in `fs.c` (File System)
      - <u>Block Allocator</u>
        - `balloc`
          - looks for a block whose bitmap bit is zero, indicating that it is free
        - `bfree`
          - finds the right bitmap block and clears the right bit
      - <u>Inode</u>
        - `ialloc(uint dev, short type)`
          - (used by `create()` when creating new file)
          - allocate an inode on device `dev`
          - marks it as allocated by  giving it type `type`
          - calls `iget()` to return an unlocked but allocated and referenced inode
        - `iget`
          - looks through the inode cache for an active entry (`ip->ref > 0`) with the desired device and inode number
            - if finds one, returns a new reference to that inode
            - records the position of the first empty slot
          - else, allocates a cache entry using the empty slot
          - ...
        - `iput`
          - releases a C pointer to an inode by decrementing the reference count
          - if last reference, the inode’s slot in the inode cache is free and can be re-used for a different inode
            - calls `itruc()` to truncate the file to zero bytes, freeing the data blocks
            - sets `ip->type` to `0`
            - calls `iupdate()` to writes the inode to disk
        - `ilock`
          - <u>sleeplocks the inode (so that no other process can `ilock()` it) and reads the inode from the disk, if it has not already been read</u>
          - calls `acquiresleep(&ip->lock)`
          - ...
        - `iunlock`
          - <u>releases the sleeplock on the inode</u>
          - calls `releasesleep(&ip->lock)`
        - `iupdate`
          - copy a modified in-memory inode to disk
          - ...
      - <u>Inode Content</u>
        - `bmap(struct inode *ip, uint bn)`
          - returns the disk block number of the `bn`th data block for the inode `ip`
            - first `NDIRECT` blocks
            - next `NINDIRECT` blocks
          - if `ip` does not have such a block yet, `bmap` allocates one
        - `itrunc`
          - truncate inode (discard contents)
            - first `NDIRECT` blocka
            - next `NINDIRECT` blocks
            - finally the indirect block itself
          - resets the `ip->size` to `0`
        - `readi`
          - read data from inode
          - ...
          - calls `bread()` to find/allocate buffer cache
          - calls `either_copyout()` to copy data from buffer cache
          - ...
        - `writei`
          - write data to inode
          - ...
          - calls `bread()` to find buffer cache
          - calls `either_copyin()` to copy data to buffer cache
          - calls `logwrite()` to write to disk
          - ...
        - `stati`
          - copies inode metadata into the `stat` structure, which is exposed to user programs via the `stat()` system call
      - <u>Directory</u>
        - <u>A directory is implemented internally much like a file. Its inode has type `T_DIR` and its data is a sequence of directory entries.</u>
        - `dirlookup`
          - <u>look for a directory entry in a directory</u>
          - ...
          - calls `iget()` to return the inode of the directory entry
        - `dirlink`
          - <u>write a new directory entry (`name`, `inum`) into the directory `dp`</u>
          - calls `dirlookup()` to check that name is not present
          - calls `readi()` to look for an empty dirent `de`, with `off` set to the offset of the available entry
            - otherwise, the loop ends with `off` set to `dp->size`
          - copies `name` and `inum` to the `de`
          - calls `writei()` to adds a new entry to the directory by writing at offset `off`
      - <u>Path</u>
        - <u>Path name lookup involves a succession of calls to dirlookup, one for each path component</u>
        - `namei`
          - evaluates path and returns the corresponding inode
          - calls `namex()`
        - `nameiparent`
          - stops before the last element, returning the inode of the parent directory and copying the final element into `name`
          - calls `namex()`
        - `namex`
          - starts by deciding where the path evaluation begins. 
            - If the path begins with a slash, evaluation begins at the root; 
            - otherwise, the current directory
          - uses `skipelem()` to consider each element of the path in turn
            - if the call is `nameiparent()` and this is the last path element, the loop stops early
            - else, the loop looks for the path element using `dirlookup()` and prepares for the next iteration
        - `skipelem(char *path, char *name)`
          - copy the next path element from path into name
          - return a pointer to the element following the copied one
    - <u>Macros</u> in `fs.h`
      - `IBLOCK`
        - number of the block containing the inode
    - <u>Data Structures</u> in `fs.h`
      - `struct superblock`
        - describes the disk layout
      - `struct dinode`
        - <u>on-disk inode structure</u>
        - `short type`: 
          - distinguishes between files, directories, and special files (devices)
          - a type of zero indicates that an on-disk inode is free
        - ...
        - `short nlink`: 
          - number of directory entries that refer to this inode
          - in order to recognize when the on-disk inode and its data blocks should be freed
        - `uint size`: size of file (bytes)
        - `uint addrs[NDIRECT+1]`: 
          - block numbers of the disk blocks holding the file’s content
          - xv6: 12 direct blocks and 1 indirect block
      - `struct dirent`
        - <u>a directory's data is a sequence of `dirent`</u>
        - contains a name and an inode number
    - <u>Data Structures</u> in `file.h`
      - `struct inode`
        - <u>in-memory copy of a `struct dinode` on disk</u>
        - ...
        - `int ref`: 
          - the number of C pointers referring to the in-memory inode
          - the kernel discards the inode from memory if the reference count drops to zero
        - ...
    - <u>Data Structures</u> in `fs.c`
      - `struct {} icache`
        - `struct spinlock lock`
        - `struct inode inode[NINODE]`
    - <u>Locks</u> in `fs.c`
      - `icache.lock`
        - the invariant that an inode is present in the cache at most once, and 
        - the invariant that a cached inode’s `ref` field counts the number of in-memory pointers to the cached inode
    - <u>Locks</u> in `file.h`
      - inode's `ip->lock` (`struct sleeplock`)
        - serializes operations on each inode and its content
        - ensures exclusive access to the inode’s fields (such as file length) as well as to the inode’s file or directory content blocks
  - **FILE DISCRIPTOR**
    - <u>Functions</u> in `file.c`
      - `filealloc`
        - scans the file table for an unreferenced file (`f->ref == 0`) and returns a new reference
      - `filedup`
        - incre- ments the reference count
      - `fileclose`
        - decrement ref count, close when reaches 0
      - `filestat`
        - <u>get metadata about file</u>
        - calls `stati()`
        - ...
      - `fileread`
        - check that the operation is allowed by the open mode and then 
        - pass the call through to either the pipe or inode implementation
        - use the I/O offset as the offset for the operation and then advance it
      - `filewrite`
        - ...
        - loops to write `MAXOPBLOCKS` blocks to disk each time
        - ...
    - <u>Functions</u> in `sysfile.c`
      - `fdalloc`
        - allocate a file descriptor for the given file
    - <u>Data Structures</u> in `file.c`
      - `struct {} ftable`
        - all the open files in the system are kept in this global file table
        - `struct spinlock lock`
        - `struct file file[NFILE]`
    - <u>Data Structures</u> in `file.h`
      - `struct file`
        - represents an open file
        - a wrapper around either an inode or a pipe, plus an I/O offset
        - `int ref`: tracks the number of references to a particular open file
        - `readable`, `writable`
      - `struct devsw`
        - map major device number to device read/write functions
    - <u>Data Structures</u> in `proc.h`
      - `struct proc {struct file *ofile[NOFILE]}`
        - an array of open files in each process, with index as <u>file descriptor</u>
  - <u>Functionality</u>
    - User Disk Read
      - `sys_read()` -> `fileread()` -> `readi()` 
        - -> `bread()`
          - -> `bget()`
          - ··> `virtio_disk_rw()`
        - -> `either_copyout()`
    - User Disk Write
      - `sys_write()` -> `filewrite()` 
        - -> `beginop()`
        - -> `writei()`
          - -> `bread()`
          - -> `either_copyout()`
          - -> `log_write()`
        - -> `endop()`
          - -> `commit()`
            - -> `write_log()` -> `bwrite()`
            - -> `write_head()` -> `bwrite()`
            - -> `install_trans()` -> `bwrite()`
            - -> `write_head()` -> `bwrite()`
- **FILE-SYSTEM SYSCALL**
  - <u>Functions</u> in `sysfile.c`
    - `sys_link`
      - create the path `new` as a link to the same inode as `old`
    - `create`
      - creates a new name for a new inode
      - is a generalization of the three file creation system calls: 
        - `open` with the O_CREATE flag makes a new ordinary file, 
        - `mkdir` makes a new directory, and 
        - `mkdev` makes a new device file
      - ...
    - `sys_open`
    - `sys_mknod`
      - makes a directory entry and corresponding inode for a special file
      - ...
- **PIPE**
  - <u>Data Structures</u> in `pipe.c`
  	- `struct pipe`
  		- `char data[PIPESIZE]`: data buffer (wraps around)
  		- `uint nread`: number of bytes read (doesn't wrap)
  		- `uint nwrite`: number of bytes written (doesn't wrap)
  		- ...
  - <u>Functions</u> in `pipe.c`
  	- `pipewrite`
  		- `acquire(&pi->lock)`
  		- loops over the bytes being written (`addr[0..n-1]`)
  			- calls `copyin()`
  			- copys to `pi->data` buffer
  			- if buffer fills
  				- `wakeup()`, wakeup sleeping readers
  				- `sleep()`, which will release `&pi->lock`
  	- `piperead`
  		- `acquire(&pi->lock)`
  		- if nothing to read, `sleep()`, which will release the lock
  		- else, loops to read from `pi->data` buffer
  			- calls `copyout()`
  		- `wakeup()`: wakeup sleeping writers
  		- `release(&pi->lock)`
  - <u>Locks</u> in `pipe.c`
  	- pipe's `pi->lock`
  		- serializes operations on each pipe

### **NETWORK**

- <u>Functions</u> in `net.c`
  - <u>networking protocol support (IP, UDP, ARP, etc.).</u>
  - `mbufalloc`: Allocates a packet buffer.
- <u>Data Structures</u> in `net.h`
  - <u>packet buffer management</u>
    - `struct mbuf`: buffer
    - ...
  - ...
- `sysnet.c`
  - network system calls.
- <u>Data Structures</u> in `e1000_dev.h`
  - <u>E1000 hardware definitions: registers and DMA ring format.</u>
    - `struct rx_desc`: descriptor format
- `e1000.c`
  - `e1000_init`
    - ...
    - allocates `mbuf` packet buffers for the E1000 to DMA into, using `mbufalloc()`, and sets descriptor ring...
    - ...
- `pci.c`
  - simple PCI-Express initialization, only works for qemu and its e1000 card.
  - `pci_init`
    - ...
    - calls `e1000_init()`













