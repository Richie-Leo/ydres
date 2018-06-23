> [Martin Fowler](https://martinfowler.com/)，[The LMAX Architecture](https://martinfowler.com/articles/lmax.html) 

LMAX is a new retail financial trading platform. As a result it has to process many trades with low latency. ==The system is built on the JVM platform and centers on a Business Logic Processor that can handle 6 million orders per second on a single thread==. The Business Logic Processor runs entirely in-memory using event sourcing. The Business Logic Processor is surrounded by Disruptors - a concurrency component that implements a network of queues that operate without needing locks. ==During the design process the team concluded that recent directions in high-performance concurrency models using queues are fundamentally at odds with modern CPU design.== 

LMAX是一个新兴的ToC金融交易平台，需要低延迟处理大量交易，系统基于Java平台构建，核心业务逻辑处理器单线程每秒可以处理600万订单。业务逻辑处理器采用Event Sourcing模式完全运行在内存中，使用大量由环状无锁队列实现的并发控制组件Disruptor完成工作。设计过程中，LMAX团队发现目前基于队列实现高性能并发控制的趋势从根本上违背了现代CPU设计理念（译注：现代CPU的设计理念是让单个CPU更高效快速的执行指令，在这方面做了大量的工作，例如CPU L1、L2、L3缓存的设计，反复执行紧凑的代码段，频繁访问连续集中的小段数据，可以使CPU有效利用率最大化，但各种并发控制机制为了充分利用多CPU处理能力都会采用锁、多线程机制，导致频繁的线程上下文切换、内核模式切换等，使CPU缓存失效，降低了CPU的有效利用率。这就是LMAX总结出的结论，基于这些原因而选择了单线程机制，并且实现了高吞吐率）。

Over the last few years we keep hearing that "the free lunch is over"[1] - we can't expect increases in individual CPU speed. So to write fast code we need to explicitly use multiple processors with concurrent software. This is not good news - writing concurrent code is very hard. Locks and semaphores are hard to reason about and hard to test - meaning we are spending more time worrying about satisfying the computer than we are solving the domain problem. Various concurrency models, such as Actors and Software Transactional Memory, aim to make this easier - but there is still a burden that introduces bugs and complexity.

过去几年一直有人提到“没有免费的午餐”：我们不能寄望于CPU速度的提升让代码运行快起来，提升速度必须采用并发机制以利用多CPU能力。这不是件容易的事情，开发并发代码很困难，锁和信号量很难理解和测试，这意味着需要耗费大量时间来对付计算机而不是解决领域问题。虽然Actor和Sorftware Transactional Memory等一些并发模型使这类事情变得简单，但仍额外引入了复杂性和无法预料的Bug。

So I was fascinated to hear about a talk at QCon London in March last year from LMAX. LMAX is a new retail financial trading platform. Its business innovation is that it is a retail platform - allowing anyone to trade in a range of financial derivative products[2]. A trading platform like this needs very low latency - trades have to be processed quickly because the market is moving rapidly. A retail platform adds complexity because it has to do this for lots of people. So the result is more users, with lots of trades, all of which need to be processed quickly.[3] 

去年三月在伦敦的QCon会议上来自LMAX的一场演讲吸引了我。LMAX是一个新兴的ToC金融交易平台，它的业务创新在于ToC模式，允许任何人交易标的范围内的金融衍生品。这样的交易平台要求非常低的延时性，市场快速变化，交易必须尽快完成处理。大用户量给这样的ToC平台增加了复杂性，大量用户，大量交易请求，所有一切必须快速完成处理。

Given the shift to multi-core thinking, this kind of demanding performance would naturally suggest an explicitly concurrent programming model - and indeed this was their starting point. But the thing that got people's attention at QCon was that this wasn't where they ended up. In fact they ended up by doing all the business logic for their platform: all trades, from all customers, in all markets - on a single thread. A thread that will process 6 million orders per second using commodity hardware.[4] 

通常情况下如果想发挥多CPU能力，对这类性能需求会很自然地提出一种典型的并发控制方案，实际上他们一开始也是这么想的，但在QCon会议上让大家感兴趣的是他们并没有这么做，实际上他们采用了单线程来处理平台上的所有业务逻辑，所有客户、所有市场的所有交易都在一个单线程中处理，运行在商用服务器上的单线程1秒可以处理600万订单。

Processing lots of transactions with low-latency and none of the complexities of concurrent code - how can I resist digging into that? Fortunately another difference LMAX has to other financial companies is that they are quite happy to talk about their technological decisions. So now LMAX has been in production for a while it's time to explore their fascinating design. 

以低延时处理大量交易，且不引入常规并发控制机制的复杂性，他们是怎么做到的？有幸的是LMAX不同于其他金融公司，他们很乐意交流他们的技术方案，目前为止LMAX已经上线运行一段时间了，我们也有幸能够揭开它的神秘面纱一窥究竟。

--------------------------------------------------------------------------------
### Overall Structure

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/180/1002/10-180-1002-01-arch-summary.png?raw=true" /><br />Figure 1: LMAX's architecture in three blobs</center>

At a top level, the architecture has three parts<br />
整体上看LMAX架构由三个部分组成

- business logic processor[5]
- input disruptor
- output disruptors

As its name implies, the business logic processor handles all the business logic in the application. As I indicated above, it does this as a single-threaded java program which reacts to method calls and produces output events. Consequently it's a simple java program that doesn't require any platform frameworks to run other than the JVM itself, which allows it to be easily run in test environments.

正如其名，业务逻辑处理器处理应用中的所有业务逻辑，前面提到它使用单线程Java程序实现，根据方法调用产生输出事件，除了JVM外不依赖其它框架，很容易测试。

Although the Business Logic Processor can run in a simple environment for testing, there is rather more involved choreography to get it to run in a production setting. Input messages need to be taken off a network gateway and unmarshaled, replicated and journaled. Output messages need to be marshaled for the network. These tasks are handled by the input and output disruptors. Unlike the Business Logic Processor, these are concurrent components, since they involve IO operations which are both slow and independent. They were designed and built especially for LMAX, but they (like the overall architecture) are applicable elsewhere.

虽然业务逻辑处理器可以在简单环境中执行测试，但放到生产环境运行还需要很多其他工作，例如从网络读取输入消息，进行解析、复制、记录日志，输出消息必须经过封装后交由网络传输，这些工作由input和output disruptor完成。与业务逻辑处理器不同，disruptor是并发控制组件，包含独立的、比较慢的IO操作。Disruptor是为LMAX平台设计开发的，但跟他们的整体架构方案一样，对其他系统也具备借鉴意义。

--------------------------------------------------------------------------------
### Business Logic Processor

#### Keeping it all in memory

The Business Logic Processor takes input messages sequentially (in the form of a method invocation), runs business logic on it, and emits output events. ==It operates entirely in-memory==, there is no database or other persistent store. Keeping all data in-memory has two important benefits. Firstly it's fast - there's no database to provide slow IO to access, nor is there any transactional behavior to execute since all the processing is done sequentially. The second advantage is that it simplifies programming - there's no object/relational mapping to do. All the code can be written using Java's object model without having to make any compromises for the mapping to a database.

业务逻辑处理器按调用顺序读取消息并进行处理，产生输出事件，完全基于内存工作，不涉及数据库和其他持久化仓库操作。将所有数据放在内存中有两个好处，首先是速度快，不存在数据库访问低速IO操作，所有处理都顺序执行，因此无需事务机制。其次简化了开发工作，无需ORM，所有代码使用Java对象模型，不需要进行数据库映射。

Using an in-memory structure has an important consequence - what happens if everything crashes? Even the most resilient systems are vulnerable to someone pulling the power. ==The heart of dealing with this is Event Sourcing== - which means that the current state of the Business Logic Processor is entirely derivable by processing the input events. As long as the input event stream is kept in a durable store (which is one of the jobs of the input disruptor) you can always recreate the current state of the business logic engine by replaying the events.

使用内存数据带来一个问题，如果宕机怎么办？再健壮的系统也顶不住某人意外拔掉了电源，解决这个问题的关键是Event Sourcing模式，业务逻辑处理器的结果完全可以通过输入事件重新推导出来，只要将输入事件保存到持久化仓库（这是其中一个disruptor的工作），你就可以重新处理事件得到正确结果。

A good way to understand this is to think of a version control system. Version control systems are a sequence of commits, at any time you can build a working copy by applying those commits. VCSs are more complicated than the Business Logic Processor because they must support branching, while the Business Logic Processor is a simple sequence.

版本控制系统有助于理解这一原理，它包含一系列提交操作，任何时候你都可以通过这些提交操作创建一份最新版本。但版本控制系统比业务逻辑处理器更复杂，因为还需要处理分支等情况，而业务逻辑控制器仅仅是顺序执行处理。

So, in theory, you can always rebuild the state of the Business Logic Processor by reprocessing all the events. In practice, however, that would take too long should you need to spin one up. So, just as with version control systems, ==LMAX can make snapshots of the Business Logic Processor state and restore from the snapshots. They take a snapshot every night during periods of low activity==. Restarting the Business Logic Processor is fast, a full restart - including restarting the JVM, loading a recent snapshot, and replaying a days worth of journals - takes less than a minute.

理论上可以通过重新处理所有事件得到最新结果，但这需要花费很长时间，因此LMAX借鉴版本控制系统建立镜像，通过镜像快速恢复系统。他们在每天晚上交易量较小的情况下创建镜像，彻底重启系统，包括重启JVM，业务逻辑处理器加载最新镜像，重新处理当天的日志，这样的重启速度非常快，不超过1分钟。

Snapshots make starting up a new Business Logic Processor faster, but not quickly enough should a Business Logic Processor crash at 2pm. ==As a result LMAX keeps multiple Business Logic Processors running all the time[6].== Each input event is processed by multiple processors, but all but one processor has its output ignored. ==Should the live processor fail, the system switches to another one==. This ability to handle fail-over is another benefit of using Event Sourcing.

镜像使得启动一个业务逻辑处理器的速度非常快，但如果下午2点发生宕机，重启速度还是无法接受（译注：1. 到下午2点时当天已经产生了很多交易，逐条处理需要耗费较长时间，2. 金融系统发生几十秒的宕机时间也会造成比较严重的影响。因此说这是无法忍受的），因此LMAX对同一个业务逻辑处理器会同时运行多个实例，每个输入事件都会交给多个处理器进行处理，只使用其中一个处理器的结果，当这个处理器发生故障时，系统切换到另外一个处理器上，这种故障恢复方式是Event Sourcing带来的另外一个优势。

By event sourcing into replicas they can switch between processors in a matter of micro-seconds. As well as taking snapshots every night, they also restart the Business Logic Processors every night. The replication allows them to do this with no downtime, so they continue to process trades 24/7.

对Event Sourcing进行冗余复制，在故障时可以在毫秒级完成切换，加上每晚创建镜像，重启业务逻辑处理器，业务可以实现7x24无故障运行。

> For more background on Event Sourcing, see the draft pattern on my site from a few years ago. The article is more focused on handling temporal relationships rather than the benefits that LMAX use, but it does explain the core idea.

==Event Sourcing is valuable because it allows the processor to run entirely in-memory, but it has another considerable advantage for diagnostics==. If some unexpected behavior occurs, the team copies the sequence of events to their development environment and replays them there. This allnows them to examine what happened much more easily than is possible in most environments.

除了上述优点，Event Sourcing还有一个优势是故障诊断。发生异常时维护人员将事件拷贝到开发环境，重新按顺序处理事件，这比其他任何方式都更容易分析定位问题原因（译注：没有使用其他框架，不存在并发控制、事务处理等复杂机制，也不存在数据库交互等，纯Java代码实现的业务逻辑，顺序处理事件的结果是非常确定的，很多系统生产环境上的一些诡异问题非常难以诊断，很大一部分原因是因为各种框架的复杂机制，Java代码、数据库等各层的多线程并发执行、事物控制等原因造成）。

This diagnostic capability extends to business diagnostics. There are some business tasks, such as in risk management, that require significant computation that isn't needed for processing orders. An example is getting a list of the top 20 customers by risk profile based on their current trading positions. ==The team handles this by spinning up a replicate domain model and carrying out the computation there, where it won't interfere with the core order processing==. These analysis domain models can have variant data models, keep different data sets in memory, and run on different machines.

这种诊断能力还有助于提升业务健康度监测，例如风险管控，它包含大量的运算处理，而这些处理不包含在正常的订单业务中，例如需要根据风险管控要求获取前20名客户名单等任务。LMAX团队建立了另外的领域模型来完成这些处理，这些领域模型可以采用不同的数据模型，内存数据结构也完全不同，独立运行在其他机器上，根本不会对核心订单处理业务造成任何影响。

#### Tuning performance

So far I've explained that the key to the speed of the Business Logic Processor is doing everything sequentially, in-memory. Just doing this (and nothing really stupid) allows developers to write code that can process 10K TPS[7]. They then found that concentrating on the simple elements of good code could bring this up into the 100K TPS range. This just needs well-factored code and small methods - essentially this allows Hotspot to do a better job of optimizing and for CPUs to be more efficient in caching the code as it's running.

前面解释了业务逻辑处理器实现高性能的关键是纯内存运行、顺序执行，做到这两点并且不犯傻，可以实现1万TPS（译注：可对比Redis、memcached等基于纯内存运行的框架，可轻松实现1-2万TPS）。接下来LMAX发现在这种简洁的情况下聚焦代码优化，可以实现10万TPS，这需要优秀的代码结构，使用小的Java方法，这有助于Hotspot更好地优化代码，以及提升CPU代码缓存效率。

It took a bit more cleverness to go up another order of magnitude. There are several things that the LMAX team found helpful to get there. ==One was to write custom implementations of the java collections that were designed to be cache-friendly and careful with garbage[8].== An example of this is using primitive java longs as hashmap keys with a specially written array backed Map implementation (LongToObjectHashMap). In general they've found that choice of data structures often makes a big difference, Most programmers just grab whatever List they used last time rather than thinking which implementation is the right one for this context.[9]

其他一些深层次的优化则需要更多技巧，LMAX所做的一个工作是重写了Java集合类的实现，以提升缓存效率，改善垃圾回收状况，其中一个例子是使用数组来实现hashmap，并且改为使用基本类型long作为hashmap的key（译注：JDK自带的HashMap实现要求使用Object类型作为key，在可以使用整数、长整数作为key的情况下必须使用Integer、Long等Object类型，一方面代码执行时需要额外的box、unbox操作导致，这些额外的代码指令会影响CPU代码缓存命中率，也会加剧Java垃圾回收的负担）。他们经常发现数据结构的选择对性能的影响非常明显，大部分程序员总是很随意地使用某种List类型，而不去思考当前场景下使用哪种实现更合适。

Another technique to reach that top level of performance is putting attention into performance testing. I've long noticed that people talk a lot about techniques to improve performance, but the one thing that really makes a difference is to test it. Even good programmers are very good at constructing performance arguments that end up being wrong, so the best programmers prefer profilers and test cases to speculation.[10] The LMAX team has also found that writing tests first is a very effective discipline for performance tests.

实现高性能的另外一个方面是重视性能测试。长期以来人们大量讨论性能优化技术，但真正发挥作用还需要测试验证，即使是优秀的程序员也容易得出某些关于性能的观点，但最终被证明是错误的，因此最好的程序员都善于利用调优工具和测试用例进行验证，LMAX团队实践得出的一个非常有效的性能测试原则是提前写测试。

#### Programming Model

This style of processing does introduce some constraints into the way you write and organize the business logic. The first of these is that you have to tease out any interaction with external services. An external service call is going to be slow, and with a single thread will halt the entire order processing machine. As a result you can't make calls to external services within the business logic. Instead you need to finish that interaction with an output event, and wait for another input event to pick it back up again.

这种处理方式给设计和开发带来了一些约束，首先必须对所有外部服务交互抽丝剥茧进行设计。调用外部服务速度慢，单线程方式下会阻塞整个订单处理过程，因此不能在业务逻辑处理中直接调用，需要产生一个输出事件触发异步调用方式，异步执行完成后再通过一个输入事件重新交给业务逻辑处理器进行后续处理。

I'll use a simple non-LMAX example to illustrate. Imagine you are making an order for jelly beans by credit card. A simple retailing system would take your order information, use a credit card validation service to check your credit card number, and then confirm your order - all within a single operation. The thread processing your order would block while waiting for the credit card to be checked, but that block wouldn't be very long for the user, and the server can always run another thread on the processor while it's waiting.

用一个非LMAX的例子来说明这种情况。假设使用信用卡支付购买糖豆，简单购物系统的做法是接收订单时先调用信用卡校验服务，然后确认订单。这些处理在一个操作中完成，调用信用卡校验服务时下单线程处于阻塞等待状态，阻塞时间不会太长，期间处理器还有其他线程处理其他用户请求。

In the LMAX architecture, you would split this operation into two. The first operation would capture the order information and finish by outputting an event (credit card validation requested) to the credit card company. The Business Logic Processor would then carry on processing events for other customers until it received a credit-card-validated event in its input event stream. On processing that event it would carry out the confirmation tasks for that order.

采用LMAX架构时，上述操作必须分成2个步骤执行，先接收订单信息并向信用卡公司输出一个事件（信用卡校验请求），然后业务逻辑处理器接着处理其他用户请求，当输入事件流中接收到信用卡校验通过的事件后，再接着执行这个订单的确认操作。

Working in this kind of event-driven, asynchronous style, is somewhat unusual - although using asynchrony to improve the responsiveness of an application is a familiar technique. It also helps the business process be more resilient, as you have to be more explicit in thinking about the different things that can happen with the remote application.

虽然采用异步方式提升应用程序的响应性是一种常用方法，但使用LMAX这种事件驱动型异步方式在某些方面还是有很大区别的，你必须更深入地考虑第三方应用可能发生的各种情况，这也有助于提升业务处理的健壮性（译注：因为单线程方式，任何考虑不周可能会阻塞或中断整个订单处理过程，造成严重后果）。

A second feature of the programming model lies in error handling. The traditional model of sessions and database transactions provides a helpful error handling capability. Should anything go wrong, it's easy to throw away everything that happened so far in the interaction. Session data is transient, and can be discarded, at the cost of some irritation to the user if in the middle of something complicated. If an error occurs on the database side you can rollback the transaction.

另外一个影响开发方式的地方在于错误处理。传统的会话管理和数据库事务提供了良好的错误处理机制，当前处理一旦发生错误，可以很方便地将详细信息抛出来。会话数据是临时数据，发生错误时可以丢弃，只是会给用户造成一些困扰而已，在数据库中发生错误时可以回滚事务。

LMAX's in-memory structures are persistent across input events, so if there is an error it's important to not leave that memory in an inconsistent state. However there's no automated rollback facility. As a consequence the LMAX team puts a lot of attention into ensuring the input events are fully valid before doing any mutation of the in-memory persistent state. They have found that testing is a key tool in flushing out these kinds of problems before going into production.

LMAX的结果保存在内存中，但却是以输入事件的形式持久化到磁盘上，发生错误时确保内存结果的一致性就非常重要（译注：持久化保存到磁盘的是一条条事件记录，而内存中保存的是根据这些事件记录计算处理后的最终结果）。这里没有自动回滚的工具可用，因此在持久化输入事件时，LMAX团队花费大量精力对输入事件进行校验，防止将这类问题引入生产环境的关键在于测试。

--------------------------------------------------------------------------------
### Input and Output Disruptors

Although the business logic occurs in a single thread, there are a number tasks to be done before we can invoke a business object method. The original input for processing comes off the wire in the form of a message, this message needs to be unmarshaled into a form convenient for Business Logic Processor to use. Event Sourcing relies on keeping a durable journal of all the input events, so each input message needs to be journaled onto a durable store. Finally the architecture relies on a cluster of Business Logic Processors, so we have to replicate the input messages across this cluster. Similarly on the output side, the output events need to be marshaled for transmission over the network.

虽然业务逻辑在单线程中处理，在执行业务处理之前还有很多其他工作需要完成。原始请求以消息形式提交过来，需要解析转换成便于业务逻辑处理器使用的形式。Event Sourcing模式要求为所有输入事件记录持久化日志，因此需要将输入消息保存到持久化仓库中。最后，架构方案要求业务逻辑处理器组成一个集群，因此必须为集群复制分发消息。在输出端，输出事件需要经过封装用于网络传输。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/180/1002/10-180-1002-02-input-activity.png?raw=true" /><br />Figure 2: The activities done by the input disruptor (using UML activity diagram notation)</center>

The replicator and journaler involve IO and therefore are relatively slow. After all the central idea of Business Logic Processor is that it avoids doing any IO. Also these three tasks are relatively independent, all of them need to be done before the Business Logic Processor works on a message, but they can done in any order. So unlike with the Business Logic Processor, where each trade changes the market for subsequent trades, there is a natural fit for concurrency.

图中的复制和日志记录包含IO操作，执行速度慢，正因如此，业务逻辑处理器的核心思想就是避免任何IO操作。中间三个任务相对独立，对于单个消息，业务逻辑处理器必须等他们全部处理完成才能开始工作。这三个任务的先后执行顺序无关紧要，天然适合于并发执行，这不同于业务逻辑处理器，每笔交易都会影响市场以及后续交易。

To handle this concurrency the LMAX team developed a special concurrency component, which they call a Disruptor[11].

LMAX团队为这样的并发场景开发了一个特殊的并发控制组件，叫做Disruptor。

> The LMAX team have released the source code for the Disruptor with an open source licence.

At a crude level you can think of a Disruptor as a multicast graph of queues where producers put objects on it that are sent to all the consumers for parallel consumption through separate downstream queues. When you look inside you see that this network of queues is really a single data structure - ==a ring buffer==. Each producer and consumer has a sequence counter to indicate which slot in the buffer it's currently working on. Each producer/consumer writes its own sequence counter but can read the others' sequence counters. This way the producer can read the consumers' counters to ensure the slot it wants to write in is available without any locks on the counters. Similarly a consumer can ensure it only processes messages once another consumer is done with it by watching the counters.

粗略理解disruptor你可以认为它是一个由队列构成的组播结构，生产者将对象放进来就可以发送给所有消费者，消费者通过独立的下发队列并行消费处理。深究的话disruptor只包含一个单一的数据结构：一个环形缓冲区。每个生产者和消费者使用一个序列号计数器来记录自己在环形缓冲区中的当前操作位置，生产者和消费者能够更新自己的计数器，能够读取但不能更新其他人的计数器。这种机制使得生产者可以通过读取消费者的计数器，用于判断即将写入的位置是否处于空闲可用状态。另外，消费者也可以通过监控其他某个消费者的计数器，用来实现譬如必须在消费者A处理完之后才能由消费者B处理这类需求。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/180/1002/10-180-1002-03-disruptor.png?raw=true" /><br />Figure 3: The input disruptor coordinates one producer and four consumers</center>

1. ==接收器receiver正在向31槽写入消息，它可以继续向前写入，只需确保没有越过业务逻辑消费者即可；==
2. ==日志器journaler和复制器replicator都在读取24槽的消息，它们可以继续向前读取到30槽的位置；==
3. ==解析器un-marshaller休眠了，休眠结束后将读取15槽，它可以继续向前读取到30槽的位置；==
4. ==业务逻辑消费者不能够超越其他三个消费者中的任何一个，因此必须在14槽位置等待解析器结束休眠向前处理；==

Output disruptors are similar but they only have two sequential consumers for marshaling and output.[12] Output events are organized into several topics, so that messages can be sent to only the receivers who are interested in them. Each topic has its own disruptor.

Output disruptor同input disruptor类似，它只管理两个串行的消费者，分别执行封装和输出操作。输出事件按话题组织，用于发送给不同接收者，每个接收者都拥有自己的disruptor。

The disruptors I've described are used in a style with one producer and multiple consumers, but this isn't a limitation of the design of the disruptor. The disruptor can work with multiple producers too, in this case it still doesn't need locks.[13]

上面描述的disruptor由一个生产者和多个消费者组成，但这不是disruptor模式的设计约束，disruptor允许多个生产者，同样无需使用锁机制。

A benefit of the disruptor design is that it makes it easier for consumers to catch up quickly if they run into a problem and fall behind. If the unmarshaler has a problem when processing on slot 15 and returns when the receiver is on slot 31, it can read data from slots 16-30 in one batch to catch up. This batch read of the data from the disruptor makes it easier for lagging consumers to catch up quickly, thus reducing overall latency.

Disruptor设计的其中一个优点是，在消费者由于某种原因导致处理速度落后的情况下能够快速追赶上来。例如解析器在处理15槽消息时遇到一个问题，在它恢复时接收器位于31槽，那么它可以一批次将16-30槽的内容读取出来进行处理，这样有利于加快处理速度，降低整体延迟，将处理速度追赶上来。

I've described things here, with one each of the journaler, replicator, and unmarshaler - this indeed is what LMAX does. But the design would allow multiple of these components to run. ==If you ran two journalers then one would take the even slots and the other journaler would take the odd slots. This allows further concurrency of these IO operations should this become necessary.==

这里列举的disruptor，每个都包含一个日志器、一个复制器、一个解析器，这是LMAX的真实使用情况。在设计方案上是允许使用多个这种组件的，比如可以使用两个日志器，一个处理偶数槽位，一个处理奇数槽位，这样可以进一步提升日志IO操作的并行性。

==The ring buffers are large: 20 million slots for input buffer and 4 million slots for each of the output buffers==. The sequence counters are 64bit long integers that increase monotonically even as the ring slots wrap.[14] ==The buffer is set to a size that's a power of two so the compiler can do an efficient modulus operation to map from the sequence counter number to the slot number==. Like the rest of the system, the disruptors are bounced overnight. This bounce is mainly done to wipe memory so that there is less chance of an expensive garbage collection event during trading. (I also think it's a good habit to regularly restart, so that you rehearse how to do it for emergencies.)

