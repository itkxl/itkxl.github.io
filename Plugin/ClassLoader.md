## 一、概述
在Android插件化中，ClassLoader是一个核心组件，它负责加载类（Class）文件，使得App可以动态的加载并执行Java代码。理解Android中的"ClassLoader"对于理解插件夸架构至关重要。

## 二、类加载器基础

### 2.1 类加载器的定义和作用
类加载器(Classloader)是Java中用于加载类(.class)文件到虚拟机（JVM）中的部分。在Java,每当我们使用到某个类而这个类还没加载到内存中时，JVM会通过类加载器区加载该类。类加载器不仅仅负责加载类，还负责将这些字节码转换成在JVM内可以运行的Java类对象。

### 2.2 Java的标准类加载机制
Java的类加兹安机制包括加载、链接（验证、准备和解析）和初始化三个步骤：
- 加载：这个是类加加载过程的第一步，涉及查找字节流(从.class文件中提取)并创建类对象
- 链接：在链接阶段，类加载器执行验证、准备、和解析三个动作。验证是为了确保被加载类的正确性，准备则是为类的静态变量分配内存，并初始化其默认值，解析则涉及到把类中符号引用转换为直接引用。
- 初始化：此阶段是为了标记为常量的字段赋值，以及执行静态代码块。

### 2.3 双亲委托机制
双亲委托机制是Java类加载体系中的一个核心概念。它定义了类加载器在加载类的行为规范。

#### 工作原理
在双亲委托的模型下，类加载器在尝试加载任何类之前，首先会将加载任务委托给其父类加载器去执行。这个过程会递进执行，即每个类加载器都会这样做，直到最顶层的启动类加载器。只有当父类加载器无法完成加载任务时（即它没有找到对应的类），这类加载器才会尝试自己去加载这个类。

#### 目的与好处
- 避免类重复加载：通过委托父加载器加兹安，可以确保所有的使用的类至多被加载一次，避免在JVM内存中出现多分同一个类的字节码。
- 安全性：由于系统类库由启动类加载器加载，这样可以防止用户自定义的类替代标准Java API，从而产生潜在的安全威胁。

## 三、Android中的ClassLoader

- ClassLoader 
    
核心方法loadClass体现了双亲委托的工作原理。
```java
  protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            //1、检查本地是否已经加载过，加载过，直接从内存中获取
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    //2、若存在父加载器，根据双亲委托机制，优先让父类进行加载
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        //3、如果父加载器不存在，说明到达顶层了，使用BootstrapClassClassLoader进行加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                }

                if (c == null) {
                    //4、如果父加载器没有找到，则尝试从当前ClassLoader进行查找
                    c = findClass(name);
                }
            }
            return c;
    }
```
在Android的实现中findBootstrapClassOrNull方法是直接返回了空。  
具体原因没有找到资料解释，个人猜测可能是和每次App启动的时候，都是Android通过fork的方式完成，这种状态下Android的核心库（如'androd.'和'java.'等）相当于已经加载好了，不需要单独的BootstrapClassLoader来单独处理这些类的加载，所以简化了这部分内容。 

 如果有同学了解原因，可以一起交流一下。
- BootClassLoader  
启动类加载器，用于加载Zygote进程已经预先加载的基本类。这个是ClassLoader基类中的内部类，应用程序无法直接访问。个人理解这个扮演了普通JVM的BootstrapClassLoader的角色。
- BaseDexClassLoader、DexClassLoader、PathClassLoader  
DexClassLoader、PathClassLoader继承自BaseDexClassLoader,低版本中(Dalvik虚拟机)，使用PathCalssLoader如果不是data下的文件，会报错。在Android高版本中，两者并无本质差别。

    - BaseDexClassLoader是没有覆写loadClass方法，说明其是遵循双亲委派机制的
    - DexPathList:包含了应用程序Dex文件的路径，这个路径可以是Apk、Dex、Jar。其负责管理这些文件，并在需要时从这些文件中查找和加载类。
    - 可以用于加载动态资源。
    - BaseDexClassLoader核心方法findClass 

