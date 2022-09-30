## Fifer: Tackling Resource Underutilization in the Serverless Era



#### Abstract

基于微服务架构的应用已越来越多地采用serverless函数作为底座运行。微服务应用通常运行量较小，运行时间短，有严格的SLO限制，其逻辑功能通过将一系列微服务进行链接，并按一定顺序运行微服务进行表达。然而，在容器供应和管理方面，目前的serverless平台将微服务应用视为传统的单片应用，采用不感知微服务特性的调度方式，造成了严重的over-provisioning问题和较低的资源利用率

为了解决serverless平台上对微服务架构应用的容器供应问题，本文提出了资源管理框架Fifer，旨在在serverless平台上合理地管理函数链(function chains)。Fifer的主要功能在于：通过高效地将任务打包给少量的容器，以提升资源的利用效率(task batching方案和function aware scaling方案的结合)；通过预测和提前启动容器的方式减少冷启动，从而最小化端对端时延

**serverless平台将微服务应用视为单片应用具体的操作是什么，这种操作为什么会有问题**



#### Introduction

公有云的出现与发展引发了基于微服务架构应用的爆发增长。微服务架构因其扩展和开发的简易性，已被多个大型云公司视为首要的应用架构，通过将应用组织成微服务链，对外提供服务。对于许多微服务应用而言，如人脸识别，它们是面向用户的，因而具有严格的端对端时延要求。因此，降低微服务应用的端对端时延(资源供应时间和任务运行时间)，能给用户带来更好的使用体验并提高用户的满意度。基于这些需求，serverless函数因能快速地进行部署和供应资源，并接替应用开发者的资源管理工作，已逐渐成为微服务架构应用的重要底座

在serverless平台上运行微服务架构应用同样给云提供者的资源管理带来了众多的挑战。虽然目前已经有众多的为serverless应用服务的资源管理框架，针对大量serverless应用的资源供应方式仍在资源利用率和SLO完成度方面存在问题，主要表现为

1. 许多RM框架简化了容器的供应，采用统一的方式为serverless应用中所有的函数都供应相同数量的容器，以满足每个函数的需求。这种方式往往导致了over-provisioning的问题，使容器需求少的函数掌握着大量的空闲容器，造成了资源的浪费
2. 许多RM框架采用请求-容器一对一的方式，令一个容器同时只能处理一个请求，使得出现负载激增的时候需要大量的容器供应来处理请求，而事实上容忍一定的排队能使用更少地容器来处理请求
3. 对于部分框架，为了减少容器的供应，它们只将请求分配给固定数量的容器进行处理，导致到负载激增时违背了SLO需求

资源利用率低下的问题可以通过松弛操作解决，即将部分请求在已运行的容器上排队，从而在不违背SLO的情况下，减少容器的供应量。Fifer采用了分阶段的资源管理(分开考虑各个函数的容器供应)，应用最新的松弛计算技术，为每个函数计算出能使用的batch大小，减少容器的供应，从而提升资源的利用率并节约集群能源消耗。此外，在收缩容器数量的同时，为了避免冷启动时延违背SLO，Fifer使用负载预测模型，提前为冷启动供应容器，从而提升SLO完成率。本研究的主要贡献在于

1. 通过实验和测量描述了冷启动的影响，测试出冷启动时延甚至比函数执行时间更长。此外，实验还证明了通过在warm container中进行排队可以有效地减少容器的供应数量

2. 本文提出了slack(松弛)的概念，并在Fifer上应用slack技术，采用stage-aware的方式，为每个函数计算出可用的batch大小，从而通过容忍一定的排队时延，减少容器的供应数量。每个函数的slack都分别决定，并分别计算各个函数的扩展阈值

   **slack和stage-aware在Fifer中的具体操作是什么，推测与Kraken一样，分阶段分配SLO，并根据SLO与运行时间的差异计算slack**

3. 研究量化描述了各种不同的负载预测模型(采用ML技术或不采用ML)的优缺点，并在Fifer上采用基于LSTM的预测模型(在负载变化较大时也能提供良好的预测)，进行负载预测，从而提前配置容器来减少冷启动时延

4. 通过将Fifer应用到Brigade框架上，并在Kubernetes集群上进行测试，证明Fifer在容器供应和提升资源效率上有重大贡献



#### Background and Motivation

