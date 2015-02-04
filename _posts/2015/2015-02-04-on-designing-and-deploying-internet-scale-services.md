---
layout: post
title: 可扩展互联网服务的设计和部署【译】
categories: tech
tags: 
- architecture
---

> 这篇译文源自当时还在微软`Windows Live Services Platform`部门的[James Hamilton](http://www.mvdirona.com/jrh/work/)的经典论文 [On Designing and Deploying Internet-Scale Services](https://www.usenix.org/legacy/event/lisa07/tech/full_papers/hamilton/hamilton_html/)，他现在是Amazon Web Services团队的成员之一，读了一遍感觉还有很多不够清晰，遂决定把它翻译成中文，以便更深入的了解。

## 概述

系统和管理员的比例通常作为一个大概指标来衡量高扩展服务的管理成本。对于规划较小很少有自动化的服务这个比率可能会低至`2：1`，但在领先并高度自动化的服务，这个比率可能高达`2500：1`。在微软的服务里，`Autopilot`常被称作Windows Live搜索团队成就高系统管理员比率后面的魔法。既然自动化管理如此重要，最重要的因素其实是服务自身。是服务高效到自动化？什么是我们通常称作的运维友好？运维友好的服务要求很少的人工介入，并且大部分复杂异常的检测和恢复不需要管理的介入。本论文总结了来自MSN和Windows Live在大规模服务多年的最佳实践。

## 介绍