```java
/**
 * 仅部分核心代码
 */
public class BaseDexClassLoader extends ClassLoader {

    /**
     * dexPath:加载的Apk/Dex/Jar的路径
     * parent：使用的父加载器
     */
    public BaseDexClassLoader(String dexPath,
            String librarySearchPath, ClassLoader parent, ClassLoader[] sharedLibraryLoaders,
            ClassLoader[] sharedLibraryLoadersAfter,
            boolean isTrusted) {
        super(parent);
        ...
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);
        ...
    }

     @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        //先从shareLibraryLoaders中尝试加载(非重点)
        //关于shareLibary可以参考一下https://developer.android.com/guide/topics/manifest/uses-library-element?hl=zh-cn，这里不做展开
        if (sharedLibraryLoaders != null) {
            for (ClassLoader loader : sharedLibraryLoaders) {
                try {
                    return loader.loadClass(name);
                } catch (ClassNotFoundException ignored) {
                }
            }
        }

        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        //从DexPathList中查找class
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c != null) {
            return c;
        }
        ...
        return c;
    }
}
```
- SystemClassLoader  
ClassLoader内部类，用于加载系统的Native so,通过打印对象
```java
Log.e("kxl",ClassLoader.getSystemClassLoader().toString());

//输出内容
dalvik.system.PathClassLoader
[DexPathList[[directory "."],
nativeLibraryDirectories=[/system/lib64, /system_ext/lib64, /system/lib64, /system_ext/lib64]]]
```
可以看到其DexPathList的directory为空，即没有相关dex被处理，但是nativeLibraryDirectories指向了/system下，用于加载对应目录下的so文件

- InMemoryDexClassLoader
Android8.0之后添加的ClassLoader,可以直接从内存中加载Dex。日常接触比较少，在之前了解加固相关内容的时候看到了这个加载器。
    - 安全性：将Dex文件加载到内存中，解密后，直接使用InMemoryDexClassLoader加载解密完成的Dex文件，减少了一次Dex落盘，使得一些脱壳工具不能通过简单的监听存储相关Api来获取解密后的Dex。
    - 性能：直接从内存中加载，减少了文件I/O的依赖，对性能有一定提升。
```java
public final class InMemoryDexClassLoader extends BaseDexClassLoader {
    /**
     * 直接从内存中加载Dex
     */
    public InMemoryDexClassLoader(@NonNull ByteBuffer @NonNull [] dexBuffers,
            @Nullable String librarySearchPath, @Nullable ClassLoader parent) {
        super(dexBuffers, librarySearchPath, parent);
    }

    public InMemoryDexClassLoader(@NonNull ByteBuffer dexBuffer, @Nullable ClassLoader parent) {
        this(new ByteBuffer[] { dexBuffer }, parent);
    }
}
```

- DelegateLastClassLoader  
在前面介绍的ClassLoader的工作原理中，有讲解过双亲委派机制。DelegateLastClassLoader比较特殊，与其正好相反，首先尝试自己加载指定的类，只有在自己无法加载的时候，才委托给父类进行加载。这个设计在插件化架构设计中非常重要，因为它可以允许插件或模块覆盖主应用中的类定义，实现更高的灵活性。

```java
  @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        Class<?> cl = findLoadedClass(name);
        if (cl != null) {
            return cl;
        }

        //最基础的类交由BootClassLoader加载，保证基本的稳定性
        try {
            return Object.class.getClassLoader().loadClass(name);
        } catch (ClassNotFoundException ignored) {
        }


        //优先尝试自己是否可以加载
        ClassNotFoundException fromSuper = null;
        try {
            return findClass(name);
        } catch (ClassNotFoundException ex) {
            fromSuper = ex;
        }

        try {
            return getParent().loadClass(name);
        } catch (ClassNotFoundException cnfe) {
            //最佳无法加载，异常是报在当前的ClassLoader中，这部分整体的设计预期
            throw fromSuper;
        }
    }
```

## 四、插件化中类加载策略