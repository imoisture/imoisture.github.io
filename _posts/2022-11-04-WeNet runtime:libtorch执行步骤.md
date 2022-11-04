# WeNet runtime/libtorch 执行步骤

## 一、拉取WeNet仓库
`git clone https://github.com/wenet-e2e/wenet.git`

## 二、编译runtime/libtorch
> 建议先看完2.1节 再执行这一步的编译。
根据 wenet/runtime/libtorch/README_CN.md 可知，需要先 cd ${YourWenetPath}/runtime/libtorch,  然后执行：
`mkdir build && cd build && cmake -DGPU=OFF .. && cmake --build .`

这一步坑比较多，可能遇到如下问题：

### 2.1 CMake问题

**2.1.1 提示CMake 版本过低，最低版本要求 cmake-3.14 。** 

	原因：使用的CMake版本过低，要升级到3.14以上。
	
**2.1.2 Protocol "https" not supported or disabled in libcurl 。** 
![](https://raw.githubusercontent.com/imoisture/imoisture.github.io/master/img/wenet-cmake-curl-error.png)

**原因：** 由于CMake在执行过程中需要调用libcurl进行依赖库下载，而使用的CMake在编译安装时没有配置curl。

 执行下载libtorch库的cmake文件路径：
 fc_base/libtorch-subbuild/libtorch-populate-prefix/src/libtorch-populate-stamp/download-libtorch-populate.cmake， 其中文件下载的cmake内容如下：↓
 
参考：CMake file函数--文件操作命令：[https://www.cnblogs.com/faithlocus/p/15613809.html](https://www.cnblogs.com/faithlocus/p/15613809.html)

🚀**3.1.1和3.1.2的问题都可以通过重新编译安装CMake来解决：**

```bash
## 1.下载 CMake源码
wget https://cmake.org/files/v3.16/cmake-3.16.0.tar.gz

## 2.解压缩
tar -zxf cmake-3.16.0.tar.gz

## 3.编译安装，指定配置curl
cd cmake-3.16.0
./bootstrap --system-curl --prefix=/usr/local/cmake
make -j 4 && make install

## 4.配置PATH
export PATH=/usr/local/cmake/bin/:$PATH

## 5.查看cmake版本
cmake --version

## 6.查看cmake是否链接 libcurl
ldd $(which cmake)

### 正确安装的情况下应该会出现：  libcurl.so.4 => /lib64/libcurl.so.4 (0x00007f0a13efe000)
```

**3.1.3  报错：INTTYPES_FORMAT没有设置**

**报错内容：**c++: error: unrecognized command line option '-std=c++14'  

![](https://raw.githubusercontent.com/imoisture/imoisture.github.io/master/img/wenet-c++14-error.png)

* 可能是当前使用的gcc版本过低, 建议升级到 gcc-5.4以上，然后重新执行。
* 若还是报错，请设置CC, CXX环境变量， 然后重试。

``` shell
CC=$YourGccPath/bin/gcc
export CC

CXX=$YourGccPath/bin/g++
export CXX
```

⚙️⚙️ **为了避免之前CMake生成的缓存文件的干扰，在重新源码编译安装完CMake之后，我删除了之前的build文件，然后重新执行如下命令：**

```bash
## DGPU=OFF 表示下载的是不含cuda的纯CPU版本的 libtorch。
mkdir build && cd build && cmake -DINTTYPES_FORMAT:STRING=C99 -DGPU=OFF .. && cmake --build .
```

*_编译成功：_*

![](https://raw.githubusercontent.com/imoisture/imoisture.github.io/master/img/wenet-compile-success.png)


## 执行decoder_main

> 参考：runtime/libtorch/README_CN.md

**4.1 下载模型文件**

``` bash
# root @ bjdd-isa-ai-chip1 in /docker_workspace/icode-api/baidu/xpu/api/example/wenet-conformer/wenet/runtime/libtorch on git:main o [5:49:35] 
wget https://wenet-1256283475.cos.ap-shanghai.myqcloud.com/models/aishell/20210601_u2%2B%2B_conformer_libtorch.tar.gz
tar -zxf 20210601_u2++_conformer_libtorch.tar.gz 
```

**4.2 准备wav格式音频文件**

要求准备一个16k采样率，单通道，16bits的 wav格式的音频文件。
README_CN.md 中并没有说明从哪里下载符合要求的 wav格式文件，我在网上找到了一些wav格式的数据集。
参考：[https://www.aishelltech.com/aishell_2](https://www.aishelltech.com/aishell_2)

``` shell
## 下载希尔贝壳提供的 wav格式音频文件
wget https://aishell-2-sample.oss-cn-beijing.aliyuncs.com/AISHELL-2-sample.zip
## 解压缩
unzip AISHELL-2-sample.zip

## 设置变量
export GLOG_logtostderr=1
export GLOG_v=3

wav_path=$PWD/iOS/data/wav/C0936/IC0936W0002.wav
model_dir=$PWD/20210601_u2++_conformer_libtorch/

## 执行测试
./build/bin/decoder_main --chunk_size -1 --wav_path $wav_path --model_path $model_dir/final.zip --unit_path $model_dir/units.txt 2>&1 | tee log.txt
```

**4.3 执行结果**

<audio id="audio" controls="" preload="none">
      <source id="wav" src="https://raw.githubusercontent.com/imoisture/imoisture.github.io/master/img/IC0936W0083.wav">
</audio>

![](https://raw.githubusercontent.com/imoisture/imoisture.github.io/master/img/wenet-run-success.png)