环形缓冲区设置得比较大，输入缓冲包含2000万槽位，每个输出缓冲包含4百万槽位。虽然环形缓冲区是循环使用的，但序列号计数器使用64位长整数表示。缓冲区大小设置为2的幂次方，这样编译器能够充分利用求模运算将序列号映射成槽号。Disruptor在晚上系统重启时进行重置，重启的主要作用是回收内存，这样可以尽可能避免在交易时段进行昂贵的垃圾回收操作（我认为经常重启是一种好习惯，可以将它当做灾难恢复的预演）。

The journaler's job is to store all the events in a durable form, so that they can be replayed should anything go wrong. ==LMAX does not use a database for this, just the file system==. They stream the events onto the disk. In modern terms, mechanical disks are horribly slow for random access, but very fast for streaming - hence the tag-line "disk is the new tape".[15]

日志器的工作是将事件持久化保存下来，用于故障恢复时使用。LMAX没有使用数据库，只使用文件系统保存日志，他们将事件以顺序流形式写入磁盘，现在的机械磁盘在随机访问时非常慢，但以顺序流方式访问则很快，因此也有说法“磁盘即新式磁带”。

Earlier on I mentioned that LMAX runs multiple copies of its system in a cluster to support rapid failover. ==The replicator keeps these nodes in sync. All communication in LMAX uses IP multicasting, so clients don't need to know which IP address is the master node==. Only the master node listens directly to input events and runs a replicator. The replicator broadcasts the input events to the slave nodes. Should the master node go down, it's lack of heartbeat will be noticed, another node becomes master, starts processing input events, and starts its replicator. Each node has its own input disruptor and thus has its own journal and does its own unmarshaling.

