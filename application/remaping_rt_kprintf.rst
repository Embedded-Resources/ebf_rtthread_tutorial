.. vim: syntax=rst

重映射串口到rt_kprintf函数
------------------

在RT-Thread中，有一个打印函数rt_kprintf()供用户使用，方便在调试的时候输出各种信息。如果要想使用rt_kprintf()，则必须将控制台重映射到rt_kprintf()，这个控制台可以是串口、CAN、USB、以太网等输出设备，用的最多的就是串口，接下来我们讲解下如何将串口重定向到
rt_kprintf()。

rt_kprintf()函数定义
~~~~~~~~~~~~~~~~

rt_kprintf()函数在kservice.c中实现，是属于内核服务类的函数，具体实现见代码清单 15‑1。

代码清单 15‑1 rt_kprintf()函数定义

1 /*\*

2 \* @brief 这个函数用于向控制台打印特定格式的字符串

3 \*

4 \* @param fmt 指定的格式

5 \*/

6 void rt_kprintf(const char \*fmt, ...)

7 {

8 va_list args;

9 rt_size_t length;

10 static char rt_log_buf[RT_CONSOLEBUF_SIZE]; **(1)**

11

12 va_start(args, fmt);

13 /\* rt_vsnprintf的返回值length表示按照fmt

14 指定的格式写入到rt_log_buf的字符长度 \*/

15 length = rt_vsnprintf(rt_log_buf, sizeof(rt_log_buf) - 1, fmt, args); **(2)**

16 /\* 如果length超过RT_CONSOLEBUF_SIZE，则进行截短

17 即最多只能输出RT_CONSOLEBUF_SIZE个字符 \*/

18 if (length > RT_CONSOLEBUF_SIZE - 1)

19 length = RT_CONSOLEBUF_SIZE - 1;

20

21 /\* 使用设备驱动 \*/

22 #ifdef RT_USING_DEVICE **(3)**

23 if (_console_device == RT_NULL)

24 {

25 rt_hw_console_output(rt_log_buf);

26 }

27 else

28 {

29 rt_uint16_t old_flag = \_console_device->open_flag;

30

31 \_console_device->open_flag \|= RT_DEVICE_FLAG_STREAM;

32 rt_device_write(_console_device, 0, rt_log_buf, length);

33 \_console_device->open_flag = old_flag;

34 }

35 #else

36 /\* 没有使用设备驱动则由rt_hw_console_output函数处理，

37 该函数需要用户自己实现 \*/

38 rt_hw_console_output(rt_log_buf); **(4)**

39 #endif

40 va_end(args);

41 }

代码清单 15‑1\ **(1)**\ ：先定义一个字符缓冲区，大小由rt_config.h中的宏RT_CONSOLEBUF_SIZE定义，默认为128。

代码清单 15‑1\ **(2)**\ ：调用rt_vsnprintf函数，将要输出的字符按照fmt指定的格式打印到预先定义好的rt_log_buf缓冲区，然后我们将缓冲区的内容输出到控制台就行了，接下来就是选择使用什么控制台。

代码清单 15‑1\ **(3)**\ ：如果使用设备驱动，则通过设备驱动函数将rt_log_buf缓冲区的内容输出到控制台。如果设备控制台打开失败则由rt_hw_console_output函数处理，这个函数需要用户单独实现。

代码清单 15‑1\ **(4)**\ ：不使用设备驱动，rt_log_buf缓冲区的内容则由rt_hw_console_output()函数处理，这个函数需要用户单独实现。

自定义rt_hw_console_output函数
~~~~~~~~~~~~~~~~~~~~~~~~~

目前，我们不使用RT-Thread的设备驱动，那通过rt_kprintf输出的内容则由rt_hw_console_output函数处理，这个函数需要用户单独实现。其实，实现这个函数也很简单，只需要通过一个控制台将rt_log_buf缓冲区的内容发送出去即可，这个控制台可以是USB、串口、CAN等，使
用的最多的控制台则是串口。这里我们只讲解如何将串口控制台重映射到rt_kprintf函数，rt_hw_console_output函数在board.c实现，具体见代码清单 15‑2。

代码清单 15‑2 重映射串口控制台到rt_kprintf函数

1 /*\*

2 \* @brief 重映射串口DEBUG_USARTx到rt_kprintf()函数

3 \* Note：DEBUG_USARTx是在bsp_usart.h中定义的宏，默认使用串口1

4 \* @param str：要输出到串口的字符串

5 \* @retval 无

6 \*

7 \* @attention

8 \*

9 \*/

10 void rt_hw_console_output(const char \*str)

11 {

12 /\* 进入临界段 \*/

13 rt_enter_critical();

14

15 /\* 直到字符串结束 \*/

16 while (*str!='\0')

17 {

18 /\* 换行 \*/

19 if (*str=='\n')

20 {

21 USART_SendData(DEBUG_USARTx, '\r');

22 while (USART_GetFlagStatus(DEBUG_USARTx, USART_FLAG_TXE) == RESET);

23 }

24

25 USART_SendData(DEBUG_USARTx, \*str++);

26 while (USART_GetFlagStatus(DEBUG_USARTx, USART_FLAG_TXE) == RESET);

27 }

28

29 /\* 退出临界段 \*/

30 rt_exit_critical();

31 }

