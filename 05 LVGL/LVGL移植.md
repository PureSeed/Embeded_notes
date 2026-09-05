# 01 LVGL移植到STM32HAL库
1. 创建STM32HAL项目
2. 将LVGL添加到项目中（只添加有用的即可）。如根目录下的.h文件，src源码，demos，porting下与STM32兼容的文件
3. 时钟、显示驱动、输入设备驱动，RTOS配置

将LVGL移植到新平台，核心就是实现 **“时间基准”、“显示驱动”和“输入设备驱动”** 这三个硬件相关的接口

### 第一步：准备工作
1. **获取源码**：从LVGL的GitHub仓库[下载或克隆]源码
2. **复制到工程**：将`lvgl`文件夹（**核心是其中的`src`目录**）复制到你的项目目录下
3. **创建配置文件**：将`lvgl/lv_conf_template.h`复制到`lvgl`文件夹的**同级目录**，**并重命名为`lv_conf.h。然后打开文件，将开头的`#if 0`改为`#if 1`以启用配置
可以参考OV_Watch\Middlewares\LVGL\GUI 文件夹

### 第二步：提供时间基准 (Tick)
LVGL需要一个心跳来驱动动画和定时器。你需要实现一个函数，提供毫秒级的系统时间。
```c
// 示例：假设你有一个硬件定时器中断，每1ms调用一次
void HW_Timer_IRQHandler(void) {
    lv_tick_inc(1); // 通知LVGL过去了1ms
}
// 或者，注册一个回调函数
uint32_t my_tick_get_cb(void) {
    // 返回从系统启动至今的毫秒数
    return get_system_ms();
}
// 在初始化时调用
lv_tick_set_cb(my_tick_get_cb);
```
> **注意**：`lv_tick_inc()`和`lv_tick_set_cb()`二选一即可。前者需在定时器中断中调用，后者则需提供一个获取当前毫秒数的函数。

### 第三步：实现显示驱动 (Display Driver)
这是移植中最关键的一步，你需要实现一个“刷新回调函数”，将LVGL绘制好的图像数据发送到屏幕。
首先，创建并配置一个显示设备(`lv_display_t`)。你需要实现的核心是**刷新回调 (`flush_cb`)**：
```C
// 1. 定义显示缓冲区和分辨率
#define MY_DISP_HOR_RES 320
#define MY_DISP_VER_RES 240
static lv_color_t buf_1[MY_DISP_HOR_RES * 10]; // 部分缓冲
static lv_color_t buf_2[MY_DISP_HOR_RES * 10]; // 第二个缓冲区（可选）
// 2. 实现刷新回调
static void my_flush_cb(lv_display_t *disp, const lv_area_t *area, lv_color_t *color_p) {
    // area 指向需要刷新的区域 (x1, y1) 到 (x2, y2)
    // color_p 是LVGL渲染好的像素数据起始地址
    // 将 color_p 中的数据发送到LCD的指定区域
    lcd_fill_area(area->x1, area->y1, area->x2, area->y2, (uint16_t *)color_p);
    // 重要：通知LVGL刷新完成，可以在缓冲区中继续绘制下一帧
    lv_display_flush_ready(disp);
}
// 3. 在初始化函数中注册
void lvgl_init(void) {
    lv_init();
    lv_display_t *disp = lv_display_create(MY_DISP_HOR_RES, MY_DISP_VER_RES);
    lv_display_set_buffers(disp, buf_1, buf_2, sizeof(buf_1), LV_DISPLAY_RENDER_MODE_PARTIAL);
    lv_display_set_flush_cb(disp, my_flush_cb);
}
```
**优化显示驱动**：
- **DMA加速**：使用DMA（直接内存访问）进行数据传输，将CPU从繁重的数据搬运中解放出来。
- **双缓冲 + 垂直同步**：采用双缓冲机制，并等待LCD的垂直同步（VSYNC）信号再切换缓冲区，能有效避免画面撕裂

