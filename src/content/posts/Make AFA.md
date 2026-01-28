---
title: 使用ZMK制作一个仿怒喵AFA分体Alice键盘
published: 2025-04-10
description: 使用ZMK构建一个分体Alice键盘吧！
tags: [ZMK, Keyboard]
image: "https://cdn-www.angrymiao.com/am_afa/am_afa_swiper_02_03.jpg"
category: Keyboard
draft: false
---

## 前言

怒喵的AFA是真的帅啊...但是价格太贵了。我决定自己做一个。这是一个烂尾的项目，项目地址在这里：[Dafeigy/NUl4ice: Alice layout keyboard using zmk firmware.](https://github.com/Dafeigy/NUl4ice)

## 主控选择

NRF52840兼容 `NiceNano! v2`，无名科技的[产品介绍页](https://www.nologo.tech/product/otherboard/NRF52840.html#产品参数)中有关于Promicro nrf52840的介绍。引脚图如下：

![ProMicroNRF52840](https://www.nologo.tech/assets/img/other/NRF52840/ProMicroNRF52840Foot.jpg)

从中可以看到，一共有 18 个GPIO口可以给我们使用，并且有8个是高速的。构建一个键盘重要的是看开发板的可用GPIO引脚个数，因为键盘需要的引脚数为**最大列数 + 最大行数**。 以最入门的分体键盘 [Corne36](https://github.com/foostan/crkbd) 为例：

![corne36 layout by keymap editor](https://s2.loli.net/2025/04/05/3fjoSJbap6eNGKv.png)

每一边的最大行数为4，最大列数为6，因此总共需要6+4=10个GPIO进行开发，我们也可以在Corne的原理图中证实这一点:

![原理图](https://pic1.imgdb.cn/item/67f1eabe0ba3d5a1d7ee291c.png)

注意到row一共有4个，col一共有6个，分别对应到了ProMicro的15-20以及7-10的GPIO口，一共十个。

### 布局参考

当然第一步要考虑布局啦。用Alice就是为了用的舒服嘛，键盘布局生成网站是 [Keyboard Layout Editor](https://www.keyboard-layout-editor.com/#/) （KLE），当然我不会自己去从头弄一个Alice布局，一来我没时间做测量（其实是不会呃呃呃），二来我本身就是用Alice配列，所以直接参考一些别人现有的工作即可。

- [4nh3k/TrongDong40: An open-source 40% Alice-like keyboard](https://github.com/4nh3k/TrongDong40) 的 40%配列Alice键盘

  ![TrongDong40-Alice-layout](https://s2.loli.net/2025/04/05/RaK3WpDw4Z2ul8X.png)

- 来自[Owlab Spring Style Alice Layout Keyboard](https://gist.github.com/twelvehouse/2a733f2ac6324826e8862a4405cd2b82) 分享的Owlab Spring的经典Alice 布局：

  ![Spring-alice-layout](https://s2.loli.net/2025/04/05/QTrRs2EiZFGCKHq.png)

主要参考这两个，其他的一些Alice布局我个人感觉不具备美感，或者是最底行的按键布局我使用起来很不习惯，所以不纳入考虑。善用搜索可以找到很多分享键盘布局KLE json文件的用户。

## ZMK固件编写

对于RGB以及LED，分体式的键盘需要单独开启宏。对于像Corne这种底灯和背光灯共用一个针脚的，需要在`.overlay`文件中修改WS2812的`chain-length`长度，这样才可以共用所有的灯效。

## ZMK固件编写

参考：[New Keyboard Shield | ZMK Firmware](https://zmk.dev/docs/development/hardware-integration/new-shield)。主要步骤如下：

- 创建一个新的包含新的Shield的 ZMK 模块。
- 创建一个新的Shield目录。
- 添加基础 Kconfig 文件。
- 添加Shield层叠加文件（`*.overlay`）定义：
  - 键盘扫描驱动程序以检测按键/释放。
  - 矩阵转换以将键盘扫描行/列值映射到键映射中的键位置。
  - 物理布局定义以选择矩阵转换和键盘扫描实例。
- 添加一个默认的键盘映射，用户可以根据需要在自己的配置中覆盖。
- 添加一个 `<my_shield>.zmk.yml` 元数据文件来记录shield的高级细节以及它支持的功能。

在正式开始前，可以先看看[ Zephyr Shields ](https://docs.zephyrproject.org/3.5.0/hardware/porting/shields.html#shields)中关于Shield的定义。Shield可以附加到一个板上以扩展其功能和服务，便于模块化的原型设计。在 Zephyr 中，Shield功能提供了 Zephyr 格式的Shield描述，以便与应用程序更容易地兼容。Shield的配置文件位于board下的文件夹中：

```bash
boards/shields/<shield>
├── <shield>.overlay
├── Kconfig.shield
└── Kconfig.defconfig
```

这些文件提供了shield配置如下：

- **<shield>.overlay**:此文件提供了一种Shield描述，以设备树格式呈现，并在编译前与板载设备树合并。
- **Kconfig.shield**：此文件定义了Shield的Kconfig 符号，用于默认Shield配置。为了方便应用程序使用，此处的默认Shield配置应与编写设备树中的配置保持一致。
- Kconfig.defconfig：此文件定义了默认Shield配置。它旨在与编写设备树中的配置保持一致。因此，Shield配置应考虑到功能激活是应用程序的责任。

分体式键盘的构建需要在`app/boards/shields`中新建一个文件夹`NULice`用来存放相关文件。这个文件夹的文件结构如下：

```bash
NULice
├── Kconfig.defconfig
├── Kconfig.shield
├── boards
│   ├── nice_nano.overlay
│   └── nice_nano_v2.overlay
├── NULice.conf
├── NULice.dtsi
├── NULice.keymap
├── NULice.zmk.yml
├── NULice_left.conf
├── NULice_left.overlay
├── NULice_right.conf
└── NULice_right.overlay

2 directories, 12 files
```

每个文件的作用将会被说明。

### Kconfig.shield

`Kconfig.shield` 文件定义了用于构建键盘的 shield 名称。分体键盘定义了多个 shield 名称，每个部分一个。例如，如果键盘由名为 `my_keyboard_left` 和 `my_keyboard_right` 的两部分组成，它将如下所示：

```c
# No whitespace after the comma or in your part name!
config SHIELD_MY_KEYBOARD_LEFT
    def_bool $(shields_list_contains,my_keyboard_left)

# No whitespace after the comma or in your part name!
config SHIELD_MY_KEYBOARD_RIGHT
    def_bool $(shields_list_contains,my_keyboard_right)
```

注意在末尾没有分号。这样将会在使用 `SHIELD_MY_KEYBOARD_LEFT` 作为Shield名称时将 `y` 标志设置为 `my_keyboard_left` 。同样，当使用 `my_keyboard_right` 作为Shield名称时， `SHIELD_MY_KEYBOARD_RIGHT` 标志将被设置为 `y` 。 `SHIELD_MY_KEYBOARD_LEFT` 和 `SHIELD_MY_KEYBOARD_RIGHT` 标志将在 `Kconfig.defconfig` 中用于设置其他Shield的属性，因此请确保它们匹配。

### Kconfig.defconfig

`Kconfig.defconfig` 文件用于设置此Shield使用时的新默认配置设置。通常在此处设置一个新的默认值是 `ZMK_KEYBOARD_NAME` 值，它控制设备通过 USB 和 BLE 的显示名称。更新后的新的默认值应该始终包含在 `Kconfig.shield` 文件中定义的Shield配置名称的条件中。对于分体键盘，中央一侧（通常是左侧）通过此文件中的配置来指定。对于该侧，分配键盘名称并设置中央配置。外围侧不分配名称。最后，需要为两侧设置分体配置：

```c
// Kconfig.defconfig
if SHIELD_MY_KEYBOARD_LEFT

# Name must be less than 16 characters long!
config ZMK_KEYBOARD_NAME
    default "My Keyboard"

config ZMK_SPLIT_ROLE_CENTRAL
    default y

endif

if SHIELD_MY_KEYBOARD_LEFT || SHIELD_MY_KEYBOARD_RIGHT

config ZMK_SPLIT
    default y

endif
```

### Shield Overlays

Shield overlay 文件包含一个设备树描述，该描述在固件构建过程中与主板设备树描述合并。此文件中需要定义三件事：

- 键盘扫描（kscan）驱动程序，决定了哪些 GPIO 引脚用于检测按键事件
- 矩阵转换，充当“桥梁”，连接 kscan 和键帽映射
- 物理布局，聚合上述内容，并且（可选地）定义物理键位，以便键盘可以与 ZMK Studio 一起使用。

分体键盘将为每个分体部分定义一个覆盖文件。例如，如果键盘被分成左半部分和右半部分，这些文件可以命名为：

- `my_keyboard_left.overlay`
- `my_keyboard_right.overlay`

这里 `my_keyboard_left` 和 `my_keyboard_right` 是 Kconfig.shield 文件中定义的 shield 名称。分体键盘 often 共享一些他们的 devicetree 描述。标准的方法是有一个核心 `my_keyboard.dtsi` (device tree include) 文件，该文件被包含到每个Shield

Overlay中。

### Kscan

kscan 节点定义了用于扫描按键按下和释放事件的控制器 GPIO 引脚。对于NiceNano或ProMicro而言，其引脚定义如下:

![Promicro](https://zmk.dev/assets/images/pinout-4ed4b6eb1e452a7be44c3a0143cd5605.png)



ZMK使用的是Arduino的引脚命名规则。ZMK 使用蓝色编码的“Arduino”引脚名称来生成设备树节点引用。例如，在图中标记为 `0` 的引脚，在设备树文件中使用 `&pro_micro 0` 来引用。要使用上述互连之外的 GPIO 引脚，可以使用每个控制器类型特有的 GPIO 标签。例如，在基于 nRF52840 的板子上，编号为 `PX.Y` 的引脚可以通过 `&gpioX Y` 标签来引用。例如，nice!nano 板子中间的 `&gpio1 7` 引脚就是通过 `P1.07` 标签暴露出来的。你可以在[这里](https://zmk.dev/docs/config/kscan)查看Kscan的详细细节。

对于分体键盘，您应该在 `my_keyboard.dtsi` 中定义您的 kscan。如果您的 `row-gpios` 或 `col-gpios` （两者都）在两部分之间是相同的，那么它们也应该在 `my_keyboard.dtsi` 中定义。例如，对于一个 `col2row` 两部分的分体键盘（18 个按键分为两半，每半都是 3x3 的宏键盘），如果用于“行”的 GPIO 引脚在两半中都相同：

```c
// my_keyboard.dtsi
/ {
    kscan0: kscan0 {
        compatible = "zmk,kscan-gpio-matrix";
        diode-direction = "col2row";
        wakeup-source;

        row-gpios
            = <&pro_micro 6 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>
            , <&pro_micro 7 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>
            , <&pro_micro 8 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>
            ;

    };
};
```

缺少的 `col-gpios` 将在你的 `my_keyboard_left.overlay` 和 `my_keyboard_right.overlay` 文件中定义。

```c
// my_keyboard_left.overlay
#include "my_keyboard.dtsi" // The shared dtsi file is included in the overlay

// Label of the kscan node in the dtsi
&kscan0 {
    col-gpios
        = <&pro_micro 19 GPIO_ACTIVE_HIGH>
        , <&pro_micro 18 GPIO_ACTIVE_HIGH>
        , <&pro_micro 15 GPIO_ACTIVE_HIGH>
        ;
};
```

```c
// my_keyboard_right.overlay
#include "my_keyboard.dtsi" // The shared dtsi file is included in the overlay

// Label of the kscan node in the dtsi
&kscan0 {
    col-gpios
        = <&pro_micro 10 GPIO_ACTIVE_HIGH>
        , <&pro_micro 11 GPIO_ACTIVE_HIGH>
        , <&pro_micro 13 GPIO_ACTIVE_HIGH>
        ;
};
```

### 矩阵变换

矩阵变换用于将行/列事件转换为“键位”事件。当按键被按下时，会生成一个 kscan 事件，该事件带有 `row` 和 `column` 值，分别对应触发事件的 `row-gpios` 和 `col-gpios` 引脚的零基索引。然后，“键位”触发的是矩阵变换中 `RC(row, column)` 的位置，其中 `row` 和 `column` 是上述提到的索引。该键位将与键映射中的行为绑定相关联。

分体键盘应在共享 `my_keyboard.dtsi` 中定义其矩阵变换。在文件顶部添加 `#include <dt-bindings/zmk/matrix_transform.h>` 。以下是对上一个示例（18 键双宏键盘）的矩阵转换示例：

```c
// my_keyboard.dtsi
#include <dt-bindings/zmk/matrix_transform.h> // Put this with the other includes at the top of your dtsi

/ {
    default_transform: keymap_transform0 {
        compatible = "zmk,matrix-transform";
        columns = <6>;
        rows = <3>;
        map = <
        //  LKey 1 |LKey 2 |LKey 3      RKey 1 |RKey 2 |RKey 3
            RC(0,0) RC(0,1) RC(0,2)     RC(0,3) RC(0,4) RC(0,5)
        //  LKey 4 |LKey 5 |LKey 6      RKey 4 |RKey 5 |RKey 6
            RC(1,0) RC(1,1) RC(1,2)     RC(1,3) RC(1,4) RC(1,5)
        //  LKey 7 |LKey 8 |LKey 9      RKey 7 |RKey 8 |RKey 9
            RC(2,0) RC(2,1) RC(2,2)     RC(2,3) RC(2,4) RC(2,5)
        >;
    };
};
```

上述变换有 6 列和 3 行，而键盘的每一半只有 3 列和 3 行。为了使外设的矩阵变换能够与 kscan 矩阵连接，在外设的矩阵变换中应用了一个偏移量:

```c
// my_keyboard_right.overlay
&default_transform { // Offset of 3 because the left side has 3 columns
    col-offset = <3>;
};
```

这个偏移量意味着当键盘右半部分由其 `row-gpios` 和 `col-gpios` 数组中索引 `0,0` 的 GPIO 引脚触发键事件时，它会被解释为 `RC(0,3)` 事件而不是 `RC(0,0)` 事件。额外的外设需要其列偏移量等于中央部分的列数与之前所有外设列数之和。你还可以使用 `row-offset` 应用行偏移量。

矩阵变换也被用来“纠正”针脚排序，使其更接近键的实际排列顺序。针脚排序异常的原因包括：

- 为了减少使用的针脚，使用了一个“高效”的 GPIO 矩阵行/列数，这并不匹配实际键开关行/列的实际布局。
- 对于非矩形键盘、拇指簇、非 `1u` 位置等.

ZMK 定义了一些内置键盘以了解更复杂的矩阵转换，可以在[这里](https://github.com/zmkfirmware/zmk/tree/main/app/boards/shields)查看下。

### 默认键盘布局

每个键盘应该提供一个默认键位布局，当构建固件时使用，用户可以通过自定义配置文件覆盖和自定义。对于“Shield键盘”，这应该放在 `boards/shields/my_keyboard/my_keyboard.keymap` 文件中。这是一个仅有一层的 3x3 宏键盘的简单键位映射示例：

```c
// my_keyboard.keymap
#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>

/ {
    keymap {
        compatible = "zmk,keymap";

        default_layer { // Layer 0
            // -------------------------------------
            // |     Z     |     M     |     K     |
            // |     A     |     B     |     C     |
            // |     D     |     E     |     F     |
            bindings = <
                &kp Z    &kp M    &kp K
                &kp A    &kp B    &kp C
                &kp D    &kp E    &kp F
            >;
        };
    };
 };
```

键映射应与矩阵转换中键的顺序完全匹配，从左到右，从上到下（它们都是用换行字符重新排列的一维数组，以便更易于阅读）。有关在 ZMK 中定义键映射的信息，请参见键映射。如果您希望使用 ZMK Studio 来配置您的键盘，请确保在键映射中为 ZMK Studio 的解锁行为分配一个键。



## 踩坑

连接问题：

### 蓝牙连接列表中无法连接

一般的说法是

### 蓝牙识别的设备名称没有变更

改名：改名一般情况下需要重新刷一下重置固件。具体做法是在编译正式的固件前先编译一个reset固件：

```bash
west build -d ../build -p -b nice_nano_v2 -- -DSHIELD=settings_reset
```

即`-DSHIELD=settings_reset`这一块。烧录Reset固件后再进行正式的新键盘固件烧录，这样就能在蓝牙中显示正确的名称了。

Reference：[ZMK文档](https://zmk.dev/docs/troubleshooting/connection-issues#:~:text=These%20issues%20can%20be%20resolved%20by%20flashing%20a,Bluetooth%20profiles%2C%20output%20selection%2C%20RGB%20underglow%20color%2C%20etc.)和[AnswerOverFlow](https://www.answeroverflow.com/m/1307444107098062938)。
