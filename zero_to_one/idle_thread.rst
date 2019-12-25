.. vim: syntax=rst

空闲线程与阻塞延时的实现
------------

在上一章节中，线程体内的延时使用的是软件延时，即还是让CPU空等来达到延时的效果。使用RTOS的很大优势就是榨干CPU的性能，永远不能让它闲着，线程如果需要延时也就不能再让CPU空等来实现延时的效果。RTOS中的延时叫阻塞延时，即线程需要延时的时候，线程会放弃CPU的使用权，CPU可以去干其它的事情
，当线程延时时间到，重新获取CPU使用权，线程继续运行，这样就充分地利用了CPU的资源，而不是干等着。

当线程需要延时，进入阻塞状态，那CPU又去干什么事情了？如果没有其它线程可以运行，RTOS都会为CPU创建一个空闲线程，这个时候CPU就运行空闲线程。在RT-
Thread中，空闲线程是系统在初始化的时候创建的优先级最低的线程，空闲线程主体主要是做一些系统内存的清理工作。但是为了简单起见，我们本章实现的空闲线程只是对一个全局变量进行计数。鉴于空闲线程的这种特性，在实际应用中，当系统进入空闲线程的时候，可在空闲线程中让单片机进入休眠或者低功耗等操作。

实现空闲线程
~~~~~~

定义空闲线程的栈
^^^^^^^^

空闲线程的栈在idle.c（idle.c第一次使用需要自行在文件夹rtthread\3.0.3\src中新建并添加到工程的rtt/source组）文件中定义，具体见代码清单 9‑1。

代码清单 9‑1 定义空闲线程的栈

1 #include <rtthread.h>

2 #include <rthw.h>

3

4 #define IDLE_THREAD_STACK_SIZE 512

5

6 ALIGN(RT_ALIGN_SIZE)

7 static rt_uint8_t rt_thread_stack[IDLE_THREAD_STACK_SIZE];

代码清单 9‑1 （1）：空闲线程的栈是一个定义好的数组，大小由IDLE_THREAD_STACK_SIZE这个宏控制，默认为512，即128个字。

定义空闲线程的线程控制块
^^^^^^^^^^^^

线程控制块是每一个线程必须的，空闲线程的的线程控制块在idle.c中定义，是一个全局变量，具体见代码清单 9‑2。

代码清单 9‑2 定义空闲线程的线程控制块

/\* 空闲线程的线程控制块 \*/

1 struct rt_thread idle;

定义空闲线程函数
^^^^^^^^

在RT-Thread中空闲线程函数主要是做一些系统内存的清理工作，但是为了简单起见，我们本章实现的空闲线程只是对一个全局变量rt_idletask_ctr进行计数，rt_idletask_ctr在idle.c中定义，默认初始值为0。空闲线程函数在idle.c定义，具体实现见代码清单 9‑3。

代码清单 9‑3 空闲线程函数

1 rt_ubase_t rt_idletask_ctr = 0;

2

3 void rt_thread_idle_entry(void \*parameter)

4 {

5 parameter = parameter;

6 while (1)

7 {

8 rt_idletask_ctr ++;

9 }

10 }

空闲线程初始化
^^^^^^^

当定义好空闲线程的栈，线程控制块和函数主体之后，我们需要空闲线程初始化函数将这三者联系在一起，这样空闲线程才能够被系统调度，空闲线程初始化函数在idle.c定义，具体实现见代码清单 9‑4。

代码清单 9‑4 空闲线程初始化

1 void rt_thread_idle_init(void)

2 {

3

4 /\* 初始化线程 \*/ **(1)**

5 rt_thread_init(&idle,

6 "idle",

7 rt_thread_idle_entry,

8 RT_NULL,

9 &rt_thread_stack[0],

10 sizeof(rt_thread_stack));

11

12 /\* 将线程插入到就绪列表 \*/ **(2)**

13 rt_list_insert_before( &(rt_thread_priority_table[RT_THREAD_PRIORITY_MAX-1]),

14 &(idle.tlist) );

15 }

代码清单 9‑4\ **(1)**\ ：创建空闲线程。

代码清单 9‑4\ **(2)** ：将空闲线程插入到就绪列表的末尾。在下一章我们会支持优先级，空闲线程默认的优先级是最低的，即排在就绪列表的最后面。