关于缓冲区大小，可以选择**部分缓冲**（如示例中的10行），可以节省内存；也可以选择**全尺寸缓冲**，通常性能更好。
> **提示**：LVGL v9开始提供了一些内置的LCD驱动支持，你可以在`lv_conf.h`中启用它们，以简化部分控制器的移植。

可以参考OV-Watch项目中的Middlewares\LVGL\GUI\porting\lv_port_disp.文件。该文件也是用于实现lvgl显示驱动的

### 第四步：实现输入设备驱动 (Input Device Driver)
如果需要触摸、键盘等交互，就需要实现输入设备驱动
创建输入设备(`lv_indev_t`)并实现**读取回调 (`read_cb`)**，LVGL会定期调用它来获取输入状态。
```c
// 1. 实现读取回调
static void my_read_cb(lv_indev_t *indev, lv_indev_data_t *data) {
    // 读取触摸屏或按键状态
    if (touchpad_is_pressed()) {
        data->state = LV_INDEV_STATE_PRESSED;
        data->point.x = touchpad_get_x();
        data->point.y = touchpad_get_y();
    } else {
        data->state = LV_INDEV_STATE_RELEASED;
    }
}
// 2. 在初始化函数中注册
void lvgl_init(void) {
    // ... (显示驱动初始化) ...
    lv_indev_t *indev = lv_indev_create();
    lv_indev_set_type(indev, LV_INDEV_TYPE_POINTER); // 触摸屏类型
    lv_indev_set_read_cb(indev, my_read_cb);
}
```
LVGL支持`LV_INDEV_TYPE_POINTER`（触摸/鼠标）、`LV_INDEV_TYPE_KEYPAD`（键盘）等多种输入设备类型。

可以参考OV-Watch项目中的Middlewares/LVGL/GUI/porting/lv_port_indev.c文件，该文件也是用于实现输入设备驱动

### 第五步：创建主循环
最后，在你的主循环中，需要**周期性地调用** `lv_timer_handler()` 函数
```c
int main(void) {
    // ... 硬件和LVGL初始化 ...
    while(1) {
        lv_timer_handler(); // 处理所有LVGL任务
        delay_ms(5);        // 建议5-10ms调用一次
    }
}
```
lv_task_handler() 内部做的事：
│
├─ ① 处理触摸输入
│     → 调 lv_port_indev.c 的 touchpad_read()
│     → CST816 坐标 → LVGL 内部
│     → 根据坐标找目标控件 → 触发事件回调
│
├─ ② 运行 LVGL 定时器
│     → 检查到点了吗？（用 lv_tick_inc 累加的时间）
│     → HomePage_timer_cb 到500ms了吗？→ 到了就调
│     → 按钮按下动画到时间了吗？→ 到了就推进一帧
│
├─ ③ 重绘脏区域
│     → 哪些控件变了？（label 文字改了、arc 值改了）
│     → 在 draw_buf 里画新像素
│     → 调 lv_port_disp.c 的 disp_flush() 推给屏幕
│
└─ ④ 返回值：下次最短多久再调我（ms）
      → 比如有个动画正在跑，还有5ms要推进 → 返回5
      → 如果没什么事，返回30

可以参考OV-Watch项目中的：`User/Tasks/Src/user_TasksInit.c:231-246`，函数名叫 **`LvHandlerTask`**，通过任务实现周期性调用

