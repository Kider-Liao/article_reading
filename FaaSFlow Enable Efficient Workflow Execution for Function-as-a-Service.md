## FaaSFlow Enable Efficient Workflow Execution for Function-as-a-Service



#### Abstract

Serverless技术通过在容器上运行函数，提供了细粒度的资源共享方式。对于数据依赖型函数，它们需要以某种预先定义的顺序被调用，形成serverless工作流(serverless workflows)。然而，传统的基于master-worker模式的工作流运行架构在serverless环境下并不适用，这主要源于master-side的工作流调度模式(由master节点调用，分派给worker节点运行)以及worker之间的高额通信开销(数据交换)

为此，本文提出了一个worker-side的工作流调度模式，用于serverless工作流运行。基于这个设计，作者实施了FaaSFlow来支持serverless领域工作流的高效运行。此外，作者还提出了存储库FaaStore的概念，用于支持位于同个结点的函数之间能进行快速数据传输



#### Introduction

目前，Serverless技术已经广泛地被用于云原生领域。通过Serverless技术，用户只需要直接提交自己的运行函数，而不是租赁虚拟机并决定相应配置。云供应商可以自动且高效地调度并运行函数。新型应用开始使用细粒度的函数，并将它们组织成工作流来提供复杂的功能逻辑(serverless函数以用户定义的逻辑顺序运行)。serverless的工作流往往具备并行分支和数据依赖型，可以通过DAG的方式表示逻辑顺序

主流的云供应商已经能够支持用户定义函数工作流，并采用master-side的工作流调度模式对serverless工作流进行调度。这种调度方式通常在master节点上增加一个中心化的工作流引擎，用于管理工作流运行状态并将函数分配到worker节点上运行(工作流引擎控制函数的调度顺序以及决定运行节点等)。然而，这种master-side的工作流调度模式会带来大的调度开销和数据传输开销。一方面，由于集群的所有函数的管理和调度工作都由master节点的工作流引擎负责，而工作流的函数往往是轻量级的，运行时间较短，因此会导致master节点需要频繁地将运行状态传输给工作节点，造成较大的调度开销且可能出现单点故障。另一方面，由于工作流中的函数之间往往存在数据依赖，而这些函数可能会因负载均衡被随机地调度到节点上，使得需要额外的数据库存储服务，存取这些函数之间的运行数据，这引发了很大的数据传输开销

为了让serverless工作流能够更高效地运行，本文提出了worker-side的工作流调度模式(WorkerSP)。在WorkerSP中，master节点将工作流DAG进行划分，并将函数的调度任务分派给各个工作节点。在每个工作节点上，workflow引擎基于前继函数的运行状态，独立地决定是否它可以启动和运行一个函数。然而，要做到合理的工作流划分并不容易，因为函数工作流的control-plane(用户定义的运行次序)和data-plane(数据依赖)可能是不同的，需要在这两个层面上来考虑将工作流划分到多个节点上。此外，由于工作节点的可用资源的影响，划分工作流时还需要考虑工作节点实时的资源可用性(确保工作节点能运行工作流上的函数)

此外，本文还提出了FaaStore，用于减少从远端存储进行数据传输的昂贵传输开销。利用FaaStore，同个宿主机上的函数可以直接利用共享的内存进行通信和数据交换，而不需要依赖远端存储。然而，利用内存来进行交换也带来了挑战，需要合理地分配出共享内存用于通信(太小且无序的共享空间不足以充分利用数据局部性的效益，太大的共享空间会给内存带来很大的压力)。

基于WorkerSP和FaaStore，作者提出并实施了serverless工作流系统FaaSFlow，支持高效的serverless工作流调度和复杂的工作流定义。master node上运行一个workflow graph scheduler，由scheduler对用户提交的serverless工作流进行解析，并把工作流划分为子图分派给各个工作节点。工作流划分的依据是工作节点的可用资源以及最小化各个工作节点之间的数据传输(有数据依赖的函数尽可能放在同个节点)。在工作节点上，FaaSFlow运行一个workflow engine来管理函数的状态，并在条件成立时启动对应的函数实例。工作节点上还运行了FaaStore来动态管理节点中被过量分配的内存(容器被分配但不使用的部分)用于数据存储和数据交换

本文做出的主要贡献在于

