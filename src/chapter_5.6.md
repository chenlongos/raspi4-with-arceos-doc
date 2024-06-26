# USB 介绍

USB（Universal Serial Bus，通用串行总线）是一种用于在计算机系统和外部设备之间传输数据的标准化通信协议和连接接口。USB最初设计用于简化计算机和外部设备之间的连接，并提高设备的可用性。

* **通用性**： USB是一种通用的连接标准，适用于各种设备，包括打印机、键盘、鼠标、摄像头、移动设备等。
* **插拔性**： USB支持热插拔，可以在计算机运行时连接或断开USB设备而无需重启计算机。
* **高带宽和速度**： USB提供不同版本，包括USB 1.0、USB 2.0、USB 3.0、USB 3.1和USB4.0等，每个版本具有不同的数据传输速度和带宽。USB 3.0及更高版本支持更快的数据传输速度。
* **电力供应**： USB接口不仅传输数据，还可以为连接的设备提供电力。这使得许多小型设备无需额外的电源适配器即可正常工作。
* **连接类型**： USB连接器有多种形状和尺寸，Type-A、Type-B、Micro-USB、USB-C等， USB-C是一种可逆连接器，可以在任何方向上插入。
* **传输类型**： USB支持不同类型的数据传输，包括控制传输、批量传输、等时传输和中断传输。

## 拓扑结构

