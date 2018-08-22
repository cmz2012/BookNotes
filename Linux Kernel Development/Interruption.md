# 5. 中断和中断处理程序
linux内核要对连接到计算机上的所有硬件设备进行管理，而想要管理这些设备，首先要能和它们互相通信。众所周知，处理器的速度跟外围硬件设备的速度往往不在一个数量级上，如果内核采取让处理器向硬件发出一个请求，然后专门等待回应的办法，效率太低。因此，为了提高内核效率，需要一种机制，让硬件在需要的时候向内核发出信号，这就是中断机制。

## 5.1 中断
中断本质上是一种特殊的电信号，有硬件设备发向处理器。处理器收到中断后，会马上向操作系统反映此信号的到来，然后就由OS负责处理这些新到来的数据。硬件设备生成中断的时候并不考虑与处理器的时钟同步，也就是说中断可以随时到来，内核随时可能因为新到来的中断而被打断。

异常与中断不停，它在产生时必须考虑与处理器时钟同步。实际上，异常也常常称为同步中断。在处理器执行到由于编程失误而导致的错误指令的时候，或者是在执行期间出现特殊情况(缺页)必须靠内核来处理的时候，处理器就会产生一个异常。

## 5.2 中断处理程序
在响应一个特定中断的时候，内核会执行一个函数，该函数叫做中断处理函数或者中断服务例程。中断处理程序与特定中断相关，一个设备可以产生很多不同的中断，对应多个中断处理程序。一个设备的中断处理程序是它设备驱动程序的一部分--设备驱动程序是用于对设备进行管理的内核代码。

在linux中，中断处理程序看起来就是普普通通的C函数。与其他内核函数的真正的区别在于：中断处理程序是被内核调用来响应中断的，运行于中断上下文中。

中断随时可能发生，必须保证中断处理程序能够快速执行，这样才能保证尽可能快地恢复中断代码的执行。对于需要完成大量工作且又要能够快速完成的中断处理程序，一般把中断处理切为两个部分：中断处理程序是上半部，只做有严格时限的工作，例如对接收的中断进行应答或者复位硬件，这些工作都是在所有中断被屏蔽的情况下完成的。能够被允许稍后完成的工作会推迟到下半部。通常情况下，下半部会在中断处理程序返回时立即执行。

## 5.2.1 中断上下文
当执行一个中断处理程序或下半部时，内核处于中断上下文中。

先回忆一下进程上下文，进程上下文是一种内核所处的操作模式，此时内核代表进程执行。在进程上下文中，可以通过current宏关联当前进程，此时可以睡眠也可以被抢占。

中断上下文与进程没什么瓜葛。与current宏也没关系。因为没有进程的背景，所以中断上下文不可以睡眠，否则怎么能对它重新调度呢。因此，不能从中断上下文中调用可能会睡眠的函数。

中断上下文具有较为严格的时间限制，因为它打断了其他代码。因为这种异步执行的特性，所以所有的中断处理程序必须尽可能的迅速、简洁。尽量把工作从中断处理程序中分离出来，放在下半部来执行。

中断处理程序不具有自己的栈，它共享被打断进程的内核栈，如果没有进程在运行，就使用idle进程的内核栈。因为中断处理程序共享别人的内核栈，因为获取空间时必须非常节省。

## 5.3 中断处理程序的实现
中断处理系统在linux中的实现是非常依赖于处理器、中断控制器类型、体系结构的设计及机器本身。(略)

比如敲一下键盘：
- 设备产生中断，通过总线把电信号发送给中断控制器
- 如果中断线是激活的，那么中断控制器就会把中断发往处理器
- 处理器停止当前任务，关闭中断系统，跳到内存中中断处理程序的入口点，开始执行代码

## 5.4 中断控制
linux内核提供了一组接口用于操作机器上的中断状态，包括禁止当前处理器的中断系统，或屏蔽掉整个机器的一条中断线的能力。一般来说，控制中断系统的原因归根结底是需要提供同步。通过禁止中断，可以确保某个中断处理程序不会抢占当前的代码。此外，禁止中断还可以禁止内核抢占。

处理器全局控制：
- local_irq_disable()禁止中断
- local_irq_enable()激活中断
- local_irq_save()保存中断
- local_irq_restore()恢复中断

屏蔽特定中断线：
- void disable_irq(unsigned int irq);
- void disable_irq_nosync(unsigned int irq);
- void enable_irq(unsigned int irq);
- void synchronize_irq(unsigned int irq);
