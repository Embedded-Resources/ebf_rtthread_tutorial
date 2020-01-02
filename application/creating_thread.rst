.. vim: syntax=rst

创建线程
----

在上一章，我们已经基于野火STM32开发板创建好了RT-Thread的工程模板，这章开始我们将真正进入如何使用RT-Thread的征程，先从最简单的创建线程开始，点亮一个LED，以慰藉下尔等初学者弱小的心灵。

硬件初始化
~~~~~

本章创建的线程需要用到开发板上的LED，所以先要将LED相关的函数初始化好，具体是在board.c的rt_hw_board_init()函数中初始化，具体见代码清单 14‑1的加粗部分。

代码清单 14‑1rt_hw_board_init()中添加硬件初始化函数

1 /*\*

2 \* @brief 开发板硬件初始化函数

3 \* @param 无

4 \* @retval 无

5 \*

6 \* @attention

7 \* RTT把开发板相关的初始化函数统一放到board.c文件中实现，

8 \* 当然，你想把这些函数统一放到main.c文件也是可以的。

9 \*/

10 void rt_hw_board_init()

11 {

12 /\* 初始化SysTick \*/

13 SysTick_Config( SystemCoreClock / RT_TICK_PER_SECOND );

14

15 /\* 硬件BSP初始化统统放在这里，比如LED，串口，LCD等 \*/

16

**17 /\* 初始化开发板的LED \*/**

**18 LED_GPIO_Config();**

19

20 /\* 调用组件初始化函数 (use INIT_BOARD_EXPORT()) \*/

21 #ifdef RT_USING_COMPONENTS_INIT

22 rt_components_board_init();

23 #endif

24

25 #if defined(RT_USING_CONSOLE) && defined(RT_USING_DEVICE)

26 rt_console_set_device(RT_CONSOLE_DEVICE_NAME);

27 #endif

28

29 #if defined(RT_USING_USER_MAIN) && defined(RT_USING_HEAP)

30 rt_system_heap_init(rt_heap_begin_get(), rt_heap_end_get());

31 #endif

32 }

执行到rt_hw_board_init()函数的时候，操作系统完全都还没有涉及到，即rt_hw_board_init()函数所做的工作跟我们以前编写的裸机工程里面的硬件初始化工作是一模一样的。运行完rt_hw_board_init()函数，接下来才慢慢启动操作系统，最后运行创建好的线程。有时候线程创
建好，整个系统跑起来了，可想要的实验现象就是出不来，比如LED不会亮，串口没有输出，LCD没有显示等等。如果是初学者，这个时候就会心急如焚，四处求救，那怎么办？这个时候如何排除是硬件的问题还是系统的问题，这里面有个小小的技巧，即在硬件初始化好之后，顺便测试下硬件，测试方法跟裸机编程一样，具体实现见代
码清单 14‑2的加粗部分。

代码清单 14‑2 rt_hw_board_init()中添加硬件测试函数

1 void rt_hw_board_init()

2 {

3 /\* 初始化SysTick \*/

4 SysTick_Config( SystemCoreClock / RT_TICK_PER_SECOND );

5

6 /\* 硬件BSP初始化统统放在这里，比如LED，串口，LCD等 \*/

7

**8 /\* 初始化开发板的LED \*/**

**9 LED_GPIO_Config(); (1)**

**10**

**11 /\* 测试硬件是否正常工作 \*/**

**12 LED1_ON;**

**13**

**14 /\* 其它硬件初始化和测试 \*/ (2)**

**15**

**16 /\* 让程序停在这里，不再继续往下执行 \*/ (3)**

**17 while (1);**

18

19 /\* 调用组件初始化函数 (use INIT_BOARD_EXPORT()) \*/

20 #ifdef RT_USING_COMPONENTS_INIT

21 rt_components_board_init();

22 #endif

23

24 #if defined(RT_USING_CONSOLE) && defined(RT_USING_DEVICE)

25 rt_console_set_device(RT_CONSOLE_DEVICE_NAME);

26 #endif

27

28 #if defined(RT_USING_USER_MAIN) && defined(RT_USING_HEAP)

29 rt_system_heap_init(rt_heap_begin_get(), rt_heap_end_get());

30 #endif

31 }

代码清单 14‑2\ **(1)**\ ：初始化硬件后，顺便测试硬件，看下硬件是否正常工作。

