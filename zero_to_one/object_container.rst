.. vim: syntax=rst

对象容器的实现
-------

什么是对象
~~~~~

在RT-Thread中，所有的数据结构都称之为对象。

对象枚举定义
^^^^^^

其中线程，信号量，互斥量、事件、邮箱、消息队列、内存堆、内存池、设备和定时器在rtdef.h中有明显的枚举定义，即为每个对象打上了一个数字标签，具体见代码清单 8‑1。

代码清单 8‑1 对象类型枚举定义

1 enum rt_object_class_type

2 {

3 RT_Object_Class_Thread = 0, /\* 对象是线程 \*/

4 RT_Object_Class_Semaphore, /\* 对象是信号量 \*/

5 RT_Object_Class_Mutex, /\* 对象是互斥量 \*/

6 RT_Object_Class_Event, /\* 对象是事件 \*/

7 RT_Object_Class_MailBox, /\* 对象是邮箱 \*/

8 RT_Object_Class_MessageQueue, /\* 对象是消息队列 \*/

9 RT_Object_Class_MemHeap, /\* 对象是内存堆 \*/

10 RT_Object_Class_MemPool, /\* 对象是内存池 \*/

11 RT_Object_Class_Device, /\* 对象是设备 \*/

12 RT_Object_Class_Timer, /\* 对象是定时器 \*/

13 RT_Object_Class_Module, /\* 对象是模块 \*/

14 RT_Object_Class_Unknown, /\* 对象未知 \*/

15 RT_Object_Class_Static = 0x80 /\* 对象是静态对象 \*/

16 };

对象数据类型定义
^^^^^^^^

在rtt中，为了方便管理这些对象，专门定义了一个对象类型数据结构，具体见代码清单 8‑2。

代码清单 8‑2 对象数据类型定义

1 struct rt_object

2 {

3 char name[RT_NAME_MAX]; **(1)** /\* 内核对象的名字 \*/

4 rt_uint8_t type; **(2)** /\* 内核对象的类型 \*/

5 rt_uint8_t flag; **(3)** /\* 内核对象的状态 \*/

6

7

8 rt_list_t list; **(4)** /\* 内核对象的列表节点 \*/

9 };

10 typedef struct rt_object \*rt_object_t; **(5)** /*内核对象数据类型重定义*/

代码清单 8‑2\ **(1)**\ ：对象名字，字符串形式，方便调试，最大长度由rt_config.h中的宏RT_NAMA_MAX决定，默认定义为8。

代码清单 8‑2\ **(2)**\ ：对象的类型，RT-Thread为每一个对象都打上了数字标签，取值由rt_object_class_type 枚举类型限定，具体见代码清单 8‑1。

代码清单 8‑2\ **(3)**\ ：对象的状态。

代码清单 8‑2\ **(4)**\ ：对象的列表节点，每个对象都可以通过自己的列表节点list将自己挂到容器列表中，什么是容器接下来会讲解到。

代码清单 8‑2\ **(5)**\ ：对象数据类型，RT-Thread中会为每一个新的结构体用typedef重定义一个指针类型的数据结构。

在线程控制块中添加对象成员
^^^^^^^^^^^^^

在RT-Thread中，每个对象都会有对应的一个结构体，这个结构体叫做该对象的控制块。如线程会有一个线程控制块，定时器会有一个定时器控制块，信号量会有信号量控制块等。这些控制块的开头都会包含一个内核对象结构体，或者直接将对象结构体的成员放在对象控制块结构体的开头。其中线程控制块的开头放置的就是对象结
构体的成员，具体见代码清单 8‑3开头的加粗部分代码。这里我们只讲解往线程控制块里面添加对象结构体成员，其它内核对象的都是直接在其开头使用struct rt_object 直接定义一个内核对象变量。

代码清单 8‑3 在线程控制块中添加对象成员

1 struct rt_thread {

2 /\* rt 对象 \*/

**3 char name[RT_NAME_MAX]; /\* 对象的名字 \*/**

**4 rt_uint8_t type; /\* 对象类型 \*/**

**5 rt_uint8_t flags; /\* 对象的状态 \*/**

**6 rt_list_t list; /\* 对象的列表节点 \*/**

7

8 rt_list_t tlist; /\* 线程链表节点 \*/

9 void \*sp; /\* 线程栈指针 \*/

10 void \*entry; /\* 线程入口地址 \*/

11 void \*parameter; /\* 线程形参 \*/

12 void \*stack_addr; /\* 线程起始地址 \*/

13 rt_uint32_t stack_size; /\* 线程栈大小，单位为字节 \*/

14 };