### 第六步：RTOS集成
如果你使用FreeRTOS等RTOS，建议：
1. 创建一个专用任务来调用 `lv_timer_handler()
2. 通过 `LV_USE_OS` 配置项启用LVGL的线程安全机制

### 关键要点总结
- **测试先行**：建议先在PC上使用SDL2等模拟器进行UI逻辑验证，再移植到硬件，能极大提升开发效率
- **性能优化**：使用DMA进行数据传输能显著提升屏幕刷新性能
- **善用模板**：LVGL源码的 `examples/porting/` 目录下提供了 `lv_port_disp_template.c` 和 `lv_port_indev_template.c` 等移植模板文件，可以直接参考和修改

-------
# 02 移植后要干什么
## 第1件事：先搭「页面栈管理器」（PageManager）—— 所有UI的地基

### 为什么先搭这个？

LVGL 本身是**单屏幕驱动**，没有「多页面切换」的概念（比如从首页点进心率页，再返回首页）。手表是多页面产品，必须自己实现页面栈——**这是整个UI系统的骨架**，后面 SquareLine 画的所有页面都要注册到这个栈里。

### 工程里的实现

**位置**：`User/Func/Src/PageManager.c` **核心逻辑**：用数组存页面栈，每个页面是个 `Page_t` 结构体：
```C
// Page_t 定义（在 ui_helpers.h 或 PageManager.h 里）
typedef struct {
    void (*init)(void);      // 页面初始化（SquareLine 生成的 xxxPage_screen_init）
    void (*deinit)(void);    // 页面销毁（作者手写，必须销毁 lv_timer 防内存泄漏）
    lv_obj_t **obj;          // 页面根对象指针（指向 ui_HomePage 这种全局变量）
} Page_t;

