.. vim: syntax=rst

支持时间片
-----

在RT-Thread中，当同一个优先级下有两个或两个以上线程的时候，线程支持时间片功能，即我们可以指定线程持续运行一次的时间，单位为tick。假如有两个线程分别为线程2和线程3，他们的优先级都为3，线程2的时间片为2，线程3的时间片为3。当执行到优先级为3的线程时，会先执行线程2，直到线程2的时间片
耗完，然后再执行线程3，具体的实验波形图看本章最后的实验现象即可。

实现时间片
~~~~~

在线程控制块中添加时间片相关成员
^^^^^^^^^^^^^^^^

在线程控制块中添加时间片相关的成员，init_tick表示初始时间片，remaining_tick表示还剩下多少时间片，具体见代码清单 12‑1的加粗部分。

代码清单 12‑1 在线程控制块中添加时间片相关成员

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

**17 rt_ubase_t init_tick; /\* 初始时间片 \*/**

**18 rt_ubase_t remaining_tick; /\* 剩余时间片 \*/**

19

20 rt_uint8_t current_priority; /\* 当前优先级 \*/

21 rt_uint8_t init_priority; /\* 初始优先级 \*/

22 rt_uint32_t number_mask; /\* 当前优先级掩码 \*/

23

24 rt_err_t error; /\* 错误码 \*/

25 rt_uint8_t stat; /\* 线程的状态 \*/

26

27 struct rt_timer thread_timer; /\* 内置的线程定时器 \*/

28 };

修改线程初始化函数
^^^^^^^^^

在线程初始化函数rt_thread_init中添加时间片相关形参，并将线程控制块中与时间片相关的成员初始化好，具体见代码清单 12‑2的加粗部分。

代码清单 12‑2 修改线程初始化函数

1 rt_err_t rt_thread_init(struct rt_thread \*thread,

2 const char \*name,

3 void (*entry)(void \*parameter),

4 void \*parameter,

5 void \*stack_start,

6 rt_uint32_t stack_size,

7 rt_uint8_t priority,

**8 rt_uint32_t tick)**

9 {

10 /\* 线程对象初始化 \*/

11 /\* 线程结构体开头部分的成员就是rt_object_t类型 \*/

12 rt_object_init((rt_object_t)thread, RT_Object_Class_Thread, name);

13 rt_list_init(&(thread->tlist));

14

15 thread->entry = (void \*)entry;

16 thread->parameter = parameter;

17

18 thread->stack_addr = stack_start;

19 thread->stack_size = stack_size;

20

21 /\* 初始化线程栈，并返回线程栈指针 \*/

22 thread->sp = (void \*)rt_hw_stack_init( thread->entry,

23 thread->parameter,

24 (void \*)((char \*)thread->stack_addr + thread->stack_size - 4) );

25

26 thread->init_priority = priority;

27 thread->current_priority = priority;

28 thread->number_mask = 0;

29

30 /\* 错误码和状态 \*/

31 thread->error = RT_EOK;

32 thread->stat = RT_THREAD_INIT;

33

**34 /\* 时间片相关 \*/**

**35 thread->init_tick = tick;**

**36 thread->remaining_tick = tick;**

37

38 /\* 初始化线程定时器 \*/

39 rt_timer_init(&(thread->thread_timer), /\* 静态定时器对象 \*/

40 thread->name, /\* 定时器的名字，直接使用的是线程的名字 \*/

41 rt_thread_timeout, /\* 超时函数 \*/

42 thread, /\* 超时函数形参 \*/

43 0, /\* 延时时间 \*/

44 RT_TIMER_FLAG_ONE_SHOT); /\* 定时器的标志 \*/

45

46 return RT_EOK;

47 }

