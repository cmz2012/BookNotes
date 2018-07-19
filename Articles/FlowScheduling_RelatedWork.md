# 相关工作汇总
## 1.DCTCP
- 会议：SIGCOMM'10
- 解决的问题：传统TCP协议算法在数据中心混合负载环境下不能有效处理延迟敏感的前台网络流
- 解决方法：DCTCP利用显式网络反馈ECN标记来量化网络拥塞程度，然后根据拥塞程度使用拥塞控制算法来调整拥塞窗口大小，可以解决拥塞端口的队列增长，缓存压力和incast问题

## 2.Better Never than Late: Meeting Deadlines in Datacenter Networks(D3)
- 会议：微软研究院技术报告
- 解决的问题：数据中心网络中许多应用的网络流具有软实时性质，而目前数据中心网络的设计和优化仅仅在于吞吐量和公平性，这种网络和应用之间的不匹配严重影响应用的性能，包括
  - 不公平的共享：一个具有软实时性质和一个不具有软实时性质的网络流之间均分网络带宽
  - 过载：多条具有软实时性质的网络流均分网络带宽而导致所有的网络流都无法在期限前完成传输
- 挑战：
  - 期限是网络流的属性而不是数据包的属性
  - 网络流的期限变化非常大
  - 大多数网络流非常短( < 50KB )，而RTT非常小( ~300us)，结果反应时间非常短，集中式的或者代价高的为网络流预留带宽机制是不切实际的
- 解决方法：D3赋予每个网络流独立的期限参数和大小参数，并计算所需要的转发速率，将这个速率写入该流的每个数据包中并发送到目的结点，当该数据包的ACK包逆向经过每个路由器时，路由器根据经过的每个流的期限计算分配速率后写入当前能为该流分配的速率，最终到达源主机端时，源主机提取出速率向量中最小的速率作为下一次为该流发送的速率。

## 3.Deadline-Aware Datacenter TCP(D2TCP)
- 会议：SIGCOMM'12
- 解决的问题：满足网络流期限，尤其是在fan-in-burst-induced拥塞状态下，使用现有的交换机硬件，能够和传统TCP共存
- 挑战：
  - D3的贪心带宽分配算法会造成优先级翻转问题，进而造成错过期限
  - 快速的流到达速率和严格的期限下，交换机不能等待收集所有流的信息再去做决策
  - 在交换机上维持每个流的状态相当复杂
  - 由于缺乏每个流的详细状态，交换机维持一些空闲带宽以作不备之需比较困难
- 解决方法：
  - 在主机端做分布式和反应式的带宽分配方法
    - 每个主机端的网络流独立的调整拥塞窗口大小，根据网络反馈信息自动调整，以避免D3中的优先级翻转问题
  - 使用ECN反馈信息和期限信息来调整拥塞窗口大小，实现拥塞避免算法
    - 在DCTCP基础上加上期限信息
    - α = (1-g) x α + g x f
    - p = α ^ d
    - w = w x (1 - p / 2), if p > 0
    - w = w + 1, if p = 0

## 4.Finishing Flows Quickly with Preemptive Scheduling(PDQ)
- 会议：SIGCOMM'12
- 解决的问题：现有的TCP系列不能满足具有软实时性网络流的需求，而D3虽然一定程度上满足了这个需要求，但是集中式方法的不可抢占造成了优先级翻转问题
- 挑战：
  - 集中式调度通信开销大，调度器要维护的数据量也大，而期限属性一般只存在与小流中，对于只维护大流的调度器来说无法调度
  - 流切换时浪费了带宽
  - 现在交换机支持的优先级队列有限，为每个不同期限的流都做优先级队列不可能
- 解决方法：
  - 主机端将流的重要信息编码到数据包包头，发送到网络中去，每收到一个ack就根据ack包含的速率信息调整发送速率
  - 交换机维持每条链路上的网络流状态，对流做重要程度比较，优先发送更重要的流，可以实时抢占
  - 接收端会根据buffer占用率在ack包头中写入新的速率

## 5.pFabric:Minimal Near-Optimal Datacenter Transport
- 会议：SIGCOMM'13
- 解决的问题：隐式改善FCT的技术不能精确计算正确的流速率来最优化调度网络流；显式计算速率的技术需要交换机知道详细的流信息并且协作来调度网络流，方法复杂且难以实现，有没有一种简单的数据中心传输能提供最优的FCT
- 解决方法：
  - 将流调度与速率控制分离开来
  - 主机端将流信息编码成优先级，放在数据包包头
  - 交换机根据数据包的优先级进行调度，最先传输最高优先级的包，最先丢弃最低优先级的包
  - 速率控制：一开始都以线速运行，直到看到持续丢包时开始降速

## 6.Adaptive Data Transmission in the Cloud
- 会议：IWQoS'13
- 解决的问题：同DCTCP
- 挑战：必须使用现有设备，不能大量修改系统和应用
  - 当多条流竞争时，怎么精确的控制流的速率
  - 为不同的流分配多少带宽
  - 怎么对未知的流进行分配带宽
- 解决方法：
  - 引入一个权重函数，从流已经发送的字节数提取出权重
  - 修改TCP的加法增加行为，将窗口增量与权重成正比，做到加权资源共享

## 7.pHost:Distributed Near-Optimal Datacenter Transport Over Commodity Network Fabric
- 会议：CoNext'15
- 解决的问题：是否有可能使用普通交换机也能达到pFabric的理想性能
- 挑战：没有交换机的辅助(ECN信息)，不知道网络上的状态，怎么达到最优的FCT
- 解决方法：
  - 替换掉原来的拥塞窗口机制，使用End-to-End的具有时间限制的Token控制机制来控制发送速率
  - 在发送端和接收端同时根据优先级信息对网络流做优先级调度
  - 交换机端只设三个优先级队列即可，最高优先级传输控制包，中优先级传输小流，低优先级传输大流
  - 如果出现丢包，那么接收端就重传token

## 8.Information-Agnostic Flow Scheduling for Commodity Data Centers
- 会议：NSDI'15
- 解决的问题：在没有流的先验知识和普通交换机环境下，最小化流的FCT
- 解决方法：
  - 使用多级反馈队列(MLFQ)来模拟SJF
  - 主机端对不同大小的流片段做优先级标示
  - 交换机根据流的优先级做优先级调度，在感知到拥塞时ECN标识
  - 主机端的速率控制使用DCTCP的方法

## 9.Scheduling Mix-flows in Commodity Datacenters with Karuna
- 会议：SIGCOMM'16
- 解决的问题：在混合流场景下，严格优先调度有期限的流会影响没有期限的流的完成时间，如何保证有期限的流在期限前完成的同时减少没有期限的流的完成时间