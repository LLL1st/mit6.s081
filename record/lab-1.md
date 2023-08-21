### 1 准备工作：

> 《xv6》的chapter1
>
> lesson1 - introduction and examples



### 2 实验

#### 2.1 sleep

##### Some hints

> - Before you start coding, read Chapter 1 of the [xv6 book](https://pdos.csail.mit.edu/6.S081/2021/xv6/book-riscv-rev2.pdf).
> - Look at some of the other programs in `user/` (e.g., `user/echo.c`, `user/grep.c`, and `user/rm.c`) to see how you can obtain the command-line arguments passed to a program.
> - If the user forgets to pass an argument, sleep should print an error message.
> - The command-line argument is passed as a string; you can convert it to an integer using `atoi` (see user/ulib.c).
> - Use the system call `sleep`.
> - See `kernel/sysproc.c` for the xv6 kernel code that implements the `sleep` system call (look for `sys_sleep`), `user/user.h` for the C definition of `sleep` callable from a user program, and `user/usys.S` for the assembler code that jumps from user code into the kernel for `sleep`.
> - Make sure `main` calls `exit()` in order to exit your program.
> - Add your `sleep` program to `UPROGS` in Makefile; once you've done that, `make qemu` will compile your program and you'll be able to run it from the xv6 shell.
> - Look at Kernighan and Ritchie's book *The C programming language (second edition)* (K&R) to learn about C.
>
> 源码阅读：
>
> ```c
> //	user/echo.c
> #include "kernel/types.h"
> #include "kernel/stat.h"
> #include "user/user.h"
> 
> int
> main(int argc, char *argv[]) ////argc表示命令行参数的数量，argv是一个指向参数字符串数组的指针
> {
>   int i;
> 
>   for(i = 1; i < argc; i++){ //循环遍历命令行参数，从索引1开始，因为0是程序的名字
>     write(1, argv[i], strlen(argv[i]));
>     if(i + 1 < argc){  //检查是否还有命令行参数未处理
>       write(1, " ", 1);  //还有的话，输出一个空格
>     } else {
>       write(1, "\n", 1);  //没有了的话，换行
>     }
>   }
>   exit(0);
> }
> ```
>
> ```c
> //	user/rm.c
> #include "kernel/types.h"
> #include "kernel/stat.h"
> #include "user/user.h"
> 
> int
> main(int argc, char *argv[]) //argc表示命令行参数的数量，argv是一个指向参数字符串数组的指针
> {
>   int i;
> 
>   if(argc < 2){
>     fprintf(2, "Usage: rm files...\n");  //用标准错误流输出错误消息
>     //fprintf中第一个参数是指向file类型的指针，表示输出的文件流
>     //文件描述符0：标准输入流，用于从键盘或其他输入设备读取输入
>     //文件描述符1：标准输出流，用于向屏幕或其他输出设备输出普通输出
>     //文件描述符2：标准错误流，用于输出错误消息和警告消息
>     exit(1);  //调用exit，终止程序的执行，返回状态码1，表示执行错误
>   }
> 
>   for(i = 1; i < argc; i++){ //循环遍历命令行参数，从索引1开始，因为0是程序的名字
>     if(unlink(argv[i]) < 0){  //argv[i]表示第i个命令行系数，即要删除的文件路径；unlink删除指定的文件
>       fprintf(2, "rm: %s failed to delete\n", argv[i]); //用标准错误流输出错误消息，并提示哪个文件删除失败
>       break;
>     }
>   }
> 
>   exit(0);  //调用exit，终止程序的执行，返回状态码0，表示执行成功
> }
> ```



##### solution

```c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char* argv[])
{
    if(argc < 2) {
        fprintf(2, "Usage: sleep ticks\n");
        exit(1);
    }
    int n = atoi(argv[1]);
    sleep(n);
    exit(0);
}
```





#### 2.2 pingpong

> The parent should send a byte to the child;
>
> the child should print "\<pid>: received ping" , write the byte on the pipe to the parent, and exit.
>
> the parent should read the byte from the child, print "\<pid>: received pong", and exit.

##### Some hints

> - Use `pipe` to create a pipe.
>- Use `fork` to create a child.
> - Use `read` to read from the pipe, and `write` to write to the pipe.
> - Use `getpid` to find the process ID of the calling process.
> - Add the program to `UPROGS` in Makefile.
> - User programs on xv6 have a limited set of library functions available to them. You can see the list in `user/user.h`; the source (other than for system calls) is in `user/ulib.c`, `user/printf.c`, and `user/umalloc.c`.



##### solution

> 用一个管道

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
   
int
main()
{
    int pingpong[2];
    pipe(pingpong);
    char buffer[1] = {'x'};
    if (fork() == 0)
    {
        int pid = getpid();
        if(read(pingpong[0], buffer, 1) != 1) {
            fprintf(2, " ping error\n");
            exit(1);
        }
        printf("%d: received ping\n", pid);
        close(pingpong[0]);
        write(pingpong[1], buffer, 1);
        close(pingpong[1]);
        exit(0);
    }
    int pid = getpid();
    write(pingpong[1], buffer, 1);
    wait(0);
    if(read(pingpong[0], buffer, 1) != 1) {
        fprintf(2, " pong error\n");
        exit(1);
    }
    close(pingpong[0]);
    printf("%d: received pong\n", pid);
    exit(0);
}
```



> 两个管道【题目也说的是两个管道】
>
> 父  →ping[1]→   `pipe(ping)`   →ping[0]→   子
>
> 进																          进
>
> 程  ⬅pong[0]⬅  `pipe(pong)`  ⬅pong[1]⬅  程

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
   
int
main()
{
    int ping[2];
    int pong[2];
    pipe(ping);
    pipe(pong);
    char buffer[1];
    if (fork() == 0)
    {
        int pid = getpid();
        close(ping[1]); //父对子的写
        close(pong[0]); //子对父的读
        if(read(ping[0], buffer, 1) != 1) {
            fprintf(2, " ping error");
            exit(1);
        }
        printf("%d: received ping\n", pid);
        write(pong[1], buffer, 1);
        close(pong[1]); //子对父的写
        exit(0);
    }
    
    int pid = getpid();
    close(ping[0]); //父对子的读
    close(pong[1]); //子对父的写
    write(ping[1], buffer, 1);
    close(ping[1]);  //父对子的写
    wait(0);
    if(read(pong[0], buffer, 1) != 1) {
        fprintf(2, "pong error");
        exit(1);
    }
    printf("%d: received pong\n", pid);
    exit(0);
}
```



#### 2.3 primes

##### Some hints

> Some hints:
>
> - Be careful to close file descriptors that a process doesn't need, because otherwise your program will run xv6 out of resources before the first process reaches 35.
> - Once the first process reaches 35, it should wait until the entire pipeline terminates, including all children, grandchildren, &c. Thus the main primes process should only exit after all the output has been printed, and after all the other primes processes have exited.
> - Hint: `read` returns zero when the write-side of a pipe is closed.
> - It's simplest to directly write 32-bit (4-byte) `int`s to the pipes, rather than using formatted ASCII I/O.
> - You should create the processes in the pipeline only as they are needed.
> - Add the program to `UPROGS` in Makefile.



##### solution

> ![image-20230519171752942](C:\Users\lstyy\AppData\Roaming\Typora\typora-user-images\image-20230519171752942.png)

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void prime(int rd){
    
    int base;
    if(read(rd, &base, 4) == 0) {
        return;
    }
    printf("prime %d\n", base);

    int p[2];
    pipe(p);
    if(fork() == 0) {
        close(p[1]);
        prime(p[0]);
        close(p[0]);
        exit(0);
    }
    int num;
    while(read(rd, &num,4)) {
        if (num % base != 0) {
            if(write(p[1], &num, 4) != 4) {
                fprintf(2, "Error");
                exit(1);
            }
        }
    }
    close(p[1]);
    wait(0);
    exit(0);
}
int
main(int argc, char *argv[])
{
    if(argc >= 2){
        fprintf(2, "Usage: primes only\n");
        exit(1);
    }
    int p[2];
    pipe(p);
    if(fork() == 0){
        close(p[1]);  //关闭写
        prime(p[0]);  //父进程把所有数据输进去，然后子进程来处理
        close(p[0]);  //关闭读
        exit(0);
    }
    close(p[0]);  //关闭读
    for(int i = 2; i <= 35; i ++){
        if(write(p[1], &i, 4) != 4) {  //i的地址写进管道里，一个占4个字节
            fprintf(2, "Error");
            exit(1);
        }
    }
    close(p[1]);  //关闭写
    wait(0);
    exit(0);
}
```



#### 2.4 find

##### Some hints

> Some hints:
>
> - Look at user/ls.c to see how to read directories.
> - Use recursion to allow find to descend into sub-directories.
> - Don't recurse into "." and "..".
> - Changes to the file system persist across runs of qemu; to get a clean file system run make clean and then make qemu.
> - You'll need to use C strings. Have a look at K&R (the C book), for example Section 5.5.
> - Note that == does not compare strings like in Python. Use strcmp() instead.
> - Add the program to `UPROGS` in Makefile.
>
> 源码阅读：
>
> ```c
> //  user/ls.c
> 
> #include "kernel/types.h"
> #include "kernel/stat.h"
> #include "user/user.h"
> #include "kernel/fs.h"  //在kernel/fs.h中#define DIRSIZ 14
> 
> //格式化文件名，接受文件路径作为输入，返回格式化后的文件名（返回的是文件名数组的地址）
> char*
> fmtname(char *path)   //path指针指向地址
> {
>   static char buf[DIRSIZ+1];  //静态字符数组，存储文件名。DIRSIZ是文件名的最大长度
>   char *p;
> 
>   // Find first character after last slash.
>   //strlen(const char *str)
>   for(p=path+strlen(path); p >= path && *p != '/'; p--)  //寻找路径中最后一个斜杠的位置
>     ;
>   p++;  //找到最后一个 / 之后+1，就找到了最后一个斜杠后第一个字符。p定位到了文件名的起始位置
> 
>   // Return blank-padded name.
>   //如果文件名的长度大于DIRSIZ，则直接返回；如果不是的话，需要格式化，以便对齐
>   if(strlen(p) >= DIRSIZ)
>     return p;
>   memmove(buf, p, strlen(p));  //user/ulib.c。将p指向的内存区域中的前strlen(p)个字节复制到buf指向的内存区域中
>   memset(buf+strlen(p), ' ', DIRSIZ-strlen(p)); //剩余空间清空。 将buf+strlen(p)指向的内存区域中的前n个字节设置为' '。
>   return buf;
> }
> 
> //接受文件路径作为输入，显示该路径中文件和目录的信息
> void
> ls(char *path)
> {
>   char buf[512], *p;
>   int fd;
>   /*
>   struct dirent {
>   ushort inum;  //目录项所对应的文件的inode编号
>   char name[DIRSIZ];  //目录项的名称，数组大小为DIRSIZ
>   };
>   */
>   struct dirent de;  //kernel/fs.h 。一个结构体，存储目录项的信息
>   /*
>   struct stat {
>   int dev;     // File system's disk device
>   uint ino;    // Inode number
>   short type;  // Type of file
>   short nlink; // Number of links to file
>   uint64 size; // Size of file in bytes
>   };
>   */
>   struct stat st;  //一个结构体，存储文件的状态信息（类型、inode号、大小等）
> 
>   if((fd = open(path, 0)) < 0){ //open(filename, flags), flags = 0/1 表示 读/写。返回fd
>     fprintf(2, "ls: cannot open %s\n", path);
>     return;
>   }
> 
>   //int fstat(int fildes, struct stat *buf)由文件描述符取得文件状态，buf是指向struct stat结构体的指针。将参数fildes所指的文件状态，复制到参数buf所指的，结构中，成功为0，失败为1
>   //由文件描述符fd取得文件状态，将文件状态复制到st所指的结构中
>   if(fstat(fd, &st) < 0){   //获取文件的状态信息
>     fprintf(2, "ls: cannot stat %s\n", path);
>     close(fd);
>     return;
>   }
> 
>   switch(st.type){  //type of file
>   case T_FILE:    //文件
>     printf("%s %d %d %l\n", fmtname(path), st.type, st.ino, st.size);  //输出格式化文件名，类型，Inode，大小
>     break;
> 
>   case T_DIR:     //目录
>     if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
>       printf("ls: path too long\n");
>       break;
>     }
>     strcpy(buf, path);
>     p = buf+strlen(buf);  //buf中存储了目录的路径，p指向buf数组中目录路径的后一位
>     *p++ = '/';  //先 *p = '/' 将'/'存储在p所指的位置，再 p++ p指向下一位
>     while(read(fd, &de, sizeof(de)) == sizeof(de)){
>       if(de.inum == 0) //inum表示目录项所对应的文件的inode编号，如果inum==0，说明该目录项没有指向任何文件，所以continue跳过当前循环，执行下一次循环
>         continue;
>       memmove(p, de.name, DIRSIZ);  //将de.name所指内存区域的前DIRSIZ个字节复制到p所指的内存区域。复制完后，不会改变指针p指向的位置，只是p指向的内存区域中的内容发生了变化
>       p[DIRSIZ] = 0;  //在目录项名称的末尾添加一个空字符，将其转换为一个以空字符结尾的字符串。等同于 *(p+DIRSIZ) = 0，将 p + DIRSIZ 所指的位置赋值为0。
>       //buf中存储了一个完整的文件路径
>       if(stat(buf, &st) < 0){
>         printf("ls: cannot stat %s\n", buf);
>         continue;
>       }
>       printf("%s %d %d %d\n", fmtname(buf), st.type, st.ino, st.size); // 输出格式化文件名，类型，Inode，大小
>     }
>     break;
>   }
>   close(fd);
> }
> 
> int
> main(int argc, char *argv[])
> {
>   int i;
> 
>   if(argc < 2){
>     ls(".");
>     exit(0);
>   }
>   for(i=1; i<argc; i++)    //argv[0]是ls
>     ls(argv[i]);
>   exit(0);
> }
> 
> ```
>
> 从main函数开始看，命令行参数数量小于2，默认输入为`ls .`
>
> 以`ls(".")`为例：
>
> 在 `ls` 函数中，
>
> 使用 `open` 函数打开指定的目录。如果打开失败，程序将打印错误信息并返回。
>
> 使用 `fstat` 函数获取目录的状态信息。如果获取失败，程序将打印错误的信息并返回。
>
> 然后利用 `switch` 条件语句：如果是一个文件，打印格式化文件名、类型、Inode、大小；如果是一个目录，则执行下列操作：
>
> 使用一个缓冲区 `buf` 存储目录的路径，并在路径末尾添加一个斜杠。
>
> 使用 `read` 函数读取目录中的每个目录项，如果inum为0，则跳过，读取下一个目录项。
>
> 将 `de.name` 所指内存区域的前DIRSIZ个字节复制到 p 所指的内存区域，末尾添加一个空字符，转换为一个以空字符结尾的字符串。
>
> 获取文件的状态信息，如果获取失败，程序将打印错误的信息，并跳过，读取下一个目录项。
>
> 打印格式化文件名、类型、Inode、大小。
>
> 
>
> 补充：
>
> - `struct dirent	` 是一个结构体，它用来表示一个目录项。一个目录项包含两个字段： `inum` 和`name`。`inum` 表示目录项所对应的文件的 inode 编号，`name` 表示目录项的名称。
>
> - 目录是一个特殊类型的文件，它的内容是一系列连续的 `struct dirent` 结构体，每个结构体都表示一个目录项。一个目录中可以包含多个目录项，每个目录项都对应一个文件或子目录。当再程序中读取一个目录时，可以使用 `read` 函数来读取目录文件的内容，每次调用 `read` 函数都会返回一个 `struct dirent	` 结构体，即一个目录项。
>
> - 在Unix文件系统中，为了标记删除文件，通常只是将文件的目录项的 inum 字段设为0，而不是真正地删除该目录项。这样做可以保证被删除的文件可以被恢复，同时也保证了目录项的连续性和文件系统的性能。



##### solution

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

char* fmtname(char *path){
    static char buf[DIRSIZ+1];
    char *p;
    for(p=path+strlen(path); p >= path && *p != '/'; p--);
    p++;
    memmove(buf, p, strlen(p));
    buf[strlen(p)] = 0;  //末尾添加空字符
    return buf;
}

void find(char* path, char* filename) {
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if((fd = open(path, 0)) < 0)
    {
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }
    if(fstat(fd, &st) < 0) {
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch(st.type) {
        case T_FILE:
            if(strcmp(fmtname(path), filename) == 0) {
                printf("%s\n", path);
            }
            break;
        case T_DIR:
            if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf) {
                printf("find: path too long\n");
                break;
            }
            strcpy(buf, path);
            p = buf + strlen(buf);
            *p++ = '/';
            while(read(fd, &de, sizeof(de)) == sizeof(de)) {
                if(de.inum == 0)
                    continue;
                if(strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
                    continue;
                memmove(p, de.name, strlen(de.name));
                p[strlen(de.name)] = 0;
                find(buf, filename);
            }
            break;
        }
        close(fd);
}

int
main(int argc, char* argv[])
{
    if(argc < 3) {
        fprintf(2, "Usage: find <path> <filename>\n");
        exit(0);
    }
    find(argv[1], argv[2]);
    exit(0);
}

```

