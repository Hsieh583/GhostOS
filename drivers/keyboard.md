# 🎹 鍵盤驅動抽象層 (Keyboard Driver)

## 概述

鍵盤驅動是最基本的輸入設備驅動之一。本文檔分析 PS/2 鍵盤的工作原理和驅動實現邏輯。

---

## 1. PS/2 鍵盤硬體架構

### I/O 端口

| 端口 | 用途 | 方向 |
|------|------|------|
| 0x60 | 資料端口 | 讀/寫 |
| 0x64 | 狀態/命令端口 | 讀(狀態)/寫(命令) |

### 狀態暫存器 (0x64 讀取)

```
位元 7: 奇偶校驗錯誤
位元 6: 逾時錯誤
位元 5: 輔助輸出緩衝區滿 (滑鼠資料)
位元 4: 鍵盤鎖定 (0=鎖定)
位元 3: 命令/資料 (0=資料, 1=命令)
位元 2: 系統標誌
位元 1: 輸入緩衝區滿 (CPU→控制器)
位元 0: 輸出緩衝區滿 (控制器→CPU)
```

---

## 2. 中斷處理流程

```
┌─────────────────────────────────────────┐
│            按下按鍵                      │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│    鍵盤控制器產生 IRQ 1                  │
│    (通過 8259 PIC 傳遞)                  │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│    CPU 接收 INT 33 (IRQ1 + 32)          │
│    跳轉到 IDT[33] 處理程序               │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│         keyboard_interrupt_handler       │
│    1. 從 0x60 讀取掃描碼                 │
│    2. 處理掃描碼                         │
│    3. 發送 EOI 到 PIC                   │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│         掃描碼 → ASCII 轉換              │
│         放入鍵盤緩衝區                   │
└─────────────────────────────────────────┘
```

---

## 3. 掃描碼 (Scan Code)

### Set 1 掃描碼範例

| 按鍵 | Make Code | Break Code |
|------|-----------|------------|
| A | 0x1E | 0x9E |
| B | 0x30 | 0xB0 |
| Enter | 0x1C | 0x9C |
| Space | 0x39 | 0xB9 |
| Left Shift | 0x2A | 0xAA |
| Left Ctrl | 0x1D | 0x9D |
| Esc | 0x01 | 0x81 |

### 掃描碼映射表

```c
// 簡化的掃描碼到 ASCII 映射
static char scancode_to_ascii[128] = {
    0,   27, '1', '2', '3', '4', '5', '6', '7', '8', '9', '0', '-', '=', '\b',
    '\t', 'q', 'w', 'e', 'r', 't', 'y', 'u', 'i', 'o', 'p', '[', ']', '\n',
    0,   'a', 's', 'd', 'f', 'g', 'h', 'j', 'k', 'l', ';', '\'', '`',
    0,   '\\', 'z', 'x', 'c', 'v', 'b', 'n', 'm', ',', '.', '/', 0,
    '*', 0, ' ',
    // ... 更多
};
```

---

## 4. 驅動程式架構

```c
// 鍵盤狀態
typedef struct {
    bool shift_pressed;
    bool ctrl_pressed;
    bool alt_pressed;
    bool caps_lock;
    bool num_lock;
    bool scroll_lock;
} keyboard_state_t;

// 鍵盤緩衝區
#define KEYBOARD_BUFFER_SIZE 256
typedef struct {
    char buffer[KEYBOARD_BUFFER_SIZE];
    int head;
    int tail;
} keyboard_buffer_t;

static keyboard_state_t kb_state;
static keyboard_buffer_t kb_buffer;

// 讀取掃描碼
static uint8_t keyboard_read_scancode(void) {
    // 等待輸出緩衝區有資料
    while (!(inb(0x64) & 0x01));
    return inb(0x60);
}

