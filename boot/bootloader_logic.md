# 🔧 啟動區塊邏輯分析 (Bootloader Logic Analysis)

## 概述

當電腦通電後，CPU 會從固定位址 `0xFFFFFFF0` 開始執行第一條指令。這份文件分析從 BIOS 到 Kernel 的完整跳轉流程。

---

## 📊 啟動流程圖

```
電源開啟
    │
    ▼
┌─────────────────────────────────────────────────┐
│           CPU 開始執行 (Reset Vector)            │
│           位址: 0xFFFFFFF0                       │
│           模式: Real Mode (16-bit)               │
└─────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────┐
│                 BIOS 初始化                      │
│  • POST (Power-On Self-Test)                    │
│  • 硬體檢測與初始化                               │
│  • 建立中斷向量表 (IVT)                          │
└─────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────┐
│             載入 MBR (Master Boot Record)        │
│  • 從啟動磁碟讀取第一個扇區 (512 bytes)           │
│  • 載入到記憶體位址 0x7C00                        │
│  • 跳轉到 0x7C00 執行                            │
└─────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────┐
│                Bootloader 執行                   │
│  • 設置段暫存器                                  │
│  • 開啟 A20 Gate                                │
│  • 載入 GDT                                     │
│  • 切換到 Protected Mode                        │
└─────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────┐
│               載入 Kernel                       │
│  • 讀取更多磁碟扇區                              │
│  • 跳轉到 Kernel 入口點                          │
└─────────────────────────────────────────────────┘
```

---

## 1. MBR (Master Boot Record) 詳解

### 結構 (512 Bytes)

| 偏移量 | 大小 | 說明 |
|--------|------|------|
| 0x000 | 446 bytes | 引導程式碼區 (Bootstrap Code) |
| 0x1BE | 16 bytes | 分區表條目 1 |
| 0x1CE | 16 bytes | 分區表條目 2 |
| 0x1DE | 16 bytes | 分區表條目 3 |
| 0x1EE | 16 bytes | 分區表條目 4 |
| 0x1FE | 2 bytes | 啟動簽名 (0x55AA) |

### 關鍵暫存器狀態 (進入 MBR 時)

```
CS:IP = 0x0000:0x7C00  ; 程式計數器指向 MBR
DL = 啟動磁碟編號      ; 0x00 = 軟碟, 0x80 = 硬碟
```

---

## 2. A20 Gate 開啟

### 歷史背景

- 8086 CPU 只有 20 條位址線 (A0-A19)
- 定址範圍: 0x00000 - 0xFFFFF (1 MB)
- 80286 有 24 條位址線，但為了相容性，A20 預設被鎖定

### 開啟方式

#### 方法一：透過鍵盤控制器 (8042)

```asm
; 等待鍵盤控制器就緒
wait_8042:
    in      al, 0x64
    test    al, 2
    jnz     wait_8042

; 發送命令：寫入輸出端口
    mov     al, 0xD1
    out     0x64, al

wait_8042_2:
    in      al, 0x64
    test    al, 2
    jnz     wait_8042_2

; 開啟 A20
    mov     al, 0xDF
    out     0x60, al
```

#### 方法二：透過 Fast A20 (現代方式)

```asm
    in      al, 0x92
    or      al, 2
    out     0x92, al
```

---

## 3. GDT (Global Descriptor Table)

### 段描述符結構 (8 bytes)

```
63          56 55   52 51  48 47           40 39          32
┌─────────────┬───────┬──────┬───────────────┬──────────────┐
│  Base 31:24 │ Flags │Limit │ Access Byte   │  Base 23:16  │
│   (8 bits)  │(4 bits)│19:16 │   (8 bits)    │   (8 bits)   │
└─────────────┴───────┴──────┴───────────────┴──────────────┘

31                              16 15                       0
┌─────────────────────────────────┬──────────────────────────┐
│         Base Address 15:0       │      Limit 15:0          │
│            (16 bits)            │        (16 bits)         │
└─────────────────────────────────┴──────────────────────────┘
```

### Access Byte 位元定義

| 位元 | 名稱 | 說明 |
|------|------|------|
| 7 | P (Present) | 段是否存在於記憶體 |
| 6-5 | DPL | 描述符特權級別 (0-3) |
| 4 | S | 描述符類型 (0=系統, 1=代碼/資料) |
| 3 | E | 可執行位元 |
| 2 | DC | 方向/一致性位元 |
| 1 | RW | 讀寫/讀執行位元 |
| 0 | A | 已存取位元 |

### 基本 GDT 設置

```asm
gdt_start:
    ; 空描述符 (必須)
    dq 0x0000000000000000

    ; 代碼段描述符
    ; Base=0, Limit=0xFFFFF, Access=0x9A, Flags=0xC
    dw 0xFFFF       ; Limit 15:0
    dw 0x0000       ; Base 15:0
    db 0x00         ; Base 23:16
    db 0x9A         ; Access: P=1, DPL=0, S=1, E=1, RW=1
    db 0xCF         ; Flags=0xC, Limit 19:16=0xF
    db 0x00         ; Base 31:24

    ; 資料段描述符
    ; Base=0, Limit=0xFFFFF, Access=0x92, Flags=0xC
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 0x92         ; Access: P=1, DPL=0, S=1, E=0, RW=1
    db 0xCF
    db 0x00

gdt_end:

gdt_descriptor:
    dw gdt_end - gdt_start - 1  ; GDT 大小
    dd gdt_start                 ; GDT 位址
```

---

## 4. 切換到 Protected Mode

### 步驟

1. **關閉中斷** (防止切換過程被中斷)
2. **開啟 A20 Gate**
3. **載入 GDT**
4. **設置 CR0 暫存器的 PE 位元**
5. **遠跳轉到 32 位元代碼**

### 程式碼範例

```asm
switch_to_pm:
    cli                     ; 關閉中斷
    
    lgdt [gdt_descriptor]   ; 載入 GDT
    
    mov eax, cr0
    or  eax, 0x1            ; 設置 PE 位元
    mov cr0, eax
    
    ; 遠跳轉，刷新 CPU pipeline
    jmp 0x08:protected_mode_entry

[bits 32]
protected_mode_entry:
    ; 現在處於 32 位元 Protected Mode
    mov ax, 0x10            ; 資料段選擇子
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    
    mov ebp, 0x90000        ; 設置堆疊
    mov esp, ebp
    
    ; 跳轉到 Kernel
    call kernel_main
```

---

## 5. 重要暫存器總結

| 暫存器 | 用途 |
|--------|------|
| CR0 | 控制暫存器，PE 位元控制保護模式 |
| CR3 | 頁目錄基址暫存器 (分頁啟用後) |
| GDTR | GDT 暫存器，存儲 GDT 位置與大小 |
| IDTR | IDT 暫存器，存儲 IDT 位置與大小 |
| CS | 代碼段選擇子 |
| DS/ES/FS/GS/SS | 資料段選擇子 |

---

## 參考資料

- [OSDev Wiki - Bootloader](https://wiki.osdev.org/Bootloader)
- [OSDev Wiki - A20 Line](https://wiki.osdev.org/A20_Line)
- [OSDev Wiki - GDT](https://wiki.osdev.org/Global_Descriptor_Table)
- Intel® 64 and IA-32 Architectures Software Developer's Manual, Volume 3A
