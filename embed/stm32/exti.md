# 关于STM32-EXTI的问题
近期项目中遇到一个严重bug,导致主板在特定情况下死机,进而看门狗复位,经过测试后发现程序一直卡在中断程序里.
这里记录下来相关经验.
PS:之前也修过一个死机的bug,是由于串口ORE中断没有处理,程序也是一直卡在中断程序.


## 如何关中断
### 通过EXTI关中断
```
void EXTI9_5_IRQHandler(void)
{
    EXTI_InitTypeDef EXTI_InitStructure;
    if(EXTI_GetITStatus(EXTI_Line8) != RESET)
    {
//        EXTI_ClearITPendingBit(EXTI_Line8); //不清中断标志

        //关闭EXTI
        EXTI_InitStructure.EXTI_Line = EXTI_Line8;  
        EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;                                      
        EXTI_InitStructure.EXTI_LineCmd = DISABLE;  
        EXTI_Init(&EXTI_InitStructure);
    }
}
```
在上述代码中,中断程序没有清中断标志位,但通过EXTI关中断,此时程序会一直不停进入这个中断,最后看门狗复位.

### 通过NVIC关中断
但如果把代码修改如下,不管EXTI,直接在NVIC关掉中断,则不会重复进入此中断.
```
void EXTI9_5_IRQHandler(void)
{
    EXTI_InitTypeDef EXTI_InitStructure;
    if(EXTI_GetITStatus(EXTI_Line8) != RESET)
    {
//        EXTI_ClearITPendingBit(EXTI_Line8); //不清中断标志

        //关闭NVIC中断
        NVIC_InitStructure.NVIC_IRQChannel = EXTI9_5_IRQn;
        NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
        NVIC_InitStructure.NVIC_IRQChannelCmd = DISABLE;
        NVIC_Init(&NVIC_InitStructure);
    }
}
```
由此推断,中断从EXTI提交到NVIC之后,关闭EXTI将不能屏蔽此中断,必须从NVIC关闭中断

## IRQ的处理
### EXTI中断源的判断
EXTI响应代码如下:
```
void EXTI9_5_IRQHandler(void)
{
    if(EXTI_GetITStatus(EXTI_Line8) != RESET)
    {
        EXTI_ClearITPendingBit(EXTI_Line8); //清中断标志
    }
}
```

官方库提供的EXTI_GetITStatus函数:
```
ITStatus EXTI_GetITStatus(uint32_t EXTI_Line)
{
  ITStatus bitstatus = RESET;
  uint32_t enablestatus = 0;
  /* Check the parameters */
  assert_param(IS_GET_EXTI_LINE(EXTI_Line));

  enablestatus =  EXTI->IMR & EXTI_Line;
  if (((EXTI->PR & EXTI_Line) != (uint32_t)RESET) && (enablestatus != (uint32_t)RESET))
  {
    bitstatus = SET;
  }
  else
  {
    bitstatus = RESET;
  }
  return bitstatus;
}
```
这个函数返回中断状态,仅当 __IMR已使能中断__ 且 __PR已设置中断标志位__,才返回SET.

#### bug出现逻辑:
```
    1.EXTI以非常高的频率触发(例如CAN唤醒).
    2.通过EXTI关闭中断,正好在IMR关闭中断时响应了中断.
    3.此时进入IRQ时,IMR没有设置使能位,而PR有设置标志位,EXTI_GetITStatus一直返回RESET.
    4.程序将一直判断不了是哪个中断源,所以没有清中断标志,导致程序不停进入中断,导致死机.
```
#### bug的修复
官方库还提供了EXTI_GetFlagStatus,此函数仅判断PR标志位,而不管IMR有没有使能中断.使用这个函数可以避免上面的bug.