什么是容器
~~~~~

在rtt中，每当用户创建一个对象，如线程，就会将这个对象放到一个叫做容器的地方，这样做的目的是为了方便管理，这时用户会问，管理什么？在RT-
Thread的组件finsh的使用中，就需要使用到容器，通过扫描容器的内核对象来获取各个内核对象的状态，然后输出调试信息。目前，我们只需要知道所有创建的对象都会被放到容器中即可。

那什么是容器，从代码上看，容器就是一个数组，是一个全局变量，数据类型为struct rt_object_information，在object.c中定义，具体见代码清单 8‑4，示意图具体见图 8‑1。

容器的定义
^^^^^

代码清单 8‑4 rtt容器的定义

1 static struct rt_object_information **(1)**

2 rt_object_container[RT_Object_Info_Unknown] = { **(2)**

3 /\* 初始化对象容器 - 线程 \*/ **(3)**

4 {

5 RT_Object_Class_Thread, **(3)-①**

6 \_OBJ_CONTAINER_LIST_INIT(RT_Object_Info_Thread), **(3)-②**

7 sizeof(struct rt_thread) **(3)-③**

8 },

9

10 #ifdef RT_USING_SEMAPHORE **(4)**

11 /\* 初始化对象容器 - 信号量 \*/

12 {

13 RT_Object_Class_Semaphore,

14 \_OBJ_CONTAINER_LIST_INIT(RT_Object_Info_Semaphore),

15 sizeof(struct rt_semaphore)

16 },

17 #endif

18

19 #ifdef RT_USING_MUTEX **(5)**

20 /\* 初始化对象容器 - 互斥量 \*/

21 {

22 RT_Object_Class_Mutex,

23 \_OBJ_CONTAINER_LIST_INIT(RT_Object_Info_Mutex),

24 sizeof(struct rt_mutex)

25 },

26 #endif

27

28 #ifdef RT_USING_EVENT **(6)**

29 /\* 初始化对象容器 - 事件 \*/

30 {

31 RT_Object_Class_Event,

32 \_OBJ_CONTAINER_LIST_INIT(RT_Object_Info_Event),

33 sizeof(struct rt_event)

34 },

35 #endif

36

37 #ifdef RT_USING_MAILBOX **(7)**

38 /\* 初始化对象容器 - 邮箱 \*/

39 {

40 RT_Object_Class_MailBox,

41 \_OBJ_CONTAINER_LIST_INIT(RT_Object_Info_MailBox),

42 sizeof(struct rt_mailbox)

43 },

44 #endif

45

46 #ifdef RT_USING_MESSAGEQUEUE **(8)**

47 /\* 初始化对象容器 - 消息队列 \*/

48 {

49 RT_Object_Class_MessageQueue,

50 \_OBJ_CONTAINER_LIST_INIT(RT_Object_Info_MessageQueue),

51 sizeof(struct rt_messagequeue)

52 },

53 #endif

54

55 #ifdef RT_USING_MEMHEAP **(9)**

56 /\* 初始化对象容器 - 内存堆 \*/

57 {

58 RT_Object_Class_MemHeap,

59 \_OBJ_CONTAINER_LIST_INIT(RT_Object_Info_MemHeap),

60 sizeof(struct rt_memheap)

61 },

62 #endif

63

64 #ifdef RT_USING_MEMPOOL **(10)**

65 /\* 初始化对象容器 - 内存池 \*/

66 {

67 RT_Object_Class_MemPool,

68 \_OBJ_CONTAINER_LIST_INIT(RT_Object_Info_MemPool),

69 sizeof(struct rt_mempool)

70 },

71 #endif

72

73 #ifdef RT_USING_DEVICE **(11)**

74 /\* 初始化对象容器 - 设备 \*/

75 {

76 RT_Object_Class_Device,

77 \_OBJ_CONTAINER_LIST_INIT(RT_Object_Info_Device),

78 sizeof(struct rt_device)

79 },

80 #endif

81

82 /\* 初始化对象容器 - 定时器 \*/ **(12)**

83 /\*

84 {

85 RT_Object_Class_Timer,

86 \_OBJ_CONTAINER_LIST_INIT(RT_Object_Info_Timer),

87 sizeof(struct rt_timer)

88 },

89 \*/

90 #ifdef RT_USING_MODULE **(13)**

91 /\* 初始化对象容器 - 模块 \*/

92 {

93 RT_Object_Class_Module,

94 \_OBJ_CONTAINER_LIST_INIT(RT_Object_Info_Module),

95 sizeof(struct rt_module)

96 },

97 #endif

98 };

