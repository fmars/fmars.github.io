## Pipe-SGD: a decentralized pipeline SGD framework for distributed deep net training
* https://arxiv.org/abs/1811.03619
* 24/04/21
* Centralized architecture: parameter server which aggregates gradients and update weights, in either sync or asysnc manner
* Decentralized architecture: all-reduce across ranks to synchronize weights
* Use pipeline to overlap computation (opt step and bwd) and communication (sync grads)
* Use compression/quantization to speed up gradient communication

## A generic communication scheduler for distributed DNN training acceleration
* https://dl.acm.org/doi/10.1145/3341301.3359642 
* 24/04/21
* A generic communication scheduler works for both sync and async training, all of pytorch, tenorflow, etc frameworks
* Implemented an hook layer between library api and backend engine to achieve comm op rescheduling
* Writing doesn't seem interesting. Only read the high level ideas


## PyTorch: an imperative style, high-performance deep learning library
* https://arxiv.org/abs/1912.01703
* 24/04/21
* 4 major trends in deep learning since 1960
  * array-based programming
  * automatic differentiation
  * open-source python ecosystem
  * hardware accelerator
* interoperability: PyTorch and NumPy tensors exchange data without any data copy
* Cache allocator: one-pool-per-stream design provides simple while efficient implementation that allows CPU tensor reallocation running asynchronous before GPU completion
* Reference counting: tensor memory release happens immediately when count reaches zero


## Long-Short Transformer
* https://arxiv.org/abs/2107.02192
* 24/04/10
* Full attention requires quadratic cost to context length
* Short-term attention: segment-wise sliding window, each token attends to tokens within the window, the half of previous and next windows
* Long-term attention: dynamic project. Use learnable weights to project K,V from (n, d) to low rank matrix (r, d), before attention
* Aggregating short and long term attention by concatenating K,Vs 

## Mixed precision training
* https://arxiv.org/abs/1710.03740
* 24/04/09
* Seems to be the foundational paper in this area
* Improve throughput of model training bound by arithmetic and memory bandwidth while not loss accuracy
* Tensor precision: weights, activations, gradients in fp16, master copy of weights (i.e. optimizer state) in fp32
  * Small valued grads would become zero when multiplied with learning rate
  * The ratio of weight value to grad value is very large. Grad would become zero in addition operation when right-shift to align
* Loss scaling: chain loss scale at the end of forward pass. Back-scale grads before optimizer step
* Arithmetic precision: use fp32 to accumulate intermediate results in fp16 vector dot-product and matrix multiplication. But the input and output types are still fp16

## LLM.int8()
* https://arxiv.org/abs/2208.07339
* 24/04/09
* Another solution to tackle outliers in activation like SmoothQuant
* Problem: extreme outliers of feature dimension in activation start to emerge when model scale exceeds 7B params. Those outliers have a significant impact on model performance. Current quantization solutions don’t handle it well
* Solution: 1) vector-wise quantization: row-wise quantize on X and column-wise quantize on W for Y=XW, 2) mixed-precision decomposition: outliers are highly systematic, concentrated on a few feature dimensions across the entire transformer. Decompose matmul, still apply fp16 to outliers while use int8 for the remainings

## SmoothQuant
* https://arxiv.org/abs/2211.10438
* 24/04/05
* Dynamic quantization -> quantize weights. Static quantization -> quantize both weights and activations
* PTQ: post training quantization -> quantize trained model for efficient inference
* QAT: quantization aware training -> simulate quantized inference behavior in training for better ML perf
* Quantization stores two scalers, scale and zero-point besides tensor
* Weight is usually easier to quantize than activation, which has more outlier values
* Tensor level, token level, channel level quantization
* Activation requires channel level quantization to be effective, but not supported by existing kernel and hardwares
* [it seems] propose to offline quantize weights to offset bias. Don’t really follow the details


## LAMB: Layer-wise Adaptive Moments optimizer for Big batch
* https://arxiv.org/pdf/1904.00962.pdf
* 24/04/02
* Scalability of SGD is limited by its inherent sequential nature
* Async training -> degraded ML performance
* Larger batch size: 1) hand-tuned LR during warmup, 2) scale LR to the order square root of batch size, 3) not work well up to certain batch size
* Hand-tuned hyperparameters -> adaptive learning rate
* For each layer's weights, LAMB calculates an individual LR based on the ratio of the weights to the square root of the second moment 

