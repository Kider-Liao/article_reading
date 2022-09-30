## Practical Scheduling for Real-World Serverless Computing



#### Abstract

serverless因其易用性和高性价比而不断高速发展，但是serverless中的重要部分函数调度，没有被予以重视。本文旨在提出一个scheduler，以迎合在实际应用中观察到的serverless函数的独特特性。作者从三个维度对调度策略进行了分类，并利用Azure function发布的现实函数的14天运行情况，分析依据函数特性的调度策略空间，总结到一些常用的策略(如随机负载均衡、延迟绑定等)是次优的。依据这些发现以及函数的三个主要特性，本文设计出Hermes用于serverless函数调度。为了避免线头阻塞(队列前端的函数因资源需求高或运行时间长等原因，引起后面的函数无法被调用)，Hermes采用了提前绑定和处理器共享的方法进行调度；Hermes采用了混合的负载均衡策略，在低负载时整合函数，在高负载时采用最小负载均衡(least-loaded balancing)来维持高性能；Hermes对负载和局部性进行感知，减少冷启动的次数



#### Introduction

FaaS因其易用性等特点越来越受人们欢迎。用户只需要用高级语言撰写函数、定义触发事件和端点并且为资源使用进行付费，即可部署出对应的服务。对于资源提供、函数扩展、函数调度等基层工作，都交由专业的计算平台进行，而不需要用户关注。serverless函数的易用性、弹性和经济效益，使得越来越多的负载迁移到了FaaS平台上，形成了多种多样的serverless应用，如简单的事件处理应用、机器学习、编译等

目前，已有许多研究对serverless平台进行了不同层面的优化，如降低启动延时、记录内存踪迹、改善函数间通信等。而本文关注于serverless函数的调度问题(FaaS的关键路径)，以优化FaaS的性能。根据对Azure Function发布的现实运行情况进行分析，发现serverless函数具有运行时间变化范围大、爆发式增长、倾斜调用等特点。根据函数特性以及过去的工作，作者采用规则性(principled)方法，构建出调度策略的一个分类方式

通过利用模拟，作者对策略空间进行了分析，并总结出常用的几种策略对于serverless的特性而言是次优的：

1. 即使是最理想的Late Binding方法对于运行时间变化大的负载而言也是次优的，因为短的函数调用会因等待长的调用完成运行导致被阻塞在调度器队列。如果我们能够采用processor-sharing的策略提前处理函数调用(采用early binding策略在调用进入系统时就进行调度)，可以减轻线头阻塞带来的问题
2. 无论是随机负载均衡还是locality-based负载均衡，在高负载下都不够高效，因此有必要采用least-loaded 负载均衡策略来在高负载下降低性能损失
3. 在低负载的情况下，least-loaded负载均衡策略因将函数调用广泛地分布到多个服务器上，增加了不必要的服务器启动和冷启动，导致了低的资源利用率且增加了函数运行时间

基于这些发现，作者开发出采用混合负载均衡策略和位置感知的混合调度器Hermes。Hermes在低负载时通过将调用进行整合，实现高的资源效率；在高负载时转用least-loaded balancing策略，降低排队时延。此外，Hermes同时是感知负载和位置的，在不牺牲资源利用率和负载均衡的情况下考虑上locality。Hermes同时采用了early binding和processor-sharing策略来避免线头阻塞

**load-aware和locality-aware指的是什么**

作者将Hermes部署在了Apache OpenWhisk(开源serverless平台)，并利用依据Azure和Twitter的生产数据生成的负载以及具有与serverless新兴用例相似特征的工作负载，对Hermes的效率进行评估。分析将关注于函数的slowdown，即函数完成时间除以函数运行时间。

本文的主要工作在于

1. 对serverless函数的调度策略进行了分类，分为从三个基本维度进行决策
2. 依据serverless函数的现实特性，证明出常用的一些调度技术是次最优的
3. 利用如上得到的结论，设计出load和locality感知的调度器Hermes，结合early-binding、异构负载均衡和processor sharing等技术
4. 将Hermes部署在Apache OpenWhisk平台，证明了Hermes在性能和资源利用率上的优化



