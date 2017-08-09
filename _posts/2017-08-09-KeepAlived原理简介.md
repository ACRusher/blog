---
layout: post
title: keepalived 高可用软件
categories: [blog ]
tags: [高可用, ]
description: 架构设计
---

# keepalived 简介

## 预备知识 VRRP

> VRRP(Virtual Router Redundancy Protocol) 虚拟路由冗余协议是为了实现路由器高可能的虚拟路由协议;<br/>
> VRRP 将多个路由器组成一个路由器组，对外提供虚拟路由器IP、虚拟MAC，组内会有一个Master、多个BackUP，Master负责处理虚拟路由器IP、虚拟MAC对应的ARP、ICMP、数据转发等功能;<br/>
> Backup只接受Master的状态通告，当Master失效的时候，会尝试去接管Master的网络功能;<br/>
> 路由器组中每个路由器都需要配置路由器组的ID（用来归为一组）、自身的权重（用户竞争Master）。

[VRRP协议参考资料](http://www.jianshu.com/p/4a0c9da69a1d)

## keepalived 原理

keepalived是在VRRP协议上改造的暂时没空整理，先放个不错的总结文章。

[原理参考](http://www.userzr.com/57.html)

