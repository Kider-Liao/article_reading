## Kraken  Adaptive Container Provisioning for Deploying Dynamic DAGs in Serverless Platforms

#### Abstract

微服务的流行引发了基于在线云服务的应用的兴起，这些应用往往有数十至数百个微服务组成，其工作流程可以建模为DAG。这些在线应用大多都是面向用户的，具有严格的SLO需求。而serverless函数因其短的资源配置时间和高弹性，成为了这些应用微服务的重要载体。然而，已有的serverless供应商往往不在意应用的工作流特性，导致在大多数情况下，他们使用了过多的容器来运行应用(对于动态DAG的应用问题加剧，因为没有提前知道函数的调用链)。因此，本文提出资源管理框架Kraken，通过感知应用工作流最小化提供给应用的容器数量，同时保障SLO能被满足。Kraken被部署在了OpenFaaS，并在多节点Kubernetes集群上进行性能评估。使用实际的DeathStarbench工作流对Kraken进行测试，发现相较于最先进的serverless资源管理器，Kraken能提高容器利用率并节约很多的容器供应



#### Introduction

- 问题背景

  由于微服务架构的易于开发和可扩展性，目前云应用将微服务架构作为主要的开发架构，由成十上百的微服务构成其业务逻辑、功能逻辑等。这些云应用通常是面向用户的，具有严格的延迟需求，因此选择合适的基础设施(容器、虚拟机)运行应用至关重要，基础设施的启动、配置延时往往影响到了应用的响应时间。然而，FaaS能够降低用户在配置基础资源上的开销，并提供良好的可扩展性和快速的函数启动，使其逐渐成为面向用户的云应用的重要开发平台

- 面临的挑战

  1. 应用的微服务在FaaS平台上需要被部署为函数function，且根据需求将函数利用工具组合成链，从而能够表示整个应用逻辑，即将应用通过DAG表示
  2. 应用DAG应当被转换为有限状态机，显式的呈现出函数之间的状态依赖和转换，并依据提供商的接口将函数工作流交由提供商管理
  3. 应用DAG中的分支结构，往往造成了状态转换的不确定性，即不同的函数调用会引发不同的工作流，给提供商的容器供应和调度带来了挑战

  **这里前两个挑战主要是用户设计方面的挑战，第三条是供应商资源配置方面的挑战，本文旨在解决供应商资源配置上的问题；除分支结构外，不同函数的运行时间差异等，同样也是供应商资源配置上的挑战(时间短的容器可以比时间长的少)**

- 已有处理方案

  1. 目前主流的serverless平台假设DAG是静态的，即一次工作流调用中会使用到所有的函数，因此根据工作负载情况为所有函数配置相同数量的容器。这往往导致了容器的过量配置
  2. 对于动态的DAG，一个请求调用往往只涉及到部分的函数，使得供应商应当根据使用情况为不同函数分配不同数量的容器。近期的一些框架如Xanadu，通过预测一个请求执行最有可能涉及的函数，从而维护一条单一的函数链。然而，这种方法没能将容器分配到所有的函数中，往往造成了under-provisioning的问题，引发了性能降低和响应时间增加的问题

- 解决方案

  本文作者提出了面向动态DAG的工作流感知型资源管理框架Kraken，在保持应用SLO的前提下最小化容器的使用数量。Kraken两个主要的构件为

  1. Kraken应用了Proactive Weighted Scaler(PWS)来提前为不同的函数分配对应的容器。容器分配的数量依据于每个函数在工作流中的权重weight以及负载到达预测模型request arrival estimation model对接下来请求到达情况的预测。每个函数的权重取决于与它关联的工作流数量以及其后继函数的数量
  2. 除了PWS外，Kraken还应用了Reactive Scaler(RS)来对函数容器进行伸缩，在PWS不能正确分配容器时弥补失误分配

- 运行效果

  Kraken被应用在开源serverless框架OpenFaaS上运行和评估。其能够以99.97%的概率保证SLO需求被满足的同时，相比最先进的serverless调度器平均减少了76%的容器分配(提升了4×的利用率)，并降低了48%的集群资源消耗



#### Background and Motivation