#### Background

- Life-cycle of a Serverless Function(serverless函数的调用周期)

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220915162305330.png" alt="image-20220915162305330" style="zoom:50%;" />

  1. 函数可以通过多种方式进行触发，如HTTP请求、定时事件等。函数调用将会被发送给Controller，由Controller终止SSL连接，进行访问控制和限制请求速率
  2. Controller可能能够将函数任务调度到一个正在运行对应的函数代码的worker上，此时只需要很少的初始化开销等即可运行函数任务。这称为"warm start"
  3. 如果当前没有正在运行对应代码的worker，或所有这类worker已经满负载了，则会将函数任务分配给其他合适的未运行函数代码的worker
  4. 被分配的worker从镜像仓库中拉取相应的函数代码，启动新的executor(容器？)来运行代码，处理函数任务。该过程称为"cold start"

  除上述架构外，还可能有其他变体架构，如增加专门的负载均衡器，增加持久化信息存储等。但通常情况下，我们认为serverless任务处理由两部分组成：Controller负载函数任务的调度，Worker负责将资源合理分配给对应的executor用于运行函数任务

- Characteristics of Serverless Functions(serverless函数的特性)

  根据Microsoft发布的Azure Function的14天函数调用追踪，作者总结出serverless函数具有以下特性

  1. Short and Highly-variable Execution Times

     函数运行时间的分布从几百毫秒到分钟以上，显示出了极端的变化性(可以通过重尾对数正态分布进行建模)。在Azure Function的数据追踪中，50%的函数运行时间在600ms以下，但99%的运行时间则到了超过140秒

     **即使是单个函数的运行时间，也具有高可变性**

  2. Skewed Function Popularity

     数据说明对0.6%的函数的调用占用了所有调用的90%，显示出函数关注度的倾斜

  3. Burstiness

     单独函数的请求到达率具有突发性，平均突发指数为-0.26。整体的调用请求呈现出昼夜模式，与大规模云系统类似

- Mismatch between Characteristics and Scheduling of Serverless Functions

  目前，许多工作都关注于减少函数冷启动的开销和频率。专用的虚拟机、轻量级的操作系统和语言抽象、快照snapshot等技术有效地减少函数冷启动的开销，而负载预测、缓存函数数据等操作有效减少了冷启动的频率

  然而，serverless函数的调度策略的关注度逐渐减少。调度作为提升云系统性能的重要因素，采用良好的调度方式能够大量地提升系统的性能。然而，目前已有的云系统调度器采用的调度策略与serverless函数的特性并不适配

  1. Existing Schedulers for Serverless functions

     Apache OpenWhisk采用的默认调度器采用纯粹的locality-based均衡策略，将某个函数随机分配到一个合适的worker后，所有对该函数的调用都被发送到该worker进行处理。该方法没考虑调用请求的数量，对highly-skewed workloads不适用

     **locality-based策略即是尽可能将请求调度到同一位置或少量的服务器中，以利用数据局部性或提高资源利用率，且减少冷启动发生**

  2. Kubernetes-based Frameworks

     部分serverless平台如OpenFaaS、Kubeless等，基于Kubernetes进行构建，并将serverless函数视为常规的workload，使用自动扩展等策略。在worker的CPU或GPU等资源利用率超过某个阈值时，系统启动更多的worker处理任务；低于某个阈值时，系统将任务整合到少量的worker上处理，提高资源利用率。然而，这种粗粒度的调度策略有滞后性，因冷启动等原因在突发性的函数请求下会引起部分函数有很长的运行时间，出现重尾的现象

  3. General Task Schedulers

     部分进行科学计算和分析的框架关注于提升数据的局部性，它们的调度器通过batching任务或将任务进行迁移使得具有数据联系的任务可以在同个机器上运行。然而，因冷启动和实时性，迁移对于serverless function而言是不实际的，因为会给函数带来更大的开销

     **这里说明了对locality进行考虑的意义，即具有数据联系的任务整合到同样的机器上以降低通信开销**

     Sparrow和Pigeon两种热门的框架调度器采用了late-binding策略，即在存在可用worker时再将任务调度到worker上运行。这种方法使得短的函数任务需要等待相对较长的时间，等待长任务完成，造成slowdown。因此，这种调度策略不适用于highly-variable的负载

     Hawk和Eagle两种调度器对Sparrow进行了改进，通过区分短任务和长任务的调度，并预留一部分节点来供短任务运行，从而降低短任务的排队时间。然而，这种方案需要预测任务的运行时间，并且在分布式调度器不能做出正确调度决策时进行补偿，使其不适用于运行时间不规律和存在冷启动问题的serverless函数负载

     **这里提到短任务采用分布式调度，长任务采用集中调度，分布式调度是怎么进行的**

  4. Cluster Schedulers

     对于一些大型集群的调度器(如Mesos、Yarn)，它们支持不同的框架自行请求和接收集群资源，从而将资源的分配和调度任务下放给框架的调度器。serverless框架可以利用这些调度器提供的资源管理器进行资源申请，但是仍然需要自行决定何时需要申请资源以及调度函数调用(这类调度器与函数调度无关)

     其它的集群调度器往往关注于长任务的调度，允许较大的调度时延，不适用于运行时间较短的serverless函数



