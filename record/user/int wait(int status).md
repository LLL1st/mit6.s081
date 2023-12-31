### 系统调用int wait(int \*status)

函数功能是：父进程一旦调用了 `wait` 就立即阻塞自己，由 `wait` 自动分析是否当前进程的某个子进程已经退出，如果让它找到了这样一个已经变成僵尸的子进程，`wait` 就会收集这个子进程的信息，并把它彻底销毁后返回；如果没有找到这样一个子进程，`wait` 就会一直阻塞在这里，直到有一个出现为止。





注：

　　当父进程忘了用 `wait()` 函数等待已终止的子进程时,子进程就会进入一种无父进程的状态,此时子进程就是僵尸进程.

　　`wait()` 要与 `fork()` 配套出现,如果在使用 fork() 之前调用 wait() , wait() 的返回值则为-1,正常情况下 wait() 的返回值为子进程的PID.

　　如果先终止父进程,子进程将继续正常进行，只是它将由init进程(PID 1)继承,当子进程终止时,init进程捕获这个状态.

　　参数 status 用来保存被收集进程退出时的一些状态，它是一个指向 int 类型的指针。但如果我们对这个子进程是如何死掉毫不在意，只想把这个僵尸进程消灭掉，（事实上绝大多数情况下，我们都会这样想），我们就可以设定这个参数为 NULL ，就像下面这样：

```
pid = wait(NULL);
```

如果成功，wait会返回被收集的子进程的进程ID，如果调用进程没有子进程，调用就会失败，此时wait返回-1，同时 errno 被置为 ECHILD 。

　　如果参数status的值不是NULL，wait就会把子进程退出时的状态取出并存入其中， 这是一个整数值（int），指出了子进程是正常退出还是被非正常结束的，以及正常结束时的返回值，或被哪一个信号结束的等信息。由于这些信息 被存放在一个整数的不同二进制位中，所以用常规的方法读取会非常麻烦，人们就设计了一套专门的宏（macro）来完成这项工作，下面我们来学习一下其中最常用的两个：

1，WIFEXITED(status) 这个宏用来指出子进程是否为正常退出的，如果是，它会返回一个非零值。

（请注意，虽然名字一样，这里的参数status并不同于wait唯一的参数–指向整数的指针status，而是那个指针所指向的整数，切记不要搞混了。）

2， WEXITSTATUS(status) 当WIFEXITED返回非零值时，我们可以用这个宏来提取子进程的返回值，如果子进程调用exit(5)退出，WEXITSTATUS(status) 就会返回5；如果子进程调用exit(7)，WEXITSTATUS(status)就会返回7。请注意，如果进程不是正常退出的，也就是说， WIFEXITED返回0，这个值就毫无意义。