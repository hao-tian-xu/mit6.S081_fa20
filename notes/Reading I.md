## **Chapter 1 Operating system interfaces**

#### Figure 1.2: Xv6 system calls

If not otherwise stated, these calls return 0 for no error, and -1 if there’s an error.

| System call                             | Description                                                  |
| :-------------------------------------- | :----------------------------------------------------------- |
| `int fork()`                            | Create a process, return child's PID.                        |
| `int exit(int status)`                  | Terminate the current process; status reported to `wait()`. No return. |
| `int wait(int *status)`                 | Wait for a child to exit; exit status in `*status`; returns child PID. |
| `int kill(int pid)`                     | Terminate process PID. Returns `0`, or `-1` for error.       |
| `int getpid()`                          | Return the current process's PID.                            |
| `int sleep(int n)`                      | Pause for `n` clock ticks.                                   |
| `int exec(char *file, char *argv[])`    | Load a file and execute it with arguments; only returns if error. |
| `char *sbrk(int n)`                     | Grow process's memory by `n` bytes. Returns start of new memory. |
| `int open(char *file, int flags)`       | Open a file; flags indicate read/write; returns an `fd` (file descriptor). |
| `int write(int fd, char *buf, int n)`   | Write `n` bytes from `buf` to file descriptor `fd`; returns `n`. |
| `int read(int fd, char *buf, int n)`    | Read `n` bytes into `buf`; returns number read; or `0` if end of file. |
| `int close(int fd)`                     | Release open file `fd`.                                      |
| `int dup(int fd)`                       | Return a new file descriptor referring to the same file as `fd`. |
| `int pipe(int p[])`                     | Create a pipe, put read/write file descriptors in `p[0]` and `p[1]`. |
| `int chdir(char *dir)`                  | Change the current directory.                                |
| `int mkdir(char *dir)`                  | Create a new directory.                                      |
| `int mknod(char *file, int, int)`       | Create a device file.                                        |
| `int fstat(int fd, struct stat *st)`    | Place info about an open file into `*st`.                    |
| `int stat(char *file, struct stat *st)` | Place info about a named file into `*st`.                    |
| `int link(char *file1, char *file2)`    | Create another name (`file2`) for the file `file1`.          |
| `int unlink(char *file)`                | Remove a file.                                               |

#### THE JOB OF AN OPERATING SYSTEM

- The job of an operating system is to 
  - share a computer among multiple programs and to 
  - provide a more useful set of services than 
  - the hardware alone supports
- An operating system manages and abstracts the low-level hardware, 
  - so that, for example, a word processor need not concern itself with which type of disk hardware is being used. 
- An operating system shares the hardware among multiple programs 
  - so that they run (or appear to run) at the same time. 
- Finally, operating systems provide controlled ways for programs to interact, 
  - so that they can share data or work together.

#### INTERFACE

- An operating system provides services to user programs through an interface.
  - On the one hand, we would like the interface to be simple and narrow because that makes it easier to get the implementation right. 
  - On the other hand, we may be tempted to offer many sophisticated features to applications.
- The trick in resolving this tension is to design interfaces that 
  - rely on a few mechanisms that 
  - can be combined to provide much generality.

#### xv6

- *kernel*
  - a special program that provides services to running programs.
- *process*
  - a running program that has memory containing instructions, data, and a stack.
    - The instructions implement the program’s computation. 
    - The data are the variables on which the computation acts. 
    - The stack organizes the program’s procedure calls.
- *system call*
  - when a process needs to invoke a kernel service, it invokes one of the calls in the operating system’s interface.
  - a process alternates between executing in *user space* and *kernel space*.
- *interface*
  - The collection of system calls that a kernel provides is the interface that user programs see.
- *services*
  - processes, memory, file descriptors, pipes, and a file system

### 1.1 Processes and memory

- consists of
  - user-space memory (instructions, data, and stack) and 
  - per-process state private to the kernel
- Xv6 *time-share*s processes:
  - it transparently switches the available CPUs among the set of processes waiting to execute
    - When a process is not executing, xv6 saves its CPU registers, restoring them when it next runs the process. 
- PID
  - The kernel associates a process identifier, or PID, with each process.