#### Taxonomy of Scheduling Policies

- Scheduling Analysis

  在设计serverless函数的调度器之前，我们应该明确函数调用周期中的三个重要问题的解决方案

  - 函数调用在什么时候被调度到一个Worker

  - 函数调用应调度给哪个Worker

  - Worker应该采用什么内部调度策略

  以下提出了一个分类方法，描述了上述问题的不同答案所产生的策略(分类即将每个问题的解决方案分为不同的解决方向，生成测策略即为这三个问题的解决方向的组合)

  1. 函数调用在什么时候被调度到一个Worker

     该问题的解决方案主要分为两种方法：early-binding和late-binding。early-binding指的是一旦函数调用被发送给Controller，马上将会被分派到某个Worker上运行，而不会在Controller中排队等待。early-binding通常适用于函数排队消耗过多Controller内存、Controller-Worker通信时延较长或是Scheduler没有系统的全局视图(无法感知Worker的资源可用性等)的系统；late-binding指的是函数调用到达Controller后，它首先在Controller中进行排队，直至有Worker能够使它立即运行时才被分配到Worker上。late-binding不会出现负载不均衡的问题，适用于能容忍一定排队时延的系统

     **late-binding不会与其他策略共用**

  2. 函数调用应调度给哪个Worker

     除了分配的时间，我们应当决定函数调用被分配的位置，即采用什么负载均衡策略决定调用的走向。OpenWhisk采用的方法是locality-based的，它通过尽可能让同个函数的调用集中在少量的服务器中运行，从而减少冷启动的发生；另外一个好的负载均衡策略是随机负载均衡，随机地将函数分配给不同的worker运行；选择负载最低的worker来运行任务是许多任务调度器常用的负载均衡方案，尽可能平衡多个worker之间的负载情况

  3. Worker应该采用什么内部调度策略

     如果采用early-binding策略，那么将会有一部分函数调用在worker上进行排队，此时worker需要采用合适的内部调度策略进行资源分配。Shortest-Remaining-Processing-Time(SRPT)策略能够最小化平均的响应时间，在任务运行时间可知的环境下非常适用(然而，serverless函数的运行时间通常是难以估计的，这种方法不适用)；Processor Sharing(PS)策略通过令每个任务能共享处理器的能力，给予公平的运行资源。Linux系统采用的Completely Fair Scheduler策略类似于PS，并被用于实际系统中；First-Come-First-Serve(FCFS)也是广泛被应用的策略，按任务到达的时间顺序进行运行

  我们将三个问题的解决方式用三个参数T/LB/S表示，从而形式化表示出三个问题的解决方案的组合(例如，OpenWhisk采用了early-binding、locality-based balancing和processor-sharing策略，那么它的表示为E/LOC/PS)

  1. T：表示了采用late-binding(L)方式还是early-binding(R)方式
  2. LB：表示了负载均衡策略，分为locality-based balancing(LOC)策略、least-loaded balancing(LL)策略和random balancing(R)策略
  3. S：表示了worker内部调度策略，分为Processor-Sharing(PS)策略和First-Come-First-Serve (FCFS)策略