实现阻塞延时
~~~~~~

阻塞延时的阻塞是指线程调用该延时函数后，线程会被剥离CPU使用权，然后进入阻塞状态，直到延时结束，线程重新获取CPU使用权才可以继续运行。在线程阻塞的这段时间，CPU可以去执行其它的线程，如果其它的线程也在延时状态，那么CPU就将运行空闲线程。阻塞延时函数在thread.c中定义，具体代码实现见代码
清单 9‑5。

代码清单 9‑5 阻塞延时代码

1 void rt_thread_delay(rt_tick_t tick)

2 {

3 struct rt_thread \*thread;

4

5 /\* 获取当前线程的线程控制块 \*/

6 thread = rt_current_thread; **(1)**

7

8 /\* 设置延时时间 \*/

9 thread->remaining_tick = tick; **(2)**

10

11 /\* 进行系统调度 \*/

12 rt_schedule(); **(3)**

13 }

代码清单 9‑5 **(1)**\ ：获取当前线程的线程控制块。rt_current_thread是一个在scheduler.c定义的全局变量，用于指向当前正在运行的线程的线程控制块。

代码清单 9‑5 **(2)**\ ：remaining_tick是线程控制块的一个成员，用于记录线程需要延时的时间，单位为SysTick的中断周期。比如我们本书当中SysTick的中断周期为10ms，调用rt_thread_delay(2)则完成2*10ms的延时。线程的定义具体见代码清单
9‑6。

代码清单 9‑6 remaining_tick定义

1 struct rt_thread

2 {

3 /\* rt 对象 \*/

4 char name[RT_NAME_MAX]; /\* 对象的名字 \*/

5 rt_uint8_t type; /\* 对象类型 \*/

6 rt_uint8_t flags; /\* 对象的状态 \*/

7 rt_list_t list; /\* 对象的列表节点 \*/

8

9 rt_list_t tlist; /\* 线程链表节点 \*/

10

11 void \*sp; /\* 线程栈指针 \*/

12 void \*entry; /\* 线程入口地址 \*/

13 void \*parameter; /\* 线程形参 \*/

14 void \*stack_addr; /\* 线程起始地址 \*/

15 rt_uint32_t stack_size; /\* 线程栈大小，单位为字节 \*/

16

**17 rt_ubase_t remaining_tick; /\* 用于实现阻塞延时 \*/**

18 };

代码清单 9‑5 **(3)**\ ：系统调度。这个时候的系统调度与上一章节的不一样，具体见代码清单 9‑7，其中加粗部分为上一章节的代码，现已用条件编译屏蔽掉。

代码清单 9‑7 系统调度

1 extern struct rt_thread idle;

2 extern struct rt_thread rt_flag1_thread;

3 extern struct rt_thread rt_flag2_thread;

4

5 void rt_schedule(void)

6 {

7 struct rt_thread \*to_thread;

8 struct rt_thread \*from_thread;

9

**10 #if 0**

**11 /\* 两个线程轮流切换 \*/**

**12 if ( rt_current_thread == rt_list_entry( rt_thread_priority_table[0].next,**

**13 struct rt_thread,**

**14 tlist) )**

**15 {**

**16 from_thread = rt_current_thread;**

**17 to_thread = rt_list_entry( rt_thread_priority_table[1].next,**

**18 struct rt_thread,**

**19 tlist);**

**20 rt_current_thread = to_thread;**

**21 }**

**22 else**

**23 {**

**24 from_thread = rt_current_thread;**

**25 to_thread = rt_list_entry( rt_thread_priority_table[0].next,**

**26 struct rt_thread,**

**27 tlist);**

**28 rt_current_thread = to_thread;**

**29 }**

30 #else

31

32

33 /\* 如果当前线程是空闲线程，那么就去尝试执行线程1或者线程2，

34 看看他们的延时时间是否结束，如果线程的延时时间均没有到期，

35 那就返回继续执行空闲线程 \*/

36 if ( rt_current_thread == &idle ) **(1)**

37 {

38 if (rt_flag1_thread.remaining_tick == 0)

39 {

40 from_thread = rt_current_thread;

41 to_thread = &rt_flag1_thread;

42 rt_current_thread = to_thread;

43 }

44 else if (rt_flag2_thread.remaining_tick == 0)

45 {

46 from_thread = rt_current_thread;

47 to_thread = &rt_flag2_thread;

48 rt_current_thread = to_thread;

49 }

50 else

51 {

52 return; /\* 线程延时均没有到期则返回，继续执行空闲线程 \*/

53 }

54 }

55 else /\* 当前线程不是空闲线程则会执行到这里 \*/ **(2)**

56 {

57 /\* 如果当前线程是线程1或者线程2的话，检查下另外一个线程,

58 如果另外的线程不在延时中，就切换到该线程

59 否则，判断下当前线程是否应该进入延时状态，如果是的话，就切换到空闲线程，

60 否则就不进行任何切换 \*/

61 if (rt_current_thread == &rt_flag1_thread)

62 {

63 if (rt_flag2_thread.remaining_tick == 0)

64 {

65 from_thread = rt_current_thread;

66 to_thread = &rt_flag2_thread;

67 rt_current_thread = to_thread;

68 }

69 else if (rt_current_thread->remaining_tick != 0)

70 {

71 from_thread = rt_current_thread;

72 to_thread = &idle;

73 rt_current_thread = to_thread;

74 }

75 else

76 {

77 return; /\* 返回，不进行切换，因为两个线程都处于延时中 \*/

78 }

79 }

80 else if (rt_current_thread == &rt_flag2_thread)

81 {

82 if (rt_flag1_thread.remaining_tick == 0)

83 {

84 from_thread = rt_current_thread;

85 to_thread = &rt_flag1_thread;

86 rt_current_thread = to_thread;

87 }

88 else if (rt_current_thread->remaining_tick != 0)

89 {

90 from_thread = rt_current_thread;

91 to_thread = &idle;

92 rt_current_thread = to_thread;

93 }

94 else

95 {

96 return; /\* 返回，不进行切换，因为两个线程都处于延时中 \*/

97 }

98 }

99 }

100 #endif

101 /\* 产生上下文切换 \*/

102 rt_hw_context_switch((rt_uint32_t)&from_thread->sp,(rt_uint32_t)&to_thread->sp);

103 }

