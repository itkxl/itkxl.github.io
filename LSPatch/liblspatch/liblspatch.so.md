## 一、概述
之前在介绍重打包的时候，在AppComponentFactory中加载了liblspatch.so,将控制权进行转交，下面对liblspatch.so这部分源码进行解析。

## 二、源码解析
代码位于patch-loader/src/main/jni下。

