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



### 样式系统




### 事件系统
**实现交互的核心，通过事件回调机制让组件能响应用户操作。** 



### 任务系统
**负责处理界面刷新、动画效果这些后台系统** 
















