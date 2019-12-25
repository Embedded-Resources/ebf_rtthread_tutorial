.. vim: syntax=rst

事件
------------------------

事件的基本概念
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

事件是一种实现线程间通信的机制，主要用于实现线程间的同步，但事件通信只能是事件类型的通信，无数据传输。与信号量不同的是，它可以实现一对多，多对多的同步。即一个线程可以等待多个事件的发生：可以是任意一个事件发生时唤醒线程进行事件处理；也可以是几个事件都发生后才唤醒线程进行事件处理。同样，事件也可以是多
个线程同步多个事件。

事件集合用32位无符号整型变量来表示，每一位代表一个事件，线程通过“逻辑与”或“逻辑或”与一个或多个事件建立关联，形成一个事件集。事件的“逻辑或”也称作是独立型同步，指的是线程感兴趣的所有事件任一件发生即可被唤醒；事件“逻辑与”也称为是关联型同步，指的是线程感兴趣的若干事件都发生时才被唤醒。

多线程环境下，线程之间往往需要同步操作，一个事件发生即是一个同步。事件可以提供一对多、多对多的同步操作。一对多同步模型：一个线程等待多个事件的触发；多对多同步模型：多个线程等待多个事件的触发。

线程可以通过创建事件来实现事件的触发和等待操作。RT-Thread的事件仅用于同步，不提供数据传输功能。

RT-Thread提供的事件具有如下特点：

-  事件只与线程相关联，事件相互独立，一个32位的事件集合（set变量），用于标识该线程发生的事件类型，其中每一位表示一种事件类型（0表示该事件类型未发生、1表示该事件类型已经发生），一共32种事件类型。

-  事件仅用于同步，不提供数据传输功能。

-  事件无排队性，即多次向线程发送同一事件(如果线程还未来得及读走)，等效于只发送一次。

-  允许多个线程对同一事件进行读写操作。

-  支持事件等待超时机制。