代码清单 14‑2\ **(2)**\ ：可以继续添加其它的硬件初始化和测试。硬件确认没有问题之后，硬件测试代码可删可不删，因为rt_hw_board_init()函数只执行一遍。

代码清单 14‑2\ **(3)**\ ：方便测试硬件好坏，让程序停在这里，不再继续往下执行，当测试完毕后，这个while(1);必须删除。

创建单线程—SRAM静态内存
~~~~~~~~~~~~~~

这里，我们创建一个单线程，线程使用的栈和线程控制块都使用静态内存，即预先定义好的全局变量，这些预先定义好的全局变量都存在内部的SRAM中。

定义线程函数
^^^^^^

线程实际上就是一个无限循环且不带返回值的C函数。目前，我们创建一个这样的线程，让开发板上面的LED灯以500ms的频率闪烁，具体实现见代码清单 14‑3。

代码清单 14‑3 20.6.1 定义线程函数

1 static void led1_thread_entry(void\* parameter)

2 {

3 while (1)

4 {

5 LED1_ON;

6 rt_thread_delay(500); /\* 延时500个tick \*/ **(1)**

7

8 LED1_OFF;

9 rt_thread_delay(500); /\* 延时500个tick \*/

10

11 }

12 }

代码清单 14‑3\ **(1)**\ ：线程里面的延时函数必须使用RT-Thread里面提供的延时函数，并不能使用我们裸机编程中的那种延时。这两种的延时的区别是RT-Thread里面的延时是阻塞延时，即调用rt_thread_delay()函数的时候，当前线程会被挂起，调度器会切换到其它就绪的线程
，从而实现多线程。如果还是使用裸机编程中的那种延时，那么整个线程就成为了一个死循环，如果恰好该线程的优先级是最高的，那么系统永远都是在这个线程中运行，根本无法实现多线程。

目前我们只创建了一个线程，当线程进入延时的时候，因为没有另外就绪的用户线程，那么系统就会进入空闲线程，空闲线程是RT-
Thread系统自己启动的一个线程，优先级最低。当整个系统都没有就绪线程的时候，系统必须保证有一个线程在运行，空闲线程就是为这个设计的。当用户线程延时到期，又会从空闲线程切换回用户线程。

定义线程栈
^^^^^

在RT-Thread系统中，每一个线程都是独立的，他们的运行环境都单独的保存在他们的栈空间当中。那么在定义好线程函数之后，我们还要为线程定义一个栈，目前我们使用的是静态内存，所以线程栈是一个独立的全局变量，具体见代码清单
14‑4。线程的栈占用的是MCU内部的RAM，当线程越多的时候，需要使用的栈空间就越大，即需要使用的RAM空间就越多。一个MCU能够支持多少线程，就得看你的RAM空间有多少。

代码清单 14‑4 定义线程栈

1 /\* 定义线程控栈时要求RT_ALIGN_SIZE个字节对齐 \*/

2 ALIGN(RT_ALIGN_SIZE)

3 /\* 定义线程栈 \*/

4 static rt_uint8_t rt_led1_thread_stack[1024];

在大多数系统中需要做栈空间地址对齐，例如在ARM体系结构中需要向4字节地址对齐。实现栈对齐的方法为，在定义栈之前，放置一条ALIGN(RT_ALIGN_SIZE)语句，指定接下来定义的变量的地址对齐方式。其中ALIGN是在rtdef.h里面定义的一个宏，根据编译器不一样，该宏的具体定义是不一样的，在
ARM编译器中，该宏的定义具体见代码清单 14‑5。ALIGN宏的形参RT_ALIGB_SIZE是在rtconfig.h中的一个宏，目前定义为4。

代码清单 14‑5ALIGN宏定义

/\* 只针对ARM 编译器，在其它编译器，该宏的实现会不一样 \*/

1 #define ALIGN(n) \__attribute__((aligned(n)))

定义线程控制块
^^^^^^^

定义好线程函数和线程栈之后，我们还需要为线程定义一个线程控制块，通常我们称这个线程控制块为线程的身份证。在C代码上，线程控制块就是一个结构体，里面有非常多的成员，这些成员共同描述了线程的全部信息，具体见代码清单 14‑6。