- Syscalls
  - `fork`
    - A process may create a new process using the fork system call
  - `exit`
    - causes the calling process to stop executing and to release resources such as memory and open files.
  - `wait`
    - returns the PID of an exited (or killed) child of the current process and copies the exit status of the child to the address passed to wait; 
    - if none of the caller’s children has exited, wait waits for one to do so. 
    - If the caller has no children, wait immediately returns -1. 
    - If the parent doesn’t care about the exit status of a child, it can pass a 0 address to wait.
  - `exec`
    - replaces the calling process’s memory with a new memory image loaded from a file stored in the file system. 
    - The file must have a particular format, which specifies which part of the file holds instructions, which part is data, at which instruction to start, etc.
      - xv6 uses the ELF format
    - When exec succeeds, it does not return to the calling program; instead, the instructions loaded from the file start executing at the entry point declared in the ELF header.
  - `sbrk`
    - A process that needs more memory at run-time (perhaps for malloc) can call sbrk(n) to grow its data memory by n bytes; sbrk returns the location of the new memory.

### 1.2 I/O and File descriptors

- *file descriptor*
  - a small integer representing a kernel-managed object that a process may read from or write to.
  - A process may obtain a file descriptor by opening a file, directory, or device, or by creating a pipe, or by duplicating an existing descriptor.
  - abstraction
    - the file descriptor interface abstracts away the differences between files, pipes, and devices, making them all look like streams of bytes.
  - implementation
    - the xv6 kernel uses the file descriptor as an index into a per-process table
  - convention
    - a process reads from file descriptor 0 (standard input), 
    - writes output to file descriptor 1 (standard output), and 
    - writes error messages to file descriptor 2 (standard error)
- Syscalls
  - `read`, `write`
    - read bytes from and write bytes to open files named by file descriptors
    - Each file descriptor that refers to a file has an offset associated with it.
  - `close`
    - releases a file descriptor, making it free for reuse by a future `open`, `pipe`, or dup system call (see below)
  - `open`
    - consists of a set of flags, expressed as bits, that control what open does
    - The possible values are defined in the file control (fcntl) header
      - `O_RDONLY`, `O_WRONLY`, `O_RDWR`, `O_CREATE`, and `O_TRUNC`, which instruct open to open the file for reading, or for writing, or for both reading and writing, to create the file if it doesn’t exist, and to truncate the file to zero length.
  - `dup`
    - duplicates an existing file descriptor, returning a new one that refers to the same underlying I/O object.
    - Both file descriptors share an offset, just as the file descriptors duplicated by fork do.
- User Programs
  - `cat`
    - The use of file descriptors and the convention that file descriptor 0 is input and file descriptor 1 is output allows a simple implementation of `cat`.
  - I/O redirection
    - The system call `exec` replaces the calling process’s memory but preserves its file table.
    - This behavior allows the shell to implement *I/O redirection* by forking, reopening chosen file descriptors in the child, and then calling `exec` to run the new program.

### 1.3 Pipes

- *pipe*
  - a small kernel buffer exposed to processes as a pair of file descriptors, one for reading and one for writing
  - Pipes provide a way for processes to communicate.
- Pipes have at least four advantages over temporary files
  - First, pipes automatically clean themselves up; with the file redirection, a shell would have to be careful to remove /tmp/xyz when done. 
  - Second, pipes can pass arbitrarily long streams of data, while file redirection requires enough free space on disk to store all the data. 
  - Third, pipes allow for parallel execution of pipeline stages, while the file approach requires the first program to finish before the second starts. 
  - Fourth, if you are implementing inter-process communication, pipes’ blocking reads and writes are more efficient than the non-blocking semantics of files.

### 1.4 File system

- The xv6 file system 
  - provides data files, which contain uninterpreted byte arrays, and directories, which contain named references to data files and other directories.
- the same underlying file, called an *inode*, can have multiple names, called *links*
  - *inode*
    - An inode holds *metadata* about a file, including its type (file or directory or device), its length, the location of the file’s content on disk, and the number of links to a file.
  - *link*
    - Each link consists of an entry in a directory; the entry contains a file name and a reference to an inode
