---
title: A vim autocomplete plugin - YouCompleteMe installation guide
tags: 'vim, plugin, YouCompleteMe'
date: 2017-07-05 13:14:32
---


[YouCompleteMe](http://valloric.github.io/YouCompleteMe/#intro) (YCM) is a fast, powerful code completion engine for vim. But unlike the other vim plugins which are easy to install, YCM needs to complier to install. This article will briefly introduce the full installation guide of YCM.

<!-- more -->

## Prerequisites

Before you install the ycm, make sure the follow packages have been installed:

### Vim

Ensure that your version of Vim is at least **7.4.1578** and that is has support for Python 2 or Python 3 scripting. Enter `vim -version` in the terminal to check the vim version. Typpe`vim --version | grep python` to check the Python support. If a plus(+) not a minus (-) before Python2 or Python3, then the vim support Python. If not, follow the guide to install vim8.0 with Python support.

- Download the [vim](https://github.com/vim/vim) source

  `git clone https://github.com/vim/vim.git`

- Install some dependencies

  `sudo apt-get install libncurses5-dev python2.7-dev` 

- Configure and Install

  `cd vim && ./configure --enable-pythoninterp --with-python-config-dir=/usr/lib/python2.7/config-x86_64-linux-gnu`

  `make && make install`

### Vundle

Vundle is a vim plugin manager and follow the installion guide in [Vundle](https://github.com/VundleVim/Vundle.vim) to install.

## Install YCM

Install YCM with Vundle is a recommended way. WithVundle, adding a `Plugin 'Valloric/YouCompleteMe'` line to your [vimrc](http://vimhelp.appspot.com/starting.txt.html#vimrc) and execute `PluginInstall` in vim.

And the enter the YCM repository directory and run follow command to fetch the YCM's dependencies.

`git submodule update --init --recursive  `

## Install libclang

The semantic completion for C-family languages need libclang and you can jump this step if you do NOT need it.

### Using the pre-built binary

Download the pre-built binary from [llvm](http://releases.llvm.org/download.html) for your corresponding system. But sometimes this can be failed for some different dependencies version and you can build from the source. 

### Compile from source

- Download the source code from [llvm](http://releases.llvm.org/download.html)

  Download the llvm, clang, compiler-rt and clang-tools-extra packages and extract the source code.

  `wget http://releases.llvm.org/4.0.1/llvm-4.0.1.src.tar.xz`

  `wget http://releases.llvm.org/4.0.1/cfe-4.0.1.src.tar.xz`

  `wget http://releases.llvm.org/4.0.1/compiler-rt-4.0.1.src.tar.xz`

  `wget http://releases.llvm.org/4.0.1/clang-tools-extra-4.0.1.src.tar.xz`

- Move the source code

  `mv llvm-4.0.1.src llvm`

  `mv cfe-4.0.1.src llvm/tools/clang`

  `mv clang-tools-extra-4.0.1.src llvm/tools/clang/tools/extra`

  `mv compiler-rt-4.0.1.src llvm/projects/compiler-rt`

- Build the LLVM and clang

  `mkdir build && cd build`

  `cmake -G "Unix Makefiles" ../llvm`

## Compile the *ycm_core*

Before you compile the ycm_core, make sure that the [cmake](https://cmake.org/download/) has been installed.

- Create the build directory

  `mkdir ~/ycm_build && cd ~/ycm_build`

- Generate the makefiles

  - No semantic support

    `cmake -G "Unix Makefiles" . ~/.vim/bundle/YouCompleteMe/third_party/ycmd/cpp`

  - Semantic support

    `cmake -G "Unix Makefiles" -DPATH_TO_LLVM_ROOT=llvm_root_dir . ~/.vim/bundle/YouCompleteMe/third_party/ycmd/cpp`

    Where ***llvm_root_dir*** is the root dir where you extract the downloaded binary distribution.

- Compile

  `cmake --build . --target ycm_core --config Release`

Now you can find the ***ycm_core*** in ***~/.vim/bundle/YouCompleteMe/third_party/ycmd/ycm_core.so***

## Test Installation

Open a test file and enter some words to check whether it works.

![](/images/vim-ycm-install/test.png)

## Issues

### GLIBC Error

**I use the python from anaconda2 and there is an problem when I finish the installation.**

**`lib/libstdc++.so.6: version 'GLIBCXX_3.4.20' not found`**

Install the libgcc package to solve

`conda install libgcc`

### Python version

**I compile the vim using python in anaconda but ycm is unavaiable. The issue is`YouCompleteMe unavailable: dlopen(/Users/jack/anaconda2/lib/python2.7/lib-dynload/_io.so, 2) `**

Compile the vim using system python.