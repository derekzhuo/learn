# pcie_drv
|目录 / 文件|功能定位 |
| ------   | ------      |
|`common/`	                | 所有硬件无关、通用、可复用的基础模块，XDMA 核心逻辑就在这里|
|`sbus2/3/4/`	            | 不同版本 / 实例的自定义总线驱动（UVAPS 相关）|
|`cdev_*.c`	                | 字符设备驱动接口（/dev/xxx），负责用户态交互|
|`uvp_pcie.c/h`	            | PCIe 总线初始化、BAR 空间映射、中断处理|
|`CMakeLists.txt` /         | Kbuild.in	构建脚本，分别用于用户态工具和内核模块编译|

核心逻辑链：用户态通过(/dev/xxx)访问设备 -> xdma_cdev.c(字符设备) -> libxdma.c(xdma引擎) -> uvp_pcie.c(PCIe枚举)

## common目录
xilinx提供的基础文件，可以划分为以下四种类型。

### 基础参数与动态配置
功能：提供模块参数动态配置（比如通过 insmod 时传参数、运行时通过 sysfs 修改）
怎么读：先看头文件里的变量定义，再看 .c 里的 module_param() 注册和参数校验逻辑

### XDMA核心引擎
libxdma.c / libxdma.h / libxdma_api.h

这是整个驱动的核心：Xilinx XDMA 引擎的封装
读的顺序：

先看 libxdma_api.h：这是对外接口，知道有哪些 DMA 操作（open/close/read/write/transfer）
再看 libxdma.h：数据结构定义（XDMA 通道、描述符、寄存器定义）
最后看 libxdma.c：实现细节（寄存器读写、描述符管理、中断处理）

### 字符设备接口（对接用户态）
xdma_cdev.c / xdma_cdev.h

功能：把 XDMA 引擎封装成标准字符设备，提供 /dev/xdma* 节点
重点看：

struct file_operations：open/read/write/ioctl/poll 这些回调函数，最终都会调用 libxdma 的接口
xdma_cdev_init()：字符设备注册流程（设备号、cdev、class、device_create）

### 后台线程与异步处理
xdma_thread.c / xdma_thread.h
功能：XDMA 异步操作、中断处理线程、超时 / 错误处理
读的时候重点看：线程入口函数、等待队列、和 libxdma 之间的交互逻辑

## PCIe与总线层
## 业务层（sbus2/3/4）

## 大块数据走DMA的整体链路
## Reg操作的整体链路
## 为什么要有两个event
## user及event的区别