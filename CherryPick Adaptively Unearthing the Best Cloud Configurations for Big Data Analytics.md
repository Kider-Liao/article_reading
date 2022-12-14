## CherryPick: Adaptively Unearthing the Best Cloud Configurations for Big Data Analytics

#### Abstract

由于具有数以十计的可供选择的VM类型和更多可能的集群大小，采用正确的配置在云上执行重复性大数据分析工作是非常困难的。错误的配置选择可能造成性能损失和增加大量的开销。然而，自动地以较低的开销为各种不同的应用和云配置选择正确的配置是非常具有挑战的。CherryPick 是一个利用贝叶斯优化为各种应用程序构建性能模型的系统。构建出的模型足够精确，且这些模型只需进行几次测试即可将最佳或接近最佳的配置与其他配置区分开来

**对性能模型的解释：An analytical model used to predict various performance metrics such as speedup of search request response time, number of computers needed for optimal search times；个人理解是，性能模型提供了权衡一个应用的性能指标的方式，通过性能模型的指导和对应的配置，能够预测出配制对应的性能指标(通过向模型输入备选的云配置，获得预测的性能指标)**



#### Introduction

在云上运行的大数据分析正在快速发展，并逐渐在各个工业领域起到了关键作用。许多演进式的技术不断被提出，例如MapReduce、Deep Learning等，用于支持大量不同的数据分析任务。这些大数据分析应用在云上运行的重要载体为VM，但由于不同分析任务的运行方式和资源需求不同，它们最合适的云配置(如虚拟机的数量和虚拟机的性能配置)不能简单地统一(不能采用统一的云配置管理所有的分析任务)

为运行的大数据分析应用选择最优的云配置是非常重要的，这有助于提升应用的服务性能以及业界竞争力。举个简单的例子就是，不恰当的云配置可能会导致在实现了相同的性能指标下，成本为恰当配置的12倍。对于重复在类似的工作负载上运行的重复性工作，正确的云配置能够节约更多的成本。然而，由于我们期望我们对云配置的选择工作能够同时达到精确、低开销、高适应性，选择最优的云配置变得更加地困难。精确、低开销、高适应性这三点可被解释为

- 精确性：由于应用的性能和开销与云实例、输入负载、内部工作流和应用配置等具有重要的联系，想要采用直接的方法对这种关系进行建模，从而精确地选择最优配置是非常困难的
- 低开销：采用蛮力搜索最优云配置是非常昂贵的，因为云供应商提供了大量的不同类型的云实例，且需要对采用的集群的大小进行选择
- 高适应性：由于人工通过学习应用的内部架构来构建性能模型是困难且不可扩展的(应用架构和依赖多样)，需要设计具有通用性的方法对各种应用进行自动化建模并选择最优配置

现有的解决方案并不能完全解决上述所有挑战，例如Ernest只训练了用于描述ML应用的模型，且只能在特定的实例类型下选择VM数量和大小。相比之下，CherryPick是一个挖掘最佳或接近最佳云配置的系统，可最大限度地降低云使用成本、保证应用程序性能并限制重复性大数据分析作业的搜索开销(推测是配置的搜索开销)，并全面地考虑到了VM数量、CPU数量等多种配置。CherryPick的核心思想是通过对应用进行精确的性能建模，使得我们可以将最优解或接近最优解的方案从其他配置方案中分离出来。另外，由于CherryPick目的不在于找到最优解，而是容忍一定的不准确性，因此CherryPick具有低开销和高适应性的特点(只需要少量的样本且不需要将应用相关的见解嵌入到建模中，即对于任何应用配置，CherryPick都能找到合适的云配置，对其他的相关工作进行了补充)

接下来的Introduction主要是对CherryPick的实现方法的简单介绍，最主要的是介绍如何使用贝叶斯优化进行建模，具体的描述还是得看第三节的具体说明



#### Background and Motivation

本节主要讲解了选择良好云配置的优势和挑战，并介绍两种strawman solutions

注：A straw man generally refers to the first rough proposal created for criticism and testing in software development. It initializes discussions and feedback to develop a new and better proposal(strawman solutions指的是初步提出的粗略的解决方案，为后续进一步提出良好的方案做铺垫)

