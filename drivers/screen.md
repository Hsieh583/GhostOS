# 🖥️ 螢幕驅動抽象層 (Screen/VGA Driver)

## 概述

VGA 文字模式是最基本的輸出方式。本文檔分析 VGA 文字模式的工作原理和驅動實現。

---

## 1. VGA 文字模式基礎

### 記憶體映射

VGA 文字模式使用記憶體映射 I/O：
- **基址**: `0xB8000`
- **大小**: 4000 bytes (80 × 25 × 2)
- **每個字元**: 2 bytes

### 字元格式

```
位元 15-12: 背景顏色
位元 11-8:  前景顏色
位元 7-0:   ASCII 字元

┌────────────────┬────────────────┐
│   Attribute    │   Character    │
│    (1 byte)    │    (1 byte)    │
├────────────────┼────────────────┤
│ BG (4) | FG(4) │   ASCII Code   │
└────────────────┴────────────────┘
```

### 顏色表

| 值 | 顏色 | 值 | 顏色 |
|----|------|----|----|
| 0 | 黑色 | 8 | 深灰 |
| 1 | 藍色 | 9 | 淺藍 |
| 2 | 綠色 | 10 | 淺綠 |
| 3 | 青色 | 11 | 淺青 |
| 4 | 紅色 | 12 | 淺紅 |
| 5 | 洋紅 | 13 | 淺洋紅 |
| 6 | 棕色 | 14 | 黃色 |
| 7 | 淺灰 | 15 | 白色 |

---

## 2. 驅動程式架構

```c
// VGA 常數定義
#define VGA_WIDTH  80
#define VGA_HEIGHT 25
#define VGA_MEMORY 0xB8000

// 顏色定義
typedef enum {
    VGA_BLACK = 0,
    VGA_BLUE = 1,
    VGA_GREEN = 2,
    VGA_CYAN = 3,
    VGA_RED = 4,
    VGA_MAGENTA = 5,
    VGA_BROWN = 6,
    VGA_LIGHT_GREY = 7,
    VGA_DARK_GREY = 8,
    VGA_LIGHT_BLUE = 9,
    VGA_LIGHT_GREEN = 10,
    VGA_LIGHT_CYAN = 11,
    VGA_LIGHT_RED = 12,
    VGA_LIGHT_MAGENTA = 13,
    VGA_YELLOW = 14,
    VGA_WHITE = 15,
} vga_color_t;

// 螢幕狀態
typedef struct {
    uint16_t* buffer;
    size_t row;
    size_t column;
    uint8_t color;
} vga_state_t;

static vga_state_t vga;

// 建立顏色屬性
static inline uint8_t vga_entry_color(vga_color_t fg, vga_color_t bg) {
    return fg | (bg << 4);
}

// 建立螢幕項目
static inline uint16_t vga_entry(unsigned char c, uint8_t color) {
    return (uint16_t) c | ((uint16_t) color << 8);
}

// 初始化
void vga_init(void) {
    vga.buffer = (uint16_t*) VGA_MEMORY;
    vga.row = 0;
    vga.column = 0;
    vga.color = vga_entry_color(VGA_LIGHT_GREY, VGA_BLACK);
    
    // 清空螢幕
    vga_clear();
}

// 清空螢幕
void vga_clear(void) {
    uint16_t blank = vga_entry(' ', vga.color);
    
    for (size_t y = 0; y < VGA_HEIGHT; y++) {
        for (size_t x = 0; x < VGA_WIDTH; x++) {
            vga.buffer[y * VGA_WIDTH + x] = blank;
        }
    }
    
    vga.row = 0;
    vga.column = 0;
    vga_update_cursor();
}
```

---

## 3. 字元輸出

```c
// 輸出單一字元
void vga_putchar(char c) {
    // 處理特殊字元
    switch (c) {
        case '\n':
            vga.column = 0;
            vga.row++;
            break;
        
        case '\r':
            vga.column = 0;
            break;
        
        case '\t':
            vga.column = (vga.column + 8) & ~7;
            break;
        
        case '\b':
            if (vga.column > 0) {
                vga.column--;
                vga_putentryat(' ', vga.color, vga.column, vga.row);
            }
            break;
        
        default:
            vga_putentryat(c, vga.color, vga.column, vga.row);
            vga.column++;
            break;
    }
    
    // 換行處理
    if (vga.column >= VGA_WIDTH) {
        vga.column = 0;
        vga.row++;
    }
    
    // 捲動處理
    if (vga.row >= VGA_HEIGHT) {
        vga_scroll();
    }
    
    vga_update_cursor();
}

// 在指定位置輸出字元
void vga_putentryat(char c, uint8_t color, size_t x, size_t y) {
    const size_t index = y * VGA_WIDTH + x;
    vga.buffer[index] = vga_entry(c, color);
}

// 輸出字串
void vga_puts(const char* str) {
    while (*str) {
        vga_putchar(*str++);
    }
}
```

---

## 4. 捲動功能

```c
void vga_scroll(void) {
    // 將所有行向上移動一行
    for (size_t y = 0; y < VGA_HEIGHT - 1; y++) {
        for (size_t x = 0; x < VGA_WIDTH; x++) {
            vga.buffer[y * VGA_WIDTH + x] = 
                vga.buffer[(y + 1) * VGA_WIDTH + x];
        }
    }
    
    // 清空最後一行
    uint16_t blank = vga_entry(' ', vga.color);
    for (size_t x = 0; x < VGA_WIDTH; x++) {
        vga.buffer[(VGA_HEIGHT - 1) * VGA_WIDTH + x] = blank;
    }
    
    vga.row = VGA_HEIGHT - 1;
}
```

