```python
emulate_tty=True,#启用这个选项后，ROS 2会为节点输出添加一些终端特性，例如颜色高亮和实时输出（避免缓冲）。这通常使得输出更易于阅读，因为像printf这样的输出会立即刷新而不是被缓冲。
#一个Node的参数
```

`from launch.actions import TimerAction, Shutdown`,引入这个`SHutdown`这个模块可以使用`on_exit=Shutdown(),`，当节点推出时，关闭启动系统。

```python
    delay_tracker_node = TimerAction(
        period=2.5,
        actions=[tracker_node],
    )
    delay_cam_node = TimerAction(
        period=0.5,
        actions=[cam_detector],
    )
    if launch_params['using_record'] == True:
        delay_recoder_node = TimerAction(
            period=1.5,
            actions=[rm_auto_record_node],
        )
        return LaunchDescription([
            robot_state_publisher,
            delay_cam_node,
            serial_driver_node,
            delay_tracker_node,
            delay_recoder_node,
        ])
    else : 
        return LaunchDescription([
            robot_state_publisher,
            delay_cam_node,
            serial_driver_node,
            delay_tracker_node,
        ])
#这里就是TimeAction的使用是隔几秒再开始运行这个节点
```

### 坐标转换接口的代码

#### 发布静态坐标相对关系

由于是静态，这个**只需要发布一次**就好了，buffer会订阅这个话题下（/tf_static）的坐标关系的，而且我看kimi的代码，是可以写一个发布方，然后发布多个坐标相对关系的。

#### 发布动态坐标相对关系

由于是动态的，一旦相对关系发生变化，我们就**需要重新发布一个新的相对关系**。

代码如下：

```c++
pose_sub_ = this->create_subscription<turtlesim::msg::Pose>("/turtle1/pose",10,
    std::bind(&TFDynaBroadcaster::do_pose,this,std::placeholders::_1));
```

#### 发布点与坐标的相对关系

这个本质就是**话题通信的发布方**，发布的消息接口是`geometry_msgs::msg::PointStamped`，就这么简单。

#### 监听坐标关系

代码里面的注释已经很详细了

```c++
TFListener():Node("TFListener_node_cpp")
  {
    //3-1.创建一个缓存对象，融合多个相对坐标相对关系成为一颗坐标树
    buffer_ = std::make_unique<tf2_ros::Buffer>(this->get_clock());
    //3-2.创建一个监听器，绑定缓存对象，会将所有广播的数据写入缓存
    listener_=std::make_shared<tf2_ros::TransformListener>(*buffer_,this);
    //3-3.编写一个定时器，循环实现转换
    timer_ = this->create_wall_timer(1s,std::bind(&TFListener::on_timer,this));
  }
  private:
  void on_timer()
  {
    //const std::string &target_frame, //重建坐标帧后的父级标，即第一个参数
    //const std::string &source_frame, //子级坐标，第二个参数
    //const tf2::TimePoint &time//转换最新时刻的时间坐标
    //try 和 catch 是用于异常处理的机制
    try
    {
       //查寻坐标转换
       auto ts = buffer_->lookupTransform("camera","laser",tf2::TimePointZero);
       RCLCPP_INFO(this->get_logger(),"------转换完成的坐标帧信息--------");
       RCLCPP_INFO(this->get_logger(),"新坐标帧：副坐标系：%s，子坐标系：%s，偏移量（%.2f,%.2f,%.2f）",
       ts.header.frame_id.c_str(),ts.child_frame_id.c_str(),
       ts.transform.translation.x,
       ts.transform.translation.y,
       ts.transform.translation.z
       );
    }//catch 块用于捕获并处理 try 块中抛出的特定类型的异常。比如坐标转换失败。
    //在这个例子中，catch(const tf2::LookupException& e) 表示捕获 tf2::LookupException 类型的异常，
    //e 是一个异常对象的引用，它包含了异常的详细信息。
    //当 try 块中抛出 tf2::LookupException 类型的异常时，程序会进入 catch 块，执行其中的代码。
    catch(const tf2::LookupException& e)
    {
      RCLCPP_INFO(this->get_logger(),"异常提示：%s",e.what());
    }
  }
```

需要注意的是想要转换的两个坐标系必须在==同一颗坐标树上==，否则会转换失败。

