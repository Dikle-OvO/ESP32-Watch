---
title: ESP32-Watch|记录
date: 2024-04-04 11:43:14
cover: cover.jpg
tags:
  - esp32
  - 教程
---
# 前期准备
+ 面包板
+ esp32S3核心板
+ FPC转接板
+ 一块屏幕
+ 带数据传输的Type-c线

笔者使用的屏幕是P169H002,像素为240*320,其驱动IC为<b>[ST7789V](https://www.semiee.com/file/Sitronix/Sitronix-ST7789V.pdf)</b>，触摸IC为<b>[CST816D](https://www.semiee.com/bcaf4ca5-4b36-4d2a-844b-4576b6d56750.html)</b>，主控型号为ESP32S3-N16R8，采用<b>vscode+esp-idf</b>的方式进行软件开发。本文章也是基于此撰写。 

资料查取网站：
<b>[乐鑫组件库](https://components.espressif.com/)</b>、<b>[乐鑫IoT官方文档](https://docs.espressif.com/projects/esp-iot-solution/zh_CN/latest/display/lcd/spi_lcd.html#interface-i-ii)</b>、<b>[乐鑫开源代码库](https://gitee.com/EspressifSystems/esp-iot-solution)</b>

## 硬件手册勘察
根据屏幕的技术手册，确定其用的是四线SPI模式0，颜色深度为16bit，命令位数为8bit。
触控IC则支持最高400khz的工作频率，支持内部上拉

## 管脚配置
注意不要使用<b>strapping</b>引脚，上电时候此类引脚的电平状态会决定芯片的启动模式。  
![参考引脚选择](image.png)
## 驱动配置
在<b>[乐鑫组件库](https://components.espressif.com/)</b>中搜索lcd驱动,笔者使用esp_lcd_ili9341,下载源代码后，在其头文件中检查其默认配置
![检查组件配置](image-1.png)    
![查阅手册相关参数](image-2.png)  
再继续查看乐鑫组件库lcd的命令配置和芯片的是否一致，组件库路径为<b>ESP_IDF\esp-idf\components</b>
![乐鑫组件库lcd](image-4.png)
总结：直接使用默认参数即可

# 配置工程
经过前期的准备，我们确认了所使用的管脚，驱动的兼容性后就可以正式开始了。
## 新建工程
我们并不需要从头开始配置一个工程文件，乐鑫已经为我们准备了，位于<b>ESP_IDF\esp-idf\examples\get-started\sample_project</b>，同时里面还有很多有用的例程，比如lcd，我们后续会提到。
+ idf_component.yml:存放于main文件夹中，记录了工程所依赖的库，编译时会根据此配置库
  ![本次工程所用到的库](image-5.png)
## 照猫画虎
<b>ESP_IDF\esp-idf\examples\peripherals\lcd\spi_lcd_touch</b>即是乐鑫官方带有触控的lcd例程，我们可参考此例程建立自己的工程。
+ 头文件（VScode会错误地提示头文件未找到：ctrl+shift+p使用add vscode configuration floder命令可修复）
+ 全局静态变量TAG
+ 配置管脚宏定义
+ 使用结构体初始化SPI（可使用esp_lcd_ili9341驱动头文件定义的宏）
+ 创建LCD屏幕的IO句柄（handle），并配置lvgl的回调函数
+ 创建LVGL的结构体配置和定义显示驱动的参数
+ 将LCD屏幕连接到SPI总线
+ 配置屏幕驱动：创建LCD屏的句柄，使得应用程序能够利用LCD的通用API操作LCD设备，创建LCD屏幕的初始化数组，
+ 配置触控驱动：我们参考[官方示例](https://gitee.com/EspressifSystems/esp-iot-solution/tree/master/examples/display/lcd/qspi_with_ram)即可，步骤和LCD屏幕差不多
+ 初始化LVGL图形库：初始化堆内存作为显存，为LVGL注册显示驱动、触屏驱动
+ 初始化LVGL定时器
+ 创建LVGL任务（推荐为LVGL单独创建任务）

### CMakeLists
修改main文件夹中的CMakeLists.txt为：
```
#Add sources from ui directory
file(GLOB_RECURSE SRC_UI ${CMAKE_SOURCE_DIR} "ui/*.c")

idf_component_register(SRCS "main.c" ${SRC_UI}
                    INCLUDE_DIRS "." "ui")
```

## 配置Ui界面
### SquareLine_Studio简述
我们选用SquareLine_Studio作为编辑Ui所用的软件，它是LVGL官方推出的可视化Ui编辑软件，所见即所得。  
它唯一不好的是近年商业化程度加深，商用需要购买License，个人免费版本有最大5个屏幕界面，50个组件的限制。
### 建立Ui工程
新建一个工程，框架选用Idf,在其中的，修改颜色深度为16bit-swap。如需添加图片、中文字体，需将相关文件添加到你的工程文件夹下的assets中。  
在此推荐一个<b>[字体网站](https://www.100font.com/)</b>，个人喜欢<b>[香萃刻宋](https://www.100font.com/thread-628.htm)</b>

### 将ui导入工程文件
在main中创建ui文件夹，SquareLine_Studio只导出ui，记得包含头文件"ui/ui.h",在任务中运行ui_init即可运行ui界面。

### 关于触摸事件回调函数
### 时间显示
同样使用回调函数。初始化：
```
void ui_timer_init()
{
    lv_timer_t *timer_clock = lv_timer_create(ui_clock_update, 1000, NULL);
}
```
刷新时间的回调函数：

### GIF的播放
+ 配置工程文件
+ 压缩GIF
+ 将GIF转换为可编译的C文件



## 配置工程选项
### 分区表
运行LVGL需要用到更多的闪存空间，我们要手动更改分区表以增加应用程序分区的大小。工程文件夹中新建partitions.csv文件，内容如下：
```
# Name,   Type, SubType, Offset,  Size, Flags
# Note: if you have increased the bootloader size, make sure to update the offsets to avoid overlap
nvs,      data, nvs,        0x10000,   0x6000,
phy_init, data, phy,               ,   0x1000,
factory,  app,  factory,           ,   0x2c0000,

```
### sdk configuration
此设置在左下角齿轮。
+ 搜索partition table，将分区方式更改为自定义分区表
+ Flash size：增大为4MB
+ LV_COLOR_16_SWAP: 勾选
+ LV_FONT：18号
+ Screen_Transparency：勾选
优化内存分配机制：
+ LV_MEM_CUSTOM：勾选
+ LV_MEMCPY_MEMSET_STD：勾选
+ CONFIG_LV_ATTRIBUTE_FAST_MEM：勾选
搜索logging，将其使能，我们就可以通过检查log来排查问题。  
搜索FPS可打开性能检查，显示帧率和CPU占用率。

# 附加功能实现
## wifi连接
  参考esp-idf\examples\wifi\getting_started\station中的示例，缝纫进自己工程即可
## 网络同步时钟SNTP
SNTP(Simple Network Time Protocol)，是NTP的简化版，旨在提供基本的时间服务(相较于SsNTP，NTP提供更精确的时间同步，包括时钟偏移和漂移调整,对于我们小型民用产品没什么必要)
{% label C\C++ time.h基础知识： blue %}

  > time_t now 记录的是1970年1月1日午夜UTC至今的秒数

  > struct tm timeinfo 则是一个包含年月日时分秒的结构体

  > 使用stdlib.h中的setenv("TZ", "CST-8", 1);tzset();设定时区

  >使用time(&now),将当前时间赋值给now变量，再使用localtime_r(&now, &timeinfo)将时间戳转变为本地时间

```
#include <time.h>
#include <sys/time.h>
#include "esp_sntp.h"
#include "stdlib.h"

static void obtain_time(void)
{
    initialize_sntp();

    // 等待时间设置成功
    time_t now = 0;
    struct tm timeinfo = {0};
    int retry = 0;
    const int retry_count = 10;
    //等待并校验时间是否正确，C标准中.tm_year从1900年开始（注意与先前的时间戳区分，其从1970年开始）
    while (timeinfo.tm_year < (2016 - 1900) && ++retry < retry_count)
    {
        ESP_LOGI(TAG, "Waiting for system time to be set... (%d/%d)", retry, retry_count);
        vTaskDelay(pdMS_TO_TICKS(2000));
        time(&now);
        localtime_r(&now, &timeinfo);
    }
    // 时间已设置
    ESP_LOGI(TAG, "Time is set");
}

static void initialize_sntp(void)
{
    ESP_LOGI(TAG, "Initializing SNTP");

    // 设置时区为中国标准时间 UTC+8
    setenv("TZ", "CST-8", 1);
    tzset();

    sntp_setoperatingmode(SNTP_OPMODE_POLL);
    sntp_setservername(0, "pool.ntp.org");
    sntp_init();
}
```