我之前提到LMAX使用多个备机组成集群，用于支持快速故障恢复。各节点由复制器进行同步，所有通讯使用IP组播方式，客户端不用关心主节点的IP地址。只在主节点上监听输入事件，运行复制器，由复制器向所有从节点广播输入事件。主节点发生宕机时，可以通过心跳监测到，将一个从节点切换成主节点，由它监听输入事件，启动复制器。

==Even with IP multicasting, replication is still needed because IP messages can arrive in a different order on different nodes. The master node provides a deterministic sequence for the rest of the processing.==

虽然使用了IP组播方式，但仍然需要复制，因为组播消息的先后顺序无法保证，为此主节点会生成一个序列号用于解决这个问题。

The unmarshaler turns the event data from the wire into a java object that can be used to invoke behavior on the Business Logic Processor. Therefore, unlike the other consumers, it needs to modify the data in the ring buffer so it can store this unmarshaled object. The rule here is that consumers are permitted to write to the ring buffer, but each writable field can only have one parallel consumer that's allowed to write to it. This preserves the principle of only having a single writer. [16]

解析器将事件数据转换成调用业务逻辑处理器所需的对象类型，跟其他消费者不一样，它需要修改环形缓冲中的数据，将解析后的对象更新回环形缓冲。原则上可以允许多个消费者更新环形缓冲，但每个可更新域同一时刻只允许一个消费者更新。也可以只允许一个更新者，将其作为一个备选原则。