![image](https://github.com/apengaaa/raspi4-with-arceos-doc/assets/83756052/d54b4bfa-2777-4649-9363-58da081f748f)


## USB host

任何 USB 系统中只有一个主机。主机系统的 USB 接口被称为主机控制器。主机控制器可以以硬件，固件或软件的组合来实现。根集线器集成在主机系统内以提供一个或多个连接点。USB Host 通过 Host Controller 与 USB device 交互。

Host 主要负责:检测 USB 设备拔插管理 Host 和 Device 之间的控制流、数据流收集 USB 总线状态和活动数据信息为连入 USB 总线的设备供电。

USB Host的工作原理是通过发送和接收USB数据包来与USB设备进行通信。

## USB Host Controller

**Host Controller**（主机控制器）是指在计算机系统中负责管理和控制USB总线的硬件或芯片。USB主机控制器的作用是与连接到计算机的USB设备进行通信，协调数据传输，并提供电源管理等功能。

* 设备管理： 主机控制器负责检测和管理连接到计算机的USB设备。当用户连接USB设备时，主机控制器负责识别设备并与其建立通信。
* 数据传输： 主机控制器协调数据在USB总线上的传输。它处理USB设备之间的数据传输请求，确保数据以正确的速率和顺序进行传输。
* 电源管理： USB主机控制器负责为连接的USB设备提供电力。它可以根据需要为设备提供适当的电流，并在设备不再需要电力时进行断电。
* 错误处理： 主机控制器能够处理USB总线上发生的错误，以确保稳定的数据传输。
* 协议支持： USB主机控制器需要支持USB规范中定义的各种协议，以确保与不同类型的USB设备的兼容性。

## USB device

USB device 可以分为 USB hub 和 USB function。

* USB Hub
   Hub 提供了一种低成本、低复杂度的 USB 接口扩展方法。HUB 的上行 Port 面向Host，下行 Port 面向设备(Hub 或功能设备)。在下行 Port 上，Hub 提供了设备连接检测和设备移除检测的能力，并给各下行 Port 供电。Hub 可以单独使能各下行 Port。

 


* USB function
  能够通过总线传输或接收数据或控制信息的设备

## USB/XHCI驱动 前情提要&背景知识
* 什么是xhci?
* 答: xhci是USB的控制器, USB1.0的控制器是UHCI/OHCI, 2.0是EHCI, EHCI不向下兼容1.0, 直到USB3.0的XHCI出现，统一了操作标准
* 具体的来说，有没有参考资料？
* 答：[参考资料](https://chenlongos.com/raspi4-with-arceos-doc/chapter_0.00.html)
* XHCI可以做什么？
* 答：分为两部分，首先XHCI控制器负责整个USB系统的数据与其他内存空间的交互，其次，XHCI控制器负责控制USB设备
    * 追问：控制和传输，有区别吗？
    * 答：有，传输不改变设备的状态，具体来说，不管是USB HUB还是USB FUNCTION，其实都是状态机
    * 追问2： HUB/FUNCTION又是什么？
    * 答：这么描述吧：
        * 首先HUB/FUNCTION都是USB设备
        * HUB是可以管理设备的设备（比如扩展坞，值得一提的是其实整个USB设备树的根-称为ROOTHUB,其实也只不过是一个特殊的扩展坞而已）
        * FUNCTION是实际上有功能的设备（例：起存储功能的U盘，起网卡功能的USB网卡...）
    * 追问3：再说说状态机的事？
    * 答：在Intel的XHCI文档中搜索state machine来了解各部分的状态
* 好吧：那么USB系统是如何构成的？
* 答：宏观上是树状结构，逻辑上USB设备都是事件驱动的事件机器：
    * 对于HUB，其内部有两种事件发送端，但是事件的接收端是统一的：
        * 事件接收端：EventRing-其实是个消息队列，当EventRing接收到事件后会想办法通知操作系统-用中断，或者操作系统主动轮询这个队列以检查新到的事件
        * 内部事件发送端：CommandRing-负责发送改变HUB状态的/调用HUB硬件实现的功能的请求事件的地方
        * 设备事件发送端：TransferRing-负责发送与设备的EndPoint交互(控制/传输都通过这个ring进行)的事件的地方
            * 追问：Endpoint是什么？
            * 答：是在物理上传输数据的部分，一个设备可以有很多个EndPoint，具体数量随USB协议版本而变化，EndPoint一般是单向的，Hub与设备的EndPoint建立通信后，这个逻辑概念上的数据传输路线就被叫做数据管道
            * 但是Endpoint0是特殊的，他必然存在，且是唯一同时可以双向通信的EndPoint，负责与HUB交换设备的"控制"事件,人们喜欢把Endpoint0的传输操作叫做"控制传输"
            * 为什么要这么设计？因为更多的管道就是更多的带宽！
        * 事件传输单元：TRB(Transfer Request Block)-每个TRB都由4位u32组成,且容纳这些数据的内存必须16位对齐，其中第四个u32的10-15位内保存的是TRB类型（即事件类型）的唯一标识符
        * 事实上，这三个Ring在数据结构上拥有相同的实现方式：
            其中RING_LENGTH随不同设备而改变，属于配置选项,同时，ring的最后一个单元总是要放一个LinkTrb以标明这里是ring的结尾，需要从头开始循环。
            ```rust
            let ring = [[u32;4];RING_LENGTH];
            let enqueue_pointer:*const [u32;4];
            let dequeue_pointer:*const [u32;4];
            let cycle_bit:bool;
            ```
            * 追问：循环？
            * 答：是的，ring表示这玩意是个用数组实现的循环队列,当然，也有队头(dequeue)和队尾(enqueue)，这两个指针中间的TRB就是正在处理的事件,同时，为了硬件纠错，还引入了cycle_bit，正在处理的事件的cycle_bit必须与环的cycle_bit一致才被视为有效事件，否则当成很久之前就已经处理过了,会直接跳过/报错。
            * TRB大体可分为三类：Control/Transfer/Event TRB,名字就已经表明了他们分别在什么地方出现
    * 对于设备，我们其实并不关心，这部分是制造厂商要负责的事情，我们只在乎数据/控制的传输，把它当成一个塞进去命令就会给反馈的黑盒即可
        * 具体说说！
        * 答：好吧，与设备的通信如同TCP协议一样，有着"三次握手"的格式
        * 首先，发送Setup TRB---这表示一次事件的开始，以及表明了接下来要传输什么数据,setup TRB的类型是有限的，已经在xhci文档/usb文档里列了出来，请自行查阅
        * 然后，发送Data TRB---这部分是可选的，有些TRB并不需要发送数据，比如GetDescriptor TRB，这个Setup TRB中就已经包含了完整的请求信息（请求的数据条目的编号）
        * 最后：发送Status TRB--标准着这次事件的结束，其中包含了一些额外的控制信息，与中断系统/连续传输有关
        * 最后的最后：敲响门铃寄存器来通知设备接受事件，并在EventRing/本设备的doorbell中断上等待设备回报回来的事件。
            * doorbell是什么？
            * 硬件上的实现，这玩意是一个数组，主要用处就是通知被分配在特定slot上的设备有事件被发起，请接收并处理
            * 其中doorbell 0比较特殊，它被称为"默认地址"-实际上是控制器自己，当一个设备刚插上HUB时，它会被分配到默认地址上，（因为这时设备还没有被分配地址），这也就意味着xhci一次性只能处理一个设备插进来的情况，如果在极短的时间插进来了多个设备，那么其他设备就得排队等待
            * 同时这也意味着，每个doorbell 都对应 一个分配出去的slot id，具体的请看xhci 文档
            * 那么什么是Slot？
            * 有这么两个概念：
                * port-HUB物理上有多少个usb口,每个port都有一个寄存器，这些寄存器是连续分配的，可以当初一个数组。
                * slot-hub一共能管理多少设备（包括下游设备，比如hub接hub）,每个设备都有一个slot id，这样就抽象掉了port号的概念。
        * 我还是不明白，控制器怎么知道如何管理这些设备，具体来说，如何进行控制？
        * 答：通过dcbaa/, Device Context Base Address Array
            * Device Context,指的是我们为设备分配的内存区域，是用于控制设备的Endpoint+Slot的区域。
            * dcbaa，是我们分配的一个数组，其上包含了各个Device Context的地址（指针）
            * dcbaap，是xhci的一个寄存器，其值由我们配置为指向dcbaa的地址，xhci正是通过这个寄存器来知道/控制设备的配置状态的。
* 好吧，能将整个流程完整的描述一遍吗？我是指程序上？
* 答：从[这里](https://github.com/arceos-usb/arceos_experiment/tree/phytium_pi_port/crates/driver_usb/guide/../src/lib.rs)的try_init函数入手，这是整个驱动的入口，同时辅以xhci文档的第四章来确定你当前看的是哪一步。同时也可以参考[飞腾派的官方嵌入式sdk](https:/gitee.com/phytium_embedded/phytium-standalone-sdk)
    * 经过与官方沟通的最新进展，sdk中的xhci并不稳定且正确，详细的请参考[沟通记录](https://github.com/arceos-usb/arceos_experiment/tree/phytium_pi_port/crates/driver_usb/question/question_5_29.md),因此，请转而参考他们的freertos仓库。
* 行！那么目前还需要干什么？
* 参考[问题文件夹](https://github.com/arceos-usb/arceos_experiment/tree/phytium_pi_port/crates/driver_usb/guide/../question)，其中是与飞腾官方所沟通的一些问题，也是我们驱动目前需要解决的问题，如果有新问题，也请加进去。

  