代码清单 14‑6 定义线程控制块

1 /\* 定义线程控制块 \*/

2 static struct rt_thread led1_thread;

初始化线程
^^^^^

一个线程的三要素是线程主体函数，线程栈，线程控制块，那么怎么样把这三个要素联合在一起？RT-Thread里面有一个叫线程初始化函数rt_thread_init()，它就是干这个活的。它将线程主体函数，线程栈（静态的）和线程控制块（静态的）这三者联系在一起，让线程可以随时被系统启动，具体见代码清单
14‑7。

代码清单 14‑7 初始化线程

1 rt_thread_init(&led1_thread, /\* 线程控制块 \*/ **(1)**

2 "led1", /\* 线程名字 \*/ **(2)**

3 led1_thread_entry, /\* 线程入口函数 \*/ **(3)**

4 RT_NULL, /\* 线程入口函数参数 \*/ **(4)**

5 &rt_led1_thread_stack[0], /\* 线程栈起始地址 \*/ **(5)**

6 sizeof(rt_led1_thread_stack), /\* 线程栈大小 \*/ **(6)**

7 3, /\* 线程的优先级 \*/ **(7)**

8 20); /\* 线程时间片 \*/ **(8)**

代码清单 14‑7\ **(1)**\ ：线程控制块指针，在使用静态内存的时候，需要给线程初始化函数rt_thread_init()传递预先定义好的线程控制块的指针。在使用动态内存的时候，线程创建函数rt_thread_create()会返回一个指针指向线程控制块，该线程控制块是rt_thread_
create()函数里面动态分配的一块内存。

代码清单 14‑7\ **(2)**\ ：线程名字，字符串形式，最大长度由rtconfig.h中定义的RT_NAME_MAX宏指定，多余部分会被自动截掉。

代码清单 14‑7\ **(3)**\ ：线程入口函数，即线程函数的名称。

代码清单 14‑7\ **(4)**\ ：线程入口函数形参，不用的时候配置为0即可。

代码清单 14‑7\ **(5)**\ ：线程栈起始地址，只有在使用静态内存的时候才需要提供，在使用动态内存的时候会根据提供的线程栈大小自动创建。

代码清单 14‑7\ **(6)**\ ：线程栈大小，单位为字节。

代码清单 14‑7\ **(7)**\ ：线程的优先级。优先级范围根据rtconfig.h中的宏RT_THREAD_PRIORITY_MAX决定，最多支持256个优先级，目前配置为32。在RT-Thread中，数值越小优先级越高，0代表最高优先级。

代码清单 14‑7\ **(8)**\
：线程的时间片大小。时间片的单位是操作系统的时钟节拍。当系统中存在相同优先级线程时，这个参数指定线程一次调度能够运行的最大时间长度。这个时间片运行结束时，调度器自动选择下一个就绪态的同优先级线程进行运行。如果同一个优先级下只有一个线程，那么时间片这个形参就不起作用。

启动线程
^^^^

当线程初始化好后，是处于线程初始态（RT_THREAD_INIT），并不能够参与操作系统的调度，只有当线程进入就绪态（RT_THREAD_READY）之后才能参与操作系统的调度。线程由初始态进入就绪态可由函数rt_thread_startup()来实现，具体见代码清单 14‑8。

代码清单 14‑8启动线程

/\* 启动线程，开启调度 \*/

1 rt_thread_startup(&led1_thread);

main.c文件内容全貌
^^^^^^^^^^^^

现在我们把线程主体，线程栈，线程控制块这三部分代码统一放到main.c中，具体内容见代码清单 14‑9。

代码清单 14‑9 main.c文件内容全貌

1 /\*

2 \\*

3 \* 包含的头文件

4 \\*

5 \*/

6 #include "board.h"

7 #include "rtthread.h"

8

9

10 /\*

11 \\*

12 \* 变量

13 \\*

14 \*/

15 /\* 定义线程控制块 \*/

16 static struct rt_thread led1_thread;

17

18 /\* 定义线程控栈时要求RT_ALIGN_SIZE个字节对齐 \*/

19 ALIGN(RT_ALIGN_SIZE)

20 /\* 定义线程栈 \*/

21 static rt_uint8_t rt_led1_thread_stack[1024];

22 /\*

23 \\*

24 \* 函数声明

25 \\*

