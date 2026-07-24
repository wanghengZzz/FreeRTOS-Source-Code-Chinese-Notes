
---
tags:
  - FreeRTOS
  - RTOS
  - embedded
  - concurrency
  - memory-model
created: 2026-07-02
---

# FreeRTOS：portSOFTWARE_BARRIER 與 portMEMORY_BARRIER

## 核心概念：為什麼需要 Barrier？

現代編譯器與 CPU 為了效能最佳化，會對指令進行**重新排序（reordering）**。這在單執行緒程式中完全安全，但在多執行緒、ISR（中斷服務程式）或多核環境中，可能導致難以察覺的 race condition。

Barrier 的目的就是**阻止這種重排**，確保記憶體操作的順序性。

---

## 兩種 Barrier 的本質差異

| 特性 | `portSOFTWARE_BARRIER` | `portMEMORY_BARRIER` |
|------|----------------------|---------------------|
| **防止對象** | 僅編譯器重排 | 編譯器 + CPU 硬體重排 |
| **作用層級** | 軟體層（編譯期） | 軟體層 + 硬體層（執行期） |
| **效能開銷** | 極低（只影響編譯產出） | 較高（會發出實際的 fence 指令） |
| **適用架構** | 強記憶體排序架構（如 x86、Cortex-M） | 弱記憶體排序架構（如 ARM A 系列、RISC-V） |

---

## portSOFTWARE_BARRIER

### 功能

只阻止**編譯器**將 barrier 前後的程式碼重新排序或合併最佳化。CPU 在執行時仍可能以不同順序執行指令。

### 典型實作

```c
// GCC / Clang
#define portSOFTWARE_BARRIER() __asm volatile( "" ::: "memory" )
```

`"memory"` clobber 告訴編譯器：假設所有記憶體內容都可能被這行程式碼讀寫，因此不得將前後的記憶體操作跨越此點重排。

### 解決的問題：編譯器最佳化重排

```c
// 原始程式碼意圖
flag = 1;            // 步驟 1：設旗標
data = compute();    // 步驟 2：準備資料

// 編譯器可能最佳化為（因為它認為順序不重要）
data = compute();    // ← 被移到前面
flag = 1;

// 加入 portSOFTWARE_BARRIER() 後，順序被鎖定
flag = 1;
portSOFTWARE_BARRIER();
data = compute();    // 保證在 flag = 1 之後編譯
```

### 適用場景

- **單核 Cortex-M 系列**：硬體保證 load/store 順序（強記憶體模型），只需防止編譯器重排即可
- 防止編譯器把迴圈條件的讀取快取到暫存器而跳過重新讀取記憶體
- 與中斷（ISR）之間的旗標同步（單核）

---

## portMEMORY_BARRIER

### 功能

同時阻止**編譯器重排**與 **CPU 硬體的亂序執行（Out-of-Order Execution）或投機執行（Speculative Execution）**，確保 barrier 前的所有記憶體操作對其他 CPU core 或匯流排上的 master 可見後，才執行 barrier 後的操作。

### 典型實作

```c
// ARM Cortex-A / ARMv7+
#define portMEMORY_BARRIER() __asm volatile( "dmb" ::: "memory" )
//                                            ^^^
//                              Data Memory Barrier 硬體指令

// ARMv8 / AArch64
#define portMEMORY_BARRIER() __asm volatile( "dmb sy" ::: "memory" )

// RISC-V
#define portMEMORY_BARRIER() __asm volatile( "fence" ::: "memory" )
```

### 解決的問題：CPU 硬體層級重排

現代高效能 CPU（特別是多核 ARM Cortex-A）具有：

- **Store Buffer**：寫入可能尚未刷新到快取/記憶體，其他 core 看不到
- **投機執行**：CPU 預測分支，提前執行後續 load
- **Write Combining**：多次寫入被合併延遲發出

```
Core 0                      Core 1
------                      ------
x = 1;                      while (ready == 0);  // 等待
portMEMORY_BARRIER();       // ← 沒有這個，Core 1 可能永遠
ready = 1;                  //   看不到 x = 1 的寫入
                            use(x);
```

沒有 `portMEMORY_BARRIER()`，Core 0 對 `x` 的寫入可能還卡在 store buffer 中，Core 1 讀到的 `x` 可能是舊值。

### 適用場景

- **多核（SMP）系統**：Core 之間的資料同步
- **弱記憶體模型架構**（ARM Cortex-A、RISC-V、Power）
- DMA 傳輸前後：確保資料真正寫入記憶體再啟動 DMA，或 DMA 完成後再讀取結果
- 與硬體周邊（MMIO）互動時確保寫入順序
- Spinlock / 無鎖資料結構的實作

---

## 兩者的關係

```
portSOFTWARE_BARRIER  ⊂  portMEMORY_BARRIER
```

`portMEMORY_BARRIER` 的效果涵蓋 `portSOFTWARE_BARRIER`。使用 `portMEMORY_BARRIER` 時，編譯器重排也一併被阻止。

> [!tip] 選用原則
> - 確定是**單核**且架構有強記憶體模型 → 用 `portSOFTWARE_BARRIER`（開銷更小）
> - **多核**，或架構是弱記憶體模型，或需要確保硬體可見性 → 用 `portMEMORY_BARRIER`

---

## 在 FreeRTOS 原始碼中的實際應用

### portSOFTWARE_BARRIER 的使用

```c
// tasks.c - 防止編譯器將 xTickCount 的讀取提前
BaseType_t xTaskCheckForTimeOut( TimeOut_t * const pxTimeOut, ... )
{
    // ...
    portSOFTWARE_BARRIER();
    // 確保讀取的是最新的 tick count，而非編譯器快取的值
    const TickType_t xConstTickCount = xTickCount;
    // ...
}
```

### portMEMORY_BARRIER 的使用

```c
// 用於 SMP port，確保 critical section 的記憶體操作對所有 core 可見
void vTaskExitCritical( void )
{
    portMEMORY_BARRIER();
    // 在釋放鎖之前，所有記憶體操作必須對其他 core 可見
    portRELEASE_ISR_LOCK();
}
```

---

## 快速記憶

```
SOFTWARE_BARRIER → 防「編譯器」偷偷重排   → 單核 / 強序架構
MEMORY_BARRIER   → 防「編譯器 + CPU」重排 → 多核 / 弱序架構
```

---

## 相關概念

- [[Volatile 關鍵字]] — 防止編譯器優化對變數的存取，但不防止重排
- [[FreeRTOS Critical Section]] — `taskENTER_CRITICAL()` / `taskEXIT_CRITICAL()`
- [[記憶體模型 Memory Model]] — 強序 vs 弱序架構
- [[ARM DMB DSB ISB 指令]] — ARM 的三種 barrier 指令差異
- [[FreeRTOS SMP 支援]]