|object002|

图 8‑1 对象容器示意图

代码清单 8‑4 **(1)**\ ：容器是一个全部变量的数组，数据类型为struct rt_object_information，这是一个结构体类型，包含对象的三个信息，分别为对象类型、对象列表节点头和对象的大小，在rtdef.h中定义，具体实现见代码清单 8‑5。

代码清单 8‑5 内核对象信息结构体定义

1 struct rt_object_information {

2 enum rt_object_class_type type; **(1)** /\* 对象类型 \*/

3 rt_list_t object_list; **(2)** /\* 对象列表节点头 \*/

4 rt_size_t object_size; **(3)** /\* 对象大小 \*/

5 };

代码清单 8‑5 **(1)**\ ：对象的类型，取值只能是rt_object_class_type枚举类型，具体取值见代码清单 8‑1。

代码清单 8‑5 **(2)**\ ：对象列表节点头，每当对象创建时，对象就会通过他们控制块里面的list节点将自己挂到对象容器中的对应列表，同一类型的对象是挂到对象容器中同一个对象列表的，容器数组的小标对应的就是对象的类型。

代码清单 8‑5\ **(3)**\ ：对象的大小，可直接通过sizeof(对象控制块类型)获取。

代码清单 8‑4 **(2)**\ ：容器的大小由RT_Object_Info_Unknown决定，RT_Object_Info_Unknown是一个枚举类型的变量，在rt_object_info_type这个枚举结构体里面定义，具体见代码清单 8‑6。

代码清单 8‑6 对象容器数组的下标定义

1 /\*

2 \* 对象容器数组的下标定义，决定容器的大小

3 \*/

4 enum rt_object_info_type

5 {

6 RT_Object_Info_Thread = 0, /\* 对象是线程 \*/

7 #ifdef RT_USING_SEMAPHORE

8 RT_Object_Info_Semaphore, /\* 对象是信号量 \*/

9 #endif

10 #ifdef RT_USING_MUTEX

11 RT_Object_Info_Mutex, /\* 对象是互斥量 \*/

12 #endif

13 #ifdef RT_USING_EVENT

14 RT_Object_Info_Event, /\* 对象是事件 \*/

15 #endif

16 #ifdef RT_USING_MAILBOX

17 RT_Object_Info_MailBox, /\* 对象是邮箱 \*/

18 #endif

19 #ifdef RT_USING_MESSAGEQUEUE

20 RT_Object_Info_MessageQueue, /\* 对象是消息队列 \*/

21 #endif

22 #ifdef RT_USING_MEMHEAP

23 RT_Object_Info_MemHeap, /\* 对象是内存堆 \*/

24 #endif

25 #ifdef RT_USING_MEMPOOL

26 RT_Object_Info_MemPool, /\* 对象是内存池 \*/

27 #endif

28 #ifdef RT_USING_DEVICE

29 RT_Object_Info_Device, /\* 对象是设备 \*/

30 #endif

31 RT_Object_Info_Timer, /\* 对象是定时器 \*/

32 #ifdef RT_USING_MODULE

33 RT_Object_Info_Module, /\* 对象是模块 \*/

34 #endif

35 RT_Object_Info_Unknown, /\* 对象未知 \*/

36 };

从代码清单 8‑6可以看出RT_Object_Info_Unknown位于枚举结构体的最后，它的具体取值由前面的成员多少决定，前面的成员是否有效都是通过宏定义来决定的，只有当在rtconfig.h中定义了相应的宏，对应的枚举成员才会有效，默认在这些宏都没有定义的情况下只有RT_Object_Info
_Thread和RT_Object_Info_Timer有效，此时RT_Object_Info_Unknown的值等于2。当这些宏全部有效，RT_Object_Info_Unknown的值等于11，即容器的大小为12，此时是最大。C语言知识：如果枚举类型的成员值没有具体指定，那么后一个值是在前一个成
员值的基础上加1。

代码清单 8‑4 **(3)**\ ：初始化对象容器—线程，线程是rtt里面最基本的对象，是必须存在的，跟其它的对象不一样，没有通过宏定义来选择，接下来下面的信号量、邮箱都通过对应的宏定义来控制是否初始化，即只有在创建了相应的对象后，才在对象容器里面初始化。

代码清单 8‑4 **(3)-①**\ ：初始化对象类型为线程。

