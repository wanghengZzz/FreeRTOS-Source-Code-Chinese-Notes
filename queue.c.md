
## Implementations of all Functions (303 to ...end)

### 1. *xQueueGenericReset*

`xQueueGenericReset()` 是 FreeRTOS 佇列（Queue）體系中的**靈魂初始化與重置器**。

無論是你在創建全新 Queue、Semaphore 或 Mutex（呼叫 `xQueueCreate` 系列），還是對一個已存在的 Queue 執行重置動作（呼叫 `xQueueReset()`），底層**通通都是由這個函數在進行記憶體指標的校正與狀態重置**。


#### 1.1 變數初始化、安全斷言與乘法溢位防禦檢查

第一步檢查 Queue 指標與項次規格，最重要的是利用 `SIZE_MAX / length >= item_size` 來防止 32-bit 乘法溢位，接著進入臨界區保護。

```C
BaseType_t xQueueGenericReset( QueueHandle_t xQueue,
                               BaseType_t xNewQueue )
{
    BaseType_t xReturn = pdPASS;
    Queue_t * const pxQueue = xQueue;

    traceENTER_xQueueGenericReset( xQueue, xNewQueue );

    /* 1. 安全斷言：傳入的 Queue 指標絕對不能為 NULL */
    configASSERT( pxQueue );

    /* 2. 防禦性條件檢查：
     * - Queue 指標有效
     * - Queue 長度 >= 1
     * - 核心溢位防護：(SIZE_MAX / uxLength) >= uxItemSize 
     *   確保 (uxLength * uxItemSize) 算出總位元組時，絕不會發生 32-bit 整數溢位！ */
    if( ( pxQueue != NULL ) &&
        ( pxQueue->uxLength >= 1U ) &&
        /* Check for multiplication overflow. */
        ( ( SIZE_MAX / pxQueue->uxLength ) >= pxQueue->uxItemSize ) )
    {
        /* 3. 進入臨界區（關閉中斷）：確保變數與雙向鏈結串列修改的原子性 */
        taskENTER_CRITICAL();
        {
```

#### 1.2 環形緩衝區 (Ring Buffer) 指標重置與鎖狀態歸零

在中斷保護下，重置佇列尾端位址、當前訊息計數、寫入指標、讀取指標以及 Rx/Tx 佇列鎖狀態。

```C
/* 4. 重置環形緩衝區 (Ring Buffer) 記憶體指標：
             * - pcTail: 指向 Queue 緩衝區的最末端邊界 (pcHead + 總記憶體大小)
             * - uxMessagesWaiting: 當前等待讀取的訊息數歸零
             * - pcWriteTo: 下一次寫入資料的位置，重置回緩衝區開頭 (pcHead) */
            pxQueue->u.xQueue.pcTail = pxQueue->pcHead + ( pxQueue->uxLength * pxQueue->uxItemSize );
            pxQueue->uxMessagesWaiting = ( UBaseType_t ) 0U;
            pxQueue->pcWriteTo = pxQueue->pcHead;

            /* 5. 核心指標精髓 (pcReadFrom)：
             * FreeRTOS 的讀取邏輯是『先移動指標，再讀資料』。
             * 將 pcReadFrom 設為『最後一個 Element 的位址』，這樣當第一個讀取動作發生時，
             * 指標往後挪動一個 ItemSize，剛好就會指向開頭 (pcHead)！ */
            pxQueue->u.xQueue.pcReadFrom = pxQueue->pcHead + ( ( pxQueue->uxLength - 1U ) * pxQueue->uxItemSize );

            /* 6. 重置 Queue Lock 計數器為解鎖狀態 (queueUNLOCKED = -1) */
            pxQueue->cRxLock = queueUNLOCKED;
            pxQueue->cTxLock = queueUNLOCKED;
```

#### 1.3 分流處理：新建佇列初始化 vs 舊佇列重置與寫入任務喚醒

判斷是「第一次新建」還是「重置舊佇列」。若為重置舊佇列，將會把原本因 Queue 滿而 Block 的寫入任務喚醒；若為新建佇列，則初始化等待清單。

```C
/* 7. 判斷處理模式：xNewQueue 參數 */
            if( xNewQueue == pdFALSE )
            {
                /* 情況 A：這是一個『已經在使用中』的 Queue 正在被呼叫 xQueueReset() 重置！
                 *
                 * 【邏輯解析】：
                 * - 等待『讀取』的任務 (xTasksWaitingToReceive)：繼續保持 Block，因為重置後 Queue 依然是空的。
                 * - 等待『寫入』的任務 (xTasksWaitingToSend)：因為 Queue 被清空有了空間，原本寫不進去而睡著的任務
                 *   現在可以醒來寫了！因此嘗試喚醒第一個高優先權的等待寫入任務。 */
                if( listLIST_IS_EMPTY( &( pxQueue->xTasksWaitingToSend ) ) == pdFALSE )
                {
                    /* 從等待寫入清單中移除任務並拉回 ReadyList */
                    if( xTaskRemoveFromEventList( &( pxQueue->xTasksWaitingToSend ) ) != pdFALSE )
                    {
                        /* 若喚醒的任務優先權高於當前任務，且開啟了搶佔，觸發上下文切換 (Yield) */
                        queueYIELD_IF_USING_PREEMPTION();
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
                /* 情況 B：這是一個『剛剛動態/靜態配置好』的新 Queue (xNewQueue == pdTRUE)
                 * 初始化該 Queue 專屬的等待寫入 (Send) 與等待讀取 (Receive) 雙向鏈結串列 Header */
                vListInitialise( &( pxQueue->xTasksWaitingToSend ) );
                vListInitialise( &( pxQueue->xTasksWaitingToReceive ) );
            }
        }
        /* 8. 退出臨界區，恢復中斷 */
        taskEXIT_CRITICAL();
    }
    else
    {
        /* 參數無效或發生乘法溢位，標記失敗 */
        xReturn = pdFAIL;
    }
```

#### 1.4 補充

##### 1.4.1 為什麼 `pcReadFrom` 要指向「最後一個 Item 的位址」？

這是 FreeRTOS 佇列控制結構中最常被問到的面試題之一！

- **寫入邏輯 (`xQueueSend`)**：先將資料複製到 `pcWriteTo` 指向的位置，然後再將 `pcWriteTo` 加上 `uxItemSize` 移至下一個空位。
    
- **讀取邏輯 (`xQueueReceive`)**：先將 `pcReadFrom` 加上 `uxItemSize` 移至有資料的位置，然後才從該位置複製資料出來！
    

**設計好處**： 若 `pcReadFrom` 重置時直接設在 `pcHead`，那第一次讀取時 `pcReadFrom` 加上 `uxItemSize` 就會**錯過第 0 個資料直接跑到第 1 個資料去**。所以初始化時把它「故意放在 `pcHead` 前一個位置（即環形緩衝區的最末端）」，第一次讀取時加上 `uxItemSize` 恰好自然溢位迴圈（Wrap around）回 `pcHead`！

##### 1.4.2 重置 Queue 時，喚醒「等待讀取」與「等待寫入」任務的差異？

當呼叫 `xQueueReset(xQueue)` 時：

|**任務等待類型**|**排隊串列 (Event List)**|**重置後的處理方式**|**理由**|
|---|---|---|---|
|**Waiting to Receive**<br><br>  <br><br>(想讀資料的任務)|`xTasksWaitingToReceive`|**繼續保持睡眠 (Blocked)**|重置後的 Queue 是空的，讀取者醒來依然讀不到資料，因此不需喚醒。|
|**Waiting to Send**<br><br>  <br><br>(想寫資料的任務)|`xTasksWaitingToSend`|**喚醒 1 個最高優先權任務**|重置前 Queue 可能是滿的，導致寫入者卡住。現在 Queue 被清空了，立刻喚醒 1 個 Task 來寫資料！|

### 2. *xQueueGenericCreateStatic*

`xQueueGenericCreateStatic()` 是 FreeRTOS 中用於靜態建立佇列（Queue）、信號量（Semaphore）與互斥鎖（Mutex）的核心底層通用 API。

當系統開啟靜態記憶體配置（`configSUPPORT_STATIC_ALLOCATION == 1`）時，你不必（也不能）使用 Heap 區塊（`pvPortMalloc`）。你必須在編譯時期或應用層預先準備好兩塊 RAM 空間：

1. **控制塊空間（`StaticQueue_t`）**：存放 Queue 的狀態指標與等待列隊。
    
2. **資料緩衝區（`pucQueueStorage`）**：存放 Queue 實際要傳遞的資料項（若 Item Size > 0）。
    