26 \*/

27 static void led1_thread_entry(void\* parameter);

28

29

30 /\*

31 \\*

32 \* main 函数

33 \\*

34 \*/

35 /*\*

36 \* @brief 主函数

37 \* @param 无

38 \* @retval 无

39 \*/

40 int main(void)

41 {

42 /\*

43 \* 开发板硬件初始化，RTT系统初始化已经在main函数之前完成，

44 \* 即在component.c文件中的rtthread_startup()函数中完成了。

45 \* 所以在main函数中，只需要创建线程和启动线程即可。

46 \*/

47

48 rt_thread_init(&led1_thread, /\* 线程控制块 \*/

49 "led1", /\* 线程名字 \*/

50 led1_thread_entry, /\* 线程入口函数 \*/

51 RT_NULL, /\* 线程入口函数参数 \*/

52 &rt_led1_thread_stack[0], /\* 线程栈起始地址 \*/

53 sizeof(rt_led1_thread_stack), /\* 线程栈大小 \*/

54 3, /\* 线程的优先级 \*/

55 20); /\* 线程时间片 \*/

56 rt_thread_startup(&led1_thread); /\* 启动线程，开启调度 \*/

57 }

58

59 /\*

60 \\*

61 \* 线程定义

62 \\*

63 \*/

64

65 static void led1_thread_entry(void\* parameter)

66 {

67 while (1)

68 {

69 LED1_ON;

70 rt_thread_delay(500); /\* 延时500个tick \*/

71

72 LED1_OFF;

73 rt_thread_delay(500); /\* 延时500个tick \*/

74

75 }

76 }

77

78 /END OF FILE/

下载验证
~~~~

将程序编译好，用DAP仿真器把程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），可以看到板子上面的LED灯已经在闪烁，说明我们创建的单线程（使用静态内存）已经跑起来了。

在当前这个例程，线程的栈，线程的控制块用的都是静态内存，必须由用户预先定义，这种方法我们在使用RT-Thread的时候用的比较少，通常的方法是在线程创建的时候动态的分配线程栈和线程控制块的内存空间，接下来我们讲解下“创建单线程—SRAM动态内存”的方法。

创建单线程—SRAM动态内存
~~~~~~~~~~~~~~

这里，我们创建一个单线程，线程使用的栈和线程控制块是在创建线程的时候RT-Thread动态分配的，并不是预先定义好的全局变量。那这些动态的内存堆是从哪里来？继续往下看。

动态内存空间的堆从哪里来
^^^^^^^^^^^^

在创建单线程—SRAM静态内存的例程中，线程控制块和线程栈的内存空间都是从内部的SRAM里面分配的，具体分配到哪个地址由编译器决定。现在我们开始使用动态内存，即堆，其实堆也是内存，也属于SRAM。现在我们的做法是在SRAM里面定义一个大数组供RT-
Thread的动态内存分配函数使用，这些代码在board.c开头实现，具体见代码清单 14‑10。

代码清单 14‑10定义RT-Thread的堆到内部SRAM

1 #if defined(RT_USING_USER_MAIN) && defined(RT_USING_HEAP) **(1)**

2

3 /\* 从内部SRAM（即DTCM）里面分配一部分静态内存来作为RT-Thread的堆空间，这里配置为4KB \*/

4 #define RT_HEAP_SIZE 1024

5 static uint32_t rt_heap[RT_HEAP_SIZE]; **(2)**

6 RT_WEAK void \*rt_heap_begin_get(void) **(3)**

7 {

8 return rt_heap;

9 }

10

11 RT_WEAK void \*rt_heap_end_get(void) **(4)**

12 {

13 return rt_heap + RT_HEAP_SIZE;

14 }

15 #endif

16

17

18 /\* 该部分代码截取自函数rt_hw_board_init() \*/

19 #if defined(RT_USING_USER_MAIN) && defined(RT_USING_HEAP)

20 rt_system_heap_init(rt_heap_begin_get(), rt_heap_end_get()); **(5)**

21 #endif

代码清单 14‑10 **(1)** ：RT_USING_USER_MAIN 和RT_USING_HEAP这两个宏，在rtconfig.h定义，RT_USING_USER_MAIN默认开启，RT_USING_HEAP在使用动态内存时需要开启。