- serverless函数链介绍

  对于微服务架构应用在serverless平台下的部署，我们将每个微服务映射成一个serverless函数，并通过同步服务，如AWS Step Functions，将函数连接成函数链。对于一次函数链调用，用户需要为每个函数的调用以及函数之间的传递次数进行支付。函数之间的传递，事实上是通过在centralized event bus中触发事件，从而调用下一个函数运行。由于serverless函数是无状态的，其容器内部不存储数据，因此不同函数的输入数据和输出数据都需要通过外部存储服务进行存储(如AWS S3)。serverless函数链的整体架构为

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220926163931467.png" alt="image-20220926163931467" style="zoom:50%;" />

- serverless函数的特点

  1. 冷启动问题

     当函数在没有初始化的容器时被调用，serverless平台会新启动一个容器来运行函数，导致了冷启动时延。容器的冷启动时延主要取决于容器启动和函数运行环境部署的时间。虽然目前有多样的技术被提出以减少容器启动时间，冷启动时延对于运行时间很短(毫秒级)的应用而言，仍旧带来了高额的开销。特别对于面向用户的应用，冷启动会严重影响SLO的达成率

     为了描述冷启动和热启动的延迟差异，作者在AWS lambda上运行基于Mxnet框架的ML图像推理应用，测试了不同模型的函数运行时间和端对端时延(请求发出到接收输出)。冷启动表现为第一条请求的运行时间和端对端时延，热启动表现为接下来100条请求的平均运行时间和端对端时延，得到结果如下图。可以看到对比两个实验结果，模型的运行时间相当，但端对端时延差距较大。特别对于大型模型，如Resnet-200，冷启动时延更高，引起了严重的代价。综合来看，冷启动增加了2000ms-7500ms的端对端时延，带来了相当大的开销

     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220926172711418.png" alt="image-20220926172711418" style="zoom:50%;" />

     为了避免冷启动问题，部分框架采用一批预热的闲置容器，从而在冷启动发生时能及时供应容器，但这种方法增加了内存的消耗，降低了资源利用率。一种可能的减少冷启动并提升资源利用率的方案是通过容许请求在容器中排队，而不是令请求与容器一一对应。当冷启动造成的时延高于请求在容器中的排队时延时，令请求排队比启动一个新的容器更加有效。因此，请求进行排队的决策需要基于冷启动时延、SLO和函数的运行时间。然而，Azure平台采用的RM框架虽然使用了排队的方式，但它只为应用提供固定数量的容器，并使所有请求在这些容器上排队，导致在请求突发时出现SLO达成率低下的问题。此外，部分开源平台的调度器，如Fission、Knative，采用了基于Pod的水平扩展策略，但这类策略的排队没有考虑应用的运行时间

     综上，我们能得到的结论是：基于冷启动时延、SLO和函数运行时间来做出请求排队的决策，能有效减少冷启动的发生，并能在不违背SLO的情况下减少容器的供应，提升资源的利用率

     **容器预热指的是怎么样的操作**

  2. 不同函数运行时间不同，同个函数运行时间可预测

     对于由多个函数构成的serverless函数链，每个函数都会单独被分配容器运行。对于目前已有的serverless平台，它们统一地根据负载为所有serverless函数分配相同数量的容器。然而，不同函数的运行时间往往是不统一的。研究表明，多数应用中不同函数的运行时间有较大的不同，部分应用的运行时间由少量函数主导。因此，如果平等地为每个函数分配容器，将会出现部分函数有很多空闲容器的问题。因此，为每个函数分别计算队列长度和容器分配数量，而不是统一地将所有请求在应用入口进行排队，可以有效提升容器利用率

     为每个函数计算容器数量的方法基于如下假设：应用每个阶段的函数的运行时间是可以提前了解或测得的、函数的运行时间是可预测的，不会有大幅度的变化。由于我们可以预先知道应用的函数构成，采用线下的测试可以估算出各个函数的运行时间，因此第一个假设是成立的。第二个假设同样成立，特别对于基于ML的应用，由于ML模型的特征向量和大小是固定的，并有确定的与输入无关的控制流，因此其运行时间不会有太大的变化。这类应用的运行时间变化性主要来自于底层的基础设施配置以及同个宿主机上的容器的运行干涉。虽然ML应用的运行时间可能与输入的大小有关，但是测试表明运行时间与输入大小存在线性关系

     与常用的SLO时延1000ms相比，面向用户的应用通常运行时间较短，因此有很大的机会可以通过slack操作减少容器的供应。如果我们可以提前了解到应用的端对端时延，乃至每个函数的时延，我们就可以根据它们与SLO时延的差距，计算出松弛的范围

     综上，RM框架应当能够依据不同阶段函数不同的运行时间，以及端对端时延与SLO时延的差距，确定每个函数的容器可以容许的队列的长度。基于这种方案，我们可以有效地减少容器的供应数量，从而提高整体的容器利用率