修改空闲线程初始化函数
'''''''''''

在空闲线。程初始化函数中指定空闲线程的时间片，通常很少线程的优先级会与空闲线程的优先级一样，时间片我们可以随便设置，这里我们示意性的设置为2，具体见代码清单 12‑3的加粗部分。

代码清单 12‑3 修改空闲线程初始化函数

1 void rt_thread_idle_init(void)

2 {

3

4 /\* 初始化线程 \*/

5 rt_thread_init(&idle,

6 "idle",

7 rt_thread_idle_entry,

8 RT_NULL,

9 &rt_thread_stack[0],

10 sizeof(rt_thread_stack),

11 RT_THREAD_PRIORITY_MAX-1,

**12 2); /\* 时间片 \*/**

13

14 /\* 启动空闲线程 \*/

15 rt_thread_startup(&idle);

16 }

修改系统时基更新函数
^^^^^^^^^^

在系统时基更新函数中添加与时间片相关的代码，具体见代码清单 12‑4的加粗部分。

代码清单 12‑4 修改系统时基更新函数

1 void rt_tick_increase(void)

2 {

3 struct rt_thread \*thread;

4

5

6 /\* 系统时基计数器加1操作,rt_tick是一个全局变量 \*/

7 ++ rt_tick;

8

**9 /\* 获取当前线程线程控制块 \*/**

**10 thread = rt_thread_self(); (1)**

**11**

**12 /\* 时间片递减 \*/**

**13 -- thread->remaining_tick; (2)**

**14**

**15 /\* 如果时间片用完，则重置时间片，然后让出处理器 \*/**

**16 if (thread->remaining_tick == 0) (3)**

**17 {**

**18 /\* 重置时间片 \*/**

**19 thread->remaining_tick = thread->init_tick; (4)**

**20**

**21 /\* 让出处理器 \*/**

**22 rt_thread_yield(); (5)**

**23 }**

24

25 /\* 扫描系统定时器列表 \*/

26 rt_timer_check();

27 }

代码清单 12‑4\ **(1)**\ ：获取当前线程线程控制块。

代码清单 12‑4\ **(2)**\ ：递减当前线程的时间片。

代码清单 12‑4\ **(3)**\ ：如果时间片用完，则重置时间片，然后让出处理器，具体是否真正的要让出处理器还要看当前线程下是否有两个以上的线程。

代码清单 12‑4\ **(4)**\ ：如果时间片耗完，则重置时间片。

代码清单 12‑4\ **(5)**\ ：调用rt_thread_yield让出处理器，该函数在thread.c中定义，具体实现见代码清单 12‑5。

代码清单 12‑5 rt_thread_yield函数定义

1 /*\*

2 \*

3 该函数将让当前线程让出处理器，调度器选择最高优先级的线程运行。当前让出处理器之后

4 ，

5 \* 当前线程还是在就绪态。

6 \*

7 \* @return RT_EOK

8 \*/

9 rt_err_t rt_thread_yield(void)

10 {

11 register rt_base_t level;

12 struct rt_thread \*thread;

13

14 /\* 关中断 \*/

15 level = rt_hw_interrupt_disable();

16

17 /\* 获取当前线程的线程控制块 \*/ **(1)**

18 thread = rt_current_thread;

19

20 /\* 如果线程在就绪态，且同一个优先级下不止一个线程 \*/ **(2)**

21 if ((thread->stat & RT_THREAD_STAT_MASK) == RT_THREAD_READY &&

22 thread->tlist.next != thread->tlist.prev)

23 {

24 /\* 将时间片耗完的线程从就绪列表移除 \*/

25 rt_list_remove(&(thread->tlist)); **(3)**

26

27 /\* 将线程插入到该优先级下的链表的尾部 \*/ **(4)**

28 rt_list_insert_before(&(rt_thread_priority_table[thread->current_priority]),

29 &(thread->tlist));

30

31 /\* 开中断 \*/

32 rt_hw_interrupt_enable(level);

33

34 /\* 执行调度 \*/

35 rt_schedule(); **(5)**

36

37 return RT_EOK;

38 }

39

40 /\* 开中断 \*/

41 rt_hw_interrupt_enable(level);

42

43 return RT_EOK;

44 }

代码清单 12‑5\ **(1)**\ ：获取当前线程线程控制块。

代码清单 12‑5\ **(2)**\ ：如果线程在就绪态，且同一个优先级下不止一个线程，则执行if里面的代码，否则函数返回。

代码清单 12‑5\ **(3)**\ ：将时间片耗完的线程从就绪列表移除。

代码清单 12‑5\ **(4)**\ ：将时间片耗完的线程插入到该优先级下的链表的尾部，把机会让给下一个线程。

代码清单 12‑5\ **(5)**\ ：执行调度。

修改main.c文件
~~~~~~~~~~

main.c文件的修改内容具体见代码清单 12‑6的加粗部分。

代码清单 12‑6 main.c文件内容

1 /\*

2 \\*

3 \* 包含的头文件

4 \\*

5 \*/

6

7 #include <rtthread.h>

8 #include <rthw.h>

9 #include "ARMCM3.h"

10

11

12 /\*

13 \\*

14 \* 全局变量

15 \\*

16 \*/

17 rt_uint8_t flag1;

18 rt_uint8_t flag2;

19 rt_uint8_t flag3;

20

21 extern rt_list_t rt_thread_priority_table[RT_THREAD_PRIORITY_MAX];

22

23 /\*

24 \\*

25 \* 线程控制块 & STACK & 线程声明

26 \\*

27 \*/

28

29

30 /\* 定义线程控制块 \*/

31 struct rt_thread rt_flag1_thread;

32 struct rt_thread rt_flag2_thread;

33 struct rt_thread rt_flag3_thread;

34

35 ALIGN(RT_ALIGN_SIZE)

36 /\* 定义线程栈 \*/

37 rt_uint8_t rt_flag1_thread_stack[512];

38 rt_uint8_t rt_flag2_thread_stack[512];

39 rt_uint8_t rt_flag3_thread_stack[512];

40

41 /\* 线程声明 \*/

42 void flag1_thread_entry(void \*p_arg);

43 void flag2_thread_entry(void \*p_arg);

44 void flag3_thread_entry(void \*p_arg);

45

46 /\*

47 \\*

48 \* 函数声明

49 \\*

50 \*/

51 void delay(uint32_t count);

52

53 /\*

54 \* @brief main函数

55 \* @param 无

56 \* @retval 无

57 \*

58 \* @attention

59 \\*

60 \*/

61 int main(void)

62 {

63 /\* 硬件初始化 \*/

64 /\* 将硬件相关的初始化放在这里，如果是软件仿真则没有相关初始化代码 \*/

65

66 /\* 关中断 \*/

67 rt_hw_interrupt_disable();

68

69 /\* SysTick中断频率设置 \*/

70 SysTick_Config( SystemCoreClock / RT_TICK_PER_SECOND );

71

72 /\* 系统定时器列表初始化 \*/

73 rt_system_timer_init();

74

75 /\* 调度器初始化 \*/

76 rt_system_scheduler_init();

77

78 /\* 初始化空闲线程 \*/

79 rt_thread_idle_init();

80

81 /\* 初始化线程 \*/

82 rt_thread_init( &rt_flag1_thread, /\* 线程控制块 \*/

83 "rt_flag1_thread", /\* 线程名字，字符串形式 \*/

84 flag1_thread_entry, /\* 线程入口地址 \*/

85 RT_NULL, /\* 线程形参 \*/

86 &rt_flag1_thread_stack[0], /\* 线程栈起始地址 \*/

87 sizeof(rt_flag1_thread_stack), /\* 线程栈大小，单位为字节 \*/

**88 2, /\* 优先级 \*/ (优先级)**

**89 4); /\* 时间片 \*/ (时间片)**

90 /\* 将线程插入到就绪列表 \*/

91 rt_thread_startup(&rt_flag1_thread);

92

93 /\* 初始化线程 \*/

94 rt_thread_init( &rt_flag2_thread, /\* 线程控制块 \*/

95 "rt_flag2_thread", /\* 线程名字，字符串形式 \*/

96 flag2_thread_entry, /\* 线程入口地址 \*/

97 RT_NULL, /\* 线程形参 \*/

98 &rt_flag2_thread_stack[0], /\* 线程栈起始地址 \*/

99 sizeof(rt_flag2_thread_stack), /\* 线程栈大小，单位为字节 \*/

**100 3, /\* 优先级 \*/ (优先级)**

**101 2); /\* 时间片 \*/ (时间片)**

102 /\* 将线程插入到就绪列表 \*/

103 rt_thread_startup(&rt_flag2_thread);

104

105

106 /\* 初始化线程 \*/

107 rt_thread_init( &rt_flag3_thread, /\* 线程控制块 \*/

108 "rt_flag3_thread", /\* 线程名字，字符串形式 \*/

109 flag3_thread_entry, /\* 线程入口地址 \*/

110 RT_NULL, /\* 线程形参 \*/

111 &rt_flag3_thread_stack[0], /\* 线程栈起始地址 \*/

112 sizeof(rt_flag3_thread_stack), /\* 线程栈大小，单位为字节 \*/

**113 3, /\* 优先级 \*/(优先级)**

**114 3); /\* 时间片 \*/(时间片)**

115 /\* 将线程插入到就绪列表 \*/

116 rt_thread_startup(&rt_flag3_thread);

117

118 /\* 启动系统调度器 \*/

119 rt_system_scheduler_start();

120 }

121

122 /\*

123 \\*

124 \* 函数实现

125 \\*

126 \*/

127 /\* 软件延时 \*/

128 void delay (uint32_t count)

129 {

130 for (; count!=0; count--);

131 }

132

133 /\* 线程1 \*/

134 void flag1_thread_entry( void \*p_arg )

135 {

136 for ( ;; )

137 {

138 flag1 = 1;

**139 rt_thread_delay(3); (阻塞延时)**

140 flag1 = 0;

**141 rt_thread_delay(3);**

142 }

143 }

144

145 /\* 线程2 \*/

146 void flag2_thread_entry( void \*p_arg )

147 {

148 for ( ;; )

149 {

150 flag2 = 1;

**151 //rt_thread_delay(2);**

**152 delay( 100 ); (软件延时)**

153 flag2 = 0;

**154 //rt_thread_delay(2);**

**155 delay( 100 );**

156 }

157 }

158

159 /\* 线程3 \*/

160 void flag3_thread_entry( void \*p_arg )

161 {

162 for ( ;; )

163 {

164 flag3 = 1;

**165 //rt_thread_delay(3);**

**166 delay( 100 ); (软件延时)**

167 flag3 = 0;

**168 //rt_thread_delay(3);**

**169 delay( 100 );**

170 }

171 }

172

173

174 void SysTick_Handler(void)

175 {

176 /\* 进入中断 \*/

177 rt_interrupt_enter();

178

179 /\* 更新时基 \*/

180 rt_tick_increase();

181

182 /\* 离开中断 \*/

183 rt_interrupt_leave();

184 }

代码清单 12‑6\ **(优先级)**\ ：线程1的优先级修改为2，线程2和线程3的优先级修改为3。

代码清单 12‑6\ **(时间片)**\ ：线程1的时间片设置为4（可是与线程1同优先级的线程没有，这里设置了时间片也没有什么鸟用，不信等下看实验现象），线程2和线程3的时间片设置为3。

代码清单 12‑6\ **(阻塞延时)**\ ：设置线程1高低电平的时间为3个tick，且延时要使用阻塞延时。

代码清单 12‑6\ **(软件延时)**\ ：将线程2和线程3的延时改成软件延时，因为这两个线程的优先级是相同的，当他们的时间片耗完的时候让出处理器进行系统调度，不会一直的占有CPU，所以可以使用软件延时，但是线程1却不可以，因为与线程1同优先级的线程没有，时间片功能不起作用，当时间片耗完的时候不
会让出CPU，会一直的占有CPU，所以不能使用软件延时。

实验现象
~~~~

进入软件调试，全速运行程序，逻辑分析仪中的仿真波形图具体见图 12‑1。

|slidin002|

图 12‑1 实验现象

从图 12‑1中可以看出线程1运行一个周期的时间为6个tick，与线程1初始化时设置的4个时间片不符，说明同一个优先级下只有一个线程时时间片不起作用。线程2和线程3运行一个周期的时间分别为2个tick和3个tick，且线程2运行的时候线程3是不运行的，从而说明我们的时间片功能起作用了，搞定。图
12‑1线程2和线程3运行的波形图现在是太密集了，一团黑，看不出代码的执行效果，我们将波形图放大之后，可以在线程要求的时间片内flag2和flag3进行了很多很多次的翻转，具体见图 12‑2。

|slidin003|

图 12‑2 实验现象2

.. |slidin002| image:: media/sliding/slidin002.png
   :width: 5.76806in
   :height: 1.8853in
.. |slidin003| image:: media/sliding/slidin003.png
   :width: 4.73377in
   :height: 1.39548in
