.. vim: syntax=rst

创建线程
=============

在上一章，我们已经基于野火STM32开发板创建好了RT-Thread的工程模板，这章开始我们将真正进入如何使用RT-Thread的征程，先从最简单的创建线程开始，点亮一个LED，以慰藉下尔等初学者弱小的心灵。

硬件初始化
~~~~~~~~~~~~~~

本章创建的线程需要用到开发板上的LED，所以先要将LED相关的函数初始化好，具体是在board.c的
rt_hw_board_init()函数中初始化，具体见代码清单 14‑1的高亮部分。

.. code-block:: c
    :caption: 代码清单 14‑1rt_hw_board_init()中添加硬件初始化函数
    :emphasize-lines: 17-18
    :linenos:

    /**
    * @brief  开发板硬件初始化函数
    * @param  无
    * @retval 无
    *
    * @attention
    * RTT把开发板相关的初始化函数统一放到board.c文件中实现，
    * 当然，你想把这些函数统一放到main.c文件也是可以的。
    */
    void rt_hw_board_init()
    {
        /* 初始化SysTick */
        SysTick_Config( SystemCoreClock / RT_TICK_PER_SECOND );

        /* 硬件BSP初始化统统放在这里，比如LED，串口，LCD等 */

        /* 初始化开发板的LED */
        LED_GPIO_Config();

        /* 调用组件初始化函数 (use INIT_BOARD_EXPORT()) */
    #ifdef RT_USING_COMPONENTS_INIT
        rt_components_board_init();
    #endif

    #if defined(RT_USING_CONSOLE) && defined(RT_USING_DEVICE)
        rt_console_set_device(RT_CONSOLE_DEVICE_NAME);
    #endif

    #if defined(RT_USING_USER_MAIN) && defined(RT_USING_HEAP)
        rt_system_heap_init(rt_heap_begin_get(), rt_heap_end_get());
    #endif
    }


执行到rt_hw_board_init()函数的时候，操作系统完全都还没有涉及到，即rt_hw_board_init()函数所做的工作跟我们以前编写的裸机工程里面的硬件初始化工作是一模一样的。运行完rt_hw_board_init()函数，接下来才慢慢启动操作系统，最后运行创建好的线程。有时候线程创
建好，整个系统跑起来了，可想要的实验现象就是出不来，比如LED不会亮，串口没有输出，LCD没有显示等等。如果是初学者，这个时候就会心急如焚，四处求救，那怎么办？这个时候如何排除是硬件的问题还是系统的问题，这里面有个小小的技巧，即在硬件初始化好之后，顺便测试下硬件，测试方法跟裸机编程一样，具体实现见代
码清单 14‑2的高亮部分。

.. code-block:: c
    :caption: 代码清单 14‑2 rt_hw_board_init()中添加硬件测试函数
    :emphasize-lines: 8-17
    :linenos:

    void rt_hw_board_init()
    {
        /* 初始化SysTick */
        SysTick_Config( SystemCoreClock / RT_TICK_PER_SECOND );

        /* 硬件BSP初始化统统放在这里，比如LED，串口，LCD等 */

        /* 初始化开发板的LED */
        LED_GPIO_Config();                         (1)

        /* 测试硬件是否正常工作 */
        LED1_ON;

        /* 其它硬件初始化和测试 */                    (2)

        /* 让程序停在这里，不再继续往下执行 */          (3)
        while (1);

        /* 调用组件初始化函数 (use INIT_BOARD_EXPORT()) */
    #ifdef RT_USING_COMPONENTS_INIT
        rt_components_board_init();
    #endif

    #if defined(RT_USING_CONSOLE) && defined(RT_USING_DEVICE)
        rt_console_set_device(RT_CONSOLE_DEVICE_NAME);
    #endif

    #if defined(RT_USING_USER_MAIN) && defined(RT_USING_HEAP)
        rt_system_heap_init(rt_heap_begin_get(), rt_heap_end_get());
    #endif
    }


代码清单 14‑2\ **(1)**\ ：初始化硬件后，顺便测试硬件，看下硬件是否正常工作。

代码清单 14‑2\ **(2)**\ ：可以继续添加其它的硬件初始化和测试。硬件确认没有问题之后，硬件测试代码
可删可不删，因为rt_hw_board_init()函数只执行一遍。

