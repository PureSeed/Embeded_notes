# 01 Boot loader

### Boot Loader（引导加载程序）是什么？
它是一个**专门负责引导的软件**，通常存放在系统的启动介质（如内部Flash或外部EEPROM）的最前面。
它的主要任务是：
- **初始化硬件**：初始化CPU频率、内存控制器（SDRAM/DDR）等，为运行大程序做准备。
- **加载应用程序**：将真正的用户应用程序或操作系统内核，从存储介质复制到内存中。
- **跳转执行**：跳转到被加载程序的起始地址，交出CPU控制权。

另外，Boot Loader通常还附带强大的**固件更新功能**（如通过串口、USB或OTA升级应用程序），因此在嵌入式开发中，它常常充当“烧录工具”的角色（比如常见的 **U-Boot**，或者STM32的 **System Bootloader**）。

### 独立Bootloader的好处：
1. **极致安全的升级体验：** Bootloader和APP存放在Flash的不同分区，升级时只会擦写APP分区。就算升级失败，Bootloader依然完好，可以重新进入升级模式，或者启动上一个可用版本，彻底告别变砖风险。
2. **支持OTA空中升级:** 可以通过蓝牙把升级包传到手表里，Bootloader会自动完成升级流程，给手表更新功能和修复bug。
3. **开发维护更高效：** Bootloader只负责基础启动和升级，功能固定，一次开发长期稳定。APP主程序可以快速迭代，不用再担心改动影响系统启动，大大提升开发效率。
4. **系统扩展性更强:** 可以轻松添加开机密码、硬件健康检查、多版本系统切换等高级功能，甚至可以实现双系统启动，极大提升产品的可玩性和安全性。

### 启动流程：
1. 开机从固定地址开始执行Bootloader，初始化基础硬件 
2. 检查升级标志位，判断是否需要升级 
3. 如果不需要升级，就跳转到APP程序地址，把控制权交给应用程序
4. APP启动，进入正常使用界面

### 内存分区
**BOOT区后面划分了一个Flag区，用于记录是否是完整的APP，这个位置是APP传输完成后才记录的，为的是保证程序完整性** ![[Pasted image 20260818232932.png]] **0x08000000:** Bootloader的起始地址，从这里开始存放启动引导程序，一直到0x08007FFF，总共占用32KB空间 **0x08008000:** 升级标志位区域，用来存储升级状态信息，告诉Bootloader要不要执行升级 **0x0800C000:** APP主程序的起始地址，剩下的大部分空间都留给主程序，最大到0x08007FFFF

### 单片机的内存
**Flash闪存：嵌入式的"硬盘"** ✅ 断电不丢失：程序和数据永久保存 ✅ 用来存储：Bootloader、APP程序、固件、永久配置参数 ✅ 读取速度快，擦写次数有限：一般可以擦写10000次以上 ✅ 我们聊的内存分区，就是在Flash里面划分不同区域

**RAM内存：嵌入式的"内存条"** ⚡ 断电即清空：只存储运行时的临时数据 ⚡ 用来存储：全局变量、堆栈、程序运行时的数据 ⚡ 读写速度极快，但容量一般比Flash小 ⚡ 不需要分区，由操作系统自动管理

**程序完整运行流程**
1. 程序烧录 → Flash分区存储
2. 开机启动 → CPU从Flash加载Bootloader
3. 初始化完成 → 跳转到Flash中的APP地址
4. 程序运行 → 数据和变量存储在RAM中

---------------------
# 02 基本实现思路
实现一个Boot Loader，核心思路可以概括为：**在Flash中划分两个区域，编写一段能接收新固件并“覆盖”旧固件的独立程序**。

### 1. 整体流程与Flash分区
首先，你需要将Flash划分为两个独立的区域。

|区域|起始地址（以STM32为例）|作用|
|---|---|---|
|**Boot Loader区**|`0x08000000`|存放Boot Loader程序本身。上电后首先执行这里。|
|**应用程序(APP)区**|`0x08008000`（偏移量可自定义）|存放你的主程序（比如LVGL界面代码）。Boot Loader负责跳转到这里。|
|**标志/参数区**|（可选）|存放一个标志位，用于指示是否需要更新固件。|