#### Preamble to Fifer

上述提到的serverless函数链给serverless平台的资源分配带来的挑战，要求我们重新设计serverless平台的调度和资源管理框架，从而有资源效率地运行函数链。实验的baseline是当前serverless平台中常用的RM框架的代表，其为每个请求分配一个容器进行处理。基于这种RM框架，作者提出能够令请求在函数链的各个阶段进行排队的管理框架RBRM。RBRM通过动态计算每个阶段可行松弛时间与函数运行时间之比，获得函数链各个函数的容器队列长度(容器的batch size)，从而在不违背SLO的情况下有效减少容器供应。下图显示了排队能有效减少容器数量

<img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220927113926544.png" alt="image-20220927113926544" style="zoom:50%;" />

然而，通过将请求在容器中排队虽然能够有效减少容器的数量，但是其不能够隐藏容器启动时的冷启动时延。虽然目前已经有许多OS层级的技术能够降低冷启动时延，但是只有通过预测并提前启动容器的方式，才能真正隐藏冷启动。因此，只有将提前启动容器与在已有容器进行请求排队进行结合，才能在最小化容器数量的同时，减少冷启动的发生，提升SLO达成率。提前启动容器的技术主要基于利用精确的预测模型对未来的负载情况进行预测



#### Overall Design of Fifer

- Fifer基本架构

  基于上述技术，作者设计出了为serverless函数链在平台上高效运行服务的RM框架Fifer，其主要架构如下。用户应用发送的调用请求将由scheduler进行收集处理，并由scheduler被分派到函数链的各个阶段的队列中。调用请求在队列中根据FIFO的原则，在出现空闲容器时，请求将被分配到容器运行。每个阶段的函数都有一个Load Monitor进行信息的收集(如负载情况、排队情况)，并将更新的信息提交给Load Balancer。Load Balance根据预测的负载情况和stage的排队时延，决定是否需要进行容器扩展或收缩

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220927120737140.png" alt="image-20220927120737140" style="zoom:50%;" />

​		**采用的是分阶段设置队列**

- Estimating execution time and slack

  获得各个阶段的slack时间和各个函数的运行时间后，我们可以准确计算出每个函数的容器容许的队列长度(batch大小)。基于线下多次测试各个函数在不同输入大小下的运行时间，构建出线性回归预测模型，使我们能够生成每个函数在不同输入大小下的平均运行时间MET。线性回归预测模型将作为Fifer的一个offline构件被添加到Fifer的架构中

  为了准确地预测各个应用的slack时间，作者将SLO延时设置为1000ms。在已知期望响应时间和应用总体的运行时间后，两者之间的差则被作为slack时间，用于计算各个阶段的函数容器容许的队列长度。由于对于不同阶段的函数，其队列长度会被分别计算，因此slack将被分配到应用运行的各个阶段中(均等分配或按运行时间比例分配)。Fifer将slack时间按各个阶段的运行时间之比分配给各个阶段的函数，以提升各个阶段的利用率

- Load Balancing

  Fifer在函数链的每个阶段设置一个队列，用于缓存提交给这个阶段的函数的所有请求①。Fifer采用Load Balancer②，依据每个阶段的Load Monitor③(LM)监测得到的数据来为每个函数进行伸缩。由于各个阶段的slack时间和函数的运行时间已知，因此Load Balancer(LB)可以依此以及分配的容器数量计算出每个阶段stage的队列长度。Fifer的Load Balancer采用两种伸缩方式：在没有足够容器运行当前所有请求时，采用动态的反应式伸缩；在一定时间间隔，采用proactive的方式进行容器供应。本小节主要讲述反应式伸缩的伸缩方式

  1. 反应式伸缩算法

     反应式伸缩的具体算法如下。首先，LB根据LM收集到的上10秒的负载情况和排队情况，判定队列最后的请求的处理时间是否已经超过了slack时间，即该请求是否因排队违背SLO。如果是，说明没有足够容器来运行当前请求，则需要增加容器数量。算法采用Estimate_Container过程估算需要添加的容器数量。需要的容器数量可以估算为排队请求数量/batch大小，但需要注意的是，当一个请求的排队时延比冷启动时延低时，令请求继续排队更可行。Estimate_Container过程考虑了这个问题，其计算出delay_factor，表示最后一个请求的调用时间，当delay_factor大于cold_start时间时，则进行扩展，扩展的数量(算法中的表述可能有误)为(PQlen-current_req)/(batchSize+1)，加1是因为batch大小是slack/exec而不是SLO/exec。算法中涉及到的变量为：PQlen(队列中的请求数量)、resp_latency(该阶段的SLO)、batchSize(每个容器的队列大小)、current_req(当前数量的容器支持的队列长度)。

     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220927160830823.png" alt="image-20220927160830823" style="zoom:50%;" />

  2. stage-aware伸缩

     由于应用各个阶段的函数的运行时间往往是不同的，因此需要为各阶段的函数分配不同数量的容器。然而，目前通用的serverless平台RM通常不是stage-aware的，它们为所有的函数都分配相同数量的容器，造成了资源的浪费。相比之下，Fifer采用的伸缩方式是stage-aware的。首先，Fifer根据函数的运行时间，按比例将slack时间分配给各个函数，使函数能有相同的batch大小。此外，Fifer会单独地根据LM监测获得的各个阶段的队列长度和排队时延，计算为各个阶段分配的容器数量，使不同阶段的函数会有不同数量的容器