- Simulation Setup

  本小节说明了仿真的一些设置，如worker的处理能力，函数调用的运行时间分布，函数调用的到达速率等，用于为接下来不同的策略组合的性能检测做好准备。仿真没有计入函数的冷启动时间，将所有函数调用作为类似的任务进行仿真

- Pruning the Policy Space

  首先，我们对不同组合的设计方法进行描述。组合方式共分为以下七种

  1. L/\*/\*：通过采用Late-binding的调度策略，在Controller处以FCFS的方式对函数调用进行调度。只有存在某个worker有可用资源运行函数时，函数才被分配到该worker上运行。Late-binding策略不关心负载均衡以及worker内部的调度策略，因为它只在worker可用时被分配，且一旦被分配即可马上运行
  2. E/LL/FCFS：Controller将函数调用分派到队列最短的worker上，并在worker上采用FCFS的方式运行函数
  3. E/LL/PS：worker上采用时分处理器的方式运行函数
  4. E/LOC/FCFS：Controller将同个函数的调用随机分配到某个运行FCFS策略的worker上(每个函数只位于一个worker上运行)
  5. E/LOC/PS：Controller将同个函数的调用随机分配到某个运行PS策略的worker上
  6. E/R/FCFS：Controller随机将函数的调用分配到worker上，worker采用FCFS策略
  7. E/R/PS：Controller随机将函数的调用分配到worker上，worker采用PS策略

  在仿真中，作者通过衡量不同策略下的slowdown情况，即请求完成时间/请求运行时间，并分析负载slowdown的99%分位点，从而正确地反映出系统的病态实例和异常情况。获得的结果如下图
  
  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220917102327541.png" alt="image-20220917102327541" style="zoom:50%;" />
  
  根据上述结果，我们能够获得如下发现：
  
  1. PS内部调度策略相较于FCFS调度策略，具有显著的优势。对于运行时间light-tailed的任务而言，由于任务的运行时间相近，根据FCFS的调度策略能带来不错的效果。然而对于运行时间heavy-tailed的函数而言，采用FCFS的方式将使运行时间短的函数花费大量的时间等待长函数运行完毕，使slowdown值大幅增加。Late-binding也会遇到相应的问题，因此它的运行与E/LL/FCFS的情况相近(Late-binding等待资源可用，相当于在当前时刻该宿主机的负载是最低的)。
  
     根据上述发现，我们可以得出的结论是：由于head-of-line blocking的存在，late-binding和FCFS的策略都会导致短函数等待较长时间，难以处理运行时间高可变的serverless函数
  
  2. 观察random和locality-based负载均衡策略，发现在采用E/FCFS的情况下，它们在负载低于0.55时就已经出现了严重的heavy-tailed现象。random的负载均衡策略在这种运行时间高可变的情况下不能有良好的表现，因为它将长短函数同样地进行随机分配，导致了worker之间负载不均衡；locality-based倾向于将同个函数的调用调度到同个宿主机上，由于serverless函数会存在部分函数调用数量远超其他函数的情况，locality-based的调度策略使得运行热门函数的worker马上就overloaded，导致负载指数在0.55时就已经出现heavy-tailed现象。相比之下，least-loaded负载均衡策略在serverless环境下，具有更好的表现
  
  3. 目前考虑的两种调度策略，PS和FCFS，都没有应用上函数运行时间方面的信息，使得这两种策略能够在任何环境中适用。然而根据过往经验，能够利用函数过去的运行信息的调度策略，通常能有更好的调度决策。为了验证运行时间的信息能否有益于serverless scheduler，作者对比了Shortest-Remaining-Processing-Time(完美知道运行时间)和PS策略的平均slowdown和99%分位点slowdown
  
     对比两种策略的平均slowdown，可以看到SRPT策略能够略优于PS策略，与预期相符。然而，对比99%分位点的slowdown，发现采用SRPT策略比PS出现了严重的heavy-tailed情况，说明其有部分函数调用的延时过大，造成了严重的slowdown。出现这个结果的原因是SRPT倾向于运行时间段的函数调用，导致长调用被长时间延期，引起了饥饿的现象。SRPT的效果在实际中可能更差，因为我们不可能完全预测到函数的运行时间，这可能导致更坏的决策
  
     综上，即使应用运行时间信息，在serverless环境下其heavy-tailed现象还是比PS严重。因此，我们只考虑execution-time-agnostic的调度策略是可行的
  
     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220917113829875.png" alt="image-20220917113829875" style="zoom:50%;" />
  
  4. 对于大规模的集群，虽然late-binding导致的线头阻塞因集群增加而减少，采用E/LL/PS的策略组合依然是最优的(其他两种策略依旧没有较好的表现，高负载下E/LL/PS优于late-binding)
  
     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220917115521520.png" alt="image-20220917115521520" style="zoom:50%;" />