這個 API 負責驗證應用層傳進來的記憶體空間是否合法、標記記憶體配置屬性（靜態），並啟動佇列結構體的初始化。

#### 2.1 條件編譯、參數合法性邏輯驗證與結構體大小防護斷言

此區塊檢查靜態記憶體指標、確保資料緩衝區與 Item Size 的邏輯匹配，並驗證外置類型 `StaticQueue_t` 與內核 `Queue_t` 大小一致。

```C
#if ( configSUPPORT_STATIC_ALLOCATION == 1 )

    QueueHandle_t xQueueGenericCreateStatic( const UBaseType_t uxQueueLength,
                                             const UBaseType_t uxItemSize,
                                             uint8_t * pucQueueStorage,
                                             StaticQueue_t * pxStaticQueue,
                                             const uint8_t ucQueueType )
    {
        Queue_t * pxNewQueue = NULL;

        traceENTER_xQueueGenericCreateStatic( uxQueueLength, uxItemSize, pucQueueStorage, pxStaticQueue, ucQueueType );

        /* 1. 安全斷言：傳入的 StaticQueue_t (控制塊記憶體) 指標不可為 NULL */
        configASSERT( pxStaticQueue );

        /* 2. 參數合法性邏輯檢查：
         * - uxQueueLength > 0：佇列深度必須大於 0。
         * - pxStaticQueue != NULL：控制塊指標必須有效。
         * - pucQueueStorage 與 uxItemSize 的互斥邏輯檢查：
         *   ▸ 若 uxItemSize > 0 (一般佇列)，則 pucQueueStorage 絕不能為 NULL (必須提供實體資料緩衝區)。
         *   ▸ 若 uxItemSize == 0 (信號量/互斥鎖)，則 pucQueueStorage 必須為 NULL (不需資料儲存區)。 */
        if( ( uxQueueLength > ( UBaseType_t ) 0 ) &&
            ( pxStaticQueue != NULL ) &&
            ( !( ( pucQueueStorage != NULL ) && ( uxItemSize == 0U ) ) ) &&
            ( !( ( pucQueueStorage == NULL ) && ( uxItemSize != 0U ) ) ) )
        {
            #if ( configASSERT_DEFINED == 1 )
            {
                /* 3. 靜態結構體大小一致性檢查：
                 * 確保暴露給使用者宣告用的 StaticQueue_t 占用的記憶體大小，
                 * 100% 等於內核內部實際使用的 Queue_t 結構體大小。
                 * 防止因為編譯器對齊 (Alignment) 設定不同導致記憶體越界踩踏！ */
                volatile size_t xSize = sizeof( StaticQueue_t );

                configASSERT( xSize == sizeof( Queue_t ) );
                ( void ) xSize; /* 避免在未開 configASSERT 時產生 unused variable 警告 */
            }
            #endif /* configASSERT_DEFINED */
```

#### 2.2 指標型態轉換、靜態記憶體標記與佇列初始化完成

將靜態記憶體指標轉型，並設置靜態配置旗標（防範刪除時誤釋放 Heap），最後交由 `prvInitialiseNewQueue` 完成內部指標綁定。

```C
/* 4. 指標轉型：將應用層傳入的 StaticQueue_t 記憶體位址轉型為內核內部的 Queue_t 指標 */
            /* MISRA Ref 11.3.1 [Misaligned access] */
            pxNewQueue = ( Queue_t * ) pxStaticQueue;

            #if ( configSUPPORT_DYNAMIC_ALLOCATION == 1 )
            {
                /* 5. 標記靜態配置旗標：
                 * 若系統同時開啟了動態與靜態配置，將 ucStaticallyAllocated 設為 pdTRUE。
                 * 當未來呼叫 vQueueDelete() 時，內核就知道這是靜態記憶體，絕不能呼叫 vPortFree()！ */
                pxNewQueue->ucStaticallyAllocated = pdTRUE;
            }
            #endif /* configSUPPORT_DYNAMIC_ALLOCATION */

            /* 6. 呼叫內部初始化函數：
             * 設定 Queue 類型 (Queue / Mutex / Semaphore)、綁定緩衝區並呼叫 xQueueGenericReset 清空指標 */
            prvInitialiseNewQueue( uxQueueLength, uxItemSize, pucQueueStorage, ucQueueType, pxNewQueue );
        }
        else
        {
            /* 參數檢查失敗，觸發斷言 */
            configASSERT( pxNewQueue );
            mtCOVERAGE_TEST_MARKER();
        }

        traceRETURN_xQueueGenericCreateStatic( pxNewQueue );

        /* 7. 回傳建立完成的 Queue Handle (本質上就是 pxNewQueue 指標) */
        return pxNewQueue;
    }

#endif /* configSUPPORT_STATIC_ALLOCATION */
```

#### 2.3 為什麼檢查 `pucQueueStorage` 與 `uxItemSize`？

在 FreeRTOS 中，Queue 的底層結構同時也用於實現 **Semaphore（信號量）** 與 **Mutex（互斥鎖）**。

| **物件類型**              | **uxItemSize (單個元素大小)** | **pucQueueStorage (資料緩衝區)** | **邏輯關係**                                               |
| --------------------- | ----------------------- | --------------------------- | ------------------------------------------------------ |
| **一般 Queue**          | `> 0` (如 `sizeof(int)`) | **必須提供** (非 `NULL`)         | 需要實際的 SRAM 空間存放佇列訊息 payload。                           |
| **Semaphore / Mutex** | `== 0`                  | **必須為 `NULL`**              | 只關心「數量/鎖狀態」（紀錄於 `uxMessagesWaiting`），不傳遞資料，因此不需要資料緩衝區。 |
程式碼中的防禦檢查：

```C
( !( ( pucQueueStorage != NULL ) && ( uxItemSize == 0U ) ) ) &&
( !( ( pucQueueStorage == NULL ) && ( uxItemSize != 0U ) ) )
```

這段邏輯完美確保了：**「有 Payload 就必須給 Storage，沒 Payload 就絕對不能給 Storage」**，避免使用者傳入矛盾的參數！
#### 2.4 為什麼要有 `StaticQueue_t` 與 `configASSERT( xSize == sizeof( Queue_t ) )`？

為了維護封裝性（Encapsulation），FreeRTOS 將真正的 `struct QueueDefinition`（即 `Queue_t`）隱藏在 `queue.c` 內部，使用者在 `main.c` 是看不到 `Queue_t` 的內部欄位的。

但當使用者要**靜態宣告**記憶體時，必須知道要分配多少位元組！

- 因此 FreeRTOS 在 `FreeRTOS.h` 提供了一個填充用結構體 `StaticQueue_t`，裡面宣告了等長的空陣列，專門給應用層做靜態記憶體佔位。
    
- `configASSERT(sizeof(StaticQueue_t) == sizeof(Queue_t))` 則是一道安檢門，確保內核改版或編譯器 Padding 調整時，`StaticQueue_t` 的空間大小依然與內部的 `Queue_t` **完全一模一樣**，避免發生嚴重的記憶體覆蓋（Memory Corruption）。


#### 2.5 `ucStaticallyAllocated` 旗標的生命安全價值

當系統同時開啟動態與靜態配置（`configSUPPORT_STATIC_ALLOCATION == 1` 且 `configSUPPORT_DYNAMIC_ALLOCATION == 1`）時：

- 使用者可以呼叫 `vQueueDelete(xQueue)` 來銷毀佇列。
    
- `vQueueDelete()` 內部會檢查 `if( pxQueue->ucStaticallyAllocated == pdFALSE )`：
    
    - **動態佇列**：呼叫 `vPortFree()` 釋放 Heap。
        
    - **靜態佇列**：**跳過 `vPortFree()`**，只清除狀態。否則對 Static RAM 執行 `free()` 會直接導致系統崩潰（HardFault）！

### 3. *xQueueGenericGetStaticBuffers*

`xQueueGenericGetStaticBuffers()` 是 FreeRTOS 中專門用來「回查（Query）佇列或信號量底層靜態記憶體緩衝區位址」的 API。

在開啓靜態記憶體配置（`configSUPPORT_STATIC_ALLOCATION == 1`）的前提下，當你需要對系統進行記憶體監控、診斷、單元測試，或是想知道某個 Queue/Semaphore/Mutex 的 `StaticQueue_t` 控制塊與 Payload 緩衝區到底放在 SRAM 的哪裡時，這個函數就能幫你把這兩個底層指標「提領」出來。

#### 3.1 斷言檢查與雙模共存（Static + Dynamic）處理解析

在系統同時開啟動態與靜態配置時，函數會先透過 `ucStaticallyAllocated` 旗標確認該 Queue 到底是不是靜態建立的。

