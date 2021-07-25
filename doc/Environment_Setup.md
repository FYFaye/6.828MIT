# MIT 6.828 环境配置
配置环境折腾了一阵子，记录一下操作步骤和踩过的坑；  
环境配置主要包括两类，一类是一个x86的模拟器，叫QEMU，另一类是编译工具链；

## 我的环境
Ubuntu 18.04  WSL 1环境

## 步骤

### 编译工具链
首先在主目录下创建一个名叫6.828的文件夹，后续会clone一个qemu的文件夹装用来跑JOS系统的虚拟机，另外一个lab文件夹用来装JOS系统；
首先输入
```
objdump -i
```
MIT官网上介绍说，第二行应该出现 elf32-i386字样；而在我的WSL中打印的不止该字样，还有其他很多elf；  
然后输入：
```
gcc -m32 -print-libgcc-file-name
```
这个主要是看你有没有GCC，没有的话就`sudo apt-get install -y build-essential gdb`进行安装；  
如果有GCC应该出现
```
/usr/lib/gcc/i486-linux-gnu/version/libgcc.a
```
或者
```
/usr/lib/gcc/x86_64-linux-gnu/version/32/libgcc.a
```
现在一般都使用x64的系统，还需要安装一下32位的库 sudo apt-get install gcc-multilib；  


### 安装QEMU 模拟器
首先需要git clone QEMU的仓库，.git后面那个qemu，是将其clone到文件夹qemu里面的意思，如果已经有了该文件夹就会报错；
```
git clone https://github.com/mit-pdos/6.828-qemu.git qemu
```

在linux环境下，需要额外安装其他的库：
```
sudo apt install libsdl1.2-dev
sudo apt install libtool-bin
sudo apt install libglib2.0-dev
sudo apt install libz-dev
sudo apt install libpixman-1-dev
```

cd到clone的6.828-qemu目录中，设置config：
```  
./configure --disable-kvm --disable-werror --target-list="i386-softmmu x86_64-softmmu"
```
如果没有Python需要先安装一下

sudo apt install python

设置config后就可以make && make install 
```
make && make install
```

然后cd到6.828目录下，输入以下命令clone JOS
```
git clone https://pdos.csail.mit.edu/6.828/2018/jos.git lab 
```
cd 到lab文件夹中，执行make,如果提示
```
+ mk obj/kern/kernel.img
```
再执行`make qemu`，WSL使用`make qemu-nox`
如果正常，会出现一个新的控制台窗口，这代表已经完成了。

退出新的控制台，要先按Ctrl + A（注意是字母A，不是alt键），在按x即可退出

