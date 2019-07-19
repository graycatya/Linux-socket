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