// 全局页面栈（工程里最多存10个页面，够手表用了）
Page_t PageStack[10];
uint8_t stack_top = 0;
Page_t *CurrentPage = NULL;  // 当前显示的页面
```

**核心 API（工程里的这三个函数，所有上层 UI 和 Task 都要调）**：
```c
void Page_Load(Page_t *page)   // 压栈：进新页面（比如首页→心率页）
void Page_Back(void)           // 弹栈：返回上一页（心率页→首页）
void Page_Back_Bottom(void)    // 清栈：返回首页（任意页→首页）
```

### 移植后先做这个的好处
SquareLine 画页面的时候，**你只要在每个页面的 `.c` 文件末尾加一行 `Page_t Page_xxx = {xxx_init, xxx_deinit, &ui_xxx}`**，就能自动接入页面切换系统——不用每个页面自己写切换逻辑。

后面的事情（下面的第2--第4件事），无非就是创建具体的页面，而**每个页面都无非是：页面注册结构体；创建LVGL控件、事件、定时器；事件和定时器的回调函数实现；销毁页面。** 
**每个页面的标准结构：** 
- `Page_t Page_XXX`     页面注册结构体
- `ui_XXXPage_init()`   创建LVGL控件、定时器、事件
- `ui_XXXPage_deinit()` 资源释放
- 各种控件的事件回调函数的具体实现

---
## 第2件事：用 SquareLine Studio 画UI（生成80%的代码）

### 为什么用 SquareLine？
LVGL 原生写 UI 是纯 C 代码（比如 `lv_obj_create` / `lv_obj_set_width`），240×240 的手表屏写个首页要 400 多行纯布局代码，改位置全靠瞎调数字。SquareLine 是 LVGL 官方的**可视化拖拽工具**，拖控件→设属性→一键导出C代码，比手写快 10 倍。

### 工程里的 SquareLine 生成文件
**位置**：`User/GUI_App/`（全部是 SquareLine 导出的，除了几个作者手动改的地方）
```
GUI_App/
├── ui.c / ui.h          ← UI总入口（作者加了 Pages_init 和 Page_Load）
├── ui_helpers.c/h       ← SquareLine 自动生成的辅助函数（不用改）
├── Screens/             ← 所有页面（SquareLine 拖拽生成布局，作者手写事件/定时器）
│   ├── ui_HomePage.c    ← 首页
│   ├── ui_HRPage.c      ← 心率页
│   └── ...（20个页面）
├── Fonts/               ← 作者用 lv_font_conv 转的自定义字体（思源黑体+iconfont）
└── IMGs/                ← 作者用 png_to_c 转的图片（罗盘指针）
```

### SquareLine 导出后必须手动改的 3 件事（工程里都改了）

#### 1. 在 ui_init() 里加页面栈初始化（SquareLine 不知道你有 PageManager）
```c
// 原来 SquareLine 生成的 ui_init() 最后是 lv_scr_load
// 作者改成：
void ui_init(void) {
    // ... SquareLine 生成的主题初始化、控件创建 ...
    Pages_init();          // ★ 作者加的：把所有 Page_t 注册进栈
    Page_Load(&Page_Home); // ★ 作者加的：默认进首页（原来 SquareLine 是 lv_scr_load）
}
```

#### 2. 给每个页面加 Page_t 定义（SquareLine 不知道 PageManager）
```c
// 比如 ui_HomePage.c 末尾，作者手动加了（SquareLine 不会生成这个）
Page_t Page_Home = {ui_HomePage_screen_init, ui_HomePage_screen_deinit, &ui_HomePage};
```

#### 3. 写 screen_deinit（SquareLine 不会生成这个！必须自己写）
SquareLine 生成的页面只有 init，没有 deinit，但 **LVGL 的 lv_timer 如果不销毁，页面切走了还会跑，轻则内存泄漏，重则崩溃**——工程里每个页面的 deinit 都是作者手写的：
```c
// ui_HomePage.c 末尾，作者手写的销毁函数
void ui_HomePage_screen_deinit(void) {
    lv_timer_del(ui_HomePageTimer);  // ★ 必须销毁定时器！
}
```

---
## 第3件事：写「事件回调」—— 把UI控件和硬件连起来

### 为什么这步是核心？
SquareLine 只管**画控件**（比如拖个按钮、滑条），但**按钮点了要干什么、滑条拉了要调什么**，SquareLine 完全不知道——这部分**100%是作者手写的**，也是整个UI系统的「业务逻辑核心」。

### 固定写法（工程里所有事件回调都遵循这个模板）
```c
// SquareLine 生成的函数名和绑定代码，但函数体是作者写的
void ui_event_BLEButton(lv_event_t * e) {
    // 前3行是 SquareLine 生成的模板，不用改
    lv_event_code_t event_code = lv_event_get_code(e);
    lv_obj_t * target = lv_event_get_target(e);
    
    // ★ 从这里开始全是作者手写的业务逻辑
    if(event_code == LV_EVENT_VALUE_CHANGED && lv_obj_has_state(target, LV_STATE_CHECKED)) {
        HWInterface.BLE.Enable();   // → 调中间层，HWInterface 再调 BSP KT6328
        ui_HomePageBLEEN = 1;       // → 同步本地状态（UI 显示用）
    }
    if(event_code == LV_EVENT_VALUE_CHANGED && !lv_obj_has_state(target, LV_STATE_CHECKED)) {
        HWInterface.BLE.Disable();
        ui_HomePageBLEEN = 0;
    }
}
```

### 工程里的事件回调分三类（对应三种交互）

|交互类型|例子|调了什么|
|---|---|---|
|硬件控制|BLE按钮、LCD亮度滑条、关机滑条|`HWInterface.BLE.Enable()`、`HWInterface.LCD.SetLight()`、`HWInterface.Power.Shutdown()`|
|页面切换|首页右滑→菜单页、电源按钮→电源页|`Page_Load(&Page_Menu)`、`Page_Back()`|
|状态同步|NFC按钮 checked/unchecked|改本地变量 `ui_HomePageNFCEN`（UI 显示用）|

---
## 第4件事：写「定时器回调」—— 后台自动刷数据

### 为什么需要定时器？
手表的动态数据（时间、步数、电量、心率、温湿度）**不是用户点按钮触发的**，而是要**后台周期性更新**——比如首页每500ms刷一次时间，心率页每1s刷一次心率。

### 固定写法（工程里所有定时器回调都遵循这个模板）
```c
// 函数体100%作者手写，SquareLine 只会生成注册代码
static void HomePage_timer_cb(lv_timer_t * timer) {
    // ① 先判断当前是不是在这个页面（别在后台页刷没用的UI，浪费CPU）
    if(Page_Get_NowPage() != &Page_Home) return;
    
    // ② 从 HWInterface 读数据（HWInterface 是中间层，数据来自其他Task或BSP）
    HWInterface.RealTimeClock.GetTimeDate(&DateTime);
    
    // ③ 判变化（没变化就不刷UI，省SPI带宽）
    if(ui_TimeMinuteValue != DateTime.Minutes) {
        ui_TimeMinuteValue = DateTime.Minutes;
        lv_label_set_text(ui_TimeMinuteLabel, ...);  // 刷 UI 控件
    }
    if(ui_BatArcValue != HWInterface.Power.power_remain) {
        ui_BatArcValue = HWInterface.Power.power_remain;
        lv_arc_set_value(ui_BatArc, ui_BatArcValue);  // 刷弧形电量条
    }
    // ... 步数、温度、湿度同理
}

