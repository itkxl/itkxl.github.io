设备：pixel6
环境：ubuntu23.10   刷机这个环境都可以


adb reboot bootloader


注意事项：
- 只要是刷pixel6,一定要双slot的bootloader版本要大于手机上的版本
- 只要是刷pixel6,就刷最新的ROM，并且保证刷双slot

pixel6降级刷入安卓12和KernelSU教程：
一、确保双slot都是最新
1. 下载最新p6的rom
proxychains4  wget https://dl.google.com/dl/android/aosp/oriole-tq3a....
2. 解压，看里面bootloader的版本是否大于当前手机的版本（按住音量-再按开机），我的是大于的bootloader-oriole-slider-1.2-9883526.img，看1.2-9883526
3. ./flash-all.sh 刷入
4. 确保双slot都是最新bootloader：
fastboot flash bootloader --slot=all bootloader-oriole-slider-1.2-9883526.img
二、刷入专用开发者安卓12
5. 下载p6专用安卓12特供ROM
https://dl.google.com/developers/android/sc/images...
6. ./flash-all.sh
正常开机，会出现开发者专用版本提示
三、刷入KernelSU
1. 根据官网消息：Installation | KernelSU
只要这几个大版本对得上就行：5.10-android12-9
那我们下载的是：android12-5.10.185_2023-09-boot.img.gz文件，apk就下载最新的即可
2. Fastboot flash boot android12-5.10.185_2023-09-boot 成功 最终效果如图
四、附上正常使用的最新版zygisk、lsposed
五、日后若再刷，只能刷最最最新版的安卓13甚至安卓14噢，千万千万不要自己刷个低版本噢，会变砖的噢。 S自己刷成砖了救砖600噢