代码清单 9‑7\ **(1)**\ ：如果当前线程是空闲线程，那么就去尝试执行线程1或者线程2，看看他们的延时时间是否结束，如果线程的延时时间均没有到期，那就返回继续执行空闲线程。

代码清单 9‑7\ **(2)**\ ：如果当前线程是线程1或者线程2的话，检查下另外一个线程，如果另外的线程不在延时中，就切换到该线程。否则，判断下当前线程是否应该进入延时状态，如果是的话，就切换到空闲线程，否则就不进行任何切换 。

代码清单 9‑7\ **(3)**\ ：系统调度，实现线程的切换。

SysTick_Handler中断服务函数
~~~~~~~~~~~~~~~~~~~~~

在系统调度函数rt_schedule()中，会判断每个线程的线程控制块中的延时成员remaining_tick的值是否为0，如果为0就要将对应的线程就绪，如果不为0就继续延时。如果一个线程要延时，一开始remaining_tick肯定不为0，当remaining_tick变为0的时候表示延时结束，那
么remaining_tick是以什么周期在递减？在哪里递减？在RT-Thread中，这个周期由SysTick中断提供，操作系统里面的最小的时间单位就是SysTick的中断周期，我们称之为一个tick，SysTick中断服务函数我们放在main.c中实现，具体见代码清单 9‑8。

代码清单 9‑8 SysTick_Handler中断服务函数

1 /\* 关中断 \*/

2 rt_hw_interrupt_disable(); **(1)**

3

4 /\* SysTick中断频率设置 \*/

5 SysTick_Config( SystemCoreClock / RT_TICK_PER_SECOND ); **(2)**

6

7 void SysTick_Handler(void) **(3)**

8 {

9 /\* 进入中断 \*/

10 rt_interrupt_enter(); **(3)-①**

11 /\* 时基更新 \*/

12 rt_tick_increase(); **(3)-②**

13

14 /\* 离开中断 \*/

15 rt_interrupt_leave(); **(3)-③**

16 }

