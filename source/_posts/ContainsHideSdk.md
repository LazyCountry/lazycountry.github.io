-----
title: Gradle工程使用@hide接口
tags: 
   - Android
   - hide
   - sdk
   - global libraries
   - gradle
-----

Android 系统存在一些系统级应用与 framework 代码耦合较深，用到了一些 hide 接口，如图

![image](ContainsHideSdk/no_hide.png)

非常影响阅读和编码。

一般我们可以通过 framework 编译出来的 class.jar 作为 Global Libraries 导入到项目中，可以解决这个问题。

![image](ContainsHideSdk/global_libraries.png)

并置顶

![image](ContainsHideSdk/global_libraries_top.png)

就可以识别到 hide 接口了。

但是 gradle sync 之后这个配置又会消失，每次更新 gradle.build 文件，重启 IDEA 都会导致需要重新进行这个配置，不胜其烦。

为了解决这个问题，这里提供个方法在 gradle 和 IDE 直接找到一个较好的平衡点 ---- 修改 Android 官方 SDK。

这里以 Android 5.0 为例。



1. 下载 7.0 SDK 包，然后从 \sdk\platforms\android-24 目录下取到 android.jar。
2. 从编译环境 out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/ 目录下渠道 classes-full-debug.jar
3. 解压 android.jar，方法：首先改名为 android.zip,然后用 winrar 解压到本地文件夹。解压 classes-full-debug.jar，方法和解压 android.jar 一样。
4. 将 classes-full-debug.zip 包里面的文件全部复制到 android.zip 对应的文件夹中，然后重新将 android.zip 文件夹打包为 android.jar。此时生成的 android.jar 是包含了所有 @hide 接口的 SDK 包。


这个时候无论你是如何 gradle sync，再次重启 IDEA 都可以找到 hide 接口了。

![image](ContainsHideSdk/have_hide.png)

这里提供个 7.0 已经包含 hide 接口的android.jar, [点击下载](http://ac-s2be5mnv.clouddn.com/20f2f78cf1c9b608.jar)