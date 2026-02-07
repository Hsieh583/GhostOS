# 🔄 系統調用流程分析 (Syscall Flow Analysis)

## 概述

當應用程式想要執行特權操作（如 Print "Hello"），它必須透過系統調用來請求 Kernel 的幫助。這份文件模擬分析整個流程。

---

## 📊 Syscall 完整流程圖

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          User Space (Ring 3)                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Application: printf("Hello");                                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  C Library (glibc): write(1, "Hello", 5);                        │   │
│   │  • fd = 1 (stdout)                                               │   │
│   │  • buf = "Hello"                                                 │   │
│   │  • count = 5                                                     │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  系統調用介面:                                                    │   │
│   │  • EAX = 4 (sys_write 系統調用號)                                │   │
│   │  • EBX = 1 (fd)                                                  │   │
│   │  • ECX = buffer 位址                                             │   │
│   │  • EDX = 5 (長度)                                                │   │
│   │  • 執行 INT 0x80 或 SYSCALL/SYSENTER                            │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
└──────────────────────────────│──────────────────────────────────────────┘
                               │ ┌────────────────────────┐
                               │ │ 特權級別切換            │
                               │ │ Ring 3 → Ring 0       │
                               │ │ CPU 自動儲存狀態       │
                               │ └────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Kernel Space (Ring 0)                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  中斷處理程序 (IDT entry 0x80)                                   │   │
│   │  • 保存所有暫存器到 Kernel Stack                                 │   │
│   │  • 驗證系統調用號                                                │   │
│   │  • 從 sys_call_table 查找處理函數                               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  sys_write() 函數:                                               │   │
│   │  • 驗證 fd 是否有效                                              │   │
│   │  • 驗證 buffer 位址是否在 User Space                            │   │
│   │  • 呼叫 VFS 層                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  VFS (Virtual File System):                                      │   │
│   │  • 查找 file 結構                                               │   │
│   │  • 呼叫對應的 file_operations->write                            │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  TTY Driver (終端驅動):                                          │   │
│   │  • 處理字元輸出                                                  │   │
│   │  • 寫入到 frame buffer 或 serial port                           │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  返回處理:                                                       │   │
│   │  • EAX = 寫入的位元組數 (成功) 或 負數 (錯誤碼)                  │   │
│   │  • 恢復暫存器                                                    │   │
│   │  • IRET 指令返回 User Space                                     │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                               │ ┌────────────────────────┐
                               │ │ 特權級別切換            │
                               │ │ Ring 0 → Ring 3       │
                               │ │ CPU 恢復狀態          │
                               │ └────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          User Space (Ring 3)                            │
├─────────────────────────────────────────────────────────────────────────┤
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  應用程式繼續執行                                                │   │
│   │  螢幕上顯示 "Hello"                                             │   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. 系統調用號碼表 (x86 Linux)

| 調用號 | 名稱 | 描述 |
|--------|------|------|
| 1 | sys_exit | 結束程序 |
| 2 | sys_fork | 建立子程序 |
| 3 | sys_read | 讀取檔案 |
| 4 | sys_write | 寫入檔案 |
| 5 | sys_open | 開啟檔案 |
| 6 | sys_close | 關閉檔案 |
| 11 | sys_execve | 執行程式 |
| 20 | sys_getpid | 取得程序 ID |
| 45 | sys_brk | 調整 heap 大小 |
| 90 | sys_mmap | 記憶體映射 |

---

## 2. IDT (Interrupt Descriptor Table) 設置

### IDT Entry 結構 (8 bytes)

```
63                                48 47          40 39          32
┌────────────────────────────────────┬─────────────┬──────────────┐
│      Offset 31:16                  │   Flags     │   Reserved   │
│         (16 bits)                  │   (8 bits)  │   (8 bits)   │
└────────────────────────────────────┴─────────────┴──────────────┘

31                              16 15                       0
┌─────────────────────────────────┬──────────────────────────┐
│        Segment Selector         │      Offset 15:0         │
│           (16 bits)             │        (16 bits)         │
└─────────────────────────────────┴──────────────────────────┘
```

### Flags 位元

| 位元 | 名稱 | 說明 |
|------|------|------|
| 7 | P | Present 位元 |
| 6-5 | DPL | 描述符特權級別 (3 = 允許 Ring 3 觸發) |
| 4 | 0 | 固定為 0 |
| 3-0 | Type | 閘道類型 (0xE = 32-bit Interrupt Gate) |

### 設置 INT 0x80 處理程序

```asm
; IDT entry for INT 0x80 (System Call)
idt_entry_80:
    dw syscall_handler & 0xFFFF      ; Offset 15:0
    dw 0x08                           ; Kernel Code Segment
    db 0x00                           ; Reserved
    db 0xEE                           ; Flags: P=1, DPL=3, Type=0xE
    dw (syscall_handler >> 16) & 0xFFFF  ; Offset 31:16
```