<center><img src="https://github.com/Richie-Leo/ydres/blob/master/img/10/180/1002/10-180-1002-04-arch-full.png?raw=true" /><br />Figure 4: The LMAX architecture with the disruptors expanded</center>

The disruptor is a general purpose component that can be used outside of the LMAX system. Usually financial companies are very secretive about their systems, keeping quiet even about items that aren't germane to their business. Not just has LMAX been open about its overall architecture, they have open-sourced the disruptor code - an act that makes me very happy. Not just will this allow other organizations to make use of the disruptor, it will also allow for more testing of its concurrency properties.

Disruptor是一个通用组件，可用于LMAX之外的其它系统。通常金融公司系统都高度保密，哪怕技术和系统不合时宜也不吭声。LMAX不仅分享了他们的整体架构方案，还开源了disruptor代码，这一举措让我感到非常欣慰。这不仅能让其他公司用上disruptor，也能帮助disruptor在并发能力方面进行更充分的测试。

--------------------------------------------------------------------------------
### Queues and their lack of mechanical sympathy 

The LMAX architecture caught people's attention because it's a very different way of approaching a high performance system to what most people are thinking about. So far I've talked about how it works, but haven't delved too much into why it was developed this way. This tale is interesting in itself, because this architecture didn't just appear. It took a long time of trying more conventional alternatives, and realizing where they were flawed, before the team settled on this one.

