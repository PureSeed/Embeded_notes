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
### 对象系统
**LVGL一切皆对象，组件、样式、屏幕皆对象。**
1. **继承体系：** 所有对象都继承自lv_obj_t这个基类，这意味着无论什么组件，都拥有一些共同的属性和方法，比如位置、大小、可见性这些属性，还有事件绑定、样式设置这些通用方法。
2. **树形结构：** 对象组件可以形成父子关系，当你移动父对象时对应的子对象也会跟着一起移动，方便管理。
3. **样式继承：** 子对象可以继承父对象的样式属性，比如你给某个父对象设置了一个样式，那么他的子对象都会沿用这个这个样式，不用单独设置，大大减少了代码量。
4. **事件冒泡:**  当子对象触发事件的时候，事件沿着会对象树向上传递，父对象也能接收到这个事件。比如：你点击了一个按钮，按钮的父对象屏幕也能接收到这个点击事件，这样就可以在父对象层面统一处理一些通用逻辑


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