代码清单 9‑8\ **(1)**\ ：关中断。在程序开始的时候把中断关闭是一个好习惯，等系统初始化完毕，线程创建完毕，启动系统调度的时候会重新打开中断。如果一开始不关闭中断的话，接下来SysTick初始化完成，然后再初始化系统和创建线程，如果系统初始化和线程创建的时间大于SysTick的中断周期的
话，那么就会出现系统或者线程都还没有准备好的情况下就先执行了SysTick中断服务函数，进行了系统调度，显然，这是不科学的。

代码清单 9‑8\ **(2)**\ ：初始化SysTick，调用固件库函数SysTick_Config来实现，配置中断周期为10ms，中断优先级为最低（无论中断优先级分组怎么分都是最低，因为这里把表示SysTick中断优先级的四个位全部配置为1，即15，在Cortex-
M内核中，优先级数值越大，逻辑优先级越低），RT_TICK_PER_SECOND是一个在rtconfig.h中定义的宏，目前等于100。

代码清单 9‑9 SysTick初始化函数（在core_cm3.h中定义）

1 \__STATIC_INLINE uint32_t SysTick_Config(uint32_t ticks)

2 {

3 /\* 非法的重装载寄存器值 \*/

4 if ((ticks - 1UL) > SysTick_LOAD_RELOAD_Msk)

5 {

6 return (1UL);

7 }

8

9 /\* 设置重装载寄存器的值 \*/

10 SysTick->LOAD = (uint32_t)(ticks - 1UL);

11

12 /\* 设置SysTick的中断优先级 \*/

13 NVIC_SetPriority (SysTick_IRQn, (1UL << \__NVIC_PRIO_BITS) - 1UL);

14

15 /\* 加载SysTick计数器值 \*/

16 SysTick->VAL = 0UL;

17

18 /\* 设置系统定时器的时钟源为 AHBCLK

19 使能SysTick 定时器中断

20 使能SysTick 定时器 \*/

21 SysTick->CTRL = SysTick_CTRL_CLKSOURCE_Msk \|

22 SysTick_CTRL_TICKINT_Msk \|

23 SysTick_CTRL_ENABLE_Msk;

24 return (0UL);

25 }

代码清单 9‑8\ **(3)-②**\ ：更新系统时基，该函数在clock.c（clock.c第一次使用需要自行在文件夹rtthread\3.0.3\src中新建并添加到工程的rtt/source组）中实现，具体见代码清单 9‑10。

系统时基更新函数
^^^^^^^^

代码清单 9‑10 时基更新函数

1 #include <rtthread.h>

2 #include <rthw.h>

3

4 static rt_tick_t rt_tick = 0; /\* 系统时基计数器 \*/ **(1)**

5 extern rt_list_t rt_thread_priority_table[RT_THREAD_PRIORITY_MAX];

6

7

8 void rt_tick_increase(void)

9 {

10 rt_ubase_t i;

11 struct rt_thread \*thread;

12 rt_tick ++; **(2)**

13

14 /\* 扫描就绪列表中所有线程的remaining_tick，如果不为0，则减1 \*/

15 for (i=0; i<RT_THREAD_PRIORITY_MAX; i++) **(3)**

16 {

17 thread = rt_list_entry( rt_thread_priority_table[i].next,

18 struct rt_thread,

19 tlist);

20 if (thread->remaining_tick > 0)

21 {

22 thread->remaining_tick --;

23 }

24 }

25

26 /\* 系统调度 \*/

27 rt_schedule(); **(4)**

28 }

代码清单 9‑10 **(1)**\ ：系统时基计数器，是一个全局变量，用来记录产生了多少次SysTick中断。

代码清单 9‑10 **(2)**\ ：系统时基计数器加一操作。

代码清单 9‑10 **(3)**\ ：扫描就绪列表中所有线程的remaining_tick，如果不为0，则减1。

代码清单 9‑10 **(4)**\ ：进行系统调度。

代码清单 9‑8\ **(3)-①和③**\ ：进入中断和离开中断，这两个函数在irq.c（irq.c第一次使用需要自行在文件夹rtthread\3.0.3\src中新建并添加到工程的rtt/source组）中实现，具体见代码清单 9‑11。

代码清单 9‑11 进入中断和离开中断函数

1 #include <rtthread.h>

2 #include <rthw.h>

3

4 /\* 中断计数器 \*/