代码清单 14‑10\ **(2)** ：从内部SRAMM里面定义一个静态数组rt_heap，大小由RT_HEAP_SIZE这个宏决定，目前定义为4KB。定义的堆大小不能超过内部SRAM的总大小。

代码清单 14‑10\ **(3)** ：rt_heap_begin_get()用于获取堆的起始地址。

代码清单 14‑10\ **(4)** ：rt_heap_end_get()用于获取堆的结束地址。

代码清单 14‑10\ **(5)**
：rt_system_heap_init()根据堆的起始地址和结束地址进行堆的初始化。rt_system_heap_init()需要两个形参，一个是堆的起始地址，另外一个是堆的结束地址，如果我们使用外部SDRAM作为堆，这两个形参直接传入外部SDRAM地址范围内的地址即可。

.. _定义线程函数-1:

定义线程函数
^^^^^^

使用动态内存的时候，线程的主体函数与使用静态内存时是一样的，具体见代码清单 14‑11。

代码清单 14‑11定义线程函数

1 static void led1_thread_entry(void\* parameter)

2 {

3 while (1)

4 {

5 LED1_ON;

6 rt_thread_delay(500); /\* 延时500个tick \*/

7

8 LED1_OFF;

9 rt_thread_delay(500); /\* 延时500个tick \*/

10

11 }

12 }

.. _定义线程栈-1:

定义线程栈
^^^^^

使用动态内存的时候，线程栈在线程创建的时候创建，不用跟使用静态内存那样要预先定义好一个全局的静态的栈空间。

定义线程控制块指针
^^^^^^^^^

使用动态内存时候，不用跟使用静态内存那样要预先定义好一个全局的静态的线程控制块空间。线程控制块是在线程创建的时候创建，线程创建函数会返回一个指针，用于指向线程控制块，所以要预先为线程栈定义一个线程控制块指针，具体见代码清单 14‑12。

代码清单 14‑12定义线程控制块指针

1 /\* 定义线程控制块指针 \*/

2 static rt_thread_t led1_thread = RT_NULL;

.. _创建线程-1:

创建线程
^^^^

使用静态内存时，使用rt_thread_init()来初始化一个线程，使用动态内存的时，使用rt_thread_create()函数来创建一个线程，两者的函数名不一样，具体的形参也有区别，具体见代码清单 14‑13。

代码清单 14‑13创建线程

1 led1_thread = /\* 线程控制块指针 \*/ **(1)**

2 rt_thread_create( "led1", /\* 线程名字 \*/ **(2)**

3 led1_thread_entry, /\* 线程入口函数 \*/ **(3)**

4 RT_NULL, /\* 线程入口函数参数 \*/ **(4)**

5 512, /\* 线程栈大小 \*/ **(5)**

6 3, /\* 线程的优先级 \*/ **(6)**

7 20); /\* 线程时间片 \*/ **(7)**

代码清单 14‑13\ **(1)**\ ：线程控制块指针，在使用静态内存的时候，需要给线程初始化函数rt_thread_init()传递预先定义好的线程控制块的指针。在使用动态内存的时候，线程创建函数rt_thread_create()会返回一个指针指向线程控制块，该线程控制块是rt_thread
_create()函数里面动态分配的一块内存。

代码清单 14‑13\ **(2)**\ ：线程名字，字符串形式，最大长度由rtconfig.h中定义的RT_NAME_MAX宏指定，多余部分会被自动截掉。

代码清单 14‑13\ **(3)**\ ：线程入口函数，即线程函数的名称。

代码清单 14‑13\ **(4)**\ ：线程入口函数形参，不用的时候配置为0即可。

代码清单 14‑13\ **(5)**\ ：线程栈大小，单位为字节。使用动态内存创建线程时，与使用静态内存线程初始化函数不一样，不再需要提供线程栈的起始地址，只需要知道线程栈的大小即可，因为它是在线程创建时动态分配的。

代码清单 14‑13\ **(6)**\ ：线程的优先级。优先级范围根据rtconfig.h中的宏RT_THREAD_PRIORITY_MAX决定，最多支持256个优先级，目前配置为32。在RT-Thread中，数值越小优先级越高，0代表最高优先级。

