### 准备工作

> Before you start coding, read Chapter 2 of the [xv6 book](https://pdos.csail.mit.edu/6.S081/2021/xv6/book-riscv-rev1.pdf), and Sections 4.3 and 4.4 of Chapter 4, and related source files:
>
> - The user-space code for systems calls is in `user/user.h` and `user/usys.pl`.
> - The kernel-space code is `kernel/syscall.h` , ` kernel/syscall.c`.
> - The process-related code is `kernel/proc.h` and `kernel/proc.c`.



为了方便阅读，按照这段话的顺序来看源码：

> 在 `main`  ( `kernel/main.c:11` )初始化完一些设备和子系统之后，它调用 `userinit`  ( `kernel/proc.c:212` )，创建了第一个进程。第一个进程执行了一小段用 RISC-V 汇编语言编写的程序， `initcode.S` ( `user/initcode.S:1` )，通过调用 `exec` 系统调用回到内核。正如我们在第一章所看到的， `exec` 使用一段新程序（在这里是 `init` ）替换当前进程的内存和寄存器。当内核执行完 `exec` ，它返回到用户空间， `init` 进程中。如果需要， `init` ( `user/init.c:15` )创建一个新的控制台设备文件，然后将其打开为文件描述符 0 ，1 和 2 ，接着在控制台启动一个 `shell` 。系统到这里就启动了。



#### kernel/main.c

```c
#include "types.h"
#include "param.h"
#include "memlayout.h"
#include "riscv.h"
#include "defs.h"

volatile static int started = 0;

// start() jumps here in supervisor mode on all CPUs.
void
main()
{
  if(cpuid() == 0){
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode table
    fileinit();      // file table
    virtio_disk_init(); // emulated hard disk
    userinit();      // first user process
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    kvminithart();    // turn on paging
    trapinithart();   // install kernel trap vector
    plicinithart();   // ask PLIC for device interrupts
  }

  scheduler();        
}

```

> 这段代码是操作系统内核的 `main` 函数，它在所有 CPU 上以管理模式运行。它首先检查当前 CPU 的编号，如果是 0，则执行一系列初始化操作，包括初始化控制台、物理页面分配器、内核页表、进程表、陷阱向量、中断控制器、缓冲区缓存、inode 表、文件表和虚拟硬盘等。然后，它创建第一个用户进程并启动调度器。如果当前 CPU 的编号不为 0，则它会等待 CPU 0 完成初始化操作，然后执行一些初始化操作，包括开启分页、安装内核陷阱向量和请求设备中断等。最后，它启动调度器。



#### kernel/proc.c

```c
// 仅截取 userinit 这部分
...
// Set up first user process.
void
userinit(void)
{
  struct proc *p;

  p = allocproc();
  initproc = p;
  
  // allocate one user page and copy init's instructions
  // and data into it.
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;

  release(&p->lock);
}
...
```



#### kernel/proc.h

> 代码太长就不粘贴了。
>
> 这个头文件主要定义了四个结构体：`context` 、`cpu` 、`trapframe` 、`proc`
>
> `context` 结构体用于保存内核上下文切换的寄存器。
>
> `cpu` 结构体表示每个 CPU 的状态，包括在该 CPU 上运行的进程、用于切换到调度程序的上下文以及与中断相关的字段。
>
> `trapframe` 结构体用于保存用户态陷阱处理代码所需的每个进程数据。它位于用户页表中 trampoline 页下方的单独一页中，但未在内核页表中特殊映射。它包括了被调用者保存的用户寄存器，例如 s0-s11，因为通过 usertrapret() 返回用户态的路径不会返回整个内核调用栈。
>
> `proc` 结构体表示每个进程的状态，包括进程锁、进程状态、睡眠通道、是否被杀死、退出状态、进程 ID、父进程、内核栈虚拟地址、进程内存大小、用户页表、trapframe 数据页、上下文、打开文件、当前目录和进程名称等字段。
>
> 补充：
>
> 内核上下文切换指的是在内核态中，从一个进程的内核执行上下文切换到另一个进程的内核执行上下文。这与用户态和内核态之间的切换不同，后者指的是从用户态进入内核态（例如，通过系统调用或中断），或从内核态返回用户态。
>
> 在内核上下文切换过程中，操作系统会保存当前进程的内核执行状态（例如，寄存器值），并恢复另一个进程先前保存的内核执行状态，以便在该进程的内核执行上下文中继续运行。这通常发生在调度程序选择新的进程运行时。
>
> 进程调度是操作系统内核中负责决定哪个进程获得 CPU 时间的组件。当调度程序选择新的进程运行时，它会执行内核上下文切换，以便在新的进程的内核执行上下文中继续运行。也就是说内核上下文切换是进程调度的一部分。



#### kernel/defs.h

> 代码太长就不粘贴了。
>
> 这个头文件主要是声明了一些操作系统内核的结构体和函数原型。



#### kernel/entry.S

> 这段汇编代码是在设置每个 CPU 的栈并跳转到 C 语言的 `start` 函数。首先，它将栈指针 `sp` 设置为 `stack0` 的地址，`stack0` 是在 `start.c` 中声明的一个变量，表示每个 CPU 的栈的起始地址。然后，它将寄存器 `a0` 设置为每个栈的大小（1024 * 4 字节）。接着，它读取当前 CPU 的编号（`hartid`），并将其加 1，然后将其乘以每个栈的大小，并将结果加到栈指针上。最后，它调用 C 语言的 `start` 函数，并在返回后无限循环。



> 



#### user/user.h

```c
// 头文件。包含了一些结构体定义和函数声明。
// 函数声明是一些系统调用和库函数

struct stat;     //结构体 `struct stat` 用来存储文件的属性信息，比如大小、时间、权限等
struct rtcdate;  //结构体 `struct rtcdate` 用来存储实时时钟的日期和时间

// system calls
int fork(void);
int exit(int) __attribute__((noreturn));
int wait(int*);
int pipe(int*);
int write(int, const void*, int);
int read(int, void*, int);
int close(int);
int kill(int);
int exec(char*, char**);
int open(const char*, int);
int mknod(const char*, short, short);
int unlink(const char*);
int fstat(int fd, struct stat*);
int link(const char*, const char*);
int mkdir(const char*);
int chdir(const char*);
int dup(int);
int getpid(void);
char* sbrk(int);
int sleep(int);
int uptime(void);

// ulib.c
int stat(const char*, struct stat*);
char* strcpy(char*, const char*);
void *memmove(void*, const void*, int);
char* strchr(const char*, char c);
int strcmp(const char*, const char*);
void fprintf(int, const char*, ...);
void printf(const char*, ...);
char* gets(char*, int max);
uint strlen(const char*);
void* memset(void*, int, uint);
void* malloc(uint);
void free(void*);
int atoi(const char*);
int memcmp(const void *, const void *, uint);
void *memcpy(void *, const void *, uint);
```



#### user/usys.pl

```assembly
#!/usr/bin/perl -w

# Generate usys.S, the stubs for syscalls.

print "# generated by usys.pl - do not edit\n";

print "#include \"kernel/syscall.h\"\n";

sub entry {
    my $name = shift;
    print ".global $name\n";
    print "${name}:\n";
    print " li a7, SYS_${name}\n";
    print " ecall\n";
    print " ret\n";
}
	
entry("fork");
entry("exit");
entry("wait");
entry("pipe");
entry("read");
entry("write");
entry("close");
entry("kill");
entry("exec");
entry("open");
entry("mknod");
entry("unlink");
entry("fstat");
entry("link");
entry("mkdir");
entry("chdir");
entry("dup");
entry("getpid");
entry("sbrk");
entry("sleep");
entry("uptime");
```

> 1. `#!/usr/bin/perl -w`: 这是一个Shebang行，指定了用于解释和执行该脚本的Perl解释器。
> 2. `print "# generated by usys.pl - do not edit\n";`: 打印一条注释，表示该文件是由usys.pl生成的，不应该手动编辑。
> 3. `print "#include \"kernel/syscall.h\"\n";`: 打印一条#include指令，包含了"kernel/syscall.h"头文件，该头文件可能定义了一些系统调用相关的常量或函数。
> 4. `sub entry { ... }`: 定义了一个名为"entry"的子例程（subroutine），用于生成每个系统调用的存根代码。
> 5. `my $name = shift;`: 将传入的参数（系统调用的名称）存储在名为$name的变量中。shift函数用于获取参数列表中的第一个参数，并将其从列表中移除。
> 6. `print ".global $name\n";`: 打印一个汇编指令，将$name标记为一个全局符号，使其可在其他文件中访问。
> 7. `print "${name}:\n";`: 打印一个标签（label），以系统调用的名称命名。
> 8. `print " li a7, SYS_${name}\n";`: 打印一个汇编指令，将系统调用的编号（由"SYS_"前缀加上系统调用名称组成的常量）加载到寄存器a7中。这个寄存器通常用于存储系统调用的编号。
> 9. `print " ecall\n";`: 打印一个汇编指令，执行一个系统调用。ecall指令用于触发异常，从而进入操作系统的系统调用处理程序。
> 10. `print " ret\n";`: 打印一个汇编指令，表示返回到调用者。
> 11. `entry("fork");`, `entry("exit");`, ... : 调用"entry"子例程，并传递每个系统调用的名称作为参数。这些调用生成了每个系统调用的存根代码。

```
# generated by usys.pl - do not edit
#include "kernel/syscall.h"
.global fork
fork:
 li a7, SYS_fork
 ecall
 ret
.global exit
exit:
 li a7, SYS_exit
 ecall
 ret
 
# ...省略了剩下的
#
```



