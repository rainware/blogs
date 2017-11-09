---
layout: post
title:  "基于aws-EC2使用nginx+ffmpeg+gpu搭建视频直播实时转码服务"
date:   2017-11-09 10:23:00 +0800
category: [安装文档, 学习文档]
tagline: "Supporting tagline"
tags: [nginx-rtmp-module, flv, ffmpeg, gpu, nginx, aws, ec2, nvidia, CUDA Toolkit, libx264]
---

# 0. 背景

  运营同学需要在管理后台中审核正在进行的直播，把内容不符合规定的主播禁播。因为直播数量太多，所以需要在一个页面上打开多个(20-60)播放窗口, 所以要求浏览器负载和带宽使用越低越好, 另外不打算使用flash插件，需要支持flv/hls播放。考虑到使用ffmpeg进行实时转码的cpu负载较大，需要调研是否使用硬件加速(nvidia gpu)两种方式的性能差别。

  涉及软硬件及版本:
  1. aws g3.4xlarge * 1, 定价: $1.14/h
  2. aws c4.2xlarge * 1, 定价: $0.398/h
  3. nginx V1.12.2 http://nginx.org/download/nginx-1.12.2.tar.gz
  4. nginx-rtmp-mdule https://github.com/arut/nginx-rtmp-module
  5. nginx-http-flv-module V1.2.0 https://github.com/winshining/nginx-http-flv-module/tree/v1.2.0
  6. ffmpeg https://git.ffmpeg.org/ffmpeg.git
  7. libx264 git://git.videolan.org/x264.git
  8. nvidia driver http://us.download.nvidia.com/XFree86/Linux-x86_64/367.122/NVIDIA-Linux-x86_64-367.122.run
  9. cuda Toolkit https://developer.nvidia.com/compute/cuda/9.0/Prod/local_installers/cuda_9.0.176_384.81_linux-run
  10. Centos 7

# 1. 使用C系列实例

### 1.1 安装ffmpeg
尝试了两种安装方式

##### 1.1.1 使用yum安装

因为ffmpeg现在还没有yum的官方安装包，需要使用第三方的yum源: Nux Dextop

* 需要先安装EPEL源, 更新后重新启动
```
yum install epel-release -y
yum update -y
shutdown -r now
```

* 安装Nux Dextop YUM repo
```
rpm --import http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro
rpm -Uvh http://li.nux.ro/download/nux/dextop/el7/x86_64/nux-dextop-release-0-5.el7.nux.noarch.rpm
```

* 安装ffmpeg
```
yum install ffmpeg ffmpeg-devel -y
```

> 这个源的ffmepg自带了libx264的库
```
configuration: --prefix=/usr --bindir=/usr/bin --datadir=/usr/share/ffmpeg --incdir=/usr/include/ffmpeg --libdir=/usr/lib64 --mandir=/usr/share/man --arch=x86_64 --optflags='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic' --enable-bzlib --disable-crystalhd --enable-gnutls --enable-ladspa --enable-libass --enable-libcdio --enable-libdc1394 --enable-libfaac --enable-nonfree --enable-libfdk-aac --enable-nonfree --disable-indev=jack --enable-libfreetype --enable-libgsm --enable-libmp3lame --enable-openal --enable-libopenjpeg --enable-libopus --enable-libpulse --enable-libschroedinger --enable-libsoxr --enable-libspeex --enable-libtheora --enable-libvorbis --enable-libv4l2 --enable-libx264 --enable-libx265 --enable-libxvid --enable-x11grab --enable-avfilter --enable-avresample --enable-postproc --enable-pthreads --disable-static --enable-shared --enable-gpl --disable-debug --disable-stripping --shlibdir=/usr/lib64 --enable-runtime-cpudetect
```

参考：https://www.vultr.com/docs/how-to-install-ffmpeg-on-centos

##### 1.1.2 编译安装

我们对rtmp流进行编解码时需要使用libx264。

* libx264

```bash
git clone http://git.videolan.org/git/x264.git
# 如果出现Found no assembler的错误，可以加上--disable-asm
./configure --enable-static --enable-shared --disable-asm
make
make install
```

> 会在当前目录下生成静态库libx264.a,动态库在/usr/local/lib,头文件/usr/local/include目录下

* ffmpeg