代码清单 14‑2\ **(3)**\ ：方便测试硬件好坏，让程序停在这里，不再继续往下执行，当测试完毕后，这个while(1);必须删除。

创建单线程—SRAM静态内存
~~~~~~~~~~~~~~~~~~~~~~~~~~

这里，我们创建一个单线程，线程使用的栈和线程控制块都使用静态内存，即预先定义好的全局变量，这些预先定义好的全局变量都存在内部的SRAM中。

定义线程函数
^^^^^^^^^^^^^^^^^

线程实际上就是一个无限循环且不带返回值的C函数。目前，我们创建一个这样的线程，让开发板上面的LED灯以500ms的频率闪烁，具体实现见代码清单 14‑3。

.. code-block:: c
    :caption: 代码清单 14‑3 20.6.1 定义线程函数
    :linenos:

    static void led1_thread_entry(void* parameter)
    {
        while (1)
        {
            LED1_ON;
            rt_thread_delay(500);   /* 延时500个tick */     (1)

            LED1_OFF;
            rt_thread_delay(500);   /* 延时500个tick */

        }
    }


代码清单 14‑3\ **(1)**\ ：线程里面的延时函数必须使用RT-Thread里面提供的延时函数，并不能使用我们
裸机编程中的那种延时。这两种的延时的区别是RT-Thread里面的延时是阻塞延时，即调用rt_thread_delay()
函数的时候，当前线程会被挂起，调度器会切换到其它就绪的线程，从而实现多线程。如果还是使用裸机编程中
的那种延时，那么整个线程就成为了一个死循环，如果恰好该线程的优先级是最高的，那么系统永远都是在这个
线程中运行，根本无法实现多线程。

目前我们只创建了一个线程，当线程进入延时的时候，因为没有另外就绪的用户线程，那么系统就会进入空闲线程，
空闲线程是RT-Thread系统自己启动的一个线程，优先级最低。当整个系统都没有就绪线程的时候，系统必须保证
有一个线程在运行，空闲线程就是为这个设计的。当用户线程延时到期，又会从空闲线程切换回用户线程。

定义线程栈
^^^^^^^^^^^^^^

在RT-Thread系统中，每一个线程都是独立的，他们的运行环境都单独的保存在他们的栈空间当中。那么在定义好线程函数之后，我们还要为线程定义一个栈，目前我们使用的是静态内存，所以线程栈是一个独立的全局变量，具体见代码清单
14‑4。线程的栈占用的是MCU内部的RAM，当线程越多的时候，需要使用的栈空间就越大，即需要使用的RAM空间就越多。一个MCU能够支持多少线程，就得看你的RAM空间有多少。

.. code-block:: c
    :caption: 代码清单 14‑4 定义线程栈
    :linenos:

    /* 定义线程控栈时要求RT_ALIGN_SIZE个字节对齐 */
    ALIGN(RT_ALIGN_SIZE)
    /* 定义线程栈 */
    static rt_uint8_t rt_led1_thread_stack[1024];


在大多数系统中需要做栈空间地址对齐，例如在ARM体系结构中需要向4字节地址对齐。实现栈对齐的方法为，在定义栈之前，放置一条ALIGN(RT_ALIGN_SIZE)语句，指定接下来定义的变量的地址对齐方式。其中ALIGN是在rtdef.h里面定义的一个宏，根据编译器不一样，该宏的具体定义是不一样的，在
ARM编译器中，该宏的定义具体见代码清单 14‑5。ALIGN宏的形参RT_ALIGB_SIZE是在rtconfig.h中的一个宏，目前定义为4。

.. code-block:: c
    :caption: 代码清单 14‑5ALIGN宏定义
    :linenos:

    /* 只针对ARM 编译器，在其它编译器，该宏的实现会不一样 */
    #define ALIGN(n) \__attribute__((aligned(n)))

定义线程控制块
^^^^^^^^^^^^^^

定义好线程函数和线程栈之后，我们还需要为线程定义一个线程控制块，通常我们称这个线程控制块为线程的身
份证。在C代码上，线程控制块就是一个结构体，里面有非常多的成员，这些成员共同描述了线程的全部信息，
具体见代码清单 14‑6。