- serverless函数链简介(函数工作流)

  许多应用可以被建模为函数链(function chains)并有严格的时延SLO需求。一条serverless函数链通常由多个独立的serverless函数组合而成，这些函数通过同步和通信技术进行联系，从而对外提供应用的某项业务功能。应用可以由多条函数链组成，实现完整的应用功能。除了特殊情况外，即应用的函数链存在环，应用可以被描述为有向无环图DAG，由点(函数)和有向边(数据同步或通信)组成。一条工作流workflow或路径path被定义为应用DAG中从一个起始节点到一个终止结点的路径，表示为一次请求调用(工作流调用)。根据应用特性，其DAG可以是静态或是动态的

  1. 静态DAG

     对于静态DAG，工作流由用户预先定义，交由serverless提供商进行编排。在静态DAG下，所有调用都对应了唯一一条确定的路径，使得每条调用请求涉及的函数是确定的(静态DAG被描述为一条单一路径)。在这种情况下，由于我们提前预知调用涉及到的函数，可以很容易地为函数分配运行容器。这类应用被定义为SDA(static DAG application)

  2. 动态DAG

     对于动态DAG，存在一些函数结点会根据自己的输入来决定调用哪个后继函数，即存在分支节点，导致了一次调用请求涉及到的函数具有不确定性。这种情况下，由于我们无法预知调用请求涉及到哪条workflow，为不同的函数分配容器变得非常困难。我们需要了解不同工作流被调用的概率，以及理解整个DAG构成，从而正确为函数分配容器。这类应用被定义为DDA(dynamic DAG application)。以下列举了动态DAG的例子

     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220919164417586.png" alt="image-20220919164417586" style="zoom:50%;" />

- 动态DAG带来的挑战和解决方案

  1. 不确定的路径(本小点指出利用函数调用频率进行决策，减少over-provision的问题)

     问题：对于DDA，由于DAG中分支结点的存在，一个调用请求往往只涉及到了它的函数子集。由于一条工作流中，一个函数只有一个后继函数，使得对于分支节点而言，它将根据它的输入内容来决定调用哪个后继函数。因此，对于DDA而言，一个调用请求涉及的函数有较大的可变范围，它的函数调用路径是不确定的。对于一些假设函数调用次数始终与函数调用次数相同的框架而言，DDA将造成容器的过量配置

     解决思路：为了降低对函数的资源过配置，应当设计出感知工作流的资源管理框架，为每个函数单独地设置运行容器，而不是统一地为所有函数分配等量的容器。这样的框架应当分析每个函数的使用频率，从而按权重为函数分配容器

     Kraken方案：Kraken引入了函数的权重，由函数的调用频率(对于某个应用)和其他DAG相关参数定义，用于估算为每个函数分配的容器数。对于被不同应用使用的函数，它在每个应用中都具有独立的权重，从而计算出用于该应用的容器数。由于应用调用模式可能会变化，为函数分配的权重也应当随之变化，从而能够更好地分配运行容器

     方案效果：为了分析应用上函数的调用频率进行容器供应的优势，作者设计了实验，对比静态分配策略(均等分配容器，代表了当前常见的serverless平台供应方式)、Xanadu策略和按调用频率分配策略三者在上述三个应用下的容器供应情况。Xanadu策略只为最有可能的一条路径(Most Likely Path)分配容器，如果请求没有按该路径运行，Xanadu将以反应式的策略为该路径上的函数提供容器，并减少MLP上的容器数量(在下个计算周期之前，它将保持MLP的容器数量，导致了过量分配)。结果如下图。根据结果可以看到，按调用频率分配的策略(probability-based)相比Xanadu和static，减少了很多的运行容器

     ![image-20220919173727167](C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220919173727167.png)

  2. 冷启动引起违背SLO(本小点指出利用DAG相关的参数进行决策，减少under-provision的问题)

     问题：容器冷启动会引发请求响应时间的大量增加，导致了出现冷启动时，会违背请求的延时SLO。虽然目前已经有大量的工作专注于减少容器冷启动开销，如预先启动容器，能够为我们的容器配置带来不小的帮助，但是由于DDA中分支的存在，使得我们难以预先为各条路径配置好合适数量的容器

     **问题总结来说，就是冷启动带来的延时要求我们为不同函数配置合适数量的容器，且尽量减少under-provision**

     关键决策因素：

     - 关键函数，即具有多个后代函数(所有可能的后代)的函数节点，通过Connectivity(后代函数数量除以总函数数量)进行衡量。如果函数配置的容器数量不足，将导致请求排队，等待后台冷启动对应的容器。对于关键函数而言，它的排队将影响到更多函数的运行，造成严重的后果。文中对比了关键函数和非关键函数配置不足时的影响，得到的结论是关键函数配置不足将引发更长的响应时间和更多地违背SLO
     - 公共函数，即位于DAG多条路径上的函数(描述为Commonality)，同样应当被赋予更高的权重。公共函数由于位于多条路径上，往往具有更高的调用频率，且在负载数量上升时会接收更多的负载。因此，公共函数应有更高的权重，来确保在可变的应用访问模式下能够保持良好弹性(适量配置更多的容器)

     解决思路：虽然根据函数的调用频率进行容器规划能够有效地避免容器过量分配的问题，但是我们应当给关键函数和公共函数更高的权重，适当地分配更多的容器，使得在负载增加时他们能有较好的弹性，减少冷启动带来的影响。

     Kraken方案：函数的weight除了考虑预测出的调用频率外，还需要考虑上DAG关联的因素Commonality和Connectivity。这些考虑被结合在Proactive Weighted Scaler中，利用weight为每个函数分配合适数量的容器

  **本节带来的解决思路就是，在利用预测和根据函数调用可能性分配容器，减少over-provision的同时，根据关键函数和公共函数的划分，适当地对这些函数over-provision而避免负载增加时冷启动违背SLO**



