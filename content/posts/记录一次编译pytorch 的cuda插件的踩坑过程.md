---
date: 2024-12-11T04:09:30Z
tags:
  - pytorch
  - cuda
  - 编译
title: 记录一次编译pytorch 的cuda插件的踩坑过程
share: true
canonicalURL: 
keywords:
  - 关键字1
  - 关键字2
description: 
series: 系列
lastmod: 
lang: cn
cover:
  image: 
author: qqq
dir: posts
---


今天在尝试复现quip#的时候发现了问题，在进行setup install的时候发现
``` bash
[1/3] /usr/local/cuda/bin/nvcc --generate-dependencies-with-compile --dependency-output /home/ubuntu/data/exp/quip-sharp/quiptools/build/temp.linux-x86_64-cpython-311/quiptools.o.d -I/home/ubuntu/miniconda3/envs/quant/lib/python3.11/site-packages/torch/include -I/home/ubuntu/miniconda3/envs/quant/lib/python3.11/site-packages/torch/include/torch/csrc/api/include -I/home/ubuntu/miniconda3/envs/quant/lib/python3.11/site-packages/torch/include/TH -I/home/ubuntu/miniconda3/envs/quant/lib/python3.11/site-packages/torch/include/THC -I/usr/local/cuda/include -I/home/ubuntu/miniconda3/envs/quant/include/python3.11 -c -c /home/ubuntu/data/exp/quip-sharp/quiptools/quiptools.cu -o /home/ubuntu/data/exp/quip-sharp/quiptools/build/temp.linux-x86_64-cpython-311/quiptools.o -D__CUDA_NO_HALF_OPERATORS__ -D__CUDA_NO_HALF_CONVERSIONS__ -D__CUDA_NO_BFLOAT16_CONVERSIONS__ -D__CUDA_NO_HALF2_OPERATORS__ --expt-relaxed-constexpr --compiler-options ''"'"'-fPIC'"'"'' -O2 -g -Xcompiler -rdynamic -lineinfo -std=c++20 -DTORCH_API_INCLUDE_EXTENSION_H '-DPYBIND11_COMPILER_TYPE="_gcc"' '-DPYBIND11_STDLIB="_libstdcpp"' '-DPYBIND11_BUILD_ABI="_cxxabi1011"' -DTORCH_EXTENSION_NAME=quiptools_cuda -D_GLIBCXX_USE_CXX11_ABI=0 -gencode=arch=compute_89,code=compute_89 -gencode=arch=compute_89,code=sm_89

FAILED: /home/ubuntu/data/exp/quip-sharp/quiptools/build/temp.linux-x86_64-cpython-311/quiptools.o

/usr/local/cuda/bin/nvcc --generate-dependencies-with-compile --dependency-output /home/ubuntu/data/exp/quip-sharp/quiptools/build/temp.linux-x86_64-cpython-311/quiptools.o.d -I/home/ubuntu/miniconda3/envs/quant/lib/python3.11/site-packages/torch/include -I/home/ubuntu/miniconda3/envs/quant/lib/python3.11/site-packages/torch/include/torch/csrc/api/include -I/home/ubuntu/miniconda3/envs/quant/lib/python3.11/site-packages/torch/include/TH -I/home/ubuntu/miniconda3/envs/quant/lib/python3.11/site-packages/torch/include/THC -I/usr/local/cuda/include -I/home/ubuntu/miniconda3/envs/quant/include/python3.11 -c -c /home/ubuntu/data/exp/quip-sharp/quiptools/quiptools.cu -o /home/ubuntu/data/exp/quip-sharp/quiptools/build/temp.linux-x86_64-cpython-311/quiptools.o -D__CUDA_NO_HALF_OPERATORS__ -D__CUDA_NO_HALF_CONVERSIONS__ -D__CUDA_NO_BFLOAT16_CONVERSIONS__ -D__CUDA_NO_HALF2_OPERATORS__ --expt-relaxed-constexpr --compiler-options ''"'"'-fPIC'"'"'' -O2 -g -Xcompiler -rdynamic -lineinfo -std=c++20 -DTORCH_API_INCLUDE_EXTENSION_H '-DPYBIND11_COMPILER_TYPE="_gcc"' '-DPYBIND11_STDLIB="_libstdcpp"' '-DPYBIND11_BUILD_ABI="_cxxabi1011"' -DTORCH_EXTENSION_NAME=quiptools_cuda -D_GLIBCXX_USE_CXX11_ABI=0 -gencode=arch=compute_89,code=compute_89 -gencode=arch=compute_89,code=sm_89

/usr/include/x86_64-linux-gnu/bits/floatn-common.h(214): error: invalid combination of type specifiers

  typedef float _Float32;

                ^

  

/usr/include/x86_64-linux-gnu/bits/floatn-common.h(251): error: invalid combination of type specifiers

  typedef double _Float64;
```
事实上只要清理干净使用的gcc和g++版本
```bash
ls /usr/bin/gcc*
ls /usr/bin/g++*

sudo apt remove gcc-10 gcc-13 g++-10 g++-13

# 通常ubuntu20.04使用9.4版本的gcc和g++
# 手动创建这些链接
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 60
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-9 60
# 设置对应版本
sudo update-alternatives --config gcc
sudo update-alternatives --config g++

```

