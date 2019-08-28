---
title: STM32系列教程
thumbnail: /gallery/STM32.jpg
date: 2019-08-27 15:38:12
tags: STM32
categories:
- STM32
---
# 项目介绍

&#160; &#160; &#160; &#160;本项目开始于2019年8月21日，并在不断更新。

&#160; &#160; &#160; &#160;面对目前主流库函数开发视频过于啰嗦、HAL库视频资源较少的情况下，旨在为大家提供CubeMX+MDK5+HAL库+库函数一站式入门学习的服务。

&#160; &#160; &#160; &#160;视频地址:[https://www.bilibili.com/video/av64690830](https://www.bilibili.com/video/av64690830)

<!--more-->

# 资源汇总

&#160; &#160; &#160; &#160;因为软件总是在不断升级换代的，所以希望大家能学会在官网出下载。

&#160; &#160; &#160; &#160;HAL库讲义:[点击这里下载](https://github.com/Reyunn/My-19-years-old)

&#160; &#160; &#160; &#160;MDK5下载:[点击这里下载](http://www2.keil.com/mdk5)

&#160; &#160; &#160; &#160;STM32支持包下载:[点击这里下载](http://www.keil.com/dd2/Pack/#/eula-container)

&#160; &#160; &#160; &#160;Cube MX下载:[点击这里下载](https://www.st.com/content/st_com/en/products/development-tools/software-development-tools/stm32-software-development-tools/stm32-configurators-and-code-generators/stm32cubemx.html)

# 项目分工

┌── 组织方: 电子科技协会,武汉理工大学
│  
├── STM32系列视频
│     │     └── 后期：甘云汉，电子科技协会
│     │ 
│     ├── 软件安装
│     │         │
│     │         ├── 【软件安装】[MDK和C51的共存](https://www.bilibili.com/video/av64690830/?p=1)
│     │         │            └── 作者：吕游，电子科技协会
│     │         │
│     │         ├── 【软件安装】[CubeMX开发环境的搭建及常见问题解决方法](https://www.bilibili.com/video/av64690830/?p=2)
│     │         │             └── 作者：卢意，电子科技协会
│     │         │
│     │         └── 【软件安装】[CubeMX点亮LED](https://www.bilibili.com/video/av64690830/?p=3)
│     │                       └── 作者：吕游，电子科技协会
│     │ 
│     ├── GPIO
│     │         │
│     │         ├── 【 GPIO-库函数】[闪烁的LED](https://www.bilibili.com/video/av64690830/?p=4)
│     │         │            └── 作者：李卓强，电子科技协会
│     │         │
│     │         ├── 【GPIO-HAL库】[游刃有余玩转IO](https://www.bilibili.com/video/av64690830/?p=5)
│     │         │             └── 作者：余阳，电子科技协会
│     │         │
│     │         └── 【GPIO-HAL库】[使用User Lable提高代码的重用性](https://www.bilibili.com/video/av64690830/?p=6)
│     │                       └── 作者：余阳，电子科技协会
│     │ 
│     ├── 串口通信
│     │         │
│     │         ├── 【 串口通信-库函数】[实现串口收发](https://www.bilibili.com/video/av64690830/?p=7)
│     │         │            └── 作者：李卓强，电子科技协会
│     │         │
│     │         ├── 【 串口通信-库函数】[回到stdio的世界](https://www.bilibili.com/video/av64690830/?p=8)
│     │         │            └── 作者：李卓强，电子科技协会
│     │         │
│     │         ├── 【串口通信-HAL库】[初学阻塞式收发](https://www.bilibili.com/video/av64690830/?p=9)
│     │         │             └── 作者：余阳，电子科技协会
│     │         │
│     │         ├── 【串口通信-HAL库】[优雅的打印Log信息](https://www.bilibili.com/video/av64690830/?p=10)
│     │         │            └── 作者：余阳，电子科技协会
│     │         │
│     │         ├── 【串口通信-HAL库】[个性化输出](https://www.bilibili.com/video/av64690830/?p=11)
│     │         │          └── 作者：余阳，电子科技协会
│     │         │
│     │         └── 【串口通信-HAL库】[Log信息终极解决方案](https://www.bilibili.com/video/av64690830/?p=12)
│     │                    └── 作者：余阳，电子科技协会
│     │ 
│     ├── 外部中断
│     │         │
│     │         ├── 【 外部中断-库函数】[手摸手带你学习外部中断](https://www.bilibili.com/video/av64690830/?p=13)
│     │         │        └── 作者：卢意，电子科技协会
│     │         │
│     │         └── 【 外部中断-HAL库】[巧妙地测量pwm频率](https://www.bilibili.com/video/av64690830/?p=14)
│     │                   └── 作者：余阳，电子科技协会
│     │ 
│     ├── 时钟树
│     │         │
│     │         └── 【时钟树-CubeMX】[时钟树的基操](https://www.bilibili.com/video/av64690830/?p=15)
│     │                   └── 作者：余阳，电子科技协会
│     │ 
│     ├── 定时器
│     │         │
│     │         ├── 【 定时器-HAL库】[定时器中断+平滑滤波](https://www.bilibili.com/video/av64690830/?p=16)
│     │         │            └── 作者：余阳，电子科技协会
│     │         │
│     │         ├── 【定时器-HAL库】[PWM的输出](https://www.bilibili.com/video/av64690830/?p=17)
│     │         │             └── 作者：余阳，电子科技协会
│     │         │
│     │         └── 【定时器-HAL库】[SPWM的输出](https://www.bilibili.com/video/av64690830/?p=18)
│     │                       └── 作者：余阳，电子科技协会
│     │ 
│     ├── ADC
│     │         │
│     │         ├── 【 ADC-HAL库】[轮询方式读取电压值](https://www.bilibili.com/video/av64690830/?p=19)
│     │         │        └── 作者：余阳，电子科技协会
│     │         │
│     │         └── 【 ADC-HAL库】[DMA方式多通道采集](https://www.bilibili.com/video/av64690830/?p=20)
│     │                   └── 作者：余阳，电子科技协会
│     │ 
│     ├── DAC
│     │         │
│     │         └── 【DAC-HAL库】[数模转换](https://www.bilibili.com/video/av64690830/?p=21)
│     │                   └── 作者：余阳，电子科技协会