如果我们使用的是HAL库，rt_hw_console_output函数就需要做不一样的修改，使用HAL库的串口发送函数接口，具体见代码清单 15‑3加粗部分。

代码清单 15‑3重映射串口控制台到rt_kprintf函数

1 /*\*

2 \* @brief 重映射串口DEBUG_USARTx到rt_kprintf()函数

3 \* Note：DEBUG_USARTx是在bsp_usart.h中定义的宏，默认使用串口1

4 \* @param str：要输出到串口的字符串

5 \* @retval 无

6 \*

7 \* @attention

8 \*

9 \*/

10 void rt_hw_console_output(const char \*str)

11 {

12 /\* 进入临界段 \*/

13 rt_enter_critical();

14

15 /\* 直到字符串结束 \*/

16 while (*str!='\0') {

17 /\* 换行 \*/

18 if (*str=='\n') {

**19** **HAL_UART_Transmit( &UartHandle,(uint8_t \*)'\r',1,1000);**

20 }

**21** **HAL_UART_Transmit( &UartHandle,(uint8_t \*)(str++),1,1000);**

22 }

23

24 /\* 退出临界段 \*/

25 rt_exit_critical();

26 }

测试rt_kprintf函数
~~~~~~~~~~~~~~

硬件初始化
^^^^^

rt_kprintf函数输出的控制台使用的是开发板上的串口（野火STM32全系列的开发板都板载了USB转串口，然后通过跳帽默认接到了STM32的串口1），所以需要先要将裸机的串口驱动添加到工程并在开发环境中指定串口驱动头文件的编译路径，然后在board.c的rt_hw_board_init()函数中
对串口初始化，具体见代码清单 15‑4的加粗部分。

代码清单 15‑4 在rt_hw_board_init中添加串口初始化代码

1 void rt_hw_board_init()

2 {

3 /\* 初始化SysTick \*/

4 SysTick_Config( SystemCoreClock / RT_TICK_PER_SECOND );

5

6 /\* 硬件BSP初始化统统放在这里，比如LED，串口，LCD等 \*/

7

8 /\* 初始化开发板的LED \*/

9 LED_GPIO_Config();

10

**11 /\* 初始化开发板的串口 \*/**

**12 USART_Config();**

13

14

15 /\* 调用组件初始化函数 (use INIT_BOARD_EXPORT()) \*/

16 #ifdef RT_USING_COMPONENTS_INIT

17 rt_components_board_init();

18 #endif

19

20 #if defined(RT_USING_CONSOLE) && defined(RT_USING_DEVICE)

21 rt_console_set_device(RT_CONSOLE_DEVICE_NAME);

22 #endif

23

24 #if defined(RT_USING_USER_MAIN) && defined(RT_USING_HEAP)

25 rt_system_heap_init(rt_heap_begin_get(), rt_heap_end_get());

26 #endif

27 }

编写rt_kprintf测试代码
^^^^^^^^^^^^^^^^

当rt_kprintf函数对应的输出控制台初始化好之后（在rt_hw_board_init()完成），系统接下来会调用函数rt_show_version()来打印RT-Thread的版本号，该函数在kservice.c中实现，具体见代码清单 15‑5。

代码清单 15‑5 rt_show_version函数实现

1 void rt_show_version(void)

2 {

3 rt_kprintf("\n \\\\ \| /\n");

4 rt_kprintf("- RT - Thread Operating System\n");

5 rt_kprintf(" / \| \\\\ %d.%d.%d build %s\n",

6 RT_VERSION, RT_SUBVERSION, RT_REVISION, \__DATE__);

7 rt_kprintf(" 2006 - 2018 Copyright by rt-thread team\n");

8 }

我们也可以在线程中用rt_kprintf打印一些辅助信息，具体见代码清单 15‑6的加粗部分。

代码清单 15‑6 使用rt_kprintf在线程中打印调试信息

1 static void led1_thread_entry(void\* parameter)

2 {

3 while (1)

4 {

5 LED1_ON;

6 rt_thread_delay(500); /\* 延时500个tick \*/

**7 rt_kprintf("led1_thread running,LED1_ON\r\n");**

8

9 LED1_OFF;

10 rt_thread_delay(500); /\* 延时500个tick \*/

**11 rt_kprintf("led1_thread running,LED1_OFF\r\n");**

12 }

13 }

下载验证
^^^^

将程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到rt_kprintf的打印信息，具体见图
15‑1。

|remapi002|

图 15‑1rt_kprintf打印信息实验现象

.. |remapi002| image:: media/remaping_rt_kprintf/remapi002.png
   :width: 4.12687in
   :height: 3.27427in
