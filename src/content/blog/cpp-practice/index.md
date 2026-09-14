---
title: 'C++工程实践'
description: '在掌握了c++基本语法的情况下，掌握一些进阶的工程实践'
publishDate: '2026-09-13 21:33:44'
language: '中文'
tags:
  - C++
  - RoboMaster
heroImage: { src: './cover.jpg', color: '#D58388' }
---

经历了c++一些基础语法的学习（比如变量、逻辑语句、指针、函数、结构体等电类大一程设课就会学到的知识）之后，我们需要掌握一些常用的小技巧,来让我们在较为大型的项目编写中效率更高。

## 面向对象

在只会写小作业的时候，我们可以把程序组织成「结构体装数据 + 一堆函数处理数据」的样子，这在代码只有几百行时没什么问题。但当项目变大——比如一个视觉程序要同时处理自瞄、能量机关和导航——这种写法的弊端就会暴露：数据是公开的，函数是散落的，你想改一个字段的含义，得先把操作它的所有函数翻出来改一遍。

面向对象（Object-Oriented Programming）的核心想法只有一句话：**把数据和操作数据的方法打包成一个整体**，这个整体就是类（class）。这一节按「封装 → 继承 → 多态」的顺序，把它拆开来看。

### 从结构体到类

先看一个反例。结构体 `Circle` 保存半径，函数 `area` 根据半径算面积：

```cpp title="struct_problem.cpp"
#include <iostream>

struct Circle {
    double radius;
};

double area(const Circle& c) { return 3.14159 * c.radius * c.radius; }

int main() {
    Circle c;
    c.radius = -5;                 // 编译通过，但圆的半径不该是负数
    std::cout << area(c) << '\n';  // 输出 78.5397，一个不存在的圆
    return 0;
}
```

半径是圆的内部状态，`area` 是依赖这个状态的行为，但两者的联系完全靠命名维持：任何人都能往 `radius` 里塞进任意数字，`area` 没有任何机会阻止。

我们希望的写法是：数据自己保管状态，操作跟着数据走，只有类允许的操作才能改它——这正是 `class` 要做的事。

### 封装：把状态藏起来

把上面的例子改写成类：

```cpp title="encapsulation.cpp"
#include <iostream>

class Circle {
public:
    double area() const { return 3.14159 * radius_ * radius_; }

    double radius() const { return radius_; }

    void setRadius(double r) {
        if (r <= 0) {
            std::cout << "半径必须为正数\n";
            return;
        }
        radius_ = r;
    }

private:
    double radius_ = 1.0;
};

int main() {
    Circle c;
    c.setRadius(-5);
    std::cout << c.radius() << ' ' << c.area() << '\n';
    c.setRadius(2);
    std::cout << c.radius() << ' ' << c.area() << '\n';
    return 0;
}
```

`public` 里的成员组成类的对外接口，`private` 里的成员只有类自己能看到。现在外界没法直接碰 `radius_` 了，只能通过 `setRadius` 函数修改，而 `setRadius` 顺手就挡掉了非正数；想读半径或者算面积，分别调用 `radius()` 和 `area()` 即可。函数签名末尾的 `const` 表示这个成员函数不会修改对象。

通过封装，在`main`函数里就可以直接调用对象内部函数的接口，极大简化了`main`函数内部的操作，也增加了安全性。这样的话，调用者只需要记住 `setRadius` 会拒绝非正数这个承诺，不需要关心内部是用 `if` 判断还是抛异常。

### 构造函数、析构函数与初始化列表

上面的 `radius_` 靠 `= 1.0` 就地初始化，但如果初始值要由创建者决定，就得用构造函数。构造函数在对象创建时自动调用，名字与类相同，没有返回值；析构函数反过来，在对象销毁时自动调用，名字是 `~` 加类名，负责收尾。C++ 最经典的资源管理手法就建立在这两者之上：构造时申请，析构时释放，也就是常说的 RAII。

