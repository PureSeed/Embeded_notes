# LVGL移植到STM32HAL库
1. 创建STM32HAL项目
2. 将LVGL添加到项目中（只添加有用的即可）。如根目录下的.h文件，src源码，demos，porting下与STM32兼容的文件
3. 时钟、显示驱动、输入设备驱动，RTOS配置

将LVGL移植到新平台，核心就是实现 **“时间基准”、“显示驱动”和“输入设备驱动”** 这三个硬件相关的接口

### 第一步：准备工作
1. **获取源码**：从LVGL的GitHub仓库[下载或克隆]源码
2. **复制到工程**：将`lvgl`文件夹（**核心是其中的`src`目录**）复制到你的项目目录下
3. **创建配置文件**：将`lvgl/lv_conf_template.h`复制到`lvgl`文件夹的**同级目录**，**并重命名为`lv_conf.h。然后打开文件，将开头的`#if 0`改为`#if 1`以启用配置

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
可以参考OV-Watch项目中的：`User/Tasks/Src/user_TasksInit.c:231-246`，函数名叫 **`LvHandlerTask`**，通过任务实现周期性调用
### 第六步：RTOS集成
如果你使用FreeRTOS等RTOS，建议：
1. 创建一个专用任务来调用 `lv_timer_handler()
2. 通过 `LV_USE_OS` 配置项启用LVGL的线程安全机制

### 关键要点总结
- **测试先行**：建议先在PC上使用SDL2等模拟器进行UI逻辑验证，再移植到硬件，能极大提升开发效率
- **性能优化**：使用DMA进行数据传输能显著提升屏幕刷新性能
- **善用模板**：LVGL源码的 `examples/porting/` 目录下提供了 `lv_port_disp_template.c` 和 `lv_port_indev_template.c` 等移植模板文件，可以直接参考和修改

----------------
# LVGL的目录结构
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

-------------
# LVGL的裁剪
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


