#### Hermes Design

基于上述发现，作者设计了Hermes。Hermes结合early-binding和processor sharing技术，并应用异构的负载均衡策略，迎合serverless函数的特性。此外，Hermes通过记录warm-containers的数量，减少冷启动的发生

- Why Hybrid Load Balancing is Necessary(为什么要采用混合调度)

  根据仿真，我们发现E/LL/PS的调度组合能够拥有最好的性能表现，然而，这种调度策略面临如下问题

  1. Low Resource Efficiency

     least-loaded负载均衡策略倾向于将函数调用平均分配到所有的服务器上运行，然而在低负载的情况下，这种方式会引起服务器的低利用率，增加了开销

  2. Increased Cold Starts

     对于serverless调度器而言，function code locality是一个重要的考量因素，因为函数的冷启动会引起很高的函数调用时延。然而，采用least-loaded负载均衡策略的调度器倾向于将函数调用分配到所有的服务器上运行，在低负载时造成了更多冷启动的发生(没有warm container的服务器往往负载更低，函数调用会被调度到这样的服务器上)

- Hermes

  Hermes采用了early-binding和processor-sharing技术，在不需要了解函数的运行状况的情况下，减少head-of-line blocking引发的高等待延迟。在负载均衡方面，为了避免least-loaded balancing缺陷，Hermes采用了如下的混合负载均衡策略，来达到在低负载时能整合函数调用，降低开销和冷启动，在高负载时能够均衡分配函数调用降低slowdown的目的

  1. 低负载情况

     在低负载情况下，Hermes尽可能将所有的函数调用集中在少部分的服务器上，从而减少服务器的启动。Hermes首先随机地将函数调用分配到某个服务器上，直至该服务器没有可用的运行实例时，再启动另一个服务器运行函数。如果正在运行的服务器出现空闲的运行实例，则首要选择是将这些服务器填充满，再去考虑启动新的服务器运行。这种调度策略在低负载下，不会出现排队问题，且减少了服务器的启动，因此能够有良好的表现

     **在这基础上，应用一些对运行时间的推测，甚至可以再度减少服务器启动？(例如排队不会超出SLO时，则继续将函数调度到同个服务器)**

  2. 高负载情况

     当所有的服务器都已经没有空闲实例时，Hermes转为least-loaded负载均衡策略，即将函数调用分配到队列最短的服务器上，这通过与服务器保持通信并同步状态的方法来实现

  3. 考虑函数局部性

     混合的负载均衡策略使Hermes在一定程度上减少了冷启动的发生而通过考虑函数代码的locality，我们可以进一步减少冷启动的情况。在低负载模式下，Hermes首先查看非空的服务器(已有实例运行)中是否存在能够直接运行函数调用的实例(该实例已运行对应函数调用的代码)，如果有则优先将函数调用分配到该服务器上；如果没有，Hermes仍旧优先选择填满已运行的服务器(即使这个服务器没有相应的实例)，而不是将函数运行在另一个空闲的有运行函数代码的实例的服务器上(优先选择整合到少量的服务器上)。在高负载模式下，如果服务器的队列相同，则优先将函数调用调度到有相应实例的服务器上

