# 三步走
### 显示图标： 学会在屏幕放置组件和图标
1. **初始化LVGL：** 调用lv_init()完成库初始化
2. **移植显示驱动：** 实现lv_port_disp_init()完成适配
3. **创建组件：** 使用lv_label_create()创建文本标签
4. **显示图标图片：** 使用lv_img_create()加载并显示图片

```c
// 创建屏幕对象
lv_obj_t *scr = lv_scr_act();

// 创建文本标签
lv_obj_t *label = lv_label_create(scr);
lv_label_set_text(label, "Hello LVGL!");
lv_obj_align(label, LV_ALIGN_CENTER, 0, -20);

// 创建图标
lv_obj_t *img = lv_img_create(scr);
lv_img_set_src(img, &my_icon);
lv_obj_align(img, LV_ALIGN_CENTER, 0, 20);
```

### 接入按键：
1. **输入设备初始化：** 调用 lv_port_indev_init() 完成输入驱动适配
2. **实现按键读取函数：** 编写 lv_keyboard_read() 获取当前按键状态
3. **配置按键映射：** 将硬件按键映射为LVGL标准按键码
4. **测试输入功能：** 使用 lv_btn_create() 创建按钮测试按键响应
```C
// 1. 按键读取函数
static void keypad_read(lv_indev_drv_t * d)
{
    static uint32_t last_key = 0;
    // 读取GPIO电平判断按键状态
    uint32_t act_key = 0;
    if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0))
    if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1))
    if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_2))

    if(act_key != 0) {
        data->state = LV_INDEV_STATE_PR;
        last_key = act_key;
    } else {
        data->state = LV_INDEV_STATE_REL;
    }
    data->key = last_key;
}

// 2. 输入设备初始化
void lv_port_indev_init(void)
{
    static lv_indev_drv_t indev_drv;
    lv_indev_drv_init(&indev_drv);
    indev_drv.type = LV_INDEV_TYPE_KEYPAD;
    indev_drv.read_cb = keypad_read;
    lv_indev_drv_register(&indev_drv);
}

```

### 动作响应：接收到按键指令后执行什么功能
1. **定义事件回调函数：** 编写回调函数，接收事件类型和组件对象
2. **绑定回调函数到组件：** 使用 lv_obj_add_event_cb() 将回调函数绑定到目标组件
3. **判断事件类型：** 在回调函数中通过 event_code 判断具体触发的事件
4. **实现具体业务逻辑：** 根据事件类型编写具体的功能代码，比如更新界面、控制硬件
5. **调试与优化：** 添加日志输出，调试事件响应流程，优化交互体验

```C
// 3. 按键事件回调函数
static void btn_event_cb(lv_event_t * e)
{
    lv_event_code_t code = lv_event_get_code(e);
    if(code == LV_EVENT_CLICKED) {
        LV_LOG_USER("Button clicked!");
        // 在这里添加按键按下后的处理逻辑
    }
}

// 4. 创建按钮并绑定事件
void lv_example_keyboard(void)
{
    lv_obj_t * btn = lv_btn_create(lv_scr_act());
    lv_obj_set_size(btn, 100, 50);
    lv_obj_align(btn, LV_ALIGN_CENTER, 0, 0);
    lv_obj_add_event_cb(btn, btn_event_cb, LV_EVENT_ALL, NULL);

    lv_obj_t * label = lv_label_create(btn);
    lv_label_set_text(label, "Press Enter");
    lv_obj_center(label);
}
```

