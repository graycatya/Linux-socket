# linux 信号

* 简介

    * 信号是由用户，系统或者进程发送给目标进程的信息，以通知目标进程某个状态的改变或系统异常。Linux信号可由如下条件产生：
        
        1. 对于前台进程，用户可以通过输入特殊的终端字符来给它发送信号。比如输入Ctrl+C通常会给进程发送一个中断信号。
        2. 系统异常。比如浮点异常和非法内存段访问。
        3. 系统状态变化。比如alarm定时器将引起SIGALRM信号。  
        4. 运行kill命令或调用kill函数。
        服务器程序必须处理(或至少忽略)一些常见的信号，以免异常终止。

* 发送信号

```
#include<sys/types.h>
#include<signal.h>

/*给其他进程发送信号*/
int kill(pid_t pid, int sig);
```

![pid_t参数](./img/signalpid.png)

Linux定义的信号值都大于0，如果sig取值为0，则kill函数不发送任何信号。但将sig设置为0可以用来检测目标进程或进程组是否存在，因为检测工作总是在信号发送之前就执行。不过这种检测方式是不可靠的。一方面由于进程PID的回绕，可能导致被检测的PID不是我们期望的进程的PID：另一方面，这种检测方法不是原子操作。

![error](./img/killerror.png)

* 信号处理方式

```
typedef void (*__sighandler_t) (int);
```

信号处理函数只带有一个整型参数，该参数用来指示信号类型。信号处理函数应该是可重入的，否则很容易引发一些竞态条件。所以在信号处理函数中严禁调用一些不安全的函数。

```
#include <bits/signum.h>
#define SIG_DEL ((__sighandler_t) 0)
#define SIG_IGN ((__sighandler_t) 1)
```

SIG_IGN表示忽略目标信号， SIG_DEL表示使用信号的默认处理方式。信号的默认处理方式有如下几种：结束进程(Term),忽略信号（lgn），结束进程并生成核心转储文件(Core),暂停进程（stop），以及继续进程（Cont）

![标准信号](./img/signals0.png)

![标准信号](./img/signals1.png)