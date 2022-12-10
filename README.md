# BPI-R64_LiveSystem
本项目为本人本科毕业项目设计，因为在自行研究时遇到了很多的问题，多亏了互联网上各位大佬的知识的分享，才能最终实现所预期的功能，也在完成时决定要将所涉及到的一些基本知识分享出来（虽然过了半年才有时间）。时至今日，仍无法理解本项目的意义何在，可能这也是最终连给A都没给我的原因吧。本项目的总体流程是，在BPI_R64裸板中需要刷入自行编译的Openwrt系统，并进行调试和修改，其次搭建Nginx平台，并将麦克风连接路由器，通过ffmpeg将麦克风接收到的直播源推送出去，用户只需要连接wifi，并通过rtmp播放器即可收听。<br>
以下内容皆为论文中个人理解所写，无摘抄任何材料（话说本来国内涉及这方面的就少，外网也不算好找，找到的也是过时的资料），个人认为论文中写的也比较详细了，因此也懒得再改了。<br>

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
![](https://github.com/BaG-Ray/BPI-R64_LiveSystem/raw/master/1.png)
由于本文是基于BPI-R64开发板，因此在目标系统中选择MediaTek Ralink ARM,次系统中选择MT7622，目标描述中选择BPI-R64。由于配置极多，本文主要列举本项目需要的配置，如表

###(3)Openwrt的烧录下载
```
make V=s -j8  //编译生成Openwrt固件
生成的固件位于
/openwrt/bin/targets/mediatek/mt7622/openwrt-mediatek-mt7622-bananapi_bpi-r64-sdcard.img.gz
```
本文中所采取的方法为通过烧录软件（如Win32DiskImager）,即可将固件烧入sd卡中，并设置通过sd卡启动固件即可成功加载Openwrt。亦可设置通过eMMC启动，将网线连入路由器的WAN口，通过TFTP协议将固件刷入flash即可。通过串口软件（如Putty）连接开发板的串口即可查看状态。

Openwrt的配置
------------
###(1)Openwrt的Overlay的扩容
Squashfs（.sfs）是一套供Linux核心使用的GPL开源只读压缩文件系统，专门为一般的只读文件系统的使用而设计[22]。通常而言，Openwrt默认都采用squashfs格式作为文件系统，这在图4-2中进行编译的时候也可以选择。Squashfs文件系统的特点为只读和压缩。因此，作为一个只读型的文件系统，而要对Openwrt所作的操作属于写入。实际上是采用了/overlay分区，也就是说Openwrt的所有记录，配置和安装软件都是属于对/overlay分区的操作。因此若是系统的问题，如无法正常进入系统，或是进入恢复模式时，重新刷入也不会丢失之前的配置。

事实上，Openwrt系统如图4-3所示分成了两个层：上层和下层。下层为只读层，也就是squashfs型openwrt，上层为可读可写层。下层中由两部分组成，sda1和sda2共同构成固件的大小，也就是编译生成的img文件的大小。其中sda1为kernel，sda2中firmware部分为固件的大小，后面的空间为未占用。但是由于下层未占用分区不能写入，因此划分出了/overlay分区，数据并不是直接写入下层，而是所有的读写都是在上层完成的。在系统启动的时候，固件会将所有的所需要的信息复制到/overlay分区，也就是上层中进行保存，比如说Openwrt根目录下的home,etc,usr等目录文件。真正空闲出来的才是剩下的未占用空间。/overlay分区最大的意义在于：由于有了上层逻辑层，就可以阻止用户对于kernel和固件区域的直接读写，进而在最终没有任何办法的时候还可以进行系统重置，重置时系统只需要抹除上层所有内容，系统重新从固件中取出初始文件复制到/overlay区中即可。若是/overlay分区剩余空间不足，则任何配置都不会被写入，也不能安装过多的软件，因此/overlay分区的扩容是非常有必要的。

本文最初采用的为Openwrt官方发布的编译好的固件，然而由于/overlay分区空间不足，因此任何的操作都不会被记录，只要开发板重启，就会恢复到最初刷入的状态。故本文自行进行了Openwrt的编译，虽然自行编译的版本可以保存，然而/overlay的空间仅有80MB，这样的大小是无法安装一些大型的应用，如docker。且若此时安装的应用将/overlay分区完全占满，此时Openwrt将会完全进入只读状态，也符合上面对/overlay分区的讨论，此时就连卸载应用都无法做到，因此只能重新刷入固件。但是正如上述所讨论的，Openwrt所有的配置都存放在/overlay中，只是重刷是不会改变情况的，必须要将sd卡完全格式化后重刷才能恢复初始状态。在后续的扩容实践摸索中，在查找了大量的资料后，共发现四种扩容/overlay的方法。本文先是尝试了挂载点方法，图4-4为/overlay的扩容，事实上可以建立一个更大的分区，比如插入u盘，图4-4中对应为sda3，将/overlay指向sda3。然而在实践中，第一种方法并没有达到预期的效果，即使将/overlay指向了更大的空间，并且在luci中软件安装的页面中的剩余空间也同样显示扩容成功，实际上只是数值显示的更大，挂载点中/root仍指向/overlay，只是此时有两个不同的区域同时指向了/overlay，使得/overlay看起来变大了，实际上软件的安装仍安装在/root指向的/overlay中，而没有安装在扩容的空间。此时若是安装应用满了，仍然会触发openwrt变成只读的状态。经研究发现，方法一只适合于软路由，而不适合于本项目中的硬路由，因为如图4-3中软路由的硬盘管理只有两个，而本文中的却不只两个，这在删除sda2并重新将/overlay指向sda3时会出现问题，事实情况如图4-5。


第二种是直接通过硬盘管理软件（如DiskGeniusEng，由于openwrt是linux系统，其文件格式为ext4，因此更适合在linux系统上使用resizepart 2指令）强行扩大/overlay空间，和方法一情况相同，该方法仍属于软路由。若是磁盘管理中只有sda1和sda2，根据图4-3，很容易就可以知道/overlay是存放于sda2，因此只要将空闲空间合并到sda2中即可扩容/overlay。
第三种方法是将编译好的固件存放于linux中，在linux中使用扩容指令直接对编译好的固件进行补0扩充，如
```
dd if=/dev/zero bs=1G count=20 >> openwrt.img
```
即直接增加固件的大小。然而在实践中这种方法会破坏固件，原因同样是不同于软路由中只有sda1和sda2。硬路由中最后的为mmcb1k1p128，装载的是BIOS，此方法扩容相当于把BIOS扩容。
第三种方法虽然实践中并不成功，但却提供了一个很好的方向。由于/overlay的空间分配是在Openwrt第一次启动时会自动分配固件整体大小的一定比例的空间。因此，在编译Openwrt的时候，可以在menuconfig中选择将固件的大小，在生成固件的时候就决定固件整体的大小，如图4-2中就设定为20400MB。此时得到的/overlay空间大小为19.2G，结果如图4-6和图4-7。


###(2)DNS的配置
由于Openwrt默认dns为127.0.0.1，通过ping指令可知无法访问互联网，因此也不能进行opkg update的更新或者软件包的安装。因此需要使用vim命令修改/etc/resolv.conf，将其改为8.8.8.8，如下修改即可ping成功。
```
vim /etc/resolv.conf
nameserver 8.8.8.8
```
###(3)Wifi模式的开启
Openwrt默认开启时是不启动wifi模式而只启动WAN模式，因此需要在Luci界面中网络的无线中手动开启，其他设备才能连入wlan，如图4-8。

若是需要实现开机自动启动wifi，则需要在编译Openwrt时修改wireless的相关源码。通过/etc/config/wireless的信息：
```
cat /etc/config/wireless
 config wifi-device 'radio0'
	option type 'mac80211'
	option path 'platform/18000000.wmac'
	option channel '1'
	option band '2g'
	option htmode 'HT20'
	option cell_density '0'
```
可知wifi采用的设备为mac80211。因此，修改linux系统中Openwrt编译的目录，打开
```
Openwrt/package/kernel/mac80211/files/lib/wifi/mac80211.sh
```
Mac80211.sh中存放的是编译Openwrt的时候对wifi设备的编译信息。
```
set wireless.radio${devidx}.disabled=1 修改为0
```
重新编译固件后可以实现开机自动开启wifi。
###(4)挂载点和硬盘管理界面的安装
uci默认是没有挂载点和硬盘管理这两个界面的，然而这两个Luci插件对本项目很有帮助，尤其是在docker的应用中，docker容器的挂载方面会很需要。挂载点插件可以通过opkg安装block-mount，也可以在编译Openwrt的菜单中进行选择。
硬盘管理由于是非luci官方插件，可以在github上下载后通过Luci在线安装，也可以使用SCP的方法，将ipk文件传入开发板中再通过opkg本地安装。此时如果打开硬盘管理则会报错，这正是因为非luci官方，因此其页面也需要作者自行设计。在安装luci-base 和 luci-compat后可正常访问，如图4-9和4-10。
```
opkg install luci-base luci-compat
```


###opkg install luci-base luci-compat
正如4.2.4所说，Openwrt中的软件包若是通过官方平台发布的，都可以通过opkg或者luci页面进行安装，非官方的软件也可以通过Luci上传或者使用SCP工具（如WinSCP）传到开发板进行本地安装。然而，在安装软件包的时候，系统会自动将所需依赖项进行下载，而某些依赖项可能因为某些原因无法下载，这时可以访问openwrt镜像库找到所需的依赖项，通过SCP工具将其放入lib目录中，重新下载安装即可。
其次，在安装软件包的时候可能会出现内核版本不兼容的问题，如图4-11。图中为所需的内核的小版本号为110，而已安装的为108。而通过opkg update是不会更新内核的小版本号，因此需要访问Openwrt的镜像库
```
downloads.openwrt.org/snapshots/targets/mediatek/mt7622/packages/
```
找到所需对应的kernel的ipk文件，下载对应的内核通过Luci安装。若是小版本号后面的内容不同，也可以通过这种方法，或者通过SCP访问开发板，打开/usr/lib/opkg/status，将后面的内容全部替换为所需的内容即可，这是因为后面的字符为md5的验证，自行编译的版本和官方发布的版本的md5是一定不符的。

4.2Nginx在硬件平台上的实现
=========================
Nginx的安装和配置
-----------------
如同4.1.2中（5）所说的，Nginx的安装可以从Luci的软件包中搜索安装，也可以通过opkg进行安装。输入nginx -v，如果出现以下信息，则表明安装成功。
```
root@Openwrt:/# nginx -v
Nginx version: nginx/1.21.3 (x86_64-pc-linux-gnu)
```
在安装完成nginx后，即可配置Nginx。Nginx的配置文件位于/etc/nginx中。其中uci.conf为测试配置文件，结合测试配置文件和Nginx官方文档。并由2.4.3中，可知Nginx是有四个模块构成，配置nginx.conf实际上就是对Nginx的各个模块进行配置。对应的user等部分为核心模块，events对应的为事件模块，http对应均衡器模块，后面补充的RTMP为协议模块。以下为nginx.conf配置内容：
```
user  root;  
//user用于设置master进程启动后，进程运行在那个用户和用户组下。本文使用root用户和用户组启动Nginx。
worker_processes  1;  
//设定Nginx worker的进程个数，其数量会直接影响性能。每个worker进程都是单线程，并调用各个模块以实现各种各样的功能。由于本文只需要实现Nginx配置RTMP服务器，因此只需一个进程运行RTMP服务即可。
events {
    worker_connections  1024;  
}
//单个进程最大连接数，最大连接数为连接数和进程数的乘积。进程数设定为1，因此最大连接数为1024。
http {
    server {
        listen      8082; 
}
} 
//listen监听的端口，实际上默认为80端口，然而80端口已经被luci占用，如果不修改为其他值，将会打不开luci。实际操作中表现为，浏览器访问192.168.1.1时将显示Nginx页面。因此此处设定为8082端口，此时访问192.168.1.1即可进入luci管理页面，访问192.168.1.1:8082即可访问Nginx页面。使用netstat指令可以看到8082可以被监听到，且识别为nginx服务器，如图4-12。
```

4.2.2Openwrt中关于Nginx拓展模块的支持
-------------------------------------
Openwrt中的Nginx和其他嵌入式平台中的Nginx有不同。由于Nginx的所有软件包，无论通过Luci平台还是SCP本地安装，本质上都是通过opkg进行安装。因此如果没有对应的软件包的ipk文件，是无法安装的。而Nginx的拓展模块，如本文所需的Nginx-RTMP-module模块在Luci平台是搜索不到的，即使是在其发布平台的github上都没有提供ipk文件。其他嵌入式平台，如CentOS中的Nginx，可以通过./configure进行配置Nginx的拓展模块，因此可以github上克隆下来再进行拓展模块的安装。然而Openwrt中的Nginx并没有这个功能，这意味着只能从编译的时候，将拓展模块编入Nginx的配置中，让Openwrt在编译的时候，把Nginx的拓展模块和Nginx一起编入Openwrt固件。
Nginx在Openwrt中的编译位置在
```
/openwrt/feeds/packages/net/nginx/makefile
```
事先从github上下载对应的拓展模块放入嵌入式系统中，找到代码中添加拓展模块的位置，在最后添加
```
    --add-module=模块文件位置/模块文件名
```

RTMP协议在Nginx上的实现
=======================
Nginx-RTMP-module模块的安装
--------------------------
正如4.2.2中所说，Nginx-RTMP-module正是Nginx中的一个模块。因此只能通过编译的方法安装Nginx-RTMP-module。本文将github克隆在了openwrt中。

然而，实际上Openwrt在编译中还是提供并可以选择安装Nginx-RTMP-module模块的。在Openwrt的编译程序中，可以使用“\”进行搜索，再输入RTMP得到的结果如图

这表明，Nginx-RTMP-module的安装位置在
```
Network -> Web Servers/Proxies -> nginx-ssl -> Configuration -> Nginx-RTMP-module
```
事实上，这个Configuration中包含了所有的Nginx模块。经过再次编译后，再次打开nginx.conf中，可以看到后面自动添加了如下信息，表明Nginx-RTMP-module已经安装成功。
```
location /stat {
            RTMP_stat all;
            RTMP_stat_stylesheet stat.xsl;
        }
        location /stat.xsl {
            root /root/nginx-RTMP-module-1.2.1/;
        }
        location /control {
            RTMP_control all;
        }
        location /RTMP-publisher {
            root /root/nginx-RTMP-module-1.2.1/test;
        }
        location / {
            root /root/nginx-RTMP-module-1.2.1/test/www;
        }
```

4.3.2RTMP的配置
---------------
如果上述没有安装Nginx-RTMP-module模块，则Nginx将不能开启RTMP，即使在Nginx.conf中添加如下的RTMP配置信息，Nginx也会返回不能识别RTMP的错误。因此在Nginx-RTMP-module安装成功后，打开Nginx.conf并添加如下信息，并且RTMP是和HTTP同级。RTMP和HTTP同级的原因，同样是因为2.4.3中所涉及的Nginx的模块化架构，事件模块（Event），协议模块（RTMP）和负载均衡器（HTTP）是同一级别的。
```
RTMP {
    server {
        listen 1935;  
//RTMP服务器的端口号，默认即为1935，表明RTMP服务器地址为192.168.1.1:1935
        application myapp {
//myapp为RTMP服务器名称，因此访问RTMP服务器不能只输入192.168.1.1:1935，而是应该访问192.168.1.1:1935/myapp/
            live on;
//服务开启
	    drop_idle_publisher 5s;
//将指定时间内闲置的发布流全部删除。若无此行，则会在直播一段时间之后无法就无法继续发布直播，并且服务器端监控显示没有数据输入。此时RTMP的连接将会产生流名错误，原因是直播流的名称已经存在。因为如果客户端因未能正常关闭与服务器的连接，而服务器上面该直播流依然存在，再次连接的时候就会提示badname。因此需要服务器需要将没有数据输入的流自动移除，同时还要保证正常连接的流，在即使没有声音输入时也会有少量数据传送至服务器。
                            }
            }
}
```

4.4麦克风的挂载和推流
---------------------
使用指令查看麦克风的设备
```
Arecord -l
List of CAPTURE Hardware Devices 
xcb_connection_has_error() 返回真
card 3: Microphone [USB Microphone], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```
因此，可以知道麦克风的设备号为card 3, device 0。对应即为hw:3,0。之后使用FFmpeg指令将麦克风信息推向RTMP服务器。
```
ffmpeg -f alsa -ar 16000 -ac 1 -i hw:3,0 -f flv "rtmp://192.168.1.1/myapp"
```
该命令中“-f ALSA”表示先输入通过ALSA麦克风采集到的音频数据；“-ar 16000”表示ALSA的音频采样率为16000Hz；“-ac 1”表示采样的为单声道；“-i hw:3,0”表示采样的设备为hw3,0，即麦克风；“-f flv”"RTMP://192.168.1.1/myapp"”表明以flv的格式推流到RTMP服务器上。