```cpp title="raii.cpp"
#include <iostream>

class IntArray {
public:
    IntArray(int size) : size_(size), data_(new int[size]) {}

    ~IntArray() { delete[] data_; }

    void set(int index, int value) { data_[index] = value; }

    int get(int index) const { return data_[index]; }

private:
    int size_;
    int* data_;
};

int main() {
    IntArray arr(3);
    arr.set(0, 1);
    arr.set(1, 2);
    arr.set(2, 3);
    std::cout << arr.get(0) + arr.get(1) + arr.get(2) << '\n';  // 输出 6
    return 0;
}
```

冒号后面的部分叫初始化列表，它在构造函数体执行之前完成成员的初始化。对 `const` 成员、引用成员，或者没有默认构造函数的成员来说，初始化列表是唯一的写法——因为进入函数体时，所有成员都已经构造完毕了。

初始化列表还有一个注意点：**成员的初始化顺序由声明顺序决定，与你写在列表里的顺序无关**。所以列表最好按声明顺序写，否则很容易踩到「拿一个还没初始化的成员去初始化另一个成员」的问题。

另外要注意：一旦为类写了构造函数，编译器就不会再自动生成默认构造函数；而本例为了简单，没有处理拷贝的情况（正式代码里应当禁用拷贝或实现深拷贝），这里只需要记住「析构函数会自动收尾」这件事。

### this 指针

构造函数里有个常见歧义：如果参数名和成员名一样，函数里的名字到底指谁？

```cpp title="this_pointer.cpp"
#include <iostream>

class Point {
public:
    Point(int x, int y) {
        this->x = x;  // this->x 是成员，右边的 x 是参数
        this->y = y;
    }

    int getX() const { return x; }
    int getY() const { return y; }

    void setX(int x) { this->x = x; }

private:
    int x;
    int y;
};

int main() {
    Point p(3, 4);
    p.setX(10);
    std::cout << p.getX() << ' ' << p.getY() << '\n';  // 输出 10 4
    return 0;
}
```

`this` 是指向「当前对象」的指针，`this->x` 明确表示成员变量，而不带前缀的 `x` 会优先匹配参数。平时在成员函数里访问成员，编译器会自动补上 `this->`，所以几乎感觉不到它的存在；出现重名歧义时，它才是唯一说得清楚的写法。此外 `return *this;` 可以返回对象自身（用于支持链式调用），这里不展开。

### 继承

假设程序里除了“猫”还会有“狗”两个对象，它们都共享「名字」「呼吸」这些成员，从头写两遍很快就会变成维护负担。继承把公共部分提取到一个父类（基类）里，子类（派生类）自动拥有父类的成员，只需要补上自己的部分：

```cpp title="inheritance.cpp"
#include <iostream>
#include <string>

class Animal {
public:
    Animal(const std::string& name) : name_(name) {}

    void breathe() const { std::cout << name_ << " 在呼吸\n"; }

protected:
    std::string name_;  // 子类可以访问，外部不行
};

class Cat : public Animal {
public:
    Cat(const std::string& name) : Animal(name) {}  // 先构造父类
    void meow() const { std::cout << name_ << " 喵\n"; }
};

int main() {
    Cat cat("咪咪");
    cat.breathe();
    cat.meow();
    return 0;
}
```

`class Cat : public Animal` 读作「Cat 是一种 Animal」。创建 `Cat` 时先调用父类构造函数、再执行子类构造函数，析构顺序相反；如果父类没有默认构造函数，子类就必须在初始化列表里显式调用它。`protected` 介于 `public` 和 `private` 之间：外部访问不了，但子类可以，适合放这种「留给子类用」的内部状态。

需要留意，`public` 在这里不能省略：`class` 的继承默认是 `private` 继承，会把父类成员全部对外藏起来，和 `struct` 默认的 `public` 继承刚好相反。

### 多态与虚函数

继承更重要的能力体现在下面这种场景：例如写一个函数，接收任意一种动物并让它发出叫声。

```cpp title="static_binding.cpp"
#include <iostream>
#include <string>

class Animal {
public:
    Animal(const std::string& name) : name_(name) {}
    void speak() const { std::cout << name_ << " 发出声音\n"; }
protected:
    std::string name_;
};

class Cat : public Animal {
public:
    Cat(const std::string& name) : Animal(name) {}
    void speak() const { std::cout << name_ << " 喵\n"; }  // 看起来覆盖了父类版本
};

void greet(const Animal& animal) { animal.speak(); }

int main() {
    Cat cat("咪咪");
    greet(cat);  // 输出：咪咪 发出声音
    return 0;
}
```