## PyTorch FSDP
* https://arxiv.org/abs/2304.11277
* 24/04/02
* PyTorch version of ZeRO
  
## PyTorch DDP
* https://arxiv.org/abs/2006.15704
* 24/03/31
* API: 1) non-intrusive so that DDP makes it almost like writing local training scripts, 2) interceptive allows various hooks to make program efficient
* Gradient reduction: 
  * grad = all_reduce(SUM)/world_size
  * gradient bucketing: 1) efficient NCCL comm by packing smaller params into a bigger bucket, 2) overlap comm of one bucket of params with backward computation of another bucket of other params
* Special handle: 1) not all params may participate in bwd. 2) not all ranks may involve the same set of params

## MegaScale
* https://arxiv.org/abs/2402.15627
* 03/30/2024
* Techniques used by TikTok to train LLM over 10,000 GPUs
* Efficient training
  * Algorithm: parallel transformer block, sliding window attention, LAMB optimizer
  * Overlap compute and comm: fuse all-gather/reduce-scatter (used for sequence parallel) with MLP computation in FFN
  * Efficient operator: kernel fusion
  * Collective comm group initialization: using redis over tcpstore
* Fault tolerance: RDMA/RNIC checkers, fast checkpointing
* Observability: heat map to detect slow node, visualize event trace across multiple ranks
* Experiments: stagger nodes are the norm. irregular GC 
* Lots of interesting papers in the reference section. Kind of like a good survey paper

## ZeRO-Offload
24/03
https://arxiv.org/abs/2101.06840
* Limitation of existing heterogeneous DL training: 1) exploit CPU memory but not compute, 2) designed for single GPU and not scale to multiple GPUs
* Solution: GPU for fwd+bwd, CPU optimizer, inline param update and sync back to GPU
* Implementation: CPU ADAM optimizer, one step delayed param update (not really sure why this help..)

## Efficiently scaling transformer inference 
24/03
https://arxiv.org/abs/2211.05102
* Measurement metrics: 1) latency, 2) throughput, 3) MFU
* Affecting factors: model size, sequence length, application requirements 
* Discussed various model partition strategies, and corresponding impact on computation, communication and memory cost.

## Sequence Parallelism
24/03
https://arxiv.org/pdf/2205.05198.pdf
* Megatron-LM (by Nvidia) implemented op sharding: combine vertical and horizental split of two matmul matrix to save one comm.
* Observation: the activitions passed through layers still takes significant amount of memory and propotionally increase to sequence length. On the other hand, the operations in between is sequence independent
* Two major techniques: 1) sequence parallel along the sequence length dimention for those activations, 2) tradeoff the recompute cost and memory saving to choose the right activations to checkpoint
  
## Flash Attention
24/03
* Standard attention is memory bandwidth heavy. Intermediate tensors require HBM access quadratically proportion to sequence length
* softmax computation can be tiled with small extra info stored
* Backward pass can recompute activations without saving those big tensors to HBM
* Standard attention requires O(Nd+N^2) HBM accesses while Flash attention requires O(N^2d^2/M) where M is the size of SRAM
## The wake-sleep algorithm for unsupervised neural networks
24/02
* Two problems in supervised learning: 1) requires a teacher to specify the desired output, 2) requires to communicate error information
* Wake-sleep: a bottom-up recognition model to learn intrisic patterns of the input data, a top-down generative model to reconstruct the input data. Interleave recognition and generative model steps.
* Wake-sleep, GAN, EM in Deep Retrieval, RLHF finetune, all seem similar?
* Quite a abstraction heavy paper. Needs at least some concrete examples before understanding what is the vast majority of paper talking about...

## PPO: Proximal Policy Optimization Algorithm
24/01
* de-facto algorithm used in reinforcement learning and RLHF
* Previous algorithms are either not efficiency or not stable. PPO achieves both at the same time
* It enforces no drastic policy model update, through a clipped surrogate objective, and adaptive coefficent for KL penalty
* Got the rough idea but probably only really understand 30% of the paper details.
* Surrogate objective, first order algorithm, conjugate gradient algorithm


## InstructGPT: Training language models to follow instructions with human feedback
24/01
* Pretrained model doesn't inherently align with users/human intent
* Pretrained model -> SFT on labeler demonstrated desired output -> RM training on labeler ranked different model outputs -> PPO (RL) optimize policy model
* Few-shot example, e.g. giving two examples of frog stories, and prompting the model to generate a new one

