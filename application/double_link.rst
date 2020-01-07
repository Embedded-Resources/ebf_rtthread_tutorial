.. vim: syntax=rst

双向链表
==========

双向链表的基本概念
~~~~~~~~~~~~~~~~~~~~~

双向链表也叫双链表，是链表的一种，是在操作系统中常用的数据结构，它的每个数据结点中都有两个指针，分别指向
直接后继和直接前驱，其头指针head是唯一确定的。所以，从双向链表中的任意一个结点开始，都可以很方便地访问它
的前驱结点和后继结点，这种数据结构形式使得双向链表在查找时更加方便，特别是大量数据的遍历。由于双向链表具
有对称性，能方便地完成各种插入、删除等操作，但需要注意前后方向的操作。

双向链表的函数接口讲解
~~~~~~~~~~~~~~~~~~~~~~

RT-Thread为我们提供了很多操作链表的函数，如链表的初始化、添加节点、删除节点等。

RT-Thread的链表节点结构体中只有两个指针，一个是指向上一个节点的指针，另一个是指向下一个节点的指针，具体见

.. code-block:: c
    :caption: 代码清单:双向链表-1链表节点结构体
    :name: 代码清单:双向链表-1
    :linenos:

    struct rt_list_node {
        struct rt_list_node *next;  /**< 指向下一个节点的指针. */
        struct rt_list_node *prev;  /**< 指向上一个节点的指针. */
    };
    typedef struct rt_list_node rt_list_t;


链表初始化函数rt_list_init()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在使用链表的时候必须要进行初始化，将链表的指针指向自己，为以后添加节点做准备 ，链表的数据结构也是需
要内存空间的，所以也需要进行内存的申请，链表初始化函数rt_list_init()的源码具体见 代码清单:双向链表-2_，
其结果具体见 链表初始化示意图_。

.. code-block:: c
    :caption: 代码清单:双向链表-2链表初始化函数rt_list_init()源码
    :name: 代码清单:双向链表-2
    :linenos:

    /**
    * @brief 初始化一个链表
    *
    * @param
    */
    rt_inline void rt_list_init(rt_list_t *l)
    {
        l->next = l->prev = l;
    }


.. image:: media/double_link/double002.png
    :align: center
    :name: 链表初始化示意图
    :alt: 链表初始化示意图


其初始化完成后可以检查一下链表初始化是否成功，判断链表是不是空的就行了，因为初始化完成的时
候，链表肯定是空的，注意，在初始化链表的时候其实链表就是链表头，需要申请内存，链表的初始化
实例具体见 代码清单:双向链表-3_。

.. code-block:: c
    :caption: 代码清单:双向链表-3链表初始化函数rt_list_init()实例
    :name: 代码清单:双向链表-3
    :linenos:

    head = rt_malloc(sizeof(rt_list_t));/* 申请动态内存 */
    if (RT_NULL == head) /* 没有申请成功 */
        rt_kprintf("动态内存申请失败！\n");
    else
        rt_kprintf("动态内存申请成功，头结点地址为%d！\n",head);

    rt_kprintf("\n双向链表初始化中......\n");
    rt_list_init(head);
    if (rt_list_isempty(head))
        rt_kprintf("双向链表初始化成功!\n\n");

向链表中插入节点
^^^^^^^^^^^^^^^^

