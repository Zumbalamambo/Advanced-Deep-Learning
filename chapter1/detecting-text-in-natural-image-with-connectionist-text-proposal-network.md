# Detecting Text in Natural Image with Connectionist Text Proposal Network

CTPN和Faster R-CNN \[1\] 出自同系，根据文本区域的特点做了专门的调整，一个重要的地方是RNN的引人，笔者在实现CTPN的时候也是直接在Faster-RCNN基础上改的。理解了Faster R-CNN之后，CTPN理解的难度也不大，下面开始分析这篇论文。

作者提供的CTPN demo源码：[https://github.com/tianzhi0549/CTPN](https://github.com/tianzhi0549/CTPN)

基于TensorFlow的CTPN开源代码：[https://github.com/eragonruan/text-detection-ctpn](https://github.com/eragonruan/text-detection-ctpn)

## 简介

传统的文字检测是一个自底向上的过程，总体上的处理方法分成连通域和滑动窗口两个方向。基于连通域的方法是先通过快速滤波器得到文字的像素点，再根据图像的低维特征（颜色，纹理等）贪心的构成线或者候选字符。基于滑动窗口的方法是通过在图像上通过多尺度的滑窗，根据图像的人工设计的特征（HOG，SIFT等），使用预先训练好的分类器判断是不是文本区域。但是这两个途径在健壮性和可信性上做的并不好，而且滑窗是一个非常耗时的过程。

卷积网络在2012年的图像分类上取得了巨大的成功，在2015年Faster R-CNN在物体检测上提供了非常好的算法框架。所以用深度学习的思想解决场景文字检测自然而然的成为研究热点。对比发现，场景文字检测和物体检测存在两个显著的不同之处

1. 场景文字检测有明显的边界，例如Wolf 准则 \[2\]，而物体检测的边界要求较松，一般IoU为0.7便可以判断为检测正确；
2. 场景文字检测有明显的序列特征，而物体检测没有这些特征；
3. 和物体检测相比，场景文字检测含有更多的小尺寸的物体。

针对以上特点，CTPN做了如下优化：

1. 在CTPN中使用更符合场景文字检测特点的锚点；
2. 针对锚点的特征使用新的损失函数；
3. RNN（双向LSTM）的引入用于处理场景文字检测中存在的序列特征；
4. Side-refinement的引入进一步优化文字区域。

## 算法详解

### 1. 算法流程

CTPN的流程和Faster R-CNN的RPN网络类似，首先使用VGG-16提取特征，在conv5进行3\*3，步长为1的滑窗。设conv5的尺寸是W\*H，这样在conv5的同一行，我们可以得到W个256维的特征向量。将同一行的特征向量输入一个双向LSTM中，在双向LSTM后接一个512维的全连接后便是CTPN的3个任务的多任务损失，结构如图1。任务1的输出是2\*k，用于预测候选区域的起始y坐标和高度h；任务2是用来对前景和背景两个任务的分类评分；任务3是k个输出的side-refinement的偏移（offset）预测。在CTPN中，任务1和任务2是完全并行的任务，而任务3要用到任务1，2的结果，所以理论上任务3和其他两个是串行的任务关系，但三者放在同一个损失和函数中共同训练，也就是我们在Faster R-CNN中介绍的近似联合训练。

![](/assets/CTPN_1.png)

###### 图1：CTPN的结构

### 2. 数据准备

和RPN的要求一样，CTPN输入图像的尺寸无硬性要求，只是为了保证特征提取的有效性，在保证图片比例不变的情况下，CTPN将输入图片的resize到600，且保证长边不大于1000。

```
def resize_im(im, scale, max_scale=None):
    f=float(scale)/min(im.shape[0], im.shape[1])
    if max_scale!=None and f*max(im.shape[0], im.shape[1])>max_scale:
        f=float(max_scale)/max(im.shape[0], im.shape[1])
    return cv2.resize(im, (0, 0), fx=f, fy=f), f
```

### 3. CTPN的锚点机制

作者通过分析RPN在场景文字检测的实验发现RPN的效果并不是特别理想，尤其是在定位文本区域的横坐标上存在很大的误差。因为在一串文本中，在不考虑语义的情况下，每个字符都是一个独立的个体，这使得文字区域的边界是很难确定的。显然，文本区域检测和物体检测最大的区别是文本区域是一个序列，如图2。如何我们能根据文本的序列特征捕捉到文本区域的边界信息，应该能够对本文区域的边界识别能够很好的预测。

![](/assets/CTPN_2.png)

###### 图2：文本区域的序列特征。

而在目前的神经网络中，RNN在处理序列数据上占有垄断性的优势地位。在RNN的训练过程中，数据是以时间片为单位输入到模型中的。所以，如何将文本区域变成可以序列化输入的顺序成为了CTPN一个重要的要求。如图2所展示的，每一个蓝色矩形是一个锚点，那么一个文本区域便是由一系列宽度固定，紧密相连的锚点构成。所以，CTPN有如下的锚点设计机制：

由于CTPN是使用的VGG-16进行特征提取，VGG-16经过4次max pooling的降采样，得到的\_feature\_stride=16，\_feature\_stride体现在在conv5上步长为1的滑窗相当于在输入图像上步长为16的滑窗。所以，根据VGG-16的网络结构，CTPN的锚点宽度w必须为16[^1]。对于一个输入序列中的所有锚点，如果我们能够判断出锚点的正负，把这一排正锚点连在一起便构成了文本区域，所以，锚点的起始坐标x也不用预测。所以在CTPN中，网络只需要预测锚点的起始y坐标以及锚点的高度h即可。

在RPN网络中，一个特征向量对应的多个尺寸和比例的锚点，同样的，CTPN也对同一个特征向量设计了10个锚点。在CTPN中，锚点的高度依次是\[11,16,23,33,48,68,97,139,198,283\]，即高度每次除以0.7。根据笔者的经验，根据你的CTPN的应用场景，比如要做一些文本区域较小的检测，这时候你可能需要设计更小的锚点。

###  4. CTPN中的RNN

我们多次强调场景文字检测一个重要的不同是文本区域具有序列特征。在上面一段，我们已经可以根据锚点构造序列化的数据。通过在W\*H的conv5层进行步长为的滑窗，每一次横向滑动得到的便是W个长度为256的特征向量$$X_t$$。设RNN的隐层节点是$$H_t$$，则RNN模型可以表示为


$$
H_t = \varphi(H_t-1, X_t), t = 1,2,...,W
$$


其中$$\varphi$$是非线性的激活函数。隐层节点的数量是128，RNN使用的是双向的LSTM，因此通过双向LSTM得到的特征向量是256维的。

```
layer {
  name: "lstm"
  type: "Lstm"
  bottom: "lstm_input"
  top: "lstm"
  lstm_param {
      num_output: 128
      weight_filler {
        type: "gaussian"
        std: 0.01
      }
      bias_filler {
        type: "constant"
      }
      clipping_threshold: 1
    }
}

...

layer {
  name: "rlstm"
  type: "Lstm"
  bottom: "rlstm_input"
  top: "rlstm-output"
  lstm_param {
    num_output: 128
   }
}

...

# merge lstm and rlstm
layer {
  name: "merge_lstm_rlstm"
  type: "Concat"
  bottom: "lstm"
  bottom: "rlstm"
  top: "merge_lstm_rlstm"
  concat_param {
    axis: 2
  }
}
```

### 5. side-refinement

将side-refinement从CTPN独立出来似乎更好理解，side-refinement对于CTPN的其它部分相当于Faster R-CNN中的RPN对于Fast R-CNN。不同之处在于side-refinement根据CTPN预测的锚点信息得到文本行，从中选择边界锚点进行位移优化，而Fast R-CNN优化的是根据RPN的输出通过NMS得到的候选区域。所以，side-refinement一个重要的步骤是如何根据锚点信息构造文本行。

#### 5.1 文本行的构造

通过CTPN可以得到候选区域的的得分，如果判定为文本区域的得分大于阈值$$\theta_1$$，则该区域用来构造文本行。文本行是由一系列大于0.7的候选区域的**邻居对**构成的，如果区域$$B_j$$是区域$$B_i$$的邻居对，需要满足如下条件：

1. $$B_j$$是距离$$B_i$$最近的正文本区域；
2. $$B_j$$和$$B_i$$的距离小于$$\theta_2$$个像素值；
3. $$B_i$$和$$B_j$$的竖直方向的重合率大于$$\theta_3$$。

在源码提供的配置文件中，$$\theta_1=0.7, \theta_2=50, \theta_3=0.7$$。文本区域是由一系列邻居对构成的。

```py
def get_successions(self, index):
    box=self.text_proposals[index]
    results=[]
    for left in range(int(box[0])+1, min(int(box[0])+cfg.MAX_HORIZONTAL_GAP+1, self.im_size[1])):
        adj_box_indices=self.boxes_table[left]
        for adj_box_index in adj_box_indices:
            if self.meet_v_iou(adj_box_index, index):
                results.append(adj_box_index)
        if len(results)!=0:
            return results
    return results
```

#### 5.2 side-refinement的损失函数

构造完文本行后，我们根据文本行的左端和右端两个锚点的特征向量计算文本行的相对位移（o）：


$$
o = (x_side - c_x^a)/w_a
$$



$$
o^* = (x^*_{side} - c_x^a)/w_a
$$


其中$$x_{side}$$便是由CTPN构造的文本行的左侧和右侧两个锚点的x坐标，即文本行的起始坐标和结尾坐标。所以$$x^*_{side}$$，便是对应的ground truth的坐标，$$c_x^a$$是锚点的中心点坐标，$$w_a$$是锚点的宽度，所以是16。设k是锚点的下标，则side-refinement使用的损失函数是smooth L1函数。

### 6. CTPN的损失函数

CTPN使用的是Faster R-CNN的近似联合训练，即将分类，预测，side-refinement作为一个多任务的模型，这些任务的损失函数共同决定模型的调整方向。

#### 6.1 文本区域得分损失$$L_s^{cl}$$

分类损失函数$$L_s^{cl}(s_i,s_i^*)$$是softmax损失函数，其中$$s_i^*=\{0,1\}$$是ground truth，即如果锚点为正锚点（前景），$$s_i^*=1$$，否则$$s_i^*=0$$。在tf的源码中\(lib/rpn\_msr/anchor\_target\_layer\_tf.py\)，一个锚点是正锚点的条件如下:

1. 每个位置上的9个anchor中overlap最大的认为是前景；
2. overlap大于0.7的认为是前景3

如果overlap小于0.3，则被判定为背景。在源码中参数RPN\_CLOBBER\_POSITIVES为true则表示如果一个样本的overlap小于0.3，且同时满足正样本的条件1，则该样本被判定为负样本。

s\_i是预测锚点i为前景的概率。

#### 6.2 纵坐标损失$$L_v^{re}$$

纵坐标的损失函数$$L_v^{re}(v_j,v_j^*)$$_使用的是smooth l1 损失函数，_$$v_j$$和$$x = y$$使用的是相对位移，表示如下


$$
v_c = (c_y - c_y^a)/h^a, v_h = log(h/h^a)
$$



$$
v_c^* = (c_y^* - c_y^a)/h^a, v_h^* = log(h^*/h^a)
$$


$$v=\{v_c, v_h\}$$_, _$$v^*=\{v^*_c, v^*_h\}$$分别是预测的坐标和ground truth的坐标

综上，CTPN的损失函数表示为


$$
L(s_i,v_j,o_k) = \frac{1}{N_s}\sum_i L_s^{cl}(s_i,s_i^*) + \frac{\lambda_1}{N_v}\sum_j L_v^{re}(v_j,v_j^*) + \frac{\lambda_2}{N_o}L^{re}_o(o_k,o_k^*)
$$


$$\lambda_1$$， $$\lambda_2$$是各任务的权重系数，$$N_s$$，$$N_v$$，$$N_o$$是归一化参数，表示对应任务的样本数量。

### 7. CTPN的训练细节

每个minibatch同样采用“Image-centric”的采样方法，每次随机取一张图片，然后在这张图片中采样128个样本，并尽量保证正负样本数量的均衡。卷积层使用的是Image-Net上无监督训练得到的结果，权值初始化使用的是均值为0，标准差为0.01的高斯分布。SGD的参数中，遗忘因子是0.9，权值衰减系数是0.0006。前16k次迭代的学习率是0.001，后4k次迭代的学习率是0.0001。

## 参考文献

\[1\] Ren, S., He, K., Girshick, R., Sun, J.: Faster R-CNN: Towards real-time object detection with region proposal networks \(2015\), in Neural Information Processing Systems \(NIPS\)

\[2\] Wolf, C., Jolion, J.: Object count / area graphs for the evaluation of object detection and segmentation algorithms. International Journal of Document Analysis 8, 280–296 \(2006\)