- Syscalls
  - `mkdir`
    - creates a new directory
  - `chdir`
    - change the calling process’s *current directory*
  - `open`
    - with the `O_CREATE` flag creates a new data file
  - `mknod`
    - creates a special file that refers to a device
    - Associated with a device file are the major and minor device numbers (the two arguments to mknod), which uniquely identify a kernel device.
    - When a process later opens a device file, the kernel diverts read and write system calls to the kernel device implementation instead of passing them to the file system.
  - `fstat`
    - retrieves information from the inode that a file descriptor refers to.
    - It fills in a `struct stat`, defined in `stat.h`
  - `link`
    - creates another file system name referring to the same inode as an existing file.
  - `unlink`
    - removes a name from the file system
    - The file’s inode and the disk space holding its content are only freed when the file’s link count is zero and no file descriptors refer to it.

### Summary

- Any operating system must 
  - multiplex processes onto the underlying hardware, 
  - isolate processes from each other, and 
  - provide mechanisms for controlled inter-process communication.



## **Chapter 2 Operating system organization**

- to support several activities at once, an operating system must fulfill three requirements:
  - multiplexing
    - The operating system must *time-share* the resources of the computer among these processes.
  - isolation
    - That is, if one process has a bug and malfunctions, it shouldn’t affect processes that don’t depend on the buggy process.
  - interaction
    - Complete isolation, however, is too strong, since it should be possible for processes to intentionally interact; pipelines are an example.
- The software running in kernel space (or in supervisor mode) is called the *kernel*.

### 2.1 Abstracting physical resources

- **Alternative Organization**: one could implement the system calls as a library, with which applications link.
  - **Pros**
    - In this plan, each application could even have its own library tailored to its needs. 
    - Applications could directly interact with hardware resources and use those resources in the best way for the application
  - Some operating systems for <u>*embedded devices*</u> or <u>*real-time systems*</u> are organized in this way.
  - **Cons**: if there is more than one application running, the applications must be well-behaved
    - each application must periodically give up the CPU so that other applications can run. 
    - Such a *cooperative* time-sharing scheme may be OK if all applications trust each other and have no bugs.
- **Kernel: Strong Isolation**
  - It’s more typical for applications to not trust each other, and to have bugs, so one often wants stronger isolation than a cooperative scheme provides.
  - To achieve strong isolation it’s helpful to forbid applications from directly accessing sensitive hardware resources, and instead to <u>*abstract the resources into services*</u>.
  - **Examples**
    - Unix applications interact with storage only through the file system’s `open`, `read`, `write`, and `close` system calls, instead of reading and writing the **disk** directly.
    - Unix transparently switches **hardware CPUs** among processes, saving and restoring register state as necessary, so that applications don’t have to be aware of time sharing.
      - This transparency allows the operating system to share CPUs even if some applications are in <u>*infinite loops*</u>.
    - Unix processes use exec to build up their memory image, instead of directly interacting with **physical memory**.
      - This allows the operating system to decide where to place a process in memory; if memory is tight, the operating system might even <u>*store some of a process’s data on disk*</u>.
    - Many forms of **interaction among Unix processes** occur via <u>*file descriptors*</u>.
  - The system-call interface is carefully designed to provide both programmer convenience and the possibility of strong isolation.

### 2.2 User mode, supervisor mode, and system calls

- To achieve strong isolation, the operating system must arrange that 
  - applications cannot modify (or even read) the operating system’s data structures and instructions and that 
  - applications cannot access other processes’ memory.
- CPUs provide **hardware support** for strong isolation.
  - CPUs provide hardware support for strong isolation. For example, RISC-V has three modes in which the CPU can execute instructions: *machine mode*, *supervisor mode*, and *user mode*. 
- In **supervisor mode** the CPU is allowed to execute *privileged instructions*
  - for example, *enabling and disabling interrupts*, *reading and writing the register* that holds the address of a page table, etc.
  - The software in supervisor mode can also execute privileged instructions and is said to be running in ***kernel space***. The software running in kernel space (or in supervisor mode) is called the ***kernel***.
- CPUs provide a special instruction that switches the CPU from user mode to supervisor mode and enters the kernel at an entry point specified by the kernel. 
  - RISC-V provides the `ecall` instruction for this purpose.
  - An application that wants to invoke a kernel function must transition to the kernel.