.. code-block:: c
    :caption: 代码清单 14‑6 定义线程控制块
    :linenos:

    /* 定义线程控制块 */
    static struct rt_thread led1_thread;


初始化线程
^^^^^^^^^^^^^^^

一个线程的三要素是线程主体函数，线程栈，线程控制块，那么怎么样把这三个要素联合在一起？RT-Thread里面有一个叫线程初始化函数rt_thread_init()，它就是干这个活的。它将线程主体函数，线程栈（静态的）和线程控制块（静态的）这三者联系在一起，让线程可以随时被系统启动，具体见代码清单
14‑7。

.. code-block:: c
    :caption: 代码清单 14‑7 初始化线程
    :linenos:

    rt_thread_init(&led1_thread,                  /* 线程控制块 */     (1)
                "led1",                       /* 线程名字 */           (2)
                led1_thread_entry,            /* 线程入口函数 */       (3)
                RT_NULL,                      /* 线程入口函数参数 */    (4)
                &rt_led1_thread_stack[0],     /* 线程栈起始地址 */      (5)
                sizeof(rt_led1_thread_stack), /* 线程栈大小 */          (6)
                3,                            /* 线程的优先级 */        (7)
                20);                          /* 线程时间片 */          (8)


代码清单 14‑7\ **(1)**\ ：线程控制块指针，在使用静态内存的时候，需要给线程初始化函数
rt_thread_init()传递预先定义好的线程控制块的指针。在使用动态内存的时候，线程创建函数
rt_thread_create()会返回一个指针指向线程控制块，该线程控制块是rt_thread_create()函数
里面动态分配的一块内存。

代码清单 14‑7\ **(2)**\ ：线程名字，字符串形式，最大长度由rtconfig.h中定义的RT_NAME_MAX宏指定，多余部分会被自动截掉。

代码清单 14‑7\ **(3)**\ ：线程入口函数，即线程函数的名称。

代码清单 14‑7\ **(4)**\ ：线程入口函数形参，不用的时候配置为0即可。

代码清单 14‑7\ **(5)**\ ：线程栈起始地址，只有在使用静态内存的时候才需要提供，在使用动态内存的时候会根据提供的线程栈大小自动创建。

代码清单 14‑7\ **(6)**\ ：线程栈大小，单位为字节。

代码清单 14‑7\ **(7)**\ ：线程的优先级。优先级范围根据rtconfig.h中的宏RT_THREAD_PRIORITY_MAX
决定，最多支持256个优先级，目前配置为32。在RT-Thread中，数值越小优先级越高，0代表最高优先级。

代码清单 14‑7\ **(8)**\
：线程的时间片大小。时间片的单位是操作系统的时钟节拍。当系统中存在相同优先级线程时，这个参数指定线程一次调度能够运行的最大时间长度。这个时间片运行结束时，调度器自动选择下一个就绪态的同优先级线程进行运行。如果同一个优先级下只有一个线程，那么时间片这个形参就不起作用。

启动线程
^^^^^^^^^^^^

当线程初始化好后，是处于线程初始态（RT_THREAD_INIT），并不能够参与操作系统的调度，只有当线程进入
就绪态（RT_THREAD_READY）之后才能参与操作系统的调度。线程由初始态进入就绪态可由函数
rt_thread_startup()来实现，具体见代码清单 14‑8。

.. code-block:: c
    :caption: 代码清单 14‑8启动线程
    :linenos:

    /* 启动线程，开启调度 */
    rt_thread_startup(&led1_thread);


main.c文件内容全貌
^^^^^^^^^^^^^^^^^^^^^^^^

现在我们把线程主体，线程栈，线程控制块这三部分代码统一放到main.c中，具体内容见代码清单 14‑9。