```C
#if ( configSUPPORT_STATIC_ALLOCATION == 1 )

    /*
     * 取得佇列 (Queue) 或信號量 (Semaphore) 的靜態記憶體緩衝區指標。
     * - xQueue: 欲查詢的 Queue Handle
     * - ppucQueueStorage: 傳出參數，用以接收 Payload 記憶體緩衝區首位址 (*ppucQueueStorage)
     * - ppxStaticQueue: 傳出參數，用以接收 StaticQueue_t 控制塊首位址 (*ppxStaticQueue)
     * 
     * 回傳值：pdTRUE 代表成功取得靜態位址；pdFALSE 代表該 Queue 是動態配置而成的，無靜態緩衝區。
     */
    BaseType_t xQueueGenericGetStaticBuffers( QueueHandle_t xQueue,
                                              uint8_t ** ppucQueueStorage,
                                              StaticQueue_t ** ppxStaticQueue )
    {
        BaseType_t xReturn;
        Queue_t * const pxQueue = xQueue;

        traceENTER_xQueueGenericGetStaticBuffers( xQueue, ppucQueueStorage, ppxStaticQueue );

        /* 1. 安全斷言檢查：
         * - pxQueue 不可為 NULL。
         * - ppxStaticQueue (傳出控制塊指標的指標) 絕不可為 NULL。 */
        configASSERT( pxQueue );
        configASSERT( ppxStaticQueue );

        #if ( configSUPPORT_DYNAMIC_ALLOCATION == 1 )
        {
            /* 2. 雙模模式 (Static + Dynamic 均開啟)：
             * 因為 Queue 有可能是透過 xQueueCreate() (動態) 建立的，
             * 所以必須先檢查 ucStaticallyAllocated 旗標是否為 pdTRUE。 */
            if( pxQueue->ucStaticallyAllocated == ( uint8_t ) pdTRUE )
            {
                /* 情況 A：確實為靜態配置的 Queue */
                if( ppucQueueStorage != NULL )
                {
                    /* pcHead 存放的是 Queue Payload 緩衝區的開頭位址 */
                    *ppucQueueStorage = ( uint8_t * ) pxQueue->pcHead;
                }

                /* 將內核 Queue_t 結構體指標轉型寫回給 StaticQueue_t 指標 */
                /* MISRA Ref 11.3.1 [Misaligned access] */
                /* coverity[misra_c_2012_rule_11_3_violation] */
                *ppxStaticQueue = ( StaticQueue_t * ) pxQueue;
                xReturn = pdTRUE;
            }
            else
            {
                /* 情況 B：這是動態從 Heap 配置的 Queue，無靜態緩衝區可供提取 */
                xReturn = pdFALSE;
            }
        }
```

#### 3.2 純靜態模式（Pure Static）處理與結果回傳

若系統關閉了動態配置（`configSUPPORT_DYNAMIC_ALLOCATION == 0`），代表系統內「所有的 Queue 必然都是靜態建立的」，因此不需檢查旗標，直接回傳指標。

```C
#else /* configSUPPORT_DYNAMIC_ALLOCATION == 0 */
        {
            /* 3. 純靜態模式：
             * 系統已關閉 Heap 配置，所有 Queue 必然都是靜態建立的，不需檢查旗標。 */
            if( ppucQueueStorage != NULL )
            {
                *ppucQueueStorage = ( uint8_t * ) pxQueue->pcHead;
            }

            *ppxStaticQueue = ( StaticQueue_t * ) pxQueue;
            xReturn = pdTRUE;
        }
        #endif /* configSUPPORT_DYNAMIC_ALLOCATION */

        traceRETURN_xQueueGenericGetStaticBuffers( xReturn );

        /* 4. 回傳查詢結果 (pdTRUE / pdFALSE) */
        return xReturn;
    }

#endif /* configSUPPORT_STATIC_ALLOCATION */
```

#### 3.3 為什麼 `ppucQueueStorage` 允許傳入 `NULL`？

注意到程式碼中的這段防禦設計：

```C
if( ppucQueueStorage != NULL )
{
    *ppucQueueStorage = ( uint8_t * ) pxQueue->pcHead;
}
```

這樣設計有兩個非常實用的原因：

1. **使用者只關心控制塊**：呼叫者有時只想取得 `StaticQueue_t` 位址，不需要 Payload 位址，此時可以對 `ppucQueueStorage` 直接傳入 `NULL`。
    
2. **Semaphore / Mutex 的特性**：Semaphore 和 Mutex 也是基於 Queue 結構實現的，但它們的 `uxItemSize == 0`，沒有 Payload 緩衝區（`pcHead` 可能為 `NULL`）。允許 `ppucQueueStorage` 為 `NULL` 可以提高 API 的容錯度與通用性。

#### 3.4 `pxQueue->pcHead` 的角色扮演

在 FreeRTOS 的 `Queue_t` 結構中：

- **`pcHead`**：指向 Queue **儲存 Payload 資料陣列**的最開頭記憶體位址（在靜態配置時，這就是你當初傳給 `xQueueCreateStatic` 的 `pucQueueStorage` 陣列）。
    
- **`pxQueue` 本身**：就是 `Queue_t`（對外顯示為 `StaticQueue_t`）**控制塊結構體**的開頭位址。
    

因此：

- `*ppucQueueStorage = pxQueue->pcHead;` 提領的是 **資料區塊**。
    
- `*ppxStaticQueue = (StaticQueue_t *) pxQueue;` 提領的是 **控制區塊**。

### 4. *xQueueGenericCreate*

`xQueueGenericCreate()` 是 FreeRTOS 中用於動態建立佇列（Queue）、信號量（Semaphore）與互斥鎖（Mutex）的核心底層 API。

與靜態建立 API（`xQueueGenericCreateStatic`）需要使用者自行準備記憶體不同，當系統開啟動態記憶體配置（`configSUPPORT_DYNAMIC_ALLOCATION == 1`）時，這個 API 會直接向 FreeRTOS 的 Heap 記憶體堆疊（透過 `pvPortMalloc`）申請所需的 RAM 空間。

#### 4.1 條件編譯、變數宣告與防禦性雙重算術溢位檢查

區塊重點在於計算記憶體總需求時，使用了 `SIZE_MAX` 進行「乘法」與「加法」的雙重溢位防禦，避免惡意或錯誤的長度參數踩爛記憶體。

```C
#if ( configSUPPORT_DYNAMIC_ALLOCATION == 1 )

    QueueHandle_t xQueueGenericCreate( const UBaseType_t uxQueueLength,
                                       const UBaseType_t uxItemSize,
                                       const uint8_t ucQueueType )
    {
        Queue_t * pxNewQueue = NULL;
        size_t xQueueSizeInBytes;
        uint8_t * pucQueueStorage;

        traceENTER_xQueueGenericCreate( uxQueueLength, uxItemSize, ucQueueType );

        /* 1. 嚴格的防禦性邏輯與雙重算術溢位檢查：
         * - uxQueueLength > 0：佇列深度必須大於 0。
         * - 乘法溢位檢查：(SIZE_MAX / uxQueueLength) >= uxItemSize
         *   確保 (uxQueueLength * uxItemSize) 算出 Payload 總長度時不會發生成員溢位。
         * - 加法溢位檢查：(SIZE_MAX - sizeof(Queue_t)) >= Payload 總長度
         *   確保加上 Queue_t 控制塊大小後，申請的總記憶體依然在 SIZE_MAX 範圍內。 */
        if( ( uxQueueLength > ( UBaseType_t ) 0 ) &&
            /* Check for multiplication overflow. */
            ( ( SIZE_MAX / uxQueueLength ) >= uxItemSize ) &&
            /* Check for addition overflow. */
            /* MISRA Ref 14.3.1 [Configuration dependent invariant] */
            /* coverity[misra_c_2012_rule_14_3_violation] */
            ( ( SIZE_MAX - sizeof( Queue_t ) ) >= ( size_t ) ( ( size_t ) uxQueueLength * ( size_t ) uxItemSize ) ) )
        {
```

#### 4.2 單一區塊 Heap 動態配置與 Payload 位址偏移計算

計算總需求大小後，呼叫一次 `pvPortMalloc` 同時切出控制塊與 Payload，並計算指標偏移，最後設定動態標記。

