.. vim: syntax=rst

RT-Thread的启动流程
=====

在目前的RTOS中，主要有两种比较流行的启动方式，暂时还没有看到第三种，接下来我将通过伪代码的方式来讲解下这两种启动方式的区别，然后再具体分析下RT-Thread的启动流程。

万事俱备，只欠东风
~~~~~~~~~

第一种我称之为万事俱备，只欠东风法。这种方法是在main函数中将硬件初始化，RTOS系统初始化，所有线程的创建这些都弄好，这个我称之为万事都已经准备好。最后只欠一道东风，即启动RTOS的调度器，开始多线程的调度，具体的伪代码实现见代码清单 16‑1。

.. code-block:: c
    :caption: 代码清单 16‑1 万事俱备，只欠东风法伪代码实现
    :linenos: 

    int main (void)
    {
        /* 硬件初始化 */
        HardWare_Init();                                         (1)

        /* RTOS 系统初始化 */
        RTOS_Init();                                             (2)

        /* 创建线程1，但线程1不会执行，因为调度器还没有开启 */       (3)
        RTOS_ThreadCreate(Task1);
        /* 创建线程2，但线程2不会执行，因为调度器还没有开启 */
        RTOS_ThreadCreate(Task2);

        /* ......继续创建各种线程 */

        /* 启动RTOS，开始调度 */
        RTOS_Start();                                           (4)
    }

    void Thread1( void *arg )                                   (5)
    {
        while(1)
        {
            /* 线程实体，必须有阻塞的情况出现 */
        }
    }

    void Thread2( void *arg )                                   (6)
    {
        while(1)
        {
            /* 线程实体，必须有阻塞的情况出现 */
        }
    }


代码清单 16‑1\ **(1)**\ ：硬件初始化。硬件初始化这一步还属于裸机的范畴，我们可以把需要使用到的硬件都初始化好而且测试好，确保无误。

代码清单 16‑1\ **(2)**\ ：RTOS系统初始化。比如RTOS里面的全局变量的初始化，空闲线程的创建等。不同的RTOS，它们的初始化有细微的差别。

代码清单 16‑1\ **(3)**\ ：创建各种线程。这里把所有要用到的线程都创建好，但还不会进入调度，因为这个时候RTOS的调度器还没有开启。

代码清单 16‑1\ **(4)**\ ：启动RTOS调度器，开始线程调度。这个时候调度器就从刚刚创建好的线程中选择一个优先级最高的线程开始运行。

代码清单 16‑1\ **(5) (6)**\ ：线程实体通常是一个不带返回值的无限循环的C函数，函数体必须有阻塞的情况出现，不然线程（如果优先权恰好是最高）会一直在while循环里面执行，导致其它线程没有执行的机会。

小心翼翼，十分谨慎
~~~~~~~~~

第二种我称之为小心翼翼，十分谨慎法。这种方法是在main函数中将硬件和RTOS系统先初始化好，然后创建一个启动线程后就启动调度器，然后在启动线程里面创建各种应用线程，当所有线程都创建成功后，启动线程把自己删除，具体的伪代码实现见代码清单 16‑2。

.. code-block:: c
    :caption: 代码清单 16‑2 小心翼翼，十分谨慎法伪代码实现
    :linenos: 

    int main (void)
    {
        /* 硬件初始化 */
        HardWare_Init();                                     (1)

        /* RTOS 系统初始化 */
        RTOS_Init();                                         (2)

        /* 创建一个线程 */
        RTOS_ThreadCreate(AppThreadStart);                   (3)

        /* 启动RTOS，开始调度 */
        RTOS_Start();                                        (4)
    }

    /* 起始线程，在里面创建线程 */
    void AppThreadStart( void *arg )                         (5)
    {
        /* 创建线程1，然后执行 */
        RTOS_ThreadCreate(Thread1);                          (6)

        /* 当线程1阻塞时，继续创建线程2，然后执行 */
        RTOS_ThreadCreate(Thread2);

        /* ......继续创建各种线程 */

        /* 当线程创建完成，关闭起始线程 */
        RTOSThreadClose(AppThreadStart);                     (7)
    }

    void Thread1( void *arg )                                (8)
    {
        while (1)
        {
            /* 线程实体，必须有阻塞的情况出现 */
        }
    }

    void Thread2( void *arg )                                (9)
    {
        while (1)
        {
            /* 线程实体，必须有阻塞的情况出现 */
        }
    }


