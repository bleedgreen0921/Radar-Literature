# Raw_ADC_Data

## **《Radar Image Reconstruction from Raw ADC Data**

**using Parametric Variational Autoencoder with**
**Domain Adaptation》**

**使用具有域自适应的参数变分自编码器从原始ADC数据重建雷达图像**

提出了一种基于**参数变分自编码器**的人体目标检测和定位框架，该框架直接使用调频连续波雷达的**原始模数转换器数据**。我们提出了一种**参数约束的变分自编码器**，具有**残差和跳跃连接**，能够在距离角图像上生成聚类和局部目标检测。此外，为了避免使用真实雷达数据在所有可能的情况下训练所提出的神经网络的问题，我们提出了**域自适应策略**，即我们首先使用基于射线追踪的模型数据训练神经网络，然后使网络适应实际传感器数据。

**核心问题与动机**

![image-20250603151607090](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250603151607090.png)

1. 传统流程的缺陷：
   - 传统雷达处理流程（2D FFT → MTI滤波 → 波束成形 → CFAR检测 → DBSCAN聚类）计算量大，易受多径反射干扰，导致**鬼影目标**和**漏检**（尤其在室内复杂环境）。
   - 现有深度学习方案多依赖**预处理后的数据**（如距离-多普勒图像RDI），未能充分利用原始ADC数据，且预处理（尤其是FFT）仍是计算瓶颈。
2. 数据瓶颈：
   - 收集覆盖所有场景的**真实雷达数据训练网络不现实**。

**解决方案：参数化VAE + 域适应**

![image-20250603151528435](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250603151528435.png)

1. **参数化频率提取层（CFEL） Complex Frequency Extraction Layer**：

   - **替代传统2D FFT**，作为网络的第一层。
   - 事实证明，在对原始ADC数据进行训练时，系统经常陷入局部最小值，因此使用**可学习的参数化滤波器核** `f_{M,N}(f_{ft}, f_{st}; m, n = (e^(j2*pi*f_{ft}*m/f_s^{ft}))*(e^(j2*pi*f_{st}*n/f_s^{st}))` 直接从时域ADC数据提取距离-多普勒特征。
   - 优势：可**自适应聚焦关键频率区域**（优于FFT的均匀分辨率），减少计算量，支持端到端处理。

2. **变分自编码器 VAE（Variational Autoencoder）架构**：

   ![image-20250603152901187](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250603152901187.png)

   ​	基于编码器-解码器的结构的一般思想是将一些输入数据压缩到较小的潜在空间中，以便仍然可以从该压缩表示中获得所需的输出数据。理想情况下，潜在空间应捕获一些高级特征，如真实目标和幻影目标的存在/不存在，然后在解码器中使用这些特征来突出显示目标并删除幻影目标。在实践中，编码器网络选择的数据表示可能无法解释，在强过拟合的情况下，单个输入示例甚至可能被映射到一些没有任何高级含义的特定数字。实现潜在空间连续性和减少过拟合的一种方法是使用VAE。在VAE中，输入不是被编码到潜在空间中的单个点中，而是被编码到潜空间上的分布中。

   - **编码器**：6个残差块，逐步下采样，输出潜在空间的分布参数（均值μ、方差σ）。
   - **潜在空间**：通过KL散度正则化确保连续性，避免过拟合。
   - **解码器**：5个上采样块，融合编码器跳跃连接，最终输出RAI（检测与聚类结果）。
   - **多天线处理**：CFEL独立处理各天线数据，后期融合（1×1卷积）。

3. **域适应（DA）策略**：

   - **步骤1**：使用**射线追踪生成的合成点目标数据**预训练网络（覆盖全范围-角度空间）。
   - **步骤2**：在**少量真实数据**上微调，添加**权重差异正则化损失** `L_DA`（公式7），约束网络权重不过度偏离预训练模型，保留通用的角度估计能力。
   - **优势**：解决真实数据稀缺问题，提升泛化性。

------

## **《Echoes Beyond Points: Unleashing the Power of Raw》**

**Radar Data in Multi-modality Fusion》**

**核心问题：**

传统处理流程（如CFAR检测）生成的**点云数据稀疏且信息损失严重**（尤其弱信号），导致检测精度受限，难以与其他传感器（如相机）有效融合。

