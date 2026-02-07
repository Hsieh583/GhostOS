# ⏰ 程序調度器 (Process Scheduler)

## 概述

程序調度器決定哪個程序應該獲得 CPU 時間。本文檔分析常見的調度算法和實現邏輯。

---

## 1. 調度基本概念

### 調度目標

| 系統類型 | 主要目標 |
|----------|----------|
| 批次處理系統 | 吞吐量、CPU 利用率 |
| 互動式系統 | 響應時間、公平性 |
| 實時系統 | 滿足截止時間 |

### 關鍵指標

- **週轉時間 (Turnaround Time)**: 從提交到完成的時間
- **等待時間 (Waiting Time)**: 在就緒佇列等待的時間
- **響應時間 (Response Time)**: 從提交到第一次響應的時間
- **CPU 利用率**: CPU 忙碌的百分比
- **吞吐量**: 單位時間完成的程序數

---

## 2. 調度算法

### 2.1 先來先服務 (FCFS)

```
就緒佇列: [P1, P2, P3] → CPU → [完成]

時間線:
┌────────────────┬──────────┬────┐
│      P1        │    P2    │ P3 │
└────────────────┴──────────┴────┘
0                24         27   30

假設到達時間都為 0
P1 burst = 24, P2 burst = 3, P3 burst = 3

等待時間: P1=0, P2=24, P3=27
平均等待時間 = (0+24+27)/3 = 17
```

**缺點**: 護航效應 (Convoy Effect) - 短程序等待長程序

### 2.2 最短作業優先 (SJF)

```
非搶占式 SJF:
┌────┬────┬────────────────────────┐
│ P2 │ P3 │           P1           │
└────┴────┴────────────────────────┘
0    3    6                        30

等待時間: P1=6, P2=0, P3=3
平均等待時間 = (6+0+3)/3 = 3
```

**優點**: 最小平均等待時間
**缺點**: 需要預知 CPU burst 時間

### 2.3 優先級調度 (Priority Scheduling)

```c
struct process {
    int pid;
    int priority;      // 0 = 最高優先級
    int burst_time;
    int waiting_time;
};

// 優先級佇列
priority_queue ready_queue;

void schedule() {
    struct process *next = priority_queue_top(&ready_queue);
    if (next) {
        priority_queue_pop(&ready_queue);
        run_process(next);
    }
}
```

**問題**: 飢餓 (Starvation)
**解決**: 老化 (Aging) - 隨時間增加優先級

### 2.4 輪轉調度 (Round Robin)

```
時間片 (Time Quantum) = 4

┌────┬────┬────┬────┬────┬────┬────┬──┐
│ P1 │ P2 │ P3 │ P1 │ P2 │ P1 │ P1 │P1│
└────┴────┴────┴────┴────┴────┴────┴──┘
0    4    7   10   14   17   21   25 26

P1 burst=20, P2=6, P3=3
```

```c
#define TIME_QUANTUM 10  // 毫秒

void round_robin_schedule() {
    while (!queue_empty(&ready_queue)) {
        struct process *p = queue_dequeue(&ready_queue);
        
        // 執行一個時間片
        int runtime = min(p->remaining_time, TIME_QUANTUM);
        run_for(p, runtime);
        p->remaining_time -= runtime;
        
        if (p->remaining_time > 0) {
            // 未完成，放回佇列尾端
            queue_enqueue(&ready_queue, p);
        } else {
            // 完成
            process_complete(p);
        }
    }
}
```

### 2.5 多層級反饋佇列 (MLFQ)

```
優先級高 ┌─────────────────────────────────┐
  Queue 0│  RR, quantum = 8ms             │
         ├─────────────────────────────────┤
  Queue 1│  RR, quantum = 16ms            │
         ├─────────────────────────────────┤
  Queue 2│  RR, quantum = 32ms            │
         ├─────────────────────────────────┤
優先級低 │  FCFS                          │
         └─────────────────────────────────┘

規則:
1. 新程序進入最高優先級佇列
2. 用完時間片則降級
3. 主動放棄 CPU 則保持級別
4. 定期將所有程序提升到最高級別 (防止飢餓)
```

---

## 3. Linux CFS 調度器概述

### 完全公平調度 (Completely Fair Scheduler)

```
設計理念:
- 每個程序應該獲得 1/n 的 CPU 時間 (n = 可執行程序數)
- 使用虛擬執行時間 (vruntime) 追蹤公平性
- 紅黑樹組織可執行程序

vruntime 計算:
vruntime += delta_exec × (NICE_0_WEIGHT / weight)

其中 weight 由 nice 值決定
```

### 紅黑樹結構

```
              ┌───────────────┐
              │   P3 (vr=100) │  ← 根節點
              └───────┬───────┘
             ┌────────┴────────┐
      ┌──────┴──────┐   ┌──────┴──────┐
      │ P1 (vr=80)  │   │ P5 (vr=120) │
      └──────┬──────┘   └──────┬──────┘
         ┌───┴───┐         ┌───┴───┐
  ┌──────┴──────┐    ┌─────┴─────┐
  │ P2 (vr=50)  │    │P4 (vr=110)│
  └─────────────┘    └───────────┘
         ▲
         │
    最左節點 = 下一個執行的程序
```

---

## 4. 上下文切換

### 切換過程