```C
/* 2. 計算 Payload (資料區塊) 總位元組大小 */
            xQueueSizeInBytes = ( size_t ) ( ( size_t ) uxQueueLength * ( size_t ) uxItemSize );

            /* 3. 一次性連續記憶體配置 (Single Block Allocation)：
             * 呼叫 pvPortMalloc 一次性申請『Queue 控制塊 (Queue_t) + Payload 緩衝區』所需的總記憶體！
             * 這種做法能減少 Heap 碎裂並提升記憶體釋放效率。 */
            /* MISRA Ref 11.5.1 [Malloc memory assignment] */
            /* coverity[misra_c_2012_rule_11_5_violation] */
            pxNewQueue = ( Queue_t * ) pvPortMalloc( sizeof( Queue_t ) + xQueueSizeInBytes );

            if( pxNewQueue != NULL )
            {
                /* 4. 計算 Payload 儲存區的起始位址：
                 * 將指標指向剛配置好的記憶體開頭，並跳過 sizeof(Queue_t) 的控制塊大小，
                 * 緊接著的後半段記憶體就是 Payload 資料區！ */
                pucQueueStorage = ( uint8_t * ) pxNewQueue;
                pucQueueStorage += sizeof( Queue_t );

                #if ( configSUPPORT_STATIC_ALLOCATION == 1 )
                {
                    /* 5. 標記動態配置屬性：
                     * 若系統開啟雙模配置 (Static + Dynamic)，將 ucStaticallyAllocated 標記為 pdFALSE。
                     * 告知 vQueueDelete() 未來刪除此佇列時『必須呼叫 vPortFree()』釋放 Heap。 */
                    pxNewQueue->ucStaticallyAllocated = pdFALSE;
                }
                #endif /* configSUPPORT_STATIC_ALLOCATION */
```

#### 4.3 佇列初始化、例外處理與結果回傳

在中斷安全的環境下呼叫 `prvInitialiseNewQueue` 配置內部環形指標，若記憶體不足則回傳 `NULL`。

```C
/* 6. 呼叫內部通用初始化函數：
                 * 設定 Queue 類型、綁定內部指標 (pcHead/pcTail/pcWriteTo) 並呼叫 xQueueGenericReset 歸零狀態 */
                prvInitialiseNewQueue( uxQueueLength, uxItemSize, pucQueueStorage, ucQueueType, pxNewQueue );
            }
            else
            {
                /* 記憶體不足，發送 Trace 失敗事件 */
                traceQUEUE_CREATE_FAILED( ucQueueType );
                mtCOVERAGE_TEST_MARKER();
            }
        }
        else
        {
            /* 參數無效或發生算術溢位，觸發斷言 */
            configASSERT( pxNewQueue );
            mtCOVERAGE_TEST_MARKER();
        }

        traceRETURN_xQueueGenericCreate( pxNewQueue );

        /* 7. 回傳建立完成的 Queue Handle (成功傳回指標，Heap 不足則傳回 NULL) */
        return pxNewQueue;
    }

#endif /* configSUPPORT_DYNAMIC_ALLOCATION */
```

### 5. *prvInitialiseNewQueue*

`prvInitialiseNewQueue()` 是 FreeRTOS 佇列模組中的**核心私有初始化函數（Internal Worker）**。

無論你是透過動態配置（`xQueueGenericCreate`）還是靜態配置（`xQueueGenericCreateStatic`）來建立佇列、信號量（Semaphore）或互斥鎖（Mutex），**最終都會匯流到這個函數來完成內部欄位的填寫**。

它最巧妙之處，在於處理 `uxItemSize == 0`（即信號量/鎖）時，對 `pcHead` 指標進行的「防撞硬體位址技巧」，以及銜接 `xQueueGenericReset()` 完成雙向鏈結串列（List）的初始化。

#### 5.1 未使用參數消除與 `pcHead` 特殊指標綁定技巧

這段程式碼處理 `ucQueueType` 的編譯器警告，並針對「一般 Queue」與「Semaphore/Mutex」進行 `pcHead` 指標的綁定。

```C
static void prvInitialiseNewQueue( const UBaseType_t uxQueueLength,
                                   const UBaseType_t uxItemSize,
                                   uint8_t * pucQueueStorage,
                                   const uint8_t ucQueueType,
                                   Queue_t * pxNewQueue )
{
    /* 1. 消除編譯器警告：
     * 若未開啟 configUSE_TRACE_FACILITY，ucQueueType 參數不會被用到，
     * 透過 (void) 強制轉換避免 warning: unused parameter。 */
    ( void ) ucQueueType;

    /* 2. 設定 pcHead 指標 (Payload 開頭位址)：
     * 分為「無資料區 (Semaphore)」與「有資料區 (Queue)」兩種處理邏輯。 */
    if( uxItemSize == ( UBaseType_t ) 0 )
    {
        /* 情況 A：uxItemSize == 0 (信號量或互斥鎖，不需實體資料儲存區)
         * 【關鍵設計】：這裡絕對『不能』將 pcHead 設為 NULL！
         * 因為在 FreeRTOS 的 Mutex 實現中，pcHead 被用來當作辨識 Mutex 的關鍵 Key (NULL 代表特殊用途)。
         * 為了避免邏輯混淆，這裡將 pcHead 直接指向 pxNewQueue 自己！
         * 這是一個安全且有效的記憶體位址，且因為 uxItemSize == 0，系統永遠不會去 dereference 它。 */
        pxNewQueue->pcHead = ( int8_t * ) pxNewQueue;
    }
    else
    {
        /* 情況 B：uxItemSize > 0 (一般佇列)
         * 將 pcHead 正式指向傳入的 Payload 緩衝區首位址 (pucQueueStorage) */
        pxNewQueue->pcHead = ( int8_t * ) pucQueueStorage;
    }
```

#### 5.2 規格寫入與關鍵重置（xQueueGenericReset）呼叫

將佇列深度與單一元素大小填入結構體，隨後呼叫 `xQueueGenericReset` 進行指標計算與 Event List 初始化。

```C
/* 3. 填入佇列基礎規格參數 */
    pxNewQueue->uxLength = uxQueueLength;
    pxNewQueue->uxItemSize = uxItemSize;

    /* 4. 呼叫通用重置 API：
     * 傳入 pdTRUE 代表這是『全新建立』的 Queue，
     * 內部會執行 vListInitialise() 來初始化 xTasksWaitingToSend 與 xTasksWaitingToReceive 串列，
     * 並計算 pcTail, pcWriteTo, pcReadFrom 等環形指標。 */
    ( void ) xQueueGenericReset( pxNewQueue, pdTRUE );
```

#### 5.3 診斷追蹤、Queue Sets 擴充配置與 Trace 紀錄

針對系統開啟的擴充功能（Trace Facility / Queue Sets）設置對應欄位，最後發送 `traceQUEUE_CREATE` 事件給除錯工具（如 SystemView / Tracealyzer）。

```C
/* 5. 條件編譯擴充：追蹤設施 (Trace Facility) */
    #if ( configUSE_TRACE_FACILITY == 1 )
    {
        /* 記錄此 Queue 的型態 (Queue, Mutex, Counting Semaphore 等) 供除錯工具辨識 */
        pxNewQueue->ucQueueType = ucQueueType;
    }
    #endif /* configUSE_TRACE_FACILITY */

    /* 6. 條件編譯擴充：佇列集合 (Queue Sets) */
    #if ( configUSE_QUEUE_SETS == 1 )
    {
        /* 初始化 Queue Set 容器指標為 NULL (代表目前未納入任何 Queue Set) */
        pxNewQueue->pxQueueSetContainer = NULL;
    }
    #endif /* configUSE_QUEUE_SETS */

    /* 7. 發送 Trace 事件巨集 (供 SystemView / Tracealyzer 監控) */
    traceQUEUE_CREATE( pxNewQueue );
}
```

### 6. *prvInitialiseMutex*

`prvInitialiseMutex()` 是 FreeRTOS 中專門用來「將剛建立好的通用佇列（Queue）改裝升級為互斥鎖（Mutex）」的內部私有初始化函數。

在 FreeRTOS 的架構設計中，**Mutex 本質上就是一個長度為 1、單一元素大小為 0 (`uxQueueLength = 1`, `uxItemSize = 0`) 的特化 Queue**。

當通用建立函數（`prvInitialiseNewQueue`）將基礎結構體填好後，`prvInitialiseMutex()` 會進場覆寫與 Mutex 專屬功能相關的欄位——**特別是實現「優先權繼承（Priority Inheritance）」所需的 Task 追蹤指標，並將 Mutex 置於「可被取得（Unlocked/Available）」的初始狀態**。

#### 6.1 特化欄位覆寫、優先權繼承準備與狀態標記

