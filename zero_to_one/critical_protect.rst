.. vim: syntax=rst

临界段的保护
===============

什么是临界段
~~~~~~~~~~~~~

临界段用一句话概括就是一段在执行的时候不能被中断的代码段。在RT-Thread里面，这个临界段最常出现的就是对全局变量的操作，全局变量就好像是一个枪把子，谁都可以对他开枪，但是我开枪的时候，你就不能开枪，否则就不知道是谁命中了靶子。

那么什么情况下临界段会被打断？一个是系统调度，还有一个就是外部中断。在RT-Thread，系统调度，最终也是产生PendSV中断，
在PendSV Handler里面实现线程的切换，所以还是可以归结为中断。既然这样，RT-Thread对临界段的保护就处理的很干脆了，
直接把中断全部关了，NMIFAULT 和硬FAULT除外。

Cortex-M内核快速关中断指令
~~~~~~~~~~~~~~~~~~~~~~~~~~~

为了快速地开关中断， Cortex-M内核 专门设置了一条 CPS 指令，有 4 种用法，具体见 代码清单:临界段的保护-1_。

.. literalinclude::
    :caption:  代码清单:临界段的保护-1 CPS指令用法
    :name: 代码清单:临界段的保护-1
    :linenos:

    CPSID I ;PRIMASK=1 ;关中断
    CPSIE I ;PRIMASK=0 ;开中断
    CPSID F ;FAULTMASK=1;关异常
    CPSIE F ;FAULTMASK=0;开异常

-   代码清单:临界段的保护-1_ 中 PRIMASK和FAULTMAST是Cortex-M内核 里面三个中断屏蔽寄存器中的两个，还有一个是BASEPRI，
    有关这三个寄存器的详细用法见下表。

.. list-table:: Cortex-M内核中断屏蔽寄存器组描述
    :widths: auto

    * - 名字
      - 功能描述
    * - PRIMASK
      - 这是个只有单一比特的寄存器。 在它被置 1 后，就关掉所有可屏蔽的异常，
        只剩下 NMI 和硬 FAULT可以响应。它的缺省值是 0，表示没有关中断。
    * - FAULTMASK
      - 这是个只有 1 个位的寄存器。当它置 1 时，只有 NMI 才能响应，所有其它的异常，
        甚至是硬 FAULT，也通通闭嘴。它的缺省值也是 0，表示没有关异常。
    * - BASEPRI
      - 这个寄存器最多有 9 位（ 由表达优先级的位数决定）。它定义了被屏蔽优先级的阈值。
        当它被设成某个值后，所有优先级号大于等于此值的中断都被关（优先级号越大，优先级越低）。
        但若被设成 0，则不关闭任何中断， 0 也是缺省值。


关中断
~~~~~~~

RT-Thread关中断的函数在contex_rvds.s中定义，在rthw.h中声明，具体实现见 代码清单:临界段的保护-2_。

.. literalinclude::
    :caption: 代码清单:临界段的保护-2 关中断
    :name: 代码清单:临界段的保护-2
    :linenos:

    ;/*
    ; * rt_base_t rt_hw_interrupt_disable();
    ; */
    rt_hw_interrupt_disable PROC;     (1)
    EXPORT rt_hw_interrupt_disable;   (2)
    MRS r0, PRIMASK;                  (3)
    CPSID I;                          (4)
    BX LR;                            (5)
    ENDP;                             (6)


-   代码清单:临界段的保护-2_ **(1)** ：关键字PROC表示汇编子程序开始。

-   代码清单:临界段的保护-2_ **(2)** ：使用EXPORT关键字导出标号rt_hw_interrupt_disable，使其具有全局属性，在外部
    头文件声明后（在rthw.h中声明），就可以在C文件中调用。

