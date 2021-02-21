+++
title = "介绍 Hydro - 新一代的 Serverless Computing"
slug = "hydro"
date = "2021-02-17T13:03:57+08:00"
description = "Hydro 是 Joseph M. Hellerstein 教授在 UC Berkeley 的 RISELab（算是 AMPLab 的继承者）开展的项目。"

tags = ["system", "cloud computing", "serverless computing"]
+++

[Hydro](https://hydro-project.github.io/) 是 [Joseph M. Hellerstein](https://dsf.berkeley.edu/jmh/index.html) 教授在 UC Berkeley 的 [RISELab](https://rise.cs.berkeley.edu/)（算是 [AMPLab](https://amplab.cs.berkeley.edu/) 的继承者）开展的项目。要了解 Hydro，可以从他们最近的一篇总结性博客开始看起：[The State of the Serverless Art](https://medium.com/riselab/the-state-of-the-serverless-art-78a4f02951eb)，然后再看它主页上罗列的论文。

## 背景 - 无服务器计算

Hydro 项目的目标是**创造一个适合于云计算的编程环境**。目前大部分人是如何做云计算的呢？我们可能会开一定数量的 Amazon EC2 虚拟机，然后在上面安装依赖包，启动我们的程序。在这中间，我们要处理很多机器管理、容错、通讯的问题。在 Joe 教授的眼中，在云上如此编程，类似于在计算机发展的早期时代我们直接写汇编代码，非常的落后。那么在云计算的时代，正确的姿势应该是什么样的呢？UC Berkeley 的一些系统大佬在 2019 年的 Berkeley view[^berkeley-view2] 中指出，**云计算的未来是无服务器计算（serverlesss computing）**。在无服务器计算中，用户只要定义需要计算的函数，计算所需的资源都有云计算提供商来管理，类似的服务有 Amazon Lamda、Azure Functions 和 Google Cloud Functions。话说，基本同一批的系统界大佬在 2009 年出过一个云计算的 Berkeley view[^berkeley-view1]，其中很多看法都已经被验证；2019 年的这版是他们的一个回顾与更新。

问题是，现在无服务器计算并不完美，Joe 在一篇论文中指出无服务器计算相比之前的云计算是「进一步，退两步」[^serverless]：

- **进步：自动伸缩（autoscaling）**。用户不需要关心有多少的虚拟机在进行计算，云计算提供商会自动的扩缩容。
- **退步：数据访问慢**。在现有的无服务器计算中，用户只能访问 Amazon S3 和 DynamoDB 这样的服务，这些存储服务或数据库比本地 SSD 之类的存储介质可慢多了。
- **退步：无法进行分布式计算**。要进行分布式计算，需要进行通信，而基于存储的通信太慢了，现有的无服务器计算框架中又不能直接进行网路通信。

## 解决之道 - Anna 和 Cloudburst

当然，Joe 的目标并不是贬低无服务器计算，而是希望通过 Hydro 项目构建出一个他理想中的无服务器计算平台：**stateful serverless infrastructure**。首先，Joe 和他的博士生 [Chenggang Wu](http://cgwu.io/) 构建了名为 [[Anna]] 的 key-value 存储系统（Anna 分别获得了 ICDE'18 和 VLDB'19 的最佳论文奖）。下面说一下 Anna 的主要特性：

- Anna V0 是一个 in-memory key-value store[^anna-v0]
  - 两个特点有
    - 速度快：Anna 基于 actor 模型（每个 actor 是 pin 到一个 CPU 核上的 thread），单机和分布式间的 actor 之间没有 coordination，减少了因为 shared-memory 造成的 contention 和 cache miss，达到了最快的速度。
    - 支持多种一致性（consistency）模型，包括 causal consistency 和 eventual consistency。Anna 利用了 lattice-based representation[^lattice] 和 CALM 定理[^calm] 实现了这些一致性模型。这些理论是基于 Joe 之前的 [Boom](http://boom.cs.berkeley.edu/) (Berkeley Order Of Magnitude ) 项目的一些工作。
  - 简单来说，Anna 利用 consistency hashing 将 key 划分给不同的 actor，为了容错会做 replication。写入的模式是 multi-master 的，然后 actor 之间会定期通过 gossip 来传播更新。为了解决一致性的问题，value 要加上 timestamp，这样就可以通过 last-writer win 至少保证 eventual consistency。最后，通过 lattice-based state representation，来实现一些更强的一致性模型。但为了追求性能，Anna 是不能支持 linearizability 的。
- Anna V1 又加了两个特性[^anna-v1]
  - 自动伸缩（autoscaling），可以根据 workload 自动的扩大或缩小集群的大小。
  - 支持多层级的存储介质，比如 RAM、SSD、硬盘，这样就使得 Anna 的应用范围更广泛了，比如可以在一些场合代替 Amazon DynamoDB 这样的分布式数据库。

基于 Anna，Joe 和他的另一个博士生 [Vikram Sreekanti](https://www.vikrams.io/) 构建了一个名为 [[Cloudburst]] 的 stateful serverless infrastructure，发表在 VLDB'20[^cloudburst]。Cloudburst 试图解决之前提到的现有无服务器计算的两个缺点:

- **为运行在 Cloudburst 上的 function 提供状态（state）存储**，且满足 **logical disaggregation with physical co-location of compute and storage (LDPC)**
  - **Logical disaggregation**: 状态存储是基于 Anna 的 key-value store，抽象简单，性能强，并且计算和存储是分离的
  - **Physical co-location of compute and storage**: 同时为了提高计算效率，执行 function 的 EC2 虚拟机会做一些 cache
- **正在运行的 function 互相之间可以直接进行 TCP 通信**，允许分布式协作，提高计算效率
  - 这个是通过在 Anna 中存储每个 function 对应的 `IP:port`，然后直接建立 TCP 连接实现的

为什么 Joe 要先做 Anna 呢？因为他认为只有高性能的 Anna，才能支持 Cloudburst 的存储需求。在 Cloudburst 的论文中，Joe 和他的学生们，用实验证明了 Cloudburst 的性能优于 Amazon Lambda 与其它存储层，如 Redis/Amazon S3。

最后，Joe 和他的 Hydro 项目组还在不断的扩展和改进 Cloudburst：

- 提出并实现了无服务器计算的 atomic fault-tolerance 概念[^fault]
- 基于 Cloudburst，构建了名为 Cloudflow 的机器学习模型在线推理系统[^prediction]

## 参考文献

[^berkeley-view1]: Armbrust, Michael, Armando Fox, Rean Griffith, Anthony D. Joseph, Randy H. Katz, Andrew Konwinski, Gunho Lee, et al. “Above the Clouds: A Berkeley View of Cloud Computing.” _UCB/EECS_, 2009. https://doi.org/10.1145/1721654.1721672.
[^berkeley-view2]: Jonas, Eric, Johann Schleier-Smith, Vikram Sreekanti, Chia Che Tsai, Anurag Khandelwal, Qifan Pu, Vaishaal Shankar, et al. “Cloud Programming Simplified: A Berkeley View on Serverless Computing.” _UCB/EECS_, 2019.
[^serverless]: Hellerstein, Joseph M., Jose Faleiro, Joseph E. Gonzalez, Johann Schleier-Smith, Vikram Sreekanti, Alexey Tumanov, and Chenggang Wu. “Serverless Computing: One Step Forward, Two Steps Back.” In _CIDR_, 2019.
[^calm]: Hellerstein, Joseph M., and Peter Alvaro. “Keeping CALM: When Distributed Consistency Is Easy.” _ArXiv_, 2019.
[^lattice]: Conway, Neil, William R. Marczak, Peter Alvaro, Joseph M. Hellerstein, and David Maier. “Logic and Lattices for Distributed Programming.” In _SoCC_, 2012. https://doi.org/10.1145/2391229.2391230.
[^anna-v0]: Wu, Chenggang, Vikram Sreekanti, and Joseph M. Hellerstein. “Autoscaling Tiered Cloud Storage in Anna.” _PVLDB_ 12, no. 6 (2019): 624–38. https://doi.org/10.1007/s00778-020-00632-7.
[^anna-v1]: Wu, Chenggang, Vikram Sreekanti, and Joseph M. Hellerstein. “Autoscaling Tiered Cloud Storage in Anna.” _PVLDB_ 12, no. 6 (2019): 624–38. https://doi.org/10.1007/s00778-020-00632-7.
[^cloudburst]: Sreekanti, Vikram, Chenggang Wu, Xiayue Charles Lin, Johann Schleier-Smith, Joseph E. Gonzalez, Joseph M. Hellerstein, and Alexey Tumanov. “Cloudburst: Stateful Functionsasaservice.” _PVLDB_ 13, no. 11 (2020): 2438–52. https://doi.org/10.14778/3407790.3407836.
[^fault]: Sreekanti, Vikram, Chenggang Wu, Saurav Chhatrapati, Joseph E. Gonzalez, Joseph M. Hellerstein, and Jose M. Faleiro. “A Fault-Tolerance Shim for Serverless Computing.” In _EuroSys_, 1–15, 2020.
[^prediction]: Sreekanti, Vikram, Harikaran Subbaraj, Chenggang Wu, Joseph M. Hellerstein, and Joseph E. Gonzalez. “Optimizing Prediction Serving on Low-Latency Serverless Dataflow.” _ArXiv_, 2020.