-  Function Scheduling

  由于同个应用开发者可能会提交多个不同类型的应用到serverless平台上，这些应用可能会共享某些函数来完成特定的任务，因此选择一个合适的调度策略来将共享函数队列中的请求调度到容器上也是非常重要的。由于来自不同应用的函数在对应的应用中被分配的slack时间不同，因此采用单纯的FIFO策略可能会造成slack较低的函数的请求响应时间违背SLO。Fifer采用了Least Slack First (LSF) scheduling policy，优先将队列中对应slack较低的应用请求调度到空闲的容器中运行,同时避免饥饿的问题。这种策略可能不是最优的，但是由于本文的主要工作在于为不同阶段的函数提供容器，而不是请求的调度问题，因此不去调查哪些调度策略才是最优的

- Bin-Packing to increase Utilization

  1. 贪心的请求调度策略

     为了增加容器的利用率，对于每个请求的调度，我们采用贪心调度策略，以最大化每个时间点上空闲容器的数量。Fifer采用的贪心调度策略在于，我们尽可能地将请求分配给空闲slot最少的容器(称为 least-remaining free-slots策略)，每个容器的slot数量为其batch大小。具体操作是每个容器都会有自己的local队列，请求会尽可能地填满前面的容器的队列，而不是分配给新的容器。Fifer还设置了10min的阈值，如果某个容器在10min时间都没有运行请求，那么它将会被回收。贪心的调度策略能够让部分不需要的容器提前被回收，并提升了各个容器的利用率

  2. 贪心的节点选择策略

     对于容器的运行节点选择，Fifer同样采用贪心的调度策略。对于每个新启动的容器，我们选择将容器运行在可用资源最少的服务器上(该服务器仍旧能够满足容器的运行需求)，从而尽可能地利用上节点的全部资源。通过这种方法，我们能够在一定时间后暂停所有核都没在运行容器的服务器，从而在集群角度减少了空闲的CPU、内存等的消耗，提升了资源的利用率

- Proactive Scaling Policy

  仅仅利用排队时延和函数运行时延来做出反应式的容器伸缩往往是不足够的，这种容器伸缩方式有滞后性，在容器启动存在冷启动时延的情况下，这将是次优的伸缩策略。为了掩盖冷启动的影响，Fifer采用了负载预测模型，对应用未来一段时间的负载情况进行预测，并利用预测结果预先进行容器供应

  proactive的伸缩算法如下。每过一个监控周期，预测模型基于前100s的负载到达率，预测接下来的负载数量。对于每个stage的函数，如果其当前容器数量不足以满足未来所有负载的SLO，即未来的负载数量超过了队列允许的最大长度，则计算出额外需要的容器，预先启动。如果预测模型能够完全准确地预测出未来的负载情况，那么所有的请求都不会违背SLO；如果预测模型的预测有误，那么reactive的伸缩策略将会根据排队时延额外地启动容器处理请求。proactive策略与reactive策略相互补充，得到Fifer完整的伸缩策略

  为了准确地获取应用的请求到达情况，Fifer采用了LSTM模型进行负载预测。对于每个监控周期(10s)，Fifer按5s的窗口大小取样前100s的负载情况。其追踪每个窗口的最高负载到达率并获得全局的最高到达率。基于采样的负载到达情况(20个窗口的最高到达率)，LSTM模型以5s为窗口大小预测未来10min的负载情况，并以预测结果的最高负载到达率作为Fcast的值，判断容器是否足够

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220928101942874.png" alt="image-20220928101942874" style="zoom:50%;" />

  **LSTM模型为什么能用前100s的样本来预测后600s的负载数，推测可能是预测后10s的负载数；或者说采用滑动窗口的方式，先用20个窗口预测21的值，再用1-21窗口预测22的值，以此类推获得10min的负载数**

  作者通过对比八种不同的预测模型(4种基于ML，以60%的负载输入进行训练；4种不基于ML，只依据前100s的负载情况进行预测)的预测效果，发现LSTM模型有最低的Root Mean Squared Error(RMSE)，说明其预测效果最佳，且需要的运行时间较低，说明LSTM模型的优势。此外，作者还测试了LSTM模型的准确度，发现LSTM模型在800s的测试中能有85%的准确度

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220928112751551.png" alt="image-20220928112751551" style="zoom:50%;" />



