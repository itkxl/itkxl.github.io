## 一、基础知识
### 1.1 AAPT
AAPT是Android SDK中的一个工具，用于处理应用资源并生成可以打包到APK中的二进制文件。其在构建过程中起着直观重要的作用，不仅可以编译资源，还可以为资源分配ID，并生成R.java文件。

源码路径 /frameworks/base/tools/aapt/

从AAPT工具编译资源的过程可知，aapt工具会编译assets、res和AndroidManifest.xml这三类资源。这里我们主要来看前两者。
- assets:这类资源放在工程的根目录assets下。里面存放的是一些原始文件，比如音频文件，MP4等。这类文件会原封不动的打包进apk中，不会进行压缩，同时其也不会生成Resource ID，如果要访问这类资源，需要通过AssetManager来获得，比如
```java
AssetManager asset = getAssets();
InputStream is = asset.open("文件名");
```
assets类资源文件与杰西莱要介绍的res目录下的资源的区别在于，后者是通过Resource获得，并且编译后会生成Resource id.

- res:这类资源放在源代码工程res目录下，此目录可以放很多种资源，除了res/raw目录下的资源外，其他的都会编译或处理。raw目录下的处理方式和asset处理方式类似，也是不做处理的打进apk中，不过此处会生成Resource id。日常开发，可以铜鼓Resource id来查找资源，比如
```java
Resource resource = getResource();
resource.getString(R.string.test);
```

### 1.2 Resource id
Resource id是一个整数值，经过aapt编译后生成R.java文件，同时生成资源索引表resourcr.arsc,该文件也是一个二进制格式的文件，程序运行的时候根据R.java中的Resource id去resource.arsc中查找相关的资源。


## 二、资源初始化过程
## 三、资源查找、解析过程
## 四、插件化中资源加载的处理







