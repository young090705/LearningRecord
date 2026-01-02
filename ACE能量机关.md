## 能量机关代码识别逻辑和个人理解

### Detector功能

订阅原图像话题，进入`imageCallback`回调函数

- 进行预处理图像，灰度图，二值化，颜色通道相减，再进行二值化，然后把第一次二值化的图像和进行通道相减后二值化的图像进行合并，并进行膨胀。
- 进行筛选风扇和流水灯，筛选没有父轮廓的轮廓，接着通过**轮廓的旋转矩形长宽比（装甲板多了个轮廓面积比上旋转矩形的面积）**筛选出可能是**装甲板**和可能是**流水灯**的集合，集合里面存储的内容是对应轮廓的中心和下标索引。
- 集合存储完之后，开始匹配对应的待击打装甲板和流水灯尾部，通过**PCA**分析，这个可以了解轮廓的方向向量，然后计算可能是装甲板的中心点和可能是流水灯装甲板的中心点的向量，这个向量与PCA的向量进行计算夹角，理论上讲是比较小的，可以限制这个角度大小来达到筛选的目的。接着利用**流水灯和装甲板**的面积的比值还有**流水灯中心与装甲板中心的距离与流水灯较长边的比值**进行筛选，如果匹配出来的个数还是大于1,就筛选角度最小（**PCA**计算出来的方向向量与我们计算的方向向量的夹角）的那个。
- 匹配完之后，就是计算四个角点了，用方向向量（由流水灯指向装甲板）和外接矩形（进行了一定程度的缩小，让四个角点落位更精确）来计算。

### 串口

```c++
Buff_sub_ = this->create_subscription<buff_interface::msg::SendPoint>(
    "/target", rclcpp::SensorDataQoS(), std::bind(&SerialDriver::sendData_buff, this, std::placeholders::_1));
Auto_sub_ = this->create_subscription<auto_aim_interfaces::msg::Gimbal>(
    "/tracker/cmd_gimbal", rclcpp::SensorDataQoS(),
    std::bind(&SerialDriver::sendData_auto, this, std::placeholders::_1));
```

由于讲自瞄和能量机关**合并**在一起了，串口这里创建了两个回调函数，`sendData_auto`用来发送为了瞄准装甲板所需要的云台转动到的位姿，以及开火建议和攻击模式。`sendData_buff`也差不多，但是少了开火建议和攻击模式（但有默认的）；

函数`reset_params`代码里面有5个参数`initial*`一开始全是false，初始化过后变为了true，用于对应参数是否有初始化，比如子弹速度,大符颜色，装甲板颜色。还有前一次的大符颜色，自瞄装甲板颜色等，除了自瞄装甲板颜色是为了让代码一直运行重新设置参数设置了只要颜色相同就返回true和子弹速度和上一次子弹速度差异过大，其他都是和上次的颜色、模式不一样就返回true。

![image-20250808122949684](/home/rm_young/.config/Typora/typora-user-images/image-20250808122949684.png)

然后这里的颜色重新设置也是有点奇怪，但是能用。

相机曝光这个参数，固定设置为2200了。在接收到数据后，直接把模式设置为3了；

```c++
packet.mode = 3;
```

其他的感觉跟君瞄差不多。

### 傅里叶变换

暂时还没有去学习，好像代码已经废弃了这一部分