LMAX架构吸引了人们的兴趣，因为它采用了一种非常另类的方法实现高性能。前面我介绍了它的工作原理，但还没怎么谈及他们为什么这么做，其中包含了很多有趣的故事，因为这个架构并不是一步登天而形成，期间耗费了很长时间尝试各种常规技术方案，后来发现存在各种缺陷，直到最终采用这种方案。

Most business systems these days have a core architecture that relies on multiple active sessions coordinated through a transactional database. The LMAX team were familiar with this approach, and confident that it wouldn't work for LMAX. This assessment was founded in the experiences of Betfair - the parent company who set up LMAX. Betfair is a betting site that allows people to bet on sporting events. It handles very high volumes of traffic with a lot of contention - sports bets tend to burst around particular events. To make this work they have one of the hottest database installations around and have had to do many unnatural acts in order to make it work. Based on this experience they knew how difficult it was to maintain Betfair's performance and were sure that this kind of architecture would not work for the very low latency that a trading site would require. As a result they had to find a different approach.

如今大部分业务系统的核心架构都采用数据库管理会话，LMAX团队非常清楚这一点，并且确定这不适合LMAX。这一判断来自于母公司Betfair的实践经验，Betfair是一个体育竞技博彩网站，针对特定赛事的投注非常集中，网站需要处理海量争相投注的请求。为支撑网站正常运营，他们的数据库成为访问量最高的数据库之一，并且不得不采用很多非常规措施。正是这样的经历让他们明白为了确保Betfair的性能需要付出怎样的代价，这样的架构无法满足金融交易平台低延迟性要求，必须寻找其他方案。

