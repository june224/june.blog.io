---
title: tensorflow serving一些优化经验
author: june
date: 2023-12-17
category: recommend
layout: post
---

### 内存泄漏

tensorflow serving（下文称tf）上线之后一段时间之后，发现pod内存一直在平稳地增长。观察服务监控发现，模型版本迭代热更新时，旧版本模型虽然tf已经卸载，但内存没有释放回操作系统。

解决办法：把 malloc 换成 jemalloc 可以解决。

### 开启预热

tf加载模型是以下步骤：
1. 创建一个DirectSession；
2. 将模型的Graph加载到 Session 中；
3. 执行Graph中的Restore Op来将变量从模型中读取到内存；
4. 执行Graph中的Init Op做相关的模型初始化；

如果配置了Warmup，执行Warmup操作，通过定义好的样本来预热模型。
tf的模型执行有个非常显著的特点是lazy initialization，也就是如果没有Warmup，当TF加载完模型，其实只是加载了Graph 和变量，Graph中的OP其实并没有做初始化，只有当客户端第一次发请求过来时，才会开始初始化OP。因此需要开启预热，否则加载完模型，首次推理请求tf延迟会特别高。

### 设置加载和卸载模型线程数

tf有两个参数num_load_threads和num_unload_threads，分别用来设置tf加载模型和卸载模型的线程数。从下图官方文档介绍，这两个参数默认为0，不额外设置线程数，加载和卸载模型会在manager的主线程操作，这会导致tf推理请求延迟。

```cpp{.line-numbers}
tensorflow::Flag("num_load_threads", &options.num_load_threads,
                       "The number of threads in the thread-pool used to load "
                       "servables. If set as 0, we don't use a thread-pool, "
                       "and servable loads are performed serially in the "
                       "manager's main work loop, may casue the Serving "
                       "request to be delayed. Default: 0"),
      tensorflow::Flag("num_unload_threads", &options.num_unload_threads,
                       "The number of threads in the thread-pool used to "
                       "unload servables. If set as 0, we don't use a "
                       "thread-pool, and servable loads are performed serially "
                       "in the manager's main work loop, may casue the Serving "
                       "request to be delayed. Default: 0"),
```

### tensorflow serving服务绑核

模型的推理预测，是要进行大量计算的，所以tf是cpu密集型服务。当tf部署在k8s上，tf的pod和其它服务的pod部署在同一个节点，就容易出现cpu资源竞争问题。特别当业务高峰期，因cpu资源竞争而导致节流的问题会比较突出，因此，可针对tf这种cpu密集型服务进行绑核处理。

绑核之后效果如下图：绑核之后，tf的毛刺得到很大减缓，非业务高峰期，模型请求毛刺从300ms下降到80ms左右。

![tfoptimization_f1](/assets/post/recommend/tfoptimization_f1.png "tfoptimization_f1")

### 调整会话线程参数

tf有两个会话线程参数`tensorflow_inter_op_parallelism`和`tensorflow_intra_op_parallelism`。  
* `tensorflow_inter_op_parallelism`：表示算子间并行操作线程池。tensorflow图中有许多独立的操作，因为在数据流图中它们之间没有定向路径，tensorflow将尝试并发运行它们，使用具有`inter_op_parallelism_threads`线程的线程池。  
* `tensorflow_intra_op_parallelism`：表示算子内并行操作的线程池。例如矩阵乘法（`tf.matmul()`）或减少（例如`tf.reduce_sum()`），TensorFlow将通过在线程池中调度任务来执行它`intra_op_parallelism_threads`线程。  
* 两个会话参数值相加起来等于pod的cpu核数为佳。绑核之后，调整了pod资源，cpu个数从16核调整10核，pod hpa min从10调整为12，inter_op和infra_op线程数分别设置为6和4。因在推荐模型inter_op计算量更大，所以inter_op分配更多的线程数。模型推理预测P99耗时在30~40ms左右。

![tfoptimization_f2](/assets/post/recommend/tfoptimization_f2.png "tfoptimization_f2")

### tf sdk侧优化

#### 请求分片

当前推荐业务进入到精排进行模型推理预测的item数从几百到上千不等，模型单次处理这么大的请求，很容易因为某个任务计算量大，占用计算资源，导致其它小任务阻塞等待，从而导致长尾效应，P99波动较大。因此可将单个大的推理请求拆分成多个小的推理请求。比如，一个推理请求需要模型给700个item打分，可分成单次200个item的4并行请求。

下图即是请求分片后的效果：模型请求分别拆成200item，100item的小请求，模型耗时从30+ms分别降到18ms和12ms。

![tfoptimization_f3](/assets/post/recommend/tfoptimization_f3.png "tfoptimization_f3")

#### 多协程特征处理以及使用对象池

在tf模型模型推理请求之前，我们需要将特征处理成tf sdk的请求报文协议格式，因一个item可能会有上百个特征，单个特征还可能是向量，所以这必然导致在特征处理这块会有频繁的对象创建和销毁问题。以及上文提到的请求分配，需要将请求拆分成多个请求分片，必然涉及到一定量的计算，所以这块需要开多协程处理。

#### 其它

除此之外，我们还尝试了开启batch，重新编译优化tensorflow serving，以及压缩tf请求报文，但这些优化手段并不符合我们当前tf的使用场景，优化效果不明显，甚至效果还是负向的。