#### Function Probability Estimation Model

根据Background部分的叙述，为了给函数配置合适的容器数量，我们需要计算出赋予每个函数的权重weight来作为参考。其中，为了获得函数调用的可能性(function invocation probability)，即预测的函数调用频率，作者将其建模为Variable Order Markov Model(VOMM，变阶马尔可夫模型)。VOMM能够为被多个应用共享的函数捕捉到该函数在每个应用的调用模式和特征，从而能够帮助我们计算出每个函数的调用可能性。VOMM是马尔可夫模型的扩展，其从当前状态到下一状态的转换概率不仅仅取决于当前状态，还取决于其前驱状态，order即当前状态到开始状态的前驱状态数量，variable说明不同状态的阶数不同。将DAG映射到VOMM，我们只需要将函数视为状态，函数调用关系视为VOMM的转换关系，每个函数的调用概率视为从开始状态转换到当前状态的概率

对于DAG的转换概率矩阵T，其大小为n\*n，t_ij表示到达j的请求会引发j调用到i的概率(j经一步转换到i的概率)。t_ij的计算可以表示为函数j转移到i的请求数/函数i的请求数。i必须是j的直接后继结点，否则t_ji为0。DAG转为矩阵T的例子为

<img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220919224522416.png" alt="image-20220919224522416" style="zoom:50%;" />

对于DAG的概率向量P，其大小为n\*1，P_d_i描述了模型在经过d个时间单元后位于状态i的概率。一个时间单位表示为DAG中所有函数的最长运行时间，一个函数的深度表示为从开始状态到该状态的边数。综上，d+1时刻的概率向量P_d+1的计算可以表示为T\*P_d，从而我们可以计算出不同时刻被调用的概率

对于时刻t，假设到达的负载数量为PL_t(可以通过负载预测模型进行预测)，每条指令到达函数时都需要一个容器来处理，则我们应在负载到达之前为深度为d的函数准备的容器数量为

<img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220919223158750.png" alt="image-20220919223158750" style="zoom:50%;" />

设冷启动的时延为a，则我们应在t-a+d的时间点上确保t+d时间点深度为d的函数能有上述数量的容器(不够则需要启动容器)。为某个函数准备的容器总量表示为将所有深度d的NC_t-d求总和

我们将上述马尔可夫模型转化为VOMM，具体操作为将context-dependent的状态分解为多个独立的context-independent的状态。例如，如果Compose_Post->Post_Storage的转换依赖于Text->Compose_Post，Media->Compose_Post等的转换，则将Compose_Post状态分解为Compose_Post|Text、Compose_Post|Media等状态，并将每条状态向Post_Storage连一条边(这是因为DAG的边权在不同时刻是可变的，受Text影响和受Media影响下，Compose_Post->Post_Storage的相关边权可能是不同的，通过拆分的方法能分别考虑不同前驱节点影响下的边权)。某个结点被拆分后，它需要的容器数量相当于所有相关节点计算出的容器数量相加