---

## 5. 游標控制

### CRT 控制暫存器

| 暫存器 | 索引 | 說明 |
|--------|------|------|
| Cursor Location High | 0x0E | 游標位置高位元組 |
| Cursor Location Low | 0x0F | 游標位置低位元組 |
| Cursor Start | 0x0A | 游標起始掃描線 |
| Cursor End | 0x0B | 游標結束掃描線 |

```c
// I/O 端口
#define VGA_CTRL_REGISTER 0x3D4
#define VGA_DATA_REGISTER 0x3D5

// 更新游標位置
void vga_update_cursor(void) {
    uint16_t pos = vga.row * VGA_WIDTH + vga.column;
    
    outb(VGA_CTRL_REGISTER, 0x0F);
    outb(VGA_DATA_REGISTER, (uint8_t)(pos & 0xFF));
    
    outb(VGA_CTRL_REGISTER, 0x0E);
    outb(VGA_DATA_REGISTER, (uint8_t)((pos >> 8) & 0xFF));
}

// 設置游標位置
void vga_set_cursor(size_t x, size_t y) {
    if (x >= VGA_WIDTH) x = VGA_WIDTH - 1;
    if (y >= VGA_HEIGHT) y = VGA_HEIGHT - 1;
    
    vga.column = x;
    vga.row = y;
    vga_update_cursor();
}

// 啟用/禁用游標
void vga_enable_cursor(bool enable) {
    if (enable) {
        // 設置游標外觀 (起始掃描線 14, 結束掃描線 15)
        outb(VGA_CTRL_REGISTER, 0x0A);
        outb(VGA_DATA_REGISTER, 0x0E);
        
        outb(VGA_CTRL_REGISTER, 0x0B);
        outb(VGA_DATA_REGISTER, 0x0F);
    } else {
        // 禁用游標
        outb(VGA_CTRL_REGISTER, 0x0A);
        outb(VGA_DATA_REGISTER, 0x20);  // 位元 5 = 禁用游標
    }
}
```

---

## 6. 格式化輸出

```c
// 簡化的 printf 實現
void vga_printf(const char* format, ...) {
    va_list args;
    va_start(args, format);
    
    while (*format) {
        if (*format == '%') {
            format++;
            switch (*format) {
                case 'd':
                case 'i': {
                    int num = va_arg(args, int);
                    vga_print_int(num);
                    break;
                }
                case 'x': {
                    unsigned int num = va_arg(args, unsigned int);
                    vga_print_hex(num);
                    break;
                }
                case 's': {
                    const char* str = va_arg(args, const char*);
                    vga_puts(str);
                    break;
                }
                case 'c': {
                    char c = (char) va_arg(args, int);
                    vga_putchar(c);
                    break;
                }
                case '%':
                    vga_putchar('%');
                    break;
            }
        } else {
            vga_putchar(*format);
        }
        format++;
    }
    
    va_end(args);
}

// 輸出整數
static void vga_print_int(int num) {
    if (num < 0) {
        vga_putchar('-');
        num = -num;
    }
    
    if (num >= 10) {
        vga_print_int(num / 10);
    }
    
    vga_putchar('0' + (num % 10));
}

// 輸出十六進制
static void vga_print_hex(unsigned int num) {
    static const char hex_chars[] = "0123456789ABCDEF";
    
    vga_puts("0x");
    
    for (int i = 28; i >= 0; i -= 4) {
        vga_putchar(hex_chars[(num >> i) & 0xF]);
    }
}
```

---

## 7. 使用範例

```c
void kernel_main(void) {
    // 初始化 VGA
    vga_init();
    
    // 設置顏色
    vga.color = vga_entry_color(VGA_GREEN, VGA_BLACK);
    
    // 輸出歡迎訊息
    vga_puts("Welcome to GhostOS!\n");
    vga_puts("===================\n\n");
    
    // 恢復預設顏色
    vga.color = vga_entry_color(VGA_LIGHT_GREY, VGA_BLACK);
    
    // 格式化輸出
    vga_printf("Memory: %d MB\n", 256);
    vga_printf("CPU: %s\n", "Intel x86");
    vga_printf("Address: %x\n", 0xB8000);
    
    // 彩色輸出
    vga.color = vga_entry_color(VGA_RED, VGA_BLACK);
    vga_puts("\n[ERROR] ");
    vga.color = vga_entry_color(VGA_LIGHT_GREY, VGA_BLACK);
    vga_puts("This is an error message.\n");
    
    vga.color = vga_entry_color(VGA_YELLOW, VGA_BLACK);
    vga_puts("[WARN] ");
    vga.color = vga_entry_color(VGA_LIGHT_GREY, VGA_BLACK);
    vga_puts("This is a warning message.\n");
}
```

---

## 參考資料

- [OSDev Wiki - Text Mode](https://wiki.osdev.org/Text_mode)
- [OSDev Wiki - VGA Hardware](https://wiki.osdev.org/VGA_Hardware)
- [OSDev Wiki - Text Mode Cursor](https://wiki.osdev.org/Text_Mode_Cursor)