#### Implementation and Evaluation

- Fifer原型设计

  Fifer的原型部署在Brigade框架之上。Brigade是Azure开发的开源serverless工作流管理框架，部署在Kubernetes集群上从而利用Kubernetes底层的容器编排机制。Brigade默认为每个工作流创建一个worker pod，用于进行容器创建、任务调度和容器删除(修改后的Brigade worker在容器完成时不会直接删除容器，而是等待其他请求调度到容器中运行)。函数链的每个stage都会维护一个队列，存储所有访问该stage的请求，而每个容器都有一个local队列，长度为batch大小。每个Pod对应一个容器

  作者使用了mongodb数据库来存储每个工作流请求的相关信息(开始时间、结束时间、调度时间等)以及容器相关指标(free-slot数量、上个使用时间等)。此外，每个函数链的一些指标也被加入到数据库中，如SLO时间、函数序列关系、预计运行时间、每个stage的slack时间、各个函数的代码等，从而可以计算出各个stage的容器的batch大小

  任务调度方面，每个请求首先被调度到对应stage的队列中。worker pod访问数据库，获取对应stage的容器(Pod)队列，并将stage队列中的请求调度到剩余free-slot最少的Pod中运行。请求将会被分配到Pod的local队列中，且数据库对应Pod的free-slot数量会随之更新

  负载均衡器的部署方面，负载均衡器由LM(load monitor)和LP(load predictor)组成。LM定期查询各个stage的队列情况，并根据排队时延决定是否需要增加容器处理请求(对比排队时延和冷启动时延)。LP模块采用LSTM模型，从数据库中获取过去一段时间的请求到达情况，并预测出未来的请求到达率，作为proactive策略的依据

  容器调度方面，Fifer尽可能将Pod调度到少量的节点上，即每次都将Pod调度到剩余资源最少的节点上运行，从而降低集群的开销。实验采用的Pod的需求是0.5CPU和1GB内存

- 大规模模拟

  作者利用实际系统中容器冷启动时间、函数运行时间、函数转移时间、函数镜像加载事件等，构造出一个与现实结果相符的仿真器，用于测试在大规模集群下Fifer的性能。通过结合使用仿真器和实际系统，能够测试出Fifer在不同环境下的性能，反映Fifer的优势

