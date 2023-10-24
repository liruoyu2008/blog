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
   - `__cdecl`：这种方式最大的特点在于**参数由调用者清除**。如果不理解，那么当你使用C++编写库函数时，或者希望API参数更加灵活可变（譬如printf函数）时，就选它；
   - `__stacall`：这种方式最大的特点在于**参数由被调用者清除**。Win32 API通常都是使用这种方式，因此，如果你的API大量调用了Win32 API或者可能会被当作回调函数给Win32 API调用时，亦或者可能会被VB、Delphi等调用时（它们可能仅支持stdcall的方式），就选它。

### 数据类型

由于P/Invoke实质上是对C接口的兼容，因此不需要考虑C++的复杂类型或特性。

除了双方对基元类型（bool类型除外）的兼容外，需要着重了解下结构体类型和字符（串）类型的传递。

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

**注意**，由于C#的对常量字符串静态存储的特殊处理，所以对于字符串的传入与传出需要格外注意两点：

- 传入：使用适当的编码方式时，C#内可以直接定义`string str`这样的形参传入到C++，C#会自动将该字符串封送一份副本到非托管内存而不是像其他引用类型（如`ref int x`）那用直接传递原始字符串的地址，所以在C++内修改该字符串实际上也只是修改了副本，托管堆内的原始`string`依然没有变化。若真的希望修改托管堆内的内容，可以传递`StringBuilder`，C#的`StringBuilder`更加类似C++内字符指针的形态（注意：要为`Stringbuilder`分配足够容纳修改后内容的空间）。
- 传出：无论何时，接收C++传出的字符串都不能用`string`。一般正确的做法是用形如`IntPtr`这样的指针形式，然后通过`MarShal`的静态方法去读取字符串。

### 封送规则

一般情况下，我们需要考虑的数据封送情况分为三种：基元类型、字符类型、结构体类型；基元类型两边基本都是一致的，可以直接传递；字符类型分为Ansi和Unicode，通过通过指定**方法级别**的**[CharSet]**特性分情况传递；结构体则只需两边保持一样的结构以定义即可传递；

如果碰到特殊情况怎么办？还可以使用**参数级别**的**[MarshalAs]**特性来指定更细致的数据封送规则。

#### MarshalAs

`MarshalAs` 特性告诉运行时如何在非托管代码和托管代码之间封送特定的参数或返回值。尽管在很多情况下，即使你没有指定 `MarshalAs` 特性，代码看起来仍然可以正常运行，但是这并不意味着你就可以忽略它。

使用 `MarshalAs` 特性有两个主要的理由：

1. **控制数据在托管和非托管之间的表示方式**：C# 和 C++ 数据类型的表示并不总是一一对应的。例如，C# 的 `bool` 类型在内存中占用 4 字节，而在某些 C++ 编译器中，`bool` 只占用 1 字节。因此，在这种情况下，就需要使用 `[MarshalAs(UnmanagedType.Bool)]` 来保证在托管和非托管代码之间正确封送布尔类型。

2. **提供更广泛的类型转换支持**：`MarshalAs` 可以帮助你处理一些自动封送(Marshal)规则无法处理或者需要特殊处理的情况。例如，托管与非托管字符或字符串之间的封送，尤其是在不同的字符编码环境中，或者在封送指针或者引用时，或者多参数采用不同字符封送规则时，这时候 `MarshalAs` 就非常有用。

总之，在使用PInvoke时，使用 `MarshalAs` 属性可以提供更明确的控制和更罕见类型的支持，并且可以帮助避免可能发生的问题和错误。

#### UmanagedType

`MarshalAs`需指定一个非托管数据类型`UnmanagedType`，它 是一个枚举，，它定义了如何在托管代码和非托管代码之间进行数据封送。这是其各个值的概述：

- **Bool**：在非托管代码中表示为 `BOOL` 类型（一个字节）。

- **I1**：表示在非托管代码中表示为有符号的 8 位整数。

- **U1**：表示在非托管代码中表示为无符号的 8 位整数。

- **I2**：在非托管代码中表示为有符号的 16 位整数。

- **U2**：在非托管代码中表示为无符号的 16 位整数。

- **I4**：在非托管代码中表示为有符号的 32 位整数。

- **U4**：在非托管代码中表示为无符号的 32 位整数。

- **I8**：在非托管代码中表示为有符号的 64 位整数。

- **Struct**：对象表示为一个非托管结构。

- **Interface**：表示为一个接口。

- **SafeArray**：表示为一个 `SAFEARRAY`。

- **ByValArray**：表示为一个内联数组。

- **LPStr**：表示在非托管代码中表示为一个指向 ANSI 字符串的指针。

- **LPWStr**：表示在非托管代码中表示为一个指向宽（Unicode）字符串的指针。

- **LPTStr**：表示在非托管代码中表示为一个指向平台-依赖的字符串（根据编译时的字符集选择 ANSI 或 Unicode）的指针。

以上并不全是所有选项，还有其他一些更先进或者具有特殊用途的 `UnmanagedType` 枚举值。具体信息，你可以查看 [.NET docs](https://docs.microsoft.com/zh-cn/dotnet/api/system.runtime.interopservices.unmanagedtype) 上的相关文档。

### 示例

[点我查看](https://github.com/liruoyu2008/PInovke)

### 最后

自从跨平台的.net core问世之后，C++/CLI这种无法跨平台的托管C++已经逐渐式微，VS2022的C++默认安装已经不提供其项目或库的模板了，而且，C++/CLI的设计初衷是为了使C++编码人员无痛转入.Net阵营，而对于对于我等.Net原著名，目前来说，P/Invoke这种历久弥坚的老技术再次成为了事实上的最优解。实际上，P/Invoke中还存在很多细节，有我知道但不便继续赘述的，也有我目前还不了解的。欢迎大家留言或提问，一起学习一起成长吧~