---

## 3. CPU 狀態切換細節

### INT 0x80 執行時 CPU 自動操作

1. **將當前 SS 和 ESP 推入 Kernel Stack** (如果發生特權級別變化)
2. **推入 EFLAGS**
3. **推入 CS 和 EIP**
4. **載入新的 CS 和 EIP** (從 IDT)
5. **載入新的 SS 和 ESP** (從 TSS)

### Kernel Stack 狀態

```
高位址
┌─────────────────┐
│    User SS      │ ← 原來的堆疊段
├─────────────────┤
│    User ESP     │ ← 原來的堆疊指標
├─────────────────┤
│    EFLAGS       │ ← 原來的標誌暫存器
├─────────────────┤
│    User CS      │ ← 原來的代碼段
├─────────────────┤
│    User EIP     │ ← 返回位址
├─────────────────┤
│  Error Code     │ ← 部分中斷會推入
├─────────────────┤
│   ... 更多      │ ← 處理程序保存的暫存器
└─────────────────┘
低位址
```

---

## 4. 系統調用處理程序範例

```asm
syscall_handler:
    ; 保存所有暫存器
    push eax
    push ebx
    push ecx
    push edx
    push esi
    push edi
    push ebp
    
    push ds
    push es
    push fs
    push gs
    
    ; 設置 Kernel 資料段
    mov ax, 0x10
    mov ds, ax
    mov es, ax
    
    ; 驗證系統調用號
    cmp eax, NR_syscalls
    jae invalid_syscall
    
    ; 查找並呼叫系統調用
    ; sys_call_table[eax](ebx, ecx, edx, esi, edi)
    push edi
    push esi
    push edx
    push ecx
    push ebx
    call [sys_call_table + eax * 4]
    add esp, 20
    
    ; 返回值在 eax
    mov [esp + 40], eax    ; 覆蓋保存的 eax
    
    jmp syscall_return

invalid_syscall:
    mov dword [esp + 40], -ENOSYS

syscall_return:
    ; 恢復暫存器
    pop gs
    pop fs
    pop es
    pop ds
    
    pop ebp
    pop edi
    pop esi
    pop edx
    pop ecx
    pop ebx
    pop eax
    
    ; 返回 User Space
    iret
```

---

## 5. 現代系統調用機制 (SYSENTER/SYSCALL)

### 為什麼不用 INT 0x80?

- INT 指令需要查詢 IDT，較慢
- 需要進行完整的中斷處理流程
- SYSENTER/SYSCALL 是專門為系統調用設計的快速路徑

### SYSENTER (Intel)

```
設置 MSR 暫存器:
- SYSENTER_CS_MSR (0x174): Kernel CS
- SYSENTER_ESP_MSR (0x175): Kernel ESP  
- SYSENTER_EIP_MSR (0x176): 系統調用入口點

執行 SYSENTER 時:
- CS = SYSENTER_CS_MSR
- EIP = SYSENTER_EIP_MSR
- SS = SYSENTER_CS_MSR + 8
- ESP = SYSENTER_ESP_MSR
```

### SYSCALL (AMD64)

```
設置 MSR 暫存器:
- STAR MSR: 包含 Kernel/User 段選擇子
- LSTAR MSR: 系統調用入口點
- SFMASK MSR: 要清除的 RFLAGS 位元

執行 SYSCALL 時:
- RCX = 返回 RIP
- R11 = RFLAGS
- RIP = LSTAR
```

---

## 6. 安全性考量

### 參數驗證

1. **位址驗證**: 確保 User 傳入的指標在 User Space 範圍內
2. **權限檢查**: 確保程序有權限執行操作
3. **長度檢查**: 防止緩衝區溢出
4. **競爭條件**: 使用 copy_from_user/copy_to_user

### copy_from_user 範例邏輯

```c
long copy_from_user(void *to, const void __user *from, unsigned long n)
{
    // 驗證位址範圍
    if (!access_ok(VERIFY_READ, from, n))
        return n;  // 返回未複製的位元組數
    
    // 安全地複製資料
    // 如果發生 page fault，會被特殊處理
    return __copy_from_user(to, from, n);
}
```

---

## 參考資料

- [OSDev Wiki - System Calls](https://wiki.osdev.org/System_Calls)
- [Linux Kernel - syscall_64.tbl](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl)
- Intel® 64 and IA-32 Architectures Software Developer's Manual, Volume 2B (SYSCALL/SYSENTER)
- [Linux Insides - System calls](https://0xax.gitbooks.io/linux-insides/content/SysCall/)