代码清单 16‑2 **(1)**\ ：硬件初始化。来到硬件初始化这一步还属于裸机的范畴，我们可以把需要使用到的硬件都初始化好而且测试好，确保无误。

代码清单 16‑2 **(2)**\ ：RTOS系统初始化。比如RTOS里面的全局变量的初始化，空闲线程的创建等。不同的RTOS，它们的初始化有细微的差别。

代码清单 16‑2 **(3)**\ ：创建一个开始线程。然后在这个初始线程里面创建各种应用线程。

代码清单 16‑2 **(4)**\ ：启动RTOS调度器，开始线程调度。这个时候调度器就去执行刚刚创建好的初始线程。

代码清单 16‑2 **(5)**\ ：我们通常说线程是一个不带返回值的无限循环的C函数，但是因为初始线程的特殊性，它不能是无限循环的，只执行一次后就关闭。在初始线程里面我们创建我们需要的各种线程。

代码清单 16‑2 **(6)**\ ：创建线程。每创建一个线程后它都将进入就绪态，系统会进行一次调度，如果新创建的线程的优先级比初始线程的优先级高的话，那将去执行新创建的线程，当新的线程阻塞时再回到初始线程被打断的地方继续执行。反之，则继续往下创建新的线程，直到所有线程创建完成。

代码清单 16‑2 **(7)**\ ：各种应用线程创建完成后，初始线程自己关闭自己，使命完成。

代码清单 16‑2 **(8) (9)**\ ：线程实体通常是一个不带返回值的无限循环的C函数，函数体必须有阻塞的情况出现，不然线程（如果优先权恰好是最高）会一直在while循环里面执行，其它线程没有执行的机会。

孰优孰劣
~~~~

那有关这两种方法孰优孰劣？我暂时没发现，我个人还是比较喜欢使用第一种。ucos第一种和第二种都可以使用，由用户选择，freertos和RT-Thread则默认使用第二种。接下来我们详细讲解下RT-Thread的启动流程，虽然说RT-Thread用的是第二种，但是RT-
Thread又拓展了main函数，稍微又高级了点。

.. _rt-thread的启动流程-1:

RT-Thread的启动流程
~~~~~~~~~~~~~~

当你拿到一个移植好的RT-Thread工程的时候，你去看main函数，只能在main函数里面看到创建线程和启动线程的代码，硬件初始化，系统初始化，启动调度器等信息都看不到。那是因为RT-Thread拓展了main函数，在main函数之前把这些工作都做好了。

我们知道，在系统上电的时候第一个执行的是启动文件里面由汇编编写的复位函数Reset_Handler，具体见代码清单 16‑3。复位函数的最后会调用C库函数__main，具体见代码清单 16‑3的高亮部分。__main函数的主要工作是初始化系统的堆和栈，最后调用C中的main函数，从而去到C的世界。

.. code-block:: c
    :caption: 代码清单 16‑3 Reset_Handler函数
    :emphasize-lines: 15
    :linenos: 

    Reset_Handler   PROC
                    EXPORT  Reset_Handler             [WEAK]
                    IMPORT  SystemInit
                    IMPORT  __main

                    CPSID   I               ; 关中断
                    LDR     R0, =0xE000ED08
                    LDR     R1, =__Vectors
                    STR     R1, [R0]
                    LDR     R2, [R1]
                    MSR     MSP, R2
                    LDR     R0, =SystemInit
                    BLX     R0
                    CPSIE   i               ; 开中断
                    LDR     R0, =__main            
                    BX      R0
                    ENDP


