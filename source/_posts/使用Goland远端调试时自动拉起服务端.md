---
title: 使用Goland远端调试时自动拉起服务端
date: 2021-01-28 20:28:30
categories: Tips
tags: [Golang]
---
### 动机

在使用Goland+dlv远程调试服务时，需要在服务器执行dlv程序，等待本地Goland连入。希望在使用Goland Debug时自动把server拉起来。
### 方法

1. 直接在Goland的Run/Debug Configuration为调试任务设置前置任务，即执行脚本：

   ```sh
   //bootstrap.sh
   ssh username@host << eeooff
       # code: bootstrap dlv in server
   eeooff
   ```

2. 这样的话，调试任务会等待`bootstrap.sh`执行完毕，然而该脚本本身就会监听调试端口，等待调试任务连入，形成死锁。

3. 增加一个`entry.sh`作为前置任务：

   ```sh
   open -a Terminal.app bootstrap.sh #in macOS
   sleep 1 # wait for bootstrap to get ready
   ```