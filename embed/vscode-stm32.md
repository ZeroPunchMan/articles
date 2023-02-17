# 基于VSCODE的STM32开发环境搭建

目的是基于免费/开源软件搭建一套STM32的开发环境,稍作修改也可以用作其他ARM单片机开发.

但现在Keil已经推出了社区版,对于非商业用途,选择Keil社区版更好,VSCODE可仅作为代码编辑工具.

PS: 参考工程： __[链接:基于STM32的DIY调试器](https://github.com/ZeroPunchMan/cmsis-dap/tree/master/firmware)__.

## 所需工具
```
1.编译工具: 在ARM的官网上有提供对应的gcc工具链,因为没有操作系统,选择none-eabi版本.
2.构建工具: linux环境下很方便,windows在网上找了个4.3版本的make
3.调试工具: openocd. 
4.调试器: st-link v2.
```

调试工具可通pyocd代替,调试器也可用jlink代替,具体使用方法搜索即可.

## 工程构建
STM32CUBEMX在生成代码时,选择Makefile即可,会自动生成需要的文件,根据项目需求,往Makefile添加文件即可.

### 代码编辑
用VSCODE打开工程目录，从插件市场搜索安装微软提供的C/C++插件,可以提供代码补全,错误提示等功能.

VSCODE的项目大部分配置文件,都在项目的根目录下的.vscode文件夹中,需要以下配置文件,来方便开发.

配置文件可以直接新建,也可以从vscode的命令创建.

#### c_cpp_properties.json 
这个文件主要用来帮助vscode分析代码,减少没用的错误提示
```
//主要配置项为
//"includePath": 即包含文件的路径
//"defines": 用到的宏定义
"configurations": [
{
    "includePath": [
        "${workspaceFolder}/**",
        "${workspaceFolder}/common/",
    ],
    "defines": [
        "__GNUC__",
        "USE_FULL_LL_DRIVER",
        "HSE_VALUE=8000000",
        "__ARM_ARCH_7M__=1"
    ],
    "cStandard": "c99",
    "cppStandard": "c++98",
    "intelliSenseMode": "gcc-arm",
}    
```

#### settings.json 
```
//"files.exclude": 在vscode中屏蔽这些路径,比如build下的各种.o文件

"files.exclude": {
    "*.ioc": true,
    ".mxproject": true,
    "build": true,
},
```

### 编译
编译只需要make xxx,在vscode中新增自定义命令,就可以从vscode方便地执行编译操作.

.vscode/tasks.json用来配置各种自定义命令,比如make命令如下:
```
"tasks": [
    {
        "label": "build", //lable为命令菜单中显示的名利
        "type": "shell",
        "command": "make", //执行的命令,默认路径为项目根目录
        "group": {
            "kind": "build",  //类型为build,设置为default,按ctrl+shift+b快捷键会直接执行
            "isDefault": true
        },
        "problemMatcher": [
            "$gcc"
        ]
    },
]

```

### 下载
下载配置与编译类似,在make中新增了一个target来执行
```
program:
	openocd -f ./stm32f103.cfg -c "program ./build/firmware.elf verify reset exit"
```

### 调试
调试主要是配置启动项,以及调试相关参数,在.vscode/launch.json中.

最主要的是"setupCommands"中的gdb启动命令,vscode会按顺序执行.
```
"setupCommands": [
    {
        "description": "set working directory",  //设置工作目录
        "text": "cd ${workspaceFolder}/build",
        "ignoreFailures": false
    },
    {
        "description": "open elf file",    //gdb打开文件
        "text": "file firmware.elf",
        "ignoreFailures": false
    },
    {
        "description": "connect gdb server",  //通关管道连接gdb server
        "text": "target remote | openocd -f interface/stlink-v2.cfg -f target/stm32f1x.cfg -c \"gdb_port pipe; log_output openocd.log\"",
        "ignoreFailures": true
    },
    {
        "description": "reset halt",  //复位并暂停CPU
        "text": "monitor reset halt",
        "ignoreFailures": true
    },
    {
        "description": "just load", //加载,把PC设置到程序入口点
        "text": "load",
        "ignoreFailures": true
    }
]
```

另外在launch中配置了预执行命令build,保证在调试开始时,已经编译最新的程序.
```
"preLaunchTask": "build",
```
