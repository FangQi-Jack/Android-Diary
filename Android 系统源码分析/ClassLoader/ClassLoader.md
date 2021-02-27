# **Java 的 ClassLoader**
## ClassLoader 的类型
* Bootstrap ClassLoader

   C/C++ 实现的类加载器，用于加载指定的 JDK 的核心类库，如 java.lang，java.uti 等这些系统类。Java 虚拟机的启动就是通过 Bootstrap ClassLoader 创建一个初始类来完成的。它用来加载以下目录中的类库：
    * $JAVA_HOME/jre/lib
    * -Xbootclasspath 参数指定的目录
* Extensions ClassLoader

   Java 中的实现类为 ExtClassLoader，用于加载 Java 的拓展类，提供除了系统类之外的额外功能。用来加载以下目录：
    * $JAVA_HOME/jre/lib/ext
    * 系统属性 java.ext.dir 指定的目录
* Application ClassLoader

   实现类为 AppClassLoader，也可以称作 System ClassLoader。用来加载以下目录：
    * 当前程序的 Classpath 目录
    * 系统属性 java.class.path 指定的目录
* Custom ClassLoader

   自定义类加载器，通过继承 java.lang.ClassLoader 的方式来实现自定义类加载器。
## ClassLoader 的继承关系
![](https://github.com/FangQi-Jack/Android-/blob/main/Android%20%E7%B3%BB%E7%BB%9F%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/ClassLoader/ClassLoader%20%E7%9A%84%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB.png)
## 双亲委托模式
类加载器查找 Class 所采用的的是双亲委托模式，也就是首先判断该 Class 是否已经加载，如果没有则不是自身去查找而是委托给父加载器进行查找，这样依次向上递归，直到委托到最顶层的 Bootstrap ClassLoader，如果 Bootstrap ClassLoader 找到了该 Class，则直接返回，否则继续依次向下查找，如果没有找到则最后交由自身去查找。
![](https://github.com/FangQi-Jack/Android-/blob/main/Android%20%E7%B3%BB%E7%BB%9F%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/ClassLoader/ClassLoader%20%E7%9A%84%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%89%98%E6%A8%A1%E5%BC%8F.png)

采用双亲委托模式主要有两点好处：

* 避免重复加载
* 更加安全
## 何时开始类的初始化
1、创建类的实例

2、访问类的静态变量（除常量【被 final 修饰的静态变量】。原因：常量是一种特殊的变量，因为编译器把它们当做值（value）而不是域（field））。如果代码中有常量，编译器并不会生成字节码来从对象中载入域的值，而是直接把这个值插入到字节码中。

3、访问类的静态方法

4、反射

5、初始化一个类时，如果父类还没有初始化，则先初始化父类

6、虚拟机启动时，定义了 main 方法的那个类先初始化



子类调用父类的静态变量，子类不会被初始化。通过数组定义引用类，不会初始化。访问类的常量，不会初始化。
## 类初始化顺序
1.类从顶至底的顺序初始化，所以声明在顶部的字段的早于底部的字段初始化。

2.超类早于子类和衍生类的初始化。

3.如果类的初始化是由于访问静态域而触发，那么只有声明静态域的类才被初始化，而不会触发超类的初始化或者子类的。

4.初始化即使静态域被子类或子接口或者它的实现类所引用。

5.接口初始化不会导致父接口的初始化。

6.静态域的初始化是在类的静态初始化期间，非静态域的初始化时在类的实例创建期间。这意味这静态域初始化在非静态域之前。

7.非静态域通过构造器初始化，子类在做任何初始化之前构造器会隐含地调用父类的构造器，他保证了非静态或实例变量（父类）初始化早于子类

# **Android 中的 ClassLoader**
## ClassLoader 的类型
* **BootClassLoader**
    
    Android 系统启动时会通过 BootClassLoader 来预加载常用类，它是由 Java 代码实现的。它是 ClassLoader 的内部类，继承了 ClassLoader，是一个单例类，在应用程序中无法直接调用。
    
* **DexClassLoader**
    
    可以加载 dex 文件和包含 dex 的压缩文件。继承自 BaseDexClassLoader，方法都在 BaseDexClassLoader 中实现。
    
* **PathClassLoader**
    
    用来加载系统类和应用程序的类。继承自 BaseDexClassLoader，方法都在 BaseDexClassLoader 中实现。
## Android 8.0 中的 ClassLoader 的继承关系
![](https://github.com/FangQi-Jack/Android-/blob/main/Android%20%E7%B3%BB%E7%BB%9F%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/ClassLoader/Android%208.0%20ClassLoader%20%E7%9A%84%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB.png)
## ClassLoader 的加载过程
Android 中的 ClassLoader 同样采用双亲委托模式。ClassLoader 的 loadClass 方法定义在 ClassLoader 中：
```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
   // First, check if the class has already been loaded
   Class<?> c = findLoadedClass(name);
   if (c == null) {
       try {
           if (parent != null) {
               c = parent.loadClass(name, false);
           } else {
               c = findBootstrapClassOrNull(name);
           }
       } catch (ClassNotFoundException e) {
           // ClassNotFoundException thrown if class not found
           // from the non-null parent class loader
       }

       if (c == null) {
           // If still not found, then invoke findClass in order
           // to find the class.
           c = findClass(name);
       }
   }
   return c;
}
```
其中 findClass 需要子类来实现，在 BaseDexClassLoader 中：
```java
public BaseDexClassLoader(String dexPath, File optimizedDirectory, String librarySearchPath, ClassLoader parent) {
   super(parent);
   this.pathList = new DexPathList(this, dexPath, librarySearchPath, null);

   if (reporter != null) {
       reporter.report(this.pathList.getDexPaths());
   }
}
    ...
@Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
   List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
   Class c = pathList.findClass(name, suppressedExceptions);
   if (c == null) {
       ClassNotFoundException cnfe = new ClassNotFoundException(
               "Didn't find class \"" + name + "\" on path: " + pathList);
       for (Throwable t : suppressedExceptions) {
           cnfe.addSuppressed(t);
       }
       throw cnfe;
   }
   return c;
}
```
在 BaseDexClassLoader 的构造方法中创建了 DexPathList。在 findClass 方法中调用了 DexPathList 的 findClass 方法：
```java
/**
    * Finds the named class in one of the dex files pointed at by
    * this instance. This will find the one in the earliest listed
    * path element. If the class is found but has not yet been
    * defined, then this method will define it in the defining
    * context that this instance was constructed with.
    *
    * @param name of class to find
    * @param suppressed exceptions encountered whilst finding the class
    * @return the named class or {@code null} if the class is not
    * found in any of the dex files
    */
public Class<?> findClass(String name, List<Throwable> suppressed) {
    for (Element element : dexElements) {
       Class<?> clazz = element.findClass(name, definingContext, suppressed);
       if (clazz != null) {
           return clazz;
       }
    }
    
    if (dexElementsSuppressedExceptions != null) {
       suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
    return null;
}
```
遍历 Element 数组 dexElements，并调用 Element 的 findClass 方法，Element 是 DexPathList 的静态内部类：
```java
/*package*/ static class Element {
    /**
    * A file denoting a zip file (in case of a resource jar or a dex jar), or a directory
    * (only when dexFile is null).
    */
   private final File path;
   private final DexFile dexFile;
   private ClassPathURLStreamHandler urlHandler;
   private boolean initialized;
   /**
    * Element encapsulates a dex file. This may be a plain dex file (in which case dexZipPath
    * should be null), or a jar (in which case dexZipPath should denote the zip file).
    */
   public Element(DexFile dexFile, File dexZipPath) {
       this.dexFile = dexFile;
       this.path = dexZipPath;
   }

   public Element(DexFile dexFile) {
       this.dexFile = dexFile;
       this.path = null;
   }

   public Element(File path) {
     this.path = path;
     this.dexFile = null;
   }
    ...
    public Class<?> findClass(String name, ClassLoader definingContext, List<Throwable> suppressed) {
           return dexFile != null ? dexFile.loadClassBinaryName(name, definingContext, suppressed) : null;
    }
    ...
}
```
Element 内部封装了 DexFile，用于加载 Dex。在 findClass 方法中会调用 DexFile 的 loadClassBinaryName 方法：
```java
public Class loadClassBinaryName(String name, ClassLoader loader, List<Throwable> suppressed) {
   return defineClass(name, loader, mCookie, this, suppressed);
}
```
其中调用了 defineClass 方法：
```java
private static Class defineClass(String name, ClassLoader loader, Object cookie, DexFile dexFile, List<Throwable> suppressed) {
   Class result = null;
   try {
       result = defineClassNative(name, loader, cookie, dexFile);
   } catch (NoClassDefFoundError e) {
       if (suppressed != null) {
           suppressed.add(e);
       }
   } catch (ClassNotFoundException e) {
       if (suppressed != null) {
           suppressed.add(e);
       }
   }
   return result;
}
```
其中调用了 defineClassNative 方法，该方法是 Native 方法。

## BootClassLoader 的创建
在 ZygoteInit 的 main 方法中：
```java
public static void main(String argv[]) {
    ...
    try {
        ...
        preload(bootTimingsTraceLog);
        ...
    }
    ...
}
```
preload 方法中又调用了 ZygoteInit 的 preloadClasses 方法：
   ```java
/**
    * The path of a file that contains classes to preload.
    */
   private static final String PRELOADED_CLASSES = "/system/etc/preloaded-classes";
    ...

   /**
    * Performs Zygote process initialization. Loads and initializes
    * commonly used classes.
    *
    * Most classes only cause a few hundred bytes to be allocated, but
    * a few will allocate a dozen Kbytes (in one case, 500+K).
    */
   private static void preloadClasses() {
       final VMRuntime runtime = VMRuntime.getRuntime();
       InputStream is;
       try {
           is = new FileInputStream(PRELOADED_CLASSES);
       } catch (FileNotFoundException e) {
           return;
       }
    ...
       try {
           BufferedReader br = new BufferedReader(new InputStreamReader(is), 256);
           int count = 0;
           String line;
           while ((line = br.readLine()) != null) {
               // Skip comments and blank lines.
               line = line.trim();
               if (line.startsWith("#") || line.equals("")) {
                   continue;
               }
               try {
                   if (false) {
                       Log.v(TAG, "Preloading " + line + "...");
                   }
                   // Load and explicitly initialize the given class. Use
                   // Class.forName(String, boolean, ClassLoader) to avoid repeated stack lookups
                   // (to derive the caller's class-loader). Use true to force initialization, and
                   // null for the boot classpath class-loader (could as well cache the
                   // class-loader of this class in a variable).
                   Class.forName(line, true, null);
                   count++;
               } catch (ClassNotFoundException e) {
                   
               } catch (UnsatisfiedLinkError e) {
                   
               } catch (Throwable t) {
                   
               }
        ...
           }
       }
   }
   ```
preloadClasses 方法用于 Zygote 进程初始化时预加载常用类。Class 的 forName 方法：
   ```java
@CallerSensitive
   public static Class<?> forName(String name, boolean initialize, ClassLoader loader) throws ClassNotFoundException {
       if (loader == null) {
           loader = BootClassLoader.getInstance();
       }
       Class<?> result;
       try {
           result = classForName(name, initialize, loader);
       } catch (ClassNotFoundException e) {
           Throwable cause = e.getCause();
           if (cause instanceof LinkageError) {
               throw (LinkageError) cause;
           }
           throw e;
       }
       return result;
   }
   ```
在 Class 的 forName 方法中创建了 BootClassLoader，并将它传入了 classForName 方法中，该方法是 Native 方法。至此，BootClassLoader 就创建完成了。
## PathClassLoader 的创建
Zygote 进程启动 SystemServer 进程时会调用 ZygoteInit 的 startSystemServer 方法，其中，Zygote 通过 fork 自身创建了子进程（SystemServer）后，如果返回的 pid 为 0 则说明当前代码运行在新创建的 SystemServer 进程中，则会执行 handleSystemServerProcess 方法：
```java
private static void handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs) throws Zygote.MethodAndArgsCaller {
    ...
    if (parsedArgs.invokeWith != null) {
            ...
    } else {
       ClassLoader cl = null;
        if (systemServerClasspath != null) {
           cl = createPathClassLoader(systemServerClasspath, parsedArgs.targetSdkVersion);
           Thread.currentThread().setContextClassLoader(cl);
       }

       /*
        * Pass the remaining arguments to SystemServer.
        */
       ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
    }
}
```
调用了 createPathClassLoader 方法：

```java
static PathClassLoader createPathClassLoader(String classPath, int targetSdkVersion) {
     String libraryPath = System.getProperty("java.library.path");
     return PathClassLoaderFactory.createClassLoader(classPath,
                                                     libraryPath,
                                                     libraryPath,
                                                     ClassLoader.getSystemClassLoader(),
                                                     targetSdkVersion,
                                                     true /* isNamespaceShared */);

}
```

在 createPathClassLoader 中又调用了PathClassLoaderFactory 的 createClassLoader 方法：

```java
public static PathClassLoader createClassLoader(String dexPath,
                                                   String librarySearchPath,
                                                   String libraryPermittedPath,
                                                   ClassLoader parent,
                                                   int targetSdkVersion,
                                                   boolean isNamespaceShared) {
       PathClassLoader pathClassloader = new PathClassLoader(dexPath, librarySearchPath, parent);
       Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "createClassloaderNamespace");
   String errorMessage = createClassloaderNamespace(pathClassloader,
                                                    targetSdkVersion,
                                                    librarySearchPath,
                                                    libraryPermittedPath,
                                                    isNamespaceShared);
   if (errorMessage != null) {
       throw new UnsatisfiedLinkError("Unable to create namespace for the classloader " +
                                      pathClassloader + ": " + errorMessage);
   }
   return pathClassloader;
}
```

在 PathClassLoaderFactory 的 createClassLoader 方法中会创建 PathClassLoader。说明 PathClassLoader 是在 SystemServer 进程中采用工厂模式创建的。