// SquareLine 生成的注册代码（作者选的 500ms 周期）
ui_HomePageTimer = lv_timer_create(HomePage_timer_cb, 500, NULL);
```

### 定时器的「数据来源」—— 和其他Task的配合
定时器回调**不直接碰BSP**（比如不直接调 `HAL_RTC_GetTime`），而是读 **HWInterface 的全局变量**——HWInterface 的数据由其他 Task 后台刷新：
```
SensorDataTask（每500ms）→ 读 BSP → 写 HWInterface.AHT21.humidity
                                                    ↑
HomePage_timer_cb（每500ms）→ 读 HWInterface.AHT21.humidity → 刷 UI
```
这种设计让 UI 层和硬件彻底解耦——UI 只管读 HWInterface，不管数据是哪个 Task 写的、怎么从硬件读的。

---
## 移植后第5件事：对接 FreeRTOS Task（UI 和系统的通信）
LVGL UI 只是整个系统的一部分，必须和其他 FreeRTOS Task 配合——**核心原则是「UI层不直接碰其他Task，通过HWInterface或消息队列通信」**。

### 工程里的三种通信方式（对应不同场景）

#### 1. UI → 硬件/硬件相关Task：直接调 HWInterface（简单同步场景）
```c
// 事件回调里直接调（HWInterface 是中间层，可能直接调BSP，可能发队列）
HWInterface.BLE.Enable();   // → 中间层里可能直接调 KT6328_Enable()，也可能发消息给 MessageSendTask
```

#### ==2. UI → 页面切换Task：发 FreeRTOS 消息队列（异步场景）==
```c
// 比如首页右滑事件，要通知 ScrRenewTask 切页面（UI 不直接切，让专门的Task切）
void ui_event_HomePage(lv_event_t * e) {
    if(lv_event_get_code(e) == LV_EVENT_GESTURE && lv_dir == LV_DIR_RIGHT) {
        uint8_t msg = 1;
        osMessageQueuePut(ScrRenew_MessageQueue, &msg, NULL, 1);  // 发消息给 ScrRenewTask
    }
}
```
### 必须放独立 RTOS 任务的：

> **任何涉及 I2C/SPI/UART 读写、延时等待、CPU 密集计算**
> 
> - 读 AHT21 温湿度（I2C 读，~20ms）→ `SensorDataUpdateTask`
> - 读心率 EM7028（I2C + 算法，~50ms）→ `HRDataUpdateTask`
> - 读 MPU6050 抬手检测（I2C + 判断逻辑）→ `MPUCheckTask`
> 
> **原因**：LVGL 的 `LvHandlerTask` 是 UI 线程，你一旦在这里阻塞了 20ms，整个屏幕就会卡 20ms，触摸就丢了 20ms 的事件。嵌入式手表的 UI 卡顿 = 用户直接扔垃圾桶。


#### 3. 其他Task → UI：写 HWInterface 全局变量（数据更新场景）
```c
// SensorDataTask 后台刷新传感器数据
void SensorDataTask(void *argument) {
    while(1) {
        HWInterface.AHT21.humidity = 65;   // 直接写 HWInterface 的变量
        HWInterface.IMU.Steps += 1;
        osDelay(500);
    }
}
// LvHandlerTask 里的定时器回调会自动读这些变量刷 UI（前面讲过）
```

----------------
# 03 LVGL的目录结构
| 文件/目录                    | 作用                                                                                                         |
| ------------------------ | ---------------------------------------------------------------------------------------------------------- |
| **`src/`**               | **核心源码目录**，包含LVGL库的所有源代码（`.c`和`.h`文件）。这是编译LVGL库时唯一必须包含的目录                                                  |
| **`examples/`**          | **官方示例代码**，展示各种控件和功能的基础用法。如果你是初学者，这个目录是很好的学习资源                                                             |
| **`demos/`**             | **完整功能演示**，包含多页面、动画等更复杂的综合应用场景。                                                                            |
| **`lv_conf_template.h`** | **配置模板文件**，用于配置LVGL的各项功能。**需要特别注意**：此文件是模板，不能直接使用。必须将其**复制**到你的**工程根目录**并**重命名为 `lv_conf.h`**，然后根据项目需求修改配置 |
| **`lvgl.h`**             | **主头文件**，在你的源代码中只需包含此文件即可使用LVGL的全部API                                                                      |
| **`lv_version.h`**       | **版本头文件**，定义了当前LVGL库的版本信息                                                                                  |
##### 核心源码 (`src/`) 目录详解
**`src/` 目录是LVGL功能的核心，其内部子目录按功能划分：**

| 子目录            | 主要功能                                                             |
| -------------- | ---------------------------------------------------------------- |
| **`core/`**    | LVGL的**核心机制**，包括对象系统、事件处理、布局管理和屏幕刷新等                             |
| **`widgets/`** | **所有内置控件**的实现，例如按钮 (`lv_btn`)、标签 (`lv_label`)、滑块 (`lv_slider`) 等 |
| **`draw/`**    | **渲染后端**，负责将图形绘制到显示缓冲区，支持软件渲染 (SW)、GPU 或 DMA2D 等硬件加速             |
| **`hal/`**     | **硬件抽象层**，包含显示驱动和输入设备驱动的接口定义                                     |
| **`font/`**    | **内置字体**文件，如常用的 `lv_font_montserrat_*` 系列                        |
| **`misc/`**    | **杂项功能**，包括定时器、内存管理、日志系统等基础工具                                    |
| **`layouts/`** | **布局管理器**，用于自动排列控件，如Flexbox和Grid布局。                              |
| **`libs/`**    | **第三方库**的支持，例如文件系统、图片解码库（如BMP, PNG, JPG）等                        |
| **`extra/`**   | **扩展功能**，包含主题、额外的布局、图片解码器等。                                      |
| **`others/`**  | 其他杂项或实验性功能模块                                                     |

### 移植后在自己项目中lvgl文件目录规划
```
你的项目/
├── lvgl/          # LVGL核心库
├── lvgl_port/     # 显示、输入等移植层代码
├── ui/            # 所有UI页面和控件
├── app/           # 业务逻辑代码
└── lv_conf.h      # LVGL配置文件
```

-------------
# 04 LVGL的裁剪
LVGL 的裁剪主要分为**源码文件裁剪**和**功能配置裁剪**两个层面，可以结合使用以达到最佳效果。通过合理裁剪，LVGL 的资源占用可以做到很低：**RAM 最低 4kB + 150byte/控件，Flash 最低 100kB**

### 源码文件裁剪：精简库体积
这是第一步，通过删除不必要的源码文件来减小库的体积。
- **保留核心目录**：解压 LVGL 源码后，通常只需保留以下几个核心部分
    - `src/`：LVGL 的核心源码，**必须保留**。   
    - `lvgl.h`：LVGL 的主头文件，**必须保留**。  
    - `lv_conf_template.h`：配置模板文件，需要复制并重命名为 `lv_conf.h

