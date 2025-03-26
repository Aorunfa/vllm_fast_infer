# About -- ongoing
使用vllm快速进行transformer模型的推理，快速理解vllm关键技术点，上手模型部署

为什么vllm成为受欢迎推理框架
- pg attention对内存的有效控制
- batch推理异步输出，先到先服务，先完成先发送

- transform无缝集成
- 支持各类量化推理
- 抢占机制，当gpu资源不够对running的后来任务进行资源抢占（cpu offload）

高吞吐量，内存控制，流式响应


应用场景
- 本地大批量推理任务，内存控制与分布式多卡推理
- 线上模型部署api访问，mini-batch的异步推理

# 关键技术
[learning link](https://qiankunli.github.io/2024/07/07/vllm.html)
本节只介绍vlm内关键的技术实现，主要包括内存控制和一些调度设计

vllm的一大核心是实现内存有效控制，主要得益于使用了分页注意力机制，可以避免提前开辟连续整块的内存空间，使用非连续的物理内存构建连续的虚拟内存，进行kv缓存，提高推理的有效吞吐量

此外，传统batch推理时间取决最长的输出，造成batch内其他任务的无效等待，vllm使用iteration层级的调度，batch内先完成的任务先离开

进一步地，同一个prompt，不同的采样方式得到的是可能不同的，例如使用并行采样(num_return_sequences>1)、beam search等采样方式可以得到多个答案。vllmz在prefill阶段为每一个prompt设置一个sequence group，sequence group内也进行iteration层级的调度。因此，在batch内的iteration层级调度数量为batchsize * len(sequence group)


## 分页注意力 + iteration层级调度
背景: 传统推理模式下，预先为kv cache矩阵开辟连续的显存空间，一般形状为(最大回答长度, 嵌入维度)，但llm输出长度是无法预测的，前分配会造成很多占用浪费，导致推理吞吐量下降

分页注意力机制page attention优化的是这

传统推理kv cache导致的显存利用率低的问题


## prefill预处理 + sequence group 
对waiting队列的任务进行prefill，先生成一个token，将已有的prompt提前进行kv-cache，初始化相应的attention page索引页，同时初始化对应的sequence group，将推理任务加入scheduler -- todo confirm

todo: prefill应该需要去计算和设置相关的东西，而不仅仅是将已有的token进行kv cache

## swapped + Preempt
swapped操作用于在gpu资源不足将running的任务暂时从gpu转移到cpu，剩余running任务抢占腾出来的资源（Preempt）。对于移出的任务，会被加入swapped队列，等待gpu的资源充足后，重新加入running队列。

swapped队列处理优先级高于waiting队列

swapped对于sequence group长度为1的，直接销毁kv cache，后续进行recomputaion。而长度超过1时，则将每个sequence对应的分页存储的kv cache一起卸载到cpu上，避免后续进行重复计算。这样做用于权衡卸载加载的时间与重计算时间。



# 本地跑批量任务

## 单卡部署

## 单机多卡部署

## 多机多卡部署


# 线上部署推理服务

## 单卡部署

## 单机多卡部署

## 多机多卡部署