但当我们硬件仿真RT-Thread工程的时候，单步执行完__main之后，并不是跳转到C中的main函数，而是跳转到component.c中的$Sub$$main函数，这是为什么？因为RT-Thread使用编译器（这里仅讲解KEIL，IAR或者GCC稍微有点区别，但是原理是一样的）自带的$Sub$$
和$Super$$这两个符号来扩展了main函数，使用$Sub$$main可以在执行main之前先执行$Sub$$main，在$Sub$$main函数中我们可以先执行一些预操作，当做完这些预操作之后最终还是要执行main函数，这个就通过调用$Super$$main来实现。当需要扩展的函数不是main
的时候，只需要将main换成你要扩展的函数名即可，即$Sub$$function和$Super$$function，具体如何使用这两个扩展符号的伪代码见代码清单 16‑4。

.. code-block:: c
    :caption: 代码清单 16‑4 $Sub$$和$Super$$的使用方法
    :linenos: 

    extern void ExtraFunc(void);     /* 用户自己实现的外部函数*/

    void $Sub$$function(void)
    {
        ExtraFunc();                 /* 做一些其它的设置工作 */
        $Super$$function();          /* 回到原始的function函数 */
    }

    /* 在执行function函数执行会先执行function的扩展函数$Sub$$function，在扩展函数里面执行一些扩展的操作，
    当扩展操作完成后，最后必须调用$Super$$function函数通过它回到我们原始的function函数 */
    void function(void)
    {
        /* 函数实体 */
    }


$Sub$$main函数
------------

知道了$Sub$$和$Super$$的用法之后，我们回到RT-Thread component.c文件中的的$Sub$$main，具体实现见代码清单 16‑5。

.. code-block:: c
    :caption: 代码清单 16‑5 main的扩展函数$Sub$$main
    :linenos: 

    int $Sub$$main(void)
    {
        rt_hw_interrupt_disable();                        (1)
        rtthread_startup();                               (2)
        return 0;
    }

代码清单 16‑5\ **(1)**\ ：关闭中断，除了硬FAULT和NMI可以响应外，其它统统关掉。该函数是在接口文件contex_rvds.S中由汇编实现的，具体见代码清单 16‑6。

.. code-block:: c
    :caption: 代码清单 16‑6 硬件中断失能和使能函数定义
    :linenos: 

    ;/*
    ; * rt_base_t rt_hw_interrupt_disable();
    ; */
    rt_hw_interrupt_disable    PROC
        EXPORT  rt_hw_interrupt_disable
        MRS     r0, PRIMASK
        CPSID   I
        BX      LR
        ENDP

    ;/*
    ; * void rt_hw_interrupt_enable(rt_base_t level);
    ; */
    rt_hw_interrupt_enable    PROC
        EXPORT  rt_hw_interrupt_enable
        MSR     PRIMASK, r0
        BX      LR
        ENDP

在Cortex-M内核中，为了快速地开关中断， 专门设置了一条 CPS 指令，有 4 种用法，具体见代码清单 16‑7。很显然，RT-Thread里面快速关中断的方法就是用了Cortex-M中的CPS指令。

.. code-block:: c
    :caption: 代码清单 16‑7 Cortex-M 内核中快速关中断指令CPS的用法
    :linenos: 

    CPSID I ;PRIMASK=1， ;关中断，只有FAULT和NMI可以响应
    CPSIE I ;PRIMASK=0， ;开中断，只有FAULT和NMI可以响应
    CPSID F ;FAULTMASK=1, ;关异常，只有NMI可以响应
    CPSIE F ;FAULTMASK=0 ;开异常，只有NMI可以响应

rtthread_startup()函数
--------------------

代码清单 16‑5\ **(2)**\ ：rtthread_startup()函数也在componet.c里面实现，具体实现见代码清单 16‑8。