1. 在master-worker架构下，采用了WorkerSP来降低调度的开销，并使用FaaStore来降低数据传输的开销
2. 基于WorkerSP和FaaStore，构建出serverless工作流系统FaaSFlow
3. 提出基于WorkerSP的工作流图划分策略，能够基于工作节点的可用资源和函数实例的数量，将工作流划分为子图分派给合适的节点运行



#### SERVERLESS WORKFLOW BACKGROUND

- serverless工作流的基本概念

  serverless事件驱动的特性使它们非常适用于无状态应用，许多应用已经由上百个函数组成。这些函数以一个预定义的顺序运行(工作流)，从而表现出复杂的业务逻辑。对于一个由serverless函数组成的工作流，它可以被映射为一个由点和边组成的DAG形式，称为serverless工作流。虽然目前大多数云供应商只提供线性的工作流(函数以一个线性的形式被调用，且输入依赖于前驱函数的输出)，但是由于线性工作流比基于DAG的工作流简单很多，本文的工作主要面向于基于DAG的工作流

- Master-side工作流调度模式

  传统的工作流模型采用的是Master-side的工作流调度模式。master节点通过应用workflow engine，从而维护函数和工作流的状态。master节点从在工作节点运行的函数中获取到函数的运行状态，并判断是否工作流上有新的函数达到了触发条件。如果某个函数的所有前继函数都已完成，该函数将被触发分派到工作节点上运行，并返回对应的运行状态。然而，即使函数和它的前继函数被分配在同一个worker node上，MasterSP中的master节点仍需要收集函数的状态并检查函数的运行条件是否被满足，这引发了大量的无用网络交互和较长的回复时延

- MasterSP的开销

  HyperFlow是目前最先进的函数工作流引擎，可以支持函数复杂的依赖关系。作者通过将HyperFlow以MasterSP的模式应用在serverless环境下，构成HyperFlow-serverless系统，用于检验MasterSP模式下的调度开销。由于采用开环控制(只记录提交到终止的时间，期间不做控制)会导致最终的端对端时延没能考虑上冷启动的排队时延，实验中采用闭环控制(在提交任务后控制计时)，只记录每个循环中的调度时间和函数任务的运行时间。通过扣除函数任务的运行时间，最终能得到一次工作流中的总调度时延，结果如下(Cyc-Soy为复杂工作流实例，Vid-WC为serverless工作流应用)

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220913112236015.png" alt="image-20220913112236015" style="zoom:50%;" />

  由结果可以看到，工作流实例和应用的调度开销在181.3ms到712ms之间。由于在FaaS中，函数调用时间往往较短，这种调度开销是非常大的。此外，对于复杂的工作流，如Gen、Soy，其调度时延更高，因为工作流中涉及到的函数节点更多，产生了更大的调度开销。对于一些时延敏感型应用，如实时网络服务，这样的调度开销是难以接受的

  MasterSP的高调度开销激励了我们探究出更有效的WorkerSP，减少了master节点与其他节点的交互。worker节点通过应用一个去中心化的workflow引擎，自行维护函数的运行状态，并且通过worker节点之间的直接交互，将两次的状态传递降低为一次(MasterSP下需要从worker节点接收运行状态，再调用后继的函数)。这种情况下，与master节点与worker节点之间维护大量的并发TCP连接相比，该系统可以更有效地减少网络跳数

- Data-Shipping Pattern

  在serverless工作流中，每个函数在运行前都需要从其前驱函数获得输入数据，并将其读入内存中用于运行(Data-Shipping Pattern)。然而，由于serverless架构下的限额，函数之间能直接传递的数据总量是十分有限的(下图列举了各个云平台下的数据传输限额)。特别对于serverless工作流的情况，一个应用涉及到更多的函数，函数之间的直接数据传输将会更加被限制，否则会导致集群网络带宽的迅速消耗。因此，用户往往需要额外的数据库服务，来进行中间数据(大型的文件和对象)的存储。

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220913120150091.png" alt="image-20220913120150091" style="zoom:50%;" />

  为了验证利用数据局部性的意义，作者对比HyperFlow-serverless系统和将所有函数运行在同个服务器时的数据传输情况，得到如下结果。将应用函数运行在同个服务器(同个容器)的情况使应用函数直接可以直接通过内部调用进行数据通信，而不需要采用外部的数据库服务。对比数据的传输量，显然可以看到在HyperFlow-serverless系统下，函数分离运行带来了更大的数据传输量(Vid和Cyc的数据传输从4.23MB和 23.95MB增长到了96.82MB和1182.3MB)。此外，如果不考虑数据局部性，运行在同个机器上的函数也会通过额外的数据库进行数据传输，这带来了巨大的延迟。因此，利用数据局部性能有效降低数据传输的开销

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220913120606276.png" alt="image-20220913120606276" style="zoom:50%;" />