- benefits(优势)：良好的云配置可以很大程度地降低大数据分析任务的开销，且对于重复性分析任务而言，找到恰当的云配置具有更重大的意义。本文在四个不同的应用上做了实验，并对比了在不同云配置下，平均开销、最低开销和最大开销之间的差异，具体结果为

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220829104248886.png" alt="image-20220829104248886" style="zoom:50%;" />

  本文提出的方法只用于重复性分析任务(有将近40%的分析任务都是重复性的，因此本方法依然具备先进性)，使一次云配置的搜索开销可以均摊到接下来的多次运行中

- challenge(挑战)：找到最合适的云配置带来的挑战主要有以下几点

  1. Complex performance model(性能模型的复杂性)

     资源的变化与运行时间的变化不存在线性关系，当资源数量提升到一定程度时，资源提升可能是会细微降低运行时间(例如分配的存储已经大于应用需求时，再增加分配的存储空间也无济于事)；另外，某一种云配置下的应用性能不是确定的。应用在同一种云配置下运行多次，其运行时间会有浮动(由于网络等原因，出现噪声)。鉴于这些原因，使得对性能模型的建模变得复杂

  2. Cost model(代价模型)

     云上的资源通常根据资源的使用时间来进行收费，并与资源的数量有关。增加应用的配置资源有利于降低应用的运行时间，但同时也会增加应用的开销。恰当增加资源数量往往能够降低开销，因此我们需要找到运行时间和资源开销的平衡点，在降低开销的同时获得可观的性能。如下图中的Regression on Spark，在开始时，资源增加能够大幅降低运行时间，因此开销降低。但资源增加到一定数量时，对运行时间的改善不多，使得总体开销增加

     <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220829111257363.png" alt="image-20220829111257363" style="zoom:50%;" />

  3. The heterogeneity of applications(应用的异构性)

     由于应用的输入负载、运行流程、资源需求等有较大差异，使得对于同样的云配置，在不同应用上的效果不同，因此不同的应用对云配置的变化的响应不同(对于计算密集型应用，期望有更多的CPU，CPU增加时性能增加且开销下降；对于内存密集型应用，期望有更多的内存空间，内存增加时性能增加且开销下降)

- strawman solutions：通过建模和搜索的方式，找到接近最佳的云配置

  1. Accurate modeling of application performance

     一种方法是对应用的性能进行准确地建模，并依据这个建模推算出最佳云配置。构建一个对多数应用都适用的模型是非常困难的，因为不同应用的内部结构和配置不同，我们的模型需要对特定的应用结构进行修改，从而使模型更加准确；对每个应用都构建一个模型是不现实的，因为应用数目过多，会极大地增加开发者的工作量。因此，对应用的性能直接进行准确地建模是不合理的

     **本方法与CherryPick的不同在于，CherryPick通过自动化的方式，依据少量的搜索，不考虑应用内部结构，为每个应用都能构建一个相对准确的模型。而本方法旨在构建出准确的模型，这种模型往往与应用内部结构有关，难以自动化进行构建**

  2. Static searching for the best cloud configuration

     另一种方法是穷举所有可能的云配置，并根据应用的表现选择最佳的云配置进行应用布置。这种方法不需要对应用构建性能模型，但由于云配置的组合方式众多，可能需要数百次应用的运行才能搜索到最佳的云配置，造成了巨大的开销。另外，由于性能存在波动，可能需要对某种配置方式进行多次尝试，才能了解到配置的真实性能，这样更加增加了开销



#### CherryPick Design

CherryPick的设计遵循的原则是利用小部分的信息去解决问题，这体现在对于CherryPick构建的预测性能的模型，只能通过小部分的云配置作为样例调整模型的准确度，否则会出现开销过大的问题。因此，CherryPick构建的模型往往无法准确地根据给定的云配置预测出对应的性能。然而，该模型可以在少量的用例中找到良好的配置，是因为经过这些用例训练出的模型已经足以将好的配置与其他配置分离出来

