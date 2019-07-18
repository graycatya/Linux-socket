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

        
        