#### 监听点到坐标的变换

这里设置的接口是挺复杂的

```c++
class TFPointListener:public rclcpp::Node
{
  public:
  TFPointListener():Node("tf_point_listener_node_cpp")
  {
    // 3-1.创建坐标变换监听器,建听坐标关系
    buffer_ = std::make_shared<tf2_ros::Buffer>(this->get_clock());
    timer_ = std::make_shared<tf2_ros::CreateTimerROS>(
      this->get_node_base_interface(),
      this->get_node_timers_interface()
    );
    /*
    CreateTimerROS 将 ROS2 节点的定时器接口绑定到 tf2_ros::Buffer，使其能够创建后台定时任务。
    维护缓冲区：定期清理过期的坐标变换数据，避免内存无限增长。
    超时检测：检查是否有等待时间过长的坐标变换请求，防止资源阻塞。
    */
    buffer_->setCreateTimerInterface(timer_);
    listener_ = std::make_shared<tf2_ros::TransformListener>(*buffer_);
    // 3-2.创建坐标点信息订阅方
    point_sub.subscribe(this,"point");
    // 3-3.创建过滤器，解析数据
    /*
    F &f, tf2_ros::Buffer &buffer, 
    const std::string &target_frame, 
    uint32_t queue_size,
     const rclcpp::node_interfaces::NodeLoggingInterface::SharedPtr &node_logging, 
     const rclcpp::node_interfaces::NodeClockInterface::SharedPtr &node_clock, 
     std::chrono::duration<TimeRepT, TimeT> buffer_timeout
    */
    filter_ = std::make_shared<tf2_ros::MessageFilter<geometry_msgs::msg::PointStamped>>(
      point_sub,
      *buffer_,
      "base_link",
      10,
      this->get_node_logging_interface(),
      this->get_node_clock_interface(),
      1s
    );
    /*
    假设收到一个时间戳为 t=10.0s 的点消息，但 Buffer 中只有 t=8.0s 到 t=9.0s 的 base_link 变换。
    MessageFilter 会等待直到 t=11.0s（10.0s + 1s）。
    如果在这 1 秒内没有更新的变换到达，消息会被丢弃。
    场景 2：
    如果变换数据实时性较好（例如每 100ms 更新一次），MessageFilter 通常能在 1s 内找到匹配的变换，消息会被正常处理。
    */
    /*
    消息同步：MessageFilter 使用节点的时钟接口来检查消息的时间戳与坐标变换数据的时间是否同步。
    超时处理：若消息的时间戳与当前时间差距过大，可能丢弃或标记为过期。
    对坐标变换的意义：
    确保过滤器在处理消息时，能够基于节点的时钟判断消息的有效性，避免使用过时的坐标变换。
    */
    filter_->registerCallback(&TFPointListener::transform_point,this);
  }
  private:
  void transform_point(const geometry_msgs::msg::PointStamped& ps)
  {
    //实现坐标点转换
    auto out = buffer_->transform(ps,"base_link");
    RCLCPP_INFO(this->get_logger(),"发极坐标系：%s，坐标（%.2f,%.2f,%.2f）",
    out.header.frame_id.c_str(),
    out.point.x,
    out.point.y,
    out.point.z
    );
  }
  /*
  订阅原始点数据，支持与过滤器链（如MessageFilter）结合。
  ​**tf2_ros::MessageFilter**
  ​作用：在消息到达时检查Buffer中是否存在目标坐标系（base_link）的变换。
  */
  std::shared_ptr<tf2_ros::Buffer> buffer_;
  std::shared_ptr<tf2_ros::TransformListener> listener_;
  std::shared_ptr<tf2_ros::CreateTimerROS> timer_;
  message_filters::Subscriber<geometry_msgs::msg::PointStamped> point_sub;
  std::shared_ptr<tf2_ros::MessageFilter<geometry_msgs::msg::PointStamped>> filter_;
};
```



### ROS2编译指令

[编译指令](https://www.cnblogs.com/T-D-C/articles/18973156#31-%E7%BC%96%E8%AF%91)

### 录制ros2bag
```C++
ros2 bag record /camera_info /image_raw
ros2 bag play --loop 包名
```
方便调试的一些按键
空格键可以暂停
`→` (右方向键)	步进一帧（仅在暂停状态下有效）