```C
#if ( configUSE_MUTEXES == 1 )

    static void prvInitialiseMutex( Queue_t * pxNewQueue )
    {
        if( pxNewQueue != NULL )
        {
            /* 1. 覆寫通用 Queue 成員以符合 Mutex 特性：
             * 通用建立函數只會設定一般的 Queue 欄位，但 Mutex 需要額外的專屬機制。
             * 
             * pxNewQueue->u.xSemaphore.xMutexHolder:
             * 記錄『當前持有此 Mutex 的 Task TCB 指標』。
             * 剛建立時無人持有，故初始化為 NULL。
             * 這是實現『優先權繼承 (Priority Inheritance)』最重要的關鍵欄位！ */
            pxNewQueue->u.xSemaphore.xMutexHolder = NULL;

            /* 2. 標記佇列類型為 Mutex (queueQUEUE_IS_MUTEX) */
            pxNewQueue->uxQueueType = queueQUEUE_IS_MUTEX;

            /* 3. 重置遞迴呼叫計數器：
             * 如果這個 Mutex 被當作『遞迴互斥鎖 (Recursive Mutex)』使用，
             * uxRecursiveCallCount 用來記錄『同一個 Task 重複 Take 了幾次』。初始值歸 0。 */
            pxNewQueue->u.xSemaphore.uxRecursiveCallCount = 0;

            traceCREATE_MUTEX( pxNewQueue );
```

#### 6.2 初始鎖狀態推入（xQueueGenericSend）與例外處理

這段程式碼包含了 Mutex 與普通二元信號量最大的差異：**剛建立的 Mutex 必須是「預設開鎖（Available）」狀態**，因此直接對自己執行一次 `xQueueGenericSend`。

```C
/* 4. 將 Mutex 設為預設的『可被取得 (Available / Unlocked)』狀態：
             *
             * 【極度關鍵設計】：
             * 剛建立的 Mutex 必須是『解鎖狀態』！
             * 這裡呼叫 xQueueGenericSend 寫入一個 0 位元組的 Token，
             * 會將 Queue 的 uxMessagesWaiting 設為 1。
             * 這樣第一個呼叫 xSemaphoreTake() 的 Task 才能『立即成功拿到鎖』而不需要等待！ */
            ( void ) xQueueGenericSend( pxNewQueue, NULL, ( TickType_t ) 0U, queueSEND_TO_BACK );
        }
        else
        {
            /* 5. 記憶體配置失敗例外處置 (Trace 紀錄) */
            traceCREATE_MUTEX_FAILED();
        }
    }

#endif /* configUSE_MUTEXES */
```

#### 6.3 為什麼 Mutex 剛建立時要執行一次 `xQueueGenericSend`？

這是 **Mutex（互斥鎖）** 與 **Binary Semaphore（二元信號量）** 在初始化邏輯上最核心的差異：

|**特性**|**互斥鎖 (Mutex)**|**二元信號量 (Binary Semaphore)**|
|---|---|---|
|**初始狀態**|**預設開鎖 (`uxMessagesWaiting = 1`)**|**預設關閉 (`uxMessagesWaiting = 0`)**|
|**設計用意**|資源在剛建立時是**空閒**的，第一個來的 Task 應該**直接拿到鎖**（不需 wait）。|用於事件同步（Task 待命等待中斷觸發），剛建立時事件尚未發生，應該**卡住等待**。|
|**初始化動作**|呼叫 `xQueueGenericSend()` 塞入一個 Token|不做 Send，等待 ISR 或其他 Task 給予 Token|
因此，`prvInitialiseMutex()` 內部的 `xQueueGenericSend()` 實質效果就是將 `uxMessagesWaiting` 設定為 1，讓鎖變成「開放搶佔」的狀態。

### 7. *xQueueCreateMutex*

`xQueueCreateMutex()` 是 FreeRTOS 中用於「動態建立互斥鎖（Mutex）」的對外底層工廠函數。

這是一個非常高明且簡潔的「組合 API（Wrapper/Factory Function）」：它本身不重複撰寫複雜的記憶體配置與重置邏輯，而是把我們前面幾個章節學到的 **通用動態建立器（`xQueueGenericCreate`）** 與 **Mutex 特化初始化器（`prvInitialiseMutex`）** 串聯起來，完成一個完美的「兩階段工廠模式（Two-Stage Factory Pattern）」。

#### 7.1 條件編譯、固定規格定義與通用 Heap 記憶體配置

區塊重點在於將 Mutex 的規格硬性設定為 `Length = 1` 且 `ItemSize = 0`，隨後呼叫 `xQueueGenericCreate` 進行動態記憶體申請。

```C
#if ( ( configUSE_MUTEXES == 1 ) && ( configSUPPORT_DYNAMIC_ALLOCATION == 1 ) )

    QueueHandle_t xQueueCreateMutex( const uint8_t ucQueueType )
    {
        QueueHandle_t xNewQueue;

        /* 1. 硬性指定 Mutex 的兩大核心規格：
         * - uxMutexLength = 1：互斥鎖的佇列深度永遠固定為 1 (同一時間只能被一個 Task 持有)。
         * - uxMutexSize = 0  ：互斥鎖不需要傳遞 Payload 資料，故單一元素大小為 0 位元組。 */
        const UBaseType_t uxMutexLength = ( UBaseType_t ) 1, uxMutexSize = ( UBaseType_t ) 0;

        traceENTER_xQueueCreateMutex( ucQueueType );

        /* 2. 第一階段工廠流程：向 Heap 申請通用 Queue 控制塊
         * 呼叫 xQueueGenericCreate(1, 0, ucQueueType)：
         * 內部會透過 pvPortMalloc 申請 sizeof(Queue_t) + 0 的記憶體空間，
         * 並呼叫 prvInitialiseNewQueue 完成底層鏈結串列 (List) 的基礎初始化。 */
        xNewQueue = xQueueGenericCreate( uxMutexLength, uxMutexSize, ucQueueType );
```

#### 7.2 Mutex 欄位升級改裝、開鎖與結果回傳

取得配置好的通用 Queue 指標後，立刻傳給 `prvInitialiseMutex` 改裝為真正的 Mutex（填入 `xMutexHolder` 並預設開鎖），最後回傳。

```C
/* 3. 第二階段工廠流程：改裝升級為 Mutex
         * 呼叫 prvInitialiseMutex：
         * 若 xNewQueue 不為 NULL (記憶體配置成功)，執行以下改裝：
         * ▸ 設定 xMutexHolder = NULL (清空鎖持有者)
         * ▸ 設定 uxQueueType = queueQUEUE_IS_MUTEX (標記類型)
         * ▸ 執行 xQueueGenericSend() 寫入預設 Token，將 Mutex 置於『Available (Unlocked)』開鎖狀態！ */
        prvInitialiseMutex( ( Queue_t * ) xNewQueue );

        traceRETURN_xQueueCreateMutex( xNewQueue );

        /* 4. 回傳建立完成的 Mutex 握柄 (即 QueueHandle_t 指標) */
        return xNewQueue;
    }

#endif /* configUSE_MUTEXES */
```

#### 7.3 為什麼這個 API 接收 `ucQueueType` 參數？

細心的你可能會發現：既然這是一個建立 Mutex 的 API，為什麼還要讓呼叫者傳入 `ucQueueType`？

這是因為在 FreeRTOS 中，Mutex 還有分為：

1. **普通互斥鎖（Standard Mutex）**：`queueQUEUE_TYPE_MUTEX`
    
2. **遞迴互斥鎖（Recursive Mutex）**：`queueQUEUE_TYPE_RECURSIVE_MUTEX`
    

透過傳入不同的 `ucQueueType`，這套代碼可以用完全相同的方式建立這兩者，未來在使用 `xSemaphoreTakeRecursive()` 時就能透過 `ucQueueType` 進行嚴格的型態安全檢查。

### 8. *xQueueCreateMutexStatic*

