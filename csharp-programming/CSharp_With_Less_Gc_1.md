# 使用 C# 但不使用 GC？前言篇

## 本篇摘要
1. 由于 GC 时机不可控，可能破坏特定类型的游戏体验，所以要减少 GC 使用甚至完全不使用；
2. 通过 `Marshal.AllocHGlobal` 和 `Marshal.FreeHGlobal` 分配和使用非托管内存；
3. 通过 `GCHandle` 混合使用托管类型和非托管类型；
4. 如何在非托管内存中构建复杂的 struct 对象


## 动机
某些游戏类型严格保证画面与逻辑的时延，例如格斗游戏、射击游戏等等。

此类游戏的玩家会预期游戏操作与游戏反馈的时延是稳定的，甚至是微小到不可察觉的，否则会带来很差的游戏体验甚至破坏游戏性。

因此，开发这类游戏通常会选用 C++ 等不带有运行时垃圾回收（以下简称 GC）机制的编程语言。因为 GC 的实际发生时机是不可能完全由游戏开发人员写下的代码来决定的，所以只能选择支持由开发人员完全管理内存的编程语言。

Unity 是非常流行的游戏开发引擎，而 C# 又是 Unity 引擎支持最全面的游戏脚本语言。众所周知，C# 是带有 GC 机制的编程语言，那么是否可能只使用 C# 而不使用 GC 或者不受 GC 严重影响呢？

## 可行性

### unsafe 代码和指针

在常见的需求下，C# 不能使用指针功能，所以只使用 C# 的游戏开发者是不能完全控制程序内存的，相当一部分的内存由 C# 的 GC 机制控制。

尽管在 unsafe 关键字修饰的代码块、方法和类之中能使用指针，但指针功能仅限于 C# 中的原生数据类型，以及全部成员都是原生数据类型的 struct。关于此限制，请参考[官方文档《Unsafe code, pointer types, and function pointers》](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/unsafe-code)。

如果要在这样的限制下进行实际的开发，开发效率或许还不如直接使用 C 语言。

### 混合使用托管内存与非托管内存

C# 依靠其运行时全面管理托管内存，还提供了 [ `Marshal.AllocHGlobal`](https://docs.microsoft.com/en-US/dotnet/api/system.runtime.interopservices.marshal.allochglobal?view=netstandard-1.1) 和 [`Marshal.FreeHGlobal`](https://docs.microsoft.com/en-US/dotnet/api/system.runtime.interopservices.marshal.freehglobal?view=netstandard-1.1) 两个 API，通过这两个方法可以让 C# 代码直接从非托管内存堆中分配和归还内存，其作用类似于 C 语言中的 `malloc` 和 `free` 函数。

事实上，在跨平台 dotnet 的实现代码中，*nix 平台上的 `AllocHGlobal` 也确实是调用 `malloc`。

在与非托管代码（例如外部 C 库）进行交互的时候，使用的也是非托管内存。

### 混合使用托管类型与非托管类型

C# 中也把对象类型分为托管类型和非托管类型。简单来说，`class`、`delegate` 和 `interface` 的实例对象，其内存必然位于托管堆上。但非托管类型的实例，则没有这样的限制——也就是说，非托管类型的对象可能位于托管内存堆上，也由可能位于非托管内存堆上。

但是，非托管类型也是有严格限制的。例如，struct 成员中只要有一个托管类型，那么这个 struct 就不满足非托管类型的定义，具体请参考[官方文档《Unmanaged types》](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/unmanaged-types)

在非托管内存中无法直接地构造托管对象，因为 GC 不会知道这个对象的存在。也就是说，像如下的 `struct`:

```csharp
public struct MyStruct {
    private int field1_;
    private object field2_;
}
```
如果 `MyStruct` 的实例分配在非托管堆上，那么我们可以期待运行中肯定会遇到各种奇怪的问题。

所以我们需要使用 [`GCHandle`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.gchandle?view=netstandard-1.1) 这个 C# 运行时提供的工具来告诉 GC，一个托管堆上的对象可能正在被非托管堆上的对象所引用，从而防止 GC 过早回收这个对象。

### sizeof 还是 SizeOf

正如在 C中为 `struct` 动态分配内存，需要用到 `sizeof` 运算符，由编译器告诉开发者需要分配的内存大小一样,如果我们需要在 C# 中为 `struct` 从非托管堆中分配合适大小的内存，我们也需要有一个 `sizeof` 运算符。

然而在 C# 中的 [`sizeof`运算符](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/sizeof) 只能在 `unsafe` 上下文之中使用，而且它还不能接受非托管类型作为参数，这大大地限制了它的能力。

