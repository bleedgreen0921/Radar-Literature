# BEV感知算法

## 第一章 绪论

### 1、BEV感知算法的概念

**BEV**，即**B**ird's-**E**ye-**V**iew，鸟瞰图（俯视图）

优点：

1. **尺度变化小**：现实世界中观察物体存在“近大远小”的效果，但是在BEV视角中，物体距离的远近对于其在BEV中的尺度影响很小，即尺度变化小。
2. **遮挡小**：现实世界中观察物体时存在”遮挡“现象，即前面的物体被后面的物体挡住，但是在BEV中可以清晰地将其区分开来。

BEV感知是一个建立在众多子任务上的一个概念。包括分类、检测、分割、跟踪、预测等等。

BEV感知输入包括毫米波雷达、激光雷达点云、相机图像等等；根据输入的不同，BEV感知算法有进一步的划分。![image-20250520104247246](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250520104247246.png)

### 2、BEV感知算法数据形式

图像数据：**BEVFormer**![image-20250520110404477](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250520110404477.png)

点云数据：点云的基本组成单元是点，点组成的集合叫点云。点云具有**稀疏性**、无序性等特点，是一种**3D表征**。

图像数据和点云数据融合：**BEVFusion**![image-20250520110448497](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250520110448497.png)

点云数据处理：**聚类算法**

1. 基于点的方法：聚合一个球形空间内的点
2. 基于体素的方法：聚合一定立体空间内的点

### 3、BEV开源数据集介绍

**KITTI数据集**：包含图像数据、点云数据、转换矩阵、目标标注等

转换矩阵：
$$
y=P^{(i)}_{rect}R^{(0)}_{rect}T^{cam}_{velo}x_{velo}
$$
`x_velo`:激光雷达坐标系中的3D点（齐次坐标形式，需补充为*x*,*y*,*z*,1）

`y`:投影后的图像坐标（归一化后为*u*,*v*）

T:**外参矩阵（Tr_velo_to_cam）**

- **作用**：将点云从Velodyne坐标系转换到参考相机（0号灰度相机）坐标系。
- **结构**：4×4齐次矩阵，包含旋转分量R（3×3）和平移分量T（3×1）：$$Tr*_velo_*to*_cam = [ R | T ]                [ 0 | 1 ]$$

R:**矫正矩阵（R0_rect）**

- **作用**：修正相机畸变，使所有相机平面共面（Rectified坐标系）。
- **结构**：3×3旋转矩阵扩展为4×4齐次矩阵，从`calib_cam_to_cam.txt`的`R_rect_00`字段获取.

P:**内参矩阵（P_rect_i）**

- **作用**：将矫正后的3D点投影到第i个相机的2D图像平面。
- **结构**：3×4矩阵，包含焦距（fx, fy）、主点（cx, cy）等参数，从`calib_cam_to_cam.txt`的`P_rect_0i`字段读取.

由于数据采集涉及到多传感器，所以要特别注意多传感器之间的位置关系；实现从2D图像到3D点云之间的双向转换 。

**nuScenes数据集**

nuScenes数据集包含以下四个主要文件夹（以完整版为例）：

1. **`maps/`**

   - 存储地图数据，包含四张二值化语义地图（对应波士顿和新加坡的四个采集区域），标注道路、车道线、交叉口等静态元素。
   - 文件格式为PNG图像，道路区域像素值为255，其他区域为0。

2. **`samples/`**

   - 关键帧数据

     ：以2Hz频率采样的标注数据，包含：

     - 6个相机的JPEG图像（分辨率1600×900）
     - 1个激光雷达的PCD点云（32线，每秒140万点）
     - 5个毫米波雷达的扫描数据（FMCW调频，探测距离250m）

   - 每个传感器对应子文件夹（如`CAM_FRONT`、`LIDAR_TOP`），按时间戳命名文件。

3. **`sweeps/`**

   - 完整时序数据

     ：未标注的连续传感器数据，采样频率为：

     - 相机：12Hz
     - 激光雷达：20Hz
     - 毫米波雷达：13Hz

   - 用于跟踪任务或视频合成，文件结构与`samples/`一致。

4. **`v1.0-version/`**

   - 存储13个JSON文件，通过关系数据库管理元数据：
     - `sample.json`：关键帧索引（每个scene约40帧）
     - `ego_pose.json`：车辆位姿（含时间戳、平移、旋转四元数）
     - `calibrated_sensor.json`：传感器标定参数（外参+相机内参）
     - 其他：标注可见性（`visibility.json`）、目标类别（`category.json`）等

### 4、BEV感知算法分类

**BEV LiDAR**![image-20250520115355014](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250520115355014.png)

