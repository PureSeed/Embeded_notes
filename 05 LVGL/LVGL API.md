# 组件特性
### 对象（lv_obj_t:）
1. 是所有对象的模板，定义了对象都具备的基础属性和方法
2. 屏幕是它的一个具体实例，可以把屏幕想象成一个最大的容器对象，它是其他组件的根节点
3. 按钮、标签这些组件都是基于lv_obj_t拓展出来的

### 屏幕(screen)
创建界面最常见的方法是创建一个带有父节点的基础小部件。例如，`NULL`
```C
//获取当前硬件的整个屏幕
lv_obj_t * my_screen = lv_obj_create(NULL);     //填NULL就是整个硬件屏幕

//创建一个新的屏幕
lv_obj_t * obj =lv_obj_create(my_screen);   //从my_screen下创建一个子类屏幕,会有默认的样式
```

### 组件(widget)
组件是用户界面的基本构建模块。例如：按钮lv_button、滑块lv_slider、下拉列表lv_dropdown、图表lv_chart等 
```C
//创建一个按钮
lv_obj_t * btn1 = lv_button_create(lv_screen_active());
lv_obj_set_size(btn1, 100, 50);   //设置按钮尺寸
lv_obj_set_pos(btn1, 20, 30);     //设置按钮位置，左上角位置为原点

//创建一个滑块
lv_obj_t * my_label1 = lv_label_create(slider1);
lv_obj_set_size(slider1, 30, 10);   //设置滑块尺寸
lv_obj_set_pos(slider1, 10, 80);     //设置滑块位置，左上角位置为原点
/* Set slider-specific attributes */
lv_slider_set_range(slider1, 0, 100);               /* Set the min and max values */
lv_slider_set_value(slider1, 40, LV_ANIM_ON);       /* Set the current value (position) */

//延时删除组件
lv_obj_delete_delayed(slider1,3000);   //释放内存

```

### 颜色(color) 状态(states) 不透明度(opacity)
```C
//设置组件背景颜色                                          每个单独的部分都可以设置为不同样式
lv_obj_set_style_bg_color(obj, lv_palette_main(LV_PALETTE_BLUE), LV_PART_MAIN);   //lv_palette_main():调色盘
lv_obj_set_style_bg_color(obj, lv_palette_main(LV_PALETTE_RED), LV_PART_SCROLLBAR);   //LV_PART_SCROLLBAR 滚动条

//不同状态可以设置为不同样式
lv_obj_set_style_bg_color(btn1, lv_palette_main(LV_PALETTE_GREEN), LV_PART_MAIN);   //默认状态
lv_obj_set_style_bg_color(btn1, lv_palette_main(LV_PALETTE_RED), LV_STATE_PRESSED);   //按下后变红

// 设置不透明度  lv_color_lighten(c,100):0就是不改变，255就是白色,设置颜色深浅
lv_obj_set_style_bg_color(slider,lv_color_lighten(lv_palette_main(LV_PALETTE_GREEN)), LV_PART_MAIN); 
lv_obj_set_style_opa(silder,LV_OPA_30, LV_PART_MAIN);

```


### 事件(event)
你可以为一个组件分配一个或多个回调，当组件件 被点击、释放、拖拽、被删除等。
```C
// 添加点击事件             回调函数         触发事件           传递参数
lv_obj_add_event_cb(btn, my_btn_event_cb, LV_EVENT_CLICKED, NULL);

//回调函数
void my_btn_event_cb(lv_event_t * e)
{
    printf("Clicked\n");
}
-----------------------------------------------------------------------------------------------
lv_obj_add_event_cb(btn, my_btn_event_cb, LV_EVENT_VALUE_CHANGED, NULL);

static void slider_callback(lv_event_t * e)
{
	//获取当前滑块的值 
	lv_obj_t * slider = lv_event_get_target(e);
	int32_t value = lv_slider_get_value(slider);
	printf("%d",value);
	
}
```

### 布局(layout) 对齐(align)
``` C
//弹性布局
//布局
lv_obj_t * scr = lv_screen_active();
lv_obj_set_layout(scr, LV_LAYOUT_FLEX);   //设置为弹性布局的容器
lv_obj_set_flex_flow(scr,LV_FLEX_FLOW_ROW);   //设置布局顺序为按行排序，不换行
//对齐  第一个参数：整体结构左对齐/右对齐或上对齐/下对齐   第二个参数：单个元素是哪种对齐   第三个参数：轨道布局的起始位置是哪种对齐
lv_obj_set_flex_align(scr,LV_FLEX_ALIGN_START,LV_FLEX_ALING_END,LV_FLEX_ALIGN_START);

//网格布局
lv_obj_set_layout(scr, LV_LAYOUT_GRID);
static int32_t column_dsc[] = {100, 400, LV_GRID_TEMPLATE_LAST};// 2 columns with 100- and 400-px width 设置网格的列数
static int32_t row_dsc[] = {200, 200, 200, LV_GRID_TEMPLATE_LAST}; // 3 100-px tall rows 设置网格的行数
//设置网格是两列三行
lv_obj_set_grid_dsc_array(obj,column_dsc,row_dsc);
for(uint8_t i=0;i<6;i++)
{
	lv_obj_t *btn = lv_button_create(obj);
	//（1）子组件  （2）列对齐  （3）哪一列  （4）占几列  （5）行对齐  （6）第几行  （7）占几行  
	lv_obj_set_grid_cell(btn,LV_GRID_ALIGN_START,i%2,1,LV_FLEX_ALIGN_START,i/2,1);  
}
//子网格的行列描述如果使用NULL 会继承父类的网格
```