**解决方法：EchoFusion**

**1. 核心思想**

- **跳过传统雷达信号处理**：直接使用原始雷达数据（如ADC、RT、RD等），保留完整的距离、速度信息。
- **极坐标对齐注意力（Polar-Aligned Attention, PAA）**：
  - **图像融合（列向）**：BEV空间中相同方位角（φ）的查询对应图像同一列，通过交叉注意力聚合语义特征。
  - **雷达融合（行向）**：BEV查询的距离（r）与雷达RT数据的距离维度对齐，按行聚合雷达特征。
  - **优势**：避免显式解码方位角（AoA），隐式学习相位差中的角度信息，实现高效跨模态融合。

**2. 整体流程**

1. 特征提取：
   - 图像：ResNet-50 + FPN提取多尺度特征。
   - 雷达：原始数据（RT为主）经定制ResNet-50处理。
2. BEV查询生成：
   - 初始化极坐标BEV查询（*Q**l*∈R*R**l*×*A**l*×*d*），覆盖距离*r*和方位角*ϕ*。
3. PAA融合：
   - **图像阶段**：同*ϕ*的查询关注图像对应列的特征。
   - **雷达阶段**：同*r*的查询关注雷达RT数据对应行的频谱特征。
4. 检测解码：
   - PolarFormer解码器生成BEV特征图，预测3D边界框（中心偏移、尺寸、朝向）。

------

## **《ADCNet: Learning from Raw Radar Data via Distillation》**

![image-20250603171438916](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250603171438916.png)



------

## 《Azimuth Super-Resolution for FMCW Radar in Autonomous Driving》

- 提出了一种名为ADC-SR的**方位超分辨率模型**，该模型考虑了复杂的ADC雷达信号，并预测了来自看不见的接收器的信号，能够产生更高分辨率的RAD图。


- 提出了一种名为hybrid SR的混合模型，用于在与我们的ADC-SR结合时提高RAD-SR的性能。为了与所有基线模型进行比较，还提出了一种下采样方法进行评估。


- 提出了一个名为Pitt radar的MIMO雷达数据集，其中包含用于基准测试的ADC信号。


- 模型不仅在我们收集的Pitt雷达和一个基准数据集上实现了令人满意的方位超分辨率，而且大大改进了现有的目标检测器。



------

## 《T-FFTRadNet: Object Detection with Swin Vision Transformers from Raw ADC Radar Signals》

1. 研究背景与核心问题

   传统方法缺陷：

   1. **点云（Point Cloud）**：雷达点云稀疏，信息提取效率低。
   2. **RAD立方体（Range-Azimuth-Doppler Cube）**：需多次FFT预处理，计算成本高。
   3. **输入标准化**：RD/RAD输入需通道归一化，增加预处理负担。

   核心问题：

   如何**直接处理原始ADC信号**，避免FFT预处理，同时兼容不同雷达配置（HD/LD）和输入类型（ADC/RD/RAD）？

2. 核心创新：T-FFTRadNet

   整体架构如下图

![image-20250605103131122](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250605103131122.png)

**1️⃣ 原始输入：ADC Matrices（Rx × Chirps）**

- 原始接收通道（Rx）与 chirps 构成的三维张量。
- 表示雷达接收到的原始 IQ 数据。

------

**2️⃣ Fourier-Net（可学习的傅里叶变换层，用于时频变换）**

这是一个将 ADC 信号映射到 Range-Angle (RA) 特征域的网络模块。替代传统FFT，直接处理原始ADC信号。

数学原理：

- 初始化权重为DFT矩阵：` w(k,m)=exp(−j2π/N*​km)`
- 通过端到端训练优化权重，学习**传感器特定特征**。

**输入：**

- 原始 ADC matrix。

**处理步骤：**

- `C`：1×1 Conv
- `P`：Permute 维度（通道交换 Rx 和 chirp 维度）
- `CPLX`：复数线性变换（对 IQ 分量做频域变换）
- `IN-2D`：InstanceNorm2D 对幅度谱进行归一化
- 输出为：**RA 特征图**（Range × Angle）

------

**3️⃣ Swin Transformer Backbone**

对 RA 特征图提取层次化特征（借助 **Swin-T 架构**）。

**操作流程：**