.. code-block:: c
    :caption: 代码清单 16‑8 rtthread_startup()函数定义
    :linenos: 

    int rtthread_startup(void)
    {
        /* 关闭中断 */
        rt_hw_interrupt_disable();                               (1)

        /* 板级硬件初始化
        * 注意: 在板级硬件初始化函数中把要堆初始化好(前提是使用动态内存)
        */
        rt_hw_board_init();                                      (2)

        /* 打印 RT-Thread 版本号 */
        rt_show_version();                                       (3)

        /* 定时器初始化 */
        rt_system_timer_init();                                  (4)

        /* 调度器初始化 */
        rt_system_scheduler_init();                              (5)

    #ifdef RT_USING_SIGNALS
        /* 信号量初始化 */
        rt_system_signal_init();                                 (6)
    #endif

        /* 创建初始线程 */
        rt_application_init();                                   (7)

        /* 定时器线程初始化 */
        rt_system_timer_thread_init();                           (8)

        /* 空闲线程初始化 */
        rt_thread_idle_init();                                   (9)

        /* 启动调度器 */
        rt_system_scheduler_start();                             (10)

        /* 绝对不会回到这里 */
        return 0;                                                (11)
    }


代码清单 16‑8 **(1)**\ ：关中断。在硬件初始化之前把中断关闭是一个很好的选择，如果没有关闭中断，在接下来的硬件初始化中如果某些外设开启了中断，那么它就有可能会响应，可是后面的RTOS系统初始化，调度器初始化这些都还没有完成，显然这些中断我们是不希望响应的。

代码清单 16‑8 **(2)**\ ：板级硬件初始化。RT-Thread把板级硬件相关的初始化都放在rt_hw_board_int()函数里面完成，该函数需要用户在board.c实现。我们通常在还没有进入系统相关的操作前把硬件都初始化好且测试好，然后在继续往下执行系统相关的操作。

代码清单 16‑8 **(3)**\ ：打印RT-Thread的版本号，该函数在kservice.c中实现，具体见代码清单 16‑9。rt_show_version()函数是通过调用rt_kprintf函数向控制台打印RT-
Thread版本相关的信息，要想成功打印，必须重映射一个控制台到rt_kprintf函数，具体实现参考上一章《重映射串口到rt_kprintf函数》。如果没有重映射控制台到rt_kprintf函数，该函数也不会阻塞，而是打印输出为空。

.. code-block:: c
    :caption: 代码清单 16‑9 rt_show_version()函数
    :linenos: 

    void rt_show_version(void)
    {
        rt_kprintf("\n \\ | /\n");
        rt_kprintf("- RT -     Thread Operating System\n");
        rt_kprintf(" / | \\     %d.%d.%d build %s\n",
                RT_VERSION, RT_SUBVERSION, RT_REVISION, __DATE__);
        rt_kprintf(" 2006 - 2018 Copyright by rt-thread team\n");
    }

代码清单 16‑8 **(4)**\ ：定时器初始化，实际上就是初始化一个全局的定时器列表，列表里面存放的是处于延时状态的线程。

代码清单 16‑8 **(5)**\ ：调度器初始化。

代码清单 16‑8 **(6)**\ ：信号初始化，RT_USING_SIGNALS这个宏默认不定义。

代码清单 16‑8 **(7)**\ ：创建初始线程。前面我们说过，RT-
Thread的启动流程是这样的：即先创建一个初始线程，等调度器启动之后，在这个初始线程里面创建各种应用线程，当所有应用线程都成功创建好后，初始线程就把自己关闭。那么这个初始线程就在rt_application_init()里面创建，该函数也在component.c里面定义，具体实现见代码清单
16‑10。

rt_application_init()函数
------------------------

