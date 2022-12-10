# BPI-R64_LiveSystem
本项目为本人本科毕业项目设计，因为在自行研究时遇到了很多的问题，多亏了互联网上各位大佬的知识的分享，才能最终实现所预期的功能，也在完成时决定要将所涉及到的一些基本知识分享出来（虽然过了半年才有时间）。时至今日，仍无法理解本项目的意义何在，可能这也是最终连给A都没给我的原因吧。本项目的总体流程是，在BPI_R64裸板中需要刷入自行编译的Openwrt系统，并进行调试和修改，其次搭建Nginx平台，并将麦克风连接路由器，通过ffmpeg将麦克风接收到的直播源推送出去，用户只需要连接wifi，并通过rtmp播放器即可收听。
以下内容皆为论文中个人理解所写，无摘抄任何材料（话说本来国内涉及这方面的就少，外网也不算好找，找到的也是过时的资料），个人认为论文中写的也比较详细了，因此也懒得再改了。
嵌入式平台的搭建
===================
基于Linux的openwrt的编译
-------------------------
###(1)编译Openwrt的准备
本文中的Openwrt编译环境为Vmware 16.1.0 build-17198959中运行的Ubuntu 20.04.4 LTS。其中编译Openwrt所需的准备如下：
```
sudo apt-get install g++ libncurses5-dev zlib1g-dev bison flex unzip autoconf gawk make gettext gcc binutils patch bzip2 libz-dev asciidoc subversion  //安装所需的依赖项
git clone https://github.com/openwrt/openwrt.git source  //使用git指令从Openwrt官网下载Openwrt源码
cd openwrt
./scripts/feeds update -a  //更新软件包
/scripts/feeds install -a   //安装最新软件包
```
###(2)编译Openwrt
运行make menuconfig进入Openwrt的配置，如图