- `Patch Partition`：将输入图像分割成非重叠窗口（如 2*2）
- `Linear Embedding`：将每个 patch 投影为固定维度的向量
- **四个 Stage**（Layer 数量为 [2, 2, 6, 2]）：
  - 每个 Stage 内有多个 Swin Transformer Block（如 W-MSA + SW-MSA）
  - `Patch Merging` 降维（H,W 降为 1/2，通道 ×2）
  - 最终输出多层特征：用于 Detection 和 Segmentation 分支

------

**4️⃣ 两大特征分支：**

**🔹 Segmentation Head：**

- 输入：RA Features（来自 Fourier-Net 或浅层 Swin）
- 操作：
  - BI（Bilinear Interpolation）
  - BB（3×3 Conv/BN/ReLU）
  - C（1×1 Conv）
- 输出：每个点的语义标签（如人体、背景）

**🔹 Detection Head：**

- 输入：RA Features（可来自 Swin 深层）
- 操作：
  - CB：3×3 Conv + BN
  - C：1×1 Conv
  - 输出：
    - 2D Bounding Box (回归)
    - 类别分类
    - 存在性检测（是否有人）

------

**5️⃣ Range-Angle Decoder：**

用于进一步从高维 Swin 特征中恢复空间信息。

- 接收高维 Swin Features
- 采用：
  - `S`：通道交换
  - `C`：1×1 Conv
  - `BB`：Conv/BN/ReLU
- 最终送入 Detection Head 与 Segmentation Head 分别回归位置信息与语义。

------

## 《RADDet: Range-Azimuth-Doppler based Radar Object Detection》

### 主要贡献：

- 介绍了一种新的数据集，其中包含RAD表示形式的雷达数据，并为各种对象类提供了相应的注释。该数据集可用于该领域的未来研究。
- 提出了一种自动注释方法，该方法在RAD数据的所有维度上以笛卡尔形式生成地面实况标签。
- 提出了一种新的雷达目标检测模型。我们的模型采用**基于ResNet的主干**。骨干网络的最终形式是在对雷达数据的深度学习模型进行系统探索后实现的。受YOLO头的启发，我们提出了一种新的双检测头，在RAD数据上具有3D检测头，而在笛卡尔坐标系中的数据上具有2D检测头。
- 我们提出的模型与众所周知的基于图像的目标检测模型进行了广泛的评估。结果表明，我们的方案在雷达数据的目标检测任务中达到了最先进的性能。我们研究结果的意义表明，雷达是基于相机和激光雷达的目标检测方法的可行竞争对手。

### 自动注释方法：

​	依靠立体视觉进行类别标签。整个过程描述如下。首先，使用OpenCV4的**半全局块匹配算法**从校正的立体图像对生成视差图。然后，将**预训练的Mask RCNN**模型应用于左图像，以提取实例分割掩模。然后将预测掩模投影到视差图上。最后，使用三角剖分，生成具有预测类别的实例点云输出。然后，使用III-A中获得的投影矩阵将点云实例转换为雷达帧。

![image-20250610202910878](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250610202910878.png)



#### 上半部分：雷达处理流程

- **2D-OSCFAR**（**Ordered Statistic CFAR**）

  - 用于雷达 Range-Doppler 图像进行目标点提取（亮点）

  - 输出候选目标点（高反射点）

- **Morphological Extension**
  - 图像形态学扩张操作，对 CFAR mask 进行空间补全，避免错漏小目标

- **Apply Mask**
  - 用扩张后的 mask 提取雷达点图中的候选区域（增强的目标热力图）

- **Azimuth Peak Detection + 3D DBSCAN**

  - 角向聚类 + 密度聚类算法，用于将雷达高能量点分组形成“目标簇”

  - 形成类目标对象的 3D 聚类轮廓

- **Polar to Cartesian Conversion**

  - 将雷达图从极坐标系（range, azimuth）转换为笛卡尔坐标系（x, y）

  - 生成更直观的目标检测雷达图

#### 下半部分：图像深度估计与目标分割流程

- **Disparity Estimation**
  - 使用双目图像计算视差图（stereo disparity）
  - 得到每个像素的**深度信息**
- **Mask-RCNN 实例分割**
  - 对 RGB 图像中各个对象进行实例分割（如行人、车辆）
  - 生成精确的 instance mask 和类别信息