#### RATIONALE OF WORKERSP/FAASTORE

- Management and Synchronization(worker端的函数管理和状态同步)

  在WorkerSP中，master节点的图调度器将函数调用和状态管理的任务下放到worker节点，它只负责将工作流的图分解成子图发放给worker节点，并确保worker节点有足够的资源运行子图中包含的函数。worker中的工作流引擎接收到工作流子图，并执行对应的函数触发和调用。由于子图可能被分派到多个worker节点，我们需要重新组织数据结构来管理工作流和同步函数调用

  工作流的函数状态被分解到不同worker节点中进行维护，并由worker节点自行进行函数的触发和调用。为了确保worker节点能够独立地维护函数状态并触发函数调用，需要引入Workflow结构来表示子图的信息。每次工作流的调用都会被赋予一个InvocationID来辨别不同调用的信息(不同次的调用涉及的函数状态不同)。Workflow结构由State和FunctionInfo组成，其中State记录了函数的运行状态以及它的前驱函数，FunctionInfo记录了函数的元数据。Workflow结构的表示如下

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220913170233210.png" alt="image-20220913170233210" style="zoom:50%;" />

  以上图为例，当FnA被调用时，工作流引擎会在本地的Workflow结构中更新它的运行状态。在FnA运行结束时，工作流引擎会搜索FnA的后继函数FnB和FnC，并将运行状态通过节点间的TCP通信或内部的RPC通信传递给对应函数。最后，函数FnB和FnC将会获得FnA的运行状态，并更新自己的PredecessorsDone。当PredecessorsDone的值与FunctionInfo中的PredecessorsCount相同时，说明某个函数已经符合运行条件，引擎会触发并调用该函数

  **invocationID标识一轮工作流的调用，如第一次调用时invocationID为1，第二次调用时为2。原因是一个工作流在生产过程中会被反复调用(一个应用请求就会引起一个工作流调用)**

-  Adaptive Storage with FaaStore(FaaStore管理异构存储)

  由于内部RPC的有限配额，应用开发者通常需要登入远端的存储，进行函数之间大数据的传输。FaaStore通过利用节点内的可用内存，提供了异构的存储服务，让同个节点中的函数的直接通信变为了可能。FaaStore根据宿主机中的内存情况，分配出合适的配额(不能使节点内存压力过大)，允许函数能将输出存放到内存中，从而使不同函数能直接通过内存通信。如果剩余内存不足以存放函数的结果，FaaStore会传输给远端的存储服务进行数据存储。通过FaaStore，每个worker节点可以将数据传输本地化，减少了不必要的通信开销。以下是FaaStore的一个设计机制

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220913182346593.png" alt="image-20220913182346593" style="zoom:50%;" />

  FaaStore用fetch和save函数取代了标准的调用远端数据库的API。在workflow.yaml文件中，用户会定义函数存取数据的key，用于帮助函数运行时找到对应的输入数据。在取数据时，FaaStore.fetch首先会根据key查找内存中的数据，如果没有对应的数据再去访问远端存储；在输出数据时，FaaStore.save会根据successor是否在同个节点以及数据量大小，决定将数据存放到In-Memory Storage或Remote storage中。



####  THE FAASFLOW SERVERLESS SYSTEM

基于上述WorkerSP的原理，作者实现了FaaSFlow serverless system。该系统由三个组件构成：Graph scheduler接收用户上传的工作流，并根据工作节点的资源可用情况以及函数之间的数据传输量，将工作流划分成子图分配给合适的工作节点；perworker workflow engine在每个工作节点上都存在，负责管理函数状态和函数任务调度；FaaStore通过在运行时利用过分配给容器的内存资源，允许同个节点函数的直接通信，以异构存储(内存通信或远端存储通信)的方式支撑工作流函数间的数据传输