Their initial approach was to follow what so many are saying these days - that to get high performance you need to use explicit concurrency. For this scenario, this means allowing orders to be processed by multiple threads in parallel. However, as is often the case with concurrency, the difficulty comes because these threads have to communicate with each other. Processing an order changes market conditions and these conditions need to be communicated.

他们一开始采用了耳熟能详的并发控制方案，在LMAX的场景下，即允许订单由多线程并行处理。然而问题随之而来，处理任何一笔订单都将影响当前市场状况，而并发随时会发生，线程间必须彼此通讯以基于最新的市场状况处理订单。

The approach they explored early on was ==the Actor model and its cousin SEDA==. The Actor model relies on independent, active objects with their own thread that communicate with each other via queues. Many people find this kind of concurrency model much easier to deal with than trying to do something based on locking primitives.

接下来他们研究了Actor和SEDA模型，Actor模型使用队列让不同线程中相互独立的对象进行通讯，很多人认为这种并发模型比锁机制更易于使用。

The team built a prototype exchange using the actor model and did performance tests on it. What they found was that the processors spent more time managing queues than doing the real logic of the application. Queue access was a bottleneck.

LMAX使用actor模型创建了一个原型进行性能测试，他们发现CPU用来处理队列的时间比执行业务逻辑的时间还多，队列访问成为性能瓶颈。

When pushing performance like this, it starts to become important to take account of the way modern hardware is constructed. The phrase Martin Thompson likes to use is "==mechanical sympathy==". The term comes from race car driving and it reflects the driver having an innate feel for the car, so they are able to feel how to get the best out of it. Many programmers, and I confess I fall into this camp, don't have much mechanical sympathy for how programming interacts with hardware. What's worse is that many programmers think they have mechanical sympathy, but it's built on notions of how hardware used to work that are now many years out of date.

当性能优化到这种程度后，是该关注当前硬件设计理念了，Martin Thompson喜欢使用术语“mechanical sympathy”（译注：意指软硬件协同调优的一种境界）来描述这个境界。这个术语来自赛车驾驶，它反映车手对车的直觉，能够帮助车手实现完美驾驶（译注：人车合一境界）。很多程序员没有领悟到这一境界，不了解底层硬件的工作原理，我承认我就属于这个阵营。更糟糕的是还有很多程序员自认为达到了这一境界，但却是基于已经过时好多年的硬件工作原理。

One of the dominant factors with modern CPUs that affects latency, is how the CPU interacts with memory. These days going to main memory is a very slow operation in CPU-terms. CPUs have multiple levels of cache, each of which of is significantly faster. So to increase speed you want to get your code and data in those caches.

影响现代CPU延时性的关键因素之一是CPU与内存的交互方式，现如今从CPU时钟这个层级来看，内存访问是个非常慢的操作。CPU有多级缓存，它们都比内存快很多，为了提升速度你会希望将代码和数据放到这些缓存中。

At one level, the actor model helps here. You can think of an actor as its own object that clusters code and data, which is a natural unit for caching. But actors need to communicate, which they do through queues - and the LMAX team observed that it's the queues that interfere with caching.

某种程度上actor模型有助于实现这一点，你可以将一个actor看做代码和数据的集合体，自然非常适合缓存。但actor之间需要通讯，使用队列方式实现，LMAX团队发现队列会影响缓存。

The explanation runs like this: in order to put some data on a queue, you need to write to that queue. Similarly, to take data off the queue, you need to write to the queue to perform the removal. This is write contention - more than one client may need to write to the same data structure. ==To deal with the write contention a queue often uses locks. But if a lock is used, that can cause a context switch to the kernel. When this happens the processor involved is likely to lose the data in its caches.==

原因如下：为了将数据放入队列中，需要写队列，同样，从队列读取数据也需要写队列，将读取元素从队列中删除。这是一种写操作冲突，多个客户端需要对同一个数据结构更新写入。为了处理这种冲突，队列通常使用锁，而一旦使用锁就会导致执行流程切换到操作系统内核代码，这种情况下CPU缓存中的数据就失效了。

==The conclusion they came to was that to get the best caching behavior, you need a design that has only one core writing to any memory location[17].== Multiple readers are fine, processors often use special high-speed links between their caches. But queues fail the one-writer principle.