```bash
git clone git://source.ffmpeg.org/ffmpeg.git
# 编译成动态库   
./configure  --enable-shared --disable-static --enable-libx264 --enable-gpl --extra-cflags=-I/usr/local/include --extra-ldflags=-L/usr/local/lib  --enable-pthreads     
# 编译成静态库  
./configure  --enable-static --disable-shared --enable-libx264 --enable-gpl --extra-cflags=-I/usr/local/include --extra-ldflags=-L/usr/local/lib  --enable-pthreads
# 编译时间较长，可以启动多个线程
make -j 10
make install
```

> 安装时可能会报错找不到一些动态库，加入到动态库搜索路径中即可
```bash
export LD_LIBRARY_PATH=/a/b/c:$LD_LIBRARY_PATH;
```
或者在/etc/ld.so.conf中加入
```
/usr/local/x264/lib
/usr/local/lib
```
然后执行`ldconfig`命令

##### 1.1.3 转码示例

```bash
# 从rtmp://xxx.xxx.xxx/xxx/xxx拉流，转码后向rtmp://yyy.yyy.yyy/yyy/yyy推流
ffmpeg -i rtmp://xxx.xxx.xxx/xxx/xxx -preset veryslow -profile:v baseline -level 3.0 -vf scale=60:-1 -an -vcodec libx264 -crf 30 -f flv  rtmp://yyy.yyy.yyy/yyy/yyy -fix_teletext_pts 0 -copyts -muxdelay 0
```

* preset 若压缩后比特率固定，preset越低，质量越高。若质量固定，preset越低，压缩后比特率约低。
* profile和level设定是为了兼容不同的播放设备，baseline + 3.0基本可以兼容所有的设备
* vf scale 设定缩放比例
* crf 设定压缩后的画面质量，0代表无损
* an 不编码音频
* vcodec 指定编码库，这里使用libx264

### 1.2 安装nginx + nginx-http-flv-module

nginx-http-flv-module是对nginx-rtmp-mdule进行了封装和扩展, 目的是为了支持flv形式的拉流

```bash
wget http://nginx.org/download/nginx-1.12.2.tar.gz
tar xzvf nginx-1.12.2.tar.gz
git clone https://github.com/winshining/nginx-http-flv-module.git
cd nginx-1.12.2
# 可能会出现set but not used [-Wunused-but-set-variable]的错误，需要在CFLAGS中加入忽视该警告的选项
CFLAGS=-Wno-unused-but-set-variable ./configure --add-module=../nginx-http-flv-module
make
make install
```

> 安装在了/usr/local/nginx中

### 1.3 配置nginx作为rtmp服务器

```bash

#user  nobody;
worker_processes  1;

#pid        logs/nginx.pid;


error_log  logs/error.log debug;

events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    server {
        listen       80;
        server_name  localhost;

        location /stat {
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }
        location /stat.xsl {
            # you can move stat.xsl to a different location
        #    root /usr/build/nginx-rtmp-module;
            root /mnt/test/nginx-rtmp-module;
        }
        # rtmp control
        location /control {
            rtmp_control all;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
      	location / {
      	    flv_live on;
      	    chunked  on;
      	}
    }
}
rtmp_auto_push on;
rtmp_auto_push_reconnect 1s;
rtmp_socket_dir /tmp;
rtmp {
    out_queue   4096;
    out_cork    8;
    max_streams 64;
    server {
        listen 1935;
        ping 30s;
        notify_method get;

      	application live {
      	    live on;
            gop_cache on;
            # 当有拉流请求时，会触发exec_pull, 从xxx.xxx.xxx拉流，转码后推向本地址所对应的流。
      	    exec_pull /usr/local/bin/ffmpeg -y -i rtmp://xxx.xxx.xxx/$app/$name -preset veryslow -profile:v baseline -level 3.0 -vf scale=60:-1 -crf 30 -an -vcodec libx264 -f flv  rtmp://localhost/$app/$name -fix_teletext_pts 0 -copyts -muxdelay 0;
      	}
    }
}

```

启动nginx
`/usr/local/nginx/sbin/nginx`

可以使用在线srs播放器进行测试
http://182.92.80.26:8085/players/srs_player.html?vhost=182.92.80.26&stream=livestream&autostart=true

# 2. 使用GPU加速实例

### 2.1 根磁盘扩展