- **Apply Mask to Disparity**
  - 将分割掩码应用于视差图中，仅保留目标区域的深度数据
- **Project to Radar Frame**
  - 将图像坐标系中的实例点云根据内参映射到雷达坐标系（空间对齐）

#### 上下融合流程：Radar ↔ Image Matching

- 在雷达坐标系中通过**点云匹配（cluster matching）**将雷达聚类与图像目标关联
- 最终生成：
  - 雷达图像中目标的位置、类别、尺寸
  - 用于自动标签的伪 ground truth（pseudo-GT）

#### 核心创新点：

- 用增强的 **CFAR + 3D 聚类算法** 提取雷达目标点
- 利用 **立体视觉深度估计 + 实例分割** 提供稠密语义信息
- 构建 **自动标签生成框架**，实现弱监督学习 / 雷达目标检测数据生成

### RadarResNet

![image-20250610204454201](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250610204454201.png)

#### RadarResNet 特征提取网络

**输入：**

- 输入是一个 **雷达三维特征图**（Tensor），包含三个视图维度：
  - **Range-Azimuth (RA)**：二维空间热图（极坐标）
  - **Range-Doppler (RD)**：目标速度信息
  - 组合成 3D 输入：尺寸为 `[256, 256, 64]`

使用 ResNet-like 结构，包含：

- **Residual Block × 2/4/8/16**：逐层加深网络，保留特征
- **MaxPool**：下采样，保持空间稀疏特征
- 激活函数：**ReLU**
- 正则：**BatchNorm**
- 输出：`[16, 16, 256]` 的高维语义特征图

#### Dual Detection Head

##### RAD YOLO Head（3D检测头）

- **目标**：从 3D 雷达特征中预测 `[x, y, z]` 位置和 `[w, h, d]` 尺寸的 **3D框**

  **输出结构**：

  ​	`[16,16,4,num_anchors,7+num_classes]`

  - 4：为 Doppler 层维度
  - 7：表示 [x, y, z, w, h, d, objectness]

  **优化**：

  - 使用 K-means 聚类选取 anchor
  - 每个 anchor 表示一个候选框
  - 最终输出送入 3D NMS

##### YOLO Head（2D检测头）

- **目标**：在笛卡尔坐标系下进行常规 `[x, y, w, h]` 框预测（如 BEV 目标检测）

  **步骤**：

  1. **Coordinate Transformation Layer**：
     - 将 `[r, θ]` 极坐标形式转换为 `[x, z]` 笛卡尔形式
     - 利用神经网络（FC层 + ResBlock）代替硬编码转换
     - 输出维度变为 `[32, 16, 256]`
  2. **YOLO Head**：
     - 常规 anchor-based 结构，输出二维框
     - 分类+置信度+box regression

##### 损失函数设计（Multi-task Loss）

`L_total=β⋅L_box+L_obj+L_class`

- `Lbox`：采用先进 IoU-based 回归损失（如 CIoU/GIoU）用于 3D 或 2D 框
- `Lobj`：Focal Loss，用于解决正负样本不平衡问题（负样本 α=0.01）
- `Lclass`：交叉熵损失，用于目标分类
- 系数 `β=0.1`：防止 box loss 主导训练过程

------

## Complex-valued Convolutional Neural Networks for Enhanced Radar Signal Denoising and Interference Mitigation

**用于增强雷达信号去噪和干扰抑制的复值卷积神经网络**

本文提出使用**复值卷积神经网络（CVCNN）**来解决雷达传感器之间的相互干扰问题。我们将之前开发的方法扩展到**复频域**，以便根据雷达数据的物理特性对其进行处理。这不仅提高了数据效率，而且改善了滤波过程中相位信息的保存，这对于进一步的处理（如角度估计）至关重要。我们的实验表明，使用CVCNN可以提高数据效率，加快网络训练速度，并大大改善干扰消除过程中相位信息的保存。

复值分析的使用引入了一种**归纳偏差（inductive bias）**，限制了网络中的自由度。然而，这种限制可以大大有利于学习行为，因为它强制信号根据数据的物理特性进行转换，降低问题的复杂性。

应用于复值数据的实值CNN（RVCNN）通常使用单独的实值通道来表示实部和虚部。在这种方法中，CNN需要基于训练样本学习复值数据的实部和虚部之间的关系。相比之下，CVCNN使用复值分析执行所有操作，因此将实部和虚部之间的关系纳入模型架构，而不是学习参数。这种方法的动机来自雷达信号本身的复值性质。