## Deep Retrieval: Learning A Retrievable Structure for Large-Scale Recommendations
23/12
* Statistic concepts: Likelihood Function, Log Likelihood Function, Maximum Likelihood Estimiation
* Recommender concetps: Collaborative Filtering, Matrix Factorization, Factorization Machine, Beam Search
* Modern recommendation system funnel: retrieval, early-stage ranking, ranking
* Collaborative filtering, and item collaborative filtering were early successful techniques
* ANN, MIPS, tree-based models are used to select high quality items from huge corpus with hundreds of millions candidates
* Deep Retrieval: 1) a efficient while accurate retrieval model, 2) D layers each with K nodes, 3) each item is associated with J paths, 4) first layer takes user emb as input, runs MLP and softmax, outputs layer emb, concatenates with original user emb and passes to next layer, 5) final similiarity score is the product of all probabilities of all layers
* SGD for model paramter learning (continuous objective). EM (Expectation Maximization) for item-to-path mapping learning (discrete objective)
  * Stochastic in SGD due to randomly select mini-batch out of global training dataset!
  * Latent cluster: each node in a layer is a latent cluster!
* Finally got some sense of how is probability distribution (in statistics) related to machine learning!

## Triton
23/12
* Challenges in CUDA programming 1) data transfer coalescing to efficienctly use memory bandwidth, 2) data management for efficient data reuse, 3) computation partitioning and scheduling for efficient parallel execution
* Hand written kernel requires significant expertise to be efficient, while facing the risk of low portability
* Multi-dimensional tiles are the center of data-flow analysis in Trition. Programmer works on program level granularity rather than treahd. Each kernel is single threaded, though automatically parallelized. This hides the details of concurrency primitives to compiler, e.g. shared memory synchronization, inter-thread communication, etc.
* Github tutorials are great learning materials. Still need more practice to fully understand Triton


## MPI: Message-Passing Inference Standard
23/12
* Easy to use, portable
* https://github.com/fmars/pbag/commit/a559bbebd328e06c98399d635760755b245465fe
* Blocking vs non-blocking, generic object vs buffer-like object
* P2p communication, broadcast
* gather(), scatter(), all_gather(), scatter_gather()/all2all()
* reduce(), all_reduce(), reduce_scatter()
## Distributed Representations of Words and Phrases and their Compositionality
23/12
* NeurIPS 2023 Test of Time Award
* Optimizations for Skip-gram model to improve both efficiency and accuracy
* Hierarchical softmax: reduce softmax computation complexity from O(n_vocab) to O(log(n_vocab))
* Negative sampling: introduce a noise distribution to pick negative samples to compute loss together with w_in and w_out
* Subsampling of frequent words: each word is discarded with probability proportional to its frequency in dataset
* Empirical results shown negative sampling outperforms others
* Seems to provide some explanation of why word embedding has addictive compositionality (e.g. Russion + river = Volga). Don't really follow though..
  
## Deep Learning with Limited Numerical Precision (Stochastic Rounding)
23/12
* Similar to FlexPoint, another approach uses low-precision tensor to train model with little accuracy loss
* Propose to use 16-bits fixed point number representation as <IL, FL>.
* Benefit of fixed point number is faster and less memory footprint.
* Stochastic rounding: the probability of up/down rounding is proportion to how is a number closer to ceiling and flooring number

## NVidia A100 Tensor Core GPU Architecture
23/12
* 3rd gen Tensor Core with Sparsity, TF32 boost perf, BF16/FP32 mixed precision boost perf
  * BF16 Tensor Core is now at the same throughput as FP16 (FP16 has more precision. Why BF16 slower??)
* Multi-Instance GPU (MIG) for virtualization
* 40GB HBM (1555GB/s) , 40MB L2 cache on-chip shared by SMs, and 192KB L1 cache for combined shared memory and L1 cache (L1 cache is essentially register file?)
* 12 NVLink for 600GB/s in total
* Sparsity: fine grained structured sparsity by 2:4 sparse pattern provides 2x throughput boost with little to zero accuracy loss at inference time
 * Fine-grained sparsity: zeroing outr specific weights distributed across the neural network
 * Coarse-grained sparsity: zeroing out entire sub-network
 
## ImageNet Classification with Deep Convolutional Neural Networks
23/12
* Overall: 5 CNN/3 FC + ReLU + 2GPU + local normalization + overlapping pooling + data augmentation + dropout
* ReLU: in terms of training time, saturating nonlinearities (e.g. tanh) are much slower than non-saturating nonlinearity (e.g. ReLU)
* Reduce overfitting: 1) artificically enlarge dataset using label-preserving transformation, a) extracting patches, b) PCA, 2) dropout
* Discussion: all of our experiments suggest that our results can be improved simply by waiting for faster GPUs and bigger datasets to become available.

