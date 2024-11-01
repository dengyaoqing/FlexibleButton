# 模块介绍
按键支持`开始按下、单击、双击、多击、短按、长按、持续长按、释放`等事件检测。  
最多支持按键注册数量为：`sizeof(btn_type_t) * 8),关键看btn_type_t变量类型占几个字节`

## 使用教程
1、设置扫描频率`FLEX_BTN_SCAN_FREQ_HZ`  
2、实现按键状态读取和事件处理回调函数  
3、创建按键组，定义按键ID和短按长按时长（一个按键对应一个结构体）  
4、初始化按键结构体并注册按键  
 

# 应用示例
## 设置扫描频率
```
#define FLEX_BTN_SCAN_FREQ_HZ 50 // How often flex_button_scan () is called
```
默认50Hz，即20ms扫描一次

## 读状态回调
```
uint8_t key_read_state(void *arg)
{
    flex_button_t *button = (flex_button_t *)arg;
    uint8_t state = 0;

    switch(button->id){
        case KEY_ID_A: 
            state = HAL_GPIO_ReadPin(KEY_A_GPIO_Port, KEY_A_Pin);
            break;

        case KEY_ID_B: 
            state = HAL_GPIO_ReadPin(KEY_B_GPIO_Port, KEY_B_Pin);
            break;

        case KEY_ID_C: 
            state = HAL_GPIO_ReadPin(KEY_C_GPIO_Port, KEY_C_Pin);
            break;

        default:
            break;
    }
    return state;
}
```
按键读状态回调函数，函数参数为按键句柄

## 事件处理回调
```
void key_event_handler(void *arg)
{
    flex_button_t *button = (flex_button_t *)arg;

    switch(button->id){
        case KEY_ID_A: //聚焦
            if(FLEX_BTN_PRESS_DOWN == button->event){
                xTaskNotify(usbTaskHandle, USB_NOTIFY_FOCUS, eSetValueWithOverwrite);
            }else if(FLEX_BTN_PRESS_RELEASE == button->event){
                xTaskNotify(usbTaskHandle, USB_NOTIFY_CAMERA_STOP, eSetValueWithOverwrite);
            }
            break;

        case KEY_ID_B: //拍照
            if(FLEX_BTN_PRESS_DOWN == button->event){
                xTaskNotify(usbTaskHandle, USB_NOTIFY_SHUTTER, eSetValueWithOverwrite);
            }else if(FLEX_BTN_PRESS_RELEASE == button->event){
                xTaskNotify(usbTaskHandle, USB_NOTIFY_CAMERA_STOP, eSetValueWithOverwrite);
            }
            break;

        case KEY_ID_C: //录像
            if(FLEX_BTN_PRESS_DOWN == button->event){
                xTaskNotify(usbTaskHandle, USB_NOTIFY_RECORD_START, eSetValueWithOverwrite);
            }else if(FLEX_BTN_PRESS_RELEASE == button->event){
                xTaskNotify(usbTaskHandle, USB_NOTIFY_RECORD_STOP, eSetValueWithOverwrite);
            }
            break;
        
        default:
            break;
    }
}
```
事件处理回调函数，函数参数为按键句柄。针对不同按键的不同事件进行处理。


## 初始化配置
* 定义按键ID
```
typedef enum{/**按键ID号 */
    KEY_ID_A = 0,
    KEY_ID_B,
    KEY_ID_C,
    KEY_ID_MAX,
}KeyID_e;
```
这里用到了三个按键依次定义ID号

* 定义短按、长按时长
```
#define KEY_SHORT_PRESS_TIME      1000
#define KEY_LONG_PRESS_TIME       2000
#define KEY_LONG_HOLD_PRESS_TIME  3000
```
时长自定义，单位：ms，在赋值时需要使用`FLEX_MS_TO_SCAN_CNT`进行转换

* 创建按键结构体数组（句柄）
`flex_button_t btnGroup[KEY_ID_MAX] = {0};`
每个按键对应一个结构体

* 按键结构体初始配置与按键注册
```
void key_init(void)
{
    uint8_t keyIndex = 0;

    memset(&btnGroup[0], 0, sizeof(btnGroup));//清除结构体
    for(keyIndex = 0; keyIndex < KEY_ID_MAX; keyIndex++){
        btnGroup[keyIndex].id = keyIndex;                       //按键ID
        btnGroup[keyIndex].pressed_logic_level = 0;             //按键按下时的逻辑电平
        btnGroup[keyIndex].usr_button_read = key_read_state;    //按键读状态回调函数（传参：按键句柄）
        btnGroup[keyIndex].cb = key_event_handler;              //按键事件处理回调（传参：按键句柄）
        btnGroup[keyIndex].short_press_start_tick = FLEX_MS_TO_SCAN_CNT(KEY_SHORT_PRESS_TIME);  //短按时长
        btnGroup[keyIndex].long_press_start_tick = FLEX_MS_TO_SCAN_CNT(KEY_LONG_PRESS_TIME);    //长按时长
        btnGroup[keyIndex].long_hold_start_tick = FLEX_MS_TO_SCAN_CNT(KEY_LONG_HOLD_PRESS_TIME);//持续按时长

        flex_button_register(&btnGroup[keyIndex]);      //每个按键初始化完成后需要注册到按键list中
    }
}
```
按键GPIO硬件初始化没有在这里提现，需要另外添加！！！
这里仅对按键结构体进行初始化配置，需要另外实现状态查询和事件处理回调函数，短按长按时长需要使用`FLEX_MS_TO_SCAN_CNT`函数转换

* 按键轮询
需要周期调用`flex_button_scan`函数进行按键扫描，扫描周期与宏定义`FLEX_BTN_SCAN_FREQ_HZ`一致！！！
