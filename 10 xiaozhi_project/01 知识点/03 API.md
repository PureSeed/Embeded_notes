#### 记录哪些内容
1. **知识点：学到的新知识 | 自己觉得很巧妙、很好的点 | 整体架构是怎么样的** 
2. **陌生的API**: 陌生和难理解的API
3. **可复用的模块代码模板：** 
4. **记录遇到的问题是如何解决的：** Bug调试日志： 现象（问题）、排查过程、根本原因、解决方案、经验总结（以后如何避坑）

#### API
```C
//自动检查函数返回值，发现错误直接打印信息并终止服务
ESP_ERROR_CHECK();

//使用默认参数创建一个标准的事件循环
esp_event_loop_create_default();

//打印程序运行过程中的正常信息，方便调试和监控程序运行状态
ESP_LOGI();

//

```


#### 01 
```C++
	/* 作用域解析运算符，用于访问标准库中的互斥锁类*/
    std::mutex mutex_;  //全局互斥锁,std::mutex是互斥锁类，不是模板

    /* std::deque< 类型>  是双端队列模板，尖括号内是表示装的是什么类型的队列比如 int，string等，std::function<void()>表示存放的是无参数无返回值的函数对象
     * std::deque< 类型>  main_tasks_ ：  创建一个叫做main_tasks_的双端队列实例
     * std::function<void()>                 // 元素类型
     * std::deque<std::function<void()>>     // 队列类型
     * std::deque<std::function<void()>> main_tasks_;  // 对象实例 main_tasks_
     * void () 只是函数类型，它能描述函数长什么样，但不能当变量类型来存东西；std::function<void()> 是可调用对象包装器类型，是“对象类型”，它才能创建变量、装各种可调用对象，表示可以存放任何无参数无返回值的函数对象
     */
    std::deque<std::function<void()>> main_tasks_;  // 主任务队列
```



