--------
# 四大系统
核心思想是：通过一个 **“对象树”** 来管理所有UI元素，将用户的交互转化为**事件**，将UI的变化转化为**渲染任务**，并通过一个**定时器驱动的主循环**来调度这一切，最终高效地绘制在屏幕上。
### 对象系统
**LVGL一切皆对象，组件、样式、屏幕皆对象。**
1. **继承体系：** 所有对象都继承自lv_obj_t这个基类，这意味着无论什么组件，都拥有一些共同的属性和方法，比如位置、大小、可见性这些属性，还有事件绑定、样式设置这些通用方法。
2. **树形结构：** 对象组件可以形成父子关系，当你移动父对象时对应的子对象也会跟着一起移动，方便管理。
3. **样式继承：** 子对象可以继承父对象的样式属性，比如你给某个父对象设置了一个样式，那么他的子对象都会沿用这个这个样式，不用单独设置，大大减少了代码量。
4. **事件冒泡:**  当子对象触发事件的时候，事件沿着会对象树向上传递，父对象也能接收到这个事件。比如：你点击了一个按钮，按钮的父对象屏幕也能接收到这个点击事件，这样就可以在父对象层面统一处理一些通用逻辑
LVGL采用**父子层级结构**来管理所有控件。
- **根节点**：每个显示设备都有一个**屏幕（Screen）** 作为根对象。
- **父子关系**：创建控件时指定父对象，它就成为父对象的子节点。子对象的位置、可见性等会跟随父对象。
- **层级顺序**：默认后创建的对象显示在上层，从而形成自然的视觉层叠。

### 样式系统
1. **样式属性：** 涵盖颜色、字体、边框、圆角、阴影、渐变等所有视觉属性。
2. **状态样式：** 支持正常、按下、选中、禁用等不同状态，让界面交互更加直观。
3. **主题系统：** 预设多种主题，一键切换界面风格。也可以创建自定义主题


### 事件系统
**实现交互的核心，通过事件回调机制让组件能响应用户操作。** 
1. **事件类型：** 支持点击、长按、滑动、值变化等几十种事件，支持事件自定义，可以创建和发送自定义事件
2. **事件绑定与参数传递：** 通过lv_obj_add_event_cb()绑定回调函数，通过event_data传递事件相关信息


### 任务系统
**负责处理界面刷新、动画效果这些后台系统，是轻量级的后台调度系统**  
1. **定时器驱动：** 基于SysTick定时器提供时钟基准
2. **任务调度：** 通过lv_timer_handler()处理所有任务，比如界面刷新、动画更新等，通常建议5-10毫秒调用一次这个函数。
3. **定时器任务：** 创建一次性或周期性定时器任务
4. **动画系统：** 基于任务系统实现流畅动画效果，创建一个动画时，LVGL会自动创建一个定时器任务，用来不断更新动画状态，实现流畅动画效果。
5. **低功耗支持：** 当没有任务需要处理时，LVGL会自动进入休眠模式。

----------------
# 内部机制
核心思想是：通过一个 **“对象树”** 来管理所有UI元素，将用户的交互转化为**事件**，将UI的变化转化为**渲染任务**，并通过一个**定时器驱动的主循环**来调度这一切，最终高效地绘制在屏幕上。
### 1. 对象树 (Widget Tree)：UI的骨架

LVGL采用**父子层级结构**来管理所有控件。
- **根节点**：每个显示设备都有一个**屏幕（Screen）** 作为根对象。
- **父子关系**：创建控件时指定父对象，它就成为父对象的子节点。子对象的位置、可见性等会跟随父对象。 
- **层级顺序**：默认后创建的对象显示在上层，从而形成自然的视觉层叠。 

### 2. 渲染机制 (Rendering)：如何绘制画面

