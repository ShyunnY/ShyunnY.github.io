---
date: '2025-01-29'
lastmod: '2025-02-24'
draft: false
title: 'Linux 内核升级 (Ubuntu)'
categories: ['linux']
tags: ['实践'] 
summary: "Linux kernal 升级实践"
---

背景:

最近我在开发一个 ebpf 程序的时候发现当前的 ubuntu kernal 版本过于低了(`5.4.0`), 因此我希望能够原地将内核版本升级到(`5.15.0`)

### 查看内核版本

我们可以通过以下命令查看当前的内核版本以及发行版
```shell
$ uname -r
# 5.4.0-31-generic

$ lsb_release -d
# Description:    Ubuntu 20.04.6 LTS
```

### 查看当前可用的内核版本

每个 Ubuntu 版本的支持内核版本可能会有所不同, 我们需要查看当前 ubuntu 可用的内核版本

> Ubuntu 的每个版本都会选择一个或多个特定的内核版本来搭配发布。虽然 Ubuntu 是基于 Linux 内核构建的, 但它不会在每个版本中都使用最新的内核版本。

```shell
$ sudo apt-get update
$ sudo apt-cache search linux-image

```

这将会列出多个版本, 我们可以根据需求下载对应的版本。 这里我选择的是 `5.15.0-113-generic` 版本。

### 更新内核

执行以下命令等待安装完成后再执行重启命令即可完成内核升级
```shell
$ sudo apt-get install linux-headers-5.15.0-113-generic
$ sudo apt-get install linux-image-5.15.0-113-generic
$ sudo apt-get install linux-modules-5.15.0-113-generic 
$ sudo apt-get install linux-modules-extra-5.15.0-113-generic

$ sudo reboot
```

### 查看结果

此更新仅仅是更新 kernal 的版本, 针对发行版是不变的
```shell
$ uname -r
# 5.15.0-113-generic

$ lsb_release -d
# Description:    Ubuntu 20.04.6 LTS
```

---

参考博客:

1. [Ubuntu内核的查看、更新、卸载、取消及启用自动更新](https://blog.csdn.net/Explorer_XZH/article/details/129395789)