- **Pre-BEV Feature Extraction**:先提取特征，再生成BEV表征，代表算法PV-RCNN等
- **Post-BEV Feature Extraction**:先转换到BEV视图，再提取特征，代表算法PointPillar等

**BEV Camera**![image-20250520163523473](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250520163523473.png)

BEV摄像头的一般流程（仅摄像头感知）。它分为三个部分，包括**二维特征提取器**、**视图变换**和**三维解码器**。

在视图变换中，有三种方法可以对3D信息进行编码——第一种是从2D特征中预测深度信息；第二种方法是从3D空间中采样2D特征；最后一种是通过纯网络隐式地对3D-2D投影进行建模。

**BEV Fusion**![image-20250520163926382](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250520163926382.png)

BEVFusion从多模态输入中提取特征，并使用视图变换将其高效地转换为共享鸟瞰图（BEV）空间。它将统一的BEV功能与全卷积BEV编码器融合在一起，并支持具有特定任务头的不同任务。

### 5、BEV感知算法的优劣

BEV融合算法的两种典型流水线设计。主要区别在于2D到3D的转换和融合模块。

通用3D检测结构：图像数据和点云数据分别使用不同的算法进行处理，得到的结果首先被转换到3D空间，然后利用先验或手工规则进行融合。

![image-20250520164203439](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250520164203439.png)



BEV感知算法结构：首先将PV特征转换为BEV，然后融合特征以获得最终预测，从而保持最原始的信息并避免手工设计。

![image-20250520164223048](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250520164223048.png)



## 第二章 BEV感知算法基础模块

2D特征之图像处理：使用VGG,ResNet等深度神经网络进行特征提取

3D特征之点处理方案：详见论文**《PointNet++: Deep Hierarchical Feature Learning on Point Sets in a Metric Space》**

![image-20250520170303389](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250520170303389.png)

3D特征之体素处理方案：

特征学习网络以原始点云为输入，将空间划分为体素，并将每个体素内的点转换为表征形状信息的向量表示。空间表示为一个稀疏的 4 维张量。卷积中间层处理 4 维张量以聚合空间上下文。最后，一个区域提议网络生成 3D 检测。

详见论文**《VoxelNet: End-to-End Learning for Point Cloud Based 3D Object Detection》**

![image-20250520171707048](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250520171707048.png)

### 2.1 从2D到3D转换模块

​	核心动机：目前在自动驾驶领域，比较火的一类研究方向是基于采集到的**环视图像信息**，去构建BEV视角下的特征完成自动驾驶感知的相关任务。所以如何准确的完成从相机视角向BEV视角下的转变就变得由为重要。

由2D图像直接生成BEV视图很困难，因此通常先将2D图像特征转换为3D空间特征，再对3D空间特征进行投影，得到BEV视图。

![image-20250520173126310](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250520173126310.png)

- 点*P*($$X_c,Y_c,Zc$$)表示相机坐标系中的3D世界坐标，*p*(*x*,*y*)为投影后的2D图像坐标。
- 右侧矩阵方程展示了标准针孔相机模型的投影关系。其中相机内参矩阵由相机参数决定，一般为固定值。也就是说，已知3D世界坐标的情况下，其在二维图像上的投影坐标是确定且唯一的。反之，当我们仅知道2D平面上的投影坐标时，无法得到确定的3D世界坐标，只能得到一条射线，射线上的点均有可能是真实的3D世界坐标；若添加限制条件已知深度坐标，则可形成一一对应关系。

**从2D到3D的转换模块：LSS（Lift, Splat, Shoot）**：

详见论文**《Lift, Splat, Shoot: Encoding Images from Arbitrary Camera Rigs by Implicitly Unprojecting to 3D》**

![image-20250520174606992](C:\Users\adminqiu\AppData\Roaming\Typora\typora-user-images\image-20250520174606992.png)

### 2.2 从3D到2D转换模块

《DETR3D: 3D Object Detection
from Multi-view Images via 3D-to-2D Queries》

《FUTR3D: A Unified Sensor Fusion Framework for 3D Detection》

《PETR: Position Embedding Transformation for Multi-View 3D Object Detection》

### 2.3 BEV感知中的Transformer

 注意力机制

- **空间注意力之STN**

  在**空间**中捕获重要区域特征

  核心：局部网络、参数化网络采样（网络生成器）和差分图像采样

- **通道注意力之SENet**

  在**通道**中捕获重要区域特征

  核心：全局池化、权重预测，为每一个通道给不同的权重

- **混合注意力之CBAM**

  同时经过了通道和空间两个注意力机智的处理，自适应细化特征。

- **self-Attention**

  计算给定的input sequence各个位置之间的彼此的影响力大小

  核心：**查询向量Query**，**键值向量Key**，**值向量Value**，**相似度计算**。