-   代码清单:临界段的保护-2_ **(3)** ：通过MRS指令将特殊寄存器PRIMASK寄存器的值存储到通用寄存器r0。当在C中调用
    汇编的子程序返回时，会将r0作为函数的返回值。所以在C中调用rt_hw_interrupt_disable()的时候，需要事
    先声明一个变量用来存储rt_hw_interrupt_disable()的返回值，即r0寄存器的值，也就是PRIMASK的值。

-   代码清单:临界段的保护-2_ **(4)** ：关闭中断，即使用CPS指令将PRIMASK寄存器的值置1。在这里，我敢肯定，一定会有
    人有这样一个疑问：关中断，不就是直接使用 CPSID I 指令就行了嘛，为什么还要第三步，即在执行CPSIDI指令
    前，要先把PRIMASK的值保存起来？这个疑问接下来在“临界段代码的应用”这个小结揭晓。

-   代码清单:临界段的保护-2_ **(5)** ：子程序返回。

-   代码清单:临界段的保护-2_ **(6)** ：ENDP表示汇编子程序结束，与PROC成对使用。

开中断
~~~~~~~~~

RT-Thread开中断的函数在contex_rvds.s中定义，在rthw.h中声明，具体实现见 代码清单:临界段的保护-3_。

.. literalinclude::
    :caption:  代码清单:临界段的保护-3 开中断
    :name: 代码清单:临界段的保护-3
    :linenos:

    ;/*
    ; * void rt_hw_interrupt_enable(rt_base_t level);
    ; */
    rt_hw_interrupt_enable PROC;     (1)
    EXPORT rt_hw_interrupt_enable;   (2)
    MSR PRIMASK, r0                  (3)
    BX LR;                           (4)
    ENDP;                            (5)


-   代码清单:临界段的保护-3_ **(1)**\ ：关键字PROC表示汇编子程序开始。

-   代码清单:临界段的保护-3_ **(2)**\ ：使用EXPORT关键字导出标号rt_hw_interrupt_enable，使其具有全局熟悉，
    在外部头文件声明后（在rthw.h中声明），就可以在C文件中调用。

-   代码清单:临界段的保护-3_ **(3)**\ ：通过MSR指令将通用寄存器r0的值存储到特殊寄存器PRIMASK。当在C中调用汇编的
    子程序返回时，会将第一个形参传入到通用寄存器r0。所以在C中调用rt_hw_interrupt_enable()的时候，需要
    传入一个形参，该形参是进入临界段之前保存的PRIMASK的值。这个时候又有人会问，开中断，不就是
    使用CPSIE I指令就行了嘛，为啥跟我等凡人想的不一样？其中奥妙将在接下来“临界段代码的应用”这个小结揭晓

-   代码清单:临界段的保护-3_ **(4)**\ ：子程序返回。

-   代码清单:临界段的保护-3_ **(5)**\ ：ENDP表示汇编子程序结束，与PROC成对使用。

临界段代码的应用
~~~~~~~~~~~~~~~~~~~

在进入临界段之前，我们会先把中断关闭，退出临界段时再把中断打开。而且Cortex-M内核设置了快速关中断
的CPS指令，那么按照我们的第一思维，开关中断的函数的实现和临界段代码的保护应该是像 代码清单:临界段的保护-4_ 那样的。

.. literalinclude::
    :caption: 代码清单:临界段的保护-4 开关中断的函数的实现和临界段代码的保护
    :name: 代码清单:临界段的保护-4
    :linenos:

    ; 开关中断函数的实现
    ;/*
    ; * void rt_hw_interrupt_disable();
    ; */
    rt_hw_interrupt_disable    PROC
        EXPORT  rt_hw_interrupt_disable
        CPSID   I                                    (1)
        BX      LR
        ENDP

    ;/*
    ; * void rt_hw_interrupt_enable(void);
    ; */
    rt_hw_interrupt_enable    PROC
        EXPORT  rt_hw_interrupt_enable
        CPSIE   I                                    (2)
        BX      LR
        ENDP

    PRIMASK = 0;                      /* PRIMASK初始值为0,表示没有关中断 */ (3)

    /* 临界段代码保护 */
    {
        /* 临界段开始 */
        rt_hw_interrupt_disable();     /* 关中断,PRIMASK = 1 */            (4)
        {
            /* 执行临界段代码，不可中断 */                                   (5)
        }
        /* 临界段结束 */
        rt_hw_interrupt_enable();      /* 开中断,PRIMASK = 0 */            (6)
    }