## A Survey on Deep Learning Hardware Accelerators for Heterogeneous HPC Platforms
23/11
* HPC seems to focus a lot on hardware and also ML/hardware co-design
* General types of DNN: MLP, CNN (capture high-level representation), RNN (capture time-series info), Tranformer (recognize long-distance dependencies between data)
* P100, V100, A100, H100 specs. On A100, reported HBM mem bandwidth is 1.5TB/s. Observed 2.5TB/s (https://github.com/fmars/pbag/blob/master/pytorch/mem_bandwidth.py)
* DGX - Digital Graphic Workstation. Nvidia GPU servers with high bandwidth connections
* Sparsity: common sparse matrix storage formats include COO, CSR, CSC
* HBM is 3D stacked memory providing better storage, bandwidth and efficiency
* 
## Flexpoint: An Adaptive Numerical Format for Efficient Training of Deep Neural Networks
23/11
* Floating point format: https://github.com/fmars/pbag/blob/master/pytorch/floating_point.py
  * Sign, exponent, precision. Normalized and denormal values
* Assumptions: 1) in a DNN, activitaions, gradients, parameters have very different ranges, 2) ranges of values in DNN changes sufficiently slowly, 3) low bit-width quantization solutions work well for inference but not training
* FlexN+M uses M-bit exponent shared across all elements of a tensor, and has a N-bit precision storing an integer value for each element
* Reduce memory, bandwidth, and computation (turns computation of DNN into fixed point operations)
* Autoflex algorithm stores a histgram of value range to predict future range
  
## Generative Adversarial Nets
23/11
* Co-train a generative model and discriminative model
* The idea isn't something never exists before. But this doesn't prevent it becoming a great work. Figuring out the right solution to solve right problem is remarkable
* The concept is intuitive to understand but still need see concrete examples to understand details
  
## On the Opportunities and Risks of Foundation Models
23/10

* https://arxiv.org/abs/2108.07258
* Research insights of foundation model, and landscape summary
* The significance of foundation model can be summarized by two words: emergence and homogenization
* The emergence is the both the source of scientific excitement and anxiety about unanticipated consequence
* Stages in foundation model ecosystem: data creation, data curation, training, adaption, deployment
   * Training data determines the theoretical information available for foundation model
   * Model architectures determine how much of this information can be extracted
   * Computer systems determin what is practically achievable
* In major ML research areas, e.g. NLP, vision, etc, the transition marked a new era, where the classic approaches were task-specific feature engineering, towards self-supervised learning that could make use of large quantities of raw data with great generality
* Paradox in AI: hard problems are easy and easy problems are hard
* In transformer, attention layers provide a new ability, in contrast to feed-forward MLP, is to adapt its computation to the input, e.g. She ate the icecream with the X [spoon|strawberries].
* TODO: retrieval-based models such as REALM, RAG, ColBERT-QA, and RETRO

  
## DALL.E 3 Improving Image Generation with Better Captions
23/10

* https://cdn.openai.com/papers/dall-e-3.pdf
* Challenge of image generation: controllability of image generation systems, which often overlook the words, word ordering, or meaning in a given caption
* Hypothesize: a fundamental issue is the poor quality of text and image pairing datset
* Solution: generate improved captions for images by caption model in the dataset, and train text-to-image models with this synthetic data
* Result: comparing text-to-image models trained with ground truth captions, short synthetic captions and descriptive synthetic captions, observed significant performance improvements 

## Scaling Law for Neural Network Models
23/10

