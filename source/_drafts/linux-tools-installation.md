---
title: Useful tools installation on Linux
tags: linux
---

This article lists some useful tools and their installation on linux platform.

## Vim

Vim is one of the most popular editor.

## ByPy

[Bypy](https://github.com/houtianze/bypy) is a python client for [Baidu Yun](https://pan.baidu.com). It can be used as client to synchronize the file with the Baidu Yun and it also can be used as a python package.

### Installation

`pip install bypy`

### Issues

- ` All server authorizations failed` when first authorizing the client?

  **Solution**: Follow the instructions the program gives. I fail to authorize on a vps but successfully on my Mac. So I copy the file in ~/.bypy/* to the vps and fix the problem.



## Proxychains-NG

[Proxychains-NG](https://github.com/rofl0r/proxychains-ng) is an Unix problem that hooks network-related libc functions  in DYNAMICALLY LINKED programs via a preloaded DLL (dlsym(), LD_PRELOAD)  and redirects the connections through SOCKS4a/5 or HTTP proxies.