.. code-block:: c
    :caption: 代码清单 14‑9 main.c文件内容全貌
    :linenos:

    /*
    *************************************************************************
    *                             包含的头文件
    *************************************************************************
    */
    #include "board.h"
    #include "rtthread.h"


    /*
    *************************************************************************
    *                               变量
    *************************************************************************
    */
    /* 定义线程控制块 */
    static struct rt_thread led1_thread;

    /* 定义线程控栈时要求RT_ALIGN_SIZE个字节对齐 */
    ALIGN(RT_ALIGN_SIZE)
    /* 定义线程栈 */
    static rt_uint8_t rt_led1_thread_stack[1024];
    /*
    *************************************************************************
    *                             函数声明
    *************************************************************************
    */
    static void led1_thread_entry(void* parameter);


    /*
    *************************************************************************
    *                             main 函数
    *************************************************************************
    */
    /**
    * @brief  主函数
    * @param  无
    * @retval 无
    */
    int main(void)
    {
        /*
        * 开发板硬件初始化，RTT系统初始化已经在main函数之前完成，
        * 即在component.c文件中的rtthread_startup()函数中完成了。
        * 所以在main函数中，只需要创建线程和启动线程即可。
        */

        rt_thread_init(&led1_thread,                 /* 线程控制块 */
                    "led1",                       /* 线程名字 */
                    led1_thread_entry,            /* 线程入口函数 */
                    RT_NULL,                      /* 线程入口函数参数 */
                    &rt_led1_thread_stack[0],     /* 线程栈起始地址 */
                    sizeof(rt_led1_thread_stack), /* 线程栈大小 */
                    3,                            /* 线程的优先级 */
                    20);                          /* 线程时间片 */
        rt_thread_startup(&led1_thread);             /* 启动线程，开启调度 */
    }

    /*
    *************************************************************************
    *                             线程定义
    *************************************************************************
    */

    static void led1_thread_entry(void* parameter)
    {
        while (1)
        {
            LED1_ON;
            rt_thread_delay(500);   /* 延时500个tick */

            LED1_OFF;
            rt_thread_delay(500);   /* 延时500个tick */

        }
    }

    /********************************END OF FILE****************************/


下载验证
~~~~~~~~~~~~

将程序编译好，用DAP仿真器把程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），可以看到板子上面的LED灯已经在闪烁，说明我们创建的单线程（使用静态内存）已经跑起来了。

在当前这个例程，线程的栈，线程的控制块用的都是静态内存，必须由用户预先定义，这种方法我们在使用RT-Thread的时候用的比较少，通常的方法是在线程创建的时候动态的分配线程栈和线程控制块的内存空间，接下来我们讲解下“创建单线程—SRAM动态内存”的方法。

创建单线程—SRAM动态内存
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这里，我们创建一个单线程，线程使用的栈和线程控制块是在创建线程的时候RT-Thread动态分配的，并不是预先定义好的全局变量。那这些动态的内存堆是从哪里来？继续往下看。

动态内存空间的堆从哪里来
^^^^^^^^^^^^^^^^^^^^^^^^^^

在创建单线程—SRAM静态内存的例程中，线程控制块和线程栈的内存空间都是从内部的SRAM里面分配的，具体分配到哪个地址由编译器决定。现在我们开始使用动态内存，即堆，其实堆也是内存，也属于SRAM。现在我们的做法是在SRAM里面定义一个大数组供RT-
Thread的动态内存分配函数使用，这些代码在board.c开头实现，具体见代码清单 14‑10。

.. code-block:: c
    :caption: 代码清单 14‑10定义RT-Thread的堆到内部SRAM
    :linenos:

    #if defined(RT_USING_USER_MAIN) && defined(RT_USING_HEAP)           (1)

    /* 从内部SRAM（即DTCM）里面分配一部分静态内存来作为RT-Thread的堆空间，这里配置为4KB */
    #define RT_HEAP_SIZE 1024
    static uint32_t rt_heap[RT_HEAP_SIZE];                               (2)
    RT_WEAK void *rt_heap_begin_get(void)                                (3)
    {
        return rt_heap;
    }

    RT_WEAK void *rt_heap_end_get(void)                                  (4)
    {
        return rt_heap + RT_HEAP_SIZE;
    }
    #endif


    /* 该部分代码截取自函数rt_hw_board_init() */
    #if defined(RT_USING_USER_MAIN) && defined(RT_USING_HEAP)
    rt_system_heap_init(rt_heap_begin_get(), rt_heap_end_get());         (5)
    #endif


