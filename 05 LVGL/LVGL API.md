
### lv_obj_t:
1. 是所有对象的模板，定义了对象都具备的基础属性和方法
2. 屏幕是它的一个具体实例，可以把屏幕想象成一个最大的容器对象，它是其他组件的根节点
3. 按钮、标签这些组件都是基于lv_obj_t拓展出来的

### 屏幕：
创建界面最常见的方法是创建一个带有父节点的基础小部件。例如，`NULL`
```C
//获取当前硬件的整个屏幕
lv_obj_t * my_screen = lv_obj_create(NULL);     //填NULL就是整个硬件屏幕

//创建一个新的屏幕
lv_obj_t * obj =lv_obj_create(my_screen);   //从my_screen下创建一个子类屏幕,会有默认的样式
```

### 组件：
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

### 事件
你可以为一个组件分配一个或多个回调，当组件件 被点击、释放、拖拽、被删除等。
```C

lv_obj_add_event_cb(btn, my_btn_event_cb, LV_EVENT_CLICKED, NULL);
...

void my_btn_event_cb(lv_event_t * e)
{
    printf("Clicked\n");
}
```



