奇怪的是，`cat` 明明是 `Cat`，`greet` 却调用了父类的 `speak`。原因是 `Animal&` 这个类型在编译期就决定了调用哪个函数，这叫静态绑定；而我们想要的是运行期根据对象的实际类型选择函数，也就是多态。让这件事发生的关键字是 `virtual`：

```cpp title="virtual_override.cpp"
#include <iostream>
#include <string>

class Animal {
public:
    Animal(const std::string& name) : name_(name) {}

    virtual void speak() const { std::cout << name_ << " 发出声音\n"; }

    virtual ~Animal() = default;

protected:
    std::string name_;
};

class Cat : public Animal {
public:
    Cat(const std::string& name) : Animal(name) {}

    void speak() const override { std::cout << name_ << " 喵\n"; }
};

void greet(const Animal& animal) { animal.speak(); }

int main() {
    Cat cat("咪咪");
    greet(cat);  // 输出：咪咪 喵
    return 0;
}
```

父类函数加上 `virtual` 后变得「可以被覆盖」，运行时会根据对象的真实类型选择版本。子类这边推荐写上 `override`：它对运行没有任何开销，但能让编译器检查「你以为覆盖了、其实没覆盖」的情况——参数类型或 `const` 不一致时，没有 `override` 的写法会悄悄变成新增一个重载，加上 `override` 就会直接编译报错。

还需要注意父类的 `virtual ~Animal() = default;`。如果一个类会被继承，并且可能通过基类指针删除子类对象，基类的析构函数就必须是虚函数，否则子类的析构不会执行：

```cpp title="virtual_dtor.cpp"
#include <iostream>

class Base {
public:
    ~Base() { std::cout << "~Base\n"; }  // 不是虚函数！
};

class Derived : public Base {
public:
    ~Derived() { std::cout << "~Derived\n"; }
};

int main() {
    Base* p = new Derived;
    delete p;  // 在绝大多数实现上只输出 ~Base，~Derived 被跳过
    return 0;
}
```

把 `~Base` 改成 `virtual ~Base() = default;`，输出就会变成先 `~Derived` 再 `~Base`。规则很好记：**只要一个类可能被继承，就给它一个虚析构函数。**

### 纯虚函数与抽象类

回头再看 `Animal`，它的 `speak` 实现其实很鸡肋：现实里没有一种叫「动物」的东西会发出声音，父类在这里只负责给子类定标准。既然父类不需要实现，可以干脆写成纯虚函数；包含纯虚函数的类叫抽象类，不能被实例化：

```cpp title="abstract.cpp"
#include <iostream>

class Animal {
public:
    virtual void speak() const = 0;  // 纯虚函数
    virtual ~Animal() = default;
};

class Cat : public Animal {
public:
    void speak() const override { std::cout << "喵\n"; }
};

int main() {
    // Animal animal;         // 错误：抽象类不能被实例化
    Animal* pet = new Cat();  // 可以：基类指针指向具体子类
    pet->speak();             // 输出：喵
    delete pet;
    return 0;
}
```

抽象类像一份合同：凡是宣称自己是一种 `Animal` 的类，都必须把 `speak` 实现出来，否则它也是抽象类，同样不能实例化。工程里常用它来表达接口——比如视觉程序可以把自瞄、能量机关等任务都抽象成同一个「检测器」接口，各自实现识别逻辑，上层调度代码只面对接口，不必知道此刻在跑的是哪一个检测器。

### 小结

封装让数据只通过接口被修改，继承让公共代码只需要写一遍，多态让调用者与具体实现解耦——这三者是面向对象最基本的骨架。到这里，我们已经能把程序拆成多个互相配合的类了；但一个正经项目会把它们分散在几十个 `.hpp` 和 `.cpp` 文件里，怎么让这些文件一起变成一个可执行文件？接下来的 CMake 解决的就是这件事。

## CMake

### 编译与链接

