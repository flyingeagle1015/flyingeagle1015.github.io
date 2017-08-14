---
title: ssh远程服务器文件修改
date: 2017-08-12 22:11:25
tags: [ssh]
categories: [application]
---

在工作中时常需要修改远程服务器上的文件，直接用ssh登录远程服务器进行修改是最简单直接的,  
但比如要临时修改服务器上的代码（不提倡这么做，这里只是举例）做调试之类的，需要语法高亮  
难道还在服务器上配置一番???  
在本地配置好的环境进行修改当然是最方便的了，那如何在本地修改远程服务器上的代码呢？  
常用的有这几个：  
1. 如果服务器有配置Samba, 采用磁盘映射的方式  
2. 如果文件少，直接用vim远程编辑, 命令：vim scp://hostname//path/to/file  
3. 本地安装sshfs, 用sshfs映射到本地进行编辑  
   挂载:  
     sshfs username@serverip:serverpath localpath // 如果配置ssh key可以实现自动化挂载  
   卸载:  
     fusermount -u mountpath  
4. 本地修改，采用rsync同步到远程服务器  