- **删除示例和演示**：`examples/` 和 `demos/` 目录仅用于学习和参考，在最终项目中可以安全删除。
- **精简 `examples/`**：如果需要保留部分示例作为参考，通常只需保留 `examples/porting/` 目录，里面包含了显示和输入设备的移植模板

### 功能配置裁剪：按需保留
这是更精细、更主要的裁剪方式。核心是修改项目根目录下的 **`lv_conf.h`** 配置文件。通过将不需要的宏定义为 `0`，可以在编译时排除相应功能。
#### 1. 基础硬件配置
这些配置直接影响内存占用，需要根据硬件情况进行设置

| 配置宏                                 | 作用            | 优化建议                                                |
| ----------------------------------- | ------------- | --------------------------------------------------- |
| **`LV_COLOR_DEPTH`**                | 设置颜色深度        | 根据屏幕支持选择，常用 `16` (RGB565)。若非必要，避免使用 `32` (ARGB8888) |
| **`LV_MEM_SIZE`**                   | 设置LVGL动态内存池大小 | 根据应用需求调整，不要超过剩余RAM的70%。                             |
| **`LV_DRAW_LAYER_SIMPLE_BUF_SIZE`** | 设置绘制图层缓冲区大小   | 如果遇到内存不足，可以尝试减小此值。                                  |

