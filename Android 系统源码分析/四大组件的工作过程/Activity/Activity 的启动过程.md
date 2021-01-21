# **Activity 的启动过程**
Activity的启动过程分为两种：1、根 Activity 的启动过程，2、普通 Activity 的启动过程。
根 Activity 指的是应用程序启动的第一个 Activity，因此根 Activity 的启动过程一般也可视为应用程序的启动过程。
## 根 Activity 的启动过程
### 根 Activity 启动过程中涉及的进程
根 Activity 的启动过程涉及到 4 个进程：Launcher 进程、AMS 进程、Zygote 进程和应用程序进程。
当点击桌面应用图标后，Launcher 进程会向 AMS 发送启动应用程序请求（Binder 通信），AMS 进程首先会检查当前应用程序进程是否已经启动并运行，如果没有，则向 Zygote 进程发送启动应用程序进程的请求（Socket 通信），最后 AMS 会向 ApplicationThread 发送启动 Activity 的消息（Binder 通信）完成 Activity 的启动。

![](https://github.com/FangQi-Jack/Android-/blob/main/Android%20%E7%B3%BB%E7%BB%9F%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6%E7%9A%84%E5%B7%A5%E4%BD%9C%E8%BF%87%E7%A8%8B/Activity/%E6%A0%B9%20Activity%20%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E8%BF%9B%E7%A8%8B%E8%B0%83%E7%94%A8.png)

![](根 Activity 启动过程进程调用.png)

### Launcher 请求 AMS 过程

![](https://github.com/FangQi-Jack/Android-/blob/main/Android%20%E7%B3%BB%E7%BB%9F%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6%E7%9A%84%E5%B7%A5%E4%BD%9C%E8%BF%87%E7%A8%8B/Activity/Launcher%20%E8%AF%B7%E6%B1%82%20AMS%20%E8%BF%87%E7%A8%8B.png)

![](Launcher 请求 AMS 过程.png)
### AMS 到 ApplicationThread 的调用过程

![](https://github.com/FangQi-Jack/Android-/blob/main/Android%20%E7%B3%BB%E7%BB%9F%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6%E7%9A%84%E5%B7%A5%E4%BD%9C%E8%BF%87%E7%A8%8B/Activity/AMS%20%E5%88%B0%20ApplicationThread%20%E7%9A%84%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B.png)

![](AMS 到 ApplicationThread 的调用过程.png)
### ActivityThread 启动 Activity 的过程

![](https://github.com/FangQi-Jack/Android-/blob/main/Android%20%E7%B3%BB%E7%BB%9F%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6%E7%9A%84%E5%B7%A5%E4%BD%9C%E8%BF%87%E7%A8%8B/Activity/ActivityThread%20%E5%90%AF%E5%8A%A8%20Activity%20%E7%9A%84%E8%BF%87%E7%A8%8B.png)

![](ActivityThread 启动 Activity 的过程.png)

## 普通 Activity 的启动过程
普通 activity 的启动过程只会涉及 AMS 进程和应用程序进程。