**整个流程是这样的：**   
上电 -> 执行Boot Loader -> 检查更新标志 -> **有更新**则接收固件写入APP区 -> **无更新**则跳转到APP区执行。

### 2. 关键代码实现步骤

#### 步骤一：设置APP起始地址和中断向量偏移
这是最关键的一步，必须告知芯片，你的APP代码不再从`0x08000000`开始。
- **在Boot Loader工程中**：无需特殊设置，它默认从`0x08000000`启动。
- **在APP工程中（如Keil MDK）**：
    1. 在“Target”选项页中，将 `IROM1` 的 `Start` 地址改为 `0x08008000`（或其他你选择的偏移量）。
    2. 在代码初始化早期（通常在`SystemInit`函数或`main`开头），调用以下函数来重映射中断向量表：  ``` c
 ```C
// NVIC 中断向量表偏移
 SCB->VTOR = FLASH_BASE | 0x8000; // 0x8000 对应 32KB 偏移
 ```
> **注意**：APP区的偏移量（如32KB）必须大于Boot Loader程序的实际大小。

#### 步骤二：实现Boot Loader的跳转函数
Boot Loader的核心功能，就是在最后把CPU控制权交给APP。
```C
// 定义APP区的起始地址（必须与工程设置一致）
#define APP_ADDRESS 0x08008000
// 定义一个函数指针类型，指向APP的复位入口
typedef void (*pFunction)(void);
void jump_to_app(void) {
    uint32_t jump_addr;
    pFunction jump_func;
    // 1. 检查APP起始地址是否存放了有效的栈顶指针（简单的完整性校验）
    if (*(__IO uint32_t*)APP_ADDRESS == 0xFFFFFFFF) {
        // 地址是空的，不跳转
        return;
    }
    // 2. 关闭所有可能干扰的外设（如中断、串口等），避免跳转后异常
    HAL_UART_DeInit(&huart1);
    HAL_DeInit(); // 复位所有外设（如果是STM32 HAL库）
    __disable_irq(); // 关闭全局中断
    // 3. 获取APP的复位中断向量地址 (向量表第二个字)
    jump_addr = *(__IO uint32_t*)(APP_ADDRESS + 4);
    jump_func = (pFunction)jump_addr;
    // 4. 设置APP的栈指针 (向量表第一个字)
    __set_MSP(*(__IO uint32_t*)APP_ADDRESS);
    // 5. 重新开启全局中断，并跳转！
    __enable_irq();
    jump_func(); // 此调用不会返回
}
```

#### 步骤三：固件接收与写入
Boot Loader需要一种通信方式（如UART、USB、SD卡、Wi-Fi等）来接收新的APP固件（通常是`.bin`文件）。
- **通信协议**：你需要定义一个简单的协议，例如：
    - `0xAA`：开始传输指令。
    - `0xBB`：固件数据包（包含地址和数据）。
    - `0xCC`：结束指令。
- **写入Flash**：收到数据包后，调用Flash擦除和写入API，将数据直接写入APP区（`0x08008000`）。**注意**：写入前必须先擦除对应的扇区。

#### 步骤四：设置更新标志
为了决定是进入升级模式还是直接跳转，可以通过以下方式设置标志：
- **外部引脚**：检测某个GPIO电平（如按键按下时进入升级）。
- **内部标志**：在Flash的最后一个扇区写入一个特定值。
- **超时机制**：Boot Loader启动后等待几秒，若无升级指令，则自动跳转。

### 3. 核心注意事项
1. **中断向量表**：APP工程中**必须**设置 `SCB->VTOR`，否则任何中断（如定时器、串口中断）都会导致程序跑飞。
2. **跳转环境**：跳转前务必关闭所有外设中断，并复位外设到初始状态。否则APP启动时，未处理的中断可能会导致HardFault。
3. **固件格式**：MCU通常接收原始的 `.bin` 文件，而不是 `.hex`。你可以使用 `fromelf` 等工具将编译生成的 `.axf` 转换为 `.bin`。
4. **安全与纠错**：建议在固件传输时增加CRC校验，并对写入Flash的APP数据进行完整性检查，避免升级失败导致设备“变砖”。

### 4. 进阶：OTA空中升级
如果你的Boot Loader通过Wi-Fi或蓝牙接收固件，原理相同，只是通信方式变了。这种情况下，通常还需要将新固件先暂存到外部SPI Flash中，校验无误后再写入内部主Flash的APP区。

----------
## 03 IAP OTA YMODEM

**IAP：** 这是升级的能力，全称In‑Application Programming(在线应用编程)，指设备可以在不拆机的情况下自己给自己升级程序，是一种升级方式。 **OTA：** 这是升级的传输方式，全称Over‑The‑Air（空中升级），指通过无线方式升级，比如蓝牙、WiFi、蜂窝网络，不用插线就能升级。 **YMODEM：** 这是传输协议，负责把固件文件准确地从电脑或手机传到设备里，保证传输过程中不会出错。

**它们的关系就像是快递系统** ‑ **IAP就是你家小区的快递柜** ，提供了接收快递的能力，不管快递是怎么送过来的，都可以存在这里。 ‑ **OTA就是无人机送货** ，是一种无线的送货方式，不用快递员上门，直接飞到你家窗户边把快递递给你。 ‑ **YMODEM就是快递的物流标准** ，规定了快递怎么打包、怎么验货、怎么签收，保证快递不会丢件不会损坏。

实际升级中的完整流程

1. 第一步：通过OTA传输：手机通过蓝牙把固件包无线发送给手表，这就是OTA升级。
2. 第二步：用YMODEM保证可靠传输：传输过程中使用YMODEM协议，把固件分成1KB一包，每包都加校验，确保固件不会传错。
3. 第三步：IAP完成升级：手表的Bootloader收到完整的固件包后，把它写入Flash的APP分区，完成升级并启动新程序。

它们不是互斥关系，而是互补关系IAP是基础能力，OTA是传输方式，YMODEM是传输协议。你可以用YMODEM配合串口线做有线IAP升级，也可以用YMODEM配合蓝牙做OTA无线升级，它们可以自由组合，适配不同的升级场景。

---
# 04 实现的流程
### 第 1 步：APP 先写完 —— OV_Watch 工程先存在
这是前提：Bootloader 是为 APP 服务的，没有 APP 就不需要 Bootloader。
```
OV_Watch 工程（已完成）
└── 链接地址：默认 0x08000000（STM32 出厂地址）
    ↑ 还没改过，后面要改