5 volatile rt_uint8_t rt_interrupt_nest; **(1)**

6

7 /*\*

8 \* 当BSP文件的中断服务函数进入时会调用该函数

9 \*

10 \* @note 请不要在应用程序中调用该函数

11 \*

12 \* @see rt_interrupt_leave

13 \*/

14 void rt_interrupt_enter(void) **(2)**

15 {

16 rt_base_t level;

17

18

19 /\* 关中断 \*/

20 level = rt_hw_interrupt_disable();

21

22 /\* 中断计数器++ \*/

23 rt_interrupt_nest ++;

24

25 /\* 开中断 \*/

26 rt_hw_interrupt_enable(level);

27 }

28

29

30 /*\*

31 \* 当BSP文件的中断服务函数离开时会调用该函数

32 \*

33 \* @note 请不要在应用程序中调用该函数

34 \*

35 \* @see rt_interrupt_enter

36 \*/

37 void rt_interrupt_leave(void) **(3)**

38 {

39 rt_base_t level;

40

41

42 /\* 关中断 \*/

43 level = rt_hw_interrupt_disable();

44

45 /\* 中断计数器-- \*/

46 rt_interrupt_nest --;

47

48 /\* 开中断 \*/

49 rt_hw_interrupt_enable(level);

50 }

代码清单 9‑11\ **(1)**\ ：中断计数器，是一个全局变量，用了记录中断嵌套次数。

代码清单 9‑11\ **(2)**\ ：进入中断函数，中断计数器rt_interrupt_nest加一操作。当BSP文件的中断服务函数进入时会调用该函数，应用程序不能调用，切记。

代码清单 9‑11\ **(3)**\ ：离开中断函数，中断计数器rt_interrupt_nest减一操作。当BSP文件的中断服务函数离开时会调用该函数，应用程序不能调用，切记。

main函数
~~~~~~

main函数和线程代码变动不大，具体见代码清单 9‑12，有变动部分代码已加粗。

代码清单 9‑12 main函数

1 /\*

2 \\*

3 \* 包含的头文件

4 \\*

5 \*/

6

7 #include <rtthread.h>

**8 #include <rthw.h> (1)**

**9 #include "ARMCM3.h"**

10

11

12 /\*

13 \\*

14 \* 全局变量

15 \\*

16 \*/

17 rt_uint8_t flag1;

18 rt_uint8_t flag2;

19

20 extern rt_list_t rt_thread_priority_table[RT_THREAD_PRIORITY_MAX];

21

22 /\*

23 \\*

24 \* 线程控制块 & STACK & 线程声明

25 \\*

26 \*/

27

28

29 /\* 定义线程控制块 \*/

30 struct rt_thread rt_flag1_thread;

31 struct rt_thread rt_flag2_thread;

32

33 ALIGN(RT_ALIGN_SIZE)

34 /\* 定义线程栈 \*/

35 rt_uint8_t rt_flag1_thread_stack[512];

36 rt_uint8_t rt_flag2_thread_stack[512];

37

38 /\* 线程声明 \*/

39 void flag1_thread_entry(void \*p_arg);

40 void flag2_thread_entry(void \*p_arg);

41

42 /\*

43 \\*

44 \* 函数声明

45 \\*

46 \*/

47 void delay(uint32_t count);

48

49 /\*

50 \* @brief main函数

51 \* @param 无

52 \* @retval 无

53 \*

54 \* @attention

55 \\*

56 \*/

57 int main(void)

