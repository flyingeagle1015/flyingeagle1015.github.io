---
title: ssh broken pipe
date: 2019-02-13 15:38:44
tags: [ssh]
categories: [application]
---

### Problem
I use Manjaro Linux at my workplace in a virtual machine hosted by VMWare Workstation. Today I am experiencing issues when I try to connect to servers via ssh. 
Every time I try to connect, SSH fails with
```
$ ssh -T git@github.com
packet_write_wait: Connection to xxx port 22: Broken pipe
```

### Solution
added "IPQoS throughput" to my ssh_config and now I can connect to all hosts again!
```
echo "IPQoS throughput" | sudo tee -a /etc/ssh/ssh_config // -a 是追加的意思，等同于 >>
```
