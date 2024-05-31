## 一、概述
之前在介绍重打包的时候，在AppComponentFactory中加载了liblspatch.so,将控制权进行转交，下面对liblspatch.so这部分源码进行解析。

## 二、源码解析
代码位于patch-loader/src/main/jni下。






1、通过反射找到AppComponentFactory，找到创建的dex，读取dex的数据，通过类加载器加载dex
2、从类加载器中找到加载的这个dex（类 方法）做了9.0的反射限制（说明dex中大量反射限制的调用）
3、动态注册了待使用的Native方法，xposedHookandFind
4、反射查找LSPApplication,找到类中的onLoad方法
5、移交控制权


initArtHooker 通过Hook的方式禁止 dex2oat

dex授信
resourceHook

HookBridge.java中native方法
NativeApi
DexParser 


到了LSPApplication.onLoad