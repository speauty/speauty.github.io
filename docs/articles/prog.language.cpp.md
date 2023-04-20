# C++ {docsify-ignore}

### UI学习

#### Qt 6.0

##### 1. Linux（Ubuntu 20.04）安装
?> 当前环境(uname -a)：`Linux BUG-MS-7C52 5.15.0-48-generic #54-Ubuntu SMP Fri Aug 26 13:26:29 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux`

[点击此处，下载安装包](https://d13lb3tujbc8s0.cloudfront.net/onlineinstallers/qt-unified-linux-x64-4.4.2-online.run)，下载完成后，直接进入下载目录，在终端执行（赋予执行权限）：`sudo chmod +x qt-unified-linux-x64-4.4.2-online.run`，然后直接执行即可（`./qt-unified-linux-x64-4.4.2-online.run`）。

然后，就崩了（`./qt-unified-linux-x64-4.4.2-online.run: error while loading shared libraries: libxcb-xinerama.so.0: cannot open shared object file: No such file or directory`），动态链接库没找到？安装：`sudo apt install --reinstall libxcb-xinerama0`，然后再来，好像旧弹出了熟悉的Qt安装程序。后面的操作，就很普通了。