.. code-block:: c
    :caption: 代码清单 16‑10 创建初始线程
    :linenos: 


    /* 使用动态内存时需要用到的宏：rt_config.h中定义 *///                 (2)
    #define RT_USING_USER_MAIN
    #define RT_MAIN_THREAD_STACK_SIZE     256
    #define RT_THREAD_PRIORITY_MAX        32

    /* 使用静态内存时需要用到的宏和变量：在component.c定义 */ //           (4)
    #ifdef RT_USING_USER_MAIN
    #ifndef RT_MAIN_THREAD_STACK_SIZE
    #define RT_MAIN_THREAD_STACK_SIZE     2048
    #endif
    #endif

    #ifndef RT_USING_HEAP
    ALIGN(8)
    static rt_uint8_t main_stack[RT_MAIN_THREAD_STACK_SIZE];
    struct rt_thread main_thread;
    #endif

    void rt_application_init(void)
    {
        rt_thread_t tid;

    #ifdef RT_USING_HEAP
        /* 使用动态内存 */ //                                          (1)
        tid =
            rt_thread_create("main",
                            main_thread_entry,
                            RT_NULL,
                            RT_MAIN_THREAD_STACK_SIZE,
                            RT_THREAD_PRIORITY_MAX / 3,     (初始线程优先级)
                            20);
        RT_ASSERT(tid != RT_NULL);
    #else
        /* 使用静态内存 */ //                                          (3)
        rt_err_t result;

        tid = &main_thread;
        result =
            rt_thread_init(tid,
                        "main",
                        main_thread_entry,
                        RT_NULL,
                        main_stack,
                        sizeof(main_stack),
                        RT_THREAD_PRIORITY_MAX / 3,      (初始线程优先级)
                        20);
        RT_ASSERT(result == RT_EOK);
        (void)result;
    #endif

        /* 启动线程 */
        rt_thread_startup(tid);//                                  (6)
    }

    /* main线程 */
    void main_thread_entry(void *parameter)//                      (5)
    {
        extern int main(void);
        extern int $Super$$main(void);

        /* RT-Thread 组件初始化 */
        rt_components_init();

        /* 调用$Super$$main()函数，去到main */
        $Super$$main();
        
    }


代码清单 16‑10\ **(1)**\ ：创建初始线程的时候，分使用动态内存和静态内存两种情况，通常我们使用动态内存，有关动态内存需要用到的宏定义具体见代码清单 16‑10 **(2)**\ 。

代码清单 16‑10\ **(3)**\ ：创建初始线程的时候，分使用动态内存和静态内存两种情况，这里是使用静态内存，有关静态内存需要用到的宏定义具体见代码清单 16‑10 **(4)**\ 。

$Super$$main()函数
----------------

代码清单 16‑10\ **(5)**\ ：初始线程入口。该函数除了调用rt_components_init()函数进行RT-Thread的组件初始化外，最终是调用main的扩展函数$Super$$main()回到main函数。这个是必须的，因为我们一开始在进入main函数之前，通过$Sub$$ma
in()函数扩展了main函数，做了一些硬件初始化，RTOS系统初始化的工作，当这些工作做完之后最终还是要回到main函数，那只能通过调用$Super$$main()函数来实现。$Sub$$和$Super$$是MDK自带的用来扩展函数的符号，通常是成对使用。

代码清单 16‑10\ **(6)**\ ：启动初始线程，这个时候初始线程还不会立即被执行，因为调度器还没有启动。

代码清单 16‑10\ **(初始线程优先级)**\ ：初始线程的优先级默认配置为最大优先级/3。控制最大优先级的宏RT_THREAD_PRIORITY_MAX在rt_config.h中定义，目前配置为32 ，那初始线程的优先级即是10，那在初始线程里面创建的各种应用线程的优先级又该如何配置？分三种
情况：1、应用线程的优先级比初始线程的优先级高，那创建完后立马去执行刚刚创建的应用线程，当应用线程被阻塞时，继续回到初始线程被打断的地方继续往下执行，直到所有应用线程创建完成，最后初始线程把自己关闭，完成自己的使命；2、应用线程的优先级与初始线程的优先级一样，那创建完后根据线程的时间片来执行，直到所
有应用线程创建完成，最后初始线程把自己关闭，完成自己的使命；3、应用线程的优先级比初始线程的优先级低，那创建完后线程不会被执行，如果还有应用线程紧接着创建应用线程，如果应用线程的优先级出现了比初始线程高或者相等的情况，请参考1和2的处理方式，直到所有应用线程创建完成，最后初始线程把自己关闭，完成自己
的使命。