- Scalability

  Hermes的设计可以应用在分布式的环境中，且能与众多常见的分布式调度器技术兼容，具有良好的可扩展性。例如，在sharded setup中，每个Hermes实例都能够自行在自己的worker子集中选择合适的worker运行函数调用；在power-of-k choices架构中，当函数调用被发送给某个Hermes实例时，改进后的Hermes能够从大型集群中取样K个worker，并在这k个worker上应用相应的调度策略选择合适的worker



#### Implementation

Hermes部署在了开源serverless平台Apache OpenWhisk，检测Hermes的性能与效率。本节首先介绍了OpenWhisk的一些技术细节，并介绍在Evaluation部分与Hermes对比的调度器baseline。最后，本节介绍了Hermes的具体实现细节

- OpenWhisk Architecture

  函数调用在到达Controller之前需要经过一个反向代理，该代理持有一个面向公众的HTTP端点(调用可以通过HTTP进行)，并负责终止调用的SSL连接。Controller利用独立的持久化存储CouchDB来存储系统状态，包括函数的列表和函数代码(Worker从CouchDB接收函数代码)。Controller持有Load Balancer负责将函数调用调度给Worker。Controller采用一个可靠的消息总线(message bus)来将函数调用发送给Worker运行

  OpenWhisk的冷启动主要存在两种情况：如果当前Worker因运行过对应的函数而缓存了函数代码或执行文件，那么Worker只需要重新启动一个容器并将代码注入容器即可；如果Worker没有缓存代码或可执行文件，那么Worker需要从CouchDB拉取函数代码，启动新的容器，并将代码注入容器

- Baseline Schedulers

  1. OpenWhisk调度器：OpenWhisk的默认调度器采用的是E/LOC/PS的策略。每个Worker应用部分固定的内存来容纳函数调用，并在PS策略下分片执行函数调用。OpernWhisk的调度器，为每个函数指定一个Worker，用哈希表记录映射，并将该函数的所有调用引向该Worker。当这个Worker容量不足时，将会重新随机选择一个Worker进行分配(所有Worker满时可能会使用额外内存)。该调度器由于容易使某部分Worker很快出现overloaded的情况，造成较大的时延，使得调度效果不太理想
  2. Late-binding调度器：虽然比起early-binding调度器，late-binding调度器在面对运行时间高可变的serverless函数时表现不佳，但是由于late-binding调度器在众多框架中比较常见，本实验仍旧将其作为baseline进行比较。late-binding调度策略下，任何函数调用只有在某个Worker存在空闲的core时，才将其调用到Worker上(应当是Worker每个容器被分配一个core)
  3. least-loaded调度器：该调度器采用E/LL/PS的策略，但是相比起Hermes，它没有在低负载时考虑locality。本实验期望证明Hermes在考虑到locality的情况下，能比该调度器有更好的调度效果

- Implementation Details

  在各种调度器中，采用数组来存储各个worker的队列长度，而对于Hermes，则增加一个哈希表来映射各个worker的warm-container情况。这些开销是非常小的，即使在大规模集群中也只需要少量MB来存储。由于OpenWhisk中所有的函数调用和完成都需要经过Controller，因此对worker队列长度和warm-container的记录是非常简单的。此外，Hermes的locality和least-loaded的状态转换只需要根据worker的队列情况(负载的到达率等)通过if语句转换即可



#### Evaluation

- 问题描述

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220918162137786.png" alt="image-20220918162137786" style="zoom:50%;" />

- 调度器baseline：OpenWhisk调度器、Late-binding调度器、least-loaded调度器

