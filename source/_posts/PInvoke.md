title: 关于C#中的平台调用P/Invoke
tags: [C#,.Net,dotnet]
categories: [学习]

date: 2023-10-20 10:49:00
---
今天来回顾下P/Invoke中的各种细节，加深下理解和记忆。

### 概念

P/Invoke，全名Platform Invocation Services，是.NET框架中的一个特性。允许托管代码（比如C#，VB等.NET语言编写的代码）调用非托管的DLL函数，包括Win32 API库和其它我们自己编写的C/C++动态链接库中的函数。

P/Invoke的名字就来自“Platform Invocation”，意思是调用平台函数，通常是调用C/C++的本地函数库。

在C#中，我们使用[DllImport]特性来导入DLL，然后声明对应的**静态方法**就可以使用了。例如调用windows api中的一个函数MessageBox，我们可以这样声明：

```C#
[DllImport("user32.dll", CharSet = CharSet.Unicode)]
public static extern int MessageBox(IntPtr hWnd, String text, String caption, uint type);
```

然后我们就可以像普通C#函数一样来使用MessageBox了：

```C#
MessageBox(new IntPtr(0), "Hello World!", "Hello Dialog", 0);
```

<!--more-->

### 原理

一句话：

**主流高级编程语言实际上并不互相支持或兼容，但他们的共同点在于都支持C。以C为桥梁（或胶水），各语言之间可以实现一定程度的互通。**

高级编程语言大都有各自的特性以及存储管理方式等，各语言之间的差异决定了不可能支持互相直接的或进行复杂的互操作。但大部分主流编程语言都有办法与C语言互动，这主要是因为C语言的普遍性和效率，以及它在操作系统和硬件级别的广泛应用。以下是一些主流语言如何与C语言交互的概览：

0. C++：完全兼容C。

1. C#：使用P/Invoke或者C++/CLI来调用C的函数库。

2. Java：使用Java Native Interface (JNI)来调用C和C++的代码。

3. Python：可以使用ctypes或cffi库来调用C的DLL，或者使用Cython、SWIG等工具将C代码包装成Python模块。

4. Go：使用cgo可以在Go代码中直接调用C的函数。

5. Rust：Rust可以使用其Foreign Function Interface (FFI)来调用C的函数。

6. Ruby：Ruby C扩展允许你在Ruby程序中调用C的函数。

7. JavaScript：在Node.js环境中，可以使用Node Foreign Function Interface (node-ffi)来调用C的函数库。在前端环境，WebAssembly技术可以让前端运行C和C++编译出来的代码。

C#通过unsafe代码可以直接操作非托管代码（包括指针等资源）。而我们经常将C/C++放在一起说，是因为C++完全兼容C，是C的一个超集。从面向对象的角度讲：

**C是接口，则C++完全实现了C。若需要使用一个C资源，给我一个C++资源我也一样可以使用。**

正是因为这个原因，C#完成了与C++库的互操作。所以我们说C#支持与C/C++的互操作。

### 声明

我们知道，C语言中提供给外部掉调用的方法是这么声明的：

```C
extern void myFunction();
```

C语言的声明简洁明了：`extern`关键字表示该方法可以被其他源文件或库调用，后面紧跟的就是方法签名了。

再看看C++是怎么声明的：

```C++
extern "C" __declspec(dllexport) void __stdcall myFunction()；
```

多了三个部分：

1. "C"：可选值为"C"和"C++"，分别表示以两种不同的方式编译链接该方法。由于C++的导出方式会重命名该方法从而导致调用方找不到该方法，因此对于P/Invoke，这里只能写"C"；
2. __declspec(dllexport)：双下划线表示该关键字是编译器相关的关键字，全称：declaration specification，声明规范。其参数可为：
   - `dllexport`：将后续的声明的函数或变量导出，被其它DLL或程序所引用；
   - `dllimport`：在编译时指定链接器从另一个DLL中获取函数或变量；
   - `noinline`：阻止编译器对指定的函数进行内联优化；
   - `thread`：将变量声明为线程局部存储（Thread Local Storage，简称TLS），即每个线程一份存储副本；
   - `deprecated`：标记函数或类型为过时，如果它们被使用，编译器将发出警告。
3. __stdcall：双下划线表示该关键字是编译器相关的关键字，这个参数的另一个可选值为`__cdelc`，也可留空（根据平台不同会默认指定为其中一个，一般为后者）。
   - `__cdecl`：推荐方式。这种方式最大的特点在于**参数由调用者清除**。或者换个说法，如果希望兼容各种C代码，希望API参数更加灵活多变（譬如printf函数），就选它；
   - `__stdcall`：这种方式最大的特点在于**参数由被调用者清除**。Win32 API通常都是使用这种方式，因此，如果你的API大量调用了Win32 API，或者是由VC++编写的（通常就肯定与Win32 API有关），就选它。

### 数据类型

由于P/Invoke实质上是对C接口的兼容，因此不需要考虑C++的复杂类型或特性。

除了双方对基元类型的兼容外，需要着重了解下结构体类型和字符（串）类型的传递。

#### 结构体

若传入参数为结构体，则在C#中需要定义对应的结构体，并且需要使用结构体特性[StructLayout.Sequential]进行标注，告诉编译器该结构体各字段应顺序存储以保持与C结构体的一致性。

此外，如果传入参数为联合体Union类型，也可通过定义结构体并对结构体和其属性分别使用特性[StructLayout.Explicit]和[FieldOffset(0)]来达到目的。

#### 字符数据

关于字符数据，有一点需要了解：

**C#中的char和string类型实际上是使用utf-16进行编码存储的，而C/C++中的char是采用ASCII编码进行存储的。**

了解了这一点，就应该知道，在需要传递char、char*数据的地方，我们理论上应该传入的是byte和byte数组的首地址，而不应该直接传递C#中的char和string。当然，如果你能够做到融汇贯通，那么你将CharSet设置为ANSI，并传入一个C#的string，也一样是可以的：

```C#
[DllImport("user32.dll",  CharSet = CharSet.ANSI)]
public static extern int MessageBox(IntPtr hWnd, char* text, String caption, uint type);
```

既然C#中的char和string不能直接对应C/C++中的char和char*，那应该对应什么呢？通常情况下，答案是`wchar_t`。

`wchar_t`是C和C++的基础数据类型之一，表示宽体字符类型，每个字符占用2个字节（某些系统可能为4个字节），并采用utf-16（某些系统可能采用utf-32）进行编解码。因此，如果需要传递的参数为wchar_t或wchar_t*，则可以直接将C#的char或string传递过去。

记住，无论在哪种语言中：

**字符或字符串都是以一个或一串字节数据的形式存储的，其本质是一样的，不同编程语言或不同命名方式相当于对这些字节数据采用了不同的解码方式而已。**

### 最后

自从跨平台的.net core问世之后，C++/CLI这种无法跨平台的托管C++日渐式微，VS2022的C++默认安装已经不提供其项目或库的模板了，而且，C++/CLI的设计初衷是为了使C++编码人员无痛转入.Net阵营，而对于对于我等.Net原住民，目前来说，P/Invoke这种历久弥坚的老技术再次成为了事实上的最优解。实际上，P/Invoke中还存在很多细节，有我知道但不便继续赘述的，也有我目前还不了解的。欢迎大家留言或提问，一起学习一起成长吧~
