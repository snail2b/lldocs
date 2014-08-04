#samba文件共享+ssh网络终端
--------------------------

#一、samba文件共享
##1. samba简介
--------------------------
首先，Samba是在Linux和UNIX系统上实现SMB协议的一个免费软件，由服务器及客户端程序构成。SMB（Server Messages Block，信息服务块）是一种在局域网上共享文件和打印机的一种通信协议，它为局域网内的不同计算机之间提供文件及打印机等资源的共享服务。SMB协议是客户机/服务器型协议，客户机通过该协议可以访问服务器上的共享文件系统、打印机及其他资源。通过设置“NetBIOS over TCP/IP”使得Samba不但能与局域网络主机分享资源，还能与全世界的电脑分享资源。

##2. 为何选择samba
--------------------------
从上面的介绍知道，samba是使用SMB协议来进行文件共享的，而在嵌入式开发中与主机进行文件共享是经常会使用到的，这也是我们使用它的原因。可我们还知道一些可以用于文件共享也是经常用到的一些方式，比如ftp和nfs，那我们为什么不用它们呢。当明白它们各自的优缺点和使用范围就可以很清楚这一点了。简单地说，FTP传输速度比较慢，而且每次使用要在主机先建立连接，NFS则主要是用于两个Linux主机之间进行文件共享和数据传输，而smb既有较快的局域网传输速度也可以用于在Linux和windows之间共享文件，类似于windows上面的网上邻居无需每次都建立连接跟访问本地文件系统一样访问共享文件夹。

##3. 在arm平台安装samba
--------------------------
这里使用的是samba较老的一个版本samba-3.6.1，可以在[这里](http://ftp.samba.org/pub/samba/stable/)下载, 之所以没使用最新版本，是因为最新版本的交叉编译问题，最新版本使用了新的编译系统waf（一个用python实现的开源编译系统），网上也没有相应的基于samba4交叉编译的资料，我也无从下手啊，完全不懂那个waf怎么配置交叉编译，所以我还是使用了一个低版本的samba。**具体步骤如下：**

- 下载好后解压
        tar -xjvf samba-3.6.1.tar.gz
- 进入source3目录
        cd samba-3.6.1/soucre3
- 创建一个脚本文件`config.sh`，因为交叉编译在执行configure脚本生成Makefile时需要指定很多参数，故这里单独创建一个脚本文件config.sh来初始化一些用到的环境变量和给configure传递参数其中各参数的功能可以执行`./configure –h`查看
```sh
    #file-name: config.sh
    #!/bin/sh
       
    export LD_LIBRARY_PATH=/usr/local/arm-none-linux/lib
    export LDFLAGS=-L/usr/local/arm-none-linux/lib
    export CPPFLAGS=-I/usr/local/arm-none-linux/arm-none-linux-gnueabi/include
    export CC=arm-none-linux-gnueabi-gcc
       
    ./configure --without-krb5 --without-ldap --without-ads --disable-cups --enable-swat=no --with-winbind=no --target=arm-none-linux-gnueabi --host=i686 --prefix=/usr/local/samba/sbin libreplace_cv_HAVE_GETADDRINFO=no samba_cv_CC_NEGATIVE_ENUM_VALUES=yes ac_cv_file__proc_sys_kernel_core_pattern=yes
    # 其中交叉编译器如果没有添加到系统环境变量`PATH`中的话，则需要使用绝对路径指定其位置
```
- 修改config.sh权限，执行`./config.sh` 生成`Makefile`文件

- 修改Makefile，找到这段，删除加粗部分代码
> CPPFLAGS=-DHAVE_CONFIG_H -I/usr/arm-linux/include -Iinclude -I./include -I. -I. -I./../lib/replace -I./../lib/tevent -I./librpc -I./.. -I./../lib/popt **-I/usr/local/include**
- 执行`make`命令
- 编译完成后将在`samba-3.6.1/soucre3/bin`目录下生成目标平台的二进制文件和库文件。
- 在目标平台中，在`/usr/local/`中新建`samba`，`samba/sbin`,`samba/sbin/lib`, `samba/sbin/private`, `samba/sbin/var`, `samba/sbin/var/locks`，然后将上一步骤生成的`samba-3.6.1/soucre3/bin`目录下的文件使用U盘或者SD卡全部拷贝到刚在目标平台文件系统中新建的`/usr/local/samba/sbin`目录中。
- 在`samba/sbin/lib`目录中创建`smb.conf`配置文件，**内容如下：**
```sh
    [global]
            workgroup = WORKGROUP           #windows中工作组名
            server string = ema_samba       #samba服务器名
            netbios name = ema              
            guest account = root            #匿名用户访问权限设置
            security = share
    [ema_board]
            path = /home/ema                #指定共享目录
            writeable = yes                 #写入权限
            comment = Home Directories
            guest ok = yes                  #允许匿名访问，若不使用匿名访问，则需要使用smbpasswd命令创建samba用户，并设置访问密码
```

##4. 启动samba
- 启动samba
    - 将`/usr/local/samba/sbin`添加进系统环境变量
    - 执行`smbd`和`nmbd`命令，即可启动samba
- 添加开机自启服务
    - 在`/etc/init.d`目录下创建一个启动脚本文件`smbd`
    - 脚本内容如下：
```sh
    #!/bin/sh
     
    PATH=/usr/local/bin/samba:$PATH
    NAME=smbd
    DEMON=/usr/local/bin/samba/$NAME
     
    if [ -f $DEMON ]; then
             $DEMON
    fi
    NAME=nmbd
    if [ -f $DEMON ]; then
             $DEMON
    fi
    exit 0
```
    - 执行`runlevel`命令，查看系统运行等级，得知当前系统运行在level 5，故在`/etc/rc5.d/`目录中建立`/etc/init.d/smbd`的软链接，并重命名，命名规则为`'S +'启动顺序的编号'+'smbd'`，然后重启系统使用`ps -aux`即可看到smbd服务已启动。

##5. 访问samba共享目录
首先将目标机连接至和主机相同的局域网内，最好采用静态ip的方式，确认互相可以ping通。
- Ubuntu主机上通过`主文件夹->网络->浏览网络`即可直接想访问本地文件系统一样访问。
- Windows主机上可以通过添加网络位置、映射网络驱动器或者直接在`运行`程序中输入`'\\目标机ip地址'`（eg:\\192.168.1.110）

##6. 参考：
-------------------------

- <http://greensoftware.net/blog/2012/01/04/compiling-samba-3-6-1-for-arm/>（解决交叉编译）
- <http://blog.chinaunix.net/uid-25201977-id-3249599.html>（解决配置文件）
- <http://wiki.beyondlogic.org/index.php/Cross_Compiling_SAMBA_for_ARM>（解决编译后文件过大）

#二、ssh网络终端
ssh网络终端的使用比较简单，就不做详细介绍了，在我们的开发板上ssh服务是自带的有且默认是开机启动的，所以我们只需要在主机上直接进行访问就是了。当然前提还是要讲主机和开发板连接到同一个局域网中。
- Linux端登陆使用`ssh`命令即可，后面加登陆名和目标Ip，格式为`登录名(一般为目标平台账户名)@目标ip地址`(eg:ema@192.168.1.110)。连接成功后根据提示输入登陆密码，即登陆成功。
- Windows端使用SecureCRT，也很简单就不介绍了。