- Graph Scheduler

  Graph Scheduler通过接收用户提交的workflow.yaml配置文件，定义工作流的运行逻辑，并交由DAG parser对运行逻辑进行解析。在分析各个工作节点的可用资源情况后，Graph Scheduler能正确地将图划分给不同的工作节点运行

  1. DAG parser(将用户定义的工作流转换为DAG)

     用户通过Workflow Definition Language(WDL)定义serverless工作流以及各个步骤之间的输入输出关系。Graph Scheduler使用DAG parser来检查用户定义的workflow是否合法，并将WDL定义出的工作流转换为DAG对象。DAG parser采用如下概念对工作流的逻辑进行描述

     - Function

       工作流中每个运行任务由一个函数实例执行，一个独立的函数将转换为DAG的一个节点

     - Sequence

       序列表示工作流中任务的接替关系。对于函数序列(一个函数运行完后，其后继函数才能开始运行)，它们之间的接替关系通过DAG的边表示，并以它们之间的数据传输延时(99%-ile latency)作为边的边权

       **根据算法的描述，边权应该是传输的数据量的多少，因为要用来判断内存是否足够容纳边**

     - Parallel

       parallel表示了并行执行的多个子任务，在工作流中用属性branches进行定义。DAG parser会分别将每个分支进行转换

     - Switch

       switch表示了工作流中根据运行条件调用不同任务的关系。由于系统会为switch的不同分支保持函数容器(即使它在某次工作流中不被调用)，switch的处理方式与parallel相同

     - Foreach

       foreach步骤并行地运行所有涉及到的任务。DAG parser将所有运行任务视为一个节点，以所有任务的数据传输延时的最大值作为foreach步骤的输出边权

     对于Parallel、Switch和Foreach步骤，DAG parser都在开头和结尾添加上Virtual node，用于保持划分时的原子一致性(virtual node在划分时怎么处理)。得到的结果如下图

     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220913210504277.png" alt="image-20220913210504277" style="zoom:50%;" />

     **将foreach步骤视为同个结点的话，它的输出应当如何考虑**

  2. Graph Partitioning in FaaS Context(FaaS下的图分解)

     对于一个工作流图，它包含了控制平面(运行顺序)以及数据平面(数据依赖)。由于在serverless环境下，因以下情况的存在，数据平面和控制平面可能不是对等的，因此图的分解应该考虑到数据平面的动态性

     - 存在自扩展和容器复用

       由于FaaS支持函数自动扩展实例以及复用热容器，工作流DAG上的单个控制节点可能运行着不同数量的实例。FaaSFlow中对每个函数节点引入了𝑆𝑐𝑎𝑙𝑒 (𝑣𝑖)，表示了一个函数节点vi的平均实例运行数量，用于在分配阶段进行决策。这个平均数量会根据上次迭代的反馈进行更新

     - foreach步骤中存在多个实例

       foreach步骤在FaaSFlow中被视为单个节点(原子操作)，但在运行过程中可能会产生多个实例运行任务。FaaSFlow引入了𝑀𝑎𝑝(𝑣𝑖)，表示foreach步骤中平均的运行数量(其他步骤的𝑀𝑎𝑝(𝑣𝑖)值为1).该平均数量也会根据反馈进行更新

     FaaSFlow利用𝑆𝑐𝑎𝑙𝑒(𝑣𝑖)和𝑀𝑎𝑝(𝑣𝑖)来处理控制平面和数据平面的差异。当工作流出现重大性能下降或违背Qos时，将会再次触发图分解工作，将工作流重新划分给各个节点。由于在工作流首次调用时，𝑆𝑐𝑎𝑙𝑒(𝑣𝑖)和𝑀𝑎𝑝(𝑣𝑖)是未知的，因此首次划分将采用hash-based的方法将工作流划分给不同的工作节点。FaaSFlow会统计每次迭代的运行数据，修改DAG中的边权和节点值(𝑆𝑐𝑎𝑙𝑒(𝑣𝑖)和𝑀𝑎𝑝(𝑣𝑖))，为之后的调度提供依据

  3. Function Grouping and Graph Scheduling

     由于最优的图分解问题是NP难问题，为了提高分解和调度效率，本文采用了贪心的分组算法，找到图分解问题的最优解或次最优解。具体算法流程为(对算法进行解读)

     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220914100115018.png" alt="image-20220914100115018" style="zoom:50%;" />

     Data：Cap[node]表示的是每个节点剩余能创建的容器数，Quota(G)表示的是为图G分配的内存限额

     **这里根据图G的计算配额来分配应当是不合适的(除非两个函数之间的函数交换不会太大)，考虑如果最长边恰好等于Quota(G)，那么它的判断将会是能够容纳，但事实上一个宿主机无法有这么大的内存来容纳这个边**

     1：将每个函数单独作为一个组，并将组随机分配到节点上。W表示每个节点分配的函数，S为分组情况

     5：将边权从大到小排序，表示贪心地尽可能让边权大的函数放在同个节点。用flag表示产生新的分组

     6-25：遍历所有的边

     7-9：找到该边的起点和终点所在的组，如果起点和终点已经在同一组则不需要继续考虑

     10-12：分别计算出起点所在的组和终点所在的组需要创建的实例数n，并预先将两个组从宿主机中取出(这里应当是Cap += nstart，表示start所在的节点容量增加nstart，如果分配失败应当减回这个值表示放回)。检查将两个组进行合并是否有节点能够容纳

     13-18：如果起点的存储方式为DB，则判断当前内存是否还能容纳该边(边代表的数据量是否高于内存)。如果足够时将边改为用内存交互(MEM)。

     19-20：判断新的组内是否存在有竞争关系的函数，有则不能构建成组(竞争关系cont可以通过其他研究的方法进行计算，本文不再引入新的计算策略)

     21-24：确定能够形成新的组后，找到合适的节点容纳新的组，并修改相应的参数(包括Cap，S等)

     26：反复循环直至没有新的组出现

     **以上的所有continue操作都表示两组不能进行合并，因此应当代表回滚操作，把修改的值改回原值；个人认为该算法没有写完整，分组时应还需要考虑到Map(vi)的情况，即foreach步骤，因此第10步Scale(si)求和应当改为Scale(si)\*Map(si)求和**