代码清单 14‑10 **(1)** ：RT_USING_USER_MAIN 和RT_USING_HEAP这两个宏，在rtconfig.h定义，
RT_USING_USER_MAIN默认开启，RT_USING_HEAP在使用动态内存时需要开启。

代码清单 14‑10\ **(2)** ：从内部SRAMM里面定义一个静态数组rt_heap，大小由RT_HEAP_SIZE这个宏决定，
目前定义为4KB。定义的堆大小不能超过内部SRAM的总大小。

代码清单 14‑10\ **(3)** ：rt_heap_begin_get()用于获取堆的起始地址。

代码清单 14‑10\ **(4)** ：rt_heap_end_get()用于获取堆的结束地址。

代码清单 14‑10\ **(5)**
：rt_system_heap_init()根据堆的起始地址和结束地址进行堆的初始化。rt_system_heap_init()需要两个形参，一个是堆的起始地址，另外一个是堆的结束地址，如果我们使用外部SDRAM作为堆，这两个形参直接传入外部SDRAM地址范围内的地址即可。


定义线程函数
^^^^^^^^^^^^^^^

使用动态内存的时候，线程的主体函数与使用静态内存时是一样的，具体见代码清单 14‑11。

.. code-block:: c
    :caption: 代码清单 14‑11定义线程函数
    :linenos:

    static void led1_thread_entry(void* parameter)
    {
        while (1)
        {
            LED1_ON;
            rt_thread_delay(500);   /* 延时500个tick */

            LED1_OFF;
            rt_thread_delay(500);   /* 延时500个tick */

        }
    }



定义线程栈
^^^^^^^^^^^^^^^

使用动态内存的时候，线程栈在线程创建的时候创建，不用跟使用静态内存那样要预先定义好一个全局的静态的栈空间。

定义线程控制块指针
^^^^^^^^^^^^^^^^^^^^^^^^^

使用动态内存时候，不用跟使用静态内存那样要预先定义好一个全局的静态的线程控制块空间。线程控制块是在
线程创建的时候创建，线程创建函数会返回一个指针，用于指向线程控制块，所以要预先为线程栈定义一个线程
控制块指针，具体见代码清单 14‑12。

.. code-block:: c
    :caption: 代码清单 14‑12定义线程控制块指针
    :linenos:

    /* 定义线程控制块指针 */
    static  rt_thread_t led1_thread = RT_NULL;



创建线程
^^^^^^^^^^^^

使用静态内存时，使用rt_thread_init()来初始化一个线程，使用动态内存的时，使用rt_thread_create()
函数来创建一个线程，两者的函数名不一样，具体的形参也有区别，具体见代码清单 14‑13。

.. code-block:: c
    :caption: 代码清单 14‑13创建线程
    :linenos:

    led1_thread =                              /* 线程控制块指针 */   (1)
    rt_thread_create( "led1",                  /* 线程名字 */        (2)
                    led1_thread_entry,   /* 线程入口函数 */         (3)
                    RT_NULL,             /* 线程入口函数参数 */      (4)
                    512,                 /* 线程栈大小 */           (5)
                    3,                   /* 线程的优先级 */         (6)
                    20);                 /* 线程时间片 */           (7)


代码清单 14‑13\ **(1)**\ ：线程控制块指针，在使用静态内存的时候，需要给线程初始化函数
rt_thread_init()传递预先定义好的线程控制块的指针。在使用动态内存的时候，线程创建函数
rt_thread_create()会返回一个指针指向线程控制块，该线程控制块是rt_thread_create()函数里面动态分配的一块内存。

代码清单 14‑13\ **(2)**\ ：线程名字，字符串形式，最大长度由rtconfig.h中定义的RT_NAME_MAX宏指定，多余部分会被自动截掉。

代码清单 14‑13\ **(3)**\ ：线程入口函数，即线程函数的名称。

代码清单 14‑13\ **(4)**\ ：线程入口函数形参，不用的时候配置为0即可。

代码清单 14‑13\ **(5)**\ ：线程栈大小，单位为字节。使用动态内存创建线程时，与使用静态内存线程初始
化函数不一样，不再需要提供线程栈的起始地址，只需要知道线程栈的大小即可，因为它是在线程创建时动态分配的。

