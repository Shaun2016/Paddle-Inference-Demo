# GPU TensorRT 加速推理(NV-GPU/Jetson)

- [GPU TensorRT 加速推理(NV-GPU/Jetson)](#gpu-tensorrt-加速推理nv-gpujetson)
  - [1. 概要](#1-概要)
  - [2. 环境准备](#2-环境准备)
  - [3, API 使用介绍](#3-api-使用介绍)
  - [4. 运行 Dynamic shape](#4-运行-dynamic-shape)
  - [5. Paddle Inference 适配 TensorRT 原理介绍](#5-paddle-inference-适配-tensorrt-原理介绍)

<a name="1"></a>

## 1. 概要

TensorRT 是一个针对 NVIDIA GPU 及 Jetson 系列硬件的高性能机器学习推理 SDK，可以使得深度学习模型在这些硬件上的部署获得更好的性能。Paddle Inference 以子图方式集成了 TensorRT，将可用 TensorRT 加速的算子组成子图供给 TensorRT，以获取 TensorRT 加速的同时，保留 PaddlePaddle 即训即推的能力。在这篇文章中，我们会介绍如何使用 TensorRT 加速推理。


当模型被 Paddle Inference 加载后，神经网络被表示为由变量和运算节点组成的计算图。在图分析阶段，Paddle Inference 会对模型进行分析同时发现图中可以使用 TensorRT 优化的子图，并使用 TensorRT 节点替换它们。在模型的推理期间，如果遇到 TensorRT 节点，Paddle Infenrence 会调用 TensorRT 对该节点进行执行，其它节点调用 GPU 原生推理。TensorRT 除了有常见的 OP 融合以及显存/内存优化外，还针对性地对 OP 进行了优化加速实现，降低推理延迟，提升推理吞吐。


目前 Paddle Inference 支持 TensorRT 的静态 shape、动态 shape 两种运行方式。静态 shape 用于模型输入 shape 除 batch 维外，其他维度大小不变的情况，静态 shape 模式下支持图像分类，分割，检测模型；动态 shape 可用于输入 size 任意变化的模型， 如动态 shape 的图像模型（FCN， Faster rcnn）、 NLP 的 Bert/Ernie 等模型，当然也包括静态 shape 支持的模型。 静态 shape 和动态 shape 都支持fp32、fp16、int8 等多种计算精度。TensorRT 支持服务器端GPU，如T4、A10， 也支持边缘端硬件，如 Jetson NX、 Jetson Nano、 Jetson TX2 等。 在边缘硬件上，除支持常规的 GPU 外，还可用 DLA 进行推理，也支持 RTX2080，3090 等游戏显卡。

用 TensorRT 首次推理时，TensorRT 需要进行各 Op 融合、显存复用、以及 Op 的 Kernel 选择等，导致首帧耗时过长。Paddle Inference 开放了 TensorRT 序列化接口，用于将 TensorRT 分析的信息进行存储，在后续推理直接载入相关序列化信息，从而减少启动耗时。

<a name="2"></a>

## 2. 环境准备

在 GPU 下使用 TensorRT 加速推理，需要安装 CUDA、cuDNN、TensorRT 和对应版本的 Paddle Inference 预编译包。
关于这几个软件的安装版本，请参考如下建议（原因：CUDA、cuDNN、TensorRT 版本众多，且有严格的版本对应关系）：

- 电脑上 CUDA、cuDNN、TensorRT 都还没安装的开发者，建议参考 Paddle Inference 提供的预编译包信息，去安装对应版本的CUDA、cuDNN、TensorRT。
- 电脑上已安装 CUDA、cuDNN，但没有安装TensorRT，建议参考Paddle Inference提供的cuda、cudnn的对应版本的TensorRT版本去安装TensorRT。
- 电脑上已安装 CUDA、cuDNN、TensorRT的开发者，去下载对应版本的 Paddle Inference 预编译包。
  - 如果 Paddle Inference 预编译包没有对应版本的，一种方式是按照 Paddle Inference 提供的预编译包信息重新安装CUDA、cuDNN、TensorRT，一种是自己源码编译对对应电脑上 CUDA、cuDNN、TensorRT 版本的 Paddle Inference 预编译包。从工程难易程度，建议选择第一种方案。

如果您需要安装 [TensorRT](https://developer.nvidia.com/nvidia-tensorrt-8x-download)，请参考 [TensorRT 文档](https://docs.nvidia.com/deeplearning/tensorrt/archives/tensorrt-801/install-guide/index.html)。

Paddle Inference 提供的 Ubuntu/Windows 平台的官方 Release 推理库均支持 TensorRT 加速推理，如果您使用的是以上平台，我们优先推荐您通过以下链接直接下载，或者您也可以参照文档进行[源码编译](./compile/index_compile.md)。

- [下载安装 Ubuntu 推理库](https://paddleinference.paddlepaddle.org.cn/install/download_lib.html#linux)
  - 此链接中名称前缀包含 `nv_jetson` 的为用于NV Jetson平台的推理库。
- [下载安装 Windows 推理库](https://paddleinference.paddlepaddle.org.cn/install/download_lib.html#windows)

**Note:**

1. 从源码编译支持 TensorRT 加速的 Paddle Infenrence 推理库时，你需要设置编译选项 TENSORRT_ROOT 为 TensorRT SDK 的根目录。
2. Windows 支持需要 TensorRT 版本 5.0 以上。
3. 使用 TensorRT 的动态 shape 输入功能要求 TensorRT 的版本在 6.0 以上。


<a name="3"></a>

## 3, API 使用介绍

在[上一节](./gpu_native_infer.md)中，我们了解到 Paddle Inference 推理流程包含了以下六步：

- 导入包
- 设置 Config
- 创建 Predictor
- 准备输入
- 执行 Predictor
- 获取输出

Paddle Inference 中启用 TensorRT 也是遵照这样的流程。我们先用一个简单的例子来介绍这一流程（我们假设您已经对Paddle Inference有一定的了解，如果您刚接触Paddle Inference，请访问[这里](https://paddleinference.paddlepaddle.org.cn/quick_start/workflow.html)， 对Paddle Inference有个初步认识）。


```python
    import numpy as np
    import paddle.inference as paddle_infer
    
    def create_predictor():
        config = paddle_infer.Config("./resnet50/model", "./resnet50/params")
        config.enable_memory_optim()
        config.enable_use_gpu(1000, 0)
        
        # 打开TensorRT。此接口的详细介绍请见下文
        config.enable_tensorrt_engine(workspace_size = 1 << 30, 
                                      max_batch_size = 1, 
                                      min_subgraph_size = 3, 
                                      precision_mode=paddle_infer.PrecisionType.Float32, 
                                      use_static = False, use_calib_mode = False)

        predictor = paddle_infer.create_predictor(config)
        return predictor

    def run(predictor, img):
        # 准备输入
        input_names = predictor.get_input_names()
        for i,  name in enumerate(input_names):
            input_tensor = predictor.get_input_handle(name)
            input_tensor.reshape(img[i].shape)   
            input_tensor.copy_from_cpu(img[i].copy())
        # 推理
        predictor.run()
        results = []
        # 获取输出
        output_names = predictor.get_output_names()
        for i, name in enumerate(output_names):
            output_tensor = predictor.get_output_handle(name)
            output_data = output_tensor.copy_to_cpu()
            results.append(output_data)
        return results

    if __name__ == '__main__':
        pred = create_predictor()
        img = np.ones((1, 3, 224, 224)).astype(np.float32)
        result = run(pred, [img])
        print ("class index: ", np.argmax(result[0][0]))
```

通过例子可以看出，我们通过 `enable_tensorrt_engine` 接口来打开 TensorRT 选项。

```python
    config.enable_tensorrt_engine(workspace_size = 1 << 30, 
                                  max_batch_size = 1, 
                                  min_subgraph_size = 3, 
                                  precision_mode=paddle_infer.PrecisionType.Float32, 
                                  use_static = False, use_calib_mode = False)
```
接下来让我们看下该接口中各个参数的作用:  

- **workspace_size**，类型：int，默认值为1 << 30 （1G）。指定 TensorRT 使用的工作空间大小，TensorRT 会在该大小限制下筛选最优的 kernel 进行推理。
- **max_batch_size**，类型：int，默认值为1。需要提前设置最大的batch大小，运行时batch大小不得超过此限定值。
- **min_subgraph_size**，类型：int，默认值为3。Paddle Inference 是以以子图的形式接入 TensorRT 的，为了避免性能损失，当子图内部节点个数大于 min_subgraph_size 的时候，才会将此子图接入 TensorRT 运行。
- **precision_mode**，类型：**paddle_infer.PrecisionType**， 默认值为 **paddle_infer.PrecisionType.Float32**。指定使用 TensorRT 的精度，支持FP32（Float32），FP16（Half），Int8（Int8）。若需要使用 TensorRT int8 离线量化校准，需设定 precision 为 **paddle_infer.PrecisionType.Int8** , 且设置 **use_calib_mode** 为 True。
- **use_static**，类型：bool， 默认值为 False。如果指定为 True，在初次运行程序的时候会将 TensorRT 的优化信息进行序列化到磁盘上，下次运行时直接加载优化的序列化信息而不需要重新生成。
- **use_calib_mode**，类型：bool， 默认值为 False。若要运行 int8 离线量化校准，需要将此选项设置为 True。


对于 Jetson 系列硬件上，除了可将模型运行在 GPU 上，还可将模型运行在 DLA 上，DLA 是一款针对深度学习操作的固定功能加速器引擎，旨在对卷积神经网络进行全硬件加速。Paddle Inference 开放了启用指定的 DLA 进行模型推理的接口，默认启动第 0 个 DLA：

```shell
// python API
config.enable_tensorrt_dla(0)
// C++ API
config.EnableTensorRtDLA(0);
```

DLA 上对运行的模型有一定要求，详情请可参考[链接](https://docs.nvidia.com/deeplearning/tensorrt/developer-guide/index.html#dla_topic)。当 DLA 遇到模型中的某些不支持的层时，会回退到 GPU 进行推理，目前 DLA 仅支持fp16 和 int8 精度。

<a name="4"></a>

## 4. 运行 Dynamic shape

当模型的输入 shape 不固定的话（如 OCR，NLP 的相关模型），需要推理框架提供动态 shape 的支持。从1.8 版本开始， Paddle Inference 对 TensorRT 子图进行了 Dynamic shape 的支持。
使用接口如下：

```python
	config.enable_tensorrt_engine(
		workspace_size = 1<<30,
		max_batch_size=1, min_subgraph_size=5,
		precision_mode=paddle_infer.PrecisionType.Float32,
		use_static=False, use_calib_mode=False)
		  
	min_input_shape = {"image":[1,3, 10, 10]}
	max_input_shape = {"image":[1,3, 224, 224]}
	opt_input_shape = {"image":[1,3, 100, 100]}

	config.set_trt_dynamic_shape_info(min_input_shape, max_input_shape, opt_input_shape)
```

从上述使用方式来看，在 `config.enable_tensorrt_engine` 接口的基础上，新加了一个 `config.set_trt_dynamic_shape_info` 的接口。“image” 对应模型文件中输入的名称。

该接口用来设置模型输入的最小、最大、以及最优的输入 shape。 其中，最优的 shape 处于最小最大 shape 之间，在推理初始化期间，会根据opt shape对 Op 选择最优的 Kernel 。   

调用了 **config.set_trt_dynamic_shape_info** 接口，推理器会运行 TensorRT 子图的动态输入模式，运行期间可以接受最小、最大 shape 间的任意 shape 的输入数据。

此接口的相关示例请参考下面链接。
- Python 样例请访问此处[链接 1](https://github.com/PaddlePaddle/Paddle-Inference-Demo/tree/master/python/gpu/resnet50) ，[链接 2](https://github.com/PaddlePaddle/Paddle-Inference-Demo/tree/master/python/gpu/yolov3)。
- C++ 样例地址请访问此处[链接 1](https://github.com/PaddlePaddle/Paddle-Inference-Demo/tree/master/c%2B%2B/gpu/resnet50) ，[链接 2](https://github.com/PaddlePaddle/Paddle-Inference-Demo/tree/master/c%2B%2B/gpu/yolov3)。

&emsp;

Paddle Inference 还提供了另外一份使用动态 shape 方法，此接口不用明确指出输入 shape 范围，但需要准备一些数据来运行模型，以便于收集模型中 Tensor 的大小，使用接口如下：

```python
    if args.tune:
        config.collect_shape_range_info(shape_file)
    if args.use_gpu:
        config.enable_use_gpu(1000, 0)
        if args.use_trt:
            # using dynamic shpae mode, the max_batch_size will be ignored.
            config.enable_tensorrt_engine(
                workspace_size=1 << 30,
                max_batch_size=1,
                min_subgraph_size=5,
                precision_mode=PrecisionType.Float32,
                use_static=False,
                use_calib_mode=False)
            if args.tuned_dynamic_shape:
                config.enable_tuned_tensorrt_dynamic_shape(shape_file, True)
```

首次运行时，需设置 `args.tune` 为 `True`，此时运行模型将会不做任何优化，`collect_shape_range_info` 将模型中间 Tensor 的所有 shape 保存到文件 `shape_file` 中。

第二次运行时，需设置 `args.tune` 为 `False`，同时设置`args.tuned_dynamic_shape` 为 True，此时运行模型将会根据生成的 `shape_file` 进行模型优化。

此接口的相关示例请参考下面链接。
- Python 样例请访问此处[链接](https://github.com/PaddlePaddle/Paddle-Inference-Demo/tree/master/python/gpu/tuned_dynamic_shape)
- C++ 样例地址请访问此处[链接](https://github.com/PaddlePaddle/Paddle-Inference-Demo/tree/master/c%2B%2B/gpu/tuned_dynamic_shape) 。


<a name="5"></a>

## 5. Paddle Inference 适配 TensorRT 原理介绍

Paddle Inference 采用子图的形式对 TensorRT 进行集成，当模型加载后，神经网络可以表示为由变量和运算节点组成的计算图。Paddle Inference 对整个图进行扫描，发现图中可以使用 TensorRT 优化的子图，并使用 TensorRT 节点替换它们。在模型的推断期间，如果遇到 TensorRT 节点，Paddle Inference 会调用 TensorRT 库对该节点进行优化，其他的节点调用 Paddle Infenrence 的 GPU 原生实现。TensorRT 在推断期间能够进行 Op 的横向和纵向融合，过滤掉冗余的 Op，并对特定平台下的特定的 Op 选择合适的 Kernel等进行优化，能够加快模型的推理速度。  

下图使用一个简单的模型展示了这个过程：  

**原始网络**

<img src=https://raw.githubusercontent.com/NHZlX/FluidDoc/add_trt_doc/doc/fluid/install/howto/inference/image/model_graph_original.png >

**转换的网络**

<img src=https://raw.githubusercontent.com/NHZlX/FluidDoc/add_trt_doc/doc/fluid/user_guides/howto/inference/image/model_graph_trt.png> 

我们可以在原始模型网络中看到，绿色节点表示可以被 TensorRT 支持的节点，红色节点表示网络中的变量，黄色表示 只能被 GPU 原生推理执行的节点。那些在原始网络中的绿色节点被提取出来汇集成子图，并由一个 TensorRT 节点代替，成为转换后网络中的 **block-25** 节点。在网络运行过程中，如果遇到该节点，Paddle Inference 将调用TensorRT 来对其执行。
