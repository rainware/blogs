---
layout: post
title:  "apscheduler学习笔记"
date:   2016-03-30 15:42:00 +0800
category: 学习笔记
tagline: "Supporting tagline"
tags: [python, scheduler, apscheduler]
---

apscheduler是python中的一个定时框架, 借鉴了Java的Quartz. 文中简称APS

[官方文档](http://apscheduler.readthedocs.org/en/latest/userguide.html)

研究之前的几个问题:

> * 如何维护已经创建的定时任务: 更新/删除等
> * 程序重启后, 如何加载已存在的定时任务, 已过期的定时任务怎么处理

读过相关文档后, 发现应该可以通过APS中的job store来实现.

# 1. 基本概念

> * triggers: 触发器, 即定时任务的触发逻辑, 来决定任务什么时间执行
> * job stores: 用来存储创建的定时任务. 默认是内存存储, 还支持sql, mongo等数据库
> * executors: executors通常是一个线程/进程池, 用来执行任务
> * schedulers: schedulers是对triggers, job stores, executors各组件的一个聚合或者封装, 用户并不直接操作这些组件, 而是通过scheduler提供的接口函数来定制和管理任务.

APS提供了适用于不同场景的scheduler, 我在这里打算在`tornado`中使用, 选用TornadoScheduler.

job store有更多的选择, 官方推荐使用SQLAlchemyJobStore + PostgreSQL, 因为其比较强大的数据完整性保护机制. 而这里我选用mongo

executors可以选用线程池和进程池, 通常使用线程池就可以了, 如果你的程序是CPU密集型, 请选用进程池以更有效的利用多核, 也可以两者并用.