LVGL的渲染机制是现代、高效的，其核心演变如下：
- **绘制缓冲 (Draw Buffer)**：LVGL不在屏幕上直接绘制，而是先绘制到一个内部缓冲区，绘制完成后再一次性刷新到屏幕[](https://docs.lvgl.io/9.1/overview/draw.html#hook-drawing)。这种方式能有效避免画面撕裂和闪烁。关键在于，这个缓冲区可以远小于屏幕尺寸，对内存受限的嵌入式设备非常友好。
- **绘制管道 (Drawing Pipeline, LVGL v9+)**：这是LVGL v9引入的重大改进。它将复杂的绘制任务在内部流水线化处理：
    1. **收集绘制任务 (Draw Tasks)**：当UI需要更新时，系统会生成一个或多个“绘制任务”。每个任务都包含了绘制位置、大小、类型（如矩形、标签）等所有必要信息
    2. **分发给绘制单元 (Draw Units)**：这些任务会被分配给不同的“绘制单元”去执行。绘制单元可以是CPU核心，也可以是专用的GPU硬件。这套机制使得LVGL能高效地利用各种硬件加速能力。

- **屏幕刷新机制**：
    1. **标记无效区域**：当UI变化时（如按钮被按下），LVGL会计算并记录下需要重绘的区域，称为“无效区域”
    2. **合并与裁剪**：在下一个刷新周期，系统会合并相邻的无效区域，并裁剪掉被遮挡的部分，以最小化绘制工作量
    3. **执行绘制**：按照从顶层到底层的顺序，将可见部分绘制到缓冲区中。
    4. **调用刷新回调**：绘制完成后，通过你注册的`flush_cb`回调函数，将缓冲区内容发送到屏幕显示

### 3. 定时器与动画 (Timer & Animation)：驱动的脉搏

- **定时器 (Timer)**：LVGL的内部定时器是其运转的动力来源。你需要在主循环中**周期性地调用 `lv_timer_handler()`，它会驱动所有定时器任务。** LVGL本身就用定时器来执行屏幕刷新和输入设备轮询。这些定时器是**非抢占式**的，会依次执行，因此在其回调中可以安全地调用任何LVGL函数。
- **动画 (Animation)**：动画是在定时器驱动下，通过插值计算实现的。你只需定义起始值、结束值、时长和变化曲线，LVGL就会在每一帧自动计算中间值并应用到目标对象上。 

### 4. 事件系统 (Event System)：交互的桥梁

事件是LVGL中对象间通信和响应用户交互的主要方式
- **注册与触发**：你可以通过 `lv_obj_add_event_cb()` 为对象注册回调函数，以监听特定事件。当用户点击、拖动或对象状态改变时，相应事件就会被触发
- **事件类型**：事件种类丰富，涵盖输入设备事件（如点击`LV_EVENT_CLICKED`）、绘制事件、通用事件（如删除`LV_EVENT_DELETE`）等
- **事件冒泡**：事件可以从子对象逐级向上传递给父对象，方便在父容器中统一处理子控件的事件

### 5. 输入设备处理 (Input Device Handling)

LVGL通过**轮询**机制处理输入。
- **读取回调**：你需要实现一个读取函数（`read_cb`），LVGL会通过一个内部定时器定期调用它，来获取触摸坐标或按键状态。
- **事件生成**：LVGL根据读取到的原始数据（如按下、移动、释放），自动合成更高级的UI事件（如点击、长按、拖动）并分发出去。 

### 6. 内存管理 (Memory Management)

LVGL拥有自己的内存管理模块。
- **动态内存**：所有控件、样式、动画等对象的创建和销毁，都通过 `lv_malloc()`, `lv_free()` 等函数在LVGL的堆区进行管理
- **内存配置**：你可以通过配置文件 `lv_conf.h` 中的 `LV_MEM_SIZE` 来指定堆的大小
- **生命周期**：控件由其父对象管理，删除父对象会自动递归删除其所有子对象，有效防止内存泄漏

总而言之，LVGL的内部机制可以看作是一个**由定时器驱动、事件响应、高效渲染**的闭环系统。它通过“对象树”组织UI，用“绘制管道”和“缓冲”机制高效绘图，用“事件系统”响应用户交互，所有模块协同工作，构成了一个流畅而强大的图形界面框架。

------
# 核心流程

| **1. 硬件初始化**     | 用户自行实现                                                                                 | 初始化MCU时钟、GPIO、SPI/I2C等总线，以及LCD、触摸屏等外设[](https://lvgl.github.io/open-docs/HTML/9.4/intro/getting_started/learn_the_basics.html#screens)。                                                                                       |
| ---------------- | -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **2. LVGL核心初始化** | `lv_init()`                                                                            | **必须第一个调用**，初始化LVGL内部管理结构                                                                                                                                                                                                     |
| **3. 配置系统心跳**    | `lv_tick_set_cb()` 或 `lv_tick_inc()`                                                   | 为LVGL提供毫秒级时间基准，用于动画和事件                                                                                                                                                                                                        |
| **3. 创建并配置显示**   | `lv_display_create()`  <br>`lv_display_set_buffers()`  <br>`lv_display_set_flush_cb()` | 创建显示设备、分配显示缓冲区、并注册底层屏驱函数。                                                                                                                                                                                                     |
| **3. 创建并配置输入**   | `lv_indev_create()`  <br>`lv_indev_set_type()`  <br>`lv_indev_set_read_cb()`           | 创建输入设备（如触摸屏）、设置类型、并注册读取函数。                                                                                                                                                                                                    |
| **4. 创建用户界面**    | 各种LVGL控件API                                                                            | 创建屏幕、控件、样式、动画等[](https://lvgl.github.io/open-docs/HTML/9.4/intro/getting_started/learn_the_basics.html#screens)。                                                                                                              |
| **5. 主循环**       | `lv_timer_handler()`                                                                   | **必须周期性调用**，驱动LVGL执行所有内部任务[](https://docs.lvgl.io/9.3/details/integration/adding-lvgl-to-your-project/connecting_lvgl.html)[](https://lvgl.github.io/open-docs/HTML/9.4/intro/getting_started/learn_the_basics.html#screens)。 |
1. 初始化时钟等外设
2. 调用lv_init(),初始化lvgl的配置
3. 添加显示设备，输入设备，系统定时器
4. 创建UI界面
5. 周期调用lv_timer_handler 函数
**可将lvgl视为.c .h文件，可添加至任何项目中**

```C
/* -------------------  主函数（启动流程） ------------------- */
int main(void)
{
    /* 第一步：LVGL核心初始化，必须最先调用 */
    lv_init();

    /* 第二步：提供系统心跳 */
    lv_tick_set_cb(my_tick_get_cb);   // 注册获取毫秒的回调

    /* 第三步：创建显示设备并配置 */
    lv_display_t *disp = lv_display_create(MY_DISP_HOR_RES, MY_DISP_VER_RES);
    lv_display_set_buffers(disp, disp_buf1, disp_buf2, sizeof(disp_buf1), LV_DISPLAY_RENDER_MODE_PARTIAL);
    lv_display_set_flush_cb(disp, my_flush_cb);

    /* 第四步：创建输入设备（触摸/按键等） */
    lv_indev_t *indev = lv_indev_create();
    lv_indev_set_type(indev, LV_INDEV_TYPE_POINTER);   // 触摸屏类型
    lv_indev_set_read_cb(indev, my_read_cb);

    /* 第五步：创建UI（一个简单的按钮示例） */
    lv_obj_t *scr = lv_screen_active();                // 获取活动屏幕
    lv_obj_t *btn = lv_button_create(scr);             // 创建按钮
    lv_obj_set_size(btn, 120, 50);                     // 设置大小
    lv_obj_center(btn);                                // 居中显示

    lv_obj_t *label = lv_label_create(btn);            // 在按钮上创建标签
    lv_label_set_text(label, "Hello LVGL");            // 设置文字
    lv_obj_center(label);

    /* 第六步：主循环 - 驱动LVGL */
    while (1) {
        lv_timer_handler();    // 处理所有LVGL内部事务（刷新、事件、动画等）
       
         usleep(5000);    //不一直占用系统资源
    }

    return 0;
}
```

-----------















