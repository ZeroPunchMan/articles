# DIY调试器: CMSIS-DAP

官方例程使用的是NXP的LPC系列MCU,DIY目标为DAP移植到STM32上

项目地址: https://github.com/ZeroPunchMan/cmsis-dap

MCU选用STM32F103C8T6,用kicad画了个板子,软件开发环境为VSCODE.

PS: __[链接:VSCODE的STM32开发环境搭建](https://ZeroPunchMan.github.io/articles/embed/vscode-stm32)__.

## 目标&功能
下图取自官方文档,DIY要做的就是实现其中的Debug Unit.
![](https://raw.githubusercontent.com/ZeroPunchMan/articles/main/img/CMSIS_DAP_INTERFACE.png)

整体功能大概就是两部分: __与主机的USB通讯__; ___与目标机的DAP接口通讯__.

## 1 USB部分
### 1.1 基本描述符
设备,接口,端点描述符没什么特别的,端点文档描述如下:
```
Endpoint 1: Bulk Out – used for commands received from host PC.
Endpoint 2: Bulk In – used for responses send to host PC.
Endpoint 3: Bulk In (optional) – used for streaming SWO trace (if enabled with SWO_STREAM).
```
按文档配置了3个端点,另外还选择配置一个CDC设备来支持与目标机的UART通讯.

PS:虽然这里有SWO的端点,但由于几乎不会用到SWO,实际代码配置中关闭了SWO的功能.

另外文档有提到,产品字符串中需要包含"CMSIS-DAP":
```
The String Settings - Product String must contain "CMSIS-DAP" somewhere in the string. This is used by the debuggers to identify a CMSIS-DAP compliant debug unit that is connected to a host computer.
``` 

### 1.2 WinUSB描述符
为了让Windows识别调试器,win8及以上的版本,需要配置OS描述符,就能使用WinUSB,而无需额外驱动.

对于USB2.0设备,有以下3个步骤:

1.Windows会通过获取索引为0xEE的字符串,要给出如下字符串,让Windows知道此设备支持OS描述符.
```
//设备返回的0xEE字符串,代码在usbd_desc.c中
__ALIGN_BEGIN uint8_t USBD_FS_WinOsStrDesc[] __ALIGN_END =
{
  0x12, //Length of the descriptor
  0x03, //Descriptor type 
  'M', 0x00, 'S', 0x00, 'F', 0x00, 'T', 0x00, '1', 0x00, '0', 0x00, '0', 0x00, //MSFT100
  OS_STR_VENDOR, //Vendor code
  0x00 //Pad field
};
```

2.Windows通过bRequest为0xdd,wIndex为0x04的标准请求获取WCID,以此知道设备支持WinUSB.
```
//设备返回的WCID,代码在usbd_dap.c中
__ALIGN_BEGIN uint8_t USBD_FS_OsCompIdDesc[] __ALIGN_END =
{
    0x28, 0x00, 0x00, 0x00,                         // length
    0x00, 0x01,                                     // version 1.0
    0x04, 0x00,                                     // Compatibility ID Descriptor index, fixed
    0x01,                                           // Number of sections
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,       // reserved 7 bytes
    0x00,                                           // interface num
    0x01,                                           // reserved
    'W', 'I', 'N', 'U', 'S', 'B', 0x00, 0x00,       // compatible id, ascii capital only
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, // sub comptible id
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00              // reserved 6 bytes
};
```

3.Windows发起bRequest为0xdd,wIndex为0x05的标准请求,获取扩展属性,此属性为WinUSB所需要的接口GUID.
```
//设备返回的GUID,代码在usbd_dap.c中
__ALIGN_BEGIN uint8_t USBD_FS_OsExtPropDesc[] __ALIGN_END =
  {
      0x8e, 0x00, 0x00, 0x00, // total length
      0x00, 0x01,             // version 1.0
      0x05, 0x00,             // ext prop index, fixed
      0x01, 0x00,             // number of secitons
      0x84, 0x00, 0x00, 0x00, // section length
      0x01, 0x00, 0x00, 0x00, // property type: A NULL-terminated Unicode String
      0x28, 0x00,             // property name length
      'D', 0x00, 'e', 0x00, 'v', 0x00, 'i', 0x00, 'c', 0x00, 'e', 0x00,
      'I', 0x00, 'n', 0x00, 't', 0x00, 'e', 0x00, 'r', 0x00, 'f', 0x00, 'a', 0x00, 'c', 0x00, 'e', 0x00,
      'G', 0x00, 'U', 0x00, 'I', 0x00, 'D', 0x00, 0x00, 0x00, // name
      0x4e, 0x00, 0x00, 0x00,                                 // property data length
      '{', 0x00, 'C', 0x00, 'D', 0x00, 'B', 0x00, '3', 0x00, 'B', 0x00, '5', 0x00, 'A', 0x00, 'D', 0x00, '-', 0x00,
      '2', 0x00, '9', 0x00, '3', 0x00, 'B', 0x00, '-', 0x00,
      '4', 0x00, '6', 0x00, '6', 0x00, '3', 0x00, '-', 0x00,
      'A', 0x00, 'A', 0x00, '3', 0x00, '6', 0x00, '-', 0x00,
      '1', 0x00, 'A', 0x00, 'A', 0x00, 'E', 0x00, '4', 0x00, '6', 0x00, '4', 0x00, '6', 0x00, '3', 0x00, '7', 0x00, '7', 0x00, '6', 0x00,
      '}', 0x00, 0x00, 0x00 // property data: GUID
};
```

到这里之后,设备已经被正确枚举,可以在设备管理器中看到设备.

### 1.3 USB通讯处理
USB的收发均采用多缓存处理,保证后台处理不会影响前台数据的完整性.

USB的DAP命令主要分为2种,普通命令和原子命令,其中普通命令收到后一包数据后可以立即处理，原子命令则分为两种：
```
1.DAP_ExecuteCommands
一个数据包内有多个命令,同普通命令,收到一包数据后,立即把包内所有命令处理完毕.
2.DAP_QueueCommands
命令由多个数据包组成,每个数据包内都可能有多条命令,收到全部数据包后,立即处理全部命令.
```

相关处理代码在dap_agent.c中,DAP_ExecuteCommands和普通命令一样处理;DAP_QueueCommands则要在收到一包非DAP_QueueCommands后,开始处理.

## 2 DAP部分
ARM提供了一个配置用头文件:DAP_config.h 

DAP功能主要配置以下3个部分

### 2.1 配置USB
主要配置项:
```
DAP_PACKET_SIZE: USB每包数据大小,项目中设置为64,即全速USB的bulk最大包容量.
DAP_PACKET_COUNT: USB包缓存数量,主机根据此数值,限制一次下发的包数.
DAP_UART_USB_COM_PORT: USB的CDC-UART功能,项目中未启用.
```

### 2.2 配置IO
预设了一些操作DAP接口的函数,用GPIO操作模拟DAP,按注释写的需求填写即可,没有特别需要说明的.

### 2.3 配置DAP
```
CPU_CLOCK: CPU时钟频率,用来计算DAP接口操作的延时
IO_PORT_WRITE_CYCLES: ///< I/O Cycles: 2=default, 1=Cortex-M0+ fast I/0.这里不清楚是否准确,按注释填了2
DAP_SWD: 为1开启SWD功能.
DAP_JTAG: 为1开启JTAG功能.
SWO_UART: 为1开启SWO,此项目中设置为0关闭SWO.
DAP_UART: 为1开启额外串口,此项目为0中关闭UART.  

__STATIC_INLINE uint8_t DAP_GetVendorString(char *str): 一系列函数,返回厂商ID,产品ID,序列号等.
```

## 成果
目前项目通过了官方的Keil测试用例,并且作为调试器使用了一段时间,未发现问题.

但在openocd和pyocd上均未识别出来,暂时不清楚原因.