- 评估方案

  1. 集群配置：具有80个核的集群，并选择一个节点作为主节点。集群配置如下表。采用Kubernetes作为底层容器编排器，负载均衡器(daemon守护进程)和数据库都位于主节点上
  
     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220928200940855.png" alt="image-20220928200940855" style="zoom:50%;" />
  
  2. 集群能源消耗：Intel Power Gadget的开源版本，测试所有节点的socket的能源消耗
  
  3. 请求到达情况：采用平均为50req/s的基于泊松的请求到达率和基于WIKI、WITS特性的请求到达率。WITS具有较为振荡的负载到达率，可能出现负载激增骤降的情况，其平均为300req/s，但峰值达到了1200req/s。WIKI代表了典型的ML推理负载情况，平均为1500req/s，并具有明显的昼夜特征和工作日特点
  
  4. 应用：集群部署了四个常见的ML应用，包括Face Security、IMG、IPA、Detect-Fatigue。这四个应用由下列微服务构成
  
     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220928203224798.png" alt="image-20220928203224798" style="zoom:50%;" />
  
     不同微服务的运行时间测试结果为
  
     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220928203239240.png" alt="image-20220928203239240" style="zoom:50%;" />
  
  5. 负责设置：通过组合任意两个不同应用，按组合后的slack时间进行排序，分为Heavy、Medium和Light，测试在不同slack时间下Fifer是否都能有优势
  
  5. 函数配置：每个函数的容器都打包在了Kubernetes的一个Pod中，并在Pod配置文件中采用imagePullPolicy字段，使得属于不同函数的Pod的容器启动时会自动从dockerhub中拉取相应的函数镜像
  
  6. 性能参数指标：请求SLO达成率、运行过程中平均的容器数量、请求响应时间的中位数和99分位点、容器资源利用率、集群层级的资源节约情况
  
  7. 实验baseline资源管理框架：Bline(反应式为每个请求分配一个容器运行，分配上为每个阶段运行相同数量的容器)、Sbatch(基于平均请求到达率固定为每个函数供应的容器数量，slack按比例分配给每个stage)、BPred(采用LSF调度策略和EWMA预测模型进行proactive容器供应，但不支持batch)
  
     除此之外，为了测试Fifer各个组件的贡献情况，作者设计了如下几种框架：只采用反应式的供应策略RScale、采用反应式和预测式的供应策略。这些框架都采用LSF的调度策略、batching策略以及贪心的请求和容器调度策略
  
  

#### Results and Analysis

- 实际系统中实验结果

  1. 从SLO达成率和启动的容器数量上看，Fifer在SLO达成率高的情况下(只在Heavy下比BPred低，但是BPred的容器数量远多于Fifer)，启动的容器数量也最少(除了在Heavy负载下因slack较小，batch大小减小，导致比SBatch容器数量略多，但SBatch的SLO违背情况比Fifer多15%)。RScale由于没有采用proactive的方法，冷启动增加，SLO达成率不如Fifer

     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220929102233943.png" alt="image-20220929102233943" style="zoom:50%;" />

  2. 从请求响应时间的99分位点来看，由于batching的影响，可以看到RScale、SBatch方案都有较为严重的拖尾现象，其请求的99分位点要远高于BPred和Bline。但是，它们需要的容器数量相比BPred和Bline都大幅降低。此外，可以看到Fifer因proactive的容器供应方式，以及灵活的容器供应，相对于RScale、SBatch，其拖尾现象没那么严重，且相比前两者有更少的容器供应数量

     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220929110015469.png" alt="image-20220929110015469" style="zoom:50%;" />

  3. 观察不同策略在IPA应用下，为各个函数运行的容器的比例，结果如下。Bline和Bpred在分配上较为平均(每个stage采用相同数量的容器)，但是由于其反应式的伸缩方案，使得运行时间最长的第一个stage被分配到的容器数量更多。SBatch由于其固定的容器分配，且每个stage相同大小的batch，使其每个stage的容器数量一致。RScale和Fifer主要将容器分配给了stage1和stage3，原因是stage2运行时间较小，使得该阶段的容器由于空闲时间长被回收。而RScale因其激进的反应式伸缩，使其分配了大量的空闲容器给stage1，导致stage1的容器分配比例更大

     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220929112445796.png" alt="image-20220929112445796" style="zoom:50%;" />

  4. 对比不同策略的容器利用率，发现Fifer在任何阶段，其每个容器处理的请求数量都是最多的，在处理相同数量的请求下，Fifer减少了容器的供应数量。原因在于Fifer尽可能地将请求集中在正在运行的容器中，并允许请求排队。此外，Fifer还采用了proactive的供应方式，减少了over-provisioning的问题
  
     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220929155214246.png" alt="image-20220929155214246" style="zoom:50%;" />
  
  5. 从集群整体的资源开销来看，由于Fifer尽可能地将容器打包到正在运行的服务器上运行，且采用proactive的策略供应容器从而减少over-provisioning，使其能够使用更少的服务器来运行函数链，整个集群的开销更少
  
- 大规模仿真系统中的实验结果

  无论是对于WITS还是WIKI类型的负载，Fifer在综合考虑延时、容器数量和SLO达成率方面时具有优势，在保证较高的SLO达成率下，能够以很少量的容器数量处理各种请求。整体实验结果与实际集群中类似，但是Fifer的优势在WITS负载下较小，因为WITS负载的可变性相较于WIKI较小，负载预测的优势不能很好体现。此外，从冷启动的数量上看，Fifer因负载预测和proactive的供应策略，大量地减少了冷启动的数量，展现出其极好的适应性