LMAX得出的结论是，为了获得最好的缓存使用效率，需要采用单核完成内存更新操作的设计方式。多客户端读取并没有关系（译注：这里指多线程并发读取，这些线程运行在不同CPU上），因为多CPU的缓存之间通常使用高速连接。而队列违背了单线程更新内存的原则。

This analysis led the LMAX team to a couple of conclusions. Firstly it led to the design of the disruptor, which determinedly follows the single-writer constraint. Secondly it led to idea of exploring the single-threaded business logic approach, asking the question of how fast a single thread can go if it's freed of concurrency management.

这些研究为LMAX带来了一系列成果，首先是完成了disruptor设计，完全符合单线程更新原则，其次是促使LMAX研究单线程业务逻辑处理方案，探究在不采用并发管理的方案下单线程到底能快到什么程度。

The essence of working on a single thread, is to ensure that you have one thread running on one core, the caches warm up, and as much memory access as possible goes to the caches rather than to main memory. This means that both the code and the working set of data needs to be as consistently accessed as possible. Also keeping small objects with code and data together allows them to be swapped between the caches as a unit, simplifying the cache management and again improving performance.

使用单线程实际上是确保了只有一个线程，只会在一个CPU上运行，缓存命中率得以提升，更多的内存访问落在缓存中而不是直接操作主内存。这意味着代码和它操作的数据需要尽可能实现顺序访问，同时代码和数据合在一起的小对象有利于成为内存和缓存的交换单元，简化缓存管理提升性能。

An essential part of the path to the LMAX architecture was the use of performance testing. The consideration and abandonment of an actor-based approach came from building and performance testing a prototype. Similarly much of the steps in improving the performance of the various components were enabled by performance tests. Mechanical sympathy is very valuable - it helps to form hypotheses about what improvements you can make, and guides you to forward steps rather than backward ones - but in the end it's the testing gives you the convincing evidence.

LMAX架构成功的一个关键因素是执行性能测试。研究actor以及放弃这个方案源自于建立原型和执行性能测试，同理，不同组件中大量提升性能的细节操作都是通过性能测试验证的。Mechanical Sympathy很有用，它可以帮助大家提出很多性能优化设想，指引大家朝正确的方向前进而不是倒退（译注：这里作者应该是意指尝试其他流行的并发控制方案，前面提到这些方案违背硬件设计理念，是一种倒退行为），最后测试会给你提供充分的证据。

Performance testing in this style, however, is not a well-understood topic. Regularly the LMAX team stresses that coming up with meaningful performance tests is often harder than developing the production code. Again mechanical sympathy is important to developing the right tests. Testing a low level concurrency component is meaningless unless you take into account the caching behavior of the CPU.

业界对这种程度的性能测试运用不多。LMAX一直强调有效的性能测试通常比开发生产代码更困难，其中Mechanical Sympathy的作用非常重要，如果不管不顾CPU的缓存行为，底层并发组件的测试毫无意义。

One particular lesson is the importance of writing tests against null components to ensure the performance test is fast enough to really measure what real components are doing. Writing fast test code is no easier than writing fast production code and it's too easy to get false results because the test isn't as fast as the component it's trying to measure.

一个典型经验教训是测试过程中需要使用一些测试伪代码替换某些组件，确保测试运行得足够快，充分地测试被测对象。开发高性能的测试伪代码并不比开发生产代码更容易，也很容易由于测试代码的速度还不如被测组件，而得出与事实完全相反的测试结论。

--------------------------------------------------------------------------------
### Should you use this architecture? 

At first glance, this architecture appears to be for a very small niche. After all the driver that led to it was to be able to run lots of complex transactions with very low latency - most applications don't need to run at 6 million TPS.

But the thing that fascinates me about this application, is that they have ended up with a design which removes much of the programming complexity that plagues many software projects. The traditional model of concurrent sessions surrounding a transactional database isn't free of hassles. There's usually a non-trivial effort that goes into the relationship with the database. Object/relational mapping tools can help much of the pain of dealing with a database, but it doesn't deal with it all. Most performance tuning of enterprise applications involves futzing around with SQL.

These days, you can get more main memory into your servers than us old guys could get as disk space. More and more applications are quite capable of putting all their working set in main memory - thus eliminating a source of both complexity and sluggishness. Event Sourcing provides a way to solve the durability problem for an in-memory system, running everything in a single thread solves the concurrency issue. The LMAX experience suggests that as long as you need less than a few million TPS, you'll have enough performance headroom.
There is a considerable overlap here with the growing interest in CQRS. An event sourced, in-memory processor is a natural choice for the command-side of a CQRS system. (Although the LMAX team does not currently use CQRS.)

So what indicates you shouldn't go down this path? This is always a tricky questions for little-known techniques like this, since the profession needs more time to explore its boundaries. A starting point, however, is to think of the characteristics that encourage the architecture.

One characteristic is that this is a connected domain where processing one transaction always has the potential to change how following ones are processed. With transactions that are more independent of each other, there's less need to coordinate, so using separate processors running in parallel becomes more attractive.

LMAX concentrates on figuring the consequences of how events change the world. Many sites are more about taking an existing store of information and rendering various combinations of that information to as many eyeballs as they can find - eg think of any media site. Here the architectural challenge often centers on getting your caches right.

Another characteristic of LMAX is that this is a backend system, so it's reasonable to consider how applicable it would be for something acting in an interactive mode. Increasingly web application are helping us get used to server systems that react to requests, an aspect that does fit in well with this architecture. Where this architecture goes further than most such systems is its absolute use of asynchronous communications, resulting in the changes to the programming model that I outlined earlier.

These changes will take some getting used to for most teams. Most people tend to think of programming in synchronous terms and are not used to dealing with asynchrony. Yet it's long been true that asynchronous communication is an essential tool for responsiveness. It will be interesting to see if the wider use of asynchronous communication in the javascript world, with AJAX and node.js, will encourage more people to investigate this style. The LMAX team found that while it took a bit of time to adjust to asynchronous style, it soon became natural and often easier. In particular error handling was much easier to deal with under this approach.

The LMAX team certainly feels that the days of the coordinating transactional database are numbered. The fact that you can write software more easily using this kind of architecture and that it runs more quickly removes much of the justification for the traditional central database.

