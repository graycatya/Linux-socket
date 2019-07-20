* Libevent 学习

1. 安装方法

```
//在libevent源码目录下执行以下操作

./configure /path //检测安装环境

make

sudo make install
```

2. API说明

    * event_base结构体

        使用libevent函数之前需要分配一个或者多个event_base结构体。每个event_base结构体持有一个事件集合，可以检测以确定那个事件是激活的。

        如果设置event_base使用锁，则可以安全地在多个线程中访问它。然而，其事件循环只能运行在一个线程中。如果需要多个线程检测IO，则需要为每个线程
        使用一个event_base.

        每个event_base都有一种用于检测那种事件已经就绪的"方法"，或者说后端。可以识别的方法有:

        * select
        * poll
        * epoll
        * kqueue
        * devpoll
        * evport
        * win32

    * 创建默认的event_base

        event_base_new()函数分配并且返回一个新的具有默认设置的event_base。函数会检测环境变量，返回一个到event_base的指针。如果发生错误，则返回NULL。选择各种方法时，函数会选择OS支持的最快方法。

        ```
        /*大多数程序使用它就够了*/
        struct event_base *event_base_new(void); /*libevent 1.4.3版后就出现*/
        ```

    * 创建复杂的event_base

        要取得什么类型的event_base有更多的控制，就需要使用event_config.

        event_config是一个容纳event_base配置信息的不透明结构体。需要event_base时，将event_config传递给event_base_with_config().

        ```
        struct event_config *event_config_new(void);
        struct event_base *event_base_with_config(const struct event_config *cfg);
        void event_config_free(struct event_config *cfg);
        ```

        要使用这些函数分配event_base.先调用event_config_new()分配一个event_config.然后，对event_config调用其它函数，设置所需的event_base特征。最后调用event_base_new_with_config()获取新的event_base.完成工作后，使用event_config_free()释放event_config.

        ```
        int event_config_avoid_method(struct event_config *cfg, const char *method);

        enum event_method_feature {
            EV_FETURE_ET = 0x01,
            EV_FETURE_O1 = 0x02,
            EV_FEATURE_FDS = 0x04
        };

        int event_config_require_features(struct event_config *cfg, enum event_method_feature feature);

        enum event_base_config_flag {
            EVENT_BASE_FLAG_NOLOCK = 0x01,
            EVENT_BASE_FLAG_IGNORE_ENV = 0x02,
            EVENT_BASE_STARTUP_IOCP = 0x04,
            EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,
            EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST = 0x10,
            EVENT_BASE_FLAG_PRECISE_TIMER = 0x20
        };

        int event_config_set_flag(struct event_config *cfg, enum event_base_config_flag flag);
        ```

        调用event_config_avoid_method()可以通过名字让libevent避免使用特定的可用后端。调用event_config_require_feature()让libevent不使用不能提供所有指定特征的后端。调用event_config_set_flag()让libevent在创建event_base时设置一个或者多个运行时标志。

        * event_config_require_features()可识别的特征值有:
            * EV_FEATURE_ET: 要求支持边缘触发的后端
            * EV_FEATURE_O1: 要求添加，删除单个事件，或者确定那个事件激活的操作时0(1)复杂度的后端
            * EV_FEATURE_FDS: 要求支持任意文件描述符，而不仅仅是套接字的后端

        * event_config_set_flag()可识别的选项值有:
            * EVENT_BASE_FLAG_NOLOCK: 不要为event_base分配锁。设置这个选项可以为event_base节省一点用于锁定和解锁的时间，但是让在多个线程中访问event_base成为不安全的。
            * EVENT_BASE_FLAG_IGNORE_ENV: 选择使用的后端时，不要检测EVENT* 环境变量。使用这个标志需要三思:这会让用户更难调试你的程序与libevent的交互。
            * EVENT_BASE_FLAG_STARTUP_IOCP: 仅用于Windows,让libevent在启动时就启用任何必须的IOCP分发逻辑，而不是按需启用。
            * EVENT_BASE_FLAG_NO_CACHE_TIME: 不是在事件循环每次准备执行超时回调时检测当前时间，而是在每次超时回调后进行检测。注意:这会消耗更多的CPU时间。
            * EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST: 告诉libevent,如果决定使用epoll后端，可以安全的使用更更快的基于changelist的后端。epoll-changelist后端可以在后端的分发函数调用之间，同样的fd多次修改其状态的情况下，避免不必要的系统调用。但是如果传递任何使用dup()或者其变体克隆的fd给libevent, epoll-changelist后端会触发一个内核bug，导致不确定的结果。在不使用epoll后端的情况下，这个标志是没有效果的。也可以通过设置 EVENNT_EPOLL_USE_CHANGELIST环境变量来打开epoll-changelist选项。

        上述操作event_config的函数都在成功时返回0，失败时返回-1.

        注意: 设置event_config.请求OS不能提供的后端是很容易的。比如说，对于libevent2.0.1-alpha,在Windows中是没有0(1)后端的；在Linux中也没有同时提供EV_FEATURE_FDS 和 EV_FEATURE_O1特征的后端。如果创建了libevent不能满足的配置，event_base_new_with_config()会返回NULL。


    * 检查event_base后端

        有时候需要检查event_base 支持哪些特性，或者当前使用哪种方法。

        * 接口 1

            ```
            const char **event_get_supported_methods(void);
            ```

            * event_get_supported_methods()函数返回一个指针，指向libevent支持的方法名字数组。这个数组的最后一个元素是NULL。

            实例:

            ```
            int i;
            const char **methods = evnet_get_supported_methods();
            printf("Starting Libevent %s. Available methods are:\n", event_get_version());
            for(i = 0; methods[i] != NULL; ++i)
            {
                printf("    %s\n", methods[i]);
            }
            ```

            这个函数返回libevent被编译以支持的方法列表。然而libevent运行的时候，操作系统可能不能支持所有方法。可能OSX版本中的kqueue的bug大多，无法使用。

        * 接口 2

            ```
            const char* event_base_get_method(const struct event_base *base);

            enum event_method_feature {
                EV_FETURE_ET = 0x01,
                EV_FETURE_O1 = 0x02,
                EV_FEATURE_FDS = 0x04
            };
            event_base_get_features(const struct event_base *base);
            ```

            event_base_get_method()返回event_base正在使用的方法。

            event_base_get_features() 返回event_base支持的特征的比特掩码。

        
        实例:

        ```
        struct event_base *base;
        enum event_method_feature f;

        base = event_base_new();
        if(!base)
        {
            puts("Couldn't get an event_base!");
        }
        else
        {
            printf("Using Libevent with backend method %s.", event_base_get_method(base) );
            f = event_base_get_features(base);
            if((f & EV_FEATURE_ET))
            {
                printf("    Edge-triggered events are supported.");
            }
            if((f & EV_FEATURE_O1))
            {
                printf("    0(1) event notification is supported.");
            }
            if((f & EV_FEATURE_FDS))
            {
                printf("    All FD types are supported");
            }
            puts("");
        }
        ```
        
    * 释放event_base

        ```
        void event_base_free(struct event_base *base);
        ```

    * event_base优先级

        libevent支持为事件设置多个优先级。然而，event_base默认只支持单个优先级。可以调用event_base_priority_init()设置event_base的优先级数目。

        ```
        int event_base_priority_init(struct event_base *base, int n_priorities);
        ```

        成功时这个函数返回0，失败时返回-1.base是要修改的event_base, n_priorities是要支持的优先级数目，这个数目至少是1.每个新的事件可用的优先级将从0(最高)到n_priorities-1(最低).

        常量EVENT_MAX_PRIORITIES表示n_priorities的上限。调用这个函数时为n_priorities给出更大的值是错误的。

        注意:必须在任何事件激活之前调用这个函数，最好在创建event_base后立即调用。


    * event_base和fork

        不是所有事件后端都在调用fork()之后可以正确工作。所以，如果使用fork()或者其他相关系统调用启动新进程之后，希望在新进程中继续使用event_base,就需要进行重新初始化。

        ```
        int event_reinit(struct event_base *base);
        ```

        实例:

        ```
        struct event_base *base = event_base_new();

        if(fork())
        {
            continue_running_parent(base);
        }
        else
        {
            event_reinit(base);
            continue_running_child(base);
        }
        ```


    * 事件循环event_loop

        * 运行循环

            一旦有了一个已经注册了某些事件的event_base(关于如何创建和注册事件请看下一节)， 就需要让libevent等待事件并且通知事件的发生。

            ```
            #define EVLOOP_ONCE     0x01
            #define EVLOOP_NONBLOCK     0x02
            #define EVLOOP_NO_EXIT_ON_EMPTY     0x04

            int event_base_loop(struct event_base *base, int flags);
            ```

        默认情况下,event_base_loop()函数运行event_base直到其中没有已经注册的事件为止。执行循环的时候，函数重复地检查是否有任何已经注册的事件被触发(比如说，读事件的文件描述符已经就绪，可以读取了;或者超时事件的超时时间即将到达)。如果有事件被触发，函数标记被触发的事件为"激活的"，并且执行这些事件。

        在flags参数中设置一个或者多个标志就可以改变event_base_loop()的行为。如果设置了EVLOOP_ONCE, 循环将等待某些事件成为激活的，执行激活的事件直到没有更多的事件可以执行，然会返回。如果设置了EVLOOP_NONBLOCK,循环不会等待事件被触发:循环将仅仅检测是否有事件已经就绪，可以立即触发，如果有，则执行事件的回调。

        完成工作后，如果正常退出，event_base_loop()返回0;如果因为后端中的某些未处理错误而退出，则返回-1.

        实例(理解event_base_loop()算法概要):

            ```
            while(any events are registered with the loop, or EVLOOP_NO_EXIT_ON_EMPTY was set)
            {
                if(EVLOOP_NONBLOCK was set, or any events are already active)
                {
                    If any registered events have triggered, mark them active.
                }
                else
                {
                    Wait until at least one event has triggered, and mark it active.
                }

                for(p = 0; p < n_priorities; ++p)
                {
                    if(any event with priority of p is active)
                    {
                        Run all active events with priority of p.
                        break;
                    }
                }

                if(EVLOOP_ONCE was set or EVLOOP_NONLOCK was set)
                {
                    break;
                }
            }
            ```

            为了方便也可以调用

            ```
            int event_base_dispatch(struct event_base *base);
            ```

            event_base_dispatch() 等同于没有设置标志的event_base_loop(). 
            
            所以，event_base_dispatch() 将一直运行，直到没有已经注册的事件了，或者调用了

            event_base_loopbreak() 或者 event_base_loopexit() 为止。

    
    * 停止循环

        如果想在移除所有已经注册的事件之前停止活动的事件循环，可以调用两个稍有不同的函数。

        ```
        int event_base_loopexit(struct event_base *base, const struct timeval *tv);

        int event_base_loopbreak(struct event_base *base);
        ```

        event_base_loopexit() 让event_base在给定时间之后停止循环。如果tv参数为NULL,event_base会立即停止循环，没有延时。

        如果event_base 当前正在执行任何激活事件的回调，则回调会继续运行，直到运行完所有激活事件的回调后才退出。

        event_base_loopbreak()让event_base立即退出循环。它与event_base_loopexit(base, NULL)的不同在于，如果event_base当前正在执行激活事件的回调，它将在执行完当前正在处理的事件后立即退出。

        注意: event_base_loopexit(base, NULL); 和 event_base_loopbreak(base)在事件循环没有运行时的行为不同；前者安排下一次事件循环在下一轮回调完成后立即停止(就好像带EVLOOP_ONCE标志调用一样);后者却仅仅停止当前正在运行的循环，如果事件循环没有运行，则没有任何效果。

        实例 1:

            ```
            #include <event2/event.h>

            /*这里有一个回调函数调用loopbreak*/
            void cb(int sock, short what, void *arg)
            {
                struct event_base *base = arg;
                event_base_loopbreak(base);
            }

            void main_loop(struct event_base *base, evutil_socket_t watchdog_fd)
            {
                struct event *watchdog_event;

                /*onstruct一个新事件，每当有任何字节要从看门狗套接字读取时，
                该事件就会被触发。当这种情况发生时，我们将调用cb函数，它将
                使循环立即退出，而不运行任何其他活动事件。*/
                watchdog_event = event_new(base, watchdog_fd, EV_READ, cb, base);

                event_add(watchdog_event, NULL);

                event_base_dispatch(base);
            }
            ```

        实例 2(执行事件循环10秒，然后退出):


            ```
            #include <event2/event.h>

            void run_base_with_ticks(struct event_base *base)
            {
                struct timeval ten_sec;
                ten_sec.tv_sec = 10;
                ten_sec.tv_usec = 0;

                /* 现在，我们以10秒为间隔运行event_base，在每个间隔之后打印“Tick”。
                要了解实现10秒计时器的更好方法，请参阅下面关于持久计时器事件的部分。*/

                while(1)
                {
                    /*初始化10秒后退出时间*/
                    event_base_loopexit(base, &ten_sec);

                    event_base_dispatch(base);

                    puts("Tick");
                }
            }
            ```


        有时候需要知道对 event_base_dispatch() 或者 event_base_loop() 的调用是正常退出的，还是因为调用 event_base_loopexit() 或者 event_base_break() 而退出的。 可以调用下述函数来确定是否调用了loopexit或者break函数。


            ```
            int event_base_got_exit(struct event_base *base);

            int event_base_got_break(struct event_base *base);
            ```


        这两个函数分别会在循环是因为调用 event_base_loopexit() 或者 event_base_break() 而退出的时候返回true，否则返回false。下次启动事件循环的时候，这些值会被重设。

    
    * 转储 event_base 的状态

        为帮助调试程序(或者调试libevent),有时候可能需要加入到event_base 的事件及其状态的完整列表。调用event_base_dump_events()可以将这个列表输出到指定的文件中。

            ```
            void event_base_dump_events(struct event_base *base, FILE *f);
            ```

        这个列表是人可读的，未来版本的libevent将会改变其格式。

    * 事件 event

        libevent 的基本操作单元是事件。每个事件代表一组条件的集合，这些条件包括:

            * 文件描述符已经就绪，可以读取或者写入
            
            * 文件描述符变为就绪状态，可以读取或者写入(仅对于边沿触发IO) 

            * 超时事件

            * 用户触发事件 

        所有事件具有相似的生命周期。调用libevent函数设置事件并且关联到 event_base之后，事件进入"已初始化(initialized)状态"。此时可以将事件添加到event_base中，这使之进入"未决(pending)"状态。在未决状态下，如果触发事件的条件发生(比如说，文件描述符的状态改变，或者超时时间到达)，则事件进入"激活(active)"状态，(用户提供的)事件回调函数将被执行。如果配置为"持久的(persistent)",事件将保持为未决状态。否则，执行完回调后，事件不再是未决的。删除操作可以让未决事件成为非未决(已初始化)的；添加操作可以让非未决事件再次成为未决的。


    * 生成新事件

        使用 event_new() 接口创建事件。

        ```
        #define EV_TIMEOUT  0x01    //超时事件
        #define EV_READ     0x02    //读事件
        #define EV_WRITE    0x04    //写事件
        #define EV_SIGNAL   0x08    //信号事件
        #define EV_PERSIST  0x10    //周期性触发
        #define EV_ET       0x20    //边缘触发，如果底层模型支持

        typedef void (*event_cakkback_fn)(evutil_socket_t, short, void*);

        struct event* event_new(struct event_base *base, evutil_socket_t fd, 
                            short what, event_callback_fn cb,
                            void *arg);

        void event_free(struct event *event);
        ```

        event_new() 试图分配和构造一个用于base的新的事件。what参数是上述标志的集合。

        如果fd非负，则它是将被观察其读写事件的文件。

        事件被激活时，libevent将调用cb函数。

        传递这些参数: 文件描述符fd，表示所有被触发事件的位字段，以及构造事件时的arg参数。

        发生内部错误，或者传入无效参数时，event_new()将返回NULL。

        注意: 新创建的事件都处于已初始化和非未决状态，调用event_add()可以使其成为未决的。要释放事件，调用event_free(). 对于未决或者激活状态的事件调用event_free()是安全的。在释放事件之前，函数将会使事件成为非激活和非未决的。


        实例:

            ```
            #include <event2/event.h>

            void cb_func(evutil_socket_t fd, short what, void *arg)
            {
                const char *data = arg;
                printf("Got an event on socket %d:%s%s%s%s [%s]",
                    (int)fd,
                    (what&EV_TIMEOUT) ? " timeout" : "",
                    (what&EV_READ) ?    " read" : "",
                    (what&EV_WRITE) ?   " write" : "",
                    (what&EV_SIGNAL) ?  " signal" : "",
                    data);

            }

            void main_loop(evutil_socket_t fd1, evutil_socket_t fd2)
            {
                struct event *ev1, *ev2;
                struct timeval five_seconds = {5,0};
                struct event_base *base = event_base_new();

                /* 调用者已经以某种方式设置了fd1、fd2，并使它们非阻塞。 */
                ev1 = event_new(base, fd1, EV_TIMEOUT|EV_READ|EV_PERSIST, cb_func, (char*)"Reading event");
                ev2 = event_new(base, fd2, EV_WRITE|EV_PERSIST, cb_func, (char*)"Writing event");

                event_add(ev1, &five_seconds);
                event_add(ev2, NULL);

                event_base_dispatch(base);
            }
            ```

    * 事件持久性

        默认情况下,每当未决事件成为激活的(因为 fd 已经准备好读取或者写入,或者因为超
        时), 事件将在其回调被执行前成为非未决的。如果想让事件再次成为未决的 ,可以
        在回调函数中 再次对其调用 event_add()。

        然而,如果设置了 EV_PERSIST 标志,事件就是持久的。这意味着即使其回调被激活
        ,事件还是会保持为未决状态 。如果想在回调中让事件成为非未决的 ,可以对其调用
        event_del ()。


        每次执行事件回调的时候,持久事件的超时值会被复位。因此,如果具有 EV_READ|EV_PERSIST 标志,以及5秒的超时值,则事件将在以下情况下成为激活的:

            * 套接字已经准备好被读取的时候

            * 从最后一次成为激活的开始，已经逝去5秒

    
    * 信号事件

        libevent 也可以监测 POSIX 风格的信号。要构造信号处理器,使用:

        ```
        #define evsignal_new(base, signum, cb, arg)     event_new(base, signum, EV_SIGNAL | EV_PERSIST, cb, arg)

        struct event *hup_event;
        struct event_base *base = event_base_new();
        /* call sighup_function on a HUP signal */
        hup_event = evsignal_new(base, SIGHUP, sighup_function, NULL);
        ```

        注意 :信号回调是信号发生后在事件循环中被执行的,所以可以安全地调用通常不能 在 POSIX 风格信号处理器中使用的函数.

        libevent 也提供了一组方便使用的宏用于处理信号事件:

        ```
        #define evsignal_add(ev, tv) event_add((ev),(tv))
        #define evsignal_del(ev) event_del(ev)
        #define evsignal_pending(ev, what, tv_out) event_pending((ev), (what), (tv_out))
        ```

        注意: libevent和大多数后端中，每个进程任何时刻只能有一个event_base可以监听信号。如果同时两个event_base添加
        信号事件，即使是不同的信号，也只有一个event_base可以取得信号。kqueue后端没有这个限制。

    * 事件的未决和非未决

        构造事件之后，在将其添加到event_base 之前实际上是不能对其做任何操作的。使用event_add() 将事件添加到event_base.

    * 设置未决事件

        ```
            int event_add(struct event *ev, const struct timeval *tv);
        ```

        在非未决的事件上调用event_add() 将使其在配置的event_base 中成为未决的。成功时函数返回0，失败时返回-1.

        如果tv位NULL，添加的事件不会超时。否则，tv以秒和微妙指定超时值。

        如果对已经未决的事件调用 event_add(),事件将保持未决状态,并在指定的超时时间被重新调度。

        注意 :不要设置 tv 为希望超时事件执行的时间。如果在 2010 年 1 月 1 日设置 “tv->tv_sec=time(NULL)+10;”,超时事件将会等待40年,而不是10秒。


    * 设置非未决事件

        ```
            int event_del(struct event *ev);
        ```

        对已经初始化的事件调用 event_del()将使其成为非未决和非激活的。如果事件不是
        未决的或者激活的,调用将没有效果。成功时函数返回 0,失败时返回-1。
        注意 :如果在事件激活后,其回调被执行前删除事件,回调将不会执行。


    * 事件的优先级

        多个事件同时触发时，libevent没有定义各个回调的执行次序。可以使用优先级来定义某些事件比其他事件更重要。

        每个 event_base 有与之相关的一个或者多个优先级。在初始化事件之后, 但是在添加到 event_base 之前,可以为其设置优先级。

        ```
        int event_priority_set(struct event *event, int priority);
        ```

        事件的优先级是一个在 0和 event_base 的优先级减去1之间的数值。成功时函数返回 0,失 败时返回-1。

        多个不同优先级的事件同时成为激活的时候 ,低优先级的事件不会运行 。libevent
        会执行高优先级的事件,然后重新检查各个事件。只有在没有高优先级的事件是激活
        的时候 ,低优先级的事件才会运行。


        实例:

        ```
        #include <event2/event.h>
        void read_cb(evutil_socket_t, short, void*);

        void write_cb(evutil_socket_t, short, void*);

        void main_loop(evutil_socket_t fd)
        {
            struct event *important, *unimportant;
            struct event_base *base;

            base = event_base_new();
            event_base_priority_init(base, 2);
            /* 现在base的优先级0，优先级1 */
            important = event_new(base, fd, EV_WRITE | EV_PERSIST, write_cb, NULL);

            unimportant = event_new(base, fd, EV_READ | EV_PERSIST, read_cb, NULL);

            event_priority_set(important, 0);
            event_priotity_set(unimportant, 1);

            /* 现在，每当fd准备好写时，写回调将发生在读回调之前。直到写回调不再活动时，读回调才会发生。 */
        }
        ```

        注意: 如果不为事件设置优先级,则默认的优先级将会是 event_base 的优先级数目除以2.


    * 检查事件状态

        有时候需要了解事件是否已经添加，检查事件代表什么。

            ```
            int event_pending(const struct event *ev, short what, struct timeval *tv_out);

            #define event_get_signal(ev) /* ... */

            evutil_socket_t event_get_fd(const struct event *ev);

            struct event_base *event_get_base(const struct event *ev);

            short event_get_events(const struct event *ev);

            event_callback_fn event_get_callback(const struct event *ev);

            void *event_get_callback_arg(const struct event *ev);

            int event_get_priority(const struct event *ev);

            void event_get_assignment(const struct event *event, struct event_base **base_out,
            evutil_socket_t *fd_out, short *events_out, event_callback_fn *callback_out, void **arg_out);
            ```

            event_pending()函数确定给定的事件是否是未决的或者激活的。如果是,而且 what
            参 数设置了 EV_READ、EV_WRITE、EV_SIGNAL 或者 EV_TIMEOUT 等标志,则
            函数会返回事件当前为之未决或者激活的所有标志 。如果提供了 tv_out 参数,并且
            what 参数中设置了 EV_TIMEOUT 标志,而事件当前正因超时事件而未决或者激活,
            则 tv_out 会返回事件 的超时值。

            event_get_fd()和 event_get_signal()返回为事件配置的文件描述符或者信号值。

            event_get_base()返回为事件配置的 event_base。event_get_events()返回事件的
            标志(EV_READ、EV_WRITE 等)。event_get_callback()和
            event_get_callback_arg() 返回事件的回调函数及其参数指针。

            event_get_assignment()复制所有为事件分配的字段到提供的指针中。任何为
            NULL 的参数会被忽略。


        实例:

            ```
            #include <event2/event.h>
            #include <stdio.h>
            /* 更改'ev'的回调函数和callback_arg，这两个函数不能挂起。 */
            int replace_callback(struct event *ev, event_callback_fn new_callback, void *new_callback_arg)
            {
            struct event_base *base;
            evutil_socket_t fd;
            short events;
            int pending;
            pending = event_pending(ev, EV_READ|EV_WRITE|EV_SIGNAL|EV_TIMEOUT, NULL);

            if (pending) 
            {
                /* 我们希望在这里捕捉到这一点，这样就不会重新分配挂起的事件。那将非常非常糟糕。 */
                fprintf(stderr, "Error! replace_callback called on a pending event!\n");
                return -1;
            }
            event_get_assignment(ev, &base, &fd, &events,
            NULL /* ignore old callback */ ,
            NULL /* ignore old callback argument */
            );
            event_assign(ev, base, fd, events, new_callback, new_callback_arg);
            return 0;
            }
            ```

    * 一次触发事件

        如果不需要多次添加一个事件，或者要在添加后立即删除事件，而事件又不需要是持久的，则可以使用event_base_once().

        ```
        int event_base_once(struct event_base*, evutil_socket_t, short, void(*)(evutil_socket_t, short, void *), void *, const struct timeval *);
        ```

        除了不支持EV_SIGNAL或者EV_PERSIST之外，这个函数的接口与event_new()相同。 安排的事件将以默认的优先级加入到event_base并执行。回调被执行后，libevent内部将会释放event结构。成功时函数返回0，失败时返回-1.

        不能删除或者手动激活使用event_base_once() 插入的事件；如果希望能够取消事件，应该使用event_new() 或者 event_assign().

    * 手动激活事件

        极少数情况下,需要在事件的条件没有触发的时候让事件成为激活的。

        ```
        void event_active(struct event *ev, int what, short ncalls);
        ```

        这个函数让事件 ev 带有标志 what(EV_READ、EV_WRITE 和 EV_TIMEOUT 的组合)成 为激活的。事件不需要已经处于未决状态,激活事件也不会让它成为未决的。


    * 事件状态之间的转换

        ![op](./img/libevent0.png)


    * 数据缓冲Bufferevent

        很多时候，除了响应事件之外，应该还希望做一定的数据缓冲。比如说，写入数据的时候，通常的运行模式是:

            * 决定要向连接写入一些数据，把数据放入到缓冲区中

            * 等待连接可以写入

            * 写入尽量多的数据

            * 记住写入了多少数据，如果还有更多数据要写入，等待连接再次可以写入

                这种缓冲IO模式很通用，libevent为此提供了一种通用机制，即bufferevent。

                bufferevent 由一个底层的传输端口(如套接字)， 一个读取缓冲区和一个写入缓冲区组成。
                与通常的事件在底层传输端口已经就绪，可以读取或者写入的时候执行回调不同的是，bufferevent
                在读取或者写入了足够量的数据之后调用用户提供的回调。

            有多种共享公用接口的bufferevent类型，编写本文时已存在以下类型:

                * 基于套接字的 bufferevent: 使用event_*接口作为后端，通过底层流式套接字发送或者接收数据的bufferevent

                * 异步 IO bufferevent: 使用Windows IOCP接口，通过底层流式套接字发送或者接收数据的bufferevent(仅用于Windows)

                * 过滤型bufferevent: 将数据传输到底层 bufferevent 对象之前，处理输入或者输出数据的bufferevent: 比如说，为了压缩或者转换数据。

                * 成对的 bufferevent: 相互传输数据的两个 bufferevent。