### **复数卷积神经网络（CVCNN）**

<img src="C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image.png" alt="image"  />

- **复数卷积**：将复数运算分解为实部-虚部的四元操作

  二维卷积运算的卷积核由大小为`kx × ky × Cin × Cout`的四维张量`W_{ijkl}`组成。这里`kx`和`ky`表示核的空间大小，`Cin`表示输入滤波器通道的数量，`Cout`表示输出滤波器通道的编号。考虑到滤波器通道的数量是2的倍数，我们总是可以将通道分为两部分，一部分用于实部，另一部分用于虚部。因此，单独的核的大小为`kx × ky × Cin/2 × Cout/2`。

  由于卷积算子（∗）是分布的，我们得到`W∗ h = (A + iB) ∗ (x + iy) = A ∗ x − B ∗ y + i(B ∗ x + A ∗ y)`

  将复向量`h=x+iy`与复核矩阵`W=A+iB`进行卷积。因此，复值卷积可以作为一系列实值卷积来执行。

- **复数批量归一化（BN）**：对复数数据采用**白化处理**与**缩放平移**

  深度CNN依赖于批归一化（BN）操作来归一化激活并加速训练。由于BN的标准公式仅适用于实值激活，复值网络需要不同的方法。提出了一种实现复数BN的可能性，使用**白化程序**来标准化复数。首先，对输入`z∈C`进行白化处理，然后执行标准BN。因此，白化步骤后的向量分别使用γ和β进行缩放和移位。

  `BN（z）=γ⊙(V)^(-1/2)（z−E[z]）+β`

  其中V是一个2×2的正定（半）协方差矩阵。由于所提出的BN需要V逆的平方根，因此通过将I加到V上来确保矩阵的正定性，这被称为Tikhonov正则化。

- **复数ReLU激活**：对实部和虚部分别应用ReLU

  `CReLU(z) = ReLU(R(z)) + iReLU(I(z))`

------

## 《CubeLearn: End-to-end Learning for Human Motion Recognition from Raw mmWave Radar Signals》

**从原始毫米波雷达信号进行人体运动识别的端到端学习**

首次使用**堆叠的复数线性层**来代替传统的DFT预处理，直接从原始毫米波FMCW雷达数据中提取信息，并与下游的深度分类器一起构建端到端的深度神经网络，该分类器直接将原始雷达复数数据立方体作为识别任务的输入。

- **现有技术局限**：

  - 传统流程依赖

    离散傅里叶变换（DFT）预处理，存在以下问题：

    - 固定基函数导致分辨率受限（如距离分辨率>5cm）。
    - 输出包含大量冗余信息（如环境反射噪声）。
    - 无法针对任务自适应优化特征提取。

  - 端到端学习尝试中，复数信号处理不充分（如仅用实部或分离实/虚部会丢失物理意义）。

### CubeLearn Architecture

![image-20250611184806770](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250611184806770.png)

- **设计目标**：替代传统DFT，构建端到端可学习的预处理模块，直接从原始复数雷达信号中提取任务相关特征。

- **核心创新**：

  - **堆叠复数线性层**：模仿DFT处理流程（距离→速度→角度），但权重可学习。

    - 距离维度：复数线性层处理单啁啾（chirp）数据。
    - 速度/角度维度：后续复数层提取跨啁啾/天线的相位变化。

  - **复数处理机制**：保留原始信号的物理意义（幅度+相位），避免信息丢失。

  - **初始化策略 CubeLearn Module Weight Initialization** ：用DFT基函数初始化权重，作为先验知识加速收敛。

    将大小为N的传统DFT应用于大小为M的输入，可以表示为`ˆ𝑥[𝑘] =
    𝑀-1∑︁ 𝑚=0
    𝑒^{−𝑗*2𝜋/𝑁*𝑘𝑚}*𝑥[𝑚] `

    由于DFT为时域到频域的线性变换，对于输入大小𝑀 以及输出大小𝑁 (𝑁 ≥ 𝑀),我们有：

    `𝑜𝑢𝑡𝑝𝑢𝑡[𝑘] =
    𝑀-1∑︁ 𝑚=0
    𝑤(𝑘,𝑚)*𝑖𝑛𝑝𝑢𝑡[𝑚] `

    初始化权重为`𝑤(𝑎, 𝑏) = 𝑒^{−𝑗2𝜋/𝑁𝑎b}`

    最终输出为`𝑜𝑢𝑡𝑝𝑢𝑡[𝑘] =
    𝑀-1 ∑︁ 𝑚=0
    𝑒^{−𝑗2𝜋/𝑁𝑘𝑚}𝑖𝑛𝑝𝑢𝑡[𝑚]`

  - **输出处理**：计算模值后输入下游分类器（保留幅度信息）。

    为了使所提出的CubeLearn模块相当于传统DFT的一个插件，我们在将处理后的数据输入下游神经网络分类器之前计算绝对（模）值。