学习资料：
[DR_CAN数学推导](https://www.bilibili.com/video/BV1Et411R78v/?spm_id_from=333.337.search-card.all.click&vd_source=6887e7ec09a5e7ce0a39d97c63affd6f)

[纯文字讲解](https://zhuanlan.zhihu.com/p/19763358)

[关于频域的讲解](https://zhuanlan.zhihu.com/p/428783752)

### 预测

### 预测模块

通过话题` "Fans_center_"`进入了回调函数`fans_callback`,先检验一下是否有R标和待击打装甲板的存在，接着就把之前的R标的相机坐标系的位置转化到`odom`坐标系下，和把待击打装甲板相机坐标系的位姿都转换到`odom`坐标系下。

```c++
if(!first_detect)
{
    F_fan = Fan(fan,fanscenter_msg->header.stamp); //这里作为基准待击打装甲板
    first_detect = true;
}
```

接着这里对待击打装甲板和R标都设置了追踪器，每帧都要对两个追踪器进行`update`，待击打装甲板的追踪器用于保存历史数据，看不出来有什么用，还有保存相**相邻两帧的旋转矩阵**用于计算角速度，但是现在不用这个思路了，所以也没什么用了。R标的追踪器只用来取位置均值，可以使位置更稳定。由于角度解算的需要，这里专门设置了一个变量`first_detect`来保留第一帧待击打装甲板的位姿，后续需要基于这个第一帧的待击打装甲板位姿来求解角度。
**程序积累够10帧的待击打**装甲板后，就可以开始计算已经旋转的角度了，计算完角度后，大符和小符都是把要击打的点和子弹速度传输给`gv.set_param`函数,后续我们就可以使用这个对象下面的成员函数`get_t`**迭代法**去获取弹丸要击打到这个点所需要的时间。这个时间存在在变量`compensate`用于后续预测待击打装甲板的位置，同样用于预测的时间变量还有`bias_time`.

接着调用下面这个函数

```c++
void Calculator::predict(AngleData &ad,double &angle_fit,double & angle_offset,double & compensate,int mode)
```

累积好**20**个数据量之后才有数据拟合，数据量不够直接不拟合。然后数据量过少计算**凹凸性**,这里需要注意这里不是在求**切线，切线需要求导！！**。

拟合参数的话，没有直接把所有数据拟合，而是借助了下面的变量来筛选好的数据来拟合。

```c++
// inliers 为符合要求的点，outliers 为不符合要求的点
std::vector<TargetAngleInfo> inliers, outliers;
// 初始时，inliers 为全部点
inliers.assign(data.begin(), data.end());
// 迭代次数
int iterTimes{data.size() < 100 ? 50 : 20}; //数据数量过多时，选择降低拟合次数
```

首先这里**根据数据量的大小**，选择了迭代次数，数据量较大就选择20次。

接着开始拟合了，选取数据的时候，数据量小于400个就直接拿去拟合，大于400个，就为了避免拟合出来的函数陷入==局部最优解==，使用了`shuffle`来打乱前面部分数据，除了最后100个数据，拟合数据取最后的200个数据。

```c++
            std::shuffle(
                inliers.begin(), inliers.end() - 100,
                std::default_random_engine(std::chrono::system_clock::now().time_since_epoch().count()));
```

把参数和样本和凹凸性丢给最小二乘法来拟合参数，拟合完之后，计算每个拟合出来的数据和检测出来的数据的差值，并将这些值存储在`error`里，然后对数据点进行筛选

```c++
// 如果数据量较大，则对点进行筛选
        if (data.size() > 700) {
            std::sort(errors.begin(), errors.end());
            const int index{static_cast<int>(errors.size() * 0.95)};
            const double threshold{errors[index]};
            // 剔除 inliers 中不符合要求的点
            for (size_t i = 0; i < inliers.size() - 100; ++i) {
                if (std::fabs(inliers[i].angle - getAngleBig(inliers[i].time_diff, params)) > threshold) {
                    outliers.push_back(inliers[i]);
                    inliers.erase(inliers.begin() + i);
                }
            }
            // 将 outliers 中符合要求的点加进来
            for (size_t i = 0; i < outliers.size(); ++i) {
                if (std::fabs(outliers[i].angle - getAngleBig(outliers[i].time_diff, params)) < threshold) {
                    inliers.emplace(inliers.begin(), outliers[i]);
                    outliers.erase(outliers.begin() + i);
                }
            }
        }
```

主要思想就是把**差距大于error百分之95**的数据移到outliers里面，又由于**每次拟合出来的参数**略有差异，所以得检查outliers哪些数据点能用，把他添加进inliers里面，随着迭代次数的增加，数据点逐渐变好，拟合出来的参数变好，逐渐接近最优解。

**在最小二乘法里**，由于我们收集到的数据难免会有一些数据点是中等离群值，还有噪点等等，所以在拟合函数算法里也考虑了这些情况，设置了损失函数，并且还添加了**惩罚项**，限制了参数a,w,t~0~.

**惩罚项**：12.07经历了大半天的学习，对于核函数有了新的理解

> 我们将最小化误差项的二范数平房和作为目标函数，这样的做法虽然很直观，但存在一个严重的问题：如果由于误匹配的原因，某个误差项的数据是错误的，误差数值会很大

在Ceres Solver中，核函数用于定义残差块的损失计算方式。如果不设置核函数，残差块的损失默认按照残差的平方来计算。如果设置了核函数，比如Huber Loss，那么==残差块的损失将根据核函数的定义==来计算，这可以提供对异常值的鲁棒性。雅可比矩阵在残差块中用于计算残差相对于每个参数的偏导数，这对于优化算法计算梯度和更新参数至关重要。

```c++
if (points.size() < 100) {
        // 在数据量较小时，可以利用凹凸性定参数边界
        if (convexity == Convexity::CONCAVE) {
            problem.SetParameterUpperBound(ret.begin(), 2, -2.8); //对第三个参数进行约束
            problem.SetParameterLowerBound(ret.begin(), 2, -4);
        } else if (convexity == Convexity::CONVEX) {
            problem.SetParameterUpperBound(ret.begin(), 2, -1.1);
            problem.SetParameterLowerBound(ret.begin(), 2, -2.3);
        }
        omega = {10., 1., 1.};
    } else {
        // 而数据量较多后，则不再需要凹凸性辅助拟合
        omega = {60., 50., 50.}; 
    }
    //对中等离群值鲁棒
    ceres::CostFunction *costFunction1 = new CostFunctor1(ret[0], 0);
    ceres::LossFunction *lossFunction1 =
        new ceres::ScaledLoss(new ceres::HuberLoss(0.1), omega[0], ceres::TAKE_OWNERSHIP);//omege[0]为第一个参数的权重
    problem.AddResidualBlock(costFunction1, lossFunction1, ret.begin()); //目的：防止角度基准值过度偏离初始估计
    ceres::CostFunction *costFunction2 = new CostFunctor1(ret[1], 1);
    ceres::LossFunction *lossFunction2 =
        new ceres::ScaledLoss(new ceres::HuberLoss(0.1), omega[1], ceres::TAKE_OWNERSHIP);
    problem.AddResidualBlock(costFunction2, lossFunction2, ret.begin()); 
    ceres::CostFunction *costFunction3 = new CostFunctor1(ret[3], 3);
    ceres::LossFunction *lossFunction3 =
        new ceres::ScaledLoss(new ceres::HuberLoss(0.1), omega[2], ceres::TAKE_OWNERSHIP);
    problem.AddResidualBlock(costFunction3, lossFunction3, ret.begin());
//这个omega参数就是为损失值提供权重，会让对应的参数值有所变化，值大了，参数就小，值小了，参数就大。
假设我们有三个参数 a,b,c ，并且我们希望它们不要偏离初始估计 ret[0],ret[1],ret[3]  太远。我们可以定义三个残差块，每个块对应一个参数：
对于参数 a ：
总损失：
min 
a,b,c (ω[0]⋅ρ(ra​)+ω[1]⋅ρ(r b​)+ω[2]⋅ρ(r c​)) 
```

**在重写Evaluate函数**的时候，雅可比矩阵只对残差函数进行求偏导，不受核函数那些的影响

```c++
double a = parameters[0][0];
        double w = parameters[0][1];
        double t0 = parameters[0][2];
        double b = parameters[0][3];
        double c = parameters[0][4];
        double cs = cos(w * (t + t0));
        double sn = sin(w * (t + t0));
        residuals[0] = -a * cs + b * t + c - y;
        if (jacobians != NULL) {
            if (jacobians[0] != NULL) {
                jacobians[0][0] = -cs;
                jacobians[0][1] = a * (t + t0) * sn;
                jacobians[0][2] = a * w * sn;
                jacobians[0][3] = t;
                jacobians[0][4] = 1;
            }
        }
//就比如这个，residuals[0] = -a * cs + b * t + c - y;用这个残差函数对每个参数进行求偏导
```

还有==残差块并不等同于残差函数==，残差块是由残差函数，鲁棒核函数，参数指针这三个组成。

迭代完之后，把最优参数返回之后，我们就在这段数据下开始预测待击打装甲板经历这些时间后`urrent_time + compensate + bias_time`需要达到的位置,预测思路是计算出**当前时间和预测时间**的角度差，然后在世界坐标系计算出三维点的变化量，然后将这些点转到**odom坐标系**下，去计算yaw和pitch角，yaw角的计算相对简单`yaw = atan2(p.y(), p.x());`,pitch角就需要用到下面的弹道解算模型了。从规则手册得知，大能量机关转速按照三角函数呈周期性变化。速度目标函数为：

$$ \text{spd}(t)=a\cdot\sin(\omega\cdot t)+b$$

我们需要将其转化为角度和时间的函数。我们对上面的公式做积分，可得：

$$ \text{angle}(t)=\int\text{spd(t)}\, dt=-\frac{a}{\omega}\cos(\omega t+t_0)+b\cdot t+c$$

综上，我们需要拟合

$$ a,\omega,t_0,b,c$$


#### 弹道解算

![image-20250808150516843](/home/rm_young/.config/Typora/typora-user-images/image-20250808150516843.png)

以上是公式推导，利用**四阶龙格库塔法**以**水平距离**为变量分别对u和p进行迭代,然后通过下面的公式可以逐渐迭代出接近**降维后所需击打点**的坐标

```c++
x += delta_x;
y += p * delta_x;
```

每次龙格库塔法迭代结束之后，实时更新pitch的值。

### 语法的理解

[std::shared_future](https://zhuanlan.zhihu.com/p/661354583)这个类是跟async有关的，future关键字只被一个线程独占，不被其他线程使用，意思就是如果我把这个类赋值给另一个变量，如果两个变量都用get来堵塞线程，那么就会报错，这就是future，而`shared_future`则不会被独占，上面的操作并不会引发报错。

`decltype`：用于自动推导类型，与`auto`通过**初始化变量**不同，它是**以一个普通表达式作为参数**，返回该表达式的类型，不会对表达式进行求值。

```c++
int i = 4;
decltype(i) a; //推导结果为int。a的类型为int。
```





## ACE自瞄

## 发布图像

### ROS2语法

`auto qos = use_sensor_data_qos ? rmw_qos_profile_sensor_data : rmw_qos_profile_default;`：

`rmw_qos_profile_sensor_data`：这个适用于传感器数据，因为传感器数据通常连续且实时性要求高。

`rmw_qos_profile_default;`:需要可靠传输的普通话题。

### 解码神经网络输出层

```c++
            const int grid0 = grid_strides[anchor_idx].grid0;
            const int grid1 = grid_strides[anchor_idx].grid1;
            const int stride = grid_strides[anchor_idx].stride;
            const int basic_pos = anchor_idx * (9 + (NUM_COLORS) + NUM_CLASSES); //9代表有8个坐标加1个目标置信度

            // yolox/models/yolo_head.py decode logic
            //  outputs[..., :2] = (outputs[..., :2] + grids) * strides
            //  outputs[..., 2:4] = torch.exp(outputs[..., 2:4]) * strides
            float x_1 = (feat_ptr[basic_pos + 0] + grid0) * stride;
            float y_1 = (feat_ptr[basic_pos + 1] + grid1) * stride;
            float x_2 = (feat_ptr[basic_pos + 2] + grid0) * stride;
            float y_2 = (feat_ptr[basic_pos + 3] + grid1) * stride;
            float x_3 = (feat_ptr[basic_pos + 4] + grid0) * stride;
            float y_3 = (feat_ptr[basic_pos + 5] + grid1) * stride;
            float x_4 = (feat_ptr[basic_pos + 6] + grid0) * stride;
            float y_4 = (feat_ptr[basic_pos + 7] + grid1) * stride;
            
            float box_objectness = (feat_ptr[basic_pos + 8]);
            int box_color = argmax(feat_ptr + basic_pos + 9, NUM_COLORS);
            int box_class = argmax(feat_ptr + basic_pos + 9 + NUM_COLORS, NUM_CLASSES);
```

**模型为每个网眼都输出了一长串数字（预测结果）**,模型输出的是相对网格（步长*步长）起始点的坐标，，加上grid1这些就转换到网格（相当于像素）坐标，就转移到特征图坐标系了，最后乘上步长就转移到我们输入图像的图像坐标系了。

### `Detector`包

在检测包里面对装甲板的筛选:

在`armor_detector`这个cpp里进行的筛选

1. 面积，如果装甲板过多，就把最后面几个面积小的筛选掉
2. 还有把不想要的颜色筛选掉
3. 还有防抖层，若装甲板置信度小于高阈值，需要相同位置存在过装甲板才放行
4. `if (std::fabs(armor.rrect.angle) > 80.0 || std::fabs(armor.rrect.angle) < 10.0)`可以筛选旋转矩形的旋转角度
5. `std::fabs(armor_->rrect.angle) > 16.0 && std::fabs(armor_->rrect.angle) < 74.0)`还有这个
6. 如果pnp计算不成功，我们也要筛掉，其次，如果我们的Tracker会跟踪装甲板，如果装甲板找不到同样的键，但是就把这个装甲板筛掉



### 关于新规则写代码的想法

在tracker初始化的时候，必须要初始化两个能量机关

怎么鉴定，我是击打到了装甲板还是未识别到，我觉得需要加个新的识别，如果识别到新的已击打装甲板

击打到装甲板好解决，那未识别到呢？

丢识别率高不高？如果高的话，可以试试加个卡尔曼滤波

感觉把已击打装甲板识别出来很有必要性，可以判定当前是可能识别丢失了一个，还是已经击打了一个，还是两个击打完了

```c++
AngleData now_angle_data;
if (fanscenter_msg->have_fan)
{
    // new
    now_angle_data = angleCal();

    agl_raw = now_angle_data.angle; 

    now_angle_data.angle = fk.run(agl_raw);
}
```

小符改动不大，把上面的代码移植到小符里面就可以了

大符改动

在tracker里面，我打算记录两帧的fan的差数，如果上一帧是2个，下帧是1个，我选择不进行匹配，过大概4帧，才确定已经是击打到装甲板了，如果真的已经击打到了，理论上来讲，就是now_fan[1]的扇叶位姿信息，不要删除后续的第一帧，然后得想个办法把is_first_dect的初始化改了

还有一些拟合数据也得清空，不这样做，我的理解是会导致

然后下一帧开始是直接用刚开始确定的两个基准板就好了。就相当于两个装甲板同时发生了跳变

丐版状态机

由于have_fan存在，零扇叶应该不用管他

第一帧肯定是两个

如果丢识别，导致当前帧的数量比上一帧烧，那么就不把当前帧的数据拿去拟合，累计丢识别大概4帧，那么我们就认为已经击打到了一个装甲模块

然后就进入一个装甲模块的状态，然后如果遇到一种情况就是上一帧为1,当前帧为2,说明1s已经过去了，进入另一组能量机关击打

需要注意的是两个状态下，拟合的函数应该会有点不一样。然后不管状态机怎么变，第一帧的两个基准向量是不会去动他的。





### 了解到的一些开源

[佛大开源](https://github.com/Ash1104/RoboMaster2021-FOSU-AWAKENLION-OpenSource/blob/master/%E5%A4%A7%E8%83%BD%E9%87%8F%E6%9C%BA%E5%85%B3%E6%8E%A8%E5%AF%BC.pdf)

[深技大](https://github.com/IC-glb/BuffOpenSource?tab=readme-ov-file)