由于安装cuda toolkit时对根目录空间有要求，所以创建aws 实例时需要将root磁盘大小设置为20G。但是当启动实例后会发现/空间仍然是10G，这是因为通过镜像启动的实例，根磁盘已经进行分区，且分区大小固定为10G。所以需要进行分区大小的扩展。

参考：https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/recognize-expanded-volume-linux.html

如下所示
```
[root@local ~]# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda    202:0    0   20G  0 disk
└─xvda1 202:1    0   10G  0 part /
xvdb    202:16   0  100G  0 disk /mnt
```

使用growpart对xvda1分区进行扩展

```
yum install cloud-utils-growpart -y
growpart /dev/xvda 1
reboot
```

安装resize2fs
```
yum install e2fsprogs
resize2fs /dev/xvda1
```

### 2.1 安装nvidia驱动

* 更新软件包缓存并获取实例的必需软件包更新
```
yum update -y
```

* 重启实例以加载最新内核版本
```
reboot
```

* 为当前运行的内核版本安装 gcc 编译器和内核标头软件包
```
yum install -y gcc kernel-devel-$(uname -r)
```

* 禁用 NVIDIA 显卡的 nouveau 开源驱动程序

    1. 将 nouveau 添加到 modprobe 黑名单文件
    ```
cat << EOF | sudo tee --append /etc/modprobe.d/blacklist.conf
blacklist vga16fb
blacklist nouveau
blacklist rivafb
blacklist nvidiafb
blacklist rivatv
EOF
    ```

    2. 编辑 /etc/default/grub 并将以下文本添加到 GRUB_CMDLINE_LINUX 行
    ```
    modprobe.blacklist=nouveau
    ```

    3. 重新生成 Grub 配置
    ```
    grub2-mkconfig -o /boot/grub2/grub.cfg
    ```

    4. 重新启动
    ```
    reboot
    ```

* 下载安装nvidia driver

如下

```
wget http://us.download.nvidia.com/XFree86/Linux-x86_64/367.122/NVIDIA-Linux-x86_64-367.122.run

/bin/bash ./NVIDIA-Linux-x86_64-367.122.run
reboot
```

重新启动后验证安装: nvidia-smi

优化GPU设置：http://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/optimize_gpu.html

1. 将 GPU 设置配置为永久

```
nvidia-smi -pm 1
```

2. 禁用实例上所有 GPU 的 autoboost 功能

```
nvidia-smi --auto-boost-default=0
```

3. 将所有 GPU 时钟速度设置为其最大频率

```
nvidia-smi -ac 2505,1177
```

参考: http://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/install-nvidia-driver.html

### 2.2 安装cuda toolkit

```
wget https://developer.nvidia.com/compute/cuda/9.0/Prod/local_installers/cuda_9.0.176_384.81_linux-run
/bin/bash ./cuda_9.0.176_384.81_linux-run
```

安装成功后将/usr/local/cuda-9.0/lib64加入到/etc/ld.so.conf中，执行`ldconfig`使之生效

参考:http://johnpreston.ews-network.net/posts/aws-ffmpeg-and-nvidia-grid-k520/

### 2.3 编译安装ffmpeg

这次编译支持nvidia硬件加速的ffmpeg

> 建议将libx264库也带上，安装方法见1.1.2

```
git clone git://source.ffmpeg.org/ffmpeg.git
./configure  --enable-shared --disable-static --enable-libx264 --enable-gpl --extra-cflags="-I/usr/local/include -I/usr/local/cuda/include" --extra-ldflags="-L/usr/local/lib -L/usr/local/cuda/lib64"  --enable-pthreads
make -j 10
make install
```

### 2.4 使用gpu加速的ffmepg转码示例

```
ffmpeg -hwaccel cuvid -i rtmp://xxx.xxx.xxx/xxx/xxx -preset slow -profile:v baseline -level 3.0 -vf scale=60:-1 -an -vcodec h264_nvenc -qmin:v 35 -qmax:v 35 -f flv  rtmp://yyy.yyy.yyy/yyy/yyy -fix_teletext_pts 0 -copyts -muxdelay 0
```

参考: https://developer.nvidia.com/ffmpeg

# 3. 使用硬件加速前后的转码效果对比

* c4.2xlarge不使用GPU

同时转码50路流，cpu空闲20%左右

* g3.4xlarge使用GPU

同时转码120路流, gpu空闲10%