在RT-Thread实现中，每个线程都拥有一个事件信息标记，它有三个属性，分别是RT_EVENT_FLAG_AND(逻辑与)，RT_EVENT_FLAG_OR(逻辑或）以及RT_EVENT_FLAG_CLEAR(清除标记）。当线程等待事件同步时，可以通过32个事件标志和这个事件信息标记来判断当前接收
的事件是否满足同步条件。

事件的应用场景
~~~~~~~

RT-Thread的事件用于事件类型的通讯，无数据传输，也就是说，我们可以用事件来做标志位，判断某些事件是否发生了，然后根据结果做处理，那很多人又会问了，为什么我不直接用变量做标志呢，岂不是更好更有效率？非也非也，若是在裸机编程中，用全局变量是最为有效的方法，这点我不否认，但是在操作系统中，使用全局
变量就要考虑以下问题了：

-  如何对全局变量进行保护呢，防止多线程同时对它进行访问？

-  如何让内核对事件进行有效管理呢？使用全局变量的话，就需要在线程中轮询查看事件是否发送，这简直就是在浪费CPU资源啊，还有等待超时机制，使用全局变量的话需要用户自己去实现。

所以，在操作系统中，还是使用操作系统给我们提供的通信机制就好了，简单方便还实用。

在某些场合，可能需要多个时间发生了才能进行下一步操作，比如一些危险机器的启动，需要检查各项指标，当指标不达标的时候，无法启动，但是检查各个指标的时候，不能一下子检测完毕啊，所以，需要事件来做统一的等待，当所有的事件都完成了，那么机器才允许启动，这只是事件的其中一个应用。

事件可使用于多种场合，它能够在一定程度上替代信号量，用于线程间同步。一个线程或中断服务例程发送一个事件给事件对象，而后等待的线程被唤醒并对相应的事件进行处理。但是它与信号量不同的是，事件的发送操作是不可累计的，而信号量的释放动作是可累计的。事件另外一个特性是，接收线程可等待多种事件，即多个事件对应一
个线程或多个线程。同时按照线程等待的参数，可选择是“逻辑或”触发还是“逻辑与”触发。这个特性也是信号量等所不具备的，信号量只能识别单一同步动作，而不能同时等待多个事件的同步。

各个事件可分别发送或一起发送给事件对象，而线程可以等待多个事件，线程仅对感兴趣的事件进行关注。当有它们感兴趣的事件发生时并且符合感兴趣的条件，线程将被唤醒并进行后续的处理动作。

事件的运作机制
~~~~~~~

接收事件时，可以根据入感兴趣的参事件类型接收事件的单个或者多个事件类型。事件接收成功后，必须使用RT_EVENT_FLAG_CLEA选项来清除已接收到的事件类型，否则不会清除已接收到的事件。用户可以自定义通过传入参数选择读取模式option，是等待所有感兴趣的事件还是等待感兴趣的任意一个事件。

发送事件时，对指定事件写入指定的事件类型，设置事件集合set的对应事件位为1，可以一次同时写多个事件类型，发送事件会触发线程调度。

清除事件时，根据入参数事件句柄和待清除的事件类型，对事件对应位进行清0操作。事件不与线程相关联，事件相互独立，一个32位的变量（事件集合set），用于标识该线程发生的事件类型，其中每一位表示一种事件类型（0表示该事件类型未发生、1表示该事件类型已经发生），一共32种事件类型具体见图 21‑1。

|event002|

图 21‑1事件集合set（一个32位的变量）

事件唤醒机制，当线程因为等待某个或者多个事件发生而进入阻塞态，当事件发生的时候会被唤醒，其过程具体见图 21‑2。

|event003|

图 21‑2事件唤醒线程示意图

线程1对事件3或事件5感兴趣（逻辑或RT_EVENT_FLAG_OR），当发生其中的某一个事件都会被唤醒，并且执行相应操作。而线程2对事件3与事件5感兴趣（逻辑与RT_EVENT_FLAG_AND），当且仅当事件3与事件5都发生的时候，线程2才会被唤醒，如果只有一个其中一个事件发生，那么线程还是会继
续等待事件发生。如果接在收事件函数中option设置了清除事件位，那么当线程唤醒后将把事件3和事件5的事件标志清零，否则事件标志将依然存在。

事件控制块
~~~~~

事件的使用很简单，每个对事件的操作的函数都是根据事件控制块来进行操作的，事件控制块包含了一个32位的set变量，其变量的各个位表示一个事件，每一位代表一个事件的发生，利用逻辑或、逻辑与等实现不同事件的不同唤醒处理，具体见代码清单 21‑1。

代码清单 21‑1事件控制块

1 struct rt_event {

2 struct rt_ipc_object parent;

3

4 rt_uint32_t set; /**< 事件标志位 \*/

5 };

6 typedef struct rt_event \*rt_event_t; /\* rt_event_t是指向事件结构体的指针 \*/

事件属于内核对象，也会在自身结构体里面包含一个内核对象类型的成员，通过这个成员可以将事件挂到系统对象容器里面。rt_event对象从rt_ipc_object中派生，由IPC容器管理。

事件函数接口讲解
~~~~~~~~

事件创建函数rt_event_create()
^^^^^^^^^^^^^^^^^^^^^^^

事件创建函数，顾名思义，就是创建一个事件，与其他内核对象一样，都是需要先创建才能使用的资源，RT-
Thread给我们提供了一个创建事件的函数rt_event_create()，当创建一个事件时，内核首先创建一个事件控制块，然后对该事件控制块进行基本的初始化，创建成功返回事件句柄；创建失败返回RT_NULL。所以，在使用创建函数之前，我们需要先定义有个事件的句柄，事件创建的源码具体见代码清单
21‑2。

代码清单 21‑2事件创建函数rt_event_create()源码

1 rt_event_t rt_event_create(const char \*name, rt_uint8_t flag) **(1)**

2 {

3 rt_event_t event; **(2)**

4

5 RT_DEBUG_NOT_IN_INTERRUPT;

6

7 /\* 分配对象 \*/

8 event = (rt_event_t)rt_object_allocate(RT_Object_Class_Event, name);

9 if (event == RT_NULL) **(3)**

10 return event;

11

12 /\* 设置阻塞唤醒的模式 \*/

13 event->parent.parent.flag = flag; **(4)**

14

15 /\* 初始化事件对象 \*/

16 rt_ipc_object_init(&(event->parent)); **(5)**

17

18 /\* 事件集合清零 \*/

19 event->set = 0; **(6)**

20

21 return event; **(7)**

22 }

23 RTM_EXPORT(rt_event_create);

代码清单 21‑2\ **(1)**\ ：name ：事件的名称，由用户自己定义。flag ：事件阻塞唤醒模式。

代码清单 21‑2\ **(2)** ： 创建一个事件控制块。

代码清单 21‑2\ **(3)**\ ：分配事件对象，调用rt_object_allocate()函数将从对象系统分配对象，为创建的事件分配一个事件的对象，并且命名对象名称，在系统中，对象的名称必须是唯一的。

代码清单 21‑2\ **(4)**\ ：设置事件的阻塞唤醒模式，创建的事件由于指定的flag不同，而有不同的意义： 使用RT_IPC_FLAG_PRIO优先级flag创建的IPC对象，在多个线程等待资源时，将由优先级高的线程优先获得资源。而使用RT_IPC_FLAG_FIFO先进先出flag创建的
IPC对象，在多个线程等待资源时，将按照先来先得的顺序获得资源。RT_IPC_FLAG_PRIO与RT_IPC_FLAG_FIFO均在rtdef.h中有定义。

代码清单 21‑2\ **(5)**\ ：初始化事件内核对象。调用rt_ipc_object_init()函数会初始化一个链表用于记录访问此事件而阻塞的线程。

代码清单 21‑2\ **(6)**\ ：事件集合清零，因为现在是创建事件，还没有事件发生，所以事件集合中所有位都为0。

代码清单 21‑2\ **(7)**\ ：创建成功返回事件对象的句柄，创建失败返回RT_NULL。。

事件创建函数的源码都那么简单，其使用更为简单，不过需要我们在使用前定义一个指向事件控制块的指针，也就是常说的事件句柄，当事件创建成功，我们就可以根据我们定义的事件句柄来调用RT-Thread的事件函数进行操作，具体见代码清单 21‑3加粗部分。

代码清单 21‑3事件创建函数rt_event_create()实例

1 /\* 定义事件控制块(句柄) \*/

2 static rt_event_t test_event = RT_NULL;

3 /\* 创建一个事件 \*/

**4 test_event = rt_event_create("test_event",/\* 事件标志组名字 \*/**

**5 RT_IPC_FLAG_PRIO); /\* 事件模式 FIFO(0x00)*/**

6 if (test_event != RT_NULL)

7 rt_kprintf("事件创建成功！\n\n");

事件删除函数rt_event_delete()
^^^^^^^^^^^^^^^^^^^^^^^

在很多场合，某些事件只用一次的，就好比在事件应用场景说的危险机器的启动，假如各项指标都达到了，并且机器启动成功了，那这个事件之后可能就没用了，那就可以进行销毁了。想要删除事件怎么办呢？RT-
Thread给我们提供了一个删除事件的函数——rt_event_delete()，使用它就能将事件进行删除了。当系统不再使用事件对象时，可以通过删除事件对象控制块来释放系统资源，具体见代码清单 21‑4。

代码清单 21‑4事件删除函数rt_event_delete()源码

1 rt_err_t rt_event_delete(rt_event_t event) **(1)**

2 {

3 /\* 事件句柄检查 \*/

4 RT_ASSERT(event != RT_NULL); **(2)**

5

6 RT_DEBUG_NOT_IN_INTERRUPT;

7

8 /\* 恢复所有阻塞在此事件的线程 \*/

9 rt_ipc_list_resume_all(&(event->parent.suspend_thread)); **(3)**

10

11 /\* 删除事件对象 \*/

12 rt_object_delete(&(event->parent.parent)); **(4)**

13

14 return RT_EOK; **(5)**

15 }

16 RTM_EXPORT(rt_event_delete);

代码清单 21‑4\ **(1)**\ ：event是我们自己定义的事件句柄，根据事件句柄进行删除操作。

代码清单 21‑4\ **(2)**\ ：检查事件句柄event是否有效，如果它是未定义或者未创建的事件句柄，那么是无法进行删除操作的。

代码清单 21‑4\ **(3)**\ ：调用rt_ipc_list_resume_all()函数将所有因为访问此事件的而阻塞的线程从阻塞态中唤醒，所有被唤醒的线程的返回值是-
RT_ERROR，一般不这样子使用，所以在删除的时候，应先确认所有的线程都无需再次使用这个事件，并且所有线程都没被此事件阻塞时候才进行删除，否则删除之后线程需要再次使用此事件的话那也会发生错误。

代码清单 21‑4\ **(4)**\ ：删除事件对象，释放事件对象占用的内存资源。

代码清单 21‑4\ **(5)**\ ：删除成功返回RT_EOK。

事件的删除函数使用是很简单的，只需要传递进我们创建的事件对象句柄，其使用方法具体见代码清单 21‑5加粗部分。

代码清单 21‑5事件删除函数rt_event_delete()使用实例

1 /\* 定义事件控制块(句柄) \*/

2 static rt_event_t test_event = RT_NULL;

3 rt_err_t uwRet = RT_EOK;

**4 /\* 删除一个事件 \*/**

**5 uwRet = rt_event_delete(test_event);**

6 if (RT_EOK == uwRet)

7 rt_kprintf("事件删除成功！\n\n");

事件发送函数rt_event_send()
^^^^^^^^^^^^^^^^^^^^^

使用该函数接口时，通过参数set指定的事件标志来设定事件的标志位，然后遍历等待在event事件对象上的等待线程链表，判断是否有线程的事件激活要求与当前事件对象标志值匹配，如果有，则唤醒该线程。简单来说，就是设置我们自己定义的事件标志位为1，并且看看有没有线程在等待这个事件，有的话就唤醒它，其源码具体
见代码清单 21‑6。

代码清单 21‑6事件发送函数rt_event_send()源码

1 rt_err_t rt_event_send(rt_event_t event, **(1)**

2 rt_uint32_t set) **(2)**

3 {

4 struct rt_list_node \*n;

5 struct rt_thread \*thread;

6 register rt_ubase_t level;

7 register rt_base_t status;

8 rt_bool_t need_schedule;

9

10 /\* 事件对象检查 \*/

11 RT_ASSERT(event != RT_NULL); **(3)**

12 if (set == 0)

13 return -RT_ERROR;

14

15 need_schedule = RT_FALSE; **(4)**

16 RT_OBJECT_HOOK_CALL(rt_object_put_hook, (&(event->parent.parent)));

17

18 /\* 关中断 \*/

19 level = rt_hw_interrupt_disable();

20

21 /\* 设置事件 \*/

22 event->set \|= set; **(5)**

23

24 if (!rt_list_isempty(&event->parent.suspend_thread)) { **(6)**

25 /\* 搜索线程列表以恢复线程 \*/

26 n = event->parent.suspend_thread.next;

27 while (n != &(event->parent.suspend_thread)) {

28 /\* 找到要恢复的线程 \*/

29 thread = rt_list_entry(n, struct rt_thread, tlist); **(7)**

30

31 status = -RT_ERROR;

32 if (thread->event_info & RT_EVENT_FLAG_AND) { **(8)**

33 if ((thread->event_set & event->set)

34 == thread->event_set) { **(9)**

35 /\* 收到了一个AND \*/

36 status = RT_EOK; **(10)**

37 }

38 } else if (thread->event_info & RT_EVENT_FLAG_OR) { **(11)**

39 if (thread->event_set & event->set) {

40 /\* 保存收到的事件集 \*/

41 thread->event_set = thread->event_set & event->set; **(12)**

42

43 /\* 收到一个OR \*/

44 status = RT_EOK; **(13)**

45 }

46 }

47

48 /\* 将节点移动到下一个节点 \*/

49 n = n->next; **(14)**

50

51 /\* 条件满足，恢复线程 \*/

52 if (status == RT_EOK) { **(15)**

53 /\* 清除事件标志位 \*/

54 if (thread->event_info & RT_EVENT_FLAG_CLEAR) **(16)**

55 event->set &= ~thread->event_set;

56

57 /\* 恢复线程 \*/

58 rt_thread_resume(thread); **(17)**

59

60 /\* 需要进行线程调度 \*/

61 need_schedule = RT_TRUE; **(18)**

62 }

63 }

64 }

65

66 /\* 开中断 \*/

67 rt_hw_interrupt_enable(level);

68

69 /\* 发起一次线程调度 \*/

70 if (need_schedule == RT_TRUE)

71 rt_schedule(); **(19)**

72

73 return RT_EOK;

74 }

75 RTM_EXPORT(rt_event_send);

代码清单 21‑6\ **(1)**\ ：event：事件发送操作的事件句柄，由用户自己定义，并且需要在创建后使用。

代码清单 21‑6\ **(2)**\ ：set：设置事件集合中的具体事件，也就是设置set中的某些位。

代码清单 21‑6\ **(3)**\ ：检查事件句柄event是否有效，如果它是未定义或者未创建的事件句柄，那么是无法进行发送事件操作的。

代码清单 21‑6\ **(4)**\ ：need_schedule用于记录是否进行线程调度，默认不进行线程调度。

代码清单 21‑6\ **(5)**\ ：设置事件发生的标志位，利用‘|’操作即保证不干扰其他事件位又能同事对多个事件位一次性标记，即使是多次向线程发送同一事件(如果线程还未来得及读走)，也等效于只发送一次。

代码清单 21‑6\ **(6)**\ ：如果当前有线程因为等待某个事件进入阻塞态，则在阻塞列表中搜索线程，并且执行\ **(7)-(18)**\ ，

代码清单 21‑6\ **(7)**\ ：从等待的线程中获取对应的线程控制块。

代码清单 21‑6\ **(8)**\ ：如果线程等待事件的模式是RT_EVENT_FLAG_AND（逻辑与），那么需要等待的事件都发生时才动作。

代码清单 21‑6\ **(9)**\ ：判断线程等待的事件是否都发生了，如果事件激活要求与事件标志值匹配，则唤醒事件。

代码清单 21‑6\ **(10)**\ ：当等待的事件都发生的时候，进行标记status动作，表示事件已经等待到了。

代码清单 21‑6\ **(11)**\ ：如果线程等待事件的模式是RT_EVENT_FLAG_OR（逻辑或），那么线程等待的所有事件标记中只要有一个或多个事件发生了就表示事件已发生，可以唤醒线程。

代码清单 21‑6\ **(12)**\
：保存收到的事件，这个很重要，因为在接收事件函数的时候，这个值是要用来进行判断的，假设有一个线程等待接收3个事件，采用RT_EVENT_FLAG_OR（逻辑或）的方式等待接收，那么有其中一个事件发生，该线程就会解除阻塞，但是我们假如没保存收到的事件的话，我们怎么知道是哪个事件发生呢?

代码清单 21‑6\ **(13)**\ ：当等待的事件发生的时候，进行标记status动作，表示事件已经等待到了

代码清单 21‑6\ **(14)**\ ：将节点移动到下一个节点，因为这是搜索所有等待的线程。

代码清单 21‑6\ **(15)**\ ：当等待的事件发生的时候，条件满足，需要恢复线程。

代码清单 21‑6\ **(16)**\ ：如果在接收中设置了RT_EVENT_FLAG_CLEAR，那么在线程被唤醒的时候，系统会进行事件标志位的清除操作，防止一直响应事件。采用event->set &= ~thread->event_set操作仅仅是清除对应事件标志位，不影响其他事件标志位。

代码清单 21‑6\ **(17)**\ ：恢复阻塞的线程。

代码清单 21‑6\ **(18)**\ ：标记一下need_schedule表示需要进行线程调度。

代码清单 21‑6\ **(19)**\ ：发起一次线程调度。

举个例子，比如我们要记录一个事件的发生，这个事件在事件集合的位置是bit0，当它还未发生的时候，那么事件集合bit0的值也是0，当它发生的时候，我们往事件集合bit0中写入这个事件，也就是0x01，那这就表示事件已经发生了，为了便于理解，一般操作我们都是用宏定义来实现 #define EVENT
(0x01 << x)， “<< x”表示写入事件集合的bit x ，具体见代码清单 21‑7加粗部分。

代码清单 21‑7事件发送函数rt_event_send()实例

**1 #define KEY1_EVENT (0x01 << 0)//设置事件掩码的位0**

**2 #define KEY2_EVENT (0x01 << 1)//设置事件掩码的位1**

3 static void send_thread_entry(void\* parameter)

4 {

5 /\* 线程都是一个无限循环，不能返回 \*/

6 while (1) {//如果KEY2被单击

7 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {

8 rt_kprintf ( "KEY1被单击\n" );

**9 /\* 发送一个事件1 \*/**

**10 rt_event_send(test_event,KEY1_EVENT);**

11 }

12 //如果KEY2被单击

13 if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {

14 rt_kprintf ( "KEY2被单击\n" );

**15 /\* 发送一个事件2 \*/**

**16 rt_event_send(test_event,KEY2_EVENT);**

17 }

18 rt_thread_delay(20); //每20ms扫描一次

19 }

20 }

事件接受函数rt_event_recv()
^^^^^^^^^^^^^^^^^^^^^

既然标记了事件的发生，那么我怎么知道他到底有没有发生，这也是需要一个函数来获取事件发生的标记， RT-Thread提供了一个接收指定事件的函数——rt_event_recv()，通过这个函数，我们知道事件集合中的哪一位，哪一个事件发生了，我们可以通过
“逻辑与”、“逻辑或”等操作对感兴趣的事件进行接收，并且这个函数实现了等待超时机制，如果此刻该事件没有发生，那么线程可以进入阻塞态进行等待，等到事件发生了就会被唤醒，很有效的体现了操作系统的实时性，如果事件正确接收则返回RT_EOK，事件接收超时则返回-RT_ETIMEOUT，其他情况返回-
RT_ERROR，事件接受函数rt_event_recv()源码具体见代码清单 21‑8。

代码清单 21‑8事件接受函数rt_event_recv()源码

1 rt_err_t rt_event_recv(rt_event_t event, **(1)**

2 rt_uint32_t set, **(2)**

3 rt_uint8_t option, **(3)**

4 rt_int32_t timeout, **(4)**

5 rt_uint32_t \*recved) **(5)**

6 {

7 struct rt_thread \*thread;

8 register rt_ubase_t level;

9 register rt_base_t status;

10

11 RT_DEBUG_IN_THREAD_CONTEXT;

12

13 /\* 检查事件句柄 \*/

14 RT_ASSERT(event != RT_NULL); **(6)**

15 if (set == 0)

16 return -RT_ERROR;

17

18 /\* 初始化状态 \*/

19 status = -RT_ERROR;

20 /\* 获取当前线程 \*/

21 thread = rt_thread_self(); **(7)**

22 /\* 重置线程错误码 \*/

23 thread->error = RT_EOK;

24

25 RT_OBJECT_HOOK_CALL(rt_object_trytake_hook, (&(event->parent.parent)));

26

27 /\* 关中断 \*/

28 level = rt_hw_interrupt_disable();

29

30 /\* 检查事件接收选项&检查事件集合 \*/

31 if (option & RT_EVENT_FLAG_AND) { **(8)**

32 if ((event->set & set) == set)

33 status = RT_EOK;

34 } else if (option & RT_EVENT_FLAG_OR) { **(9)**

35 if (event->set & set)

36 status = RT_EOK;

37 } else {

38 /\* 应设置RT_EVENT_FLAG_AND或RT_EVENT_FLAG_OR \*/

39 RT_ASSERT(0); **(10)**

40 }

41

42 if (status == RT_EOK) {

43 /\* 返回接收的事件 \*/

44 if (recved)

45 \*recved = (event->set & set); **(11)**

46

47 /\* 接收事件清除 \*/

48 if (option & RT_EVENT_FLAG_CLEAR) **(12)**

49 event->set &= ~set;

50 } else if (timeout == 0) { **(13)**

51 /\* 不等待 \*/

52 thread->error = -RT_ETIMEOUT;

53 } else {

54 /\* 设置线程事件信息 \*/

55 thread->event_set = set; **(14)**

56 thread->event_info = option;

57

58 /\* 将线程添加到阻塞列表中 \*/

59 rt_ipc_list_suspend(&(event->parent.suspend_thread), **(15)**

60 thread,

61 event->parent.parent.flag);

62

63 /\* 如果有等待超时，则启动线程计时器 \*/

64 if (timeout > 0) {

65 /\* 重置线程超时时间并且启动定时器 \*/

66 rt_timer_control(&(thread->thread_timer), **(16)**

67 RT_TIMER_CTRL_SET_TIME,

68 &timeout);

69 rt_timer_start(&(thread->thread_timer)); **(17)**

70 }

71

72 /\* 开中断 \*/

73 rt_hw_interrupt_enable(level);

74

75 /\* 发起一次线程调度 \*/

76 rt_schedule(); **(18)**

77

78 if (thread->error != RT_EOK) {

79 /\* 返回错误代码 \*/

80 return thread->error; **(19)**

81 }

82

83 /\* 接收一个事件，失能中断 \*/

84 level = rt_hw_interrupt_disable();

85

86 /\* 返回接收到的事件 \*/

87 if (recved)

88 \*recved = thread->event_set; **(20)**

89 }

90

91 /\* 开中断 \*/

92 rt_hw_interrupt_enable(level);

93

94 RT_OBJECT_HOOK_CALL(rt_object_take_hook, (&(event->parent.parent)));

95

96 return thread->error; **(21)**

97 }

98 RTM_EXPORT(rt_event_recv);

代码清单 21‑8\ **(1)**\ ：event：事件发送操作的事件句柄，由用户自己定义，并且需要在创建事件后使用。

代码清单 21‑8\ **(2)**\ ：set：事件集合中的事件标志，在这里是指线程对哪些事件标志感兴趣。

代码清单 21‑8\ **(3)**\ ：option ：接收选项，有RT_EVENT_FLAG_AND、RT_EVENT_FLAG_OR，可以与RT_EVENT_FLAG_CLEAR通过“|”按位或操作符连接使用。

代码清单 21‑8\ **(4)**\ ：timeout是设置等待的超时时间。

代码清单 21‑8\ **(5)**\ ：recved用于保存接收到的事件标志结果，用户通过它的值判断是否成功接收到事件。

代码清单 21‑8\ **(6)**\ ：检查事件句柄event是否有效，如果它是未定义或者未创建的事件句柄，那么是无法接收事件的。

代码清单 21‑8\ **(7)**\ ：获取当前线程信息，即获取调用接收事件的线程。

代码清单 21‑8\ **(8)**\ ：如果指定的option接收选项是RT_EVENT_FLAG_AND，那么判断事件集合里面的信息与线程感兴趣的信息是否全部吻合，如果满足条件则标记接收成功。

代码清单 21‑8\ **(9)**\ ：如果指定的option接收选项是RT_EVENT_FLAG_OR，那么判断事件集合里面的信息与线程感兴趣的信息是否有吻合的部分（有其中一个满足即可），如果满足条件则标记接收成功。

代码清单 21‑8\ **(10)**\ ：其他情况，接收选项应设置RT_EVENT_FLAG_AND或RT_EVENT_FLAG_OR，他们无法同时使用，也不能不使用。

代码清单 21‑8\ **(11)**\ ：满足接收事件的条件，则返回接收的事件，读取recved即可知道接收到了哪个事件。

代码清单 21‑8\ **(12)**\ ：如果指定的option接收选项选择了RT_EVENT_FLAG_CLEAR，在接收完成的时候会清除对应的事件集合的标志位。

代码清单 21‑8\ **(13)**\ ：如果timeout= 0，那么接收不到事件就不等待，直接返回-RT_ETIMEOUT错误码。

代码清单 21‑8\ **(14)**\ ：timeout不为0，需要等待，那么需要配置线程接收事件的信息，event_set与event_info在线程控制块中有定义，event_set表示当前线程等待哪些感兴趣的事件，event_info表示事件接收选项option。

代码清单 21‑8\ **(15)**\ ：将等待的线程添加到阻塞列表中。

代码清单 21‑8\ **(16)**\ ：根据timeout的值重置线程超时时间。

代码清单 21‑8\ **(17)**\ ：启动定时器开始计时。

代码清单 21‑8\ **(18)**\ ：发起一次线程调度。

代码清单 21‑8\ **(19)**\ ：返回错误代码。

代码清单 21‑8\ **(20)**\ ：返回接收到的事件

代码清单 21‑8\ **(21)**\ ：返回接收成功结果。

当用户调用这个接口时，系统首先根据set参数和接收选项来判断它要接收的事件是否发生，如果已经发生，则根据参数option上是否设置有RT_EVENT_FLAG_CLEAR来决定是否清除事件的相应标志位，其中recved参数用于保存收到的事件；
如果事件没有发生，则把线程感兴趣的事件和接收选项填写到线程控制块中，然后把线程挂起在此事件对象的阻塞列表上，直到事件发生或等待时间超时，事件接受函数rt_event_recv()具体用法见代码清单 21‑9加粗部分。

代码清单 21‑9事件接受函数rt_event_recv()实例

1 static void receive_thread_entry(void\* parameter)

2 {

3 rt_uint32_t recved;

4 /\* 线程都是一个无限循环，不能返回 \*/

5 while (1) {

**6 /\* 等待接收事件标志 \*/**

**7 rt_event_recv(test_event, /\* 事件对象句柄 \*/**

**8 KEY1_EVENT|KEY2_EVENT, /\* 接收线程感兴趣的事件 \*/**

**9 RT_EVENT_FLAG_AND|RT_EVENT_FLAG_CLEAR,/\* 接收选项 \*/**

**10 RT_WAITING_FOREVER, /\* 指定超时事件,一直等 \*/**

**11 &recved); /\* 指向接收到的事件 \*/**

**12 if (recved == (KEY1_EVENT|KEY2_EVENT)) { /\* 如果接收完成并且正确 \*/**

13 rt_kprintf ( "Key1与Key2都按下\n");

14 LED1_TOGGLE; //LED1 反转

15 } else

16 rt_kprintf ( "事件错误！\n");

17 }

18 }

19

事件实验
~~~~

事件标志组实验是在RT-Thread中创建了两个线程，一个是发送事件线程，一个是接收事件线程，两个线程独立运行，发送事件线程通过检测按键的按下情况发送不同的事件，接收事件线程则接收这两个事件，并且判断两个事件是否都发生，如果是则输出相应信息，LED进行翻转。接收线程的等待时间是RT_WAITING_
FOREVER，一直在等待事件的发生，接收事件之后进行清除事件标记，具体见代码清单 21‑10加粗部分。

代码清单 21‑10事件实验

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief RT-Thread 3.0 + STM32 事件标志组

8 \\*

9 \* @attention

10 \*

11 \* 实验平台:基于野火STM32全系列（M3/4/7）开发板

12 \* 论坛 :http://www.firebbs.cn

13 \* 淘宝 :https://fire-stm32.taobao.com

14 \*

15 \\*

16 \*/

17

18 /\*

19 \\*

20 \* 包含的头文件

21 \\*

22 \*/

23 #include "board.h"

24 #include "rtthread.h"

25

26

27 /\*

28 \\*

29 \* 变量

30 \\*

31 \*/

32 /\* 定义线程控制块 \*/

**33 static rt_thread_t receive_thread = RT_NULL;**

**34 static rt_thread_t send_thread = RT_NULL;**

**35 /\* 定义事件控制块(句柄) \*/**

**36 static rt_event_t test_event = RT_NULL;**

37

38 /\* 全局变量声明 \/

39 /\*

40 \* 当我们在写应用程序的时候，可能需要用到一些全局变量。

41 \*/

**42 #define KEY1_EVENT (0x01 << 0)//设置事件掩码的位0**

**43 #define KEY2_EVENT (0x01 << 1)//设置事件掩码的位1**

44 /\*

45 \\*

46 \* 函数声明

47 \\*

48 \*/

49 static void receive_thread_entry(void\* parameter);

50 static void send_thread_entry(void\* parameter);

51

52 /\*

53 \\*

54 \* main 函数

55 \\*

56 \*/

57 /*\*

58 \* @brief 主函数

59 \* @param 无

60 \* @retval 无

61 \*/

62 int main(void)

63 {

64 /\*

65 \* 开发板硬件初始化，RTT系统初始化已经在main函数之前完成，

66 \* 即在component.c文件中的rtthread_startup()函数中完成了。

67 \* 所以在main函数中，只需要创建线程和启动线程即可。

68 \*/

69 rt_kprintf("这是一个[野火]- STM32全系列开发板-RTT事件标志组实验！\n");

**70 /\* 创建一个事件 \*/**

**71 test_event = rt_event_create("test_event",/\* 事件标志组名字 \*/**

**72 RT_IPC_FLAG_PRIO); /\* 事件模式 FIFO(0x00)*/**

**73 if (test_event != RT_NULL)**

**74 rt_kprintf("事件创建成功！\n\n");**

75

76 receive_thread = /\* 线程控制块指针 \*/

77 rt_thread_create( "receive", /\* 线程名字 \*/

78 receive_thread_entry, /\* 线程入口函数 \*/

79 RT_NULL, /\* 线程入口函数参数 \*/

80 512, /\* 线程栈大小 \*/

81 3, /\* 线程的优先级 \*/

82 20); /\* 线程时间片 \*/

83

84 /\* 启动线程，开启调度 \*/

85 if (receive_thread != RT_NULL)

86 rt_thread_startup(receive_thread);

87 else

88 return -1;

89

90 send_thread = /\* 线程控制块指针 \*/

91 rt_thread_create( "send", /\* 线程名字 \*/

92 send_thread_entry, /\* 线程入口函数 \*/

93 RT_NULL, /\* 线程入口函数参数 \*/

94 512, /\* 线程栈大小 \*/

95 2, /\* 线程的优先级 \*/

96 20); /\* 线程时间片 \*/

97

98 /\* 启动线程，开启调度 \*/

99 if (send_thread != RT_NULL)

100 rt_thread_startup(send_thread);

101 else

102 return -1;

103 }

104

105 /\*

106 \\*

107 \* 线程定义

108 \\*

109 \*/

110

**111 static void receive_thread_entry(void\* parameter)**

**112 {**

**113 rt_uint32_t recved;**

**114 /\* 线程都是一个无限循环，不能返回 \*/**

**115 while (1) {**

**116 /\* 等待接收事件标志 \*/**

**117 rt_event_recv(test_event, /\* 事件对象句柄 \*/**

**118 KEY1_EVENT|KEY2_EVENT,/\* 接收线程感兴趣的事件 \*/**

**119 RT_EVENT_FLAG_AND|RT_EVENT_FLAG_CLEAR,/\* 接收选项 \*/**

**120 RT_WAITING_FOREVER,/\* 指定超时事件,一直等 \*/**

**121 &recved); /\* 指向接收到的事件 \*/**

**122 if (recved == (KEY1_EVENT|KEY2_EVENT)) { /\* 如果接收完成并且正确 \*/**

**123 rt_kprintf ( "Key1与Key2都按下\n");**

**124 LED1_TOGGLE; //LED1 反转**

**125 } else**

**126 rt_kprintf ( "事件错误！\n");**

**127 }**

**128 }**

129

**130 static void send_thread_entry(void\* parameter)**

**131 {**

**132 /\* 线程都是一个无限循环，不能返回 \*/**

**133 while (1) { //如果KEY2被单击**

**134 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {**

**135 rt_kprintf ( "KEY1被单击\n" );**

**136 /\* 发送一个事件1 \*/**

**137 rt_event_send(test_event,KEY1_EVENT);**

**138 }**

**139 //如果KEY2被单击**

**140 if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {**

**141 rt_kprintf ( "KEY2被单击\n" );**

**142 /\* 发送一个事件2 \*/**

**143 rt_event_send(test_event,KEY2_EVENT);**

**144 }**

**145 rt_thread_delay(20); //每20ms扫描一次**

**146 }**

**147 }**

148 /END OF FILE/

实验现象
~~~~

程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到rt_kprintf的打印信息，按下开发版的
K1按键发送事件1，按下K2按键发送事件2；我们按下K1与K2试试，在串口调试助手中可以看到运行结果，并且当时间1与事件2都发生的时候，开发板的LED会进行翻转，具体见图 21‑3。

|event004|

图 21‑3事件标志组实验现象

.. |event002| image:: media/event/event002.png
   :width: 5.76806in
   :height: 0.99135in
.. |event003| image:: media/event/event003.png
   :width: 5.76806in
   :height: 5.10332in
.. |event004| image:: media/event/event004.png
   :width: 5.76806in
   :height: 2.84824in
