# **Dalvik 和 ART**
# Dalvik 虚拟机
## DVM 和 JVM 的区别
**1、基于的架构不同**
JVM 基于栈，所需的指令更多。DVM 基于寄存器，指令更紧凑，更简洁。
**2、执行的字节码不同**
JVM 执行的是 .class 文件和 jar 文件中获取的相应的字节码，而 DVM 会将所有 .class 文件转换为一个 .dex 文件，然后从 .dex 文件读取指令和数据。
**3、DVM 允许在有限的内存中同时运行多个进程**
DVM 进过优化，允许在有限的内存中同时运行多个进程，Android 中的每一个应用都运行在一个 DVM 实例中，每个 DVM 实例都运行在独立的进程空间中。
**4、DVM 由 Zygote 创建和初始化**
当系统需要创建一个应用程序进程时 Zygote 会 fork 自身，快速的创建和初始化一个 DVM 实例。Zygote 也是一个 DVM 进程。
**5、DVM 有共享机制**
DVM 有预加载-共享机制，不同应用之间在运行时可以共享相同的类，拥有更高的效率。JVM 不存在共享机制，每个程序都是彼此独立的。
## DVM 的运行时堆
DVM 的运行时堆采用标记-清除算法进行 GC，它由两个 Space 以及多个辅助数据结构组成，两个 Space 分别是 zygote space 和 Allocation Space。Zygote Space 用来管理 Zygote 进程在启动过程中预加载和创建的各种对象，不会触发 GC，Zygote 进程和应用程序进程会共享 Zygote Space。在 Zygote 进程 fork 第一个子进程前，会把 Zygote Space 分为两部分，原来以及被使用的那部分叫 Zygote Space，未使用的部分叫 Allocation Space，以后的对象都会在 Allocation Space 上进行分配和释放，Allocation Space 不是进程间共享的。此外还包含：
* Card Table：用于 DVM Concurrent GC，当第一次进行垃圾标记后，记录垃圾信息。
* Heap Bitmap：有两个，一个用来记录上次 GC 存活的对象，一个用来记录本次 GC 存活的对象。
* Mark Stack：在 GC 的标记阶段使用，用来遍历存活的对象。

# ART 虚拟机
## ART 和 DVM 的区别
1、DVM 在应用每次运行时，字节码都需要通过 JIT 编译器编译为机器码，运行效率低。而在 ART 虚拟机中，系统在安装应用程序时会进行一次 AOT（预编译），将字节码预先编译成机器码并存储在本地，程序运行时就不需要再执行编译了，运行效率大大提升，耗电量也会降低。Android 7.0 版本后，ART 虚拟机也加入了 JIT，作为 AOT 的补充，在应用程序安装时并不会将所有字节码编译成机器码，而是在运行中将热点代码编译成机器码。
2、DVM 时 32 位的，ART 是 64 位的并兼容 32 位。
3、ART 对垃圾回收机制进行了改进。比如更频繁的执行并行垃圾收集，将 GC 暂停有 2 次减少为 1 次。
4、ART 的运行时堆空间划分和 DVM 不同。
## ART 的运行时堆
ART 采用了多种垃圾收集方案，每个方案会运行不同的垃圾收集器，默认采用 CMS(Concurrent Mark-Sweep)方案，该方案主要使用了sticky-CMS 和 partial-CMS。不同的 CMS 方案，ART 的运行时堆的空间也会有不同的划分，默认由 4 个 Space 和多个辅助数据结构组成，4 个 Space 分别是Zygote Space、Allocation Space、Image Space、Large Object Space。前两者与 DVM 中相同，Image Space 用来存放一些预加载类，Large Object Space 用来分配一些大对象。Image Space 也是进程间共享的。此外还包含两个 Mod Union Table，一个 Card Table，两个 Heap Bitmap，两个 Object Map，三个 Object Stack。

# DVM 和 ART 的诞生
init 启动 Zygote 时会调用 app_main.cpp 的 main 函数：
```
int main(int argc, char* const argv[]) {
    ...
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    }
    ...
}
```
start 函数的具体实现在 AppRuntime 的父类 AndroidRuntime 中，在 start 方法中会调用 startVm 函数创建 JVM，并为其注册 JNI 方法，并调用 jni_invocation 的 init 函数：
```
bool JniInvocation::Init(const char* library) {
    #ifdef _ANDROID_
        char buffer[PROP_VALUE_MAX];
    #else
        char* buffer = NULL;
    #endif
        library = GetLibrary(library, buffer);
        const int kDlopenFlags = RTLD_NOW | RTLD_NODELETE;
        handle = dlopen(library, kDlopenFlags);
    ...
}
```
其中调用了 GetLibrary 函数：
```
#ifdef __ANDROID__
static const char* kLibrarySystemProperty = "persist.sys.dalvik.vm.lib.2";
static const char* kDebuggableSystemProperty = "ro.debuggable";
#endif
static const char* kLibraryFallback = "libart.so";

template<typename T> void UNUSED(const T&) {}

const char* JniInvocation::GetLibrary(const char* library, char* buffer) {
#ifdef __ANDROID__
const char* default_library;

char debuggable[PROP_VALUE_MAX];
__system_property_get(kDebuggableSystemProperty, debuggable);

if (strcmp(debuggable, "1") != 0) {
// Not a debuggable build.
// Do not allow arbitrary library. Ignore the library parameter. This
 // will also ignore the default library, but initialize to fallback
 // for cleanliness.
library = kLibraryFallback;
default_library = kLibraryFallback;
} else {
// Debuggable build.
// Accept the library parameter. For the case it is NULL, load the default
// library from the system property.
if (buffer != NULL) {
if (__system_property_get(kLibrarySystemProperty, buffer) > 0) {
default_library = buffer;
} else {
default_library = kLibraryFallback;
}
} else {
// No buffer given, just use default fallback.
default_library = kLibraryFallback;
}
}
#else
UNUSED(buffer);
const char* default_library = kLibraryFallback;
#endif
if (library == NULL) {
library = default_library;
}
return library;
}
```
其中 persist.sys.dalvik.vm.lib.2 是一个系统属性，取值可是 libdvm.so 或者 libart.so。可以看到最终返回了 library，它代表了 libdvm.so 或者 libart.so，在 init 函数中，会调用 dlopen 函数来加载 GetLibrary 的返回结果，因此，JniInvocation 的 init 函数的主要作用是初始化 ART 或者 DVM 环境，初始化完毕后会调用 startVm 函数启动相应的虚拟机。
**ART 或 DVM 虚拟机在 Zygote 进程中创建和初始化，因此 Zygote 进程就持有了 ART 和 DVM 实例，而后通过 Zygote 进程 fork 创建的应用程序进程也都得到了 ART 或 DVM 实例。**