* [https://arxiv.org/abs/2001.08361](https://arxiv.org/abs/2001.08361)
* The loss scales as a power-law with model size, dataset size, and the amount of compute used for training
* Constant values in power law equations derived from empirical data
* Equations of num parameters and flops per token in a Transformer model
* Insightful analysis on equations related to overfitting and efficient batch size

# MoE 
23/10

OpenAI techniques for training large neural networks

* [https://openai.com/research/techniques-for-training-large-neural-networks](https://openai.com/research/techniques-for-training-large-neural-networks) 


## Sparsely-Gated MoE 
23/10


* [https://arxiv.org/abs/1701.06538](https://arxiv.org/abs/1701.06538) 
* MoE has been created 20 years ago: y = sum(G(x) * E(x))
* Gating network: 
    * 1) sparsity: only top-k experts will be computed, 
    * 2) a noise on the input x for Gating computation
* Addressing Issues
    * Shrinking batch problem: data parallel standard layers but model parallel experts (only one replica)
    * Network bandwidth: increase n_layer and hidden dimension in experts
    * Balancing Expert utilization: introduce additional loss function to encourage equal importance
* Q: How is routing logic actually implemented? 


## Switch Transformers
23/10


* [https://arxiv.org/abs/2101.03961](https://arxiv.org/abs/2101.03961) 
* Limitation of Sparsely-Gated MoE: computation efficiency, complexity, instability
* Switch Transformer
    * Parameter count, independent of total computation performed, is a separately important axis on which to scale
    * Simplify expert routing: instead of top-k, make k=1 in routing
    * Efficient sparse routing: 
        * Set expert capacity, and skip computation for tokens that exceeds expert capacity
        * Single auxiliary loss function to balance expert load and importance
* Solve instability: 1) selective precision, 2) parameter initialization, 3) regularization
* Advanced results: 
    * 1) advanced performance on time and step basis analysis, 
    * 2) advanced performance on fine tune, distillation, etc
* Perf analysis (compute & comm) over data, model, and expert parallel 


## GShard 
23/09


* [https://arxiv.org/abs/2006.16668](https://arxiv.org/abs/2006.16668) 
* A easy to use library that partitions gated MoE model with high efficiency
* Design principles: 
    * 1) use gated MoE to achieve sub-linear scaling between computation/communication requirements and model capacity
    * 2) separate model arch code with infra (partitioning and perf optimization) code
    * 3) still use SPMD to retain compiler scalability
* Model: 1) top-k is independent of num of experts, 2) shard MoE across devices but replicate other layers
* Parallelization: 
    * partition input tokens into groups, each of which runs gating and routing in parallel
    * einsum to denote the algorithm
    * use annotation to denote sharding (dimension, shard) and replication info
* XLA compiler partitioner implementation details: communication primitives, operators
* Experiment on Gated MoE, with perf (memory, compute, communication) analysis and best practice

# LoRA 

23/09

## LORA: LOW-RANK ADAPTATION OF LARGE LANGUAGE MODELS
Terms
- The rank of a matrix
- autoregressive model

Existing transfer learning strategies
- Section 6, a survey of well known transfer learning strategies
- Naive fine tune
  - Substantial computation requirements
  - Overfitting due to small size of domain specific data
- Adding adapter layer
  - Extra latency
  - Optimizing input layer transformation (TODO what does this do)]

Strategy
- Over-parameterized matrix delta W has a lower intrinsic dimension
- Use B[d*r] @ A[r*k] to simulate delta W[d*k]


## Pathway: Asynchronous distributed dataflow for ML

Context
- Unlike SPMD in pytorch (all GPUs execute the same program in a synchronous manner), TF uses MPMD in an asynchronous manner with a controller to decide which TPU runs what on which TPUs

Problem statement 
- In TF one training program can only run within a Pod (usually with a few thousand TPU cores). No existing way to run across Pods, which are connected through Datacenter network
Native dataflow in asynchronous execution is inefficient

Solution
- Parallel dispatch: controller has the global information thus can precompute / compile tensor shape ahead of execution, which is used for tensor allocation and RDMA across TPUs
- Some scheduling/data management implementations to avoid deadlock, optimize cross Pod execution

## GPipe: Pipeline Parallelism

Problem statement
- Single GPU has memory limitation, cannot fit larger models
- Basic pipeline parallelism faces memory inefficiencies

Solution
- To solve memory limitation of a single GPU, shard model on layer dimension and allocate to different GPUs
- To reduce inefficiency caused by pipeline bubble, introduce microbatch
- To further reduce memory consumption, implement re-materialization (i.e. activation checkpointing)

Perf Analysis
- Bubble time:O(K-1M+K-1), K is num GPUs, M is num of microbatch, K is num of GPUs 
  - In our setting, it’s O(pipe_depth-1max_n_microbatches_per_group-pipe_depth-1)
  - If we ensure max_n_microbatches_per_group=1, M becomes to  global_batch_size/microbatch_size
  - Observation from paper: bubble time is negligible when M>=4K
- Peak memory requirements per GPU
  - Vanilla version: O(d_model *global_batch_size *n_layer )
  - Pipeline: O( d_model *global_batch_size *n_layerpipe_depth )
  - Pipeline + checkpointing:  O(d_model*global_batch_size*(1+n_layerpipe_depth * n_microbatch) )