```

### 第 2 步：规划 Flash 分区 —— 在纸上算地址
STM32F411CEU6 总 Flash 512KB，作者按 Sector 大小规划：
```
Sector 0: 0x08000000 - 0x08003FFF   16KB   Bootloader
Sector 1: 0x08004000 - 0x08007FFF   16KB   Bootloader（总共 32KB）
Sector 2: 0x08008000 - 0x0800BFFF   16KB   APP FLAG 标记区（只用前 8 字节）
Sector 3: 0x0800C000 - 0x0800FFFF   16KB   APP 起点
Sector 4-7: ...                            APP 剩余空间
```
**为什么 Bootloader 占 Sector 0-1 而不是只占 Sector 0？**
Sector 0 只有 16KB，但 Bootloader 要包含 Ymodem 协议栈 + ST7789 LCD 驱动 + 菜单字符串——16KB 可能不够。所以占两个 Sector（32KB）更保险。

### 第 3 步：改 APP 工程 —— OV_Watch 的 Keil 配置
Bootloader 要跳到 APP，APP 必须从 `0x0800C000` 链接。**需要改两处：** 
**3a. Keil Target → IROM1 设置**
```
原来：Start=0x08000000, Size=0x80000 (512KB)
改成：Start=0x0800C000, Size=0x74000 (464KB)
```
APP 从 0x0800C000 开始，最大能用到 0x0800C000 + 0x74000 = 0x0807FFFF（Flash 末尾）。

**3b. APP 的 main() 开头加 VTOR 重映射**（`OV_Watch/Core/Src/main.c:78`）
```c
SCB->VTOR = 0x0000C000U;  
```
APP自己重映射中断向量表——Bootloader跳过去后，APP的中断向量表在0x0800C000，必须告诉CPU去那里找，否则中断会跳到Bootloader的向量表直接HardFault

### 第 4 步：拿 ST 官方 IAP 示例，CubeMX 重生成框架
ST 官方示例文件（就是 `Ymodem/` 里的 4 个 .c 和 4 个 .h）是 **STM32F4 通用** 的，作者的 F411 有特定外设，所以用 CubeMX 重新搭：

**4a. 新建 CubeMX 工程，选 STM32F411CEUx**

**4b. 配外设（和 OV_Watch 几乎一样，因为硬件相同）：**
- SPI1 → LCD（ST7789）
- USART1 → KT6328 蓝牙 + Ymodem 串口（共用一个 UART）
- TIM3 CH3 → LCD 背光 PWM
- ADC1 → 电池电压检测
- GPIO → KEY1 按键、电源控制、BLE 使能

**4c. 加 FreeRTOS？  不加。Bootloader是裸机的——它只跑开机判定 + 升级菜单，不需要 RTOS。

**4d. 时钟配置**（`main.c:240-261`）：HSI 8MHz → PLL → SYSCLK 50MHz，和 OV_Watch 一样。

**4e. Keil 工程设置：**
```
IROM1：Start=0x08000000, Size=0x80000  ← 默认就对，不用改
```

### 第 5 步：把 ST 官方 IAP 文件加进来 + 改 Sector 地址

**5a. 复制 `Ymodem/` 4 个 .c 和 4 个 .h 到工程**

**5b. 改 `flash_if.h` 的 Sector 地址**（ST 原版是 F4 全系列，F411 只有 Sector 0-7，共 8 个）：
```c
// 行 32-35：Flash 分区表
#define ADDR_FLASH_SECTOR_0  0x08000000  // Bootloader
#define ADDR_FLASH_SECTOR_1  0x08004000  // Bootloader
#define ADDR_FLASH_SECTOR_2  0x08008000  // APP FLAG
#define ADDR_FLASH_SECTOR_3  0x0800C000  // APP 起点