------

##  Data-Driven Radar Processing Using a
Parametric Convolutional Neural Network

在本文中，我们提出了一种**参数卷积神经网络**，该网络通过**2D sinc**滤波器或**2D小波滤**波器核来模拟快速和慢速雷达数据的雷达预处理，以提取各种人类活动的分类特征。在训练过程中，只学习2D sinc滤波器或2D小波的滤波器参数，从而为分类任务优化特征表示。

### 传统路线

传统的信号预处理涉及1D**运动目标指示（MTI）**滤波，以消除静态目标的响应，以及影响前几个距离箱的Tx-Rx泄漏。静止物体的反射可能会掩盖其他运动目标的反射，从而限制了它们在RDM或多普勒频谱图上的可见度。因此，MTI滤波器用于抑制这些静止物体和泄漏的贡献。

在几个MTI滤波器中，一个简单的1D MTI滤波器沿快速时间减去平均值，以消除扰乱第一个距离仓的Tx-Rx泄漏，然后沿慢速时间减去平均数，以消除由于静态或零多普勒目标引起的反射。

在沿快速时间（即帧内啁啾时间）应用1D加窗后，通过执行第一次FFT来提取目标的距离信息。通过监测目标峰值沿慢时间（即啁啾间时间）的变化来提取目标的多普勒信息。此操作的结果是一个二维矩阵，表示在距离和速度上的接收功率谱，也称为RDM。接收和去噪的中频数据存储在大小为Nc×Ns的矩阵中，其中Nc是在慢时间切片中考虑的啁啾数量，Ns是每个啁啾的发射样本数量。

在传统的处理流水线中，上述预处理之后是特征图像生成，如多普勒频谱图或距离多普勒时间雷达数据立方体，然后将其输入到深度卷积神经网络（DCNN）或LSTM网络进行分类。

### 提出的参数卷积层

通过分析其独特的距离-速度剖面，可以区分不同的活动。有些活动可能有非常不同的特征，但有些活动可能只有轻微的差异。因此，为了准确地区分这些动作，需要在特定频带上具有更高的分辨率。然而，当应用2D STFT时，整个可观测距离-速度空间被离散化为相等的区间。通过将原始ADC雷达数据直接馈送到神经网络，神经网络可以学习滤波器内核，提取比固定预处理步骤更有意义的特征。然而，与计算机视觉或其他领域不同，该特征在原始ADC雷达数据中并不存在。因此，卷积层中通常使用的（3×3）或（5×5）的小滤波器核大小无法提取有意义的特征，这就是为什么必须考虑更大的过滤器尺寸，如（64×32）。因此，可训练滤波器权重的数量急剧增加，导致过拟合并陷入局部最小值。然而，由于已知雷达信号的信息具有时频特性，因此滤波器核可以限制为特定的滤波器类型，这些滤波器类型能够提取有意义的时频特征。

Morlet小波由两个参数唯一定义，即**高斯窗口的频率**和**标准偏差**。因此，神经网络只需要学习滤波器参数，而不是每个滤波器权重。我们将滤波器核由参数滤波器函数定义的卷积层称为参数卷积层。在每个训练步骤中，对滤波器参数进行优化，并相应地重新初始化滤波器内核。这样，通过使用雷达信号的先验知识，预处理被集成到神经网络中，可以根据训练数据进行优化。

本文提出了将第一卷积层约束为使用1D带通滤波器，该滤波器定义为具有不同截止频率的两个低通sinc滤波器的差。