- Effective computation after activation checkpointing
  - 65% is effective computation
  - 22% is re-compute forward pass for activation checkpointing
  - 10% roughly evenly distributed across 1) load balance, 2) bubble overhead, 3) others
- Overall, GPipe is able to 
  - partition the model memory footprint with the cost of computation inefficiency
  - reduce peak memory requirement with the cost of extra forward pass computation

Further reading
- A more sophisticated pipeline implementation, PipeDream by MSFT
- https://arxiv.org/abs/1806.03377 

## Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism 

Problem Statement
- 1) large model size cannot fit into one gpu memory, 2) efficiently scale computation to multiple GPUs
- Data parallel: by increasing batch size proportionally to worker size, one observed that nearly linear scaling in throughput. However, larger global batch size resulted into ML issue (e.g. accuracy, longer time convergence)
- Model parallel: 
  - parameter server style solutions face consistency issue
  - Gpipe related solutions requires dedicated framework/compiler/etc

Solution
- Tensor parallel, dedicated for transformer, easy to implement with a few changes on pytorch
- Partition 2 layered MLP by column wise followed by row wise -> 2 synchronization needed 
- Partition self-attention by splitting Wq,Wk,Wv by attention head
- Partition embedding table similarly

Perf Analysis: Megatron vs Data Parallel
- Data parallel faces problem due to increasing number of global batch size, which resulting into ML issues
(TODO) Different requirements on communication
- Megatron: O(batch_size * n_ctx * 
- Data parallel: O(
- In Megatron, communication and computation have to happen sequentially, whereas in data parallel they can overlap with each other
- Megatron also requires input X to be copies across GPUs

Perf Analysis: Metatron vs GPipe
- Metatron is specialized for transformer architecture
- [TODO] GPipe has to use activation checkpointing thus causing redundant computation. This is because to maintain reasonable GPU efficiency, it needs to have a big enough number for microbatch size. Meanwhile, in order to reduce bubbles, it needs to have a big enough number of microbatch. So it has at least a big number for memory.
- [TODO] GPipe has less network communication requirements as the comm is proportional to n_gpu rather than n_layers

Metatron-LM
- Due to Megatron’s strict requirements on network (both throughput and latency), it doesn’t scale well beyond 8 GPUs, otherwise inter-node network is required: i.e. n_op_shard <=8
- n_op_shard can divide n_heads and emb etc


## ZeRO: Memory Optimizations Toward Training Trillion Parameter Models

Problem statement
- DP requires the model to fit into a single gpu. Also larger batch size cause slower convergence
- MP (Megatron) doesn’t scale beyond a node
- All other solutions introduce a significant amount of redundant memory

  
Context: model memory footprint analysis 
- Model state: optimizer state, parameter, gradient = (2+2+12)*X
- Residual state: activation (proportional to batch_size, n_ctx, and d_model), buffer, fragmentation

Solution
- ZeRO-DP: partition param, grad, and opt. Copy param before forward. Each shard only need to store and update corresponding grad and opt
- ZeRO-R: some engineering things

Perf Analysis
- Unlike Megatron, which split the operation, where each shard computes the part of computation, thus requires to all_reduce the results of each step (e.g. self attention, mlp). Since it’s intermediate computation data, the size is proportional to batch_size, n_ctx, etc. Thus very large. 
- Besides, since all the shards compute on the same input X, it need to copy X to all shards
- ZeRO copies the params to all shards. And only synchronize on gradient and weights, which is only proportional to model param size, which is fixed and relatively small
- Also it doesn’t need to copy input X, since each shard works on its own input, and its own forward and backward passes. It’s more of a upgraded version of DP

## Efficient Large-Scale Language Model Training on GPU Clusters Using Megatron-LM

## Scaling Distributed Machine Learning with the Parameter Server
- A general purpose ML system capable of scaling to industry scale
- Architecture: parameter server, worker, resource manager
- Consistency: sequential, eventual, bounded delay
- Fault tolerance: consistent hashing with replication
- Network optimization: compression, caching, filter
- Versioning: vector clock to support versioning with range query



# Distributed System

- 04/23

1. [Kafka: a Distributed Messaging System for Log Processing](https://notes.stephenholiday.com/Kafka.pdf)
2. [ZooKeeper: Wait-free coordination for Internet-scale systems](https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf)