问题先从「多文件怎么编译」说起。假设我们把上一节的工具函数整理成一个真实工程：

```text title="目录结构"
cpp-practice/
├── include/
│   └── math_utils.hpp
└── src/
    ├── main.cpp
    └── math_utils.cpp
```

`.hpp` 放函数声明，`.cpp` 放函数实现：

```cpp title="include/math_utils.hpp"
#pragma once

int add(int a, int b);
```

```cpp title="src/math_utils.cpp"
#include "math_utils.hpp"

int add(int a, int b) { return a + b; }
```

```cpp title="src/main.cpp"
#include <iostream>

#include "math_utils.hpp"

int main() {
    std::cout << "1 + 2 = " << add(1, 2) << '\n';
    return 0;
}
```

`#pragma once` 用来防止同一个头文件被重复展开；`#include "..."` 在预处理阶段把头文件内容原样粘贴进来。真正的编译以每个 `.cpp` 为单位（称为一个翻译单元），各自生成 `.o` 目标文件，最后链接器把目标文件和用到的库拼成可执行文件。头文件里只有声明，所以任何包含 `math_utils.hpp` 的源文件都能通过编译；如果忘了写 `math_utils.cpp`，就会在链接阶段报 `undefined reference`。

不借助任何工具，编译这个工程只需要：

```bash title="终端"
g++ -std=c++17 src/main.cpp src/math_utils.cpp -Iinclude -o main
./main    # 输出：1 + 2 = 3
```

文件一多，这条命令就会越写越长；换台机器、换个库，路径和参数又要重来；更麻烦的是每次都全量重新编译。构建工具要解决的正是「怎么描述并执行这套流程」。

### 为什么是 CMake

在 CMake 之前流行的是 Makefile，它也能做到增量编译，但语法古老（规则行必须以 Tab 开头）、且与平台绑定。CMake 换了个思路：我们只写一份平台无关的 `CMakeLists.txt` 描述工程，CMake 再据此生成对应平台的构建文件（Linux 下是 Makefile，Windows 下是 Visual Studio 工程等）。整个构建分成两步：

```bash title="终端"
cmake -S . -B build     # 配置：读取 CMakeLists.txt，在 build/ 下生成构建文件
# 其中-S . 为Source directory（源码目录）是当前目录
# -B build 为Build directory（构建目录）是 build/
cmake --build build     # 构建：调用底层的 make 等工具完成编译链接
```

另外有一种写法如下：
```bash title="终端"
mkdir build   # 在当前目录建立build文件夹，mkdir为“make directory”
cd build    # 进入build文件夹
cmake ..    # ..表示源码目录是当前目录的上一级目录
make -j8    # 使用 Make 进行编译，并同时开启最多 8 个并行编译任务
```

注意所有生成物都在 `build/` 里，源码目录保持干净——这就是所谓的 out-of-source 构建，想从头再来直接删掉 `build/` 即可。记得把 `build/` 写进 `.gitignore`。

### 第一个 CMakeLists.txt

上面那个工程对应的 `CMakeLists.txt` 只有几行：

```cmake title="CMakeLists.txt"
cmake_minimum_required(VERSION 3.22)
project(cpp_practice LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(main src/main.cpp src/math_utils.cpp)
target_include_directories(main PRIVATE include)
```

逐行看：

- `cmake_minimum_required` 声明所需的 CMake 最低版本。这里取 3.22，也就是 Ubuntu 22.04 自带的版本。
- `project` 声明工程名与使用的语言。
- 两行 `set` 让所有目标统一使用 C++17；`CMAKE_CXX_STANDARD_REQUIRED ON` 表示编译器不支持就报错，而不是悄悄退回旧标准。
- `add_executable` 生成名为 `main` 的可执行文件，源文件需要一一列出。
- `target_include_directories` 告诉编译器，`#include "math_utils.hpp"` 时去 `include/` 目录里找。这里的 `PRIVATE` 后面再解释。

然后执行：

```bash title="终端"
cmake -S . -B build     # 配置
cmake --build build     # 构建
./build/main            # 输出：1 + 2 = 3
```

### 目标、库与传播

