


## Task Control Block (TCB) (370 to 453 lines)

```C
/*

* Task control block. A task control block (TCB) is allocated for each task,

* and stores task state information, including a pointer to the task's context

* (the task's run time environment, including register values)

*/

typedef struct tskTaskControlBlock /* The old naming convention is used to prevent breaking kernel aware debuggers. */

{

volatile StackType_t * pxTopOfStack; /**< Points to the location of the last item placed on the tasks stack. THIS MUST BE THE FIRST MEMBER OF THE TCB STRUCT. */

  

#if ( portUSING_MPU_WRAPPERS == 1 )

xMPU_SETTINGS xMPUSettings; /**< The MPU settings are defined as part of the port layer. THIS MUST BE THE SECOND MEMBER OF THE TCB STRUCT. */

#endif

  

#if ( configUSE_CORE_AFFINITY == 1 ) && ( configNUMBER_OF_CORES > 1 )

UBaseType_t uxCoreAffinityMask; /**< Used to link the task to certain cores. UBaseType_t must have greater than or equal to the number of bits as configNUMBER_OF_CORES. */

#endif

  

ListItem_t xStateListItem; /**< The list that the state list item of a task is reference from denotes the state of that task (Ready, Blocked, Suspended ). */

ListItem_t xEventListItem; /**< Used to reference a task from an event list. */

UBaseType_t uxPriority; /**< The priority of the task. 0 is the lowest priority. */

StackType_t * pxStack; /**< Points to the start of the stack. */

#if ( configNUMBER_OF_CORES > 1 )

volatile BaseType_t xTaskRunState; /**< Used to identify the core the task is running on, if the task is running. Otherwise, identifies the task's state - not running or yielding. */

UBaseType_t uxTaskAttributes; /**< Task's attributes - currently used to identify the idle tasks. */

#endif

char pcTaskName[ configMAX_TASK_NAME_LEN ]; /**< Descriptive name given to the task when created. Facilitates debugging only. */

  

#if ( configUSE_TASK_PREEMPTION_DISABLE == 1 )

BaseType_t xPreemptionDisable; /**< Used to prevent the task from being preempted. */

#endif

  

#if ( ( portSTACK_GROWTH > 0 ) || ( configRECORD_STACK_HIGH_ADDRESS == 1 ) )

StackType_t * pxEndOfStack; /**< Points to the highest valid address for the stack. */

#endif

  

#if ( portCRITICAL_NESTING_IN_TCB == 1 )

UBaseType_t uxCriticalNesting; /**< Holds the critical section nesting depth for ports that do not maintain their own count in the port layer. */

#endif

  

#if ( configUSE_TRACE_FACILITY == 1 )

UBaseType_t uxTCBNumber; /**< Stores a number that increments each time a TCB is created. It allows debuggers to determine when a task has been deleted and then recreated. */

UBaseType_t uxTaskNumber; /**< Stores a number specifically for use by third party trace code. */

#endif

  

#if ( configUSE_MUTEXES == 1 )

UBaseType_t uxBasePriority; /**< The priority last assigned to the task - used by the priority inheritance mechanism. */

UBaseType_t uxMutexesHeld;

#endif

  

#if ( configUSE_APPLICATION_TASK_TAG == 1 )

TaskHookFunction_t pxTaskTag;

#endif

  

#if ( configNUM_THREAD_LOCAL_STORAGE_POINTERS > 0 )

void * pvThreadLocalStoragePointers[ configNUM_THREAD_LOCAL_STORAGE_POINTERS ];

#endif

  

#if ( configGENERATE_RUN_TIME_STATS == 1 )

configRUN_TIME_COUNTER_TYPE ulRunTimeCounter; /**< Stores the amount of time the task has spent in the Running state. */

#endif

  

#if ( configUSE_C_RUNTIME_TLS_SUPPORT == 1 )

configTLS_BLOCK_TYPE xTLSBlock; /**< Memory block used as Thread Local Storage (TLS) Block for the task. */

#endif

  

#if ( configUSE_TASK_NOTIFICATIONS == 1 )

volatile uint32_t ulNotifiedValue[ configTASK_NOTIFICATION_ARRAY_ENTRIES ];

volatile uint8_t ucNotifyState[ configTASK_NOTIFICATION_ARRAY_ENTRIES ];

#endif

  

/* See the comments in FreeRTOS.h with the definition of

* tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE. */

#if ( tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE != 0 )

uint8_t ucStaticallyAllocated; /**< Set to pdTRUE if the task is a statically allocated to ensure no attempt is made to free the memory. */

#endif

  

#if ( INCLUDE_xTaskAbortDelay == 1 )

uint8_t ucDelayAborted;

#endif

  

#if ( configUSE_POSIX_ERRNO == 1 )

int iTaskErrno;

#endif

} tskTCB;
```

在嵌入式系統中，CPU 核心只有一兩個，但可能同時有十幾個任務在輪流執行。每當 FreeRTOS 要換人執行時，它必須知道「上一個任務進行到哪裡了」以及「下一個任務的狀態是什麼」。**TCB 就是保存這些關鍵資訊的記憶體結構。**

### 1. 棧（Stack）管理指針
這部分是 TCB 中最關鍵的成員，決定了任務切換時能不能正確恢復現場
```C
volatile StackType_t * pxTopOfStack; /**< THIS MUST BE THE FIRST MEMBER OF THE TCB STRUCT. */
StackType_t * pxStack;                /**< Points to the start of the stack. */
#if ( ( portSTACK_GROWTH > 0 ) || ( configRECORD_STACK_HIGH_ADDRESS == 1 ) )
    StackType_t * pxEndOfStack;       /**< Points to the highest valid address for the stack. */
#endif
```
- **`pxTopOfStack`（棧頂指針）**：**這必須是 TCB 的第一個成員！** 當任務被暫停時，CPU 暫存器的所有資料（現場畫面）都會被推入該任務的棧中，這個指針就記錄了最後推入的位置。下次任務復活時，FreeRTOS 會直接讀取 TCB 偏移量為 0 的位置（也就是它），快速把暫存器恢復。
- **`pxStack`（棧底指針）**：指向這塊棧記憶體的起始起點，主要用來釋放記憶體或檢查是否越界。
- **`pxEndOfStack`（棧頂邊界）**：用來檢測**棧溢出（Stack Overflow）**。

### 2. 狀態與鏈結模組：任務在哪個隊列？
FreeRTOS 管理任務不是一個個去數，而是透過鏈表（List）把相同狀態的任務串在一起。
```C
ListItem_t xStateListItem;  /**< Ready, Blocked, Suspended */
ListItem_t xEventListItem;  /**< Used to reference a task from an event list. */
UBaseType_t uxPriority;     /**< The priority of the task. 0 is the lowest. */
```
- **`xStateListItem`（狀態鏈表節點）**：如果這個任務在排隊等待執行，它就會被掛到 `pxReadyTasksLists`（就緒鏈表）中；如果在延時（VTaskDelay），就會被掛到 `pxDelayedTaskList`（阻塞鏈表）中。這個節點就是那個「掛鉤」。
- **`xEventListItem`（事件鏈表節點）**：當任務在等待信號量、隊列或事件時，它會透過這個節點掛到對應的信號量/隊列等待隊伍裡
- **`uxPriority`（優先級）**：數字越大，優先級越高。FreeRTOS 會根據它來決定先幫誰「掛鉤」。
- **`xStateListItem` 的 Value**：記錄的是**觸發時間點**（Tick 數）。系統以此來決定誰先超時
- **`xEventListItem` 的 Value**：記錄的是**任務的優先級**（Priority）。
	- **原因**：當多個任務都在等同一個信號量或隊列時，一旦信號量被釋放，應該讓**優先級最高**的任務先獲得。因為 `xEventListItem` 的 Value 綁定了優先級，它在事件隊伍裡會**自動按優先級從高到低排序**。排在最前面的，永遠是優先級最高的任務，內核可以直接一刀切切到最優先的任務。
- `xEventListItem` 就像是任務派出去**在各個互斥鎖、隊列、信號量門口排隊的「替身」**，而
- `xStateListItem` 是留在系統核心大本營**記錄基本生存狀態（醒著、睡著、超時）的「本尊」**

### 3. 多核心（Multi-Core）專屬配置
如果你的晶片是雙核或多核（例如 ESP32 或新版 Cortex-M 運行的 FreeRTOS），這部分才會啟用：
```C
#if ( configUSE_CORE_AFFINITY == 1 ) && ( configNUMBER_OF_CORES > 1 )
    UBaseType_t uxCoreAffinityMask; /**< 核心親和性掩碼 */
#endif
#if ( configNUMBER_OF_CORES > 1 )
    volatile BaseType_t xTaskRunState;      /**< 正在哪個核心運行，或處於什麼狀態 */
    UBaseType_t uxTaskAttributes;           /**< 任務屬性（如空閒任務） */
#endif
```
- **`uxCoreAffinityMask`（核心親和性）**：用來指定「這個任務只能在 Core 0 跑」或「只能在 Core 1 跑」，防止任務在不同核心之間反覆橫跳導致快取（Cache）失效。
- **`xTaskRunState`**：多核環境下，用來標記這個任務當前是被 Core 0 執行、Core 1 執行，還是沒在執行。

### 4. 高級特性的機制維護
FreeRTOS 許多強大的機制，其底層數據也是存放在 TCB 裡：
#### 4.1 互斥鎖與優先級翻轉（Mutexes）
- 當發生優先級繼承時（一個高優先級任務在等低優先級任務手裡的鎖），低優先級任務的 `uxPriority` 會被臨時提得很高。此時 `uxBasePriority` 就用來記住它**原本真實的優先級**，等它釋放鎖（`uxMutexesHeld` 歸零）後改回去。
```C
#if ( configUSE_MUTEXES == 1 )
    UBaseType_t uxBasePriority; /**< 基礎優先級 */
    UBaseType_t uxMutexesHeld;  /**< 持有了多少個互斥鎖 */
#endif
```
#### 4.2 任務通知（Task Notifications）
- FreeRTOS 的「任務通知」非常高效，原因就在這裡：它直接在 TCB 內部開闢了數組（`ulNotifiedValue`）。別人想通知這個任務，不需要透過複雜的隊列，直接修改這個 TCB 裡的變量即可，速度極快。
```C
#if ( configUSE_TASK_NOTIFICATIONS == 1 )
    volatile uint32_t ulNotifiedValue[ configTASK_NOTIFICATION_ARRAY_ENTRIES ];
    volatile uint8_t ucNotifyState[ configTASK_NOTIFICATION_ARRAY_ENTRIES ];
#endif
```
#### 4.3 線程局部存儲（Thread Local Storage - TLS）
-  有些全局變量（如 C 標準庫的 `errno`），如果多個任務同時改它會出亂子。TLS 允許每個任務在自己的 TCB 裡擁有一套私有的指針陣列，實現「各用各的全局變量」。
```C
#if ( configNUM_THREAD_LOCAL_STORAGE_POINTERS > 0 )
    void * pvThreadLocalStoragePointers[ configNUM_THREAD_LOCAL_STORAGE_POINTERS ];
#endif
```
#### 4.4 調試與輔助工具
- **`pcTaskName`**：這純粹是給人看的（例如你在邏輯分析儀或偵錯器裡看到 "Task_LED"），CPU 本身根本不在乎任務叫什麼。
- **`ulRunTimeCounter`**：用來統計 CPU 使用率。它記錄了這個任務總共霸占了 CPU 多少微秒。
```C
char pcTaskName[ configMAX_TASK_NAME_LEN ]; /**< 任務名稱字串 */
#if ( configUSE_TRACE_FACILITY == 1 )
    UBaseType_t uxTCBNumber;  /**< TCB 固定的 ID 號 */
    UBaseType_t uxTaskNumber; /**< 給第三方追蹤工具用的號碼 */
#endif
#if ( configGENERATE_RUN_TIME_STATS == 1 )
    configRUN_TIME_COUNTER_TYPE ulRunTimeCounter; /**< 任務執行總時間 */
#endif
```
- 當一個任務被創建時發生了什麼？
當你調用 `xTaskCreate()` 時，FreeRTOS 主要做了三件事：
1. 在記憶體裡切出一塊空間當作 **Stack**（用來給函數跑、存局部變量）。
2. 在記憶體裡切出一塊空間當作這個 **TCB** 結構體。
3. 把 TCB 裡的 `pxStack` 指向剛才的 Stack，把優先級、名字填好，然後把 `xStateListItem` 掛進「就緒鏈表」。
## Lists for ready & blocked tasks (472 to 540 lines)

```C
/* Lists for ready and blocked tasks. --------------------

* xDelayedTaskList1 and xDelayedTaskList2 could be moved to function scope but

* doing so breaks some kernel aware debuggers and debuggers that rely on removing

* the static qualifier. */

PRIVILEGED_DATA static List_t pxReadyTasksLists[ configMAX_PRIORITIES ]; /**< Prioritised ready tasks. */

PRIVILEGED_DATA static List_t xDelayedTaskList1; /**< Delayed tasks. */

PRIVILEGED_DATA static List_t xDelayedTaskList2; /**< Delayed tasks (two lists are used - one for delays that have overflowed the current tick count. */

PRIVILEGED_DATA static List_t * volatile pxDelayedTaskList; /**< Points to the delayed task list currently being used. */

PRIVILEGED_DATA static List_t * volatile pxOverflowDelayedTaskList; /**< Points to the delayed task list currently being used to hold tasks that have overflowed the current tick count. */

PRIVILEGED_DATA static List_t xPendingReadyList; /**< Tasks that have been readied while the scheduler was suspended. They will be moved to the ready list when the scheduler is resumed. */

  

#if ( INCLUDE_vTaskDelete == 1 )

  

PRIVILEGED_DATA static List_t xTasksWaitingTermination; /**< Tasks that have been deleted - but their memory not yet freed. */

PRIVILEGED_DATA static volatile UBaseType_t uxDeletedTasksWaitingCleanUp = ( UBaseType_t ) 0U;

  

#endif

  

#if ( INCLUDE_vTaskSuspend == 1 )

  

PRIVILEGED_DATA static List_t xSuspendedTaskList; /**< Tasks that are currently suspended. */

  

#endif

  

/* Global POSIX errno. Its value is changed upon context switching to match

* the errno of the currently running task. */

#if ( configUSE_POSIX_ERRNO == 1 )

int FreeRTOS_errno = 0;

#endif

  

/* Other file private variables. --------------------------------*/

PRIVILEGED_DATA static volatile UBaseType_t uxCurrentNumberOfTasks = ( UBaseType_t ) 0U;

PRIVILEGED_DATA static volatile TickType_t xTickCount = ( TickType_t ) configINITIAL_TICK_COUNT;

PRIVILEGED_DATA static volatile UBaseType_t uxTopReadyPriority = tskIDLE_PRIORITY;

PRIVILEGED_DATA static volatile BaseType_t xSchedulerRunning = pdFALSE;

PRIVILEGED_DATA static volatile TickType_t xPendedTicks = ( TickType_t ) 0U;

PRIVILEGED_DATA static volatile BaseType_t xYieldPendings[ configNUMBER_OF_CORES ] = { pdFALSE };

PRIVILEGED_DATA static volatile BaseType_t xNumOfOverflows = ( BaseType_t ) 0;

PRIVILEGED_DATA static UBaseType_t uxTaskNumber = ( UBaseType_t ) 0U;

PRIVILEGED_DATA static volatile TickType_t xNextTaskUnblockTime = ( TickType_t ) 0U; /* Initialised to portMAX_DELAY before the scheduler starts. */

PRIVILEGED_DATA static TaskHandle_t xIdleTaskHandles[ configNUMBER_OF_CORES ]; /**< Holds the handles of the idle tasks. The idle tasks are created automatically when the scheduler is started. */

  

/* Improve support for OpenOCD. The kernel tracks Ready tasks via priority lists.

* For tracking the state of remote threads, OpenOCD uses uxTopUsedPriority

* to determine the number of priority lists to read back from the remote target. */

static const volatile UBaseType_t uxTopUsedPriority = configMAX_PRIORITIES - 1U;

  

/* Context switches are held pending while the scheduler is suspended. Also,

* interrupts must not manipulate the xStateListItem of a TCB, or any of the

* lists the xStateListItem can be referenced from, if the scheduler is suspended.

* If an interrupt needs to unblock a task while the scheduler is suspended then it

* moves the task's event list item into the xPendingReadyList, ready for the

* kernel to move the task from the pending ready list into the real ready list

* when the scheduler is unsuspended. The pending ready list itself can only be

* accessed from a critical section.

*

* Updates to uxSchedulerSuspended must be protected by both the task lock and the ISR lock

* and must not be done from an ISR. Reads must be protected by either lock and may be done

* from either an ISR or a task. */

PRIVILEGED_DATA static volatile UBaseType_t uxSchedulerSuspended = ( UBaseType_t ) 0U;

  

#if ( configGENERATE_RUN_TIME_STATS == 1 )

  

/* Do not move these variables to function scope as doing so prevents the

* code working with debuggers that need to remove the static qualifier. */

PRIVILEGED_DATA static configRUN_TIME_COUNTER_TYPE ulTaskSwitchedInTime[ configNUMBER_OF_CORES ] = { 0U }; /**< Holds the value of a timer/counter the last time a task was switched in. */

PRIVILEGED_DATA static volatile configRUN_TIME_COUNTER_TYPE ulTotalRunTime[ configNUMBER_OF_CORES ] = { 0U }; /**< Holds the total amount of execution time as defined by the run time counter clock. */

  

#endif
```

### 1. 就緒隊列：`pxReadyTasksLists`
```C
PRIVILEGED_DATA static List_t pxReadyTasksLists[ configMAX_PRIORITIES ];
```
-  這不是一個鏈表，而是一個**鏈表陣列**！陣列的大小取決於你配置了多少個優先級（`configMAX_PRIORITIES`）
- **怎麼運作**：優先級為 3 的就緒任務，就會被掛到 `pxReadyTasksLists[3]` 裡。這樣做的好處是，當調度器（Scheduler）要尋找下一個執行的任務時，它只需要從高到低檢查哪個陣列索引裡有任務，就能**以 $O(1)$ 的極快速度**找到最高優先級的任務，完全不需要遍歷所有任務。
### 2. 延時隊列的「時間溢出」雙保險機制
```C
PRIVILEGED_DATA static List_t xDelayedTaskList1;
PRIVILEGED_DATA static List_t xDelayedTaskList2;
PRIVILEGED_DATA static List_t * volatile pxDelayedTaskList;
PRIVILEGED_DATA static List_t * volatile pxOverflowDelayedTaskList;
```
這是 FreeRTOS 設計非常聰明的地方。 系統是用一個計數器 `xTickCount`（比如 32 位的整數）來記錄時間。隨著時間推移，這個計數器**一定會溢出（從 `0xFFFFFFFF` 變回 `0`）**
- **問題來了**：如果現在 `xTickCount` 是 `0xFFFFFFF0`，某個任務說「我要延時 30 個 Tick」。那它預期醒來的時間就是 `0xFFFFFFF0 + 30 = 14`（發生了溢出）。如果你把它跟那些沒溢出的任務放在同一個鏈表裡按時間排序，整個時間線就全亂了
- **解法**：FreeRTOS 準備了兩個延時鏈表（`xDelayedTaskList1` 和 `xDelayedTaskList2`）
	- **當前鏈表（`pxDelayedTaskList` 指向）**：存放**沒溢出**的延時任務（醒來時間大於當前 Tick）
	- **溢出鏈表（`pxOverflowDelayedTaskList` 指向）**：專門存放**已經溢出**的延時任務（如上面那個要在第 14 個 Tick 醒來的任務）
- **翻轉**：當 `xTickCount` 真的溢出歸零的那一瞬間，內核會**交換這兩個指針**。原本的「溢出鏈表」瞬間變成「當前鏈表」，完美解決了時間溢出的斷代問題
### 3. 特殊狀態隊列
```C
PRIVILEGED_DATA static List_t xPendingReadyList;   // 暫掛就緒鏈表
PRIVILEGED_DATA static List_t xTasksWaitingTermination; // 等待銷毀鏈表
PRIVILEGED_DATA static List_t xSuspendedTaskList;  // 掛起鏈表
```
- **`xPendingReadyList`（極其重要）**：當你在代碼中調用 `vTaskSuspendAll()` 暫停了調度器，此時如果有中斷（ISR）觸發並喚醒了一個任務，這個任務**不能**直接放進前面的 `pxReadyTasksLists`（因為調度器被鎖住了，亂動鏈表會引發競爭）。中斷會先把任務丟進這個 `xPendingReadyList`。等稍後你恢復調度器時，內核再一次性把這裡面的任務搬回真正的就緒鏈表。
- **`xTasksWaitingTermination`**：當你調用 `vTaskDelete()` 刪除一個任務時，如果它是在刪除自己，它無法自己釋放自己的 TCB 和 Stack 記憶體（因為它還在運行）。內核會把它掛到這個鏈表，然後由系統內建的 **Idle 任務（空閒任務）** 在背景默默地幫它收屍、釋放記憶體。
- **`xSuspendedTaskList`**：存放那些調用了 `vTaskSuspend()`、徹底不參與排隊的「死忠睡眠」任務，除非別人調用 `vTaskResume()` 喚醒它。
### 4. 核心狀態監視器（核心變數）
```C
PRIVILEGED_DATA static volatile TickType_t xTickCount; // 系統的心跳計數器（每響一次 Tick 加 1）

PRIVILEGED_DATA static volatile UBaseType_t uxTopReadyPriority; // 當前就緒任務中的最高優先級。調度器看它就知道該去 pxReadyTasksLists 的哪個位置抓人。

PRIVILEGED_DATA static volatile TickType_t xNextTaskUnblockTime; // 記錄「下一個最近要醒過來的任務」的時間點。調度器每次心跳只需要對比這個時間，不需要每次都去檢查延時鏈表，大幅提升效率。
```
### 5. 關於開頭和調試（Debug）的註釋解釋
```C
/*
 * xDelayedTaskList1 and xDelayedTaskList2 could be moved to function scope but   * doing so breaks some kernel aware debuggers...*/
```
**這是在說**：雖然這些變數從 C 語言的規範來看，可以寫進函數內部優化空間，但 FreeRTOS 故意把它們放在全局（私有靜態），是為了方便像 **OpenOCD、J-Link、ST-Link** 這些硬體仿真器。當你的程式崩潰時，這些偵錯工具能直接去特定的記憶體地址讀取這些鏈表，在電腦畫面上直接秀出「目前有哪些任務在 Ready、哪些在 Blocked」，這是對開發者非常貼心的設計。

## Definitions of Private functions (542 to 820 lines)
### 1. 任務創建與初始化核心 (Task Creation & Initialization)
#### *prvCreateIdleTasks*

```C
/* Creates the idle tasks during scheduler start. */
static BaseType_t prvCreateIdleTasks( void );
```
- 作用： 在啟動排程器（Scheduler）時，負責自動創建系統預設的 Idle Task。如果是在 SMP（多核心）環境下，它也會一併創建 Passive Idle Tasks。
- **注意/補充：** 此函數是在 `vTaskStartScheduler()` 內部被呼叫的，使用者不需要也不應該手動呼叫。
	- 如果 Idle Task 創建失敗（通常是因為記憶體不足），排程器將無法成功啟動。
#### *prvInitialiseNewTask*

```C
/* 
Called after a Task_t structure has been allocated either statically or dynamically to fill in the structure's members.
*/
static void prvInitialiseNewTask( TaskFunction_t pxTaskCode,
                                  const char * const pcName,
                                  const configSTACK_DEPTH_TYPE uxStackDepth,
                                  void * const pvParameters,
                                  UBaseType_t uxPriority,
                                  TaskHandle_t * const pxCreatedTask,
                                  TCB_t * pxNewTCB,
                                  const MemoryRegion_t * const xRegions ) PRIVILEGED_FUNCTION;
```
- **作用：** 這是所有任務創建方式（無論是動態、靜態還是 MPU 限制任務）都會流經的**底層核心初始化函數**。負責將傳入的參數填入 TCB（任務控制塊）結構體中、初始化任務堆疊（Stack）、並填入初始上下文。
- 注意/補充：
	- 帶有 `PRIVILEGED_FUNCTION` 巨集，代表在 MPU（記憶體保護單元）啟動的系統中，此函數必須運行在特權模式下。
	- 它會在此時將堆疊空間填滿特定的數值（如 `0xa5`），以便後續做堆疊溢位檢測或剩餘空間計算。
#### *prvAddNewTaskToReadyList*

```C
/*
Called after a new task has been created and initialised to place the task under the control of the scheduler.
*/
static void prvAddNewTaskToReadyList( TCB_t * pxNewTCB ) PRIVILEGED_FUNCTION;
```
- **作用：** 將剛初始化完成的新任務，正式加入到系統的 **Ready List（就緒列表）** 中，讓它開始接受排程器的調度。
- 注意/補充：
	- 如果這是系統創建的**第一個任務**，此函數內部會順便觸發 `prvInitialiseTaskLists()` 來初始化所有的任務列表。
	- 它還會判斷新任務的優先級是否高於當前運行的任務，若是，則會觸發上下文切換（Context Switch）或標記需要 Yield。
### 2. 任務創建的多種變體 (Task Creation Variants)
這裡包含了 4 種創建任務的私有變體，分別對應「靜態/動態記憶體」與「是否啟用 MPU（Restricted）」。
#### *prvCreateTask (動態配置)*

```C
/* Create a task with allocated buffer for both TCB and stack. Returns a handle to the task if it is created successfully. Otherwise, returns NULL. */

#if ( configSUPPORT_DYNAMIC_ALLOCATION == 1 )
    static TCB_t * prvCreateTask( TaskFunction_t pxTaskCode,
                                  const char * const pcName,
                                  const configSTACK_DEPTH_TYPE uxStackDepth,
                                  void * const pvParameters,
                                  UBaseType_t uxPriority,
                                  TaskHandle_t * const pxCreatedTask ) PRIVILEGED_FUNCTION;
#endif
```
- **作用：** 標準動態配置任務。內部透過 `pvPortMalloc` 自動從 Heap 中配置 TCB 和 Stack 的記憶體空間。
- **注意/補充：** 依賴 `configSUPPORT_DYNAMIC_ALLOCATION` 必須為 1。
#### *prvCreateStaticTask (靜態配置)*

```C
/* Create a task with static buffer for both TCB and stack. Returns a handle to
the task if it is created successfully. Otherwise, returns NULL. */

#if ( configSUPPORT_STATIC_ALLOCATION == 1 )
    static TCB_t * prvCreateStaticTask( TaskFunction_t pxTaskCode,
                                        const char * const pcName,
                                        const configSTACK_DEPTH_TYPE uxStackDepth,
                                        void * const pvParameters,
                                        UBaseType_t uxPriority,
                                        StackType_t * const puxStackBuffer,
                                        StaticTask_t * const pxTaskBuffer,
                                        TaskHandle_t * const pxCreatedTask ) PRIVILEGED_FUNCTION;
#endif
```

- **作用：** 靜態配置任務。TCB 與 Stack 的記憶體空間由外部（使用者）提供指標傳入，不佔用 FreeRTOS Heap。
- **注意/補充：** 用於安全要求極高、不允許動態記憶體碎片化的系統。
#### *prvCreateRestrictedTask* (MPU 動態)

```C
/* Create a restricted task with static buffer for task stack and allocated buffer for TCB. Returns a handle to the task if it is created successfully. Otherwise, returns NULL.
*/

#if ( ( portUSING_MPU_WRAPPERS == 1 ) && ( configSUPPORT_DYNAMIC_ALLOCATION == 1 ) )
    static TCB_t * prvCreateRestrictedTask( const TaskParameters_t * const pxTaskDefinition,
                                            TaskHandle_t * const pxCreatedTask ) PRIVILEGED_FUNCTION;
#endif
```

- **作用：** 創建受 MPU 保護的任務，其 TCB 是動態配置的，但可以定義專屬的記憶體保護區域（Memory Regions）。
#### *prvCreateRestrictedStaticTask (MPU 靜態)*

```C
/* Create a restricted task with static buffer for both TCB and stack. Returns a handle to the task if it is created successfully. Otherwise, returns NULL.
*/

#if ( ( portUSING_MPU_WRAPPERS == 1 ) && ( configSUPPORT_STATIC_ALLOCATION == 1 ) )
    static TCB_t * prvCreateRestrictedStaticTask( const TaskParameters_t * const pxTaskDefinition,
                                                  TaskHandle_t * const pxCreatedTask ) PRIVILEGED_FUNCTION;
#endif

```

- **作用：** 創建受 MPU 保護且完全靜態配置的任務（TCB 與 Stack 均由外部提供）。
### 3. 多核心 SMP 專屬調度 (SMP Specific Functions)
**注意：** 以下函數全包在 `#if ( configNUMBER_OF_CORES > 1 )` 中，專供 FreeRTOS SMP 版本使用。
#### 3.1 *prvCheckForRunStateChange* 

```C
/* Checks to see if another task moved the current task out of the ready list while it was waiting to enter a critical section and yields, if so. */

static void prvCheckForRunStateChange( void );
```
- **作用：** 當一個任務正在等待進入臨界區（Critical Section）時，檢查是否有其他核心的排程動作將該任務移出了 Ready List。如果是，則主動讓出（Yield）當前核心。也就是當前在等 lock 的 task 被其他 core scheduler 更改狀態（可能是因為等太久要被 suspended 了）然後當前 task 拿到 lock 會呼叫 *`prvCheckForRunStateChange`* 檢查當前 task 狀態有沒有被更改，如果有被改成 suspend (blocked) 就釋放 lock 然後 yield，把 CPU 核心讓給下一個真正處於 Ready 狀態的任務。
- **注意/補充：** 用於防止多核心競態條件（Race Conditions）導致的狀態不同步。

#### 3.2 *prvYieldForTask*

```C
/* Yields a core, or cores if multiple priorities are not allowed to run simultaneously, to allow the task pxTCB to run. */

static void prvYieldForTask( const TCB_t * pxTCB );
```
- **作用：** 為了讓某個特定的高優先級任務 `pxTCB` 能夠立即執行，此函數會去評估並使某一個（或多個）核心進行 Yield。
- **注意/補充：** 在多核心環境下，決定「哪一個核心該讓路」是一件複雜的事，此函數負責這項核心調度決策。
#### 3.3 *prvSelectHighestPriorityTask*

```C
/* Selects the highest priority available task for the given core. */

static void prvSelectHighestPriorityTask( BaseType_t xCoreID );
```
- **作用：** 為指定的核心 `xCoreID` 挑選出當前 Ready List 中優先級最高的可用任務來執行。
- **注意/補充：** 與單核心直接取最高優先級不同，多核心需要排除「已經在其他核心運行中」的任務。
### 4. 狀態管理與列表操作 (State & List Management)
#### 4.1 *prvInitialiseTaskLists*

```C
/* Utility to ready all the lists used by the scheduler. This is called automatically upon the creation of the first task. */

static void prvInitialiseTaskLists( void ) PRIVILEGED_FUNCTION;
```

- **作用：** 初始化 FreeRTOS 核心所使用的所有任務鏈表（Ready Lists, Delayed Lists, Overflow Delayed Lists, Suspended Lists, Pending Ready Lists 等）。
- **注意/補充：** 它是自動被呼叫的（當第一個任務被創建時），不需要手動調用。
#### 4.2 *prvAddCurrentTaskToDelayedList*

```C
/* The currently executing task is entering the Blocked state. Add the task to either the current or the overflow delayed task list. */

static void prvAddCurrentTaskToDelayedList( TickType_t xTicksToWait,
                                            const BaseType_t xCanBlockIndefinitely ) PRIVILEGED_FUNCTION;
```

- **作用：** 當正在運行的任務呼叫了阻礙性 API（如 `vTaskDelay` 或等待 Semaphore 逾時）而進入 Blocked 狀態時，此函數負責將該任務移出 Ready List，並加入到 Delayed 延時鏈表中。
- **注意/補充：** FreeRTOS 內部有兩個延時鏈表（當前與溢出），此函數會判斷 Tick 是否會溢出，並精確放入正確的 List 中。
	- 如果 `xCanBlockIndefinitely` 為 `pdTRUE` 且延時時間為 `portMAX_DELAY`，在啟用掛起功能時，它會被直接放入 Suspended List。
#### 4.3 *prvResetNextTaskUnblockTime*

```C
/* Set xNextTaskUnblockTime to the time at which the next Blocked state task will exit the Blocked state. */

static void prvResetNextTaskUnblockTime( void ) PRIVILEGED_FUNCTION;
```

- **作用：** 重新計算並更新 `xNextTaskUnblockTime` 這個全域變數。這個變數記錄了「下一個任務要被喚醒的時間點」。
- **注意/補充：** 每次有任務退出 Blocked 狀態、或是 Delayed List 發生變更時，都需要呼叫此函數更新時間，以便 Tick 中斷能精準判斷何時該喚醒任務。

### 5. 任務銷毀與終止 (Task Deletion & Termination)

#### 5.1 *prvDeleteTCB*

```C
/* Utility to free all memory allocated by the scheduler to hold a TCB, including the stack pointed to by the TCB.
This does not free memory allocated by the task itself (i.e. memory allocated by calls to pvPortMalloc from within the tasks application code). */

#if ( INCLUDE_vTaskDelete == 1 )
    static void prvDeleteTCB( TCB_t * pxTCB ) PRIVILEGED_FUNCTION;
#endif
```

- **作用：** 釋放由系統為該 TCB 配置的所有記憶體（包括任務的 Stack 和 TCB 結構體本身）。
- **注意/補充：** * **重要警告：** 它**只會**釋放 FreeRTOS 自動配置的核心記憶體。如果該任務在運行期間自己呼叫 `pvPortMalloc()` 配置了其他記憶體，或是獲取了硬體資源，此函數**不會**主動釋放，必須在刪除任務前由應用端手動處理。
#### 5.2 *prvCheckTasksWaitingTermination*

```C
/* Used only by the idle task. This checks to see if anything has been placed in the list of tasks waiting to be deleted. If so the task is cleaned up and its TCB deleted. */
static void prvCheckTasksWaitingTermination( void ) PRIVILEGED_FUNCTION;
```

- **作用：** 檢查 `xTasksWaitingTermination` 鏈表（等待銷毀的任務鏈表）。如果裡面有任務，就由這個函數來執行實質的記憶體釋放。
- **注意/補充：** *經典考點：當任務「自己刪除自己」（`vTaskDelete(NULL)`）時，任務無法在運行中釋放自己的 Stack，因此會被掛到這個等待終止列表中。
	- 此函數是在 **Idle Task** 內部被循環呼叫的。也就是說，**如果你的 Idle Task 沒有機會執行（例如被高優先級任務佔滿 CPU），被刪除任務的記憶體就永遠不會被回收！**

### 6. 監控、除錯與工具函數 (Diagnostics & Utilities)

#### 6.1 *prvTaskIsTaskSuspended*

```C
/* Utility task that simply returns pdTRUE if the task referenced by xTask is currently in the Suspended state, or pdFALSE if the task referenced by xTask is in any other state. */

#if ( INCLUDE_vTaskSuspend == 1 )
    static BaseType_t prvTaskIsTaskSuspended( const TaskHandle_t xTask ) PRIVILEGED_FUNCTION;
#endif
```
- **作用：** 工具函數，用來快速查詢指定的任務控制控制代碼 `xTask` 當前是否處於掛起（Suspended）狀態。返回 `pdTRUE` 或 `pdFALSE`。

#### 6.2 *prvListTasksWithinSingleList*


```C
/* Fills an TaskStatus_t structure with information on each task that is referenced from the pxList list (which may be a ready list, a delayed list, a suspended list, etc.). THIS FUNCTION IS INTENDED FOR DEBUGGING ONLY, AND SHOULD NOT BE CALLED FROM NORMAL APPLICATION CODE. */

#if ( configUSE_TRACE_FACILITY == 1 )
    static UBaseType_t prvListTasksWithinSingleList( TaskStatus_t * pxTaskStatusArray,
                                                     List_t * pxList,
                                                     eTaskState eState ) PRIVILEGED_FUNCTION;
#endif
```

- **作用：** 遍歷指定的單一鏈表（例如某個優先級的 Ready List），並將其中所有任務的狀態資訊填入到 `TaskStatus_t` 陣列中。
- **注意/補充：** 專為 Debug 設計（如生成 `vTaskList` 報告），**不應在正式的生產環境商用代碼中呼叫**，因為這會長時間關閉中斷或鎖定排程，影響即時性。

#### 6.3 *prvSearchForNameWithinSingleList*

```C
/* Searches pxList for a task with name pcNameToQuery - returning a handle to the task if it is found, or NULL if the task is not found. */

#if ( INCLUDE_xTaskGetHandle == 1 )
    static TCB_t * prvSearchForNameWithinSingleList( List_t * pxList,
                                                     const char pcNameToQuery[] ) PRIVILEGED_FUNCTION;
#endif
```

- **作用：** 在指定的單一鏈表內，比對任務名稱字串，尋找與 `pcNameToQuery` 相同的任務並返回其 TCB 指標。
- **注意/補充：** 字串比對速度較慢，這也是 `xTaskGetHandle()` 較耗時的原因。

#### 6.4 *prvTaskCheckFreeStackSpace*

```C
/* When a task is created, the stack of the task is filled with a known value. This function determines the 'high water mark' of the task stack by determining how much of the stack remains at the original preset value. */

#if ( ( configUSE_TRACE_FACILITY == 1 ) || ( INCLUDE_uxTaskGetStackHighWaterMark == 1 ) || ( INCLUDE_uxTaskGetStackHighWaterMark2 == 1 ) )
    static configSTACK_DEPTH_TYPE prvTaskCheckFreeStackSpace( const uint8_t * pucStackByte ) PRIVILEGED_FUNCTION;
#endif
```

- **作用：** 從 Stack 的末端（或起始端，取決於 Stack 生長方向）開始往另一端掃描，計算還有多少位元組仍保持著初始值（`0xa5`），以此算出該任務歷史運行的「最小剩餘堆疊空間（High Water Mark）」。

#### 6.5 *prvGetExpectedIdleTime*

```C
/* Return the amount of time, in ticks, that will pass before the kernel will next move a task from the Blocked state to the Running state or before the tick count overflows (whichever is earlier). This conditional compilation should use inequality to 0, not equality to 1.
This is to ensure portSUPPRESS_TICKS_AND_SLEEP() can be called when user defined low power mode implementations require configUSE_TICKLESS_IDLE to be set to a value other than 1. */

#if ( configUSE_TICKLESS_IDLE != 0 )
    static TickType_t prvGetExpectedIdleTime( void ) PRIVILEGED_FUNCTION;
#endif
```

- **作用：** 計算在下一個任務被喚醒前，系統還可以保持空閒（Idle）多少個 Tick 時間。
- **注意/補充：** 專供 **Tickless Idle（低功耗模式）** 使用。外設驅動會根據這個回傳的時間決定要讓 MCU 進入多深度的睡眠模式。
- **原始碼註釋強調：** 巨集判斷使用 `!= 0`（而不是 `== 1`），是為了相容使用者自定義的大於 1 的低功耗模式設定。

#### 6.6 *prvWriteNameToBuffer*

```C
/* Helper function used to pad task names with spaces when printing out human readable tables of task information. */

#if ( configUSE_STATS_FORMATTING_FUNCTIONS > 0 )
    static char * prvWriteNameToBuffer( char * pcBuffer,
                                        const char * pcTaskName ) PRIVILEGED_FUNCTION;
#endif

```

- **作用：** 格式化工具函數。在輸出任務狀態表格（`vTaskList`）時，負責將任務名稱複製到 Buffer 中，並自動補齊空格以對齊排版。

#### 6.7 *prvSnprintfReturnValueToCharsWritten*

```C

/* Convert the snprintf return value to the number of characters
written. The following are the possible cases:

1. The buffer supplied to snprintf is large enough to hold the
generated string. The return value in this case is the number
of characters actually written, not counting the terminating
null character.

2. The buffer supplied to snprintf is NOT large enough to hold
the generated string. The return value in this case is the number of characters that would have been written if the buffer had been sufficiently large, not counting the terminating null character.

3. Encoding error. The return value in this case is a negative
number.

From 1 and 2 above ==> Only when the return value is non-negative
and less than the supplied buffer length, the string has been
completely written.

*/

```

- **作用：** 安全性轉換函數。用來把標準 `snprintf` 的返回值轉換成實際寫入 Buffer 的字元數。
- **注意/補充：** 解決 `snprintf` 在「Buffer 不夠大」或「編碼錯誤」時傳回預期長度而非實際寫入長度的標準 C 庫陷阱，防止 Buffer 溢位。

### 7. 額外補充與擴充鉤子 (Additions & Hooks)

#### 7.1 *freertos_tasks_c_additions_init*

```C
/* freertos_tasks_c_additions_init() should only be called if the user definable macro FREERTOS_TASKS_C_ADDITIONS_INIT() is defined, as that is the only macro called by the function. */

#ifdef FREERTOS_TASKS_C_ADDITIONS_INIT
    static void freertos_tasks_c_additions_init( void ) PRIVILEGED_FUNCTION;
#endif
```

- **作用：** 如果使用者定義了 `FREERTOS_TASKS_C_ADDITIONS_INIT` 這個巨集，核心會在任務初始化階段呼叫此私有函數，用來初始化使用者自己在 `freertos_tasks_c_additions.h` 中定義的第三方或自定義擴充元件。

#### 7.2 *vApplicationPassiveIdleHook*

```C
#if ( configUSE_PASSIVE_IDLE_HOOK == 1 )
    extern void vApplicationPassiveIdleHook( void );
#endif
```

- **作用：** **注意，這是一個 `extern` 函數，非 static。** 這是多核心 SMP 專用的 Hook 函數（鉤子）。當任何一個非主核心（Passive Core）進入各自的 Passive Idle Task 時，會呼叫這個由使用者實作的鉤子函數。
- **注意/補充：** 常用於多核心獨立的低功耗管理或核心狀態觀測。
- `prvIdleTask` 跟 `prvPassiveIdleTask` 的差異
	- 在 FreeRTOS SMP 中，系統啟動時會創建 **1 個 `prvIdleTask`** 和 **(核心數 - 1) 個 `prvPassiveIdleTask`**。

|  特性   |                           Idle Task (主空閒任務)                           |   Passive Idle Task (被動空閒任務)    |
| :---: | :-------------------------------------------------------------------: | :-----------------------------: |
|  數量   |                               全系統只有 1 個                               |  `configNUMBER_OF_CORES - 1` 個  |
| 執行核心  |                         通常固定在 Core 0 運行（主核心）                          | 運行在 Core 1, Core 2... 等其他被動核心上  |
| 核心職責  | 1. 系統資源回收（呼叫 `prvCheckTasksWaitingTermination()` 清理被刪除任務的 Stack/TCB）。 | 1. 讓沒事做的被動核心有代碼可以跑（防止 CPU 懸空）。  |
| 核心職責  |                        2. 執行主空閒鉤子（Idle Hook）。                         | 2. 執行被動空閒鉤子（Passive Idle Hook）。 |
| 記憶體清理 |                             **會** 執行清理工作                              |    **不會** 執行清理工作（避免多核心競爭衝突）     |

## Implementations of all Functions (822 to ...end)
### 1. *prvCheckForRunStateChange*

```C
#if ( configNUMBER_OF_CORES > 1 )
    static void prvCheckForRunStateChange( void )
    {
        UBaseType_t uxPrevCriticalNesting;
        const TCB_t * pxThisTCB;
        BaseType_t xCoreID = ( BaseType_t ) portGET_CORE_ID();

        /* This must only be called from within a task. */
        portASSERT_IF_IN_ISR();

        /* This function is always called with interrupts disabled
         * so this is safe. */
        pxThisTCB = pxCurrentTCBs[ xCoreID ];

        while( pxThisTCB->xTaskRunState == taskTASK_SCHEDULED_TO_YIELD )
        {
            /* We are only here if we just entered a critical section
             * or if we just suspended the scheduler, and another task
             * has requested that we yield.
             *
             * This is slightly complicated since we need to save and restore
             * the suspension and critical nesting counts, as well as release
             * and reacquire the correct locks. And then, do it all over again
             * if our state changed again during the reacquisition. */
            uxPrevCriticalNesting = portGET_CRITICAL_NESTING_COUNT( xCoreID );

            if( uxPrevCriticalNesting > 0U )
            {
                portSET_CRITICAL_NESTING_COUNT( xCoreID, 0U );
                portRELEASE_ISR_LOCK( xCoreID );
            }
            else
            {
                /* The scheduler is suspended. uxSchedulerSuspended is updated
                 * only when the task is not requested to yield. */
                mtCOVERAGE_TEST_MARKER();
            }

            portRELEASE_TASK_LOCK( xCoreID );
            portMEMORY_BARRIER();
            configASSERT( pxThisTCB->xTaskRunState == taskTASK_SCHEDULED_TO_YIELD );

            portENABLE_INTERRUPTS();

            /* Enabling interrupts should cause this core to immediately service
             * the pending interrupt and yield. After servicing the pending interrupt,
             * the task needs to re-evaluate its run state within this loop, as
             * other cores may have requested this task to yield, potentially altering
             * its run state. */

            portDISABLE_INTERRUPTS();

            xCoreID = ( BaseType_t ) portGET_CORE_ID();
            portGET_TASK_LOCK( xCoreID );
            portGET_ISR_LOCK( xCoreID );

            portSET_CRITICAL_NESTING_COUNT( xCoreID, uxPrevCriticalNesting );

            if( uxPrevCriticalNesting == 0U )
            {
                portRELEASE_ISR_LOCK( xCoreID );
            }
        }
    }
#endif /* #if ( configNUMBER_OF_CORES > 1 ) */
```

#### 1.1 準備階段：確認身分與環境

```C
UBaseType_t uxPrevCriticalNesting;
const TCB_t * pxThisTCB;
BaseType_t xCoreID = ( BaseType_t ) portGET_CORE_ID();

/* 斷言：這個函數絕對不能在中斷（ISR）中被呼叫，只能在 Task 中呼叫 */
portASSERT_IF_IN_ISR();

/* 取得當前核心正在跑的 Task TCB 指標 */
pxThisTCB = pxCurrentTCBs[ xCoreID ];
```
- **解析：** 進入函數時，中斷已經是關閉狀態了。它先獲取當前核心的 ID，並抓出自己的 TCB。

#### 1.2 **解析：** 進入函數時，中斷已經是關閉狀態了。它先獲取當前核心的 ID，並抓出自己的 TCB。

```C
while( pxThisTCB->xTaskRunState == taskTASK_SCHEDULED_TO_YIELD )
{
```

- **解析：** 這是最重要的檢查點！`taskTASK_SCHEDULED_TO_YIELD` 代表**別的核心已經對這個 Task 發出了「請你讓路」的請求**。如果成立，代表這個 Task 雖然剛搶到鎖，但它已經不合法了，必須執行退場。

#### 1.3 解除武裝：備份狀態並把鎖全部放開

```C
/* 1. 備份當前的臨界區巢狀計數（Critical Nesting） */
    uxPrevCriticalNesting = portGET_CRITICAL_NESTING_COUNT( xCoreID );

    if( uxPrevCriticalNesting > 0U )
    {
        /* 如果原本就在臨界區內，將計數歸零，並釋放 ISR 鎖（核心間的中斷鎖） */
        portSET_CRITICAL_NESTING_COUNT( xCoreID, 0U );
        portRELEASE_ISR_LOCK( xCoreID );
    }
    else
    {
        /* 走到這代表臨界區計數為 0，此時是因為「排程器被掛起（Suspended）」而需要 Yield */
        mtCOVERAGE_TEST_MARKER();
    }

    /* 2. 釋放任務鎖（Task Lock），讓別的核心可以存取任務鏈表 */
    portRELEASE_TASK_LOCK( xCoreID );
    
    /* 記憶體屏障，確保前面的放鎖操作已經實質寫入硬體暫存器 */
    portMEMORY_BARRIER();
    
    configASSERT( pxThisTCB->xTaskRunState == taskTASK_SCHEDULED_TO_YIELD );
```

- **解析：** 退場前，必須把 `ISR_LOCK` 和 `TASK_LOCK` 這兩把大鎖全部解開。但在解開前，必須用 `uxPrevCriticalNesting` 記住剛才的臨界區層級，等以後醒來時才能「完美還原」。

#### 1.4 執行退場：開啟中斷，迎接命中註定的 Yield

```C
/* 3. 真正開啟中斷！ */
    portENABLE_INTERRUPTS();

    /* 【時空交錯點】
     * 當中斷一開啟，因為剛才別的核心有發出 Yield 請求，
     * 硬體會立刻觸發一個 Pending 的排程中斷（例如 ARM 的 PendSV）。
     * 這個核心會在這裡「被強行切換出去」，換別的任務來跑。
     * * ...... (這個 Task 在這裡沉睡了很久) ......
     * * 當這個 Task 很久以後重新被排程器選中、再度醒來並回到這行程式碼時，
     * 它會繼續往下執行。 */

    /* 4. 醒來了，立刻再次關閉中斷 */
    portDISABLE_INTERRUPTS();
```

- **解析：** 這是這段程式碼最精妙的魔術。`portENABLE_INTERRUPTS()` 開啟中斷的瞬間，硬體中斷立刻介入，**當前 Task 就在這裡被換下場（Yield）**。等到它下一次又符合資格「醒過來」的時候，程式指標才會走到下一行的 `portDISABLE_INTERRUPTS()`。

#### 1.5 武裝重整：把鎖拿回來，還原案發現場

既然醒過來了，就要把剛才放開的鎖重新搶回來，繼續執行原本未完成的臨界區代碼。

```C
/* 重新取得當前的 Core ID（因為醒來時可能已經被轉移到另一個核心跑了！） */
    xCoreID = ( BaseType_t ) portGET_CORE_ID();
    
    /* 重新搶回 Task 鎖和 ISR 鎖 */
    portGET_TASK_LOCK( xCoreID );
    portGET_ISR_LOCK( xCoreID );

    /* 還原當初備份的臨界區巢狀計數 */
    portSET_CRITICAL_NESTING_COUNT( xCoreID, uxPrevCriticalNesting );

    /* 如果當初本來就不是因為臨界區進來的（uxPrevCriticalNesting == 0），
     * 那我們剛才多拿的 ISR 鎖就要放掉。 */
    if( uxPrevCriticalNesting == 0U )
    {
        portRELEASE_ISR_LOCK( xCoreID );
    }
} /* While 迴圈結束，回到頂端重新檢查 xTaskRunState */
```

- **解析：** 注意！多核心中任務可能會「換核運行（Task Migration）」。任務在 Core 0 睡著，醒來時可能在 Core 1。所以必須重新呼叫 `portGET_CORE_ID()`。
	- 重新把鎖拿回來後，**迴圈會再次回到最上面**。為什麼？因為在剛才重新搶鎖的過程中，說不定「又」有其他核心在背後對我們發出 Yield 請求了。所以必須不斷 While 檢查，直到狀態完全乾淨（不是 `taskTASK_SCHEDULED_TO_YIELD`），函數才會安全退出。
#### 1.6 細節補充

##### 1.6.1 為什麼在 User Space 可以開中斷？
在 FreeRTOS 的上下文環境中，這裡的程式碼其實**不是運行在一般的 User Space（如 Linux 的應用層）**，它依然屬於 **Kernel Space**。
- **FreeRTOS 的「任務上下文」：** 即使 FreeRTOS 啟用了 MPU（記憶體保護），將任務分為「特權任務（Privileged）」與「非特權任務（Unprivileged，即你指的 User Space）」，**`prvCheckForRunStateChange` 這個函數本身也帶有 `PRIVILEGED_FUNCTION` 屬性**，代表它必須運行在特權模式下。
- **硬體層面的中斷控制：** 在 ARM 架構中，只有在特權模式（Privileged Mode）下，CPU 才有權限去修改 `PRIMASK`、`BASEPRI` 或 `CPSR` 暫存器來開關中斷。非特權的 User 任務如果直接呼叫開關中斷指令，會直接觸發硬體異常（Usage Fault）。
- **既然是核心程式，為什麼敢在這裡開中斷？** 因為不開中斷，**排程器的軟中斷（如 PendSV）就進不來，這個核心就會被永遠卡死在這個 `while` 迴圈裡！** 為了讓「別的核心發出的 Yield 請求」能實質生效，當前核心必須短暫地開啟中斷，給硬體一個觸發排程切換的窗口。
##### 1.6.2 既然至少拿了 Task Lock，`uxPrevCriticalNesting` 怎麼會等於 0？
這牽涉到 FreeRTOS SMP 對於 **「臨界區（Critical Section）」** 與 **「自旋鎖（Spinlock）」** 的底層拆分。核心真相：拿 Task Lock $\neq$ 增加 Critical Nesting Count
在 FreeRTOS SMP 中，有兩種方式會要求任務 Yield 並進入這個函數：
- 情況 A：透過進入臨界區（Nesting > 0）
	 - 當代碼呼叫 `taskENTER_CRITICAL()` 時：
		 1. 增加臨界區巢狀計數（`uxCriticalNesting++`，此時變大於 0）。
		 2. 獲取 `ISR_LOCK` 與 `TASK_LOCK`。
		 3. 呼叫 `prvCheckForRunStateChange()`。
		 4. 此時 `uxPrevCriticalNesting > 0` 成立。

- 透過掛起排程器（Nesting == 0）
	- 當代碼呼叫 `vTaskSuspendAll()` 時：
		- 排程器被掛起（Suspended），此時並沒有進入臨界區，所以 `uxCriticalNesting` 依然是 0。
		- 就在排程器被掛起的期間，別的核心（Core 1）發出了 Yield 請求。
		- 當前核心在後續某些檢查點（例如準備恢復排程或在掛起期間呼叫了某些核心 API），核心**內部**為了安全，會主動去獲取 `TASK_LOCK` 來保護鏈表，並直接呼叫 `prvCheckForRunStateChange()`。
##### 1.6.3 Task Lock 是用來幹嘛的？
在單核心系統中，要保護核心的任務鏈表（Ready List 等），警衛只需要「關閉中斷」就萬事大吉了。但在多核心（SMP）系統中，因為多個核心可以同時執行，FreeRTOS 引入了兩把最核心的自旋鎖：**`ISR_LOCK`** 和 **`TASK_LOCK`**。
其中，**`TASK_LOCK` 的唯一使命，就是保護「全系統所有與任務（Task）相關的共享資料結構」。**
全系統中只有一份的共享帳本：
- **`pxReadyTasksLists`**（所有優先級的就緒任務鏈表）
- **`xDelayedTaskList` / `xOverflowDelayedTaskList`**（延時任務鏈表）
- **`xTasksWaitingTermination`**（等待被 Idle Task 清理、銷毀的任務鏈表）
- **`pxCurrentTCBs`**（記錄每個核心當前正在跑哪一個任務的陣列）

### 2. *prvYieldForTask*

```C
#if ( configNUMBER_OF_CORES > 1 )
    static void prvYieldForTask( const TCB_t * pxTCB )
    {
        BaseType_t xLowestPriorityToPreempt;
        BaseType_t xCurrentCoreTaskPriority;
        BaseType_t xLowestPriorityCore = ( BaseType_t ) -1;
        BaseType_t xCoreID;
        const BaseType_t xCurrentCoreID = ( BaseType_t ) portGET_CORE_ID();

        #if ( configRUN_MULTIPLE_PRIORITIES == 0 )
            BaseType_t xYieldCount = 0;
        #endif /* #if ( configRUN_MULTIPLE_PRIORITIES == 0 ) */

        /* This must be called from a critical section. */
        configASSERT( portGET_CRITICAL_NESTING_COUNT( xCurrentCoreID ) > 0U );

        #if ( configRUN_MULTIPLE_PRIORITIES == 0 )
            /* No task should yield for this one if it is a lower priority
             * than priority level of currently ready tasks. */
            if( pxTCB->uxPriority >= uxTopReadyPriority )
        #else
            /* Yield is not required for a task which is already running. */
            if( taskTASK_IS_RUNNING( pxTCB ) == pdFALSE )
        #endif
        {
            xLowestPriorityToPreempt = ( BaseType_t ) pxTCB->uxPriority;

            /* xLowestPriorityToPreempt will be decremented to -1 if the priority of pxTCB
             * is 0. This is ok as we will give system idle tasks a priority of -1 below. */
            --xLowestPriorityToPreempt;

            for( xCoreID = ( BaseType_t ) 0; xCoreID < ( BaseType_t ) configNUMBER_OF_CORES; xCoreID++ )
            {
                xCurrentCoreTaskPriority = ( BaseType_t ) pxCurrentTCBs[ xCoreID ]->uxPriority;

                /* System idle tasks are being assigned a priority of tskIDLE_PRIORITY - 1 here. */
                if( ( pxCurrentTCBs[ xCoreID ]->uxTaskAttributes & taskATTRIBUTE_IS_IDLE ) != 0U )
                {
                    xCurrentCoreTaskPriority = ( BaseType_t ) ( xCurrentCoreTaskPriority - 1 );
                }

                if( ( taskTASK_IS_RUNNING( pxCurrentTCBs[ xCoreID ] ) != pdFALSE ) && ( xYieldPendings[ xCoreID ] == pdFALSE ) )
                {
                    #if ( configRUN_MULTIPLE_PRIORITIES == 0 )
                        if( taskTASK_IS_RUNNING( pxTCB ) == pdFALSE )
                    #endif
                    {
                        if( xCurrentCoreTaskPriority <= xLowestPriorityToPreempt )
                        {
                            #if ( configUSE_CORE_AFFINITY == 1 )
                                if( ( pxTCB->uxCoreAffinityMask & ( ( UBaseType_t ) 1U << ( UBaseType_t ) xCoreID ) ) != 0U )
                            #endif
                            {
                                #if ( configUSE_TASK_PREEMPTION_DISABLE == 1 )
                                    if( pxCurrentTCBs[ xCoreID ]->xPreemptionDisable == pdFALSE )
                                #endif
                                {
                                    xLowestPriorityToPreempt = xCurrentCoreTaskPriority;
                                    xLowestPriorityCore = xCoreID;
                                }
                            }
                        }
                        else
                        {
                            mtCOVERAGE_TEST_MARKER();
                        }
                    }

                    #if ( configRUN_MULTIPLE_PRIORITIES == 0 )
                    {
                        /* Yield all currently running non-idle tasks with a priority lower than
                         * the task that needs to run. */
                        if( ( xCurrentCoreTaskPriority > ( ( BaseType_t ) tskIDLE_PRIORITY - 1 ) ) &&
                            ( xCurrentCoreTaskPriority < ( BaseType_t ) pxTCB->uxPriority ) )
                        {
                            prvYieldCore( xCoreID );
                            xYieldCount++;
                        }
                        else
                        {
                            mtCOVERAGE_TEST_MARKER();
                        }
                    }
                    #endif /* #if ( configRUN_MULTIPLE_PRIORITIES == 0 ) */
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }

            #if ( configRUN_MULTIPLE_PRIORITIES == 0 )
                if( ( xYieldCount == 0 ) && ( xLowestPriorityCore >= 0 ) )
            #else /* #if ( configRUN_MULTIPLE_PRIORITIES == 0 ) */
                if( xLowestPriorityCore >= 0 )
            #endif /* #if ( configRUN_MULTIPLE_PRIORITIES == 0 ) */
            {
                prvYieldCore( xLowestPriorityCore );
            }

            #if ( configRUN_MULTIPLE_PRIORITIES == 0 )
                /* Verify that the calling core always yields to higher priority tasks. */
                if( ( ( pxCurrentTCBs[ xCurrentCoreID ]->uxTaskAttributes & taskATTRIBUTE_IS_IDLE ) == 0U ) &&
                    ( pxTCB->uxPriority > pxCurrentTCBs[ xCurrentCoreID ]->uxPriority ) )
                {
                    configASSERT( ( xYieldPendings[ xCurrentCoreID ] == pdTRUE ) ||
                                  ( taskTASK_IS_RUNNING( pxCurrentTCBs[ xCurrentCoreID ] ) == pdFALSE ) );
                }
            #endif
        }
    }
#endif /* #if ( configNUMBER_OF_CORES > 1 ) */
```

這段程式碼 `prvYieldForTask` 是 FreeRTOS SMP（多核心）中非常具有決定性的一個私有函數。

簡單來說，它的核心使命是：**「當系統中有一個高優先級的任務（`pxTCB`）準備好要執行了，此時有那麼多顆 CPU 核心（Cores），我該叫哪一個核心把手上的工作停下來（Yield），讓給這個高優先級的任務跑？」**

多核心排程最怕的就是「把正在跑重要高優先級任務的核心給踢下場」，所以這個函數的核心邏輯就是**全盤掃描所有核心，找出當前「最弱（優先級最低）且符合條件」的核心來開刀**。

#### 2.1 準備與基本條件檢查

```C
/* 斷言：呼叫此函數時，當前核心必須已經處於臨界區（拿了鎖、關了中斷） */
configASSERT( portGET_CRITICAL_NESTING_COUNT( xCurrentCoreID ) > 0U );

#if ( configRUN_MULTIPLE_PRIORITIES == 0 )
    /* 如果系統規定所有核心「同時只能跑相同優先級的任務」
     * 那麼只有當新任務的優先級 >= 目前最高準備就緒優先級時，才需要叫別人讓位 */
    if( pxTCB->uxPriority >= uxTopReadyPriority )
#else
    /* 正常的 SMP 環境：只要這個任務當前沒有在任何核心上運行，就值得為它找一個核心讓位 */
    if( taskTASK_IS_RUNNING( pxTCB ) == pdFALSE )
#endif
```

#### 2.2 初始化排 preempt 門檻值

```C
xLowestPriorityToPreempt = ( BaseType_t ) pxTCB->uxPriority;
/* 門檻值減 1。代表等一下掃描各核心時，對方的優先級必須「小於等於」這個門檻，才符合被搶佔的資格 */
--xLowestPriorityToPreempt;
```
- **注意：** 如果新任務的優先級是 `0`（最低），減 1 就會變成 `-1`。沒關係，後面會把 Idle Task 當成 `-1` 來處理。

#### 2.3 全核心大掃描：尋找最適合開刀的「倒楣鬼」
接下來會進入一個 `for` 迴圈，逐一檢查系統中的每一個核心（`xCoreID`）：

```C
for( xCoreID = ( BaseType_t ) 0; xCoreID < ( BaseType_t ) configNUMBER_OF_CORES; xCoreID++ )
{
    xCurrentCoreTaskPriority = ( BaseType_t ) pxCurrentTCBs[ xCoreID ]->uxPriority;

    /* 特殊優待：如果該核心正在跑 Idle Task，將其優先級視為 -1（比所有使用者任務都低）
     * 這樣可以確保 Idle Task 一定會優先被搶佔 */
    if( ( pxCurrentTCBs[ xCoreID ]->uxTaskAttributes & taskATTRIBUTE_IS_IDLE ) != 0U )
    {
        xCurrentCoreTaskPriority = ( BaseType_t ) ( xCurrentCoreTaskPriority - 1 );
    }

    /* 確保該核心有在跑任務，而且目前還沒有被標記為「準備讓權（Yield Pending）」 */
    if( ( taskTASK_IS_RUNNING( pxCurrentTCBs[ xCoreID ] ) != pdFALSE ) && ( xYieldPendings[ xCoreID ] == pdFALSE ) )
    {
        /* 核心過濾關卡 1：該核心上的任務優先級，是否低於新任務（符合搶佔門檻） */
        if( xCurrentCoreTaskPriority <= xLowestPriorityToPreempt )
        {
            #if ( configUSE_CORE_AFFINITY == 1 )
                /* 核心過濾關卡 2（選配）：檢查新任務是否具備「核心親和性（Affinity）」
                 * 亦即新任務被限定只能在某些核心上跑。如果它不能在這顆 Core 上跑，就不能強拆這顆 Core */
                if( ( pxTCB->uxCoreAffinityMask & ( ( UBaseType_t ) 1U << ( UBaseType_t ) xCoreID ) ) != 0U )
            #endif
            {
                #if ( configUSE_TASK_PREEMPTION_DISABLE == 1 )
                    /* 核心過濾關卡 3（選配）：檢查該核心當前運行的任務是否呼叫了「禁止搶佔」 */
                    if( pxCurrentTCBs[ xCoreID ]->xPreemptionDisable == pdFALSE )
                #endif
                {
                    /* 關關難過關關過！這顆核心符合所有被搶佔條件，
                     * 且它是目前為止「最弱（優先級最低）」的核心。記下它的 ID 與優先級。 */
                    xLowestPriorityToPreempt = xCurrentCoreTaskPriority;
                    xLowestPriorityCore = xCoreID;
                }
            }
        }
        
        /* 這裡還有一段針對 configRUN_MULTIPLE_PRIORITIES == 0 的特殊處理：
         * 如果不允許核心跑不同優先級，它會一口氣把所有優先級低於新任務的核心全部 Yield 掉 */
    }
}
```

#### 2.4 configRUN_MULTIPLE_PRIORITIES == 0 的額外優化

在迴圈內部，如果設定為 `0`（嚴格要求執行最高優先級任務），還包含這段代碼：
```C
#if ( configRUN_MULTIPLE_PRIORITIES == 0 )
                    {
                        /* 如果有些核心在跑非閒置任務，但優先級比新任務低，
                           直接叫它們全部 Yield，把資源吐出來給高優先級任務 */
                        if( ( xCurrentCoreTaskPriority > ( ( BaseType_t ) tskIDLE_PRIORITY - 1 ) ) &&
                            ( xCurrentCoreTaskPriority < ( BaseType_t ) pxTCB->uxPriority ) )
                        {
                            prvYieldCore( xCoreID ); // 直接對該核心發送 Yield 訊號
                            xYieldCount++;
                        }
                    }
                    #endif
```

#### 2.5 結尾處的「防呆與自我檢查機制」（除錯用的機制）

如果系統設定不允許不同優先級同時執行，那麼當前正在處理這個新任務的 CPU 核心，必須確保自己沒有做出『讓高優先級任務乾等，自己卻繼續霸佔 CPU 跑低優先級任務』的蠢事。
##### 2.5.1 前置條件
```C
#if ( configRUN_MULTIPLE_PRIORITIES == 0 )
```
- 這個檢查**只有在 `configRUN_MULTIPLE_PRIORITIES == 0` 的時候才會啟用**。
- 因為當這個設定為 `0` 時，FreeRTOS 的硬性規定是：**全系統所有核心，必須盡可能地去執行目前最高優先級的任務**。絕對不允許 Core A 在跑優先級 5 的任務，而 Core B 卻在跑優先級 2 的任務（除非優先級 5 的任務不夠分）。
##### 2.5.2 觸發檢查的時機 (`if` 條件)
```C
if( ( ( pxCurrentTCBs[ xCurrentCoreID ]->uxTaskAttributes & taskATTRIBUTE_IS_IDLE ) == 0U ) &&
    ( pxTCB->uxPriority > pxCurrentTCBs[ xCurrentCoreID ]->uxPriority ) )
```
這裡有兩個條件必須同時成立，才會進去執行斷言（Assert）：
- **`... != taskATTRIBUTE_IS_IDLE`**：當前核心（`xCurrentCoreID`）目前跑的**不是** Idle（閒置）任務。
- **`pxTCB->uxPriority > pxCurrentTCBs[ xCurrentCoreID ]->uxPriority`**：新準備好的任務（`pxTCB`），其優先級**大於**當前核心正在跑的任務優先級。
##### 2.5.3 核心斷言檢查 (`configASSERT`)

```C
configASSERT( ( xYieldPendings[ xCurrentCoreID ] == pdTRUE ) ||
              ( taskTASK_IS_RUNNING( pxCurrentTCBs[ xCurrentCoreID ] ) == pdFALSE ) );
```
既然發現有更高優先級的任務醒了，根據排程規則，當前核心就必須做出應對。於是它用 `configASSERT(...)` 來檢查系統狀態是否符合預期：
它要求以下兩個條件**至少要有一個成立（OR 關係）**：
- **條件 A：`xYieldPendings[ xCurrentCoreID ] == pdTRUE`** 當前核心已經把自己的「引發禮讓（Yield）標記」設為 `True` 了。意思是：「我知道有高優先級任務來了，我已經排隊準備切換上下文（Context Switch）讓位了。」
- **條件 B：`taskTASK_IS_RUNNING(...) == pdFALSE`** 當前核心原本在跑的那個低優先級任務，目前已經變成「非運行狀態」了。意思是：「我已經成功把工作切換掉了，我現在沒在跑那個低優先級的任務了。」
#### 2.6 決策與執行 Yield

當迴圈跑完所有核心後，我們手上拿到了 `xLowestPriorityCore`（全場優先級最低的核心）：

```C
#if ( configRUN_MULTIPLE_PRIORITIES == 0 )
                // 如果前面沒有批量 Yield 任何核心 (xYieldCount == 0)，且有找到合適的目標核心
                if( ( xYieldCount == 0 ) && ( xLowestPriorityCore >= 0 ) )
            #else
                // 允許不同優先級的情況下，只要有找到目標核心就動手
                if( xLowestPriorityCore >= 0 )
            #endif
            {
                // 正式向該核心發送中斷/訊號，強制它引發上下文切換 (Context Switch)
                prvYieldCore( xLowestPriorityCore );
            }
```
#### 2.7 額外補充

##### 2.7.1 使用到的 macro

|**巨集名稱**|**說明**|
|---|---|
|`configNUMBER_OF_CORES`|系統運行的 CPU 核心總數（此函式僅在 `> 1` 即多核時生效）。|
|`configRUN_MULTIPLE_PRIORITIES`|`0` 表示**不允許**多個核心同時執行不同優先級的任務（所有核心必須盡量執行最高優先級任務）；`1` 表示**允許**。|
|`configUSE_CORE_AFFINITY`|核心親和性。`1` 表示任務可以綁定只能在某些特定核心運行。|
|`configUSE_TASK_PREEMPTION_DISABLE`|任務是否允許關閉「被強佔」的功能。|
##### 2.7.2 `mtCOVERAGE_TEST_MARKER()` 是什麼？
```C
else
{
    mtCOVERAGE_TEST_MARKER();
}
```
這是在生產環境（Production）中完全沒有功能的巨集。FreeRTOS 加上這個是為了**程式碼覆蓋率測試（Code Coverage Testing）**。在測試階段，這個巨集會被替換成特殊指令，用來確保測試案例有確實執行到 `else` 這個分支。在實際出貨時，它通常被定義為空，也會被編譯器完全忽略。

### 3. *prvSelectHighestPriorityTask*
(1005 to 1271 lines)
#### 3.1 核心目的
- 當某個 CPU 核心（`xCoreID`）需要進行上下文切換（Context Switch）時，它會呼叫這個函數來**幫自己挑選下一個最該執行的任務**。
- 當指定的 CPU 核心（`xCoreID`）需要更換任務時，從系統的「就緒任務鏈表（Ready Tasks Lists）」中，**由高到低**搜尋最適合該核心執行的任務，並將其掛載到該核心上。

#### 3.2 防止霸道任務的公平性修正 (Round-Robin Fix)

在進入尋找任務的迴圈前，函數會先處理一個多核心環境下的**排程公平性漏洞**：

當一個正在執行的任務主動呼叫 `taskYIELD()` 讓出 CPU 時，它依然處於就緒狀態。在多核心架構下，新創建的任務會被加到就緒列表的**末尾**。如果直接開始搜尋，這個剛剛讓權的任務會因為排在列表前面而**再度被選中**，導致新任務或其他等待更久的同優先權任務無法被公平分配到 CPU。 **解決方案**：若當前任務仍在就緒列表中，將其移出並重新插入到列表的**最末端**，確保公平性。
```C
        if( listIS_CONTAINED_WITHIN( &( pxReadyTasksLists[ pxCurrentTCBs[ xCoreID ]->uxPriority ] ),
                                     &pxCurrentTCBs[ xCoreID ]->xStateListItem ) == pdTRUE )
        {
            ( void ) uxListRemove( &pxCurrentTCBs[ xCoreID ]->xStateListItem );
            vListInsertEnd( &( pxReadyTasksLists[ pxCurrentTCBs[ xCoreID ]->uxPriority ] ),
                            &pxCurrentTCBs[ xCoreID ]->xStateListItem );
        }
```

#### 3.3 就緒列表搜尋迴圈

透過 `uxCurrentPriority` 從最高優先權往低處搜尋，直到找到可執行的任務（`xTaskScheduled == pdTRUE`）。
##### 3.3.1 限制單一優先權模式 

(configRUN_MULTIPLE_PRIORITIES == 0)
- 如果當前搜尋的優先權 `uxCurrentPriority` 已經低於全局最高就緒優先權 `uxTopReadyPriority`，代表此核心**無法再執行一般的常規任務**（因為其他核心正在執行更高優先權的任務）。
- 此時，強制將搜尋範圍直接降到最低的 `tskIDLE_PRIORITY`，且後續**只允許挑選真正的 Idle 任務**。

##### 3.3.2 檢查就緒任務列表

當發現某個優先權列表不為空時：

1. **鎖定最高優先權**：將 `xDecrementTopPriority` 設為 `pdFALSE`，避免全域的 `uxTopReadyPriority` 被錯誤調低。
2. **遍歷鏈結串列**：從 Head 開始走訪該優先權的所有任務控制塊（TCB）。
3. **狀態檢查與核心親和性過濾**：
	- **狀況一：任務未在任何核心運行 (`taskTASK_NOT_RUNNING`)**
		- 檢查核心親和性掩碼（Core Affinity Mask），若該任務允許在當前的 `xCoreID` 上執行，則進行**上下文切換 (Context Switch)**：
			1. 將當前核心舊任務狀態設為 `taskTASK_NOT_RUNNING`
			2. 將新任務狀態繫結為 `xCoreID`
			3. 更新全域指針 `pxCurrentTCBs[ xCoreID ] = pxTCB`
			4. 成功排程，退出迴圈
	 - 狀況二：任務本來就在此核心運行 (`pxTCB == pxCurrentTCBs[ xCoreID ]`)
		 - 保持現狀，確保狀態正確，成功排程並退出。
	 - 狀況三：任務正在其他核心運行
		 - 直接跳過（不可重複調度）。
```C
while( xTaskScheduled == pdFALSE )
{
    #if ( configRUN_MULTIPLE_PRIORITIES == 0 )
    {
        if( uxCurrentPriority < uxTopReadyPriority )
        {
            /* We can't schedule any tasks, other than idle, that have a
             * priority lower than the priority of a task currently running
             * on another core. */
            uxCurrentPriority = tskIDLE_PRIORITY;
        }
    }
    #endif

    if( listLIST_IS_EMPTY( &( pxReadyTasksLists[ uxCurrentPriority ] ) ) == pdFALSE )
    {
        const List_t * const pxReadyList = &( pxReadyTasksLists[ uxCurrentPriority ] );
        const ListItem_t * pxEndMarker = listGET_END_MARKER( pxReadyList );
        ListItem_t * pxIterator;

        /* The ready task list for uxCurrentPriority is not empty, so uxTopReadyPriority
         * must not be decremented any further. */
        xDecrementTopPriority = pdFALSE;

        for( pxIterator = listGET_HEAD_ENTRY( pxReadyList ); pxIterator != pxEndMarker; pxIterator = listGET_NEXT( pxIterator ) )
        {
            /* MISRA Ref 11.5.3 [Void pointer assignment] */
            /* More details at: https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/MISRA.md#rule-115 */
            /* coverity[misra_c_2012_rule_11_5_violation] */
            pxTCB = ( TCB_t * ) listGET_LIST_ITEM_OWNER( pxIterator );

            #if ( configRUN_MULTIPLE_PRIORITIES == 0 )
            {
                /* When falling back to the idle priority because only one priority
                 * level is allowed to run at a time, we should ONLY schedule the true
                 * idle tasks, not user tasks at the idle priority. */
                if( uxCurrentPriority < uxTopReadyPriority )
                {
                    if( ( pxTCB->uxTaskAttributes & taskATTRIBUTE_IS_IDLE ) == 0U )
                    {
                        continue;
                    }
                }
            }
            #endif /* #if ( configRUN_MULTIPLE_PRIORITIES == 0 ) */

            if( pxTCB->xTaskRunState == taskTASK_NOT_RUNNING )
            {
                #if ( configUSE_CORE_AFFINITY == 1 )
                    if( ( pxTCB->uxCoreAffinityMask & ( ( UBaseType_t ) 1U << ( UBaseType_t ) xCoreID ) ) != 0U )
                #endif
                {
                    /* If the task is not being executed by any core swap it in. */
                    pxCurrentTCBs[ xCoreID ]->xTaskRunState = taskTASK_NOT_RUNNING;
                    #if ( configUSE_CORE_AFFINITY == 1 )
                        pxPreviousTCB = pxCurrentTCBs[ xCoreID ];
                    #endif
                    pxTCB->xTaskRunState = xCoreID;
                    pxCurrentTCBs[ xCoreID ] = pxTCB;
                    xTaskScheduled = pdTRUE;
                }
            }
            else if( pxTCB == pxCurrentTCBs[ xCoreID ] )
            {
                configASSERT( ( pxTCB->xTaskRunState == xCoreID ) || ( pxTCB->xTaskRunState == taskTASK_SCHEDULED_TO_YIELD ) );

                #if ( configUSE_CORE_AFFINITY == 1 )
                    if( ( pxTCB->uxCoreAffinityMask & ( ( UBaseType_t ) 1U << ( UBaseType_t ) xCoreID ) ) != 0U )
                #endif
                {
                    /* The task is already running on this core, mark it as scheduled. */
                    pxTCB->xTaskRunState = xCoreID;
                    xTaskScheduled = pdTRUE;
                }
            }
            else
            {
                /* This task is running on the core other than xCoreID. */
                mtCOVERAGE_TEST_MARKER();
            }

            if( xTaskScheduled != pdFALSE )
            {
                /* A task has been selected to run on this core. */
                break;
            }
        }
    }
    else
    {
        if( xDecrementTopPriority != pdFALSE )
        {
            uxTopReadyPriority--;
            #if ( configRUN_MULTIPLE_PRIORITIES == 0 )
            {
                xPriorityDropped = pdTRUE;
            }
            #endif
        }
    }

    /* There are configNUMBER_OF_CORES Idle tasks created when scheduler started.
     * The scheduler should be able to select a task to run when uxCurrentPriority
     * is tskIDLE_PRIORITY. uxCurrentPriority is never decreased to value blow
     * tskIDLE_PRIORITY. */
    if( uxCurrentPriority > tskIDLE_PRIORITY )
    {
        uxCurrentPriority--;
    }
    else
    {
        /* This function is called when idle task is not created. Break the
         * loop to prevent uxCurrentPriority overrun. */
        break;
    }
}
```
##### 3.3.3 後置處理 A：多核心 Idle 任務觸發 
(`configRUN_MULTIPLE_PRIORITIES == 0`)

如果因為單一優先權限制，導致某個核心被迫調降優先權去執行 Idle 任務（`xPriorityDropped == pdTRUE`），函數最後會執行以下操作：

```C
for( x = ( BaseType_t ) 0; x < ( BaseType_t ) configNUMBER_OF_CORES; x++ )
{
    if( ( pxCurrentTCBs[ x ]->uxTaskAttributes & taskATTRIBUTE_IS_IDLE ) != 0U )
    {
        prvYieldCore( x );
    }
}
```

**目的**：當原本限制其他核心的高優先權任務結束後，必須主動通知（Yield）其他正在執行 Idle 任務的核心，讓它們重新評估排程，以便立刻換上新釋放出來的高優先權任務。

##### 3.3.4 後置處理 B：被驅逐任務的重排程優化
(`configUSE_CORE_AFFINITY == 1`)

```C
#if ( configUSE_CORE_AFFINITY == 1 )
{
    if( xTaskScheduled == pdTRUE )
    {
        if( ( pxPreviousTCB != NULL ) && ( listIS_CONTAINED_WITHIN( &( pxReadyTasksLists[ pxPreviousTCB->uxPriority ] ), &( pxPreviousTCB->xStateListItem ) ) != pdFALSE ) )
        {
            /* A ready task was just evicted from this core. See if it can be
             * scheduled on any other core. */
            UBaseType_t uxCoreMap = pxPreviousTCB->uxCoreAffinityMask;
            BaseType_t xLowestPriority = ( BaseType_t ) pxPreviousTCB->uxPriority;
            BaseType_t xLowestPriorityCore = -1;
            BaseType_t x;

            if( ( pxPreviousTCB->uxTaskAttributes & taskATTRIBUTE_IS_IDLE ) != 0U )
            {
                xLowestPriority = xLowestPriority - 1;
            }

            if( ( uxCoreMap & ( ( UBaseType_t ) 1U << ( UBaseType_t ) xCoreID ) ) != 0U )
            {
                /* pxPreviousTCB was removed from this core and this core is not excluded
                 * from it's core affinity mask.
                 *
                 * pxPreviousTCB is preempted by the new higher priority task
                 * pxCurrentTCBs[ xCoreID ]. When searching a new core for pxPreviousTCB,
                 * we do not need to look at the cores on which pxCurrentTCBs[ xCoreID ]
                 * is allowed to run. The reason is - when more than one cores are
                 * eligible for an incoming task, we preempt the core with the minimum
                 * priority task. Because this core (i.e. xCoreID) was preempted for
                 * pxCurrentTCBs[ xCoreID ], this means that all the others cores
                 * where pxCurrentTCBs[ xCoreID ] can run, are running tasks with priority
                 * no lower than pxPreviousTCB's priority. Therefore, the only cores where
                 * which can be preempted for pxPreviousTCB are the ones where
                 * pxCurrentTCBs[ xCoreID ] is not allowed to run (and obviously,
                 * pxPreviousTCB is allowed to run).
                 *
                 * This is an optimization which reduces the number of cores needed to be
                 * searched for pxPreviousTCB to run. */
                uxCoreMap &= ~( pxCurrentTCBs[ xCoreID ]->uxCoreAffinityMask );
            }
            else
            {
                /* pxPreviousTCB's core affinity mask is changed and it is no longer
                 * allowed to run on this core. Searching all the cores in pxPreviousTCB's
                 * new core affinity mask to find a core on which it can run. */
            }

            uxCoreMap &= ( ( 1U << configNUMBER_OF_CORES ) - 1U );

            for( x = ( ( BaseType_t ) configNUMBER_OF_CORES - 1 ); x >= ( BaseType_t ) 0; x-- )
            {
                UBaseType_t uxCore = ( UBaseType_t ) x;
                BaseType_t xTaskPriority;

                if( ( uxCoreMap & ( ( UBaseType_t ) 1U << uxCore ) ) != 0U )
                {
                    xTaskPriority = ( BaseType_t ) pxCurrentTCBs[ uxCore ]->uxPriority;

                    if( ( pxCurrentTCBs[ uxCore ]->uxTaskAttributes & taskATTRIBUTE_IS_IDLE ) != 0U )
                    {
                        xTaskPriority = xTaskPriority - ( BaseType_t ) 1;
                    }

                    uxCoreMap &= ~( ( UBaseType_t ) 1U << uxCore );

                    if( ( xTaskPriority < xLowestPriority ) &&
                        ( taskTASK_IS_RUNNING( pxCurrentTCBs[ uxCore ] ) != pdFALSE ) &&
                        ( xYieldPendings[ uxCore ] == pdFALSE ) )
                    {
                        #if ( configUSE_TASK_PREEMPTION_DISABLE == 1 )
                            if( pxCurrentTCBs[ uxCore ]->xPreemptionDisable == pdFALSE )
                        #endif
                        {
                            xLowestPriority = xTaskPriority;
                            xLowestPriorityCore = ( BaseType_t ) uxCore;
                        }
                    }
                }
            }

            if( xLowestPriorityCore >= 0 )
            {
                prvYieldCore( xLowestPriorityCore );
            }
        }
    }
}
#endif /* #if ( configUSE_CORE_AFFINITY == 1 ) */
```
當核心 `xCoreID` 選中了一個新任務，原本在該核心運行的舊任務（`pxPreviousTCB`）就被「驅逐（Evicted）」。FreeRTOS 會嘗試幫這個舊任務找尋其他可以容身的核心。

為了減少多核心搜尋的開銷，程式碼實作了一個精妙的遮罩過濾：
```C
if( ( uxCoreMap & ( ( UBaseType_t ) 1U << ( UBaseType_t ) xCoreID ) ) != 0U )
{
    uxCoreMap &= ~( pxCurrentTCBs[ xCoreID ]->uxCoreAffinityMask );
}
```

1. 舊任務（`pxPreviousTCB`）是因為新任務（`pxCurrentTCBs[xCoreID]`）優先權更高而被搶佔（Preempted）。
2. 根據 FreeRTOS 的排程原則：當初新任務要搶佔核心時，它必然已經挑選了所有可用核心中「執行任務優先權最低」的那一個。
3. 這代表：**凡是新任務可以執行的其他核心，目前運行的任務優先權都絕對高於（或等於）舊任務**。
4. **結論**：舊任務絕對不可能去搶佔那些「新任務也可以執行的核心」。因此，搜尋時直接用 `&= ~` 把新任務的親和性核心全部抹除，大幅縮小搜尋範圍！

### 4. *prvCreateStaticTask*

```C
#if ( configSUPPORT_STATIC_ALLOCATION == 1 )

    static TCB_t * prvCreateStaticTask( TaskFunction_t pxTaskCode,
                                        const char * const pcName,
                                        const configSTACK_DEPTH_TYPE uxStackDepth,
                                        void * const pvParameters,
                                        UBaseType_t uxPriority,
                                        StackType_t * const puxStackBuffer,
                                        StaticTask_t * const pxTaskBuffer,
                                        TaskHandle_t * const pxCreatedTask )
    {
        TCB_t * pxNewTCB;

        configASSERT( puxStackBuffer != NULL );
        configASSERT( pxTaskBuffer != NULL );

        #if ( configASSERT_DEFINED == 1 )
        {
            /* Sanity check that the size of the structure used to declare a
             * variable of type StaticTask_t equals the size of the real task
             * structure. */
            volatile size_t xSize = sizeof( StaticTask_t );
            configASSERT( xSize == sizeof( TCB_t ) );
            ( void ) xSize; /* Prevent unused variable warning when configASSERT() is not used. */
        }
        #endif /* configASSERT_DEFINED */

        if( ( pxTaskBuffer != NULL ) && ( puxStackBuffer != NULL ) )
        {
            /* The memory used for the task's TCB and stack are passed into this
             * function - use them. */
            /* MISRA Ref 11.3.1 [Misaligned access] */
            /* More details at: https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/MISRA.md#rule-113 */
            /* coverity[misra_c_2012_rule_11_3_violation] */
            pxNewTCB = ( TCB_t * ) pxTaskBuffer;
            ( void ) memset( ( void * ) pxNewTCB, 0x00, sizeof( TCB_t ) );
            pxNewTCB->pxStack = ( StackType_t * ) puxStackBuffer;

            #if ( tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE != 0 )
            {
                /* Tasks can be created statically or dynamically, so note this
                 * task was created statically in case the task is later deleted. */
                pxNewTCB->ucStaticallyAllocated = tskSTATICALLY_ALLOCATED_STACK_AND_TCB;
            }
            #endif /* tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE */

            prvInitialiseNewTask( pxTaskCode, pcName, uxStackDepth, pvParameters, uxPriority, pxCreatedTask, pxNewTCB, NULL );
        }
        else
        {
            pxNewTCB = NULL;
        }

        return pxNewTCB;
    }
    ...

#endif /* SUPPORT_STATIC_ALLOCATION */
```
- 這個函數 `prvCreateStaticTask` 是 FreeRTOS 用來**靜態建立任務**的核心內部函數。

- 與常見的 `xTaskCreate()`（動態配置，從 Heap 抓記憶體）不同，這個函數對應的是 `xTaskCreateStatic()`。它的核心特點是：**完全不使用動態記憶體配置（Zero Heap Allocation）**，任務所需的 TCB（任務控制塊）和 Stack（堆疊區）全部由使用者在外部配置好，再以指標傳入。

- 這在安全至上（如汽車、醫療）或記憶體受限、不允許發生 Heap 碎片化的嵌入式系統中非常關鍵。

- 這個函數內容較為簡單就只介紹幾個重點


#### 4.1 標記靜態配置標籤 (Memory Management Tag)

```C
#if ( tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE != 0 )
    {
        /* Tasks can be created statically or dynamically, so note this
         * task was created statically in case the task is later deleted. */
        pxNewTCB->ucStaticallyAllocated = tskSTATICALLY_ALLOCATED_STACK_AND_TCB;
    }
    #endif
```

##### 4.1.1 為什麼需要這個標籤？
如果系統中**同時允許**靜態與動態建立任務（`tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE` 成立），那麼當未來有人呼叫 `vTaskDelete()` 想要刪除這個任務時，核心必須知道「這傢伙是靜態建立的」。

##### 4.1.2 運作邏輯
有了 `tskSTATICALLY_ALLOCATED_STACK_AND_TCB` 這個標記，刪除任務時核心就知道**不應該**去呼叫 `vPortFree()` 釋放這塊記憶體（因為它是靜態的，釋放會導致 OS 崩潰），而是只要把它從就緒列表中移除即可。

#### 4.2 小重點
**資料隱藏（Information Hiding）的體現**：
FreeRTOS 為了不讓使用者直接碰到複雜的 `TCB_t` 結構（避免被亂改），對外只給 `StaticTask_t`。此函數內部才透過強制轉型還原，這種設計模式在 C 語言的大型專案（如 Linux Kernel、RTOS）中非常常見。

### 5. *xTaskCreateStatic*

這個函數 `xTaskCreateStatic` 是 FreeRTOS 開放給應用層（使用者）直接呼叫的 **API 核心進入點**。
職責：
1. 呼叫底層函數完成記憶體綁定與初始化。
2. **（多核心環境下）** 設定任務的預設核心親和性（Core Affinity）。
3. 將任務正式推入就緒列表（Ready List），讓排程器（Scheduler）看見它並開始執行。

```C
    TaskHandle_t xTaskCreateStatic( TaskFunction_t pxTaskCode,
                                    const char * const pcName,
                                    const configSTACK_DEPTH_TYPE uxStackDepth,
                                    void * const pvParameters,
                                    UBaseType_t uxPriority,
                                    StackType_t * const puxStackBuffer,
                                    StaticTask_t * const pxTaskBuffer )
    {
        TaskHandle_t xReturn = NULL;
        TCB_t * pxNewTCB;

        traceENTER_xTaskCreateStatic( pxTaskCode, pcName, uxStackDepth, pvParameters, uxPriority, puxStackBuffer, pxTaskBuffer );

        pxNewTCB = prvCreateStaticTask( pxTaskCode, pcName, uxStackDepth, pvParameters, uxPriority, puxStackBuffer, pxTaskBuffer, &xReturn );

        if( pxNewTCB != NULL )
        {
            #if ( ( configNUMBER_OF_CORES > 1 ) && ( configUSE_CORE_AFFINITY == 1 ) )
            {
                /* Set the task's affinity before scheduling it. */
                pxNewTCB->uxCoreAffinityMask = configTASK_DEFAULT_CORE_AFFINITY;
            }
            #endif

            prvAddNewTaskToReadyList( pxNewTCB );
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }

        traceRETURN_xTaskCreateStatic( xReturn );

        return xReturn;
    }
```
### 6. *xTaskCreateStaticAffinitySet*

和 `xTaskCreateStatic` 幾乎一樣，只是在`xTaskCreateStaticAffinitySet` 可以傳入自定義的 `uxCoreAffinityMask`，而非默認的 `configTASK_DEFAULT_CORE_AFFINITY`

```C
    #if ( ( configNUMBER_OF_CORES > 1 ) && ( configUSE_CORE_AFFINITY == 1 ) )
        TaskHandle_t xTaskCreateStaticAffinitySet( TaskFunction_t pxTaskCode,
                                                   const char * const pcName,
                                                   const configSTACK_DEPTH_TYPE uxStackDepth,
                                                   void * const pvParameters,
                                                   UBaseType_t uxPriority,
                                                   StackType_t * const puxStackBuffer,
                                                   StaticTask_t * const pxTaskBuffer,
                                                   UBaseType_t uxCoreAffinityMask )
        {
            TaskHandle_t xReturn = NULL;
            TCB_t * pxNewTCB;

            traceENTER_xTaskCreateStaticAffinitySet( pxTaskCode, pcName, uxStackDepth, pvParameters, uxPriority, puxStackBuffer, pxTaskBuffer, uxCoreAffinityMask );

            pxNewTCB = prvCreateStaticTask( pxTaskCode, pcName, uxStackDepth, pvParameters, uxPriority, puxStackBuffer, pxTaskBuffer, &xReturn );

            if( pxNewTCB != NULL )
            {
                /* Set the task's affinity before scheduling it. */
                pxNewTCB->uxCoreAffinityMask = uxCoreAffinityMask;

                prvAddNewTaskToReadyList( pxNewTCB );
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }

            traceRETURN_xTaskCreateStaticAffinitySet( xReturn );

            return xReturn;
        }
    #endif /* #if ( ( configNUMBER_OF_CORES > 1 ) && ( configUSE_CORE_AFFINITY == 1 ) ) */
```
### 7. *prvCreateRestrictedStaticTask*

這個函數 `prvCreateRestrictedStaticTask` 是 FreeRTOS 中結合了記憶體保護單元（MPU, Memory Protection Unit）**與**靜態配置（Static Allocation）的進階內部函數。

#### 7. 1 關鍵概念：什麼是「受限（Restricted）」任務？

在標準的 FreeRTOS 中，所有任務都可以存取整個微控制器的所有記憶體（Flash、RAM、周邊暫存器）。萬一某個任務發生指標越界（Wild Pointer），整台機器可能就會直接崩潰。

當啟用 MPU 時，任務會被分為：
- **特權任務（Privileged）**：可以存取所有空間（通常核心作業系統執行於此模式）。
- **受限/非特權任務（Restricted / Unprivileged）**：只能存取自己的 Stack、指定的 Flash，以及透過 `xRegions` 額外開放給它的特定記憶體區塊（例如某個硬體暫存器或全域緩衝區）。一旦企圖越界存取，硬體會立刻觸發 `MemManage_Handler` 異常，把受害者隔離，保護系統其他部分。

##### 7.1.1 *TaskParameters_t*
與之前看的 `prvCreateStaticTask` 帶了一大堆參數不同，這個函數只帶了兩個參數，因為所有的設定都被打包進了 `pxTaskDefinition`：

```C
static TCB_t * prvCreateRestrictedStaticTask( const TaskParameters_t * const pxTaskDefinition,
                                              TaskHandle_t * const pxCreatedTask )
```
為什麼要打包？（架構思維）
因為 MPU 任務除了傳統任務需要的參數外，還必須帶有核心存取權限、自定義的 MPU 區域陣列（`xRegions`）等大體積資訊。如果全當成函數參數傳遞，會造成暫存器與堆疊極大的傳遞負擔，因此 FreeRTOS 選擇用 *`TaskParameters_t`* 結構體一次打包傳入。
##### 7.1.2 與 *prvCreateStaticTask* 關鍵差異

- 注意看 `prvInitialiseNewTask` 的最後一個參數。
	- 在之前看的**普通靜態任務**中，最後一個參數是傳 `NULL`。
	- 在**受限靜態任務**中，這裡傳入了 `pxTaskDefinition->xRegions`。
- **後續連鎖反應**：這個 `xRegions`（通常包含 3 個可由使用者自訂的記憶體區域）會被存入 TCB 中。當排程器未來切換到這個任務執行時，硬體 MPU 會動態加載這些區域設定。這意味著，這個任務在執行時，眼睛只能看見這幾個特定「房間」，其他地方只要敢摸，硬體直接開槍（觸發 Memory Fault）。

```C
prvInitialiseNewTask( pxTaskDefinition->pvTaskCode,
                          pxTaskDefinition->pcName,
                          pxTaskDefinition->usStackDepth,
                          pxTaskDefinition->pvParameters,
                          pxTaskDefinition->uxPriority,
                          pxCreatedTask, pxNewTCB,
                          pxTaskDefinition->xRegions ); // <-- 重點在此！
}
```

### 8. *xTaskCreateRestrictedStatic*

這個函數 `xTaskCreateRestrictedStatic` 是 FreeRTOS 開放給使用者（應用層）呼叫的**公開 API 進入點**。

返回值：
- `pdPASS` (1)：任務建立成功，已進入排程準備執行。
- `errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY` (-1)：建立失敗（通常是因為傳入的 TCB 或 Stack 指標為空）。
- **錯誤回報**：如果靜態指標有問題導致 `pxNewTCB == NULL`，雖然這個函數叫「Static」不牽涉 Heap 釋放，但它依然會返回 `errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY`，主要是為了維持 FreeRTOS 家族 API 返回值的一致性。

```C
BaseType_t xTaskCreateRestrictedStatic( const TaskParameters_t * const pxTaskDefinition,
                                        TaskHandle_t * pxCreatedTask )
{
    TCB_t * pxNewTCB;
    BaseType_t xReturn;

    traceENTER_xTaskCreateRestrictedStatic( pxTaskDefinition, pxCreatedTask );

    configASSERT( pxTaskDefinition != NULL );

    pxNewTCB = prvCreateRestrictedStaticTask( pxTaskDefinition, pxCreatedTask );

    if( pxNewTCB != NULL )
    {
        #if ( ( configNUMBER_OF_CORES > 1 ) && ( configUSE_CORE_AFFINITY == 1 ) )
        {
            /* Set the task's affinity before scheduling it. */
            pxNewTCB->uxCoreAffinityMask = configTASK_DEFAULT_CORE_AFFINITY;
        }
        #endif

        prvAddNewTaskToReadyList( pxNewTCB );
        xReturn = pdPASS;
    }
    else
    {
        xReturn = errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY;
    }

    traceRETURN_xTaskCreateRestrictedStatic( xReturn );

    return xReturn;
}
```
### 9. xTaskCreateRestrictedStaticAffinitySet

和 *`xTaskCreateStaticAffinitySet`* 一樣概念只是是 `Restricted` 的

### 10. *prvCreateRestrictedTask*

這個函數 `prvCreateRestrictedTask` 是 FreeRTOS 中一個非常有趣的「混血兒」內部函數。

它在啟用 **MPU（記憶體保護單元）** 且支援 **動態配置（Dynamic Allocation）** 的環境下編譯。有趣的地方在於：它的 **TCB（任務控制塊）是從 Heap 動態配置的**，但它的 **Stack（堆疊）卻是使用者手動傳入的靜態/指定區塊**！

#### 10.1 為什麼 MPU 的動態任務，堆疊還要手動傳入？

這跟硬體 **MPU（記憶體保護單元）的對齊限制（Alignment Restrictions）** 有極大關係。

在許多常見的微控制器（例如 ARM Cortex-M4 MPU）中，MPU 保護的記憶體區塊大小必須是 **2 的冪次方**（例如 32、64、128... 1024 位元組），而且該區塊的**起始位址，必須與該區塊的大小對齊**。
- 如果完全交給動態記憶體配置 `pvPortMalloc()` 去抓 Stack 空間，Heap 吐出來的位址通常無法完美符合 MPU 的嚴格對齊要求。
- **解決方案**：FreeRTOS 規定，即使是動態建立受限任務，使用者也必須自己在外部透過編譯器語法（如 `__attribute__((aligned(1024)))`）宣告一個對齊好的 Stack，然後包在 `TaskParameters_t` 裡傳進來。而比較沒有對齊限制的 TCB，就放心地交給系統動態 `pvPortMalloc`。
 - 在程式碼中可以看到 `pxNewTCB->ucStaticallyAllocated` 被 assigned to `tskSTATICALLY_ALLOCATED_STACK_ONLY` 表示**只有堆疊是靜態配置的**。
 
```C
#if ( tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE != 0 )
    {
        /* Tasks can be created statically or dynamically, so note
         * this task had a statically allocated stack in case it is
         * later deleted.  The TCB was allocated dynamically. */
        pxNewTCB->ucStaticallyAllocated = tskSTATICALLY_ALLOCATED_STACK_ONLY;
    }
    #endif
```

### 11. *xTaskCreateRestricted*

類似於 *[[#8. xTaskCreateRestrictedStatic]]* ，只是內部是呼叫
*[[#10.prvCreateRestrictedTask]]* 因此不多贅述

### 12. *xTaskCreateRestrictedAffinitySet*


類似於 *[[#9. xTaskCreateRestrictedStaticAffinitySet]]*，只是內部是呼叫 *[[#10.prvCreateRestrictedTask]]* 因此不多贅述

### 13. *prvCreateTask*

這個函數 `prvCreateTask` 是 FreeRTOS 中最經典、最常被間接執行的「純動態配置任務」內部核心函數（對應使用者呼叫的 `xTaskCreate()`）。

當你不需要 MPU 限制、也不想自己手動宣告靜態記憶體時，這個函數會負責去 Heap（堆疊群）中把任務所需的 TCB 和 Stack **全部動態申請出來**。

#### 13.1 *prvCreateTask 角色定位*

- **編譯條件**：必須啟用動態配置（`configSUPPORT_DYNAMIC_ALLOCATION == 1`）。
- **核心職責**：同時從 Heap 中配置 TCB 與 Stack 的記憶體空間，並根據硬體架構的「堆疊生長方向」調整配置順序，最後初始化任務。

#### 13.2 堆疊生長方向與配置順序

- 這段函數最耐人尋味的是它用 `#if ( portSTACK_GROWTH > 0 )` 拆成了兩套完全相反的記憶體配置順序。
- **為什麼要看堆疊生長方向（Stack Growth）來決定誰先 `malloc`？** 這是為了防止**堆疊溢位（Stack Overflow）時直接摧毀 TCB 結構**體！ 在動態配置中，連續呼叫兩次 `pvPortMalloc` 拿到的記憶體區塊，在 Heap 裡通常是**前後緊鄰**的。
##### 13.3 狀況一：堆疊向上生長 
(`portSTACK_GROWTH > 0`)
- **生長特點**：記憶體位址越用越大。
- **配置順序**：**先配 TCB，後配 Stack**。
- **防禦邏輯**：此時 Stack 的位址會大於 TCB 的位址（Stack 在 TCB 的「上方」）。當任務執行過程中堆疊爆掉（溢位）時，它是往位址更大的地方（上方）衝過去，**不會**回頭踩死位址比較小的 TCB 數據。
##### 13.4 狀況二：堆疊向下生長
(`portSTACK_GROWTH < 0`，絕大多數微控制器如 ARM Cortex-M 都是這種)
- **生長特點**：記憶體位址越用越小。
- **配置順序**：**先配 Stack，後配 TCB**。
- **防禦邏輯**：此時 TCB 的位址會大於 Stack 的位址（TCB 在 Stack 的「上方」）。當堆疊爆掉（溢位）時，它是往位址更小的地方（下方）衝過去，**同樣不會**衝上去踩死位址較大的 TCB。

```C
#if ( configSUPPORT_DYNAMIC_ALLOCATION == 1 )
    static TCB_t * prvCreateTask( TaskFunction_t pxTaskCode,
                                  const char * const pcName,
                                  const configSTACK_DEPTH_TYPE uxStackDepth,
                                  void * const pvParameters,
                                  UBaseType_t uxPriority,
                                  TaskHandle_t * const pxCreatedTask )
    {
        TCB_t * pxNewTCB;

        /* If the stack grows down then allocate the stack then the TCB so the stack
         * does not grow into the TCB.  Likewise if the stack grows up then allocate
         * the TCB then the stack. */
        #if ( portSTACK_GROWTH > 0 )
        {
            /* Allocate space for the TCB.  Where the memory comes from depends on
             * the implementation of the port malloc function and whether or not static
             * allocation is being used. */
            /* MISRA Ref 11.5.1 [Malloc memory assignment] */
            /* More details at: https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/MISRA.md#rule-115 */
            /* coverity[misra_c_2012_rule_11_5_violation] */
            pxNewTCB = ( TCB_t * ) pvPortMalloc( sizeof( TCB_t ) );

            if( pxNewTCB != NULL )
            {
                ( void ) memset( ( void * ) pxNewTCB, 0x00, sizeof( TCB_t ) );

                /* Allocate space for the stack used by the task being created.
                 * The base of the stack memory stored in the TCB so the task can
                 * be deleted later if required. */
                /* MISRA Ref 11.5.1 [Malloc memory assignment] */
                /* More details at: https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/MISRA.md#rule-115 */
                /* coverity[misra_c_2012_rule_11_5_violation] */
                pxNewTCB->pxStack = ( StackType_t * ) pvPortMallocStack( ( ( ( size_t ) uxStackDepth ) * sizeof( StackType_t ) ) );

                if( pxNewTCB->pxStack == NULL )
                {
                    /* Could not allocate the stack.  Delete the allocated TCB. */
                    vPortFree( pxNewTCB );
                    pxNewTCB = NULL;
                }
            }
        }
        #else /* portSTACK_GROWTH */
        {
            StackType_t * pxStack;

            /* Allocate space for the stack used by the task being created. */
            /* MISRA Ref 11.5.1 [Malloc memory assignment] */
            /* More details at: https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/MISRA.md#rule-115 */
            /* coverity[misra_c_2012_rule_11_5_violation] */
            pxStack = ( StackType_t * ) pvPortMallocStack( ( ( ( size_t ) uxStackDepth ) * sizeof( StackType_t ) ) );

            if( pxStack != NULL )
            {
                /* Allocate space for the TCB. */
                /* MISRA Ref 11.5.1 [Malloc memory assignment] */
                /* More details at: https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/MISRA.md#rule-115 */
                /* coverity[misra_c_2012_rule_11_5_violation] */
                pxNewTCB = ( TCB_t * ) pvPortMalloc( sizeof( TCB_t ) );

                if( pxNewTCB != NULL )
                {
                    ( void ) memset( ( void * ) pxNewTCB, 0x00, sizeof( TCB_t ) );

                    /* Store the stack location
```

### 14. *xTaskCreate*

類似於 *[[#8. xTaskCreateRestrictedStatic]]* ，只是內部是呼叫 *[[#13. prvCreateTask]]* 因此不多贅述

### 15. *xTaskCreateAffinitySet*

類似於 *[[#9. xTaskCreateRestrictedStaticAffinitySet]]*，只是內部是呼叫 *[[#13. prvCreateTask]]* 因此不多贅述

### 16. *prvInitialiseNewTask*

其核心任務是**初始化一個新建立的任務**（不論是動態還是靜態配置）。它負責設定任務控制區塊（TCB）、計算並對齊堆疊、設定優先權，並**偽造一個中斷現場**，好讓排程器（Scheduler）第一次切換到該任務時能順利啟動。

#### 16.1 MPU 特權模式解析與堆疊初始化填滿

```C
#if ( portUSING_MPU_WRAPPERS == 1 )
    if( ( uxPriority & portPRIVILEGE_BIT ) != 0U )
    {
        xRunPrivileged = pdTRUE;
    }
    ...
    uxPriority &= ~portPRIVILEGE_BIT;
#endif

#if ( tskSET_NEW_STACKS_TO_KNOWN_VALUE == 1 )
    ( void ) memset( pxNewTCB->pxStack, ( int ) tskSTACK_FILL_BYTE, ... );
#endif
```

- **特權級別提取**：在 MPU 環境下，FreeRTOS 會把「這個任務是否具備特權（Privileged）」的資訊，藏在優先權欄位的最高位元（`portPRIVILEGE_BIT`）。這裡會把它提取出來存進 `xRunPrivileged`，並將優先權還原。
- **魔術字串填滿**：如果啟用了除錯機制，它會用 `0xa5`（`tskSTACK_FILL_BYTE`）把整塊 Stack 填滿。這樣未來可以用來觀測這個任務的**堆疊歷史最高使用量（Stack High Water Mark）**。
#### 16.2 精密計算與對齊：找出「堆疊頂端 (Top of Stack)」

因為不同 CPU 架構（如 ARM、x86）的堆疊生長方向截然不同，此處必須精準算出第一個要寫入的記憶體位址：

```C
#if ( portSTACK_GROWTH < 0 )
    pxTopOfStack = &( pxNewTCB->pxStack[ uxStackDepth - ( configSTACK_DEPTH_TYPE ) 1 ] );
    pxTopOfStack = ( StackType_t * ) ( ( ( portPOINTER_SIZE_TYPE ) pxTopOfStack ) & ( ~( ( portPOINTER_SIZE_TYPE ) portBYTE_ALIGNMENT_MASK ) ) );
```
- **向下生長（如 ARM Cortex-M）**：堆疊頂端在記憶體的**最高位址**。所以它拿陣列的最後一個元素 `uxStackDepth - 1` 作為起點。
- **記憶體對齊（微秒必爭的關鍵）**：利用 `& (~portBYTE_ALIGNMENT_MASK)` 進行向下對齊（例如 8 位元組對齊）。這非常重要，因為許多硬體架構（如 ARM）如果硬要在未對齊的位址上進行 64 位元的暫存器推疊（Push/Pop），會直接觸發硬體異常（Alignment Fault）。
#### 16.3 安全複製任務名稱

將字串格式的任務名稱複製到 TCB 的 `pcTaskName` 陣列中。

```C

/* Store the task name in the TCB. */
    if( pcName != NULL )
    {
        for( x = ( UBaseType_t ) 0; x < ( UBaseType_t ) configMAX_TASK_NAME_LEN; x++ )
        {
            pxNewTCB->pcTaskName[ x ] = pcName[ x ];

            /* Don't copy all configMAX_TASK_NAME_LEN if the string is shorter than
             * configMAX_TASK_NAME_LEN characters just in case the memory after the
             * string is not accessible (extremely unlikely). */
            if( pcName[ x ] == ( char ) 0x00 )
            {
                break;
            }
            else
            {
                幕後測試標記
                mtCOVERAGE_TEST_MARKER();
            }
        }

        /* Ensure the name string is terminated in the case that the string length
         * was greater or equal to configMAX_TASK_NAME_LEN. */
        pxNewTCB->pcTaskName[ configMAX_TASK_NAME_LEN - 1U ] = '\0';
    }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }
```

#### 16.4 初始化任務優先權

驗證並設定任務的優先權。若啟用互斥鎖（Mutex），還需初始化基礎優先權。

**優先權繼承機制預留**：`uxBasePriority` 用於記錄任務最初的優先權。當任務因為「互斥鎖」引發優先權翻轉（Priority Inversion）而暫時被提權時，系統後續能透過 `uxBasePriority` 降回原本的優先權。

```C
/* This is used as an array index so must ensure it's not too large. */
    configASSERT( uxPriority < configMAX_PRIORITIES );

    if( uxPriority >= ( UBaseType_t ) configMAX_PRIORITIES )
    {
        uxPriority = ( UBaseType_t ) configMAX_PRIORITIES - ( UBaseType_t ) 1U;
    }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }

    pxNewTCB->uxPriority = uxPriority;
    #if ( configUSE_MUTEXES == 1 )
    {
        pxNewTCB->uxBasePriority = uxPriority;
    }
    #endif /* configUSE_MUTEXES */
```

#### 16.5 初始化任務鏈結串列節點（List Items）

FreeRTOS 的任務調度本質上是核心鏈結串列（List）的操作。此處會初始化 TCB 內嵌的兩個核心鏈結串列節點：狀態節點與事件節點。

```C
vListInitialiseItem( &( pxNewTCB->xStateListItem ) );
    vListInitialiseItem( &( pxNewTCB->xEventListItem ) );

    /* Set the pxNewTCB as a link back from the ListItem_t.  This is so we can get
     * back to  the containing TCB from a generic item in a list. */
    listSET_LIST_ITEM_OWNER( &( pxNewTCB->xStateListItem ), pxNewTCB );

    /* Event lists are always in priority order. */
    listSET_LIST_ITEM_VALUE( &( pxNewTCB->xEventListItem ), ( TickType_t ) configMAX_PRIORITIES - ( TickType_t ) uxPriority );
    listSET_LIST_ITEM_OWNER( &( pxNewTCB->xEventListItem ), pxNewTCB );
```

- **擁有者設定 (`listSET_LIST_ITEM_OWNER`)**：將節點的擁有者指向 TCB 自身。如此一來，排程器在遍歷就緒鏈結串列（Ready List）時，只要拿到節點指標，就能反向推導出對應的 TCB 記憶體位址。
- **事件鏈結串列排序權重**：事件鏈結串列（如等待信號量或佇列的鏈結串列）是**依優先權高低排序**的。核心透過 `configMAX_PRIORITIES - uxPriority` 來計算排序值，確保優先權越高的任務排在越前面。

#### 16.6 MPU 與 TLS 執行緒區域儲存初始化

設定 MPU 記憶體區塊，並在啟用 C 語言執行時期 TLS 支持時初始化 TLS 區塊。

```C
#if ( portUSING_MPU_WRAPPERS == 1 )
    {
        vPortStoreTaskMPUSettings( &( pxNewTCB->xMPUSettings ), xRegions, pxNewTCB->pxStack, uxStackDepth );
    }
    #else
    {
        /* Avoid compiler warning about unreferenced parameter. */
        ( void ) xRegions;
    }
    #endif

    #if ( configUSE_C_RUNTIME_TLS_SUPPORT == 1 )
    {
        /* Allocate and initialize memory for the task's TLS Block. */
        configINIT_TLS_BLOCK( pxNewTCB->xTLSBlock, pxTopOfStack );
    }
    #endif
```

#### 16.7 偽造中斷堆疊現場

這是整個初始化過程中最關鍵的一步。FreeRTOS 會在這裡呼叫硬體移植層（Port）的 `pxPortInitialiseStack`。

```C

/* Initialize the TCB stack to look as if the task was already running,
     * but had been interrupted by the scheduler.  The return address is set
     * to the start of the task function. Once the stack has been initialised
     * the top of stack variable is updated. */
    #if ( portUSING_MPU_WRAPPERS == 1 )
    {
        #if ( portHAS_STACK_OVERFLOW_CHECKING == 1 )
        {
            #if ( portSTACK_GROWTH < 0 )
            {
                pxNewTCB->pxTopOfStack = pxPortInitialiseStack( pxTopOfStack, pxNewTCB->pxStack, pxTaskCode, pvParameters, xRunPrivileged, &( pxNewTCB->xMPUSettings ) );
            }
            #else /* portSTACK_GROWTH */
            {
                pxNewTCB->pxTopOfStack = pxPortInitialiseStack( pxTopOfStack, pxNewTCB->pxEndOfStack, pxTaskCode, pvParameters, xRunPrivileged, &( pxNewTCB->xMPUSettings ) );
            }
            #endif /* portSTACK_GROWTH */
        }
        #else /* portHAS_STACK_OVERFLOW_CHECKING */
        {
            pxNewTCB->pxTopOfStack = pxPortInitialiseStack( pxTopOfStack, pxTaskCode, pvParameters, xRunPrivileged, &( pxNewTCB->xMPUSettings ) );
        }
        #endif /* portHAS_STACK_OVERFLOW_CHECKING */
    }
    #else /* portUSING_MPU_WRAPPERS */
    {
        #if ( portHAS_STACK_OVERFLOW_CHECKING == 1 )
        {
            #if ( portSTACK_GROWTH < 0 )
            {
                pxNewTCB->pxTopOfStack = pxPortInitialiseStack( pxTopOfStack, pxNewTCB->pxStack, pxTaskCode, pvParameters );
            }
            #else /* portSTACK_GROWTH */
            {
                pxNewTCB->pxTopOfStack = pxPortInitialiseStack( pxTopOfStack, pxNewTCB->pxEndOfStack, pxTaskCode, pvParameters );
            }
            #endif /* portSTACK_GROWTH */
        }
        #else /* portHAS_STACK_OVERFLOW_CHECKING */
        {
            pxNewTCB->pxTopOfStack = pxPortInitialiseStack( pxTopOfStack, pxTaskCode, pvParameters );
        }
        #endif /* portHAS_STACK_OVERFLOW_CHECKING */

        #if ( portSTACK_GROWTH < 0 )
        {
            configASSERT( ( ( portPOINTER_SIZE_TYPE ) ( pxTopOfStack - pxNewTCB->pxTopOfStack ) ) < ( ( portPOINTER_SIZE_TYPE ) uxStackDepth ) );
        }
        #else /* portSTACK_GROWTH */
        {
            configASSERT( ( ( portPOINTER_SIZE_TYPE ) ( pxNewTCB->pxTopOfStack - pxTopOfStack ) ) < ( ( portPOINTER_SIZE_TYPE ) uxStackDepth ) );
        }
        #endif /* portSTACK_GROWTH */
    }
    #endif /* portUSING_MPU_WRAPPERS */

```

- **運作原理**：排程器進行任務切換時，是透過「彈出堆疊暫存器（Pop Registers）」來還原上一個任務的現場。`pxPortInitialiseStack` 的工作就是**在堆疊裡人工填入偽造的暫存器數值**。
- 最後，將計算完成後的真實堆疊指針寫回 `pxNewTCB->pxTopOfStack`。

### 17. *prvAddNewTaskToReadyList*

這段 FreeRTOS 核心原始碼程式碼是 `prvAddNewTaskToReadyList`。它的核心任務非常明確：**當系統成功建立一個新工作（Task）後，負責把這個工作的 TCB（工作控制區塊）正式註冊到系統的「就緒鏈結串列（Ready List）」中，讓排程器（Scheduler）可以看得到它並開始調度。**

#### `pxCurrentTCB` 的設定時機

- Single Core: 如果排程器尚未啟動（`xSchedulerRunning == pdFALSE`），系統會不斷比較新舊工作的優先權。誰的優先權高，`pxCurrentTCB` 指標就指向誰。這樣當排程器一啟動，就能**一槍命中**直接執行最高優先權的工作。
- SMP: 完全不碰 `pxCurrentTCB`。因為多核心系統啟動時，各個核心必須先綁定各自的 `IdleTask`，其分派邏輯是在 `prvCreateIdleTasks()` 中完成的，因此初始化新工作時不需要、也不能提前鎖定單一的執行指針。

#### 為什麼搶佔（Yield）巨集放的位置不一樣？

- Single Core (放 CS 外面)
	- 在單核心中，`taskYIELD_ANY_CORE_IF_USING_PREEMPTION` 通常會觸發一個軟體中斷（如 ARM 的 PendSV）。如果把觸發中斷的動作放在臨界區（關中斷）內部，中斷不會立刻響應，反而可能導致不必要的硬體巢狀判斷。放在外面，一出臨界區（開中斷），硬體就能**立刻切換上下文**，效率最高。
- SMP  (放 CS 裡面)
	 - 多核心的搶佔往往涉及「跨核心中斷（IPI，Inter-Processor Interrupt）」**。當 A 核心建立了一個超高優先權的工作，它需要評估 B 核心、C 核心目前執行的工作優先權，並挑選一個倒楣鬼核心發送中斷叫它讓位。 這整個「評估各核心狀態 $\rightarrow$ 挑選核心 $\rightarrow$ 發送 IPI」的過程必須是**絕對原子性（Atomic）**的。如果放在臨界區外面，在評估的瞬間其他核心可能已經自己切換了工作，會導致 IPI 發錯對象。因此，SMP 架構下必須在**鎖保護內完成 Yield 宣告。
	
```C
#if ( configNUMBER_OF_CORES == 1 )

    static void prvAddNewTaskToReadyList( TCB_t * pxNewTCB )
    {
        /* Ensure interrupts don't access the task lists while the lists are being
         * updated. */
        taskENTER_CRITICAL();
        {
            uxCurrentNumberOfTasks = ( UBaseType_t ) ( uxCurrentNumberOfTasks + 1U );

            if( pxCurrentTCB == NULL )
            {
                /* There are no other tasks, or all the other tasks are in
                 * the suspended state - make this the current task. */
                pxCurrentTCB = pxNewTCB;

                if( uxCurrentNumberOfTasks == ( UBaseType_t ) 1 )
                {
                    /* This is the first task to be created so do the preliminary
                     * initialisation required.  We will not recover if this call
                     * fails, but we will report the failure. */
                    prvInitialiseTaskLists();
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
            else
            {
                /* If the scheduler is not already running, make this task the
                 * current task if it is the highest priority task to be created
                 * so far. */
                if( xSchedulerRunning == pdFALSE )
                {
                    if( pxCurrentTCB->uxPriority <= pxNewTCB->uxPriority )
                    {
                        pxCurrentTCB = pxNewTCB;
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }

            uxTaskNumber++;

            #if ( configUSE_TRACE_FACILITY == 1 )
            {
                /* Add a counter into the TCB for tracing only. */
                pxNewTCB->uxTCBNumber = uxTaskNumber;
            }
            #endif /* configUSE_TRACE_FACILITY */
            traceTASK_CREATE( pxNewTCB );

            prvAddTaskToReadyList( pxNewTCB );

            portSETUP_TCB( pxNewTCB );
        }
        taskEXIT_CRITICAL();

        if( xSchedulerRunning != pdFALSE )
        {
            /* If the created task is of a higher priority than the current task
             * then it should run now. */
            taskYIELD_ANY_CORE_IF_USING_PREEMPTION( pxNewTCB );
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
    }

#else /* #if ( configNUMBER_OF_CORES == 1 ) */

    static void prvAddNewTaskToReadyList( TCB_t * pxNewTCB )
    {
        /* Ensure interrupts don't access the task lists while the lists are being
         * updated. */
        taskENTER_CRITICAL();
        {
            uxCurrentNumberOfTasks++;

            if( xSchedulerRunning == pdFALSE )
            {
                if( uxCurrentNumberOfTasks == ( UBaseType_t ) 1 )
                {
                    /* This is the first task to be created so do the preliminary
                     * initialisation required.  We will not recover if this call
                     * fails, but we will report the failure. */
                    prvInitialiseTaskLists();
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }

                /* All the cores start with idle tasks before the SMP scheduler
                 * is running. Idle tasks are assigned to cores when they are
                 * created in prvCreateIdleTasks(). */
            }

            uxTaskNumber++;

            #if ( configUSE_TRACE_FACILITY == 1 )
            {
                /* Add a counter into the TCB for tracing only. */
                pxNewTCB->uxTCBNumber = uxTaskNumber;
            }
            #endif /* configUSE_TRACE_FACILITY */
            traceTASK_CREATE( pxNewTCB );

            prvAddTaskToReadyList( pxNewTCB );

            portSETUP_TCB( pxNewTCB );

            if( xSchedulerRunning != pdFALSE )
            {
                /* If the created task is of a higher priority than another
                 * currently running task and preemption is on then it should
                 * run now. */
                taskYIELD_ANY_CORE_IF_USING_PREEMPTION( pxNewTCB );
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
        taskEXIT_CRITICAL();
    }

#endif /* #if ( configNUMBER_OF_CORES == 1 ) */
```

### 18. *prvSnprintfReturnValueToCharsWritten*

在標準 C 語言中，`snprintf(buffer, n, ...)` 的安全機制是：不論格式化後的字串有多長，它最多只會寫入 `n - 1` 個字元，並在最後補上 `\0`，確保記憶體不溢位。

然而，**`snprintf` 的回傳值並不是「實際寫入 buffer 的字元數」**，而是「如果 buffer 夠大，它『預期』寫入的總字元數」。

- 如果 buffer 大小 `n = 10`，但你想印出 15 個字元，`snprintf` 會回傳 **15**，但 buffer 裡其實只被寫入了 **9** 個字元（外加一個 `\0`）。
    
FreeRTOS 需要精確知道**到底有多少字元被真正寫入到記憶體中**，以便正確移動下一次寫入的指標。這個函數就是用來校正這個數值的。

#### 18.1 條件編譯與函數定義

```C
#if ( ( configUSE_TRACE_FACILITY == 1 ) && ( configUSE_STATS_FORMATTING_FUNCTIONS > 0 ) )

    static size_t prvSnprintfReturnValueToCharsWritten( int iSnprintfReturnValue,
                                                        size_t n )
    {
        size_t uxCharsWritten;
```
- `iSnprintfReturnValue`：剛剛呼叫 `snprintf` 所得到的回傳值。
- `n`：當初傳給 `snprintf` 的安全緩衝區總大小（Buffer Size）。

#### 18.2 編碼錯誤處理（Encoding Error）

```C
if( iSnprintfReturnValue < 0 )
        {
            /* Encoding error - Return 0 to indicate that nothing
             * was written to the buffer. */
            uxCharsWritten = 0;
        }
```

**核心邏輯**：根據 C 標準，如果 `snprintf` 遇到編碼錯誤（例如寬字元轉換失敗），它會回傳**負數**。此時代表沒有任何有效文字被寫入，因此將實際寫入字數 `uxCharsWritten` 設為 `0`。

#### 18.3 緩衝區溢位與字串截斷（Truncation）

```C
else if( iSnprintfReturnValue >= ( int ) n )
        {
            /* This is the case when the supplied buffer is not
             * large to hold the generated string. Return the
             * number of characters actually written without
             * counting the terminating NULL character. */
            uxCharsWritten = n - 1U;
        }
```
- **核心邏輯**：當預期寫入的長度 `iSnprintfReturnValue` **大於或等於** 緩衝區總大小 `n` 時，代表發生了截斷（Buffer 太小裝不下）。
- **校正計算**：因為 `snprintf` 會強行在 buffer 的最後一個位置補上 `\0`，所以實際被寫入的有效字元數必定是 **`n - 1`**。

#### 18.4 完美寫入（Success）

字串長度小於緩衝區大小的正常狀態。

```C
else
        {
            /* Complete string was written to the buffer. */
            uxCharsWritten = ( size_t ) iSnprintfReturnValue;
        }

        return uxCharsWritten;
    }

#endif /* configUSE_STATS_FORMATTING_FUNCTIONS */
```

- **核心邏輯**：如果回傳值大於等於 0 且小於 `n`，代表整個字串都被完整、安全地寫入進緩衝區了。此時 `snprintf` 的回傳值就是最真實的實際寫入字數，直接轉換成 `size_t` 型態回傳即可。

### 19. vTaskDelete

這段原始碼是 FreeRTOS 的 `vTaskDelete()` 函數，其核心任務是**將指定的任務從系統中徹底刪除**。
在深入分析程式碼之前，你必須先理解 FreeRTOS 刪除任務的**核心哲學：誰來釋放記憶體？**

1. **刪除別人（其他任務）**：可以直接在當前釋放該任務的 TCB 和堆疊記憶體。
2. **自殺（刪除自己）**：任務無法在執行中「砍掉自己腳下的立足點」（不能釋放目前正在使用的堆疊）。因此，它必須把記憶體釋放的遺願交給 **Idle 任務（空閒任務）** 來非同步清理。

#### 19.1 函數入口與基礎宣告

```C
#if ( INCLUDE_vTaskDelete == 1 )

    void vTaskDelete( TaskHandle_t xTaskToDelete )
    {
        TCB_t * pxTCB;
        BaseType_t xDeleteTCBInIdleTask = pdFALSE; // 標記是否交由 Idle 任務進行記憶體清理
        BaseType_t xTaskIsRunningOrYielding;

        traceENTER_vTaskDelete( xTaskToDelete );

        taskENTER_CRITICAL();
        {
            /* 如果傳入 NULL，代表目前執行的任務想要刪除自己 */
            pxTCB = prvGetTCBFromHandle( xTaskToDelete );
            configASSERT( pxTCB != NULL );
```

#### 19.2 從狀態鏈結串列與事件鏈結串列中移除

不論任務是死是活，都要先從排程器的鏈結串列中抹去。

```C
/* 1. 從狀態鏈結串列（Ready/Delayed/Suspended List）中移除 */
            if( uxListRemove( &( pxTCB->xStateListItem ) ) == ( UBaseType_t ) 0 )
            {
                /* 如果移除後，該優先權的 Ready 鏈結串列變空了，就清除排程器的優先權位元圖標記 */
                taskRESET_READY_PRIORITY( pxTCB->uxPriority );
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }

            /* 2. 檢查此工作是否同時在等待某個事件（例如 Queue、Semaphore 或 Event Group） */
            if( listLIST_ITEM_CONTAINER( &( pxTCB->xEventListItem ) ) != NULL )
            {
                /* 如果是，也將其從事件鏈結串列中拔除 */
                ( void ) uxListRemove( &( pxTCB->xEventListItem ) );
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }

            /* 遞增工作流水號，通知核心偵錯器（如 OpenOCD）工作鏈結串列已改變，需要重新整理 */
            uxTaskNumber++;
```

#### 19.3 處理「運行中」或「自殺」任務：放入終止鏈結串列

如果被刪除的工作此時此刻正在某個核心上運作（或是正準備让出 CPU），我們**不能立即釋放它的記憶體**。

```C
/* 檢查工作是否正在執行，或者已經被標記為要 Yield */
            xTaskIsRunningOrYielding = taskTASK_IS_RUNNING_OR_SCHEDULED_TO_YIELD( pxTCB );

            if( ( xSchedulerRunning != pdFALSE ) && ( xTaskIsRunningOrYielding != pdFALSE ) )
            {
                /* 核心魔術：不能當場釋放記憶體！將 TCB 塞進「等待終止鏈結串列」 */
                vListInsertEnd( &xTasksWaitingTermination, &( pxTCB->xStateListItem ) );

                /* 增加計數，告訴 Idle 工作：有死掉的冤魂等著你來超渡（清理記憶體） */
                ++uxDeletedTasksWaitingCleanUp;

                traceTASK_DELETE( pxTCB );

                /* 標記：記憶體清理交給 Idle 任務 */
                xDeleteTCBInIdleTask = pdTRUE;

                /* 呼叫預刪除鉤子函數（主要給 Windows 模擬器使用） */
                #if ( configNUMBER_OF_CORES == 1 )
                    portPRE_TASK_DELETE_HOOK( pxTCB, &( xYieldPendings[ 0 ] ) );
                #else
                    portPRE_TASK_DELETE_HOOK( pxTCB, &( xYieldPendings[ pxTCB->xTaskRunState ] ) );
                #endif
```

#### 19.4 多核心（SMP）環境下的強制驅逐（Eviction）

在多核心環境下，可能發生 **「A 核心上的工作，下指令刪除正在 B 核心上爽爽執行的工作」**。為了防止被刪除的工作繼續亂動，必須立刻將它驅逐。

```C
#if ( configNUMBER_OF_CORES > 1 )
                {
                    /* 如果這個被刪除的工作正在某個核心上執行 */
                    if( taskTASK_IS_RUNNING( pxTCB ) == pdTRUE )
                    {
                        /* 如果它是在「當前核心」執行（也就是自殺） */
                        if( pxTCB->xTaskRunState == ( BaseType_t ) portGET_CORE_ID() )
                        {
                            configASSERT( uxSchedulerSuspended == 0 );
                            /* 當場引發上下文切換，交出主導權 */
                            taskYIELD_WITHIN_API();
                        }
                        else
                        {
                            /* 如果是在「別人的核心」執行，對那個核心發送跨核心中斷（IPI），強制它踢掉該工作 */
                            prvYieldCore( pxTCB->xTaskRunState );
                        }
                    }
                }
                #endif /* #if ( configNUMBER_OF_CORES > 1 ) */
            }
```

#### 19.5 處理「非運行中」任務的刪除

如果要刪除的工作目前處於 Blocked（阻塞）或 Suspended（掛起）狀態（也就是躺著沒在動），處理起來最簡單。

```C
else
            {
                /* 既然工作沒在執行，可以直接把全域工作總數減 1 */
                --uxCurrentNumberOfTasks;
                traceTASK_DELETE( pxTCB );

                /* 重設下一次排程器該喚醒工作的時間點，以防剛才刪掉的工作正是下一個要醒來的工作 */
                prvResetNextTaskUnblockTime();
            }
        }
        taskEXIT_CRITICAL(); // 離開臨界區
```

#### 19.6 記憶體真實釋放與強制排程

出了臨界區後，根據先前的標記決定要不要當場動手術釋放記憶體。

```C
/* 如果不需要交給 Idle（代表刪除的是非執行中工作），就在這裡當場釋放 TCB 和 Stack 記憶體 */
        if( xDeleteTCBInIdleTask != pdTRUE )
        {
            prvDeleteTCB( pxTCB );
        }

        /* 單核心下：如果任務是刪除自己（自殺），出臨界區後必須立刻引發 Yield，否則程式會崩潰 */
        #if ( configNUMBER_OF_CORES == 1 )
        {
            if( xSchedulerRunning != pdFALSE )
            {
                if( pxTCB == pxCurrentTCB )
                {
                    configASSERT( uxSchedulerSuspended == 0 );
                    taskYIELD_WITHIN_API(); // 迫使排程器立刻切換到別的工作
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
        }
        #endif /* #if ( configNUMBER_OF_CORES == 1 ) */

        traceRETURN_vTaskDelete();
    }

#endif /* INCLUDE_vTaskDelete */
```

#### 19.7 核心邏輯總結對照表
| **刪除情境**                               | **是否進入 xTasksWaitingTermination 鏈結串列？** | **誰負責呼叫 prvDeleteTCB 釋放記憶體？** | **是否會立刻引發 taskYIELD？**    |
| -------------------------------------- | --------------------------------------- | ----------------------------- | ------------------------- |
| **工作 A 刪除工作 B** (工作 B 此時在阻塞/掛起狀態)      | 否                                       | **工作 A** 當場釋放                 | 否                         |
| **工作 A 刪除自己** (單核心自殺)                  | **是**                                   | **Idle 任務** 隨後非同步釋放           | **是** (必須立刻讓出 CPU)        |
| **工作 A 刪除工作 B** (多核心 SMP，工作 B 在別的核心執行) | **是**                                   | **Idle 任務** 隨後非同步釋放           | **是** (對 B 核心發送跨核心中斷強制驅逐) |

#### 19.8 補充

##### 19.8.1 `uxCurrentNumberOfTasks` 與 `uxTaskNumber` 差在哪裡？

這兩個變數長得很像，但它們的**職責完全不同**。一個是給「排程器內部管理」用的，另一個則是專門給「外部偵錯工具（Debugger）」看的。

- `uxCurrentNumberOfTasks`（目前工作總數）
	- **本質**：一個**會上下浮動**的計數器。
	- **運作機制**：當你新建工作時，它就 `+1`；當你刪除工作時，它就 `-1`。
	- **用途**：用來記錄**當前系統中活著的工作到底有幾個**。核心會用它來判斷這是不是系統中第一個建立的工作（用來初始化鏈結串列），或者用來限制最大工作數量。
- `uxTaskNumber`（工作列表「版本號」/ 流水號）
	- **本質**：一個**只會往上加、絕不減少**的「變更計數器」（Change Counter）。
	- **運作機制**：
		- 新建工作時：`uxTaskNumber++`（並把這個數字當作該工作的身份證字號 `uxTCBNumber`）。
		- **刪除工作時**：你注意到了嗎？程式碼裡竟然也執行了 `uxTaskNumber++`！
	- **用途**：這是專門為了 **Kernel-aware Debugger（支援 RTOS 核心的偵錯器，如 OpenOCD、J-Link Ozone）** 設計的。
		- 當偵錯器透過模擬器暫停晶片時，它如果每次都去重新讀取所有工作的鏈結串列，會非常浪費時間。
		- 因此，偵錯器會盯著 `uxTaskNumber` 這個變數。只要發現這個變數的數字變了（不管是因為新增還是刪除工作），偵錯器就知道：「工作列表結構變了！我需要重新整理 UI 畫面上的工作狀態。」
##### 19.8.2 為什麼刪除工作時需要呼叫 `prvResetNextTaskUnblockTime`？
這是一個為了**提升 CPU 執行效率**而設計的精妙機制。

- 背景知識：什麼是 `xNextTaskUnblockTime`？
	- 當工作呼叫 `vTaskDelay(50)` 時，它會被丟進「延時鏈結串列（Delayed List）」，並計算出它該醒來的時間（例如目前的系統 Tick 是 10，那它醒來的時間就是 60）。
	- FreeRTOS 為了不讓 CPU 在每一次 Tick 中斷（每 1ms 觸發一次）時，都去苦苦搜尋整個延時鏈結串列，它設計了一個全域變數叫做 **`xNextTaskUnblockTime`**。這個變數**只記錄「全系統中，下一個最快該醒來的工作時間」**。
- 沒呼叫會發生什麼慘劇？假設目前系統有兩個工作在睡覺：
	- **工作 A**：預計在第 **50 個 Tick** 醒來（目前全系統最早清醒，所以 `xNextTaskUnblockTime = 50`）。
	- **工作 B**：預計在第 **100 個 Tick** 醒來。
	- 這時候，突然發生了一件事：**工作 A 在第 30 個 Tick 時，被人家用 `vTaskDelete()` 給無情地刪除了！**
	- 如果我們**沒有**呼叫 `prvResetNextTaskUnblockTime()`，系統的 `xNextTaskUnblockTime` 就會依然停留在 **50**。
	- 當時間走到第 50 個 Tick 時，Tick 中斷服務程式（ISR）會高興地跳出來說：「時間到 50 了！我要去延時鏈結串列叫醒工作！」結果進去鏈結串列一看，發現工作 A 早就死掉了，鏈結串列裡最快要醒來的是工作 B（第 100 個 Tick）。
	- 這會導致兩個嚴重的問題：
		- **浪費 CPU 效能**：排程器在第 50 個 Tick 做了一次毫無意義的檢查。
		- **時間邏輯大亂**：如果沒有及時把 `xNextTaskUnblockTime` 更新成 **100**，排程器接下來在第 51, 52, 53... 個 Tick 都會以為「有工作超時沒醒來」，進而瘋狂地反覆檢查鏈結串列，導致 CPU 資源被無意義的中斷操作吃乾抹淨。

### 20. *xTaskDelayUntil*

`xTaskDelayUntil()` 是 FreeRTOS 中用來實現**絕對時間延時**（或稱**週期性任務排程**）的神級函數。

許多初學者分不清它跟 `vTaskDelay()` 的差別：
- **`vTaskDelay()`（相對延時）**：從「呼叫該函數的當下」開始睡 50ms。這會導致任務每次執行的間隔，都會加上任務本身的執行時間與被搶佔的時間，產生**時間漂移（Drift）**。
- **`xTaskDelayUntil()`（絕對延時）**：精準計算「上一次醒來」到「下一次該醒來」的絕對時間差。不論任務中間執行了多久，都能確保**每一次執行的起點間隔是完全恆定的**。

#### 20.1 函數入口與初始化

這部分進行引數檢查，並透過掛起排程器（`vTaskSuspendAll`）進入安全存取區。這裡不用臨界區（Critical Section），是因為接下來任務可能會進入睡眠，需要引發排程。

```C
#if ( INCLUDE_xTaskDelayUntil == 1 )

    BaseType_t xTaskDelayUntil( TickType_t * const pxPreviousWakeTime,
                                const TickType_t xTimeIncrement )
    {
        TickType_t xTimeToWake;
        BaseType_t xAlreadyYielded, xShouldDelay = pdFALSE;

        traceENTER_xTaskDelayUntil( pxPreviousWakeTime, xTimeIncrement );

        /* 防呆機制：指標不能為空，且週期時間必須大於 0 */
        configASSERT( pxPreviousWakeTime );
        configASSERT( ( xTimeIncrement > 0U ) );

        /* 掛起排程器，確保在此區塊內系統 Tick 计数器（xTickCount）不會被中斷更新 */
        vTaskSuspendAll();
        {
            /* 擷取當前的系統 Tick 快照 */
            const TickType_t xConstTickCount = xTickCount;

            configASSERT( uxSchedulerSuspended == 1U );

            /* 計算出理論上任務下一次該醒來的絕對 Tick 時間點 */
            xTimeToWake = *pxPreviousWakeTime + xTimeIncrement;
```

#### 20.2 核心溢位判斷（Case 1：當前 Tick 计数器已溢位翻轉）

由於系統時間 `xTickCount` 是一個無號整數（Unsigned Integer），當它達到最大值後會歸零（溢位）。FreeRTOS 在這裡展現了極其嚴密的數學思維來處理溢位。
- 解釋一下什麼時候 xShouldDelay = pdTRUE 會被執行
	1. xConstTickCount < pxPreviousWakeTime 代表 xConstTickCount 溢位
	2. xTimeToWake < pxPreviousWakeTime 代表下次醒來時間也溢位
	3. xTimeToWake > xConstTickCount 代表 xTimeToWake 和 xConstTickCount 都溢位，但xTimeToWake 仍然大於目前系統 Tick 的時間，所以還是要 delay

```C
/* 情況一：當前的 Tick 計數器（xConstTickCount）小於上一次醒來的時間。
             * 這代表從上一次呼叫到現在，系統 Tick 已經發生過一次「溢位翻轉（Wrap-around）」回 0 了 */
            if( xConstTickCount < *pxPreviousWakeTime )
            {
                /* 在系統 Tick 已溢位的狀況下，只有當「下一次醒來時間也溢位」
                 * 且「醒來時間大於當前 Tick」時，才代表這個目標時間點還在未來，需要延時。 */
                if( ( xTimeToWake < *pxPreviousWakeTime ) && ( xTimeToWake > xConstTickCount ) )
                {
                    xShouldDelay = pdTRUE;
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
```

#### 20.3 核心溢位判斷（Case 2：當前 Tick 计数器正常未溢位）

這是最常見的狀況，但仍必須考慮到「下一次醒來的時間點 `xTimeToWake` 剛好跨越溢位邊界」的可能性。
- 已知目前系統 Tick 沒溢位
	 - xTimeToWake < pxPreviousWakeTime => 下次醒來時間溢位 => 要 delay
	 - xTimeToWake > xConstTickCount => 醒來時間沒溢位，但仍然比目前時間晚 => 要 delay

```C
/* 情況二：當前的 Tick 計數器正常，沒有比上一次醒來時間小（未發生溢位） */
            else
            {
                /* 在此狀況下，满足以下兩種條件之一就代表目標時間在未來，需要延時：
                 * 1. xTimeToWake < *pxPreviousWakeTime : 醒來時間算出來溢位了（繞回前面去了）。
                 * 2. xTimeToWake > xConstTickCount     : 醒來時間沒有溢位，且確實大於目前時間。 */
                if( ( xTimeToWake < *pxPreviousWakeTime ) || ( xTimeToWake > xConstTickCount ) )
                {
                    xShouldDelay = pdTRUE;
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
```

#### 20.4 更新時間軸並將任務推入 Delayed 鏈結串列

如果經過上述判斷，目標時間確實「在未來」，就把任務移出 Ready List，丟到 Delayed List 去睡覺。

```C
/* 關鍵步驟：不論這一次需不需要睡覺，都必須更新上一週期醒來時間的指針。
             * 這確保了下一次呼叫時，時間軸基礎點是連續的，徹底消除時間漂移。 */
            *pxPreviousWakeTime = xTimeToWake;

            if( xShouldDelay != pdFALSE )
            {
                traceTASK_DELAY_UNTIL( xTimeToWake );

                /* 注意！核心底層函數 prvAddCurrentTaskToDelayedList 需要的是「相對剩餘時間」，
                 * 而不是絕對時間，所以這裡傳入 (xTimeToWake - xConstTickCount) 算出還要睡幾個 Tick。 */
                prvAddCurrentTaskToDelayedList( xTimeToWake - xConstTickCount, pdFALSE );
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
```

#### 20.5 恢復排程器與上下文切換（Yield）

最後，打開排程器。如果我們剛才成功把自己送去睡覺了，就要立刻讓出 CPU 主導權。

```C
/* 恢復排程器。如果恢復過程中發現有更高優先權任務醒了，xTaskResumeAll 會回傳 pdTRUE */
        xAlreadyYielded = xTaskResumeAll();

        /* 如果 xTaskResumeAll 還沒引發上下文切換，但我們剛才已經把自己放進 Delayed List 了，
         * 這裡必須主動呼叫 taskYIELD_WITHIN_API() 強制讓出 CPU，讓別的工作執行。 */
        if( xAlreadyYielded == pdFALSE )
        {
            taskYIELD_WITHIN_API();
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }

        traceRETURN_xTaskDelayUntil( xShouldDelay );

        /* 回傳 pdTRUE 代表任務真的延時成功了；
         * 回傳 pdFALSE 代表目標時間其實早就過去了（任務執行太久過載），沒有進行延時。 */
        return xShouldDelay;
    }

#endif /* INCLUDE_xTaskDelayUntil */
```

### 21. vTaskDelay

相比於我們上一題聊到的 `xTaskDelayUntil()`（絕對週期延時），這段原始碼主角 **`vTaskDelay()`** 則是 FreeRTOS 裡最經典、最常被使用的**相對延時**函數。

它的邏輯比 `xTaskDelayUntil` 簡單直覺許多：**「不管現在幾點，反正老子從這一刻開始，就是要躺平睡足 `xTicksToDelay` 個 Tick 時間。」**

#### 21.1 函數入口與延時條件檢查

```C
#if ( INCLUDE_vTaskDelay == 1 )

    void vTaskDelay( const TickType_t xTicksToDelay )
    {
        /* 標記變數：用來記錄排程器在恢復時，是否已經順便幫我們做完上下文切換了 */
        BaseType_t xAlreadyYielded = pdFALSE;

        traceENTER_vTaskDelay( xTicksToDelay );

        /* 防禦性設計：只有當延時時間大於 0，才真正執行睡眠邏輯。
         * 如果傳入 0，代表不睡覺，純粹只是想引發一次強制的排程（Yield），讓出 CPU 給同優先權的工作。 */
        if( xTicksToDelay > ( TickType_t ) 0U )
        {
```

#### 21.2 掛起排程器與移入延時鏈結串列

```C
/* 核心關鍵：掛起排程器（Suspend All）。
             * 這會暫停工作的切換，但允許硬體中斷正常響應。 */
            vTaskSuspendAll();
            {
                configASSERT( uxSchedulerSuspended == 1U );

                traceTASK_DELAY();

                /* 核心魔法：將目前正在執行的任務（自己），從 Ready List 移除，
                 * 並塞進 Delayed List（阻塞/延時鏈結串列）中，設定睡眠時間為 xTicksToDelay。 */
                prvAddCurrentTaskToDelayedList( xTicksToDelay, pdFALSE );
            }
            /* 恢復排程器。如果在掛起期間有其他更高優先權的工作醒來，
             * 或者系統 Tick 有更新，xTaskResumeAll() 會在內部直接觸發 Yield，並回傳 pdTRUE。 */
            xAlreadyYielded = xTaskResumeAll();
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
```

#### 21.3 強制上下文切換（讓出 CPU）

任務已經把自己塞進睡袋了，最後一步就是必須把 CPU 的發言權交出來。

```C
/* * 【生死關頭的檢查】
         * 如果 xAlreadyYielded == pdFALSE，代表剛才的 xTaskResumeAll() 沒有引發上下文切換。
         * 但因為我們剛才已經把自己放進 Delayed List 了，如果這裡不主動引發 Yield，
         * 目前這個工作就會違反物理定律，在「已經處於阻塞狀態」的情況下繼續往下執行！
         * 因此，必須在這裡強制呼叫 taskYIELD_WITHIN_API() 讓自己睡去。
         */
        if( xAlreadyYielded == pdFALSE )
        {
            taskYIELD_WITHIN_API();
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }

        traceRETURN_vTaskDelay();
    }

#endif /* INCLUDE_vTaskDelay */
```

#### 21.4 補充

##### 21.4.1 為什麼這裡用 `vTaskSuspendAll()`，而不是用關中斷的 `taskENTER_CRITICAL()`？

在修改工作的鏈結串列（把目前工作移出 Ready List、丟進 Delayed List）時，為了防止鏈結串列被破壞，必須進行**互斥保護**。FreeRTOS 有兩種武器：
1. `taskENTER_CRITICAL()`：直接關閉 CPU 中斷。
2. `vTaskSuspendAll()`：不關中斷，只禁止排程器切換工作。

FreeRTOS 的選擇理由：
- `prvAddCurrentTaskToDelayedList()` 這個函數在內部需要翻閱、計算、並把節點插入到鏈結串列的正確時間順序位置中，這需要耗費一定的 CPU 週期。
- 如果使用臨界區（關中斷），這段時間內系統將**無法響應任何硬體中斷**（例如 UART 接收、馬達定時器中斷），這會嚴重降低系統的「即時性（Real-time）」。
- 使用 `vTaskSuspendAll()`，硬體中斷依然可以爽快地響應、執行中斷服務程式（ISR），只是在中斷裡不允許引發工作切換。這樣既保護了鏈結串列的安全，又兼顧了系統對硬體事件的極速響應。

##### 21.4.2 當 `xTicksToDelay == 0` 時，到底發生了什麼事？

程式碼一進去就判斷 `if( xTicksToDelay > 0U )`。如果我們寫 `vTaskDelay(0)`，它會直接跳到 `else`，然後一路走到最後的：

```C
if( xAlreadyYielded == pdFALSE ) {
    taskYIELD_WITHIN_API(); 
}
```

這意味著，`vTaskDelay(0)` 的本質就是 **「純粹的、無條件的讓出 CPU」**。
- 它不會把自己放進延時鏈結串列。
- 它依然待在 Ready List 裡面。
- 它只是強制引發一次排程，去看看 Ready List 裡面有沒有**同等優先權**的其他兄弟正在排隊？有的話，排隊的兄弟先執行；如果沒有，排程器轉了一圈，還是會回來繼續執行它。這在編寫合作式多工（Cooperative Multitasking）的程式碼時非常實用。

##### 21.4.3 為什麼會區分 pdTRUE 與 pdFALSE？（底層運作機制）
當你呼叫 `vTaskSuspendAll()` 掛起排程器時，排程器會進入鎖定狀態，**禁止任何工作切換**。

然而，這段期間**硬體中斷（ISR）依然是開著的**。如果中斷服務程式觸發了某些會「喚醒工作」的事件（例如 Queue 收到資料、Semaphore 被釋放、或是 System Tick 時間到了），這些本該醒來進入 Ready 狀態的工作，因為排程器被鎖住，核心無法立刻將它們安置到正常的 Ready List 中。

為了儲存這些臨時醒來的工作，FreeRTOS 設計了一個臨時收容所，叫做 **`xPendingReadyList`（懸起準備鏈結串列）**。

當你呼叫 `xTaskResumeAll()` 準備解鎖排程器時，核心會執行以下邏輯：
1. **清空收容所**：用一個 `while` 迴圈，把 `xPendingReadyList` 裡面的所有工作通通搬回正常的 Ready List。
2. **評估優先權**：在搬移的過程中，核心會進行比對：「剛才這些在中斷中醒來的工作，**有沒有任何一個人的優先權，比『目前正在執行』的工作還要高（或相等）？**」
	1. **情況 A（有更高優先權工作）**：核心會在退出函數前，自動引發一次 `taskYIELD()`，把 CPU 讓給那個剛醒來的厲害工作。此時函數最終會回傳 **`pdTRUE`**。
	2. **情況 B（沒有更高優先權工作，或收容所根本是空的）**：目前的工作依然是全場優先權最高的老大，不需要切換，函數就會回傳 **`pdFALSE`**。

### 22. *eTaskGetState*

`eTaskGetState()` 是 FreeRTOS 中用來查詢特定任務目前處於什麼「狀態」的函數。它的回傳值是一個列舉型態（`eTaskState`），包含：`eRunning`（執行中）、`eReady`（就緒）、`eBlocked`（阻塞）、`eSuspended`（掛起）或 `eDeleted`（已刪除）。

這個函數的精妙之處在於：**FreeRTOS 的 TCB（任務控制區塊）內部其實沒有一個專門記錄「目前狀態」的變數。** 那排程器是怎麼知道工作現在是死是活呢？答案是**看它目前被掛在哪個鏈結串列（List）上**！`eTaskGetState()` 就像一個私家偵探，透過檢查任務的 `xStateListItem` 和 `xEventListItem` 這兩個鉤子掛在誰家，來推導出任務的真實狀態。

#### 22.1 函數入口與單核心自我查詢優化

這部分定義了函數的啟動條件，並優先處理單核心下最簡單的情況：「工作自己查自己」。

```C
#if ( ( INCLUDE_eTaskGetState == 1 ) || ( configUSE_TRACE_FACILITY == 1 ) || ( INCLUDE_xTaskAbortDelay == 1 ) )

    eTaskState eTaskGetState( TaskHandle_t xTask )
    {
        eTaskState eReturn;
        List_t const * pxStateList;
        List_t const * pxEventList;
        List_t const * pxDelayedList;
        List_t const * pxOverflowedDelayedList;
        const TCB_t * const pxTCB = xTask; // 將不具型態的 Handle 轉回 TCB 結構指標

        traceENTER_eTaskGetState( xTask );

        configASSERT( pxTCB != NULL );

        /* 優化機制：如果是單核心，且查詢的對象就是目前正在執行的自己 */
        #if ( configNUMBER_OF_CORES == 1 )
            if( pxTCB == pxCurrentTCB )
            {
                /* 既然自己還能動、還能呼叫這個 API，那狀態絕對是 eRunning（執行中） */
                eReturn = eRunning;
            }
            else
        #endif
        {
```

#### 22.2 進入臨界區：快照任務的鏈結串列容器

如果查的是別人（或是多核心環境），為了防止排程器或中斷突然去移動鏈結串列，必須進入臨界區，把該任務「目前掛在哪裡」的容器地址拍照存證。

```C
taskENTER_CRITICAL();
            {
                /* 獲取該任務的狀態列表項（xStateListItem）目前在哪個鏈結串列中 */
                pxStateList = listLIST_ITEM_CONTAINER( &( pxTCB->xStateListItem ) );
                
                /* 獲取該任務的事件列表項（xEventListItem）目前在哪個鏈結串列中 */
                pxEventList = listLIST_ITEM_CONTAINER( &( pxTCB->xEventListItem ) );
                
                /* 取得當前系統的延時列表與溢位延時列表的指標 */
                pxDelayedList = pxDelayedTaskList;
                pxOverflowedDelayedList = pxOverflowDelayedTaskList;
            }
            taskEXIT_CRITICAL(); // 迅速離開臨界區，維持系統即時性
```

#### 22.3 判斷是否為 Ready 狀態（懸起就緒）或 Blocked 狀態

出臨界區後，開始拿剛才拿到的指針進行連鎖 `if-else` 排查。

```C
/* 檢查 A：它是否在「懸起就緒鏈結串列（xPendingReadyList）」中？ */
            if( pxEventList == &xPendingReadyList )
            {
                /* 還記得我們之前聊過嗎？在排程器鎖定期間醒來的工作會被安置在這裡。
                 * 只要待在這裡，不管它的狀態鏈結串列掛在哪，它本質上都已經算 eReady 了 */
                eReturn = eReady;
            }
            /* 檢查 B：它的狀態鏈結串列，是否在正常的延時鏈結串列或溢位延時鏈結串列中？ */
            else if( ( pxStateList == pxDelayedList ) || ( pxStateList == pxOverflowedDelayedList ) )
            {
                /* 如果在睡覺名單內，狀態就是 eBlocked（阻塞） */
                eReturn = eBlocked;
            }
```

#### 22.4 判斷是否為 Suspended 狀態（掛起）

如果任務掛在掛起鏈結串列（`xSuspendedTaskList`），事情通常沒那麼簡單。它有可能是真的被手動掛起（Suspend），也有可能是因為無限期等待（Delay Until Max / Wait Forever）而被丟進來。

```C
#if ( INCLUDE_vTaskSuspend == 1 )
                else if( pxStateList == &xSuspendedTaskList )
                {
                    /* 它的確在掛起鏈結串列中。但它是純掛起，還是無限期阻塞（如等待 Semaphore）？
                     * 檢查它的事件鏈結串列有沒有掛在任何 RTOS 物件上 */
                    if( listLIST_ITEM_CONTAINER( &( pxTCB->xEventListItem ) ) == NULL )
                    {
                        #if ( configUSE_TASK_NOTIFICATIONS == 1 )
                        {
                            BaseType_t x;

                            /* 如果它沒有掛在任何事件物件上，預設為 eSuspended */
                            eReturn = eSuspended;

                            /* 【魔鬼細節】：它雖然沒掛在事件物件上，但會不會是在無限期等待「任務通知（Task Notification）」？
                             * 遍歷所有的通知通道，如果有一個通道在等待通知，那它真正的法律狀態其實是「阻塞」而不是掛起 */
                            for( x = ( BaseType_t ) 0; x < ( BaseType_t ) configTASK_NOTIFICATION_ARRAY_ENTRIES; x++ )
                            {
                                if( pxTCB->ucNotifyState[ x ] == taskWAITING_NOTIFICATION )
                                {
                                    eReturn = eBlocked;
                                    break;
                                }
                            }
                        }
                        #else /* if ( configUSE_TASK_NOTIFICATIONS == 1 ) */
                        {
                            eReturn = eSuspended; // 沒啟用通知功能，那就是純掛起
                        }
                        #endif /* if ( configUSE_TASK_NOTIFICATIONS == 1 ) */
                    }
                    else
                    {
                        /* 如果事件列表項有容器，代表它是在無限期（portMAX_DELAY）等待某個 IPC 事件 */
                        eReturn = eBlocked;
                    }
                }
            #endif /* if ( INCLUDE_vTaskSuspend == 1 ) */
```

#### 22.5 判斷是否為 Deleted 狀態（已刪除）

```C
#if ( INCLUDE_vTaskDelete == 1 )
                else if( ( pxStateList == &xTasksWaitingTermination ) || ( pxStateList == NULL ) )
                {
                    /* 如果它出現在「等待終止鏈結串列」中（代表它自殺了，Idle 任務還沒來收屍），
                     * 或者它徹底不屬於任何鏈結串列（NULL），那它就是 eDeleted（已刪除） */
                    eReturn = eDeleted;
                }
            #endif
```

#### 22.6 收尾：多核心 SMP 執行中狀態與 Ready 狀態判定

如果上面的情況都不符合，那工作一定就是 Ready（就緒）或者是在別的核心跑（Running）。

```C
else
            {
                #if ( configNUMBER_OF_CORES == 1 )
                {
                    /* 單核心下，前面排除了那麼多，且已經確認它不是 Currently Running（pxCurrentTCB），
                     * 那它此時此刻唯一的可能就是：乖乖在 Ready List 裡排隊的 eReady 狀態。 */
                    eReturn = eReady;
                }
                #else /* #if ( configNUMBER_OF_CORES == 1 ) */
                {
                    /* 多核心（SMP）環境下，我們必須額外檢查：
                     * 這個工作此時此刻有沒有可能正在「別的核心」上面爽爽執行中？ */
                    if( taskTASK_IS_RUNNING( pxTCB ) == pdTRUE )
                    {
                        eReturn = eRunning; // 是的，別的核心正在跑它
                    }
                    else
                    {
                        eReturn = eReady; // 沒有在跑，那就是就緒狀態
                    }
                }
                #endif /* #if ( configNUMBER_OF_CORES == 1 ) */
            }
        }

        traceRETURN_eTaskGetState( eReturn );

        return eReturn; // 回傳最終鑑定結果
    }

#endif /* 各種 INCLUDE 巨集結束 */
```

#### 22.7 總結

| **檢查項目**     | **滿足條件**                                             | **推導出的最終狀態**     |
| ------------ | ---------------------------------------------------- | ---------------- |
| **自我查詢（單核）** | `pxTCB == pxCurrentTCB`                              | **`eRunning`**   |
| **臨時收容所**    | 掛在 `xPendingReadyList`                               | **`eReady`**     |
| **延時名單**     | 掛在 `pxDelayedTaskList` 或 `pxOverflowDelayedTaskList` | **`eBlocked`**   |
| **掛起名單**     | 掛在 `xSuspendedTaskList` 且**沒等任何事件與通知**               | **`eSuspended`** |
| **掛起名單**     | 掛在 `xSuspendedTaskList` 但**有等事件或通知**                 | **`eBlocked`**   |
| **終止名單**     | 掛在 `xTasksWaitingTermination` 或不屬於任何清單               | **`eDeleted`**   |
| **多核檢查**     | `taskTASK_IS_RUNNING( pxTCB ) == pdTRUE`             | **`eRunning`**   |
| **其餘情況**     | 以上皆非                                                 | **`eReady`**     |

### 23. *uxTaskPriorityGet*

**`uxTaskPriorityGet()`** 是 FreeRTOS 裡用來查詢特定任務優先權（Priority）的 API。

#### 23.1 函數入口與變數宣告

這部分定義了函數的啟動條件，並檢查是否啟用了此功能。

```C
#if ( INCLUDE_uxTaskPriorityGet == 1 )

    UBaseType_t uxTaskPriorityGet( const TaskHandle_t xTask )
    {
        TCB_t const * pxTCB;  // 宣告一個內部 TCB 結構指標
        UBaseType_t uxReturn; // 宣告用來存回傳優先權的變數

        traceENTER_uxTaskPriorityGet( xTask );
```

#### 23.2 進入臨界區與優先權擷取

這是函數的核心操作區塊。透過進入臨界區（Critical Section）來確保讀取數值時的資料完整性。

```C
/* 核心關鍵：進入臨界區（在單核下通常是關中斷，多核 SMP 下會額外拿取 Spinlock）
         * 防止在讀取 TCB 欄位的途中，其他任務或中斷突然去修改該任務的優先權 */
        portBASE_TYPE_ENTER_CRITICAL();
        {
            /* 核心魔法：將傳入的 TaskHandle_t 轉換為內部的 TCB_t 指標。
             * 備註：如果 xTask 傳入的是 NULL，prvGetTCBFromHandle 內部會自動
             * 將它指向「目前正在呼叫此 API 的工作自己（pxCurrentTCB）」 */
            pxTCB = prvGetTCBFromHandle( xTask );
            configASSERT( pxTCB != NULL );

            /* 直接從 TCB 結構中讀取當前的優先權數值 */
            uxReturn = pxTCB->uxPriority;
        }
        portBASE_TYPE_EXIT_CRITICAL(); // 迅速離開臨界區，恢復中斷響應
```

### 24. *uxTaskPriorityGetFromISR*

**`uxTaskPriorityGetFromISR()`** 是 *[[#23. uxTaskPriorityGet]]* 的中斷安全版本 (Interrupt-safe version)

在 FreeRTOS 中，凡是結尾帶有 **`FromISR`** 的 API，都是專門設計給**中斷服務程式（ISR, Interrupt Service Routine）呼叫 wilderness 的。因為中斷的執行環境與普通工作（Task）完全不同，它必須極度精簡，而且**絕對不允許進入睡眠（Blocked）或主動讓出 CPU（Yield）**。

這段原始碼最精妙的價值，不在於讀取優先權（那只是單純的 `uxPriority` 賦值），而是在於註解中提到的 **「中斷嵌套與安全邊界機制」**。

#### 24.1 函數入口與變數宣告

這部分進行基本的變數初始化。注意這裡多了一個 `uxSavedInterruptStatus` 變數，這是中斷版本專有的。

```C
#if ( INCLUDE_uxTaskPriorityGet == 1 )

    UBaseType_t uxTaskPriorityGetFromISR( const TaskHandle_t xTask )
    {
        TCB_t const * pxTCB;
        UBaseType_t uxReturn;
        
        /* 關鍵變數：用來儲存進入「中斷臨界區」前的 CPU 中斷狀態（遮罩狀態） */
        UBaseType_t uxSavedInterruptStatus;

        traceENTER_uxTaskPriorityGetFromISR( xTask );
```

#### 24.2 核心防禦：中斷安全邊界檢查

```C
/* * 【核心安全防線】：檢查目前發出此中斷的硬體優先權是否合法！
         * * 在支援中斷嵌套的晶片（如 ARM Cortex-M）中，FreeRTOS 設定了一個安全邊界：
         * configMAX_SYSCALL_INTERRUPT_PRIORITY（最大系統呼叫中斷優先權）。
         * * 1. 高於此邊界的高強度中斷：完全不受 FreeRTOS 控制，也絕不允許呼叫任何 FreeRTOS API。
         * 2. 低於或等於此邊界的中斷：才可以呼叫帶有 FromISR 結尾的 API。
         * * 如果使用者將此中斷的硬體優先權設得太高，卻又在裡面呼叫本函數，
         * 這裡會直接觸發 configASSERT 報錯，防止作業系統核心資料結構被中斷踩壞。
         */
        portASSERT_IF_INTERRUPT_PRIORITY_INVALID();
```

#### 24.3 中斷臨界區保護與數值擷取

```C
/* MISRA Ref 4.7.1：提醒檢查返回值（此處依據規範安全處理） */
        /* * 核心機制：進入「中斷安全臨界區」
         * 1. 這裡不能用普通的 taskENTER_CRITICAL()，因為那是給工作（Task）用的。
         * 2. taskENTER_CRITICAL_FROM_ISR() 會在硬體層面遮蔽「不高於安全邊界」的中斷。
         * 3. 它會回傳「進入前的中斷遮罩狀態」，我們必須用 uxSavedInterruptStatus 存起來。
         */
        uxSavedInterruptStatus = ( UBaseType_t ) taskENTER_CRITICAL_FROM_ISR();
        {
            /* 將傳入的 Handle 轉為內部 TCB 指標。如果是 NULL，自動指向目前被中斷強行暫停的那個工作 */
            pxTCB = prvGetTCBFromHandle( xTask );
            configASSERT( pxTCB != NULL );

            /* 直接擷取工作目前的優先權 */
            uxReturn = pxTCB->uxPriority;
        }
        /* 離開中斷臨界區：傳入剛才保存的狀態，將 CPU 中斷還原回進入前的模樣 */
        taskEXIT_CRITICAL_FROM_ISR( uxSavedInterruptStatus );

        traceRETURN_uxTaskPriorityGetFromISR( uxReturn );

        return uxReturn;
    }

#endif /* INCLUDE_uxTaskPriorityGet */
```

#### 24.4 補充

##### 24.4.1 為什麼 `FromISR` 的臨界區，必須要有一個 `uxSavedInterruptStatus` 變數來傳遞狀態？

這跟中斷嵌套（Interrupt Nesting）的物理特性有關。

- **工作的臨界區（普通版）**：`taskENTER_CRITICAL()` 內部使用的是一個簡單的**計數器變數 (`uxCriticalNesting`)**。因為工作切換是受作業系統全面掌控的，核心只需要單純記錄「加鎖了幾次、解鎖了幾次」就好。
- **中斷的臨界區（ISR 版）**：中斷是隨機、硬體層面強行觸發的。當 **中斷 A** 正在執行並呼叫 `taskENTER_CRITICAL_FROM_ISR()` 時，晶片內部的硬體中斷暫存器（如 ARM 的 `BASEPRI`）會被改寫以屏蔽其他中斷。
- **嵌套悲劇**：如果中斷 A 執行到一半，突然發生了優先權更高的 **中斷 B** 闖進來，中斷 B 也呼叫了這個 API。如果沒有把各自進入前的「原始暫存器狀態」當作局部變數（存於堆疊中）保存下來，中斷返回時，硬體暫存器的狀態就會被亂改，導致中斷 A 醒來後，發現原本該屏蔽的中斷突然被打開了，造成嚴重的時序錯亂。

##### 24.4.2 在中斷裡拿到的這個優先權，出函數後會是「過期」的嗎？

**答案是：依舊有可能，但機率極低。**

- **為什麼說機率極低？**：因為中斷（ISR）的優先權是全系統中最高的。當 CPU 正在執行這段中斷程式碼時，**沒有任何「工作（Task）」能夠上台執行**。這意味著，普通工作絕對不可能在這時候去呼叫 `vTaskPrioritySet()` 來修改優先權。
- **唯一的例外**：此時如果有另一個更高優先權（且同樣在安全邊界內）的**中斷 C** 突然爆發，搶佔了當前中斷，並且中斷 C 內部呼叫了某些 API 觸發了 Mutex 釋放，間接導致該工作的優先權發生了改變。

但即便如此，FreeRTOS 的哲學依然不變：這個 API 只是用來做巨觀的狀態檢查、除錯或記錄（Logging）。中斷服務程式本身絕對不應該根據這個查出來的優先權數值，去操作任何核心的排程鏈結串列，因為那是排程器自己進臨界區後才會進行的安全底層操作。

##### 24.4.3 prvGetTCBFromHandle 的 NULL 轉換魔法

問題的癥結點，在於註解中所說的「自動指向」，其實是發生在 **`prvGetTCBFromHandle( xTask )` 函數的內部**，而不是在外層。

- 原始碼還原：`prvGetTCBFromHandle` 裡面長怎樣？

```C
/* FreeRTOS 核心底層的轉換邏輯（偽程式碼展開） */
privTASK_TCB * prvGetTCBFromHandle( TaskHandle_t xTask )
{
    /* 1. 這裡進行了第一階段判斷：呼叫者傳進來的 Handle 是不是 NULL？ */
    if( xTask == NULL )
    {
        /* 【神祕的自動指向】
         * 如果你呼叫 API 時傳入 NULL（例如：uxTaskPriorityGetFromISR( NULL );），
         * 這裡會自動幫你替換成 pxCurrentTCB！
         * pxCurrentTCB 就是「目前正拿到 CPU 控制權、但剛剛被這個中斷硬生生打斷」的那個工作。
         */
        return pxCurrentTCB; 
    }
    else
    {
        /* 如果不是 NULL，就直接把 Handle 轉成 TCB 指標回傳 */
        return ( privTASK_TCB * ) xTask;
    }
}
```

完美的兩階段安全防線

現在我們把 `prvGetTCBFromHandle` 的內部邏輯，放回你原本看到的 `uxTaskPriorityGetFromISR` 函數中，整個邏輯鏈就會閉合：

```C
UBaseType_t uxTaskPriorityGetFromISR( const TaskHandle_t xTask )
{
    // ... 前置作業 ...

    uxSavedInterruptStatus = ( UBaseType_t ) taskENTER_CRITICAL_FROM_ISR();
    {
        /* * 【第一階段：寬容的代換】
         * 如果你傳入 xTask = NULL
         * 經過 prvGetTCBFromHandle(NULL) 處理後，回傳值會變成 pxCurrentTCB。
         * 所以此時賦值結果：pxTCB = pxCurrentTCB;（它絕對不是 NULL！）
         */
        pxTCB = prvGetTCBFromHandle( xTask );

        /* * 【第二階段：嚴格的斷言】
         * 既然上面已經把 NULL 自動代換成當前工作了，為什麼這裡還要 configASSERT？
         * 這是為了防範另一種毀滅性狀況：
         * 如果使用者傳入的 xTask 不是 NULL，而是一個「早就已經被刪除的無效 Handle（野指標）」，
         * 或者系統在剛開機初始化階段，連第一個工作都還沒跑起來（pxCurrentTCB 本身就是 NULL），
         * 這時候 pxTCB 就真的會是 NULL！
         * 為了防止接下來訪問 pxTCB->uxPriority 導致晶片觸發 HardFault，這裡必須進行斷言攔截！
         */
        configASSERT( pxTCB != NULL );

        uxReturn = pxTCB->uxPriority;
    }
    taskEXIT_CRITICAL_FROM_ISR( uxSavedInterruptStatus );

    return uxReturn;
}
```

什麼叫做「被中斷強行暫停的那個工作」？

在 MCU（微處理器）運行的世界裡，CPU 無時無刻都在執行程式碼。

1. 在中斷爆發的前一微秒，CPU 正在快快樂樂地執行 **工作 A** 的代碼（此時全域變數 `pxCurrentTCB` 指向工作 A）。
2. 突然，硬體中斷（如定時器、UART）爆發，CPU 硬生生暫停了工作 A，強行跳進中斷服務程式（ISR）。
3. 如果你在 ISR 裡面想要知道：「到底是哪一個倒楣鬼剛才被我打斷了？我想查它的優先權。」
4. 你不需要費心去操作 TCB 指標，你只需要呼叫：`uxTaskPriorityGetFromISR( NULL );`
5. 1. FreeRTOS 就會心領神會，透過 `prvGetTCBFromHandle` 自動幫你指回那個被強行暫停的 `pxCurrentTCB`（工作 A）。

### 25. *uxTaskBasePriorityGet*

類似於 *[[#23. uxTaskPriorityGet]]* ，只是是讀取 `pxTCB->uxBasePriority`

### 26. *uxTaskBasePriorityGetFromISR*

類似於 *[[#24. uxTaskPriorityGetFromISR]]* ，只是是讀取 `pxTCB->uxBasePriority`

### 27. *vTaskPrioritySet*

**`vTaskPrioritySet()`** 是 FreeRTOS 中最複雜、最核心的 API 之一。它的工作是**動態修改某個任務的優先權**。

修改優先權之所以複雜，是因為它會直接觸發**排程器（Scheduler）的骨牌效應**：如果把別人的優先權調高，它是不是該立刻搶佔（Preempt）目前的 CPU？如果把自己（執行中任務）的優先權調低，台下是不是有其他更高優先權的兄弟正在拍桌等著上台？

#### 27.1 函數入口與輸入數值防禦

```C
#if ( INCLUDE_vTaskPrioritySet == 1 )

    void vTaskPrioritySet( TaskHandle_t xTask,
                           UBaseType_t uxNewPriority )
    {
        TCB_t * pxTCB;
        UBaseType_t uxCurrentBasePriority, uxPriorityUsedOnEntry;
        BaseType_t xYieldRequired = pdFALSE; // 標記：是否需要觸發上下文切換（Yield）

        #if ( configNUMBER_OF_CORES > 1 )
            BaseType_t xYieldForTask = pdFALSE; // 多核心專用：是否需要為目標任務騰出核心
        #endif

        traceENTER_vTaskPrioritySet( xTask, uxNewPriority );

        configASSERT( uxNewPriority < configMAX_PRIORITIES );

        /* 防禦性設計：如果傳入的優先權越界，強行壓回最大允許值 */
        if( uxNewPriority >= ( UBaseType_t ) configMAX_PRIORITIES )
        {
            uxNewPriority = ( UBaseType_t ) configMAX_PRIORITIES - ( UBaseType_t ) 1U;
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
```

#### 27.2 進入臨界區與排程決策（調高優先權）

```C
taskENTER_CRITICAL(); // 鎖定系統，準備動刀修改核心結構
        {
            /* 還記得上一題聊到的機制嗎？傳入 NULL 會自動代換成當前工作 pxCurrentTCB */
            pxTCB = prvGetTCBFromHandle( xTask );
            configASSERT( pxTCB != NULL );

            traceTASK_PRIORITY_SET( pxTCB, uxNewPriority );

            /* 考量 Mutex 機制：如果啟用 Mutex，要比對的是它的「基礎優先權（Base Priority）」，
             * 避免誤動到它因為優先權繼承而被暫時拉高的臨時優先權 */
            #if ( configUSE_MUTEXES == 1 )
            {
                uxCurrentBasePriority = pxTCB->uxBasePriority;
            }
            #else
            {
                uxCurrentBasePriority = pxTCB->uxPriority;
            }
            #endif

            /* 只有當新舊優先權不相等時，才需要大費周章調整 */
            if( uxCurrentBasePriority != uxNewPriority )
            {
                /* 情況 A：新優先權比原本的還要「高」 */
                if( uxNewPriority > uxCurrentBasePriority )
                {
                    #if ( configNUMBER_OF_CORES == 1 )
                    {
                        /* 單核心環境下：如果是幫「別人」調高優先權 */
                        if( pxTCB != pxCurrentTCB )
                        {
                            /* 檢查：調高後的優先權，是否超越了「目前正在跑的自己」？
                             * 如果是，代表別人比自己更有資格執行，標記需要 Yield 讓位 */
                            if( uxNewPriority > pxCurrentTCB->uxPriority )
                            {
                                xYieldRequired = pdTRUE;
                            }
                            else
                            {
                                mtCOVERAGE_TEST_MARKER();
                            }
                        }
                        else
                        {
                            /* 如果是調高「目前正在跑的自己」，自己本來就是老大，不需要 Yield */
                        }
                    }
                    #else /* 多核心環境（SMP） */
                    {
                        /* 多核心環境下：只要有人優先權變高，稍後就要評估是否去搶佔別的核心 */
                        xYieldForTask = pdTRUE;
                    }
                    #endif
                }
```

#### 27.3 排程決策（調低優先權）

```C
/* 情況 B：新優先權比原本的還要「低」，且目標工作「目前正在執行」 */
                else if( taskTASK_IS_RUNNING( pxTCB ) == pdTRUE )
                {
                    /* 既然目前正在跑的工作自願降薪（降優先權），
                     * 代表 Ready List 裡可能有其他原本在排隊的高優先權大哥，現在可以上台了！ */
                    #if ( configUSE_TASK_PREEMPTION_DISABLE == 1 )
                        if( pxTCB->xPreemptionDisable == pdFALSE ) // 確保目前沒禁止搶佔
                    #endif
                    {
                        xYieldRequired = pdTRUE; // 標記需要 Yield
                    }
                }
                else
                {
                    /* 如果是把一個「本來就沒在跑、在台下排隊或睡覺的工作」調低優先權，
                     * 那對目前的排程毫無影響，不需要 Yield */
                }
```

#### 27.4 寫入新優先權與調整事件鏈結串列（Event List）

完成決策後，正式修改 TCB 欄位，並更新事件列表排序值。

```C
/* 記錄修改前舊的優先權，稍後搬移 Ready List 時要用 */
                uxPriorityUsedOnEntry = pxTCB->uxPriority;

                #if ( configUSE_MUTEXES == 1 )
                {
                    /* 如果這個工作目前沒有繼承別人的優先權，或者新設定的值比繼承來的值還要大，
                     * 才真正改寫當前優先權。否則保留原先繼承來的高優先權。 */
                    if( ( pxTCB->uxBasePriority == pxTCB->uxPriority ) || ( uxNewPriority > pxTCB->uxPriority ) )
                    {
                        pxTCB->uxPriority = uxNewPriority;
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }

                    /* 無論如何，基礎優先權（uxBasePriority）都直接設定為新值 */
                    pxTCB->uxBasePriority = uxNewPriority;
                }
                #else
                {
                    pxTCB->uxPriority = uxNewPriority; // 沒啟用 Mutex，直接改寫
                }
                #endif

                /* 核心機制：更新事件鏈結串列項（xEventListItem）的 Value。
                 * 在 FreeRTOS 中，事件列表（如 Queue 等待隊列）是依據優先權「由大到小」排序。
                 * 為了符合鏈結串列由小到大的升序排列特性，儲存的值會被特殊處理成：(configMAX_PRIORITIES - 優先權)。
                 * 這裡確保該數值沒有被拿去做別的用途（如儲存 Blocked 時間）才進行修改。 */
                if( ( listGET_LIST_ITEM_VALUE( &( pxTCB->xEventListItem ) ) & taskEVENT_LIST_ITEM_VALUE_IN_USE ) == ( ( TickType_t ) 0U ) )
                {
                    listSET_LIST_ITEM_VALUE( &( pxTCB->xEventListItem ), ( ( TickType_t ) configMAX_PRIORITIES - ( TickType_t ) uxNewPriority ) );
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
```

##### 27.4.1 為什麼 uxNewPriority > pxTCB->uxPriority 才改 pxTCB->uxPriority？

這行判斷式是整個 FreeRTOS 為了結合 **「Mutex（互斥鎖）的優先權繼承機制（Priority Inheritance）」** 所設計的核心防線。

簡單一句話回答：**為了防止「外部刻意調低優先權」的動作，意外破壞了目前工作正在執行的「緊急救災狀態（臨時被拉高的優先權）」。**

假設系統中有三個工作：**工作 A（優先權 1，最低）**、**工作 B（優先權 2）**、**工作 C（優先權 3，最高）**。

時序演進：
1. **工作 A** 搶先上台，並拿鎖了一把 **Mutex**。
2. 突然，最高優先權的 **工作 C** 醒了，它也想要這把 Mutex，但因為被工作 A 鎖住了，工作 C 只能進入 Blocked 睡覺等鎖。
3. **【優先權繼承觸發】**：為了不讓中間的 工作 B 攔路搶劫（造成優先權翻轉），FreeRTOS 核心出手救災，把 **工作 A 的 `uxPriority` 暫時拉高到 3（跟工作 C 一樣高）**！
4. **【致命插曲發生】**：就在工作 A 穿著黃馬褂（優先權 3）拼命執行，想要趕快把事情做完並還鎖時，另一個核心（或某個中斷）突然不識相地呼叫了： `vTaskPrioritySet( 工作A, 2 );` （想要把工作 A 的基礎優先權從 1 調高到 2）。

為了防範上面的慘劇，FreeRTOS 寫下了你提問的這行神來之筆：

```C
/* * 情況：我們要將 工作 A 的優先權改為 uxNewPriority = 2。
 * 此時 工作 A 處於救災霸體：uxBasePriority = 1, uxPriority = 3。
 */
if( ( pxTCB->uxBasePriority == pxTCB->uxPriority ) || ( uxNewPriority > pxTCB->uxPriority ) )
{
    /* * 檢查條件一：pxTCB->uxBasePriority == pxTCB->uxPriority
     * 代表這隻工作目前「沒有」繼承任何人的優先權，它是正常狀態，可以直接改！
     * * 檢查條件二：uxNewPriority > pxTCB->uxPriority (也就是 2 > 3 成立嗎？不成立！)
     * 代表你這次新設定的值（2），比它目前因為救災而繼承來的臨時優先權（3）還要低。
     * 核心此時會心領神會：「我不能讓你這個低頭銜破壞了它目前的救災霸體！」
     * * 結論：條件不成立，跳過此區塊！ pxTCB->uxPriority 依然保持 3（安全！）。
     */
    pxTCB->uxPriority = uxNewPriority;
}
else
{
    /* 跳到這裡：雖然我不改它目前的救災優先權(uxPriority=3)，
     * 但你交代的新優先權我還是有幫你記帳記在基礎優先權裡。 */
    mtCOVERAGE_TEST_MARKER();
}

/* 記帳：無論如何，基礎優先權都確實被改成了 2 */
pxTCB->uxBasePriority = uxNewPriority;
```

那什麼時候 `uxNewPriority > pxTCB->uxPriority` 會成立？

如果在上述同一個救災場景中，外部突然呼叫 `vTaskPrioritySet( 工作A, 4 );`（想把它的優先權調到比皇上還要高的 4）：
- 條件 `4 > 3` 成立！
- 核心會認為：「哦！你新給的優先權比它繼承來的（3）還要更高、更緊急！」那沒問題，系統允許你當場把 `uxPriority` 改寫成全新的超高優先權 **4**。

#### 27.5 搬移 Ready List 鏈結串列

這是對系統鏈結串列結構影響最大的一步。如果任務正處於 Ready 狀態，必須把它從「舊优先權鏈結串列」拔掉，塞進「新優先權鏈結串列」。

```C
/* 檢查：這個任務目前是否正在 Ready List 中排隊？ */
                if( listIS_CONTAINED_WITHIN( &( pxReadyTasksLists[ uxPriorityUsedOnEntry ] ), &( pxTCB->xStateListItem ) ) != pdFALSE )
                {
                    /* 既然它在 Ready List 中，但優先權變了，它就待錯了房間。
                     * 先將它從舊的優先權房間（Ready List[舊]）中刪除。 */
                    if( uxListRemove( &( pxTCB->xStateListItem ) ) == ( UBaseType_t ) 0 )
                    {
                        /* 如果拔掉它之後，那個舊的 Ready List 房間空無一人，
                         * 必須通知硬體（通常是清除 Bitmap 點位），代表該優先權已經沒有工作在準備了。 */
                        portRESET_READY_PRIORITY( uxPriorityUsedOnEntry, uxTopReadyPriority );
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }

                    /* 將它塞進符合它新身分的 Ready List[新] 房間中 */
                    prvAddTaskToReadyList( pxTCB );
                }
                else
                {
                    /* 如果工作正在睡覺（Blocked）或掛起（Suspended），我們只需要改完上面的優先權變數就好，
                     * 不需要動鏈結串列。多核心下則清除搶佔標記。 */
                    #if ( configNUMBER_OF_CORES == 1 )
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }
                    #else
                    {
                        xYieldForTask = pdFALSE;
                    }
                    #endif
                }
```

#### 27.6 執行搶佔與臨界區退出

最後一關，落實剛才做出的排程決策，決定是否立刻引發 Context Switch。

```C
/* 根據前面第 2, 3 步做出的決策，落實 Yield 動作 */
                if( xYieldRequired != pdFALSE )
                {
                    /* 執行中任務的優先權被調低了，要求目前核心立刻 Yield 讓位 */
                    taskYIELD_TASK_CORE_IF_USING_PREEMPTION( pxTCB );
                }
                else
                {
                    #if ( configNUMBER_OF_CORES > 1 )
                        if( xYieldForTask != pdFALSE )
                        {
                            /* 多核心環境：某工作優先權被拉高了，去看看別的核心有沒有比它弱的工作，
                             * 強行驅逐低優先權任務，把核心讓給這個剛變強的工作。 */
                            taskYIELD_ANY_CORE_IF_USING_PREEMPTION( pxTCB );
                        }
                        else
                    #endif
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }
                }

                ( void ) uxPriorityUsedOnEntry; // 消除沒使用港口優化時的編譯器警告
            }
        }
        taskEXIT_CRITICAL(); // 安全離開臨界區，恢復中斷

        traceRETURN_vTaskPrioritySet();
    }

#endif /* INCLUDE_vTaskPrioritySet */
```

### 28. *vTaskCoreAffinitySet*

**`vTaskCoreAffinitySet()`** 是 FreeRTOS 在**多核心（SMP，對稱多處理）**架構下非常關鍵的 API，用來設定**「核心親和性（Core Affinity）」**。

簡單來說，在多核心晶片（如雙核 ESP32、多核 ARM）中，預設情況下排程器會把工作隨機指派給任意一個有空的核心執行。但有時因為硬體周邊綁定、快取記憶體優化（Cache Locality）等因素，你會希望：**「工作 A 只能在 Core 0 跑，絕對不准去 Core 1 攪和。」** 這時候你就需要透過一個 Mask（遮罩）來限制它。

而這個函數最精彩的地方，就在於**當你修改遮罩的瞬間，目標工作可能正大搖大擺地在某個核心上執行**，FreeRTOS 必須立刻實施「強制驅逐」或「重新指派排程」。

#### 28.1 條件編譯與函數入口

這個函數只有在開啟「多核心」且「啟用核心親和性功能」時才會存在。

```C
#if ( ( configNUMBER_OF_CORES > 1 ) && ( configUSE_CORE_AFFINITY == 1 ) )

    void vTaskCoreAffinitySet( const TaskHandle_t xTask,
                               UBaseType_t uxCoreAffinityMask )
    {
        TCB_t * pxTCB;
        BaseType_t xCoreID;

        traceENTER_vTaskCoreAffinitySet( xTask, uxCoreAffinityMask );

        /* 進入多核心臨界區，拿取 Spinlock，防止其他核心同時修改排程結構 */
        taskENTER_CRITICAL();
        {
            /* 還記得我們先前聊過的魔法嗎？傳入 NULL 會自動代換成當前核心正在跑的自己 */
            pxTCB = prvGetTCBFromHandle( xTask );
            configASSERT( pxTCB != NULL );

            /* 正式將新的核心親和性遮罩（例如：0x01 代表只在 Core 0 跑）寫入 TCB */
            pxTCB->uxCoreAffinityMask = uxCoreAffinityMask;
```

#### 28.2 狀況 A：目標工作「正在某個核心上執行」

如果排程器已經開跑，且這個被修改遮罩的工作此時此刻正在某個核心上爽爽執行，核心必須立刻進行安全檢查。

```C
/* 只有在排程器已經運作（xSchedulerRunning）時才需要處理即時搶佔 */
            if( xSchedulerRunning != pdFALSE )
            {
                /* 情況 A：這個被改遮罩的工作，此時此刻「正在某個核心上運行」 */
                if( taskTASK_IS_RUNNING( pxTCB ) == pdTRUE )
                {
                    /* 抓出它目前究竟在哪個核心上跑（xTaskRunState 儲存了當前的 Core ID） */
                    xCoreID = ( BaseType_t ) pxTCB->xTaskRunState;

                    /* * 【核心安全性檢查：驅逐機制】
                     * 舉例：工作原本在 Core 1 跑 (xCoreID = 1)。
                     * 你剛剛把它的遮罩改成 0x01 (只准在 Core 0 跑，二進制為 01)。
                     * 這裡進行位元運算：(01 & (1 << 1)) = (01 & 10) = 0。
                     * 結果等於 0，代表：「糟糕！它目前待的核心，已經不在新允許的名單內了！」
                     */
                    if( ( uxCoreAffinityMask & ( ( UBaseType_t ) 1U << ( UBaseType_t ) xCoreID ) ) == 0U )
                    {
                        /* 立刻發動跨核心中斷 (IPI)，強制驅逐該核心上的工作，讓它下台重新排程！ */
                        prvYieldCore( xCoreID );
                    }
                }
```

#### 28.3 狀況 B：目標工作「在台下就緒排隊」

如果工作目前沒在跑（在 Ready List 裡），但在修改遮罩前，排程器可能已經向某個核心發出過 Yield 請求了。這時候需要更換目標。

```C
/* 情況 B：這個工作目前「沒有在運行」（在 Ready List 排隊或 Blocked） */
                else
                {
                    #if ( configUSE_PREEMPTION == 1 )
                    {
                        /* * 【多核心排程的時序魔鬼細節】
                         * 當一個工作醒來時，SMP 排程器會挑選一個適合的核心並對它發出 Yield 請求（去搶佔它）。
                         * 但在那個核心還來不及跳進中斷換人上台的這短短微秒內，你突然呼叫本函數改了這工作的 Affinity！
                         * 這會導致原本被請求的核心因為「親和性限制」無法挑選這個工作了。
                         * 為了不讓這工作被遺忘，這裡呼叫 prvYieldForTask( xTask )：
                         * 核心會重新評估，挑選一個符合新遮罩規定的「全新核心」去發出 Yield 請求，確保它能順利上台。
                         */
                        prvYieldForTask( xTask );
                    }
                    #else /* 沒開啟搶佔功能 */
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }
                    #endif /* if( configUSE_PREEMPTION == 1 ) */
                }
            }
        }
        taskEXIT_CRITICAL(); // 解鎖臨界區，釋放 Spinlock

        traceRETURN_vTaskCoreAffinitySet();
    }
#endif /* 巨集結束 */
```

#### 28.4 補充

##### 28.4.1 深入理解 `prvYieldCore` 與 `prvYieldForTask` 的差異

- **`prvYieldCore( xCoreID )`（對準核心轟炸）**： 這是**精準打擊**。核心明確知道某個核心（例如 Core 1）上正在跑一個「已經違反法律（親和性不符）」的工作。於是直接向 Core 1 發送硬體中斷（IPI），強迫 Core 1 的 CPU 拋下工作，進入排程重新選人。
- **`prvYieldForTask( xTask )`（幫工作找對象）**： 這是**宏觀調度**。這個工作目前在 Ready 鏈結串列裡，我們不知道誰該讓位給它。`prvYieldForTask` 內部會去掃描全晶片所有的核心，看看哪個核心上正在跑的工作優先權最低、且最符合這個工作的新核心遮罩，然後再去對那個被挑中的核心發出 Yield 請求。

### 29. *vTaskCoreAffinityGet*

`*vTaskCoreAffinityGet*` 用來**查詢特定任務目前綁定了哪些 CPU 核心（獲取其核心親和性遮罩）**

#### 29.1 portBASE_TYPE_ENTER_CRITICAL 與 taskENTER_CRITICAL 的恩怨情仇

- 原始碼追蹤：它們在底層其實是一家人

如果我們順著 FreeRTOS 的標頭檔（如 `task.h` 或 `portable.h`）一路追查下去，你會發現這兩個巨集最終的定義是完全指向同一個底層實現的

```C
/* 在 FreeRTOS 核心的底層定義中（單核心環境） */
#define taskENTER_CRITICAL()          portENTER_CRITICAL()
#define taskEXIT_CRITICAL()           portEXIT_CRITICAL()

/* 在許多 Port 層或歷史對齊定義中 */
#define portBASE_TYPE_ENTER_CRITICAL() portENTER_CRITICAL()
#define portBASE_TYPE_EXIT_CRITICAL()  portEXIT_CRITICAL()
```

不論是 `taskENTER_CRITICAL()` 還是 `portBASE_TYPE_ENTER_CRITICAL()`，它們的終點都是 **`portENTER_CRITICAL()`**（在單核心下是關閉中斷、累加嵌套計數；在多核心 SMP 下則是拿取硬體自旋鎖 Spinlock）。

#### 29.2 為什麼核心原始碼要區分這兩種寫法？（三大原因）

##### 29.2.1 模組化邊界（Layer Separation）

FreeRTOS 的架構設計有嚴格的階層觀念：

- **Task 體系（`taskENTER_CRITICAL`）**：這是提供給**應用層開發者**、或是隸屬於高階任務排程邏輯時使用的標準 API。
- **Port 移植層（`port...`）**：這是跟特定硬體晶片架構（如 ARM Cortex-M、RISC-V、ESP32）直接打交道的底層程式碼。

當一個函數（例如 `vTaskCoreAffinityGet`）它的本質是單純讀取一個與硬體平台對齊的底層基本型態變數（`BaseType_t`）時，原作者（Richard Barry）在撰寫原始碼時，直覺上會傾向使用帶有 `port...` 字頭的臨界區巨集，來強調這是對底層平台數據的保護。

##### 29.2.2 早期版本的資料型態對齊（MISRA-C 規範與相容性）

在非常早期的 FreeRTOS 版本中，`BaseType_t` 還沒被完全統一命名，當時叫做 `portBASE_TYPE`（這就是巨集名稱裡 `portBASE_TYPE_` 的由來）。 當時的設計哲學是：如果一個函數的回傳值或主要操作對象是 `portBASE_TYPE`，為了符合 MISRA-C 的嚴格命名對齊規範，進入臨界區時就會搭配使用 `portBASE_TYPE_ENTER_CRITICAL()`。這純粹是為了代碼美學與靜態語法檢查（Lint）的歷史產物。

##### 29.2.3 不同的「程式碼維護歷史」與遞迴嵌套

FreeRTOS 經歷了超過 20 年的演進：
- `vTaskPrioritySet()` 是最元老級的核心函數（可能在 v4.0 版本就存在了），當時大範圍使用了 `taskENTER_CRITICAL()`。
- `vTaskCoreAffinityGet()` 是近期為了支援**多核心 SMP** 才新增的高階函數。

在後續新加入多核心支援時，為了在某些特定平台上優化「純讀取函數」的臨界區開銷，核心開發團隊保留了 `portBASE_TYPE_ENTER_CRITICAL` 的寫法，以便移植層（Portable Layer）的專家可以針對這種「只讀不寫」的短臨界區進行晶片級的硬體優化（雖然在大部分官方 Port 中，它們目前都被簡單地 `#define` 成同一個東西）。

### 30. *vTaskPreemptionDisable*

**`vTaskPreemptionDisable()`** 是 FreeRTOS 為了滿足某些特定即時場景（例如：需要保證任務執行一段關鍵程式碼時「絕對不被其他任務搶佔」，但又不希望大費周章去關閉全域中斷或鎖定整個排程器）所提供的「單一任務級別禁用搶佔（Task-level Preemption Disable）」功能。

直接把 TCB 結構體內部的 `xPreemptionDisable` 欄位設定為 `pdTRUE`。一旦打上這個標記，即使之後有更高優先權的任務醒來、或是系統時鐘中斷（System Tick）爆發，排程器在評估是否要進行 Context Switch 時，只要看到這個標記，就會默默退讓，直到該任務主動解除這個限制。

#### 30.1 進入臨界區與禁用搶佔標記

```C
/* 核心關鍵：進入臨界區（單核關中斷/增計數，多核 SMP 拿取 Spinlock）
         * 防止在我們正在讀取並修改 TCB 欄位的這個微秒，
         * 其他核心或中斷突然去移動、刪除或修改同一個工作的 TCB 結構。 */
        taskENTER_CRITICAL();
        {
            /* 還記得我們聊過無數次的 FreeRTOS 經典代換嗎？
             * 如果你傳入的 xTask 是 NULL，內部會自動幫你轉成「目前正在呼叫此 API 的工作自己（pxCurrentTCB）」 */
            pxTCB = prvGetTCBFromHandle( xTask );
            configASSERT( pxTCB != NULL ); // 防禦性斷言，確保指標合法

            /* 將 TCB 內的 xPreemptionDisable 欄位設為 pdTRUE。
             * 之後排程器在執行調度決策時（例如我們在 vTaskPrioritySet 看過的條件判斷），
             * 只要看到這個欄位為 pdTRUE，就不會對它發動強行搶佔（Preemption）。 */
            pxTCB->xPreemptionDisable = pdTRUE;
        }
        taskEXIT_CRITICAL(); // 迅速解鎖，安全離開臨界區
```

#### 30.2 回顧 `vTaskPrioritySet()`

```C
/* 這是 vTaskPrioritySet 內部的程式碼片段 */
else if( taskTASK_IS_RUNNING( pxTCB ) == pdTRUE )
{
    #if ( configUSE_TASK_PREEMPTION_DISABLE == 1 )
        if( pxTCB->xPreemptionDisable == pdFALSE ) // 🔴 魔鬼就在這裡！
    #endif
    {
        xYieldRequired = pdTRUE; // 只有當沒禁用搶佔時，才允許 Yield
    }
}
```

當你把一個正在執行的任務優先權調低，本來台下排隊的大哥應該要立刻「搶佔」它。但排程器在臨界區內一看：「哦！這傢伙呼叫過 `vTaskPreemptionDisable()`，它的 `xPreemptionDisable` 是 `pdTRUE` 耶！」 於是排程器就會**直接跳過** `xYieldRequired = pdTRUE;` 的設定。這隻工作就能繼續在台上安穩地把剩下的事情做完。

#### 禁用任務搶佔（本函數） vs 鎖定排程器（`vTaskSuspendAll`）有什麼差？

這兩者常常被拿來比較，它們的「防禦範圍」有著天壤之別：
- **鎖定排程器 `vTaskSuspendAll()`（地圖砲級別）**： 呼叫後，整個作業系統的排程器直接被凍結。**全系統所有工作**都無法進行 Context Switch。雖然中斷還能進來，但中斷喚醒的所有工作（例如塞進 `xPendingReadyList` 的那些工作）都必須在旁邊罰站。
- **禁用任務搶佔 `vTaskPreemptionDisable()`（精準防禦級別）**： 它只對**被指定的特定工作**（或呼叫它的自己）有效。
	- **對它自己**：別的高優先權工作醒來時，不能把它踢下台。
	- **對全系統**：如果此時有另外兩個核心（多核環境下）、或是其他不相干的低優先權工作之間需要因為中斷而調度，**完全不受影響**。系統依然在平行運作。
### 31. *vTaskPreemptionEnable*

**`vTaskPreemptionEnable()`** 是用來**解除單一任務的禁用搶佔狀態（恢復可搶佔狀態）**。

在作業系統排程理論中，當一個任務自「不可搶佔狀態（Non-preemptible State）」退出並恢復為「可搶佔狀態」時，系統可能在過去這段時間內已經積欠了數次高優先權任務的喚醒事件或時鐘中斷。因此，該函數除了清除 TCB 中的禁用標記外，最關鍵的任務是**引發補償性的延遲調度（Deferred Scheduling）**，以確保系統的即時性（Real-time Responsiveness）不受到破壞。

#### 31.1 清除禁用標記與延遲調度檢查

此區塊負責修改任務的搶佔屬性，並評估是否需要對處理器核心發出上下文切換請求。

```C
/* 【變更搶佔屬性】：將 TCB 內部的 xPreemptionDisable 屬性重置為 pdFALSE。
             * 自此時序點開始，該任務重新納入排程器的搶佔式排程（Preemptive Scheduling）評估矩陣中。 */
            pxTCB->xPreemptionDisable = pdFALSE;

            /* 驗證作業系統排程器（Scheduler）是否處於運行狀態（Running State） */
            if( xSchedulerRunning != pdFALSE )
            {
                /* 【排程狀態評估】：評估此恢復可搶佔屬性的任務，目前是否正在某個處理器核心上處於執行態（Running State） */
                if( taskTASK_IS_RUNNING( pxTCB ) == pdTRUE )
                {
                    /* 1. 自 TCB 的狀態變數中擷取當前的處理器核心識別碼。
                     * 2. 由於該任務在「不可搶佔區段（Non-preemptive Section）」執行期間，
                     * 可能已阻止了更高優先權任務的即時搶佔動作。
                     * 3. 為了補償此排程延遲，此處立即調用 prvYieldCore() 向該處理器核心發送
                     * 核間中斷（IPI，在多核下）或懸起排程中斷（如 PendSV，在單核下），
                     * 強制該核心重新觸發排程算法，以落實高優先權任務的搶佔。 */
                    xCoreID = ( BaseType_t ) pxTCB->xTaskRunState;
                    prvYieldCore( xCoreID );
                }
            }
        }
        taskEXIT_CRITICAL(); // 退出臨界區，釋放自旋鎖並還原硬體中斷屏蔽狀態
```

#### 31.2 為什麼不比對 Ready List 的優先權，就直接調用 `prvYieldCore`？

在常規的排程優化思維中，我們可能會認為應該先在臨界區內檢查「就緒鏈結串列（Ready List）中是否存在優先權高於當前任務的執行緒」，若存在才引發 `prvYieldCore`，以避免不必要的硬體開銷。然而，FreeRTOS 採取了延後求值（Lazy Evaluation）**與**臨界區最小化（Critical Section Minimization）的設計哲學：

1. **降低中斷延遲（Interrupt Latency）**： 在臨界區內遍歷 Ready List 或檢查 Bitmap 位元圖會消耗非固定時間（Non-deterministic Time）。為了縮短關閉中斷或持有 Spinlock 的時間，核心選擇直接懸起上下文切換中斷（Context Switch Interrupt）。
2. **調度器的二次驗證機制**： `prvYieldCore` 引發的排程異常（如 `PendSV_Handler`）在實際執行時，排程器大腦（`vTaskSwitchContext`）必然會去掃描 Ready List 以挑選最高優先權的任務。
	- 如果 Ready List 中**不存在**更高優先權的任務，排程器將不執行上下文切換，原任務繼續執行，此時僅耗費極少數的指令週期。
	- 如果 Ready List 中**存在**更高優先權的任務，則正好順理成章完成切換，落實即時作業系統的確定性（Determinism）。

### 32. *vTaskSuspend*

**`vTaskSuspend()`** 是 FreeRTOS 之中最重量級且邏輯最複雜的核心 API 之一，其功能是**將一個任務置於掛起狀態（Suspended State）**。

在作業系統理論中，掛起狀態代表該任務將**完全脫離排程器的調度視野**。它與阻塞狀態（Blocked）不同，阻塞狀態通常帶有超時時間（Timeout），一旦時間到期或事件發生，系統會自動將其喚醒；而掛起狀態是**無限期的（Infinite）**，除非其他任務主動調用 `vTaskResume()`，否則該任務將永遠停滯。

#### 32.1 函數入口與狀態鏈結串列移出

```C
#if ( INCLUDE_vTaskSuspend == 1 )

    void vTaskSuspend( TaskHandle_t xTaskToSuspend )
    {
        TCB_t * pxTCB;

        traceENTER_vTaskSuspend( xTaskToSuspend );

        /* 進入作業系統臨界區，保障鏈結串列操作的原子性（Atomicity），在多核下此處會獲取 Spinlock */
        taskENTER_CRITICAL();
        {
            /* 若傳入句柄為 NULL，自動轉換為執行此 API 的當前任務指標（pxCurrentTCB） */
            pxTCB = prvGetTCBFromHandle( xTaskToSuspend );
            configASSERT( pxTCB != NULL );

            traceTASK_SUSPEND( pxTCB );

            /* 【移出狀態鏈結串列】：將該任務從當前的狀態鏈結串列（例如 Ready List 或 Delayed List）中拔除。
             * 若移出後發現原先的就緒鏈結串列（Ready List）長度歸零，
             * 則調用巨集重置該優先權在排程器點位圖（Bitmap）中的狀態。 */
            if( uxListRemove( &( pxTCB->xStateListItem ) ) == ( UBaseType_t ) 0 )
            {
                taskRESET_READY_PRIORITY( pxTCB->uxPriority );
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }

            /* 【同步清理事件鏈結串列】：評估該任務是否同時正在等待某個同步事件（例如 Queue 或 Semaphore）。
             * 若其 `xEventListItem` 的貨櫃（Container）不為 NULL，代表它在某些等待隊列中，此處必須一併移出，
             * 避免該同步事件觸發時，核心誤去調度一個已經被掛起的任務。 */
            if( listLIST_ITEM_CONTAINER( &( pxTCB->xEventListItem ) ) != NULL )
            {
                ( void ) uxListRemove( &( pxTCB->xEventListItem ) );
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }

            /* 【寫入掛起鏈結串列】：正式將該任務的狀態列表項（xStateListItem）插入到全域的掛起列表末端（xSuspendedTaskList） */
            vListInsertEnd( &xSuspendedTaskList, &( pxTCB->xStateListItem ) );
```

#### 32.2 任務通知狀態重置 (Task Notification Reset)

若該任務在被掛起前正處於等待通知的阻塞狀態，核心必須在此處進行狀態回滾，避免時序混淆。

```C
#if ( configUSE_TASK_NOTIFICATIONS == 1 )
            {
                BaseType_t x;

                /* 遍歷任務通知陣列（Task Notification Array） */
                for( x = ( BaseType_t ) 0; x < ( BaseType_t ) configTASK_NOTIFICATION_ARRAY_ENTRIES; x++ )
                {
                    /* 若任務在某個通知點上處於 `taskWAITING_NOTIFICATION`（阻塞等待通知狀態） */
                    if( pxTCB->ucNotifyState[ x ] == taskWAITING_NOTIFICATION )
                    {
                        /* 由於任務已被強制移入掛起鏈結串列，它已不可能接收到原本期待的通知。
                         * 核心在此處將其狀態變更為 `taskNOT_WAITING_NOTIFICATION`，以完成狀態機的回滾與解鎖。 */
                        pxTCB->ucNotifyState[ x ] = taskNOT_WAITING_NOTIFICATION;
                    }
                }
            }
            #endif /* if ( configUSE_TASK_NOTIFICATIONS == 1 ) */
```

#### 32.3 多核心環境下的任務驅逐機制 (SMP Core Eviction)

這是多核心（SMP）環境下的核心關鍵。如果被掛起的任務此刻正在**另一個核心**上執行，系統必須在退出臨界區前對其執行**強制驅逐（Core Eviction）**。

```C
#if ( configNUMBER_OF_CORES > 1 )
            {
                if( xSchedulerRunning != pdFALSE )
                {
                    /* 由於有任務被掛起（可能影響全系統下一次需要被喚醒的任務時間），
                     * 調用 prvResetNextTaskUnblockTime() 重新計算並校正系統的下一次解除阻塞時間（Unblock Time） */
                    prvResetNextTaskUnblockTime();

                    /* 【多核心執行態檢查】：評估這個被掛起的任務，當前是否正在某個處理器核心上處於執行態（Running State） */
                    if( taskTASK_IS_RUNNING( pxTCB ) == pdTRUE )
                    {
                        /* 狀況 A：被掛起的任務就是發出此 API 的當前核心任務自己 */
                        if( pxTCB->xTaskRunState == ( BaseType_t ) portGET_CORE_ID() )
                        {
                            configASSERT( uxSchedulerSuspended == 0 );
                            /* 當前核心任務自願掛起，立即引發 API 內部的上下文切換（Yield）讓出 CPU */
                            vTaskYieldWithinAPI();
                        }
                        /* 狀況 B：【核間強制驅逐（Core Eviction）】
                         * 被掛起的任務正在「其他處理器核心」上執行。
                         * 必須在釋放 Spinlock 之前，向該核心發送核間中斷（IPI），強制該核心丟棄此任務並重新排程。
                         * 否則，若不及時驅逐，該遠端核心可能會在不知情的狀況下繼續變更狀態鏈結串列，導致核心結構損壞。 */
                        else
                        {
                            prvYieldCore( pxTCB->xTaskRunState );
                        }
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
            #endif /* #if ( configNUMBER_OF_CORES > 1 ) */
        }
        taskEXIT_CRITICAL(); // 釋放臨界區與自旋鎖
```

#### 32.4 單核心環境下的調度補償 (Uniprocessor Scheduling Resolution)

此區塊處理單核心（UP）環境下的調度邏輯，包含下一次解除阻塞時間的更新，以及當前任務（`pxCurrentTCB`）被掛起時的特殊邊界條件。

```C
#if ( configNUMBER_OF_CORES == 1 )
        {
            UBaseType_t uxCurrentListLength;

            if( xSchedulerRunning != pdFALSE )
            {
                /* 單核心環境下：重新進入短臨界區，校正系統下一次的解除阻塞時間 */
                taskENTER_CRITICAL();
                {
                    prvResetNextTaskUnblockTime();
                }
                taskEXIT_CRITICAL();
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }

            /* 【當前任務自我掛起檢查】：評估被掛起的任務是否為目前正在運行的自己 */
            if( pxTCB == pxCurrentTCB )
            {
                if( xSchedulerRunning != pdFALSE )
                {
                    /* 若排程器已開跑，自己已經被丟進掛起列表了，必須立刻引發非同步排程中斷（Yield），讓出執行權 */
                    configASSERT( uxSchedulerSuspended == 0 );
                    portYIELD_WITHIN_API();
                }
                else
                {
                    /* 【排程器未開跑時的邊界條件】：若排程器還沒跑（初始化階段），
                     * 目前指向的 pxCurrentTCB 卻被掛起了，必須調整指標轉向其他任務。 */

                    /* 依據 MISRA C 2012 規範，使用暫存變數作為獨立時序點讀取 volatile 變數 */
                    uxCurrentListLength = listCURRENT_LIST_LENGTH( &xSuspendedTaskList );

                    /* 若此時掛起列表的長度等於全系統的總任務數，代表 Ready 鏈結串列已空無一人 */
                    if( uxCurrentListLength == uxCurrentNumberOfTasks )
                    {
                        /* 全系統已無工作可跑，強行將 pxCurrentTCB 清空為 NULL，
                         * 確保下一個全新建立的任務能無條件取得指標控制權 */
                        pxCurrentTCB = NULL;
                    }
                    else
                    {
                        /* 還有其他任務就緒，調用內部排程大腦切換上下文指標 */
                        vTaskSwitchContext();
                    }
                }
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
        #endif /* #if ( configNUMBER_OF_CORES == 1 ) */

        traceRETURN_vTaskSuspend();
    }
```

### 33. *prvTaskIsTaskSuspended*

**`prvTaskIsTaskSuspended()`** 是 FreeRTOS 內部的靜態輔助函數（以 `prv` 開頭且為 `static`），主要在任務恢復（Resume）邏輯（如 `vTaskResume()`）中被調用，用來**精準判斷一個任務目前是否真的處於「掛起狀態（Suspended State）」**。

在作業系統排程理論中，這個判斷非常微妙且極具欺騙性。因為一個任務的 `xStateListItem` 如果待在掛起狀態鏈結串列（`xSuspendedTaskList`）中，**它不一定代表該任務是被用戶主動掛起的**。有一種特殊情況是：用戶調用了無限期等待的阻塞 API（例如 `vTaskDelay(portMAX_DELAY)` 且沒開啟超時），FreeRTOS 為了節省記憶體開銷，也會將該任務暫時寄存在 `xSuspendedTaskList` 裡。

#### 33.1 函數入口與防禦性斷言

```C
#if ( INCLUDE_vTaskSuspend == 1 )

    static BaseType_t prvTaskIsTaskSuspended( const TaskHandle_t xTask )
    {
        BaseType_t xReturn = pdFALSE; // 預設回傳值為假（非掛起狀態）
        const TCB_t * const pxTCB = xTask; // 透過指標轉換，將抽象句柄轉為唯讀的 TCB 結構指標

        /* * 【關鍵調用規範限制】：
         * 由於此函數內部需要存取、遍歷 `xPendingReadyList`（中斷就緒延遲鏈結串列），
         * 為了防止在多核心或中斷並發環境下發生資料時序競爭（Race Condition），
         * 呼叫此函數的外層邏輯【必須】先進入臨界區（Critical Section）。
         */

        /* 防禦性斷言：檢查傳入的任務控制區塊控制代碼是否為空。
         * 在作業系統語意中，自我查詢「目前正在執行的自己是否被掛起」是矛盾且不合邏輯的，
         * 因為能執行此代碼代表自己正處於 Running 狀態，因此 xTask 絕對不能為 NULL。 */
        configASSERT( xTask );
```

#### 33.2 三層巢狀條件判斷與狀態排除法 (State Elimination Process)

這是該函數的核心邏輯。核心透過連續三個關卡，逐一排除「偽掛起」的特殊狀態。

```C
/* 【第一道關卡：物理位置檢查】
         * 檢查任務的狀態列表項（xStateListItem）是否確實存在於全域的掛起鏈結串列（xSuspendedTaskList）中。
         * 若不在，則連基本資格都沒有，直接返回 pdFALSE。 */
        if( listIS_CONTAINED_WITHIN( &xSuspendedTaskList, &( pxTCB->xStateListItem ) ) != pdFALSE )
        {
            /* 【第二道關卡：非同步 Resume 狀態排除】
             * 檢查該任務是否同時存在於 `xPendingReadyList`（中斷就緒延遲鏈結串列）中。
             * 這裡應對的病態時序是：中斷服務程式（ISR）剛剛呼叫了 xTaskResumeFromISR() 準備喚醒該任務，
             * 任務已經被放進 Pending List 準備上台，但排程器還來不及刷新。
             * 若發現它已經在 Pending 列表中，代表它即將甦醒，核心在此處排除其掛起資格。 */
            if( listIS_CONTAINED_WITHIN( &xPendingReadyList, &( pxTCB->xEventListItem ) ) == pdFALSE )
            {
                /* 【第三道關卡：無限期阻塞（Infinite Blocked）排除】
                 * 檢查任務的事件列表項（xEventListItem）的貨櫃（Container）是否為 NULL。
                 * 1. 若不為 NULL，代表任務是因為呼叫了諸如 xQueueReceive(..., portMAX_DELAY) 的無限期等待，
                 * 它目前隸屬於某個 Queue 的事件等待隊列中（本質上是 Blocked 狀態，只是寄存在掛起列表）。
                 * 2. 若為 NULL（即條件成立），代表它沒有在等任何同步物件事件。此時進入最終評估。 */
                if( listIS_CONTAINED_WITHIN( NULL, &( pxTCB->xEventListItem ) ) != pdFALSE )
                {
                    #if ( configUSE_TASK_NOTIFICATIONS == 1 )
                    {
                        BaseType_t x;

                        /* 雖然它沒有在等任何 RTOS 同步物件（Queue/Semaphore），
                         * 但它依然有可能正在單獨阻塞等待「任務通知（Task Notification）」。
                         * 先假設它是真的掛起（xReturn = pdTRUE）。 */
                        xReturn = pdTRUE;

                        /* 遍歷任務通知陣列，檢查是否在任何一個通知通道上處於 `taskWAITING_NOTIFICATION`（阻塞等待狀態） */
                        for( x = ( BaseType_t ) 0; x < ( BaseType_t ) configTASK_NOTIFICATION_ARRAY_ENTRIES; x++ )
                        {
                            if( pxTCB->ucNotifyState[ x ] == taskWAITING_NOTIFICATION )
                            {
                                /* 若正在等待通知，代表其本質依然是 Blocked（阻塞態），而非真正的 Suspended。
                                 * 立即修正回傳值為 pdFALSE，並終止迴圈。 */
                                xReturn = pdFALSE;
                                break;
                            }
                        }
                    }
                    #else /* 若未啟用任務通知機制 */
                    {
                        /* 通過了前面三道關卡的層層剝離，且沒有任務通知的干擾，
                         * 核心判定此任務為 100% 真正的純粹掛起狀態（Suspended State）。 */
                        xReturn = pdTRUE;
                    }
                    #endif /* if ( configUSE_TASK_NOTIFICATIONS == 1 ) */
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
```

#### 33.3 補充

##### 33.3.1 為什麼 FreeRTOS 要把「無限期阻塞」與「主動掛起」的任務混存在同一個 `xSuspendedTaskList` 裡？

- **傳統作業系統的作法**：
	- 在常規的案頭作業系統（如 Linux 或 Windows）中，通常會為每一種同步狀態設計專屬的鏈結串列。例如，會有一個專門存放「因超時而阻塞」的 `DelayedTaskList`，和一個專門存放「被主動掛起」的 `SuspendedTaskList`。
- **FreeRTOS 的精簡哲學**：
	- 在微控制器（MCU）環境下，RAM 的大小極度受限（通常只有幾十到幾百 KB）。
	- `DelayedTaskList` 的內部實作是**依照時間先後順序（Wakeup Time Tick）進行排序的**。
	- 如果用戶呼叫了 `vTaskDelay(portMAX_DELAY)`，代表這個任務**永遠不需要因為時間到期而被喚醒**（超時時間為無限大）。
	- 如果強行把這個「永遠不需要定時喚醒」的任務塞進 `DelayedTaskList` 中，排程器每次在 System Tick 中斷裡檢查「現在有沒有任務到期該醒來」時，就必須被迫遍歷或略過這個永遠不會到期的節點，這會白白浪費 CPU 的運算週期，且破壞了排程器 $O(1)$ 或固定時間複雜度的確定性。
- **副作用與解法**：
	- 為此，FreeRTOS 選擇了一個聰明的做法：**直接把這些「不需要定時喚醒（無超時）」的阻塞任務，借放在 `xSuspendedTaskList` 裡面。** 因為排程器平常根本不會去掃描掛起鏈結串列，這達成了完美的效能優化。
	- 然而，這種結構複用的副作用，就是導致單憑「任務在哪條鏈結串列」無法直接判定任務的真實核心狀態。這也就是為什麼必須存在 `prvTaskIsTaskSuspended()` 這個函數——它透過檢查 `xEventListItem` 的指針貨櫃與任務通知狀態機，在軟體層面完成**二次去重與精密解碼**，這正是嵌入式作業系統在「記憶體開銷」與「演算法時間複雜度」之間取得完美平衡的教科書級典範。

##### 33.3.2 為什麼都進入三層條件後還可能在等待通知?

因為：**「任務通知（Task Notification）是 FreeRTOS 裡唯一一種『不需要依附於任何外部 List 物件』的特殊阻塞機制！」**

###### 33.3.2.1 為什麼任務通知不使用 Event List？

在 FreeRTOS 中，常規的 IPC（進程間通訊）物件與任務通知，在底層的記憶體模型與排程邏輯上有著本質上的物理差異。

1. 常規同步物件（Queue/Semaphore/Mutex）的阻塞模型
	- 當一個任務因為呼叫 `xQueueReceive( xQueue, portMAX_DELAY )` 而進入無限期阻塞時：
	- 這個任務必須去 **`xQueue` 的等待隊列（`xWriteListItem` / `xReadListItem`）** 裡面排隊。
	- 於是，任務 TCB 內部的 `xEventListItem` 指標，就會被塞進該 Queue 的鏈結串列中。
	- 此時，`listIS_CONTAINED_WITHIN( NULL, &( pxTCB->xEventListItem ) )` 就會返回 **`pdFALSE`**（因為它隸屬於 Queue，不是 NULL）。
2. 任務通知（Task Notification）的「輕量化」阻塞模型
	1. 任務通知（Task Notification）是 FreeRTOS 為了優化效能而設計的「輕量級 IPC」。它的核心哲學是：**直接將通知控制變數（陣列、狀態機）內嵌在任務自己的 TCB 結構體中，不建立任何外部的鏈結串列物件。**
	2. 當一個任務呼叫 `xTaskNotifyWait(..., portMAX_DELAY)` 進入無限期阻塞時：
	3. **它不需要去任何地方排隊**。因為它是「被動等待別人來敲自己的門」，全系統沒有一個全域的「Notification List」存在。
	4. 核心只會做兩件事：
		1. 將 TCB 內部的狀態改為 `pxTCB->ucNotifyState = taskWAITING_NOTIFICATION;`。
		2. 直接把這個任務的 `xStateListItem` 拔掉，塞進掛起列表 `xSuspendedTaskList` 寄存。
	5. **關鍵來了**：在這個過程中，這個任務的 **`xEventListItem` 根本沒有被觸碰過！它依然保持著未連結狀態（也就是指向 NULL）！**

```C
/* 第一層：任務在 xSuspendedTaskList 裡嗎？ */
if( listIS_CONTAINED_WITHIN( &xSuspendedTaskList, &( pxTCB->xStateListItem ) ) != pdFALSE )
{
    /* 🔴 通過：這隻任務目前確實因為「無超時時間」而被寄存在掛起列表中 */

    /* 第二層：中斷有沒有剛好要喚醒它（在 PendingReady 嗎）？ */
    if( listIS_CONTAINED_WITHIN( &xPendingReadyList, &( pxTCB->xEventListItem ) ) == pdFALSE )
    {
        /* 🔴 通過：中斷服務程式（ISR）目前沒有試圖喚醒它 */

        /* 第三層：它的 xEventListItem 是 NULL 嗎？（沒有在等 Queue/Semaphore） */
        if( listIS_CONTAINED_WITHIN( NULL, &( pxTCB->xEventListItem ) ) != pdFALSE )
        {
            /* 🔴 通過！排程器此時心想：
             * 「它在掛起列表，而且它沒有在等任何 Queue，也沒有在等任何 Semaphore...」
             * 「那它一定就是被用戶手動呼叫 vTaskSuspend() 掛起的囉？」
             * * 🛑 慢著！還有最後一種微小的可能：
             * 如果它是因為呼叫了 xTaskNotifyWait(..., portMAX_DELAY) 呢？
             * 因為任務通知不使用外部 List，所以它的 xEventListItem 確實也是 NULL！
             */

            xReturn = pdTRUE; // 先假設它是真的被手動掛起

            /* 最終防線：親自去它的 TCB 肚子裡，檢查任務通知的狀態機陣列 */
            for( x = ( BaseType_t ) 0; x < ( BaseType_t ) configTASK_NOTIFICATION_ARRAY_ENTRIES; x++ )
            {
                if( pxTCB->ucNotifyState[ x ] == taskWAITING_NOTIFICATION )
                {
                    /* 抓到了！它雖然沒有等任何外部物件（第三層過關），
                     * 但它其實是在無限期等待「任務通知」！它的本質是 Blocked，不是 Suspended！ */
                    xReturn = pdFALSE; // 駁回掛起判定
                    break;
                }
            }
        }
    }
}
```

###### 33.3.2.2 總結

這個函數之所以要寫得這麼繁瑣，是因為在計算機科學中，**「狀態（State）」與「物理儲存位置（Memory Container）」不一定是一對一完全對等。**

FreeRTOS 為了追求極致的 RAM 優化：
- 把「被動手動掛起（`vTaskSuspend`）」的任務
- 和「無限期等待任務通知（`xTaskNotifyWait`）」的任務

**在物理上都塞進了同一個 `xSuspendedTaskList` 貨櫃裡，且兩者的 `xEventListItem` 都指向了 NULL。** 因此，前三層條件判斷只能幫核心過濾掉常規的 Queue/Semaphore 阻塞。面對任務通知這種「隱形」的阻塞機制，排程器必須突破表面資料結構的盲點，直接深入到 TCB 內部的通知狀態機陣列進行微觀檢查，才能給出 100% 精準的答案。

### 34. *vTaskResume*

**`vTaskResume()`** 是與 `vTaskSuspend()` 完美對應的核心 API，其功能是**將一個處於掛起狀態（Suspended State）的任務重新喚醒，使其恢復到就緒狀態（Ready State）**。

在作業系統排程理論中，恢復（Resume）一個任務看似只是簡單的鏈結串列搬移，但其實存在許多時序上的高難度邊界條件。特別是結合了我們上一題深入解析的 `prvTaskIsTaskSuspended()` 狀態判定，此函數在進入臨界區（Critical Section）後，必須確認目標任務是真的「被用戶主動掛起」，而非「無限期阻塞等待通知或 IPC 佇列」。此外，它在多核心（SMP）環境下的安全檢查與補償性排程切換（Yield）邏輯，也展現了嚴密的並行控制哲學。

#### 34.1 函數入口與靜態語義檢查

```C
#if ( INCLUDE_vTaskSuspend == 1 )

    void vTaskResume( TaskHandle_t xTaskToResume )
    {
        /* 將傳入的抽象控制代碼（Handle）強制轉換為內部的任務控制區塊指標（TCB_t *） */
        TCB_t * const pxTCB = xTaskToResume;

        traceENTER_vTaskResume( xTaskToResume );

        /* 防禦性斷言：傳入的控制代碼絕對不能為 NULL。
         * 在作業系統語義中，一個正在執行（Running）的任務不可能呼叫 API 來「喚醒自己」，
         * 因為自己本來就處於活動狀態，此操作屬於邏輯矛盾。 */
        configASSERT( xTaskToResume );

        /* 根據處理器核心數量進行編譯期條件分支對齊 */
        #if ( configNUMBER_OF_CORES == 1 )

            /* 【單核心環境過濾】：
             * 排除「目標任務為當前執行任務（pxCurrentTCB）」以及「指標為空」的無效操作。 */
            if( ( pxTCB != pxCurrentTCB ) && ( pxTCB != NULL ) )
        #else

            /* 【多核心環境過濾】：
             * 多核（SMP）下，目標任務可能正在另一個核心上跑（Running）。雖然恢復一個正在跑的任務是不合法的，
             * 但在「未進入臨界區、未拿自旋鎖」前，直接讀取其運行狀態（Run State）是不安全的時序競爭。
             * 因此多核下此處僅簡單過濾 NULL，隨後將其移入臨界區內做精準狀態檢查。 */
            if( pxTCB != NULL )
        #endif
```

#### 34.2 臨界區保護與精密狀態確認 (State Verification)

```C
{
            /* 進入作業系統臨界區。在單核下屏蔽硬體中斷；
             * 在多核 SMP 下，此處會獲取全系統的硬體自旋鎖（Spinlock），以凍結全域鏈結串列狀態。 */
            taskENTER_CRITICAL();
            {
                /* * 【核心安全防線】：調用先前拆解過的內部輔助函數。
                 * 1. 查核該任務的 xStateListItem 是否在 xSuspendedTaskList 中。
                 * 2. 排除中斷非同步喚醒的時序。
                 * 3. 排除因無限期等待（portMAX_DELAY）而借存於掛起列表的「無限期阻塞任務」（IPC/任務通知）。
                 * 只有通過層層排除，確認其為「純粹的掛起狀態」，此條件才會成立！
                 */
                if( prvTaskIsTaskSuspended( pxTCB ) != pdFALSE )
                {
                    traceTASK_RESUME( pxTCB );
```

#### 34.3 鏈結串列變更與補償性搶佔 (List Migration & Preemption)

當確認任務符合喚醒資格後，執行狀態鏈結串列的物理搬移，並通知排程器進行搶佔評估。

```C
/* 【物理鏈結串列搬移】：將任務的狀態列表項（xStateListItem）從
                     * 掛起鏈結串列（xSuspendedTaskList）中徹底拔除。 */
                    ( void ) uxListRemove( &( pxTCB->xStateListItem ) );
                    
                    /* 【重新加入調度】：將該 TCB 依照其當前優先權，重新插入對應的
                     * 就緒鏈結串列（pxReadyTasksLists）中，並更新優先權點位圖（Bitmap）。 */
                    prvAddTaskToReadyList( pxTCB );

                    /* * 【補償性搶佔評估 (Compensatory Preemption)】
                     * 剛被喚醒的任務優先權可能極高。
                     * 此巨集在單核下會比對優先權，若高於當前任務則懸起 PendSV 排程中斷；
                     * 在多核 SMP 下，則會評估全系統所有核心的執行狀態，挑選一個優先權最低的核心，
                     * 發送核間中斷（IPI）強制其 Yield，以確保即時系統的搶佔確定性。
                     */
                    taskYIELD_ANY_CORE_IF_USING_PREEMPTION( pxTCB );
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
            taskEXIT_CRITICAL(); // 釋放臨界區與自旋鎖，允許排程中斷即時爆發並執行上下文切換
        }
```

### 35. *xTaskResumeFromISR*

**`xTaskResumeFromISR()`** 是 FreeRTOS 專門提供給中斷服務程式（ISR, Interrupt Service Routine）使用的任務喚醒 API。

在作業系統理論中，ISR 的執行環境受到極嚴格的限制：**ISR 必須以極短的時間執行完畢，且絕對不允許進入阻塞狀態（Blocking）**。因此，所有在 ISR 中操作核心資料結構的 API，都必須採取與常規 API（如 `vTaskResume`）完全不同的同步與排程策略。

這段原始碼展示了硬體中斷與 RTOS 核心之間最驚心動魄的協調過程：它必須應對「中斷嵌套（Interrupt Nesting）」**的硬體限制，並且引入了**「延時就緒列表（Pending Ready List）」機制，來解決當排程器被鎖定時，中斷無法直接修改就緒列表的棘手問題。

#### 35.1 條件編譯、變數宣告與硬體優先權斷言

```C
#if ( ( INCLUDE_xTaskResumeFromISR == 1 ) && ( INCLUDE_vTaskSuspend == 1 ) )

    BaseType_t xTaskResumeFromISR( TaskHandle_t xTaskToResume )
    {
        BaseType_t xYieldRequired = pdFALSE; // 排程讓步標記，回傳給使用者決定是否立即執行上下文切換
        TCB_t * const pxTCB = xTaskToResume; // 指標型態轉換
        UBaseType_t uxSavedInterruptStatus;  // 儲存硬體中斷屏蔽狀態的變數（ISR 臨界區必備）

        traceENTER_xTaskResumeFromISR( xTaskToResume );

        configASSERT( xTaskToResume );

        /* * 【硬體安全防線：中斷優先權驗證】
         * 在支援中斷嵌套（Interrupt Nesting）的晶片（如 ARM Cortex-M）中，核心定義了「最大系統呼叫優先權」。
         * 高於此優先權的高急迫中斷是永久啟用的，但也絕對【禁止】呼叫任何 FreeRTOS API。
         * 此處透過硬體暫存器比對，若發現此 ISR 的硬體優先權過高，將直接觸發斷言失敗（Assert Fault）。
         */
        portASSERT_IF_INTERRUPT_PRIORITY_INVALID();
```

#### 35.2 ISR 專用臨界區與任務狀態確認

```C
/* * 【ISR 專用臨界區】：
         * 傳統的 taskENTER_CRITICAL() 會直接關閉硬體中斷（或將優先權拉到最高），這在 ISR 內部會破壞嵌套狀態。
         * taskENTER_CRITICAL_FROM_ISR() 採用「讀取-儲存-屏蔽」機制：
         * 它先讀取當前硬體的中斷屏蔽狀態並存在 uxSavedInterruptStatus 中，然後將中斷屏蔽拉高，
         * 藉此防範更高級別的中斷進來打擾，確保接下來資料操作的原子性（Atomicity）。
         */
        uxSavedInterruptStatus = taskENTER_CRITICAL_FROM_ISR();
        {
            /* 呼叫先前詳細解析過的狀態機過濾函數，確認該任務是否真的處於「純粹的掛起狀態」 */
            if( prvTaskIsTaskSuspended( pxTCB ) != pdFALSE )
            {
                traceTASK_RESUME_FROM_ISR( pxTCB );
```

#### 35.3 分支 A：排程器未鎖定（常規直接喚醒，限單核）

當核心的排程器處於正常運作狀態時，中斷可以直接介入並移轉任務的狀態鏈結串列。

```C
/* 【排程器可控性檢查】：評估目前應用層是否呼叫了 vTaskSuspendAll()？
                 * 若 uxSchedulerSuspended == 0，代表排程器未被鎖定，就緒列表（Ready List）可以被安全存取。 */
                if( uxSchedulerSuspended == ( UBaseType_t ) 0U )
                {
                    #if ( configNUMBER_OF_CORES == 1 )
                    {
                        /* 【單核心搶佔評估】：若剛被喚醒的任務優先權高於當前核心上執行的任務 */
                        if( pxTCB->uxPriority > pxCurrentTCB->uxPriority )
                        {
                            xYieldRequired = pdTRUE; // 標記需要引發上下文切換

                            /* * 【防禦性排程延遲】：
                             * 在 `xYieldPendings[0]` 點位記下 pdTRUE。
                             * 這是為了防範工程師在寫 ISR 時，忘記在結束前呼叫 `portYIELD_FROM_ISR()`。
                             * 核心在此處留下一條導火線，在下一次 System Tick 觸發時會強行補做 Context Switch。
                             */
                            xYieldPendings[ 0 ] = pdTRUE;
                        }
                        else
                        {
                            mtCOVERAGE_TEST_MARKER();
                        }
                    }
                    #endif /* #if ( configNUMBER_OF_CORES == 1 ) */

                    /* 將任務從掛起列表（xSuspendedTaskList）移除，並直接塞入對應优先權的 Ready List 之中 */
                    ( void ) uxListRemove( &( pxTCB->xStateListItem ) );
                    prvAddTaskToReadyList( pxTCB );
                }
```

#### 35.4 分支 B：排程器被鎖定（二級緩衝機制）

如果中斷爆發的這一瞬間，應用層剛好呼叫了 `vTaskSuspendAll()` 鎖定了排程器，中斷絕對禁止修改 Ready List，此時核心必須啟動「二級緩衝」。

```C
else
                {
                    /* * 【核心二級緩衝機制：延時就緒（Deferred Ready）】
                     * 應用層目前鎖定了排程器（uxSchedulerSuspended > 0），代表核心禁止任何人觸碰 Ready List。
                     * 但中斷此時又必須把任務喚醒，怎麼辦？
                     * 核心在此處繞過 Ready List，直接將該任務的事件列表項（xEventListItem）
                     * 強行插入到全域的【中斷就緒延遲鏈結串列（xPendingReadyList）】末端。
                     * 當稍後應用層呼叫 vTaskResumeAll() 解鎖排程器時，核心大腦會自動去掃描這條鏈結串列，
                     * 把裡面的任務批次搬移回 Ready List。這是一種精妙的非同步緩衝設計。
                     */
                    vListInsertEnd( &( xPendingReadyList ), &( pxTCB->xEventListItem ) );
                }
```

#### 35.5 多核心排程補償與退出臨界區

此區塊負責多核心（SMP）環境下的跨核心搶佔評估，並還原中斷屏蔽狀態，完成函數返回。

```C
#if ( ( configNUMBER_OF_CORES > 1 ) && ( configUSE_PREEMPTION == 1 ) )
                {
                    /* 【多核心宏觀調度】：
                     * 由於在多核心環境下，可能有多個 CPU 正在跑不同的任務。
                     * 調用 prvYieldForTask() 去掃描全晶片，挑選一個最適合讓位給 pxTCB 的核心並發出 IPI 中斷。 */
                    prvYieldForTask( pxTCB );

                    /* 檢查當前執行此 ISR 的 CPU 核心，是否在剛才的調度中被標記了需要讓步？
                     * 若是，則將 xYieldRequired 設為 pdTRUE，通知外層 ISR 準備進行切換。 */
                    if( xYieldPendings[ portGET_CORE_ID() ] != pdFALSE )
                    {
                        xYieldRequired = pdTRUE;
                    }
                }
                #endif /* #if ( ( configNUMBER_OF_CORES > 1 ) && ( configUSE_PREEMPTION == 1 ) ) */
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
        /* 退出 ISR 臨界區：將剛才儲存的歷史中斷狀態（uxSavedInterruptStatus）還原回硬體暫存器 */
        taskEXIT_CRITICAL_FROM_ISR( uxSavedInterruptStatus );

        traceRETURN_xTaskResumeFromISR( xYieldRequired );

        /* 回傳布林值：
         * pdTRUE  -> 台下有更高優先權的任務醒了，提示使用者在 ISR 結束前應呼叫 portYIELD_FROM_ISR()。
         * pdFALSE -> 喚醒的任務優先權較低，不影響當前排程。 */
        return xYieldRequired;
    }

#endif /* 巨集條件結束 */
```

#### 35.6 為什麼不論單核或多核，當 `uxSchedulerSuspended` 啟用時，中斷絕不能直接動 Ready List？

1. **破壞排程器的確定性限制**：
	當應用層任務（Task）呼叫 `vTaskSuspendAll()` 時，它可能正在執行需要高度維護鏈結串列完整性的操作（例如：正在遍歷 Ready List 以計算當前任務數量）。此時，Ready List 的指標正處於短暫的變動狀態。

	在計算機作業系統理論中，這涉及了兩種不同級別的「鎖定機制」：
	1. **硬體級鎖（Critical Section, CS）**：透過關閉中斷實現，開銷極大，會破壞即時性（引發中斷延遲）。
	2. **軟體級鎖（Suspend Scheduler）**：透過 `uxSchedulerSuspended++` 實現，不關中斷，開銷極小，對即時性極其友好。
	3. 當應用層任務（Task A）需要執行一些長距離、不希望被別的任務搶佔的操作（例如遍歷 Ready List 刪除節點）時，它有兩種選擇。FreeRTOS 為了保證硬體中斷能被即時響應，官方強烈建議使用 `vTaskSuspendAll()`
	
2. **中斷的無情搶佔**：
	硬體中斷（ISR）的優先權高於所有任務。如果中斷在此刻爆發，且直接呼叫 `prvAddTaskToReadyList()` 強行往 Ready List 裡塞入節點，這會直接在記憶體層面將正在被 Task 修改的鏈結串列給**物理性踩壞（Memory Corruption）**。
	
3. **解法：空間換時間的非同步緩衝**：
	為了徹底切斷這種時序衝突，FreeRTOS 開闢了第三地：**`xPendingReadyList`**。
	- 當排程器被鎖定時，中斷來了，它**完全不碰** Ready List，而是把任務丟進 `xPendingReadyList`。因為此時全系統只有這個 ISR 會去動這條隊列，所以操作是 100% 絕對安全的。
	- 任務被丟進 `xPendingReadyList` 後，因為它還不在 Ready List 裡，所以它「名義上醒了，但實際上還無法被核心排程執行」。
	- 直到中斷結束，控制權回到原本的 Task，Task 完成工作並呼叫 `vTaskResumeAll()`。在解鎖的函數內部，排程器才會用同步（Synchronous）的方式，把 `xPendingReadyList` 裡面的任務一個個安全地搬移回 Ready List。

#### 35.7 xPendingReadyList 的雙端訪問權限與生命週期

在 FreeRTOS 的架構中，`xPendingReadyList` 是一條跨越「中斷地帶（ISR Space）」與「任務地帶（Task Space）」的橋樑。它採用了非對稱的存取權限設計。

這條鏈結串列的生命週期非常嚴謹，我們可以將它視為一個由中斷負責儲存、由排程器負責清理的暫存器：

- 唯寫端（Producers）：硬體中斷服務程式（ISR）
	當系統滿足以下兩個條件時，中斷 API 會將任務的 `xEventListItem` 塞進 `xPendingReadyList` 的**末端**：
	1. 某個中斷事件（如 DMA 傳輸完成、定時器溢出）試圖喚醒或就緒某個任務。
	2. 此時台下的應用層剛好呼叫了 `vTaskSuspendAll()`，導致 `uxSchedulerSuspended > 0`。
	**中斷的行為**：它只管把任務「丟進去登記」，登記完後 ISR 就立刻結束並退出，中斷**絕對不會**當場去處理這條鏈結串列裡面的其他工作。
- 唯讀/清除端（Consumer）：應用層排程器大腦（Task 境內）
	當應用層的關鍵代碼執行完畢，呼叫 **`xTaskResumeAll()`** 準備解鎖排程器時，真正的魔鬼細節才開始。

### 36. *prvCreateIdleTasks*

**`prvCreateIdleTasks()`**是 FreeRTOS 在系統啟動、排程器準備開跑（`vTaskStartScheduler()`）時，在後台自動呼叫的內部初始化函數。

在作業系統排程理論中，空閒任務（Idle Task）是系統的「保底執行緒」。當 Ready List 裡面空無一人、沒有任何使用者任務想執行時，CPU 絕不能停下來不執行指令，排程器此時就會強制調度 Idle Task 上台。Idle Task 的優先權通常是全系統最低的 0（`tskIDLE_PRIORITY`），主要負責回收已刪除任務的記憶體（清除 TCB 與 Stack）、低功耗休眠（Low Power Tickless Mode Entry），或是在多核心環境下充當各個核心的「專屬背鍋俠」。

這段原始碼展示了 FreeRTOS 為了適應現代**對稱多處理架構（SMP, Symmetric Multiprocessing）**，在核心設計上所做出的重大進化：它引入了「主動空閒任務（Active Idle Task）」與「被動空閒任務（Passive Idle Task）」的分工，並且相容了靜態（Static）與動態（Dynamic）記憶體配置。

#### 36.1 任務名稱安全拷貝與 Null-Termination

```C
static BaseType_t prvCreateIdleTasks( void )
{
    BaseType_t xReturn = pdPASS;
    BaseType_t xCoreID;
    char cIdleName[ configMAX_TASK_NAME_LEN ] = { 0 }; // 任務名稱緩衝區
    TaskFunction_t pxIdleTaskFunction = NULL;          // 空閒任務函數指標
    UBaseType_t xIdleTaskNameIndex;

    /* * 【MISRA C 安全規範與字串拷貝】：
     * 為了避免緩衝區溢位，迴圈長度上限被扣除了 `taskRESERVED_TASK_NAME_LENGTH` 預留空間。
     * 將全域巨集定義的 `configIDLE_TASK_NAME`（通常是 "IDLE"）逐字元複製到 cIdleName。
     */
    for( xIdleTaskNameIndex = 0U; xIdleTaskNameIndex < ( configMAX_TASK_NAME_LEN - taskRESERVED_TASK_NAME_LENGTH ); xIdleTaskNameIndex++ )
    {
        cIdleName[ xIdleTaskNameIndex ] = configIDLE_TASK_NAME[ xIdleTaskNameIndex ];

        if( cIdleName[ xIdleTaskNameIndex ] == ( char ) 0x00 )
        {
            break;
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
    }

    /* 確保字串結尾強制加上 '\0'，防止字串處理函數發生非法記憶體越界讀取 */
    cIdleName[ xIdleTaskNameIndex ] = '\0';
```

#### 36.2 ## 多核心 SMP 下的空閒任務分工與名稱後綴 (Task Name Suffixing)

```C
/* 【多核心排程補償】：遍歷全晶片所有的處理器核心（configNUMBER_OF_CORES） */
    for( xCoreID = ( BaseType_t ) 0; xCoreID < ( BaseType_t ) configNUMBER_OF_CORES; xCoreID++ )
    {
        #if ( configNUMBER_OF_CORES == 1 )
        {
            /* 單核心（UP）架構下：全系統只需要一個標準的空閒任務主函數 */
            pxIdleTaskFunction = &prvIdleTask;
        }
        #else /* 多核心（SMP）架構下 */
        {
            /* * 【SMP 核心設計：主動與被動空閒任務分工】
             * 在多核環境中，作業系統必須保證「每一個核心在沒事做時，都有獨立的空閒任務可以跑」。
             * - Core 0 指派 `prvIdleTask`（主動空閒任務）：除了保底外，還負責清理被刪除任務的系統記憶體。
             * - 其他 Core 指派 `prvPassiveIdleTask`（被動空閒任務）：只做純粹的無窮迴圈背鍋，不分擔系統清理工作，
             * 避免多個核心同時清理鏈結串列引發複雜的死結（Deadlock）。
             */
            if( xCoreID == 0 )
            {
                pxIdleTaskFunction = &prvIdleTask;
            }
            else
            {
                pxIdleTaskFunction = &prvPassiveIdleTask;
            }
        }
        #endif /* #if ( configNUMBER_OF_CORES == 1 ) */

        /* 【除錯名稱後綴附加】：多核環境下，如果名字都叫 "IDLE" 會讓開發者在追蹤 Trace 或使用
         * 模擬器（如 J-Link/Ozone）除錯時感到混淆。此處將核心編號（0~9）以 ASCII 字元型態
         * 覆蓋到剛才預留的字串尾端，使其變為 "IDLE0"、"IDLE1" 等。 */
        #if ( configNUMBER_OF_CORES > 1 )
        {
            cIdleName[ xIdleTaskNameIndex ] = ( char ) ( xCoreID + '0' );
            cIdleName[ xIdleTaskNameIndex + 1U ] = '\0';
        }
        #endif /* if ( configNUMBER_OF_CORES > 1 ) */
```

#### 36.3 分支 A：靜態記憶體配置下的空閒任務建立 (Static Allocation)

如果使用者在配置中啟用了靜態配置（`configSUPPORT_STATIC_ALLOCATION == 1`），系統將透過回呼函數（Callback）向應用層索取 TCB 與 Stack 空間。

```C
#if ( configSUPPORT_STATIC_ALLOCATION == 1 )
        {
            StaticTask_t * pxIdleTaskTCBBuffer = NULL;
            StackType_t * pxIdleTaskStackBuffer = NULL;
            configSTACK_DEPTH_TYPE uxIdleTaskStackSize;

            /* 核心呼叫使用者必須實作的應用層回呼函數（Hook），獲取靜態配置的 RAM 位址 */
            #if ( configNUMBER_OF_CORES == 1 )
            {
                vApplicationGetIdleTaskMemory( &pxIdleTaskTCBBuffer, &pxIdleTaskStackBuffer, &uxIdleTaskStackSize );
            }
            #else
            {
                /* SMP 架構下，主動與被動空閒任務的記憶體分開獲取 */
                if( xCoreID == 0 )
                {
                    vApplicationGetIdleTaskMemory( &pxIdleTaskTCBBuffer, &pxIdleTaskStackBuffer, &uxIdleTaskStackSize );
                }
                else
                {
                    vApplicationGetPassiveIdleTaskMemory( &pxIdleTaskTCBBuffer, &pxIdleTaskStackBuffer, &uxIdleTaskStackSize, ( BaseType_t ) ( xCoreID - 1 ) );
                }
            }
            #endif /* if ( configNUMBER_OF_CORES == 1 ) */
            
            /* 呼叫靜態任務建立 API。優先權參數傳入 `portPRIVILEGE_BIT`。
             * 備註：在具有 MPU（記憶體保護單元）的硬體架構中，這會強行賦予 Idle Task 核心特權模式（Privileged Mode）。
             * 雖然它的排程優先權本質上是 0，但它擁有核心級的硬體存取權限。 */
            xIdleTaskHandles[ xCoreID ] = xTaskCreateStatic( pxIdleTaskFunction,
                                                             cIdleName,
                                                             uxIdleTaskStackSize,
                                                             ( void * ) NULL,
                                                             portPRIVILEGE_BIT, 
                                                             pxIdleTaskStackBuffer,
                                                             pxIdleTaskTCBBuffer );

            if( xIdleTaskHandles[ xCoreID ] != NULL )
            {
                xReturn = pdPASS;
            }
            else
            {
                xReturn = pdFAIL;
            }
        }
```

#### 36.4 分支 B：動態記憶體配置下的空閒任務建立與建立失敗防護 (Dynamic Allocation)

如果採用常規的動態配置，核心會直接在堆疊（Heap）中切出預設的最小 Stack 空間給空閒任務。

```C
#else /* 採用動態配置系統記憶體（configSUPPORT_STATIC_ALLOCATION == 0） */
        {
            /* 核心呼叫常規的 xTaskCreate，直接從 Heap 中動態分派 TCB 與最小限制的 Stack（configMINIMAL_STACK_SIZE） */
            xReturn = xTaskCreate( pxIdleTaskFunction,
                                   cIdleName,
                                   configMINIMAL_STACK_SIZE,
                                   ( void * ) NULL,
                                   portPRIVILEGE_BIT, 
                                   &xIdleTaskHandles[ xCoreID ] );
        }
        #endif /* configSUPPORT_STATIC_ALLOCATION */

        /* 【集體建立狀態安全稽核】：
         * 如果在多核心環境下，其中任何一個核心的空閒任務因為 RAM 不足而建立失敗（pdPASS != pdFAIL），
         * 系統將立即中斷迴圈，這會導致隨後的 vTaskStartScheduler() 啟動失敗，防止作業系統帶著殘缺的核心強行開跑。 */
        if( xReturn != pdPASS )
        {
            break;
        }
```

#### 36.5 多核心啟動前的初始調度綁定 (Initial Core Affinity Tethering)

當空閒任務成功建立後，在多核心排程器正式運轉前，必須將對應的任務控制區塊（TCB）強行綁定到各自的核心執行狀態機上。

```C
else
        {
            #if ( configNUMBER_OF_CORES == 1 )
            {
                mtCOVERAGE_TEST_MARKER();
            }
            #else
            {
                /* * 【SMP 啟動前的核心強制隸屬綁定 (Affinity Tethering)】：
                 * 這是多核心作業系統非常核心的引導程序（Bootstrapping）。
                 * 1. 強制設定該任務控制區塊中的 `xTaskRunState` 數值等於特定的 `xCoreID`。
                 * 2. 直接將各核心的當前執行任務指標 `pxCurrentTCBs[xCoreID]` 初始化為這個剛建立好的空閒任務。
                 * 這樣一來，等一下排程器一打開、硬體定時器開始計時的那一瞬間，
                 * 每個處理器核心一睜開眼，就有一隻已經就位、合法的保底任務（"IDLEx"）可以立刻執行，
                 * 避免多核心並發初始化時，各個 CPU 核心因為讀到 NULL 指標而集體 HardFault 崩潰。
                 */
                xIdleTaskHandles[ xCoreID ]->xTaskRunState = xCoreID;
                pxCurrentTCBs[ xCoreID ] = xIdleTaskHandles[ xCoreID ];
            }
            #endif
        }
    } // For 迴圈結束

    return xReturn;
}
```

#### 36.6 多核心（SMP）作業系統中的「空閒不對稱性（Idle Asymmetry）」

##### 36.6.1 在傳統的單核心 FreeRTOS 中，`prvIdleTask` 身上背負著兩大職責：
1. **垃圾回收（Garbage Collection）**：當使用者在應用層呼叫 `vTaskDelete()` 刪除某個任務時，FreeRTOS 為了追求效率，中斷和目前任務是不會當場去釋放該任務的堆疊與 TCB 記憶體的。任務只會被丟進 `xTasksWaitingTermination`（等待銷毀鏈結串列），然後由 `prvIdleTask` 在後台慢慢釋放記憶體。
2. **低功耗管理（Tickless Idle）**：當發現全系統沒有任務要跑，且距離下一次中斷還有很久時，Idle Task 會引導 CPU 進入深睡眠。

##### 36.6.2 多核心下的並行災難

如果 FreeRTOS 在多核心環境下，讓 4 個核心的空閒任務都執行一模一樣的 `prvIdleTask`： 這意味著 4 個 CPU 核心在沒事做的時候，會**同時跳進垃圾回收邏輯，去存取、爭搶、修改同一個 `xTasksWaitingTermination` 鏈結串列**。為了保護這條鏈結串列，4 個核心就必須在這裡瘋狂爭奪 Spinlock（自旋鎖），這會引發嚴重的**鎖競爭（Lock Contention）**，平白浪費晶片的功耗與效能。

FreeRTOS 的核心架構師在這裡展現了精妙的「不對稱設計」：

- **Core 0** 執行 **主動空閒任務（`prvIdleTask`）**，專職負責全系統的記憶體資源回收。
- **Core 1 ~ N** 執行 **被動空閒任務（`prvPassiveIdleTask`）**，它們在沒事做的時候，只會待在自己的底層無窮迴圈裡。它們**絕對不碰**記憶體回收鏈結串列，只在原地靜靜等待核間中斷（IPI）或新的搶佔任務到來。

### 37. *vTaskStartScheduler*

在你的 `main()` 函數完成了硬體初始化、並呼叫 `xTaskCreate()` 建立好你的各個任務之後，最後一步必定是呼叫 `vTaskStartScheduler()`。

在作業系統理論中，這叫做「排程器引導啟動（Scheduler Bootstrapping）」**。當你跨過這條線之前，CPU 只是一個在執行傳統硬體 `main()` 函數的單純晶片；但當這隻函數執行到尾聲時，核心會**物理性地強行改寫 CPU 的堆疊指標（SP）與程式計數器（PC），將控制權正式移交給作業系統排程器大腦。從此，整個微控制器便進入了多工搶佔式的即時作業系統（RTOS）新世界。

#### 37.1 親和性斷言、空閒任務與軟體定時器任務的建立

此區塊在排程器開跑前，先完成核心生存所需的「保底基礎設施」建立。

```C
void vTaskStartScheduler( void )
{
    BaseType_t xReturn;

    traceENTER_vTaskStartScheduler();

    /* 【多核心親和性安全性檢查】：
     * 如果啟用了核心親和性（Core Affinity），代表任務可以用 Bitmask（位元遮罩）來指定自己要在哪些 CPU 核心上跑。
     * 核心在此處進行靜態斷言：儲存 Mask 的資料型態（UBaseType_t）其二進位總位元數，
     * 絕對不能小於硬體真實的核心總數（configNUMBER_OF_CORES），否則 Mask 會放不下。 */
    #if ( configUSE_CORE_AFFINITY == 1 ) && ( configNUMBER_OF_CORES > 1 )
    {
        configASSERT( ( sizeof( UBaseType_t ) * taskBITS_PER_BYTE ) >= configNUMBER_OF_CORES );
    }
    #endif /* #if ( configUSE_CORE_AFFINITY == 1 ) && ( configNUMBER_OF_CORES > 1 ) */

    /* 【建立全系統保底空閒任務】：
     * 呼叫我們上一題剛解析完的函數，為每個 CPU 核心分派專屬的 IDLE 任務。 */
    xReturn = prvCreateIdleTasks();

    /* 【建立守護行程：軟體定時器服務任務】：
     * 如果使用者在 FreeRTOSConfig.h 啟用了定時器機制，
     * 核心會在這裡自動幫你呼叫 xTimerCreateTimerTask() 來建立全域定時器服務執行緒（Tmr Svc）。 */
    #if ( configUSE_TIMERS == 1 )
    {
        if( xReturn == pdPASS )
        {
            xReturn = xTimerCreateTimerTask();
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
    }
    #endif /* configUSE_TIMERS */
```

#### 37.2 物理性關閉中斷與全域核心變數初始化

如果基礎設施任務建立成功（記憶體足夠），核心開始初始化排程大腦的全域狀態。

```C
if( xReturn == pdPASS )
    {
        /* 提供給使用者擴充核心邏輯的巨集 hook */
        #ifdef FREERTOS_TASKS_C_ADDITIONS_INIT
        {
            freertos_tasks_c_additions_init();
        }
        #endif

        /* * 🚨【最高安全防線：物理關閉全域中斷】
         * 在這裡，核心透過 portDISABLE_INTERRUPTS() 直接拉高 CPU 的中斷屏蔽暫存器。
         * 為什麼要在此刻關閉中斷？
         * 因為在接下來切換到第一個任務的關鍵引導期間，硬體的 System Tick（時鐘中斷）絕對不能提早爆發，
         * 否則排程器大腦在狀態還沒完全就位時被非同步搶佔，會導致指針錯亂而瞬間死機。
         * * 💡【不用擔心！那中斷什麼時候才會重新開啟？】
         * FreeRTOS 在當初建立任務、初始化它們的 TCB 虛擬堆疊時，就已經在該任務堆疊的
         * 狀態暫存器位置（如 ARM 的 xPSR）裡，將「啟用中斷」的那一個 Bit 預先寫死成 1 了！
         * 當隨後第一個任務上台、執行堆疊彈出（Pop）時，CPU 會硬體自動、順便地恢復全域中斷！
         */
        portDISABLE_INTERRUPTS();

        /* 如果啟用了執行緒區域儲存（Thread Local Storage, TLS），將 C 執行期環境指標指向第一個運作任務的 TLS 區塊 */
        #if ( configUSE_C_RUNTIME_TLS_SUPPORT == 1 )
        {
            configSET_TLS_BLOCK( pxCurrentTCB->xTLSBlock );
        }
        #endif

        /* 初始化核心三大全域時序變數 */
        xNextTaskUnblockTime = portMAX_DELAY;            // 初始化下一個要醒來的任務時間為無限遠
        xSchedulerRunning = pdTRUE;                      // 🚨 正式點亮排程器運轉綠燈！
        xTickCount = ( TickType_t ) configINITIAL_TICK_COUNT; // 初始化時間滴答計數（通常為 0）

        /* 設定硬體的高精度計時器，用來統計每個任務佔用 CPU 時間的百分比（CPU Usage） */
        portCONFIGURE_TIMER_FOR_RUN_TIME_STATS();

        traceTASK_SWITCHED_IN();
        traceSTARTING_SCHEDULER( xIdleTaskHandles );
```

#### 37.3 跳入硬體移植層：驚心動魄的跨界大躍進 (Port Launching)

此區塊呼叫硬體移植層接口，正式結束 `main()` 的物理生命。

```C
/* * 🚀【跳入硬體移植層：不歸路（The Point of No Return）】
         * `xPortStartScheduler()` 是一個完全由組合語言（Assembly）與底層硬體暫存器堆砌出來的函數。
         * 在這個函數內部，核心會執行以下極端的硬體操作：
         * 1. 配置硬體定時器（如 ARM Systick），設定好每秒產生幾次中斷（如 1000Hz）。
         * 2. 觸發底層排程軟體中斷（如 PendSV）。
         * 3. 組合語言會去撈取 `pxCurrentTCB` 裡面指向的第一個任務的 Stack Pointer（SP）。
         * 4. 執行一連串的 POP 指令，把虛擬堆疊裡的暫存器數值強行灌進 CPU 的物理暫存器中（R0~R15）。
         * 5. 最終 PC（程式計數器）被強行指向該任務的函數入口——作業系統排程正式引爆！
         * * 備備註：因為這個函數會直接把 CPU 的執行上下文洗掉，所以理論上它【絕對不會返回】。
         */
        ( void ) xPortStartScheduler();
    }
```

### 38. *vTaskEndScheduler*

在作業系統排程理論中，這叫做「排程器終止與環境復原（Scheduler De-initialization / Tear-down）」**。它的功能與 `vTaskStartScheduler()` 恰好完全相反：它是要把已經全速運轉的作業系統「徹底關閉」，物理性地停掉硬體 Systick 定時器中斷，並將 CPU 的控制權**重新交還給最初引導系統的裸機裸奔（Bare-metal）環境。

🚨 **注意**：在大多數嵌入式微控制器（如 ARM Cortex-M）中，核心**並沒有實作**這個函數（底層 `vPortEndScheduler()` 通常是空的或直接死迴圈）。因為在硬體世界裡，RTOS 一旦開跑，除非斷電或重置（Reset），否則沒有理由退回到 `main()`。此函數主要用於 **Windows/Linux Simulator 模擬環境**，或是某些支援將執行緒控制權完整還原給傳統非作業系統 Bootloader 的特殊高階晶片。

#### 38.1 條件編譯與核心守護任務（Timer & Idle）的強制銷毀

此區塊在排程器關閉前，手動將核心啟動時自動建立的保底基礎設施執行緒全部抹除。

```C
void vTaskEndScheduler( void )
{
    traceENTER_vTaskEndScheduler();

    /* 【功能相依性檢查】：終銷毀任務必須依賴 vTaskDelete() 的機制。
     * 如果使用者在 config 裡把 INCLUDE_vTaskDelete 設為 0，此處將無法清理核心任務。 */
    #if ( INCLUDE_vTaskDelete == 1 )
    {
        BaseType_t xCoreID;

        /* 【銷毀軟體定時器服務任務】：
         * 如果當初有建立，此處呼叫內部接口獲取其 Handle，並呼叫 vTaskDelete 強行將其從就緒列表中拔除並銷毀。 */
        #if ( configUSE_TIMERS == 1 )
        {
            /* Delete the timer task created by the kernel. */
            vTaskDelete( xTimerGetTimerDaemonTaskHandle() );
        }
        #endif /* #if ( configUSE_TIMERS == 1 ) */

        /* 【銷毀全核心空閒任務】：
         * 遍歷所有 CPU 核心的陣列，將不論是主動還是被動的 IDLE 任務（"IDLE0", "IDLE1" 等）通通強制刪除。 */
        for( xCoreID = 0; xCoreID < ( BaseType_t ) configNUMBER_OF_CORES; xCoreID++ )
        {
            vTaskDelete( xIdleTaskHandles[ xCoreID ] );
        }
```

#### 38.2 終極垃圾清理：防範記憶體流失（Memory Leak）

因為負責清理殘局的 Idle 任務在上一瞬間已經被我們物理消滅了，核心必須在此刻親自動手清空終止隊列。

```C
/* * 🚨【核心關鍵救援：手動垃圾回收】
         * 在 FreeRTOS 理論中，被刪除任務的 TCB 與 Stack 記憶體是由 Idle 任務在後台慢慢回收的。
         * 但是！就在上面幾行程式碼中，我們已經把全系統的 Idle 任務都刪除了！
         * 這意味著，如果目前 `xTasksWaitingTermination`（終止等待鏈結串列）裡面還躺著之前被使用者
         * 刪除、但 Idle 任務還來不及釋放的任務，它們的記憶體將永久遺失（Memory Leak）。
         * 因此，核心在這裡「反客為主」，在沒有 Idle 任務的情況下，親自呼叫
         * prvCheckTasksWaitingTermination()，用同步的方式暴力清空這條鏈結串列，釋放所有殘留的 Heap RAM。
         */
        prvCheckTasksWaitingTermination();
    }
    #endif /* #if ( INCLUDE_vTaskDelete == 1 ) */
```

#### 38.3 物理切斷硬體中斷與狀態封印

核心在此處切斷與系統時間滴答相關的硬體中斷，並將作業系統狀態變數拉回離線狀態。

```C
/* * 【關閉硬體計時中斷】：
     * 呼叫 portDISABLE_INTERRUPTS() 拉高屏蔽，防止在此刻排程器即將瓦解的危險時期，
     * 硬體 Systick 或者是定時器中斷突然爆發引發二次排程。
     */
    portDISABLE_INTERRUPTS();
    
    /* 🚨【正式關閉排程器運轉綠燈】：
     * 將全域狀態變數 xSchedulerRunning 設為 pdFALSE。
     * 從這一刻起，全系統的時鐘中斷增量函數（xTaskIncrementTick）與 IPC 機制將徹底拒絕服務。 */
    xSchedulerRunning = pdFALSE;
```

#### 38.4 跳入硬體解構層：復原 CPU 初始上下文 (Port Tear-down)

```C
/* * 🛑【跳入硬體移植層：復原裸機（The Return to Bare-Metal）】
     * 呼叫硬體特定的解構函數 `vPortEndScheduler()`。
     * 在這個函數內部（通常在模擬器環境，如 Windows Port），它會執行以下反向操作：
     * 1. 徹底關閉並註銷作業系統專用的硬體定時器中斷。
     * 2. 恢復系統最初始的中斷向量表（Vector Table ISRs）。
     * 3. 恢復當初進入 `vTaskStartScheduler()` 之前，在 `main()` 函數裡保存的
     * 原始 CPU 堆疊指標（SP）與主執行緒暫存器狀態。
     * 4. 物理執行跳轉指令（Jump），硬生生退回到當初呼叫 vTaskStartScheduler() 的下一行代碼！
     */
    vPortEndScheduler();

    /* * 【備註】：
     * 這個函數必須由某個正在執行的現存任務（Task）主動呼叫。
     * 當 vPortEndScheduler() 真的成功退回 main() 之後，
     * 呼叫它的那個任務本身並不會被 FreeRTOS 核心自動刪除。
     * 應用程式（main 境內）必須自己寫代碼去釋放或處理那個結束後的任務殘留。
     */
    traceRETURN_vTaskEndScheduler();
}
```

#### 38.5 vTaskEndScheduler 為何只刪除 Idle tasks 和 timer task

在作業系統的設計哲學中，**「誰建立的資源，就該由誰負責銷毀」**（Who allocates, deallocates）。核心不碰使用者 Task 的 Ready List，原因可以拆解為以下三個層次：

##### 38.5.1 核心任務與使用者任務的「所有權（Ownership）」本質不同

- 仔細觀察 `vTaskEndScheduler()` 的刪除對象：
	- 它刪除了 **Idle Task** (`xIdleTaskHandles`)
	- 它刪除了 **Timer Task** (`xTimerGetTimerDaemonTaskHandle()`)
- 為什麼這兩個要刪？
	-  因為這兩個任務是**當初排程器啟動時，核心（OS Kernel）自己偷偷在後台建立的**。它們的 TCB 和 Stack 所消耗的 Heap 空間，是 OS 的「內部私有財產」。既然 OS 現在要關門了，它當然必須把自己的私有財產清理乾淨，否則就是 OS 自己造成的 Memory Leak。
- 為什麼使用者的 Task 不能由 OS 刪除？
	 - 你在 `main()` 或其他地方呼叫 `xTaskCreate()` 建立的任務，屬於應用層（Application Space）的資產。FreeRTOS 核心只負責「排程」它們，並不「擁有」它們。

##### 38.5.2 致命的記憶體災難：靜態配置（Static Allocation）的硬體盲點

如果 FreeRTOS 真的寫了一個 `while` 迴圈，把 Ready List 裡的所有 Task 通通丟進 `vTaskDelete()`：

- 在 FreeRTOS 中，任務可以用兩種方式建立：
	1. **動態配置 (`xTaskCreate`)**：TCB 和 Stack 是從 Heap 裡切出來的。
	2. **靜態配置 (`xTaskCreateStatic`)**：工程師自己在全域變數、或者陣列裡開闢空間，將記憶體位址傳給核心。
- 💥 如果核心硬去刪除，會發生什麼事？
	1. 當 `vTaskDelete()` 收到一個靜態配置的任務時，如果核心試圖去釋放（Free）它，因為那塊記憶體**根本不是從 Heap 裡配置出來的**（它可能是一塊唯讀區段或固定的全域 SRAM），這會直接導致 Heap 管理器（如 `heap_4.c`）的內部鏈結指針徹底踩壞，晶片瞬間觸發 **HardFault 崩潰**。
	2. 核心根本無法百分之百確定使用者的 Ready List 裡面，哪一個任務是動態的、哪一個是靜態的，因此「不亂動使用者的資源」是核心自我保護的最高原則。

##### 38.5.3 使用者任務背後的「附屬資源（Attached Resources）」黑洞

- 一個正在運作的使用者 Task，它身上可能綁著非常多作業系統周邊元件，例如：
	- 它可能鎖定著一個 **Mutex（互斥鎖）**。
	- 它可能正在等待一個 **Queue（佇列）**。
	- 它可能配置了一塊當地的記憶體緩衝區（Buffer）。

- 如果 `vTaskEndScheduler()` 簡單粗暴地把這個 Task 從 Ready List 拔掉並消滅：
	- 該 Task 身上鎖定的 Mutex 將**永遠不會被釋放**，變成記憶體中的孤兒。
	- 其他還在等待該資源的硬體或裸機環境將陷入永久死結。


### 39. *vTaskSuspendAll*

 **`vTaskSuspendAll()`** 是 FreeRTOS 用來實作「排程器軟體鎖」的核心 API。

如同我們在前面所有討論中提到的，它的核心功能是：**「告知排程大腦，目前要暫停任務切換（搶佔），但保持硬體中斷（ISR）的大門敞開。」**
這段原始碼是整個 FreeRTOS 核心中價值最高、也最難讀懂的魔鬼細節之一。它包含了兩套完全不同的並行控制邏輯：

1. **單核心環境（UP）**：展現了令人嘆為觀止的、**不需要進 CS（關中斷）卻能達成 100% 執行緒安全**的「純暫存器原子時序魔術」。
2. **多核心環境（SMP）**：展示了跨多個 CPU 核心時，為了防止中斷與其他核心同時修改變數，結合了中斷遮罩、任務鎖（Task Lock）、中斷鎖（ISR Lock）以及任務核心遷移檢查的**地獄級多執行緒同步網**。

#### 39.1 單核心（UP）分支：神級的無鎖原子時序魔術

當系統只有一個 CPU 核心時，核心故意**不關中斷**，直接進行全域變數遞增。官方用了一大段註解解釋為什麼這樣是絕對安全的。
注意 `uxSchedulerSuspended` 是 volatile 的所以是從記憶體讀取
##### 39.1.1 為什麼全域變數一定會被載入暫存器（Register）

在微處理器（如 ARM Cortex-M 或 RISC-V）的硬體設計中，它們屬於 **「載入/儲存架構（Load/Store Architecture）」**。

核心硬體限制：

**CPU 的運算單元（ALU）無法直接對 SRAM 記憶體裡的資料進行算術運算。** 當你寫 `uxSchedulerSuspended = uxSchedulerSuspended + 1;` 時，在處理器的物理世界裡，這行 C 語言**絕對不可能**透過一條指令完成。編譯器（如 GCC）必須將其翻譯成至少三條組合語言指令：

```
LDR R0, [R1]  ; 1. Load:  把暫存器 R1 指向的 SRAM 變數值，讀入 CPU 暫存器 R0
ADD R0, R0, #1 ; 2. Modify: 在 CPU 內部的 ALU 將 R0 的數值加 1
STR R0, [R1]  ; 3. Store:  把運算完的暫存器 R0 的值，寫回 SRAM 記憶體中
```

```C
void vTaskSuspendAll( void )
{
    traceENTER_vTaskSuspendAll();

    #if ( configNUMBER_OF_CORES == 1 )
    {
        /* * 💡【單核心無鎖安全的神級註解翻譯與拆解】：
         * 為什麼 `uxSchedulerSuspended++` 這麼多指令的操作，不進 CS 關中斷卻不會被別的 Task 踩壞？
         * * 假設目前 Task A 執行 `vTaskSuspendAll()`，CPU 將執行以下 4 個組合語言步驟：
         * 1. LOAD:  將變數 `uxSchedulerSuspended` (值為 0) 讀入 CPU 暫存器 R0。
         * 💥【就是這一瞬間】：中斷爆發！中斷強制切換上下文，讓 Task B 上台跑。
         * 2. 關鍵在於：Task B 跑的時候，去看記憶體裡的 `uxSchedulerSuspended` 依然是 0（因為 Task A 還沒寫回去）。
         * 3. Task B 執行完下台，輪到 Task A 重新上台。
         * 4. 🚨【核心物理鐵律】：Task A 重新上台時，硬體會 100% 還原 Task A 當初下台時的 CPU 暫存器！
         * 所以 Task A 的 R0 裡面依然躺著當初讀到的「0」。
         * 5. INCREMENT: Task A 繼續執行 R0 = R0 + 1 (R0 變 1)。
         * 6. STORE:     Task A 將 R0 (值為 1) 寫回記憶體 `uxSchedulerSuspended`。
         * * 結論：即使中間被打斷、換了無數個 Task，只要這個變數最後被寫回記憶體的那一刻是原子性的（單一指令），
         * 數據就絕對不會被別的 Task 踩爛！這省去了關閉中斷的巨大開銷！
         */

        /* 編譯器屏障（Compiler Barrier）：防止極端優化的編譯器亂序調整程式碼的上下順序 */
        portSOFTWARE_BARRIER();

        /* 將排程器鎖定計數加 1。採用累加計數（Increment）是為了支援函數巢狀呼叫（Nesting）。
         * 也就是說，如果你的代碼連續呼叫兩次 vTaskSuspendAll()，核心會記錄它，你必須呼叫兩次 xTaskResumeAll() 才能真正解鎖。 */
        uxSchedulerSuspended = ( UBaseType_t ) ( uxSchedulerSuspended + 1U );

        /* 記憶體屏障（Memory Barrier）：物理強制 CPU 必須先完成這行記憶體寫入，才能執行後續的代碼 */
        portMEMORY_BARRIER();
    }
```
#### 39.2 多核心（SMP）分支：防禦性斷言與中斷遮罩鎖定

在多核心環境下，上面的「單核暫存器魔術」直接失效。因為多個核心可以同時去搬運和修改記憶體。核心在此處啟動最高規格的防禦。

```C
#else /* #if ( configNUMBER_OF_CORES != 1 ) -> 進入多核心（SMP）地獄模式 */
    {
        UBaseType_t ulState;
        BaseType_t xCoreID;

        /* 【防禦性檢查】：此函數的本質是鎖定排程器，絕對禁止在中斷服務程式（ISR）內部呼叫！ */
        portASSERT_IF_IN_ISR();

        if( xSchedulerRunning != pdFALSE )
        {
            /* * 🚨【多核心防線一：短暫關閉中斷】
             * 在多核環境下，修改 `uxSchedulerSuspended` 前必須同時取得 Task 鎖與 ISR 鎖。
             * 為了防止目前的 Task 在剛拿到 Task 鎖、還來不及加加變數時，就被本地中斷打斷而引發上下文切換，
             * 核心在此處必須「先手動關閉本地中斷」，並保存原本的中斷狀態。
             */
            ulState = portSET_INTERRUPT_MASK();

            /* 獲取目前程式碼正處在哪一個 CPU 核心的 ID (0 ~ N) */
            xCoreID = ( BaseType_t ) portGET_CORE_ID();

            /* 【嚴重斷言】：此函數不允許在已經進入臨界區（Critical Section）的情況下重複呼叫 */
            configASSERT( portGET_CRITICAL_NESTING_COUNT( xCoreID ) == 0 );

            portSOFTWARE_BARRIER();

            /* 🔒【多核心防線二：奪取 Task 級自旋鎖（Spinlock）】
             * 防止此時其他 CPU 核心（例如 Core 1）上的代碼同時來呼叫 vTaskSuspendAll() 修改全域排程器狀態。
             */
            portGET_TASK_LOCK( xCoreID );
```

#### 39.3 多核心（SMP）分支：任務跨核心遷移（Migration）的動態稽核

此區塊是 FreeRTOS SMP 最精妙的設計。在鎖定排程器前，核心必須動態檢查這個任務是不是在剛剛搶鎖的期間，被別的核心給「偷渡搬遷」了。

```C
/* 如果這是這顆核心第一次準備鎖定排程器（從 0 變 1） */
            if( uxSchedulerSuspended == 0U )
            {
                /* * 🕵️‍♂️【多核心驚悚時序：跨核心狀態稽核】
                 * 在多核 SMP 環境下，當前這個 Task 在排隊等待上面 `portGET_TASK_LOCK` 釋放的這段微秒間，
                 * 別的核心（如 Core 0）可能剛好爆發中斷，並把目前這個 Task 進行了狀態轉換或重新排程。
                 * `prvCheckForRunStateChange()` 會去稽核目前這個 Task 的最新狀態，
                 * 確保它的執行核心和狀態沒有在搶鎖的間隙被別的核心強行修改。
                 */
                prvCheckForRunStateChange();
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }

            /* * 🚨【重新校正核心 ID】：
             * 承接上一步！因為 `prvCheckForRunStateChange()` 在執行過程中，
             * 為了配合多核心平衡排程，極有可能把目前這個 Task「從 Core 0 物理遷移（Migrate）到了 Core 1」上執行！
             * 任務是在完全不同的 CPU 核心上甦醒並繼續執行下面程式碼的！
             * 因此，此處必須重新呼叫 `portGET_CORE_ID()`，重新撈取最新的、正確的核心 ID，
             * 否則等一下解錯別顆核心的鎖，系統會立刻死鎖（Deadlock）當機！
             */
            xCoreID = ( BaseType_t ) portGET_CORE_ID();
```

#### 39.4 多核心（SMP）分支：中斷鎖定、變數實質修改與安全解鎖

在確認了核心 ID 並確保沒有其他核心競爭後，短暫鎖定中斷 Spinlock，完成變數異動並全面解鎖。

```C
/* 🔒【多核心防線三：奪取 ISR 級自旋鎖】
             * 這是為了解決我們前面幾題討論的「中斷隨時會進來寫入 xPendingReadyList」的狀況。
             * 拿到這個鎖，代表此時全晶片沒有任何中斷 API（FromISR）敢來碰排程變數。
             */
            portGET_ISR_LOCK( xCoreID );

            /* 全面安全！全系統不論是其他 CPU 核心、還是非同步的中斷，通通被三道大鎖隔絕在外。
             * 核心終於可以安全地對 `uxSchedulerSuspended` 進行原子加 1 異動。 */
            ++uxSchedulerSuspended;
            
            /* 🔓【安全釋放】：解開中斷自旋鎖 */
            portRELEASE_ISR_LOCK( xCoreID );

            /* 🔓【安全釋放】：恢復本地中斷開關。
             * 雖然此時中斷大門重新打開了，但因為 `uxSchedulerSuspended` 已經成功變成了非 0 值，
             * 接下來任何爆發的中斷一看到這個警示燈，都會乖乖把就緒任務改塞進 `xPendingReadyList`，
             * 達成了完美的排程鎖定保護。 */
            portCLEAR_INTERRUPT_MASK( ulState );
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
    }
    #endif /* #if ( configNUMBER_OF_CORES == 1 ) */

    traceRETURN_vTaskSuspendAll();
}
```

#### 39.5 portSOFTWARE_BARRIER 與 portMEMORY_BARRIER 差在哪?

##### 39.5.1 portSOFTWARE_BARRIER（軟體/編譯器屏障）

- **防禦對象**：**編譯器（Compiler，如 GCC、Clang）的自作聰明。**
- **本質**：它是寫給編譯器看的**優化限制令**。在 GCC 中，它的底層實作通常是：

```C
__asm__ volatile("" ::: "memory");
```
- 為什麼需要它？
	- 現代編譯器非常聰明（開啟 `-O2` 或 `-O3` 優化時）。如果它發現你在呼叫 `vTaskSuspendAll()` 之前或之後有一些無關緊要的程式碼，編譯器為了追求速度，可能會把 `uxSchedulerSuspended++` 的組合語言順序**往前挪或往後挪（Instruction Reordering）**。`portSOFTWARE_BARRIER()` 的存在就是向編譯器下達禁令：「編譯器聽著！這是一面隱形的牆。所有寫在牆上面的程式碼，編譯出來的組合語言絕對不能跨過這面牆挪到下面去；下面的也絕對不能挪上來！」
- **硬體代價**：**零開銷（Zero Hardware Cost）。** 它在編譯完成後，不會產生任何一條物理 CPU 指令，它只影響編譯階段組合語言的排列順序。

##### 39.5.2 portMEMORY_BARRIER（硬體/記憶體屏障）

- **防禦對象**：**CPU 晶片硬體（Processor Hardware）的亂序執行。**
- **本質**：它會物理性地產生一條**硬體 CPU 指令**（在 ARM 上通常是 **`DMB` (Data Memory Barrier)** 或 **`DSB` (Data Synchronization Barrier)**）。
- **為什麼需要它？** 即使編譯器很乖，生出了順序正確的組合語言。但現代的高階 CPU 晶片（如帶有緩衝區、流水線或多核架構的晶片）在執行指令時，為了追求極致效能，內部的硬體邏輯會採用 **「亂序執行（Out-of-Order Execution）」**。CPU 可能会把 `STR R0, [R1]`（把暫存器數值寫回 SRAM）這條指令丟進晶片的「寫入緩衝區（Write Buffer）」排隊，然後 CPU 就急著去執行後面的業務程式碼了。這會導致記憶體裡的 `uxSchedulerSuspended` **在物理上還沒真正更新，後面的業務代碼就已經跑了**。`portMEMORY_BARRIER()` 的存在是向 CPU 硬體下達物理禁令：「CPU 晶片聽著！立刻給我執行 DMB/DSB 指令！在 `uxSchedulerSuspended` 這一行沒有實實在在地把數值寫進 SRAM 記憶體顆粒之前，硬體流水線必須給我全部暫停，絕對不准執行後面的任何一條指令！」
- **硬體代價**：**有開銷（Has Hardware Cost）。** 它會讓 CPU 流水線短暫空轉（Stall），確保記憶體總線（Bus）的絕對同步。

##### 39.5.3 Barrier 的本質差異

| 特性 | `portSOFTWARE_BARRIER` | `portMEMORY_BARRIER` |
|------|----------------------|---------------------|
| **防止對象** | 僅編譯器重排 | 編譯器 + CPU 硬體重排 |
| **作用層級** | 軟體層（編譯期） | 軟體層 + 硬體層（執行期） |
| **效能開銷** | 極低（只影響編譯產出） | 較高（會發出實際的 fence 指令） |
| **適用架構** | 強記憶體排序架構（如 x86、Cortex-M） | 弱記憶體排序架構（如 ARM A 系列、RISC-V） |

##### 39.5.4 詳細補充

[[Details about Barriers]]

#### 39.6 `portSET_INTERRUPT_MASK` 詳解

##### 39.6.1 核心功能與物理本質（以 ARM Cortex-M 為例）

在微處理器世界中，關閉中斷有兩種做法：一種是粗暴地「關閉全域所有中斷」（如修改 `PRIMASK` 暫存器）；另一種則是 FreeRTOS 偏愛的「智慧型局部屏蔽」。

在 ARM Cortex-M3/M4/M7 等硬體架構下，`portSET_INTERRUPT_MASK()` 的底層通常是在操作 CPU 內部的 **`BASEPRI`（基礎中斷優先權遮罩暂存器）**：

物理執行程序：
1. **讀取舊狀態**：CPU 先把當前 `BASEPRI` 暫存器的數值讀出來，存進一個區域變數（例如 `ulReturn`）。
2. **寫入新遮罩值**：將全域設定的 `configMAX_SYSCALL_INTERRUPT_PRIORITY`（核心全面接管的最高中斷優先權等級）硬寫入 `BASEPRI`。
3. **返回**：將第 1 步保存的舊數值返回給呼叫者。

執行完這行指令的當下，CPU 將會**物理性地拒絕響應**所有優先權低於該設定線的硬體中斷，但比這條線更高階的硬體中斷（例如超高精度的電機控制中斷、非屏蔽中斷 NMI）依然能自由爆發，這確保了 RTOS 不會拖累頂級硬體的即時響應。

##### 39.6.2 它與常見的 `taskENTER_CRITICAL()` 有什麼區別？

在 FreeRTOS 中，你常常會看到 `taskENTER_CRITICAL()` 和 `portSET_INTERRUPT_MASK()`，它們都可以用來關閉中斷，但兩者的應用舞台完全不同：

| **特性對比**       | **taskENTER_CRITICAL()**                       | **portSET_INTERRUPT_MASK()**                      |
| -------------- | ---------------------------------------------- | ------------------------------------------------- |
| **執行環境**       | **只能在 Task 應用層（Task Space）呼叫**。                | **可以在硬體中斷（ISR Space）內部呼叫**。                       |
| **記帳機制**       | 依賴全域任務控制塊或排程器的全域變數 `uxCriticalNesting` 進行計數遞增。 | 依賴 C 語言函數堆疊（Stack）上的**區域變數**（如 `ulState`）保存暫存器數值。 |
| **多核心（SMP）行為** | 會去搶奪重量級的 `Task Lock` 自旋鎖。                      | 只純粹修改目前這顆 CPU 核心的物理暫存器。                           |
| **典型後台調用**     | `vTaskSuspendAll()`、`xQueueSend()`。            | 所有帶有後綴的中斷安全 API，如 `xQueueSendFromISR()`。          |

### 40. *prvGetExpectedIdleTime*

**`prvGetExpectedIdleTime()`** 是 FreeRTOS 實作低功耗休眠機制（Tickless Idle Mode）時的核心大腦。

在作業系統的低功耗管理理論中，**「Tickless Idle」** 是一個非常高級的技術。常規情況下，RTOS 每毫秒都會被硬體時鐘（System Tick 中斷）喚醒一次，這意味著晶片每秒要醒來 1000 次，根本無法進入深睡眠。而 Tickless 機制允許系統在沒事做的時候，**暫時把硬體時鐘中斷關掉（或者拉長中斷時間）**，讓晶片一口氣睡上幾百毫秒甚至幾秒鐘。

而 `prvGetExpectedIdleTime()` 的唯一職責，就是**在晶片準備躺下睡覺前，無情且精準地審查全系統的排程狀態，計算出「我們現在到底最多可以睡多久（預期的空閒時間）才不會耽誤下一個任務醒來？」**。如果系統發現有任何潛在的任務需要立即執行，它就會回傳 `0`，一票否決這次的睡眠計畫。

#### 40.1 條件編譯與高優先權就緒任務的位元級稽核

函數一開頭，會根據使用者的優化設定，去檢查 Ready List 裡面是不是藏著比 Idle 優先權（0）更高的任務正在排隊。

```C
#if ( configUSE_TICKLESS_IDLE != 0 )

    static TickType_t prvGetExpectedIdleTime( void )
    {
        TickType_t xReturn;
        BaseType_t xHigherPriorityReadyTasks = pdFALSE;

        /* * 【第一關：檢查是否有高優先權任務就緒（針對非搶佔式/合作式排程）】
         * 理論上，如果現在是 Idle Task 在跑，Ready List 裡應該沒有比它優先權更高的任務。
         * 但如果使用者關閉了搶佔（configUSE_PREEMPTION = 0），高優先權任務就算醒來，
         * 也必須等目前的任務主動讓位才能跑。此時，高優先權任務會卡在 Ready List 裡。
         * 核心在睡前必須檢查這種特殊狀況，否則一睡著就會耽誤到人家。
         */
        #if ( configUSE_PORT_OPTIMISED_TASK_SELECTION == 0 )
        {
            /* 常規模式：uxTopReadyPriority 是一個純數字（如 3 代表目前就緒的最高優先權是 3）。
             * 如果這個數字大於 tskIDLE_PRIORITY（即 0），代表上方有大哥在排隊！ */
            if( uxTopReadyPriority > tskIDLE_PRIORITY )
            {
                xHigherPriorityReadyTasks = pdTRUE;
            }
        }
        #else
        {
            const UBaseType_t uxLeastSignificantBit = ( UBaseType_t ) 0x01;

            /* * 💡【硬體優化模式：Bit-map 遮罩運算】
             * 當啟用了硬體優化時，`uxTopReadyPriority` 變成了一個 32-bit 的「位元圖（Bit-map）」。
             * 最低位元（Bit 0 = 0x01）代表優先權 0 (Idle 優先權)。
             * 如果除了最右邊那一位（0x01）以外，還有其他任何一個位元被舉起（值大於 0x01），
             * 就代表上方（如優先權 1、2、3）有任務已經就緒了！
             */
            if( uxTopReadyPriority > uxLeastSignificantBit )
            {
                xHigherPriorityReadyTasks = pdTRUE;
            }
        }
        #endif /* if ( configUSE_PORT_OPTIMISED_TASK_SELECTION == 0 ) */
```

##### 40.1.1 **`configUSE_PORT_OPTIMISED_TASK_SELECTION`** 是什麼？

在 FreeRTOS 中代表的就是：**「排程器在挑選下一個要執行的任務時，要使用『純軟體查表』還是『硬體指令加速』？」**

在即時作業系統（RTOS）中，每次晶片要切換任務（排程）時，大腦最常做的一件事就是：**「從目前所有準備好的任務（Ready List）中，找出優先權最高的是哪一個？」**

這個巨集就是用來決定**怎麼找出這個最高優先權**的開關：

1. 當設為 `0`：通用軟體選型（純軟體跑迴圈）
	1. **代表什麼**：核心會用最傳統的 C 語言 `for` 或 `while` 迴圈，從最高的優先權（例如 31）開始一個一個往下檢查，直到看到哪一個優先權的陣列裡面有任務。
	2. **物理代價**：時間複雜度是 $O(N)$。如果你的系統開了 32 個優先權，最壞情況下 CPU 要連續做 32 次判斷（跑 32 次迴圈）才能找到。優先權開越多，排程器花的時間就越長，而且每次切換任務的時間不固定。
	3. **唯一優點**：不挑晶片。因為是純 C 語言寫的，所以不論是 8-bit 的古董 8051 還是最新的晶片，通通都能跑。
2. 當設為 `1`：硬體優化選型（一條硬體指令秒殺）
	1. **代表什麼**：核心會在全域維護一個 32 位元的整數（我們叫它 Bit-map 位元圖）。
		1. 優先權 0 有任務，Bit 0 就變成 1。
		2. 優先權 5 有任務，Bit 5 就變成 1。
	2. **物理代價**：排程大腦**完全不跑迴圈**，而是直接調用特定 CPU 核心內部自帶的「前導零計數（Count Leading Zeros）」硬體指令（在 ARM Cortex-M 晶片上叫 **`CLZ`** 組合語言指令）。
	3. **極致效能**：這條硬體指令可以在 **1 個 CPU 週期（納秒級）** 內，直接用晶片電路硬體算出這個 32-bit 變數中，最左邊的 1 出現在第幾個位元。時間複雜度是絕對的 **$O(1)$**，不論你開了多少任務，CPU 永遠用一條指令一瞬間鎖定最高優先權。
	4. **代價與限制**：
		1. 晶片必須支援這種硬體指令（常規的 ARM Cortex-M3/M4/M7、RISC-V 高階核都有支援）。
		2. 因為一個整數只有 32 位元，所以全系統的**任務優先權總數上限不能超過 32**（若在 64 位元架構下則不超過 64）。

### 41. *xTaskResumeAll*

**`xTaskResumeAll()`** 是整個 FreeRTOS 核心中時序最複雜、精細度最高的 API 之一，負責解除由 `vTaskSuspendAll()` 所設定的「排程器軟體鎖」。

當你的程式碼離開了不能被搶佔的區域，呼叫 `xTaskResumeAll()` 時，排程器大腦就會重新上線。但請注意：**它不只是單純把警示燈關掉，它還必須收拾在「排程器鎖定期間」全系統累積的所有非同步殘局。**

在鎖定期間，發生了兩件大事，`xTaskResumeAll()` 必須一併清算：

1. **錯過的硬體時鐘（`xPendedTicks`）**：排程器鎖定時，硬體 Tick 中斷雖然會進來，但不敢增加系統時間（`xTickCount`），只能用一個小帳本記著「我漏掉了幾次 Tick」。
2. **被中斷喚醒卻動彈不得的任務（`xPendingReadyList`）**：某些中斷可能在期間釋放了 Semaphore 或 Queue，把一些任務給「喚醒」了。但因為排程器被鎖定，這些任務被塞進了「臨時收容所（Pending List）」。

#### 41.1 進入臨界區與巢狀解除計數

函數一開頭，核心立刻拉起最高規格的臨界區保護（`taskENTER_CRITICAL()`），來減少計數器並準備收容所清理。

```C
BaseType_t xTaskResumeAll( void )
{
    TCB_t * pxTCB = NULL;
    BaseType_t xAlreadyYielded = pdFALSE;

    traceENTER_xTaskResumeAll();

    /* 多核心環境下，如果排程器根本還沒開跑，直接跳過 */
    #if ( configNUMBER_OF_CORES > 1 )
        if( xSchedulerRunning != pdFALSE )
    #endif
    {
        /* 🚨【拉起最高防線】：
         * 由於接下來要瘋狂操作全域的排程鏈結串列，在此必須進入臨界區（關閉中斷/拿 Spinlock）。 */
        taskENTER_CRITICAL();
        {
            const BaseType_t xCoreID = ( BaseType_t ) portGET_CORE_ID();

            /* 【防禦性斷言】：如果目前的計數器本來就是 0，代表你根本沒有呼叫 vTaskSuspendAll() 
             * 就亂呼叫 Resume，核心會在此直接死機警告工程師。 */
            configASSERT( uxSchedulerSuspended != 0U );

            /* 巢狀解除：將暫停計數器減 1 */
            uxSchedulerSuspended = ( UBaseType_t ) ( uxSchedulerSuspended - 1U );
            
            /* 釋放多核心下的 Task 級自旋鎖 */
            portRELEASE_TASK_LOCK( xCoreID );

            /* 🚨【大門正式開啟】：只有當計數器扣到 0，代表所有的巢狀鎖都解開了，才執行實質善後 */
            if( uxSchedulerSuspended == ( UBaseType_t ) 0U )
            {
                if( uxCurrentNumberOfTasks > ( UBaseType_t ) 0U )
                {
```

#### 41.2 清空臨時收容所：將 Pending 任務搬回戰場

此區塊用一個 `while` 迴圈，將排程器鎖定期間被中斷私自喚醒、暫時關在 `xPendingReadyList` 的任務通通釋放，移回常規的就緒列表。

這裡有個重要 [issue](https://forums.freertos.org/t/should-the-microblaze-freertos-port-define-portmemory-barrier/16704/6)



```C
/* * 📦【清理臨時收容所 (xPendingReadyList)】
                     * 在排程器關閉期間，硬體中斷（如串口接收、定時器）如果調用了 xQueueSendFromISR 喚醒了某個任務，
                     * 為了防止打擾當時正在跑的任務，核心會把被喚醒的任務暫時丟進 `xPendingReadyList`。
                     * 現在排程器醒了，我們必須用一個迴圈把它清空。
                     */
                    while( listLIST_IS_EMPTY( &xPendingReadyList ) == pdFALSE )
                    {
                        /* 1. 拿到收容所鏈結串列的第一個任務 TCB */
                        pxTCB = listGET_OWNER_OF_HEAD_ENTRY( ( &xPendingReadyList ) );
                        
                        /* 2. 將它從事件等待列表（如 Queue 的等待列表）中拔除 */
                        listREMOVE_ITEM( &( pxTCB->xEventListItem ) );
                        portMEMORY_BARRIER();
                        
                        /* 3. 將它從狀態列表中拔除 */
                        listREMOVE_ITEM( &( pxTCB->xStateListItem ) );
                        
                        /* 4. 正式將其掛回 FreeRTOS 的常規就緒列表（Ready List）！ */
                        prvAddTaskToReadyList( pxTCB );

                        #if ( configNUMBER_OF_CORES == 1 )
                        {
                            /* * 🚨【單核心搶佔檢查】：
                             * 如果這個剛剛被我們救出來的任務，它的優先權比「目前正在跑的任務」還要高，
                             * 那代表等一下離開臨界區時，我們必須發動一次強行讓位（Yield）！
                             * 核心在此處將該核心的讓位標記 `xYieldPendings` 舉起為 pdTRUE。
                             */
                            if( pxTCB->uxPriority > pxCurrentTCB->uxPriority )
                            {
                                xYieldPendings[ xCoreID ] = pdTRUE;
                            }
                            else
                            {
                                mtCOVERAGE_TEST_MARKER();
                            }
                        }
                        #else /* #if ( configNUMBER_OF_CORES != 1 ) */
                        {
                            /* 多核心環境下，任務在塞進 PendingList 的那一瞬间，
                             * 其他核心就已經發動過核間中斷引發讓位了，因此這裡不需要重複記錄。 */
                        }
                        #endif /* #if ( configNUMBER_OF_CORES == 1 ) */
                    }

                    if( pxTCB != NULL )
                    {
                        /* 只要有任務被搬移，代表時間軸的解除阻塞時間可能變更，
                         * 呼叫此函數重新計算下一個醒來任務的時間（低功耗 Tickless 核心的關鍵）。 */
                        prvResetNextTaskUnblockTime();
                    }
```

#### 41.3 時光倒流大補帖：補償被遺忘的 Tick

在排程器鎖定期間，硬體時鐘每發生一次，`xPendedTicks` 就會加 1。此區塊要把那些因為排程器偷懶而少算的時間，用迴圈一次補回來！

```C
/* * ⏳【時光倒流大補帖：補回漏掉的時鐘中斷 (xPendedTicks)】
                     * 排程器鎖定時，硬體 Systick 中斷進來不敢呼叫完整的排程器更新，
                     * 只把漏掉的次數存在 `xPendedTicks` 變數裡。
                     * 現在排程器恢復了，我們要用一個 do-while 迴圈，把時間補回來。
                     */
                    {
                        TickType_t xPendedCounts = xPendedTicks; /* 建立非 volatile 的區域變數副本加速運算 */

                        if( xPendedCounts > ( TickType_t ) 0U )
                        {
                            do
                            {
                                /* * 🚨核心連環呼叫：
                                 * 沒跳一次 `xTaskIncrementTick()`，系統絕對時間 `xTickCount` 就會加 1。
                                 * 同時，這個函數會去檢查有沒有 delayed 的任務到期了。
                                 * 如果到期的任務需要引發搶佔（回傳 pdTRUE），就把當前核心的讓位標記設為 pdTRUE。
                                 */
                                if( xTaskIncrementTick() != pdFALSE )
                                {
                                    xYieldPendings[ xCoreID ] = pdTRUE;
                                }
                                else
                                {
                                    mtCOVERAGE_TEST_MARKER();
                                }

                                --xPendedCounts;
                            } while( xPendedCounts > ( TickType_t ) 0U );

                            /* 記帳本清零！時間全部補償完畢 */
                            xPendedTicks = 0;
                        }
                        else
                        {
                            mtCOVERAGE_TEST_MARKER();
                        }
                    }
```

#### 41.4 實施物理讓位：迎來新王上台

如果前面的善後程序中，有任何一個條件觸發了 `xYieldPendings`，核心將在此處發動最終的上下文切換（Context Switch）。

```C
/* 👑【新王登基：執行實質讓位】 */
                    if( xYieldPendings[ xCoreID ] != pdFALSE )
                    {
                        /* 如果啟用了搶佔機制，將傳回值設為 pdTRUE，告知外面：「我們換人跑了！」 */
                        #if ( configUSE_PREEMPTION != 0 )
                        {
                            xAlreadyYielded = pdTRUE;
                        }
                        #endif /* #if ( configUSE_PREEMPTION != 0 ) */

                        #if ( configNUMBER_OF_CORES == 1 )
                        {
                            /* 🚀【物理發動中斷】：
                             * 在單核心下，呼叫此巨集。它會直接去拉高硬體的 PendSV 軟體中斷。
                             * 這樣等一下執行到 `taskEXIT_CRITICAL()`、中斷大門一打開的瞬間，
                             * PendSV 中斷會立刻爆發，把目前這個舊任務踹下台，換剛剛甦醒的大哥任務上台！
                             */
                            taskYIELD_TASK_CORE_IF_USING_PREEMPTION( pxCurrentTCB );
                        }
                        #endif /* #if ( configNUMBER_OF_CORES == 1 ) */
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }
                }
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
        /* 解除臨界區保護，中斷重新開放 */
        taskEXIT_CRITICAL();
    }

    traceRETURN_xTaskResumeAll( xAlreadyYielded );

    /* 回傳是否真的發生了任務切換 */
    return xAlreadyYielded;
}
```

### 42. *xTaskGetTickCount*

**`xTaskGetTickCount()`** 的主要功能是**獲取系統自啟動以來的總 Tick 數（滴答計數）**，常用於延時、超時判斷或時間戳記錄。

#### 42.1 函數宣告與局部變數

```C
TickType_t xTaskGetTickCount( void )
{
    TickType_t xTicks;
```

- **`TickType_t`**：這是 FreeRTOS 定義的類型。如果是 32 位元系統，它通常是 `uint32_t`；如果是 8/16 位元微控制器且配置了 `configUSE_16_BIT_TICKS`，它就是 `uint16_t`。
- **`xTicks`**：定義一個局部變數，用來暫存讀取到的全局 Tick 值，確保線程安全後再回傳。

#### 42.2 臨界區保護（核心關鍵）

```C
/* Critical section required if running on a 16 bit processor. */
    portTICK_TYPE_ENTER_CRITICAL();
    {
        xTicks = xTickCount;
    }
    portTICK_TYPE_EXIT_CRITICAL();
```

> [!warning] **為什麼讀取一個變數需要進入「臨界區（Critical Section）」？** 這是這段程式碼最精妙的地方！註解寫著：_如果運行在 16 位元處理器上，需要臨界區_。

- **非原子性操作（Non-atomic Read）**：假設 `TickType_t` 是 **32 位元** 變數（`uint32_t`），但你的 MCU 是 **16 位元** 或 **8 位元**（例如 Arduino 的 ATmega328P 或某些 MSP430）。當 CPU 要讀取 `xTickCount` 時，它必須分**兩次指令**來讀取（先讀低 16 位元，再讀高 16 位元）。
- **潛在的 Bug（競態條件）**：假設當前 `xTickCount` 是 `0x0000FFFF`。CPU 剛讀完低 16 位元（`0xFFFF`），此時**突然發生時鐘中斷（Tick Interrupt）**，中斷服務程式將 `xTickCount` 加 1，變成了 `0x00010000`。中斷結束後，CPU 繼續讀取高 16 位元，讀到了 `0x0001`。最終組合出來的 `xTicks` 就變成了 `0x0001FFFF`！這比實際的時間整整快了 65536 個 Tick！

### 43. *xTaskGetTickCountFromISR*

這是 FreeRTOS 中專門在中斷服務程式（ISR, Interrupt Service Routine）中用來獲取系統 Tick 計數的 API —— **`xTaskGetTickCountFromISR()`**

與非 ISR 版本相比，它加入了**中斷優先級檢查**與**中斷安全（Interrupt-safe）的遮罩機制**。

```C
TickType_t xTaskGetTickCountFromISR( void )
{
    TickType_t xReturn;
    UBaseType_t uxSavedInterruptStatus;

    traceENTER_xTaskGetTickCountFromISR();

    /* RTOS ports that support interrupt nesting have the concept of a maximum
     * system call (or maximum API call) interrupt priority.  Interrupts that are
     * above the maximum system call priority are kept permanently enabled, even
     * when the RTOS kernel is in a critical section, but cannot make any calls to
     * FreeRTOS API functions.  If configASSERT() is defined in FreeRTOSConfig.h
     * then portASSERT_IF_INTERRUPT_PRIORITY_INVALID() will result in an assertion
     * failure if a FreeRTOS API function is called from an interrupt that has been
     * assigned a priority above the configured maximum system call priority.
     * Only FreeRTOS functions that end in FromISR can be called from interrupts
     * that have been assigned a priority at or (logically) below the maximum
     * system call  interrupt priority.  FreeRTOS maintains a separate interrupt
     * safe API to ensure interrupt entry is as fast and as simple as possible.
     * More information (albeit Cortex-M specific) is provided on the following
     * link: https://www.FreeRTOS.org/RTOS-Cortex-M3-M4.html */
    portASSERT_IF_INTERRUPT_PRIORITY_INVALID();

    uxSavedInterruptStatus = portTICK_TYPE_SET_INTERRUPT_MASK_FROM_ISR();
    {
        xReturn = xTickCount;
    }
    portTICK_TYPE_CLEAR_INTERRUPT_MASK_FROM_ISR( uxSavedInterruptStatus );

    traceRETURN_xTaskGetTickCountFromISR( xReturn );

    return xReturn;
}
```

> [!summary] **為什麼中斷中要用獨立的 FromISR 函數？**

1. **效率要求**：中斷處理必須極快，不能使用會讓任務進入阻塞（Blocked）或進行任務切換的常規臨界區巨集。
2. 1. **嵌套保護**：中斷中可能會有更高優先級的中斷嵌套發生，必須使用「中斷遮罩（Interrupt Mask）」而非單純的「關閉全局中斷」。

中斷安全臨界區（安全讀取）:
- **`portTICK_TYPE_SET_INTERRUPT_MASK_FROM_ISR()`**：這不會關閉所有的中斷。它只會**遮罩掉優先級等於或低於最高系統調用的中斷**。這樣做可以防止其他中斷（例如 Tick 定時器中斷）在此時插隊並修改 `xTickCount`。它會回傳修改前的中斷遮罩狀態。
- **`xReturn = xTickCount;`**：在受到遮罩保護的環境下，安全地將全域變數 `xTickCount` 複製到局部變數 `xReturn`，避免在 8/16 位元 MCU 上發生數據撕裂（Data Tearing）。
- **`portTICK_TYPE_CLEAR_INTERRUPT_MASK_FROM_ISR( uxSavedInterruptStatus )`**：將中斷遮罩恢復成進來之前的樣子，還原系統狀態。

### 44. *uxTaskGetNumberOfTasks*

這是在 FreeRTOS 中用來**獲取當前系統中總任務數量**的 API —— **`uxTaskGetNumberOfTasks()`**。

#### 44.1 為什麼這裡不需要臨界區？

```C
/* A critical section is not required because the variables are of type
     * BaseType_t. */
    traceRETURN_uxTaskGetNumberOfTasks( uxCurrentNumberOfTasks );

    return uxCurrentNumberOfTasks;
}
```

還記得前面的 `xTaskGetTickCount()` 需要進臨界區保護嗎？為什麼這裡讀取 `uxCurrentNumberOfTasks` 卻**完全不用**任何鎖（Lock）或臨界區（Critical Section）？
- **硬體天性**：`uxCurrentNumberOfTasks` 的類型是 `UBaseType_t`。FreeRTOS 在移植到不同 MCU 時，會保證 `BaseType_t` 的位元長度與該 CPU 的**資料總線寬度（Data Bus Width）**完全一致。
- **單週期讀取（Atomic Read）**：因為變數長度等於 CPU 寬度（例如在 32 位元 CPU 讀取 32 位元變數），CPU 只需要**一條機器指令**就能把資料從記憶體載入到暫存器中。
- **免除競態條件**：由於一條指令在執行時是絕對不可能被中斷（Interrupt）給打斷的，因此它天然具備「原子性」。既然不會發生讀到一半被改寫的數據撕裂（Data Tearing）問題，FreeRTOS 為了極致的效能，便直接省去了進入與退出臨界區的 CPU 開銷。

### 45. *pcTaskGetName*

取得 task 的名字

```C
char * pcTaskGetName( TaskHandle_t xTaskToQuery )

{

TCB_t * pxTCB;

  

traceENTER_pcTaskGetName( xTaskToQuery );

  

/* If null is passed in here then the name of the calling task is being

* queried. */

pxTCB = prvGetTCBFromHandle( xTaskToQuery );

configASSERT( pxTCB != NULL );

  

traceRETURN_pcTaskGetName( &( pxTCB->pcTaskName[ 0 ] ) );

  

return &( pxTCB->pcTaskName[ 0 ] );

}
```

### 46. *prvSearchForNameWithinSingleList*

這是在 FreeRTOS 內核中用於根據任務名稱字串，在單個鏈表（List）中搜尋對應任務控制塊（TCB）的私有函數 —— **`prvSearchForNameWithinSingleList()`**。

它是 `xTaskGetHandle()` 的底層核心實作。FreeRTOS 的任務管理本質上就是各種鏈表（如 Ready List、Blocked List、Suspended List），這個函數的工作就是**遍歷指定的鏈表，並逐字元比對任務名稱**。

註解中提到 `/* This function is called with the scheduler suspended. */`。這代表**此函數被調用時，調度器必須是掛起（Suspended）狀態**。因為遍歷鏈表非常耗時，若中斷或任務切換修改了鏈表，會導致指針崩潰。

### 47. *xTaskGetHandle*

這是在 FreeRTOS 中用來透過任務名稱字串（String）反查任務控制塊指標（TaskHandle_t）的公有 API —— **`xTaskGetHandle()`**。

它的底層邏輯非常清晰：利用你上一步看到的 `prvSearchForNameWithinSingleList()` 函數，**依照特定的優先權與狀態順序**，一一把 FreeRTOS 內部所有的任務鏈表（List）翻一遍，直到找到對應的名字為止。

### 48. *xTaskGetStaticBuffers*

這是在 FreeRTOS 支援靜態記憶體配置（Static Allocation）時提供的一個進階 API —— **`xTaskGetStaticBuffers()`**。

它的主要功能是：**獲取一個「經由靜態配置建立的任務」其底層所使用的堆疊（Stack）緩衝區與任務控制塊（TCB）的記憶體指標**。這在需要將任務狀態導出、進行特殊的記憶體檢查（Memory Auditing）、或是重置某些硬體保護區（MPU）時非常有用。

### 49. *uxTaskGetSystemState*

這是在 FreeRTOS 中用來獲取全系統所有任務當前狀態（快照）的重量級 API —— **`uxTaskGetSystemState()`**。

它的主要功能是遍歷系統中所有可能狀態的鏈表（Ready, Blocked, Suspended, Deleted），並將每個任務的詳細資訊（如優先權、任務狀態、剩餘堆疊、任務名稱等）填入用戶提供的結構體陣列中。這也是實現 `vTaskList()`（列出所有任務資訊）和 `vTaskGetRunTimeStats()`（CPU 使用率統計）的底層核心。

> [!info] **參數與配置說明**

- **條件編譯**：必須設定 `configUSE_TRACE_FACILITY == 1` 才可用。
    
- **`pxTaskStatusArray`**：指向一個 `TaskStatus_t` 結構體陣列的指標，用來接收數據。
    
- **`uxArraySize`**：傳入該陣列的大小（元素個數）。
    
- **`pulTotalRunTime`**：用來傳出「系統啟動以來的總運行時間計數」（用於計算 CPU 百分比）。
    
- **回傳值**：實際寫入陣列的任務數量。如果傳入的 `uxArraySize` 太小，則會返回 0。

#### 49.1 宣告、追蹤與掛起調度器

```C
UBaseType_t uxTask = 0, uxQueue = configMAX_PRIORITIES;

        traceENTER_uxTaskGetSystemState( pxTaskStatusArray, uxArraySize, pulTotalRunTime );

        vTaskSuspendAll();
        {
```

- **`uxTask`**：作為陣列的索引（Index），紀錄目前已經收集了多少個任務的資訊。
- **安全保護**：與 `xTaskGetHandle` 一樣，這裡大範圍掃描所有鏈表並拷貝大量數據，是非常耗時的操作。因此**使用 `vTaskSuspendAll()` 鎖定調度器**，避免任務切換破壞鏈表，同時保持硬體中斷正常響應。

#### 49.2 安全防禦：檢查陣列空間是否足夠

```C
/* Is there a space in the array for each task in the system? */
            if( uxArraySize >= uxCurrentNumberOfTasks )
            {
```

**核心防護**：傳入的陣列大小 `uxArraySize` **必須大於或等於**當前系統的總任務數 `uxCurrentNumberOfTasks`。如果空間不足以容納所有任務，內核為了防止記憶體越界寫入（Buffer Overflow），會直接跳過整個搜集邏輯（回傳 0）。

#### 49.3 第一步：搜集「就緒狀態（Ready）」的任務

```C
/* Fill in an TaskStatus_t structure with information on each
                 * task in the Ready state. */
                do
                {
                    uxQueue--;
                    uxTask = ( UBaseType_t ) ( uxTask + prvListTasksWithinSingleList( &( pxTaskStatusArray[ uxTask ] ), &( pxReadyTasksLists[ uxQueue ] ), eReady ) );
                } while( uxQueue > ( UBaseType_t ) tskIDLE_PRIORITY );
```

- **`prvListTasksWithinSingleList()`**：這是核心的內部私有函數。它負責遍歷指定的單個鏈表，把裡面的任務數據拷貝到 `&(pxTaskStatusArray[uxTask])` 開始的記憶體空間中，並**返回該鏈表中的任務總數**。
- **邏輯**：利用 `do-while`，從最高優先權到最低優先權，把所有 Ready 鏈表掃一遍，並累加 `uxTask` 索引。

#### 49.4 第二步：搜集「阻塞狀態（Blocked）」與「延時」的任務

```C
/* Fill in an TaskStatus_t structure with information on each
                 * task in the Blocked state. */
                uxTask = ( UBaseType_t ) ( uxTask + prvListTasksWithinSingleList( &( pxTaskStatusArray[ uxTask ] ), ( List_t * ) pxDelayedTaskList, eBlocked ) );
                uxTask = ( UBaseType_t ) ( uxTask + prvListTasksWithinSingleList( &( pxTaskStatusArray[ uxTask ] ), ( List_t * ) pxOverflowDelayedTaskList, eBlocked ) );
```

- 分別掃描標準延時鏈表與溢出延時鏈表，將搜集到的任務狀態標記為 `eBlocked`。

#### 49.5 第三步：條件搜集「等待刪除（Deleted）」與「掛起（Suspended）」任務

```C
#if ( INCLUDE_vTaskDelete == 1 )
                {
                    /* Fill in an TaskStatus_t structure with information on
                     * each task that has been deleted but not yet cleaned up. */
                    uxTask = ( UBaseType_t ) ( uxTask + prvListTasksWithinSingleList( &( pxTaskStatusArray[ uxTask ] ), &xTasksWaitingTermination, eDeleted ) );
                }
                #endif

                #if ( INCLUDE_vTaskSuspend == 1 )
                {
                    /* Fill in an TaskStatus_t structure with information on
                     * each task in the Suspended state. */
                    uxTask = ( UBaseType_t ) ( uxTask + prvListTasksWithinSingleList( &( pxTaskStatusArray[ uxTask ] ), &xSuspendedTaskList, eSuspended ) );
                }
                #endif
```

- 根據系統巨集配置，若有啟用相關功能，則繼續去掃描等待終止鏈表和掛起鏈表。

#### 49.6 第四步：獲取系統總運行時間統計值

```C
#if ( configGENERATE_RUN_TIME_STATS == 1 )
                {
                    if( pulTotalRunTime != NULL )
                    {
                        #ifdef portALT_GET_RUN_TIME_COUNTER_VALUE
                            portALT_GET_RUN_TIME_COUNTER_VALUE( ( *pulTotalRunTime ) );
                        #else
                            *pulTotalRunTime = ( configRUN_TIME_COUNTER_TYPE ) portGET_RUN_TIME_COUNTER_VALUE();
                        #endif
                    }
                }
                #else /* if ( configGENERATE_RUN_TIME_STATS == 1 ) */
                {
                    if( pulTotalRunTime != NULL )
                    {
                        *pulTotalRunTime = 0;
                    }
                }
                #endif /* if ( configGENERATE_RUN_TIME_STATS == 1 ) */
```

- **功能**：如果啟用了運行時間統計（`configGENERATE_RUN_TIME_STATS == 1`），且用戶有傳入用來接收時間的變數指標（`pulTotalRunTime != NULL`），內核會調用硬體計時器接口獲取當前總時間戳。
- **用途**：各任務 TCB 裡也記錄了各自被 CPU 執行的時間。拿各任務時間除以此處獲取的總時間 `*pulTotalRunTime`，就能精確算出每個任務的 **CPU 使用率百分比**。

#### 49.7 恢復調度器與返回

```C
}
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
        ( void ) xTaskResumeAll();

        traceRETURN_uxTaskGetSystemState( uxTask );

        return uxTask;
    }
```

- 恢復調度器，並回傳搜集到的任務計數 `uxTask`。

#### 49.8 補充

- **不用統計 `xPendingReadyTaskList` 的原因**：`xPendingReadyTaskList` 裡面的任務，其狀態指針（`xStateListItem`）此時**依然留在原來的 Blocked 鏈表中**。

### 50. *xTaskGetIdleTaskHandle*

這是在 FreeRTOS 中用來獲取系統空閒任務（Idle Task）控制代碼（TaskHandle_t）的公有 API —— **`xTaskGetIdleTaskHandle()`**。

這個函數特別限定在**單核心（Single Core）配置**下使用。空閒任務是 FreeRTOS 在啟動調度器（`vTaskStartScheduler()`）時自動建立的最低優先權任務（優先權為 0），當系統沒有其他任務需要執行時，CPU 就會一直運行 Idle Task。
```C
#if ( configNUMBER_OF_CORES == 1 )

TaskHandle_t xTaskGetIdleTaskHandle( void )

{

traceENTER_xTaskGetIdleTaskHandle();

  

/* If xTaskGetIdleTaskHandle() is called before the scheduler has been

* started, then xIdleTaskHandles will be NULL. */

configASSERT( ( xIdleTaskHandles[ 0 ] != NULL ) );

  

traceRETURN_xTaskGetIdleTaskHandle( xIdleTaskHandles[ 0 ] );

  

return xIdleTaskHandles[ 0 ];

}

#endif /* if ( configNUMBER_OF_CORES == 1 ) */
```
### 51. *xTaskGetIdleTaskHandleForCore*

跟 [[#50. xTaskGetIdleTaskHandle]]  一樣只是在 SMP 下
```C
TaskHandle_t xTaskGetIdleTaskHandleForCore( BaseType_t xCoreID )

{

traceENTER_xTaskGetIdleTaskHandleForCore( xCoreID );

  

/* Ensure the core ID is valid. */

configASSERT( taskVALID_CORE_ID( xCoreID ) == pdTRUE );

  

/* If xTaskGetIdleTaskHandle() is called before the scheduler has been

* started, then xIdleTaskHandles will be NULL. */

configASSERT( ( xIdleTaskHandles[ xCoreID ] != NULL ) );

  

traceRETURN_xTaskGetIdleTaskHandleForCore( xIdleTaskHandles[ xCoreID ] );

  

return xIdleTaskHandles[ xCoreID ];

}
```

### 52. vTaskStepTick

**什麼是 Tickless Idle？** 在標準 RTOS 中，硬體定時器每 1ms 會觸發一次中斷來遞增 `xTickCount`。這意味著即使系統完全沒事做，MCU 依然每 1ms 就會被叫醒一次，非常耗電。

開啟 `configUSE_TICKLESS_IDLE` 後，當內核發現未來 50ms 都沒有任務要執行，它就會**關閉這個每毫秒觸發的中斷**，並設定一個 50ms 後才觸發的單次硬體鬧鐘，然後讓 MCU 進入深度睡眠。當 50ms 過去或外部中斷叫醒 MCU 後，內核必須呼叫 `vTaskStepTick( 50 )` 把少算的 50 毫秒一次補回來。

#### 52.1 條件編譯說明

```C
/* This conditional compilation should use inequality to 0, not equality to 1.
 * This is to ensure vTaskStepTick() is available when user defined low power mode
 * implementations require configUSE_TICKLESS_IDLE to be set to a value other than
 * 1. */
#if ( configUSE_TICKLESS_IDLE != 0 )
```

**註解含意**：這裡刻意使用 `!= 0` 而不是 `== 1`。因為 FreeRTOS 允許使用者將 `configUSE_TICKLESS_IDLE` 設定為 `2`（自定義低功耗實現）。只要不是 `0`（關閉），這個快進時間的函數就必須存在。

#### 52.2 預估時間與防越界斷言

```C
void vTaskStepTick( TickType_t xTicksToJump )
    {
        TickType_t xUpdatedTickCount;

        traceENTER_vTaskStepTick( xTicksToJump );

        /* Correct the tick count value after a period during which the tick
         * was suppressed.  Note this does *not* call the tick hook function for
         * each stepped tick. */
        xUpdatedTickCount = xTickCount + xTicksToJump;
        configASSERT( xUpdatedTickCount <= xNextTaskUnblockTime );
```

- **`xTicksToJump`**：傳入參數，代表這次睡眠實際度過的時間（單位是 Tick）。
- **`xNextTaskUnblockTime`**：這是內核全域變數，紀錄著**全系統中，下一個最早準備解鎖（醒來）的任務時間點**。
- **`configASSERT` 安全檢查**：快進後的預估時間 `xUpdatedTickCount` **絕對不能超過**下一個任務要醒來的時間。因為如果超過了，代表系統睡過頭了（漏掉了任務的解鎖點），這在即時操作系統中是不允許的嚴重錯誤。

#### 52.3 精妙的邊界處理：剛好卡在任務解鎖點

```C
if( xUpdatedTickCount == xNextTaskUnblockTime )
        {
            /* Arrange for xTickCount to reach xNextTaskUnblockTime in
             * xTaskIncrementTick() when the scheduler resumes.  This ensures
             * that any delayed tasks are resumed at the correct time. */
            configASSERT( uxSchedulerSuspended != ( UBaseType_t ) 0U );
            configASSERT( xTicksToJump != ( TickType_t ) 0 );

            /* Prevent the tick interrupt modifying xPendedTicks simultaneously. */
            taskENTER_CRITICAL();
            {
                xPendedTicks++;
            }
            taskEXIT_CRITICAL();
            xTicksToJump--;
        }
```

>[!danger] **核心設計：為什麼快進時間要故意「少算 1 步」並增加 `xPendedTicks`？**

當快進後的理想時間**剛好等於**下一個任務的解鎖時間時，FreeRTOS 做了一個非常聰明的微調：
1. **`xPendedTicks++`**：這是一個暫存計數器，代表「被推遲、等待處理的 Tick 中斷」。
2. **`xTicksToJump--`**：故意讓這次要加的時間減 1。

**背後的邏輯**：
在 Tickless 模式下，當調度器恢復（Resume）時，內核會自動調用 `xTaskIncrementTick()`。這個函數的工作是把 `xPendedTicks` 消化掉，並且每消化一個，就會去檢查有沒有任務時間到了、需要被喚醒。

為了確保那個「時間剛好到了」的任務能被**標準的核心喚醒流程**（走 `xTaskIncrementTick` 內的 Ready 鏈表轉移邏輯）給正確拉起來，此處故意把最後 1 個 Tick 的補償權力交給 `xPendedTicks`，而不是直接粗暴地改寫 `xTickCount`。這避免了在解鎖邊界上發生狀態不一致的 Bug。

#### 52.4 正式快進時間與返回

```C
else
        {
            mtCOVERAGE_TEST_MARKER();
        }

        xTickCount += xTicksToJump;

        traceINCREASE_TICK_COUNT( xTicksToJump );
        traceRETURN_vTaskStepTick();
    }
```

- **`xTickCount += xTicksToJump;`**：正式將全域變數系統時間往前推。如果是走上一段 `if` 分支（剛好到期），這裡加的就是 `xTicksToJump - 1`；如果是提前被外部中斷叫醒（走 `else`），這裡就直接補上完整的睡眠時間。

### 53. *xTaskCatchUpTicks*

**`xTaskCatchUpTicks()`** 是 FreeRTOS 中用於手動補償或快進系統時間的進階 API

它的核心功能是：**將系統時間「快進」指定的 Tick 數（`xTicksToCatchUp`），並在快進的過程中，確保所有在期間該醒來的任務都能被正確、依序地喚醒。** 這與前文看到的 `vTaskStepTick()` 不同：`vTaskStepTick()` 適合用於完全沒有任務需要喚醒的低功耗安全跳躍；而 `xTaskCatchUpTicks()` 則是透過**模擬推遲中斷**的方式，讓內核在恢復調度器時，老老實實地把漏掉的 Tick 流程全部「補課」執行一遍。

> [!info] **基本資訊**

- **功能定義**：手動讓內核補足 `xTicksToCatchUp` 個 Tick 的時間。
- **返回值**：`BaseType_t`。如果補課期間有任何一個被喚醒的任務其優先權高於當前運行的任務，導致了任務切換（Context Switch），則返回 `pdTRUE`（即 1）；否則返回 `pdFALSE`（即 0）。

#### 53.1 函數宣告與前置斷言（關鍵限制）

```C
BaseType_t xTaskCatchUpTicks( TickType_t xTicksToCatchUp )
{
    BaseType_t xYieldOccurred;

    traceENTER_xTaskCatchUpTicks( xTicksToCatchUp );

    /* Must not be called with the scheduler suspended as the implementation
     * relies on xPendedTicks being wound down to 0 in xTaskResumeAll(). */
    configASSERT( uxSchedulerSuspended == ( UBaseType_t ) 0U );
```

- **`configASSERT( uxSchedulerSuspended == 0 )`**：**絕對不能在調度器已經被掛起的情況下調用此函數！** * **原因**：此函數的設計極度依賴接下來的 `xTaskResumeAll()` 來消化並清空 `xPendedTicks`。如果你在調用前就已經手動掛起了調度器，那麼後續內核內部的 `xTaskResumeAll()` 就不會真正執行解鎖與清空暫存中斷的邏輯，這會導致時間補償完全失效。

#### 53.2 掛起調度器與操作暫存計數器

```C
/* Use xPendedTicks to mimic xTicksToCatchUp number of ticks occurring when
     * the scheduler is suspended so the ticks are executed in xTaskResumeAll(). */
    vTaskSuspendAll();

    /* Prevent the tick interrupt modifying xPendedTicks simultaneously. */
    taskENTER_CRITICAL();
    {
        xPendedTicks += xTicksToCatchUp;
    }
    taskEXIT_CRITICAL();
```

- **`vTaskSuspendAll()`**：手動鎖定調度器，開啟一個安全的緩衝區。
- **`xPendedTicks += xTicksToCatchUp;`**：這是最核心的魔術！正如你上一題所了解的，`xPendedTicks` 代表「被推遲、等待處理的 Tick 數量」。這裡直接用加法，**假裝在調度器鎖定的這段時間內，硬體連續發生了 `xTicksToCatchUp` 次時鐘中斷**。
- **`taskENTER_CRITICAL()`**：因為真實的硬體時鐘中斷（Tick ISR）隨時可能發生並修改 `xPendedTicks`，所以必須在常規臨界區內進行累加，防止數據撕裂與競態條件。

#### 53.3 接棒補課與返回狀態

```C
xYieldOccurred = xTaskResumeAll();

    traceRETURN_xTaskCatchUpTicks( xYieldOccurred );

    return xYieldOccurred;
}
```

- **`xTaskResumeAll()`**：這就是讓內核開始「補課」的發動機。當調度器恢復時，它看見 `xPendedTicks` 裡面突然累積了這麼多 Tick，就會啟動一個 `while` 迴圈，連續調用 `xTicksToCatchUp` 次 `xTaskIncrementTick()`。
- **依序喚醒**：每執行一次，`xTickCount` 就加 1，並且內核會精準檢查該時間點是否有任務需要解鎖。這樣可以**完美保證即使一次跳躍很多個 Tick，期間所有到期的任務都會按照原本的先後順序被一一喚醒**，不會漏掉任何一個。
- **`xYieldOccurred`**：記錄補課結束後是否需要切換任務，並將其返回。

#### 53.4 跟 vTaskStepTick 的比較

| **特性**   | **vTaskStepTick()**           | **xTaskCatchUpTicks()**              |
| -------- | ----------------------------- | ------------------------------------ |
| **底層機制** | 直接改寫 `xTickCount` 變數（數學快進）    | 累加 `xPendedTicks`（模擬中斷補課）            |
| **安全前提** | **必須確保** 跳躍期間 **沒有任何任務** 需要解鎖 | 跳躍期間 **可以有任務需要解鎖**，內核會依序喚醒           |
| **執行效率** | 極快（$O(1)$ 常數時間）               | 較慢（$O(N)$，需循環呼叫 `xTicksToCatchUp` 次） |
| **應用場景** | 晶片底層低功耗模式（確認大家都睡死時）           | 第三方應用（如仿真器同步、應用層時間校準）                |

### 54. *xTaskAbortDelay*

**`xTaskAbortDelay()`** 是在 FreeRTOS 中用來強行中止某個任務的延時或阻塞狀態，使其立刻醒來（轉為就緒態）的進階公有 API

它的核心功能是：不管目標任務是在調用 `vTaskDelay()` 睡覺，還是在等待隊列（Queue）、信號量（Semaphore）等事件而處於 Blocked 狀態，只要對它呼叫此函數，內核就會**強行把它從等待鏈表中拉出來，丟回 Ready List**。這在需要處理「緊急撤銷」、「超時重置」或「異常喚醒」的場景中非常關鍵。

> [!info] **基本資訊**

- **條件編譯**：必須設定 `INCLUDE_xTaskAbortDelay == 1` 才可用。
- **參數與返回值**：傳入目標任務的 `TaskHandle_t`。如果目標任務確實處於 Blocked 狀態且成功被喚醒，返回 `pdPASS`（1）；如果任務根本沒在阻塞（例如已經是 Ready 或 Suspended），則不做任何事，返回 `pdFAIL`（0）。

#### 54.1 安全防禦與掛起調度器

```C
TCB_t * pxTCB = xTask;
        BaseType_t xReturn;

        traceENTER_xTaskAbortDelay( xTask );

        configASSERT( pxTCB != NULL );

        vTaskSuspendAll();
        {
```

- **類型轉換**：`TaskHandle_t` 在內核中本質上就是 `TCB_t *`。
- **調度器鎖定**：因為要對核心鏈表（Blocked List、Event List）進行修改，為了防止搬移到一半發生任務切換導致指針崩潰，調用 `vTaskSuspendAll()` 鎖定調度器。

#### 54.2 狀態檢查與移除狀態鏈表（State List）

```C
/* A task can only be prematurely removed from the Blocked state if
             * it is actually in the Blocked state. */
            if( eTaskGetState( xTask ) == eBlocked )
            {
                xReturn = pdPASS;

                /* Remove the reference to the task from the blocked list.  An
                 * interrupt won't touch the xStateListItem because the
                 * scheduler is suspended. */
                ( void ) uxListRemove( &( pxTCB->xStateListItem ) );
```

- **核心前提**：只有當目標任務的狀態確實是 `eBlocked` 時，中止才有意義。
- **第一步搬移**：呼叫 `uxListRemove`，把該任務從原先掛著的延時鏈表（如 `pxDelayedTaskList`）中拔除。
- **安全註解含意**：註解提到中斷（ISR）絕對不會去碰觸任務的 `xStateListItem`（因為調度器鎖定期間中斷不允許引發任務狀態轉移），所以在這一步不需要關閉中斷。

#### 54.3 雙重阻斷解除：事件鏈表（Event List）與核心標記

```C
/* Is the task waiting on an event also?  If so remove it from
                 * the event list too.  Interrupts can touch the event list item,
                 * even though the scheduler is suspended, so a critical section
                 * is used. */
                taskENTER_CRITICAL();
                {
                    if( listLIST_ITEM_CONTAINER( &( pxTCB->xEventListItem ) ) != NULL )
                    {
                        ( void ) uxListRemove( &( pxTCB->xEventListItem ) );

                        /* This lets the task know it was forcibly removed from the
                         * blocked state so it should not re-evaluate its block time and
                         * then block again. */
                        pxTCB->ucDelayAborted = ( uint8_t ) pdTRUE;
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }
                }
                taskEXIT_CRITICAL();
```

- **處理事件鏈表**：如果這個任務是因為等待 Queue 或 Semaphore 而阻塞，它的 `xEventListItem` 就會掛在該 Queue 的等待鏈表裡。必須確認並用 `uxListRemove` 把它從事件鏈表中也拔除，否則該 Queue 釋放時會引發指針異常。
- **臨界區保護**：**這裡為什麼要進 `taskENTER_CRITICAL()` 關中斷？** 因為即使調度器掛起了，硬體中斷（ISR）依然在運作。中斷裡可能會呼叫 `xQueueSendFromISR` 等 API 釋放信號，這會去動到這個 Queue 的事件鏈表。為了防止和中斷搶奪同一個事件鏈表，必須加鎖保護。
- **關鍵標記 `ucDelayAborted = pdTRUE`**：這是靈魂步驟。當這個被強行喚醒的任務下一次拿到 CPU 時，它會去檢查這個標記。它會發現：「噢！原來我不是因為時間到（Timeout）醒來的，也不是因為真的等到事件醒來的，我是被別人強行拍醒的！」這時任務就不會去走常規的超時處理邏輯。

#### 54.4 丟回就緒鏈表（Ready List）

```C
/* Place the unblocked task into the appropriate ready list. */
                prvAddTaskToReadyList( pxTCB );
```

- 兩邊鏈表都拔乾淨後，呼叫 `prvAddTaskToReadyList` 正式把任務塞回對應優先權的就緒鏈表中。此時，任務正式具備了重新競爭 CPU 的資格。

#### 54.5 任務切換判定（單核心與多核心分流）

```C
/* A task being unblocked cannot cause an immediate context
                 * switch if preemption is turned off. */
                #if ( configUSE_PREEMPTION == 1 )
                {
                    #if ( configNUMBER_OF_CORES == 1 )
                    {
                        /* Preemption is on, but a context switch should only be
                         * performed if the unblocked task has a priority that is
                         * higher than the currently executing task. */
                        if( pxTCB->uxPriority > pxCurrentTCB->uxPriority )
                        {
                            /* Pend the yield to be performed when the scheduler
                             * is unsuspended. */
                            xYieldPendings[ 0 ] = pdTRUE;
                        }
                        else
                        {
                            mtCOVERAGE_TEST_MARKER();
                        }
                    }
                    #else /* #if ( configNUMBER_OF_CORES == 1 ) */
                    {
                        taskENTER_CRITICAL();
                        {
                            prvYieldForTask( pxTCB );
                        }
                        taskEXIT_CRITICAL();
                    }
                    #endif /* #if ( configNUMBER_OF_CORES == 1 ) */
                }
                #endif /* #if ( configUSE_PREEMPTION == 1 ) */
```

- **搶佔啟用檢查（`configUSE_PREEMPTION == 1`）**：只有在允許搶佔時，叫醒新任務才需要考慮要不要立刻切換。
- **單核心分支（`configNUMBER_OF_CORES == 1`）**：如果被拍醒的任務其優先權（`pxTCB->uxPriority`）比目前正在執行的任務（`pxCurrentTCB->uxPriority`）還要高，那應該要立刻切換過去。但因為現在調度器還鎖著，所以先將 `xYieldPendings[0]` 設為 `pdTRUE`（做個記號），等一下解鎖時內核會自動補做任務切換。
- **多核心分支（SMP 環境）**：在多核心系統中，需要呼叫內核內部的 `prvYieldForTask( pxTCB )`。這會去評估哪一個核心此時正在執行優先權最低的任務，並向該核心發送核間中斷（IPI），強行讓該核心讓出位置給這個剛剛醒來的任務。

#### 54.6 解鎖調度器與返回

```C
}
            else
            {
                xReturn = pdFAIL;
            }
        }
        ( void ) xTaskResumeAll();

        traceRETURN_xTaskAbortDelay( xReturn );

        return xReturn;
    }
```

- 解開調度器鎖定，若前面有做 `xYieldPendings` 記號，會在 `xTaskResumeAll()` 內順便觸發切換，最後將狀態返回。

#### 54.7 補充

> [!tip] **應用層如何得知自己被「強行拍醒」？**

當一個任務呼叫 `xQueueReceive( xQueue, &data, portMAX_DELAY )` 陷入無限等待時：
- 正常情況下：拿到資料返回 `pdPASS`。
- 如果被別的任務呼叫 `xTaskAbortDelay()` 強行中斷： 這個 API 會導致 `xQueueReceive()` 內部**立刻結束阻塞並返回 `pdFALSE`**。
- 開發者可以透過返回值判斷，並藉由調用 `xTaskGetSystemState` 或自定義全局旗標來確認這是一次異常中止，進而安全地引導任務進入異常復原程序。

### 55. *xTaskIncrementTick*

> [!info] **函數定位與核心功能**

每次硬體定時器（Tick Timer）觸發中斷時，底層架構就會呼叫此函數。它是 FreeRTOS 系統時間流逝的「推進器」，負責**更新系統時間**、**喚醒到期的阻塞任務**、並決定是否要**發起任務切換（Context Switch）**。

#### 55.1 正常分支出：系統時間遞增與溢出保護

```C
if( uxSchedulerSuspended == ( UBaseType_t ) 0U )
    {
        /* Minor optimisation.  The tick count cannot change in this
         * block. */
        const TickType_t xConstTickCount = xTickCount + ( TickType_t ) 1;

        /* Increment the RTOS tick, switching the delayed and overflowed
         * delayed lists if it wraps to 0. */
        xTickCount = xConstTickCount;

        if( xConstTickCount == ( TickType_t ) 0U )
        {
            taskSWITCH_DELAYED_LISTS();
        }
```

- **調度器檢查**：只有在調度器正常運作（未鎖定）時，才能進入這個主要邏輯分支。
- **溢出交換（Overflow Swap）**：系統時間 `xTickCount` 會不斷加 1。當計數器滿了歸零（溢出）時，內核會呼叫 `taskSWITCH_DELAYED_LISTS()`，將「當前延時鏈表」與「溢出延時鏈表」的指針進行交換。這是 FreeRTOS 解決時間翻轉問題最優雅的 $O(1)$ 實作。

#### 55.2 高效喚醒機制：檢查並釋放超時任務

```C
if( xConstTickCount >= xNextTaskUnblockTime )
        {
            for( ; ; )
            {
                if( listLIST_IS_EMPTY( pxDelayedTaskList ) != pdFALSE )
                {
                    xNextTaskUnblockTime = portMAX_DELAY;
                    break;
                }
                else
                {
                    pxTCB = listGET_OWNER_OF_HEAD_ENTRY( pxDelayedTaskList );
                    xItemValue = listGET_LIST_ITEM_VALUE( &( pxTCB->xStateListItem ) );

                    if( xConstTickCount < xItemValue )
                    {
                        xNextTaskUnblockTime = xItemValue;
                        break;
                    }
                    /* ... 拔除鏈表並放入就緒態的邏輯 ... */
```

- **`xNextTaskUnblockTime`**：這是一個全域變數，紀錄了「下一個最早該醒來的任務時間」。如果系統時間還沒到這個點，這整段耗時的檢查代碼會被直接跳過（極大化 ISR 效能）。
- **鏈表排序特性**：因為 `pxDelayedTaskList` 永遠是**按超時時間從小到大排序**，內核只需檢查頭部節點（Head Entry）。如果頭部的任務都還沒超時，後面的任務也必定沒超時，直接 `break` 退出迴圈。
- `if( xConstTickCount < xItemValue )` 的作用是什麼？
	- 這段程式碼的核心作用是：**「設定下一次進來補課/檢查的鬧鐘，然後立刻退出，不再浪費 CPU 時間。」**

```C
if( xConstTickCount < xItemValue )
{
    /* 電腦時間（now）還沒到這個任務想醒來的時間（target） */
    xNextTaskUnblockTime = xItemValue; // 把下一次全系統最早醒來的時間，更新為這個任務的時間
    break;                             // 退出 for(;;) 迴圈
}
```

#### 55.3 解除阻塞與判定搶佔切換

```C
/* It is time to remove the item from the Blocked state. */
                    listREMOVE_ITEM( &( pxTCB->xStateListItem ) );

                    /* Is the task waiting on an event also?  If so remove
                     * it from the event list. */
                    if( listLIST_ITEM_CONTAINER( &( pxTCB->xEventListItem ) ) != NULL )
                    {
                        listREMOVE_ITEM( &( pxTCB->xEventListItem ) );
                    }

                    /* Place the unblocked task into the appropriate ready list. */
                    prvAddTaskToReadyList( pxTCB );

                    #if ( configUSE_PREEMPTION == 1 )
                    {
                        #if ( configNUMBER_OF_CORES == 1 )
                        {
                            if( pxTCB->uxPriority > pxCurrentTCB->uxPriority )
                            {
                                xSwitchRequired = pdTRUE;
                            }
                        /* ... 多核心分支省略 ... */
```

- **雙重拔除**：當任務超時，它不僅要從延遲狀態鏈表中移除，如果它同時還在等待 Semaphore 或 Queue，也要一併從事件鏈表（Event List）中強制拔除。
- **搶佔標記**：剛被喚醒的任務被塞回 Ready 鏈表後，會立即與當前正在執行的任務（`pxCurrentTCB`）比對優先權。如果更高，則標記 `xSwitchRequired = pdTRUE`。

#### 55.4 同優先權任務的時間片輪轉（Time Slicing）

**時間片邏輯**：如果啟用了 Time Slicing，內核會檢查「當前優先權的 Ready 鏈表」中有沒有超過 1 個任務在排隊（因為正在跑的 task 也包含在 `pxReadyTasksLists`，不會因為正在跑所以被移除，因此要判斷長度 > 1）。如果有，也設定 `xSwitchRequired = pdTRUE`，強迫當前任務交出 CPU，讓下一個同級任務執行。這就是 RTOS 實現「輪流執行」的核心所在。

```C
/* Tasks of equal priority to the currently running task will share
         * processing time (time slice) if preemption is on... */
        #if ( ( configUSE_PREEMPTION == 1 ) && ( configUSE_TIME_SLICING == 1 ) )
        {
            #if ( configNUMBER_OF_CORES == 1 )
            {
                if( listCURRENT_LIST_LENGTH( &( pxReadyTasksLists[ pxCurrentTCB->uxPriority ] ) ) > 1U )
                {
                    xSwitchRequired = pdTRUE;
                }
            /* ... 多核心分支省略 ... */
```

#### 55.5 調度器鎖定時的「時間推遲」分支

```C
}
    else
    {
        xPendedTicks += 1U;

        /* The tick hook gets called at regular intervals, even if the
         * scheduler is locked. */
        #if ( configUSE_TICK_HOOK == 1 )
        {
            vApplicationTickHook();
        }
        #endif
    }

    traceRETURN_xTaskIncrementTick( xSwitchRequired );

    return xSwitchRequired;
}
```

- **掛起保護**：如果此時有任務呼叫了 `vTaskSuspendAll()` 把調度器鎖死了，中斷依然會發生並呼叫此函數，但會進入這個 `else` 分支。
- **`xPendedTicks` 記帳**：此時不允許更動任何任務鏈表，因此只把被耽誤的心跳次數記錄在 `xPendedTicks` 中，等待日後調度器解鎖時，再進行「快進補課」。


### 56. *vTaskSetApplicationTaskTag*

`*vTaskSetApplicationTaskTag*` 是 FreeRTOS 提供的一個進階功能：**為任務綁定「專屬鉤子函數（Task Hook）」**。

雖然函數名字叫 `SetApplicationTaskTag`（設定應用程式任務標籤），但它在 FreeRTOS 裡的本質**不是**用來貼簡單的文字標籤，而是**讓任務指針指向一個你自定義的函數**（`TaskHookFunction_t`）。這個功能最常被用來做**進階效能分析（Profiling）**、**動態追蹤（Tracing）**，或者在**任務切換（Context Switch）時觸發特定的硬體動作**（例如切換 GPIO 引腳）。

> [!info] **核心參數含意**

- `xTask`：目標任務的控制句柄（Handle）。如果傳入 `NULL`，代表要對「目前正在運行的任務」進行設定
- `pxHookFunction`：你自定義的函數指針。如果傳入 `NULL`，則代表取消（清空）該任務的鉤子函數

#### 56.1 功能開關

```C
#if ( configUSE_APPLICATION_TASK_TAG == 1 )
// ... 函數實作 ...
#endif /* configUSE_APPLICATION_TASK_TAG */
```

- **裁剪內核**：FreeRTOS 為了節省記憶體，預設不會開啟這個功能。你必須在 `FreeRTOSConfig.h` 中手動設定 `#define configUSE_APPLICATION_TASK_TAG 1`
- **記憶體代價**：開啟此功能後，系統中的**每一個**任務控制塊（TCB）都會多佔用一個指針的記憶體空間（例如在 32 位元架構上會多出 4 萬能位元組），用來儲存這個標籤

#### 56.2 臨界區保護與賦值

```C
/* Save the hook function in the TCB.  A critical section is required as
         * the value can be accessed from an interrupt. */
        taskENTER_CRITICAL();
        {
            xTCB->pxTaskTag = pxHookFunction;
        }
        taskEXIT_CRITICAL();
```

- **為什麼要進入臨界區（Critical Section）？** 這行程式碼只是簡單的指針賦值（`xTCB->pxTaskTag = pxHookFunction`），為什麼需要大費周章地關閉中斷？ 因為這個 `pxTaskTag` 之後非常有可能會**在中斷服務程式（ISR）中被讀取**（最典型的場景是在硬體時鐘中斷裡，利用 `traceTASK_SWITCHED_IN` 巨集來自動執行這個 Hook）。為了防止賦值到一半被中斷打斷導致讀到髒資料（Data Corruption），必須用 `taskENTER_CRITICAL()` 嚴密保護。

#### 56.3 這個函數可以用在哪？

光看這個設定函數可能覺得有點抽象，因為它只是把函數指標存進 TCB 裡面。真正威力爆發的地方，在於如何**調用**它。

實戰場景：用邏輯分析儀精準測量「特定任務」的執行時間

假設你有一個優先權極高的緊急任務，你想知道它每次被喚醒到執行完畢，具體佔用了 CPU 多少微秒。你可以這樣做：

1. 在你的應用層寫好 Hook 功能

```C
/* 自定義的 Hook 函數：當任務切換進來時，拉高 GPIO 引腳 */
BaseType_t xMyHookFunction( void * pvParameter )
{
    // 這裡可以使用低階驅動直接拉高某個測試引腳 (例如 GPIO_PIN_1)
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET);
    return 0;
}

/* 自定義的 Hook 函數：當任務切換出去時，拉低 GPIO 引腳 */
BaseType_t xMyHookFunctionOut( void * pvParameter )
{
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET);
    return 0;
}
```

2. 綁定此 Hook 到你的任務中

```C
void vEmergencyTask( void * pvParameters )
{
    /* 在任務開頭，把我們剛剛寫的 Hook 函數綁定給自己 */
    vTaskSetApplicationTaskTag( NULL, xMyHookFunction );
    
    for( ;; )
    {
        // 執行緊急任務邏輯...
    }
}
```

3. 配合 FreeRTOS 的 Trace 巨集自動觸發

在 `FreeRTOSConfig.h` 中，你可以利用內核在任務切換時必定會踩到的「埋點」來自動調用它：

```C
/* 當任務被切換進來（開始吃 CPU）時觸發 */
#define traceTASK_SWITCHED_IN()                                                 \
    {                                                                           \
        /* 檢查當前任務有沒有綁定 Hook */                                        \
        if( xTaskGetApplicationTaskTag( pxCurrentTCB ) != NULL )                \
        {                                                                       \
            /* 如果有，立刻執行該 Hook（拉高 GPIO） */                             \
            xTaskCallApplicationTaskHook( pxCurrentTCB, NULL );                 \
        }                                                                       \
    }
```

### 57. *xTaskGetApplicationTaskTag*

和 [[#56. vTaskSetApplicationTaskTag]] 類似只是是 Get

### 58. *xTaskGetApplicationTaskTagFromISR*

上一個函數是負責「寫入」（Setter），而這個 `xTaskGetApplicationTaskTagFromISR()` 則是專門在中斷服務程式（ISR）中負責「讀取」（Getter）的

最經典的搭配場景，就是你在 `traceTASK_SWITCHED_IN()` 中斷埋點裡，呼叫這個函數來抓取目前任務的 Hook 並執行它。

#### 58.1 中斷級臨界區保護（ISR-Safe Critical Section）

```C
uxSavedInterruptStatus = taskENTER_CRITICAL_FROM_ISR();
        {
            xReturn = pxTCB->pxTaskTag;
        }
        taskEXIT_CRITICAL_FROM_ISR( uxSavedInterruptStatus );
```

這段是全函數技術含量最高的地方，剛好呼叫了你上一篇問的臨界區問題：
- **為什麼不用 `taskENTER_CRITICAL()`？** 因為這是專門在 **ISR（中斷服務程式）** 中執行的 API。標準的 `taskENTER_CRITICAL` 會硬性去動到任務層級的嵌套計數器，這在中斷裡會導致核心崩潰。中斷裡必須使用帶有 `_FROM_ISR` 後綴的專用版本。
- **`uxSavedInterruptStatus` 的作用是什麼？** 中斷是有優先權（Priority）的。當前這個中斷在執行時，可能已經屏蔽了低優先權中斷。 `taskENTER_CRITICAL_FROM_ISR()` 會做兩件事：
	1. 讀取並**記錄**當前硬體的中斷屏蔽狀態（存入 `uxSavedInterruptStatus`）。
	2. 拔高屏蔽級別，把其他可能來攪局的高優先權中斷也暫時擋住，確保大門關好。
- **還原現場**：
	- 讀取完 `pxTaskTag` 後，呼叫 `taskEXIT_CRITICAL_FROM_ISR(uxSavedInterruptStatus)`。它會把剛才記錄的狀態還給 CPU。晶片該是什麼中斷級別就回到什麼級別，非常優雅且安全。

>[!tip] 為什麼讀取（Getter）也要加臨界區？

你可能會想：「上一篇寫入（Setter）怕被拆解成多條指令所以要加 CS 擋中斷，那我現在在中斷裡單純『讀取』，難道也會被打斷嗎？」 **答案是：會的！** 因為中斷是可以**嵌套（Nested Interrupt）**的。如果你的晶片支援中斷嵌套（例如 ARM Cortex-M 的高優先權中斷可以打斷低優先權中斷），當前這個低優先權 ISR 在讀取指針讀到一半時，如果被另一個更高優先權的 ISR 打斷，且那個高優先權 ISR 剛好去修改了同一個 Tag，就會發生資料毀損。 **因此，`FROM_ISR` 的臨界區是為了解決「中斷被更高階中斷搶佔」的並行問題。**

### 59. *xTaskCallApplicationTaskHook*

> [!info] **核心參數與回傳值含意**

- `xTask`：你想觸發哪一個任務的 Hook？傳入 `NULL` 代表執行「當前運行中任務」的 Hook。
- `pvParameter`：**極其強大的萬能指針（`void *`）**。你可以傳遞任何參數、結構體給該 Hook 函數，甚至傳入 `NULL`。
- **回傳值**：如果該任務沒綁定 Hook，硬體會回傳 `pdFAIL`（0）；如果有綁定，則會回傳**你那個自定義 Hook 函數自己回傳的值**（這讓 Hook 函數有能力反向控制內核邏輯）。

#### 59.1 為什麼需要 `pvParameter` 與回傳值？

> [!question] **官方工程師在註解中沒說的事**

為什麼這個函數需要回傳值 BaseType_t？因為它能用來做「中斷決策」。

核心場景：在中斷中利用 Task Hook 決定是否進行任務切換（Yield）

假設你寫了一個自定義 Hook，你想在中斷裡執行它，並且根據 Hook 執行的結果，決定要不要立刻切換任務（Context Switch）：

> [!tip] **為什麼 `xTaskCallApplicationTaskHook` 內部沒有 Critical Section（臨界區）？**

你可能會注意到，Set 和 Get 都有加臨界區保護，但這個 Call（執行）卻沒有加。 **原因：** 因為這個函數是要去**執行一整段你寫的程式碼**。如果內核自作聰明在執行前幫你加上 `taskENTER_CRITICAL()` 關閉中斷，而你寫的 Hook 函數裡面又剛好很長，那全系統的中斷就會被關死太久而當機。FreeRTOS 選擇把「要不要加鎖保護」的權力，百分之百交還給實現該 Hook 的開發者自己決定。

### 60. *vTaskSwitchContext*

這段程式碼是整個 FreeRTOS 內核的**靈魂心臟**：`vTaskSwitchContext`（任務上下文切換）

不論是時間片到了（Tick 中斷）、還是高優先權任務突然醒來，底層硬體（例如 ARM 的 PendSV 中斷）最終都必定會呼叫這個函數。它的唯一核心任務，就是「把舊任務踢下來，換上目前全系統優先權最高的就緒任務」。

```C
/* * 根據 configNUMBER_OF_CORES 的設定，
 * 編譯器會自動選擇編譯【單核心版】或【多核心 SMP 版】。
 */
#if ( configNUMBER_OF_CORES == 1 )
    /* 單核心版本函數實作... */
    void vTaskSwitchContext( void ) { ... }
#else
    /* 多核心 SMP 版本函數實作... */
    void vTaskSwitchContext( BaseType_t xCoreID ) { ... }
#endif
```
#### 60.1 Single Core

##### 60.1.1 調度器掛起保護（防禦臨界區）

```C
if( uxSchedulerSuspended != ( UBaseType_t ) 0U )
        {
            /* 調度器目前被鎖定（掛起）- 絕對不允許切換任務 */
            xYieldPendings[ 0 ] = pdTRUE;
        }
        else
        {
            xYieldPendings[ 0 ] = pdFALSE;
            traceTASK_SWITCHED_OUT();
```

- **運作邏輯**：如果開發者呼叫了 `vTaskSuspendAll()` 鎖定了調度器，此時 `uxSchedulerSuspended` 不為 0。內核**絕對不能**換人跑。
- **補課機制**：內核會把 `xYieldPendings[ 0 ]` 設為 `pdTRUE`（記上一筆賒帳），等開發者呼叫 `xTaskResumeAll()` 解鎖調度器時，系統一看到這記號，就會立刻補做任務切換。

##### 60.1.2 CPU 運行時間統計（CPU Profiling）

```C
#if ( configGENERATE_RUN_TIME_STATS == 1 )
            {
                #ifdef portALT_GET_RUN_TIME_COUNTER_VALUE
                    portALT_GET_RUN_TIME_COUNTER_VALUE( ulTotalRunTime[ 0 ] );
                #else
                    ulTotalRunTime[ 0 ] = portGET_RUN_TIME_COUNTER_VALUE();
                #endif

                /* 計算舊任務這次吃了多少 CPU 時間，累加到它的計數器中 */
                if( ulTotalRunTime[ 0 ] > ulTaskSwitchedInTime[ 0 ] )
                {
                    pxCurrentTCB->ulRunTimeCounter += ( ulTotalRunTime[ 0 ] - ulTaskSwitchedInTime[ 0 ] );
                }
                
                ulTaskSwitchedInTime[ 0 ] = ulTotalRunTime[ 0 ];
            }
            #endif /* configGENERATE_RUN_TIME_STATS */
```

- **核心計算**：`當前總時間 - 舊任務切換進來的時間 = 舊任務這次連續跑了多久`。
- **應用場景**：這就是 FreeRTOS 用來計算 **CPU 使用率（Task Runtime Stats）** 的底層依據。透過它，你可以知道哪些任務是吞噬 CPU 的大怪獸。

##### 60.1.3 堆疊溢位與環境保護

```C
/* 檢查舊任務有沒有發生 Stack Overflow（堆疊爆掉） */
            taskCHECK_FOR_STACK_OVERFLOW();

            /* 如果有開啟 POSIX 支援，換下來前先把舊任務的錯誤碼（errno）存回 TCB */
            #if ( configUSE_POSIX_ERRNO == 1 )
            {
                pxCurrentTCB->iTaskErrno = FreeRTOS_errno;
            }
            #endif
```

- **安全防禦**：在舊任務被踢下去的最後一刻，內核會去檢查它的 Stack 邊界魔術數字（Magic Number）有沒有被擦掉。如果爆了，會直接觸發 `vApplicationStackOverflowHook` 報警。

##### 60.1.4 移交權柄：選出新任務、環境復原

```C
/* 核心重頭戲：選出下一個全系統優先權最高的任務！ */
            taskSELECT_HIGHEST_PRIORITY_TASK();
            traceTASK_SWITCHED_IN();

            /* 執行硬體層級的 Hook（例如配置硬體 MPU 保護新任務的記憶體） */
            portTASK_SWITCH_HOOK( pxCurrentTCB );

            /* 把新任務上次留下的錯誤碼（errno）還給全域變數 */
            #if ( configUSE_POSIX_ERRNO == 1 )
            {
                FreeRTOS_errno = pxCurrentTCB->iTaskErrno;
            }
            #endif

            /* 執行緒區域儲存（TLS）切換：讓 C 語言標準庫（如 newlib）指標指向新任務 */
            #if ( configUSE_C_RUNTIME_TLS_SUPPORT == 1 )
            {
                configSET_TLS_BLOCK( pxCurrentTCB->xTLSBlock );
            }
            #endif
```

- **`taskSELECT_HIGHEST_PRIORITY_TASK()`**：這行執行完後，**全域指標 `pxCurrentTCB` 就已經指向新任務了！**
- **無縫接軌**：復原新任務的 `FreeRTOS_errno` 和 `TLSBlock`（Thread Local Storage）。當這個函數退出、硬體中斷結束時，CPU 就會開始跑新任務的代碼。

#### 60.2 SMP

多核心版本在核心邏輯（時間統計、選新任務、TLS 切換）上與單核心完全一樣，但**最本質的差異在於「進出函數時的加鎖與多核陣列架構」**。

##### 60.2.1 自旋鎖（Spinlock）與雙重鎖保護

```C
/* * 為了防止 Core 0 和 Core 1 同時進來修改 TCB 或 Ready List，
         * 必須獲取硬體級別的自旋鎖（Spinlock）
         */
        portGET_TASK_LOCK( xCoreID ); /* 必須先拿任務鎖 */
        portGET_ISR_LOCK( xCoreID );  /* 再拿中斷鎖 */
        {
            /* SMP 規範：不允許在嵌套臨界區中呼叫上下文切換 */
            configASSERT( portGET_CRITICAL_NESTING_COUNT( xCoreID ) == 0 );
```

- **防止死鎖（Deadlock）**：SMP 內核嚴格規定了拿鎖的順序（先 TASK 鎖、後 ISR 鎖）。如果兩個核心拿鎖順序顛倒，晶片就會永久卡死。
- **排他性**：當 Core 0 拿到了這兩把鎖在幫自己選新任務時，Core 1 如果也想換任務，就必須在 `portGET_TASK_LOCK` 外面原地打轉（Spinning）等待，直到 Core 0 釋放。

#### 60.3 補充

##### 60.3.1 為什麼多核 SMP 必須「先拿 Task Lock，再拿 ISR Lock」？

在多核心（SMP）架構中，為了防止死鎖（Deadlock），全系統所有核心都必須遵守**嚴格的拿鎖順序（Lock Ordering）**。

FreeRTOS 定義了兩把自旋鎖（Spinlock）：
- **Task Lock (任務鎖)**：保護任務狀態、延時列表、調度器狀態（如 `uxSchedulerSuspended`）。
- **ISR Lock (中斷鎖)**：保護就緒列表（Ready Lists），因為中斷處理程式（ISR）也會頻繁存取 Ready List。

##### 60.3.2 `portALT_GET_RUN_TIME_COUNTER_VALUE` 與 `portGET_RUN_TIME_COUNTER_VALUE` 差在哪？

這兩個巨集（Macros）的功能完全一樣，都是去讀取一個**高精度的硬體定時器計數器**，用來計算任務跑了多少微秒。它們的差別在於「巨集展開的語法限制（Alternative 替代方案）」。

- **`portGET_RUN_TIME_COUNTER_VALUE()` (標準版)**： 它是一個**有回傳值**的巨集。它的用法像是一個標準函數： `ulTotalRunTime = portGET_RUN_TIME_COUNTER_VALUE();`
- **`portALT_GET_RUN_TIME_COUNTER_VALUE(x)` (替代版/Alt 版)**： 它是將**接收變數當作參數傳進去**的巨集。用法是： `portALT_GET_RUN_TIME_COUNTER_VALUE( ulTotalRunTime );`

為什麼要多此一舉設計兩個？

因為 FreeRTOS 要適應成百上千種不同的 C 編譯器。在某些古老的微控制器（MCU）硬體架構或某些嚴格符合 MISRA C 規範的編譯器中，巨集如果直接寫成 `return` 一個硬體暫存器的值，可能會引發編譯器警告或類型轉換錯誤。提供一個將變數傳入直接進行修改的 `ALT`（Alternative）版本，是為了**極致的跨平台相容性**。

##### 60.3.3 C-Runtime's TLS Block 是用來幹嘛的？

**TLS** 的全稱是 **Thread Local Storage（執行緒區域儲存）**。

在標準的 C 語言中，全域變數（Global Variable）是全系統共享的。但這在多任務（多執行緒）環境下會帶來嚴重的災難。

❌ 經典災難場景：標準庫的 `strtok()` 函數

C 標準庫中的字串分割函數 `strtok()`，內部使用了一個**靜態全域變數**來記錄上一次分割到了哪裡。

1. **Task A** 呼叫 `strtok("Hello World", " ")`，標準庫內部的靜態變數記錄了 "World" 的位置。
2. 突然發生任務切換！**Task B** 醒來，也呼叫了 `strtok("Goodbye ABC", " ")`。標準庫內部的靜態變數被 Task B **覆蓋**成了 "ABC" 的位置。
3. 任務切換回來，**Task A** 繼續呼叫 `strtok(NULL, " ")` 想要拿剩下的字串，結果拿到了 Task B 的殘留資料。**程式邏輯直接崩潰！**

🛡️ TLS 的解決方案

為了讓標準 C 庫（例如嵌入式常見的 `newlib`）在多任務作業系統下安全執行，編譯器和作業系統聯合推出了 TLS。

每個任務在建立時，TCB 裡面都會分配一塊專屬於自己的記憶體，叫做 `xTLSBlock`。 當 `vTaskSwitchContext` 切換任務時，執行： `configSET_TLS_BLOCK( pxCurrentTCB->xTLSBlock );`

這行程式碼會去修改 C 標準庫內部的指針，讓標準庫知道：「**現在是 Task A 在跑，請把剛才 `strtok()` 那些全域變數讀寫權，切換到 Task A 的專屬記憶體區！**」這樣，每個任務雖然呼叫同一個 C 標準庫函數，但讀寫的都是自己獨立的副本，互不干擾。

🎛️ 4. `portTASK_SWITCH_HOOK` 裡為什麼不直接用 `xTaskSwitchContext` 傳進來的 `xCoreID`？

因為傳進來的 xCoreID 可能是其他核心的 CoreID，要執行的是正在跑 Task switch 這個 core 定義的 hook function。

### 61. *vTaskPlaceOnEventList*

`vTaskPlaceOnEventList` 是 FreeRTOS 處理阻塞機制（Blocking Mechanism）的核心函數

當一個任務嘗試去讀取一個**目前沒有資料的 Queue（佇列）**，或是想去拿一個**已被別人佔用的 Semaphore（信號量）**，且開發者設定了超時等待時間（`xTicksToWait > 0`）時，內核就會呼叫這個函數。

它的核心職責就是：**「把當前任務從就緒列表中踢下來，一隻腳踩進事件等待列表（Event List），另一隻腳踩進延時列表（Delayed List），讓任務進入睡眠狀態。」**

> [!warning] **⚠️ 呼叫此函數的絕對前提（鐵律）**

>官方用大寫寫了一行極其重要的註解：**此函數被呼叫時，調度器必須已經被掛起（Suspended），且正在被存取的 Queue 必須已經被上鎖（Locked）。** 這是為了防止在移動節點的過程中，被其他任務或中斷打斷導致鏈表結構損毀。

#### 61.1 事件鏈表插入

```C
/* 將當前任務的 xEventListItem 插入到對應的事件列表中 */
    vListInsert( pxEventList, &( pxCurrentTCB->xEventListItem ) );
```

- FreeRTOS 的通用鏈表組件 `vListInsert()`，天生是按照 `xItemValue` 的大小進行升序（由小到大）排序的。
- 但是，事件列表（Event List）的需求是：當事件觸發時（例如 Queue 有資料了），**優先權最高（Priority 最大）的任務必須最先被喚醒**。也就是說，事件列表必須按照優先權降序（由大到小）排序。

#### 61.2 加入延時列表（Time-out 機制）

```C
/* 將當前任務加入延時（阻塞）列表，設定超時時間 */
    prvAddCurrentTaskToDelayedList( xTicksToWait, pdTRUE );
```

這一步讓任務真正進入阻塞狀態。為什麼任務進了 `pxEventList`，還要進 `DelayedList`？這就是經典的「雙重喚醒渠道」設計。

當一個任務因為等待 Queue 而阻塞時，它接下來只有兩種命運可以醒來：

1. **命運 A（因事件醒來）**：在超時時間到之前，另一個任務往 Queue 丟了資料。那個任務會去翻 `pxEventList`，把排在最前面的高優先權任務撈出來喚醒。
2. **命運 B（因超時醒來）**：如果這期間一直沒有人丟資料，硬體 Tick 中斷每毫秒累加，直到時間到了。內核的 `xTaskIncrementTick()` 會去翻 `DelayedList`，發現這個任務時間到了，也會把它撈出來喚醒。

>**這行程式碼執行完後：**
	任務會被從 `pxReadyTasksLists`（就緒列表）中完全移除。調度器接下來就不會再給它任何 CPU 時間，任務正式進入睡眠。

### 62. *vTaskPlaceOnUnorderedEventList*

`vTaskPlaceOnUnorderedEventList` 是 FreeRTOS 專門為 **Event Groups（事件組）** 模組量身打造的阻塞機制核心

你剛才學的 `vTaskPlaceOnEventList` 是按照**優先權高低**來排序鏈表的（用於 Queue、Semaphore）。但今天這個函數的名字裡多了一個關鍵字：**`Unordered`（無序）**。

在事件組（Event Groups）的場景中，多個任務可能在等待不同的「事件位元（Bits）组合」，中斷或釋放者必須**遍歷（Walk through）整個列表**去檢查誰滿足條件，而不是只叫醒優先權最高的那一個。因此，它不需要耗費 CPU 時間去維護優先權順序，而是追求**極致快速的尾端插入**。

核心參數含意
- `pxEventList`：目標事件組的等待鏈表。
- `xItemValue`：**此處傳入的是該任務所期待的「事件位元（Event Bits）」**（例如：等待 Bit 0 和 Bit 2 亮起），而非上一篇的優先權。
- `xTicksToWait`：允許等待的超時滴答數。

#### 62.1 嚴格的前置條件斷言（Assertions）

```C
/* THIS FUNCTION MUST BE CALLED WITH THE SCHEDULER SUSPENDED. */
    configASSERT( uxSchedulerSuspended != ( UBaseType_t ) 0U );
```

- **強制要求掛起調度器**：修改鏈表和任務狀態是極其危險的非原子操作。此函數內部沒有關閉中斷（Critical Section），所以它**強制硬性要求**呼叫者在進來前，必須先呼叫 `vTaskSuspendAll()` 把調度器鎖死，確保不會發生任務搶佔。

#### 62.2 標籤注入：將「期待位元」與「使用中標記」綁定

```C
/* Store the item value in the event list item. */
    listSET_LIST_ITEM_VALUE( &( pxCurrentTCB->xEventListItem ), xItemValue | taskEVENT_LIST_ITEM_VALUE_IN_USE );
```

在之前的優先權事件列表中，`xItemValue` 存放的是用來排序的數學公式結果。但在這裡，它被挪作他用：
- **存放期待的 Bits**：把當前任務關心的 Event Bits（例如 `0x0005`）直接塞進 `xItemValue`。
- **`taskEVENT_LIST_ITEM_VALUE_IN_USE`**：這是一個控制位元（通常是最高位元，如 32 位元系統下的 `0x80000000`）。內核用它來做一個記號：「**這個任務的事件值現在存放的是 Event Bits，不再是優先權值囉！**」，防止內核其他模組誤讀。

#### 62.3 O(1) 複雜度的極速插入：`listINSERT_END`

```C
/* Place the event list item of the TCB at the end of the appropriate event list. */
    listINSERT_END( pxEventList, &( pxCurrentTCB->xEventListItem ) );
```

這是它與 `vTaskPlaceOnEventList` **最關鍵的技術分水嶺**：
- 之前用的 `vListInsert()` 需要從頭遍歷鏈表，找到合適的優先權位置放進去，時間複雜度是 $O(N)$。
- 這裡因為是無序的（Unordered），內核直接呼叫 `listINSERT_END()`。利用鏈表的雙向環狀結構，**直接把任務掛到鏈表的最後尾端**。時間複雜度是完美且固定的 $O(1)$，速度極快。

為什麼中斷不來攪局？（官方註解的底層秘密）

註解提到：`interrupts don't access event groups directly`。 因為 FreeRTOS 為了保證事件組的複雜邏輯不會卡死中斷，不允許在中斷（ISR）裡直接操作 Event Group。中斷如果想設定事件組，必須透過 `xEventGroupSetBitsFromISR()` 發送一個命令給**定時器守護任務（Timer Daemon Task）**，讓該任務在普通任務層級間接操作。因此，這裡不加中斷鎖是絕對安全的。

#### 62.4 納入時間管理

```C
/* 同時加入延時列表，時間到了沒等到 Bits 也要醒來 */
    prvAddCurrentTaskToDelayedList( xTicksToWait, pdTRUE );
```

與所有阻塞函數相同，雙重喚醒管道。要麼別人補齊了 Event Bits 把你叫醒，要麼時間到了硬體 Tick 中斷把你叫醒。

#### 62.5 Ordered vs Unordered

|**特性**|**vTaskPlaceOnEventList (有序)**|**vTaskPlaceOnUnorderedEventList (無序)**|
|---|---|---|
|**主要應用組件**|Queue, Semaphore, Mutex|Event Groups (事件組)|
|**`xItemValue` 存什麼**|`configMAX_PRIORITIES - uxPriority` (排序用)|`Event Bits|
|**鏈表插入函數**|`vListInsert()` (依值由小到大排序)|`listINSERT_END()` (直接塞到最後面)|
|**插入時間複雜度**|$O(N)$ (受當前阻塞任務數量影響)|$O(1)$ (極速，常數時間)|
|**中斷存取防禦**|依靠 **Queue Lock** 擋住中斷|依靠 **調度器掛起 + 中斷間接調度機制**|

### 63. *vTaskPlaceOnEventListRestricted*

這是 FreeRTOS 任務阻塞機制的**第三個孿生兄弟：核心受限版（Restricted）**：`vTaskPlaceOnEventListRestricted`。

正如函數名字中的 `Restricted`（受限）與開頭的 `#if ( configUSE_TIMERS == 1 )` 所暗示，**這個函數絕對不能被一般應用層開發者呼叫**。它是 FreeRTOS 內核專門為 **Timer Daemon Task（軟體定時器守護任務）** 量身打造的專用阻塞 API。

定時器命令佇列（Timer Command Queue）在設計上非常特殊：**全系統只有一個核心任務（Timer Task）會去讀取並等待這個佇列**。既然「永遠只有一個人會排隊」，內核就能大膽省略所有排序與競爭的保護邏輯，實現究極的效能優化。

> [!danger] **⚠️ 核心特權：為什麼叫 Restricted？**
> 普通 Queue 或 Semaphore 可能同時有 Task A、Task B、Task C 在排隊等待，所以需要按優先權排序（Ordered）。但等待軟體定時器命令的**永遠只有 Timer Task 自己**。因此，它不需要去理會別人的優先權，擁有內核開後門的「極速排隊特權」。

#### 63.1 用 $O(1)$ 的 `listINSERT_END` 替代 $O(N)$

```C
/* Place the event list item of the TCB in the appropriate event list.
         * In this case it is assume that this is the only task that is going to
         * be waiting on this event list, so the faster vListInsertEnd() function
         * can be used in place of vListInsert. */
        listINSERT_END( pxEventList, &( pxCurrentTCB->xEventListItem ) );
```

- **設計智慧**：一般版 `vTaskPlaceOnEventList` 為了按優先權排序，必須呼叫 `vListInsert()` 從頭走訪鏈表（複雜度 $O(N)$）。
- **極速優化**：這裡因為內核大膽假設（Assume）**「這個事件鏈表永遠只有 Timer Task 一個人在等」**，所以鏈表長度不是 0 就是 1。內核直接使用 `listINSERT_END()` 盲目地把節點丟到尾端。這把原本需要依數量決定速度的排序操作，直接優化成了恆定的 **$O(1)$ 常數時間**，把定時器的切換開銷降到最低。

#### 63.2 永不超時的無窮等待機制

```C
/* If the task should block indefinitely then set the block time to a
         * value that will be recognised as an indefinite delay inside the
         * prvAddCurrentTaskToDelayedList() function. */
        if( xWaitIndefinitely != pdFALSE )
        {
            xTicksToWait = portMAX_DELAY;
        }
```

- **特殊參數 `xWaitIndefinitely`**：這是其他阻塞函數沒有的參數。定時器守護任務在沒有收到任何命令時，如果設定了永久等待（`pdTRUE`），內核會強制將等待時間覆蓋為 `portMAX_DELAY`（即 `0xFFFFFFFF`）。
- **底層對接**：後續的延時列表管理函數 `prvAddCurrentTaskToDelayedList` 看到這個數值與標記，就會知道這個任務「除非有人呼叫定時器 API 發命令，否則死都不會因超時而醒來」。

#### 63.3 動態追蹤與睡眠落實

```C
traceTASK_DELAY_UNTIL( ( xTickCount + xTicksToWait ) );
        prvAddCurrentTaskToDelayedList( xTicksToWait, xWaitIndefinitely );
```

- 將 Timer Task 正式移入延時列表，並從就緒列表中剔除。此時 CPU 釋放，改去跑其他使用者的應用程式任務。

#### 63.4 比較 `vTaskPlaceOnUnorderedEventList` 和 `vTaskPlaceOnUnorderedEventList` 

| **API 函數名稱**                      | **適用場景**                | **排序規則**        | **時間複雜度** | **隱藏的底層假設**                      |
| --------------------------------- | ----------------------- | --------------- | --------- | -------------------------------- |
| `vTaskPlaceOnEventList`           | Queue, Semaphore, Mutex | **按優先權排序** (降序) | $O(N)$    | 可能有很多不同優先權的任務同時在搶這個資源。           |
| `vTaskPlaceOnUnorderedEventList`  | Event Groups (事件組)      | **無序** (隨意放尾端)  | $O(1)$    | 所有人都在等 Bits，觸發時必須全走訪，不用浪費時間排序。   |
| `vTaskPlaceOnEventListRestricted` | Timer Daemon (軟體定時器)    | **無序** (直接放尾端)  | $O(1)$    | **全系統只有我（Timer 任務）一個人在等，沒人跟我搶。** |
### 64. *xTaskRemoveFromEventList*

這是 FreeRTOS 任務阻塞與喚醒機制的**逆向終點站：喚醒（Unblock/Wakeup）**：`xTaskRemoveFromEventList`。

如果說前面學的 `vTaskPlaceOnEventList` 是把任務送去「睡覺」，那這個函數就是當 **「Queue 收到資料」或「Semaphore 被釋放（Give）」** 時，內核用來**把任務從事件列表中撈出來、拍拍它並叫它起來幹活** 的關鍵 API。

> [!warning] **呼叫前置鐵律** 註解警告：這個函數**必須在臨界區（Critical Section）中呼叫**，不論是在任務層級還是在中斷（ISR）層級。

#### 64.1 調度器狀態分支（正常喚醒 vs 賒帳暫存）

任務被移出事件列表後，接下來要回到就緒列表（Ready List）。這裡發生了分水嶺：

```C
if( uxSchedulerSuspended == ( UBaseType_t ) 0U )
    {
        /* 情況 A：調度器正常運作中。
         * 1. 把任務從 DelayedList（延時睡眠列表）拔掉 */
        listREMOVE_ITEM( &( pxUnblockedTCB->xStateListItem ) );
        /* 2. 把任務放回 ReadyList（就緒列表） */
        prvAddTaskToReadyList( pxUnblockedTCB );

        #if ( configUSE_TICKLESS_IDLE != 0 )
        {
            /* 如果開啟低功耗 Tickless 模式，重設下一個喚醒時間，確保晶片能盡量睡久一點 */
            prvResetNextTaskUnblockTime();
        }
        #endif
    }
```

- 為什麼要從 `DelayedList` 拔掉？
	- 因為這個任務是**因為拿到資料而提前醒來**，而不是因為超時（Timeout）。如果不及時在 `DelayedList` 把它擦掉，等一下滴答中斷時間到了，它又會被重複喚醒一次，導致邏輯大亂。

```C
else
    {
        /* 情況 B：調度器此時被掛起了（Suspended，例如正在執行某個臨界區）。
         * 此時內核不能動 ReadyList。所以把任務暫時塞到暫存區 xPendingReadyList 尾端。 */
        listINSERT_END( &( xPendingReadyList ), &( pxUnblockedTCB->xEventListItem ) );
    }
```

- **安全隔離**：如果此時調度器被鎖住，內核就不能直接移動任務到 Ready 列表。內核會將其安置到 `xPendingReadyList`。等隨後有人呼叫 `xTaskResumeAll()` 解鎖調度器時，內核才會把這裡的任務批量倒回 ReadyList。

#### 64.2 搶佔決策（單核心 vs 多核心 SMP）

這任務醒來了，那**現在要不要立刻切換任務，讓這個剛醒來的任務上去跑？**
##### 64.2.1 Single Core

```C
#if ( configNUMBER_OF_CORES == 1 )
    {
        /* 如果剛醒來的任務，優先權高於當前正在運行的任務 */
        if( pxUnblockedTCB->uxPriority > pxCurrentTCB->uxPriority )
        {
            /* 回傳 pdTRUE，告訴呼叫者：有大咖醒了，該強迫換人跑了！ */
            xReturn = pdTRUE;
            xYieldPendings[ 0 ] = pdTRUE; // 記上一筆任務切換請求
        }
        else
        {
            xReturn = pdFALSE; // 醒來的任務優先權不高，不急著切換
        }
    }
```

##### 64.2.2 SMP

```C
/* 多核心 SMP 版本的處理片段 */
xReturn = pdFALSE;
if( configUSE_PREEMPTION == 1 )
{
    /* 內核會去掃描全系統所有的 Core，評估這個新任務最適合去搶哪一個 Core 的資源 */
    prvYieldForTask( pxUnblockedTCB );

    /* 關鍵在這裡！prvYieldForTask 可能會去把別的 Core 的 xYieldPendings 設為 pdTRUE（發射跨核中斷）。
     * 但這裡的 if 條件只關心：【我當前正在跑的核心（portGET_CORE_ID()）】有沒有被標記為需要切換？ */
    if( xYieldPendings[ portGET_CORE_ID() ] != pdFALSE )
    {
        xReturn = pdTRUE; // 只有當前 Core 真的需要切換時，才回傳 pdTRUE
    }
}
```

> [!important] **為什麼 `xReturn` 必須鎖定「當前 Core」？** 因為這個函數的回傳值，通常會被當作參數直接餵給 `xQueueSendFromISR(..., &xHigherPriorityTaskWoken)` 尾巴的變數。中斷程式是在**當前核心**被觸發的，在退出中斷前，呼叫者會執行`portYIELD_FROM_ISR(xHigherPriorityTaskWoken)`。這個巨集只能去切換**當前核心**。因此，`xReturn` 必須忠實反映「當前核心是否需要 Yield」。

#### 64.2.3 補充

為什麼 `uxSchedulerSuspended != 0` 時不用把任務從 `delayedlist` 拔除，而只是放入 `pendingReadylist`

當 `uxSchedulerSuspended != 0` 時，中斷不直接操作 `readylist/delayedlist` 絕非因為 CS 鎖不住，而是：
1. **維持暫存記錄**：`xPendingReadyList` 作為調度器凍結期間的「非同步事件緩衝區」，讓 `xTaskResumeAll()` 在解凍時能對所有新喚醒的任務進行集中、統一的宏觀調度評估。
2. **降低中斷延遲（核心目的）**：把耗時的鏈表搬移動作（從延時列表拔除、分類塞入就緒列表）**延遲（Defer）**到任務層級的 `xTaskResumeAll()` 中執行。中斷層級只負責最簡單的暫存，從而實現了 FreeRTOS **極致短暫的核心中斷關閉時間**。

### 65. *vTaskRemoveFromUnorderedEventList*

這份函數是 FreeRTOS 喚醒機制的**最後一塊拼圖**：`vTaskRemoveFromUnorderedEventList`（從無序事件列表中移除並喚醒任務）。

它是 `vTaskPlaceOnUnorderedEventList` 的逆向操作，專門服務於 **Event Groups（事件組）**。當某個任務（或定時器守護任務）更新了事件位元（Bits），發現排隊中的某個任務所期待的條件被滿足了，就會呼叫這個函數把該任務叫醒。

> [!important] **參數設計的細微差異** 注意它的第一個參數是 `ListItem_t * pxEventListItem`（**任務的節點本身**），而不是常規版的 `List_t * pxEventList`（整個鏈表）。 因為無序鏈表沒有優先權順序，呼叫者（Event Groups）是透過**手動走訪（Walkthrough）**整個列表，揪出哪一個任務符合 Bits 條件，然後直接把該任務的「節點指標」餵給這個函數來指定解鎖。

#### 65.1 回傳解鎖時的「事件狀態」

```C
/* 將解鎖時的事件狀態（新 Bits 值）寫入節點的 Value 中，並打上 IN_USE 標記 */
    listSET_LIST_ITEM_VALUE( pxEventListItem, xItemValue | taskEVENT_LIST_ITEM_VALUE_IN_USE );
```

- **傳遞戰果**：當任務因為事件組符合條件而被喚醒時，它必須知道「到底是哪些 Bits 成立把我叫醒的？」。內核利用這個節點的 `xItemValue` 當作臨時通訊兵，把目前的事件狀態塞進去。等一下任務真正醒來時，會去讀取這個值來做後續判斷。

#### 65.2 斷開事件鏈表

```C
/* 透過節點找到它所屬的 TCB 擁護者 */
    pxUnblockedTCB = listGET_LIST_ITEM_OWNER( pxEventListItem );
    configASSERT( pxUnblockedTCB );
    
    /* 將該任務從事件組的無序等待鏈表中拔除 */
    listREMOVE_ITEM( pxEventListItem );
```

- 讓任務與事件組脫離關係，不再參與後續的事件追蹤。

#### 65.3 核心大盲點解破：為什麼這裡敢直接操作 ReadyList？

```C
/* 強制斷言：調度器此時【必須】是被掛起的！ */
    configASSERT( uxSchedulerSuspended != ( UBaseType_t ) 0U );

    /* ！！！突破盲點的時刻！！！
     * 調度器明明被掛起了，內核卻直接把任務從延時列表拔掉，並直接塞進 ReadyList！ */
    listREMOVE_ITEM( &( pxUnblockedTCB->xStateListItem ) );
    prvAddTaskToReadyList( pxUnblockedTCB );
```

為什麼上一篇常規版在調度器掛起時要丟進 `xPendingReadyList` 賒帳，而這一篇卻敢直接動 `ReadyList`？

Ans: `Interrupts do not access event flags.`（中斷不會直接存取事件組） `The scheduler is suspended so interrupts will not be accessing the ready lists.`

1. **常規版 `xTaskRemoveFromEventList`**：可以被**中斷（ISR）呼叫（例如 `xQueueSendFromISR`）。當中斷發生時，任務層級可能正在修改鏈表，所以如果調度器掛起，中斷進來**絕對不能**亂動 `ReadyList/DelayedList`，否則會發生 Race Condition，所以必須丟到 `xPendingReadyList` 緩衝。
2. **本函數 `vTaskRemoveFromUnorderedEventList`**：FreeRTOS 嚴格規定，**中斷（ISR）絕對不允許直接操作事件組**！中斷如果想動事件組，必須依賴定時器任務進行非同步延遲調度。
3. **結論**：這意味著這個函數**只會在任務層級（Task Level）被呼叫**。既然它是在任務層級執行，且此時調度器已經被掛起（`uxSchedulerSuspended != 0`），那就意味著**全系統絕對沒有任何其他任務可以搶佔它**，同時也**沒有任何中斷會進來操作 ReadyList**。環境是純淨且絕對安全的！因此，內核不需要浪費時間去丟 `xPendingReadyList` 賒帳，直接大刀闊斧修改 `ReadyList` 效率最高！

#### 66.4 延遲搶佔標記

```C
#if ( configNUMBER_OF_CORES == 1 )
    {
        if( pxUnblockedTCB->uxPriority > pxCurrentTCB->uxPriority )
        {
            /* 新醒來的任務優先權比我高，需要切換任務。
             * 但因為此時調度器被掛起了，我們不能立刻觸發中斷切換。
             * 所以在當前 Core 的 xYieldPendings[0] 記下一筆 pdTRUE。
             * 當隨後解鎖調度器（xTaskResumeAll）時，內核一看到這個標記，就會立刻補做 Context Switch。 */
            xYieldPendings[ 0 ] = pdTRUE;
        }
    }
```

- **按兵不動**：在單核心下，如果新任務優先權高，先做個記號 `xYieldPendings[0] = pdTRUE`。等調度器恢復時再執行切換，確保不破壞掛起期間的語意。

```C
#else /* 多核心 SMP 版本 */
    {
        #if ( configUSE_PREEMPTION == 1 )
        {
            /* 多核心環境下，必須進入臨界區保護，
             * 去評估要不要發射跨核心中斷（IPI）叫別的核心讓路 */
            taskENTER_CRITICAL();
            {
                prvYieldForTask( pxUnblockedTCB );
            }
            taskEXIT_CRITICAL();
        }
        #endif
    }
    #endif
```

#### 66.5 FreeRTOS 兩大喚醒 API 對比

| **特性**          | **xTaskRemoveFromEventList (常規版)**               | **vTaskRemoveFromUnorderedEventList (無序事件組版)**  |
| --------------- | ------------------------------------------------ | ----------------------------------------------- |
| **誰能呼叫它**       | **任務（Task）與 中斷（ISR）** 皆可                         | **僅限任務（Task）** (中斷禁入)                           |
| **呼叫時的前提條件**    | 必須在 **Critical Section** 內                       | 必須在 **調度器掛起** (`uxSchedulerSuspended != 0`) 狀態下 |
| **若調度器被掛起時的行為** | **不敢動 ReadyList** ──► 暫存入 `xPendingReadyList` 賒帳 | **直接動 ReadyList** ──► 搬移節點一步到位                  |
| **為什麼敢/不敢**     | 因為中斷可能會在凍結期間進來干擾鏈表。                              | 因為中斷不碰事件組，且調度器已掛起，環境100%安全。                     |

### 67. *vTaskSetTimeOutState*

這個函數是 FreeRTOS 用來處理相對時間超時與防溢出機制（Timeout & Anti-overflow Mechanism）的幕後大功臣：`vTaskSetTimeOutState`。

當你呼叫一個阻塞函數（例如 `xQueueReceive()`）並設定了超時時間（例如 50ms），如果佇列一直沒有資料，任務會進去睡覺。但如果任務睡到一半，因為某些特殊原因被短暫喚醒（例如收到了非目標信號），隨後發現事情還沒做完、需要「繼續睡完剩下的時間」時，內核要怎麼知道**已經過去了多久？還剩下多少時間要睡？** 這個函數就是用來**為當前的時間點拍下一張「絕對時間快照」**，作為日後計算剩餘時間的基準點。

> [!important] **什麼是 `TimeOut_t` 結構體？** 在 FreeRTOS 中，這個結構體非常簡單，它只記錄兩個東西：
> 
> - `xTimeOnEntering`：進入函數這一刻的系統滴答計數（`xTickCount`）。
>     
> - `xOverflowCount`：進入函數這一刻，系統計數器已經發生過幾次「翻轉溢出」（`xNumOfOverflows`）。

#### 67.1 這張「快照」後續怎麼用？

單看 `vTaskSetTimeOutState` 會覺得它很廢，因為它只負責記錄。這張照片真正發揮威力的地方，是在後續被 **`xTaskCheckForTimeOut()`** 呼叫的時候。

經典應用場景：佇列反覆阻塞
想像一個場景，任務 A 想從 Queue 讀取資料，設定超時 100 滴答（Ticks）。

```
時間軸 (Timeline) 
───►【步驟 1】任務呼叫 vTaskSetTimeOutState() 拍下快照。 (此時 xTickCount = 1000, 預計 1100 超時)
		│ 
		▼ 進入睡眠... 
───►【步驟 2】在第 1040 Ticks 時，突然一個無關的中斷把任務 A 叫醒。 任務 A 醒來一看：「咦？Queue 裡面還是沒資料啊，可憐的我必須繼續睡。」 
		│
		▼
───►【步驟 3】內核呼叫 xTaskCheckForTimeOut()，把當前的時間 (1040) 跟當初的【快照(1000)】做減法： $$1040 - 1000 = 40 \text{ (代表已經消耗了 40 Ticks)}$$ $$100 - 40 = 60 \text{ (代表還剩下 60 Ticks 可以睡)}$$ 
		│
		▼
───►【步驟 4】內核自動把超時時間修正為 60 Ticks，再次把任務送回睡眠列表。
```

**溢出怎麼辦？** 如果在睡覺期間，`xTickCount` 從 `4294967290` 翻轉溢出變成了 `10`。 `xTaskCheckForTimeOut` 就會去對比前後的 `xOverflowCount`。如果發現後來的溢出次數多了 1，它會透過補碼數學運算，自動把這跨越溢出邊界的差額精準計算出來，絕對不會漏算 1 毫秒。

```C
void vTaskSetTimeOutState( TimeOut_t * const pxTimeOut )
{
    traceENTER_vTaskSetTimeOutState( pxTimeOut );

    configASSERT( pxTimeOut );
    taskENTER_CRITICAL();
    {
        pxTimeOut->xOverflowCount = xNumOfOverflows;
        pxTimeOut->xTimeOnEntering = xTickCount;
    }
    taskEXIT_CRITICAL();

    traceRETURN_vTaskSetTimeOutState();
}
```

### 68. *vTaskInternalSetTimeOutState*

跟 [[#67. vTaskSetTimeOutState]] 一樣，只差有沒有進 CS，因此在呼叫此函數前要確保前已經身處臨界區之內


### 69. *xTaskCheckForTimeOut*

這就是 FreeRTOS 時間管理機制中的**終極大腦**：`xTaskCheckForTimeOut`。

它是前面幾天我們學過的所有時間函數（`vTaskSetTimeOutState`、`vTaskInternalSetTimeOutState`）的**最終消費者**。在常規的阻塞 API（如 `xQueueReceive` 或 `xSemaphoreTake`）中，內核通常會用一個 `while` 迴圈包住阻塞邏輯。如果任務中途因為非目標事件被喚醒，就會呼叫這個函數來精準計算：**「我到底超時了沒？如果還沒，我剩下多少時間需要繼續睡？」**

#### 69.1 基礎宣告與時間差計算

```C
/* 微小優化：用區域常數鎖定當前的 Tick 數，避免在計算中發生非預期的微幅變動 */
        const TickType_t xConstTickCount = xTickCount;
        
        /* 計算從「拍快照那一刻」到「現在」已經過去了多少時間（Elapsed Time） */
        const TickType_t xElapsedTime = xConstTickCount - pxTimeOut->xTimeOnEntering;
```

- **無懼溢出的無號數減法**：在 C 語言的無號數（`uint32_t`）運算中，即使 `xConstTickCount` 發生了溢出歸零（例如變成 5），而當時進入時間 `xTimeOnEntering` 是 `0xFFFFFFF0`，相減的結果 $5 - 0xFFFFFFF0$ 依然會透過二補數精準得到真實過去的時間 `21`。

#### 69.2 特殊攔截：人為中止與無窮等待

```C
#if ( INCLUDE_xTaskAbortDelay == 1 )
            /* 情況一：如果其他任務強行呼叫了 xTaskAbortDelay() 想要強制拔除此任務的睡眠 */
            if( pxCurrentTCB->ucDelayAborted != ( uint8_t ) pdFALSE )
            {
                pxCurrentTCB->ucDelayAborted = ( uint8_t ) pdFALSE; // 清除標記
                xReturn = pdTRUE; // 假裝成超時，逼外層迴圈退出並醒來
            }
            else
        #endif

        #if ( INCLUDE_vTaskSuspend == 1 )
            /* 情況二：如果最初設定的阻塞時間是 portMAX_DELAY（死等） */
            if( *pxTicksToWait == portMAX_DELAY )
            {
                xReturn = pdFALSE; // 永遠不超時，繼續回去睡
            }
            else
        #endif
```

#### 69.3 極端特例：完全繞了一大圈的「超級超時」

這是整個函數中最難理解的硬核邏輯：

```C
if( ( xNumOfOverflows != pxTimeOut->xOverflowCount ) && ( xConstTickCount >= pxTimeOut->xTimeOnEntering ) )
        {
            /* 情況三：系統計數器發生了翻轉（溢出），且當前的 Tick 數竟然還大於等於當初進入的時間！ */
            xReturn = pdTRUE;
            *pxTicksToWait = ( TickType_t ) 0;
        }
```

為什麼這樣代表必定超時？
想像一個 32 位元的 Tick 系統：
- 任務在 `xTimeOnEntering = 4,000,000,000` 時進去睡覺。
- 當任務醒來檢查時，`xNumOfOverflows` 改變了（說明時間軸已經跨越了 `4,294,967,295` 歸零了）。
- 如果此時 `xConstTickCount` 讀出來是 `4,100,000,000`。

這意味著：時間從 40 億漲到 42.9 億，**歸零**，然後又一路上漲越過了 40 億來到了 41 億！這代表時間整整過去了**一整個週期（超過 42 億個 Ticks）**。既然任何任務的最大等待時間都不可能超過 `portMAX_DELAY - 1`，那時間過去了一整圈，肯定早就不知超時到哪裡去了。所以直接判定 `pdTRUE`（超時）。

#### 69.4 核心骨幹：續睡 recalibration（連動私有版快照）

這是最常見的日常場景，也是這個函數的精華所在：

```C
else if( xElapsedTime < *pxTicksToWait )
        {
            /* 情況四：已經過去的時間（xElapsedTime）比當初預期要等待的時間（*pxTicksToWait）還要短！
             * 代表這是【中途被意外喚醒】（例如搶資源失敗），還沒真正超時。 */
            
            /* 1. 將剩餘需要等待的時間，扣除已經過去的時間 */
            *pxTicksToWait -= xElapsedTime;
            
            /* 2. 關鍵連動！！！呼叫全裸優化版函數，將時間快照更新為「當下」 */
            vTaskInternalSetTimeOutState( pxTimeOut );
            
            xReturn = pdFALSE; // 告訴外層：還沒超時喔！
        }
        else
        {
            /* 情況五：過去的時間大於或等於預期等待的時間，這是【真正的時間到期超時】 */
            *pxTicksToWait = ( TickType_t ) 0;
            xReturn = pdTRUE; // 告訴外層：時間到，可以名正言順醒來了！
        }
```

為什麼在這裡需要呼叫 `vTaskInternalSetTimeOutState` 更新快照？

這是一個非常漂亮的數學遞減思維。 假設最初你想睡 100 滴答。

- **第一次意外醒來**：過去了 40 滴答。內核把 `*pxTicksToWait` 修改成 $100 - 40 = 60$ 滴答。同時，把快照 `xTimeOnEntering` **更新為現在的時間點** 
- **第二次意外醒來**：又過去了 10 滴答。因為快照已經更新過了，所以這一次算出來的 `xElapsedTime` 就會是純粹的 10。內核可以很乾淨地用新的剩餘時間去扣減：$60 - 10 = 50$ 滴答
- 如果這裡不更新快照，下一次計算 `xElapsedTime` 時就會從最原始的起點開始算，整個多層阻塞的代碼會變得極其臃腫且容易出錯。

### 70. *vTaskMissedYield*

這個函數看起來只有短短一行核心代碼，但它卻是 FreeRTOS 處理**非同步內核同步（Kernel Synchronization）與多核心（SMP, Symmetric Multiprocessing）任務調度**時極度精妙的「記帳本」。

它是一個被宣告為 `PRIVILEGED_FUNCTION` 的內部私有 API，絕對不允許一般應用程式呼叫。它的唯一使命是：**「記下一筆帳，告訴目前的 CPU 核心：你欠系統一次上下文切換（Context Switch），等手邊的緊急工作忙完，請立刻補上！」**

#### 70.1 為什麼叫「錯過的調度（Missed Yield）」？核心應用場景

要理解這個函數，就必須知道它是寫來服務誰的。這個函數在整個 FreeRTOS 原始碼中，幾乎**專門服務於 `queue.c` 裡面的 `prvUnlockQueue()`（解鎖佇列）**。

想像以下驚險的核心運作場景：

```C
【核心場景軸】 
1. 任務 A 正在讀取 Queue，為了安全，系統暫時將調度器掛起（Suspended）。
2. 此時，一個硬體中斷（ISR）爆發，往這個 Queue 塞了新資料
3. 中斷發現：有一個【更高優先權的任務 B】正因為等這個 Queue 而在睡覺，現在資料來了，任務 B 應該立刻醒來並執行！
4. 依照 RTOS 的即時原則，CPU 應該馬上放棄任務 A，改去執行任務 B（這叫 taskYIELD）。
5. 🚨 糟糕！現在調度器被掛起了，且我們正處於解除佇列鎖定的關鍵核心迴圈中。 如果這時候強行切換硬體上下文，會把整個內核的狀態機（State Machine）直接踩碎！
6. 內核大腦決定：此時不能當場切換！於是呼叫了 vTaskMissedYield()。 在當前核心的帳本上寫下 xYieldPendings[core] = pdTRUE（這就是「錯過/延遲」了這次調度）。
7. 任務 A 繼續安全地執行完解鎖佇列的程式碼。
8. 恢復安全：當任務 A 呼叫 xTaskResumeAll() 準備完全恢復調度器時，內核回頭一看帳本：「喔！剛剛欠了一次調度！」
9. 補發調度：核心立刻補發一次真正的硬體 taskYIELD()，任務 B 順利在最安全的時刻被換上來執行。
```

#### 70.2 `taskYIELD()` vs `vTaskMissedYield()` 對比

| **特性**   | **taskYIELD() (主動觸發)**                         | **vTaskMissedYield() (延遲記帳)**                 |
| -------- | ---------------------------------------------- | --------------------------------------------- |
| **執行時機** | 系統處於安全狀態，隨時可以切換任務。                             | 內核正處於敏感狀態（如解鎖隊列），不適合當場切換。                     |
| **硬體動作** | **立刻**觸發硬體中斷（如 ARM 的 PendSV），直接強制進行 CPU 上下文切換。 | **不觸發任何硬體動作**，僅在記憶體變數中將標記設為 `pdTRUE`。         |
| **執行代價** | 較高。需要保存當前所有暫存器、切換堆疊、加載新任務暫存器。                  | 極低。僅是一次記憶體寫入（1 個時脈週期）。                        |
| **誰來收尾** | 硬體中斷執行完切換後，直接執行新任務。                            | 後續由解鎖函數（如 `xTaskResumeAll`）在安全退出時，檢查此標記並補發切換。 |
### 71. *uxTaskGetTaskNumber*

這個函數 `uxTaskGetTaskNumber` 是 FreeRTOS 專門為**外部除錯、系統分析工具（如 Percepio Tracealyzer、SystemView）以及開發者自定義追蹤**所保留的「VIP 通道」。

> [!info] **核心定位：什麼是 `uxTaskNumber`？** 在 FreeRTOS 中，每個任務都有一個記憶體地址（即 `TaskHandle_t` 指標）。但地址是隨機且不直觀的（例如 `0x200014B0`）。 為了讓除錯工具或 CLI 介面能用「簡單的整數（如 1, 2, 3）」來識別任務，內核在 TCB 中預留了 `uxTaskNumber` 欄位。這個函數就是用來讀取這個編號的。

#### 71.1 條件編譯看門狗

```C
#if ( configUSE_TRACE_FACILITY == 1 )
```
- **功能裁剪**：FreeRTOS 為了極致節省 Flash 和 RAM，任何非必要的除錯功能都會用宏保護起來。只有當你在 `FreeRTOSConfig.h` 中將 `configUSE_TRACE_FACILITY` 定義為 1 時，內核才會把 `uxTaskNumber` 欄位塞進 TCB 中，這個函數也才會被編譯。如果設為 0，這段程式碼完全不佔用任何晶片空間。

#### 71.2 這裡為什麼不需要 Critical Section（臨界區）保護？
1. **它是原子操作（Atomic Operation）**：`uxTaskNumber` 的型態是 `UBaseType_t`（在 32 位元 MCU 上就是 `uint32_t`）。對 CPU 來說，讀取一個與核心位元數對齊的記憶體地址，是一條指令就能搞定的硬體原子操作，**絕對不會讀到一半被中斷打斷、導致讀到錯亂的半個資料**。
2. **它不參與排程決策**：這個值是死是活、改多改少，完全不會影響 ReadyList 或 CPU 目前要跑哪一個任務。它純粹是給人看的（診斷用），所以不需要為了它去關閉中斷、犧牲系統的即時響應性能。

### 72. *vTaskSetTaskNumber*

跟 [[#71. uxTaskGetTaskNumber]] 類似但是是修改 `uxTaskNumber`

### 73. *portTASK_FUNCTION (prvPassiveIdleTask)*


這是 FreeRTOS 邁向**多核心（SMP, Symmetric Multiprocessing）架構**後的核心產物：`prvPassiveIdleTask`（被動空閒任務）。

portTASK_FUNCTION 是一個巨集

```C
#define portTASK_FUNCTION( vFunction, pvParameters ) void vFunction( void *pvParameters )
```

在傳統單核心系統中，全系統只有一個「空閒任務（Idle Task）」。但在多核心系統（例如 2 核、4 核甚至更多核）中，當多個核心同時沒有任務可跑時，每個核心都必須有一個死迴圈可以待命。

FreeRTOS 的設計哲學是：全系統只能有 **1 個「主動空閒任務（Active Idle Task）」**（通常在主核心運行，負責清理已刪除任務的記憶體），而其餘核心在沒事做時，執行的全部都是這個 **「被動空閒任務（Passive Idle Task）」**。

#### 73.1 條件編譯與開場讓步（Startup Yield）

```C
#if ( configNUMBER_OF_CORES > 1 )
    static portTASK_FUNCTION( prvPassiveIdleTask, pvParameters )
    {
        ( void ) pvParameters; // 防止編譯器產生 "Unused Parameter" 警告

        taskYIELD(); // 開場直接主動讓出 CPU 一次

        for( ; configCONTROL_INFINITE_LOOP(); )
        {
```

- **多核心守護**：只有當配置的核心數 `configNUMBER_OF_CORES > 1` 時，才會編譯此函數。
- **開場 `taskYIELD()` 的妙處**：當次核心（Secondary Cores）剛啟動、初始化完成並進入這個任務時，可能系統中已經有其他準備就緒的使用者任務被分配到了這個核心。所以一進來先無條件交出一次指揮權，確保其他即時任務能第一時間被執行。
- **`configCONTROL_INFINITE_LOOP()`**：FreeRTOS 為了通過某些安全認證（如 MISRA C），將傳統的 `for(;;)` 抽象化成的巨集，本質上就是 `1`（死迴圈）。

#### 73.2 合作式（非搶占）模式下的讓步

```C
#if ( configUSE_PREEMPTION == 0 )
            {
                /* 如果關閉了搶占模式（使用合作式排程），我們必須不停地強制進行任務切換，
                 * 看看是否有其他任務變成可用狀態。 */
                taskYIELD();
            }
            #endif /* configUSE_PREEMPTION */
```

- **邏輯分析**：在非搶占式（Cooperative）系統中，高優先權任務「醒來」時是無法主動踢走當前任務的。如果目前核心正在跑空閒任務，它就必須不斷自覺地呼叫 `taskYIELD()` 刷新排程，給其他剛醒來的任務上場的機會。

#### 73.3 搶占模式下的「精準禮讓」演算法

```C
#if ( ( configUSE_PREEMPTION == 1 ) && ( configIDLE_SHOULD_YIELD == 1 ) )
            {
                /* 如果開啟了搶占，且使用者允許空閒優先權任務主動讓步給同級任務... */
                if( listCURRENT_LIST_LENGTH( &( pxReadyTasksLists[ tskIDLE_PRIORITY ] ) ) > ( UBaseType_t ) configNUMBER_OF_CORES )
                {
                    taskYIELD();
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
            #endif
```

深度拆解：為什麼就緒列表長度大於核心數（`> configNUMBER_OF_CORES`）代表有其他任務想跑？

假設這是一個 **4 核心系統（`configNUMBER_OF_CORES = 4`）**：

- 系統中必然會有 1 個 Active Idle Task 和 3 個 Passive Idle Task，**總共 4 個空閒任務**。
- 空閒任務的優先權都是最低的 `tskIDLE_PRIORITY`（也就是 0）。
- 當全系統都沒事做時，這 4 個空閒任務都會被掛在 `pxReadyTasksLists[0]`（優先權 0 的就緒列表）裡面。此時鏈表的長度剛好等於 `4`（也就是核心數）。

**關鍵翻轉**：

如果今天使用者自己建立了一個普通的任務（例如 `vMyTask`），而且**故意把它的優先權也設為 0**。 當 `vMyTask` 進入就緒狀態時，它也會被塞進 `pxReadyTasksLists[0]`。此時鏈表長度就變成了 **`5`**。

當被動空閒任務執行到這裡，發現 `5 > 4`（鏈表長度大於核心數），它心裡就明白了：「**不對勁！這個清單裡除了我們這 4 個混日子的空閒任務之外，還塞進了別人的任務！**」 為了不讓空閒任務霸佔整格時間片（Timeslice），它會立刻執行 `taskYIELD()`，主動把這個同為優先權 0 的使用者任務換上來執行。

- **無鎖優化**：註解特別提到 `A critical region is not required here...`（這裡不需要臨界區保護）。因為這只是純讀取鏈表長度，即使因為非同步讀到了一個稍微落後的錯誤值，大不了就是下一輪再讓步，對內核安全毫無影響，卻能幫次核心省下昂貴的跨核心鎖（Spinlock）開銷。

#### 73.4 次核心低功耗管理：被動空閒鉤子（Passive Idle Hook）

```C
#if ( configUSE_PASSIVE_IDLE_HOOK == 1 )
            {
                /* 呼叫使用者自定義的被動空閒鉤子函數 */
                vApplicationPassiveIdleHook();
            }
            #endif /* configUSE_PASSIVE_IDLE_HOOK */
```

- **功耗控制的戰略要地**：在多核心晶片（如 ESP32、RP2040）中，省電是極其重要的課題。
- 當某個次核心沒事做、進入被動空閒任務時，我們不能讓它一直空轉燒電。開發者可以開啟這個 Hook，在 `vApplicationPassiveIdleHook()` 裡面寫下硬體省電指令（例如 ARM 的 `WFI - Wait For Interrupt`，或直接關閉該核心的時鐘）。

> [!danger] **⚠️ 鐵律警告 (Strict Rule)** 註解用全大寫強調：`vApplicationPassiveIdleHook() MUST NOT, UNDER ANY CIRCUMSTANCES, CALL A FUNCTION THAT MIGHT BLOCK.` **空閒任務絕對不允許進入阻塞（Block）狀態！**（例如在裡面呼叫 `vTaskDelay` 或等待訊號量）。因為空閒任務是核心排程的最後防線，如果連空閒任務都去睡覺了，排程器在選擇下一個任務時就會面臨「開天窗」的無解窘境，直接引發系統崩潰（Panic）。

### 74. *portTASK_FUNCTION (prvIdleTask)*

這段程式碼是 **FreeRTOS** 核心中非常關鍵的 **空閒任務（Idle Task）**。當系統中沒有其他更高優先級的任務處於就緒狀態（Ready）時，排程器就會執行這個任務。它的主要職責包括：釋放已被刪除任務的記憶體、執行用戶自訂的 Hook 函式，以及進入省電模式（Tickless Idle）。

#### 74.1 任務宣告與參數初始化

```C
/*
 * -----------------------------------------------------------
 * The idle task.
 * ----------------------------------------------------------
 */
static portTASK_FUNCTION( prvIdleTask, pvParameters )
{
    /* Stop warnings. */
    ( void ) pvParameters;

    /** THIS IS THE RTOS IDLE TASK - WHICH IS CREATED AUTOMATICALLY WHEN THE
     * SCHEDULER IS STARTED. **/
```

- **`portTASK_FUNCTION`**：這是一個巨集（Macro），用來相容不同編譯器特定的語法擴充，它在本質上等同於 `void prvIdleTask( void *pvParameters )`。
- **`( void ) pvParameters;`**：空閒任務不需要傳入參數，這行程式碼純粹是為了騙過編譯器，防止編譯時跳出「變數未使用（Unused parameter）」的警告。

#### 74.2 安全機制與多核心（SMP）初始讓出

```C
/* In case a task that has a secure context deletes itself, in which case
     * the idle task is responsible for deleting the task's secure context, if
     * any. */
    portALLOCATE_SECURE_CONTEXT( configMINIMAL_SECURE_STACK_SIZE );

    #if ( configNUMBER_OF_CORES > 1 )
    {
        /* SMP all cores start up in the idle task. This initial yield gets the application
         * tasks started. */
        taskYIELD();
    }
    #endif /* #if ( configNUMBER_OF_CORES > 1 ) */
```

- **`portALLOCATE_SECURE_CONTEXT`**：用於支援 ARM TrustZone 等硬體安全架構。如果某個擁有安全模組上下文（Secure Context）的任務把自己刪除了，空閒任務必須有能力接管並釋放該安全上下文，因此它自己需要先配置好安全空間。
- **多核心支援（SMP）**：如果配置了多核心（`configNUMBER_OF_CORES > 1`），系統啟動時所有核心都會先進入空閒任務。這裡主動呼叫 `taskYIELD()` 讓出 CPU，是為了引導各個核心去撈取並執行真正的應用程式任務。

#### 74.3 主虛擬無窮迴圈與回收已刪除任務

```C
for( ; configCONTROL_INFINITE_LOOP(); )
    {
        /* See if any tasks have deleted themselves - if so then the idle task
         * is responsible for freeing the deleted task's TCB and stack. */
        prvCheckTasksWaitingTermination();
```

- **`configCONTROL_INFINITE_LOOP()`**：通常展開後就是 `1`，即 `for(;;)` 死迴圈。
- **`prvCheckTasksWaitingTermination()`（極重要）**：這是空閒任務最核心的職責之一。當你在其他任務呼叫 `vTaskDelete(NULL)` 自殺時，FreeRTOS **無法在當下立刻釋放** 該任務自身的 TCB（任務控制塊）和堆疊（Stack）記憶體（因為自己不能砍掉自己正在使用的腳踏板）。因此，FreeRTOS 會把被刪除的任務丟進一個等待釋放的鏈結串列，由空閒任務在這裡進行實質的記憶體回收（`free`）。

#### 74.4 合作式排程下的讓出機制

```C
#if ( configUSE_PREEMPTION == 0 )
        {
            /* If we are not using preemption we keep forcing a task switch to
             * see if any other task has become available.  If we are using
             * preemption we don't need to do this as any task becoming available
             * will automatically get the processor anyway. */
            taskYIELD();
        }
        #endif /* configUSE_PREEMPTION */
```

- **合作式排程（Non-preemptive）**：當 `configUSE_PREEMPTION == 0` 時，高優先級任務不會自動搶佔 CPU。此時，空閒任務必須自覺地、不斷地呼叫 `taskYIELD()`，主動詢問並交出主控權，看看有沒有其他同為優先級 0 的任務或被喚醒的任務需要執行。

#### 74.5 同優先級任務的時間片輪轉（Time-slicing）最佳化

```C
#if ( ( configUSE_PREEMPTION == 1 ) && ( configIDLE_SHOULD_YIELD == 1 ) )
        {
            /* When using preemption tasks of equal priority will be
             * timesliced.  If a task that is sharing the idle priority is ready
             * to run then the idle task should yield before the end of the
             * timeslice. */
            if( listCURRENT_LIST_LENGTH( &( pxReadyTasksLists[ tskIDLE_PRIORITY ] ) ) > ( UBaseType_t ) configNUMBER_OF_CORES )
            {
                taskYIELD();
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
        #endif /* ( ( configUSE_PREEMPTION == 1 ) && ( configIDLE_SHOULD_YIELD == 1 ) ) */
```

- **`configIDLE_SHOULD_YIELD`**：如果用戶定義此項為 `1`，代表**空閒任務不應該霸佔整段時間片**。
- **邏輯判斷**：它會檢查優先級為 0（最低優先級 `tskIDLE_PRIORITY`）的就緒任務串列長度。如果長度大於當前的核心數（`configNUMBER_OF_CORES`），代表除了空閒任務本身之外，**還有其他用戶建立的優先級 0 任務正在排隊**。此時，空閒任務會立刻 `taskYIELD()`，把 CPU 讓給使用者的任務，不會浪費多餘的 Tick。

#### 74.6 執行使用者自訂的 Idle Hook

```C
#if ( configUSE_IDLE_HOOK == 1 )
        {
            /* Call the user defined function from within the idle task. */
            vApplicationIdleHook();
        }
        #endif /* configUSE_IDLE_HOOK */
```

- **`vApplicationIdleHook()`**：當開啟此功能時，系統每次進入空閒狀態都會呼叫這個由使用者實現的函式。
- **常見用途**：用來計算 CPU 使用率、餵看門狗（Watchdog）、或者是讓某些不影響即時性的背景低優先級燈號閃爍。

#### 74.7 低功耗省電模式（Tickless Idle）前期評估

```C
#if ( configUSE_TICKLESS_IDLE != 0 )
        {
            TickType_t xExpectedIdleTime;

            /* It is not desirable to suspend then resume the scheduler on
             * each iteration of the idle task.  Therefore, a preliminary
             * test of the expected idle time is performed without the
             * scheduler suspended.  The result here is not necessarily valid. */
            xExpectedIdleTime = prvGetExpectedIdleTime();

            if( xExpectedIdleTime >= ( TickType_t ) configEXPECTED_IDLE_TIME_BEFORE_SLEEP )
            {
```

- **Tickless Idle 概念**：傳統 RTOS 每毫秒都會觸發一次 Tick 中斷，這會不斷喚醒 CPU。Tickless 模式允許在系統預期會空閒很長一段時間時，**直接關閉/調整計時器中斷**，讓 CPU 深度睡眠。
- **`prvGetExpectedIdleTime()`**：第一次快取評估，估算下一個任務被喚醒前還有多少時間（Ticks）。
- **第一層過濾**：如果預期空閒時間大於設定的門檻值（`configEXPECTED_IDLE_TIME_BEFORE_SLEEP`），才會考慮進入睡眠，避免頻繁開關睡眠模式帶來的硬體切換開銷。

#### 74.8 進入低功耗睡眠模式（核心臨界區維護）

```C
vTaskSuspendAll();
                {
                    /* Now the scheduler is suspended, the expected idle
                     * time can be sampled again, and this time its value can be used. */
                    configASSERT( xNextTaskUnblockTime >= xTickCount );
                    xExpectedIdleTime = prvGetExpectedIdleTime();

                    /* Define the following macro to set xExpectedIdleTime to 0
                     * if the application does not want portSUPPRESS_TICKS_AND_SLEEP() to be called. */
                    configPRE_SUPPRESS_TICKS_AND_SLEEP_PROCESSING( xExpectedIdleTime );

                    if( xExpectedIdleTime >= ( TickType_t ) configEXPECTED_IDLE_TIME_BEFORE_SLEEP )
                    {
                        traceLOW_POWER_IDLE_BEGIN();
                        portSUPPRESS_TICKS_AND_SLEEP( xExpectedIdleTime );
                        traceLOW_POWER_IDLE_END();
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }
                }
                ( void ) xTaskResumeAll();
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
        #endif /* configUSE_TICKLESS_IDLE */
```

- **`vTaskSuspendAll()`**：鎖定排程器。因為接下來要進行硬體定時器的重新配置，必須確保這期間不會被其他中斷或排程打斷（避免競態條件 Race Condition）。
- **第二次精準評估**：鎖定排程器後，再次讀取精確的 `xExpectedIdleTime`。
- **`portSUPPRESS_TICKS_AND_SLEEP`**：這是真正硬體相關的底層巨集。它會修正硬體計時器的比較暫存器，使其在 `xExpectedIdleTime` 之後才觸發中斷，隨後讓 CPU 進入 Sleep/Stop 模式。當硬體中斷（或外部事件）將 CPU 喚醒後，它還會負責補回因睡眠而跳過的 Tick 數。

#### 74.9 多核心被動空閒 Hook 函式

```C
#if ( ( configNUMBER_OF_CORES > 1 ) && ( configUSE_PASSIVE_IDLE_HOOK == 1 ) )
        {
            /* Call the user defined function from within the idle task.  This
             * allows the application designer to add background functionality
             * without the overhead of a separate task.
             *
             * This hook is intended to manage core activity such as disabling cores that go idle.
             *
             * NOTE: vApplicationPassiveIdleHook() MUST NOT, UNDER ANY CIRCUMSTANCES,
             * CALL A FUNCTION THAT MIGHT BLOCK. */
            vApplicationPassiveIdleHook();
        }
        #endif /* #if ( ( configNUMBER_OF_CORES > 1 ) && ( configUSE_PASSIVE_IDLE_HOOK == 1 ) ) */
    }
}
```

- **`vApplicationPassiveIdleHook()`**：專門用於多核心（SMP）環境。當主核心之外的其他「被動核心（Passive Cores）」也進入空閒狀態時會呼叫此函式。
- **主要目的**：通常用於管理多核心的功耗，例如在背景直接將這個閒置的輔助核心關閉（Power-down）或使其降頻。
- **⚠️ 致命禁忌**：這個 Hook 函式**絕對不能包含任何可能導致阻塞（Block）的 API**（例如 `vTaskDelay` 或等待信號量），否則會導致 FreeRTOS 核心崩潰。

#### 74.10 `configPRE_SUPPRESS_TICKS_AND_SLEEP_PROCESSING` 的用途

這是一個**應用層給的否決權**。

在進入 tickless 睡眠的流程中，Kernel 用 `prvGetExpectedIdleTime()` 算出預期空閒時間，但 Kernel 本身不知道硬體外圍是否正忙碌。這個巨集讓你有機會在最後一刻**把 `xExpectedIdleTime` 歸零**，讓緊接著的條件判斷失敗，從而跳過 `portSUPPRESS_TICKS_AND_SLEEP`：

典型使用場景：

- DMA 傳輸進行中（Kernel 不感知，但你不能讓時脈停）
- UART TX FIFO 還沒排空
- 某個外圍正在等待精確的時序

預設實作是空巨集（什麼都不做），由移植者或應用層自行覆寫。

#### 74.11 `portSUPPRESS_TICKS_AND_SLEEP` 具體做了什麼

這是**真正的低功耗睡眠實作**，移植層（port layer）負責實現，以 ARM Cortex-M 為例，其流程大致如下：

```C
┌─────────────────────────────────────────────────────┐
│  1. 停掉 SysTick（關閉週期性 tick 中斷）              │
│                                                     │
│  2. 用低功耗計時器（如 RTC、LPTIM）設定喚醒時間        │
│     = xExpectedIdleTime 個 tick 後觸發中斷           │
│                                                     │
│  3. 執行 WFI/WFE → CPU 進入 Sleep/Stop/Standby     │
│     （其他中斷也可以提早喚醒）                        │
│                                                     │
│  4. 喚醒後，計算「實際睡了多少個 tick」               │
│                                                     │
│  5. 呼叫 vTaskStepTick(實際經過的tick數)             │
│     → 補償 Kernel 的 tick 計數，讓時間正確           │
│                                                     │
│  6. 重新啟動 SysTick                                │
└─────────────────────────────────────────────────────┘
```

- 進入此函式時 **Scheduler 已被 suspend**，確保沒有任務切換干擾
- 必須處理「提早被其他中斷喚醒」的情況，並正確計算短少的 tick 數
- `vTaskStepTick()` 是核心補償機制，讓睡眠期間流逝的時間對 Kernel 透明可見

#### 74.12 為什麼 `vApplicationPassiveIdleHook` 要在主核心也呼叫？

這需要理解 SMP 下 Idle Task 的架構：
```C
單核 (configNUMBER_OF_CORES == 1)
  └─ 只有一個 Idle Task → 呼叫 vApplicationIdleHook

多核 SMP (configNUMBER_OF_CORES > 1)
  ├─ Core 0 Idle Task → 呼叫 vApplicationIdleHook（系統級）
  │                   → 呼叫 vApplicationPassiveIdleHook（核心級）
  ├─ Core 1 Idle Task → 只呼叫 vApplicationPassiveIdleHook
  └─ Core N Idle Task → 只呼叫 vApplicationPassiveIdleHook
```

|         | `vApplicationIdleHook` | `vApplicationPassiveIdleHook`       |
| ------- | ---------------------- | ----------------------------------- |
| **語意**  | 系統閒置時的全域工作             | 「這顆核心」閒置時的管理                        |
| **例子**  | 背景垃圾回收、統計              | 對該核心進行 clock gating、WFI、關閉 L1 cache |
| **呼叫者** | 僅 Core 0               | 所有核心（含 Core 0）                      |
| **限制**  | 不能 block               | **絕對不能 block**                      |
當 Core 0 本身也進入 Idle 狀態時，它**同樣是一顆閒置的實體核心**，需要進行**per-core 的電源管理**（例如把這顆核心的時脈降頻或 WFI）。這個動作是每個核心自己負責自己的，Passive Hook 恰好提供了這個每核心的入口點。

如果主核心沒有呼叫 Passive Hook，那麼在所有應用任務都在其他核心執行的情況下，Core 0 閒置時就無法自動管理自身的功耗，會白白空轉耗電。

### 75. *eTaskConfirmSleepModeStatus*

這個函式 `eTaskConfirmSleepModeStatus()` 是 FreeRTOS 低功耗機制（Tickless Idle）中的「終極守門員」。

在空閒任務（Idle Task）計算完預期可以睡多久，並鎖定排程器之後、硬體真正進入睡眠之前的**最後一刻**，硬體狀態可能因為中斷而發生改變。這個函式的作用就是**進行最後的安全檢查**，確認現在到底還能不能睡？以及能「睡到什麼程度」？

#### 75.1 函式宣告與環境變數初始化

```C
#if ( configUSE_TICKLESS_IDLE != 0 )

    eSleepModeStatus eTaskConfirmSleepModeStatus( void )
    {
        #if ( INCLUDE_vTaskSuspend == 1 )
            /* The idle task exists in addition to the application tasks. */
            const UBaseType_t uxNonApplicationTasks = configNUMBER_OF_CORES;
        #endif /* INCLUDE_vTaskSuspend */

        eSleepModeStatus eReturn = eStandardSleep;

        traceENTER_eTaskConfirmSleepModeStatus();
```

- **`uxNonApplicationTasks`**：計算系統中「非應用程式」的任務數量。在多核心（SMP）環境下，每個核心都會有一個專屬的空閒任務（Idle Task），所以非應用程式任務的數量就等於核心數（`configNUMBER_OF_CORES`）。
- **`eReturn = eStandardSleep`**：預設傳回狀態。如果接下來的檢查都安全過關，系統就會進入標準的定時睡眠模式（睡到下一個任務準備喚醒的時間點）。

#### 75.2 關鍵安全檢查：一票否決的「終止睡眠」條件

```C
/* This function must be called from a critical section. */

        if( listCURRENT_LIST_LENGTH( &xPendingReadyList ) != 0U )
        {
            /* A task was made ready while the scheduler was suspended. */
            eReturn = eAbortSleep;
        }
        else if( xYieldPendings[ portGET_CORE_ID() ] != pdFALSE )
        {
            /* A yield was pended while the scheduler was suspended. */
            eReturn = eAbortSleep;
        }
        else if( xPendedTicks != 0U )
        {
            /* A tick interrupt has already occurred but was held pending
             * because the scheduler is suspended. */
            eReturn = eAbortSleep;
        }
```

執行此函式前排程器已被鎖定，中斷雖然能觸發，但更新的任務狀態會被暫存起來。如果發生以下三種狀況之一，就必須立刻放棄睡眠（傳回 `eAbortSleep`）：
1. **`xPendingReadyList` 不為空**：代表在排程器鎖定期間，某個硬體中斷（例如收到網路封包、序列埠資料）把某個任務給**喚醒**了，該任務正被放在臨時就緒鏈結串列中。此時絕對不能睡，必須立刻處理它。
2. **`xYieldPendings[...]` 為真**：代表當前核心（`portGET_CORE_ID()`）有尚未執行的讓出 CPU（Yield）請求。可能是有中斷服務要求立刻切換任務，因此不能睡眠。
3. **`xPendedTicks != 0U`**：代表系統定時器（Tick 中斷）在排程器鎖定期間已經觸發過了，但因為排程器被鎖住，這個系統時鐘還沒被加進全域變數。如果這時候睡過去，系統時鐘就會直接漏算、發生嚴重失準。

#### 75.3 進階檢查：是否能進入「無限期深度睡眠」？

```C
#if ( INCLUDE_vTaskSuspend == 1 )
            else if( listCURRENT_LIST_LENGTH( &xSuspendedTaskList ) == ( uxCurrentNumberOfTasks - uxNonApplicationTasks ) )
            {
                /* If all the tasks are in the suspended list (which might mean they
                 * have an infinite block time rather than actually being suspended)
                 * then it is safe to turn all clocks off and just wait for external
                 * interrupts. */
                eReturn = eNoTasksWaitingTimeout;
            }
        #endif /* INCLUDE_vTaskSuspend */
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }

        traceRETURN_eTaskConfirmSleepModeStatus( eReturn );

        return eReturn;
    }

#endif /* configUSE_TICKLESS_IDLE */
```

- **`uxCurrentNumberOfTasks - uxNonApplicationTasks`**：這代表**目前系統中，由使用者建立的「應用程式任務」總數**。
- **邏輯判斷**：如果當前「掛起任務鏈結串列（`xSuspendedTaskList`）」的長度，剛好等於所有的應用程式任務總數，這代表**全天下所有的使用者任務，此時此刻全部都在躺平（掛起或無限期阻塞中，例如等待一個沒有設定超時時間的信號量）**。
- **`eNoTasksWaitingTimeout`（深度睡眠）**：既然沒有任何任務在等待定時器超時，代表接下來的未來的幾分鐘、甚至幾天，**只要沒有外部硬體中斷（如按鈕、感測器觸發），系統就絕對不需要醒來**。此時，FreeRTOS 會允許硬體關閉包含定時器在內的所有內部時脈（Clock Gating），進入省電效率極高的深睡眠，完全只靠外部中斷來喚醒。

#### 75.4 傳回值決策總結表

| **傳回狀態值**                    | **意義**  | **系統後續動作**                                       |
| ---------------------------- | ------- | ------------------------------------------------ |
| **`eAbortSleep`**            | 終止睡眠    | 有突發事件或時鐘更新，立刻退出低功耗流程，恢復正常排程。                     |
| **`eStandardSleep`**         | 標準低功耗睡眠 | 設定定時器在 $X$ 毫秒後喚醒 CPU（$X$ = 下一個任務要醒來的時間）。         |
| **`eNoTasksWaitingTimeout`** | 無限期深度睡眠 | 關閉系統滴答定時器（SysTick），徹底躺平，純粹等待外部硬體中斷（如 GPIO 觸發）喚醒。 |

### 76. *vTaskSetThreadLocalStoragePointer*

`vTaskSetThreadLocalStoragePointer()` 是 FreeRTOS 用來實現 **執行緒區域儲存區（Thread Local Storage, TLS）** 的核心 API。

在多工環境下，全域變數是所有任務共用的。但有時候，我們希望某個變數在**每個任務中都有獨立的獨立副本**（例如 C 標準庫中的 `errno`，或是每個任務專屬的用戶設定）。TLS 就是在任務的 TCB（任務控制區塊）中開闢一個指標陣列，讓每個任務可以掛載自己專屬的資料結構。

#### 76.1 條件編譯與函數簽章

```C
#if ( configNUM_THREAD_LOCAL_STORAGE_POINTERS != 0 )

    void vTaskSetThreadLocalStoragePointer( TaskHandle_t xTaskToSet,
                                            BaseType_t xIndex,
                                            void * pvValue )
    {
        TCB_t * pxTCB;

        traceENTER_vTaskSetThreadLocalStoragePointer( xTaskToSet, xIndex, pvValue );
```

- **`configNUM_THREAD_LOCAL_STORAGE_POINTERS`**：這是在 `FreeRTOSConfig.h` 中設定的常數。如果設為 `0`，代表關閉 TLS 功能，整個函數就不會被編譯，以節省 Flash 空間。

- **參數說明**：
	- **`xTaskToSet`**：目標任務的控制代碼（Handle）。如果傳入 `NULL`，代表要設定**目前正在執行**的任務。
	- **`xIndex`**：陣列索引。指定要把指標存放在 TLS 陣列的哪一個格子。
	- **`pvValue`**：要儲存的指標。這通常指向一塊由 `pvPortMalloc()` 動態配置出來的自訂結構體記憶體。

#### 76.2 邊界檢查與獲取任務控制區塊 (TCB)

```C
if( ( xIndex >= 0 ) &&
            ( xIndex < ( BaseType_t ) configNUM_THREAD_LOCAL_STORAGE_POINTERS ) )
        {
            pxTCB = prvGetTCBFromHandle( xTaskToSet );
            configASSERT( pxTCB != NULL );
```

- **`if( ( xIndex >= 0 ) && ... )`**：進行嚴格的陣列邊界檢查。傳入的索引值必須大於等於 0，且小於系統配置的 TLS 最大長度，防止記憶體越界寫入（Buffer Overflow）。
- **`prvGetTCBFromHandle( xTaskToSet )`**：這是 FreeRTOS 內部的隱藏函數。它會把使用者傳入的 `TaskHandle_t` 轉換成內部真正的 `TCB_t`（任務控制區塊）結構體指標。如果傳入 `NULL`，它會貼心地自動返回目前運行中任務的 TCB。
- **`configASSERT( pxTCB != NULL )`**：斷言確保獲取的 TCB 指標有效。如果此時系統抓到空指標，程式會在這裡攔截並報錯。

#### 76.3 指標賦值與函數返回

```C
pxTCB->pvThreadLocalStoragePointers[ xIndex ] = pvValue;
        }

        traceRETURN_vTaskSetThreadLocalStoragePointer();
    }

#endif /* configNUM_THREAD_LOCAL_STORAGE_POINTERS */
```

- **`pxTCB->pvThreadLocalStoragePointers[ xIndex ] = pvValue;`（核心操作）**： 每個任務的 TCB 裡面都藏有一個結構體成員叫做 `pvThreadLocalStoragePointers`（本質上是一個 `void *` 的陣列）。這行程式碼就是把你的自訂資料指標 `pvValue`，精準地塞進該任務專屬的陣列格子裡。
- 當作業系統在進行任務切換（Context Switch）時，新任務會帶來它自己的 TCB，因此你調用 `vTaskGetThreadLocalStoragePointer()` 讀取同一個 `xIndex` 時，拿到的就會是當前任務的資料，從而實現了「執行緒隔離」。

#### 76.4 典型使用情境

```C
// 定義一個任務專屬的設定結構體
typedef struct {
    int xTaskID;
    char cTaskName[10];
} TaskConfig_t;

void vVariablesTask( void * pvParameters ) {
    // 1. 為當前任務配置專屬的記憶體空間
    TaskConfig_t * pxMyConfig = (TaskConfig_t *) pvPortMalloc( sizeof( TaskConfig_t ) );
    pxMyConfig->xTaskID = 1;
    
    // 2. 將這塊記憶體掛載到 TLS 的第 0 號格子
    vTaskSetThreadLocalStoragePointer( NULL, 0, (void *) pxMyConfig );

    for( ;; ) {
        // 3. 之後在任何地方，都可以透過 Get 函數把這塊專屬記憶體拿出來用
        TaskConfig_t * pxSavedConfig = (TaskConfig_t *) vTaskGetThreadLocalStoragePointer( NULL, 0 );
        // 執行任務邏輯...
    }
}
```

### 77. *pvTaskGetThreadLocalStoragePointer*

跟 [[#76. vTaskSetThreadLocalStoragePointer]] 只是是 get，如果 `pxTCB->pvThreadLocalStoragePointers[ xIndex ]`存在返回對應的 pointer，反之返回 NULL

```C
#if ( configNUM_THREAD_LOCAL_STORAGE_POINTERS != 0 )

  

void * pvTaskGetThreadLocalStoragePointer( TaskHandle_t xTaskToQuery,

BaseType_t xIndex )

{

void * pvReturn = NULL;

TCB_t * pxTCB;

  

traceENTER_pvTaskGetThreadLocalStoragePointer( xTaskToQuery, xIndex );

  

if( ( xIndex >= 0 ) &&

( xIndex < ( BaseType_t ) configNUM_THREAD_LOCAL_STORAGE_POINTERS ) )

{

pxTCB = prvGetTCBFromHandle( xTaskToQuery );

configASSERT( pxTCB != NULL );

  

pvReturn = pxTCB->pvThreadLocalStoragePointers[ xIndex ];

}

else

{

pvReturn = NULL;

}

  

traceRETURN_pvTaskGetThreadLocalStoragePointer( pvReturn );

  

return pvReturn;

}

  

#endif /* configNUM_THREAD_LOCAL_STORAGE_POINTERS */
```
### 78. *vTaskAllocateMPURegions*

`vTaskAllocateMPURegions()` 是 FreeRTOS 在啟用 **MPU（Memory Protection Unit，記憶體保護單元）** 核心時非常關鍵的進階 API。

在安全要求極高的嵌入式系統中（例如車用、醫療、航太），我們會讓應用程式任務運行在非特權模式（Unprivileged Mode）下。這樣一來，該任務就只能存取被授權的記憶體區塊。如果它意圖去讀寫其他任務的堆疊或核心暫存器，硬體 MPU 就會立刻觸發異常（Memory Management Fault）並將其攔截，防止整個系統崩潰。

而 `vTaskAllocateMPURegions()` 的存在，就是為了讓你在任務建立之後，**動態地為該任務重新配置或修改專屬的 MPU 記憶體保護區域**。

#### 78.1 條件編譯與函數宣告

```C
#if ( portUSING_MPU_WRAPPERS == 1 )

    void vTaskAllocateMPURegions( TaskHandle_t xTaskToModify,
                                  const MemoryRegion_t * const pxRegions )
    {
        TCB_t * pxTCB;

        traceENTER_vTaskAllocateMPURegions( xTaskToModify, pxRegions );
```

- **`portUSING_MPU_WRAPPERS == 1`**：這是一個安全防護開關。只有當你在 `FreeRTOSConfig.h` 中開啟了 MPU 支援，這段程式碼才會被編譯。否則，標準的 FreeRTOS 是不包含 MPU 包裝層的。

- **參數說明**：
	- **`xTaskToModify`**：想要修改 MPU 設定的目標任務控制代碼（Handle）。
	- **`pxRegions`**：指向一個 `MemoryRegion_t` 結構體陣列的指標。這個陣列定義了新的記憶體邊界（起始地址、大小、存取權限如唯讀或讀寫）。

#### 78.2 寫入硬體暫存器配置（核心操作）

```C
vPortStoreTaskMPUSettings( &( pxTCB->xMPUSettings ), pxRegions, NULL, 0 );

        traceRETURN_vTaskAllocateMPURegions();
    }

#endif /* portUSING_MPU_WRAPPERS */
```

- **`vPortStoreTaskMPUSettings`（最核心）**：這是一個與晶片架構高度相關（Port-specific）的底層函數（通常實作在 `port.c` 中，例如 ARM Cortex-M4/M7 MPU 專屬程式碼）。
- **運作機制**：它會把使用者在 `pxRegions` 中定義的記憶體配置，**精準地寫入該任務 TCB 內的 `xMPUSettings` 結構體中**。
- **什麼時候生效？**：請注意，這行程式碼執行完時，硬體 MPU 暫存器不一定會立刻改變。當 FreeRTOS 進行 **任務切換（Context Switch）** 時，排程器會把即將登台運作的任務之 `xMPUSettings` 倒進 CPU 的硬體 MPU 暫存器中。這樣就能確保每個任務在執行的那一瞬間，都擁有完全隔離且正確的記憶體防護罩。

### 79. *prvInitialiseTaskLists*

`prvInitialiseTaskLists()` 是 FreeRTOS 排程器核心中的**地基中的地基**。

在 FreeRTOS 中，任務（Task）的本質是被放在不同的鏈結串列（Linked List）中管理。不論是準備執行的任務、正在延時（Sleep）的任務，還是被掛起（Suspend）的任務，都有專屬的「大帳本」來記錄。這個函數的作用，就是在排程器啟動前，**把這些全域的鏈結串列全部初始化清空**，準備讓任務進駐。

### 80. *prvCheckTasksWaitingTermination*

`prvCheckTasksWaitingTermination()` 在 FreeRTOS 中扮演著「記憶體清道夫（或稱回收站大總管）」的角色。

當你在程式中呼叫 `vTaskDelete()` 刪除一個任務時，如果該任務是「自己刪除自己（自殺）」，它不能在當下立刻把自己佔用的記憶體（TCB 和 Stack）釋放掉，因為它自己此時此刻還運行在該堆疊上！

因此，FreeRTOS 會把死掉的任務丟進一個名為 `xTasksWaitingTermination` 的「墓園串列」，而**真正的屍體清理（記憶體釋放）工作，就是由這個只在空閒任務（Idle Task）中運行的 `prvCheckTasksWaitingTermination()` 偷偷在背景完成的。**

#### 80.1 條件編譯與效能守門員（外層迴圈）

```C
#if ( INCLUDE_vTaskDelete == 1 )
    {
        TCB_t * pxTCB;

        /* uxDeletedTasksWaitingCleanUp is used to prevent taskENTER_CRITICAL()
         * being called too often in the idle task. */
        while( uxDeletedTasksWaitingCleanUp > ( UBaseType_t ) 0U )
        {
```

- **`INCLUDE_vTaskDelete == 1`**：唯有在 `FreeRTOSConfig.h` 中開啟了刪除功能，這段記憶體回收機制才會被編譯。
- **`uxDeletedTasksWaitingCleanUp`（效能關鍵計數器）**：這是一個全域變數，記錄了「目前有幾個任務死掉等著被收屍」。
- **為什麼要用 `while` 檢查它？**：空閒任務是一個死迴圈，隨時都在瘋狂運轉。如果沒有這個計數器保護，空閒任務每次運轉都要頻繁地進入臨界區（Critical Section）去查看墓園串列是不是空的。這會導致極大的 CPU 效能浪費。透過這個計數器，只要大於 0，代表真的有任務死掉，才需要進去處理。

#### 80.2 單核心環境的清理邏輯 (Single-Core Logic)

```C
#if ( configNUMBER_OF_CORES == 1 )
            {
                taskENTER_CRITICAL();
                {
                    /* 從刪除串列的頭部獲取 TCB 指標 */
                    pxTCB = listGET_OWNER_OF_HEAD_ENTRY( ( &xTasksWaitingTermination ) );
                    /* 將該任務徹底移出串列 */
                    ( void ) uxListRemove( &( pxTCB->xStateListItem ) );
                    /* 更新系統總任務數與待清理計數器 */
                    --uxCurrentNumberOfTasks;
                    --uxDeletedTasksWaitingCleanUp;
                }
                taskEXIT_CRITICAL();

                /* 關鍵：在臨界區之外釋放記憶體 */
                prvDeleteTCB( pxTCB );
            }
```

- **臨界區最小化**：注意看！FreeRTOS 只在臨界區（`taskENTER_CRITICAL`）內做簡單的「鏈結串列移出」與「計數器相減」等極快的操作。
- **為什麼 `prvDeleteTCB( pxTCB )` 要放在臨界區外面？**：因為釋放記憶體（呼叫 `pvPortFree()`）是一個相對耗時的硬體操作。如果在關閉中斷的臨界區內釋放記憶體，會大幅增加系統的中斷延遲（Interrupt Latency）。既然單核環境下只要把指針移出串列就安全了，那真正釋放記憶體的髒活就可以移到臨界區外安全地執行。

#### 80.3 多核心環境的清理邏輯 (Multi-Core / SMP Logic)

多核心環境下，收屍工作變得異常極端且危險，因為**多個核心的 Idle Task 可能同時在跑這段代碼**，或者目標核心還來不及切換掉該任務。

```C
#else /* #if( configNUMBER_OF_CORES == 1 ) */
            {
                pxTCB = NULL;

                taskENTER_CRITICAL();
                {
                    /* 雙重檢查鎖（Double-Check）：可能在等待進入臨界區時，別的核心已經把屍體收走了 */
                    if( uxDeletedTasksWaitingCleanUp > ( UBaseType_t ) 0U )
                    {
                        pxTCB = listGET_OWNER_OF_HEAD_ENTRY( ( &xTasksWaitingTermination ) );

                        /* 核心安全檢查：確保這個被刪除的任務，目前沒有在任何其他核心上運行 */
                        if( pxTCB->xTaskRunState == taskTASK_NOT_RUNNING )
                        {
                            ( void ) uxListRemove( &( pxTCB->xStateListItem ) );
                            --uxCurrentNumberOfTasks;
                            --uxDeletedTasksWaitingCleanUp;
                        }
                        else
                        {
                            /* 該任務雖然被刪除了，但另一個核心此時還在對它進行最後的 Context Switch 切換
                             * 我們必須立刻放棄，退出臨界區，等下一次空閒時再來嘗試。 */
                            taskEXIT_CRITICAL();
                            break;
                        }
                    }
                }
                taskEXIT_CRITICAL();

                /* 如果成功搶到屍體，且狀態安全，才在臨界區外釋放記憶體 */
                if( pxTCB != NULL )
                {
                    prvDeleteTCB( pxTCB );
                }
            }
            #endif /* #if( configNUMBER_OF_CORES == 1 ) */
```

#### 80.3.1 多核架構下的兩大巨坑

在多核心（SMP）環境下，這段程式碼處理了兩個非常精妙的**競態條件（Race Condition）**：

- **搶屍體（搶先體驗）**：Core 0 和 Core 1 此時都沒事做，都掉進了 Idle Task。當 Core 0 拿到了臨界區鎖（Spinlock）進去收屍時，Core 1 只能在外面排隊。等 Core 0 收完屍體出來、輪到 Core 1 進去時，計數器已經變成 0 了。所以程式碼進去後必須**再判斷一次 `if( uxDeletedTasksWaitingCleanUp > 0 )`**，否則 Core 1 會抓到空指針導致系統當機。
- **靈魂還在漂（`taskTASK_NOT_RUNNING`）**：假設你在 Core 0 呼叫 `vTaskDelete(TaskA)`，而 `TaskA` 當時正在 Core 1 上全速運轉。雖然 Core 0 把 `TaskA` 塞進了墓園串列，但 Core 1 可能需要花費幾個微秒的時間來觸發中斷、把 `TaskA` 切換下來。如果此時 Core 0 的 Idle Task 猴急地想要去釋放 `TaskA` 的記憶體，Core 1 就會直接對著已經踩空的記憶體崩潰。因此必須檢查 `pxTCB->xTaskRunState == taskTASK_NOT_RUNNING`，確認全天下都沒有核心在跑它了，才允許收屍。

| **特性**       | **單核心機制 (configNUMBER_OF_CORES == 1)** | **多核心機制 (SMP 架構)**                     |
| ------------ | -------------------------------------- | -------------------------------------- |
| **臨界區防護**    | 僅防止本地中斷干擾                              | 使用 Multicore Spinlock 防止其他核心同時搶奪       |
| **雙重檢查**     | 不需要（串列狀態此時是絕對靜止的）                      | **極度需要**，防止與其他核心的 Idle Task 衝突         |
| **任務運行狀態檢查** | 不需要（既然我都進 Idle 了，代表沒別人在跑）              | **必須檢查 `xTaskRunState`**，防止其他核心還在執行該任務 |
| **記憶體釋放時機**  | 為了縮短中斷關閉時間，一律在臨界區外釋放                   | 只有在成功剔除且狀態安全時，才在臨界區外釋放                 |
### 81. *vTaskGetInfo*

`vTaskGetInfo()` 是 FreeRTOS 的「全面健檢診斷儀」。

當你想寫一個系統監控任務（比如透過 UART 印出當前所有 Task 的狀態、CPU 使用率、剩餘堆疊），或者在使用除錯工具（如 FreeRTOS SystemView）時，這個函數就是核心靈魂。它負責把隱藏在核心內部的 `TCB_t`（任務控制區塊）私有變數，打包成一個公開的 `TaskStatus_t` 結構體供外界讀取。

#### 81.1 條件編譯與獲取任務控制區塊 (TCB)

```C
#if ( configUSE_TRACE_FACILITY == 1 )

    void vTaskGetInfo( TaskHandle_t xTask,
                       TaskStatus_t * pxTaskStatus,
                       BaseType_t xGetFreeStackSpace,
                       eTaskState eState )
    {
        TCB_t * pxTCB;

        traceENTER_vTaskGetInfo( xTask, pxTaskStatus, xGetFreeStackSpace, eState );

        /* 如果傳入 NULL，代表使用者想要查詢「目前正在執行此代碼」的任務狀態 */
        pxTCB = prvGetTCBFromHandle( xTask );
        configASSERT( pxTCB != NULL );
```

- **`configUSE_TRACE_FACILITY`**：系統診斷開關。必須在 `FreeRTOSConfig.h` 中將其設為 `1` 才能啟用此函數，否則不編譯以省空間。
- **`eState` 參數的妙用**：這是一個優化核心效能的設計。如果你在呼叫此函數前，已經透過其他方式知道了任務狀態，可以直接傳進來；如果不知道，傳入 `eInvalid`，核心就會花額外的時間幫你現場計算。

#### 81.2 複製基礎元數據 (Basic Metadata)

```C
/* 開始將 TCB 內部的私有資料填入使用者的 TaskStatus_t 結構體中 */
        pxTaskStatus->xHandle = pxTCB;                                           // 任務控制代碼
        pxTaskStatus->pcTaskName = ( const char * ) &( pxTCB->pcTaskName[ 0 ] ); // 任務名稱
        pxTaskStatus->uxCurrentPriority = pxTCB->uxPriority;                     // 當前優先級
        pxTaskStatus->pxStackBase = pxTCB->pxStack;                              // 堆疊起始地址
        
        /* 根據硬體堆疊增長方向，紀錄堆疊邊界 */
        #if ( ( portSTACK_GROWTH > 0 ) || ( configRECORD_STACK_HIGH_ADDRESS == 1 ) )
            pxTaskStatus->pxTopOfStack = ( StackType_t * ) pxTCB->pxTopOfStack;
            pxTaskStatus->pxEndOfStack = pxTCB->pxEndOfStack;
        #endif
        
        pxTaskStatus->xTaskNumber = pxTCB->uxTCBNumber;                         // 任務的專屬 ID
```

- **`uxCurrentPriority`**：注意它是紀錄「當前」優先級。如果任務因為拿了 Mutex 發生了優先級繼承（Priority Inheritance），這裡拿到的會是臨時被拉高的優先級。
- **`uxTCBNumber`**：每個任務被建立時，核心都會給它一個遞增的流水號，這對於除錯工具做追蹤非常方便。

#### 81.3 處理進階條件編譯功能 (SMP 與 運行統計)

```C
/* 多核心（SMP）親和性設定 */
        #if ( ( configUSE_CORE_AFFINITY == 1 ) && ( configNUMBER_OF_CORES > 1 ) )
        {
            pxTaskStatus->uxCoreAffinityMask = pxTCB->uxCoreAffinityMask; // 綁定在哪個核心跑
        }
        #endif

        /* 紀錄原始優先級（處理 Mutex 繼承前的真實優先級） */
        #if ( configUSE_MUTEXES == 1 )
        {
            pxTaskStatus->uxBasePriority = pxTCB->uxBasePriority;
        }
        #else
        {
            pxTaskStatus->uxBasePriority = 0;
        }
        #endif

        /* 統計該任務總共壓榨了 CPU 多少運行時間 */
        #if ( configGENERATE_RUN_TIME_STATS == 1 )
        {
            pxTaskStatus->ulRunTimeCounter = ulTaskGetRunTimeCounter( xTask );
        }
        #else
        {
            pxTaskStatus->ulRunTimeCounter = ( configRUN_TIME_COUNTER_TYPE ) 0;
        }
        #endif
```

- **`uxBasePriority`**：如果任務因為 Mutex 被拉高了優先級，這個欄位可以幫你查出它「原本的」真實優先級是多少。
- **`ulRunTimeCounter`**：這是計算 CPU 使用率（如 `vTaskList()` 輸出的百分比）的最核心指標。它紀錄了這個任務從系統開機到現在，總共佔用了 CPU 多少時間。

#### 81.4 判斷任務當前狀態 (The Fiddly State Machine)

```C
/* 判斷狀態：正如官方註解所說，獲取任務狀態是一件很 fiddly (棘手/繁瑣) 的事 */
        if( eState != eInvalid )
        {
            /* 如果使用者有傳入預設狀態，且該任務目前正在核心上跑 */
            if( taskTASK_IS_RUNNING( pxTCB ) == pdTRUE )
            {
                pxTaskStatus->eCurrentState = eRunning;
            }
            else
            {
                pxTaskStatus->eCurrentState = eState;

                #if ( INCLUDE_vTaskSuspend == 1 )
                {
                    /* 歷史巨坑：在 FreeRTOS 中，如果一個任務設定了「無限期等待（portMAX_DELAY）」
                     * 它其實會被核心丟進 SuspendedTaskList。但從語意上看，它應該屬於 Blocked（阻塞）狀態！ */
                    if( eState == eSuspended )
                    {
                        vTaskSuspendAll(); // 鎖定排程器，開始清查
                        {
                            /* 如果它掛在某個事件串列（如 Queue/Semaphore）上，代表它其實是在 Block 等東西 */
                            if( listLIST_ITEM_CONTAINER( &( pxTCB->xEventListItem ) ) != NULL )
                            {
                                pxTaskStatus->eCurrentState = eBlocked;
                            }
                            else
                            {
                                #if ( configUSE_TASK_NOTIFICATIONS == 1 )
                                {
                                    BaseType_t x;

                                    /* 檢查它是不是在 Blocked 等待任務通知（Task Notification） */
                                    for( x = ( BaseType_t ) 0; x < ( BaseType_t ) configTASK_NOTIFICATION_ARRAY_ENTRIES; x++ )
                                    {
                                        if( pxTCB->ucNotifyState[ x ] == taskWAITING_NOTIFICATION )
                                        {
                                            pxTaskStatus->eCurrentState = eBlocked;
                                            break;
                                        }
                                    }
                                }
                                #endif /* configUSE_TASK_NOTIFICATIONS */
                            }
                        }
                        ( void ) xTaskResumeAll();
                    }
                }
                #endif /* INCLUDE_vTaskSuspend */

                /* 特殊狀態檢查：如果任務同時出現在 xPendingReadyList 中，代表它已經準備好要跑了
                 * 不管它此時人在哪個狀態串列，一律正名為 Ready 狀態 */
                taskENTER_CRITICAL();
                {
                    if( listIS_CONTAINED_WITHIN( &xPendingReadyList, &( pxTCB->xEventListItem ) ) != pdFALSE )
                    {
                        pxTaskStatus->eCurrentState = eReady;
                    }
                }
                taskEXIT_CRITICAL();
            }
        }
        else
        {
            /* 如果使用者偷懶傳入 eInvalid，那就調用常規函數現場查表計算 */
            pxTaskStatus->eCurrentState = eTaskGetState( pxTCB );
        }
```

- **「掛羊頭賣狗肉」的 `eSuspended`**：核心為了最佳化，會把 `vTaskDelay(portMAX_DELAY)` 的任務和真正的 `vTaskSuspend()` 任務放在同一個鏈結串列（`xSuspendedTaskList`）中。這段代碼透過檢查 `xEventListItem` 和 `ucNotifyState`，完美地把「真正被凍結（Suspended）」與「只是在無限期等東西（Blocked）」的任務區分開來。

#### 81.5 計算堆疊高水位線 (Stack High Water Mark)

```C
/* 獲取剩餘堆疊空間：這是一個極度耗時的動作，所以可以透過 xGetFreeStackSpace 來跳過 */
        if( xGetFreeStackSpace != pdFALSE )
        {
            /* prvTaskCheckFreeStackSpace 會從堆疊底端開始往上掃描，
             * 計算有多少個位元組依然保持著當初初始化的魔術數字（0xa5） */
            #if ( portSTACK_GROWTH > 0 )
            {
                pxTaskStatus->usStackHighWaterMark = prvTaskCheckFreeStackSpace( ( uint8_t * ) pxTCB->pxEndOfStack );
            }
            #else
            {
                pxTaskStatus->usStackHighWaterMark = prvTaskCheckFreeStackSpace( ( uint8_t * ) pxTCB->pxStack );
            }
            #endif
        }
        else
        {
            pxTaskStatus->usStackHighWaterMark = 0; // 不查詢，直接回傳 0 以求極速
        }

        traceRETURN_vTaskGetInfo();
    }

#endif /* configUSE_TRACE_FACILITY */
```

- **效能殺手（`usStackHighWaterMark`）**：要計算堆疊到底還剩多少空間，CPU 必須用一個迴圈把整塊堆疊記憶體**一條一條讀出來**，去數有幾個 `0xa5`（未使用的標記）。這會消耗非常多的 CPU 週期。
- **開發建議**：在正式產品釋出（Release）時，若需要調用此函數，務必將 `xGetFreeStackSpace` 設為 `pdFALSE`，以免嚴重拖慢即時系統的反應速度。

### 82. *prvListTasksWithinSingleList*

`prvListTasksWithinSingleList()` 是 FreeRTOS 系統診斷功能的「前線常務工人」。

當你呼叫高階 API（例如 `uxTaskGetSystemState()`）來獲取全系統所有任務的健康狀態時，核心需要去掃描各式各樣的鏈結串列（就緒串列、延時串列、掛起串列等）。

因為每個串列的遍歷邏輯完全一樣，FreeRTOS 的設計者就把這段重複的「點名」邏輯抽出來，寫成了這個輔助函數。它的工作很純粹：**傳入一個鏈結串列，它就把裡面所有的任務通通抓出來，逐一做完健檢（呼叫 `vTaskGetInfo`），並排好隊塞進你的輸出陣列中。**

#### 82.1 條件編譯、變數宣告與安全哨兵

```C
#if ( configUSE_TRACE_FACILITY == 1 )

    static UBaseType_t prvListTasksWithinSingleList( TaskStatus_t * pxTaskStatusArray,
                                                     List_t * pxList,
                                                     eTaskState eState )
    {
        UBaseType_t uxTask = 0;
        const ListItem_t * pxEndMarker = listGET_END_MARKER( pxList );
        ListItem_t * pxIterator;
        TCB_t * pxTCB = NULL;
```

- **`configUSE_TRACE_FACILITY`**：系統診斷開關。此函數依賴此巨集，必須在 `FreeRTOSConfig.h` 開啟。
- **`pxEndMarker`（終點標記）**：FreeRTOS 的鏈結串列（List）是雙向循環串列，它的末尾有一個固定的迷你節點（MiniListItem）作為邊界標記。先把它拿出來，是為了後面迴圈知道什麼時候該「煞車」，防止陷入無窮迴圈。
- **`uxTask`**：這是一個區域計數器，紀錄「在這個特定的串列裡，總共找到了幾個任務」，同時它也是寫入目標陣列的索引值（Index）。

#### 82.2 串列長度檢查與遍歷初始化

```C
if( listCURRENT_LIST_LENGTH( pxList ) > ( UBaseType_t ) 0 )
        {
            /* Populate an TaskStatus_t structure within the
             * pxTaskStatusArray array for each task that is referenced from
             * pxList.  See the definition of TaskStatus_t in task.h for the
             * meaning of each TaskStatus_t structure member. */
            for( pxIterator = listGET_HEAD_ENTRY( pxList ); pxIterator != pxEndMarker; pxIterator = listGET_NEXT( pxIterator ) )
            {
```

#### 82.3 核心操作：解鎖 TCB 靈魂與現場健檢

```C
/* MISRA Ref 11.5.3 [Void pointer assignment] */
                /* More details at: https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/MISRA.md#rule-115 */
                /* coverity[misra_c_2012_rule_11_5_violation] */
                pxTCB = listGET_LIST_ITEM_OWNER( pxIterator );

                vTaskGetInfo( ( TaskHandle_t ) pxTCB, &( pxTaskStatusArray[ uxTask ] ), pdTRUE, eState );
                uxTask++;
            }
        }
```

### 83. *prvTaskCheckFreeStackSpace*

`prvTaskCheckFreeStackSpace()` 是 FreeRTOS 內部用來計算「堆疊高水位線（Stack High Water Mark）」的底層核心演算法。

它的主要任務是分析特定任務自啟動以來，**未曾被使用過的最小堆疊空間大小**。FreeRTOS 在建立任務時，會將分配的堆疊記憶體全部填入一個特定的魔術數字 `tskSTACK_FILL_BYTE`（通常為 `0xa5`）。此函數藉由從記憶體邊界開始檢索，統計尚未被修改的 `0xa5` 位元組數量，進而推算出安全的堆疊剩餘空間。

#### 83.1 條件編譯與函數宣告

```C
#if ( ( configUSE_TRACE_FACILITY == 1 ) || ( INCLUDE_uxTaskGetStackHighWaterMark == 1 ) || ( INCLUDE_uxTaskGetStackHighWaterMark2 == 1 ) )

    static configSTACK_DEPTH_TYPE prvTaskCheckFreeStackSpace( const uint8_t * pucStackByte )
    {
        configSTACK_DEPTH_TYPE uxCount = 0U;
```

- **條件編譯控制**：此靜態函數僅在啟用系統追蹤功能（`configUSE_TRACE_FACILITY`）或任一獲取堆疊高水位線的 API（`INCLUDE_uxTaskGetStackHighWaterMark` / `INCLUDE_uxTaskGetStackHighWaterMark2`）時才會進行編譯，以優化未啟用此功能系統的 Flash 空間。
- **參數 `pucStackByte`**：此指標指向該任務堆疊空間的起始檢索邊界。核心會根據晶片架構的堆疊增長方向，傳入對應的記憶體邊界位址（高位址或低位址）。
- **計數器 `uxCount`**：用於累加連續未被修改的填滿位元組（Byte）數量。

#### 83.2 記憶體檢索與架構抽象化

```C
while( *pucStackByte == ( uint8_t ) tskSTACK_FILL_BYTE )
        {
            pucStackByte -= portSTACK_GROWTH;
            uxCount++;
        }
```

- **`tskSTACK_FILL_BYTE`**：數值定義為 `0xa5`（二進位 `10100101`）。此數值在隨機存取記憶體（RAM）中不易自然生成，適合用來識別未經修改的初始狀態。
- **指標運算與硬體抽象（關鍵設計）**： 核心透過 `pucStackByte -= portSTACK_GROWTH` 實現了跨平台的硬體抽象。不同的微處理器架構，其堆疊增長方向不同：
	- **向下增長（Downward Growth，例如 ARM Cortex-M）**：`portSTACK_GROWTH` 定義為 `-1`。運算式等同於 `pucStackByte -= (-1)`，即指標**遞增**（從低位址往高位址檢索）。
	- **向上增長（Upward Growth）**：`portSTACK_GROWTH` 定義為 `1`。運算式等同於 `pucStackByte -= (1)`，即指標**遞減**（從高位址往低位址檢索）。
  此設計允許單一程式碼邏輯在編譯時期直接適應不同的硬體架構，免除運行時期的條件判斷。
- **結束條件**：一旦檢索到任一位元組不等於 `0xa5`，即代表該記憶體位址曾被任務的函式呼叫或區域變數寫入過，迴圈會立即終止。

#### 83.3 單位轉換與數值回傳

```C
uxCount /= ( configSTACK_DEPTH_TYPE ) sizeof( StackType_t );

        return uxCount;
    }

#endif /* ( ( configUSE_TRACE_FACILITY == 1 ) || ( INCLUDE_uxTaskGetStackHighWaterMark == 1 ) || ( INCLUDE_uxTaskGetStackHighWaterMark2 == 1 ) ) */
```

- **單位轉換**：迴圈內累加的 `uxCount` 是以 **Byte（位元組）** 為單位。然而，FreeRTOS 的堆疊配置與計量單位通常是以 **Word（字組）** 為基礎。因此，必須除以 `sizeof(StackType_t)` 進行單位轉換：
	- 在 32 位元架構下，`sizeof(StackType_t)` 為 4。
	- `sizeof(StackType_t)` 為 2。
- **回傳值**：最終返回符合 `configSTACK_DEPTH_TYPE` 型態的數值，代表目前歷史未使用的最小堆疊字組數。

|**項目**|**堆疊向下增長架構 (如 ARM Cortex-M)**|**堆疊向上增長架構**|
|---|---|---|
|**`portSTACK_GROWTH` 巨集值**|`-1`|`1`|
|**`pucStackByte` 的起始配置點**|`pxTCB->pxStack`（配置區域的最低位址）|`pxTCB->pxEndOfStack`（配置區域的最高位址）|
|**指標移動方向**|位址遞增（`pucStackByte++`）|位址遞減（`pucStackByte--`）|
|**回傳數值代表意義**|自初始化至最極端使用狀態下，未被寫入的剩餘堆疊 Word 數。||

### 84. *uxTaskGetStackHighWaterMark2*

這個函數 `uxTaskGetStackHighWaterMark2()` 是 FreeRTOS 用來獲取特定任務「堆疊高水位線（Stack High Water Mark）」的公開 API。它返回的是該任務自建立以來，**歷史未使用的最小堆疊空間大小**（單位為字組 Word）。

此函數的設計目的是 為了解決早期版本 `uxTaskGetStackHighWaterMark()` 在 8 位元或 16 位元微控制器架構下，堆疊深度數值可能發生型態溢位（Overflow）的問題。透過引進 `configSTACK_DEPTH_TYPE` 自訂型態，可在不破壞舊版本相容性的前提下，讓開發者自行定義更寬的變數型態（如 `uint32_t`）。

#### 84.1 條件編譯與函數宣告

```C
#if ( INCLUDE_uxTaskGetStackHighWaterMark2 == 1 )

/* uxTaskGetStackHighWaterMark() and uxTaskGetStackHighWaterMark2() are the
 * same except for their return type.  Using configSTACK_DEPTH_TYPE allows the
 * user to determine the return type.  It gets around the problem of the value
 * overflowing on 8-bit types without breaking backward compatibility for
 * applications that expect an 8-bit return type. */
    configSTACK_DEPTH_TYPE uxTaskGetStackHighWaterMark2( TaskHandle_t xTask )
    {
        TCB_t * pxTCB;
        uint8_t * pucEndOfStack;
        configSTACK_DEPTH_TYPE uxReturn;

        traceENTER_uxTaskGetStackHighWaterMark2( xTask );
```

- **功能開關（`INCLUDE_uxTaskGetStackHighWaterMark2`）**：必須在 `FreeRTOSConfig.h` 中將此巨集定義為 `1`，編譯器才會將此函數納入編譯，用以節省未啟用此功能的系統空間。
- **自訂回傳型態（`configSTACK_DEPTH_TYPE`）**：這是此「Version 2」API 的核心改良點。官方註解明確指出，透過此配置型態，開發者可以自由決定回傳值長度（例如在 32 位元系統上配置為 `uint32_t`），避免 8 位元型態在面對大容量堆疊時發生溢位。
- **參數 `xTask`**：目標任務的控制代碼（Handle）。若傳入 `NULL`，表示查詢當前正在執行此程式碼的任務。

#### 84.2 依據硬體架構決定檢索起點

```C
#if portSTACK_GROWTH < 0
        {
            pucEndOfStack = ( uint8_t * ) pxTCB->pxStack;
        }
        #else
        {
            pucEndOfStack = ( uint8_t * ) pxTCB->pxEndOfStack;
        }
        #endif
```

此處的條件編譯是用來決定**核心檢測未用記憶體時的「最末端邊界位址」**。

- **`portSTACK_GROWTH < 0`（向下增長架構，如 ARM Cortex-M）**： 堆疊指標（SP）是由高位址向低位址移動。這代表整塊被配置的堆疊記憶體中，最低位址（`pxTCB->pxStack`）是最不可能被立即使用的區域（只有當堆疊即將溢位時才會到達此處）。因此，檢索起點 `pucEndOfStack` 設為該配置空間的基底最低位址。
- **`portSTACK_GROWTH > 0`（向上增長架構）**： 堆疊指標是由低位址向高位址移動。這代表整塊配置記憶體中，最高位址（`pxTCB->pxEndOfStack`）是最末端的邊界。因此，檢索起點必須設為最高位址。

#### 84.3 調用內部分析函數與數值回傳

```C
uxReturn = prvTaskCheckFreeStackSpace( pucEndOfStack );

        traceRETURN_uxTaskGetStackHighWaterMark2( uxReturn );

        return uxReturn;
    }

#endif /* INCLUDE_uxTaskGetStackHighWaterMark2 */
```

- **`prvTaskCheckFreeStackSpace`**：此為內部靜態函數，接收由前一區塊決定的起點指標 `pucEndOfStack`。它會從該位址開始向反方向逐一檢查記憶體內容，計算連續保持初始魔術數字（`0xa5`）的空間長度，並將其轉換為字組（Word）單位。
- **`uxReturn`**：儲存分析結果並將其傳回。此數值越大，代表堆疊剩餘的安全空間越多；若數值接近 `0`，則代表該任務曾面臨堆疊溢位（Stack Overflow）的風險。

### 85. *uxTaskGetStackHighWaterMark*

跟 [[#84. uxTaskGetStackHighWaterMark2]] 一樣只是返回值形態不同

|**項目**|**uxTaskGetStackHighWaterMark() (本函數)**|**uxTaskGetStackHighWaterMark2()**|
|---|---|---|
|**引入版本**|FreeRTOS 早期版本即存在|FreeRTOS 較新版本引進|
|**傳回值型態**|`UBaseType_t` (依架構固定為 16 或 32 位元)|`configSTACK_DEPTH_TYPE` (由使用者自訂)|
|**設計出發點**|標準通用接口，符合多數 16/32 位元架構需求。|解決 8 位元微控制器上，堆疊深度超過 255（8 位元上限）時發生的資料溢位問題。|
|**應用建議**|在 32 位元微處理器（如 STM32, ESP32）中，兩者行為與效能完全一致，通常維持使用此標準版即可。|若開發環境涉及 8 位元架構，或有嚴格的跨平台變數型態對齊需求，建議採用 Version 2。|

### 86. *prvDeleteTCB*

`prvDeleteTCB()` 是 FreeRTOS 處理任務刪除流程中，**執行記憶體實體釋放**的底層核心函數。

當任務透過 `vTaskDelete()` 被宣告刪除，且其資源被移出運行與延時等狀態串列後，最終會由空閒任務（Idle Task）或刪除執行緒呼叫此函數，將該任務所佔用的任務控制區塊（TCB）**與**任務堆疊（Stack）從記憶體中徹底釋放。由於 FreeRTOS 支援靜態與動態記憶體配置，此函數內部包含了大量的條件編譯，用以彈性適應不同的記憶體分配情境。

#### 86.1 條件編譯與硬體層級清理

```C
#if ( INCLUDE_vTaskDelete == 1 )

    static void prvDeleteTCB( TCB_t * pxTCB )
    {
        /* This call is required specifically for the TriCore port.  It must be
         * above the vPortFree() calls.  The call is also used by ports/demos that
         * want to allocate and clean RAM statically. */
        portCLEAN_UP_TCB( pxTCB );
```

- **功能功能開關（`INCLUDE_vTaskDelete`）**：此函數完全封裝在任務刪除功能的巨集內。若應用程式不允許在執行期刪除任務，則此段程式碼不會進入編譯，以節省記憶體空間。
- **硬體平台清理（`portCLEAN_UP_TCB`）**：這是一個架構相關的巨集。特定處理器架構（如 Infineon TriCore）在硬體層級有特殊的上下文暫存器管理機制，必須在 TCB 的記憶體被釋放**之前**，先呼叫此硬體抽象層接口進行暫存器資源的復位或銷毀。對於多數標準架構（如 ARM Cortex-M），此巨集通常被定義為空。

#### 86.2 執行期執行緒區域儲存區（TLS）資源釋放

```C
#if ( configUSE_C_RUNTIME_TLS_SUPPORT == 1 )
        {
            /* Free up the memory allocated for the task's TLS Block. */
            configDEINIT_TLS_BLOCK( pxTCB->xTLSBlock );
        }
        #endif
```

- **TLS 支援（`configUSE_C_RUNTIME_TLS_SUPPORT`）**：執行緒區域儲存區（Thread Local Storage）允許每個任務擁有獨立的全域變數副本（如標準 C 函式庫中的 `errno`）。
- **記憶體解構（`configDEINIT_TLS_BLOCK`）**：若開啟此功能，任務建立時會額外配置一塊 TLS 記憶體區塊。在銷毀 TCB 之前，必須先呼叫此配置的解構巨集，釋放 TLS 專屬的記憶體，防止發生記憶體洩漏（Memory Leak）。

#### 86.3 純動態配置環境下的資源釋放

```C
#if ( ( configSUPPORT_DYNAMIC_ALLOCATION == 1 ) && ( configSUPPORT_STATIC_ALLOCATION == 0 ) && ( portUSING_MPU_WRAPPERS == 0 ) )
        {
            /* The task can only have been allocated dynamically - free both
             * the stack and TCB. */
            vPortFreeStack( pxTCB->pxStack );
            vPortFree( pxTCB );
        }
```

- **特定配置最佳化**：此分支屬於優化路徑。當系統設定為「僅支援動態配置」（`configSUPPORT_DYNAMIC_ALLOCATION == 1` 且 `configSUPPORT_STATIC_ALLOCATION == 0`），且未啟用記憶體保護單元（`portUSING_MPU_WRAPPERS == 0`）時，核心可以明確斷定全系統所有任務的 TCB 與 Stack 一律來自於堆疊（Heap）動態申請。

#### 86.4 混合配置環境下的動態判斷邏輯

```C
#elif ( tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE != 0 )
        {
            /* The task could have been allocated statically or dynamically, so
             * check what was statically allocated before trying to free the
             * memory. */
            if( pxTCB->ucStaticallyAllocated == tskDYNAMICALLY_ALLOCATED_STACK_AND_TCB )
            {
                /* Both the stack and TCB were allocated dynamically, so both
                 * must be freed. */
                vPortFreeStack( pxTCB->pxStack );
                vPortFree( pxTCB );
            }
            else if( pxTCB->ucStaticallyAllocated == tskSTATICALLY_ALLOCATED_STACK_ONLY )
            {
                /* Only the stack was statically allocated, so the TCB is the
                 * only memory that must be freed. */
                vPortFree( pxTCB );
            }
            else
            {
                /* Neither the stack nor the TCB were allocated dynamically, so
                 * nothing needs to be freed. */
                configASSERT( pxTCB->ucStaticallyAllocated == tskSTATICALLY_ALLOCATED_STACK_AND_TCB );
                mtCOVERAGE_TEST_MARKER();
            }
        }
        #endif /* configSUPPORT_DYNAMIC_ALLOCATION */
    }

#endif /* INCLUDE_vTaskDelete */
```

當系統同時支援靜態與動態配置任務時（`tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE != 0`），核心無法在編譯期得知該任務的來源，因此必須在執行期透過 TCB 內部的標記變數 `ucStaticallyAllocated` 進行分支判斷：

- **情境一：`tskDYNAMICALLY_ALLOCATED_STACK_AND_TCB`** 堆疊與 TCB 皆由動態申請。核心依序呼叫 `vPortFreeStack()` 與 `vPortFree()` 將兩者釋放。
- **情境二：`tskSTATICALLY_ALLOCATED_STACK_ONLY`** 此為特定高階 API（如 `xTaskCreateRestrictedWithInternalBuffer()`）會產生的特殊狀態：堆疊是靜態陣列，但 TCB 是動態申請。此時核心**僅能釋放 TCB 記憶體**（`vPortFree( pxTCB )`），而靜態堆疊記憶體則交由使用者或原生命週期管理。
- **情境三：其餘狀態（皆為靜態配置）** 若兩者皆為靜態配置（`tskSTATICALLY_ALLOCATED_STACK_AND_TCB`），核心不需執行任何 `Free` 動作。分支內的 `configASSERT` 與 `mtCOVERAGE_TEST_MARKER()` 用於確保狀態標記的正確性並供代碼覆蓋率測試使用。

#### 86.5 複習核心記憶體釋放狀態

|**ucStaticallyAllocated 狀態值**|**堆疊（Stack）來源**|**TCB 來源**|**核心執行的釋放動作**|
|---|---|---|---|
|`tskDYNAMICALLY_ALLOCATED_STACK_AND_TCB`|動態配置 (Heap)|動態配置 (Heap)|釋放 Stack 空間 + 釋放 TCB 空間|
|`tskSTATICALLY_ALLOCATED_STACK_ONLY`|靜態配置 (陣列)|動態配置 (Heap)|**僅**釋放 TCB 空間|
|`tskSTATICALLY_ALLOCATED_STACK_AND_TCB`|靜態配置 (陣列)|靜態配置 (變數)|不執行任何記憶體釋放動作|

### 87. *prvResetNextTaskUnblockTime*

`prvResetNextTaskUnblockTime()` 在 FreeRTOS 的時間管理機制中，扮演著「計時中斷效能最佳化守門員」的角色。

當一個任務呼叫 `vTaskDelay()` 或 `vTaskDelayUntil()` 進入阻塞狀態（Blocked）時，它會被放入延時串列（`pxDelayedTaskList`）。FreeRTOS 的設計非常聰明：**延時串列是按照「喚醒時間」由小到大排序的**。這意味著，全系統中最先需要被喚醒的任務，永遠排在該串列的**最前端（Head Entry）**。

為了避免系統在每次「晶片計時器中斷（Tick Interrupt）」發生時，都花費昂貴的 CPU 週期去掃描整個延時串列，FreeRTOS 引入了一個全域變數 **`xNextTaskUnblockTime`**。這個變數專門紀錄「下一個任務需要被喚醒的精準時間點」。每次中斷只要用當前的 `xTickCount` 與這個變數做一次快取比較即可。而本函數的目的，就是負責**即時更新這個變數的數值**。

#### 87.1 判斷延時串列是否為空（無任務等待喚醒）

```C
static void prvResetNextTaskUnblockTime( void )
{
    /* 檢查當前的延時任務串列是否為空 */
    if( listLIST_IS_EMPTY( pxDelayedTaskList ) != pdFALSE )
    {
        /* The new current delayed list is empty.  Set xNextTaskUnblockTime to
         * the maximum possible value so it is  extremely unlikely that the
         * if( xTickCount >= xNextTaskUnblockTime ) test will pass until
         * there is an item in the delayed list. */
         
        /* 若沒有任何任務在等待延時，則將下一次喚醒時間設為硬體最大值 */
        xNextTaskUnblockTime = portMAX_DELAY;
    }
```

- **`listLIST_IS_EMPTY`**：核心內部的巨集，用來確認目前系統中是否沒有任何任務正在進行定時阻塞。
- **安全防護機制（`portMAX_DELAY`）**： 若當前沒有任何任務需要被喚醒，排程器會將 `xNextTaskUnblockTime` 設為最大可能值（在 32 位元系統上為 `0xFFFFFFFF`）。這樣做的目的是為了**保護系統計時中斷（Tick Interrupt）的執行效率**。在中斷服務程式中，有一行關鍵判斷：`if( xTickCount >= xNextTaskUnblockTime )`。當該變數被設為最大值時，這個條件判斷在接下來的運作中幾乎不可能成立，系統便能直接跳過喚醒流程，大幅縮短中斷處理時間。

#### 87.2 獲取最前端任務的喚醒時間

```C
else
    {
        /* The new current delayed list is not empty, get the value of
         * the item at the head of the delayed list.  This is the time at
         * which the task at the head of the delayed list should be removed
         * from the Blocked state. */
         
        /* 串列不為空，直接讀取排在最前端（最早需要被喚醒）任務的解鎖時間 */
        xNextTaskUnblockTime = listGET_ITEM_VALUE_OF_HEAD_ENTRY( pxDelayedTaskList );
    }
}
```

- **`listGET_ITEM_VALUE_OF_HEAD_ENTRY`**： FreeRTOS 的鏈結串列節點中，`xItemValue` 欄位在延時串列的情境下，存放的就是該任務**預期被喚醒的目標 Tick 值**。
- **極致的 $\mathcal{O}(1)$ 最佳化**：由於串列本身在任務插入時就已經完成了時間排序，核心不需要使用任何迴圈去尋找「誰最早醒來」。核心只需要直接讀取第一筆資料（Head Entry），並將該值賦予 `xNextTaskUnblockTime`。當下一次計時器中斷來臨時，只要 `xTickCount` 累加到了這個數值，系統就會立刻得知「排在最前端的任務時間到了」，並精準地將其移出阻塞狀態。

### 88. *xTaskGetCurrentTaskHandle* & *xTaskGetCurrentTaskHandleForCore*

這個函數組合（`xTaskGetCurrentTaskHandle` 與 `xTaskGetCurrentTaskHandleForCore`）是 FreeRTOS 用來獲取當前正在執行之任務控制代碼（Task Handle）的公開接口。

隨著 FreeRTOS 支援對稱多處理（SMP）架構，此處展示了單核心與多核心環境下，系統在處理「目前執行任務」時的硬體抽象與執行緒安全（Thread Safety）設計差異。

#### 88.1 條件編譯與單核心下的原子性讀取

```C
#if ( ( INCLUDE_xTaskGetCurrentTaskHandle == 1 ) || ( configUSE_RECURSIVE_MUTEXES == 1 ) ) || ( configNUMBER_OF_CORES > 1 )

    #if ( configNUMBER_OF_CORES == 1 )
        TaskHandle_t xTaskGetCurrentTaskHandle( void )
        {
            TaskHandle_t xReturn;

            traceENTER_xTaskGetCurrentTaskHandle();

            /* A critical section is not required as this is not called from
             * an interrupt and the current TCB will always be the same for any
             * individual execution thread. */
            xReturn = pxCurrentTCB;

            traceRETURN_xTaskGetCurrentTaskHandle( xReturn );

            return xReturn;
        }
```

- **多重條件編譯觸發**：此函數集除了在明確開啟 `INCLUDE_xTaskGetCurrentTaskHandle` 時會編譯外，當開啟遞迴互斥鎖（`configUSE_RECURSIVE_MUTEXES == 1`）時也會強制編譯。這是因為遞迴鎖需要調用此 API 來檢查「目前嘗試獲取鎖的任務」是否就是「鎖的當前擁有者」，以決定是否允許重入（Re-entrancy）。
- **無鎖化讀取（Lock-free Read）**：在單核心環境（`configNUMBER_OF_CORES == 1`）下，全域變數 `pxCurrentTCB` 是一個純指標。在 32 位元或 16 位元架構中，對單一指標的讀取本身就是**原子操作（Atomic Operation）**。如同官方註解所述，此函數不會在非同步的中斷服務程式（ISR）中被呼叫，且在當前執行緒中，此指標具有確定性，因此不需要加入臨界區區塊保護。

#### 88.2 多核心環境下的排程安全與中斷屏蔽

```C
#else /* #if ( configNUMBER_OF_CORES == 1 ) */
        TaskHandle_t xTaskGetCurrentTaskHandle( void )
        {
            TaskHandle_t xReturn;
            UBaseType_t uxSavedInterruptStatus;

            traceENTER_xTaskGetCurrentTaskHandle();

            uxSavedInterruptStatus = portSET_INTERRUPT_MASK();
            {
                xReturn = pxCurrentTCBs[ portGET_CORE_ID() ];
            }
            portCLEAR_INTERRUPT_MASK( uxSavedInterruptStatus );

            traceRETURN_xTaskGetCurrentTaskHandle( xReturn );

            return xReturn;
        }
    #endif /* #if ( configNUMBER_OF_CORES == 1 ) */
```

- **指標陣列化 `pxCurrentTCBs[]`**：在多核心（SMP）環境下，多個核心會同時執行不同的任務。因此，原本的單一指標被擴展為指標陣列，陣列的大小等於系統的核心總數。
- **中斷屏蔽保護（`portSET_INTERRUPT_MASK`）**： 在多核心環境下，獲取當前核心 ID（`portGET_CORE_ID()`）並檢索陣列的操作**並非單一週期的原子操作**。如果在執行 `portGET_CORE_ID()` 之後、讀取陣列之前發生了硬體中斷或任務切換，該任務可能會被排程器遷移（Migration）至另一個核心上繼續執行。這將導致讀取到錯誤的核心索引值，造成資料不一致。透過中斷屏蔽，可以確保「獲取核心 ID」與「查表取值」這兩個步驟在同一運作脈絡下連續完成，達到執行緒安全。

#### 88.3 跨核心任務控制代碼檢索

```C
TaskHandle_t xTaskGetCurrentTaskHandleForCore( BaseType_t xCoreID )
    {
        TaskHandle_t xReturn = NULL;

        traceENTER_xTaskGetCurrentTaskHandleForCore( xCoreID );

        if( taskVALID_CORE_ID( xCoreID ) != pdFALSE )
        {
            #if ( configNUMBER_OF_CORES == 1 )
                xReturn = pxCurrentTCB;
            #else /* #if ( configNUMBER_OF_CORES == 1 ) */
                xReturn = pxCurrentTCBs[ xCoreID ];
            #endif /* #if ( configNUMBER_OF_CORES == 1 ) */
        }

        traceRETURN_xTaskGetCurrentTaskHandleForCore( xReturn );

        return xReturn;
    }

#endif /* ( ( INCLUDE_xTaskGetCurrentTaskHandle == 1 ) || ( configUSE_RECURSIVE_MUTEXES == 1 ) ) */
```

- **`taskVALID_CORE_ID`**：防禦性程式設計巨集，用來驗證傳入的 `xCoreID` 是否在合法的硬體核心編號範圍內（例如 0 到 `configNUMBER_OF_CORES - 1`），避免陣列存取越界（Out-of-Bounds）。
- **跨核心快取讀取**：此函數允許核心 A 查詢核心 B 當前正在執行什麼任務。
- **未屏蔽中斷的原因**：此函數內部沒有使用 `portSET_INTERRUPT_MASK()`。因為這是針對特定核心 ID 進行的直接索引讀取，即便目標核心在同一瞬間發生中斷並修改了它自己的 `pxCurrentTCBs[xCoreID]`，本核心讀取該主記憶體位址的動作依然是硬體層級原子的，且本地的中斷屏蔽無法阻止其他核心的硬體行為，故此處直接執行分支讀取以最佳化效率。

### 89. *xTaskGetSchedulerState*

`xTaskGetSchedulerState()` 是 FreeRTOS 用來獲取「排程器當前工作狀態」的公開 API。

#### 89.1 條件編譯與函數宣告

```C
#if ( ( INCLUDE_xTaskGetSchedulerState == 1 ) || ( configUSE_TIMERS == 1 ) )

    BaseType_t xTaskGetSchedulerState( void )
    {
        BaseType_t xReturn;

        traceENTER_xTaskGetSchedulerState();
```

**雙重條件編譯控制**：此函數會在滿足以下任一條件時被編譯：
1. `INCLUDE_xTaskGetSchedulerState == 1`：使用者明確啟用此查詢 API。
2. `configUSE_TIMERS == 1`：系統啟用了軟體計時器。因為計時器元件內部的某些 API（例如 `xTimerGenericCommand()`）需要即時獲取排程器狀態，以決定要將指令直接寫入隊列，還是在排程器未啟動前暫存於特定結構。

#### 89.2 第一層檢查：排程器是否已啟動

```C
if( xSchedulerRunning == pdFALSE )
        {
            xReturn = taskSCHEDULER_NOT_STARTED;
        }
```

- **`xSchedulerRunning` 全域變數**：這是系統初始化狀態的標記。當系統剛完成開機、在 `main()` 函式中建立任務時，此全域變數恆為 `pdFALSE`。
- **狀態轉移點**：只有當應用程式呼叫了 `vTaskStartScheduler()`，且作業系統核心成功接管硬體計時器中斷並開始第一次任務切換後，此變數才會被內部更改為 `pdTRUE`。
- **回傳值**：若為 `pdFALSE`，直接返回 `taskSCHEDULER_NOT_STARTED`，代表目前仍處於硬體初始化與任務建立階段，核心排程尚未運作。

#### 89.3 第二層檢查：多核心臨界區與掛起狀態判定

```C
else
        {
            #if ( configNUMBER_OF_CORES > 1 )
                taskENTER_CRITICAL();
            #endif
            {
                if( uxSchedulerSuspended == ( UBaseType_t ) 0U )
                {
                    xReturn = taskSCHEDULER_RUNNING;
                }
                else
                {
                    xReturn = taskSCHEDULER_SUSPENDED;
                }
            }
            #if ( configNUMBER_OF_CORES > 1 )
                taskEXIT_CRITICAL();
            #endif
        }

        traceRETURN_xTaskGetSchedulerState( xReturn );

        return xReturn;
    }

#endif /* ( ( INCLUDE_xTaskGetSchedulerState == 1 ) || ( configUSE_TIMERS == 1 ) ) */
```

當確認排程器已經在運作（`xSchedulerRunning == pdTRUE`）後，核心需要進一步確認排程器當前是否被「人為掛起（Suspended）」：

- **`uxSchedulerSuspended` 巢狀計數器**：當應用程式呼叫 `vTaskSuspendAll()` 時，此變數會遞增；呼叫 `xTaskResumeAll()` 時則會遞減。只要此變數大於 `0`（即不等於 `0U`），代表排程器當前被鎖定，此時硬體中斷雖然正常接收，但**中斷服務程式不會觸發任何任務切換（Context Switch）**。
- **多核心（SMP）環境下的臨界區保護**： 在單核心環境中，讀取 `uxSchedulerSuspended` 是一個原子操作。然而在多核心（`configNUMBER_OF_CORES > 1`）架構下，多個核心可能同時執行不同的任務，並且其中一個核心可能正在執行 `vTaskSuspendAll()` 修改排程器鎖定狀態。為了確保當前核心讀取該變數時的**記憶體可見性（Memory Visibility）與資料一致性**，FreeRTOS 在多核心架構下會使用 `taskENTER_CRITICAL()` 獲取**硬體旋鎖（Spinlock）**，鎖定跨核心的存取脈絡，查明狀態後隨即釋放。

#### 89.4 補充

|**回傳常量**|**對應數值**|**排程器實際硬體與核心行為**|**觸發此狀態的典型 API / 事件**|
|---|---|---|---|
|**`taskSCHEDULER_NOT_STARTED`**|`0`|排程器未啟動。硬體 Tick 中斷尚未開啟，所有已建立的任務均處於就緒或初始化狀態，CPU 仍由 `main()` 控制。|系統剛啟動，尚未呼叫 `vTaskStartScheduler()`。|
|**`taskSCHEDULER_RUNNING`**|`1`|排程器正常運作中。核心會根據優先權與時間片，在 Tick 中斷時自動執行任務切換，系統處於完全多工狀態。|`vTaskStartScheduler()` 執行成功後。|
|**`taskSCHEDULER_SUSPENDED`**|`2`|排程器被鎖定。硬體 Tick 中斷仍會累加系統時間（`xTickCount`），但**禁止任務切換**。當前任務將獨佔 CPU 直到解鎖。|呼叫了 `vTaskSuspendAll()`，常用於保護一段不希望被中斷干擾、但又不希望關閉硬體中斷的關鍵程式碼。|
### 90. *xTaskPriorityInherit*

`xTaskPriorityInherit()` 是 FreeRTOS 用來解決即時作業系統（RTOS）中經典之「優先權翻轉（Priority Inversion）」問題的核心機制。

當一個高優先權任務（即當前執行中的 `pxCurrentTCB`）嘗試獲取一個已被低優先權任務（`pxMutexHolder`）佔有的互斥鎖（Mutex）時，高優先權任務將會被迫進入阻塞狀態。為了防止中優先權任務搶佔該低優先權任務、導致高優先權任務無限期等待，排程器會呼叫此函數，**將低優先權任務的優先權暫時提升至與高優先權任務相同的級別**。這就是「優先權繼承（Priority Inheritance）」的核心演算法。

#### 90.1 條件判定與事件串列數值更新

```C
#if ( configUSE_MUTEXES == 1 )

    BaseType_t xTaskPriorityInherit( TaskHandle_t const pxMutexHolder )
    {
        TCB_t * const pxMutexHolderTCB = pxMutexHolder;
        BaseType_t xReturn = pdFALSE;

        traceENTER_xTaskPriorityInherit( pxMutexHolder );

        /* 若互斥鎖是由中斷服務程式獲取，則持有者為 NULL，此情境不適用優先權繼承 */
        if( pxMutexHolder != NULL )
        {
            /* 檢查：互斥鎖持有者的當前優先權，是否低於嘗試獲取鎖的任務（當前任務） */
            if( pxMutexHolderTCB->uxPriority < pxCurrentTCB->uxPriority )
            {
                /* 調整持有者任務的事件串列項數值（Event List Item Value）。
                 * 僅在該數值未被用於其他事件（如等待信號量或隊列）時才進行重設。 */
                if( ( listGET_LIST_ITEM_VALUE( &( pxMutexHolderTCB->xEventListItem ) ) & taskEVENT_LIST_ITEM_VALUE_IN_USE ) == ( ( TickType_t ) 0U ) )
                {
                    /* 更新事件串列中的優先權排序權重值 */
                    listSET_LIST_ITEM_VALUE( &( pxMutexHolderTCB->xEventListItem ), ( TickType_t ) configMAX_PRIORITIES - ( TickType_t ) pxCurrentTCB->uxPriority );
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
```

- **中斷排除機制**：若 `pxMutexHolder` 為 `NULL`，代表此互斥鎖可能在中斷服務程式（ISR）中被操作（雖然標準 Mutex 不建議在中斷中使用），核心將直接跳過繼承邏輯。
- **觸發條件**：必須是「目前鎖持有者的優先權（`uxPriority`）」低於「當前請求鎖任務的優先權（`pxCurrentTCB->uxPriority`）」。如果持有者的優先權本來就比較高，則不需處理。
- **`xEventListItem` 權重更新**：FreeRTOS 鏈結串列（List）是以數值由小到大排序。為了讓高優先權任務在事件串列中排在最前面，其內部的排序值計算公式為 `configMAX_PRIORITIES - uxPriority`。此處更新該數值，可確保其在事件驅動時具備正確的優先順序。

#### 90.2 就緒任務的串列搬移與多核心主動讓權

```C
/* 若被修改優先權的任務目前處於「就緒（Ready）」狀態，必須將其搬移至對應新優先權的就緒串列 */
                if( listIS_CONTAINED_WITHIN( &( pxReadyTasksLists[ pxMutexHolderTCB->uxPriority ] ), &( pxMutexHolderTCB->xStateListItem ) ) != pdFALSE )
                {
                    /* 將該任務自舊優先權的就緒串列中移除 */
                    if( uxListRemove( &( pxMutexHolderTCB->xStateListItem ) ) == ( UBaseType_t ) 0 )
                    {
                        /* 若移除後該原就緒串列變為空，則清除對應的優先權位元映像標記（Bit map） */
                        portRESET_READY_PRIORITY( pxMutexHolderTCB->uxPriority, uxTopReadyPriority );
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }

                    /* 在正式移入新串列前，更新其 TCB 內部的優先權數值 */
                    pxMutexHolderTCB->uxPriority = pxCurrentTCB->uxPriority;
                    /* 將任務加入新優先權的就緒串列中 */
                    prvAddTaskToReadyList( pxMutexHolderTCB );
                    
                    #if ( configNUMBER_OF_CORES > 1 )
                    {
                        /* 多核心（SMP）特定邏輯：由於此任務的優先權已被調高，
                         * 若它當前並未在任何核心上運作，應立即向系統發出讓權（Yield）請求，使其能儘速執行。 */
                        if( taskTASK_IS_RUNNING( pxMutexHolderTCB ) != pdTRUE )
                        {
                            prvYieldForTask( pxMutexHolderTCB );
                        }
                    }
                    #endif /* if ( configNUMBER_OF_CORES > 1 ) */
                }
                else
                {
                    /* 若任務此時不處於就緒狀態（例如因等待其他資源而處於阻塞態），
                     * 則不需操作串列，僅需直接變更其 TCB 內部的優先權數值即可。 */
                    pxMutexHolderTCB->uxPriority = pxCurrentTCB->uxPriority;
                }

                traceTASK_PRIORITY_INHERIT( pxMutexHolderTCB, pxCurrentTCB->uxPriority );

                /* 成功執行優先權繼承 */
                xReturn = pdTRUE;
            }
```

- **就緒串列重組**：FreeRTOS 依據優先權設有多個就緒串列（`pxReadyTasksLists[priority]`）。當任務優先權改變時，它在記憶體中的物理位置必須同步遷移。否則排程器在依據位元映像（Bitmap）尋找最高優先權任務時，會無法正確索引到它。
- **多核心（SMP）最佳化**：在多核心環境下，調高一個「當前未在運作（Not Running）」之任務的優先權後，呼叫 `prvYieldForTask()` 可以觸發跨核心中斷（Inter-Processor Interrupt, IPI），強制使其他正在執行低優先權任務的核心進行上下文切換（Context Switch），讓這個被提升的任務立即上線執行，進一步縮短高優先權任務的被阻塞時間。

#### 90.3 巢狀繼承（Nested Inheritance）與回傳判定

```C
else
            {
                /* 進入此分支，代表持有者的當前優先權（uxPriority）已經大於或等於當前任務。
                 * 此時需進一步檢查其「原生基礎優先權（uxBasePriority）」。 */
                if( pxMutexHolderTCB->uxBasePriority < pxCurrentTCB->uxPriority )
                {
                    /* 持有者的原生優先權低於當前任務，但當前優先權卻高於或等於當前任務。
                     * 這代表該持有者先前已經因為「佔有了另一個互斥鎖」，而繼承了另一個更高優先權任務的優先權。
                     * 此情境屬於多重互斥鎖引發的巢狀繼承（Nested Inheritance），核心仍視同發生了繼承行為。 */
                    xReturn = pdTRUE;
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }

        traceRETURN_xTaskPriorityInherit( xReturn );

        return xReturn;
    }

#endif /* configUSE_MUTEXES */
```

- **`uxBasePriority` 的存在意義**：FreeRTOS 任務結構體內維護了兩個優先權變數：`uxPriority`（當前實際執行的優先權）與 `uxBasePriority`（任務最初被建立時、未發生任何繼承的原生優先權）。
- **巢狀繼承（Nested/Multiple Inheritance）**：
	- 若低優先權任務 A 佔有了 Mutex X 與 Mutex Y。
	- 高優先權任務 B（優先權 10）嘗試獲取 Mutex X $\rightarrow$ 任務 A 的 `uxPriority` 被提升至 10。
	- 超高優先權任務 C（優先權 15）嘗試獲取 Mutex Y $\rightarrow$ 進入此函數，此時任務 A 的 `uxPriority`（10）小於任務 C（15），將會再次觸發前述區塊，將任務 A 提升至 15。
	- 另一種情況：若先發生步驟 2，再發生步驟 1。當任務 B 嘗試獲取 Mutex X 時，任務 A 的 `uxPriority` 已經是 15，不小於任務 B 的 10。核心便會切入此 `else` 分支。透過檢查 `uxBasePriority`，核心得知任務 A 本質上仍是低優先權任務，因此正確返回 `pdTRUE`，用以維持內部互斥鎖計數器的正確對齊。

#### 90.4 補充 *portRESET_READY_PRIORITY*

```C
#define portRESET_READY_PRIORITY( uxPriority, uxReadyPriorities ) \
        ( uxReadyPriorities ) &= ~( 1UL << ( uxPriority ) )
```
### 91. *xTaskPriorityDisinherit*

`xTaskPriorityDisinherit()` 是 FreeRTOS 優先權繼承機制的後半段。它的核心任務是：**「當任務釋放（Give）互斥鎖（Mutex）時，負責解除（還原）先前因為優先權翻轉而暫時被提升的優先權。」**

當一個低優先權任務完成了它的臨界區操作，並呼叫 `xSemaphoreGive()` 歸還互斥鎖時，排程器會觸發此函數。如果該任務先前曾被迫「繼承」了高優先權，此函數會評估是否該將其**降回原生的基礎優先權（Base Priority）**，以維護系統排程的公平性與即時性。

完整情境時序模擬：
1. **任務 A（低優先權，例如 3）** 拿走了 Mutex。
2. **任務 B（高優先權，例如 10）** 呼叫 `xSemaphoreTake(Mutex, 100 毫秒)` 企圖拿鎖。
3. 因為鎖被任務 A 拿走了，任務 B 進入阻塞（Blocked）狀態開始等。此時觸發優先權繼承，**任務 A 被短暫提升到優先權 10**。
4. 100 毫秒過去了，任務 A 依然抓著鎖不放。
5. 系統的 Tick 中斷發現任務 B 的系統計時器到期了。排程器強行把任務 B 從阻塞串列移回就緒串列。
6. 任務 B 甦醒，並在 `xQueueGenericReceive()`（互斥鎖的底層實作 API）中發現自己是因為「超時」而醒來，根本沒拿到鎖。
7. **就在任務 B 準備離開 API 並回傳 `pdFALSE`（代表沒拿到鎖）的前一刻**，核心會呼叫 `vTaskPriorityDisinheritAfterTimeout`，去處理那個還在爽用高優先權的任務 A。
8. 因為任務 A 本來優先級低於任務 B，如果有任務 C 優先級大於 A 但小於 B 且沒使用`vTaskPriorityDisinheritAfterTimeout`，任務 A 就可以一直跑，這對於 B 不公平，因為任務 A 本來的優先權低於 B
9. 此外，任務 A 被設為 「目前還在排隊等這個 Mutex 的所有任務中，優先權最高（最大）的那個值」**，絕對**不是當前正在執行任務的優先級！如果沒有其他在等 mutex 的任務的話，A任務的優先級會被設為 A 原本的 `uxBasePriority`，因此在 B 拿不到鎖時就會呼叫此 API 調整 A 任務的優先級為 `max("A 的 uxBasePriority", "目前還在排隊等這個 Mutex 的所有任務中，優先權最高（最大）的那個值")`


RTOS 互斥鎖與優先權的「借貸哲學」

1. **為什麼要提升優先權？**
   - **不是**為了讓鎖持有者 A 趕快做完。
   - **而是**為了保護「正在等待該鎖的高優先權任務 B」，不被其他無關的中優先權任務搶佔。

2. **為什麼超時（Timeout）要立刻降級？**
   - 因為當 B 放棄等待後，保護 A 的「正當理由」已經消失。
   - 如果不降級，A 就會非法持有高優先權（特權洩漏），進而錯誤地欺負其他原本優先權比 A 高、但跟鎖無關的就緒任務。

3. **結論**
   - 寧可讓 A 降回低優先權慢慢磨（即使這會延遲鎖的釋放），也絕對不能允許 A 頂著不屬於它的特權，去干擾系統中其他不相干的任務排程。
#### 91.1 條件編譯、安全斷言與持有鎖計數遞減

```C
#if ( configUSE_MUTEXES == 1 )

    BaseType_t xTaskPriorityDisinherit( TaskHandle_t const pxMutexHolder )
    {
        TCB_t * const pxTCB = pxMutexHolder;
        BaseType_t xReturn = pdFALSE;

        traceENTER_xTaskPriorityDisinherit( pxMutexHolder );

        if( pxMutexHolder != NULL )
        {
            /* A task can only have an inherited priority if it holds the mutex.
             * If the mutex is held by a task then it cannot be given from an
             * interrupt, and if a mutex is given by the holding task then it must
             * be the running state task. */
             
            /* 安全防護斷言：釋放鎖的任務，必須是目前正在 CPU 上執行的任務 */
            configASSERT( pxTCB == pxCurrentTCB );
            /* 確保該任務內部紀錄的「持有鎖計數器」大於 0 */
            configASSERT( pxTCB->uxMutexesHeld );
            
            /* 遞減該任務所持有的互斥鎖總數 */
            ( pxTCB->uxMutexesHeld )--;
```

- **`configASSERT( pxTCB == pxCurrentTCB )`**：這是 FreeRTOS 的一項核心嚴格規範。**互斥鎖不允許在中斷（ISR）中釋放，且只有當初「強佔」該鎖的任務，自己才有資格歸還它。** 因此，歸還鎖的任務必定是當前正在運行的任務（`pxCurrentTCB`）。
- **`uxMutexesHeld` 巢狀計數器**：FreeRTOS 的 TCB 內部使用此變數來紀錄該任務目前「手上有幾個互斥鎖」。每歸還一個，計數器就減 1。

#### 91.2 檢查優先權狀態與多重鎖判定

```C
/* 檢查：該任務目前的優先權，是否與它最初的原生基礎優先權不同？ */
            if( pxTCB->uxPriority != pxTCB->uxBasePriority )
            {
                /* 關鍵簡化設計：只有當任務把手中「最後一個」互斥鎖都還清了，才執行降級 */
                if( pxTCB->uxMutexesHeld == ( UBaseType_t ) 0 )
                {
```

- **`uxPriority != uxBasePriority`**：如果這兩個數值不相等，代表該任務目前正處於「優先權獲提升」的繼承狀態。
- **FreeRTOS 的多重鎖簡化設計（重要特點）**：
	- 如果一個任務同時拿了 A、B、C 三個互斥鎖，並因此繼承了某個超高優先權。當它歸還鎖 A 時，只要 `uxMutexesHeld` 还不為 0（手上還有 B、C 鎖），**FreeRTOS 就「不會」立刻幫它調降優先權**。
- FreeRTOS 為了追求極致的 $\mathcal{O}(1)$ 執行效率與節省記憶體，內部**沒有**維護一個完整的「優先權繼承歷史堆疊（History Stack）」。它採用較為寬鬆但高效的策略：只要任務身上還掛著任何一個鎖，就繼續維持目前的高優先權，直到手上的鎖全部還清（`uxMutexesHeld == 0`），才一口氣跌回最初的原生基礎優先權 `uxBasePriority`。

#### 91.3 優先權降級、就緒串列搬移與多核心主動釋放 CPU

```C
/* 由於任務正在執行，將其從目前的就緒串列中暫時移除 */
                    if( uxListRemove( &( pxTCB->xStateListItem ) ) == ( UBaseType_t ) 0 )
                    {
                        /* 若移除後該串列空了，清除對應的優先權位元映像標記（Bitmap） */
                        portRESET_READY_PRIORITY( pxTCB->uxPriority, uxTopReadyPriority );
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }

                    /* 正式將優先權打回原形（降回基礎優先權） */
                    traceTASK_PRIORITY_DISINHERIT( pxTCB, pxTCB->uxBasePriority );
                    pxTCB->uxPriority = pxTCB->uxBasePriority;

                    /* 依據調降後的全新優先權，重新計算並重設事件串列項的排序權重 */
                    listSET_LIST_ITEM_VALUE( &( pxTCB->xEventListItem ), ( TickType_t ) configMAX_PRIORITIES - ( TickType_t ) pxTCB->uxPriority );
                    
                    /* 將降級後的任務，塞入對應新優先權的就緒串列中 */
                    prvAddTaskToReadyList( pxTCB );
                    
                    #if ( configNUMBER_OF_CORES > 1 )
                    {
                        /* 多核心（SMP）特定邏輯：由於本任務的優先權降低了，
                         * 它可能已經不再是當前核心上最該被執行的任務，必須通知核心進行讓權（Yield）。 */
                        if( taskTASK_IS_RUNNING( pxTCB ) == pdTRUE )
                        {
                            prvYieldCore( pxTCB->xTaskRunState );
                        }
                    }
                    #endif /* if ( configNUMBER_OF_CORES > 1 ) */
```

- **串列遷移與 Bitmap 重置**：如同優先權提升時需要搬家，降級時也必須從高優先權 ready list 搬回低優先權 ready list。呼叫 `portRESET_READY_PRIORITY` 負責關閉高優先權在狀態地圖上的 Bit。
- **多核心（SMP）`prvYieldCore`**：在單核心環境中，當前任務降級後是否被換場，可以留到函數最後統一回傳旗標處理。但在多核心環境下，當前核心（或目標核心）的排程平衡非常敏感。因為此任務優先權變低了，必須立刻呼叫 `prvYieldCore` 觸發當前硬體核心的排程切換，讓更具有高優先權資格的其他就緒任務浮上來使用 CPU。

#### 91.4 排程切換旗標設定與收尾

```C
/* 返回 true，代表系統需要立刻進行一次上下文切換（Context Switch） */
                    xReturn = pdTRUE;
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }

        traceRETURN_xTaskPriorityDisinherit( xReturn );

        return xReturn;
    }

#endif /* configUSE_MUTEXES */
```

- **`xReturn = pdTRUE` 的必要性**： 當一個任務將互斥鎖還回去，且自己的優先權調降後，代表：
	1. 原先因為沒有鎖而被卡死（Blocked）的那些**真正的高優先權任務**，現在終於有機會醒來了。
	2. 即使沒有其他任務在等這個鎖，由於當前任務自己的優先權「變弱」了， ready list 上可能早就有其他中等優先權的任務在排隊。
-  因此，此處明確回傳 `pdTRUE`。上層呼叫者（如 `xSemaphoreGive`）收到此回傳值後，會立刻在退出 API 前執行一次 `portYIELD_WITHIN_API()` 觸發排程切換，確保高優先權任務能以 0 延遲的姿態立刻接管 CPU。

### 92. *vTaskPriorityDisinheritAfterTimeout*

這個函數 `vTaskPriorityDisinheritAfterTimeout()` 處理的是 FreeRTOS 互斥鎖（Mutex）機制中一個非常微妙且高級的邊緣情境（Edge Case）：**「當高優先權任務因為等不到鎖而『超時（Timeout）』放棄時，鎖持有者該如何降級？」**

#### 92.1 為什麼需要這個函數？
想像一下：低優先權任務 A 拿了互斥鎖。高優先權任務 B 也想拿鎖，於是任務 A 透過「優先權繼承」被提升到了與 B 一樣的高優先權。 然而，任務 B 設定了阻塞超時時間（例如 `vTaskDelay(100)`）。時間到了，鎖依然在任務 A 手上，**任務 B 決定不等了（超時退出）並移出等待隊列**。

這時候問題來了：引起任務 A 優先權提升的「主因（任務 B）」已經不見了，任務 A 應該要降級。但核心不能直接把它打回原形，因為可能還有一個「中優先權任務 C」也正在排隊等這個鎖。

因此，排程器呼叫此函數，並傳入目前**還在死守這個鎖的最高優先權任務是幾級（`uxHighestPriorityWaitingTask`）**，來精準決定任務 A 該降到哪個合理的位階。

#### 92.2 估算目標新優先權

```C
#if ( configUSE_MUTEXES == 1 )

    void vTaskPriorityDisinheritAfterTimeout( TaskHandle_t const pxMutexHolder,
                                              UBaseType_t uxHighestPriorityWaitingTask )
    {
        TCB_t * const pxTCB = pxMutexHolder;
        UBaseType_t uxPriorityUsedOnEntry, uxPriorityToUse;
        const UBaseType_t uxOnlyOneMutexHeld = ( UBaseType_t ) 1;

        traceENTER_vTaskPriorityDisinheritAfterTimeout( pxMutexHolder, uxHighestPriorityWaitingTask );

        if( pxMutexHolder != NULL )
        {
            /* 互斥鎖持有者絕對必須至少持有一個鎖，否則為核心嚴重錯誤 */
            configASSERT( pxTCB->uxMutexesHeld );

            /* 決定鎖持有者應該被調整至哪一個優先權。
             * 數值將取以下兩者的「較大值」：
             * 1. 持有者原本的基礎優先權（uxBasePriority）
             * 2. 目前仍在等待該互斥鎖的所有任務中，最高的那一個優先權（uxHighestPriorityWaitingTask） */
            if( pxTCB->uxBasePriority < uxHighestPriorityWaitingTask )
            {
                uxPriorityToUse = uxHighestPriorityWaitingTask;
            }
            else
            {
                uxPriorityToUse = pxTCB->uxBasePriority;
            }
```

- **防禦性動態防線**：計算 `uxPriorityToUse` 是此函數最聰明的地方。如果任務 B 退出了，但還有任務 C（優先權 8）在等鎖，且鎖持有者原本只有優先權 3。那麼鎖持有者不會直接跌回 3，而是維持在 8，繼續幫任務 C 擋住其他低優先權排程，直到鎖被釋放。

#### 92.3 變更判定與多重鎖簡化限制

```C
/* 檢查：鎖持有者當前的優先權，是否真的需要改變？ */
            if( pxTCB->uxPriority != uxPriorityToUse )
            {
                /* FreeRTOS 優先權繼承的優化簡化設計：
                 * 只有當該任務目前「只持有一個互斥鎖」時，才允許執行超時解除繼承。
                 * 如果它手頭上還抓著其他互斥鎖，那些鎖可能也觸發了獨立的優先權繼承，
                 * 為了避免複雜的鏈狀追蹤，此處會選擇直接維持原狀。 */
                if( pxTCB->uxMutexesHeld == uxOnlyOneMutexHeld )
                {
                    /* 安全斷言：一個因為等不到鎖而超時的任務，絕對不可能是持有鎖的自己 */
                    configASSERT( pxTCB != pxCurrentTCB );

                    /* 紀錄進入函數時的舊優先權，以便後續判斷該任務目前的物理串列狀態 */
                    uxPriorityUsedOnEntry = pxTCB->uxPriority;
                    /* 正式更新 TCB 內部的實際優先權 */
                    traceTASK_PRIORITY_DISINHERIT( pxTCB, uxPriorityToUse );
                    pxTCB->uxPriority = uxPriorityToUse;
```

 **`pxTCB->uxMutexesHeld == 1`**：這是 FreeRTOS 為了追求極致 $\mathcal{O}(1)$ 效能而做出的折衷。如果低優先權任務同時持有複數個鎖，核心很難在不耗費大量記憶體與 CPU 度的情況下，釐清當前的高優先權究竟是由哪一個鎖的等待者所撐起來的。因此，只要手上還有其他鎖，就先不進行超時降級，這是一種「寧可讓它暫時維持高優先權，也不要放錯優先權導致死鎖」的安全策略。

#### 92.4 事件串列權重重設

```C
/* 只有在該事件串列項（Event List Item）沒有被挪作他用時，才重設其數值。
                     * 如果此任務當前正在就緒態，此數值通常是乾淨未被佔用的。 */
                    if( ( listGET_LIST_ITEM_VALUE( &( pxTCB->xEventListItem ) ) & taskEVENT_LIST_ITEM_VALUE_IN_USE ) == ( ( TickType_t ) 0U ) )
                    {
                        /* 依據調降後的目標優先權，更新其在鏈結串列中的排序權重值 */
                        listSET_LIST_ITEM_VALUE( &( pxTCB->xEventListItem ), ( TickType_t ) configMAX_PRIORITIES - ( TickType_t ) uxPriorityToUse );
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }
```

**`taskEVENT_LIST_ITEM_VALUE_IN_USE`**：這是一個遮罩，用來檢查任務是否正因為其他事件（例如等待某個 Queue 或 Semaphore）而被掛在某個事件串列中。如果沒有，核心會安全地更新 `xEventListItem` 的值（`configMAX_PRIORITIES - uxPriorityToUse`），確保其內部的鏈結串列排序權重與新的優先權完全同步。

#### 92.5 非執行中任務的就緒串列實體遷移（含多核心處理）

```C
/* 關鍵差異點：超時發生時，這個「鎖持有者任務」並不是當前正在運行的任務（pxCurrentTCB）。
                     * 它此時可能正處於 Ready、Blocked 或 Suspended 狀態。
                     * 我們只在「它處於 Ready（就緒）狀態」時，才需要幫它的狀態串列進行實體搬家。 */
                    if( listIS_CONTAINED_WITHIN( &( pxReadyTasksLists[ uxPriorityUsedOnEntry ] ), &( pxTCB->xStateListItem ) ) != pdFALSE )
                    {
                        /* 自原本高優先權的就緒串列中移除 */
                        if( uxListRemove( &( pxTCB->xStateListItem ) ) == ( UBaseType_t ) 0 )
                        {
                            /* 若該級別串列清空，清除對應的優先權位元映像標記（Bitmap） */
                            portRESET_READY_PRIORITY( uxPriorityUsedOnEntry, uxTopReadyPriority );
                        }
                        else
                        {
                            mtCOVERAGE_TEST_MARKER();
                        }

                        /* 將任務塞入新計算出的低優先權就緒串列中 */
                        prvAddTaskToReadyList( pxTCB );
                        
                        #if ( configNUMBER_OF_CORES > 1 )
                        {
                            /* 多核心（SMP）特定邏輯：由於該任務的優先權被調降了，
                             * 若它此時正好在另一個核心上運作，必須強制該核心執行讓權（Yield），
                             * 重新檢視排程是否需要更換為其他更高優先權的任務。 */
                            if( taskTASK_IS_RUNNING( pxTCB ) == pdTRUE )
                            {
                                prvYieldCore( pxTCB->xTaskRunState );
                            }
                        }
                        #endif /* if ( configNUMBER_OF_CORES > 1 ) */
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }

        traceRETURN_vTaskPriorityDisinheritAfterTimeout();
    }

#endif /* configUSE_MUTEXES */
```

**與常規解除繼承的本質差異**：

在正常的 `xTaskPriorityDisinherit()` 中，歸還鎖的任務必定是 `pxCurrentTCB`（當前运行者）。

但在這裡，是因為**別的任務等不下去超時了**，鎖持有者很可能此時「正在就緒串列排隊」或是「因為其他原因在 Blocked 狀態」。因此，程式碼多了一層 `listIS_CONTAINED_WITHIN` 的判斷。只有確認它目前人待在舊優先權的就緒串列（`pxReadyTasksLists[uxPriorityUsedOnEntry]`）中，才執行 `uxListRemove` 與 `prvAddTaskToReadyList` 進行物理上的串列搬家。

### 93. *vTaskYieldWithinAPI*

這個函數 `vTaskYieldWithinAPI()` 是 FreeRTOS 邁入**多核心（SMP, Symmetric Multiprocessing）架構**後非常核心的排程控制機制。

在傳統的**單核心** FreeRTOS 中，當一個 API（例如釋放 Semaphore 或 Mutex）決定要進行任務切換時，它會直接呼叫 `portYIELD()`。即使當時處於臨界區（Critical Section），單核心的臨界區只是單純的「關閉中斷」，在某些架構下強行 Yield 雖然不建議，但核心頂多把臨界區嵌套層數帶到下一個任務去。

然而在多核心（configNUMBER_OF_CORES > 1）**環境下，臨界區的底層通常依賴**硬體旋鎖（Spinlock）來防止多個 CPU 核心同時篡改全域排程資料。如果一個核心在拿著 Spinlock（處於臨界區內）的狀態下貿然執行 `portYIELD()` 切換到其他任務，這個 Spinlock 就會被活生生卡死，**直接導致其他正在等待該鎖的 CPU 核心陷入永久死鎖（Deadlock）！**

因此，`vTaskYieldWithinAPI()` 的存在目的就是：**安全評估當前核心的狀態，決定「立刻讓權」還是「延後（Pending）讓權」**。

#### 93.1 函數宣告與本地中斷鎖定

```C
#if ( configNUMBER_OF_CORES > 1 )

    void vTaskYieldWithinAPI( void )
    {
        UBaseType_t ulState;

        traceENTER_vTaskYieldWithinAPI();

        /* 鎖定當前 CPU 核心的本地中斷，並保存先前的中斷狀態 */
        ulState = portSET_INTERRUPT_MASK();
        {
```

- **`portSET_INTERRUPT_MASK()`**：在進行排程狀態檢查前，必須先關閉**當前核心**的硬體中斷。這是為了防止在檢查臨界區層數時，突然來了一個中斷（ISR）篡改了核心狀態，導致判斷失準。
- **本地性（Local）**：注意，這個操作只影響執行這行程式碼的這顆 CPU 核心，其他核心此時依然在正常運作。

#### 93.2 核心識別與臨界區安全檢查

```C
/* 獲取目前執行此段程式碼的硬體核心 ID（例如 Core 0 或 Core 1） */
            const BaseType_t xCoreID = ( BaseType_t ) portGET_CORE_ID();

            /* 檢查當前核心的臨界區嵌套計數器是否為 0 */
            if( portGET_CRITICAL_NESTING_COUNT( xCoreID ) == 0U )
            {
                /* 情況 A：不在臨界區內，可以安全地立刻執行任務切換 */
                portYIELD();
            }
            else
            {
                /* 情況 B：正處於臨界區內！絕對不能立刻 Yield。
                 * 將該核心的延後讓權旗標（Yield Pending）標記為 true。 */
                xYieldPendings[ xCoreID ] = pdTRUE;
            }
        }
```

- **`portGET_CORE_ID()`**：多核心環境下，陣列索引都需要知道「我是誰」。核心透過硬體暫存器得知目前的 Core ID。
- **`portGET_CRITICAL_NESTING_COUNT(xCoreID) == 0U`**：
	- **如果等於 0**：代表當前任務沒有持有任何跨核心的 Spinlock，處於安全狀態。此時呼叫 `portYIELD()` 會立刻觸發軟體中斷（如 ARM 的 PendSV），實施**當下立即的上下文切換**。
	- **如果不等於 0**：代表當前任務正躲在臨界區裡面。如果這時候切換任務，Spinlock 就釋放不掉了。所以 FreeRTOS 選擇「記帳」——把 `xYieldPendings[xCoreID]` 設為 `pdTRUE`。

#### 93.3 中斷恢復與收尾

```C
/* 恢復當前 CPU 核心先前的中斷遮罩狀態 */
        portCLEAR_INTERRUPT_MASK( ulState );

        traceRETURN_vTaskYieldWithinAPI();
    }
    
#endif /* #if ( configNUMBER_OF_CORES > 1 ) */
```

- **`portCLEAR_INTERRUPT_MASK(ulState)`**：還原第一步關閉的中斷。
- **後續聯動機制（記帳之後誰來還？）**： 當程式碼被標記 `xYieldPendings[xCoreID] = pdTRUE` 後，任務會繼續執行。直到它執行完關鍵區域，呼叫 **`taskEXIT_CRITICAL()`** 退出最後一層臨界區時，FreeRTOS 的退出機制會順便去檢查 `xYieldPendings[xCoreID]`。一旦發現是 `pdTRUE`，核心就會在**退出臨界區的那個安全瞬間，補做剛剛被延後的 `portYIELD()`**。

### 94. *vTaskEnterCritical (single core)*

這個版本的 `vTaskEnterCritical()` 是一個非常特殊的 FreeRTOS 臨界區（Critical Section）實作。它被兩個嚴格的條件所限制：**單核心環境（`configNUMBER_OF_CORES == 1`）** 且 **臨界區巢狀計數器儲存在任務控制區塊中（`portCRITICAL_NESTING_IN_TCB == 1`）**。

為什麼要將計數器存在 TCB 裡？

在傳統單核心 FreeRTOS 中，臨界區計數器通常是一個「全域變數」（`uxCriticalNesting`）。但某些特定的處理器架構（Port）允許**任務在處於臨界區內時被換場（Context Switch）**，或者需要讓每個任務「獨立記住」自己到底進出了臨界區幾次。這時，核心就會把計數器 `uxCriticalNesting` 直接塞進每個任務的身份證（TCB）中，實現任務間的臨界狀態隔離。

#### 94.1 條件編譯與硬體級中斷關閉

```C
#if ( ( portCRITICAL_NESTING_IN_TCB == 1 ) && ( configNUMBER_OF_CORES == 1 ) )

    void vTaskEnterCritical( void )
    {
        traceENTER_vTaskEnterCritical();

        /* 第一步：在 CPU 硬體層級立刻關閉中斷（例如 ARM 的 CPSID i 指令）
         * 這是為了防止在後續修改 TCB 數值時，被突如其來的中斷打斷 */
        portDISABLE_INTERRUPTS();
```

- **`portDISABLE_INTERRUPTS()`**：這是臨界區的物理防線。不論排程器有沒有啟動，這一行都會直接讓當前 CPU 核心停止回應可屏蔽硬體中斷，確保接下來的操作擁有絕對的原子性（Atomicity）。

#### 94.2 排程器狀態檢查與 TCB 巢狀計數遞增

```C
/* 檢查排程器是否已經正式啟動（vTaskStartScheduler 被呼叫後此值為 pdTRUE） */
        if( xSchedulerRunning != pdFALSE )
        {
            /* 將當前正在運作任務（pxCurrentTCB）結構體內部的臨界區巢狀計數器加 1 */
            ( pxCurrentTCB->uxCriticalNesting )++;
```

- **`xSchedulerRunning` 的防護作用**：在系統剛開機、硬體初始化階段（`main()` 函式內），排程器尚未啟動，此時系統中根本還沒有所謂的「當前任務」，`pxCurrentTCB` 指標可能是 `NULL` 或是無效值。如果此時盲目修改 `pxCurrentTCB->uxCriticalNesting` 會直接引發記憶體存取錯誤（Segmentation Fault/Hard Fault）。因此，只有在排程器運作後，核心才會去紀錄 TCB 內部的計數。
- **支援巢狀呼叫（Nesting）**：透過 `uxCriticalNesting++`，FreeRTOS 允許你在程式碼中**重複呼叫** `vTaskEnterCritical()`（例如 A 函式開了臨界區，它呼叫的 B 函式內部也開了臨界區）。系統不會因為 B 函式退出就誤開中斷，必須等到計數器扣回 0 時才會真正開放中斷。

#### 94.3 ISR 安全斷言（Assert）與遞迴防護

```C
/* * 安全檢查：這不是「中斷安全（Interrupt-safe）」版本的進入臨界區函式。
             * 如果在非中斷環境（Task 空間）呼叫此函式，通過；若在中斷（ISR）中呼叫，則觸發中斷斷言錯誤。
             * * 為什麼只在巢狀計數器「等於 1（第一次進入）」時才執行 portASSERT_IF_IN_ISR()？
             * 這是為了防止「遞迴 Assert」導致的堆疊溢位（Stack Overflow）。
             */
            if( pxCurrentTCB->uxCriticalNesting == 1U )
            {
                portASSERT_IF_IN_ISR();
            }
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }

        traceRETURN_vTaskEnterCritical();
    }

#endif /* #if ( ( portCRITICAL_NESTING_IN_TCB == 1 ) && ( configNUMBER_OF_CORES == 1 ) ) */
```

- **`portASSERT_IF_IN_ISR()` 的致命錯誤**：一般普通的 Task API（沒有帶 `FromISR` 結尾的 API）絕對不能在中斷服務程式（ISR）中使用。如果誤用，這個 Macro 會直接鎖死系統並噴出錯誤，提醒開發者這裡用錯 API 了（中斷環境應該使用 `taskENTER_CRITICAL_FROM_ISR()`）。
- **為什麼限縮在 `== 1U` 時檢查？**： 這是一個精妙的防禦性編程（Defensive Programming）設計。如果 `portASSERT_IF_IN_ISR()` 失敗了，它內部可能會呼叫使用者的 Hook 函式來列印錯誤訊息（例如 `printf`）。而這些錯誤處理函式內部**很有可能也會用到臨界區**。 如果每次進臨界區都檢查，那麼在 Assert 失敗後觸發的巢狀呼叫中，就會再次觸發 Assert，導致系統陷入**無窮遞迴的 Assert 漩渦**，最後因為堆疊被推爆而直接死機。只在第一次（`== 1`）檢查，就能完全避開這個潛在災難。

### 95. *vTaskEnterCritical (SMP)*

這個版本的 `vTaskEnterCritical()` 是 FreeRTOS 專門為**多核心（SMP, Symmetric Multiprocessing）架構**所設計的臨界區（Critical Section）實作（當 `configNUMBER_OF_CORES > 1` 時生效）。

在單核心系統中，進臨界區只需要「關閉本地硬體中斷」即可安枕無憂。但在**多核心環境**下，這招失效了！因為即使 Core 0 關閉了中斷，Core 1 此時依然在全速運轉，隨時可能透過排程器修改全域核心資料（如 Ready 串列），導致資料毀損。

因此，多核心的 `vTaskEnterCritical()` 採取了「雙層防禦機制」：

1. **第一層：** 關閉「目前核心」的本地硬體中斷。
    
2. **第二層：** 透過**多核心旋鎖（Spinlock）**，強行讓其他企圖修改排程資料的核心在外面「原地打轉（Spin）」，直到目前核心退出臨界區為止。

#### 95.1 本地中斷屏蔽與核心 ID 獲取

```C
#if ( configNUMBER_OF_CORES > 1 )

    void vTaskEnterCritical( void )
    {
        traceENTER_vTaskEnterCritical();

        /* 【第一層防線】立即關閉當前 CPU 核心的本地硬體中斷
         * 防止在後續讀取 Core ID 或獲取旋鎖時被中斷服務（ISR）打斷 */
        portDISABLE_INTERRUPTS();
        {
            /* 獲取目前執行此段程式碼的硬體核心 ID（例如 Core 0, Core 1）
             * 因為多核心系統的所有臨界區計數與狀態都是以核心為單位進行陣列索引 */
            const BaseType_t xCoreID = ( BaseType_t ) portGET_CORE_ID();
```

- **`portDISABLE_INTERRUPTS()`**：只對當前核心有效。這能確保此核心在去搶多核心硬體旋鎖（Spinlock）時，不會因為突發的中斷而分心，避免引發鎖定時間過長的風險。
- **`portGET_CORE_ID()`**：在多核心架構下，這是極度關鍵的硬體原語（Primitive），核心必須知道自己是誰，才能去操作屬於該核心的專屬臨界區嵌套計數器。

#### 95.2 跨核心雙重旋鎖（Spinlock）判定與獲取

```C
/* 只有在排程器正式啟動後，才需要進行任務與中斷的跨核心鎖定 */
            if( xSchedulerRunning != pdFALSE )
            {
                /* 檢查當前核心是否是「第一次」企圖進入臨界區（巢狀計數為 0） */
                if( portGET_CRITICAL_NESTING_COUNT( xCoreID ) == 0U )
                {
                    /* 依序獲取任務鎖與中斷鎖，防止其他 CPU 核心同時篡改全域排程資料 */
                    portGET_TASK_LOCK( xCoreID );
                    portGET_ISR_LOCK( xCoreID );
                }

                /* 將該核心的臨界區巢狀計數器加 1 */
                portINCREMENT_CRITICAL_NESTING_COUNT( xCoreID );
```

**為什麼是兩把鎖？為什麼是這個順序？**

- **`portGET_TASK_LOCK()`**：保護任務排程資料（如 Ready Lists、Event Lists）。
- **`portGET_ISR_LOCK()`**：保護與中斷相關的核心資料。
- **死鎖防禦（Deadlock Prevention）**：FreeRTOS 嚴格規定，多核心環境下不論在核心源碼哪裡，只要同時需要這兩把鎖，**必須先拿 TASK_LOCK，再拿 ISR_LOCK**。如果順序顛倒（例如另一個核心先拿 ISR 再拿 TASK），兩個核心在交會時就會引發嚴重的**多核心永久死鎖**。

#### 95.3 ISR 誤用斷言與多核心狀態同步（最核心精妙處）

```C
/* 如果這是第一次進入臨界區（計數器剛剛加完後剛好等於 1） */
                if( portGET_CRITICAL_NESTING_COUNT( xCoreID ) == 1U )
                {
                    /* 安全檢查：非 FromISR 結尾的 API 絕不能在中斷（ISR）中呼叫 */
                    portASSERT_IF_IN_ISR();

                    /* 如果排程器此時沒有被掛起（Suspended） */
                    if( urSchedulerSuspended == 0U )
                    {
                        /* 【多核心特有】檢查當前任務是否在等待旋鎖的過程中，
                         * 被其他 CPU 核心強制要求讓出 CPU（Evicted/Yielded） */
                        prvCheckForRunStateChange();
                    }
                }
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }

        traceRETURN_vTaskEnterCritical();
    }

#endif /* #if ( configNUMBER_OF_CORES > 1 ) */
```

- **`portASSERT_IF_IN_ISR()`**：同樣具備「只在計數等於 1 時才檢查」的防禦性編程設計，防止因斷言失敗引發遞迴呼叫導致堆疊潰縮。
- **`prvCheckForRunStateChange()` 的多核心大智慧**： 這是多核心 FreeRTOS 最精彩的一手。想像這個場景：
	1. `Core 1` 的任務 A 想要呼叫 `vTaskEnterCritical()`。
	2. 在上面執行到 `portGET_TASK_LOCK()` 時，因為 `Core 0` 正抓著這把鎖，所以 `Core 1` 的任務 A 就卡在硬體旋鎖那裡**原地狂奔等待（Spinning）**。
	3. 在任務 A 苦苦等待的這段時間內，`Core 0` 的排程器突然發現有一個更高優先權的任務醒了，指定要在 `Core 1` 上執行！於是 `Core 0` 修改了任務 A 的狀態，將其標記為 `taskTASK_SCHEDULED_TO_YIELD`（準備被驅逐下線）。
	4. 終於，`Core 0` 釋放了鎖，`Core 1` 的任務 A 順利拿到了鎖，進入到這一步。
	5. 此時，任務 A 其實已經被「宣判死刑（需要 Yield）」了。`prvCheckForRunStateChange()` 的目的就是讓任務 A 在剛踏進臨界區的第一步，**立刻自我檢查**。如果發現自己已經被標記為要讓權，它會在臨界區內妥善處理好這個多核心狀態的轉換，確保整個 SMP 系統的排程時序不會混亂。官方註解也特別提醒：因為這個自我檢查會去確認狀態，所以絕對不能在進行上下文切換的底層函式 `vTaskSwitchContext()` 內部使用。

### 96. *vTaskEnterCriticalFromISR*

在多核心（SMP）環境下，中斷服務程式（ISR）同樣面臨跨核心競爭全域資源的問題。這個函數就是為了解決「**在中斷環境下，如何安全地進入臨界區**」。

與普通任務的 `vTaskEnterCritical` 相比，它最大的差別在於**效率優化**——它**只拿 `ISR_LOCK`，不拿 `TASK_LOCK`**。

#### 96.1 變數宣告與核心識別

```C
#if ( configNUMBER_OF_CORES > 1 )

    UBaseType_t vTaskEnterCriticalFromISR( void )
    {
        /* 用來儲存關閉中斷前，CPU 原本的中斷屏蔽狀態（Mask） */
        UBaseType_t uxSavedInterruptStatus = 0;
        
        /* 獲取目前執行此段程式碼的硬體核心 ID（如 Core 0, Core 1） */
        const BaseType_t xCoreID = ( BaseType_t ) portGET_CORE_ID();

        traceENTER_vTaskEnterCriticalFromISR();
```

- **`uxSavedInterruptStatus`**：中斷狀態必須被保存並回傳，因為中斷退出時需要將 CPU 恢復到進入前的狀態。
- **`portGET_CORE_ID()`**：多核心架構必備，用來識別當前是哪一個 CPU 核心正在處理這個中斷。

#### 96.2 關閉本地中斷與獲取跨核心旋鎖

```C
/* 確保排程器已經啟動，否則不執行鎖定邏輯 */
        if( xSchedulerRunning != pdFALSE )
        {
            /* 【第一層防線】屏蔽當前核心的中斷，並回傳舊的中斷狀態 */
            uxSavedInterruptStatus = portSET_INTERRUPT_MASK_FROM_ISR();

            /* 檢查當前核心是否是「第一次」進入中斷臨界區（巢狀計數為 0） */
            if( portGET_CRITICAL_NESTING_COUNT( xCoreID ) == 0U )
            {
                /* 【關鍵：第二層防線】獲取中斷旋鎖（ISR_LOCK） */
                portGET_ISR_LOCK( xCoreID );
            }

            /* 遞增當前核心的臨界區巢狀計數器 */
            portINCREMENT_CRITICAL_NESTING_COUNT( xCoreID );
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
```

- 為什麼只拿 `ISR_LOCK`？
	- 在普通任務的臨界區中，核心會同時拿 `TASK_LOCK` 與 `ISR_LOCK`。
	- 但在 **ISR** 中，中斷服務程式必須極度輕量、快速。ISR 通常只會動到與中斷相關的共用資料（例如 Queue 的指標），而**不會去改動排程器的 Ready 串列**。因此，ISR 只需要拿 `ISR_LOCK` 就能保證資料安全，不拿 `TASK_LOCK` 可以大幅減少其他核心任務排程被卡死的機率。
- **巢狀計數（Nesting）保護**： 同一個中斷內部可能會呼叫多個需要臨界區保護的核心 API。透過 `portGET_CRITICAL_NESTING_COUNT` 檢查，確保只有在第一次進入時才去搶硬體旋鎖，避免同核心重入鎖定導致的死鎖。

#### 96.3 返回中斷狀態

```C
traceRETURN_vTaskEnterCriticalFromISR( uxSavedInterruptStatus );

        /* 回傳先前保存的中斷狀態給呼叫者 */
        return uxSavedInterruptStatus;
    }

#endif /* #if ( configNUMBER_OF_CORES > 1 ) */
```

- 為什麼要 return 這個值？
	- 呼叫此 API 的標準寫法為： `uxSavedInterruptStatus = taskENTER_CRITICAL_FROM_ISR();` 退出時必須將此數值傳回： `taskEXIT_CRITICAL_FROM_ISR( uxSavedInterruptStatus );` 這樣硬體才能精確還原該核心進入臨界區前的中斷優先權遮罩，避免破壞原本的中斷嵌套時序。

### 97. *vTaskExitCritical (single core)*

這個函數是先前我們討論過的 `vTaskEnterCritical()` 的**天生配對**（Counterpart）。

它的核心任務是：**安全地「退出」臨界區**。如同俄羅斯套娃一樣，臨界區可以一層套一層（巢狀呼叫）。只有當你把最外層的套娃也拆掉（巢狀計數器歸零）時，系統才會真正把 CPU 的硬體中斷重新打開。

#### 97.1 函數宣告與嚴格的安全防禦（Asserts）

```C
#if ( ( portCRITICAL_NESTING_IN_TCB == 1 ) && ( configNUMBER_OF_CORES == 1 ) )

    void vTaskExitCritical( void )
    {
        traceENTER_vTaskExitCritical();

        /* 只有在排程器運作後，才需要進行 TCB 巢狀計數器的操作與防護 */
        if( xSchedulerRunning != pdFALSE )
        {
            /* 【防錯機制 1】確保目前任務的巢狀計數大於 0。
             * 如果等於 0 還呼叫此函數，代表開發者「超釋放」臨界區（呼叫 Exit 的次數比 Enter 還多） */
            configASSERT( pxCurrentTCB->uxCriticalNesting > 0U );

            /* 【防錯機制 2】此 API 專為普通任務（Task）設計，絕對不能在中斷（ISR）中使用。
             * 在中斷中釋放臨界區應使用 portEXIT_CRITICAL_FROM_ISR() */
            portASSERT_IF_IN_ISR();
```

- **`configASSERT( pxCurrentTCB->uxCriticalNesting > 0U )`**：這是一個極度重要的 Debug 守衛。如果程式不小心多呼叫了一次 `vTaskExitCritical()`，這個 Assert 會直接讓系統停住，防止中斷在不該被打開的時候被意外開啟，進而引發難以追蹤的 Race Condition。
- **`portASSERT_IF_IN_ISR()`**：防止開發者在中斷服務程式中誤用此 API。

#### 97.2 遞減計數與關鍵的中斷開啟

```C
/* 再次確認計數器大於 0，進行防禦性編程 */
            if( pxCurrentTCB->uxCriticalNesting > 0U )
            {
                /* 遞減當前 TCB 中的臨界區巢狀計數器 */
                ( pxCurrentTCB->uxCriticalNesting )--;

                /* 【最關鍵的瞬間】只有當計數器扣到剛好等於 0 時 */
                if( pxCurrentTCB->uxCriticalNesting == 0U )
                {
                    /* 真正調用底層硬體指令，重新開啟 CPU 的硬體中斷 */
                    portENABLE_INTERRUPTS();
                }
                else
                {
                    /* 測試覆蓋率標記，代表此時仍在巢狀臨界區內，中斷依然維持關閉 */
                    mtCOVERAGE_TEST_MARKER();
                }
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
```

- **`(pxCurrentTCB->uxCriticalNesting)--`**：每呼叫一次 Exit，就把當前任務的臨界區層數剝掉一層。
- **`portENABLE_INTERRUPTS()`**：
	- 如果計數器減完後**大於 0**（例如從 2 變成 1），代表還有其他外層函數在保護這段代碼，此時**絕對不能開中斷**。
	- 只有當計數器**等於 0**，代表所有臨界區都已經安全退出，此時才能真正執行 `portENABLE_INTERRUPTS()` 開放中斷。
- **`xSchedulerRunning == pdFALSE` 的處理**：
	- 在系統開機、排程器啟動之前（例如在 `main()` 裡初始化硬體），因為還沒有任何「任務」存在，`pxCurrentTCB` 為無效指標。此時如果呼叫進來，函數會安全地直接退場，避免指針空指針異常。

### 98. *vTaskExitCritical (SMP)*

`vTaskExitCritical()` 是 FreeRTOS 專為**多核心（SMP, Symmetric Multiprocessing）架構**所設計的退出臨界區實作。

如果你還記得我們在第一堂課討論的 `vTaskYieldWithinAPI()`，這兩者是**完美互補、前後呼叫的因果關係**！在多核心環境中，為了避免死鎖，當任務在臨界區內想讓權（Yield）時，系統不會立刻切換任務，而是「記帳」在 `xYieldPendings` 中。現在，在 `vTaskExitCritical()` 退出臨界區的最後一刻，系統終於要把這個「延後讓權（Deferred Yield）」的帳給結清了。

#### 98.1 獲取核心 ID 與嚴格安全斷言

```C
#if ( configNUMBER_OF_CORES > 1 )

    void vTaskExitCritical( void )
    {
        /* 獲取目前執行此段程式碼的硬體核心 ID（例如 Core 0, Core 1） */
        const BaseType_t xCoreID = ( BaseType_t ) portGET_CORE_ID();

        traceENTER_vTaskExitCritical();

        /* 只有在排程器正式啟動後，才執行臨界區計數與解鎖操作 */
        if( xSchedulerRunning != pdFALSE )
        {
            /* 【防錯機制 1】安全檢查：確保當前核心的巢狀計數大於 0。
             * 如果為 0 還呼叫此函數，代表 Enter 和 Exit 的呼叫次數不匹配。 */
            configASSERT( portGET_CRITICAL_NESTING_COUNT( xCoreID ) > 0U );

            /* 【防錯機制 2】此 API 專為普通任務（Task）設計，絕對不能在中斷（ISR）中呼叫。
             * 在中斷中釋放臨界區應使用 vTaskExitCriticalFromISR()。 */
            portASSERT_IF_IN_ISR();
```

- **多核心防錯**：利用 `portGET_CRITICAL_NESTING_COUNT( xCoreID )` 獨立取得當前核心的巢狀計數器，並進行 `configASSERT` 檢查，確保臨界區開關對稱。
- **ISR 隔離**：中斷中絕對禁止呼叫此函數。

#### 98.2 遞減計數與多核心旋鎖（Spinlock）解鎖

```C
if( portGET_CRITICAL_NESTING_COUNT( xCoreID ) > 0U )
            {
                /* 遞減當前核心的臨界區巢狀計數器 */
                portDECREMENT_CRITICAL_NESTING_COUNT( xCoreID );

                /* 【關鍵瞬間】只有當計數器歸零，代表完全退出最外層臨界區時 */
                if( portGET_CRITICAL_NESTING_COUNT( xCoreID ) == 0U )
                {
                    BaseType_t xYieldCurrentTask;

                    /* 1. 在「依然處於臨界區（保護中）」的狀態下，
                     * 安全地讀取此核心是否有尚未處理的延後讓權請求。 */
                    xYieldCurrentTask = xYieldPendings[ xCoreID ];

                    /* 2. 釋放多核心旋鎖（解鎖順序與獲取時完全相反：LIFO） */
                    portRELEASE_ISR_LOCK( xCoreID );   /* 先放 ISR_LOCK */
                    portRELEASE_TASK_LOCK( xCoreID );  /* 再放 TASK_LOCK */
                    
                    /* 3. 恢復當前核心的本地硬體中斷開關 */
                    portENABLE_INTERRUPTS();
```

- 為什麼要在解鎖前讀取 `xYieldPendings`？
	- `xYieldPendings` 是全域陣列，如果先釋放鎖、開中斷，再去讀取這個陣列，在多核心高併發環境下，這個狀態可能會被其他核心的排程器干擾。在「關鎖且關中斷」的最後一刻將狀態讀出，是確保原子性（Atomicity）的經典設計。

- **解鎖順序（LIFO）**：
	- **進入**臨界區時：先拿 `TASK_LOCK` $\rightarrow$ 再拿 `ISR_LOCK`
	- **退出**臨界區時：先放 `ISR_LOCK` $\rightarrow$ 再放 `TASK_LOCK`
	- 這種相反的解鎖順序（後進先出）是作業系統中避免多鎖競爭引發死鎖（Deadlock）的鐵律

#### 98.3 結清延後讓權的帳單（Deferred Yield）

```C
/* 4. 如果剛剛在臨界區內部有被標記需要讓權（xYieldCurrentTask != pdFALSE）
                     * 現在我們已經完全退出了臨界區、釋放了所有旋鎖、也開啟了中斷。
                     * 此時可以極度安全地執行任務切換了！ */
                    if( xYieldCurrentTask != pdFALSE )
                    {
                        /* 清除此核心的延後讓權標記，並立刻進行任務切換 */
                        xYieldPendings[ xCoreID ] = pdFALSE; // （一般 portYIELD 內部或之後會重置）
                        portYIELD();
                    }
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }

        traceRETURN_vTaskExitCritical();
    }

#endif /* #if ( configNUMBER_OF_CORES > 1 ) */
```

- **安全讓權時機**：
	- 如果任務在臨界區內執行 `vTaskYieldWithinAPI()`，FreeRTOS 只是把 `xYieldPendings[CoreID]` 設為 `pdTRUE`。 一直等到此處 `vTaskExitCritical()` 執行，將所有鎖都釋放、中斷也重開，確認一切安全後，再呼叫 `portYIELD()`。 這徹底避免了「**拿著多核心 Spinlock 卻跑去換場切換任務**」而導致全系統卡死（Deadlock）的滅頂之災。

### 99. *vTaskExitCriticalFromISR*

當多核心（SMP）環境下的中斷服務程式（ISR）完成了與其他核心共用資料的存取後，它必須呼叫 `vTaskExitCriticalFromISR` 來恢復原本的 CPU 中斷狀態，並釋放跨核心的 `ISR_LOCK`。

有一個非常有意思的技術細節先賣個關子：**為什麼這個函數裡面，完全沒有像 Task 版本那樣去檢查 `xYieldPendings` 或是呼叫 `portYIELD()` 呢？**

#### 99.1 變數宣告、核心識別與安全守衛（Assert）

```C
#if ( configNUMBER_OF_CORES > 1 )

    void vTaskExitCriticalFromISR( UBaseType_t uxSavedInterruptStatus )
    {
        /* 定義用來存取目前 Core ID 的變數 */
        BaseType_t xCoreID;

        traceENTER_vTaskExitCriticalFromISR( uxSavedInterruptStatus );

        /* 確保排程器已經在運作。若還沒啟動，代表此時不可能有跨核心競爭，直接跳過 */
        if( xSchedulerRunning != pdFALSE )
        {
            /* 獲取目前執行此 ISR 的硬體核心 ID */
            xCoreID = ( BaseType_t ) portGET_CORE_ID();

            /* 【防錯機制】安全斷言：確保目前核心的臨界區巢狀計數大於 0。
             * 如果為 0 還想 Exit，代表開發者呼叫 ExitCritical 的次數多於 EnterCritical。 */
            configASSERT( portGET_CRITICAL_NESTING_COUNT( xCoreID ) > 0U );
```

- **`uxSavedInterruptStatus`**：這是呼叫者（ISR）必須傳入的參數，它保存了進入臨界區之前，CPU 當時的中斷屏蔽狀態（Mask）。
- **`configASSERT(...)`**：如果巢狀計數已經是 0 卻還來呼叫 Exit，系統會立刻在這個斷言停住。這能精確抓出 ISR 內部程式碼邏輯不對稱（開鎖次數大於關鎖次數）的嚴重錯誤。

#### 99.2 遞減巢狀計數與解開中斷旋鎖

```C
if( portGET_CRITICAL_NESTING_COUNT( xCoreID ) > 0U )
            {
                /* 遞減目前核心的中斷臨界區巢狀計數器 */
                portDECREMENT_CRITICAL_NESTING_COUNT( xCoreID );

                /* 【關鍵瞬間】只有當計數器歸零（代表最外層臨界區也退出了） */
                if( portGET_CRITICAL_NESTING_COUNT( xCoreID ) == 0U )
                {
                    /* 1. 釋放中斷旋鎖（ISR_LOCK），讓其他核心的 ISR 可以繼續存取全域資源 */
                    portRELEASE_ISR_LOCK( xCoreID );
                    
                    /* 2. 恢復當前核心在進入臨界區前的硬體中斷屏蔽狀態 */
                    portCLEAR_INTERRUPT_MASK_FROM_ISR( uxSavedInterruptStatus );
                }
                else
                {
                    /* 仍在巢狀臨界區內部，什麼都不做，維持關閉中斷與鎖定狀態 */
                    mtCOVERAGE_TEST_MARKER();
                }
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
```

#### 99.3 為什麼 ISR 版本的 Exit 不需要檢查 `xYieldPendings` 且不執行 Yield？


在 Task 版本的 `vTaskExitCritical()` 中，最後會去檢查 `xYieldPendings[xCoreID]`，如果為真就會執行 `portYIELD()`。**為什麼 ISR 版本不這樣做？**

1. **硬體架構限制（CPU ISR Context）**：
	在大多數微處理器（例如 ARM Cortex-M）中，ISR 執行在特權模式或獨立的中斷棧（Interrupt Stack）中。**絕對不允許在中斷處理過程中突然切換任務上下文（Context Switch）**。如果強行切換，會破壞硬體的中斷返回機制（例如 LR 暫存器的 EXC_RETURN 運作），直接導致硬體錯誤（Hard Fault）。
2. **FreeRTOS 的標準 ISR 讓權規範**：
	 FreeRTOS 的設計原則是：**所有的任務切換動作，必須延遲到整個 ISR 徹底執行完畢、即將退出中斷的最後一刻才進行**。 這也就是為什麼我們在寫 ISR 程式碼時，都是這樣寫的：

```C
void My_GPIO_ISR( void )
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    // 1. 進入中斷臨界區
    UBaseType_t state = taskENTER_CRITICAL_FROM_ISR();

    // 2. 呼叫 FromISR API（可能會喚醒高優先權任務，將 xHigherPriorityTaskWoken 設為 pdTRUE）
    xQueueSendFromISR( xQueue, &data, &xHigherPriorityTaskWoken );

    // 3. 退出中斷臨界區（在此處不觸發 Yield）
    taskEXIT_CRITICAL_FROM_ISR( state );

    // 4. 在整個 ISR 的最末端，手動且安全地觸發 Yield
    portYIELD_FROM_ISR( xHigherPriorityTaskWoken ); 
}
```

因此，`vTaskExitCriticalFromISR` 只需要單純專注於「解鎖與開中斷」，任務切換的責任交由 ISR 末端的 `portYIELD_FROM_ISR` 去處理即可。

#### 99.4 Task 臨界區 vs ISR 臨界區

| **特性**     | **任務級臨界區 (vTaskExitCritical)**                   | **中斷級臨界區 (vTaskExitCriticalFromISR)**                             |
| ---------- | ------------------------------------------------ | ----------------------------------------------------------------- |
| **呼叫環境**   | 僅限一般任務（Task Context）                             | 僅限中斷服務程式（ISR Context）                                             |
| **釋放的旋鎖**  | `TASK_LOCK` 與 `ISR_LOCK`                         | **僅 `ISR_LOCK`**                                                  |
| **中斷恢復行為** | 執行 `portENABLE_INTERRUPTS()` 開啟全部中斷              | 執行 `portCLEAR_INTERRUPT_MASK_FROM_ISR()` 恢復先前的 Mask               |
| **任務切換機制** | 會檢查 `xYieldPendings` 並**在退出時當場執行 `portYIELD()`** | **完全不檢查 `xYieldPendings`**，讓權交由 ISR 尾端的 `portYIELD_FROM_ISR()` 執行 |

### 100. *prvWriteNameToBuffer*

當你啟用 `vTaskList()` 或 `vTaskGetRunTimeStats()` 這類偵錯 API，想在 Serial 終端機印出精美的系統狀態表時，核心就是靠這個函數**將任務名稱後面補空白（Padding）**，好讓後續的「任務狀態」、「優先權」、「剩餘 Stack」等欄位能夠**像表格一樣對齊**。

#### 100.1 條件編譯與函數宣告

```C
#if ( configUSE_STATS_FORMATTING_FUNCTIONS > 0 )

    static char * prvWriteNameToBuffer( char * pcBuffer,
                                        const char * pcTaskName )
    {
        /* 迴圈計數器，用來追蹤字元寫入的位置 */
        size_t x;
```

- **`configUSE_STATS_FORMATTING_FUNCTIONS`**：這個巨集必須在 `FreeRTOSConfig.h` 中定義為 `1`，否則編譯器會自動略過此函數，以節省 Flash 空間。
- **`static` 作用域**：這是一個內部輔助函數（Helper Function），只在 `tasks.c` 內部使用，不開放給外部 Application 呼叫。

#### 100.2 原始任務名稱複製

```C
/* 首先，將任務的原始名稱完整複製到目標緩衝區 (pcBuffer) 中 */
        ( void ) strcpy( pcBuffer, pcTaskName );
```

#### 100.3 空白字元填補（Align 關鍵邏輯）

```C
/* 從複製完的字串末端開始，往後填補空白，直到達到預設的最大長度限制 */
        for( x = strlen( pcBuffer ); x < ( size_t ) ( ( size_t ) configMAX_TASK_NAME_LEN - 1U ); x++ )
        {
            pcBuffer[ x ] = ' ';
        }
```

#### 100.4 字串結束與高效指標返回

```C
/* 在最後一個位置寫入 Null 終止符 (0x00)，完成字串格式化 */
        pcBuffer[ x ] = ( char ) 0x00;

        /* 返回當前字串結束點（指向 \0 的記憶體位置） */
        return &( pcBuffer[ x ] );
    }

#endif /* ( configUSE_STATS_FORMATTING_FUNCTIONS > 0 ) */
```

**`return &( pcBuffer[ x ] )` 的效能智慧**： 這是這段 C 程式碼最精妙的細節。 如果這個函數只回傳 `void`，那麼外部呼叫者（如 `vTaskList`）在填完名稱後，為了要在後面繼續寫入「任務狀態（如 'R' 或 'B'）」，就必須自己重新呼叫一次 `strlen(pcBuffer)` 來尋找字串末端。 這裡直接把結束符號 `\0` 的地址（`&(pcBuffer[x])`）丟回去，**呼叫者就能直接從這個地址往下繼續寫資料**，省去了一次 $O(N)$ 的 `strlen` 運算，非常符合嵌入式系統對效能的極致壓榨。

### 101. *vTaskListTasks*

如果你在調試 FreeRTOS 時，曾經在終端機印出 `vTaskList()` 來看過那張精美的「任務狀態清單」，那你今天算是摸到這項功能的心臟了！

`vTaskListTasks()` 是 FreeRTOS 的「系統狀態排版官」。它的運作邏輯非常簡單粗暴：

- 先跟核心要一張「所有任務當前狀態的二進位快照（Snapshot）」。
    
- 透過動態記憶體配置（Malloc）一塊暫存區來放這些資料。
    
- 一個個讀取任務，將狀態轉換成對應的字元（如 'R'、'B'、'S'）。
    
- 調用我們上一堂課分析的 `prvWriteNameToBuffer()` 來幫任務名稱補空白對齊。
    
- 使用 C 標準庫的 `snprintf()` 將優先權、剩餘堆疊、任務 ID（以及多核心下的 Core Affinity）格式化成一列文字，填入你給它的 Buffer 裡。

#### 101.1 條件編譯、函數宣告與緩衝區初始化

這個區塊做好了最基本的準備，首先將傳入的 `pcWriteBuffer` 清空，確保它不會夾帶舊的垃圾字串。

```C
#if ( ( configUSE_TRACE_FACILITY == 1 ) && ( configUSE_STATS_FORMATTING_FUNCTIONS > 0 ) )

    void vTaskListTasks( char * pcWriteBuffer,
                         size_t uxBufferLength )
    {
        TaskStatus_t * pxTaskStatusArray;
        size_t uxConsumedBufferLength = 0;
        size_t uxCharsWrittenBySnprintf;
        int iSnprintfReturnValue;
        BaseType_t xOutputBufferFull = pdFALSE;
        UBaseType_t uxArraySize, x;
        char cStatus;

        traceENTER_vTaskListTasks( pcWriteBuffer, uxBufferLength );

        /* 1. 先將寫入緩衝區的第一個字元設為 \0，確保初始狀態為空字串 */
        *pcWriteBuffer = ( char ) 0x00;
```

- **`configUSE_TRACE_FACILITY`** 與 **`configUSE_STATS_FORMATTING_FUNCTIONS`**：這兩個巨集必須同時在 `FreeRTOSConfig.h` 開啟，才能編譯這段代碼。
- **安全性防護**：`*pcWriteBuffer = 0x00` 是一個非常安全的防禦習慣，確保萬一後續分配記憶體失敗，呼叫者讀到的也是一個合法的「空字串」，而不是隨機的記憶體髒資料。

#### 101.2 動態快照與堆疊空間申請

在這個階段，核心會統計目前有幾個任務，並當場在 Heap（堆疊）中申請一塊夠大的陣列來存放所有的任務狀態結構體（`TaskStatus_t`）。

```C
/* 2. 快照目前的任務總數。因為在執行期間，其他核心或中斷可能會突然建立/刪除任務 */
        uxArraySize = uxCurrentNumberOfTasks;

        /* 3. 為每一個任務配置一個 TaskStatus_t 結構體空間。
         * 注意：若 configSUPPORT_DYNAMIC_ALLOCATION 設為 0，pvPortMalloc 會直接被編譯成 NULL */
        /* MISRA Ref 11.5.1 [Malloc memory assignment] */
        pxTaskStatusArray = pvPortMalloc( uxCurrentNumberOfTasks * sizeof( TaskStatus_t ) );

        if( pxTaskStatusArray != NULL )
        {
            /* 4. 取得系統當前的二進位二進位快照，將資料塞進剛剛配置的陣列中，並回傳實際取得的任務數量 */
            uxArraySize = uxTaskGetSystemState( pxTaskStatusArray, uxArraySize, NULL );
```

- 為什麼不直接讀取核心鏈結串列？
	- 因為讀取鏈結串列時需要關閉中斷。如果直接在鏈結串列上進行耗時的 `snprintf` 字串轉換，中斷會被關閉太久，系統的即時性會直接崩潰。
	- **FreeRTOS 的解法**：快速關中斷 $\rightarrow$ 拷貝一份二進位快照（`uxTaskGetSystemState`） $\rightarrow$ 開中斷 $\rightarrow$ 慢慢在快照上慢慢做字串轉換。這就是**時間複雜度與即時性**的權衡！

#### 101.3 任務狀態 State 轉換（State Mapping）

這一段利用一個大型的 `switch-case`，將任務的列舉型態（Enum）轉換成我們在終端機上看到的單一英文字元。

```C
/* 5. 開始遍歷快照中的每一個任務 */
            for( x = 0; x < uxArraySize; x++ )
            {
                switch( pxTaskStatusArray[ x ].eCurrentState )
                {
                    case eRunning:
                        cStatus = tskRUNNING_CHAR;    /* 通常是 'X' 或 'R' */
                        break;

                    case eReady:
                        cStatus = tskREADY_CHAR;      /* 通常是 'R' */
                        break;

                    case eBlocked:
                        cStatus = tskBLOCKED_CHAR;    /* 通常是 'B' */
                        break;

                    case eSuspended:
                        cStatus = tskSUSPENDED_CHAR;  /* 通常是 'S' */
                        break;

                    case eDeleted:
                        cStatus = tskDELETED_CHAR;    /* 通常是 'D' */
                        break;

                    case eInvalid: /* Fall through. */
                    default:       
                        cStatus = ( char ) 0x00;      /* 異常防禦 */
                        break;
                }
```

#### 101.4 任務名稱格式化與長度檢查

這一步將任務名稱複製進 Buffer，並自動填滿空白進行對齊。

```C
/* 6. 檢查當前剩餘的緩衝區空間，是否足夠塞入任務名稱最大寬度？ */
                if( ( uxConsumedBufferLength + configMAX_TASK_NAME_LEN ) <= uxBufferLength )
                {
                    /* 7. 調用 prvWriteNameToBuffer 寫入任務名稱，並補齊空白字元使表格對齊 */
                    pcWriteBuffer = prvWriteNameToBuffer( pcWriteBuffer, pxTaskStatusArray[ x ].pcTaskName );
                    
                    /* 累加已被消耗的 Buffer 長度（不計算末端的 \0） */
                    uxConsumedBufferLength = uxConsumedBufferLength + ( configMAX_TASK_NAME_LEN - 1U );
```

- **`prvWriteNameToBuffer` 的聯動**：
	- 這裡就是我們上一堂課分析的函數。它會回傳寫完名稱後「最新的結束位置（Null 終止符地址）」。
	- 外部直接用 `pcWriteBuffer = prvWriteNameToBuffer(...)` 接收，下一步就能直接從這個新地址繼續往後拼接其他狀態數據。

#### 101.5 多核心 vs 單核心排版寫入（snprintf）

這裡有兩個分支：如果是多核心 SMP 系統，會額外多印出一個 `uxCoreAffinityMask`（核心親和性遮罩，代表這個任務被允許在哪顆 CPU 核心上跑）；如果是單核心則不印。

```C
/* 檢查 Buffer 扣掉 \0 後，是否還有至少 1 byte 的空間來寫剩餘數據？ */
                    if( uxConsumedBufferLength < ( uxBufferLength - 1U ) )
                    {
                        /* 8. 依據多核心或單核心編譯設定，選擇格式化字串 */
                        #if ( ( configUSE_CORE_AFFINITY == 1 ) && ( configNUMBER_OF_CORES > 1 ) )
                            /* 多核心版本：多印一欄 0xX (Core Affinity Mask) */
                            iSnprintfReturnValue = snprintf( pcWriteBuffer,
                                                             uxBufferLength - uxConsumedBufferLength,
                                                             "\t%c\t%u\t%u\t%u\t0x%x\r\n",
                                                             cStatus,
                                                             ( unsigned int ) pxTaskStatusArray[ x ].uxCurrentPriority,
                                                             ( unsigned int ) pxTaskStatusArray[ x ].usStackHighWaterMark,
                                                             ( unsigned int ) pxTaskStatusArray[ x ].xTaskNumber,
                                                             ( unsigned int ) pxTaskStatusArray[ x ].uxCoreAffinityMask );
                        #else 
                            /* 單核心版本 */
                            iSnprintfReturnValue = snprintf( pcWriteBuffer,
                                                             uxBufferLength - uxConsumedBufferLength,
                                                             "\t%c\t%u\t%u\t%u\r\n",
                                                             cStatus,
                                                             ( unsigned int ) pxTaskStatusArray[ x ].uxCurrentPriority,
                                                             ( unsigned int ) pxTaskStatusArray[ x ].usStackHighWaterMark,
                                                             ( unsigned int ) pxTaskStatusArray[ x ].xTaskNumber );
                        #endif 

                        /* 9. 安全轉換 snprintf 的返回值，避免溢出或負數，取得本次實際寫入的字元數 */
                        uxCharsWrittenBySnprintf = prvSnprintfReturnValueToCharsWritten( iSnprintfReturnValue, uxBufferLength - uxConsumedBufferLength );

                        /* 更新緩衝區消耗進度與指標指針位置 */
                        uxConsumedBufferLength += uxCharsWrittenBySnprintf;
                        pcWriteBuffer += uxCharsWrittenBySnprintf;
                    }
                    else
                    {
                        xOutputBufferFull = pdTRUE;
                    }
                }
                else
                {
                    xOutputBufferFull = pdTRUE;
                }

                /* 若緩衝區爆滿，立刻中斷迴圈，防止記憶體越界寫入 */
                if( xOutputBufferFull == pdTRUE )
                {
                    break;
                }
            }
```

#### 101.6 記憶體釋放與收尾

動態記憶體配置的鐵律：有借有還，再借不難。格式化完成後，立刻將快照陣列歸還給 Heap。

```C
/* 10. 釋放剛剛動態配置的快照陣列空間 */
            vPortFree( pxTaskStatusArray );
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }

        traceRETURN_vTaskListTasks();
    }

#endif /* ( ( configUSE_TRACE_FACILITY == 1 ) && ( configUSE_STATS_FORMATTING_FUNCTIONS > 0 ) ) */
```

#### 101.7 為什麼生產環境（Production）不建議呼叫此 API？

> [!WARNING]
> ### 🛑 為什麼生產環境（Production）不建議使用 `vTaskListTasks()`？
> 
> 1. **記憶體碎片化（Heap Fragmentation）**：
>    該函數內部調用了 `pvPortMalloc`。在即時系統中，頻繁在執行期進行動態記憶體配置，極容易引發堆疊碎片化，增加分配失敗（Out of Memory）的風險。
> 
> 2. **`snprintf` 程式碼膨脹（Code Bloat）**：
>    `snprintf()` 是 C 標準庫中的大怪獸。在許多輕量級的 MCU 專案中，一旦調用 `snprintf`，連結器（Linker）會強行拉入一堆格式化與浮點數處理代碼，導致 Flash 與 RAM 被瞬間吃掉大半。
> 
> 3. **中斷關閉時間過長**：
>    雖然它將字串轉換移到了臨界區外，但獲取二進位快照 `uxTaskGetSystemState()` 時依然需要關閉中斷。如果系統中有數十個任務，這一次快照的操作時間將會拉長，損害系統的即時響應。
> 
> **💡 生產環境最佳替代方案**：
> 如果系統需要監控任務狀態，應直接手動調用 `uxTaskGetSystemState()` 獲取二進位數據。將原始數據透過 CAN 或 UDP 直接送回上位機，讓上位機去負責格式化排版，把寶貴的 MCU 算力留給即時任務！

### 102. *vTaskGetRunTimeStatistics*

這個函數是 FreeRTOS 的「CPU 消耗診斷官」。

當你要分析「到底是哪一個任務把 CPU 榨乾」或者「各任務的 CPU 使用率百分比（CPU Usage %）」時，核心就是靠這個 `vTaskGetRunTimeStatistics()` 來幫你產生一張漂亮的 CPU 佔用統計表。

與我們上一節分析的 `vTaskListTasks()` 非常相似，它同樣利用快照與 `snprintf()` 進行字串格式化，但它內部多了一套**精妙的純整數百分比運算**，完全避免了在微控制器（MCU）中極度昂貴的浮點數（Floating-point）運算。

#### 102.1 條件編譯、宣告與緩衝區初使化

這個區塊做了基本的環境檢查，並將接收格式化字串的 `pcWriteBuffer` 先清空為空字串。

```C
#if ( ( configGENERATE_RUN_TIME_STATS == 1 ) && ( configUSE_STATS_FORMATTING_FUNCTIONS > 0 ) && ( configUSE_TRACE_FACILITY == 1 ) )

    void vTaskGetRunTimeStatistics( char * pcWriteBuffer,
                                    size_t uxBufferLength )
    {
        TaskStatus_t * pxTaskStatusArray;
        size_t uxConsumedBufferLength = 0;
        size_t uxCharsWrittenBySnprintf;
        int iSnprintfReturnValue;
        BaseType_t xOutputBufferFull = pdFALSE;
        UBaseType_t uxArraySize, x;
        configRUN_TIME_COUNTER_TYPE ulTotalTime = 0;
        configRUN_TIME_COUNTER_TYPE ulStatsAsPercentage;

        traceENTER_vTaskGetRunTimeStatistics( pcWriteBuffer, uxBufferLength );

        /* 1. 先將寫入緩衝區的第一個字元設為 \0，確保初始狀態為空字串 */
        *pcWriteBuffer = ( char ) 0x00;
```

- **關鍵巨集 `configGENERATE_RUN_TIME_STATS`**：必須在 `FreeRTOSConfig.h` 中設為 `1`。此外，你還必須提供一個高精度的硬體定時器（通常比系統的 Tick 慢 10~20 倍）來作為 `portCONFIGURE_TIMER_FOR_RUN_TIME_STATS()` 和 `portGET_RUN_TIME_COUNTER_VALUE()`。
- **`configRUN_TIME_COUNTER_TYPE`**：這是計數器的型別，依據硬體與設定，可能是 32-bit（`uint32_t`）或 64-bit（`uint64_t`）。

#### 102.2 動態記憶體分配與獲取系統快照

系統會動態分配一塊記憶體來儲存所有任務狀態的二進位快照。

```C
/* 2. 快照目前的任務總數 */
        uxArraySize = uxCurrentNumberOfTasks;

        /* 3. 配置臨時陣列，用來存放每個任務的狀態資訊 */
        /* MISRA Ref 11.5.1 [Malloc memory assignment] */
        pxTaskStatusArray = pvPortMalloc( uxCurrentNumberOfTasks * sizeof( TaskStatus_t ) );

        if( pxTaskStatusArray != NULL )
        {
            /* 4. 取得系統二進位快照。
             * 注意：第三個參數傳入 &ulTotalTime，核心會順便把「系統啟動以來的總運行時間」寫入此變數 */
            uxArraySize = uxTaskGetSystemState( pxTaskStatusArray, uxArraySize, &ulTotalTime );
```

- **`ulTotalTime` 的獲取**：這行與 `vTaskList` 最關鍵的分水嶺，就在於傳入了 `&ulTotalTime`。這個時間是用來作為後續計算百分比（分母）的基準線。

#### 102.3 防禦零除錯（Divide by Zero）與百分比整數化預處理

這是 FreeRTOS 在嵌入式系統中實現「零浮點數百分比運算」的最精彩一手！

```C
/* 5. 關鍵整數運算：先將總時間除以 100 */
            ulTotalTime /= ( ( configRUN_TIME_COUNTER_TYPE ) 100U );

            /* 6. 防禦性檢查：確保總運行時間大於 0（避免除以零的 CPU 崩潰） */
            if( ulTotalTime > 0U )
            {
```

通常我們要計算百分比，公式是：

$$\text{Percentage} = \frac{\text{TaskRunTime}}{\text{TotalTime}} \times 100$$

如果直接在微控制器上寫這行，會因為小數點而被迫拉入**昂貴的浮點數庫（Floating-point library）**，或者因為先乘以 100 導致 32 位元整數發生**溢位（Overflow）**。

- **FreeRTOS 的拆解法**：
    
    先將分母 `TotalTime` 除以 100，即：`ulTotalTime /= 100`。
    
    後面計算每個任務時，直接執行：
    
    $$\text{Percentage} = \frac{\text{TaskRunTime}}{\text{TotalTime}_{/100}}$$
    
    如此一來，**完全使用純整數除法，不帶任何一個浮點數，同時保證數值絕不溢位**！

#### 102.4 遍歷任務、名稱排版與緩衝區溢位檢查

遍歷所有任務，並將任務名稱後補空白對齊，隨時監控緩衝區剩餘空間。

```C
/* 7. 開始迭代快照中的所有任務 */
                for( x = 0; x < uxArraySize; x++ )
                {
                    /* 8. 計算該任務的 CPU 使用率百分比（整數除法，無條件捨去） */
                    ulStatsAsPercentage = pxTaskStatusArray[ x ].ulRunTimeCounter / ulTotalTime;

                    /* 9. 檢查當前剩餘空間，是否足夠塞入下一個任務名稱的最大寬度？ */
                    if( ( uxConsumedBufferLength + configMAX_TASK_NAME_LEN ) <= uxBufferLength )
                    {
                        /* 10. 調用 prvWriteNameToBuffer 寫入任務名稱並補齊空白 */
                        pcWriteBuffer = prvWriteNameToBuffer( pcWriteBuffer, pxTaskStatusArray[ x ].pcTaskName );
                        uxConsumedBufferLength = uxConsumedBufferLength + ( configMAX_TASK_NAME_LEN - 1U );

                        /* 檢查 Buffer 扣掉 \0 後，是否還有空間來寫統計數據 */
                        if( uxConsumedBufferLength < ( uxBufferLength - 1U ) )
                        {
```

#### 102.5 格式化字串寫入（snprintf 分支處理）

由於不同的編譯器與晶片架構對 64-bit 或是長整型（`long`）的支援不同，這裡利用巨集和 `if-else` 分成了多個 `snprintf` 寫入分支：

```C
/* 11. 如果計算出來的百分比大於 0% */
                            if( ulStatsAsPercentage > 0U )
                            {
                                #ifdef portLU_PRINTF_SPECIFIER_REQUIRED
                                {
                                    /* 如果硬體平台要求 %lu 格式（例如 64-bit 計數器） */
                                    iSnprintfReturnValue = snprintf( pcWriteBuffer,
                                                                     uxBufferLength - uxConsumedBufferLength,
                                                                     "\t%lu\t\t%lu%%\r\n",
                                                                     pxTaskStatusArray[ x ].ulRunTimeCounter,
                                                                     ulStatsAsPercentage );
                                }
                                #else
                                {
                                    /* 否則使用標準的 %u 格式以縮減庫體積 */
                                    iSnprintfReturnValue = snprintf( pcWriteBuffer,
                                                                     uxBufferLength - uxConsumedBufferLength,
                                                                     "\t%u\t\t%u%%\r\n",
                                                                     ( unsigned int ) pxTaskStatusArray[ x ].ulRunTimeCounter,
                                                                     ( unsigned int ) ulStatsAsPercentage );
                                }
                                #endif 
                            }
                            /* 12. 如果百分比等於 0%，代表該任務消耗小於 1% 的 CPU */
                            else
                            {
                                #ifdef portLU_PRINTF_SPECIFIER_REQUIRED
                                {
                                    iSnprintfReturnValue = snprintf( pcWriteBuffer,
                                                                     uxBufferLength - uxConsumedBufferLength,
                                                                     "\t%lu\t\t<1%%\r\n",
                                                                     pxTaskStatusArray[ x ].ulRunTimeCounter );
                                }
                                #else
                                {
                                    iSnprintfReturnValue = snprintf( pcWriteBuffer,
                                                                     uxBufferLength - uxConsumedBufferLength,
                                                                     "\t%u\t\t<1%%\r\n",
                                                                     ( unsigned int ) pxTaskStatusArray[ x ].ulRunTimeCounter );
                                }
                                #endif 
                            }

                            /* 13. 計算並累加實際寫入字元數，指針向下推進 */
                            uxCharsWrittenBySnprintf = prvSnprintfReturnValueToCharsWritten( iSnprintfReturnValue, uxBufferLength - uxConsumedBufferLength );
                            uxConsumedBufferLength += uxCharsWrittenBySnprintf;
                            pcWriteBuffer += uxCharsWrittenBySnprintf;
                        }
                        else
                        {
                            xOutputBufferFull = pdTRUE;
                        }
                    }
                    else
                    {
                        xOutputBufferFull = pdTRUE;
                    }

                    /* 緩衝區爆滿，提早退出 */
                    if( xOutputBufferFull == pdTRUE )
                    {
                        break;
                    }
                }
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
```

- **`portLU_PRINTF_SPECIFIER_REQUIRED`**：在某些 8-bit 或 16-bit MCU 上，`int` 是 16 位的，此時必須用 `%lu` 才能正確列印出 32 位的時間計數器。如果是 32-bit MCU，`int` 與 `long` 同寬，可以直接轉成 `unsigned int` 配合 `%u` 來節約 printf 庫檔案的空間。
- **`<1%` 的貼心排版**：如果任務執行時間很短（不為 0 但不到總時間的 1%），因為整數除法會無條件捨去成 `0`，FreeRTOS 貼心地將其排版成 `<1%`，這能讓工程師一眼看出「這個任務有在跑，只是吃極少資源」，而不是誤以為任務死掉了。

#### 102.6 釋放暫存空間與收尾

最後，將動態申請的快照陣列還給系統，避免記憶體洩漏。

```C
/* 14. 釋放動態快照陣列 */
            vPortFree( pxTaskStatusArray );
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }

        traceRETURN_vTaskGetRunTimeStatistics();
    }

#endif /* ( ( configGENERATE_RUN_TIME_STATS == 1 ) && ( configUSE_STATS_FORMATTING_FUNCTIONS > 0 ) ) */
```

### 103. *uxTaskResetEventItemValue*

這個函數 `uxTaskResetEventItemValue` 是 FreeRTOS 核心中非常精妙的「空間管理大師」。

在資源極度珍貴的嵌入式微控制器中，FreeRTOS 為了不在任務控制塊（TCB, Task Control Block）中額外浪費 4 個位元組（Bytes）來儲存「喚醒事件狀態」，它玩了一個「欄位借用（Hijacking）」的魔術：

> 💡 **核心設計內幕** 當一個任務因為等待「事件組（Event Groups）」而進入阻塞時，FreeRTOS 會**暫時借用**任務的 `xEventListItem` 的 Value 欄位，把任務正在等待的「事件位元（Event Bits）」硬塞進去。
> 
> 當任務被喚醒時，就會呼叫這個函數：
> 
> 1. **取出**當初臨時借放的事件位元值（作為回傳值傳給 API）。
>     
> 2. **還原（Reset）**該欄位為預設的「優先權對齊值」，好讓該任務下次去排隊隊列（Queue）或信号量（Semaphore）時，能正確地按優先權順序排隊。

#### 103.1 取出「被借用」的事件值

這一步是把之前臨時塞進 `xEventListItem` 的資料拔出來。

```C
/* 1. 讀取當前運行中任務 (pxCurrentTCB) 的事件列表項 (Event List Item) 的當前數值 */
    uxReturn = listGET_LIST_ITEM_VALUE( &( pxCurrentTCB->xEventListItem ) );
```

**欄位借用（Data Hijacking）**：
- 在等待 Queue 或 Semaphore 時，這個 Value 放的是用來排序的優先權。
- 在等待 Event Group 時，這個 Value 被暫時改寫成「任務等待的事件 Trigger Bits」（例如 `0x00000005`）。
- 這裡透過 `listGET_LIST_ITEM_VALUE` 順利把這個事件值讀取出來，存入 `uxReturn` 中。

### 104. *pvTaskIncrementMutexHeldCount*

這個函數 `pvTaskIncrementMutexHeldCount` 是 FreeRTOS 互斥鎖（Mutex）機制中非常關鍵的「記帳員」。

在嵌入式系統中，Mutex 與普通的二元訊號量（Binary Semaphore）最大的不同，在於 Mutex 支援「優先權繼承（Priority Inheritance）」機制，用來防止惡名昭彰的「優先權反轉（Priority Inversion）」問題。

為了正確實作優先權繼承，FreeRTOS 必須精確追蹤「當前任務到底手握幾個 Mutex」。只有當任務把手中「所有」的 Mutex 都釋放掉（計數歸零）時，它才能安全地恢復到自己原本的真實優先權。而這個函數，就是在任務成功取得（Take）一個 Mutex 時，用來把計數器加 1 的核心幕後推手。

### 105. *ulTaskGenericNotifyTake*

在 FreeRTOS 中，任務通知（Task Notifications）被譽為「輕量級的訊號量/事件組」。它的速度比傳統的 Semaphore 快了將近 45%，且完全不需要額外動態配置 IPC 結構體的記憶體，因為它的數據就直接藏在每個任務的 TCB（任務控制塊）內部。

`ulTaskGenericNotifyTake()` 就是任務通知機制中，用來「獲取（Take）通知」的核心實現。它的行為模式非常像 **Semaphore Take**：

- 如果通知值大於 0，它會立刻扣減數值（或清零）並返回。
    
- 如果通知值為 0，它會根據你指定的逾時時間（Timeout），優雅地將自己掛起並進入阻塞狀態。

在 FreeRTOS 的世界裡，你可以把 `ulTaskGenericNotifyTake` 想像成一個任務對系統說：

> **「我現在要在這裡等通知。如果我的專案信箱裡有信，我就把信拿走並開始工作；如果信箱是空的，那我就先躺平睡覺，等到有信或者時間到了再叫醒我。」**

在 FreeRTOS 中，**每一個任務（Task）的桌上，都內建了一個專屬的「計數信箱」**（這就是 `ulNotifiedValue`）。

當這個任務執行 `ulTaskGenericNotifyTake()` 時，就是它**走去檢查信箱**的時候：

情境 A：信箱裡一封信都沒有（計數器 = 0）

1. 任務走過去一看，空的。
    
2. 任務說：「那我願意在這裡等 500 毫秒（`xTicksToWait`）。」
    
3. 系統就會讓這個任務**立刻進入睡眠（Blocked 狀態）**，完全不佔用 CPU 算力。
    
4. **被喚醒的兩種可能**：
    
    - 突然，硬體中斷（例如有人按了按鈕）「丟了一封信」進來，任務會**立刻醒來**。
        
    - 如果 500 毫秒過去了還是沒信，任務也會醒來，但會一臉怨念地發現信箱還是空的（回傳 0）。
        

情境 B：信箱裡本來就有信（計數器 > 0）

1. 任務走過去一看，發現裡面有 3 封信。
    
2. 任務**完全不用睡覺**，它會立刻執行「拿信（Take）」的動作，並直接往下執行工作。

#### 105.1 暫停排程器與雙重臨界區（防範中斷賽跑）

這是整段內核代碼中最精妙、也最考驗作業系統功力的「雙鎖機制」。當任務發現沒有通知（Value == 0）且願意等待時，它會啟動防禦。

```C
/* 2. 快速判斷：如果目前沒有通知，且呼叫者願意等待（Ticks > 0） */
        if( ( pxCurrentTCB->ulNotifiedValue[ uxIndexToWaitOn ] == 0U ) && ( xTicksToWait > ( TickType_t ) 0 ) )
        {
            /* 3. 暫停排程器（vTaskSuspendAll）：因為接下來的「將任務加入延時鏈結串列」
             * 是一個非確定性（Non-deterministic）的操作。我們不希望排程器此時切換任務。 */
            vTaskSuspendAll();
            {
                /* 4. 進入 CPU 臨界區（關中斷）：確保原子性（Atomicity）。
                 * 防止我們在檢查狀態的瞬間，被外部中斷服務程式（ISR）搶先送入通知而導致信號丟失。 */
                taskENTER_CRITICAL();
                {
                    /* 雙重檢查：再次確認這段期間內，通知值是否依然為 0 */
                    if( pxCurrentTCB->ulNotifiedValue[ uxIndexToWaitOn ] == 0U )
                    {
                        /* 5. 將任務狀態標記為：正在等待通知 */
                        pxCurrentTCB->ucNotifyState[ uxIndexToWaitOn ] = taskWAITING_NOTIFICATION;

                        /* 標記此任務需要進入阻塞 */
                        xShouldBlock = pdTRUE;
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }
                }
                taskEXIT_CRITICAL();
```

為什麼要同時用 `vTaskSuspendAll` 與 `taskENTER_CRITICAL`？

- **`vTaskSuspendAll()`（暫停排程器）**：
	- **作用**：禁止任務切換，但**允許硬體中斷正常響應**。
	- **原因**：將當前任務移入「阻塞延遲鏈結串列」（`prvAddCurrentTaskToDelayedList`）需要搜尋鏈結串列，耗時較長。如果在關閉中斷的情況下做這件事，會導致系統的**中斷延遲時間（Interrupt Latency）變長**。
- **`taskENTER_CRITICAL()`（關閉中斷）**：
	- **作用**：在極短的時間內徹底鎖死中斷。
	- **原因**：如果只暫停排程器而不關中斷，萬一在我們把 `ucNotifyState` 設為 `taskWAITING_NOTIFICATION` 的前一刻，一個硬體中斷（ISR）呼叫了 `xTaskNotifyFromISR()`。
	- 此時，中斷會在我們還沒準備好時把通知送達。等中斷結束，我們又傻傻地把狀態改成 `taskWAITING_NOTIFICATION` 並投入睡眠。這會導致**這個剛剛送達的通知直接「被遺忘（Lost Notification）」**，任務將無期限地死鎖！

#### 105.2 任務入隊掛起與強制上下文切換（Yield）

當安全確認完畢後，釋放臨界區，並正式將任務加入等待佇列，最後引發一次任務調度。

```C
/* 6. 排程器依然暫停，但臨界區已解開（中斷已重開）。
                 * 現在可以安全且不影響中斷響應地，將自己塞入等待鏈結串列 */
                if( xShouldBlock == pdTRUE )
                {
                    traceTASK_NOTIFY_TAKE_BLOCK( uxIndexToWaitOn );
                    
                    /* 7. 將當前任務移出 Ready 鏈結串列，並塞入 Delayed/Blocked 鏈結串列中等待 Ticks 逾時 */
                    prvAddCurrentTaskToDelayedList( xTicksToWait, pdTRUE );
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
            /* 8. 恢復排程器。xTaskResumeAll 會回傳在暫停期間是否發生了任務搶佔 */
            xAlreadyYielded = xTaskResumeAll();

            /* 9. 如果任務真的進入了阻塞（xShouldBlock），且恢復排程器時還沒引發排程 */
            if( ( xShouldBlock == pdTRUE ) && ( xAlreadyYielded == pdFALSE ) )
            {
                /* 強制進行上下文切換（Context Switch），讓出 CPU 給其他 Ready 任務 */
                taskYIELD_WITHIN_API();
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
```

#### 105.3 數值消費與模擬二元/計數訊號量

當任務被喚醒（不論是因為收到通知，還是逾時醒來），它會來到這裡。這段代碼展現了任務通知如何透過一個參數，同時模擬出「二元訊號量」與「計數訊號量」的行為。

```C
/* 10. 再次進入臨界區，安全地讀取與修改通知值 */
        taskENTER_CRITICAL();
        {
            traceTASK_NOTIFY_TAKE( uxIndexToWaitOn );
            
            /* 讀取當前的通知值，準備作為回傳值 */
            ulReturn = pxCurrentTCB->ulNotifiedValue[ uxIndexToWaitOn ];

            if( ulReturn != 0U )
            {
                /* 11. 核心分水嶺：
                 * 如果 xClearCountOnExit == pdTRUE（模擬二元訊號量）：
                 *     直接將通知值「清零（Reset to 0）」。
                 * 如果 xClearCountOnExit == pdFALSE（模擬計數訊號量）：
                 *     將通知值「減 1（Decrement by 1）」。 */
                if( xClearCountOnExit != pdFALSE )
                {
                    pxCurrentTCB->ulNotifiedValue[ uxIndexToWaitOn ] = ( uint32_t ) 0U;
                }
                else
                {
                    pxCurrentTCB->ulNotifiedValue[ uxIndexToWaitOn ] = ulReturn - ( uint32_t ) 1;
                }
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }

            /* 12. 狀態還原：將通知狀態重設為未等待 */
            pxCurrentTCB->ucNotifyState[ uxIndexToWaitOn ] = taskNOT_WAITING_NOTIFICATION;
        }
        taskEXIT_CRITICAL();

        traceRETURN_ulTaskGenericNotifyTake( ulReturn );

        /* 回傳「消費前」的原始通知值 */
        return ulReturn;
    }

#endif /* configUSE_TASK_NOTIFICATIONS */
```

#### 105.4 實際開發時，什麼時候會寫到這行扣？

最經典的例子就是 **「按鍵控制 LED 燈」** 或 **「感測器資料處理」**：

- **按鍵任務（負責處理邏輯）**： 日常狀態下，它就在瘋狂執行 `ulTaskGenericNotifyTake(..., pdTRUE, portMAX_DELAY)`。因為沒有按鍵按下，它就**永遠卡在這裡睡覺**，不消耗任何微控制器的電力。
    
- **外部中斷（ISR，負責偵測硬體）**： 當使用者的手指真的按下了按鈕，晶片觸發硬體中斷。中斷服務程式會呼叫 `xTaskNotifyFromISR()`，往按鍵任務的信箱裡「丟一封信」。
    
- **結果**： 按鍵任務瞬間被系統喚醒，執行接下來的 `printf("按鍵被按下了！\n");`，處理完之後，繞回迴圈開頭，又呼叫 `ulTaskGenericNotifyTake` 繼續睡覺。

這個函數的精髓就在於：**不用讓 CPU 寫 `while(1)` 去瘋狂輪詢（Polling）檢查狀態，而是用極低的功耗、極快的速度實現「事件驅動」！**

### 106. *xTaskGenericNotifyWait*

上一篇我們講到的 `ulTaskGenericNotifyTake()` 是用來模擬「訊號量（Semaphore）」，專門處理「數字/數量」的增加與減少。

而這篇要介紹的 `xTaskGenericNotifyWait()` 則是任務通知機制中，用來模擬「事件組（Event Group）/ 位元旗標（Bitmask）」的終極 API！

簡單一句話搞懂 `NotifyWait` 與 `NotifyTake` 的本質差異

- **`NotifyTake` (計數型)**：專注於「數量」。適合做按鍵計數、資源計數、簡單的通知。
    
- **`NotifyWait` (位元型)**：專注於「32 位的 Bitmask 旗標」。你可以用不同的 Bit 代表不同的事件（例如 `Bit 0` 表示 UART 收到資料、`Bit 1` 表示 DMA 完成、`Bit 2` 表示按下按鈕），並靈活地在**進入等待前**與**離開函數後**清除特定的 Bit。

在 FreeRTOS 的原始碼中，每個任務控制塊（TCB）內部都有一個名為 `ucNotifyState` 的欄位，用來紀錄該任務「目前的通知狀態」。

你提到的清單中重複寫了 `taskNOTIFICATION_RECEIVED`，FreeRTOS 底層定義的 **3 種完整狀態 (Enum)** 分別為：

1. **`taskNOT_WAITING_NOTIFICATION`**（未等待通知）
    
2. **`taskWAITING_NOTIFICATION`**（正在等待通知）
    
3. **`taskNOTIFICATION_RECEIVED`**（已收到通知 / 通知掛起）

 `taskNOT_WAITING_NOTIFICATION`（預設 / 常規狀態）

- **字面意思**：任務**沒有**在等通知。
    
- **代表狀態**：
    
    - **剛建立時**：任務剛被創建時的初始狀態。
        
    - **正在忙碌**：任務正在執行自己的 C 語言代碼，沒有呼叫任何 `NotifyWait` 或 `NotifyTake`。
        
    - **剛處理完**：任務已經呼叫過 `NotifyWait` 或 `NotifyTake`，並且已經把通知讀取完畢、離開了函數。
        
- **CPU 行為**：任務正常執行，沒有因為等待通知而進入阻塞（Blocked）。
    

`taskWAITING_NOTIFICATION`（阻塞等待中）

- **字面意思**：任務正在「躺平等通知」。
    
- **代表狀態**：
    
    - 任務呼叫了 `ulTaskNotifyTake()` 或 `xTaskNotifyWait()`，並且設定了等待時間（`xTicksToWait > 0`）。
        
    - 呼叫的當下，系統發現「沒有未處理的通知」。
        
    - 於是系統將 `ucNotifyState` 標記為 `taskWAITING_NOTIFICATION`，並把該任務移入延遲/阻塞鏈結串列（Blocked List），**讓出 CPU 控制權**。
        
- **CPU 行為**：任務進入睡眠，消耗 0% CPU 算力，直到被中斷（ISR）或其他任務喚醒，或是等待逾時（Timeout）。
    

`taskNOTIFICATION_RECEIVED`（有新通知 / 通知掛起）

- **字面意思**：信箱裡有「未讀新信件」！
    
- **代表狀態**：
    
    - **情況 A（喚醒睡眠中任務）**：任務原本在 `taskWAITING_NOTIFICATION` 狀態睡覺，此時一個 ISR 或另一個任務呼叫了 `xTaskNotify()` 發送通知。系統會立刻將狀態改為 `taskNOTIFICATION_RECEIVED`，並把任務**拉回 Ready List 喚醒**。
        
    - **情況 B（先發後讀 / 預存通知）**：任務還沒呼叫 `NotifyWait`，但別人已經先發了通知給它。此時狀態標記為 `taskNOTIFICATION_RECEIVED`。當任務**後續**呼叫 `NotifyWait` 時，會發現燈號已經亮起，因而**完全不需阻塞睡眠，立刻讀取並直接返回**。

#### 106.1 條件編譯、參數說明與斷言檢查

```C
#if ( configUSE_TASK_NOTIFICATIONS == 1 )

    BaseType_t xTaskGenericNotifyWait( UBaseType_t uxIndexToWaitOn,
                                       uint32_t ulBitsToClearOnEntry,
                                       uint32_t ulBitsToClearOnExit,
                                       uint32_t * pulNotificationValue,
                                       TickType_t xTicksToWait )
    {
        BaseType_t xReturn, xAlreadyYielded, xShouldBlock = pdFALSE;

        /* 進入函數的偵錯追蹤點 (Trace Hook) */
        traceENTER_xTaskGenericNotifyWait( uxIndexToWaitOn, ulBitsToClearOnEntry, ulBitsToClearOnExit, pulNotificationValue, xTicksToWait );

        /* 1. 安全斷言：確保通知陣列索引未越界 */
        configASSERT( uxIndexToWaitOn < configTASK_NOTIFICATION_ARRAY_ENTRIES );
```

- **`ulBitsToClearOnEntry`**：進入（Entry）函數且尚未進入睡眠前，要將通知值中的哪些 Bit 清零（`0`）。
    
- **`ulBitsToClearOnExit`**：離開（Exit）函數前，要將通知值中的哪些 Bit 清零（`0`）。
    
- **`pulNotificationValue`**：用來接收「被 `ulBitsToClearOnExit` 清除**前**」的原始 32 位元通知值。
    
- **`xTicksToWait`**：逾時時間。

#### 106.2 雙鎖保護與「進入時位元清零（Entry Clearing）」

如果目前沒有未處理的通知（狀態不是 `taskNOTIFICATION_RECEIVED`），任務會準備進入睡眠。在進入睡眠前，它會先執行 `ulBitsToClearOnEntry` 的清零動作。

```C
/* 2. 檢查：如果尚未收到通知，且呼叫者願意等待（Ticks > 0） */
        if( ( pxCurrentTCB->ucNotifyState[ uxIndexToWaitOn ] != taskNOTIFICATION_RECEIVED ) && ( xTicksToWait > ( TickType_t ) 0 ) )
        {
            /* 3. 暫停排程器：保護後續鏈結串列的操作 */
            vTaskSuspendAll();
            {
                /* 4. 進入臨界區（關閉中斷）：保證狀態檢查與清零動作的原子性 (Atomicity) */
                taskENTER_CRITICAL();
                {
                    /* 雙重檢查：確認在這期間沒有被 ISR 搶先寫入通知 */
                    if( pxCurrentTCB->ucNotifyState[ uxIndexToWaitOn ] != taskNOTIFICATION_RECEIVED )
                    {
                        /* 5. 核心動作 [Entry Clear]：
                         * 在正式掛起睡眠前，將指定的 Bit 清零！
                         * 透過按位與非操作（&= ~ulBitsToClearOnEntry）完成。 */
                        pxCurrentTCB->ulNotifiedValue[ uxIndexToWaitOn ] &= ~ulBitsToClearOnEntry;

                        /* 6. 將任務狀態改為：正在等待通知 */
                        pxCurrentTCB->ucNotifyState[ uxIndexToWaitOn ] = taskWAITING_NOTIFICATION;

                        /* 標記需要進入阻塞 */
                        xShouldBlock = pdTRUE;
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }
                }
                taskEXIT_CRITICAL();
```

為什麼需要 `ulBitsToClearOnEntry`？

假設你只關心「**接下來**發生的新事件」，而不關心「**過去**殘留的舊 Bit 狀態」。你就可以傳入 `ulBitsToClearOnEntry = 0xFFFFFFFF`，在開始等待前，直接把過去累積的舊旗標一次塗銷抹乾淨！

#### 106.3 任務加入阻塞鏈結串列與上下文切換

此區塊將排程器解除並引發 CPU 任務切換，讓當前任務安心睡眠。

```C
/* 7. 臨界區已解開，中斷已恢復，現在安全地將任務塞入 Delayed 鏈結串列 */
                if( xShouldBlock == pdTRUE )
                {
                    traceTASK_NOTIFY_WAIT_BLOCK( uxIndexToWaitOn );
                    prvAddCurrentTaskToDelayedList( xTicksToWait, pdTRUE );
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
            /* 8. 恢復排程器 */
            xAlreadyYielded = xTaskResumeAll();

            /* 9. 強制進行上下文切換，將 CPU 控制權讓給其他 Ready 任務 */
            if( ( xShouldBlock == pdTRUE ) && ( xAlreadyYielded == pdFALSE ) )
            {
                taskYIELD_WITHIN_API();
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
```

#### 106.4 喚醒處理、輸出結果與「離開時位元清零（Exit Clearing）」

當任務醒來（不論是收到通知醒來，還是逾時醒來），它會在這裡提取通知值，並執行 `ulBitsToClearOnExit`。

```C
/* 10. 再次進入臨界區，處理收尾與數值讀取 */
        taskENTER_CRITICAL();
        {
            traceTASK_NOTIFY_WAIT( uxIndexToWaitOn );

            /* 11. 如果呼叫者傳入了接收指針，將當前 32 位元的通知值複製一份出去。
             * 注意：此時輸出的數值，是「尚未」經過 ulBitsToClearOnExit 清除的完整數值！ */
            if( pulNotificationValue != NULL )
            {
                *pulNotificationValue = pxCurrentTCB->ulNotifiedValue[ uxIndexToWaitOn ];
            }

            /* 12. 檢查喚醒的原因：是收到通知醒來，還是逾時醒來？ */
            if( pxCurrentTCB->ucNotifyState[ uxIndexToWaitOn ] != taskNOTIFICATION_RECEIVED )
            {
                /* 沒收到通知（逾時喚醒 Timeout） */
                xReturn = pdFALSE;
            }
            else
            {
                /* 13. 成功收到通知！
                 * 核心動作 [Exit Clear]：根據參數，將對應的 Bit 清零（例如清空已處理完的事件）。 */
                pxCurrentTCB->ulNotifiedValue[ uxIndexToWaitOn ] &= ~ulBitsToClearOnExit;
                xReturn = pdTRUE;
            }

            /* 14. 還原通知狀態為非等待中 */
            pxCurrentTCB->ucNotifyState[ uxIndexToWaitOn ] = taskNOT_WAITING_NOTIFICATION;
        }
        taskEXIT_CRITICAL();

        traceRETURN_xTaskGenericNotifyWait( xReturn );

        /* 15. 回傳 pdTRUE (成功收到通知) 或 pdFALSE (逾時) */
        return xReturn;
    }

#endif /* configUSE_TASK_NOTIFICATIONS */
```

#### 106.5 補充

在實務開發中，我們不會為了「只等一個簡單的按鈕」去呼叫複雜的 `xTaskGenericNotifyWait()`（那用 `ulTaskGenericNotifyTake` 或 Semaphore 就夠了）。

你會**選擇呼叫 `xTaskGenericNotifyWait()` 的核心時機**，通常集中在以下 **4 大高級應用場景**。這也是它作為「事件組（Event Group）與佇列（Queue）替代品」發揮最大功用之處：

什麼時候該呼叫 xTaskGenericNotifyWait？

##### 106.5.1 當你要一個任務「同時等待多種不同的事件 (Bitmask 旗標)」

**代替對象**：`Event Groups`（事件組）

**好處**：不需要額外動態配置 Event Group 的記憶體，省 RAM 且執行速度極快。

實務場景：通訊與系統狀態監控任務

假設你有一個「網路傳輸任務」，它需要處理以下 3 種硬體中斷：

- **Bit 0 (0x01)**：UART 收到一串數據包。
    
- **Bit 1 (0x02)**：DMA 搬移完成。
    
- **Bit 2 (0x04)**：硬體溢位錯誤。

```C
uint32_t ulNotificationValue;

// 呼叫時機：任務進入無限迴圈，等待任何一個 Bit 被寫入
if( xTaskNotifyWait( 0x00,              // 進入前不清除任何 Bit
                     0xFFFFFFFF,        // 離開時把所有 Bit 清零（代表一次處理完）
                     &ulNotificationValue, // 拿到完整的 32 位元事件 Flag
                     portMAX_DELAY ) == pdTRUE )
{
    // 檢查是哪一個 Bit 被觸發了？
    if( ( ulNotificationValue & 0x01 ) != 0 ) {
        /* 處理 UART 數據 */
    }
    if( ( ulNotificationValue & 0x02 ) != 0 ) {
        /* 處理 DMA 搬移 */
    }
    if( ( ulNotificationValue & 0x04 ) != 0 ) {
        /* 處理錯誤重置 */
    }
}
```

##### 106.5.2 當你要「接收一個 32 位的數值或指標 (Pointer)」

**代替對象**：`Queue (長度為 1 的佇列)`

**好處**：不需要建立 Queue 結構體，發送與接收速度提升 45% 以上。

實務場景：ADC 採樣數據或記憶體位址傳遞

當中斷服務程式（ISR）採樣到一個 32 位的數據（或指向數組的記憶體指標），想直接塞給處理任務時：

- **發送端 (ISR)**：`xTaskNotifyFromISR( ..., ulADCValue, eSetValueWithOverwrite, ... )`
    
- **接收端 (Task)**：呼叫 `xTaskNotifyWait()` 把值讀出來！

```C
uint32_t ulReceivedADCValue;

// 呼叫時機：等待 Sensor/ADC 送來一個 32 位元的最新數據
if( xTaskNotifyWait( 0x00,              // 進入不清零
                     0xFFFFFFFF,        // 離開時將通知清零（等待下一次新品）
                     &ulReceivedADCValue, // 傳入變數位址，用來接收數值
                     pdMS_TO_TICKS(100) ) == pdTRUE )
{
    // 此時 ulReceivedADCValue 就是 ISR 傳過來的真實數據（如：4095）
    printf("收到 ADC 採樣數值: %lu\n", ulReceivedADCValue);
}
```

##### 106.5.3 當你在發起新的硬體動作前，想「強制抹除歷史舊紀錄」

> **利用參數**：`ulBitsToClearOnEntry`

實務場景：SPI / I2C 總線異步傳輸

假設你的 SPI 傳輸偶爾會因為干擾而觸發異常，留下了舊的「完成旗標（Bit 0）」。 在下一次啟動新的 SPI 傳輸前，你希望**確保絕對不會被上次殘留的舊訊號誤導**。

```C
// 1. 觸發硬體 SPI 開始傳送
HAL_SPI_Transmit_DMA(&hspi1, txData, 100);

// 2. 呼叫時機：進入等待前，先用 0xFFFFFFFF 把「以前累積的舊 Bit」通通塗銷抹乾淨！
xTaskNotifyWait( 0xFFFFFFFF,        // ulBitsToClearOnEntry: 進入時將舊狀態一筆勾銷
                 0xFFFFFFFF,        // ulBitsToClearExit: 離開時也清空
                 &ulNotificationValue,
                 pdMS_TO_TICKS(500) );
```

##### 106.5.4 當你需要實現「覆蓋式 (Overwrite) 狀態切換」

實務場景：系統狀態機 (State Machine)

假設有一個主控任務，其他任務會隨時切換它的狀態（例如：`1: IDLE`, `2: RUNNING`, `3: ERROR`, `4: SLEEP`）。

發送端使用 `eSetValueWithOverwrite` 隨時更新狀態，而主控任務只需要呼叫 `xTaskNotifyWait`：

```C
uint32_t ulSystemState;

// 呼叫時機：監聽系統目前的最新狀態
if( xTaskNotifyWait( 0, 0xFFFFFFFF, &ulSystemState, portMAX_DELAY ) == pdTRUE )
{
    switch( ulSystemState )
    {
        case SYSTEM_STATE_IDLE:    /* ... */ break;
        case SYSTEM_STATE_RUNNING: /* ... */ break;
        case SYSTEM_STATE_ERROR:   /* ... */ break;
    }
}
```

```C
┌──────────────────────────────────────────────────────────┐
│                  你需要處理什麼樣的資料？                  │
└────────────────────────────┬─────────────────────────────┘
                             │
       ┌─────────────────────┴─────────────────────┐
       ▼                                           ▼
【只是簡單的數量 / 觸發通知】                  【需要 32-bit 資料 / 多旗標】
       │                                           │
       ├─► 想要計數或二元開關                      ├─► 傳遞 32-bit 數值 / 指針
       │   👉 呼叫 `ulTaskNotifyTake()`             │   👉 呼叫 `xTaskNotifyWait()`
       │                                           │
       └─► 想要模擬 Semaphore                    └─► 同時監控多個 Bit (Event Group)
           👉 呼叫 `ulTaskNotifyTake()`                👉 呼叫 `xTaskNotifyWait()`
```

簡短總結：

當你**不需要資料內容**、只要「醒來」時，用 `ulTaskNotifyTake()`； 當你**需要拿到資料（Bit 旗標、數值、指標）**，或者**需要精細控制 Entry/Exit 抹除時機**時，就是呼叫 `xTaskGenericNotifyWait()` 的最佳時刻！

### 107. *xTaskGenericNotify*

`xTaskGenericNotifyWait()` 與 `ulTaskGenericNotifyTake()` 是任務通知機制中的「收件人（Receiver）」**，那麼現在這個 `xTaskGenericNotify()` 就是 FreeRTOS 任務通知體系中最核心、最萬能的**「郵差（Sender）」！

在 FreeRTOS 中，所有高階的發送 API（例如：`xTaskNotify()`、`xTaskNotifyGive()`、`xTaskNotifyAndQuery()`）底層其實**全部都是呼叫這個 `xTaskGenericNotify()` 函數**。

#### 107.1 斷言檢查、舊值備份與進入臨界區

```C
#if ( configUSE_TASK_NOTIFICATIONS == 1 )

    BaseType_t xTaskGenericNotify( TaskHandle_t xTaskToNotify,
                                   UBaseType_t uxIndexToNotify,
                                   uint32_t ulValue,
                                   eNotifyAction eAction,
                                   uint32_t * pulPreviousNotificationValue )
    {
        TCB_t * pxTCB;
        BaseType_t xReturn = pdPASS;
        uint8_t ucOriginalNotifyState;

        traceENTER_xTaskGenericNotify( xTaskToNotify, uxIndexToNotify, ulValue, eAction, pulPreviousNotificationValue );

        /* 1. 安全斷言：確保陣列索引未越界，且目標任務 Handle 不為 NULL */
        configASSERT( uxIndexToNotify < configTASK_NOTIFICATION_ARRAY_ENTRIES );
        configASSERT( xTaskToNotify );
        
        /* 在 FreeRTOS 中，TaskHandle_t 本質上就是指向 TCB 結構體的指標 */
        pxTCB = xTaskToNotify;

        /* 2. 進入臨界區（關閉中斷）：確保修改目標任務 TCB 的過程完全原子化 */
        taskENTER_CRITICAL();
        {
            /* 3. 如果呼叫者想知道「發送前」的舊值，將其導出到指標變數中 */
            if( pulPreviousNotificationValue != NULL )
            {
                *pulPreviousNotificationValue = pxTCB->ulNotifiedValue[ uxIndexToNotify ];
            }

            /* 4. 關鍵備份：紀錄目標任務在「收到這次通知前」的原始狀態。
             * 這將作為後續判斷該任務「是否正在睡眠、需不需要被喚醒」的唯一依據！ */
            ucOriginalNotifyState = pxTCB->ucNotifyState[ uxIndexToNotify ];

            /* 5. 將目標任務的通知狀態直接更新為：已收到通知 (taskNOTIFICATION_RECEIVED) */
            pxTCB->ucNotifyState[ uxIndexToNotify ] = taskNOTIFICATION_RECEIVED;
```

**`ucOriginalNotifyState` 的妙用**：代碼隨後就會把狀態硬寫成 `taskNOTIFICATION_RECEIVED`，所以必須先用 `ucOriginalNotifyState` 記下它發送**前**是睡覺中（`taskWAITING_NOTIFICATION`）還是本來就醒著，否則等一下會無法判斷要不要喚醒它。

#### 107.2 五大發送動作（`eNotifyAction` 核心 Switch-Case）

這段代碼展現了任務通知如何透過 `eAction` 參數，一次模擬「二元訊號量」、「計數訊號量」、「事件組 Bitmask」與「覆蓋/不覆蓋佇列」。

```C
/* 6. 根據傳入的 eAction 策略，對目標任務的 ulNotifiedValue 進行修改 */
            switch( eAction )
            {
                case eSetBits:
                    /* 模擬 Event Group：對指定的 Bit 進行按位或 (Bitwise OR) 操作 */
                    pxTCB->ulNotifiedValue[ uxIndexToNotify ] |= ulValue;
                    break;

                case eIncrement:
                    /* 模擬 Semaphore Give：將通知計數器直接 +1 */
                    ( pxTCB->ulNotifiedValue[ uxIndexToNotify ] )++;
                    break;

                case eSetValueWithOverwrite:
                    /* 模擬信箱覆蓋 (Overwrite)：強制用新值覆蓋舊值 */
                    pxTCB->ulNotifiedValue[ uxIndexToNotify ] = ulValue;
                    break;

                case eSetValueWithoutOverwrite:
                    /* 模擬 Queue 發送 (無覆蓋)：
                     * 只有在目標任務「原本沒有未讀通知」時才寫入；如果原本就有未讀通知，則放棄寫入並回傳 pdFAIL */
                    if( ucOriginalNotifyState != taskNOTIFICATION_RECEIVED )
                    {
                        pxTCB->ulNotifiedValue[ uxIndexToNotify ] = ulValue;
                    }
                    else
                    {
                        /* 目標任務尚有未處理的通知，無法寫入 */
                        xReturn = pdFAIL;
                    }

                    break;

                case eNoAction:
                    /* 只通知、不改值：純粹作為一個「喚醒訊號」使用 */
                    break;

                default:
                    /* 防止傳入非法 Enum 的防禦性斷言 */
                    configASSERT( xTickCount == ( TickType_t ) 0 );
                    break;
            }

            traceTASK_NOTIFY( uxIndexToNotify );
```

#### 107.3 喚醒目標任務與鏈結串列（List）結構調整

當資料寫入完成後，系統檢查目標任務剛才是否在「睡眠（Blocked）」狀態。如果是，就將它從等待清單拉回就緒清單（Ready List）。

```C
/* 7. 檢查：如果目標任務剛才正處於 waiting 狀態（正在睡覺等通知） */
            if( ucOriginalNotifyState == taskWAITING_NOTIFICATION )
            {
                /* 8. 將目標任務從 Blocked / Delayed 鏈結串列中拔除 */
                listREMOVE_ITEM( &( pxTCB->xStateListItem ) );
                
                /* 9. 將目標任務塞入 Ready List（準備好執行清單）中，標誌其已被喚醒 */
                prvAddTaskToReadyList( pxTCB );

                /* 斷言檢查：等待通知的任務絕不應該同時掛在事件鏈結串列上 */
                configASSERT( listLIST_ITEM_CONTAINER( &( pxTCB->xEventListItem ) ) == NULL );
```

#### 107.4 Tickless 低功耗重置與搶佔式排程（Preemption Yield）

最後，處理低功耗模式的定時器重置，並檢查新喚醒的任務優先權是否比自己高。如果是，立刻觸發搶佔（Preemption）！

```C
#if ( configUSE_TICKLESS_IDLE != 0 )
                {
                    /* 10. 如果開啟了 Tickless 低功耗模式：
                     * 目標任務原本可能預計在第 1000 個 Tick 才逾時醒來，但現在第 100 個 Tick 就被我們發通知喚醒了。
                     * 因此必須重置「下一次系統喚醒時間 (xNextTaskUnblockTime)」，避免 CPU 低功耗睡眠時間計算錯誤。 */
                    prvResetNextTaskUnblockTime();
                }
                #endif

                /* 11. 優先權檢查與搶佔觸發：
                 * 檢查被喚醒的任務優先權是否「大於」當前正在執行的任務？
                 * 如果是，且系統開啟了搶佔（Preemption），則立刻請求一次 Task Yield，讓更高優先權的任務馬上執行！ */
                taskYIELD_ANY_CORE_IF_USING_PREEMPTION( pxTCB );
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
        taskEXIT_CRITICAL();

        traceRETURN_xTaskGenericNotify( xReturn );

        /* 12. 回傳 pdPASS (成功) 或 pdFAIL (無覆蓋模式下因已有通知而失敗) */
        return xReturn;
    }

#endif /* configUSE_TASK_NOTIFICATIONS */
```

#### 107.5 taskYIELD_ANY_CORE_IF_USING_PREEMPTION 為啥可以在 CS 內呼叫

在 RTOS 中，直覺上我們常覺得「臨界區（Critical Section, CS）內部不應該做 Task Yield（切換任務）」，因為臨界區的目的就是關閉中斷、保護數據不被搶佔。

但 `taskYIELD_ANY_CORE_IF_USING_PREEMPTION()` 卻**必須且只能在 Critical Section 內部呼叫**。這背後有 FreeRTOS 極其精妙的三大內核設計：

1. 請求與執行的分離：Yield 只是「掛起請求」，不會立刻切換

在 FreeRTOS 中（以典型的 ARM Cortex-M 為例）：
- **`taskENTER_CRITICAL()`**：會將 CPU 中斷屏蔽屏蔽到 `configMAX_SYSCALL_INTERRUPT_PRIORITY`，以保護資料。
- **Yield 的實現**：FreeRTOS 的上下文切換是透過觸發一個低優先權的中斷——**PendSV** 來完成的。
- **PendSV 的優先權**：被設為系統中**最低（Lowest Priority）**。
- 當代碼在 Critical Section 內執行到 Yield 時，系統只是去把 PendSV 的「掛起位元（Pending Bit）」設為 1，**發起切換請求**。

**關鍵點**：因為當前還在 Critical Section 內，CPU 的中斷被屏蔽了，**PendSV 中斷根本進不去**！ 真正的上下文切換（Context Switch）會**延遲到執行完 `taskEXIT_CRITICAL()`、中斷重新解鎖的「下一納秒」才真正觸發**。

2. 狀態檢查的「原子性（Atomicity）」需求

為什麼 Yield 的發起動作**一定要鎖在 Critical Section 裡面**？

在 `xTaskGenericNotify()` 中，我們剛把目標任務 `pxTCB` 放入 Ready List。接著我們需要做判斷：

> 「這個剛被喚醒的 `pxTCB`，優先權有沒有比當前正在跑的任務高？需不需要切換 CPU 給它？」

如果我們把 Yield 放到 `taskEXIT_CRITICAL()` 外面：

- 剛解開臨界區，別的中斷（ISR）或核心可能瞬間修改了 `pxTCB` 的狀態或優先權。
    
- 發生**競態條件（Race Condition）**，導致排程器做出了錯誤的切換決策。

3. 每個任務都有獨立的「臨界區嵌套計數」與 CPU 狀態

即便在某些硬體架構上，Yield 會「立刻」強行切換任務，FreeRTOS 也完全不怕，因為：

- **臨界區狀態是跟著任務的 TCB/Stack 走的**：每個任務的 Task Control Block 都有一個 `uxCriticalNesting` 變數，且任務切換時會把 CPU 的中斷暫存器（如 `PRIMASK` / `BASEPRI`）壓入該任務自己的 Stack（堆疊）中。

- **上下文流轉過程**：
	1. 任務 A 在 CS 內 Yield ➡️ 任務 A 保存了「自己處於 CS 內」的 Stack 狀態。
	2. 系統切換到任務 B ➡️ 任務 B 恢復它「自己的中斷狀態」（可能根本不在 CS 內，中斷正常開啟）。
	3. 未來切換回任務 A ➡️ 任務 A 恢復「自己處於 CS 內」的狀態，繼續執行 `taskEXIT_CRITICAL()`，完美收尾。

4. 多核心（SMP）環境下的 IPI (跨核中斷) 機制

`taskYIELD_ANY_CORE_IF_USING_PREEMPTION( pxTCB )` 是 FreeRTOS 在 SMP（多核心）環境下的巨集：

- 如果被喚醒的高優先權任務應該跑在 **Core 0（當前核心）**：它會設置當前核心的 Yield Flag（如前述，延遲到退出 CS 時觸發）。
    
- 如果該任務應該搶佔 **Core 1（另一個核心）**：當前核心會向 Core 1 發送一個 **IPI (Inter-Processor Interrupt，核間中斷)**。
    
    - 當前核心發完 IPI 後，繼續在自己的 CS 內安全地把 code 執行完。
        
    - Core 1 收到 IPI 後，會在 Core 1 上獨自引發 Yield。兩者完全不衝突！

### 108. *xTaskGenericNotifyFromISR*

這函數是 FreeRTOS 中**專門給硬體中斷（ISR）使用的萬能通知發送器**。

如果說上一篇介紹的 `xTaskGenericNotify()` 是任務與任務之間的「郵差」，那麼這個帶有 `FromISR` 後綴的版本，就是**由硬體中斷發起的「特快專遞」**。

雖然它的業務邏輯與 Task 級別的發送器非常相似，但為了適應中斷上下文（Interrupt Context）的嚴苛環境，它在底層做出了三大極為關鍵的硬體防禦與架構調整：

1. **中斷優先權安全斷言**（防止在中斷優先權太高時呼叫 API 搞垮核心）。
    
2. **中斷級臨界區保護**（保存/恢復 CPU 的中斷狀態暫存器）。
    
3. **排程器掛起時的延遲處理（`xPendingReadyList`）與上下文切換旗標（`pxHigherPriorityTaskWoken`）**。

#### 108.1 中斷優先權檢查、安全斷言與進入 ISR 臨界區

此區塊驗證硬體中斷優先權的合法性，並保存當前 CPU 中斷開關狀態後進入中斷臨界區。

```C
#if ( configUSE_TASK_NOTIFICATIONS == 1 )

    BaseType_t xTaskGenericNotifyFromISR( TaskHandle_t xTaskToNotify,
                                          UBaseType_t uxIndexToNotify,
                                          uint32_t ulValue,
                                          eNotifyAction eAction,
                                          uint32_t * pulPreviousNotificationValue,
                                          BaseType_t * pxHigherPriorityTaskWoken )
    {
        TCB_t * pxTCB;
        uint8_t ucOriginalNotifyState;
        BaseType_t xReturn = pdPASS;
        UBaseType_t uxSavedInterruptStatus;

        traceENTER_xTaskGenericNotifyFromISR( xTaskToNotify, uxIndexToNotify, ulValue, eAction, pulPreviousNotificationValue, pxHigherPriorityTaskWoken );

        /* 1. 安全斷言：確保指標與陣列索引合法 */
        configASSERT( xTaskToNotify );
        configASSERT( uxIndexToNotify < configTASK_NOTIFICATION_ARRAY_ENTRIES );

        /* 2. 核心硬體安全檢查（以 ARM Cortex-M 為例）：
         * 檢查當前觸發此 ISR 的硬體中斷優先權，是否高於 configMAX_SYSCALL_INTERRUPT_PRIORITY？
         * 如果優先權太高（數字太小），該中斷是不受 FreeRTOS 管轄的，絕對不允許呼叫任何 API！
         * 若違反此規則，系統會在 debug 模式下直接進入 Assert 崩潰，防止難以排查的記憶體踩壞。 */
        portASSERT_IF_INTERRUPT_PRIORITY_INVALID();

        pxTCB = xTaskToNotify;

        /* 3. 進入 ISR 級別臨界區：
         * 與 taskENTER_CRITICAL() 不同，這會保存當前 CPU 的中斷屏蔽狀態（BASEPRI / PRIMASK），
         * 並回傳給 uxSavedInterruptStatus 變數，以利後續安全恢復。 */
        uxSavedInterruptStatus = ( UBaseType_t ) taskENTER_CRITICAL_FROM_ISR();
        {
            /* 備份發送前的原始通知值 */
            if( pulPreviousNotificationValue != NULL )
            {
                *pulPreviousNotificationValue = pxTCB->ulNotifiedValue[ uxIndexToNotify ];
            }

            /* 4. 備份發送前的原始通知狀態，並將狀態更新為已收到通知 */
            ucOriginalNotifyState = pxTCB->ucNotifyState[ uxIndexToNotify ];
            pxTCB->ucNotifyState[ uxIndexToNotify ] = taskNOTIFICATION_RECEIVED;
```

- **`portASSERT_IF_INTERRUPT_PRIORITY_INVALID()`**：這是無數嵌入式新手工程師的血淚教訓！在 ARM Cortex-M 架構中，只有優先權低於/等於 `configMAX_SYSCALL_INTERRUPT_PRIORITY` 的中斷，才可以呼叫以 `FromISR` 結尾的 API。
    
- **`taskENTER_CRITICAL_FROM_ISR()`**：中斷裡面不能呼叫一般的 `taskENTER_CRITICAL()`，必須使用回傳狀態值的 ISR 專用版本。

#### 108.2 5 大發送動作處置（Modify Notification Value）

在中斷臨界區內，安全地對目標任務控制塊（TCB）中的 32-bit 通知值進行更新。

```C
/* 5. 根據 eAction 操作目標 Task 的通知數值 (與非 ISR 版本邏輯一致) */
            switch( eAction )
            {
                case eSetBits:
                    /* 模擬 Event Group Bit 位元操作 */
                    pxTCB->ulNotifiedValue[ uxIndexToNotify ] |= ulValue;
                    break;

                case eIncrement:
                    /* 模擬 Semaphore Give 計數 +1 */
                    ( pxTCB->ulNotifiedValue[ uxIndexToNotify ] )++;
                    break;

                case eSetValueWithOverwrite:
                    /* 覆蓋寫入最新值 */
                    pxTCB->ulNotifiedValue[ uxIndexToNotify ] = ulValue;
                    break;

                case eSetValueWithoutOverwrite:
                    /* 無覆蓋寫入：若舊通知未被讀取，則放棄寫入並返回 pdFAIL */
                    if( ucOriginalNotifyState != taskNOTIFICATION_RECEIVED )
                    {
                        pxTCB->ulNotifiedValue[ uxIndexToNotify ] = ulValue;
                    }
                    else
                    {
                        xReturn = pdFAIL;
                    }

                    break;

                case eNoAction:
                    /* 純觸發通知，不修改數值 */
                    break;

                default:
                    configASSERT( xTickCount == ( TickType_t ) 0 );
                    break;
            }

            traceTASK_NOTIFY_FROM_ISR( uxIndexToNotify );
```

#### 108.3 喚醒任務與「排程器掛起（Suspended）」下的防禦處理

這是 ISR API 中最為深奧、最具 OS 精髓的部分。如果中斷發生時，某個 Task 正好暫停了排程器（`vTaskSuspendAll`），中斷該如何處理？

```C
/* 6. 如果目標任務原本正在睡眠等待通知 (taskWAITING_NOTIFICATION) */
            if( ucOriginalNotifyState == taskWAITING_NOTIFICATION )
            {
                configASSERT( listLIST_ITEM_CONTAINER( &( pxTCB->xEventListItem ) ) == NULL );

                /* 7. 檢查排程器狀態：
                 * 情況 A (uxSchedulerSuspended == 0)：排程器正常運作中。
                 * 直接將任務從 Delayed List 移除，並塞入 Ready List 準備執行。 */
                if( uxSchedulerSuspended == ( UBaseType_t ) 0U )
                {
                    listREMOVE_ITEM( &( pxTCB->xStateListItem ) );
                    prvAddTaskToReadyList( pxTCB );

                    #if ( configUSE_TICKLESS_IDLE != 0 )
                    {
                        /* 如果開啟 Tickless 模式，重置下一次喚醒 Tick 時間 */
                        prvResetNextTaskUnblockTime();
                    }
                    #endif
                }
                /* 情況 B (uxSchedulerSuspended != 0)：排程器被某些任務暫停了！
                 * 警告：在排程器被暫停時，ISR 絕不能直接去操作 Ready List / Delayed List！
                 * 解決方案：將被喚醒的任務暫時掛入 xPendingReadyList（掛起就緒鏈結串列）。
                 * 等到後續 Task 呼叫 xTaskResumeAll() 解鎖排程器時，再由 Task 層級把這些任務搬回 Ready List。 */
                else
                {
                    listINSERT_END( &( xPendingReadyList ), &( pxTCB->xEventListItem ) );
                }
```

##### 108.3.1 為什麼task 不能在 event list

要理解為什麼「此時 task 不能在 event list 上」，我們必須先了解 FreeRTOS 的 TCB 裡面 **兩大核心鏈結串列節點 (List Items)** 的分工，以及 Task Notification（任務通知）與傳統 IPC（Queue/Semaphore）的本質差異。

在 FreeRTOS 的每個 `TCB_t` 結構體中，都內建了兩個用來排隊的雙向鏈結串列節點：

1. **`xStateListItem`（狀態節點）**：
    
    - 代表任務**目前的排程狀態**。
        
    - 會被掛在 `ReadyList`（就緒）、`DelayedList`（阻塞睡眠中）或 `SuspendedList`（掛起）上。
        
2. **`xEventListItem`（事件節點）**：
    
    - 代表任務正在**等待哪一個「IPC 物件」**。
        
    - 例如：等待 Queue 資料時，這個節點會被掛到該 Queue 的 `xTasksWaitingToReceive` 佇列上。

場景 A：任務阻塞在傳統 Queue / Semaphore / Event Group

當 Task 呼叫 `xQueueReceive()` 且需要等待時：

- **`xStateListItem`** ➡️ 掛到 **Delayed List**（追蹤逾時 Timeout）。
    
- **`xEventListItem`** ➡️ 掛到 **Queue 的 Event List**（讓 Queue 知道誰在等它）。
    
- **結論**：此時 `xEventListItem` 的容器（Container）**「不為 NULL」**！

場景 B：任務阻塞在 Task Notification (NotifyWait / NotifyTake)

當 Task 呼叫 `xTaskNotifyWait()` 且需要等待時：

- 任務通知是**直接儲存在自己的 TCB 裡**，根本**沒有任何 Queue / Semaphore 物件結構體**！
    
- **`xStateListItem`** ➡️ 掛到 **Delayed List**（追蹤逾時 Timeout）。
    
- **`xEventListItem`** ➡️ **完全沒用到，保持為 NULL**（不掛在任何 Event List 上）！
#### 108.4 搶佔標記（`pxHigherPriorityTaskWoken`）與退出 ISR 臨界區

中斷服務程式不應該自己貿然切換 Task。它會透過指標參數通知使用者「有沒有更高優先權的任務被喚醒了」。

```C
/* 8. 評估是否需要觸發上下文切換 (Context Switch) */
                #if ( configNUMBER_OF_CORES == 1 )
                {
                    /* 單核心環境：如果被喚醒的任務優先權 > 當前正在跑的任務優先權 */
                    if( pxTCB->uxPriority > pxCurrentTCB->uxPriority )
                    {
                        /* 將使用者傳入的指標設為 pdTRUE，告知 ISR 結束時應該發起切換 */
                        if( pxHigherPriorityTaskWoken != NULL )
                        {
                            *pxHigherPriorityTaskWoken = pdTRUE;
                        }

                        /* 在系統層級標記：有一個搶佔請求正在等待處置 */
                        xYieldPendings[ 0 ] = pdTRUE;
                    }
                    else
                    {
                        mtCOVERAGE_TEST_MARKER();
                    }
                }
                #else /* 多核心 (SMP) 環境 */
                {
                    #if ( configUSE_PREEMPTION == 1 )
                    {
                        /* 多核心環境：評估應該搶佔哪一個核心並發送 IPI */
                        prvYieldForTask( pxTCB );

                        if( xYieldPendings[ portGET_CORE_ID() ] == pdTRUE )
                        {
                            if( pxHigherPriorityTaskWoken != NULL )
                            {
                                *pxHigherPriorityTaskWoken = pdTRUE;
                            }
                        }
                    }
                    #endif
                }
                #endif
            }
        }
        /* 9. 退出 ISR 臨界區，恢復原先的 CPU 中斷屏蔽狀態 */
        taskEXIT_CRITICAL_FROM_ISR( uxSavedInterruptStatus );

        traceRETURN_xTaskGenericNotifyFromISR( xReturn );

        return xReturn;
    }

#endif /* configUSE_TASK_NOTIFICATIONS */
```

### 109. *vTaskGenericNotifyGiveFromISR*

`xTaskGenericNotifyFromISR()` 是 ISR 裡的「全能型發送器」，那麼這個 `vTaskGenericNotifyGiveFromISR()` 就是專門為「訊號量 Give (釋放/加一)」**打造的**輕量化專用特快車！

它在 FreeRTOS 中主要被高階巨集 `vTaskNotifyGiveFromISR()` 所呼叫，用來取代傳統的 `xSemaphoreGiveFromISR()`。

### 與全能型 `xTaskGenericNotifyFromISR` 的核心差異

1. **無 `eAction` 參數**：這裏沒有 switch-case，因為它的動作是**硬編碼（Hardcoded）直接進行 `( Value )++`**。
    
2. **回傳型態為 `void`**：因為「數值加一」永遠不會像「無覆蓋模式」那樣發生失敗，所以不需要回傳 `pdPASS` / `pdFAIL`。
    
3. **執行效率更高**：少了參數判斷與舊值備份，程式碼路徑更短、中斷響應（Latency）更快！

### 110. *xTaskGenericNotifyStateClear*

`xTaskGenericNotifyStateClear()` 是 FreeRTOS 任務通知體系中一個極為輕量且專一的 API。

它的核心任務非常單純：**「抹除『未讀通知』的狀態標記，但不觸碰/不修改通知的真實數值（Value）」**。

在 FreeRTOS 高階封裝中，我們常呼叫的 `xTaskNotifyStateClear()` 或 `xTaskNotifyStateClearIndexed()`，底層全都是由這個函數一手包辦。

#### 110.1 State Clear vs Value Clear

在 FreeRTOS 中，新手很容易搞混 **`xTaskNotifyStateClear()`** 與 **`ulTaskNotifyValueClear()`**，這兩者有著本質上的不同：

```C
┌────────────────────────────────────────────────────────────────────────┐
│                        TCB Notification 結構                           │
│                                                                        │
│   1. ucNotifyState  [8-bit]  ──► 相當於「信箱的紅燈 (有/無未讀信件)」     │
│   2. ulNotifiedValue [32-bit] ──► 相當於「信箱裡面的信件資料內容」         │
└────────────────────────────────────────────────────────────────────────┘
```

- **`xTaskNotifyStateClear()`（即本文函數）**：
    
    - **只動 `ucNotifyState`**。
        
    - 把信箱的「紅燈（`taskNOTIFICATION_RECEIVED`）」關掉，標示為無未讀信件。
        
    - **優勢與用途**：如果別人在你忙碌時發了通知給你，而你經過評估後決定「**忽略這次觸發，但保留裡面的資料**」，或是單純想**查詢/測試「剛才到底有沒有人發通知給我？」**（利用回傳值是 `pdPASS` 還是 `pdFAIL` 來判斷），就可以呼叫此 API。
        
- **`ulTaskNotifyValueClear()`**：
    
    - **只動 `ulNotifiedValue`**。
        
    - 將 32-bit 的通知數值進行 Bitmask 清零（例如把 Bit 0 清掉），但不去改變 `ucNotifyState` 的燈號狀態。


### 111. *ulTaskGenericNotifyValueClear*

與前面介紹的「熄滅通知燈號」的 `xTaskGenericNotifyStateClear()` 不同，這個 **`ulTaskGenericNotifyValueClear()`** 是專為 **Bitmask（位元旗標）** 設計的「精準數值抹除器」。

它的核心職責是：**「針對目標任務 32-bit 的通知數值（`ulNotifiedValue`），精準抹除指定被置 1 的 Bit 位元，並回傳『清除前』的原始數值。」**

在 FreeRTOS 高階封裝中，我們呼叫的 `ulTaskNotifyValueClear()` 或 `ulTaskNotifyValueClearIndexed()`，底層全都是由這個函數來執行。

深度概念解析：為什麼回傳的是「清除前」的值？

這是這個 API 最精妙之處！想像一下實務上的應用場景：

假設你的任務使用 Notification 來接收多種事件（Bit 0 代表 UART, Bit 1 代表 DMA, Bit 2 代表 SPI）：

1. 任務的 `ulNotifiedValue` 目前是 `0x07` (`0b0111`，代表三個事件都發生了）。
    
2. 任務現在只想處理 UART (Bit 0) 與 DMA (Bit 1)，所以傳入 `ulBitsToClear = 0x03` (`0b0011`)。
    
3. 函數執行後：
    
    - `ulNotifiedValue` 變成了 `0x04` (`0b0100`，SPI 旗標被安全保留，留給下次處理）。
        
    - 回傳值 `ulReturn` 則是 **`0x07`**（清除前的完整原始值）。
        

**好處**：呼叫者拿到了回傳值 `0x07`，就能直接拿這個回傳值去作 `if (ulReturn & 0x01)` 的條件判斷，同時內部資料結構又已經安全地把這兩個 Bit 抹清了，一次呼叫直接完成「讀取 + 指定 Bit 清零」！

### 112. *ulTaskGetRunTimeCounter*

`ulTaskGetRunTimeCounter()` 是 FreeRTOS 中用於**效能分析（Profiling）與 CPU 使用率統計**的底層核心 API。

在 FreeRTOS 中，當我們啟動運行時間統計功能（`configGENERATE_RUN_TIME_STATS == 1`）時，系統會紀錄每個任務佔用 CPU 的累計時間。高階函數如 `vTaskGetRunTimeStats()`（用來列印出所有 Task 的 CPU 使用率 %）底層就是靠這個函數取得數據的。

這個函數最精妙的地方在於：**它連「正在執行中（Running）」的任務，都能即時算入「此時此刻」剛消耗的 CPU 時間！**

#### 112.1 臨界區計算與「即時運行時間（Live Delta）」補算

進入臨界區後，檢查目標 Task 是否為「當前正在 CPU 上跑的任務」。如果是，必須額外補算**從上一次切換進來（Switched In）到當前這一個微秒**所經過的時間差！

```C
/* 2. 進入臨界區（關閉中斷）：防止計算過程中發生任務切換導致時間數值錯亂 */
        taskENTER_CRITICAL();
        {
            /* 3. 關鍵判斷：目標任務「此時此刻」是否正在 CPU 上執行？ */
            if( taskTASK_IS_RUNNING( pxTCB ) == pdTRUE )
            {
                /* 4. 讀取高精度硬體計時器 (High-Resolution Hardware Timer) 當前的計數值 */
                #ifdef portALT_GET_RUN_TIME_COUNTER_VALUE
                    portALT_GET_RUN_TIME_COUNTER_VALUE( ulTotalTime );
                #else
                    ulTotalTime = portGET_RUN_TIME_COUNTER_VALUE();
                #endif

                /* 5. 計算「從切換進來執行 (Switched-In) 到現在」經過的時間增量 (Delta)：
                 * - 單核心：直接用目前時間 - ulTaskSwitchedInTime[0]
                 * - 多核心 (SMP)：使用 xTaskRunState 作為 Core ID 索引，找出該 Core 的切換時間點 */
                #if ( configNUMBER_OF_CORES == 1 )
                    ulTimeSinceLastSwitchedIn = ulTotalTime - ulTaskSwitchedInTime[ 0 ];
                #else
                    ulTimeSinceLastSwitchedIn = ulTotalTime - ulTaskSwitchedInTime[ pxTCB->xTaskRunState ];
                #endif
            }

            /* 6. 計算最終時間：
             * pxTCB->ulRunTimeCounter 是「過去每一次切換切出累積的歷史總時間」
             * ulTimeSinceLastSwitchedIn 是「當前這一次切換進來後跑了多久 (若沒在跑則為 0)」 */
            ulTaskRunTime = pxTCB->ulRunTimeCounter + ulTimeSinceLastSwitchedIn;
        }
        taskEXIT_CRITICAL();

        traceRETURN_ulTaskGetRunTimeCounter( ulTaskRunTime );

        /* 7. 回傳該任務自系統開機（或計數器重置）以來獲得的總 CPU 計時數值 */
        return ulTaskRunTime;
    }

#endif /* if ( configGENERATE_RUN_TIME_STATS == 1 ) */
```

為什麼不能直接回傳 `pxTCB->ulRunTimeCounter`？

這是這個 API 最核心的設計細節！

在 FreeRTOS 的排程器機制中：

- `pxTCB->ulRunTimeCounter` 變數**只會在任務「被切換出去（Switched Out）」的瞬間才進行累加更新**。
    

想像一下這個場景：

> 如果有一個優先權極高的 Task A，它連續在 CPU 上狂奔了 10 秒鐘都沒有休眠，此時如果另一個中斷或 Task B 呼叫 `ulTaskGetRunTimeCounter(TaskA)` 來查詢 Task A 跑了多久：
> 
> - 如果直接拿 `pxTCB->ulRunTimeCounter` 讀取，讀到的會是 **10 秒前的舊數據**（因為 Task A 還沒被切換出去，值還沒加進去）。
>     
> - **FreeRTOS 的解決方案**：先看它是不是正在跑，如果是，立刻拿「**當前計時器時間 - 當初切進來的時間點**」，算出這 10 秒時間並加上去，就能得到完美的 **即時精確時間**！

### 113. *ulTaskGetRunTimePercent*

略

### 114. *ulTaskGetIdleRunTimeCounter*

`ulTaskGetIdleRunTimeCounter()` 是 FreeRTOS 中專門用來統計 **空閒任務（Idle Task）所佔用 CPU 總時間** 的核心 API。

在嵌入式系統中，我們常需要計算 **CPU 使用率（CPU Usage / Load）**。常見的公式為：

$$\text{CPU Usage (\%)} = 100\% - \left( \frac{\text{Idle Task Run Time}}{\text{Total System Run Time}} \right) \times 100\%$$

為了算這個公式，系統必須知道「Idle Task 到底跑了多久」。此函數的優點在於它**支援單核心與多核心（SMP）環境**，能將所有核心上的 Idle Task 所消耗的時間進行總和，且同樣包含「當前正在執行的 Live Time 補算」。

#### 114.1 雙重條件編譯、變數宣告與進入點

此 API 必須同時在 `FreeRTOSConfig.h` 開啟運行時間統計與 Idle Task 句柄獲取功能時才會被編譯。

```C
#if ( ( configGENERATE_RUN_TIME_STATS == 1 ) && ( INCLUDE_xTaskGetIdleTaskHandle == 1 ) )

    configRUN_TIME_COUNTER_TYPE ulTaskGetIdleRunTimeCounter( void )
    {
        configRUN_TIME_COUNTER_TYPE ulTotalTime = 0, ulTimeSinceLastSwitchedIn = 0, ulIdleTaskRunTime = 0;
        BaseType_t i;

        traceENTER_ulTaskGetIdleRunTimeCounter();

        /* 註：此 API 不需要傳入 TaskHandle，因為它會自動遍歷系統中所有核心的 xIdleTaskHandles 陣列 */
```

#### 114.2 臨界區保護、多核心 Idle Task 時間累加與 Live Delta 補算

進入臨界區後，讀取高精度硬體計時器當前值，並以 `for` 迴圈遍歷每一個 CPU Core，加總所有 Idle Task 的累計執行時間。

```C
/* 1. 進入臨界區（關閉中斷）：防止計算過程中發生任務切換或核心狀態變更 */
        taskENTER_CRITICAL();
        {
            /* 2. 讀取高精度硬體計量 Timer 當前的總時間數值 */
            #ifdef portALT_GET_RUN_TIME_COUNTER_VALUE
                portALT_GET_RUN_TIME_COUNTER_VALUE( ulTotalTime );
            #else
                ulTotalTime = portGET_RUN_TIME_COUNTER_VALUE();
            #endif

            /* 3. 遍歷系統中的每一個 CPU 核心 (單核心時 configNUMBER_OF_CORES 為 1) */
            for( i = 0; i < ( BaseType_t ) configNUMBER_OF_CORES; i++ )
            {
                /* 4. 檢查： Core i 的 Idle Task 此時此刻是否正在 CPU 上運行？ */
                if( taskTASK_IS_RUNNING( xIdleTaskHandles[ i ] ) == pdTRUE )
                {
                    /* 5. 若 Idle Task 正在運行，補算從「切入 (Switched In) 到現在」的即時時間增量 */
                    #if ( configNUMBER_OF_CORES == 1 )
                        ulTimeSinceLastSwitchedIn = ulTotalTime - ulTaskSwitchedInTime[ 0 ];
                    #else
                        ulTimeSinceLastSwitchedIn = ulTotalTime - ulTaskSwitchedInTime[ xIdleTaskHandles[ i ]->xTaskRunState ];
                    #endif
                }
                else
                {
                    /* 若該 Core 的 Idle Task 目前處於休眠/被搶佔狀態，增量設為 0 */
                    ulTimeSinceLastSwitchedIn = 0;
                }

                /* 6. 累加該 Core 的 Idle Task 時間：
                 * 歷史累計時間 (ulRunTimeCounter) + 當前執行增量 (ulTimeSinceLastSwitchedIn) */
                ulIdleTaskRunTime += ( xIdleTaskHandles[ i ]->ulRunTimeCounter + ulTimeSinceLastSwitchedIn );
            }
        }
        taskEXIT_CRITICAL();

        traceRETURN_ulTaskGetIdleRunTimeCounter( ulIdleTaskRunTime );

        /* 7. 回傳所有核心 Idle Task 消耗的總 CPU 計時數值 */
        return ulIdleTaskRunTime;
    }

#endif /* if ( ( configGENERATE_RUN_TIME_STATS == 1 ) && ( INCLUDE_xTaskGetIdleTaskHandle == 1 ) ) */
```

### 115. *ulTaskGetIdleRunTimePercent*

略

### 116. *prvAddCurrentTaskToDelayedList*

`prvAddCurrentTaskToDelayedList()` 是 FreeRTOS 內核中**讓當前任務「進入睡眠 / 阻塞 (Blocked)」的最核心底層函數**！

無論任務是因為呼叫了 `vTaskDelay()`、等待 Queue 資料（`xQueueReceive`）、等待 Semaphore，還是等待任務通知（`xTaskNotifyWait`），只要**確定無法立刻取得資源且需要等待**，FreeRTOS 底層最後**通通都會呼叫這個函數**把當前任務（`pxCurrentTCB`）從「Ready List」拔掉，並塞入「Delayed List」或「Suspended List」。

#### 116.1 變數初始化與中斷延遲取消旗標重置

第一步準備指向上次/當前延遲清單的指標，並重置 `ucDelayAborted` 旗標（以利後續判斷任務是「時間到自然醒」還是「被其他任務強行中止等待」）。

```C
static void prvAddCurrentTaskToDelayedList( TickType_t xTicksToWait,
                                            const BaseType_t xCanBlockIndefinitely )
{
    TickType_t xTimeToWake;
    const TickType_t xConstTickCount = xTickCount;
    /* 取得當前使用的 Delayed List 與 Overflow Delayed List 指標 */
    List_t * const pxDelayedList = pxDelayedTaskList;
    List_t * const pxOverflowDelayedList = pxOverflowDelayedTaskList;

    #if ( INCLUDE_xTaskAbortDelay == 1 )
    {
        /* 1. 重置 Delay Aborted 旗標：
         * 任務即將進入 Blocked 狀態，將此旗標清為 pdFALSE。
         * 如果之後有其他 Task 呼叫 xTaskAbortDelay() 強行喚醒它，
         * 這個旗標會被置為 pdTRUE，讓該 Task 醒來後知道自己是被『強行中斷等待』的。 */
        pxCurrentTCB->ucDelayAborted = ( uint8_t ) pdFALSE;
    }
    #endif
```

#### 116.2 從 Ready List 移除並更新硬體優先權點陣圖

任務既然要睡覺了，就必須從準備好執行的 Ready List 中抽離。如果該優先權已經沒有其他 Task 了，必須同步關閉硬體優先權點陣圖中的對應 Bit。

```C
/* 2. 從 Ready List 中移除當前任務 (pxCurrentTCB)：
     * 注意：FreeRTOS 每個 Task 的 xStateListItem 同一時間只能掛在一個 List 上。
     * 要掛入 Delayed List 前，必須先從 Ready List 移除。 */
    if( uxListRemove( &( pxCurrentTCB->xStateListItem ) ) == ( UBaseType_t ) 0 )
    {
        /* 3. 更新優先權位元圖 (Priority Bitmap)：
         * 如果移除該任務後，該優先權鏈結串列 (Ready List) 已經『變為空清單』，
         * 則呼叫 portRESET_READY_PRIORITY 抹除該優先權在硬體 Bitmask 中的 1。
         * 這樣排程器在尋找最高優先權任務時，就能以 O(1) 的時間直接跳過這個優先權！ */
        portRESET_READY_PRIORITY( pxCurrentTCB->uxPriority, uxTopReadyPriority );
    }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }
```

#### 116.3 無限期阻塞處置（Suspended List）

如果任務設置了 `portMAX_DELAY`（無限期等待），且系統允許無限期阻塞，FreeRTOS 會將其優雅地放入 `xSuspendedTaskList`，避免無謂地參與 Tick 喚醒計算。

```C
#if ( INCLUDE_vTaskSuspend == 1 )
    {
        /* 4. 判斷是否為「永久阻塞 (Block Indefinitely)」：
         * 條件：等待時間為 portMAX_DELAY 且 xCanBlockIndefinitely 為 pdTRUE */
        if( ( xTicksToWait == portMAX_DELAY ) && ( xCanBlockIndefinitely != pdFALSE ) )
        {
            /* 直接將任務掛入 xSuspendedTaskList（掛起清單）。
             * 好處：不需要幫它計算喚醒時間 (xTimeToWake)，也不需要把它掛在 Delayed List 上，
             * 這樣 SysTick 中斷每次檢查延遲清單時，就完全不用理會這個任務，大幅提升系統效能！ */
            listINSERT_END( &xSuspendedTaskList, &( pxCurrentTCB->xStateListItem ) );
        }
        else
        {
```

#### 116.4 喚醒時間計算、Tick 溢位 (Overflow) 分流與 SysTick 最佳化

對於有具體等待時間的任務，計算預計喚醒時間 `xTimeToWake`。最精妙的是 **Tick 溢位處理** 與 **`xNextTaskUnblockTime` 更新**！

```C
/* 5. 計算預計喚醒時間點：當前 Tick + 預計等待 Tick */
            xTimeToWake = xConstTickCount + xTicksToWait;

            /* 將任務的 xStateListItem 數值設定為喚醒時間，
             * vListInsert 會自動根據這個數值進行『從小到大 (升冪)』排序插入！ */
            listSET_LIST_ITEM_VALUE( &( pxCurrentTCB->xStateListItem ), xTimeToWake );

            /* 6. 核心精髓：檢查 Tick 計數器是否發生溢位 (Overflow)？ */
            if( xTimeToWake < xConstTickCount )
            {
                /* 情況 A：喚醒時間算出來比當前時間還小，代表 32-bit (或 16-bit) Tick 溢位了！
                 * 放入『溢位延遲清單 (pxOverflowDelayedList)』暫存。
                 * 當未來 TickCount 真的溢位歸零時，FreeRTOS 會將兩組 Delayed List 指標互換。 */
                traceMOVED_TASK_TO_OVERFLOW_DELAYED_LIST();
                vListInsert( pxOverflowDelayedList, &( pxCurrentTCB->xStateListItem ) );
            }
            else
            {
                /* 情況 B：正常未溢位，放入當前的延遲清單 (pxDelayedList) */
                traceMOVED_TASK_TO_DELAYED_LIST();
                vListInsert( pxDelayedList, &( pxCurrentTCB->xStateListItem ) );

                /* 7. SysTick 中斷效能最佳化 (xNextTaskUnblockTime)：
                 * 如果這個剛塞進去的任務，其喚醒時間比原本預計『下一次要喚醒的任務時間』還要早，
                 * 則更新 xNextTaskUnblockTime。
                 * 這樣 SysTick ISR 每次觸發時，只需要看 xTickCount == xNextTaskUnblockTime，
                 * 不需要每次都去掃描整條 Delayed List！ */
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
    #else /* INCLUDE_vTaskSuspend == 0 時的替代邏輯 (不支援永久掛起，一律算時間) */
    {
        /* 邏輯與上方 else 區塊完全相同 */
        xTimeToWake = xConstTickCount + xTicksToWait;
        listSET_LIST_ITEM_VALUE( &( pxCurrentTCB->xStateListItem ), xTimeToWake );

        if( xTimeToWake < xConstTickCount )
        {
            traceMOVED_TASK_TO_OVERFLOW_DELAYED_LIST();
            vListInsert( pxOverflowDelayedList, &( pxCurrentTCB->xStateListItem ) );
        }
        else
        {
            traceMOVED_TASK_TO_DELAYED_LIST();
            vListInsert( pxDelayedList, &( pxCurrentTCB->xStateListItem ) );

            if( xTimeToWake < xNextTaskUnblockTime )
            {
                xNextTaskUnblockTime = xTimeToWake;
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }

        ( void ) xCanBlockIndefinitely;
    }
    #endif /* INCLUDE_vTaskSuspend */
}
```

### 117. *xTaskGetMPUSettings*

`xTaskGetMPUSettings()` 是 FreeRTOS 支援 **MPU（Memory Protection Unit，記憶體保護單元）** 版本時的核心 API。

當系統啟動 MPU 包裹器（`portUSING_MPU_WRAPPERS == 1`）時，每個任務都有其獨立的記憶體存取權限與保護區域（例如 Stack 邊界防護、對待特權/非特權模式的 Flash/SRAM 區域劃分）。這個函數的職責非常簡單且純粹：**「獲取指定任務（Task）的 MPU 結構體（`xMPU_SETTINGS`）記憶體位址」**。

什麼是 `xMPU_SETTINGS`？

在無 MPU 的常規嵌入式系統中，所有任務共用同一個記憶體空間，一個任務寫錯指標（野指標）可能會把全域變數或另一個任務的堆疊（Stack）蓋掉導致系統崩潰。

但在 **FreeRTOS-MPU** 架構下，系統將任務分為 **Privileged（特權模式）** 與 **Unprivileged（非特權模式）**：

- 每個 Task 的 TCB 裡面都會含有一個 `xMPU_SETTINGS` 結構體。
    
- 結構體中紀錄了該 Task **專屬的 3 ~ 8 個 MPU 區域（Regions）**（如：Task 自己的 Stack 區域 Read/Write 權限、特定周邊暫存器的存取許可等）。
    
- **上下文切換（Context Switch）時的應用**： 當排程器準備切換到 Task A 執行時，PendSV ISR 會呼叫底層移植層，拿 Task A 的 `xMPUSettings` 指標，快速重新寫入 MCU 的硬體 MPU 暫存器。這能確保 **Task A 執行期間，一旦試圖越界存取非授權記憶體，會立刻觸發硬體 MemManage Fault，而不是默默破壞系統資料**！

### 118. vApplicationGetIdleTaskMemory & vApplicationGetPassiveIdleTaskMemory

這段程式碼是 FreeRTOS 為了**靜態記憶體配置（Static Allocation）**所提供的**「預設 Idle Task 記憶體提供器（Default Memory Provider）」**。

在傳統 FreeRTOS 啟用靜態記憶體配置（`configSUPPORT_STATIC_ALLOCATION == 1`）時，系統啟動呼叫 `vTaskStartScheduler()` 會建立空閒任務（Idle Task）。因為不能用 Heap（`pvPortMalloc`），Kernel 必須向外部索取「預先配置好的 TCB 與 Stack 記憶體區塊」。

以往這需要開發者**手動在應用層編寫 Hook 函數**；而在新版 FreeRTOS 中，若開啟了 `configKERNEL_PROVIDED_STATIC_MEMORY == 1`，Kernel 就會直接自動幫你把這套靜態記憶體「傳包做好」，極大地簡化了程式碼開發！

#### 118.1 條件編譯條件與 Core 0 (單核/主核) Idle Task 靜態記憶體配置

此處為第一核心（Core 0）或單核心系統中的 Idle Task 提供靜態 TCB 與 Stack 緩衝區。

```C
#if ( ( configSUPPORT_STATIC_ALLOCATION == 1 ) && ( configKERNEL_PROVIDED_STATIC_MEMORY == 1 ) && ( portUSING_MPU_WRAPPERS == 0 ) )

/*
 * Kernel 預設提供的 vApplicationGetIdleTaskMemory() 實作。
 * 用於在啟用靜態配置時，自動為 Idle Task 提供記憶體區塊。
 * 若使用者將 configKERNEL_PROVIDED_STATIC_MEMORY 設為 0，則可於應用層自訂此函數。
 */
    void vApplicationGetIdleTaskMemory( StaticTask_t ** ppxIdleTaskTCBBuffer,
                                        StackType_t ** ppxIdleTaskStackBuffer,
                                        configSTACK_DEPTH_TYPE * puxIdleTaskStackSize )
    {
        /* 1. 使用 static 關鍵字宣告配置於 SRAM (.bss/.data 段) 的靜態 TCB 與 Stack 陣列。
         * 其生命週期伴隨整個系統開機期間，絕不會被 Stack 釋放。 */
        static StaticTask_t xIdleTaskTCB;
        static StackType_t uxIdleTaskStack[ configMINIMAL_STACK_SIZE ];

        /* 2. 將靜態 TCB 結構體位址、Stack 陣列首位址與堆疊深度輸出給 Kernel */
        *ppxIdleTaskTCBBuffer = &( xIdleTaskTCB );
        *ppxIdleTaskStackBuffer = &( uxIdleTaskStack[ 0 ] );
        *puxIdleTaskStackSize = configMINIMAL_STACK_SIZE;
    }
```

#### 118.2 多核心 (SMP) 環境被動核心 (Passive Cores) 的 Idle Task 靜態記憶體配置

在多核心（SMP）環境下，除了 Core 0 外，其他核心（Core 1 ~ Core N-1）稱為 Passive Cores，每個核心都需要獨立的 Idle Task。此區塊為這些被動核心配置靜態記憶體陣列。

```C
#if ( configNUMBER_OF_CORES > 1 )

        /*
         * 多核心 (SMP) 專用：為被動核心 (Core 1 到 Core N-1) 提供 Idle Task 靜態記憶體。
         * xPassiveIdleTaskIndex 的範圍為 0 到 (configNUMBER_OF_CORES - 2)。
         */
        void vApplicationGetPassiveIdleTaskMemory( StaticTask_t ** ppxIdleTaskTCBBuffer,
                                                   StackType_t ** ppxIdleTaskStackBuffer,
                                                   configSTACK_DEPTH_TYPE * puxIdleTaskStackSize,
                                                   BaseType_t xPassiveIdleTaskIndex )
        {
            /* 1. 根據被動核心數量 (configNUMBER_OF_CORES - 1)，宣告靜態 TCB 陣列與 Stack 二維陣列 */
            static StaticTask_t xIdleTaskTCBs[ configNUMBER_OF_CORES - 1 ];
            static StackType_t uxIdleTaskStacks[ configNUMBER_OF_CORES - 1 ][ configMINIMAL_STACK_SIZE ];

            /* 2. 根據傳入的核心索引，指派對應核心專屬的 TCB 與 Stack 緩衝區位址 */
            *ppxIdleTaskTCBBuffer = &( xIdleTaskTCBs[ xPassiveIdleTaskIndex ] );
            *ppxIdleTaskStackBuffer = &( uxIdleTaskStacks[ xPassiveIdleTaskIndex ][ 0 ] );
            *puxIdleTaskStackSize = configMINIMAL_STACK_SIZE;
        }

    #endif /* #if ( configNUMBER_OF_CORES > 1 ) */

#endif /* #if ( ( configSUPPORT_STATIC_ALLOCATION == 1 ) && ( configKERNEL_PROVIDED_STATIC_MEMORY == 1 ) && ( portUSING_MPU_WRAPPERS == 0 ) ) */
```

- **`configSUPPORT_STATIC_ALLOCATION == 1`**：系統啟動了靜態記憶體配置支援。
    
- **`configKERNEL_PROVIDED_STATIC_MEMORY == 1`**：告訴 FreeRTOS「**由 Kernel 開箱即用提供預設的 Idle/Timer 靜態記憶體**」，開發者不用再手寫 `vApplicationGetIdleTaskMemory()` callback！
    
- **`portUSING_MPU_WRAPPERS == 0`**：未開啟 MPU（記憶體保護單元）。因為在 MPU 環境下，Idle Task 的 Stack 必須做硬體邊界對齊（Alignment），需要專門的 MPU 版本 Memory Provider。

在 `vApplicationGetIdleTaskMemory()` 內部宣告的 `xIdleTaskTCB` 與 `uxIdleTaskStack` 加上了 **`static`** 修飾符：

- 如果不加 `static`，變數會分配在「呼叫者的 Function Stack」上，當函數執行完了 `return`，這塊記憶體空間就會被破壞。
    
- 加上 `static` 後，編譯器會將其放入 RAM 的 **`.bss` 或 `.data` 區段**，擁有與全域變數相同的永久生命週期，確保 Idle Task 運作全程記憶體安全無虞。

### 119. *vApplicationGetTimerTaskMemory*

這個 **`vApplicationGetTimerTaskMemory()`** 函數與前面介紹的 `vApplicationGetIdleTaskMemory()` 互為姐妹函數。它是 FreeRTOS 內核專為 **「軟體定時器服務任務（Timer Service Task / Daemon Task）」** 所提供的**靜態記憶體預設提供器**。

當你在 `FreeRTOSConfig.h` 中開啟了軟體定時器功能（`configUSE_TIMERS == 1`）並使用靜態記憶體配置（`configSUPPORT_STATIC_ALLOCATION == 1`）時，RTOS 啟動過程中會自動建立一個背景任務（`Tmr Svc`）來管理所有的 Timer 輪詢與 Callback。

若開啟了 `configKERNEL_PROVIDED_STATIC_MEMORY == 1`，FreeRTOS 內核就會自動提供這個函數，替這個 Timer 任務安排好靜態的 TCB 與 Stack 空間，無需開發者手動在應用層寫 Callback。

```C
#if ( ( configSUPPORT_STATIC_ALLOCATION == 1 ) && ( configKERNEL_PROVIDED_STATIC_MEMORY == 1 ) && ( portUSING_MPU_WRAPPERS == 0 ) && ( configUSE_TIMERS == 1 ) )

/*
 * Kernel 預設提供的 vApplicationGetTimerTaskMemory() 實作。
 * 當 configKERNEL_PROVIDED_STATIC_MEMORY 設為 1 時，由內核自動為 Timer Service Task 提供靜態記憶體。
 * 若應用層想要自己指定 Timer 任務的記憶體位址，可將該巨集設為 0 或不定義。
 */
    void vApplicationGetTimerTaskMemory( StaticTask_t ** ppxTimerTaskTCBBuffer,
                                         StackType_t ** ppxTimerTaskStackBuffer,
                                         configSTACK_DEPTH_TYPE * puxTimerTaskStackSize )
    {
        /* 1. 使用 static 宣告永久配置於 RAM (.bss / .data 段) 的靜態 TCB 與 Stack 陣列。
         * 注意：Timer Task 的 Stack 深度使用的是獨立的 configTIMER_TASK_STACK_DEPTH 巨集，
         * 而不是 Idle Task 所使用的 configMINIMAL_STACK_SIZE。 */
        static StaticTask_t xTimerTaskTCB;
        static StackType_t uxTimerTaskStack[ configTIMER_TASK_STACK_DEPTH ];

        /* 2. 將靜態 TCB 結構體位址、Stack 陣列首位址與堆疊深度輸出給 Kernel */
        *ppxTimerTaskTCBBuffer = &( xTimerTaskTCB );
        *ppxTimerTaskStackBuffer = &( uxTimerTaskStack[ 0 ] );
        *puxTimerTaskStackSize = configTIMER_TASK_STACK_DEPTH;
    }

#endif /* #if ( ( configSUPPORT_STATIC_ALLOCATION == 1 ) && ( configKERNEL_PROVIDED_STATIC_MEMORY == 1 ) && ( portUSING_MPU_WRAPPERS == 0 ) && ( configUSE_TIMERS == 1 ) ) */
```

|**巨集名稱**|**設定值**|**意義與作用**|
|---|---|---|
|**`configSUPPORT_STATIC_ALLOCATION`**|`1`|系統支援靜態記憶體配置（不使用 Heap）。|
|**`configKERNEL_PROVIDED_STATIC_MEMORY`**|`1`|告訴 RTOS 內核：「由內核預設提供 Idle 與 Timer Task 的靜態記憶體，不需要使用者手動寫 Hook/Callback」。|
|**`portUSING_MPU_WRAPPERS`**|`0`|未開啟 MPU（記憶體保護單元）。因為 MPU 環境下 Stack 必須滿足硬體對齊規範，需使用專門的 MPU 版本。|
|**`configUSE_TIMERS`**|`1`|**開啟軟體定時器功能**。若此值為 `0`，則系統根本不會建立 Timer Task，此函數自然也不會編譯進去。|
為什麼 Timer Task 的 Stack 使用 `configTIMER_TASK_STACK_DEPTH`？

- **Idle Task**：平常沒事只做清除刪除任務（Yield / Garbage Collection）或進入 Low Power Mode，邏輯極為簡單，因此 Stack 使用最低限制 **`configMINIMAL_STACK_SIZE`**。
    
- **Timer Task (`Tmr Svc`)**：負責執行所有你透過 `xTimerCreate()` 註冊的 **Software Timer Callback 函數**。
    
    - 由於 Callback 函數是在 **Timer Task 的 Context（上下文）** 中執行的，如果在 Callback 裡面呼叫了複雜的函數或宣告了區域變數，會消耗較多的 Stack。
        
    - 因此 FreeRTOS 讓 Timer Task 擁有獨立的堆疊大小設定 **`configTIMER_TASK_STACK_DEPTH`**，方便開發者根據 Callback 的複雜度靈活調大 Stack，防止 Stack Overflow！

### 120. *vTaskResetState*

`vTaskResetState()` 是 FreeRTOS 內核中一個專門用於「將 `tasks.c` 檔案層級的靜態全域變數重置為初始狀態」的內部 API。

在一般的嵌入式系統中，MCU 開機後 RTOS 只會啟動一次（呼叫 `vTaskStartScheduler()` 後就不再退出）。但在某些特定情境下——例如**軟體熱重啟（Soft Reboot / Warm Reset）**、**單元測試（Unit Testing，需要在同一個程序中反覆啟動與停止內核）**，或是**重新啟動排程器**時，就必須呼叫 `vTaskResetState()` 將 `tasks.c` 中記錄的所有狀態計數器、 Tick 計數、指標等歸零，否則舊的狀態會汙染下一次的內核啟動。

#### 120.1 函數宣告與單核心當前任務指標 (TCB) 重置

第一步宣告核心 ID 循環變數，並在單核心環境下將「當前執行任務指標（`pxCurrentTCB`）」清空為 `NULL`。

```C
/*
 * 重置 tasks.c 內部的靜態變數狀態。
 * 這些變數通常在開機初始化時歸零。
 * 若應用程式需要在『不重啟 MCU 硬體』的情況下重新啟動排程器 (Restart Scheduler)，
 * 必須在呼叫 vTaskStartScheduler() 之前先呼叫此函數。
 */
void vTaskResetState( void )
{
    BaseType_t xCoreID;

    /* 1. 重置當前任務指標 (TCB Pointer)：
     * 在單核心環境下，將指向當前正在執行任務的 pxCurrentTCB 清為 NULL。
     * 代表目前沒有任何 Task 正在 Running 狀態。 */
    #if ( configNUMBER_OF_CORES == 1 )
    {
        pxCurrentTCB = NULL;
    }
    #endif /* #if ( configNUMBER_OF_CORES == 1 ) */
```

#### 120.2 待清理刪除任務數與 POSIX Errno 重置

重置待清理的垃圾任務計數器以及線程安全的 POSIX 錯誤碼全域變數。

```C
/* 2. 重置待回收刪除任務數：
     * uxDeletedTasksWaitingCleanUp 記錄已被呼叫 vTaskDelete() 但還沒被 Idle Task 釋放記憶體的 Task 數量。
     * 重置時將其歸零。 */
    #if ( INCLUDE_vTaskDelete == 1 )
    {
        uxDeletedTasksWaitingCleanUp = ( UBaseType_t ) 0U;
    }
    #endif /* #if ( INCLUDE_vTaskDelete == 1 ) */

    /* 3. 重置 FreeRTOS POSIX Errno：
     * 若啟用了 POSIX 相容錯誤碼功能，清空線程區域錯誤碼。 */
    #if ( configUSE_POSIX_ERRNO == 1 )
    {
        FreeRTOS_errno = 0;
    }
    #endif /* #if ( configUSE_POSIX_ERRNO == 1 ) */
```

#### 120.3 核心排程器狀態與 Tick 系統全域變數重置

重置任務總數、系統 Tick 計數器、最高就緒優先權、排程器運行標誌以及各核心的 Yield 搶佔標記。

```C
/* 4. 重置任務總數與 Tick 計數系統：
     * - uxCurrentNumberOfTasks: 當前已建立的 Task 總數歸零。
     * - xTickCount: 恢復為初始 Tick 數值 (通常為 0，但在測試溢位時可能為非 0)。
     * - uxTopReadyPriority: 最高 Ready 優先權恢復為 Lowest (tskIDLE_PRIORITY = 0)。
     * - xSchedulerRunning: 排程器運行狀態設為 pdFALSE (未運行)。
     * - xPendedTicks: 暫停排程器期間累積的未處理 Tick 數歸零。 */
    uxCurrentNumberOfTasks = ( UBaseType_t ) 0U;
    xTickCount = ( TickType_t ) configINITIAL_TICK_COUNT;
    uxTopReadyPriority = tskIDLE_PRIORITY;
    xSchedulerRunning = pdFALSE;
    xPendedTicks = ( TickType_t ) 0U;

    /* 5. 遍歷所有 CPU 核心，重置搶佔/Yield 請求旗標 (xYieldPendings) */
    for( xCoreID = 0; xCoreID < configNUMBER_OF_CORES; xCoreID++ )
    {
        xYieldPendings[ xCoreID ] = pdFALSE;
    }
```

#### 120.4 溢位次數、任務序號、解鎖時間與暫停狀態重置

重置 Tick 溢位計數器、自動遞增的 Task 流水號、下一任務解鎖時間點，以及排程器暫停計數器。

```C
/* 6. 重置時間追蹤與排程器暫停狀態：
     * - xNumOfOverflows: Tick 計數器發生 32-bit 溢位的累計次數歸零。
     * - uxTaskNumber: 給除錯器 (Debugger) 使用的 Task 遞增流水號歸零。
     * - xNextTaskUnblockTime: 最近一個即將醒來的 Blocked Task 喚醒時間歸零。
     * - uxSchedulerSuspended: vTaskSuspendAll() 的巢狀暫停計數器歸零。 */
    xNumOfOverflows = ( BaseType_t ) 0;
    uxTaskNumber = ( UBaseType_t ) 0U;
    xNextTaskUnblockTime = ( TickType_t ) 0U;

    uxSchedulerSuspended = ( UBaseType_t ) 0U;
```

#### 120.5 運行時間統計 (Run-Time Stats) 重置與退出

若開啟了 CPU 使用率統計功能，重置所有核心的最後切入時間與總運行時間記錄。

```C
/* 7. 重置 CPU 使用率與運行時間統計 (Profiling) 變數：
     * 將所有 CPU 核心的 ulTaskSwitchedInTime (切入時間點) 與 ulTotalRunTime (總運行時間) 清零。 */
    #if ( configGENERATE_RUN_TIME_STATS == 1 )
    {
        for( xCoreID = 0; xCoreID < configNUMBER_OF_CORES; xCoreID++ )
        {
            ulTaskSwitchedInTime[ xCoreID ] = 0U;
            ulTotalRunTime[ xCoreID ] = 0U;
        }
    }
    #endif /* #if ( configGENERATE_RUN_TIME_STATS == 1 ) */
}
/*-----------------------------------------------------------*/
```

為什麼普通系統很少呼叫 `vTaskResetState()`？

1. **內核設計哲學**：FreeRTOS 通常作為嵌入式韌體的核心，開機後 `main()` 初始化完畢呼叫 `vTaskStartScheduler()`，系統就會一直跑在無窮迴圈中，直到斷電。
    
2. **記憶體與 List 未清除警告**：
    
    - `vTaskResetState()` **只重置變數**，它**不會**去釋放已動態配置的 TCB 記憶體或 Queue/Semaphore，也**不會**去清空 `pxReadyTasksLists` 等鏈結串列！
        
    - 如果你在系統運作到一半時貿然呼叫 `vTaskResetState()`，而不手動銷毀舊任務與動態 Heap，會造成嚴重的 **Memory Leak（記憶體洩漏）** 與 **Dangling Pointers（野指標）**。
        

什麼時候「必須」呼叫它？

- **單元測試 (Unit Testing Frameworks)**：在寫 FreeRTOS 內核單元測試（如 CUnity、GoogleTest）時，每個 Test Case 跑完後需要重置 RTOS 狀態，以便下一個 Test Case 能從乾淨的環境重新啟動 Scheduler。
    
- **應用層軟體重啟 (Application Soft Reset)**：當系統發送「軟體重置指令」，應用層自行手動刪除所有 Task、清空 List 後，在第二次呼叫 `vTaskStartScheduler()` 前呼叫。