与静态搜索的方式相比，CherryPick可以动态地根据性能模型当前的理解和置信区间来调整搜索方案，具体体现在动态选择一个最合适的配置用例以最好地区分不同配置的性能并消除不必要的试验。另外，通过性能模型，我们可以在拥有足够小的置信区间后决定搜索的停止，从而能够比静态搜索更快地搜索到良好的云配置方案。CherryPick的工作流程可以表示为

<img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220829164456663.png" alt="image-20220829164456663" style="zoom:50%;" />

以下描述CherryPick实现的细节

- Problem formulation(问题描述)

  我们令向量x表示云配置方案，包括实例类型、CPU、RAM等资源，T(x)表示在资源配置x下应用任务的运行时间，需要限制在某个可容忍的时间内，P(x)表示资源配置x在单位时间下的开销，C(x)表示总开销(假设采用的集群大小是固定的)，那么我们的问题可以简单地表示为

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220829171842457.png" alt="image-20220829171842457" style="zoom:50%;" />

- Solution with Bayesian Optimization(贝叶斯优化BO的简介)

  贝叶斯优化是处理类似上述优化问题的框架，其目标在于在目标函数不确定而可以通过实验观察得到的情况下，根据取样和计算对模型进行训练，使其能够计算出目标函数的置信区间(这里的置信区间不特指某个点，而是对整个坐标域都有置信区间)。一个贝叶斯优化的例子如下(虚线表示实际值，蓝色区域表示各点的置信区间，在两个样本点下通过BO取得置信区间)

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220829180223285.png" alt="image-20220829180223285" style="zoom:50%;" />

  BO通过一个预定义的采集函数来决定下一个样本点的选择，且这个采集函数会根据获取得到的置信区间进行更新。CherryPick的各个步骤都离不开BO的预测和置信区间训练，包括步骤2中根据样本点利用BO更新C(x)的置信区间和采集函数、步骤3中利用采集函数选择最合适的样例进行训练、步骤4中根据BO得到的置信区间决定是否获得了最优的云配置。此外，BO还具备适应观测噪声的能力，在外界存在噪声的情况下，其依然能够推断出目标函数的置信区间，这对于我们处理云中存在的不确定性具有重大的意义

- Why do we use Bayesian Optimization?(采用贝叶斯优化的原因)

  1. 贝叶斯优化是非参数的(不对数据的分布作出假设)，它不要求对应的目标函数属于某种预定义的形式，这使得贝叶斯优化能够被应用于大量的应用和云配置

     非参数模型在于对分布本身的形式进行推测，参数模型在于假设数据符合某种分布并进行验证

  2. 由于贝叶斯优化通过选择最有价值的样本对模型进行训练，使其只需要少量的样本就可以推测出接近最优的解

  3. 贝叶斯优化具有处理不确定性的能力。CherryPick需要解决两个主要的不确定性问题，包括少量数据造成模型不够准确且可能出现重大误差、云环境的不稳定性(网络拥塞等问题)导致同个应用的多次运行会有不同的性能结果。然而，由于BO计算的是应用性能的置信区间，其覆盖了应用的不稳定性导致的不确定区域，使其在不确定性下依然能够指导样本的选择，并找到合适的配置

  其他的各项技术与贝叶斯优化相比，缺失了部分的优势

  1. 线性回归和线性强化学习无法应用于非线性模型，使其不能应用于所有的应用
  2. 深度神经网络、协方差矩阵自适应进化策略等非参数模型训练技术需要大量的样本构建出模型
  3. 强化学习难以处理不确定性，且需要大量的样本构建模型