#### Overall Design of Kraken

<img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220920111419079.png" alt="image-20220920111419079" style="zoom:50%;" />

- Kraken整体设计与运行流程

  Kraken的运行流程如上。用户通过调用触发的方式，向serverless平台上的应用提交请求(1)

  Kraken中，Proactive Weighted Scaler(PWS)提前计算出各个函数处理到达的请求需要的容器数量，并提交给底层资源编排器(6)启动容器，避免冷启动的发生。为了合理作出决定，PWS首先需要从监控器和资源编排日志中获取相关的系统指标，并根据各个函数的调用概率，估算出各个函数在DAG中的权值(WEIGHT ESTIMATOR(2a)负责)。除此之外，根据应用开发者提供的DAG Descriptor提取出的DAG相关的参数如Commonality和Connectivity同样需要用于估算函数的weight，从而利用上关键函数和公有函数的特性。Load Predictor(2b)根据系统指标预测出未来的负载情况，并基于估算出的函数权重，决定好预先分配给各个函数的容器数量。事实上，PWS计算出的容器只有一部分会启动，因为函数会通过batching的方式将请求组织成批(每个函数的容器能够同时处理多条请求)，从而在不违背SLO的情况下能减少容器供应

  为了减少预测失误的影响，Kraken采用了REACTIVE SCLALER(RS)，来根据实际情况进行容器的伸缩。Overload Detector通过监控在容器中的排队时延，从而跟踪每个函数的overloading情况。当出现较为严重的排队时延时，Overload Detector将会计算出需要的额外容器，并触发函数扩展。Function Idler在检测到容器数量溢出时，将触发回收逻辑，逐渐减少函数的容器供应

  综上Kraken通过配合PWS和RS，在保证SLO要求的同时，利用函数调用概率、批处理请求和容器驱逐等方法，最小化容器的供应数量

- Proactive Weighted Scaler(整体的算法过程如下，一次proactive_weighted_scaler计算的是单个函数的容器配置数量)

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220920164148096.png" alt="image-20220920164148096" style="zoom:50%;" />

  1. Weight Estimator

     与SDA函数调用的确定性相比，DDA的函数的调用模式预先是不可知的，一次调用往往只涉及到部分的函数，使得如果根据负载数量为所有函数提供相同数量的容器，将会造成极大的浪费。因此，Kraken应用了Weight Estimator，为每个函数赋予了weight，并依此按比例为函数分配容器。函数权重的估算和需要容器的计算算法为𝐸𝑠𝑡𝑖𝑚𝑎𝑡𝑒_𝐶𝑜𝑛𝑡𝑎𝑖𝑛𝑒𝑟过程

     Probability计算：函数的调用概率(invocation probability)被表示为DAG中从起始节点到函数对应节点的概率。根据Probability Estimation Model的计算，深度为d的函数被调用的概率为T^d \* P0，其中T为DAG图中每条边的边权矩阵，通过系统日志以及函数的调用情况计算得到。枚举函数所有可能的深度，并将其叠加，即可获得一个函数在一次调用中可能被调用的概率。在算法中，该过程由Comput_Prob表示

     Connectivity计算：根据background部分的描述，由于cold start往往会级联影响后面的函数，在分配容器时我们不仅需要考虑函数的调用概率，还需要考虑冷启动的影响。对于关键函数，如果为其分配更多的容器，能够在源头上减少级联影响的发生。因此，Kraken将Connectivity作为权重的考虑指标，为关键函数分配更多的容器，从而降低关键函数的响应时间，同时改善了后续函数的响应。Connectivity的计算为函数的后代函数数量/函数总数，其中后代函数包括了所有可能被该函数影响的函数，例如在如下应用中，Compose_Review的Connectivity为0.3

     ![image-20220920171240281](C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220920171240281.png)

     Commonality计算：用户调用模式的变化等因素会引发工作流激活模式变化，导致Probability Estimation Model错误地估计函数的调用概率。如果根据错误的函数调用概率来按比例分配容器，将会导致错误的容器分配方案。特别地，在出现某个函数的调用频率大幅增加时，错误的分配方案将会引发冷启动的问题，涉及该函数的调用的响应时间都会受到影响。为了解决这个问题，Kraken考虑了函数的Commonality参数，表示与该函数相关的路径数与总的路径数之比。例如，上述Compose_Review的Commonality参数的值为6/7(原文中说明该应用的总路径数量为5，个人觉得不太正确)。由于Commonality高的函数位于更多的路径上，它们的调用次数有更大的可能增加，且under-provisioning的影响范围更广，因此将Commonality计入函数权重，可以为这类函数增加供应的容器数量，从而在它们的调用频率增加时能减少冷启动问题。为了避免将Connectivity和Commonality计入权重引发over-provisioning的问题，我们需要将这两个参数的值进行限制

  2. Proactive Container Provisioning

     根据估算出的函数权重weight，可以预测DAG中需要的容器数量。由于容器的启动存在冷启动时延，需要提前对负载进行预测，并根据预测结果预先为各个函数启动相应数量的容器，从而及时准备好容器来处理用户的请求。Kraken应用Load Predictor来对未来的负载状况进行预测，该Predictor采用轻量级的EWMA模型来进行预测，减少预测时间带来的延迟。每过PW时间，Kraken将触发PWS的逻辑，预测下一PW时间所需要的容器数量，其中PW代表了所有函数的容器冷启动的最长时间。PWS首先预测t+PW时间的请求数量(根据当前请求和预测请求)，并根据函数的batch情况计算出需要处理的batch数量(表示为请求数量/batch大小)。结合需要处理的batch数量以及函数的权重情况，获得每个函数需要的容器数量，并启动Scale_Containers逻辑通知底层的资源编排器启动相应数量的容器