- Per-Worker Workflow Engine

  与MasterSP不同的是，在WorkerSP下，每个worker节点都被分配了对应的子图并能够独立地触发和调用函数，而不是等待master节点分发任务。此时，只有函数运行状态在worker节点间直接传递

  1.  Decentralized Engines(具体的原理已经在RATIONALE OF WORKERSP/FAASTORE提及)

     对于传统的工作，去中心化调度器(如Omega、Sparrow)通常关注于采用合理的资源分配策略来运行任务，而不关注任务之间的数据依赖关系等。而对于serverless工作流，它主要关注于运行函数状态的维护以及令函数以规定的顺序运行，因此高效的状态同步和通信是serverless下去中心化调度器的主要关注点

     对于WorkerSP下的去中心化引擎，它在迭代开始之时会被分配最新的运行子图，并在运行过程中自行触发和调用函数，维护函数的运行状态(内部函数同步高效)，以及向其他worker发送函数状态(发送数据量小，高效通信和状态同步)。为了降低内存消耗，在某轮工作流的所有函数调用完成后，应当删除这轮的State对象，且在所有函数实例被回收后(新的一轮迭代或是移除workflow)，删除对应的Workflow对象

  2. Red-Black Deployment and Management

     当Graph Scheduler根据运行的反馈指标，发现工作流出现重大性能下降或违背Qos时，将会重新更新分配给每个worker的子图。FaaSFlow采用了Red-Black Deployment来对子图版本进行管理，即确保只有新的子图中的函数会被触发，而运行旧的子图的容器会在运行完所有的函数任务(函数任务返回状态)后被回收(如果某个函数同时存在于新旧子图，那么运行它的容器应当不会被回收)。在某个函数任务被触发时，worker engine将其加入队列中等待容器调用执行。如果没有运行该函数的容器能够处理该任务时，将会触发冷启动，扩展出新的运行该函数的容器