代码清单 14‑13\ **(6)**\ ：线程的优先级。优先级范围根据rtconfig.h中的宏RT_THREAD_PRIORITY_MAX
决定，最多支持256个优先级，目前配置为32。在RT-Thread中，数值越小优先级越高，0代表最高优先级。

代码清单 14‑13\ **(7)**\
：线程的时间片大小。时间片的单位是操作系统的时钟节拍。当系统中存在相同优先级线程时，这个参数指定线程一次调度能够运行的最大时间长度。这个时间片运行结束时，调度器自动选择下一个就绪态的同优先级线程进行运行。如果同一个优先级下只有一个线程，那么时间片这个形参就不起作用。


启动线程
^^^^^^^^^^^^

当线程创建好后，是处于线程初始态（RT_THREAD_INIT），并不能够参与操作系统的
调度，只有当线程进入就绪态（RT_THREAD_READY）之后才能参与操作系统的调度。
线程由初始态进入就绪态可由函数rt_thread_startup()来实现，具体见代码清单 14‑14。

.. code-block:: c
    :caption: 代码清单 14‑14 启动线程
    :linenos:

    if (led1_thread != RT_NULL)
        rt_thread_startup(led1_thread);         /* 启动线程，开启调度 */
    else
        return -1;


main.c文件内容全貌
^^^^^^^^^^^^^^^^^^^^^

现在我们把线程主体，线程栈指针，线程控制块这三部分代码统一放到main.c中，具体见代码清单 14‑15。

.. code-block:: c
    :caption: 代码清单 14‑15main.c文件内容全貌
    :linenos:

    #if defined(RT_USING_USER_MAIN) && defined(RT_USING_HEAP)
    #define RT_HEAP_SIZE 1024
    /* 从内部SRAM里面分配一部分静态内存来作为rtt的堆空间，这里配置为4KB */
    static uint32_t rt_heap[RT_HEAP_SIZE];
    RT_WEAK void *rt_heap_begin_get(void)
    {
        return rt_heap;
    }

    RT_WEAK void *rt_heap_end_get(void)
    {
        return rt_heap + RT_HEAP_SIZE;
    }
    #endif

    /* 该部分代码截取自函数rt_hw_board_init() */
    #if defined(RT_USING_USER_MAIN) && defined(RT_USING_HEAP)
    //rt_system_heap_init((void*)HEAP_BEGIN, (void*)SRAM_END);
    rt_system_heap_init(rt_heap_begin_get(), rt_heap_end_get());
    #endif


    /*
    *************************************************************************
    *                             包含的头文件
    *************************************************************************
    */
    #include "board.h"
    #include "rtthread.h"


    /*
    *************************************************************************
    *                               变量
    *************************************************************************
    */
    /* 定义线程控制块指针 */
    static rt_thread_t led1_thread = RT_NULL;

    /*
    *************************************************************************
    *                             函数声明
    *************************************************************************
    */
    static void led1_thread_entry(void* parameter);


    /*
    *************************************************************************
    *                             main 函数
    *************************************************************************
    */
    /**
    * @brief  主函数
    * @param  无
    * @retval 无
    */
    int main(void)
    {
        /*
        * 开发板硬件初始化，RTT系统初始化已经在main函数之前完成，
        * 即在component.c文件中的rtthread_startup()函数中完成了。
        * 所以在main函数中，只需要创建线程和启动线程即可。
        */

        led1_thread =                          /* 线程控制块指针 */
        rt_thread_create( "led1",              /* 线程名字 */
                        led1_thread_entry,   /* 线程入口函数 */
                        RT_NULL,             /* 线程入口函数参数 */
                        512,                 /* 线程栈大小 */
                        3,                   /* 线程的优先级 */
                        20);                 /* 线程时间片 */

        /* 启动线程，开启调度 */
        if (led1_thread != RT_NULL)
            rt_thread_startup(led1_thread);
        else
            return -1;
    }

    /*
    *************************************************************************
    *                             线程定义
    *************************************************************************
    */

    static void led1_thread_entry(void* parameter)
    {
        while (1)
        {
            LED1_ON;
            rt_thread_delay(500);   /* 延时500个tick */

            LED1_OFF;
            rt_thread_delay(500);   /* 延时500个tick */

        }
    }

    /*******************************END OF FILE****************************/