- Request Batching

  serverless框架的常用做法是对于每个函数，都为每个请求分配一个容器提供服务。这种方法虽然能够获得较低的响应时间，最小化SLO的违反情况，但是通过一定的松弛操作，我们可以使用更少的容器，来获得相近的SLO达成率。松弛指的是利用函数运行时间和期望SLO延时的差距，通过让一些请求排队来减少容器的供应数量。对于每个工作流，我们可以将应用期望SLO延时按函数的运行时间的比例分配给工作流中的各个函数，获得分阶段运行时间StageSLO。各个应用对应的StageSLO分配如下，可以看到需要函数运行时间与期望SLO有较大差距

  **VOMM将context-dependent分解为context-independent结点可能也是基于这种考虑，划分出不同工作流后，按同深度的函数运行时间来为不同深度的函数分配SLO配额(下图中同深度，即同阶段的函数对应的阶段运行时间相同)**

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220920213848604.png" alt="image-20220920213848604" style="zoom:50%;" />

  利用函数的StageSLO与实际运行时间的差距，我们可以采用批处理的方式，令多个请求在同个容器上排队，从而减少容器的供应数量。每个函数f采用的batch大小的计算为lower_bound(StageSLO(f)/ExecTime(f))，表示了函数f的一个容器可以在不违背期望SLO的情况下最多可以接纳的请求的数量。我们在算法中考虑上每个函数采用的batch大小，从而计算出为每个函数实际需要分配的容器数量，减少了为每个函数分配的容器