-   代码清单:临界段的保护-4_ **(1)**\ ：关中断直接使用了CPSID I，没有跟 代码清单:临界段的保护-2_ 一样事先将PRIMASK的值保存在r0中。

-   代码清单:临界段的保护-4_  **(2)**\ ：开中断直接使用了CPSIE I，而不是像 代码清单:临界段的保护-3_ 那样从传进来的形参来恢复PRIMASK的值。

-   代码清单:临界段的保护-4_ **(3)**\ ：假设PRIMASK初始值为0，表示没有关中断。

-   代码清单:临界段的保护-4_ **(4)**\ ：临界段开始，调用关中断函数rt_hw_interrupt_disable()，此时PRIMASK的值等于1，
    确实中断已经关闭。

-   代码清单:临界段的保护-4_ **(5)**\ ：执行临界段代码，不可中断。

-   代码清单:临界段的保护-4_ **(6)**\ ：临界段结束，调用开中断函数rt_hw_interrupt_enable()，此时PRIMASK的值等于0，
    确实中断已经开启。

乍一看，代码清单:临界段的保护-4_ 的这种实现开关中断的方法确实有效，没有什么错误，但是我们忽略了一种情况，就是当临
界段是出现嵌套的时候，这种开关中断的方法就不行了，具体怎么不行具体见 代码清单:临界段的保护-5_。

.. literalinclude::
    :caption: 代码清单:临界段的保护-5 开关中断的函数的实现和嵌套临界段代码的保护（有错误，只为讲解）
    :name: 代码清单:临界段的保护-5
    :linenos:

    ; 开关中断函数的实现
    ;/*
    ; * void rt_hw_interrupt_disable();
    ; */
    rt_hw_interrupt_disable    PROC
        EXPORT  rt_hw_interrupt_disable
        CPSID   I
        BX      LR
        ENDP

    ;/*
    ; * void rt_hw_interrupt_enable(void);
    ; */
    rt_hw_interrupt_enable    PROC
        EXPORT  rt_hw_interrupt_enable
        CPSIE   I
        BX      LR
        ENDP

    PRIMASK = 0;                            /* PRIMASK初始值为0,表示没有关中断 */
    /* 临界段代码 */
    {
        /* 临界段1开始 */
        rt_hw_interrupt_disable();           /* 关中断,PRIMASK = 1 */
        {
            /* 临界段2 */
            rt_hw_interrupt_disable();       /* 关中断,PRIMASK = 1 */
            {

            }
            rt_hw_interrupt_enable();        /* 开中断,PRIMASK = 0 */  (注意)
        }
        /* 临界段1结束 */
        rt_hw_interrupt_enable();            /* 开中断,PRIMASK = 0 */
    }

-   代码清单:临界段的保护-5_ **(注意)** ：当临界段出现嵌套的时候，这里以一重嵌套为例。临界段1开始和结束的
    时候PRIMASK分别等于1和0，表示关闭中断和开启中断，这是没有问题的。临界段2开始的时候，PRIMASK等于1，
    表示关闭中断，这是没有问题的，问题出现在临界段2结束的时候，PRIMASK的值
    等于0，如果单纯对于临界段2来说，这也是没有问题的，因为临界段2已经结束，可是临界段2是嵌套在临界段1中
    虽然临界段2已经结束，但是临界段1还没有结束，中断是不能开启的，如果此时有外部中断来临，那么临界段1就
    会被中断，违背了我们的初衷，那应该怎么办？正确的做法具体见。