代码清单 8‑4 **(3)-②**\ ：初始化对象列表节点头里面的next和prev两个节点指针分别指向自身，具体见图 8‑1。_OBJ_CONTAINER_LIST_INIT()是一个带参宏，用于初始化一个节点list，在object.c中定义，具体见代码清单 8‑7。

代码清单 8‑7 \_OBJ_CONTAINER_LIST_INIT()宏定义

1 #define \_OBJ_CONTAINER_LIST_INIT(c) \\

2{&(rt_object_container[c].object_list), &(rt_object_container[c].object_list)}

代码清单 8‑4 **(3)-③**\ ：获取线程对象的大小，即整个线程控制块的大小。

代码清单 8‑4 **(4)**\ ：初始化对象容器—信号量，由宏RT_USING_SEMAPHORE决定。

代码清单 8‑4 **(5)**\ ：初始化对象容器—互斥量，由宏RT_USING_MUTEX决定。

代码清单 8‑4 **(6)**\ ：初始化对象容器—事件，由宏RT_USING_EVENT决定。

代码清单 8‑4 **(7)**\ ：初始化对象容器—邮箱，由宏RT_USING_MAILBOX决定。

代码清单 8‑4 **(8)**\ ：初始化对象容器—消息队列，由宏RT_USING_MESSAGEQUEUE决定。

代码清单 8‑4 **(9)**\ ：初始化对象容器—内存堆，由宏RT_USING_MEMHEAP决定。

代码清单 8‑4 **(10)**\ ：初始化对象容器—内存池，由宏RT_USING_MEMPOOL决定。

代码清单 8‑4 **(11)**\ ：初始化对象容器—设备，由宏RT_USING_DEVICE决定。

代码清单 8‑4 **(12)**\ ：初始化对象容器—定时器，每个线程在创建的时候都会自带一个定时器，但是目前我们还没有在线程中加入定时器，所以这部分初始化我们先注释掉，等加入定时器的时候再释放。

代码清单 8‑4 **(13)**\ ：初始化对象容器—模块，由宏RT_USING_MODULE决定。

容器的接口实现
~~~~~~~

容器接口相关的函数均在object.c中实现。

获取指定类型的对象信息
^^^^^^^^^^^

从容器中获取指定类型的对象的信息由函数rt_object_get_information()实现，具体定义见代码清单 8‑8。

代码清单 8‑8 rt_object_get_information()函数定义

1 struct rt_object_information \*

2 rt_object_get_information(enum rt_object_class_type type)

3 {

4 int index;

5

6 for (index = 0; index < RT_Object_Info_Unknown; index ++) {

7 if (rt_object_container[index].type == type) {

8 return &rt_object_container[index];

9 }

10 }

11

12 return RT_NULL;

13 }

我们知道，容器在定义的时候，大小是被固定的，由RT_Object_Info_Unknown这个枚举值决定，但容器里面的成员是否初始化就不一定了，其中线程和定时器这两个对象默认会被初始化，剩下的其它对象由对应的宏决定。rt_object_get_information()会遍历整个容器对象，如果对象的
类型等于我们指定的类型，那么就返回该容器成员的地址，地址的类型为struct rt_object_information。

对象初始化
^^^^^

每创建一个对象，都需要先将其初始化，主要分成两个部分的工作，首先将对象控制块里面与对象相关的成员初始化，然后将该对象插入到对象容器中，具体的代码实现见代码清单 8‑9。

代码清单 8‑9 对象初始化rt_object_init()函数定义

1 /*\*

2 \* 该函数将初始化对象并将对象添加到对象容器中

3 \*

4 \* @param object 要初始化的对象

5 \* @param type 对象的类型

6 \* @param name 对象的名字，在整个系统中，对象的名字必须是唯一的

7 \*/

8 void rt_object_init(struct rt_object \*object, **(1)**

9 enum rt_object_class_type type, **(2)**

10 const char \*name) **(3)**

11 {

12 register rt_base_t temp;

13 struct rt_object_information \*information;

14

15 /\* 获取对象信息，即从容器里拿到对应对象列表头指针 \*/

16 information = rt_object_get_information(type); **(4)**

17

18 /\* 设置对象类型为静态 \*/

19 object->type = type \| RT_Object_Class_Static; **(5)**

20

21 /\* 拷贝名字 \*/

22 rt_strncpy(object->name, name, RT_NAME_MAX); **(6)**

23

24 /\* 关中断 \*/

25 temp = rt_hw_interrupt_disable(); **(7)**

26

27 /\* 将对象插入到容器的对应列表中，不同类型的对象所在的列表不一样 \*/

28 rt_list_insert_after(&(information->object_list), &(object->list)); **(8)**

29

30 /\* 使能中断 \*/

31 rt_hw_interrupt_enable(temp); **(9)**

32 }