main函数
------

当我们拿到一个移植好RT-Thread的例程的时候，不出意外，你首先看到的是main函数，当你认真一看main函数里面只是创建并启动一些线程，那硬件初始化，系统初始化，这些统统在哪里？这些RT-
Thread通过扩展main函数的方式都在component.c里面实现了，具体过程往回看本章的其它小节的详细讲解。

.. code-block:: c
    :caption: 代码清单 16‑11 main函数
    :linenos: 

    /**
    * @brief  主函数
    * @param  无
    * @retval 无
    */
    int main(void)
    {	
        /* 
        * 开发板硬件初始化，RTT系统初始化已经在main函数之前完成，
        * 开发板硬件初始化，RT-Thread系统初始化已经在main函数之前完成，
        * 即在component.c文件中的rtthread_startup()函数中完成了。(1)
        * 所以在main函数中，只需要创建线程和启动线程即可。
        */
        (2)
        thread1 =                               /* 线程控制块指针 */
        rt_thread_create("thread1",             /* 线程名字，字符串形式 */
                        thread1_entry,         /* 线程入口函数 */
                        RT_NULL,               /* 线程入口函数参数 */
                        HREAD1_STACK_SIZE,     /* 线程栈大小，单位为字节 */
                        THREAD1_PRIORITY,      /* 线程优先级，数值越大，优先级越小 */
                        THREAD1_TIMESLICE);    /* 线程时间片 */

        if (thread1 != RT_NULL)
            rt_thread_startup(thread1);
        else
            return -1;
        (3)
        thread2 =                               /* 线程控制块指针 */
        rt_thread_create("thread2",             /* 线程名字，字符串形式 */
                        thread2_entry,         /* 线程入口函数 */
                        RT_NULL,               /* 线程入口函数参数 */
                        THREAD2_STACK_SIZE,    /* 线程栈大小，单位为字节 */
                        THREAD2_PRIORITY,      /* 线程优先级，数值越大，优先级越小 */
                        THREAD2_TIMESLICE);    /* 线程时间片 */

        if (thread2 != RT_NULL)
            rt_thread_startup(thread2);
        else
            return -1;
        (4)
        thread3 =                               /* 线程控制块指针 */
        rt_thread_create("thread3",             /* 线程名字，字符串形式 */
                        thread3_entry,          /* 线程入口函数 */
                        RT_NULL,                /* 线程入口函数参数 */
                        THREAD3_STACK_SIZE,     /* 线程栈大小，单位为字节 */
                        THREAD3_PRIORITY,       /* 线程优先级，数值越大，优先级越小 */
                        THREAD3_TIMESLICE);     /* 线程时间片 */

        if (thread3 != RT_NULL)
            rt_thread_startup(thread3);
        else
            return -1;

        /* 执行到最后，通过LR寄存器执行的地址返回 */       (5)
    }


代码清单 16‑11\ **(1)**\ ：开发板硬件初始化，RT-Thread系统初始化已经在main函数之前完成，即在component.c文件中的rtthread_startup()函数中完成了，所以在main函数中，只需要创建线程和启动线程即可。

代码清单 16‑11\ **(2) (3) (4)**\ ：创建各种应用线程，当创建的应用线程的优先级比main线程的优先级高、低或者相等时候，程序是如何执行的？具体看代码清单 16‑10\ **(初始线程优先级)**\ 的分析。

代码清单 16‑11\ **(5)**\ ：main线程执行到最后，通过LR寄存器指定的链接地址退出，在创建main线程的时候，线程栈对应LR寄存器的内容是rt_thread_exit()函数，在rt_thread_exit里面会把main线程占用的内存空间都释放掉。

至此，RT-Thread的整个启动流程我们就讲完了。
