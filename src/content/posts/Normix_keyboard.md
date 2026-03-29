---
title: Normix_keyboard
published: 2026-03-18
description: '不如跟着Cornix复刻一个出来吧！'
image: 'https://pic1.imgdb.cn/item/69b981881eedde682494331a.png'
tags: ['keyboard']
category: 'keyboard'
draft: false 
lang: 'zh-cn'
---

## Normix：仿制Cornix的尝试

开发中。开发计划：

- [x] E73最小系统
- [x] E73 Bootloader烧录
- [x] 最小系统固件烧录与测试键盘功能验证
- [ ] 正式设计原理图与PCB

## ZMK固件编写

好的。既然是要复刻，那久尽量复刻吧，然后把一些细节的东西交给大家自行修改以适配自己需求。这里最需要提的其实是这个ZMK固件，Cornix的固件想都不用想，现在的个人项目的蓝牙固件基本都是ZMK改过来的。和使用NiceNano这种Promicro的整板方案不同，使用E73模块有些问题：

1. 引脚对不上。有些PIN在NiceNano里面引出来了，有些则没有。但总体来说，E73模块上的可用引脚还是更多一点的。
2. 天线净空区。为了更好的无线性能，模块天线区域下方的区域应该禁止铺铜，这和使用Promicro兼容的开发板不太一样。

对于引脚缺失的问题，其实可以通过修改`.overlay`文件中的kscan节点内容即可。kscan 节点定义了用于扫描按键按下和释放事件的控制器 GPIO 引脚，正常使用Promicro封装的话，是这样写的：

```bash
&kscan0 {
    col-gpios
        = <&pro_micro 14 GPIO_ACTIVE_HIGH>
        , <&pro_micro 15 GPIO_ACTIVE_HIGH>
        , <&pro_micro 18 GPIO_ACTIVE_HIGH>
        , <&pro_micro 19 GPIO_ACTIVE_HIGH>
        , <&pro_micro 20 GPIO_ACTIVE_HIGH>
        , <&pro_micro 21 GPIO_ACTIVE_HIGH>
        ;
};
```

为了方便对比，放一下Promicro封装的图：

![无名科技提供的Promicro引脚图](https://www.nologo.tech/assets/img/other/NRF52840/ProMicroNRF52840Foot.jpg)

然而，E73的原理图中，并没有Promicro对应的D15的P1.13引脚印出来：

![E73](https://pic1.imgdb.cn/item/69b97f871eedde6824943307.png)

解决方法也很简单：换个引脚就好。对于那些没有具体指向的，你可以使用`gpiox y`这样的来指定GPIOX.Y，如`gpio1 7`则对应`P1.07`。

## 最小系统固件烧录与测试键盘功能验证

烧录一个Bootloader。主要难点是不开启外部晶振的话，需要在全局设置中添加:

```bash
# prg.conf
CONFIG_CLOCK_CONTROL_NRF_K32SRC_RC=y
```

已验证，没问题。