// 行 52：APP 链接地址
#define APPLICATION_ADDRESS  0x0800C000
```

**5c. 改 `flash_if.c` 的 `GetSector()`** —— ST 原版有 Sector 0-11 的 if-else 链，作者把 Sector 8-11 的分支注释掉，改成 Sector 7 的 else 兜底：
```c
// ST 原版：
// else if((Address < ADDR_FLASH_SECTOR_11) ...  { sector = FLASH_SECTOR_10; }
// else  { sector = FLASH_SECTOR_11; }

// 作者改成：
else /*(Address < FLASH_END_ADDR) && (Address >= ADDR_FLASH_SECTOR_7))*/
{
    sector = FLASH_SECTOR_7;   // ← F411 只有 8 个 Sector，到 7 就结束
}
```

### 第 6 步：复制硬件驱动 —— BSP 文件夹
从 OV_Watch 工程复制过来，一行都没改：
```
OV_Watch/BSP/LCD/*  →  IAP_F411/BSP/LCD/*
OV_Watch/BSP/KEY/*  →  IAP_F411/BSP/KEY/*
OV_Watch/BSP/POWER/* →  IAP_F411/BSP/POWER/*
OV_Watch/BSP/KT6328/* → IAP_F411/BSP/KT6328/*
```
为什么需要？  Bootloader 要显示 "Bootload" 文字（LCD）、要读 KEY1 判定长按（KEY）、要点亮屏幕背光（POWER + TIM3 PWM）、BLE 模块要上电（KT6328）。Bootloader 不是纯串口工具，它自己就是一个**可交互的极简应用**。

### 第 7 步：写 Bootloader main() —— 作者的核心贡献

这是作者真正"自己写"的部分。逻辑顺序：

**7a. 硬件初始化**（main.c:111-131）
```
delay_init → Key_Port_Init → KT6328 → Power_Init → PWM_Start → LCD_Init + 亮背光
```
LCD 初始化必须在判定之前——因为升级模式下屏幕要显示 "Bootload"。

**7b. 长按 KEY1 判定**（main.c:134-150）
```
读 KEY1 → 延时 500ms → 再读 KEY1 → 还按着就进 Main_Menu()
```
**为什么用 500ms 长按而不是直接读？** 手表是电池供电，用户正常开机不会按着 KEY1 不放。长按 500ms 是为了**正常开机时不误触进升级模式**——如果开机时手不小心碰了一下按键（松手在 500ms 内），第二次读就跳过 if 了。

**7c. APP FLAG 机制**（main.c:155-203 + menu.c 里配合修改）
这是作者最关键的设计——让 Bootloader 知道"有没有合法 APP"：
```
在 0x08008000（Sector 2 开头）存 8 字节 "APP FLAG"
→ 升级完写这个标记
→ 开机读这个标记，有就跳 APP，没有就显示 No App
```
**为什么要这个标记？** 如果没有 APP FLAG，Bootloader 跳 APP 时会直接读 0x0800C000 里的内容——如果那里全是 0xFFFFFFFF（空 Flash），硬跳过去会 HardFault。APP FLAG 是个"保险"。

**7d. 跳转 APP**（main.c:183-194）
```
关 SysTick → 关总中断 → 设 MSP → 读 PC → 跳转
```
跳转前必须关掉SysTick，否则 APP 还没初始化时钟，SysTick 中断可能乱触发。
关闭全局中断。跳转过程中如果发生中断，中断向量表可能还指向 Bootloader，会导致不可预期的行为。等 APP 启动后，它自己会重新配置中断。

### 第 8 步：改 menu.c —— 加 APP FLAG 擦写
ST 原版的 `SerialDownload()` 只收固件、写 Flash，**不知道 APP FLAG 这回事**。作者加了两处（menu.c 里有 `// user operation` 注释标记）：

**8a. 收固件前先擦 Sector 2**（menu.c:71）
```c
// 旧的 APP FLAG 必须清掉，因为升级过程如果断电，
// 旧 APP 没写完整但标记还在 → Bootloader 跳一个坏 APP → 死机
FLASH_If_Erase_One_Sector(2U);
```

**8b. 收完固件后写新 APP FLAG**（menu.c:87-92）
```c
const char *str_flag = "APP FLAG";
FLASH_If_Write(&flashdestination, (uint32_t*)APP_FLAG, 2);  // 写 8 字节
```

**8c. menu.c 里选 '3' 跳 APP 也加了完整清理代码**（menu.c:207-218）——和 main.c:183-194 的跳转代码一模一样，因为 ST 原版 menu 里选 '3' 跳转也是裸跳，作者复制粘贴了 main.c 里的完整版本。

### 第 9 步：补 flash_if.c —— 加两个 ST 原版没有的函数

`menu.c` 里写 APP FLAG 需要直接写一个 word 到任意 Flash 地址，但 ST 原版 `FLASH_If_Write()` 是**从 APPLICATION_ADDRESS 开始连续写**的——写不了 0x08008000 这个独立地址。所以作者在 `flash_if.c` 里加了两个函数：

**`FLASH_ProgramWord()`**（flash_if.c:283-294）
```c
// 底层直接向 Flash 的 CR 寄存器写 PG 位，然后写 DR
// 跳过 HAL 封装，直接操作寄存器
FLASH->CR |= FLASH_PSIZE_WORD;
FLASH->CR |= FLASH_CR_PG;
*(__IO uint32_t*)Address = Data;
```

**`FLASH_OB_DisableWRP()`**（flash_if.c:263-280）
```c
// 作者自己实现的关写保护，ST 原版是用 HAL_FLASHEx_OB_DisableWRP
// 但作者选择绕过 HAL 直接操作 OPTCR_BYTE2_ADDRESS
```

### 第 10 步：测试
```
1. 只烧 Bootloader，不烧 APP
   → 上电 → Bootloader 读 0x08008000 = 0xFFFFFFFF（空 Flash）
   → 没 APP FLAG → 显示 "No App!" → Power_DisEnable() 断电 ✅

2. 长按 KEY1 进升级模式
   → LCD 显示 "Bootload + OV-Watch V2.4.1" ✅
   → 串口收到 ST 官方菜单提示 ✅

3. 用 Ymodem 发 APP 的 bin 文件
   → 擦 Sector 2（清旧标记） ✅
   → Ymodem 收 bin，写入 0x0800C000 起 ✅
   → 写完写 "APP FLAG" 到 0x08008000 ✅

4. 复位，正常开机（不按 KEY1）
   → Bootloader 读 0x08008000 = "APP FLAG" ✅
   → 关 SysTick + 关中断 → 读 MSP/PC → 跳转 0x0800C000 ✅
   → APP 启动，VTOR 重映射到 0x0000C000 ✅
   → FreeRTOS 正常调度，LVGL 正常显示 ✅
   → 中断正常工作（KEY/RTC/SPI DMA 回调都不 HardFault） ✅
```
---
## 总结：开发顺序一句话版
```
OV_Watch APP 先写完
    → 改 APP 链接地址到 0x0800C000
    → 加 APP 的 VTOR 重映射
    → 拿 ST 官方 IAP 示例（Ymodem + flash_if + menu + common）
    → 用 CubeMX 重新搭 F411 硬件框架
    → 改 flash_if.h 的 Sector 地址和 APPLICATION_ADDRESS
    → 改 flash_if.c 的 GetSector() 适配 F411 的 Sector 7
    → 从 OV_Watch 复制 BSP 驱动（LCD/KEY/POWER/KT6328）
    → 写 Bootloader main()：长按判定 + APP FLAG 读 + 跳转
    → 改 menu.c：SerialDownload 加擦旧标记 + 写新标记
    → 补 flash_if.c 缺的 FLASH_ProgramWord 和 FLASH_OB_DisableWRP
    → 烧录测试
```

**这个开发顺序很典型——工业界做 STM32 IAP Bootloader 基本都是"拿 ST 官方 demo 改地址、改 Sector、补硬件、加 APP FLAG 判定"这四步**，没有人从零写 Ymodem 协议栈。作者唯一真正的贡献是 **APP FLAG 机制的设计** 和 **跳转 APP 前的完整清理代码**（关 SysTick + 关中断）——这两点如果没做好，Bootloader 就是不稳定的。

1. **解锁 Flash**：`HAL_FLASH_Unlock()`
2. **擦除 APP 区**：调用 `FLASH_If_Erase()`
3. **写入固件**：循环调用 `FLASH_If_Write()`，把收到的固件数据按字写入 Flash
4. **写入 APP FLAG**：在指定地址写入 `"APP FLAG"` 标记
5. **上锁 Flash**：`HAL_FLASH_Lock()`
6. **跳转到 APP**：设置 MSP、取复位向量、跳转

----------
```                
				上电/复位
                   │
                   ▼
           Reset_Handler (startup.s)
                   │
                   ▼
           Bootloader main() 开始
                   │
              ┌────┴────┐
              ▼         ▼
     行84: VTOR=0     行104-131: 全外设初始化
              │         │
              └────┬────┘
                   ▼
            行134: 读 KEY1
                   │
              ┌────┴────────────┐
              ▼                 ▼
         按下？YES           没按下？NO
              │                 │
     行137: 延时500ms           │
              │                 │
     行138: 还按着？             │
         ┌────┴────┐           │
        YES        NO           │
         │          │           │
  行141-142:       │           │
  画屏幕           │           │
         │          │           │
  行147-148:       │           │
  FLASH_If_Init    │           │
  Main_Menu()      │           │
  ← 死等串口输入   │           │
                   │           │
             行157-163: 读 0x08008000
                   │
             行178: strcmp "APP FLAG"?
                ┌──┴──┐
               YES    NO
                │      │
     行183-186:        │
     关SysTick+关中断  │
                │      │
     行189-193:        │
     设MSP+取PC        │
                │      │
     行194:            │
     Jump_To_App()     │
     ┌──────┐          │
     │      │          │
     ▼      │          ▼
 APP main   │    行199: 显示"No App!"
 VTOR=0xC000│    行210-218: while(1) + Power_DisEnable()
 →LVGL/RTOS │    (关闭电池，手表断电)
```