向链表指定节点后面插入节点rt_list_insert_after()
'''''''''''''''''''''''''''''''''''''''''''''''''''''

插入节点需要先申请节点大小的内存，然后根据插入的位置（在某个节点（rt_list_t *l）后面）进行插入操作，具体见 代码清单:双向链表-4_。

.. code-block:: c
    :caption: 代码清单:双向链表-4向链表指定节点后面插入节点rt_list_insert_after()源码
    :name: 代码清单:双向链表-4
    :linenos:

    rt_inline void rt_list_insert_after(rt_list_t *l, rt_list_t *n)
    {
        l->next->prev = n;			(1)
        n->next = l->next;			(2)

        l->next = n;			(3)
        n->prev = l;			(4)
    }


这是数据结构的基本使用方法，其过程具体见 插入节点的过程示意图_。

.. image:: media/double_link/double003.png
    :align: center
    :name: 插入节点的过程示意图
    :alt: 插入节点的过程示意图


向链表指定节点前面插入节点rt_list_insert_before()
''''''''''''''''''''''''''''''''''''''''''''''''''''''''

插入节点需要先申请节点大小的内存，然后根据插入的位置（在某个节点（rt_list_t \*l）前面）进行插入操作，具体见 代码清单:双向链表-5_。

.. code-block:: c
    :caption: 代码清单:双向链表-5向链表指定节点前面插入节点rt_list_insert_before()源码
    :name: 代码清单:双向链表-5
    :linenos:

    rt_inline void rt_list_insert_before(rt_list_t *l, rt_list_t *n)
    {
        l->prev->next = n;			(1)
        n->prev = l->prev;			(2)

        l->prev = n;			(3)
        n->next = l;			(4)
    }


这是数据结构的基本使用方法，其过程具体见 插入节点的过程示意图1_。

.. image:: media/double_link/double004.png
    :align: center
    :name: 插入节点的过程示意图1
    :alt: 插入节点的过程示意图


插入节点的实例也很简单，但是要注意的是要申请内存，具体见 代码清单:双向链表-6_ 高亮部分。

.. code-block:: c
    :caption: 代码清单:双向链表-6向链表插入节点函数实例
    :emphasize-lines: 5-9,18,20
    :name: 代码清单:双向链表-6
    :linenos:

    /* 插入节点：顺序插入与从末尾插入 */

    rt_kprintf("添加节点和尾节点添加......\n");

    /* 动态申请第一个结点的内存 */
    node1 = rt_malloc(sizeof(rt_list_t));

    /* 动态申请第二个结点的内存 */
    node2 = rt_malloc(sizeof(rt_list_t));

    rt_kprintf("添加第一个节点与第二个节点.....\n");

    /* 因为这是在某个节点后面添加一个节点函数
    为后面的rt_list_insert_before（某个节点之前）
    添加节点做铺垫,两个函数添加完之后的顺序是
    head -> node1 -> node2 */

    rt_list_insert_after(head,node2);

    rt_list_insert_before(node2,node1);

    if ((node1->prev == head) && (node2->prev == node1))
        rt_kprintf("添加节点成功!\n\n");
    else
        rt_kprintf("添加节点失败!\n\n");


从链表删除节点函数rt_list_remove()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

删除节点与添加节点一样，其实删除节点更简单，只需要知道删除哪个节点即可，
把该节点前后的节点链接起来，那它就删除了，然后该节点的指针指向节点本身即
可，不过要注意的是也要讲该节点的内存释放掉，因为该节点是动态分配内存的，
否则会导致内存泄漏，源码具体见 代码清单:双向链表-7_，其实现过程具体见 节点删除过程示意图_。

.. code-block:: c
    :caption: 代码清单:双向链表-7从链表删除节点函数rt_list_remove()源码
    :name: 代码清单:双向链表-7
    :linenos:

    rt_inline void rt_list_remove(rt_list_t *n)
    {
        n->next->prev = n->prev;		(1)
        n->prev->next = n->next;		(2)

        n->next = n->prev = n;		(3)
    }


.. image:: media/double_link/double005.png
    :align: center
    :name: 节点删除过程示意图
    :alt: 节点删除过程示意图


删除节点的用法也是简单，具体见

.. code-block:: c
    :caption: 代码清单:双向链表-8从链表删除节点函数rt_list_remove()实例
    :name: 代码清单:双向链表-8
    :linenos:

    rt_kprintf("删除节点......\n");	/* 删除已有节点 */
    rt_list_remove(node1);
    rt_free(node1);/* 释放第一个节点的内存 */
    if (node2->prev == head)
        rt_kprintf("删除节点成功\n\n");


双向链表的实验
~~~~~~~~~~~~~~~~~~~~~

双向链表实验实现如下功能：

-  调用rt_list_init初始双向链表。

-  调用rt_list_insert_after与rt_list_insert\_ before向链表中增加节点。

-  调用rt_list_remove删除指定节点。

-  调用rt_list_isempty判断链表是否为空。

-  测试操作是否成功。

删除节点的时候要注意释放掉内存，具体见 代码清单:双向链表-9_ 高亮部分。

.. code-block:: c
    :caption: 代码清单:双向链表-9双向链表实验
    :emphasize-lines: 104-148
    :name: 代码清单:双向链表-9
    :linenos:

    /**
    *********************************************************************
    * @file    main.c
    * @author  fire
    * @version V1.0
    * @date    2018-xx-xx
    * @brief   RT-Thread 3.0 + STM32 双向链表实验
    *********************************************************************
    * @attention
    *
    * 实验平台:基于野火STM32全系列（M3/4/7）开发板
    * 论坛    :http://www.firebbs.cn
    * 淘宝    :https://fire-stm32.taobao.com
    *
    **********************************************************************
    */

    /*
    *************************************************************************
    *                             包含的头文件
    *************************************************************************
    */
    #include "board.h"
    #include "rtthread.h"


    /*
    ******************************************************************
    *                               变量
    ******************************************************************
    */
    /* 定义线程控制块 */
    static rt_thread_t test1_thread = RT_NULL;
    static rt_thread_t test2_thread = RT_NULL;

    /************************* 全局变量声明 ****************************/
    /*
    * 当我们在写应用程序的时候，可能需要用到一些全局变量。
    */

    /*
    *************************************************************************
    *                             函数声明
    *************************************************************************
    */
    static void test1_thread_entry(void* parameter);
    static void test2_thread_entry(void* parameter);

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
        rt_kprintf("这是一个[野火]- STM32全系列开发板-RTT双向链表操作实验！\n");

        test1_thread =                          /* 线程控制块指针 */
            rt_thread_create( "test1",              /* 线程名字 */
                            test1_thread_entry,   /* 线程入口函数 */
                            RT_NULL,             /* 线程入口函数参数 */
                            512,                 /* 线程栈大小 */
                            2,                   /* 线程的优先级 */
                            20);                 /* 线程时间片 */

        /* 启动线程，开启调度 */
        if (test1_thread != RT_NULL)
            rt_thread_startup(test1_thread);
        else
            return -1;

        test2_thread =                          /* 线程控制块指针 */
            rt_thread_create( "test2",              /* 线程名字 */
                            test2_thread_entry,   /* 线程入口函数 */
                            RT_NULL,             /* 线程入口函数参数 */
                            512,                 /* 线程栈大小 */
                            3,                   /* 线程的优先级 */
                            20);                 /* 线程时间片 */

        /* 启动线程，开启调度 */
        if (test2_thread != RT_NULL)
            rt_thread_startup(test2_thread);
        else
            return -1;
    }

    /*
    *************************************************************************
    *                             线程定义
    *************************************************************************
    */

    static void test1_thread_entry(void* parameter)
    {
        rt_list_t *head;  /* 定义一个双向链表的头节点 */
        rt_list_t *node1;  /* 定义一个双向链表的头节点 */
        rt_list_t *node2;  /* 定义一个双向链表的头节点 */

        head = rt_malloc(sizeof(rt_list_t));/* 申请动态内存 */
        if (RT_NULL == head) /* 没有申请成功 */
            rt_kprintf("动态内存申请失败！\n");
        else
            rt_kprintf("动态内存申请成功，头结点地址为%d！\n",head);

        rt_kprintf("\n双向链表初始化中......\n");
        rt_list_init(head);
        if (rt_list_isempty(head))
            rt_kprintf("双向链表初始化成功!\n\n");

        /* 插入节点：顺序插入与从末尾插入 */
        rt_kprintf("添加节点和尾节点添加......\n");

        /* 动态申请第一个结点的内存 */
        node1 = rt_malloc(sizeof(rt_list_t));

        /* 动态申请第二个结点的内存 */
        node2 = rt_malloc(sizeof(rt_list_t));

        rt_kprintf("添加第一个节点与第二个节点.....\n");

        /* 因为这是在某个节点后面添加一个节点函数
        为后面的rt_list_insert_before（某个节点之前）
        添加节点做铺垫,两个函数添加完之后的顺序是
        head -> node1 -> node2 */

        rt_list_insert_after(head,node2);

        rt_list_insert_before(node2,node1);

        if ((node1->prev == head) && (node2->prev == node1))
            rt_kprintf("添加节点成功!\n\n");
        else
            rt_kprintf("添加节点失败!\n\n");

        rt_kprintf("删除节点......\n");	/* 删除已有节点 */
        rt_list_remove(node1);
        rt_free(node1);/* 释放第一个节点的内存 */
        if (node2->prev == head)
            rt_kprintf("删除节点成功\n\n");

        /* 线程都是一个无限循环，不能返回 */
        while (1) {
            LED1_TOGGLE;
            rt_thread_delay(500);     //每500ms扫描一次
        }
    }

    static void test2_thread_entry(void* parameter)
    {

        /* 线程都是一个无限循环，不能返回 */
        while (1) {
            rt_kprintf("线程运行中!\n");
            LED2_TOGGLE;
            rt_thread_delay(1000);     //每1000ms扫描一次
        }
    }
    /*****************************END OF FILE****************************/



双向链表的实验现象
~~~~~~~~~~~~~~~~~~~~~~~~~~~

打开配RT-Thread套例程的第13个例程“13，Fire-F103霸道-RT-Thread-双向链表”，
本次实验以野火STM32F103霸道开发板来进行实验，其它开发板都有对应的例程，都是一
样的。点击编译，然后下载到开发板上，打开我们野火的串口调试助手，它里面输出了
信息表明双向链表的操作已经全部完成，开发板上的LED也在闪烁，具体见 双向链表的实验现象_。

.. image:: media/double_link/double006.png
    :align: center
    :name: 双向链表的实验现象
    :alt: 双向链表的实验现象