- 负载workload

  实验采用了五种workload，两个依照的是实际的生产负载，三个测试新兴的serverless用例以及极端的情况。

  首先，实验使用从Azure production trace中直接衍生出的负载MS Trace。作者从trace种选择50个不同的函数，并根据不同的负载级别将整体函数调用数量进行伸缩。作者随机选择其中一个函数，并使该函数的调用具有高热度，而其他函数只有中等热度，来反映函数热度的倾斜。作者采用Javascript来实现对应的函数(不同语言来实现函数没有太大开销区别)。对于每个函数，其函数调用的运行时间分布都服从μ=-0.38，δ=2.36的对数正态分布

  为了确保Hermes能够适应其它的负载情况，检验其健壮性，实验还设置了对如下情况考虑的负载

  - Arrival Pattern

    为了避免对Azure Function中缺失的数据做出低级的假设，如每分钟函数的到达数量等，我们使用开环泊松分布(每个时间段，事件的发生次数是独立且满足泊松分布的)来对函数的到达速率进行建模。为了显示出函数热度的倾斜性，作者令某个函数占用90%的调用，而其他函数平分10%的调用，运行时间分布与MS Trace相同。通过该方法设计出的trace称为MS Representative

    **MS Representative与MS Trace相比，变化了什么，是在到达速率和调用倾斜上都有变化吗**

  - Skew

    为了显示极端倾斜情况下Hermes的运行情况，Single-Function负载只采用单一函数，所有的函数调用都使用该函数，代表分析型应用程序的特征(许多函数被反复调用做同类的任务)；Multiple-Functions-Balanced令50种函数均分所有的函数调用，代表了零倾斜的极端情况。这两种负载的到达速率和运行时间分布都与MS Representative相同

  - Execution Time Distribution

    为了检测Hermes在同构的运行时间下能否有好的表现，作者设计出Homogeneous-Execution-Times，在到达速率、倾斜情况和平均运行时间与MS Trace相同的情况下，运行时间采用light-weighted的指数分布，减少了部分函数运行时间过长的情况

- 实验结果

  1. 性能分析

     对比各个scheduler在不同负载下slowdown的99分位点，得到的结论是：对于MS Trace, MS Representative, 和Single-Function负载，Late-binding和OpenWhisk调度器能支持的负载数量低于其他两个调度器，在较低负载时它们的99分位点就开始飙升；相比于纯Least-Loaded均衡器，Hermes在较低负载下有更好的表现；对于Multiple-Functions-Balanced负载，其函数调用没有热度倾斜。OpenWhisk调度器因能将函数调度进行聚合减少冷启动，在低负载下比其他调度器slowdown更低

     以上结果说明Hermes采用的Least-Loaded负载均衡策略能支持更高的负载，而Hermes因能在低负载下考虑locality，相比纯LL策略有更好的表现。

  2. 冷启动分析

     对比各个调度器在不同负载下的冷启动比例，得到的结论是：Hermes在不同负载下都有较低的冷启动比例，因为它记录了warm-container并尽可能将负载引导到warm-container上运行；Least-Loaded策略在低负载下会引发大量的冷启动，因为它将负载分配到了所有的服务器上，使所有服务器都要运行不同函数的容器；OpenWhisk调度器Single-Function, MS Trace, 和MS Representative负载下引发了大量的冷启动，因为它很快地造成了部分服务器overloaded，但它在Multiple-Functions-Balanced负载下冷启动比例很低，因为它尽可能将负载分配给了相同的worker

     **为什么overloaded引起了更多的冷启动，将同个函数集中应当能减少冷启动？只是overloaded造成了排队时延增加使得性能不好吧**

  3. 资源消耗分析

     从使用的core方面来看，不同调度器在负载下应用到的core的数量类似；但是从使用的服务器数量来看， Least-Loaded的调度器在低负载时就已经用到所有的服务器，其他方案应用的资源情况类似(OpenWhisk在低负载时应用的服务器数量稍微多一些，因为它为每个函数随机分配服务器) 

  4. 健壮性分析

     应用Homogeneous-Execution-Times负载，对比在不同运行时间分布的负载情况下各个调度器的slowdown中位数和99分位点，发现OpenWhisk因其倾斜性引发了较大的slowdown，采用Hermes和LL调度器时在任何负载情况下都没有严重的slowdown

  