58 {

59 /\* 硬件初始化 \*/

60 /\* 将硬件相关的初始化放在这里，如果是软件仿真则没有相关初始化代码 \*/

61

**62 /\* 关中断 \*/**

**63 rt_hw_interrupt_disable(); (2)**

**64**

**65 /\* SysTick中断频率设置 \*/**

**66 SysTick_Config( SystemCoreClock / RT_TICK_PER_SECOND ); (3)**

67

68 /\* 调度器初始化 \*/

69 rt_system_scheduler_init();

70

**71 /\* 初始化空闲线程 \*/**

**72 rt_thread_idle_init(); (4)**

73

74 /\* 初始化线程 \*/

75 rt_thread_init( &rt_flag1_thread, /\* 线程控制块 \*/

76 "rt_flag1_thread", /\* 线程名字，字符串形式 \*/

77 flag1_thread_entry, /\* 线程入口地址 \*/

78 RT_NULL, /\* 线程形参 \*/

79 &rt_flag1_thread_stack[0], /\* 线程栈起始地址 \*/

80 sizeof(rt_flag1_thread_stack) ); /\* 线程栈大小，单位为字节 \*/

81 /\* 将线程插入到就绪列表 \*/

82 rt_list_insert_before( &(rt_thread_priority_table[0]),&(rt_flag1_thread.tlist) );

83

84 /\* 初始化线程 \*/

85 rt_thread_init( &rt_flag2_thread, /\* 线程控制块 \*/

86 "rt_flag2_thread", /\* 线程名字，字符串形式 \*/

87 flag2_thread_entry, /\* 线程入口地址 \*/

88 RT_NULL, /\* 线程形参 \*/

89 &rt_flag2_thread_stack[0], /\* 线程栈起始地址 \*/

90 sizeof(rt_flag2_thread_stack) ); /\* 线程栈大小，单位为字节 \*/

91 /\* 将线程插入到就绪列表 \*/

92 rt_list_insert_before( &(rt_thread_priority_table[1]),&(rt_flag2_thread.tlist) );

93

94 /\* 启动系统调度器 \*/

95 rt_system_scheduler_start();

96 }

97

98 /\*

99 \\*

100 \* 函数实现

101 \\*

102 \*/

103 /\* 软件延时 \*/

104 void delay (uint32_t count)

105 {

106 for (; count!=0; count--);

107 }

108

109 /\* 线程1 \*/

110 void flag1_thread_entry( void \*p_arg )

111 {

112 for ( ;; )

113 {

114 #if 0

115 flag1 = 1;

116 delay( 100 );

117 flag1 = 0;

118 delay( 100 );

119

120 /\* 线程切换，这里是手动切换 \*/

121 rt_schedule();

122 #else

123 flag1 = 1;

**124 rt_thread_delay(2); (5)**

125 flag1 = 0;

**126 rt_thread_delay(2);**

127 #endif

128 }

129 }

130

131 /\* 线程2 \*/

132 void flag2_thread_entry( void \*p_arg )

133 {

134 for ( ;; )

135 {

136 #if 0

137 flag2 = 1;

138 delay( 100 );

139 flag2 = 0;

140 delay( 100 );

141

142 /\* 线程切换，这里是手动切换 \*/

143 rt_schedule();

144 #else

145 flag2 = 1;

**146 rt_thread_delay(2); (6)**

147 flag2 = 0;

**148 rt_thread_delay(2);**

149 #endif

150 }

151 }

152

153

**154 void SysTick_Handler(void) (7)**

**155 {**

**156 /\* 进入中断 \*/**

**157 rt_interrupt_enter();**

**158**

**159 rt_tick_increase();**

**160**

**161 /\* 离开中断 \*/**

**162 rt_interrupt_leave();**

**163 }**

代码清单 9‑12\ **(1)**\ ：新包含的两个头文件。

代码清单 9‑12\ **(2)**\ ：关中断。

代码清单 9‑12\ **(3)**\ ：初始化SysTick。

代码清单 9‑12\ **(4)**\ ：创建空闲线程。

代码清单 9‑12\ **(5)**\ 和\ **(6)**\ ：延时函数均由原来的软件延时替代为阻塞延时，延时时间均为2个SysTick中断周期，即20ms。

代码清单 9‑12\ **(7)**\ ：SysTick中断服务函数。

实验现象
~~~~

进入软件调试，全速运行程序，从逻辑分析仪中可以看到两个线程的波形是完全同步，就好像CPU在同时干两件事情，具体仿真的波形图见图 9‑1和图 9‑2。

|idleth002|

图 9‑1 实验现象1

|idleth003|

图 9‑2 实验现象2

从图 9‑1和图 9‑2可以看出，flag1和flag2的高电平的时间为(0.1802-0.1602)s，刚好等于阻塞延时的20ms，所以实验现象跟代码要实现的功能是一致的。

.. |idleth002| image:: media/idle_thread/idleth002.png
   :width: 4.53472in
   :height: 2.02441in
.. |idleth003| image:: media/idle_thread/idleth003.png
   :width: 4.48611in
   :height: 2.32731in