- Memory Reclaimation in FaaStore

  由于RPC对有效负载的数据大小有限制，函数在进行大型数据交换时，需要依靠远端的数据库服务，这带来了巨大的传输开销。因此，FaaSFlow支持同个宿主机上的函数通过内存进行大型数据缓存，并且为每个工作流调用指定专用的内存空间

  1. Adaptive In-Memory Storage Quota(自适应的内存存储配额)

     虽然将数据缓存在内存中能减少同宿主机之间函数的通信开销，但是这也占用了更多的内存资源。如果内存存储配额分配过大，会给内存造成更大的压力，甚至可能出现换页或是out-of-memory的情况。因此，FaaSFlow谨慎地考虑了这个问题，并设置合适的配额为函数之间通信提供便利

     由于一个函数不能够完全地利用上容器的内存空间MEM(vi)，因此我们可以利用过量配置的内存空间来支持数据的传输。假设一个函数的内存使用为S，那么我们可以从中回收MEM(vi)-S-μ的空间，其中μ表示的是为突发情况保留的内存空间。依此，我们能够从图G中回收的内存空间为

     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220914172737895.png" alt="image-20220914172737895" style="zoom:50%;" />

     **回收的内存空间中也没有考虑Scale的问题，推测从同个函数自动扩展出的实例都用同个内存空间来缓存数据(在Function Grouping and Graph Scheduling中考虑内存空间是否能容纳边也没有考虑Scale)**

  2. Memory Reclaimation(内存回收)

     内存的回收如下图所示。在FaaStore回收过量配置的内存空间之前，它首先申请一个相等于Quota(G)大小的内存，并将它和相应的WorkflowID相关联。接下来，通过重新给容器设置新的cgroup组，从而将回收的内存从容器中释放。这些步骤确保了分配出的内存是专用于特定workflow的，宿主机上所有相同workflow的函数都可以共享这块内存

     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220914181746387.png" alt="image-20220914181746387" style="zoom:50%;" />



#### EVALUATION

实验以如下的四种科学性工作流以及四种实际应用作为输入基准，将它们运行在8节点的集群上。其中一个节点作为master节点，部署了Graph Scheduler并产生工作流调用，而剩余7个节点作为worker节点，部署distributed workflow engines和FaaStore来执行计算。实验采用CouchDB作为远端的key-value存储，并利用Redis作为内存存储

<img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220914200719451.png" alt="image-20220914200719451" style="zoom:50%;" />

实验分为以下几个部分

1. 通过采用一个闭环控制的客户端线程，确保系统中同时至多只有一个工作流调用，从而能直接分析出调度时延和数据传输时延，避免冷启动的影响。在测试调度时延时，分别发送不同输入基准的请求，计算平均的调度时延，发现FaaSFlow比传统系统HyperFlow-serverless具有更低的调度延时，且在应用调度延时上降低了74%(原因在于几乎消除了master和worker之间的信息传递，通过节点间TCP传输的只有函数的运行状态)；在数据传输方面，对比FaaSFlow和HyperFlow-serverless的数据传输时延(包括所有的数据传输)，发现FaaSFlow大量的降低了数据传输的时延，原因在于FaaSFlow使同个宿主机的函数可以通过内存进行数据交互，减少了通过远端存储进行数据传输的时延

2. 采用开环控制的客户端线程(同时可能有多个工作流调用)，在考虑上排队时延和冷启动时延的同时，计算端对端的时延和吞吐量。实验中通过以不同的速率向系统发送不同基准输入的workflow调用，共发送1000条调用，得到下图数据。可以看到FaaSFlow的工作流平均处理时延在不同应用下都低于HyperFlow-serverless。特别地，对于Cyc和Gen，由于网络带宽的限制，有大量的应用超时，而FaaSFlow通过本地数据交换，避免了这个问题

   <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220914210932411.png" alt="image-20220914210932411" style="zoom:50%;" />

3. 为每个输入基准维护一个闭环控制的客户端线程，并行提交调用，同时确保潜在的冷启动不会影响排队延迟，从而避免对端到端延迟的级联影响(测量消除冷启动影响后的端对端延时)。本实验主要在于检测多个不同的workflow同时运行带来的性能影响，分别对比不同基准输入在共同协作和单独运行下的运行情况。通过同时运行这8个benchmark，并计算出相应的端对端延时，发现HyperFlow-serverless中，多个benchmark因为竞争共享的网络带宽资源，性能大量地下降。相比之下，FaaSFlow能够显著地降低这种影响，因为其使用内存存储数据减少了带宽的使用