代码清单 8‑9\ **(1)**\ ：要初始化的对象。我们知道每个对象的控制块开头的成员都是对象信息相关的成员，比如一个线程控制块，它的开头前面4个成员都是与对象信息相关的，在调用rt_object_init()函数的时候，只需将线程控制块强制类型转化为struct
rt_object作为第一个形参即可。

代码清单 8‑9\ **(2)**\ ：对象的类型，是一个数字化的枚举值，具体见代码清单 8‑1。

代码清单 8‑9\ **(3)**\ ：对象的名字，字符串形式，在整个系统中，对象的名字必须是唯一的。

代码清单 8‑9\ **(4)**\ ：获取对象信息，即从容器里拿到对应对象列表头指针。容器是一个定义好的全局数组，可以直接操作。

代码清单 8‑9\ **(5)**\ ：设置对象类型为静态。

代码清单 8‑9\ **(6)**\ ：拷贝名字。rt_strncpy()是字符串拷贝函数，在kservice.c（kservice.c第一次使用需要在rtthread\3.0.3\src下新建，然后添加到工程rtt/source组中）中定义，在rtthread.h声明，具体代码实现见代码清单
8‑10。

代码清单 8‑10 rt_strncpy()函数定义

1 /*\*

2 \* 该函数将指定个数的字符串从一个地方拷贝到另外一个地方

3 \*

4 \* @param dst 字符串拷贝的目的地

5 \* @param src 字符串从哪里拷贝

6 \* @param n 要拷贝的最大长度

7 \*

8 \* @return 结果

9 \*/

10 char \*rt_strncpy(char \*dst, const char \*src, rt_ubase_t n)

11 {

12 if (n != 0)

13 {

14 char \*d = dst;

15 const char \*s = src;

16

17 do

18 {

19 if ((*d++ = \*s++) == 0)

20 {

21 /\* NUL pad the remaining n-1 bytes \*/

22 while (--n != 0)

23 \*d++ = 0;

24 break;

25 }

26 }

27 while (--n != 0);

28 }

29

30 return (dst);

31 }

代码清单 8‑9\ **(7)**\ ：关中断，接下来链表的操作不希望被中断。

代码清单 8‑9\ **(8)**\ ：将对象插入到容器的对应列表中，不同类型的对象所在的列表不一样。比如创建了两个线程，他们在容器列表中的示意图具体见。

|object003|

图 8‑2 在容器中插入两个线程

代码清单 8‑9\ **(9)**\ ：使能中断。

调用对象初始化函数
^^^^^^^^^

对象初始化函数在线程初始化函数里面被调用，具体见代码清单 8‑11的加粗部分。如果创建了两个线程，在线程初始化之后，线程通过自身的list节点将自身挂到容器的对象列表中，在容器中的示意图具体见图 8‑2。

代码清单 8‑11 在线程初始化中添加对象初始化功能

1 rt_err_t rt_thread_init(struct rt_thread \*thread,

**2 const char \*name, (1)**

3 void (*entry)(void \*parameter),

4 void \*parameter,

5 void \*stack_start,

6 rt_uint32_t stack_size)

7 {

**8 /\* 线程对象初始化 \*/**

**9 /\* 线程结构体开头部分的4个成员就是rt_object_t成员 \*/**

**10 rt_object_init((rt_object_t)thread, RT_Object_Class_Thread, name); (2)**

11

12

13 rt_list_init(&(thread->tlist));

14

15 thread->entry = (void \*)entry;

16 thread->parameter = parameter;

17

18 thread->stack_addr = stack_start;

19 thread->stack_size = stack_size;

20

21 /\* 初始化线程栈，并返回线程栈指针 \*/

22 thread->sp = (void \*)rt_hw_stack_init

23 ( thread->entry,

24 thread->parameter,

25 (void \*)((char \*)thread->stack_addr + thread->stack_size - 4)

26 );

27

28 return RT_EOK;

29 }

实验现象
~~~~

本章没有实验，充分理解本章内容即可。

.. |object002| image:: media/object_container/object002.png
   :width: 3.91667in
   :height: 4.95295in
.. |object003| image:: media/object_container/object003.png
   :width: 3.736in
   :height: 3.66854in