CMake 里最重要的概念是**目标（target）**：每个 `add_executable`、`add_library` 生成的都是一个目标，目标的属性包括源文件、头文件路径、编译选项、依赖的库等。现代 CMake 的写法就是围绕目标来配置，而不是设置一堆全局变量。

当 `math_utils` 这类工具代码被多个可执行文件共用时，应该把它编译成库：

```cmake title="CMakeLists.txt"
cmake_minimum_required(VERSION 3.22)
project(cpp_practice LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_library(math STATIC src/math_utils.cpp)
target_include_directories(math PUBLIC include)

add_executable(main src/main.cpp)
target_link_libraries(main PRIVATE math)
```

`add_library` 把 `math_utils.cpp` 编译成静态库 `libmath.a`；`main` 只需要 `target_link_libraries(main PRIVATE math)`，既完成了链接，也自动获得了库的头文件路径——因为 `include/` 是用 `PUBLIC` 挂到 `math` 上的。三个关键字的区别可以总结成一句话：

| 关键字 | 我自己编译时需要 | 使用我的人编译时需要 | 典型场景 |
| --- | --- | --- | --- |
| `PRIVATE` | 是 | 否 | 只在自己源文件里用到的依赖 |
| `PUBLIC` | 是 | 是 | 头文件里就 `#include` 的依赖（如 `include/`） |
| `INTERFACE` | 否 | 是 | 纯头文件库 |

也就是：`PRIVATE` 是「我用但你别管」，`PUBLIC` 是「我用你也得用」，`INTERFACE` 是「我不用但你得用」。属性会沿着链接关系自动传播给使用者，这正是它比全局的 `include_directories()` 更可靠的地方——全局设置会污染每一个目标，而目标属性只影响该影响的人。

### 引入第三方库：OpenCV

RM 视觉离不开 OpenCV，引入方式与上面的 `math` 库没有本质区别，只是库的位置交给 `find_package` 去找：

```cmake title="CMakeLists.txt"
cmake_minimum_required(VERSION 3.22)
project(rm_vision LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(OpenCV REQUIRED)

add_executable(main src/main.cpp)
target_link_libraries(main PRIVATE opencv_core opencv_imgproc)
```

```cpp title="src/main.cpp"
#include <iostream>

#include <opencv2/core.hpp>
#include <opencv2/imgproc.hpp>

int main() {
    cv::Mat gray(480, 640, CV_8UC1, cv::Scalar(128));
    cv::Mat color;
    cv::cvtColor(gray, color, cv::COLOR_GRAY2BGR);
    std::cout << color.cols << "x" << color.rows
              << ", channels = " << color.channels() << '\n';  // 输出：640x480, channels = 3
    return 0;
}
```

`find_package(OpenCV REQUIRED)` 会在系统约定路径里寻找 OpenCV 的 CMake 配置，找到后把各个模块注册成可以链接的目标；`opencv_core`、`opencv_imgproc` 就是导入目标，它们的头文件路径等属性由 OpenCV 自己带过来，所以不需要再手写 `target_include_directories`。Eigen 这种纯头文件库同理：`find_package(Eigen3 REQUIRED)` 之后链接 `Eigen3::Eigen` 即可。

### 几条工程习惯

最后是几条实践建议：

- 永远在源码目录之外构建，不要用 `cmake .`，否则生成物散落在源码里，清理起来非常麻烦。
- 加上 `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`，CMake 会生成 `build/compile_commands.json`，VS Code 的 clangd 插件靠它提供补全和跳转。
- 构建行为诡异时，最省事的做法通常是 `rm -rf build` 重来，CMake 缓存被旧配置污染后，排查成本远高于重建。

### 小结

CMake 自己不编译代码，它只是把我们关于工程的描述翻译成构建工具能执行的指令。掌握「一份 `CMakeLists.txt` + 若干 target + 用 `target_link_libraries` 连接依赖」这套骨架，就足以应付绝大多数工程；以后面对陌生的 CMake 项目，从 `add_executable` / `add_library` 两行读起，基本不会迷路。RM 视觉工程的结构也大多如此：`include/` 放头文件，`src/` 放实现，再用 `find_package` 引入 OpenCV、Eigen 这些依赖。