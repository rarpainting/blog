# Tensorflow

## CPU

[ubuntu16.04 解决 tensorflow 提示未编译使用 SSE3、SSE4.1、SSE4.2、AVX、AVX2、FMA 的问题](https://blog.csdn.net/nicholas_wong/article/details/70215127)

基本解决问题

但是在使用时会启动 TPU , 导致启动失败

```shell
from tensorflow.contrib.learn.python.tpu import tpu_estimator
from tensorflow.contrib.tpu.pyhon.tpu.tpu_config import *
from tensorflow_estimator python.estimator.tpu.tpu_config import *

...

No module named 'tensorflow_estimator python.estimator.tpu'
```

## GPU

[neidia-docker2](https://www.cnblogs.com/sddai/p/10463085.html)

### matebook13

用辣鸡 mx150 来尝试一把编译 GPU 版本

#### 配置

```shell
$ uname -a

Linux debian 4.18.0-0.bpo.3-amd64 #1 SMP Debian 4.18.20-2~bpo9+1 (2018-12-08) x86_64 GNU/Linux

$ neofetch
OS: Debian GNU/Linux 9.6 (stretch) x86_64 
Model: Aspire E1-471G V2.16 
Kernel: 4.18.0-0.bpo.3-amd64 
Uptime: 1 day, 18 hours, 33 minutes 
Packages: 3254 
Shell: fish 2.4.0 
Resolution: 1366x768 
DE: XFCE 
WM: Xfwm4 
WM Theme: Default 
Theme: Xfce [GTK2], Adwaita [GTK3] 
Icons: Tango [GTK2], Adwaita [GTK3] 
Terminal: terminator 
CPU: Intel i5-3230M (4) @ 3.2GHz 
GPU: NVIDIA GeForce 610M/710M/810M/820M / GT 620M/625M/630M/720M 
Memory: 4691MB / 7799MB
```

禁用开源显卡驱动 nouveau

安装 nvidia 闭源驱动及工具

```shell
$ sudo apt install nvidia-driver nvidia-cuda-dev nvidia-detect nvidia-modprobe nvidia-support libcuda1 libcudart9.2 libcupti-dev
```

```shell
ln -s /usr/lib/nvidia-cuda-toolkit/libdevice /usr/share/cuda
```

**同时需要关闭系统的 security boot**


在 [nvidia](https://developer.nvidia.com/cudnn) 下载 cudnn, nvidia 官方发布的 深度神经网络 GPU 库, 执行

```shell
$ tar -xf cudnn[...].tgz
$ mv cuda /usr/local
```

```shell
$ bazel build --config=opt --config=cuda //tensorflow/tools/pip_package:build_pip_package


WARNING: The following configs were expanded more than once: [cuda]. For repeatable flags, repeats are counted twice and may lead to unexpected behavior.
Loading: 
Loading: 0 packages loaded
ERROR: Skipping '//tensorflow/tools/pip_package:build_pip_package': error loading package 'tensorflow/tools/pip_package': Encountered error while reading extension file 'cuda/build_defs.bzl': no such package '@local_config_cuda//cuda': Traceback (most recent call last):
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 1265
		_create_local_cuda_repository(repository_ctx)
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 1045, in _create_local_cuda_repository
		copy_rules.append(make_copy_dir_rule(repository_ct..."))
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 1045, in copy_rules.append
		make_copy_dir_rule(repository_ctx, name = "cuda-bin", s..."), ...")
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 923, in make_copy_dir_rule
		_read_dir(repository_ctx, src_dir)
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 956, in _read_dir
		_execute(repository_ctx, ["find", src_dir, ..."], ...)
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 887, in _execute
		auto_configure_fail("\n".join([error_msg.strip() if ... ""]))
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 324, in auto_configure_fail
		fail(("\n%sCuda Configuration Error:%...)))

Cuda Configuration Error: Repository command failed

WARNING: Target pattern parsing failed.
ERROR: error loading package 'tensorflow/tools/pip_package': Encountered error while reading extension file 'cuda/build_defs.bzl': no such package '@local_config_cuda//cuda': Traceback (most recent call last):
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 1265
		_create_local_cuda_repository(repository_ctx)
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 1045, in _create_local_cuda_repository
		copy_rules.append(make_copy_dir_rule(repository_ct..."))
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 1045, in copy_rules.append
		make_copy_dir_rule(repository_ctx, name = "cuda-bin", s..."), ...")
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 923, in make_copy_dir_rule
		_read_dir(repository_ctx, src_dir)
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 956, in _read_dir
		_execute(repository_ctx, ["find", src_dir, ..."], ...)
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 887, in _execute
		auto_configure_fail("\n".join([error_msg.strip() if ... ""]))
	File "/home/zhan/workspace/tensorflow/third_party/gpus/cuda_configure.bzl", line 324, in auto_configure_fail
		fail(("\n%sCuda Configuration Error:%...)))

Cuda Configuration Error: Repository command failed

```

to be continue...

### nvidia-docker