// 中斷處理程序
void keyboard_interrupt_handler(void) {
    uint8_t scancode = keyboard_read_scancode();
    
    // 處理特殊鍵
    switch (scancode) {
        case 0x2A:  // Left Shift pressed
        case 0x36:  // Right Shift pressed
            kb_state.shift_pressed = true;
            goto eoi;
        case 0xAA:  // Left Shift released
        case 0xB6:  // Right Shift released
            kb_state.shift_pressed = false;
            goto eoi;
        case 0x1D:  // Ctrl pressed
            kb_state.ctrl_pressed = true;
            goto eoi;
        case 0x9D:  // Ctrl released
            kb_state.ctrl_pressed = false;
            goto eoi;
        case 0x3A:  // Caps Lock
            kb_state.caps_lock = !kb_state.caps_lock;
            keyboard_set_leds();
            goto eoi;
    }
    
    // 忽略 break codes (最高位元為 1)
    if (scancode & 0x80) {
        goto eoi;
    }
    
    // 轉換為 ASCII
    char ascii = scancode_to_ascii[scancode];
    if (ascii == 0) {
        goto eoi;
    }
    
    // 處理 Shift
    if (kb_state.shift_pressed) {
        ascii = shift_map[scancode];
    }
    
    // 處理 Caps Lock (只影響字母)
    if (kb_state.caps_lock && ascii >= 'a' && ascii <= 'z') {
        if (!kb_state.shift_pressed) {
            ascii -= 32;  // 轉大寫
        }
    } else if (kb_state.caps_lock && ascii >= 'A' && ascii <= 'Z') {
        if (!kb_state.shift_pressed) {
            ascii += 32;  // 轉小寫
        }
    }
    
    // 放入緩衝區
    keyboard_buffer_push(ascii);
    
eoi:
    // 發送 End of Interrupt
    outb(0x20, 0x20);
}

// 讀取字元 (阻塞式)
char keyboard_getchar(void) {
    while (keyboard_buffer_empty()) {
        // 等待 (可以用 HLT 節省 CPU)
        asm volatile("hlt");
    }
    return keyboard_buffer_pop();
}
```

---

## 5. 初始化流程

```c
void keyboard_init(void) {
    // 1. 禁用鍵盤
    keyboard_wait_input();
    outb(0x64, 0xAD);  // 禁用第一個 PS/2 端口
    
    // 2. 清空輸出緩衝區
    inb(0x60);
    
    // 3. 設置控制器配置
    keyboard_wait_input();
    outb(0x64, 0x20);  // 讀取配置
    keyboard_wait_output();
    uint8_t config = inb(0x60);
    
    config |= 0x01;    // 啟用 IRQ1
    config &= ~0x10;   // 啟用第一個 PS/2 端口時鐘
    
    keyboard_wait_input();
    outb(0x64, 0x60);  // 寫入配置
    keyboard_wait_input();
    outb(0x60, config);
    
    // 4. 啟用鍵盤
    keyboard_wait_input();
    outb(0x64, 0xAE);  // 啟用第一個 PS/2 端口
    
    // 5. 重置鍵盤
    keyboard_wait_input();
    outb(0x60, 0xFF);  // 重置命令
    // 等待 ACK (0xFA) 和自檢結果 (0xAA)
    
    // 6. 設置 LED
    kb_state.caps_lock = false;
    kb_state.num_lock = true;
    kb_state.scroll_lock = false;
    keyboard_set_leds();
    
    // 7. 註冊中斷處理程序
    register_interrupt_handler(33, keyboard_interrupt_handler);
    
    // 8. 初始化緩衝區
    kb_buffer.head = 0;
    kb_buffer.tail = 0;
}
```

---

## 6. LED 控制

```c
void keyboard_set_leds(void) {
    uint8_t leds = 0;
    
    if (kb_state.scroll_lock) leds |= 0x01;
    if (kb_state.num_lock)    leds |= 0x02;
    if (kb_state.caps_lock)   leds |= 0x04;
    
    // 發送設置 LED 命令
    keyboard_wait_input();
    outb(0x60, 0xED);
    
    // 等待 ACK
    keyboard_wait_output();
    inb(0x60);  // 應該是 0xFA
    
    // 發送 LED 狀態
    keyboard_wait_input();
    outb(0x60, leds);
    
    // 等待 ACK
    keyboard_wait_output();
    inb(0x60);
}
```

---

## 參考資料

- [OSDev Wiki - PS/2 Keyboard](https://wiki.osdev.org/PS/2_Keyboard)
- [OSDev Wiki - 8042 Controller](https://wiki.osdev.org/%228042%22_PS/2_Controller)
- IBM Personal System/2 Hardware Interface Technical Reference
