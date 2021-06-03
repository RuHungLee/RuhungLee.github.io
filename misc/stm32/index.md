###### tags: `os`

# stm32 和 freertos 的學習筆記

## 目錄

- [x] stm32 : stm32 外部設備連接狀況與記憶體映射
- [x] stm32 : stm32 時鐘
- [x] stm32 : st 公司提供的啟動流程和硬體驅動程式
- [x] cortex m4 : 中斷和異常處理
- [x] cortex m4 : 睡眠模式
- [x] freertos : 任務排程
- [x] freertos : context switch
- [x] freertos : cpu 使用率量測

## 參考資料 

stm32f4 reference manual
https://reurl.cc/Gd4Rxp

## stm32 外部設備連接狀況與記憶體映射


stm32 上連接外部設備主要的 bus 有三條，一條 AHB，用來連接高速設備，例如 SRAM、Flash、DMA，兩條 APB，用來連接較低速設備，例如 GPIO 以及 Timer。這三條 bus 都有各自的 clk，要啟用某條 bus 上的裝置前，必須設置好該 bus 的 clk。

![](https://i.imgur.com/x7sDXM4.png)

## stm32 時鐘


系統時鐘 ( SYSCLK ) 可以由 HSI，HSE，或是 PLL 來驅動。

HSI RC : 16MHz 的 RC 震盪器。
HSE OSC : 8MHz 的晶振。
PLL : 將 HSI 或 HSE 除頻和倍頻，可以產生客製頻率。

AHB，高速 APB，以及低速 APB 的鐘源來自 SYSCLK，它們會經過不同的  prescalers 產生各自的 clk。AHB 最高頻率為 180 Mhz，高速 APB 90 Mhz，低速 APB 45 Mhz。

![](https://i.imgur.com/5v4BfSV.png)
`只要APB1的時鐘分頻數不為1，TIMx的時鐘頻率就會為APB1時鐘頻率的2倍`


接下來看 st 提供的 system_stm32f10x.c 是如何初始化系統時鐘並設置除頻器的。

st 提供的啟用程式流程:

1.將 data segment 搬到 SRAM。
2.調用 SystemInit() 啟用系統時鐘。
3.執行 main 函數。

SystemInit 調用 SetSysClock 配置系統時鐘，並設置 HCLK、PCLK2、PCLK1 的 prescalers。
`/* Configure the Flash Latency cycles and enable prefetch buffer */`

首先根據 MACRO 定義決定要調用哪一個設定函數。如果都沒有的話，會將 HSI 當作系統時鐘。

```c
//system_stm32f10x.c
static void SetSysClock(void)
{
#ifdef SYSCLK_FREQ_HSE
  SetSysClockToHSE();
#elif defined SYSCLK_FREQ_24MHz
  SetSysClockTo24();
#elif defined SYSCLK_FREQ_36MHz
  SetSysClockTo36();
#elif defined SYSCLK_FREQ_48MHz
  SetSysClockTo48();
#elif defined SYSCLK_FREQ_56MHz
  SetSysClockTo56();
#elif defined SYSCLK_FREQ_72MHz
  SetSysClockTo72();
#endif

 /* If none of the define above is enabled, the HSI is used as System clock
    source (default after reset) */
}
```

以 72 Mhz 的系統時鐘為例，調用 SetSysClockTo72。

SetSysClockTo72 流程如下:


1.開啟 HSE 並等待其穩定。

`RCC->CR |= ((uint32_t)RCC_CR_HSEON);`

![](https://i.imgur.com/2HF1MeH.png)

```????
    /* Select regulator voltage output Scale 1 mode, System frequency up to 168 MHz */
    RCC->APB1ENR |= RCC_APB1ENR_PWREN;
    PWR->CR |= PWR_CR_VOS;
```

2.調整 HCLK(AHB)、PCLK2(APB2)、PCLK1(APB1) 的 prescalers。

```c
    /* HCLK = SYSCLK / 1*/
    RCC->CFGR |= RCC_CFGR_HPRE_DIV1;
      
    /* PCLK2 = HCLK / 2*/
    RCC->CFGR |= RCC_CFGR_PPRE2_DIV2;
    
    /* PCLK1 = HCLK / 4*/
    RCC->CFGR |= RCC_CFGR_PPRE1_DIV4;
```


CFGR 暫存器的 5~8 bits 用來指定 AHB 的 prescaler，11~13 bits 用來指定 APB1 的，14~16 bits 用來指定 APB2 的。

3.調整 PLL 的除頻和倍頻值。

```
    /* Configure the main PLL */
    RCC->PLLCFGR = PLL_M | (PLL_N << 6) | (((PLL_P >> 1) -1) << 16) |
                   (RCC_PLLCFGR_PLLSRC_HSE) | (PLL_Q << 24);
```

**f (VCO clock) = f (PLL clock input) × (PLLN / PLLM)**

PLL_N 被指定在 PLL_CFGR 暫存器的 7~15 bits，PLL_P 在 16~17 bits。


![](https://i.imgur.com/DHzhzbh.png)



4.開啟 PLL 並等待 PLL 穩定。

```
    /* Enable the main PLL */
    RCC->CR |= RCC_CR_PLLON;

    /* Wait till the main PLL is ready */
    while((RCC->CR & RCC_CR_PLLRDY) == 0)
    {
    }
```

5.將 PLL 當作系統時鐘的時鐘源。


```
    /* Select the main PLL as system clock source */
    RCC->CFGR &= (uint32_t)((uint32_t)~(RCC_CFGR_SW));
    RCC->CFGR |= RCC_CFGR_SW_PLL;

    /* Wait till the main PLL is used as system clock source */
    while ((RCC->CFGR & (uint32_t)RCC_CFGR_SWS ) != RCC_CFGR_SWS_PLL);
    {
    }
```

## stm32 時鐘

stm32 時鐘可以依照支援功能分成三類。
* Advanced control timer : timer1、timer8
* General-purpose timers : timer2 ~ timer5
* General-purpose timers : timer9 ~ timer14
* Basic timers : timer6、timer7



### General-purpose timers : timer2 ~ timer5

![](https://i.imgur.com/owUZZIQ.png)


time base unit 包括了 counter register、prescaler register、以及 auto-reload register



APB2_timer_clock : 輸入時間源
prescaler register ( PSC ) : 除頻暫存器
counter register : 計數暫存器
auto-reload register ( ARR ) : 裝載數值暫存器

:::info
PSC ( prescaler register ) 和 ARR ( auto-reload register ) 加上時鐘頻率 ( APB2_timer_clock ) 決定了一個 timer 的溢出週期。
:::


時鐘配置範例

```
TIM_DeInit(TIM2);	
TIM_TimeBaseStructure.TIM_Period = arr;           
TIM_TimeBaseStructure.TIM_Prescaler = psc;	           
TIM_TimeBaseStructure.TIM_ClockDivision = TIM_CKD_DIV1 ;	
TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;
TIM_TimeBaseStructure.TIM_RepetitionCounter = 0;  
TIM_TimeBaseInit(TIM2, &TIM_TimeBaseStructure);
  
TIM_ARRPreloadConfig(TIM2, ENABLE);			 
TIM_Cmd(TIM2, ENABLE); 
```

## 中斷和異常處理


m4 預設的中斷表放在快閃記憶體(flash) 0x4 的位置，如下圖示。CPU 開始執行時會將 MSP = initial SP value，PC = Reset。

```
detection of external signals with a pulse width lower than the APB2 clock period. Refer
to the electrical characteristics section of the STM32F4xx datasheets for details on this
parameter
```

![](https://i.imgur.com/SeWJ3VR.png)


m4 的中斷分為兩種，一種是由 core processor 產生的內核中斷，中斷號 1 ~ 15，一種是由外部設備產生的 IRQ 中斷，中斷號 16 以上，內核中斷由 SCP 暫存器處理，IRQ 中斷由 NVIC 暫存器處理。這裡先討論 IRQ 的部份，每一個中斷 channel 都會有暫存器紀錄其優先權，每個暫存器可以紀錄 4 個優先權，每個優先權以 8bits 表示，8bits 會紀錄兩個優先權，主優先權和次優先權。以下是兩個關於優先權的簡單規則。

```
NMI 屬於哪一種中斷?
```

1.中斷可以搶佔主優先權比自己低的。
2.當兩個中斷都屬於 pending 狀況時，主優先權高的優先執行，如果主優先權一樣，次優先權高的先執行。

![](https://i.imgur.com/PeLRKG8.png)

**EXTI (External interrupt controller)**

stm32f4x 上共有 23 條 EXTI，負責產生外部中斷給 NVIC。以 stm32f405 來說，前 16 條 EXTI 的輸入源為 GPIO 針腳，總共有 140 條 GPIO 連接到 16 條 EXTI 上。 



![](https://i.imgur.com/dhaVQTO.png)

SYSCFG_EXTICR1 可以用來設定 EXTI 要接受輸入的 pin 腳。

![](https://i.imgur.com/24ulPvG.png)

falling trigger 和 rising trigger 暫存器 : 用來設定中斷觸發條件，可以是上升沿觸發、下降沿觸發、上升下降都觸發。
software interrupt/event 暫存器 : 用來產生軟中斷，將 1 寫入暫存器就可以產生中斷。
interrupt mask : enable EXTI。

![](https://i.imgur.com/Zll6JqM.png)

![](https://i.imgur.com/BzOtiek.png)

以下範例程式用來初始化一個以PA0當作輸入源的中斷。

```
void Interrupts_Configuration(void)
{
    EXTI_InitTypeDef EXTI_InitStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

   
    //將 PA0 設定為 EXTI0 的輸入源。
    SYSCFG_EXTILineConfig(EXTI_PortSourceGPIOA, EXIT_PinSource0);

    //設定 EXTI 暫存器，類型為中斷，觸發時機為偵測到上升沿
    EXTI_InitStructure.EXTI_Line = EXTI_Line0;
    EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;
    EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Rising;
    EXTI_InitStructure.EXTI_LineCmd = ENABLE;
    EXTI_Init(&EXTI_InitStructure);

    //因為 EXTI 的中斷訊號最後要送進 NVIC，所以 NVIC 要跟著一起設定。
    NVIC_InitStructure.NVIC_IRQChannel = EXTI0_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0x0F;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0x0F;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

}
```

**異常處理**

以下範例是在 privileged 的 thread mode 下產生異常。

進入異常處理函數前暫存器值如下。

![](https://i.imgur.com/v5Gd0aE.png)

進入異常處理函數後，可以看到 stack 長了 0x20 的大小(0x20009fd8 - 0x20009fb8)，這 0x20 的空間存的是進入異常函數時 push 進堆保存的暫存器值，此外，可以發現 lr 被設置為 0xfffffff9。

![](https://i.imgur.com/0gy4A97.png)

原先 thread mode 下的暫存器被存在堆裡。(stacking)

![](https://i.imgur.com/kEEsomD.png)

ret 0xfffffff9 會回到 thread mode，並且以 MSP 當作當前 stack pointer。

![](https://i.imgur.com/vUpQUIf.png)

## m4 睡眠模式


**睡眠模式分類**

開發人員可以在適當的時機讓處理器進入睡眠模式，並精準的設置處理器要在什麼樣的事件、中斷發生時被喚醒。

sleep mode : 關閉 processor clock。
deep sleep mode : 關閉 processor clock、system clock、PLL、flash memory。

**進入睡眠模式**

1.WFI 指令 ( wait for interrupt )

如果沒有需要處理的中斷，立刻進入 SCB 配置的睡眠模式。

2.WFE 指令 (wait for event)

執行 WFE 指令時，如果 event register 為 0，就進入睡眠模式 ; 如果為 1，會將 event register 清 0，不會進入睡眠模式。

3.Sleep-on-exit

如果 SCR 暫存器中的 SLEEPONEXIT 設定為 1，那麼在每次處理完所有異常並準備返回線程模式時，都會立刻進入 sleep mode。

這種用法適用於僅在異常發生時執行任務的應用。


**離開睡眠模式**

1.Wakeup from WFI or sleep-on-exit

2.Wakeup from WFE

處理器會在以下幾種情況下 wake up
* 偵測到優先權夠高的異常。
* 偵測到 external event signal。
* 在多處理器下，其它處理器執行了 SEV 指令。

**optional Wakeup Interrupt Controller**


STM32低功耗研究報告
https://kknews.cc/zh-tw/tech/9vbp5jj.html
https://blog.csdn.net/qq_21524907/article/details/59121031?utm_medium=distribute.pc_relevant.none-task-blog-baidujs_title-0&spm=1001.2101.3001.4242




## freertos 任務排程


### rtos 排程演算法 RMS

RMS 排程演算法，採用搶佔式、靜態優先順序的策略，每個週期性任務會分配一個與週期成反比的優先順序，意及週期越短，優先序越高，週期越長，優先序越低。

![](https://i.imgur.com/tW8glbs.png)


### xTaskCreate : 在作業系統創建任務。

追蹤第一次創建任務流程。

tasks.c 中有定義多個與創建 task 相關的函數，而這些函數會依據 configSUPPORT_STATIC_ALLOCATION、configSUPPORT_DYNAMIC_ALLOCATION、portUSING_MPU_WRAPPERS 三個 macros 來決定是否啟用。以我目前使用的 config 檔來說，configSUPPORT_STATIC_ALLOCATION 預設為 0，configSUPPORT_DYNAMIC_ALLOCATION 預設為 1，portUSING_MPU_WRAPPERS 預設為 0，啟用的(會被編譯的)只有 xTaskCreate 一個函數。


1.呼叫 xTaskGenericCreate ( 撰寫程式碼時一般會寫為 xTaskCreate ， xTaskCreate 是一個 MACRO，實際上就是 xTaskGenericCreate)

`#define xTaskCreate( pvTaskCode, pcName, usStackDepth, pvParameters, uxPriority, pxCreatedTask ) xTaskGenericCreate( ( pvTaskCode ), ( pcName ), ( usStackDepth ), ( pvParameters ), ( uxPriority ), ( pxCreatedTask ), ( NULL ), ( NULL ) )`


2.調用 prvAllocateTCBAndStack 分配記憶體給任務的 TCB 和 stack。

`pxNewTCB = prvAllocateTCBAndStack( usStackDepth, puxStackBuffer );`

pvPortMalloc 是 prvAllocateTCBAndStack 中實際請求記憶體的函數，freeRtos 有提供不同的 malloc 機制，在編譯期間可以選擇自己想要使用的機制。

```
xNewTCB = ( tskTCB * ) pvPortMalloc( sizeof( tskTCB ) );
pxNewTCB->pxStack = ( portSTACK_TYPE * ) pvPortMallocAligned( ( ( ( size_t )usStackDepth ) * sizeof( portSTACK_TYPE ) ), puxStackBuffer );
```

3.計算 stack 頂 ( pxTopOfStack )。如果 portSTACK_GROWTH 小於 0 ，代表處理器的 stack 是由高位長到低位，pxTopOfStack 會等於 pxStack 加上需要的 stack 大小 - 1。如果 portSTACK_GROWTH 不為 0，代表處理器的 stack 是由低位長到高位，pxTopOfStack 等於 pxStack。


```
		#if( portSTACK_GROWTH < 0 )
		{
			pxTopOfStack = pxNewTCB->pxStack + ( usStackDepth - ( unsigned short ) 1 );
			pxTopOfStack = ( portSTACK_TYPE * ) ( ( ( portPOINTER_SIZE_TYPE ) pxTopOfStack ) & ( ( portPOINTER_SIZE_TYPE ) ~portBYTE_ALIGNMENT_MASK  ) );

			/* Check the alignment of the calculated top of stack is correct. */
			configASSERT( ( ( ( unsigned long ) pxTopOfStack & ( unsigned long ) portBYTE_ALIGNMENT_MASK ) == 0UL ) );
		}
		#else
		{
			pxTopOfStack = pxNewTCB->pxStack;
			
			/* Check the alignment of the stack buffer is correct. */
			configASSERT( ( ( ( unsigned long ) pxNewTCB->pxStack & ( unsigned long ) portBYTE_ALIGNMENT_MASK ) == 0UL ) );

			/* If we want to use stack checking on architectures that use
			a positive stack growth direction then we also need to store the
			other extreme of the stack space. */
			pxNewTCB->pxEndOfStack = pxNewTCB->pxStack + ( usStackDepth - 1 );
		}

```

4.調用 prvInitialiseTCBVariables 初始化 TCB 結構的 pcTaskName、
uxBasePriority、xGenericListItem、xEventListItem。


`prvInitialiseTCBVariables( pxNewTCB, pcName, uxPriority, xRegions, usStackDepth );`

freeRtos 中的 TCB 結構大概長的像這樣，但實際結構還是要看 os 版本以及 config 檔中的定義。綠色的部份是 xStateItem，這個 itemlist 會根據目前任務狀態，藉由 pxNext 和 pxPrevious 串在不同的 link list 上 ( pxReadyTasksLists、xDelayedTaskList、xPendingReadyList、xTasksWaitingTermination、xSuspendedTaskList )。pvOwner 指向 TCB，pvConainter 指向目前存在的 link list 的管理結構 ( xList )。 

![](https://i.imgur.com/ExaoflA.png)



5.調用 pxPortInitialiseStack 初始化 stack。這部份是與硬體相關的程式碼，在不同架構下使用的函數不一樣，以 m4 來說，這個函數被定義在 FreeRtos/portable/GCC/ARM_CM4F/port.c 檔案裡。初始化後的 stack 會讓任務看起來好像已經執行過，只是被排程器中斷而已。

`xNewTCB->pxTopOfStack = pxPortInitialiseStack( pxTopOfStack, pxTaskCode, pvParameters );`



![](https://i.imgur.com/rt6RN8B.png)
```
// FreeRtos/portable/GCC/ARM_CM4F/port.c
portSTACK_TYPE *pxPortInitialiseStack( portSTACK_TYPE *pxTopOfStack, pdTASK_CODE pxCode, void *pvParameters )
{
	/* Simulate the stack frame as it would be created by a context switch
	interrupt. */
	
	/* Offset added to account for the way the MCU uses the stack on entry/exit
	of interrupts, and to ensure alignment. */
	pxTopOfStack--;
		
	*pxTopOfStack = portINITIAL_XPSR;	/* xPSR */
	pxTopOfStack--;
	*pxTopOfStack = ( portSTACK_TYPE ) pxCode;	/* PC */
	pxTopOfStack--;
	*pxTopOfStack = 0;	/* LR */
	
	/* Save code space by skipping register initialisation. */
	pxTopOfStack -= 5;	/* R12, R3, R2 and R1. */
	*pxTopOfStack = ( portSTACK_TYPE ) pvParameters;	/* R0 */


	/* A save method is being used that requires each task to maintain its
	own exec return value. */
	pxTopOfStack--;
	*pxTopOfStack = portINITIAL_EXEC_RETURN;

	pxTopOfStack -= 8;	/* R11, R10, R9, R8, R7, R6, R5 and R4. */
	
	return pxTopOfStack;
}
```

6.因為接下來因為會操作到全域的 link list，所以要先進入 CS。

`taskENTER_CRITICAL();` 

7.如果 pxCurrentTCB 是 NULL，代表目前沒有其他的任務，或者所有的任務都處於 suspeneded 狀態，將 pxCurrentTCB 設定為該任務，又如果 uxCurrentNumberOfTasks 為 1，代表這個任務是第一個創建的任務，需要調用 prvInitialiseTaskLists 初始化一些排程需要用到的 link list 結構。

在 pxCurrentTCB 不是 NULL 的狀況下，如果 xSchedulerRunning == pdFALSE ( 排程器還沒有啟動 )，要選擇優先權高的當作 pxCurrentTCB。 


```
			uxCurrentNumberOfTasks++;
			if( pxCurrentTCB == NULL )
			{
				/* There are no other tasks, or all the other tasks are in
				the suspended state - make this the current task. */
				pxCurrentTCB =  pxNewTCB;

				if( uxCurrentNumberOfTasks == ( unsigned portBASE_TYPE ) 1 )
				{
					/* This is the first task to be created so do the preliminary
					initialisation required.  We will not recover if this call
					fails, but we will report the failure. */
					prvInitialiseTaskLists();
				}
			}
			else
			{
				/* If the scheduler is not already running, make this task the
				current task if it is the highest priority task to be created
				so far. */
				if( xSchedulerRunning == pdFALSE )
				{
					if( pxCurrentTCB->uxPriority <= uxPriority )
					{
						pxCurrentTCB = pxNewTCB;
					}
				}
			}
```


prvInitialiseTaskLists 除了為每個優先權都初始化了一條 link list 以外，還初始化了 xDelayedTaskList1、xDelayedTaskList2、xPendingReadyList、xTasksWaitingTermination、xSuspendedTaskList 。
```
static void prvInitialiseTaskLists( void )
{
unsigned portBASE_TYPE uxPriority;

	for( uxPriority = ( unsigned portBASE_TYPE ) 0U; uxPriority < configMAX_PRIORITIES; uxPriority++ )
	{
		vListInitialise( ( xList * ) &( pxReadyTasksLists[ uxPriority ] ) );
	}

	vListInitialise( ( xList * ) &xDelayedTaskList1 );
	vListInitialise( ( xList * ) &xDelayedTaskList2 );
	vListInitialise( ( xList * ) &xPendingReadyList );

	#if ( INCLUDE_vTaskDelete == 1 )
	{
		vListInitialise( ( xList * ) &xTasksWaitingTermination );
	}
	#endif

	#if ( INCLUDE_vTaskSuspend == 1 )
	{
		vListInitialise( ( xList * ) &xSuspendedTaskList );
	}
	#endif

	/* Start with pxDelayedTaskList using list1 and the pxOverflowDelayedTaskList
	using list2. */
	pxDelayedTaskList = &xDelayedTaskList1;
	pxOverflowDelayedTaskList = &xDelayedTaskList2;
}
```

link list 的初始化過程如下。xList 是一個用來紀錄 link list 的結構塊，其中的 xListEnd 是一個 xMiniListitem 的結構，用來表示 link list 的最後一個 listitem，這個 item 會永遠在 link list 的尾部。

![](https://i.imgur.com/V6cbqPN.png)


8.紀錄目前最高的優先權，可以加快 context switch 的速度。

```
			if( pxNewTCB->uxPriority > uxTopUsedPriority )
			{
				uxTopUsedPriority = pxNewTCB->uxPriority;
			}
```

9.將任務加入 readyQueeue 當中。

`prvAddTaskToReadyQueue( pxNewTCB );`


vListInsertEnd 執行一個雙向鏈表插入操作，其中 xList->pxIndex 是雙向鏈表的頭，每次插入新的 xListItem 都必須更新 xList->pxIndex。下圖是依序將 task1 和 task2 鏈入 link list 的狀況。


![](https://i.imgur.com/NrP51Fm.png)


```
void vListInsertEnd( xList *pxList, xListItem *pxNewListItem )
{
volatile xListItem * pxIndex;

	/* Insert a new list item into pxList, but rather than sort the list,
	makes the new list item the last item to be removed by a call to
	pvListGetOwnerOfNextEntry.  This means it has to be the item pointed to by
	the pxIndex member. */
	pxIndex = pxList->pxIndex;

	pxNewListItem->pxNext = pxIndex->pxNext;
	pxNewListItem->pxPrevious = pxList->pxIndex;
	pxIndex->pxNext->pxPrevious = ( volatile xListItem * ) pxNewListItem;
	pxIndex->pxNext = ( volatile xListItem * ) pxNewListItem;
	pxList->pxIndex = ( volatile xListItem * ) pxNewListItem;

	/* Remember which list the item is in. */
	pxNewListItem->pvContainer = ( void * ) pxList;

	( pxList->uxNumberOfItems )++;
}
```


### vTaskStartScheduler : 啟動作業系統排程器

追蹤 vTaskStartScheduler。

1.創建 idle task 讓作業系統中至少存在一個任務，這個任務優先權是最小可能優先權，確保在有其他任務的狀況下，不會使用到任何 cpu time。

https://www.freertos.org/RTOS-idle-task.html
`xReturn = xTaskCreate( prvIdleTask, ( signed char * ) "IDLE", tskIDLE_STACK_SIZE, ( void * ) NULL, ( tskIDLE_PRIORITY | portPRIVILEGE_BIT ), &xIdleTaskHandle );`

The idle task is responsible for freeing memory allocated by the RTOS to tasks that have since been deleted. It is therefore important in applications that make use of the vTaskDelete() function to ensure the idle task is not starved of processing time. The idle task has no other active functions so can legitimately be starved of microcontroller time under all other conditions.


2.調用 xPortStartScheduler 設定計時器與開始執行第一個任務。

Systick 是 m4 內部的計時器，被放置在 NVIC 內。xPortStartScheduler 因為需要調整 Systick，所以屬於硬體相關程式碼。以 m4 為例，該函數被定義在 FreeRtos/portable/GCC/ARM_CM4F/port.c。

```
portBASE_TYPE xPortStartScheduler( void )
{
	/* Make PendSV and SysTick the lowest priority interrupts. */
	*(portNVIC_SYSPRI2) |= portNVIC_PENDSV_PRI;
	*(portNVIC_SYSPRI2) |= portNVIC_SYSTICK_PRI;

	/* Start the timer that generates the tick ISR.  Interrupts are disabled
	here already. */
	prvSetupTimerInterrupt();

	/* Initialise the critical nesting count ready for the first task. */
	uxCriticalNesting = 0;

	/* Ensure the VFP is enabled - it should be anyway. */
	vPortEnableVFP();

	/* Lazy save always. */
	*( portFPCCR ) |= portASPEN_AND_LSPEN_BITS;

	/* Start the first task. */
	vPortStartFirstTask();

	/* Should not get here! */
	return 0;
}
/*-----------------------------------------------------------*/
```


portNVIC_SYSPRI2 是下面圖表中的暫存器，17~24 bits 用來設定 pendSV 的優先權，25~32 bits 用來設定 systick exception 的優先權。
```
	*(portNVIC_SYSPRI2) |= portNVIC_PENDSV_PRI;
	*(portNVIC_SYSPRI2) |= portNVIC_SYSTICK_PRI;
```


![](https://i.imgur.com/y1JLoMS.png)



這裡的 portNVIC_SYSTICK_LOAD 就是下面圖表的 SYST_RVR 暫存器，計數器會從該值開始倒數，倒數至 0 時，再重新裝載回該值。portNVIC_SYSTICK_CTRL 則是下面圖表的 SYST_CSR 暫存器， `*(portNVIC_SYSTICK_CTRL) = portNVIC_SYSTICK_CLK | portNVIC_SYSTICK_INT | portNVIC_SYSTICK_ENABLE;`用來設定 clock 的時鐘源、倒數到 0 時產生異常，以及啟用時鐘。


```
#define portNVIC_SYSTICK_CLK		0x00000004
#define portNVIC_SYSTICK_INT		0x00000002
#define portNVIC_SYSTICK_ENABLE		0x00000001
```


```
void prvSetupTimerInterrupt( void )
{
	/* Configure SysTick to interrupt at the requested rate. */
	*(portNVIC_SYSTICK_LOAD) = ( configCPU_CLOCK_HZ / configTICK_RATE_HZ ) - 1UL;
	*(portNVIC_SYSTICK_CTRL) = portNVIC_SYSTICK_CLK | portNVIC_SYSTICK_INT | portNVIC_SYSTICK_ENABLE;
}
```

![](https://i.imgur.com/2wawZT2.png)

![](https://i.imgur.com/MTh5WQB.png)

![](https://i.imgur.com/2KvkKEg.png)



```
#define portSET_INTERRUPT_MASK()				__asm volatile 	( " cpsid i " )
#define portCLEAR_INTERRUPT_MASK()				__asm volatile 	( " cpsie i " )
```

先讓 MSP 還原回最初始的位置，也就是 vector table 上 offset 等於 0 的位置，接著執行 cpsie i 啟用中斷並執行 svc 0 跳至 vPortSVCHandler。

```
static void vPortStartFirstTask( void )
{
	__asm volatile(
					" ldr r0, =0xE000ED08 	\n" /* Use the NVIC offset register to locate the stack. */
					" ldr r0, [r0] 			\n"
					" ldr r0, [r0] 			\n"
					" msr msp, r0			\n" /* Set the msp back to the start of the stack. */
					" cpsie i				\n" /* Globally enable interrupts. */
					" svc 0					\n" /* System call to start first task. */
					" nop					\n"
				);
}
```
將 r4-r11 pop 到暫存器，在執行 bx 0xfffffffd，利用 psp unstacking，pop r0~r3 以及 lr 和 pc。

```
void vPortSVCHandler( void )
{
	__asm volatile (
					"	ldr	r3, pxCurrentTCBConst2		\n" /* Restore the context. */
					"	ldr r1, [r3]					\n" /* Use pxCurrentTCBConst to get the pxCurrentTCB address. */
					"	ldr r0, [r1]					\n" /* The first item in pxCurrentTCB is the task top of stack. */
					"	ldmia r0!, {r4-r11, r14}		\n" /* Pop the registers that are not automatically saved on exception entry and the critical nesting count. */
					"	msr psp, r0						\n" /* Restore the task stack pointer. */
					"	mov r0, #0 						\n"
					"	msr	basepri, r0					\n"
					"	bx r14							\n"
					"									\n"
					"	.align 2						\n"
					"pxCurrentTCBConst2: .word pxCurrentTCB				\n"
				);
}
```

![](https://i.imgur.com/dLX53JV.png)


### Systick 中斷處理

Systick 歸 0 時會觸發 systick 中斷，調用 xTaskIncrementTick 遞增 xTickCount 及檢查是否有任務因為等待時間到了而被叫醒，如果有任務被叫醒的話，就將任務從原本的 xStateListItem 中解鏈，並加入 ReadyList 鏈表當中，此外，如果有 context switch 需求的話 ( ReadyList 出現更高優先權的任務 )，會執行 portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT，嘗試觸發 penSV 中斷。

:::info
為了防止 ISR 執行時期 ( handler mode ) 被 context switch 到 thread mode，freeRtos 中並沒有在 xPortSysTickHandler 中直接執行 context switch，取而代之的是實行在中斷優先權最低的 xPortPendSVHandler。
https://www.mdeditor.tw/pl/p2UY/zh-tw 這篇文章有詳述為什麼 context switch 會實行在 portSV 異常而不是 systick 異常。
:::

```
void xPortSysTickHandler( void )
{
	/* The SysTick runs at the lowest interrupt priority, so when this interrupt
	executes all interrupts must be unmasked.  There is therefore no need to
	save and then restore the interrupt mask value as its value is already
	known. */
	portDISABLE_INTERRUPTS();
	{
		/* Increment the RTOS tick. */
		if( xTaskIncrementTick() != pdFALSE )
		{
			/* A context switch is required.  Context switching is performed in
			the PendSV interrupt.  Pend the PendSV interrupt. */
			portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;
		}
	}
	portENABLE_INTERRUPTS();
}
```

### context switch

PendSV_Handler 呼叫 vTaskSwitchContext，將 pxCurrentTCB 指定為目前 ready list 中優先權最高的 task tcb。返回 PendSV_Handler 後從 pxCurrentTCB 取出 stack 位置，做 unstacking 讓 pc 等於新任務的執行位置。

http://fastbitlab.com/free-rtos/

https://freertos.blog.csdn.net/article/details/51418383?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-16.control&dist_request_id=1328627.13073.16153929756225199&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-16.control

```
void xPortPendSVHandler( void )
{
	/* This is a naked function. */

	__asm volatile
	(
	"	mrs r0, psp							\n"
	"	isb									\n"
	"										\n"
	"	ldr	r3, pxCurrentTCBConst			\n" /* Get the location of the current TCB. */
	"	ldr	r2, [r3]						\n"
	"										\n"
	"	tst r14, #0x10						\n" /* Is the task using the FPU context?  If so, push high vfp registers. */
	"	it eq								\n"
	"	vstmdbeq r0!, {s16-s31}				\n"
	"										\n"
	"	stmdb r0!, {r4-r11, r14}			\n" /* Save the core registers. */
	"										\n"
	"	str r0, [r2]						\n" /* Save the new top of stack into the first member of the TCB. */
	"										\n"
	"	stmdb sp!, {r3}						\n"
	"	mov r0, %0 							\n"
	"	msr basepri, r0						\n"
	"	dsb									\n"
	"	isb									\n"
	"	bl vTaskSwitchContext				\n"
	"	mov r0, #0							\n"
	"	msr basepri, r0						\n"
	"	ldmia sp!, {r3}						\n"
	"										\n"
	"	ldr r1, [r3]						\n" /* The first item in pxCurrentTCB is the task top of stack. */
	"	ldr r0, [r1]						\n"
	"										\n"
	"	ldmia r0!, {r4-r11, r14}			\n" /* Pop the core registers. */
	"										\n"
	"	tst r14, #0x10						\n" /* Is the task using the FPU context?  If so, pop the high vfp registers too. */
	"	it eq								\n"
	"	vldmiaeq r0!, {s16-s31}				\n"
	"										\n"
	"	msr psp, r0							\n"
	"	isb									\n"
	"										\n"
	#ifdef WORKAROUND_PMU_CM001 /* XMC4000 specific errata workaround. */
		#if WORKAROUND_PMU_CM001 == 1
	"			push { r14 }				\n"
	"			pop { pc }					\n"
		#endif
	#endif
	"										\n"
	"	bx r14								\n"
	"										\n"
	"	.align 4							\n"
	"pxCurrentTCBConst: .word pxCurrentTCB	\n"
	::"i"(configMAX_SYSCALL_INTERRUPT_PRIORITY)
	);
}
```




```
void vTaskSwitchContext( void )
{
	if( uxSchedulerSuspended != ( UBaseType_t ) pdFALSE )
	{
		/* The scheduler is currently suspended - do not allow a context
		switch. */
		xYieldPending = pdTRUE;
	}
	else
	{
		xYieldPending = pdFALSE;
		traceTASK_SWITCHED_OUT();

		#if ( configGENERATE_RUN_TIME_STATS == 1 )
		{
				#ifdef portALT_GET_RUN_TIME_COUNTER_VALUE
					portALT_GET_RUN_TIME_COUNTER_VALUE( ulTotalRunTime );
				#else
					ulTotalRunTime = portGET_RUN_TIME_COUNTER_VALUE();
				#endif

				/* Add the amount of time the task has been running to the
				accumulated time so far.  The time the task started running was
				stored in ulTaskSwitchedInTime.  Note that there is no overflow
				protection here so count values are only valid until the timer
				overflows.  The guard against negative values is to protect
				against suspect run time stat counter implementations - which
				are provided by the application, not the kernel. */
				if( ulTotalRunTime > ulTaskSwitchedInTime )
				{
					pxCurrentTCB->ulRunTimeCounter += ( ulTotalRunTime - ulTaskSwitchedInTime );
				}
				else
				{
					mtCOVERAGE_TEST_MARKER();
				}
				ulTaskSwitchedInTime = ulTotalRunTime;
		}
		#endif /* configGENERATE_RUN_TIME_STATS */

		/* Check for stack overflow, if configured. */
		taskCHECK_FOR_STACK_OVERFLOW();

		/* Select a new task to run using either the generic C or port
		optimised asm code. */
		taskSELECT_HIGHEST_PRIORITY_TASK();
		traceTASK_SWITCHED_IN();

		#if ( configUSE_NEWLIB_REENTRANT == 1 )
		{
			/* Switch Newlib's _impure_ptr variable to point to the _reent
			structure specific to this task. */
			_impure_ptr = &( pxCurrentTCB->xNewLib_reent );
		}
		#endif /* configUSE_NEWLIB_REENTRANT */
	}
}
```


taskSELECT_HIGHEST_PRIORITY_TASK MACRO 讓 pxCurrentTCB 等於目前優先權最高的 task。

```
	#define taskSELECT_HIGHEST_PRIORITY_TASK()														\
	{																								\
	UBaseType_t uxTopPriority;																		\
																									\
		/* Find the highest priority list that contains ready tasks. */								\
		portGET_HIGHEST_PRIORITY( uxTopPriority, uxTopReadyPriority );								\
		configASSERT( listCURRENT_LIST_LENGTH( &( pxReadyTasksLists[ uxTopPriority ] ) ) > 0 );		\
		listGET_OWNER_OF_NEXT_ENTRY( pxCurrentTCB, &( pxReadyTasksLists[ uxTopPriority ] ) );		\
	} /* taskSELECT_HIGHEST_PRIORITY_TASK() */
```

### vTaskDelay : 讓任務暫時進入 blocking 狀態。

vTaskDelay 將任務串入 delay list 當中，並呼叫 portYIELD_WITHIN_API 執行 context switch。

傳入參數 xTicksToDelay 相當於延遲的 systick 中斷次數，如果想要讓任務延遲 n ms，需要傳入 n/portTICK_RATE_MS。

configTICK_RATE_HZ : systick 中斷頻率。
portTICK_PERIOD_MS/portTICK_RATE_MS : systick 中斷週期(毫秒)。
vTaskDelay(1000 / portTICK_PERIOD_MS) ＝ (1000) / ( ( TickType_t ) 1000 / configTICK_RATE_HZ )

舉例來說，如果要 delay 500ms，需要的 systick 中斷數會是 500 乘上每毫秒產生的中斷次數，也就是 500 * (1 / portTICK_RATE_MS)。

```
	void vTaskDelay( const TickType_t xTicksToDelay )
	{
	BaseType_t xAlreadyYielded = pdFALSE;

		/* A delay time of zero just forces a reschedule. */
		if( xTicksToDelay > ( TickType_t ) 0U )
		{
			configASSERT( uxSchedulerSuspended == 0 );
			vTaskSuspendAll();
			{
				traceTASK_DELAY();

				/* A task that is removed from the event list while the
				scheduler is suspended will not get placed in the ready
				list or removed from the blocked list until the scheduler
				is resumed.

				This task cannot be in an event list as it is the currently
				executing task. */
				prvAddCurrentTaskToDelayedList( xTicksToDelay, pdFALSE );
			}
			xAlreadyYielded = xTaskResumeAll();
		}
		else
		{
			mtCOVERAGE_TEST_MARKER();
		}

		/* Force a reschedule if xTaskResumeAll has not already done so, we may
		have put ourselves to sleep. */
		if( xAlreadyYielded == pdFALSE )
		{
			portYIELD_WITHIN_API();
		}
		else
		{
			mtCOVERAGE_TEST_MARKER();
		}
	}
```


prvAddCurrentTaskToDelayedList 調用 uxListRemove( &( pxCurrentTCB->xStateListItem ) 將目前任務從 ready list 中解鏈，並將任務優先權指定為 uxTopReadyPriority。再來，如果 xTicksToWait 等於 portMAX_DELAY，會直接將任務鏈入 xSuspendedTaskList，並結束函數，否則，會先計算任務應該在 tickcount 為多少時醒來 ( xTimeToWake = xConstTickCount + xTicksToWait ) ，如果 xTimeToWake 發生 overflow，也就是 xTimeToWake < xConstTickCount，就調用 vListInsert 將任務鏈入 pxOverflowDelayedTaskList，沒有則鏈入 pxDelayedTaskList。

```

static void prvAddCurrentTaskToDelayedList( TickType_t xTicksToWait, const BaseType_t xCanBlockIndefinitely )
{
TickType_t xTimeToWake;
const TickType_t xConstTickCount = xTickCount;


	/* Remove the task from the ready list before adding it to the blocked list
	as the same list item is used for both lists. */
	if( uxListRemove( &( pxCurrentTCB->xStateListItem ) ) == ( UBaseType_t ) 0 )
	{
		/* The current task must be in a ready list, so there is no need to
		check, and the port reset macro can be called directly. */
		portRESET_READY_PRIORITY( pxCurrentTCB->uxPriority, uxTopReadyPriority );
	}
	else
	{
		mtCOVERAGE_TEST_MARKER();
	}

	#if ( INCLUDE_vTaskSuspend == 1 )
	{
		if( ( xTicksToWait == portMAX_DELAY ) && ( xCanBlockIndefinitely != pdFALSE ) )
		{
			/* Add the task to the suspended task list instead of a delayed task
			list to ensure it is not woken by a timing event.  It will block
			indefinitely. */
			vListInsertEnd( &xSuspendedTaskList, &( pxCurrentTCB->xStateListItem ) );
		}
		else
		{
			/* Calculate the time at which the task should be woken if the event
			does not occur.  This may overflow but this doesn't matter, the
			kernel will manage it correctly. */
			xTimeToWake = xConstTickCount + xTicksToWait;

			/* The list item will be inserted in wake time order. */
			listSET_LIST_ITEM_VALUE( &( pxCurrentTCB->xStateListItem ), xTimeToWake );

			if( xTimeToWake < xConstTickCount )
			{
				/* Wake time has overflowed.  Place this item in the overflow
				list. */
				vListInsert( pxOverflowDelayedTaskList, &( pxCurrentTCB->xStateListItem ) );
			}
			else
			{
				/* The wake time has not overflowed, so the current block list
				is used. */
				vListInsert( pxDelayedTaskList, &( pxCurrentTCB->xStateListItem ) );

				/* If the task entering the blocked state was placed at the
				head of the list of blocked tasks then xNextTaskUnblockTime
				needs to be updated too. */
				if( xTimeToWake < xNextTaskUnblockTime )
				{
					xNextTaskUnblockTime = xTimeToWake;
				}
				else
				{
					mtCOVERAGE_TEST_MARKER();
				}
			}
		}
	}
	#else /* INCLUDE_vTaskSuspend */
	{
		/* Calculate the time at which the task should be woken if the event
		does not occur.  This may overflow but this doesn't matter, the kernel
		will manage it correctly. */
		xTimeToWake = xConstTickCount + xTicksToWait;

		/* The list item will be inserted in wake time order. */
		listSET_LIST_ITEM_VALUE( &( pxCurrentTCB->xStateListItem ), xTimeToWake );

		if( xTimeToWake < xConstTickCount )
		{
			/* Wake time has overflowed.  Place this item in the overflow list. */
			vListInsert( pxOverflowDelayedTaskList, &( pxCurrentTCB->xStateListItem ) );
		}
		else
		{
			/* The wake time has not overflowed, so the current block list is used. */
			vListInsert( pxDelayedTaskList, &( pxCurrentTCB->xStateListItem ) );

			/* If the task entering the blocked state was placed at the head of the
			list of blocked tasks then xNextTaskUnblockTime needs to be updated
			too. */
			if( xTimeToWake < xNextTaskUnblockTime )
			{
				xNextTaskUnblockTime = xTimeToWake;
			}
			else
			{
				mtCOVERAGE_TEST_MARKER();
			}
		}

		/* Avoid compiler warning when INCLUDE_vTaskSuspend is not 1. */
		( void ) xCanBlockIndefinitely;
	}
	#endif /* INCLUDE_vTaskSuspend */
}

```

vListInsert 在做插入時會維持 delaylist 上 itemvalue 由小排至大的順序。

```
void vListInsert( List_t * const pxList, ListItem_t * const pxNewListItem )
{
ListItem_t *pxIterator;
const TickType_t xValueOfInsertion = pxNewListItem->xItemValue;

	/* Only effective when configASSERT() is also defined, these tests may catch
	the list data structures being overwritten in memory.  They will not catch
	data errors caused by incorrect configuration or use of FreeRTOS. */
	listTEST_LIST_INTEGRITY( pxList );
	listTEST_LIST_ITEM_INTEGRITY( pxNewListItem );

	/* Insert the new list item into the list, sorted in xItemValue order.

	If the list already contains a list item with the same item value then the
	new list item should be placed after it.  This ensures that TCB's which are
	stored in ready lists (all of which have the same xItemValue value) get a
	share of the CPU.  However, if the xItemValue is the same as the back marker
	the iteration loop below will not end.  Therefore the value is checked
	first, and the algorithm slightly modified if necessary. */
	if( xValueOfInsertion == portMAX_DELAY )
	{
		pxIterator = pxList->xListEnd.pxPrevious;
	}
	else
	{
		/* *** NOTE ***********************************************************
		If you find your application is crashing here then likely causes are
		listed below.  In addition see http://www.freertos.org/FAQHelp.html for
		more tips, and ensure configASSERT() is defined!
		http://www.freertos.org/a00110.html#configASSERT

			1) Stack overflow -
			   see http://www.freertos.org/Stacks-and-stack-overflow-checking.html
			2) Incorrect interrupt priority assignment, especially on Cortex-M
			   parts where numerically high priority values denote low actual
			   interrupt priorities, which can seem counter intuitive.  See
			   http://www.freertos.org/RTOS-Cortex-M3-M4.html and the definition
			   of configMAX_SYSCALL_INTERRUPT_PRIORITY on
			   http://www.freertos.org/a00110.html
			3) Calling an API function from within a critical section or when
			   the scheduler is suspended, or calling an API function that does
			   not end in "FromISR" from an interrupt.
			4) Using a queue or semaphore before it has been initialised or
			   before the scheduler has been started (are interrupts firing
			   before vTaskStartScheduler() has been called?).
		**********************************************************************/

		for( pxIterator = ( ListItem_t * ) &( pxList->xListEnd ); pxIterator->pxNext->xItemValue <= xValueOfInsertion; pxIterator = pxIterator->pxNext ) /*lint !e826 !e740 The mini list structure is used as the list end to save RAM.  This is checked and valid. */
		{
			/* There is nothing to do here, just iterating to the wanted
			insertion position. */
		}
	}

	pxNewListItem->pxNext = pxIterator->pxNext;
	pxNewListItem->pxNext->pxPrevious = pxNewListItem;
	pxNewListItem->pxPrevious = pxIterator;
	pxIterator->pxNext = pxNewListItem;

	/* Remember which list the item is in.  This allows fast removal of the
	item later. */
	pxNewListItem->pvContainer = ( void * ) pxList;

	( pxList->uxNumberOfItems )++;
}
```


`portYIELD_WITHIN_API();`




## 週期性任務

Freertos 中可以控制任務週期的函數有 vtaskDelay 以及 vTaskDelayUntil，分別為相對延遲函數以及絕對延遲函數。

相對延遲函數 vtaskDelay 計算下次啟用時間的方法是將目前時間 ( 調用 vtaskDelay時間 ) 加上傳入參數的延遲時間。

xTimeToWake = xConstTickCount + xTicksToWait

絕對延遲函數 vTaskDelayUntil 計算下次啟用的時間的方法是將上次啟用時間加上傳入參數的延遲時間。

xTimeToWake = *pxPreviousWakeTime + xTimeIncrement
*pxPreviousWakeTime = xTimeToWake;

從 xTimeToWake 的計算方式可以看出，如果要比較精準的控制函數週期，需要使用 vTaskDelayUntil。

### RMS 演算法 ( 經典的週期性任務調度算法 )

RMS 算法是單處理器下最佳的靜態優先權調度演算法，也就是說如果一個任務集沒辦法被這個算法調度，那麼這個任務集也沒辦法給任何靜態優先權演算法調度。

主要概念是給週期高的任務大優先權，週期小的任務低優先權。

https://baike.baidu.com/item/RMS/7521847

## 作業系統效能評測

#### 實驗環境

![](https://i.imgur.com/guy0XZM.png)

1.core clk 約 156 MHz，系統時鐘鐘源為 core clk，每個 clk 上升時系統時鐘加 1，一秒遞加 156M 次
2.設定每秒產生 1000 個 systick 中斷，以每秒系統時鐘遞加 156M 次來說，計數器值應該設定為 156M / 1000，計數器每次歸 0 產生一個中斷。


#### 實驗方法
freertos 中提供了一個能測量每個任務在 cpu 中使用率的函數 vTaskGetRunTimeStats 。

在調用函數前必須設置一個頻率夠高的計時器，函數的原理是藉由計算每個任務被該計時器中斷的次數，來猜測其可能的 CPU 佔用時間。

> void vTaskGetRunTimeStats( char *pcWriteBuffer );
> 	
> See the Run Time Stats page for a full description of this feature.
> 
> configGENERATE_RUN_TIME_STATS, configUSE_STATS_FORMATTING_FUNCTIONS and configSUPPORT_DYNAMIC_ALLOCATION must all be defined as 1 for this function to be available. The application must also then provide definitions for portCONFIGURE_TIMER_FOR_RUN_TIME_STATS() and portGET_RUN_TIME_COUNTER_VALUE to configure a peripheral timer/counter and return the timer’s current count value respectively. The counter should be at least 10 times the frequency of the tick count.
> https://www.freertos.org/a00021.html#vTaskGetRunTimeStats
> 


### 計算 context switch 花費時間 (未完成)
### 計算 interrupt 延遲 (未完成)
### 計算 CPU 使用率 (未完成)


## Debug

st-util &
arm-none-eabi-gdb ./xxx.elf
tar ext :<port>