### 滚动(scroll)
```C
lv_obj_t * scr = lv_obj_active();     
lv_obj_t * obj =lv_obj_create(scr);
lv_obj_t * btn =lv_button_create(obj);
//滚动条
lv_obj_set_scrollbar_mode(obj,LV_SCROLLBAR_MODE_AUTO);
//修改滚动条样式
lv_obj_set_style_bg_color(obj,lv_palette_main(LV_PALETTE_RED),LV_PART_SCROLLBAR);
//添加滚动结束的回调函数
lv_obj_add_event_cb(obj,scroll_callback,LV_EVENT_SCROLL_END,NULL);

//吸附功能
lv_obj_add_flag(btn,LV_OBJ_FLAG_SNAPPABLE);
lv_obj_set_scroll_snap_x(obj,LV_SCROLL_SNAP_CENTER);
//添加滚动一次功能
lv_obj_add_flag(btn,LV_OBJpFLAG_SCROLL_ONE);


//回调函数
static void scroll_callback(lv_event_t *e)
{

}
```

------------------------------------
# 组件展示

## 静态组件

#### 标签(lv_label)
``` C
uint16_t label_value=0;

lv_obj_t *label = lv_label_create(btn);   //添加label，添加到btn（按钮）上
lv_label_set_text_fmt("value:%d",lable_value);   //添加字符串信息（标签的内容）
lv_obj_add_event_cb(btn,btn_event_cb,LV_EVENT_CLICKED,NULL); //添加点击按钮的回调

static void btn_event_cb(lv_event_t *e)
{
	label_value++;
	//根据触发对象找到对应的label
	lv_obj_t *btn = lv_event_get_target(e);
	lv_obj_t *label = lv_event_get_child(btn,0);
	lv_label_set_text_fmt(label,"value:%d",label_value);
}
```

#### 动画(lv_anim)
``` C 
//声明两个结构体，
static lv_anim_t   anim_template;
static lv_anim_t * running_anim;

lv_anim_init(&anim_template);  //初始化

/* MANDATORY SETTINGS
 *------------------*/

/* Set the "animator" function */
lv_anim_set_exec_cb(&anim_template, (lv_anim_exec_xcb_t) lv_obj_set_x);

/* Set target of the Animation */
lv_anim_set_var(&anim_template, widget);

/* Length of the Animation [ms] */
lv_anim_set_duration(&anim_template, duration_in_ms);

/* Set start and end values. E.g. 0, 150 */
lv_anim_set_values(&anim_template, start, end);

/* OPTIONAL SETTINGS
 *------------------*/

/* Time to wait before starting the Animation [ms] */
lv_anim_set_delay(&anim_template, delay);

/* Set path (curve). Default is linear */
lv_anim_set_path_cb(&anim_template, lv_anim_path_ease_in);

/* Set anim_template callback to indicate when the Animation is completed. */
lv_anim_set_completed_cb(&anim_template, completed_cb);

/* Set anim_template callback to indicate when the Animation is deleted (idle). */
lv_anim_set_deleted_cb(&anim_template, deleted_cb);

/* Set anim_template callback to indicate when the Animation is started (after delay). */
lv_anim_set_start_cb(&anim_template, start_cb);

/* When ready, play the Animation backward with this duration. Default is 0 (disabled) [ms] */
lv_anim_set_reverse_duration(&anim_template, time);

/* Delay before reverse play. Default is 0 (disabled) [ms] */
lv_anim_set_reverse_delay(&anim_template, delay);

/* Number of repetitions. Default is 1. LV_ANIM_REPEAT_INFINITE for infinite repetition */
lv_anim_set_repeat_count(&anim_template, cnt);

/* Delay before repeat. Default is 0 (disabled) [ms] */
lv_anim_set_repeat_delay(&anim_template, delay);

/* Apply the start value immediately (default true); false: apply start value after
 * delay when the Animation actually starts. */
lv_anim_set_early_apply(&anim_template, true/false);

/* START THE ANIMATION
 *------------------*/
running_anim = lv_anim_start(&anim_template);   /* Start the Animation */

```

#### 矢量动画(lv_lottie)
``` C





```

#### 日历图(lv_calendar)
``` C
//创建日历
lv_obj_t *calendar = lv_calendar_create(lv_screen_active());
//设置尺寸
lv_obj_set_size(calendar,300,300);
//设置日历所处的位置
lv_obj_align(calendar,LV_ALIGN_CENTER,0,0);
//设置日历的日期
lv_calendar_set_today_date(calendar,2026,02,6);
//设置日历显示的月份
lv_calendar_set_month_shown(calendar,2026,02);
//添加年和月份的选择
lv_calendar_add_header_arrow(calendar);
//修改为中国农历
lv_calendar_set_chinese_mode(calendar,true);
//修改字体类型=>修改为包含中文的字体
lv_obj_set_style_text_font(calendar,&lv_font_source_han_sans_sc_14_cjk,LV_PART_MAIN);


//添加点击事件
lv_obj_add_event_cb(calendar,calendar_event_cb,LV_EVENT_VALUE_CHANGED,NULL);

void calendar_event_cb(lv_event *e)
{
	//获取当前日历
	lv_obj_t *calendar = lv_event_get_current_target(e);
	lv_calendar_dat_et date;
	lv_calendar_get_pressed_date(calendar,&date);
	//将点击的日期高亮
	lv_calendar_set_highlighted_dates(calendar,&date,1);
}
```
 
#### 表格(lv_table)
``` C


```

#### 图表(lv_chart)
``` C




```










































