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

- 初始化权重为DFT矩阵：` w(k,m)=exp(−j2π/N*​km)`
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
