# 介绍

 xps9570 clover 开发记录
# 硬件配置

## 已驱动

* Machine :Dell XPS 9570
* CPU: Intel i7-8750H
* GPU: UHD630 + Nvidia Gefore 1050Ti(屏蔽)
* RAM: 16GB RAM
* Display: 4K Sharp Display
* SSD: SM961
* Audio: Realtek ALC3266
* WLAN + Bluetooth : apple 原生网卡
## 未驱动

* Goodix fingerpint reader (无解)
* Nvidia Geforce 1050Ti (无解，已屏蔽)

# 开发记录
## I2C
I2C会导致很高的kernel task占用，我有一种感觉这个是最后需要解决的问题
I2C 两条线路是I2C1 于I2C0  
这两个线路分别是I2C1 的TPD0触摸板和I2C0 的TPL1的触摸屏
TPD0的为APIC 轮询pin:0x33  
TPL1的为APIC 轮询pin:0x3b  
实际上大于0x2F的就是无法使用APIC 中断了
所以正确的做法是改为GPIO中断
附上代码

 		Method (_CRS, 0, NotSerialized)          {
           Name (SBFB, ResourceTemplate ()
            {
                I2cSerialBusV2 (0x002C, ControllerInitiated, 0x00061A80,
                    AddressingMode7Bit, "\\_SB.PCI0.I2C1",
                    0x00, ResourceConsumer, _Y40, Exclusive,
                    )
            })

            
  
            Name (SBFG, ResourceTemplate ()
            {
                GpioInt (Level, ActiveLow, ExclusiveAndWake, PullDefault, 0x0000,
                    "\\_SB.PCI0.GPI0", 0x00, ResourceConsumer, ,
                    )
                    {   // Pin list
                        
                        0x009e
                    }
            })
       
            Return (ConcatenateResTemplate (SBFB,SBFG))
        }


并且将返回值的SBFI置换为SBFG，因为默认是0x17中断，但是实际上要用0x9E中断或者0x1b中断，实测下来kernel task稳定在11%。即使是在快速长久触发GPIO中断下的触摸板时

TPL1触摸屏同理

# APPLEALC开发记录
很多CLOVER建议是用30或者72的点
查询源码得知

分别为layout——id为ALC298的点是  
3，11，13，21，22，28，29，30，47，66，72，99  
30是Constanta - Realtek ALC298 for Xiaomi Mi Notebook Air 13.3 Fingerprint 2018  
72是Custom - Realtek ALC298 for Dell XPS 9560 by KNNSpeed  
显然72更适合xps9570，为什么有些人会觉得72不适合或者30不适合  
让我们查看源代码  
<01271c30 01271d00 01271ea0 01271f90  

01771c40 01771d00 01771e17 01771f90 01770c02  
01871c70 01871d10 01871e81 01871f00  
02171c20 02171d10 02171e21 02171f00>  