### 2.3 Kernel organization

- A key design question is what part of the operating system should run in supervisor mode.
- **Monolithic kernel**
  - One possibility is that the entire operating system resides in the kernel, so that the implementations of all system calls run in supervisor mode.
  - Pros
    - It is easier for different parts of the operating system to cooperate.
  - Cons
    - Interfaces between different parts of the operating system are often complex
    - A mistake is fatal, because an error in supervisor mode will often cause the kernel to fail.
- **Microkernel**
  - Minimize the amount of operating system code that runs in supervisor mode, and execute the bulk of the operating system in user mode.
  - This organization allows the kernel to be relatively simple, as most of the operating system resides in user-level servers.
- Xv6 is implemented as <u>*a monolithic kernel, like most Unix operating systems*</u>.

### 2.4 Code: xv6 organization

#### Figure 2.2: Xv6 kernel source files

| File             | Description                                          |
| ---------------- | ---------------------------------------------------- |
| `bio.c `         | Disk block cache for the file system.                |
| `console.c`      | Connect to the user keyboard and screen.             |
| `entry.S `       | Very first boot instructions.                        |
| `exec.c `        | `exec()` system call.                                |
| `file.c `        | File descriptor support.                             |
| `fs.c `          | File system.                                         |
| `kalloc.c `      | Physical page allocator.                             |
| `kernelvec.S `   | Handle traps from kernel, and timer interrupts.      |
| `log.c `         | File system logging and crash recovery.              |
| `main.c `        | Control initialization of other modules during boot. |
| `pipe.c `        | Pipes.                                               |
| `plic.c `        | RISC-V interrupt controller.                         |
| `printf.c `      | Formatted output to the console.                     |
| `proc.c `        | Processes and scheduling.                            |
| `sleeplock.c `   | Locks that yield the CPU.                            |
| `spinlock.c `    | Locks that don’t yield the CPU.                      |
| `start.c `       | Early machine-mode boot code.                        |
| `string.c `      | C string and byte-array library.                     |
| `swtch.S `       | Thread switching.                                    |
| `syscall.c `     | Dispatch system calls to handling function.          |
| `sysfile.c `     | File-related system calls.                           |
| `sysproc.c `     | Process-related system calls.                        |
| `trampoline.S `  | Assembly code to switch between user and kernel.     |
| `trap.c C `      | code to handle and return from traps and interrupts. |
| `uart.c `        | Serial-port console device driver.                   |
| `virtio_disk.c ` | Disk device driver.                                  |
| `vm.c `          | Manage page tables and address spaces.               |



### 2.5 Process overview

- ***process***
  - The **unit of isolation** in xv6 (as in other Unix operating systems)
  - prevents one process from wrecking or spying on another process’s memory, CPU, file descriptors, etc. It also prevents a process from wrecking the kernel itself, so that a process can’t subvert the kernel’s isolation mechanisms
- **Mechanisms**
  - **user/supervisor mode flag**
  - To help enforce isolation, the process abstraction <u>*provides the illusion to a program that it has its own private machine*</u>.
    - **address spaces**
      - A process provides a program with what <u>*appears to be a private memory system*</u>, or *address space*, which other processes cannot read or write.
      - Xv6 uses page tables (which are implemented by hardware) to give each process its own address space.
        - ​	<img src="image.assets/Screen Shot 2022-05-05 at 14.04.33.png" alt="Screen Shot 2022-05-05 at 14.04.33" style="zoom:25%;" />
    - **time-slicing of threads**
      - A process also provides the program with what <u>*appears to be its own CPU*</u> to execute the program’s instructions.
      - **state**
        - The xv6 kernel maintains many pieces of state for each process, which it gathers into a `struct proc`. 
        - A process’s most important pieces of kernel state are 
          - its page table, 
          - its kernel stack, and 
          - its run state.
      - **thread of execution**
        - Each process has a thread of execution (or *thread* for short) that executes the process’s instructions.
        - A thread can be suspended and later resumed.
          - Much of the state of a thread is stored on the thread’s stacks. 
          - Each process has two stacks: a user stack and a kernel stack (p->kstack).

















