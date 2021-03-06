# Non-uniform Grid Deep Learning Super Sampling

这是我们的demo.


采样后结果，分别是采样1000个点和10000个点。
![](imgs/blur_point1000.jpg)

![](imgs/blur_point10000.png)

使用cycleGAN的结果:

![](imgs/pred_cycleGAN_epoch40_0.png)

# What We Have Done
- [x] 点采样
  - [x] random
  - [x] fourier
- [x] 颜色采样
  - [x] center
  - [x] vertex 
- [x] 模型方法
  - [x] Super Resoultion CNN (SRCNN), for single img (ECCV 2014)
  - [x] Conditional Normalizing Flows (CNF), for single img (ICLR 2020)
  - [x] GAN/cycleGAN (目前加了模型，但是还没有想好怎么做，直接img-to-img translation 似乎没有必要)
  - [x] Masked Auto-Encoder (MAE)
  - [x] SwinIR
  - [ ] CUT
  - [x] MAE
- [x] 便于使用的脚本 
- [x] 训练代码（train.py)
- [x] 推理代码（predict.py）
- [x] 测试代码（eval.py）

# What We Need To Do
- [ ] 比较不同采样
- [ ] 比较不同算法的训练阶段
- [ ] 比较不同算法的测试阶段
- [ ] 可视化

# 使用方法
首先下载数据集，在release里面。然后可以通过`--data_root`指定你放数据的地方。

然后训练模型：
```
python train.py --alg SRCNN
```
更多的参数设置请查看`arguments.py`。在`scripts/`目录下面有一些实例脚本，可以先尝试。