- Design options and decisions(具体设计)

  **这一部分涉及到贝叶斯优化等具体的知识，目前还没有相关的知识储备，需要进一步学习。进行了一轮阅读之后，对采集函数、每个点的分布函数、准随机序列(尽量填补空间的随机)等有了基本的了解**

  **在简单地对这一部分内容进行阅读时进行的一个思考：是否阅读这些论文最主要先理解其方法论，即什么方法可以解决什么问题，在具体研究中，如果认为该方法适用于研究，则再进行具体学习，因为整个方法涉及到递归的知识储备，需要花较长的时间学习**

  **本节还对将云的配置进行译码做了一定的解释，说明了如何将云的配置映射到我们具体使用的函数中。虽然这里的云配置考虑到了多种配置信息，包括CPU、内存、VM数目、core数目等，但是为了缩小贝叶斯优化的搜索空间，对许多配置信息进行了概化和离散化，例如将CPU配置空间划分为fast和slow两部分。这种划分方式既保留了重要的信息，同时缩小了搜索空间，具有重要意义(但是我个人看来，这种方法是否过于概化了云的各种配置，使即使我们获得了推荐的配置信息，仍然有较大的配置空间。这种配置方式的效果可能要在Experiment部分进行验证，又或者说事实上云供应商提供的配置，使得fast、slow这类信息对应的配置种类不多，影响不大，但是如果全部考虑的话会使得需要考虑的空间指数增长)**

- Handling uncertainties in clouds(处理不确定性)

  事实上，在实际的云环境中，由于用户之间共享环境造成了一定的干扰，且可能出现系统失败或资源过载等问题，因此同样的处理任务的多次运行获得的运行时间和开销不同。因此，对于目标函数C和时间T的分布，应当引入对应的噪声ε(视为正态分布)，得到

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220830104643387.png" alt="image-20220830104643387" style="zoom:50%;" />

  然而，由于C~(x)不是正态分布的，不能简单地根据观测数据C~(x)推测得到C(x)的置信区间。一个直接的解决方案是对同个样本做多次测试，取得的平均值即为C(x)的值。然而，这种方法造成了巨大的开销。文中具体使用的方法是将求目标函数C(x)的最小值变为求log(C(x))的最小值，其目的在于此时对于观测值C~(x)，其表达式变为

  <img src="C:\Users\pentakill\AppData\Roaming\Typora\typora-user-images\image-20220830105749688.png" alt="image-20220830105749688" style="zoom:50%;" />

  此时log(1+ε)可以被视为具有正态分布的观察噪声，则logC~(x)为观察噪声log(1+ε)与观察值logC(x)的结合，从而可以被解决

  **由于这方面概率论知识不足，个人感觉是将ε与C(x)的关系从乘法变为了加法，相当于把两者的分布分离开了，使得在噪声下的目标函数也是可预测的。这有点像数图中去除高斯噪声的方法，先求log后采用均值滤波消除高斯噪声**



#### Implementation(系统设计部分)

整个CherryPick系统设计分为如下几个部分

- SearchController

  SearchController主要有三个功能。首先，它通过接收用户提供的具有代表性的应用负载、最终目标、性能限制，从云中的各种配置中筛选出合适的候选配置，并将这个信息提交给Bayesian Optimization Engine。其次，它负责将应用负载下载到云端，主要操作是启动标准虚拟机、下载工作负载和应用、捕获当前包含应用及负载的镜像。另外，SearchController还通过接收Bayesian Optimization Engine的结果，决定是否已经获得了最优或接近最优的配置

- CloudMonitor

  CloudMonitor主要用途在于定期在云上运行CherryPick给定的基准工作负载，测试当前各个云上的噪声情况，并将其反馈给Bayesian Optimization Engine进行推理预测

- Bayesian Optimization Engine

  Bayesian Optimization Engine是实行贝叶斯优化工作的核心部件，它根据接收的SearchController传达的候选配置，通过标准的贝叶斯优化库进行计算，并选择下一个样本配置，提交对应的运行请求到Cloud Controller，进行样本的运行。Bayesian Optimization Engine由于采用了对应的python库，其主要工作在与和其他模块的沟通，并提供一个通用的接口，且调用Cloud Controller的接口进行样本的运行和结果的采集

- Cloud Controller

  Cloud Controller是一个通用部件，它负责了整个CherryPick系统对云资源的操作。Cloud Controller封装了不同云上的管理API，通过定义一个标准接口，使得其他模块能够使用统一的入口来进行资源的操作和VM的创建等。在接口处接收到对应的指令后，Cloud Controller再通过调用云提供商提供的API进行VM的创建和管理。另外，Cloud Controller还需要能够直接与创建的VM进行通信(利用SSH)，从而将工作负载和应用加载到创建好的虚拟机上并通过命令要求VM执行对应的计算任务