跟 [[#7. xQueueCreateMutex]] 類似只不過是靜態配置

### 9. *xQueueGetMutexHolder*

`xQueueGetMutexHolder()` 是 FreeRTOS 中用來「查詢當前是哪一個 Task 正持有該 Mutex（互斥鎖）」的底層 API。

通常應用層程式碼不會直接呼叫 `xQueueGetMutexHolder()`，而是透過頭文件定義的巨集 `xSemaphoreGetMutexHolder()` 來使用它。

這個函數看似非常簡單（只是讀取結構體裡面的一個指標），但其內部使用了**臨界區（Critical Section）護航**，且 FreeRTOS 團隊特別在註釋中標註了一個在多工作業系統中非常關鍵的**競態條件（Race Condition）陷阱**！

#### 9.1 條件編譯、斷言檢查與臨界區型態驗證

區塊進入臨界區（Critical Section），防止在讀取指標時被中斷或 Task 切換打斷，並嚴格檢查傳入的 Handle 是否真的是 Mutex。

```C
#if ( ( configUSE_MUTEXES == 1 ) && ( INCLUDE_xSemaphoreGetMutexHolder == 1 ) )

    TaskHandle_t xQueueGetMutexHolder( QueueHandle_t xSemaphore )
    {
        TaskHandle_t pxReturn;
        Queue_t * const pxSemaphore = ( Queue_t * ) xSemaphore;

        traceENTER_xQueueGetMutexHolder( xSemaphore );

        /* 1. 安全斷言：傳入的 Semaphore Handle 絕對不可為 NULL */
        configASSERT( xSemaphore );

        /* 2. 進入臨界區 (Disable Interrupts / Lock Scheduler)：
         * 確保在讀取 xMutexHolder 欄位時，不會被更高優先權的 Task 搶佔，
         * 避免讀到寫到一半的無效記憶體位址。 */
        taskENTER_CRITICAL();
        {
            /* 3. 安全型態檢查：
             * 檢查 uxQueueType 是否真的為 Mutex (queueQUEUE_IS_MUTEX)。
             * 防止誤將一般 Queue 或 Counting Semaphore 傳入此 API 導致存取違規。 */
            if( pxSemaphore->uxQueueType == queueQUEUE_IS_MUTEX )
            {
                /* 提取當前持有鎖的 Task TCB 指標 */
                pxReturn = pxSemaphore->u.xSemaphore.xMutexHolder;
            }
            else
            {
                /* 若傳入的不是 Mutex，安全回傳 NULL */
                pxReturn = NULL;
            }
        }
```

#### 9.2 離開臨界區、Trace 記錄與結果回傳

離開臨界區恢復中斷，並將查詢到的 `TaskHandle_t`（如果目前無人持有或非 Mutex 則為 `NULL`）傳回給呼叫者。

```C
/* 4. 離開臨界區，恢復中斷與調度器 */
        taskEXIT_CRITICAL();

        traceRETURN_xQueueGetMutexHolder( pxReturn );

        /* 5. 回傳當前持有該 Mutex 的 Task Handle (若無人持有則回傳 NULL) */
        return pxReturn;
    }

#endif /* if ( ( configUSE_MUTEXES == 1 ) && ( INCLUDE_xSemaphoreGetMutexHolder == 1 ) ) */
```

#### 9.3 原始碼註釋大揭密：競態條件（Race Condition）陷阱！

FreeRTOS 原始碼註釋中寫著一段非常經典的警告：

> _"This is a good way of determining if the calling task is the mutex holder, but not a good way of determining the identity of the mutex holder..."_

這句話是什麼意思？為什麼拿來檢查「我是不是持有者」很安全，但拿來檢查「別人是不是持有者」卻很危險？

情況 A：檢查「我自己」是不是持有者（100% 安全）

```C
// 判斷當前執行的 Task 自己有沒有持有這個鎖
if( xSemaphoreGetMutexHolder( xMyMutex ) == xTaskGetCurrentTaskHandle() )
{
    // 只有當我自己持有鎖時，我才去釋放它
    xSemaphoreGive( xMyMutex );
}
```

**理由**：因為**當前正在執行這行程式碼的人就是「你」**。如果你持有鎖，在你主動釋放鎖之前，沒有其他人能把你的鎖偷走！所以查詢結果 $100\%$ 可信。

情況 B：檢查「別人」是不是持有者（潛在危險！）

```C
// 企圖查詢 Task B 是不是持有人
if( xSemaphoreGetSemaphoreHolder( xMyMutex ) == xTaskBHandle )
{
    // ❌ 這裡存在 Time-of-Check to Time-of-Use (TOCTOU) 漏洞！
    do_something();
}
```

**危險原因**：

1. 在 `taskEXIT_CRITICAL()` 離開臨界區的那一微秒（microsecond）。
    
2. CPU 觸發了 Task 切換，Task B 剛好執行完了 `xSemaphoreGive()` 把鎖釋放了。
    
3. 當 CPU 重新切回到你的 `if` 敘述時，`xSemaphoreGetMutexHolder` 雖然回傳了 Task B，但**此時 Task B 其實已經不持有鎖了**！

#### 9.4 為什麼需要 `taskENTER_CRITICAL()`？

有人會問：`xMutexHolder` 不就是一個 32-bit 的指標變數嗎？在 32-bit MCU (如 ARM Cortex-M) 上讀取 32-bit 指標不是原子操作（Atomic Operation）嗎？

**兩個關鍵原因：**

1. **防止編譯器與架構層面的重排與不對稱讀取**：在 8-bit / 16-bit 嵌入式系統上（如 AVR, PIC24），讀取 32-bit 指標需要多條指令，如果不進臨界區，讀到一半被中斷打斷，會拼湊出撕裂的無效地址（Torn Read）。
    
2. **語意完整性**：保障 `uxQueueType` 的型態檢查與 `xMutexHolder` 的讀取發生在同一時間點，防止在判定型態為 Mutex 後，該 Queue 在背景被動態刪除或變更。

### 10. *xQueueGetMutexHolderFromISR*

`xQueueGetMutexHolderFromISR()` 是 FreeRTOS 中專門提供給中斷服務程式（ISR, Interrupt Service Routine）用來查詢「當前是哪一個 Task 正持有該 Mutex（互斥鎖）」的底層 API。

通常在應用層，我們會透過 `xSemaphoreGetMutexHolderFromISR()` 巨集來呼叫它。

這個 API 與前面介紹的非 ISR 版本（`xQueueGetMutexHolder`）最大的亮點差異在於：**它完全沒有使用 `taskENTER_CRITICAL()` 進入臨界區！** 這是 FreeRTOS 對於系統底層併發與中斷架構極致理解下的精心設計。

#### 10.1 斷言檢查與無臨界區（No Critical Section）的安全讀取

區塊重點在於說明「為何在中斷環境下查詢 Mutex 持有者不需要關中斷」，並進行 Queue 型態的驗證。

```C
#if ( ( configUSE_MUTEXES == 1 ) && ( INCLUDE_xSemaphoreGetMutexHolder == 1 ) )

    TaskHandle_t xQueueGetMutexHolderFromISR( QueueHandle_t xSemaphore )
    {
        TaskHandle_t pxReturn;

        traceENTER_xQueueGetMutexHolderFromISR( xSemaphore );

        /* 1. 安全斷言：傳入的 Semaphore Handle 絕對不可為 NULL */
        configASSERT( xSemaphore );

        /* 2. 【核心架構設計】：為何這裡完全不需進入臨界區 (Critical Section)？
         * 在 FreeRTOS 原則中：Mutex『絕不可能』在中斷服務程式 (ISR) 中被 Take 或 Give。
         * 因為 Mutex 綁定了 Task 的優先權繼承 (Priority Inheritance)，而 ISR 並沒有 Task 優先權概念。
         * 
         * 既然任何 ISR 都無法變更 Mutex 狀態，且當前 ISR 執行時已經搶佔了所有 Task，
         * 意味著在該 ISR 執行期間，xMutexHolder 的值是『絕對凍結、不可能改變』的。
         * 因此這裡可以直接安全讀取，完全不需呼叫 taskENTER_CRITICAL_FROM_ISR()！ */
        if( ( ( Queue_t * ) xSemaphore )->uxQueueType == queueQUEUE_IS_MUTEX )
        {
            /* 3. 提取當前持有鎖的 Task TCB 指標 */
            pxReturn = ( ( Queue_t * ) xSemaphore )->u.xSemaphore.xMutexHolder;
        }
        else
        {
            /* 4. 若傳入的不是 Mutex (例如傳成了普通 Queue 或 Counting Semaphore)，安全回傳 NULL */
            pxReturn = NULL;
        }
```

#### 10.2 Trace 記錄與結果回傳

無縫完成 Trace 事件發送並傳回查詢到的 `TaskHandle_t`。

```C
traceRETURN_xQueueGetMutexHolderFromISR( pxReturn );

        /* 5. 回傳當前持有該 Mutex 的 Task Handle (若無人持有或非 Mutex 則回傳 NULL) */
        return pxReturn;
    }

#endif /* if ( ( configUSE_MUTEXES == 1 ) && ( INCLUDE_xSemaphoreGetMutexHolder == 1 ) ) */
```

#### 10.3 深度剖析：為什麼 ISR 版本「完全不需要保護」？

我們比較一下兩個版本的程式碼：

- **Task 版本 (`xQueueGetMutexHolder`)**：裡面包了 `taskENTER_CRITICAL()`。
    
- **ISR 版本 (`xQueueGetMutexHolderFromISR`)**：**完全沒有臨界區！**

**背後的硬核原因如下：**

1. **Mutex 禁用于 ISR**：FreeRTOS **嚴禁**在中斷（ISR）中使用 `xSemaphoreTake()` 或 `xSemaphoreGive()`。因為 Mutex 的核心機制是「優先權繼承」，ISR 不屬於任何 Task，沒有優先權可言，因此根本沒有 `xSemaphoreTakeFromISR()` 這個 API。
    
2. **中斷執行時的凍結狀態**：當 CPU 進入 ISR 時，**所有 Task 的執行通通被暫停（Block/Preempted）**。
    
    - 沒有任何 Task 能在這段時間內釋放或搶奪 Mutex。
        
    - 沒有任何 ISR 能在這段時間內變更 Mutex。
        
3. **結論**：在 ISR 執行的當下，`xMutexHolder` 這個記憶體位址的內容是 **100% 靜態且凍結的**，直接讀取絕對不會發生競態條件（Race Condition），因此省去了關閉中斷的開銷！

實戰場景：什麼時候會在 ISR 裡查詢 Mutex 持有者？

既然 ISR 裡面不能操作 Mutex，為什麼還要提供這個 API？

最經典的用途就是在 **看板狗中斷（Watchdog ISR）** 或 **系統診斷/除錯 ISR（Timer ISR / Fault Handler）** 中：

- **情境**：看門狗中斷發起，發現系統某個關鍵任務卡死（Deadlock / Starvation）。
    
- **除錯動作**：看門狗 ISR 呼叫 `xSemaphoreGetMutexHolderFromISR( xSharedResourceMutex )`。
    
- **結果**：ISR 發現卡死當下是 `Task_A` 正扣著關鍵 Mutex，將 `Task_A` 的資訊紀錄至 Log 或 Crash Dump 中，達到精準除錯的目的！

### 11. *xQueueGiveMutexRecursive*

`xQueueGiveMutexRecursive()` 是 FreeRTOS 中專門用來「釋放遞迴互斥鎖（Recursive Mutex）」的 API。

在前面的基礎中我們知道，普通的 Mutex 只要 Take 一次就必須 Give 一次。但 **遞迴鎖（Recursive Mutex）** 允許同一個 Task 重複 Take 很多次（例如在遞迴函數或層層包裹的函式庫中），因此釋放時也**必須呼叫相同次數的 `xQueueGiveMutexRecursive()`，直到遞迴計數器歸零時，鎖才會真正被還給系統**。

這段程式碼最令人驚嘆的，是 FreeRTOS 展示了「無鎖併發程式設計（Lock-Free Design）」的極致——它在檢查鎖持有者與修改遞迴計數時，**完全不需要進入臨界區（Critical Section）**！

#### 11.1 身份權限檢查與「零臨界區保護」設計

區塊重點在於無鎖（Lock-free）驗證呼叫者是否為持有者，並遞減遞迴呼叫次數（`uxRecursiveCallCount`）。

```C
#if ( configUSE_RECURSIVE_MUTEXES == 1 )

    BaseType_t xQueueGiveMutexRecursive( QueueHandle_t xMutex )
    {
        BaseType_t xReturn;
        Queue_t * const pxMutex = ( Queue_t * ) xMutex;

        traceENTER_xQueueGiveMutexRecursive( xMutex );

        /* 1. 安全斷言：確定傳入的 Mutex Handle 非空 */
        configASSERT( pxMutex );

        /* 2. 【核心神級設計】：無臨界區保護的身份驗證！
         *
         * 為什麼這裡讀取 xMutexHolder 欄位『不需要』 taskENTER_CRITICAL()？
         *
         * 情況 A：如果『當前 Task』就是持有人：
         *   除了你自己之外，沒有任何其他 Task 或 ISR 有權力/有可能去修改 xMutexHolder。
         *   所以這個讀取動作是 100% 執行緒安全的 (Thread-Safe)。
         *
         * 情況 B：如果『當前 Task』不是持有人：
         *   xMutexHolder 目前屬於其他人或 NULL。即便其他 Task 此時正在修改它，
         *   xMutexHolder 也『絕不可能巧合地變成當前 Task 的 Handle』(因為只有你自己去 Take 鎖時才能寫入自己的 Handle)。
         *   判斷結果一定為 false，因此完全不影響邏輯正確性！ */
        if( pxMutex->u.xSemaphore.xMutexHolder == xTaskGetCurrentTaskHandle() )
        {
            traceGIVE_MUTEX_RECURSIVE( pxMutex );

            /* 3. 遞減遞迴計數器：
             * 因為只有持有鎖的 Task 能走到這裡，且全系統只有這一個 Task 能修改此計數器，
             * 因此 (uxRecursiveCallCount)-- 同樣『不需要』臨界區保護！
             * 另外，既然持有鎖，計數器必大於 0，故不需擔心下溢 (Underflow)。 */
            ( pxMutex->u.xSemaphore.uxRecursiveCallCount )--;
```

#### 11.2 遞迴層數解開、底層 Token 歸還與錯誤處置

判斷遞迴計數是否已解到最外層（歸零）。若歸零則呼叫 `xQueueGenericSend` 將 Token 塞回 Queue，喚醒其他排隊等待的 Task；若非持有人呼叫則回傳 `pdFAIL`。

```C
/* 4. 檢查遞迴計數是否已徹底解開至 0？ */
            if( pxMutex->u.xSemaphore.uxRecursiveCallCount == ( UBaseType_t ) 0 )
            {
                /* 遞迴已徹底完結！正式呼叫 xQueueGenericSend() 歸還底層 Token。
                 * 這一步會真正將 Mutex 置回 Available 狀態，
                 * 並自動喚醒 (Unblock) 其它正在排隊等待這個 Mutex 的高優先權 Task！ */
                ( void ) xQueueGenericSend( pxMutex, NULL, queueMUTEX_GIVE_BLOCK_TIME, queueSEND_TO_BACK );
            }
            else
            {
                /* 遞迴層數尚未歸零 (例如 Take 了 3 次，目前才 Give 第 1 或第 2 次)，
                 * 鎖依然保留給當前 Task，不呼叫 xQueueGenericSend()。 */
                mtCOVERAGE_TEST_MARKER();
            }

            xReturn = pdPASS;
        }
        else
        {
            /* 5. 權限錯誤處置：
             * 呼叫 Give 的 Task 根本不是這個 Mutex 的持有者！
             * 遞迴鎖嚴禁「幫別人解鎖」，故直接宣告失敗。 */
            xReturn = pdFAIL;

            traceGIVE_MUTEX_RECURSIVE_FAILED( pxMutex );
        }

        traceRETURN_xQueueGiveMutexRecursive( xReturn );

        return xReturn;
    }

#endif /* configUSE_RECURSIVE_MUTEXES */
```

#### 11.3 普通 Mutex vs 遞迴 Mutex 解鎖 API 對比

| **特性**         | **普通 Mutex (xSemaphoreGive)**  | **遞迴 Mutex (xSemaphoreGiveRecursive)** |
| -------------- | ------------------------------ | -------------------------------------- |
| **底層 API**     | `xQueueGenericSend()`          | `xQueueGiveMutexRecursive()`           |
| **多次 Give 影響** | 第一次 Give 即解鎖，重複 Give 會失敗       | 必須 Give 與 Take **相同次數**後才會真正解鎖         |
| **非持有者 Give**  | ❌ **嚴禁** (但普通 Semaphore 允許)    | ❌ **嚴禁** (回傳 `pdFAIL`)                 |
| **臨界區開銷**      | 需要 (內部 `xQueueGenericSend` 有鎖) | **完全不需要** (僅歸零瞬間呼叫 Send)               |

### 12. *xQueueTakeMutexRecursive*

`xQueueTakeMutexRecursive` 是 FreeRTOS 中實現**遞迴互斥鎖（Recursive Mutex）獲取**的核心函數。

遞迴互斥鎖的最大特點是：**同一個任務（Task）可以重複多次獲取同一個鎖而不會造成自我死鎖（Self-Deadlock）**。此函數內部透過記錄「持有者 Tasks」與「遞迴計數器 Call Count」來實現這個機制。

#### 12.1 條件編譯與函數簽名

```C
#if ( configUSE_RECURSIVE_MUTEXES == 1 )

    BaseType_t xQueueTakeMutexRecursive( QueueHandle_t xMutex,
                                         TickType_t xTicksToWait )
```

**說明：**

- **`configUSE_RECURSIVE_MUTEXES`**：功能開關。只有在 `FreeRTOSConfig.h` 設定為 `1` 時才會編譯此 API。
    
- **`xMutex`**：欲獲取的互斥鎖句柄。
    
- **`xTicksToWait`**：若鎖已被其他任務佔用，此任務願意進入阻塞狀態等待的最大 Tick 數（可設為 `portMAX_DELAY` 無限期等待）。

#### 12.2 指針轉換與安全檢查

```C
BaseType_t xReturn;
        Queue_t * const pxMutex = ( Queue_t * ) xMutex;

        traceENTER_xQueueTakeMutexRecursive( xMutex, xTicksToWait );

        configASSERT( pxMutex );

        traceTAKE_MUTEX_RECURSIVE( pxMutex );
```

**說明：**

- FreeRTOS 中的 Mutex 底層結構本質上就是 `Queue_t`，因此將句柄轉型為 `Queue_t*`。
    
- **`configASSERT( pxMutex )`**：檢查傳入的句柄是否為合法指標（非 NULL）。
    
- **`traceENTER_...` / `traceTAKE_...`**：FreeRTOS 的追蹤巨集（Trace Macros），供 Tracealyzer 等系統除錯工具觀察行為。

#### 12.3 情境 A：當前任務「已經持有」該鎖（遞迴重入）

```C
if( pxMutex->u.xSemaphore.xMutexHolder == xTaskGetCurrentTaskHandle() )
        {
            ( pxMutex->u.xSemaphore.uxRecursiveCallCount )++;

            /* Check if an overflow occurred. */
            configASSERT( pxMutex->u.xSemaphore.uxRecursiveCallCount );

            xReturn = pdPASS;
        }
```

**說明：**

- **核心判斷**：比較當前的 Mutex 擁有者 (`xMutexHolder`) 與呼叫此 API 的任務 (`xTaskGetCurrentTaskHandle()`)。
    
- **重入處理**：若該任務已經持有此鎖，**完全不需要再去競爭或阻塞 Queue**。
    
- 直接將遞迴計數器 **`uxRecursiveCallCount` 加 1**。
    
- **`configASSERT(...)`**：防止計數器加到溢位（如從最高值溢位變成 0）。
    
- 直接設定 `xReturn = pdPASS` 表示順利重入取得鎖。

#### 12.4 情境 B：當前任務「尚未持有」該鎖（第一次獲取）

```C
else
        {
            xReturn = xQueueSemaphoreTake( pxMutex, xTicksToWait );

            /* pdPASS will only be returned if the mutex was successfully
             * obtained.  The calling task may have entered the Blocked state
             * before reaching here. */
            if( xReturn != pdFAIL )
            {
                ( pxMutex->u.xSemaphore.uxRecursiveCallCount )++;

                /* Check if an overflow occurred. */
                configASSERT( pxMutex->u.xSemaphore.uxRecursiveCallCount );
            }
            else
            {
                traceTAKE_MUTEX_RECURSIVE_FAILED( pxMutex );
            }
        }
```

**說明：**

- 若當前任務不持有該鎖，代表這是第一次獲取，需要進入競爭流程。
    
- 呼叫 **`xQueueSemaphoreTake(pxMutex, xTicksToWait)`** 嘗試搶鎖：
    
    - 若鎖正被其他任務佔用，此任務會依 `xTicksToWait` 進入阻塞（Blocked）狀態。
        
- **成功搶到鎖 (`xReturn != pdFAIL`)**：
    
    - 將遞迴計數器 `uxRecursiveCallCount` 加 1（原先為 0，此時變為 1）。
        
    - 底層的 `xQueueSemaphoreTake` 會自動把 `xMutexHolder` 設為當前任務。
        
- **搶鎖失敗 (`xReturn == pdFAIL`)**：
    
    - 說明在 `xTicksToWait` 時間內鎖都未被釋放，觸發失敗追蹤巨集。

#### 12.5 函數結束與回傳

```C
traceRETURN_xQueueTakeMutexRecursive( xReturn );

        return xReturn;
    }

#endif /* configUSE_RECURSIVE_MUTEXES */
```

**說明：**

- 執行返回追蹤巨集。
    
- 回傳 `pdPASS`（成功取得鎖）或 `pdFAIL`（逾時失敗）。

### 13. *xQueueCreateCountingSemaphoreStatic*

`xQueueCreateCountingSemaphoreStatic` 是 FreeRTOS 中用於靜態配置記憶體並建立「計數信號量」（Counting Semaphore）的核心 API。

核心概念：

1. **計數信號量 (Counting Semaphore)**：常用於**資源管理**（例如有多個相同的硬體資源供任務競爭）或**事件計數**（如中斷發生了幾次，任務就處理幾次）。
    
2. **靜態配置 (Static Allocation)**：記憶體不透過 FreeRTOS 堆疊（Heap，即 `pvPortMalloc`）動態分配，而是由開發者在編譯時期指定好記憶體空間（例如全域變數或靜態結構體）。這樣可確保記憶體佈局確定，避免記憶體碎片化，非常適合用於安全關鍵（Safety-Critical）系統。

#### 13.1 條件編譯巨集與函數簽名

```C
#if ( ( configUSE_COUNTING_SEMAPHORES == 1 ) && ( configSUPPORT_STATIC_ALLOCATION == 1 ) )

    QueueHandle_t xQueueCreateCountingSemaphoreStatic( const UBaseType_t uxMaxCount,
                                                       const UBaseType_t uxInitialCount,
                                                       StaticQueue_t * pxStaticQueue )
```

**說明：**

- **功能開關**：只有在 `FreeRTOSConfig.h` 中同時開啟 `configUSE_COUNTING_SEMAPHORES == 1`（啟動計數信號量）與 `configSUPPORT_STATIC_ALLOCATION == 1`（啟動靜態記憶體配置）時，此 API 才有效。
    
- **參數分析**：
    
    - `uxMaxCount`：信號量可達到的最大計數值（例如資源總數）。
        
    - `uxInitialCount`：信號量建立時的初始計數值。
        
    - `pxStaticQueue`：由呼叫端自行提供的 `StaticQueue_t` 結構體指標，用於儲存這個信號量的狀態資訊。

#### 13.2 參數合法性檢查

```C
if( ( uxMaxCount != 0U ) &&
            ( uxInitialCount <= uxMaxCount ) )
```

**說明：**

- 防呆檢查：
    
    1. `uxMaxCount != 0U`：最大容量不能為 0，否則信號量毫無意義。
        
    2. `uxInitialCount <= uxMaxCount`：初始計數值不能大於最大許可值。

#### 13.3 底層靜態佇列建立

```C
xHandle = xQueueGenericCreateStatic( uxMaxCount, queueSEMAPHORE_QUEUE_ITEM_LENGTH, NULL, pxStaticQueue, queueQUEUE_TYPE_COUNTING_SEMAPHORE );
```

> **說明：**
> 
> - 在 FreeRTOS 中，所有的互斥鎖（Mutex）與信號量（Semaphore）**底層都是基於 Queue（佇列）實現的**。
>     
> - 此處呼叫泛用靜態佇列建立 API：
>     
>     - **`uxMaxCount`**：佇列深度等於信號量的最大可計數值。
>         
>     - **`queueSEMAPHORE_QUEUE_ITEM_LENGTH`**：項目大小為 `0`。因為信號量只需要計數，不需要在佇列內儲存實際的資料內容。
>         
>     - **`pucQueueStorageBuffer = NULL`**：因為資料長度為 0，所以不需要緩衝區空間。
>         
>     - **`pxStaticQueue`**：傳入儲存佇列控制結構（`Queue_t`）的靜態記憶體位址。
>         
>     - **`queueQUEUE_TYPE_COUNTING_SEMAPHORE`**：指定這是一個計數信號量。

#### 13.4 初始化計數值與狀態追蹤

```C
if( xHandle != NULL )
            {
                ( ( Queue_t * ) xHandle )->uxMessagesWaiting = uxInitialCount;

                traceCREATE_COUNTING_SEMAPHORE();
            }
            else
            {
                traceCREATE_COUNTING_SEMAPHORE_FAILED();
            }
```

**說明：**

- 如果靜態佇列順利建立完成 (`xHandle != NULL`)：
    
    - FreeRTOS 會將 `Queue_t` 中的 **`uxMessagesWaiting`** 成員作為「當前可用信號量計數」。
        
    - 將 `uxMessagesWaiting` 強制設定為傳入的初始值 `uxInitialCount`。
        
    - 呼叫 `traceCREATE_COUNTING_SEMAPHORE()` 記錄建立成功。
        
- 若建立失敗則觸發失敗紀錄巨集。

### 14. *xQueueCreateCountingSemaphore*