# 记录 & 进度
# 介绍
项目简介：目前超采样算法都是从图像到图像的输入和输出，图像可以看作一种致密的均匀网格（dense uniform grid）。在此，我们试图更改输入为更一般化的非均匀系数样本的网格，尝试基于深度学习的超采样图像重建。(右图只是示意，图中网格过于稀疏，实际难以保证质量。）

任务：可以使用Voronoi-Delaunay随机撒点三角化算法预先在图像上覆盖生成网格，逐顶点采样图像样本。参考图像超采样深度学习相关文献，设计类似输入为网格样本的超采样算法， 重建原始图像。


要求：设计相关神经网络算法，可控制调整网格密度和点分布，生成图像重建结果，比较质量；尝试尽可能降低网格密度并获得高质量还原图像；尝试减少神经网络深度（可附加其他预处理），提升性能。

# 方法

# 记录：
Model: SRCNN, VDSR
Dataset: mini-Set5, celebA
Result:
- Set5: Loss在~200 iter后loss趋于稳定，不断抖动，不再降低，模型饱和。Reconstruct出来的Image基本上就是Delaunay采样生成的图片加上高斯模糊得到，几乎没有细节和形状上的纠正
- ![loss-set5](imgs/loss-set5.png)
- celebA: (相比Set5，Domain更加集中，数据量更大) Loss在~5000 iter后趋于稳定，但仍有一定的下降趋势。Reconstruction效果不错，在高斯模糊的基础上还有一定的细节补充和形状纠正
- ![loss-celebA](imgs/loss-set5.png)
可能的提升点:
  1. 模型容量的问题 (为了让我的2G显存装得下，极大地削减了VDSR的表达能力) TODO: ZYJ在服务器上训VDSR(未削减版) + celebA
  2. CNN本身不鼓励清晰的细节 TODO: Canny Edge处理，加入针对Edge的loss; 学习TecoGAN等方法
   
## 算法流水线
training:
```
Input: img_origin
1. Sample on img_origin and get img_sampled 
2. Input img_sampled into SuperResolution Network and get img_reconstructed.
3. Compute loss and backward.
```

testing：
```
Input: img_origin
1. Sample on img_origin and get img_sampled 
2. Input img_sampled into SuperResolution Network and get img_reconstructed.
```

## 数据集
Set14, [link](https://deepai.org/dataset/set14-super-resolution)
celebA, [link](https://jbox.sjtu.edu.cn/l/U1y10y)
## 采样
原图：
![](imgs/img_origin.jpeg)
首先使用Voronoi-Delaunay进行三角采样，获得如下图所示的结果：
![](imgs/delaunay.jpg)

考虑用多种方法采样：
1. 先随机采点，再三角化，再取三角中心的颜色作为该三角的值（已实现）
2. 先进行傅里叶变换，在高频处（细节）多采样，在低频处少采样。再三角化，再取三角中心的颜色作为该三角的值。（待实现）

### 基于傅里叶变换的图像三角化
超参数`frequency_domain_range`来判断高频和低频的分界线。超参数`frequency_sample_prob`来给所有高频点加一个概率来采样，不是所有都采。

算法伪代码：
1. 傅里叶变换，获得频域图。
2. 将低频部分根据frequency_domain_range来mask掉。
3. 逆变换，获得高频在时域的图。
4. 对高频pixel采样，以frequency_sample_prob的概率采样。
5. 再进行全局随机采样。
6. 用4、5两步获得的采样点进行三角化。

下图为频域可视化。
![](imgs/fourier_domain.png)

下图为傅里叶高频采样结果。
![](imgs/fourier_triangulation.png)





## Backbone/Baseline
1.SRCNN, ECCV 2014, [link](https://github.com/yjn870/SRCNN-pytorch)
下面两幅图分别是SRCNN和SRCNN2在一个epoch后进行预测的结果。说明了只增加网络宽度和深度后，模型拟合会更慢，会导致算法效果变差。但是在很多个epoch后是否还会这样不清楚。

![](imgs/pred_SRCNN1_epoch0.png)

![](imgs/predict_SRCNN2_epoch0.png)

# swinIR 使用说明
```
# 001 Classical Image Super-Resolution (middle size)
# Note that --training_patch_size is just used to differentiate two different settings in Table 2 of the paper. Images are NOT tested patch by patch.
# (setting1: when model is trained on DIV2K and with training_patch_size=48)
python main_test_swinir.py --task classical_sr --scale 2 --training_patch_size 48 --model_path model_zoo/swinir/001_classicalSR_DIV2K_s48w8_SwinIR-M_x2.pth --folder_lq testsets/Set5/LR_bicubic/X2 --folder_gt testsets/Set5/HR
python main_test_swinir.py --task classical_sr --scale 3 --training_patch_size 48 --model_path model_zoo/swinir/001_classicalSR_DIV2K_s48w8_SwinIR-M_x3.pth --folder_lq testsets/Set5/LR_bicubic/X3 --folder_gt testsets/Set5/HR
python main_test_swinir.py --task classical_sr --scale 4 --training_patch_size 48 --model_path model_zoo/swinir/001_classicalSR_DIV2K_s48w8_SwinIR-M_x4.pth --folder_lq testsets/Set5/LR_bicubic/X4 --folder_gt testsets/Set5/HR
python main_test_swinir.py --task classical_sr --scale 8 --training_patch_size 48 --model_path model_zoo/swinir/001_classicalSR_DIV2K_s48w8_SwinIR-M_x8.pth --folder_lq testsets/Set5/LR_bicubic/X8 --folder_gt testsets/Set5/HR

# (setting2: when model is trained on DIV2K+Flickr2K and with training_patch_size=64)
python main_test_swinir.py --task classical_sr --scale 2 --training_patch_size 64 --model_path model_zoo/swinir/001_classicalSR_DF2K_s64w8_SwinIR-M_x2.pth --folder_lq testsets/Set5/LR_bicubic/X2 --folder_gt testsets/Set5/HR
python main_test_swinir.py --task classical_sr --scale 3 --training_patch_size 64 --model_path model_zoo/swinir/001_classicalSR_DF2K_s64w8_SwinIR-M_x3.pth --folder_lq testsets/Set5/LR_bicubic/X3 --folder_gt testsets/Set5/HR
python main_test_swinir.py --task classical_sr --scale 4 --training_patch_size 64 --model_path model_zoo/swinir/001_classicalSR_DF2K_s64w8_SwinIR-M_x4.pth --folder_lq testsets/Set5/LR_bicubic/X4 --folder_gt testsets/Set5/HR
python main_test_swinir.py --task classical_sr --scale 8 --training_patch_size 64 --model_path model_zoo/swinir/001_classicalSR_DF2K_s64w8_SwinIR-M_x8.pth --folder_lq testsets/Set5/LR_bicubic/X8 --folder_gt testsets/Set5/HR


# 002 Lightweight Image Super-Resolution (small size)
python main_test_swinir.py --task lightweight_sr --scale 2 --model_path model_zoo/swinir/002_lightweightSR_DIV2K_s64w8_SwinIR-S_x2.pth --folder_lq testsets/Set5/LR_bicubic/X2 --folder_gt testsets/Set5/HR
python main_test_swinir.py --task lightweight_sr --scale 3 --model_path model_zoo/swinir/002_lightweightSR_DIV2K_s64w8_SwinIR-S_x3.pth --folder_lq testsets/Set5/LR_bicubic/X3 --folder_gt testsets/Set5/HR
python main_test_swinir.py --task lightweight_sr --scale 4 --model_path model_zoo/swinir/002_lightweightSR_DIV2K_s64w8_SwinIR-S_x4.pth --folder_lq testsets/Set5/LR_bicubic/X4 --folder_gt testsets/Set5/HR


# 003 Real-World Image Super-Resolution (use --tile 400 if you run out-of-memory)
# (middle size)
python main_test_swinir.py --task real_sr --scale 4 --model_path model_zoo/swinir/003_realSR_BSRGAN_DFO_s64w8_SwinIR-M_x4_GAN.pth --folder_lq testsets/RealSRSet+5images --tile

# (larger size + trained on more datasets)
python main_test_swinir.py --task real_sr --scale 4 --large_model --model_path model_zoo/swinir/003_realSR_BSRGAN_DFOWMFC_s64w8_SwinIR-L_x4_GAN.pth --folder_lq testsets/RealSRSet+5images


# 004 Grayscale Image Deoising (middle size)
python main_test_swinir.py --task gray_dn --noise 15 --model_path model_zoo/swinir/004_grayDN_DFWB_s128w8_SwinIR-M_noise15.pth --folder_gt testsets/Set12
python main_test_swinir.py --task gray_dn --noise 25 --model_path model_zoo/swinir/004_grayDN_DFWB_s128w8_SwinIR-M_noise25.pth --folder_gt testsets/Set12
python main_test_swinir.py --task gray_dn --noise 50 --model_path model_zoo/swinir/004_grayDN_DFWB_s128w8_SwinIR-M_noise50.pth --folder_gt testsets/Set12


# 005 Color Image Deoising (middle size)
python main_test_swinir.py --task color_dn --noise 15 --model_path model_zoo/swinir/005_colorDN_DFWB_s128w8_SwinIR-M_noise15.pth --folder_gt testsets/McMaster
python main_test_swinir.py --task color_dn --noise 25 --model_path model_zoo/swinir/005_colorDN_DFWB_s128w8_SwinIR-M_noise25.pth --folder_gt testsets/McMaster
python main_test_swinir.py --task color_dn --noise 50 --model_path model_zoo/swinir/005_colorDN_DFWB_s128w8_SwinIR-M_noise50.pth --folder_gt testsets/McMaster


# 006 JPEG Compression Artifact Reduction (middle size, using window_size=7 because JPEG encoding uses 8x8 blocks)
python main_test_swinir.py --task jpeg_car --jpeg 10 --model_path model_zoo/swinir/006_CAR_DFWB_s126w7_SwinIR-M_jpeg10.pth --folder_gt testsets/classic5
python main_test_swinir.py --task jpeg_car --jpeg 20 --model_path model_zoo/swinir/006_CAR_DFWB_s126w7_SwinIR-M_jpeg20.pth --folder_gt testsets/classic5
python main_test_swinir.py --task jpeg_car --jpeg 30 --model_path model_zoo/swinir/006_CAR_DFWB_s126w7_SwinIR-M_jpeg30.pth --folder_gt testsets/classic5
python main_test_swinir.py --task jpeg_car --jpeg 40 --model_path model_zoo/swinir/006_CAR_DFWB_s126w7_SwinIR-M_jpeg40.pth --folder_gt testsets/classic5

```

# 参考
1. Neural supersampling for real-time rendering, 2020, [link](https://research.fb.com/wp-content/uploads/2020/06/Neural-Supersampling-for-Real-time-Rendering.pdf)
2. A review, [link](https://cseweb.ucsd.edu/~ravir/tianchengsiga.pdf)
3. Learning Likelihoods with Conditional Normalizing Flows, 2019, [link](http://arxiv.org/abs/1912.00042)