下载验证
~~~~~~~~~~~~

将程序编译好，用DAP仿真器把程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），可以看到板子上面的LED灯已经在闪烁，说明我们创建的单线程（使用动态内存）已经跑起来了。在往后的实验中，我们创建内核对象均采用动态内存分配方案。

创建多线程—SRAM动态内存
~~~~~~~~~~~~~~~~~~~~~~~~~~

创建多线程只需要按照创建单线程的套路依葫芦画瓢即可，接下来我们创建两个线程，线程1让一个LED灯闪烁，
线程2让另外一个LED闪烁，两个LED闪烁的频率不一样，具体实现见代码清单 10‑16的高亮部分，两个线程的
优先级不一样。

.. code-block:: c
    :caption: 代码清单 14‑16创建多线程—SRAM动态内存
    :emphasize-lines: 17,25,59-71,93-102
    :linenos:

    /*
    *************************************************************************
    *                             包含的头文件
    *************************************************************************
    */
    #include "board.h"
    #include "rtthread.h"


    /*
    *************************************************************************
    *                               变量
    *************************************************************************
    */

    /* 定义线程控制块指针 */
    static rt_thread_t led1_thread = RT_NULL;
    static rt_thread_t led2_thread = RT_NULL;

    /*
    *************************************************************************
    *                             函数声明
    *************************************************************************
    */
    static void led1_thread_entry(void* parameter);
    static void led2_thread_entry(void* parameter);


    /*
    *************************************************************************
    *                             main 函数
    *************************************************************************
    */
    /**
    * @brief  主函数
    * @param  无
    * @retval 无
    */
    int main(void)
    {
        /*
        * 开发板硬件初始化，RTT系统初始化已经在main函数之前完成，
        * 即在component.c文件中的rtthread_startup()函数中完成了。
        * 所以在main函数中，只需要创建线程和启动线程即可。
        */

        led1_thread =                          /* 线程控制块指针 */
        rt_thread_create( "led1",              /* 线程名字 */
                        led1_thread_entry,   /* 线程入口函数 */
                        RT_NULL,             /* 线程入口函数参数 */
                        512,                 /* 线程栈大小 */
                        3,                   /* 线程的优先级 */
                        20);                 /* 线程时间片 */

        /* 启动线程，开启调度 */
        if (led1_thread != RT_NULL)
            rt_thread_startup(led1_thread);
        else
            return -1;

        led2_thread =                          /* 线程控制块指针 */
        rt_thread_create( "led2",              /* 线程名字 */
                        led2_thread_entry,   /* 线程入口函数 */
                        RT_NULL,             /* 线程入口函数参数 */
                        512,                 /* 线程栈大小 */
                        4,                   /* 线程的优先级 */
                        20);                 /* 线程时间片 */

        /* 启动线程，开启调度 */
        if (led2_thread != RT_NULL)
            rt_thread_startup(led2_thread);
        else
            return -1;
    }

    /*
    *************************************************************************
    *                             线程定义
    *************************************************************************
    */

    static void led1_thread_entry(void* parameter)
    {
        while (1)
        {
            LED1_ON;
            rt_thread_delay(500);   /* 延时500个tick */

            LED1_OFF;
            rt_thread_delay(500);   /* 延时500个tick */

        }
    }

    static void led2_thread_entry(void* parameter)
    {
        while (1)
        {
            LED2_ON;
            rt_thread_delay(300);   /* 延时300个tick */
            LED2_OFF;
            rt_thread_delay(300);   /* 延时300个tick */
        }
    }
    /****************************END OF FILE****************************/


目前多线程我们只创建了两个，如果要创建3个、4个甚至更多都是同样的套路，容易忽略的地方是线程栈的大小，每个线程的优先级。大的线程，栈空间要设置大一点，重要的线程优先级要设置的高一点。


下载验证
~~~~~~~~~~

将程序编译好，用DAP仿真器把程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板
子都配套有对应的程序），可以看到板子上面的两个LED灯以不同的频率在闪烁，说明我们创建的单线程（使
动态内存）已经跑起来了。在往后的实验中，我们创建内核对象均采用动态内存分配方案。