这是layout为30的  
首先是0x17为内置speaker输出 01170c02是EAPD的参数，看起来十分好但是0x18的是70108100这个是
![avatar](https://7.daliansky.net/pinconfigs.png)
是外置输入mic，black linein输入3.5mm接口  
实际上是正确的，但是他的GPIOmute是错误的
应该是0x50010018的10进制。实际加入后并无效果

layout是72的为
<01271c10 01271d01 01271ea6 01271f90 
01771c20 01771d01 01771e17 01771f90 
01871c30 01871d10 01871eab 01871f03 
01a71c40 01a71d10 01a71e8b 01a71f03 
02171c50 02171d10 02171e2b 02171f03 01470c02 01770c02 01a70c02 02170c02>
0x18的是完全错误的，但是节点没有0x18的节点，输入定义了但是没有给节点

所以我的思路是对0x18驱动对0x1a屏蔽，也就是分别为0x17内置扬声器
0x21耳机输出，0x18外置mic输入，0x12内置mic驱动，其余屏蔽
实际看下来只有layout 为11的比较相似。或许layout 为11更好驱动一些
# 耳机无声（已解决）
使用HDEF来注入的话会导致外置mic无法驱动，但是不用的话又会导致耳机输出无声，使用alc298fix+辅助kext或许是导致耳机无mic输入的关键
注入点正确，但是不显示为外置，而依然是内置mic，怀疑是intel智音系统会自动将外置mic转为内置mic，在win10下也是无法通过3.5mm耳机孔来观看是否是外置还是内置，全部显示为外置，但是在linux下可以显示为外置。

# HDMI audio 开发记录
使用weg的手册，但是codec和connect-type均正确却无输出。原因未知

尝试使用fakePCIID
# fan传感器
获取DSDT的不知道哪里的参数，伪造了个转速，实际原因是因为fan的储存位置不知，其次是如何控制，关于解锁EC可给出建议。但是EC写数据是非常危险的，很可能导致无法开机。
解锁:0x30a3
上锁:0x34a3  

# 关于解锁MSR和BIOS  

这个暂时我还没想过怎么叙述  

# 关于0.8ghz锁频  

这个之前猜测是I2C的原因，后来发现锁频时主要是核显崩溃或者是核显的其他状态，我觉得是一种自我保护机制，此时风扇转速很慢，在采用cpufriend变频时发现了很容易0。8ghz锁频
所以深度修改了变频代码。cinebench可以到达3143分，在电池来跑分下
只是这种代码还在测试中，开发完毕会更新github
结果有一天还是锁频了，所以猜只要是和GPU的都会在过热时锁频，那么这种锁频有可能是weg的bug或许是核显过热，也就是风扇无法正常启动
建议在win下改成酷冷或者极速，让风扇转动积极一些，同时升高温度墙，但是撞墙后不会一直锁0.8ghz，所以猜测是dell主板的某些机制导致自锁0.8ghz来保证温度。  


如果温度墙在60度的话。那么GPU瞬间到达90度，会有（当前频率-k(90-温度墙)*100mhz）=实际频率，那么很有可能直接得到一个很低的实际频率小于800mhz所以让CPu强制锁800mhz

那么如果将GPU温度墙增高岂不是更好  

# 关于鸡血代码
采用原生XCMP+hWP 代码变频，实际上效果显著，瞬间睿频最大可到4ghz（单核）待机时稳定0.8ghz，可是目前不成熟，有的机子会导致睿频锁死
有的会温度墙锁死。还在开发中  


# 已知问题

### ~~无外置mic输入~~（已解决）
### 无hdmiaudio
### i2C驱动错误
# 鸣谢

* [Apple](https://www.apple.com) for macOS
* [Rehabman](https://github.com/RehabMan)：提供了大量的黑苹果驱动，国外黑苹果论坛的大佬，向大佬致敬！
* [Lilu](https://github.com/acidanthera/Lilu)： 向该内核扩展项目伟大的逆向工程师与开发者致敬！
* [WhateverGreen](https://github.com/acidanthera/WhateverGreen)：感谢所有参与该开源内核扩展项目的伟大开发者！
* [FireWolf](https://github.com/0xFireWolf/)： 提供DPCD最大链路速率修复、Intel HDMI无限循环连接修复、LSPCON驱动支持等核心的通用性开源贡献，使得基于UHD630的一些新机型，特别是XPS 9570提供了非常强大的技术难关攻坚，非常感谢他！
* [bavariancake](https://github.com/bavariancake/XPS9570-macOS)：提供一份针对XPS 9570 Hackintosh的详尽方案，他的仓库对XPS 9570驱动的每个细节进行了记录，是XPS 9570黑苹果在Github上的开源先驱，为本仓库贡献了大部分配置模板，感谢他的辛勤付出和劳动！
* [Xigtun](https://github.com/Xigtun/xps-9570-mojave)：与@bavariancake一样，属于早期开源XPS 9570黑苹果配置的无私贡献者，本仓库早期是在
*基础上对[bavariancake](https://github.com/bavariancake/XPS9570-macOS)的配置进行深度融合，加以改进才得到如今比较完美的配置，谢谢这位无私的同仁！
* @807133286 ：最早在Xigtun仓库中提出了可移植的[触控板驱动方案](https://github.com/Xigtun/xps-9570-mojave/issues/23)，给XPS 9570拥有将近白苹果触控板的体验，是改善XPS 9570触控版的灵感来源！ 
* 
*[LuletterSoul](https://github.com/LuletterSoul/Dell-XPS-15-9570-macOS-Mojave):深度整合了各种优良的EFI并且使用最新方法来完善，EFI几乎是完美的，本MD参考他的。
* [远景论坛](http://bbs.pcbeta.com/forum-559-1.html)：谢谢诸位大神提供的通用教程，让我能够以小白的身份轻松入门！
* [黑果小兵](https://blog.daliansky.net/): 我想，国内需要更多这样无私的、高水平的黑苹果布道者，感谢他！