代码清单 14‑13\ **(7)**\
：线程的时间片大小。时间片的单位是操作系统的时钟节拍。当系统中存在相同优先级线程时，这个参数指定线程一次调度能够运行的最大时间长度。这个时间片运行结束时，调度器自动选择下一个就绪态的同优先级线程进行运行。如果同一个优先级下只有一个线程，那么时间片这个形参就不起作用。

.. _启动线程-1:

启动线程
^^^^

当线程创建好后，是处于线程初始态（RT_THREAD_INIT），并不能够参与操作系统的调度，只有当线程进入就绪态（RT_THREAD_READY）之后才能参与操作系统的调度。线程由初始态进入就绪态可由函数rt_thread_startup()来实现，具体见代码清单 14‑14。

代码清单 14‑14 启动线程

1 if (led1_thread != RT_NULL)

2 rt_thread_startup(led1_thread); /\* 启动线程，开启调度 \*/

3 else

4 return -1;

.. _main.c文件内容全貌-1:

main.c文件内容全貌
^^^^^^^^^^^^

现在我们把线程主体，线程栈指针，线程控制块这三部分代码统一放到main.c中，具体见代码清单 14‑15。

代码清单 14‑15main.c文件内容全貌

1 #if defined(RT_USING_USER_MAIN) && defined(RT_USING_HEAP)

2 #define RT_HEAP_SIZE 1024

3 /\* 从内部SRAM里面分配一部分静态内存来作为rtt的堆空间，这里配置为4KB \*/

4 static uint32_t rt_heap[RT_HEAP_SIZE];

5 RT_WEAK void \*rt_heap_begin_get(void)

6 {

7 return rt_heap;

8 }

9

10 RT_WEAK void \*rt_heap_end_get(void)

11 {

12 return rt_heap + RT_HEAP_SIZE;

13 }

14 #endif

15

16 /\* 该部分代码截取自函数rt_hw_board_init() \*/

17 #if defined(RT_USING_USER_MAIN) && defined(RT_USING_HEAP)

18 //rt_system_heap_init((void*)HEAP_BEGIN, (void*)SRAM_END);

19 rt_system_heap_init(rt_heap_begin_get(), rt_heap_end_get());

20 #endif

21

22

23 /\*

24 \\*

25 \* 包含的头文件

26 \\*

27 \*/

28 #include "board.h"

29 #include "rtthread.h"

30

31

32 /\*

33 \\*

34 \* 变量

35 \\*

36 \*/

37 /\* 定义线程控制块指针 \*/

38 static rt_thread_t led1_thread = RT_NULL;

39

40 /\*

41 \\*

42 \* 函数声明

43 \\*

44 \*/

45 static void led1_thread_entry(void\* parameter);

46

47

48 /\*

49 \\*

50 \* main 函数

51 \\*

52 \*/

53 /*\*

54 \* @brief 主函数

55 \* @param 无

56 \* @retval 无

57 \*/

58 int main(void)

59 {

60 /\*

61 \* 开发板硬件初始化，RTT系统初始化已经在main函数之前完成，

62 \* 即在component.c文件中的rtthread_startup()函数中完成了。

63 \* 所以在main函数中，只需要创建线程和启动线程即可。

64 \*/

65

66 led1_thread = /\* 线程控制块指针 \*/

67 rt_thread_create( "led1", /\* 线程名字 \*/

68 led1_thread_entry, /\* 线程入口函数 \*/

69 RT_NULL, /\* 线程入口函数参数 \*/

70 512, /\* 线程栈大小 \*/

71 3, /\* 线程的优先级 \*/

72 20); /\* 线程时间片 \*/

73

74 /\* 启动线程，开启调度 \*/

75 if (led1_thread != RT_NULL)

76 rt_thread_startup(led1_thread);

77 else

78 return -1;

79 }

80

81 /\*

82 \\*

83 \* 线程定义

84 \\*

85 \*/

86

87 static void led1_thread_entry(void\* parameter)

88 {

89 while (1)

90 {

91 LED1_ON;

92 rt_thread_delay(500); /\* 延时500个tick \*/

93

94 LED1_OFF;

95 rt_thread_delay(500); /\* 延时500个tick \*/

96

97 }

98 }

99

100 /END OF FILE/

.. _下载验证-1:

下载验证
~~~~