For my part, I find this a very exciting story. Much of my goal is to concentrate on software that models complex domains. An architecture like this provides good separation of concerns, allowing people to focus on Domain-Driven Design and keeping much of the platform complexity well separated. The close coupling between domain objects and databases has always been an irritation - approaches like this suggest a way out.

--------------------------------------------------------------------------------
#### Footnotes
1: The Free Lunch is Over

This is the title of a [famous essay](http://www.gotw.ca/publications/concurrency-ddj.htm) by Herb Sutter. He describes the "free lunch" as the ever increasing clock speed of processors that regularly gave us more CPU performance every year. His point was that such clock cycle increases were no longer going to happen, instead performance increases would come in terms of multiple cores. But to take advantage of multiple cores, you need software that is capable of working concurrently - so without a shift in programming style people would no longer get the performance lunch for free.

2: I shall remain silent on what I think about the value of this innovation

3: User Base

All trading systems need low latency, since one trade can affect later trades and there's a lot of competition based on rapid reaction. Most trading platforms are for professionals - banks, brokers, etc - and typically have hundreds of users. A retail system has the potential for many more users, Betfair has millions of users and LMAX is designed for that scale. (The LMAX team isn't allowed to disclose its actual volumes.)
As it turns out, although a retail system has a lot of users, most of the activity in comes from market makers. During volatile periods an instrument can get hundreds of updates per second, with unusual micro-bursts of hundreds of transactions within a single microsecond.

4: Hardware

The 6 million TPS benchmark was measured on a 3Ghz dual-socket quad-core Nehalem based Dell server with 32GB RAM.

5: The team does not use the name Business Logic Processor, in fact they have no name for that component, just referring to it as the business logic or core services. I've given it a name to make it easier to talk about in this article.

6: Currently LMAX runs two Business Logic Processors in its main data center and a third at a disaster recovery site. All three process input events.

7: What's in a transaction

When people talk about transaction timing, one of the problems is what exactly is in a transaction. In some cases it's little more than inserting a new record in a database. LMAX's transactions are reasonably complex, more complex than a typical retail sale.

Placing an order in an exchange involves:

- checking the target market is open to take orders
- checking the order is valid for that market
- choosing the right matching policy for the type of order
- sequencing the order so that each order is matched at the best possible price and matched with the right liquidity
- creating and publicizing the trades made as a consequence of the match
- updating prices based on the new trades

8: At this scale of latency, you have to be aware of the garbage collector. For almost all systems these days, a modern GC compaction isn't going to have any noticeable effect on performance. However when you are trying to process millions of transactions per second with minimum jitter, a GC pause becomes a problem. The thing to remember is that short lived objects are ok, as they get collected quickly. So are objects that are permanent, since they will live for ever. The problematic objects are those that will get promoted to an older generation, but will eventually die. As this fragments the older generation region, it will trigger the compaction.

9: I rarely think about which collection implementation to use. This is perfectly reasonable when you're not in performance critical code. Different contexts suggest different behavior.

10: An interesting side-note. While the LMAX team shares much of the current interest in functional programming, they believe that the OO approach provides a better approach for this kind of problem. They've noticed that as they work to write faster code, they move away from a functional style towards OO style. Partly this because of the copying of data that functional styles require to maintain immutability. But it's also because objects provide a better model of a complex domain with a richer choice of data structures.

11: The name "disruptor" was inspired from a couple of sources. One is the the fact that the LMAX team sees this component as something that disrupts current thinking on concurrency. The other is a response to the fact that Java is introducing a phaser, so it's natural to include disruptors too.

12: It would be possible to journal the output events too. This would have the advantage of not needing to recalculate them should they need to be replayed for downstream services. In practice, however, this isn't worthwhile. The business logic is deterministic and very fast, so there's no gain from storing the results.

13: Although it does need to use CAS instructions in this case. See the [disruptor technical paper](http://disruptor.googlecode.com/files/Disruptor-1.0.pdf) for more information.

14: This does mean that if they process a billion transactions per second the counter will wrap in 292 years, causing some hell to break loose. They have decided that fixing this is not a high priority.

15: SSDs are better at random access, but a disk-like IO system slows them down.

16: Another complication when writing fields is you have to ensure that any fields being written to are separated into different cache lines.

17: Ensuring a single writer to a memory location

A complication in following the single-writer principle is that processors don't grab memory one location at a time. Rather they sweep up multiple contiguous locations, called a cache line, into cache in one go. Accessing memory in cache line chunks is obviously more efficient, but also means that you have to ensure you don't have locations within that cache line that are written by different cores. So, for example, the Disruptor's sequence counter are padded to ensure they appear in separate cache lines.

#### Acknowledgments
Financial institutions are usually secretive with their technical work, usually with little reason. This is a problem as it hampers the ability for the profession to learn from experience. So I'm especially thankful for LMAX's openness in discussing their experiences - both with this article and in their other material.

The main creators of the Disruptor are Martin Thompson, Mike Barker, and Dave Farley

Martin Thompson and Dave Farley gave me a detailed walk-through of the LMAX architecture that served as the basis for this article. They also responded swiftly to email questions to improve my early drafts.
Concurrent programming is a tricky field that requires lots of attention to be competent at - and I have not put that effort in. As a result I'm entirely dependent upon others for understanding on concurrency and am thankful for their patient advice.

#### Further Reading
If you'd prefer a video description of the LMAX architecture from LMAX team members, your best bet is the [QCon presentation](http://www.infoq.com/presentations/LMAX) given in San Francisco 2010 by Martin Thompson and Michael Barker.

The [source code for the Disruptor](http://code.google.com/p/disruptor/) is available as open source. There is also a good technical paper (pdf) that goes into more depth as well as a collection of [blogs and articles](http://code.google.com/p/disruptor/wiki/BlogsAndArticles) on it.

Various members of the LMAX team have their own blogs: [Martin Thompson](http://mechanical-sympathy.blogspot.com/), [Michael Barker](http://bad-concurrency.blogspot.com/), and [Trisha Gee](http://mechanitis.blogspot.com/).