#### 2. 核心功能裁剪
按需启用或禁用LVGL的各个功能模块

| 配置宏                      | 作用                                                     | 优化建议                                                    |
| ------------------------ | ------------------------------------------------------ | ------------------------------------------------------- |
| **`LV_USE_*`**           | 启用/禁用各类**控件**，如 `LV_USE_BTN` (按钮), `LV_USE_LABEL` (标签) | **只启用项目需要的控件**。例如，若用不到滑块，就设置 `#define LV_USE_SLIDER 0`。 |
| **`LV_USE_THEME_*`**     | 启用/禁用内置**主题**                                          | 只保留一个主题，或完全禁用，使用自定义样式。                                  |
| **`LV_USE_FONT_*`**      | 启用/禁用内置**字体**                                          | 只保留需要的字体，或使用自定义的更小编译字体。                                 |
| **`LV_USE_DRAW_*`**      | 启用/禁用**绘制功能**，如 `LV_DRAW_COMPLEX` (复杂绘制)               | 如果不需要阴影、圆弧等复杂绘制，可以禁用以节省资源。                              |
| **`LV_USE_FILE_SYSTEM`** | 启用/禁用**文件系统**功能                                        | 如果不需要从文件系统加载资源，可以禁用。                                    |

#### 3. 控件依赖关系
关闭某些控件时，要注意其**依赖关系**，否则可能导致编译错误。例如，关闭了下拉列表 (`LV_USE_DROPDOWN`) 后，可能也需要关闭标签的文字选择功能 (`LV_LABEL_TEXT_SELECTION`)。

### 其他优化技巧
- **减少显示缓冲区**：LVGL允许使用**部分帧缓冲**，即缓冲区可以小于屏幕尺寸。这能显著降低RAM占用，但可能会略微影响性能。
- **启用编译器优化**：在编译时使用 `-Os`（针对GCC）等优化选项，可以针对代码体积进行优化
- **外部存储素材**：对于图片、大字体等资源，可以存放在外部Flash（如W25Q系列）中，需要时再加载到内存

### 一个典型的裁剪步骤示例
1. **复制配置文件**：将 `lv_conf_template.h` 复制到项目根目录，并重命名为 `lv_conf.h`。
2. **启用配置文件**：打开 `lv_conf.h`，将文件开头的 `#if 0` 改为 `#if 1`。
3. **设置基础参数**：根据你的屏幕，设置 `LV_COLOR_DEPTH` 和屏幕分辨率等。
4. **裁剪功能模块**：将项目中**不需要**的控件、主题、字体等宏定义为 `0`。可以多次编译测试，逐步裁剪。
5. **调整内存大小**：根据编译后的内存报告，微调 `LV_MEM_SIZE` 等内存配置。

总而言之，LVGL的裁剪是一个从宏观到微观的优化过程。建议先从源码文件入手，然后通过 `lv_conf.h` 进行精细的“功能点餐”，并配合编译器优化，最终为你的项目打造一个轻量高效的LVGL环境。

----------------------


