```
┌─────────────────────────────────────────────────────────────┐
│                    Context Switch                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Process A (Running)              Process B (Ready)         │
│  ┌─────────────────┐              ┌─────────────────┐       │
│  │ Registers       │              │ (saved state)    │      │
│  │ PC, SP, etc.    │              │                 │       │
│  └────────┬────────┘              └─────────────────┘       │
│           │                                ▲                │
│           │ 1. Save state                  │                │
│           ▼                                │                │
│  ┌─────────────────┐              ┌────────┴────────┐       │
│  │ PCB of A        │              │ PCB of B        │       │
│  │ (save registers)│              │ (load registers)│       │
│  └─────────────────┘              └────────┬────────┘       │
│           │                                │                │
│           │ 2. Update PCB                  │                │
│           ▼                                │                │
│  ┌─────────────────┐                       │                │
│  │ Scheduler       │───────────────────────┘                │
│  │ (select next)   │    3. Load state                       │
│  └─────────────────┘                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 切換成本

| 項目 | 大約時間 |
|------|----------|
| 保存暫存器 | 幾十個 cycles |
| 更新 PCB | 幾十個 cycles |
| 載入新暫存器 | 幾十個 cycles |
| TLB 刷新 | 數百到數千 cycles |
| Cache 污染 | 可能更長 |

---

## 5. 實現範例

### 簡單的調度器

```c
// 程序控制塊
typedef struct {
    int pid;
    int state;              // READY, RUNNING, BLOCKED
    int priority;
    uint32_t esp;           // 堆疊指標
    uint32_t ebp;           // 基址指標
    uint32_t eip;           // 指令指標
    pde_t *page_directory;  // 頁目錄
    // ... 更多
} pcb_t;

// 就緒佇列
#define MAX_PROCESSES 64
static pcb_t processes[MAX_PROCESSES];
static int current_process = -1;

// 調度器主迴圈
void scheduler() {
    while (1) {
        // 禁用中斷
        cli();
        
        // 找下一個可執行的程序
        int next = find_next_runnable();
        
        if (next != -1 && next != current_process) {
            // 保存當前程序狀態
            if (current_process != -1) {
                save_context(&processes[current_process]);
            }
            
            // 切換到新程序
            current_process = next;
            load_context(&processes[next]);
            
            // 切換頁表
            switch_page_directory(processes[next].page_directory);
        }
        
        // 啟用中斷
        sti();
        
        // 執行當前程序
        // (通過中斷返回到用戶空間)
    }
}

// 計時器中斷處理 (搶占式調度)
void timer_interrupt_handler() {
    // 觸發調度
    schedule();
    
    // 發送 EOI
    outb(0x20, 0x20);
}
```

### 組語上下文切換

```asm
; void switch_context(pcb_t *old, pcb_t *new)
switch_context:
    ; 保存舊程序的暫存器
    mov eax, [esp+4]    ; old PCB
    mov [eax+0], ebx
    mov [eax+4], ecx
    mov [eax+8], edx
    mov [eax+12], esi
    mov [eax+16], edi
    mov [eax+20], ebp
    
    ; 保存堆疊指標
    mov [eax+24], esp
    
    ; 保存返回位址作為 EIP
    mov ecx, [esp]
    mov [eax+28], ecx
    
    ; 載入新程序的暫存器
    mov eax, [esp+8]    ; new PCB
    mov ebx, [eax+0]
    mov ecx, [eax+4]
    mov edx, [eax+8]
    mov esi, [eax+12]
    mov edi, [eax+16]
    mov ebp, [eax+20]
    mov esp, [eax+24]
    
    ; 跳轉到新程序
    push dword [eax+28]
    ret
```

---

## 6. 同步原語

### 自旋鎖 (Spinlock)

```c
typedef struct {
    volatile int locked;
} spinlock_t;

void spinlock_acquire(spinlock_t *lock) {
    while (__sync_lock_test_and_set(&lock->locked, 1)) {
        // 自旋等待
        while (lock->locked) {
            asm volatile("pause");
        }
    }
}

void spinlock_release(spinlock_t *lock) {
    __sync_lock_release(&lock->locked);
}
```

### 信號量 (Semaphore)

```c
typedef struct {
    int value;
    spinlock_t lock;
    queue_t wait_queue;
} semaphore_t;

void semaphore_wait(semaphore_t *sem) {
    spinlock_acquire(&sem->lock);
    
    while (sem->value <= 0) {
        // 加入等待佇列並睡眠
        queue_enqueue(&sem->wait_queue, current_process);
        current_process->state = BLOCKED;
        
        spinlock_release(&sem->lock);
        schedule();
        spinlock_acquire(&sem->lock);
    }
    
    sem->value--;
    spinlock_release(&sem->lock);
}

void semaphore_signal(semaphore_t *sem) {
    spinlock_acquire(&sem->lock);
    
    sem->value++;
    
    if (!queue_empty(&sem->wait_queue)) {
        // 喚醒一個等待的程序
        pcb_t *p = queue_dequeue(&sem->wait_queue);
        p->state = READY;
    }
    
    spinlock_release(&sem->lock);
}
```

---

## 參考資料

- [OSDev Wiki - Scheduling Algorithms](https://wiki.osdev.org/Scheduling_Algorithms)
- [Linux Kernel - CFS Scheduler](https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt)
- Operating Systems: Three Easy Pieces, Chapters on Scheduling
