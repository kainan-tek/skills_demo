---
name: "google-cpp-style"
description: "Google C++ Style Guide for code standards. Invoke when writing C++ code, reviewing code, or ensuring compliance with Google C++ coding conventions. 谷歌 C++ 编码规范，用于代码标准检查。在编写 C++ 代码、审查代码或确保符合 Google C++ 编码规范时调用。"
version: "1.0.0"
triggers: ["coding style", "code review", "naming convention", "变量命名", "编码规范", "代码风格", "C++ style", "Google style", "代码检查", "code standards", "命名规范", "格式规范"]
---

# Google C++ Style Guide (谷歌C++编码规范)

本规范基于 Google C++ Style Guide 官方文档，适用于 C++ 项目开发。

## 目录

1. [头文件规范](#头文件规范)
2. [作用域](#作用域)
3. [类设计](#类设计)
4. [函数规范](#函数规范)
5. [命名规范](#命名规范)
6. [注释规范](#注释规范)
7. [格式规范](#格式规范)
8. [其他重要规则](#其他重要规则)
9. [Android 项目特殊规则](#android-项目特殊规则)

---

## 头文件规范

### 自包含头文件 (Self-contained Headers)

- 所有头文件应该是自包含的（可独立编译）
- 头文件扩展名使用 `.h`，非头文件但用于包含的文件使用 `.inc`
- 内联函数和模板定义必须放在头文件中

### 头文件保护 (#define Guard)

- 所有头文件必须有 `#define` 保护符防止重复包含
- 格式：`<PROJECT>_<PATH>_<FILE>_H_`

```cpp
#ifndef FOO_BAR_BAZ_H_
#define FOO_BAR_BAZ_H_

// ... 内容 ...

#endif  // FOO_BAR_BAZ_H_
```

### 包含顺序 (Include Order)

按以下顺序包含头文件，每组之间用空行分隔：

1. 相关头文件（如 `foo.cc` 对应的 `foo.h`）
2. C 系统头文件（如 `<sys/types.h>`、`<unistd.h>`）
3. C++ 标准库头文件（如 `<string>`、`<vector>`）
4. 其他库的头文件
5. 本项目的头文件

```cpp
#include "foo/server/fooserver.h"

#include <sys/types.h>
#include <unistd.h>

#include <string>
#include <vector>

#include "base/basictypes.h"
#include "foo/server/bar.h"
#include "third_party/absl/flags/flag.h"
```

### 前向声明 (Forward Declarations)

- **尽量避免使用前向声明**，应直接包含所需的头文件
- 前向声明可能隐藏依赖关系，导致编译问题

---

## 作用域

### 命名空间 (Namespaces)

- 几乎所有代码都应放在命名空间中
- 命名空间名称应基于项目名和路径
- **禁止使用 `using namespace` 指令**
- **禁止使用内联命名空间 (inline namespace)**

```cpp
namespace mynamespace {

class MyClass {
 public:
  void Foo();
};

}  // namespace mynamespace
```

### 内部链接 (Internal Linkage)

- 在 `.cc` 文件中不需要外部引用的定义，应使用匿名命名空间或 `static` 声明
- **不要在 `.h` 文件中使用匿名命名空间或 `static`**

```cpp
namespace {
// 内部使用的函数
void InternalHelper() { ... }
}  // namespace
```

### 局部变量 (Local Variables)

- 在最小作用域内声明变量
- 在声明时初始化变量

```cpp
int i = f();  // Good
int i;
i = f();      // Bad
```

### 静态和全局变量 (Static and Global Variables)

- **禁止**使用非平凡析构的静态存储期对象
- 使用 `constexpr` 或 `constinit` 标记常量初始化

```cpp
constexpr std::array<int, 3> kArray = {1, 2, 3};  // Good
const std::string kFoo = "foo";                   // Bad: 非平凡析构
```

---

## 类设计

### 构造函数职责

- 构造函数中**禁止调用虚函数**
- 避免在构造函数中进行可能失败的初始化
- 如需复杂初始化，考虑使用工厂函数或 `Init()` 方法

### 隐式转换 (Implicit Conversions)

- **必须**使用 `explicit` 关键字标记单参数构造函数和类型转换运算符
- 拷贝和移动构造函数不需要 `explicit`

```cpp
class Foo {
 public:
  explicit Foo(int x);           // Good
  explicit operator bool() const; // Good
};
```

### 可拷贝和可移动类型

- 类的公共接口必须明确说明是可拷贝、仅可移动、还是两者皆不可

```cpp
class Copyable {
 public:
  Copyable(const Copyable&) = default;
  Copyable& operator=(const Copyable&) = default;
};

class MoveOnly {
 public:
  MoveOnly(MoveOnly&&) = default;
  MoveOnly& operator=(MoveOnly&&) = default;
  MoveOnly(const MoveOnly&) = delete;
  MoveOnly& operator=(const MoveOnly&) = delete;
};
```

### 结构体 vs 类

- 使用 `struct` 仅用于被动数据对象（所有成员公开）
- 其他情况使用 `class`

### 继承 (Inheritance)

- 组合通常比继承更合适
- 所有继承应为 `public`
- 使用 `override` 或 `final` 标记虚函数重写
- **禁止**在虚函数中使用默认参数

```cpp
class Derived : public Base {
 public:
  void DoSomething() override;  // Good
};
```

### 访问控制

- 类数据成员应为 `private`（常量除外）
- 测试夹具类的数据成员可为 `protected`

### 声明顺序

类定义中的声明顺序：

1. `public:` 部分
2. `protected:` 部分
3. `private:` 部分

每个部分内：

1. 类型和类型别名（typedef, using, enum, 嵌套类）
2. 静态常量
3. 工厂函数
4. 构造函数和赋值运算符
5. 析构函数
6. 其他成员函数
7. 数据成员

---

## 函数规范

### 输入和输出

- 优先使用返回值而非输出参数
- 非可选输入参数应为值或 `const` 引用
- 输出和输入/输出参数应为引用

### 编写短函数

- 函数应短小精悍
- 如果函数超过 40 行，考虑拆分

### 函数重载

- 仅在调用者无需了解具体调用哪个重载时使用重载

### 默认参数

- 非虚函数允许使用默认参数
- 默认参数值必须保证始终相同
- **禁止**在虚函数中使用默认参数

### 尾置返回类型

- 仅在必要时使用尾置返回类型语法

```cpp
// 传统风格（推荐）
int Foo(int x);

// 尾置返回类型（必要时使用）
auto Foo(int x) -> int;
```

---

## 命名规范

### 通用规则

| 类型 | 命名风格 | 示例 |
|------|----------|------|
| 文件名 | 小写下划线 | `my_useful_class.cc` |
| 类型/类名 | 大驼峰 (PascalCase) | `MyClass`, `UrlTable` |
| 变量名 | 小写下划线 | `my_variable` |
| 常量名 | k + 大驼峰 | `kDaysInAWeek` |
| 函数名 | 大驼峰 | `AddTableEntry()` |
| 命名空间 | 小写下划线 | `my_namespace` |
| 枚举值 | k + 大驼峰 | `kEnumValue` |
| 宏名 | 大写下划线 | `PI_ROUNDED` |

### 文件命名

- 全部小写，使用下划线连接
- `.h` 头文件，`.cc` 源文件

```text
my_useful_class.h
my_useful_class.cc
```

### 类型命名

- 类、结构体、类型别名、枚举、模板参数：使用大驼峰

```cpp
class UrlTable { ... };
struct UrlTableProperties { ... };
using UrlTableMap = std::map<std::string, UrlTable*>;
enum class UrlTableError { ... };
template <typename T>
class MyTemplate { ... };
```

### 变量命名

- 普通变量：小写下划线
- 类成员变量：小写下划线，末尾加下划线
- 静态成员变量：`s` 前缀 + 小写下划线 + 末尾下划线

```cpp
std::string table_name;       // 普通变量
std::string name_;            // 类成员变量
static int s_instance_count_; // 静态成员变量
```

### 常量命名

- 使用 `k` 前缀 + 大驼峰

```cpp
const int kDaysInAWeek = 7;
constexpr int kMaxSize = 100;
```

### 函数命名

- 普通函数：大驼峰
- 访问器/修改器：与变量名对应
- Android 项目：可使用小驼峰以保持与 Android API 风格一致

```cpp
// 标准风格（大驼峰）
void AddTableEntry();
void DoSomething();

// 访问器/修改器风格
void set_count(int count);
int count() const;

// Android 项目风格（小驼峰，与 Android API 保持一致）
void writeData(const char* data, size_t size);
void updateHeader();
int getSampleRate() const;
```

### 命名空间命名

- 全部小写，使用下划线连接

```cpp
namespace search_engine {
namespace index_util {
}  // namespace index_util
}  // namespace search_engine
```

---

## 注释规范

### 注释风格

- 使用 `//` 或 `/* */`，保持一致即可
- 注释应使用正确的标点、拼写和语法

### 文件注释

- 每个文件开头应有版权声明和内容概述

```cpp
// Copyright 2024 Google LLC
// 本文件包含 Foo 类的实现，用于...
```

### 类注释

- 每个类定义应有注释说明其用途和使用方法

```cpp
// Foo 类用于表示...
// 示例用法：
//   Foo foo;
//   foo.DoSomething();
class Foo { ... };
```

### 函数注释

- 函数声明处注释说明函数用途
- 注释应描述输入、输出和副作用

```cpp
// 计算两个数的和。
// 参数:
//   a: 第一个加数
//   b: 第二个加数
// 返回:
//   两数之和
int Add(int a, int b);
```

### 变量注释

- 复杂或不明显的变量应注释

```cpp
int num_entries_;  // 条目总数（不包括已删除的）
```

### TODO 注释

- 使用 `TODO` 注释标记待办事项

```cpp
// TODO(username): 描述需要做的事情
```

---

## 格式规范

### 行长度

- 每行代码不超过 80 字符

### 缩进

- 使用空格缩进，不使用 Tab
- 每级缩进 2 个空格

### 函数声明和定义

- 返回类型与函数名在同一行
- 参数尽量放在同一行，否则换行对齐

```cpp
ReturnType ClassName::FunctionName(Type par_name1, Type par_name2) {
  DoSomething();
  DoSomethingElse();
}
```

### Lambda 表达式

- 短 lambda 可单行
- 长 lambda 应换行

```cpp
std::sort(v.begin(), v.end(), [](int x, int y) {
  return Weight(x) < Weight(y);
});
```

### 函数调用

- 参数尽量放在同一行
- 换行时对齐参数

```cpp
bool result = DoSomething(argument1, argument2, argument3);

bool result = DoSomething(averyveryveryverylongargument1,
                          argument2,
                          argument3);
```

### 条件语句

- `if`、`else` 关键字后有空格
- 条件语句体即使只有一行也要使用大括号

```cpp
if (condition) {
  DoSomething();
} else {
  DoSomethingElse();
}
```

### 循环和 switch

```cpp
switch (var) {
  case 0: {
    DoSomething();
    break;
  }
  case 1: {
    DoSomethingElse();
    break;
  }
  default: {
    DoDefault();
    break;
  }
}
```

### 指针和引用

- `*` 和 `&` 紧跟类型名

```cpp
char* c;
const std::string& str;
```

### 布尔表达式

- 长布尔表达式换行时，运算符在行首

```cpp
if (condition_one &&
    condition_two &&
    condition_three) {
  DoSomething();
}
```

### 返回值

- 不要在 `return` 表达式中添加不必要的括号

```cpp
return result;          // Good
return (result);        // Bad
```

### 变量和数组初始化

- 使用 `=`、`()` 或 `{}` 初始化

```cpp
int x = 3;
int x(3);
int x{3};
std::vector<int> v = {1, 2, 3};
```

### 预处理指令

- 预处理指令从行首开始，不缩进

```cpp
if (condition) {
#ifdef DEBUG
  LogDebug();
#endif
  DoSomething();
}
```

### 类格式

- 访问说明符缩进 1 个空格
- 成员缩进 2 个空格

```cpp
class MyClass : public BaseClass {
 public:
  MyClass();
  void DoSomething();

 private:
  int member_;
};
```

### 构造函数初始化列表

- 冒号后换行，成员初始化对齐

```cpp
MyClass::MyClass(int var)
    : BaseClass(var),
      member_(var),
      another_member_(0) {
  DoSomething();
}
```

### 命名空间格式

- 命名空间内容不缩进

```cpp
namespace my_namespace {

void Foo() {
  DoSomething();
}

}  // namespace my_namespace
```

### 水平空白

- 不要在行尾添加空白
- 二元运算符两侧加空格
- 逗号后加空格

---

## 其他重要规则

### 智能指针和所有权

- 优先使用单一、固定的所有者管理动态分配对象
- 使用智能指针转移所有权
- 优先使用 `std::unique_ptr`
- 谨慎使用 `std::shared_ptr`

```cpp
std::unique_ptr<Foo> FooFactory();
void FooConsumer(std::unique_ptr<Foo> ptr);
```

### 异常

- **禁止使用 C++ 异常**

### RTTI

- 避免使用运行时类型信息 (RTTI)
- 优先使用虚函数替代 `dynamic_cast`

### 类型转换

- 使用 C++ 风格的类型转换
- **禁止使用 C 风格的类型转换**

```cpp
int64_t y = int64_t{1} << 42;          // Good
float f = static_cast<float>(d);       // Good
int i = (int)d;                        // Bad
```

### 流

- 使用流进行简单的 I/O 操作
- 避免使用流处理面向外部用户的 I/O

### 前置自增/自减

- 优先使用前置形式 (`++i`)，除非需要后置语义

```cpp
for (int i = 0; i < 10; ++i) {  // Good
  DoSomething(i);
}
```

### const 使用

- 在 API 中尽可能使用 `const`
- 函数参数使用 `const T&` 或 `const T*`
- 不修改对象状态的成员函数标记为 `const`

```cpp
void Process(const std::string& input);
int GetValue() const;
```

### constexpr

- 使用 `constexpr` 定义真正的常量

```cpp
constexpr int kMaxSize = 100;
constexpr double Pi = 3.14159265358979;
```

### 整数类型

- 使用 `int` 表示已知不会太大的整数
- 需要特定大小时使用 `<stdint.h>` 中的类型（如 `int64_t`）
- 避免使用无符号类型表示非负数

```cpp
int count;                    // 普通计数
int64_t large_count;          // 可能很大的计数
uint32_t bit_field;           // 位模式（合理使用）
```

### auto 使用

- 仅在使代码更清晰时使用 `auto`
- 不要仅为了少打字而使用 `auto`

```cpp
auto widget = std::make_unique<Widget>();  // Good: 类型明显
auto it = my_map.find(key);                 // Good: 迭代器类型冗长
auto x = SomeFunction();                    // Bad: 类型不明显
```

### Lambda 表达式

- 适当使用 lambda 表达式
- lambda 逃逸当前作用域时优先使用显式捕获

```cpp
std::sort(v.begin(), v.end(), [](int x, int y) {
  return Weight(x) < Weight(y);
});
```

### std::move 和 std::forward

- 使用 `std::move` 转移所有权
- 使用 `std::forward` 完美转发
- 仅在需要移动语义时使用

```cpp
// 移动所有权
void TransferOwnership(std::unique_ptr<Foo> ptr);

// 完美转发
template <typename T>
void Wrapper(T&& arg) {
  Process(std::forward<T>(arg));
}

// 避免不必要的移动
std::string GetName() {
  std::string name = "foo";
  return name;  // Good: RVO 会自动优化
  // return std::move(name);  // Bad: 阻止 RVO
}
```

### 宏

- 避免定义宏，优先使用内联函数、枚举和常量
- 宏名使用项目特定前缀
- **禁止使用宏定义 C++ API 的部分内容**

```cpp
#define MYPROJECT_MAX_SIZE 100  // 如果必须使用宏
```

### nullptr

- 使用 `nullptr` 表示空指针，不使用 `NULL` 或 `0`

```cpp
int* ptr = nullptr;  // Good
int* ptr = NULL;     // Bad
int* ptr = 0;        // Bad
```

### sizeof

- 优先使用 `sizeof(varname)` 而非 `sizeof(type)`

```cpp
MyStruct data;
memset(&data, 0, sizeof(data));  // Good
memset(&data, 0, sizeof(MyStruct));  // Less preferred
```

---

## Android 项目特殊规则

Android 项目在遵循 Google C++ Style Guide 的基础上，有以下特殊约定：

### 函数命名

- 可使用小驼峰命名函数，以保持与 Android Native API 风格一致
- 例如：`writeData()`, `updateHeader()`, `getSampleRate()`

### RAII 资源管理

- 使用 RAII 管理资源（文件句柄、音频流等）
- 析构函数应标记为 `noexcept`

```cpp
class WAVFile {
public:
    WAVFile() = default;
    ~WAVFile() noexcept { close(); }
    
    // 禁止拷贝
    WAVFile(const WAVFile&) = delete;
    WAVFile& operator=(const WAVFile&) = delete;
};
```

### Android Log 使用

- 使用 `ALOGI`, `ALOGE`, `ALOGW` 等宏输出日志
- 定义 `LOG_TAG` 宏标识日志来源

```cpp
#define LOG_TAG "audio_test_client"

ALOGI("AudioRecord initialized successfully");
ALOGE("Failed to initialize AudioRecord");
```

### 禁止异常

- Android Native 代码禁止使用 C++ 异常
- 使用返回值或错误码表示错误

---

## 代码检查工具

使用 `cpplint.py` 检测风格错误：

```bash
cpplint.py my_file.cc
```

---

## 参考资源

- [Google C++ Style Guide (官方)](https://google.github.io/styleguide/cppguide.html)
- [Google C++ Style Guide (中文翻译)](https://zh-google-styleguide.readthedocs.io/)

---

## 总结要点

1. **头文件**：自包含、正确保护、合理包含顺序
2. **命名**：类型大驼峰、变量小写下划线、常量 k 前缀、静态成员 s 前缀
3. **格式**：80 字符行宽、2 空格缩进、一致的大括号风格
4. **类设计**：私有成员、explicit 构造函数、明确拷贝/移动语义
5. **函数**：短小精悍、明确输入输出
6. **禁止**：异常、RTTI、C 风格转换、using namespace
7. **推荐**：智能指针、const、constexpr、nullptr
8. **Android 项目**：函数可用小驼峰、RAII 资源管理、禁止异常
