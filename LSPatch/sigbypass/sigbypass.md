## 一、引言
前面介绍了LSPosed的重打包能力，重打包的同时进行了重新签名，如果应用内部有签名校验的逻辑，则就会使App无法正常运行，下一个要解决的问题就是签名校验。
接下来会从Native和Java两个视角来看LSPosed过签名校验的方式。
## 二、签名信息获取

一般来说，我们获取签名信息都是通过PackageManager直接获取
```java
    PackageInfo info = context.getPackageManager.getPackageInfo(packageName,flags);
    
    //Android P版本
    SigningInfo signingInfo = info.signingInfo;

    //小于P版本
    Signature[] signatures = packageInfo.signatures
```
除此之外，还有一套api,主要差异主要是在获取PackageInfo的方式不同。
```java
    PackageInfo info = context.getPackageManager.getPackageArchiveInfo(apkFilePath,flags);
```
表面看两者差异不大，实际上却有着本质的区别，前者是IPC调用PackageManagerService获取已经安装的apk的签名信息，后者是客户端层面通过PakcageParser读取apk文件获取签名信息。

简单的操作，可以通过动态代理PackageManager的相关方法，处理跨进程调用的，但实际上远远不够，记接下来，我们来看LSPosed是如何处理的。
## 三、Java层签名绕过
Java层的绕过主要是为了解决第二节中介绍的两种获取PackageInfo的Api，分别是本地调用和跨进程调用。
### 3.1 处理跨进程调用
Android的Java层跨进程数据传递离不开Parcelable，故LSPosed也是从此处出发，整体上比动态代理PackageManager更方便一些。
```java
    private static void proxyPackageInfoCreator(Context context) {
        //获取原始的PackageInfo.CREATOR，这个 CREATOR 负责从 Parcel 中创建 PackageInfo 对象。
        Parcelable.Creator<PackageInfo> originalCreator = PackageInfo.CREATOR;
        //创建用于代理的Creator的对象
        Parcelable.Creator<PackageInfo> proxiedCreator = new Parcelable.Creator<>() {
            @Override
            public PackageInfo createFromParcel(Parcel source) {
                //先调用原始的originalCreator创建PackageInfo对象，然后对PackageInfo对象进行改造
                PackageInfo packageInfo = originalCreator.createFromParcel(source);
                replaceSignature(context, packageInfo);
                return packageInfo;
            }

            @Override
            public PackageInfo[] newArray(int size) {
                return originalCreator.newArray(size);
            }
        };
        //通过反射将PackageInfo中的CREATOR对象替换成代理对象
        XposedHelpers.setStaticObjectField(PackageInfo.class, "CREATOR", proxiedCreator);

        // 清空 Parcel 的 mCreators 和 sPairedCreators,这两者是系统缓存，为了避免重复创建Creator对象，这里清空后，会重新走生成逻辑。
        try {
            Map<?, ?> mCreators = (Map<?, ?>) XposedHelpers.getStaticObjectField(Parcel.class, "mCreators");
            mCreators.clear();
        } catch (NoSuchFieldError ignore) {
        } catch (Throwable e) {
            Log.w(TAG, "fail to clear Parcel.mCreators", e);
        }
        try {
            Map<?, ?> sPairedCreators = (Map<?, ?>) XposedHelpers.getStaticObjectField(Parcel.class, "sPairedCreators");
            sPairedCreators.clear();
        } catch (NoSuchFieldError ignore) {
        } catch (Throwable e) {
            Log.w(TAG, "fail to clear Parcel.sPairedCreators", e);
        }
    }
```
由于跨进程传输数据会通过Parcelable.Creator来创建对象，通过上述方案，可以将这部分调用获取的数据都处理掉，最终完成绕过签名校验。

处理CREATOR这种方式其实不仅仅绕过了通过Context.getPakcageManager来获取签名信息，同时也绕过了客户端自己绕过PackageManager对象，直接和远端的PackageManagerSerivce直接通讯获取签名信息。
大概思路如下，自己构建Parcel属于，通过transact方法完成调用，TRANSACTION_getPackageInfo()内部也是获取PackageManager内部的相关数据，感兴趣可以自己看一下。
```java
    public PackageInfo getPakcageInfo(){
        try {
            PackageManager packageManager = getBaseContext().getPackageManager();
            Object IPC_PM_Obj = Utils.getObjectField(packageManager, "mPM");
            IBinder mRemote = (IBinder) Utils.getObjectField(IPC_PM_Obj, "mRemote");
            Parcel _data = Parcel.obtain();
            Parcel _reply = Parcel.obtain();
            _data.writeInterfaceToken("android.content.pm.IPackageManager");
            _data.writeString(getPackageName());
            _data.writeLong(PackageManager.GET_SIGNATURES);
            _data.writeInt(android.os.Process.myUid());
            boolean _status = mRemote.transact(TRANSACTION_getPackageInfo(), _data, _reply, 0);
            _reply.readException();
            PackageInfo packageInfo = _reply.readTypedObject(PackageInfo.CREATOR);
            _data.recycle();
            _reply.recycle();
            return packageInfo;
        } catch (Throwable e) {

        }
        return null;
    }
```