> 注意点：
>
> `find.c` 基本就是按照 `ls.c` 来写的，但是有一个细节，在 `find.c` 的 `fmtname` 函数中，多了这一行 `buf[strlen(p)] = 0;` 。而为什么 `ls.c` 中没有呢？
>
> 可以在 `ls.c` 的 `fmtname` 函数中 `static char buf[DIRSIZ+1];` 后面加上：
>
> ```
> static char buf[DIRSIZ+1];
> if(buf[DIRSIZ] == 0) {
>     printf("kongzifu");
> }
> ```
>
> 运行后可以发现 `buf[DIRSIZ]` 这里已经是空字符了，所以后面也没必要加。
>
> 同理， `find.c` 的 `fmtname` 函数中 `static char buf[DIRSIZ+1];` 完后 `buf[DIRSIZ]` 也是空字符，但是因为 `find.c` 的 `fmtname` 函数中 `buf` 数组中存储的是路径最后一个斜杠后的文件名，不需要把数组后面的位置填充为空格字符，所以要在文件名末尾添加空字符，即 `buf[strlen(p)] = 0;` 。



#### 2.5 xargs

##### Some hints

> Some hints:
>
> - Use `fork` and `exec` to invoke the command on each line of input. Use `wait` in the parent to wait for the child to complete the command.
> - To read individual lines of input, read a character at a time until a newline ('\n') appears.
> - kernel/param.h declares MAXARG, which may be useful if you need to declare an argv array.
> - Add the program to `UPROGS` in Makefile.
> - Changes to the file system persist across runs of qemu; to get a clean file system run make clean and then make qemu.
>
> 题目给的示例：
>
> 