.. literalinclude::
    :caption: 代码清单:临界段的保护-6 开关中断的函数的实现和嵌套临界段代码的保护（正确）
    :name: 代码清单:临界段的保护-6
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

    PRIMASK = 0;        /* PRIMASK初始值为0,表示没有关中断 */                   (1)
    rt_base_t level1;                                                         (2)
    rt_base_t level2;

    /* 临界段代码 */
    {
        /* 临界段1开始 */
        level1 = rt_hw_interrupt_disable();   /* 关中断,level1=0,PRIMASK=1 */  (3)
        {
            /* 临界段2 */
            level2 = rt_hw_interrupt_disable(); /* 关中断,level2=1,PRIMASK=1 */(4)
            {

            }
            rt_hw_interrupt_enable(level2);     /* 开中断,level2=1,PRIMASK=1 */(5)
        }
        /* 临界段1结束 */
        rt_hw_interrupt_enable(level1);      /* 开中断,level1=0,PRIMASK=0 */   (6)
    }



-   代码清单:临界段的保护-6_ **(1)**\ ：假设PRIMASK初始值为0,表示没有关中断。

-   代码清单:临界段的保护-6_ **(2)**\ ：定义两个变量，留着后面用。

-   代码清单:临界段的保护-6_ **(3)**\ ：临界段1开始，调用关中断函数rt_hw_interrupt_disable()，
    rt_hw_interrupt_disable()函数先将PRIMASK的值存储在通用寄存器r0，一开始我们假设PRIMASK的值等于0，
    所以此时r0的值即为0。然后执行汇编指令CPSID I关闭中断，即设置PRIMASK等于1，在返回的时候r0当做函数
    的返回值存储在level1，所以level1等于r0等于0。

-   代码清单:临界段的保护-6_ **(4)**\
    ：临界段2开始，调用关中断函数rt_hw_interrupt_disable()，rt_hw_interrupt_disable()函数先将PRIMASK
    的值存储在通用寄存器r0，临界段1开始的时候我们关闭了中断，即设置PRIMASK等于1，所以此时r0的值等于1。
    然后执行汇编指令 CPSIDI关闭中断，即设置PRIMASK等于1，在返回的时候r0当做函数的返回值存储在level2，
    所以level2等于r0等于1。

-   代码清单:临界段的保护-6_ **(5)**\ ：临界段2结束，调用开中断函数rt_hw_interrupt_enable(level2)，level2作为函数
    的形参传入到通用寄存器r0，然后执行汇编指令 MSR r0, PRIMASK 恢复PRIMASK的值。此时
    PRIAMSK = r0 = level2 =1。关键点来了，为什么临界段2结束了，PRIMASK还是等于1，按道理应该是等于0。
    因为此时临界段2是嵌套在临界段1中的，还是没有完全离开临界段的范畴，所以不能把中断打开，如果临界段是
    没有嵌套的，使用当前的开关中断的方法的话，那么PRIMASK确实是等于1，具体举例见 代码清单:临界段的保护-7_。

.. literalinclude::
    :caption: 代码清单:临界段的保护-7 开关中断的函数的实现和一重临界段代码的保护（正确）
    :name: 代码清单:临界段的保护-7
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
    PRIMASK = 0;                           /* PRIMASK初始值为0,表示没有关中断 */
    rt_base_t level1;

    /* 临界段代码 */
    {
    /* 临界段开始 */
    level1 = rt_hw_interrupt_disable();/* 关中断,level1=0,PRIMASK=1 */
    {

    }
    /* 临界段结束 */
    rt_hw_interrupt_enable(level1);    /* 开中断,level1=0,PRIMASK=0 */(注意点)
    }

-   代码清单:临界段的保护-7_ **(6)** ：临界段1结束，PRIMASK等于0，开启中断，与进入临界段1遥相呼应。

实验现象
~~~~~~~~~

本章没有实验，充分理解本章内容即可，这么简单，其实也没啥好理解的。