### 3.2 处理本地解析调用
本地解析调用，核心是处理PackageParser这个对象，PackageManager.getPackageArchiveInfo这个方法内部是通过创建PakcageParser对象解析apk文件，最终拿到签名信息。
```java
    XposedBridge.hookAllMethods(PackageParser.class, "generatePackageInfo", new XC_MethodHook() {
        @Override
        protected void afterHookedMethod(MethodHookParam param) {
            var packageInfo = (PackageInfo) param.getResult();
            if (packageInfo == null) return;
            replaceSignature(context, packageInfo);
        }
    }); 
```
LSPosed这里通过Java Hook的方式，对PackageParser做了处理，将其generatePackageInfo的返回值进行了处理，达成修改签名返回的
## 四、Native签名绕过
在介绍获取信息的api中，PackageManager.getPackageArchiveInfo是通过解析本地文件，既然系统可以在客户端层直接读取apk中的前面信息，自然我们也可以自己读取apk，然后进行校验签名。
这一节是介绍Native签名绕过，并非是Nativce层有api可以直接获取签名，而是通过Native Hook的方式处理App解析Apk获取签名。

这里处理的位置选择的是文件相关的操作，核心思路是如果让用户读取到的是一个没有被修改过的apk，最终达到绕过校验的目的。
之前在做IO监控的时候（相关文章，感兴趣可以看一下），对文件打开的open方法进行了PLT HOOK， 如果在proxy方法中，返回一个未修改apk的fd，就可以达到目的。
之前IO监控相关只处理的Java相关的文件操作方法，此处LSPosed的Hook行为行为更加底层，为libc.so中的__openat。
```cpp
    LSP_DEF_NATIVE_METHOD(void, SigBypass, enableOpenatHook, jstring origApkPath, jstring cacheApkPath) {
        // 找到__openat的方法并进行代理，代理方法为__openat，详情见下方
        auto sym_openat = SandHook::ElfImg("libc.so").getSymbAddress<void *>("__openat");
        auto r = HookSymNoHandle(handler, sym_openat, __openat);
        if (!r) {
            LOGE("Hook __openat fail");
            return;
        }

        //记录LSPosed预置的原apk的路径以及用户读取的apk路径
        //前者在重打包的时候是预置在了assets目录下，app打开后，会将其写到磁盘中
        //后者一般即为base.apk的路径
        lsplant::JUTFString str1(env, origApkPath);
        lsplant::JUTFString str2(env, cacheApkPath);
        apkPath = str1.get();
        redirectPath = str2.get();
        LOGD("apkPath %s", apkPath.c_str());
        LOGD("redirectPath %s", redirectPath.c_str());
    }

    //创建__openat的代理方法
    CREATE_HOOK_STUB_ENTRY(
            "__openat",
            int, __openat,
            (int fd, const char* pathname, int flag, int mode), {
                //如果发现用户打开的是base.apk的路径，则返回违背修改的apk的fd。
                if (pathname == apkPath) {
                    LOGD("redirect openat");
                    return backup(fd, redirectPath.c_str(), flag, mode);
                }
                return backup(fd, pathname, flag, mode);
    });
```

## 五、对抗

此处只是简单介绍一下，真正生产环境的对抗手段更加多且复杂。

- java
检测Parcelable.Creator的ClassLoader,正常的Parcelable.Creator的classLoader应该是BootClassLoader，但是LSPosed创建的对象是使用的普通的ClassLoader

- native
Java层检测相对来说都是比较容易绕过的，可以做一些基础的进行简单的处理，相对核心的主要是放在Native层。

    - 1、通过svc的方式读取apk文件，然后进行解析，可以过掉LSPosed的IO重定向。
    - 2、通过fd反查文件权限，插件文件的权限，data下的base.apk属于系统文件
    - 3、通过fd反查文件路径，比较路径长度以及内容是否一致
    - 4、对proc/self/fd中的文件进行检查，是否有可疑文件，因为只要app打开的文件，此处会进行记录