[`Marshal.SizeOf`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.marshal.sizeof?view=netstandard-1.2#System_Runtime_InteropServices_Marshal_SizeOf__1) 也不是符合我们需要的 API，因为它返回的是在串行化目的下的占用内存的大小。而我们需要的是一个对象在计算目的下的内存占用（需要考虑例如内存对齐等复杂情况，参考 [SO 上的问答](https://stackoverflow.com/questions/63475796/marshal-sizeof-returning-unexpected-value)），这两者显然不一样。

其实 CIL 中直接就有指令可以提供任何类型的内存大小，然而直到 .net5 这个指令才被 C# 作为语言标准库功能提供给所有开发者，[`Unsafe.SizeOf<T>()`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.unsafe.sizeof?view=net-5.0#System_Runtime_CompilerServices_Unsafe_SizeOf__1)。当然，因为它只是对 CIL 指令的简单封装，因此我们在不支持 .net5 标准的环境中，总是可以[手写一个](https://github.com/DotNetCross/Memory.Unsafe/tree/master/src/DotNetCross.Memory.Unsafe)。而且， Unity 中也有相应的 [API](https://docs.unity3d.com/ScriptReference/Unity.Collections.LowLevel.Unsafe.UnsafeUtility.SizeOf.html)

### 返回引用特性

关于这个特性的详细说明，请参考[官方文档](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/ref-returns)。

在 C# 7 之前的语言规范中，一个 `struct` 除非作为数组中的一个元素并和数组一起返回，否则在作为返回值时 `struct` 总是被整个复制到返回位置。换句话说，即使我们设计了一个指针类：

```csharp

public struct Point3D {
    public double X;
    public double Y;
    public double Z;
}

public unsafe struct Point3DPointer {
    private Point3D* dataPtr_;
    public T Get()
        => *(this.dataPtr);
}

public static void UpdateX<T>(this Point3DPointer pointer, double x)
    => pointer.Get().X = newValue;  // 编译错误

```

我们也无法对指针所指的对象（例子里的 `Point3D`）进行修改，最多只能整个替换，显然这么做很可能会带来性能损失——而且，既然要整个替换，那为什么还要用指针呢？

而在新特性(以及 `Unsafe` 类)的加持下，我们可以这么写：

```csharp

public struct Pointer<T> where T : struct {
    private IntPtr data_;

    /*** 注意 ref 关键字  ***/
    public ref T GetRef()
        => unsafe { return ref Unsafe.AsRef<T>((void*)data_); }
}

public static void UpdateX<T>(this Pointer<Point3D> pointer, double x)
    => pointer.GetRef().X = newValue;  // 没问题

```

也即，在 C# 7 之后，我们可以使用泛型指针用来指向非托管内存上的对象，以达到表示和传递引用的目的。其形式就类似于使用 class 来表示托管内存上的托管对象。

## 小结

至此，我们已经大致上可以预见，如何使用 C# 来使用和管理非托管堆上的非托管对象。
首先智能指针大致上有这样的设计：

```csharp
public readonly struct Pointer<T> {
    private readonly IntPtr addr_;

    internal Pointer<T>(IntPtr addr)
        => this.addr_ = addr;

    public ref T GetRef()
        => unsafe { return ref Unsafe.AsRef<T>((void*)this.addr_); }

    public ref readonly T GetRefReadOnly()
        => ref this.GetRef();
}
```

### 创建

通过包装 `Marshal.AllocHGlobal` API，来分配非托管内存，然后通过智能指针（例如`Pointer<T>`）的工厂方法，构建非托管对象，并返回智能指针，代码雏形如下：

```csharp

public interface IConstructFn<T> {
    void Construct(ref T position);
}

// 工厂方法的集合
public static class Pointer {
    // 其中一个工厂方法
    public static Pointer<T> Create<T, C>(C construct)
        where C : IConstructFn<T>
    {
        var addr = Marshal.AllocHGlobal(Unsafe.SizeOf<T>());
        ref T position = Unsafe.AsRef<T>((void*)addr);
        construct.Construct(ref position)
        return new Pointer<T>(addr);
    }
}

```
这样的 `Pointer<T>` 因为只是对真实指针的一个包装，因此在作为参数传递时和返回值时，几乎不会影响性能。

### 使用

现在假设我们要借助 `Pointer<T>` 来传递非常大的结构体对象，进入某个函数进行一定的计算处理，使用起来的代码大致像这样：

```csharp
public struct MyHugeStruct
{
    public ulong u64field;
    public double f64field;
    public uint u32field;
    public int i32field;
    public Pointer<MyHugeStruct> nextPtr;
}

public static ulong CalculateMyHugeStruct(Pointer<MyHugeStruct> ptr)
{
    ref readonly var myHugeStruct = ref ptr.GetRefReadonly();
    // 计算
    return myHugeStruct.u64field;
}

```

### 销毁

对超出生命周期的对象进行销毁，也属于智能指针的职责。

销毁一个动态创建的（即从非托管堆上分配的）对象，通常有引用计数策略和独占两种，类似于 C++ 中的
`std::shared_ptr<T>` 和 `std::unique_ptr<T>`。

这里限于篇幅，不另外介绍。但要实现一个具有使用价值的智能指针，必须考虑这一点。