- Reactive Scaler (RS)

  由于负载错误预测和函数调用概率错误计算的存在，我们无法通过proactive的方式准确地为各个函数配置正确的容器数量，这可能影响SLO的达成情况，或配置了过多的容器。因此，Kraken使用了RS部件，在出现容器过载情况时增加给函数分配的容器，在over-provisioning时减少给函数分配的容器。对于容器供应不足的情况，Overload Detector检测每个函数拥有的容器数量，并估算出该函数的请求预计的等待时间(Delay_Estimator)。如果发现该函数的部分请求的等待时间超过了冷启动函数的时间(这些请求在算法中表示为#delayed_requests)，则将这些请求转移到新启动的容器上等待运行。新启动的容器数量为#delayed_requests/batch_size。对于容器过度供应的情况，Function Idler module检测到当前负载数量/batch_size小于已有的容器数量，说明不需要这么多容器处理请求，此时将会慢慢减少函数的容器供应数量。通过PWS和RS的协作，并利用上batching技术，Kraken能够做到在保障SLO达成的情况下，尽可能减少容器的分配数量

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220920223013800.png" alt="image-20220920223013800" style="zoom:50%;" />



#### Implementation and Evaluation

- Kraken原型机部署(下表有一些误解，具体看Overall Design of Kraken的模型图)

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220921104344521.png" alt="image-20220921104344521" style="zoom:50%;" />

  Kraken采用Python和Go语言部署在开源serverless平台OpenFaaS上(以Kubernetes作为主要的容器编排器)。OpenFaaS默认使用Alert Manager module，依据Prometheus捕获的系统指标通知容器编排器启动容器来解决检测到的负载提升的问题。为了迎合Kraken的设计，系统关闭了Alert Manager Module并部署了PWS和RS来执行容器伸缩策略

  PWS和RS都需要通过Load Monitor和Replica Tracker模块，从Prometheus和Kubernetes系统日志收集相关的信息，包括当前各个函数的容器数量、负载历史记录、函数的请求到达率等，以便进行容器伸缩的决策。对于每个函数的负载情况，需要为每个应用单独计算，避免共享函数的负载和调用概率受到不同应用的影响。PWS还通过接收用户的DAG descriptor，用于得出函数的关系以及关键函数、公有函数的信息

- Large Scale Simulation

  作者通过设计一个多线程仿真器，利用从实际集群中获取的冷启动时延和函数运行时间，帮助作者模拟Kraken在具有11k个核，能支持超过7000条请求的大规模集群下的运行效果。此外，利用仿真器，作者能够对比Kraken和具有100%负载预测正确率的系统的资源效率

- Evaluation Methodology

  能源测量：作者通过使用开源的Intel Power Gadget，对每个结点的所有套接字进行能源的监控，测量结点的能源开销

  负载生成：作者采用具有不同特性的负载到达模式作为输入，提交给基于Hey的HTTP请求负载生成器，以不同的特性发送请求。采用的负载到达率包括了实际应用中Wiki和Twitter的请求到达模式，以及均值为100的基于泊松的合成请求到达模式

  应用函数：每个请求都会对上述DDA(social network、media service和hotel reservation)的工作流进行一次调用。作者将每个应用根据对应结构在OpenFaaS上部署为相应的函数链。为了模拟出应用真实的特性，每个函数都调用sleep函数模拟其运行时间。函数之间的转换通过函数调用进行，每个函数根据预定义的函数转换概率(对于分支节点，对每个后继函数的转换概率会变化)，随机调用其后继函数运行，从而模拟出多样的调用模式

  测量指标与baseline设置：

  - 指标：为测试Kraken的性能特点，作者通过测量和对比一些指标来彰显Kraken的优越性，包括平均的容器运行数量、SLO达成率、平均应用响应时间、端对端请求延迟的分位点、容器利用率以及集群范围的能源节约量
  - baseline：实验对比Kraken与先进的容器供应策略Archipelago、Fifer和Xanadu在上述测量指标上的差异，突出Kraken的优势。此外，作者通过抑制Kraken中的一些决策，显示Kraken对不同因素的考虑的意义。抑制策略包括：静态分配函数调用频率(SProb)、根据调用模式动态变换函数调用频率(DProb)，这些策略都只将调用频率作为函数权值，而不考虑Commonality和Connectivity的情况。作者还对比了Kraken和理想预测下的策略Oracle，显示Kraken能够做到接近最优



#### Analysis of Results

作者在现实系统和仿真平台上，测试了每个应用单独在平台上运行时的运行指标，对比不同供应策略的性能指标，证实Kraken的优越性。此外，作者证实了Kraken在多应用同时运行在平台上时也能获得类似结果

- 现实系统的结果

  1. 从容器的运行数量上看，对于所有应用，Arch、Fifer和Xanadu相较于Kraken运行了更多的容器。Arch过量运行的原因在于其假设所有函数在一次请求中都会被调用，且为每个请求分配一个容器进行处理。Fifer通过采用batching的方法减少了容器的供应数量，但是其没有考虑工作流的调用模式，即各个函数的调用概率，导致了过量分配的问题。Xanadu再分配是考虑了工作流的特性，但是没有应用batching方案，且其只预先分配Most Likely Path(MLP)上的函数，导致其他函数需要通过reactive的方式供应，且MLP上的函数过量供应容器的问题(从结果图可以看出Xanadu为少部分函数分配了大量的容器)。Kraken在容器方面的减少量主要与应用工作流数量和松弛操作的限额相关(a和b的工作流数量大于c，且a的函数松弛限额更高，因此优化程度上a>b>c)。

     此外，SProb和DProb的运行容器数量都小于Kraken，因为它们没有考虑Commonality和Connectivity的问题，但是这些增添的容器数量是必要的，它们被用于提高SLO的完成度

     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220921162746118.png" alt="image-20220921162746118" style="zoom:50%;" />

  2. 从端对端响应时间和SLO完成率上来看，Kraken相比起其他供应策略，在减少资源使用量的同时，还能够保持可观的响应时间，且保证了良好的SLO完成率(与Fifer相近，仅次于Arch，但是需要的容器数量远小于Fifer和Arch)。Xanadu虽然在响应时间和容器供应上比Kraken略多一些，但是因其响应式的供应，导致其SLO完成率远低于Kraken。在工作流更多的应用上，其MLP预测将会更不准确，导致更长的响应时间和更低的SLO完成率。DProb和SProb在SLO完成率上不尽人意，且有较大的响应时延，因为其没有考虑DAG相关的构造(关键函数和公有函数)，且SProb有极长的排队延时，因为它不会根据用户访问模式的变化调整函数的权重

     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220921164105189.png" alt="image-20220921164105189" style="zoom:50%;" />

  3. 从容器利用率上看，Kraken的每个容器能够处理的请求数量相比Fifer、Arch和Xanadu要高出许多，拥有更高的容器利用率，因为Kraken限制了为函数分配的容器，使得同等负载下容器需要处理的请求数量更多。DProb和SProb的利用率更高，因为它们分配更少的容器，但是它们的SLO达成率不佳

  4. 从响应时间的99%分位点上看，Kraken虽然比Fifer、Arch有更长的响应时间，但是其容器供应量远小于这两种分配方案。DProb、SProb容器供应量低，但是其响应时间99%分位点远高于Kraken，主要来源于冷启动和排队时延。Xanadu的响应时间与容器供应效果都不及Kraken

  5. 从能耗上看，Kraken相比Fifer、Arch和Xanadu都只需要更少的能耗，因为其通过batching和proactive scaling的方式，使系统利用上内存来对请求进行排队，利用上DAG相关信息合理分配容器，使得处理相同数量的请求需要的容器数量更少，能耗更低

- 仿真结果

  作者利用仿真器，将实验在模拟的11k核大规模集群上进行复现，采用均值为1000的泊松到达率，均值为284的WIKI请求调用模式核均值为3332的Twitter请求调用模式，测量Kraken在大规模集群下的容器使用量和SLO完成率。结果显示，Kraken在大规模集群下，仍能大量地减少容器使用量，甚至具有优于实际系统的效果(大规模系统中Kraken的效果更好)。通过对比不同模式的请求调用，发现具有较大可变性的调用模式，如Twitter的调用模式，往往引起响应时间重尾的现象。然而，即使在该调用模式下，Kraken的99%分位点仍保持在了SLO延迟内，说明其良好的适应性和弹性

  利用仿真器，作者还对比了Kraken和能够完美预测未来负载情况的资源供应策略Oracle。Oracle在Kraken的基础上，拥有完美的load predictor，能准确的根据未来请求调用模式进行容器供应。结果显示，相比于Kraken，Oracle能有更少的容器供应，原因在于Kraken需要考虑Commonality和Connectivity来减少因预测失误造成的冷启动。此外，对比Kraken的容器数量变化趋势，Oracle的变化趋势与负载变化趋势保持一致，而Kraken具有滞后性(需要冷启动进行补充)

  作者还测试了稀疏请求调用和有限SLO限制(低于原本的1000ms)下Kraken的性能表现。在稀疏请求调用下，Kraken能够保证SLO的满足，而Xanadu在这种情况下虽然能够使用更少的容器，但是有55%的请求无法满足SLO。在有限SLO限制下，Kraken在保持99.5%的SLO完成率的同时，相比Arch、Fifer和Xanadu减少了许多容器供应。其次，相比于只考虑Commonality和只考虑Connectivity的情况，Kraken更有弹性，能够为关键函数和公有函数提供更多的容器，降低了尾部的请求响应时间