将程序编译好，用DAP仿真器把程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），可以看到板子上面的LED灯已经在闪烁，说明我们创建的单线程（使用动态内存）已经跑起来了。在往后的实验中，我们创建内核对象均采用动态内存分配方案。

创建多线程—SRAM动态内存
~~~~~~~~~~~~~~

创建多线程只需要按照创建单线程的套路依葫芦画瓢即可，接下来我们创建两个线程，线程1让一个LED灯闪烁，线程2让另外一个LED闪烁，两个LED闪烁的频率不一样，具体实现见代码清单 10‑16的加粗部分，两个线程的优先级不一样。

代码清单 14‑16创建多线程—SRAM动态内存

1 /\*

2 \\*

3 \* 包含的头文件

4 \\*

5 \*/

6 #include "board.h"

7 #include "rtthread.h"

8

9

10 /\*

11 \\*

12 \* 变量

13 \\*

14 \*/

15 /\* 定义线程控制块指针 \*/

16 static rt_thread_t led1_thread = RT_NULL;

**17 static rt_thread_t led2_thread = RT_NULL;**

18

19 /\*

20 \\*

21 \* 函数声明

22 \\*

23 \*/

24 static void led1_thread_entry(void\* parameter);

**25 static void led2_thread_entry(void\* parameter);**

26

27

28 /\*

29 \\*

30 \* main 函数

31 \\*

32 \*/

33 /*\*

34 \* @brief 主函数

35 \* @param 无

36 \* @retval 无

37 \*/

38 int main(void)

39 {

40 /\*

41 \* 开发板硬件初始化，RTT系统初始化已经在main函数之前完成，

42 \* 即在component.c文件中的rtthread_startup()函数中完成了。

43 \* 所以在main函数中，只需要创建线程和启动线程即可。

44 \*/

45

46 led1_thread = /\* 线程控制块指针 \*/

47 rt_thread_create( "led1", /\* 线程名字 \*/

48 led1_thread_entry, /\* 线程入口函数 \*/

49 RT_NULL, /\* 线程入口函数参数 \*/

50 512, /\* 线程栈大小 \*/

51 3, /\* 线程的优先级 \*/

52 20); /\* 线程时间片 \*/

53

54 /\* 启动线程，开启调度 \*/

55 if (led1_thread != RT_NULL)

56 rt_thread_startup(led1_thread);

57 else

58 return -1;

59

**60 led2_thread = /\* 线程控制块指针 \*/**

**61 rt_thread_create( "led2", /\* 线程名字 \*/**

**62 led2_thread_entry, /\* 线程入口函数 \*/**

**63 RT_NULL, /\* 线程入口函数参数 \*/**

**64 512, /\* 线程栈大小 \*/**

**65 4, /\* 线程的优先级 \*/**

**66 20); /\* 线程时间片 \*/**

**67**

**68 /\* 启动线程，开启调度 \*/**

**69 if (led2_thread != RT_NULL)**

**70 rt_thread_startup(led2_thread);**

**71 else**

**72 return -1;**

73 }

74

75 /\*

76 \\*

77 \* 线程定义

78 \\*

79 \*/

80

81 static void led1_thread_entry(void\* parameter)

82 {

83 while (1)

84 {

85 LED1_ON;

86 rt_thread_delay(500); /\* 延时500个tick \*/

87

88 LED1_OFF;

89 rt_thread_delay(500); /\* 延时500个tick \*/

90

91 }

92 }

93

**94 static void led2_thread_entry(void\* parameter)**

**95 {**

**96 while (1)**

**97 {**

**98 LED2_ON;**

**99 rt_thread_delay(300); /\* 延时300个tick \*/**

**100**

**101 LED2_OFF;**

**102 rt_thread_delay(300); /\* 延时300个tick \*/**

**103**

**104 }**

**105 }**

106 /END OF FILE/

目前多线程我们只创建了两个，如果要创建3个、4个甚至更多都是同样的套路，容易忽略的地方是线程栈的大小，每个线程的优先级。大的线程，栈空间要设置大一点，重要的线程优先级要设置的高一点。

.. _下载验证-2:

下载验证
~~~~

将程序编译好，用DAP仿真器把程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），可以看到板子上面的两个LED灯以不同的频率在闪烁，说明我们创建的单线程（使用动态内存）已经跑起来了。在往后的实验中，我们创建内核对象均采用动态内存分配方案。
