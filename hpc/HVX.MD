# 目录
- [目录](#目录)
- [移动端算法优化背景](#移动端算法优化背景)
- [高通移动端系统架构](#高通移动端系统架构)
- [Hexagon DSP介绍](#hexagon-dsp介绍)
  - [高通SOC架构图](#高通soc架构图)
  - [HVX的优势](#hvx的优势)
  - [DSP硬件架构](#dsp硬件架构)
  - [HVX架构-线程模型](#hvx架构-线程模型)
  - [HVX架构-内存](#hvx架构-内存)
  - [HVX架构-SIMD扩展](#hvx架构-simd扩展)
- [HVX软件架构](#hvx软件架构)
  - [远程过程调用（RPC）](#远程过程调用rpc)
- [Hexagon SDK 下载和安装](#hexagon-sdk-下载和安装)
- [Hexagon SDK 手机端运行](#hexagon-sdk-手机端运行)
  - [算法实例分析](#算法实例分析)
-------------------------------------------------------------
# 移动端算法优化背景
- 落地难点
  1. 受限于芯片的封装面积、功耗、散热等因素，移动端处理器的性能远低于服务器端处理器的性能，除此之外，在移动端上还有内存、存储空间、网络带宽等居多限制。
  2. 越来越多的移动端影像算法应用的需求，同时对于影像算法的效果的要求也是越来越高，这又对移动端平台的计算能力提出了更高的要求。
  3. 移动端很多应用场景对于影像算法的实时性要求很高，以手机相机为例，拍照时间长、拍照卡顿、手机发烫这些问题会直接减低用户的使用体验。


- 优化方向
  1. 在平台方面：移动平台的算力要持续提升，以高通芯片为例，从 855 到 865 再到 888，可以看到每代芯片在算力上都有很大的提升。
  2. 在算法方面：在确保效果的前提下尽量的简化算法的复杂度，这也是体现算法研发功底的地方。
  3. 在算法优化方面：要结合算法的特性和平台的计算资源对算法做代码级的优化以提升算法的性能，要做到选择最优的实现方法以及充分发挥硬件的计算能力。
  4. 在系统架构方面：要做更高维度的优化，合理的进行资源的调度和分配，优化算法处理流程，提升整体的性能。
   
# 高通移动端系统架构
![image](../image/soc.jpg)
- CPU：对应上图中的① (Kryo subsystem)，基于 ARM Cortex 架构的 64 位处理器，目前移动端的 CPU 处理器基本上都是基于 ARM 架构的，为应对多媒体密集的计算需求引入 Neon 指令集扩展，并且支持浮点运算，优点是开发简单、使用方便，可移植性好，缺点是计算能力偏弱，而且由于 CPU 资源竞争导致性能上存在很大的波动，功耗偏高，适用于逻辑控制模块，计算量不是很大或者不便于做矢量处理的模块，实现的时候一般采用 Neon (intrinsic 或者汇编) + 多线程的方式。
- GPU：对应上图中的③ (Multimedia subsystem) 中的 Qualcomm Adreno 部分，在 Android 手机的 GPU 领域主要是高通 Adreno 系列和 ARM Mali 系列两大类，GPU 的优点是并行线程多，支持向量化的处理，算力强，存在的缺点是功耗高，存在 UI 渲染资源竞争导致性能存在抖动，适用于适合计算密度高，逻辑简单的算法处理，实现的时候一般采用 OpenCL，OpenGL 或者 Vulkan 等方式。
- DSP：对应上图中的⑨ (Compute subsystem) ，DSP 是专有处理器，优点是计算能力很强，具有更大的 SIMD 指令位宽，更灵活的超长指令字设计，直接高效的内存访问模型，运行效率稳定，资源竞争少，功耗低，缺点是开发难度极大，算法中计算量大或者性能瓶颈模块往往需要采用 DSP来处理，一般采用 DSP 提供的专有指令 (intrinsic 或者汇编)来实现，如高通 hexagon DSP 中的 HVX 指令。
- NPU：对应上图中的⑫ (Qualcomm Neural Processing Unit)，专门用于神经网络硬件加速的处理器，针对神经网络的特点专门设计了硬件结构和处理器指令，带来的好处是相比于 CPU 和 GPU，功耗会低很多，处理时间会有数量级上的提升，缺点是神经网络的设计依赖于 NPU 支持的算子操作，灵活性不高，模型推理的时候需要进行量化，在精度上会有一些影响。实现的时候一般采用 NPU 提供的推理框架，比如高通的 SNPE 和 QNN。

# Hexagon DSP介绍
Hexagon 是高通公司数字信号处理器 (DSP) 产品系列的品牌名称，Hexagon 也称为 QDSP6，代表“第六代数字信号处理器”。 第一代 QDSP6 V1 在 2006 年便已经推向市场，距现在已经过去十几年。据高通称，Hexagon 架构旨在为各种应用提供低功耗的性能，用于高通骁龙芯片，面向智能手机、汽车、可穿戴设备和其他移动设备，也用于蜂窝电话网络的组件。

## 高通SOC架构图
![image](../image/dsp.png)
- 模块 DSP 主要包含主处理器（标量处理器）、协处理器 HVX（Hexagon Vector Extensions 矢量处理器）及其他处理核心。

## HVX的优势
- 更强的数据并行处理能力，1024b 指令位宽，且有多个线程可并行，线程里面每个 packet 可同时并行执行四条指令。
- 更低的功耗。
- 更高的运行稳定性。
![image](../image/preformance.webp)

## DSP硬件架构
![image](../image/dsp-arch.png)
如上图所示，我们所开发的主要涉及三个硬件资源：
- 主处理器（标量处理单元）
- HVX 协处理器（矢量处理单元）
- memory （VTCM 等）

## HVX架构-线程模型
![image](../image/dsp-thread.png)
- 主处理器主要负责标量运算，有 4 个（目前新架构已调整至 6 个）硬件线程（每个拥有 4 路 VLIW，共享 L1/L2）。
- 向量化运算由 2 个 HVX context 组成（目前新架构已调整至 4 个），分别会被多个标量线程控制。
- 主处理器和 HVX 都是可以有多个软件线程，由 QURT 实时操作系统进行硬件线程选择及调度，开发者不可控。
  
## HVX架构-内存
![image](../image/dsp-memory.webp)
- Vector 单元可以直接访问 L2 cache 
  * 可以有效降低大块图像内存在 L1 上的平铺开销
  * 提供单指令周期数据加载使用
  * 可以支持全带宽
  * 简化编程难度
- L1/L2 由硬件保持同步
- 提供 DDR 到 L2 cache 的流式预加载
- Vector 单元支持灵活的加载存储操作
  * 非对齐方式读写
  * 基于字节的条件状态判断
- （VTCM）向量化 TCM 支持
  * 支持 vscatter/vgather 指令
  * 相对于 L2 cache，数据可以常驻，不存在数据刷出缓冲的问题
  * 支持 vmem 和 vmemu 指令直接访问 

## HVX架构-SIMD扩展
![image](../image/dsp-simd.png) 
- 强大的 SIMD 扩展支持
  * 支持 1024b SIMD * 4 Vector-slot VLIW
  * 4096 result bits/cycle
- 支持 256 8x8 mpy，64 16x16 mpy
  * ![image](../image/mpy.png)
- 拥有 32 个 1024b 寄存器 R0-R31，4个判断寄存器 P0-P3 
- 支持 8/16/32/64-bits 的定点运算
- 提供特殊的 ISA
  * 滑窗滤波指令
  * LUT 指令
  * 直方图指令等

# HVX软件架构
![image](../image/soft.png)
上图为 HVX 软件架构，主要分以下几个部分：
- User Application模块，host 端算法实现，主要作用是触发 FastRPC 调用，调度device 应用。
- FastRPC 模块，远程调用模块，实现 host 与 device 之间的通信，把 device 端算法 so 库从 host 端动态加载到 device 设备，host 调用 device 时会触发 FastRPC 调用，FastRPC 调用在 device 端会创建一个任务进程，完成 host 与 device 之间的调用过程。host 端会阻塞等待 device 端执行完成，属于阻塞等待。
- User Application (Hexagon SO) 模块，device 端算法实现，通过编译会生成一个 Hexagon so 文件。主要用于 device 端算法运行，同时还包含一系列辅助 API 接口等。
- libdspCV_skel.so，这块主要是 dspcv 模块，在 Hexagon SDK 中的一个库文件，用来控制 device 端部分资源。例如时钟控制、多线程等功能封装。
- QuRT RTOS 模块，QuRT 的作用，主要负责系统资源维护，使用最多的主要是线程调用及 VTCM 等资源，根据线程的优先级将软件线程调度到硬件线程上，由这个 QuRT 操作系统来调度。
- CDSPPM & DCVS V2 模块，功率管理模块，主要提供时钟带宽的投票，用来管理时钟、power。
- Hardware thread，HVX context，最底层的硬件线程和HVX句柄。
  
## 远程过程调用（RPC）
  * 远程调用示意图
    ![image](../image/fast-rpc.png)
  * 在介绍 HVX 动态加载过程之前先简单介绍远程调用的过程。远程调用，简单的理解就是一个节点请求另一个节点提供的服务。一个节点调用另一个节点的函数。
    ![image](../image/rpc.png)
    * 不在同一地址空间中，不能像本地调用一样直接调用函数，本地调用参数传递直接给到寄存器或者是压栈，传给被调用函数。但是远程调用不能这样直接调用。

    * 远程调用时，首先客户端需要告诉服务器，需要调用函数，这里函数和进程 ID 存在一个映射，客户端远程调用的时候，需要查一下函数，找到对应的 ID，然后执行函数的代码。

    * 客户端需要把本地参数传给远程函数，本地调用的过程中，直接压栈即可，但是在远程调用过程中不在同一个内存里，无法直接传递函数的参数，因此需要客户端将要调用的函数名以及参数打包，然后通过网络发送到服务器。

    * 服务器解包数据，得到调用的函数以及参数，运行函数，并将结果打包，通过网络发送到客户端。

    * 客户端解包数据，得到函数结果。
  
- FastRPC
  * 跟 RPC 的调用类似，通常算法会编译成两个库，一个在 host 端运行的 Stub 库，一个在 device 端运行的 Skel 库。两个库都会存放在设备的 host 文件系统中，在每个 host 端的线程中，host端第一次调用 device 端应用时， FastRPC 会将 device 端的 Skel 库动态加载到 device 端（这个操作会有额外耗时，所以初始化操作可异步进行）。然后，device 端的会根据 domain 信息进行应用匹配，并调用相应的算法，host 端会阻塞等待 FastRPC 调用结束。FastRPC 函数及结构体接口由 IDL 文件来映射（关于IDL 文件，将在后续内容进行讨论）。

   * host 端与 device 端数据零拷贝操作由高通基于 ION/DMA_BUF 实现，虚实转换由 SMMU 来进行映射完成，降低了开发者的使用难度。
   * ![image](../image/fast-rpc-2.webp)
   * FastRPC 用来作为两个处理器 host 和 device 的通信，每次触发 FastRPC 需要 0.5~2ms 的额外开销，尽量少触发 FastRPC 调用，函数调用尽量合并在 device 端去调用。
  
# Hexagon SDK 下载和安装
- HVX 开发工具分为 windows 和 Ubuntu环境，本专栏主要以 Ubuntu 环境为主进行介绍。

- HVX 的开发工具是 Hexagon SDK（文章基于版本 Ubuntu 20.04 进行演示）

1. SDK 下载
  * [下载工具](https://link.zhihu.com/?target=https%3A//developer.qualcomm.com/software/hexagon-dsp-sdk/tools) 下载 SDK 安装包（hexagon_sdk_lnx_3_5_installer_00006_1.zip），文章基于 SDK3.5.4 版本进行介绍。
  ![image](../image/sdk.png)

2. SDK 安装
   ``` 
   # 解压安装包 hexagon_sdk_lnx_3_5_installer_00006_1.zip
   # 在解压后的目录下进行如下操作
   sudo chmod a+x ./qualcomm_hexagon_sdk_3_5_4_eval.bin
   ./qualcomm_hexagon_sdk_3_5_4_eval.bin
   ```
3. SDK 目录介绍
- SDK 根目录
  * ![image](../image/sdk-file.png)

    | 文件夹               | 描述                                                                                                 |
    | -------------------- | ---------------------------------------------------------------------------------------------------- |
    | build                | 编译所需的编译脚本文件                                                                               |
    | docs                 | HVX 开发相关说明文档                                                                                 |
    | examples             | SDK 例子，HVX 样例位于 common 文件夹                                                                 |
    | incs                 | SDK 头文件目录，包含 HVX 函数及指令等头文件                                                          |
    | libs                 | SDK 必需库文件目录，包含 HVX 开发运行所涉及的必备库及部分实现代码，例如 dspcv，fastcv，hexagon_nn 等 |
    | scripts              | 常用脚本目录                                                                                         |
    | setup_sdk_env.source | 环境变量设置脚本                                                                                     |
    | tools                | 常用工具集合，包含 ndk，qaic 及签名工具（elfsigner）等                                               |
- tools 目录
  * ![image](../image/tools.webp)
- tools/HEXAGON_Tools 目录
  * ![image](../image/Tools.png)
    | 文件夹             | 描述                                                                             |
    | ------------------ | -------------------------------------------------------------------------------- |
    | HVX 模拟器仿真样例 | ${HVX_SDK_PATH}\tools\HEXAGON_Tools\8.3.07\Examples\HVX                          |
    | DSP 开发手册文档   | ${HVX_SDK_PATH}\tools\HEXAGON_Tools\8.3.07\Documents\Hexagon_Document_Bundle.pdf |
4. Android NDK
- HVX SDK 需要依赖 Andriod NDK 来进行编译测试，NDK 需放置于${HVX_SDK_PATH}/tools 目录，Android NDK 需要开发者下载配置
- [NDK](https://link.zhihu.com/?target=https%3A//developer.android.google.cn/ndk/downloads/revision_history%3Fhl%3Dzh-cn)文章中使用 Linux 版本 android-ndk-r19c。（3.5.4版本 SDK 使用 android-ndk-r19c 即可
-----------------------------------------------------------

# Hexagon SDK 手机端运行
该部分以 ${HVX_SDK_PATH}/examples/common/gaussian7x7 为例进行说明
1. 工程编译
   * 设置环境变量:
    ``` 
    cd ${HVX_SDK_PATH}  
    source setup_sdk_env.source
    ```
  * Andorid 端应用编译：
    ``` 
    cd ${HVX_SDK_PATH}/examples/common/gaussian7x7
    make tree V=android_Release_aarch64 CDSP_FLAG=1
    ```
    Android 端可执行程序位于${HVX_SDK_PATH}/examples/common/gaussian7x7/android_Release_aarch64/ship/ gaussian7x7 目录
  * CDSP 端应用编译：
    ``` 
    make tree V=hexagon_Release_dynamic_toolv83_v66 VERBOSE=1
    ```
    CDSP 端算法 libgaussian7x7_skel.so 库位于${HVX_SDK_PATH}/examples/common/gaussian7x7/hexagon_Release_dynamic_toolv83_v68/ship/libgaussian7x7_skel.so

  * 相关编译选项解释：
    * toolv83: 表示 tools 版本是 8.3
    * V66: 表示 DSP 架构版本是 V66(SM8150, SM8250 使用 V66, SM8350 使用 V68)
    * CDSP_FLAG=1: 表示引用加载至 CDSP 端运行。SOC 中存在多个 DSP（如 ADSP，CDSP 等），编译时需显式指定
2. 签名
  手机系统中存在安全及认证机制，CDSP 库文件需要进行签名认证，以确保可以被正确加载运行。
  * 签名方法:
    签名方法通常有两种：开发签名和量产签名。（sm8150 之后，可以使用 Unsiged PD 方式进行算法验证测试，但部分硬件资源使用受限）
    * 开发签名
      应用计算法处在开发阶段（Debug Fuse Enabled on的阶段）时，可以采用开发签名进行调试。开发签名需要获取 设备端的序列号，然后生成相应的签名库文件 Testsig.so
      1. 获取序列号：
        ``` 
        adb wait-for-device root
        adb remount
        adb push ${HVX_SDK_PATH}/3.5.4/tools/elfsigner/getserial/CDSP/android_Release/getserial /data
        adb shell chmod 777 /data/getserial
        adb shell /data/getserial 
        ```
        如果getserial 失败了, 用下面的指令:
        ``` 
        adb shell cat /sys/devices/soc0/serial_number  //这里返回的是十进制,需要转化成十六进制
        ```
      2. 根据序列号生成 testsig.so 开发签名库
        ``` 
        cd ${HVX_SDK_PATH}/tools/elfsigner/
        elfsigner.py -t 0xXXXXXXXX // #0xXXXXXXXX为前面获取的序列号转换成十六进制的值
        ```
      3. 将签名库 testsig.so push 至手机端：
        ```
        adb wait-for-device root
        adb remount
        adb shell mkdir -p /vendor/lib/rfsa/adsp

        #testsig-0x6E07C1CE.so 为根据测试机序列号生成的开发签名库
        adb push ${HVX_SDK_PATH}/tools/elfsigner/output/testsig-0x6E07C1CE.so /vendor/lib/rfsa/adsp/ 
        ```
    * 量产签名
      量产签名主要用于批量生产时签名，需要对 DSP firmware 进行重新编译 。firmware 编译过程中会提取指定目录下算法库文件的哈希信息，然后存储于系统中，运行时会进行检测。（该方法需要针对每次算法调整都做签名）
    * Unsiged PD
      从 8150 开始，增加 Unsiged PD feature，即在 host 端进行 CDSP 初始化时开启 unsiged PD 功能
      该方式可除部分硬件资源使用受限外，对于开发者而言更加便利。
      ``` 
        // Unsigned PD
        if (1 == UNSIGNED_PD)
        {
            if (remote_session_control)
            {
                struct remote_rpc_control_unsigned_module data;
                data.enable = 1;
                data.domain = CDSP_DOMAIN_ID;
                retVal = remote_session_control(DSPRPC_CONTROL_UNSIGNED_MODULE, (void*)&data, sizeof(data));
                printf("remote_session_control returned %d for configuring unsigned PD.\n", retVal);
            }
            else
            {
                printf("Unsigned PD not supported on this device.\n");
            }
        }

      ```
3. 算法实机测试
  首先将编译生成的测试程序及库文件 push 至测试机中，该示例的测试应用编译路径为 ${HVX_SDK_PATH}/examples/common/gaussian7x7
     * push Android 端测试程序 
       ```
       adb wait-for-device root
       adb remount
       #进入gaussian7x7例子目录
       cd ${HVX_SDK_PATH}/examples/common/gaussian7x7  
       adb push android_Release_aarch64/ship/gaussian7x7 /vendor/bin/
       adb shell chmod +x /vendor/bin/gaussian7x7 
       ```
     * push DSP 端算法库文件
        ```
        adb push hexagon_Release_dynamic_toolv83_v66/ship/libgaussian7x7_skel.so /vendor/lib/rfsa/adsp/
        ```
     * 运行测试
        ``` 
        adb shell
        cd /vendor/bin
        ./gaussian7x7
        ```
        ![image](../image/out.png)
        上述为手机端运行测试流程，基于 hexagon-sim模拟器的算法运行测试会在后续章节进行介绍。
-----------------------------
## 算法实例分析
  继续 gaussian7x7（${HVX_SDK_PATH}/examples/common/gaussian7x7）为例进行说明。程序代码、编译文件和运行过程
1. SDK 工程目录结构
   ![image](../image/project-file.webp)
   * asm_src：算法 HVX 汇编代码实现
   * inc：IDL 文件
   * src：CPU 侧代码实现和算法 DSP 侧实现(包含HVX Intrinsic代码,DSP 侧代码提供了 HVX 汇编及 HVX Intrinsic 两种代码实现)
   * android.min：CPU 侧代码编译的 makefile 配置文件
   * hexagon.min：DSP 侧代码编译的 makefile 配置文件
  
      处理器间（CPU，DSP）通信由 Fastrpc 完成。算法调用过程解析通过 idl 编译生成的函数接口映射来处理。
      1. IDL 映射文件
        inc/gaussian7x7.idl 为该例程的映射文件，用来定义 CPU 和 DSP 同步使用的接口，包括函数、结构体等。
        RPC 调用过程需要调用反射机制实现，HVX 的调用反射基于 IDL 来实现，使用 IDL 来定义调用接口，以使 CPU 能完成 DSP 的函数调用。
        ![image](../image/dsp-cpu.webp)
        编译器根据 idl 文件编译生成 gaussian7x7.h、gaussian7x7_stub.c 和gaussian7x7_skel.c 三个文件。

        ![image](../image/guss.png)
        下面介绍一下gaussian7x7的idl定义：
        ``` 
        AEEResult Gaussian7x7u8
        ( 
            in sequence<uint8> src, // input buffer of unsigned 8-bit values
            in uint32 srcWidth,  // width of region of interest contained in src image 
            in uint32 srcHeight, // height of region of interest contained in src image
            in uint32 srcStride,           // stride of the src image 
            rout sequence<uint8> dst,  // output buffer of unsigned 8-bit values 
            in uint32 dstStride,           // stride of the dst image 
            in int32 LOOPS,                // number of times to iterate 
            in int32 wakeupOnly,           // flag to skip processing 
            inrout int32 dspUsec,           // profiling result in uSec 
            inrout int32 dspCyc             // profiling result in cycles 
        );
        ```
       * Sequence为表示数组参数，转义接口为 data 指针及 数组 size
       * in 表示为参数为输入属性，生成为 const 类型
       * 因此 in sequence<uint8> src 对应的接口参数为 const uint8* imgSrc, int srcLen
       * rout sequence<uint8> dst 中 rout 表示输出属性，生成对应的接口参数为 uint8* imgDst, int dstLen
       * in uint32 srcWidth 生成对应的接口参数为 uint32 srcWidth
       * rout int32 dspUsec 生成对应的接口参数为 int32* dspUsec

      生成三个文件位于 android_Release_aarch64 和 hexagon_Release_dynamic_toolv83_v66 文件夹内，如下图所示
      ![image](../image/guss-rela.png)
      ![image](../image/guss-rela-2.webp)
      在编程过程中, CPU 端会将 gaussian7x7.h 和 gaussian7x7_stub.c 代码编译后链接至 CPU 端的应用程序，DSP 端会将 gaussian7x7.h 和 gaussian7x7_skel.c 代码编译后链接生成 DSP 端运行库。
      基于 IDL 生成函数接口如下，位于 gaussian7x7.h 中
      ``` 
      QAIC_HEADER_EXPORT AEEResult __QAIC_HEADER(benchmark_gaussian7x7)(remote_handle64 _h, const uint8* src, int srcLen, uint32 srcWidth, uint32 srcHeight, uint32 srcStride, uint8* dst, int dstLen, uint32 dstStride, int32 LOOPS, int32 wakeupOnly, int32* dspUsec, int32* dspCyc) __QAIC_HEADER_ATTRIBUTE;
      ```
  2. CPU 端算法流程
    CPU 端的流程图如下（基于 ${HVX_SDK_PATH}/examples /common /gaussian7x7/gaussian7x7.c）:
    ![image](../image/cpu.png)
  - 通常在硬件设备和用户空间共享数据时，会基于 ION（后续 DMA BUF Heap）实现共享大尺寸连续物理内存，以减少内存拷贝开销实现 zero copy。该例程中使用 rpcmem_init 函数进行初始化（sm8350不再需要 rpcmem 初始化调用）
      ```
      rpcmem_init();
      ```

  - 初始化DSP，设置时钟，带宽等参数
    ```
    // call dspCV_initQ6_with_attributes() to bump up Q6 clock frequency
    // Since this app is not real-time, and can fully load the DSP clock & bus resources 
    // throughout its lifetime, vote for the maximum available MIPS & BW.
    dspCV_Attribute attrib[] =
    {
        {DSP_TOTAL_MCPS, 1000},                 // Slightly more MCPS than are available on current targets
        {DSP_MCPS_PER_THREAD, 500},             // drive the clock to MAX on known targets
        {PEAK_BUS_BANDWIDTH_MBPS, 12000},       // 12 GB/sec is slightly higher than the max realistic max BW on existing targets.
        {BUS_USAGE_PERCENT, 100},               // This app is non-real time, and constantly reading/writing memory
    };

    retVal = dspCV_initQ6_with_attributes(attrib, sizeof(attrib)/sizeof(attrib[0]));
    printf("return value from dspCV_initQ6() : %d \n", retVal);
    VERIFY(0 == retVal);
    ```
  - 基于 rpcmem 申请 buf，高通内部使用 ION 进行硬件设备内存共享，可以有效江都 Fastrpc 通信时间，基于常规堆内存分配，会引发数据拷贝操作
    ``` 
    // allocate ion buffers on CDSP side
    VERIFY(0 != (src = (uint8_t*)rpcmem_alloc(ION_HEAP_ID_SYSTEM, RPCMEM_DEFAULT_FLAGS, srcSize)));
    printf("src - allocated %d\n", (int)srcSize);
    VERIFY(0 != (dst = (uint8_t*)rpcmem_alloc(ION_HEAP_ID_SYSTEM, RPCMEM_DEFAULT_FLAGS, dstSize)));
    printf("dst - allocated %d\n", (int)dstSize);
    ```
  - 生成伪随机图像数据
    ```
    // populate src buffer (with a simple pattern)
    for (j = 0; j < srcHeight; j++)
    {
        uint8_t *ptr = &src[j * srcStride];
        for (i = 0; i < srcWidth; i++)
        {
            *ptr++ = i + j;             // some incrementing pattern fill
        }
    }
    ```
  - 进行算法调用测试
    ```
    unsigned long long t1 = GetTime();
    for (i = 0; i < LOOPS; i++)
    {
        // For HVX case, note that src, srcStride, dst, dstStride all must be multiples of 128 bytes.
        // The HVX code for this example function does not handle unaligned inputs.
        retVal = gaussian7x7_Gaussian7x7u8(src, srcSize, srcWidth, srcHeight, srcStride, dst, dstSize, dstStride);
    }

    unsigned long long t2 = GetTime();
    VERIFY(0 == retVal);
    #ifdef __hexagon__
    printf("run time of gaussian7x7_Gaussian7x7u8: %llu PCycles (from %llu-%llu) for %d iterations\n", t2-t1, t1, t2, LOOPS);
    printf("To apply timefilter to profiling results, add this to simulation cmd line: --dsp_clock 800 --timefilter_ns %d-%d\n", (int)(t1/0.8), (int)(t2/0.8));
    #else
    printf("run time of gaussian7x7_Gaussian7x7u8: %llu microseconds for %d iterations\n", t2-t1, LOOPS);
    #endif
    printf("return value from gaussian7x7_Gaussian7x7u8: %d \n", retVal);

    // validate results
    Gaussian7x7u8_ref(src, srcWidth, srcHeight, srcStride, ref, dstStride);
    ```
  - 运算结果比较
    ```
    int bitexactErrors = 0;
    printf( "Checking for bit-exact errors... \n");
    for (j = 3; j < dstHeight-3; j++)
    {
        for (i = 3; i < dstWidth-3; i++)
        {
            if (ref[j * dstStride + i] != dst[j * dstStride + i])
            {
                bitexactErrors++;
            }
        }
    }
    printf( "Number of bit-exact errors: %d \n", bitexactErrors);
    VERIFY(0 == bitexactErrors);
    ```
  - 释放资源
    ```
    if(src)
    { 
        rpcmem_free(src);
    }
    if(dst)
    { 
        rpcmem_free(dst);
    }
    // free ion buffers
    rpcmem_deinit();

    if(ref)
    {
        free(ref);
    }

    printf("calling dspCV_deinitQ6()... \n");
    retVal = dspCV_deinitQ6();
    printf("return value from dspCV_deinitQ6(): %d \n", retVal);

    if (0 == (retVal | nErr))
    {
        printf("- success\n");
        return 0;
    }
    else
    {
        printf("- failure\n");
        return -1;
    }
    ```
3. DSP 端算法流程
  DSP 端的流程图如下（基于 ${HVX_SDK_PATH}/examples /common /gaussian7x7/gaussian7x7_imp.c）:
  ![image](../image/dsp-algo.png)
  回调函数的流程图如下:
  ![image](../image/call-back.png)
  DSP 端函数接口如下：
  ``` 
  AEEResult gaussian7x7_Gaussian7x7u8(const uint8* imgSrc, int srcLen, 
  uint32 srcWidth, uint32 srcHeight, uint32 srcStride, uint8* imgDst, 
  int dstLen, uint32 dstStride) 
  ```
  - 系统架构及参数有效性检测
    ```
    // only supporting HVX version in this example.
    #if (__HEXAGON_ARCH__ < 60)
    return AEE_EUNSUPPORTED;
    #endif

    // record start time (in both microseconds and pcycles) for profiling
    #ifdef PROFILING_ON
    uint64 startTime = HAP_perf_get_time_us();
    uint64 startCycles = HAP_perf_get_pcycles();
    #endif
    // Only supporting 128-byte aligned!!
    if (!(imgSrc && imgDst && ((((uint32)imgSrc | (uint32)imgDst | srcWidth | srcStride | dstStride) & 127) == 0)
            && (srcHeight >= 7)))
    {
        return AEE_EBADPARM;
    }
    ```
  以上是异常检测的代码实现，包括有：

    1. 如果 DSP 版本小于 60，没有 HVX 硬件，退出
    2. 如果 src，dst 地址是NULL，退出
    3. 如果 src，dst 地址不对齐，退出，因为代码实现(Gaussian7x7)只支持128对齐的数据
    4. 如果输入图像高度小于7，退出，Gaussian7x7代码无法正确运行
  - 初始化并发参数
    ``` 
    // Determine if it is safe (from an audio/voice/camera concurrency perspective) to run a compute function now
    dspCV_ConcurrencyAttribute attrib[1] = 
    {
        {COMPUTE_RECOMMENDATION, 0},  // query for compute concurrency recommendation
    };
    dspCV_concurrency_query(attrib, 1);
    if (COMPUTE_RECOMMENDATION_NOT_OK == attrib[0].value)
    {
        // return error back to application
        return AEE_EBADSTATE;
    }

    // Determine if HVX is available and in what configuration
    dspCV_hvx_config_t hvxInfo = {0};
    ```
  - 设置 HVX 运行模式为 DSPCV_HVX_MODE_128B，早期的 HVX 有 128B 和 64B 两种模式，sm845之后只有 128B 模式
    ```
    // for sake of example, assume only 128B implementation is available (i.e. intrinsics)
    hvxInfo.mode = DSPCV_HVX_MODE_128B;
    ```
  - 进行多线程运行设置
    ```
    // Call utility function to prepare for a multi-threaded HVX computation sequence.
    dspCV_hvx_prepare_mt_job(&hvxInfo);

    // Check results and react accordingly. Treat failure to acquire HVX as a failure
    if (hvxInfo.numUnits <= 0)
    {
        dspCV_hvx_cleanup_mt_job(&hvxInfo);
        return AEE_EFAILED;
    }

    int numWorkers = hvxInfo.numThreads;
    // split src image into horizontal stripes, for multi-threading.
    dspCV_worker_job_t   job;
    dspCV_synctoken_t    token;

    // init the synchronization token for this dispatch. 
    dspCV_worker_pool_synctoken_init(&token, numWorkers);
    ```
  - 创建线程，以 gaussian7x7_callback 为回调函数。主线程使用 worker_pool_synctoken_wait(&token); 进行线程同步，该函数基于下述dspCV_worker_pool_synctoken_jobdone 来同步任务完成状态
    ```
    unsigned int i;
    for (i = 0; i < numWorkers; i++)
    {
        // for multi-threaded impl, use this line.
      (void) dspCV_worker_pool_submit(job);
        // This line can be used instead of the above to directly invoke the 
        // callback function without dispatching to the worker pool. 
        //job.fptr(job.dptr);
    }
    dspCV_worker_pool_synctoken_wait(&token);
    ```
  - 回调函数（gaussian7x7_callback）， 使用 dspCV_hvx_lock 锁 HVX 资源；使用 dspCV_worker_pool_synctoken_jobdone 函数结束子线程任务运算
  ```
  static void gaussian7x7_callback(void* data)
  {
      gaussian7x7_callback_t    *dptr = (gaussian7x7_callback_t*)data;

      // lock HVX, 128B mode preferred. Main thread has already confirmed HVX reservation.
      int lockResult = dspCV_hvx_lock(DSPCV_HVX_MODE_128B, 0);
      // 64B mode is also acceptable
      if (0 > lockResult) 
      {
          lockResult = dspCV_hvx_lock(DSPCV_HVX_MODE_64B, 0);
      }
      
      if (0 > lockResult)
      {
          // this example doesn't handle cases where HVX could not be locked
          FARF(ERROR,"Warning - HVX is reserved but could not be locked. Worker thread bailing!");
          return;
      }
      // ....
      // ....
      
      // If HVX was locked, unlock it.
      dspCV_hvx_unlock();
      // release multi-threading job token
      dspCV_worker_pool_synctoken_jobdone(dptr->token);
  }
  ```
  - 回调函数内部循环体
    循环体中通过 unsigned int jobCount = worker_pool_atomic_inc_return(&(dptr->jobCount)) - 1; 通过原子计数来计算当前任务的执行数据地址偏移
    算法实现主要位于 Gaussian7x7u8PerRow 函数中，函数采用逐行实现的思路。
    ```
    // atomically add 1 to the job count to claim a stripe.
    unsigned int jobCount = dspCV_atomic_inc_return(&(dptr->jobCount)) - 1;

    // if all horizontal stripes have been claimed for processing, break out and exit the callback
    if (jobCount * dptr->rowsPerJob >= dptr->height)
    {
        break;
    }

    // Set pointers to appropriate line of image for this stripe
    unsigned char *src = dptr->src + (dptr->srcStride * dptr->rowsPerJob * jobCount);
    unsigned char *dst = dptr->dst + (dptr->dstStride * dptr->rowsPerJob * jobCount);
    // ...
    Gaussian7x7u8PerRow(pSrc, dptr->srcWidth, dst, lockResult);
    //....
    ```
  - 其他
    DSP 端进行数据处理前，可以通过 L2 预取操作以加速数据的存取。

    数据预取操作会使用硬件提前完成数据从 DDR 到 L2 cache 的搬运操作，有效提高数据 load 的效率。

    通常会采用 Ping-Pong 的思想进行数据预取，DSP 侧使用 L2fetch 函数在当前循环操作中预取下一次循环的数据，以使得数据搬运和数据运行并行化
    ```
    // initiate L2 prefetch (first 7 rows)
    long long L2FETCH_PARA = CreateL2pfParam(dptr->srcStride, dptr->srcWidth, 7, 0);
    L2fetch( (unsigned int)src, L2FETCH_PARA);
    // next prefetches will just add 1 row
    L2FETCH_PARA = CreateL2pfParam(dptr->srcStride, dptr->srcWidth, 1, 0);
    ```
https://zhuanlan.zhihu.com/p/473724533