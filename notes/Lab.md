# <u>lab. Xv6 and Unix utilities</u>

- 使用系统调用增加用户程序
	- xv6的库函数有限，在 `user.h` 中列出
- 增加用户程序 user program
	- `user/programName.c`
		- 主函数为 `int main(int argc, char *argv[]) {}`
			- 最后需调用 `exit()` 来退出程序
	- Makefile 内增加 `$U/_programName`
- 系统调用
	- `pipe()` 生成两个端口，`fork()` 后在父进程和子进程中同时存在四个端口，都需要关闭
		- 通过 `read` 和 `write` 来进行 `pipe` 的读写
	- `read()` 在 `pipe` 的写入端关闭后返回 0
- 算法
	- concurrent prime:    <img src="image.assets/sieve.gif" alt="sieve"  />



# <u>Lab: system calls</u>

- 增加系统调用函数

- 增加用户程序的系统调用

	- `user/user.h` 头文件内增加需要调用的原型（prototype，包含函数和结构体等，其他程序可以调用）

	- `user/usys.pl` perl文件内增加桩（stub）`entry("syscallName");`

	- `kernel/syscall.h` 头文件内定义系统调用的序号 `#define SYS_syscallName number`

	- Makefile 引用 `usys.pl` 生成真正的系统调用桩 `usys.S`，使用 `ecall` 来过渡到 Kernel

		- ```assembly
			.global syscallName
			syscallName:
			   li a7, SYS_syscallName
			   ecall
			   ret
			```

			

- 增加系统调用 syscall

	- kernel 对应的c文件中增加函数 `sys_syscallName()`
	- `kernel/syscall.c` 中的 `syscall()` 函数是真正执行系统调用的位置
		- 通过一个数组完成从系统调用的序号到实际函数的映射（而非hashMap）
	- `kernel/defs.h` 头文件内增加需要调用的原型



# <u>Lab: Page Table</u>

- 使每个进程使用单独的核页表kernel page table
	- 处理核页表过程中高度关注其固有结构：课本Figure 3.3
	- 了解进程的页表结构
		- 了解页表（虚拟内存）的机制（如satp），结构（如walk），初始化（如kvminit），增删等
			- 了解页表项PTE的初始化，增删，权限调整等
			- 了解物理内存的结构（如freelist），分配（kalloc），清理（kfree）等



# <u>Lab: Traps</u>

- 理解了trap和process的运行结构后就是很简单的一个lab



# <u>Lab: Lazy Allocation</u>

- sbrk时不增加实际的物理内存，只增加进程的大小，在实际使用中碰到pagefault判断是该情况后增加物理内存及其映射，并返回程序继续执行
	- 修改sbrk相关内存分配的代码
	- 修改usertrap中该情况部分的代码，分配物理内存
		- 注意分配出问题后及时释放物理内存
	- 修改uvmunmap，使得遇到未初始化的页表不报错
		- 修改后就不需要修改其他进程清理的代码，如freeproc, proc_freepagetable等
	- 修改fork，使得调用uvmcopy后遇到未初始化的页表不报错
	- 修改walkaddr，使得调用syscall调用sbrk增加的未分配内存时（如读写）和pagefault的处理方式相同



# <u>Lab: Copy-on-Write Fork for xv6</u>

## *Thinking*

- fork时子进程只复制页表及其映射，不复制物理内存，并将写入权限去除，改为cow标记；如此在任何相关进程需要写入时触发page fault，通过cow标记进行判断进而分配物理内存。
	- fork
		- 修改uvmcopy，不复制内存，并处理标记
		- 修改usertrap，检查是否为cow，分配内存，映射，并处理标记
		- 增加物理内存映射数量的记录，当映射数量为0时清理该物理内存页
	- kernel mode
		- copyout使用user提供的va查找物理地址并写入，由于kernel拥有写入权限，因此需要确定va指向的pa没有cow标记
	- 细节问题
		- 判断是否超过MAXVA
		- 判断是否在stack的guard page中



# <u>Lab: Multithreading</u>

## *Thinking*

- 第一个问题练习了线程转换的基本逻辑，有两个关键点
	- 在scheduler中调用一个汇编语言的程序switch来存储和载入context
	- 一个进程第一次运行之前将context的ra和sp伪装成需要执行的函数，如此scheduler第一次执行时就会开始执行该函数
- 第二个问题练习了多线程中使用锁保护共享的数据结构
- 第三个问题练习了coordination，即sleep/wakeup（wait/broadcast）的逻辑，对锁的使用比较关键
	- sleep本身会释放一个锁（释放之前会获得进程锁），而sleep之前对状态的更新需要首先获得这个锁
	- 如果没有调用sleep，或sleep返回（sleep被wakeup后会再获得该锁），则最终释放这个锁



# <u>Lab: locks</u>

- 这个lab主要练习lock的使用，同时熟悉物理内存和硬盘缓存的机制
- 通过将物理内存和硬盘块分组，各组拥有各自的lock，从而减少contention
  - 前者通过各cpu拥有单独freelist来完成
  - 后者通过将硬盘块block分组（通过hashtable）来完成
- coding和debugging的步骤
  - baby step：将任务尽量细分为可测试的小部分，每项完成后进行<u>测试</u>和<u>版本保存</u>，例如
    1. 先调整数据结构，而不调整锁
    2. 增加锁单不删除全局锁
    3. 尽可能删除全局锁
  - 打印信息帮助调试
  - 科学方法进行调试（10分钟法则）



# <u>Lab: file system</u>

## *Thinking*

- lab
  - 使用baby step的方法，确实非常有效，每一步都做测试，保证该部分没有问题，不至于陷入过于混乱的状态

### FILE SYSTEM

- Buffer Cache
  - 对硬盘的所有读写都从这里进行，相当于CPU和硬盘之间的proxy
    - 同步访问磁盘块，以确保内存中只有一个块的副本，并且每次只有一个内核线程使用该副本
    - 缓存常用的区块，这样它们就不需要从慢速磁盘上重新读取
  - 数据结构
    - 通过一个array来储存所有buffer
    - 通过一个linked list来管理数组中的buffer，顺序为LRU
  - 读写
    - 写操作bwrite（buffer中的内容写入disk）由log调用
    - 读操作bread视情况从硬盘读取数据到新的buffer中
- Logging
  - 对硬盘的写操作都从这里进行，相当于Buffer写入到硬盘的proxy（用log_write取代bwrite）
- Inode
  - 包含文件的metadata（类型，大小等）和数据块的地址，硬盘和内存中都存在
  - 数据结构
    - 硬盘中有特定的blocks来存储硬盘中的inodes
    - 通过一个array来储存所有内存中的inodes
  - 读写
    - readi/writei通过buffer cache和loggin的读写函数实现完整的读写inode的功能
  - Directory
    - 特定类型的inode
    - 数据结构
      - 在inode对应的数据块中存储特定结构的数据，该结构包含inode的编号和该子目录的名称
      - 实际上是一个树状递归结构
- File Descripor
  - file是统一系统资源（硬盘、console、其他设备，以及pipe等）的媒介
    - 实际是一个包裹inode或其他资源的wrapper，因此同一个资源可以有多个wrapper
  - 数据结构
    - 通过一个array来存储所有file结构
    - 每个process通过一个array来存储该process打开的file及其file descriptor
  - 读写
    - fileread/filewrite判断不同类型决定不同的读写函数







