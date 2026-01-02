### 9.13语法

```c
detector_->data_save_.open(path_params_.save_path, ios::out | ios::trunc);//trunc指的是如果文件存在，就清空里面的所有内容
detector_->data_save_ << fixed; //以小数点的形式输出，不以科学计数法的形式输出，例如 3.14159 而不是可能被格式化为 3.14159e+00。
```

### 10.30语法

```c++
#define PROPERTY_WRAPPER_EXPAND_PARAMS(...) __VA_ARGS__
```

这段代码通常使用于一下情况来去除括号

```c++
DEFINE_PROPERTY(Pair, public, public, (std::map<int, float>))
```

当 `TYPE` 包含像 `std::map<int, float>` 这样的 **模板类型且有逗号** 时，直接把它写作宏实参会把逗号错误地当作宏参数分隔符 → 编译错误（“宏参数太多”）

```c++
#define A hello
#define B world
#define PASTE(a,b) a##b

PASTE(A,B)   // 结果通常是 A##B -> AB   而不是 helloworld
```

这个就是##的使用注意事项，注意如果参数传的是宏定义的时候，要使用==二级宏==

如下：

```c++
#define PRIMITIVE_PASTE(a,b) a##b
#define PASTE(a,b) PRIMITIVE_PASTE(a,b)

PASTE(A,B)   // 先展开为 PRIMITIVE_PASTE(hello, world) ->helloworld
```

### 11.1语法

- move函数：我感觉本质就是创造另一个指针指向`move(value)`value变量所申请的内存。这样可以避免内存复制、提高性能，适用于大对象。

### 11.6语法

```c++
#pragma once   //这个与下面的inline相比,他可以确保这个头文件在同个cpp文件下不会被重复展开
#include "transform6D.hpp"

/**
 * @brief 位姿节点类型别名
 */
using PoseNode = Transform6D; //!< 位姿节点类型

/**
 * @brief 坐标系类型
 */
struct CoordFrame
{
    static std::string WORLD;  //!< 世界坐标系标识
    static std::string CAMERA; //!< 相机坐标系标识
    static std::string JOINT;  //!< 转轴坐标系标识
    static std::string GYRO;   //!< 陀螺仪坐标系标识
};

//! 世界坐标系
inline std::string CoordFrame::WORLD = "world"; //这里inline的作用是：当多个cpp文件被包含这个头文件时，链接阶段时，代码会把这个定义视为多
//! 相机坐标系
inline std::string CoordFrame::CAMERA = "camera";
//! 转轴坐标系
inline std::string CoordFrame::JOINT = "joint";
//! 陀螺仪坐标系
inline std::string CoordFrame::GYRO = "gyro";

```

### 11.22语法

```c++
std::numeric_limits<double>::epsilon();
//上面的数值可以用来表示精度差值，比如我们要表示两个double，float类型的数值相等，那么Tp delta = A1 * B2 - A2 * B1; std::abs(delta) < std::numeric_limits<double>::epsilon()

```

### 12.3语法

`// 会抛出异常：Cannot find a connection between camera_link and device_link`这个意味着两个坐标系在不同的坐标树下。tf_buffer_是可以创建多个独立的树的，但是不能执行跨树转换。

### 12.4语法

==下面是关于C++代码的==

来自 YouTube:

[https://www.youtube.com/@TheCherno ](https://www.youtube.com/@TheCherno)- 有些人说他写代码的方式不是最优化，但如果你对他这种风格稍微宽容一点，你会学到很多关于这门语言如何运作以及如何利用它的知识。

[https://www.youtube.com/@cppweekly ](https://www.youtube.com/@cppweekly)- 他涵盖了很多好的主题，定义并讲解了很多要避免的代码异味。

[https://www.youtube.com/@JacobSorber ](https://www.youtube.com/@JacobSorber)- 他更多地涵盖了 C，但对指针和其他来自该语言的东西有很多见解（它们不一样）

这些是最好的文档，我两个都用，取决于例子和解释有多好。

https://www.google.com/search?q=cppreference&oq=cpprefere&aqs=chrome.0.0i512j69i57j0i512l3j69i60l3.5228j0j7&sourceid=chrome&ie=UTF-8

https://cplusplus.com/reference/

对于学习和阅读这门语言，LearnCPP 真的很好，过去几年没用过它，但据我记得，它有深入的解释和例子。

https://www.learncpp.com/

至于如何获得经验，就去构建东西。 我建议不要使用任何 IDE，这将是一个更具实践性和沉浸式的学习体验，你将学习如何设置你的构建（makefile、cmake），链接如何工作，如何包含文件等等。（这取决于你愿意花多少时间来调试设置问题）



### 12.6语法

`decltype`关键字用于自动推导类型，`std::vector<TargetAngleInfo> inliers;decltype(inliers) sample;`这里推导出来的类型就是`std::vector<TargetAngleInfo>`.

但同样是自动推导类型，他与auto存在一定的差异

```c++
const std::vector<int>& ref = vec;
auto a = ref;               // a 的类型是 std::vector<int>（ cv 和引用都被丢弃）
decltype(ref) b = ref;      // b 的类型是 const std::vector<int>& （完全保留）
```

